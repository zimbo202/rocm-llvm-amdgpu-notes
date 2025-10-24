# GCNRewritePartialRegUses.cpp 代码功能详解

## 1. Pass的主要功能概括

<a name="ref-block_0"></a>**GCNRewritePartialRegUses** 是一个LLVM机器函数优化Pass，其主要功能是**重写部分使用的超大寄存器，将其转换为最小尺寸的寄存器**。 llvm-project:8-29[<sup>↗</sup>](#block_0) 

**作用**：
- 在 `RenameIndependentSubregs` pass 之后运行，处理该pass留下的大型部分使用寄存器
- 将大型寄存器（如VReg_1024）中只使用了部分子寄存器的情况，缩减为最小必要尺寸的寄存器（如VReg_128）
- 通过子寄存器索引的右移，实现寄存器的紧凑化

**效果**：
- 避免在寄存器压力计算时需要追踪子寄存器lane掩码
- 为不了解lane掩码的代码创建更多优化机会
- 减少寄存器分配的复杂度和内存占用

## 2. 主要功能实现步骤提取

通过遍历代码文件，主要功能由以下步骤和子功能实现：

### 核心执行流程：
1. **主入口执行** (`run` 方法) - 遍历所有虚拟寄存器
2. **单个寄存器重写** (`rewriteReg` 方法) - 处理每个部分使用的寄存器
3. **最小寄存器类查找** (`getMinSizeReg` 方法) - 计算最优寄存器类
4. **右移子寄存器映射** (`getRegClassWithShiftedSubregs` 方法) - 查找右移后的寄存器类
5. **活跃区间更新** (`updateLiveIntervals` 方法) - 更新寄存器活跃信息

### 辅助工具函数：
6. **子寄存器右移** (`shiftSubReg` 方法)
7. **子寄存器索引查找** (`getSubReg` 方法)
8. **超级寄存器类掩码获取** (`getSuperRegClassMask` 方法)
9. **可分配对齐寄存器类掩码** (`getAllocatableAndAlignedRegClassMask` 方法)

## 3. 各步骤的详细描述与分析

### 步骤1: 主入口执行 (`run` 方法)

<a name="ref-block_10"></a>这是Pass的主入口函数，负责初始化并遍历所有虚拟寄存器。 llvm-project:435-444[<sup>↗</sup>](#block_10) 

**功能**：
- 初始化MRI（机器寄存器信息）、TRI（目标寄存器信息）、TII（目标指令信息）
- 遍历函数中的所有虚拟寄存器
- 对每个寄存器调用 `rewriteReg` 方法尝试重写
- 返回是否有任何修改发生

### 步骤2: 单个寄存器重写 (`rewriteReg` 方法)

<a name="ref-block_9"></a>这是核心重写逻辑，判断寄存器是否可以优化并执行重写。 llvm-project:388-433[<sup>↗</sup>](#block_9) 

**功能**：
- 收集寄存器的所有使用的子寄存器（存入 `SubRegs` 映射表）
- 如果整个寄存器都被使用（无子寄存器），则无法优化，返回false
- 调用 `getMinSizeReg` 查找更小的寄存器类
- 如果找到更优的寄存器类，创建新的虚拟寄存器
- 更新所有使用该寄存器的机器操作数，将旧寄存器和子寄存器索引替换为新的
- 如果存在活跃区间信息，调用 `updateLiveIntervals` 更新

### 步骤3: 最小寄存器类查找 (`getMinSizeReg` 方法)

<a name="ref-block_7"></a>计算能容纳所有使用子寄存器的最小寄存器类。 llvm-project:279-329[<sup>↗</sup>](#block_7) 

**功能**：
- 计算所有使用的子寄存器的偏移范围（`Offset` 到 `End`）
- **情况1：存在覆盖子寄存器**：如果某个子寄存器正好覆盖整个使用范围，将该子寄存器右移到最右侧位置
- **情况2：无覆盖子寄存器**：
  - 找到所有子寄存器中对齐要求最高的（`MaxAlign`）
  - 计算第一个具有最大对齐要求的子寄存器的偏移
  - 计算该子寄存器右移后应该的位置（保持对齐约束）
  - 计算总的右移量（`RShift`）
- 调用 `getRegClassWithShiftedSubregs` 查找满足条件的寄存器类

### 步骤4: 右移子寄存器映射 (`getRegClassWithShiftedSubregs` 方法)

<a name="ref-block_6"></a>给定右移量，查找满足所有约束的最小寄存器类。 llvm-project:205-277[<sup>↗</sup>](#block_6) 

**功能**：
- 获取原寄存器类的对齐要求
- 初始化候选寄存器类位向量（必须可分配且满足对齐要求）
- 对每个需要右移的子寄存器：
  - 计算右移后的新子寄存器索引
  - 如果是覆盖子寄存器，新索引设为 `NoSubRegister`（表示整个寄存器）
  - 获取该子寄存器对应的寄存器类
  - 根据新子寄存器索引获取超级寄存器类掩码
  - 与候选集求交集，逐步缩小候选范围
- 从候选集中选择寄存器位宽最小的寄存器类
- 验证选择的寄存器类满足所有约束条件

### 步骤5: 活跃区间更新 (`updateLiveIntervals` 方法)

<a name="ref-block_8"></a>更新寄存器的活跃区间分析信息，确保后续分析正确。 llvm-project:333-386[<sup>↗</sup>](#block_8) 

**功能**：
- 如果旧寄存器没有活跃区间信息，直接返回
- 为新寄存器创建空的活跃区间
- 遍历旧寄存器的所有子范围（subranges）：
  - 根据lane掩码找到对应的子寄存器映射
  - 如果新子寄存器索引非零，创建对应的子范围
  - 如果新子寄存器索引为零（覆盖子寄存器），将其设置为主范围
- 处理子范围不精确匹配的情况：如果子范围与使用的子寄存器不完全对应，重新计算整个活跃区间
- 移除旧寄存器的活跃区间

### 步骤6: 子寄存器右移 (`shiftSubReg` 方法)

<a name="ref-block_3"></a>计算子寄存器右移指定位数后的新索引。 llvm-project:167-171[<sup>↗</sup>](#block_3) 

**功能**：
- 获取原子寄存器的偏移量
- 减去右移量得到新偏移
- 调用 `getSubReg` 查找具有该偏移和相同大小的子寄存器索引

### 步骤7: 子寄存器索引查找 (`getSubReg` 方法)

<a name="ref-block_2"></a>根据偏移量和大小查找对应的子寄存器索引，使用缓存优化性能。 llvm-project:152-165[<sup>↗</sup>](#block_2) 

**功能**：
- 使用 `{Offset, Size}` 对作为键查找缓存
- 如果缓存未命中，遍历所有子寄存器索引
- 找到偏移和大小都匹配的子寄存器索引
- 将结果存入缓存并返回

### 步骤8: 超级寄存器类掩码获取 (`getSuperRegClassMask` 方法)

<a name="ref-block_4"></a>获取通过指定子寄存器索引投影到给定寄存器类的所有寄存器类位掩码。 llvm-project:173-186[<sup>↗</sup>](#block_4) 

**功能**：
- 使用 `{RC, SubRegIdx}` 对作为键查找缓存
- 如果缓存未命中，使用 `SuperRegClassIterator` 迭代超级寄存器类
- 找到通过指定子寄存器索引投影的寄存器类掩码
- 将结果存入缓存并返回

### 步骤9: 可分配对齐寄存器类掩码 (`getAllocatableAndAlignedRegClassMask` 方法)

<a name="ref-block_5"></a>获取所有可分配且满足特定对齐要求的寄存器类的位向量。 llvm-project:188-203[<sup>↗</sup>](#block_5) 

**功能**：
- 使用对齐位数作为键查找缓存
- 如果缓存未命中，遍历所有寄存器类
- 检查每个寄存器类是否可分配且满足对齐要求
- 构建位向量并存入缓存

## 4. 步骤与子功能之间的关系

### 主调用关系链：

```
run (步骤1)
  └─> rewriteReg (步骤2) - 对每个虚拟寄存器
       ├─> getMinSizeReg (步骤3) - 查找最小寄存器类
       │    └─> getRegClassWithShiftedSubregs (步骤4)
       │         ├─> shiftSubReg (步骤6) - 计算右移后的子寄存器
       │         │    └─> getSubReg (步骤7) - 查找子寄存器索引
       │         ├─> getSuperRegClassMask (步骤8) - 获取候选寄存器类
       │         └─> getAllocatableAndAlignedRegClassMask (步骤9) - 过滤候选集
       └─> updateLiveIntervals (步骤5) - 更新活跃区间
```

### 功能层次关系：

1. **顶层控制流**：`run` 方法是整个Pass的入口，控制整体执行流程

2. **核心算法层**：
   - `rewriteReg` 实现单个寄存器的重写逻辑
   - `getMinSizeReg` 实现寄存器优化的核心算法（确定右移策略）
   - `getRegClassWithShiftedSubregs` 执行实际的寄存器类查找和验证

3. **辅助工具层**：
   - `shiftSubReg` 和 `getSubReg` 提供子寄存器索引计算
   - `getSuperRegClassMask` 和 `getAllocatableAndAlignedRegClassMask` 提供寄存器类查询和过滤

4. **维护更新层**：`updateLiveIntervals` 确保优化后分析信息的一致性

### 数据流关系：

1. **输入**：部分使用的虚拟寄存器及其子寄存器使用模式
2. **计算**：通过对齐和偏移计算确定最优右移量
3. **查找**：在寄存器类空间中搜索满足约束的最小寄存器类
4. **转换**：更新所有机器操作数的寄存器引用
5. **输出**：优化后的寄存器分配和更新的活跃区间

### 优化策略：

<a name="ref-block_1"></a>所有辅助函数（步骤6-9）都使用**缓存机制**来避免重复计算，这对于处理大量寄存器时的性能至关重要。 llvm-project:102-123[<sup>↗</sup>](#block_1) 

## Notes

- 这个Pass专门为AMD GPU架构设计，利用了AMDGPU寄存器结构的特点
- Pass必须在 `RenameIndependentSubregs` 之后运行，因为它依赖于该Pass产生的寄存器使用模式
- 优化的关键在于子寄存器的"右移"操作，通过调整子寄存器索引来使用更小的寄存器类
- 对齐约束是算法的重要考虑因素，确保生成的寄存器分配满足硬件要求
- 活跃区间的更新是可选的（取决于LiveIntervals分析是否可用），但对于后续优化pass的正确性很重要
### Citations
<a name="block_0"></a>**File:** llvm/lib/Target/AMDGPU/GCNRewritePartialRegUses.cpp (L8-29) [<sup>↩</sup>](#ref-block_0)
```cpp
/// \file
/// RenameIndependentSubregs pass leaves large partially used super registers,
/// for example:
///   undef %0.sub4:VReg_1024 = ...
///   %0.sub5:VReg_1024 = ...
///   %0.sub6:VReg_1024 = ...
///   %0.sub7:VReg_1024 = ...
///   use %0.sub4_sub5_sub6_sub7
///   use %0.sub6_sub7
///
/// GCNRewritePartialRegUses goes right after RenameIndependentSubregs and
/// rewrites such partially used super registers with registers of minimal size:
///   undef %0.sub0:VReg_128 = ...
///   %0.sub1:VReg_128 = ...
///   %0.sub2:VReg_128 = ...
///   %0.sub3:VReg_128 = ...
///   use %0.sub0_sub1_sub2_sub3
///   use %0.sub2_sub3
///
/// This allows to avoid subreg lanemasks tracking during register pressure
/// calculation and creates more possibilities for the code unaware of lanemasks
//===----------------------------------------------------------------------===//
```
<a name="block_1"></a>**File:** llvm/lib/Target/AMDGPU/GCNRewritePartialRegUses.cpp (L102-123) [<sup>↩</sup>](#ref-block_1)
```cpp
  /// Cache for getSubReg method: {Offset, Size} -> SubReg index.
  mutable SmallDenseMap<std::pair<unsigned, unsigned>, unsigned> SubRegs;

  /// Return bit mask that contains all register classes that are projected into
  /// RC by SubRegIdx. The result is cached in SuperRegMasks data-member.
  const uint32_t *getSuperRegClassMask(const TargetRegisterClass *RC,
                                       unsigned SubRegIdx) const;

  /// Cache for getSuperRegClassMask method: { RC, SubRegIdx } -> Class bitmask.
  mutable SmallDenseMap<std::pair<const TargetRegisterClass *, unsigned>,
                        const uint32_t *>
      SuperRegMasks;

  /// Return bitmask containing all allocatable register classes with registers
  /// aligned at AlignNumBits. The result is cached in
  /// AllocatableAndAlignedRegClassMasks data-member.
  const BitVector &
  getAllocatableAndAlignedRegClassMask(unsigned AlignNumBits) const;

  /// Cache for getAllocatableAndAlignedRegClassMask method:
  ///   AlignNumBits -> Class bitmask.
  mutable SmallDenseMap<unsigned, BitVector> AllocatableAndAlignedRegClassMasks;
```
<a name="block_2"></a>**File:** llvm/lib/Target/AMDGPU/GCNRewritePartialRegUses.cpp (L152-165) [<sup>↩</sup>](#ref-block_2)
```cpp
unsigned GCNRewritePartialRegUsesImpl::getSubReg(unsigned Offset,
                                                 unsigned Size) const {
  const auto [I, Inserted] = SubRegs.try_emplace({Offset, Size}, 0);
  if (Inserted) {
    for (unsigned Idx = 1, E = TRI->getNumSubRegIndices(); Idx < E; ++Idx) {
      if (TRI->getSubRegIdxOffset(Idx) == Offset &&
          TRI->getSubRegIdxSize(Idx) == Size) {
        I->second = Idx;
        break;
      }
    }
  }
  return I->second;
}
```
<a name="block_3"></a>**File:** llvm/lib/Target/AMDGPU/GCNRewritePartialRegUses.cpp (L167-171) [<sup>↩</sup>](#ref-block_3)
```cpp
unsigned GCNRewritePartialRegUsesImpl::shiftSubReg(unsigned SubReg,
                                                   unsigned RShift) const {
  unsigned Offset = TRI->getSubRegIdxOffset(SubReg) - RShift;
  return getSubReg(Offset, TRI->getSubRegIdxSize(SubReg));
}
```
<a name="block_4"></a>**File:** llvm/lib/Target/AMDGPU/GCNRewritePartialRegUses.cpp (L173-186) [<sup>↩</sup>](#ref-block_4)
```cpp
const uint32_t *GCNRewritePartialRegUsesImpl::getSuperRegClassMask(
    const TargetRegisterClass *RC, unsigned SubRegIdx) const {
  const auto [I, Inserted] =
      SuperRegMasks.try_emplace({RC, SubRegIdx}, nullptr);
  if (Inserted) {
    for (SuperRegClassIterator RCI(RC, TRI); RCI.isValid(); ++RCI) {
      if (RCI.getSubReg() == SubRegIdx) {
        I->second = RCI.getMask();
        break;
      }
    }
  }
  return I->second;
}
```
<a name="block_5"></a>**File:** llvm/lib/Target/AMDGPU/GCNRewritePartialRegUses.cpp (L188-203) [<sup>↩</sup>](#ref-block_5)
```cpp
const BitVector &
GCNRewritePartialRegUsesImpl::getAllocatableAndAlignedRegClassMask(
    unsigned AlignNumBits) const {
  const auto [I, Inserted] =
      AllocatableAndAlignedRegClassMasks.try_emplace(AlignNumBits);
  if (Inserted) {
    BitVector &BV = I->second;
    BV.resize(TRI->getNumRegClasses());
    for (unsigned ClassID = 0; ClassID < TRI->getNumRegClasses(); ++ClassID) {
      auto *RC = TRI->getRegClass(ClassID);
      if (RC->isAllocatable() && TRI->isRegClassAligned(RC, AlignNumBits))
        BV.set(ClassID);
    }
  }
  return I->second;
}
```
<a name="block_6"></a>**File:** llvm/lib/Target/AMDGPU/GCNRewritePartialRegUses.cpp (L205-277) [<sup>↩</sup>](#ref-block_6)
```cpp
const TargetRegisterClass *
GCNRewritePartialRegUsesImpl::getRegClassWithShiftedSubregs(
    const TargetRegisterClass *RC, unsigned RShift, unsigned CoverSubregIdx,
    SubRegMap &SubRegs) const {

  unsigned RCAlign = TRI->getRegClassAlignmentNumBits(RC);
  LLVM_DEBUG(dbgs() << "  Shift " << RShift << ", reg align " << RCAlign
                    << '\n');

  BitVector ClassMask(getAllocatableAndAlignedRegClassMask(RCAlign));
  for (auto &[OldSubReg, NewSubReg] : SubRegs) {
    LLVM_DEBUG(dbgs() << "  " << TRI->getSubRegIndexName(OldSubReg) << ':');

    auto *SubRegRC = TRI->getSubRegisterClass(RC, OldSubReg);
    if (!SubRegRC) {
      LLVM_DEBUG(dbgs() << "couldn't find target regclass\n");
      return nullptr;
    }
    LLVM_DEBUG(dbgs() << TRI->getRegClassName(SubRegRC)
                      << (SubRegRC->isAllocatable() ? "" : " not alloc")
                      << " -> ");

    if (OldSubReg == CoverSubregIdx) {
      // Covering subreg will become a full register, RC should be allocatable.
      assert(SubRegRC->isAllocatable());
      NewSubReg = AMDGPU::NoSubRegister;
      LLVM_DEBUG(dbgs() << "whole reg");
    } else {
      NewSubReg = shiftSubReg(OldSubReg, RShift);
      if (!NewSubReg) {
        LLVM_DEBUG(dbgs() << "none\n");
        return nullptr;
      }
      LLVM_DEBUG(dbgs() << TRI->getSubRegIndexName(NewSubReg));
    }

    const uint32_t *Mask = NewSubReg ? getSuperRegClassMask(SubRegRC, NewSubReg)
                                     : SubRegRC->getSubClassMask();
    if (!Mask)
      llvm_unreachable("no register class mask?");

    ClassMask.clearBitsNotInMask(Mask);
    // Don't try to early exit because checking if ClassMask has set bits isn't
    // that cheap and we expect it to pass in most cases.
    LLVM_DEBUG(dbgs() << ", num regclasses " << ClassMask.count() << '\n');
  }

  // ClassMask is the set of all register classes such that each class is
  // allocatable, aligned, has all shifted subregs and each subreg has required
  // register class (see SubRegRC above). Now select first (that is largest)
  // register class with registers of minimal size.
  const TargetRegisterClass *MinRC = nullptr;
  unsigned MinNumBits = std::numeric_limits<unsigned>::max();
  for (unsigned ClassID : ClassMask.set_bits()) {
    auto *RC = TRI->getRegClass(ClassID);
    unsigned NumBits = TRI->getRegSizeInBits(*RC);
    if (NumBits < MinNumBits) {
      MinNumBits = NumBits;
      MinRC = RC;
    }
  }
#ifndef NDEBUG
  if (MinRC) {
    assert(MinRC->isAllocatable() && TRI->isRegClassAligned(MinRC, RCAlign));
    for (auto [OldSubReg, NewSubReg] : SubRegs)
      // Check that all registers in MinRC support NewSubReg subregister.
      assert(MinRC == TRI->getSubClassWithSubReg(MinRC, NewSubReg));
  }
#endif
  // There might be zero RShift - in this case we just trying to find smaller
  // register.
  return (MinRC != RC || RShift != 0) ? MinRC : nullptr;
}
```
<a name="block_7"></a>**File:** llvm/lib/Target/AMDGPU/GCNRewritePartialRegUses.cpp (L279-329) [<sup>↩</sup>](#ref-block_7)
```cpp
const TargetRegisterClass *
GCNRewritePartialRegUsesImpl::getMinSizeReg(const TargetRegisterClass *RC,
                                            SubRegMap &SubRegs) const {
  unsigned CoverSubreg = AMDGPU::NoSubRegister;
  unsigned Offset = std::numeric_limits<unsigned>::max();
  unsigned End = 0;
  for (auto [SubReg, SRI] : SubRegs) {
    unsigned SubRegOffset = TRI->getSubRegIdxOffset(SubReg);
    unsigned SubRegEnd = SubRegOffset + TRI->getSubRegIdxSize(SubReg);
    if (SubRegOffset < Offset) {
      Offset = SubRegOffset;
      CoverSubreg = AMDGPU::NoSubRegister;
    }
    if (SubRegEnd > End) {
      End = SubRegEnd;
      CoverSubreg = AMDGPU::NoSubRegister;
    }
    if (SubRegOffset == Offset && SubRegEnd == End)
      CoverSubreg = SubReg;
  }
  // If covering subreg is found shift everything so the covering subreg would
  // be in the rightmost position.
  if (CoverSubreg != AMDGPU::NoSubRegister)
    return getRegClassWithShiftedSubregs(RC, Offset, CoverSubreg, SubRegs);

  // Otherwise find subreg with maximum required alignment and shift it and all
  // other subregs to the rightmost possible position with respect to the
  // alignment.
  unsigned MaxAlign = 0;
  for (auto [SubReg, SRI] : SubRegs)
    MaxAlign = std::max(MaxAlign, TRI->getSubRegAlignmentNumBits(RC, SubReg));

  unsigned FirstMaxAlignedSubRegOffset = std::numeric_limits<unsigned>::max();
  for (auto [SubReg, SRI] : SubRegs) {
    if (TRI->getSubRegAlignmentNumBits(RC, SubReg) != MaxAlign)
      continue;
    FirstMaxAlignedSubRegOffset =
        std::min(FirstMaxAlignedSubRegOffset, TRI->getSubRegIdxOffset(SubReg));
    if (FirstMaxAlignedSubRegOffset == Offset)
      break;
  }

  unsigned NewOffsetOfMaxAlignedSubReg =
      alignTo(FirstMaxAlignedSubRegOffset - Offset, MaxAlign);

  if (NewOffsetOfMaxAlignedSubReg > FirstMaxAlignedSubRegOffset)
    llvm_unreachable("misaligned subreg");

  unsigned RShift = FirstMaxAlignedSubRegOffset - NewOffsetOfMaxAlignedSubReg;
  return getRegClassWithShiftedSubregs(RC, RShift, 0, SubRegs);
}
```
<a name="block_8"></a>**File:** llvm/lib/Target/AMDGPU/GCNRewritePartialRegUses.cpp (L333-386) [<sup>↩</sup>](#ref-block_8)
```cpp
void GCNRewritePartialRegUsesImpl::updateLiveIntervals(
    Register OldReg, Register NewReg, SubRegMap &SubRegs) const {
  if (!LIS->hasInterval(OldReg))
    return;

  auto &OldLI = LIS->getInterval(OldReg);
  auto &NewLI = LIS->createEmptyInterval(NewReg);

  auto &Allocator = LIS->getVNInfoAllocator();
  NewLI.setWeight(OldLI.weight());

  for (auto &SR : OldLI.subranges()) {
    auto I = find_if(SubRegs, [&](auto &P) {
      return SR.LaneMask == TRI->getSubRegIndexLaneMask(P.first);
    });

    if (I == SubRegs.end()) {
      // There might be a situation when subranges don't exactly match used
      // subregs, for example:
      // %120 [160r,1392r:0) 0@160r
      //    L000000000000C000 [160r,1392r:0) 0@160r
      //    L0000000000003000 [160r,1392r:0) 0@160r
      //    L0000000000000C00 [160r,1392r:0) 0@160r
      //    L0000000000000300 [160r,1392r:0) 0@160r
      //    L0000000000000003 [160r,1104r:0) 0@160r
      //    L000000000000000C [160r,1104r:0) 0@160r
      //    L0000000000000030 [160r,1104r:0) 0@160r
      //    L00000000000000C0 [160r,1104r:0) 0@160r
      // but used subregs are:
      //    sub0_sub1_sub2_sub3_sub4_sub5_sub6_sub7, L000000000000FFFF
      //    sub0_sub1_sub2_sub3, L00000000000000FF
      //    sub4_sub5_sub6_sub7, L000000000000FF00
      // In this example subregs sub0_sub1_sub2_sub3 and sub4_sub5_sub6_sub7
      // have several subranges with the same lifetime. For such cases just
      // recreate the interval.
      LIS->removeInterval(OldReg);
      LIS->removeInterval(NewReg);
      LIS->createAndComputeVirtRegInterval(NewReg);
      return;
    }

    if (unsigned NewSubReg = I->second)
      NewLI.createSubRangeFrom(Allocator,
                               TRI->getSubRegIndexLaneMask(NewSubReg), SR);
    else // This is the covering subreg (0 index) - set it as main range.
      NewLI.assign(SR, Allocator);

    SubRegs.erase(I);
  }
  if (NewLI.empty())
    NewLI.assign(OldLI, Allocator);
  assert(NewLI.verify(MRI));
  LIS->removeInterval(OldReg);
}
```
<a name="block_9"></a>**File:** llvm/lib/Target/AMDGPU/GCNRewritePartialRegUses.cpp (L388-433) [<sup>↩</sup>](#ref-block_9)
```cpp
bool GCNRewritePartialRegUsesImpl::rewriteReg(Register Reg) const {

  // Collect used subregs.
  SubRegMap SubRegs;
  for (MachineOperand &MO : MRI->reg_nodbg_operands(Reg)) {
    if (MO.getSubReg() == AMDGPU::NoSubRegister)
      return false; // Whole reg used.
    SubRegs.try_emplace(MO.getSubReg());
  }

  if (SubRegs.empty())
    return false;

  auto *RC = MRI->getRegClass(Reg);
  LLVM_DEBUG(dbgs() << "Try to rewrite partial reg " << printReg(Reg, TRI)
                    << ':' << TRI->getRegClassName(RC) << '\n');

  auto *NewRC = getMinSizeReg(RC, SubRegs);
  if (!NewRC) {
    LLVM_DEBUG(dbgs() << "  No improvement achieved\n");
    return false;
  }

  Register NewReg = MRI->createVirtualRegister(NewRC);
  LLVM_DEBUG(dbgs() << "  Success " << printReg(Reg, TRI) << ':'
                    << TRI->getRegClassName(RC) << " -> "
                    << printReg(NewReg, TRI) << ':'
                    << TRI->getRegClassName(NewRC) << '\n');

  for (auto &MO : make_early_inc_range(MRI->reg_operands(Reg))) {
    MO.setReg(NewReg);
    // Debug info can refer to the whole reg, just leave it as it is for now.
    // TODO: create some DI shift expression?
    if (MO.isDebug() && MO.getSubReg() == 0)
      continue;
    unsigned NewSubReg = SubRegs[MO.getSubReg()];
    MO.setSubReg(NewSubReg);
    if (NewSubReg == AMDGPU::NoSubRegister && MO.isDef())
      MO.setIsUndef(false);
  }

  if (LIS)
    updateLiveIntervals(Reg, NewReg, SubRegs);

  return true;
}
```
<a name="block_10"></a>**File:** llvm/lib/Target/AMDGPU/GCNRewritePartialRegUses.cpp (L435-444) [<sup>↩</sup>](#ref-block_10)
```cpp
bool GCNRewritePartialRegUsesImpl::run(MachineFunction &MF) {
  MRI = &MF.getRegInfo();
  TRI = static_cast<const SIRegisterInfo *>(MRI->getTargetRegisterInfo());
  TII = MF.getSubtarget().getInstrInfo();
  bool Changed = false;
  for (size_t I = 0, E = MRI->getNumVirtRegs(); I < E; ++I) {
    Changed |= rewriteReg(Register::index2VirtReg(I));
  }
  return Changed;
}
```