# SILoadStoreOptimizer.cpp 代码功能详解

## 1. Pass 主要功能概括

<a name="ref-block_0"></a>`SILoadStoreOptimizer` 是 AMDGPU 后端的一个机器指令优化 pass，主要实现两个核心优化功能： llvm-project:9-57[<sup>↗</sup>](#block_0) 

**作用：**
- **指令合并优化**：将具有相邻偏移量的多个内存访问指令合并为单个更宽的访问指令（如将两个 `ds_read_b32` 合并为 `ds_read2_b32`）
- **常量偏移提升**：通过调整基址寄存器，将常量偏移提升到指令的立即数字段，减少地址计算开销

**效果：**
- 减少内存访问指令数量，提高内存带宽利用率
- 降低地址计算指令数量，提升整体性能
- 支持多种指令类型：DS（LDS/GDS）、SMEM、VMEM、FLAT/GLOBAL、MIMG 等

## 2. 主要功能步骤和子功能提取

该 pass 的实现包含以下关键步骤和子功能：

### 核心控制流程：
- **`run`**：Pass 的主入口函数
- **`collectMergeableInsts`**：收集可合并指令列表
- **`optimizeBlock`**：优化指令块
- **`optimizeInstsWithSameBaseAddr`**：优化具有相同基址的指令

### 常量偏移提升功能：
- **`promoteConstantOffsetToImm`**：提升常量偏移到立即数
- **`processBaseWithConstOffset`**：分析基址和常量偏移
- **`computeBase`**：计算新的基址寄存器
- **`updateBaseAndOffset`**：更新指令的基址和偏移

### 合并可行性检查：
- **`checkAndPrepareMerge`**：检查并准备合并操作
- **`offsetsCanBeCombined`**：检查偏移量是否可合并
- **`dmasksCanBeCombined`**：检查 MIMG 的 dmask 是否可合并
- **`widthsFit`**：检查合并后的宽度是否合法
- **`canSwapInstructions`**：检查指令是否可以交换顺序

### 指令合并实现：
- **`mergeRead2Pair`**：合并 DS read 指令对
- **`mergeWrite2Pair`**：合并 DS write 指令对
- **`mergeSMemLoadImmPair`**：合并 SMEM load 指令对
- **`mergeBufferLoadPair/mergeBufferStorePair`**：合并 buffer 访问指令
- **`mergeTBufferLoadPair/mergeTBufferStorePair`**：合并 typed buffer 访问指令
- **`mergeFlatLoadPair/mergeFlatStorePair`**：合并 FLAT/GLOBAL 访问指令
- **`mergeImagePair`**：合并 MIMG 指令对

### 辅助功能：
- **`getNewOpcode`**：获取合并后的新操作码
- **`getSubRegIdxs`**：获取子寄存器索引
- **`getTargetRegisterClass`**：获取目标寄存器类
- **`copyToDestRegs/copyFromSrcRegs`**：处理寄存器复制

## 3. 各步骤/子功能的具体描述分析

### 3.1 主控制流程

<a name="ref-block_21"></a>**`run` 函数** llvm-project:2554-2598[<sup>↗</sup>](#block_21) 

这是 pass 的主入口，遍历函数中的每个基本块，维护访问信息映射表和锚点列表，反复调用优化过程直到无法继续优化。

<a name="ref-block_18"></a>**`collectMergeableInsts` 函数** llvm-project:2330-2420[<sup>↗</sup>](#block_18) 

这个函数遍历基本块中的指令，识别可以合并的内存访问指令。它会：
- 首先尝试对每条指令进行常量偏移提升优化
- 识别指令类型（DS_READ、DS_WRITE、BUFFER_LOAD 等）
- 检查指令是否具有可合并的地址
- 将具有相同基址的指令添加到同一个合并列表中
- 按偏移量对列表进行排序

### 3.2 常量偏移提升优化

<a name="ref-block_17"></a>**`promoteConstantOffsetToImm` 函数** llvm-project:2163-2313[<sup>↗</sup>](#block_17) 

这是一个独立的优化功能，实现启发式算法：
1. 从当前指令中提取基址寄存器和 64 位常量偏移
2. 在同一基本块中搜索具有相同基址的其他指令
3. 寻找一个"锚点"指令，该指令与当前指令的偏移差距最大但仍在 13 位范围内
4. 重新计算锚点的基址，并将相关指令的偏移提升到立即数字段

<a name="ref-block_16"></a>**`processBaseWithConstOffset` 函数** llvm-project:2104-2161[<sup>↗</sup>](#block_16) 

分析基址寄存器的定义链，识别形如 `V_ADD_CO_U32` 和 `V_ADDC_U32` 的地址计算模式，提取出 32 位低位寄存器、32 位高位寄存器和 64 位常量偏移。

<a name="ref-block_15"></a>**`computeBase` 函数** llvm-project:2011-2066[<sup>↗</sup>](#block_15) 

根据提取的基址信息和偏移量，生成新的 64 位基址寄存器，包括生成加法指令和寄存器序列指令。

### 3.3 合并可行性检查

<a name="ref-block_5"></a>**`checkAndPrepareMerge` 函数** llvm-project:1202-1253[<sup>↗</sup>](#block_5) 

这是合并前的核心检查函数：
- 验证两条指令的指令类和子类是否匹配
- 根据指令类型调用相应的检查函数（`offsetsCanBeCombined` 或 `dmasksCanBeCombined`）
- 对于 load 指令，尝试将第二条指令提升到第一条指令位置
- 对于 store 指令，尝试将第一条指令下沉到第二条指令位置
- 使用 `canSwapInstructions` 检查指令重排序的合法性

<a name="ref-block_4"></a>**`offsetsCanBeCombined` 函数** llvm-project:1033-1155[<sup>↗</sup>](#block_4) 

检查两个偏移量是否可以合并，处理多种情况：
- 验证偏移量对齐
- 对于 DS 指令，尝试使用普通偏移或 ST64（stride 64）模式
- 如果直接偏移超出 8 位范围，尝试调整基址偏移（`BaseOff`）
- 对于 TBUFFER 指令，检查 format 兼容性

<a name="ref-block_3"></a>**`dmasksCanBeCombined` 函数** llvm-project:958-996[<sup>↗</sup>](#block_3) 

专门用于 MIMG 指令的检查，验证两个 dmask（数据掩码）是否可以合并而不产生重叠。

<a name="ref-block_2"></a>**`canSwapInstructions` 函数** llvm-project:917-932[<sup>↗</sup>](#block_2) 

检查两条指令是否可以安全交换顺序，考虑：
- 内存别名分析（通过 AliasAnalysis）
- 寄存器依赖关系（def-use 链）

### 3.4 指令合并实现

<a name="ref-block_8"></a>**`mergeRead2Pair` 函数** llvm-project:1338-1396[<sup>↗</sup>](#block_8) 

合并两个 DS read 指令为 `ds_read2_b32/b64` 或 `ds_read2st64_b32/b64`：
- 选择合适的操作码（根据是否使用 ST64 模式）
- 如果需要，调整基址（加上 `BaseOff`）
- 创建新的合并指令，包含两个偏移量
- 生成 COPY 指令将结果分配到原始目标寄存器
- 删除原始指令

<a name="ref-block_9"></a>**`mergeWrite2Pair` 函数** llvm-project:1414-1478[<sup>↗</sup>](#block_9) 

合并两个 DS write 指令，类似于 read，但处理数据源而非目标。

**其他 merge 函数**：
<a name="ref-block_11"></a>- `mergeSMemLoadImmPair` llvm-project:1516-1546[<sup>↗</sup>](#block_11) ：合并 SMEM load 指令
<a name="ref-block_12"></a>- `mergeBufferLoadPair` llvm-project:1548-1587[<sup>↗</sup>](#block_12) ：合并 buffer load 指令
<a name="ref-block_10"></a>- `mergeImagePair` llvm-project:1480-1514[<sup>↗</sup>](#block_10) ：合并 image load 指令

这些函数的共同模式是：
1. 确定新操作码
2. 创建更宽的目标/源寄存器
3. 构建新的合并指令
4. 处理子寄存器的复制
5. 删除原始指令

<a name="ref-block_13"></a>**`getNewOpcode` 函数** llvm-project:1740-1894[<sup>↗</sup>](#block_13) 

根据指令类型和合并后的宽度，返回相应的新操作码。对于 S_LOAD 类指令，还需要考虑是否使用受约束的操作码（针对 XNACK）。

### 3.5 寄存器和操作数处理

<a name="ref-block_6"></a>**`copyToDestRegs` 函数** llvm-project:1257-1295[<sup>↗</sup>](#block_6) 

将合并后的宽寄存器结果复制回原始目标寄存器，使用适当的子寄存器索引。

<a name="ref-block_7"></a>**`copyFromSrcRegs` 函数** llvm-project:1299-1322[<sup>↗</sup>](#block_7) 

对于 store 指令，使用 `REG_SEQUENCE` 将两个源寄存器组合成一个宽寄存器。

<a name="ref-block_14"></a>**`getSubRegIdxs` 函数** llvm-project:1896-1927[<sup>↗</sup>](#block_14) 

根据合并指令的宽度和顺序，返回适当的子寄存器索引对（如 `sub0`, `sub1`, `sub2_sub3` 等）。

### 3.6 优化主循环

<a name="ref-block_20"></a>**`optimizeInstsWithSameBaseAddr` 函数** llvm-project:2455-2544[<sup>↗</sup>](#block_20) 

对于具有相同基址的指令列表，遍历相邻的指令对：
- 调用 `checkAndPrepareMerge` 验证是否可以合并
- 根据指令类型调用相应的 merge 函数
- 更新合并后的指令信息
- 从列表中移除已合并的指令
- 设置 `OptimizeListAgain` 标志以便继续迭代优化

<a name="ref-block_19"></a>**`optimizeBlock` 函数** llvm-project:2425-2453[<sup>↗</sup>](#block_19) 

对收集到的所有可合并指令列表进行优化，反复调用 `optimizeInstsWithSameBaseAddr` 直到无法继续优化。

## 4. 步骤/子功能之间的关系

### 4.1 整体执行流程关系

```
run() [主入口]
  └─> 遍历每个基本块
      ├─> collectMergeableInsts() [收集阶段]
      │   ├─> promoteConstantOffsetToImm() [独立优化]
      │   │   ├─> processBaseWithConstOffset() [分析]
      │   │   └─> computeBase() [生成新基址]
      │   ├─> 识别指令类型
      │   └─> addInstToMergeableList() [分组]
      │
      └─> optimizeBlock() [优化阶段]
          └─> optimizeInstsWithSameBaseAddr() [迭代优化]
              ├─> checkAndPrepareMerge() [验证]
              │   ├─> offsetsCanBeCombined()
              │   ├─> dmasksCanBeCombined()
              │   ├─> widthsFit()
              │   └─> canSwapInstructions()
              │
              └─> merge*Pair() [合并执行]
                  ├─> getNewOpcode()
                  ├─> getSubRegIdxs()
                  ├─> copyToDestRegs() / copyFromSrcRegs()
                  └─> 创建新指令，删除旧指令
```

### 4.2 关键依赖关系

1. **收集与优化的分离**：`collectMergeableInsts` 和 `optimizeBlock` 分离，允许先完整收集信息再进行优化决策

2. **两级迭代结构**：
   - 外层：按基本块的代码段迭代
   - 内层：对同一基址的指令列表反复优化，直到无法继续合并

3. **独立优化与合并优化的协同**：
   - `promoteConstantOffsetToImm` 作为独立优化，在收集阶段就执行
   - 这可能创造出新的合并机会（通过调整偏移使指令更接近）
   - 即使指令最终无法合并，常量提升本身也有性能收益

4. **检查与执行的解耦**：
   - `checkAndPrepareMerge` 负责所有合法性检查和指令重排序
   - 各个 `merge*Pair` 函数只负责生成合并后的指令
   - 这种设计确保了合并操作的安全性

5. **辅助函数的支撑作用**：
   - `getNewOpcode`、`getSubRegIdxs`、`getTargetRegisterClass` 等函数为合并操作提供必要的目标相关信息
   - `canSwapInstructions` 提供依赖性分析，确保指令重排序的正确性

<a name="ref-block_1"></a>### 4.3 数据流关系 llvm-project:109-186[<sup>↗</sup>](#block_1) 

`CombineInfo` 结构体是连接各功能的核心数据结构，包含：
- 指令迭代器、偏移量、宽度、指令类等信息
- `hasSameBaseAddress()` 方法用于分组
- `hasMergeableAddress()` 方法用于过滤

这个结构在收集阶段创建，在检查阶段验证，在合并阶段更新。

## Notes

该优化 pass 的设计体现了几个重要特点：

1. **多阶段优化**：将独立的常量偏移提升与指令合并分开，提升了优化的灵活性和效果

2. **迭代优化**：通过 `OptimizeAgain` 标志支持多轮合并，因为一次合并可能创造出新的合并机会（例如 dword + dword → dwordx2，然后 dwordx2 + dwordx2 → dwordx4）

3. **安全性保证**：通过别名分析、依赖分析和多种合法性检查，确保优化不会改变程序语义

4. **目标特定性**：针对 AMDGPU 架构的特性（如 DS ST64 模式、SMEM 对齐要求、XNACK 约束等）进行了专门优化

5. **广泛的指令支持**：支持 DS、SMEM、VMEM、FLAT/GLOBAL、MIMG 等多种内存访问指令类型，每种都有针对性的合并策略
### Citations
<a name="block_0"></a>**File:** llvm/lib/Target/AMDGPU/SILoadStoreOptimizer.cpp (L9-57) [<sup>↩</sup>](#ref-block_0)
```cpp
// This pass tries to fuse DS instructions with close by immediate offsets.
// This will fuse operations such as
//  ds_read_b32 v0, v2 offset:16
//  ds_read_b32 v1, v2 offset:32
// ==>
//   ds_read2_b32 v[0:1], v2, offset0:4 offset1:8
//
// The same is done for certain SMEM and VMEM opcodes, e.g.:
//  s_buffer_load_dword s4, s[0:3], 4
//  s_buffer_load_dword s5, s[0:3], 8
// ==>
//  s_buffer_load_dwordx2 s[4:5], s[0:3], 4
//
// This pass also tries to promote constant offset to the immediate by
// adjusting the base. It tries to use a base from the nearby instructions that
// allows it to have a 13bit constant offset and then promotes the 13bit offset
// to the immediate.
// E.g.
//  s_movk_i32 s0, 0x1800
//  v_add_co_u32_e32 v0, vcc, s0, v2
//  v_addc_co_u32_e32 v1, vcc, 0, v6, vcc
//
//  s_movk_i32 s0, 0x1000
//  v_add_co_u32_e32 v5, vcc, s0, v2
//  v_addc_co_u32_e32 v6, vcc, 0, v6, vcc
//  global_load_dwordx2 v[5:6], v[5:6], off
//  global_load_dwordx2 v[0:1], v[0:1], off
// =>
//  s_movk_i32 s0, 0x1000
//  v_add_co_u32_e32 v5, vcc, s0, v2
//  v_addc_co_u32_e32 v6, vcc, 0, v6, vcc
//  global_load_dwordx2 v[5:6], v[5:6], off
//  global_load_dwordx2 v[0:1], v[5:6], off offset:2048
//
// Future improvements:
//
// - This is currently missing stores of constants because loading
//   the constant into the data register is placed between the stores, although
//   this is arguably a scheduling problem.
//
// - Live interval recomputing seems inefficient. This currently only matches
//   one pair, and recomputes live intervals and moves on to the next pair. It
//   would be better to compute a list of all merges that need to occur.
//
// - With a list of instructions to process, we can also merge more. If a
//   cluster of loads have offsets that are too large to fit in the 8-bit
//   offsets, but are close enough to fit in the 8 bits, we can add to the base
//   pointer and use the new reduced offsets.
//
```
<a name="block_1"></a>**File:** llvm/lib/Target/AMDGPU/SILoadStoreOptimizer.cpp (L109-186) [<sup>↩</sup>](#ref-block_1)
```cpp
  struct CombineInfo {
    MachineBasicBlock::iterator I;
    unsigned EltSize;
    unsigned Offset;
    unsigned Width;
    unsigned Format;
    unsigned BaseOff;
    unsigned DMask;
    InstClassEnum InstClass;
    unsigned CPol = 0;
    bool IsAGPR;
    bool UseST64;
    int AddrIdx[MaxAddressRegs];
    const MachineOperand *AddrReg[MaxAddressRegs];
    unsigned NumAddresses;
    unsigned Order;

    bool hasSameBaseAddress(const CombineInfo &CI) {
      if (NumAddresses != CI.NumAddresses)
        return false;

      const MachineInstr &MI = *CI.I;
      for (unsigned i = 0; i < NumAddresses; i++) {
        const MachineOperand &AddrRegNext = MI.getOperand(AddrIdx[i]);

        if (AddrReg[i]->isImm() || AddrRegNext.isImm()) {
          if (AddrReg[i]->isImm() != AddrRegNext.isImm() ||
              AddrReg[i]->getImm() != AddrRegNext.getImm()) {
            return false;
          }
          continue;
        }

        // Check same base pointer. Be careful of subregisters, which can occur
        // with vectors of pointers.
        if (AddrReg[i]->getReg() != AddrRegNext.getReg() ||
            AddrReg[i]->getSubReg() != AddrRegNext.getSubReg()) {
         return false;
        }
      }
      return true;
    }

    bool hasMergeableAddress(const MachineRegisterInfo &MRI) {
      for (unsigned i = 0; i < NumAddresses; ++i) {
        const MachineOperand *AddrOp = AddrReg[i];
        // Immediates are always OK.
        if (AddrOp->isImm())
          continue;

        // Don't try to merge addresses that aren't either immediates or registers.
        // TODO: Should be possible to merge FrameIndexes and maybe some other
        // non-register
        if (!AddrOp->isReg())
          return false;

        // TODO: We should be able to merge instructions with other physical reg
        // addresses too.
        if (AddrOp->getReg().isPhysical() &&
            AddrOp->getReg() != AMDGPU::SGPR_NULL)
          return false;

        // If an address has only one use then there will be no other
        // instructions with the same address, so we can't merge this one.
        if (MRI.hasOneNonDBGUse(AddrOp->getReg()))
          return false;
      }
      return true;
    }

    void setMI(MachineBasicBlock::iterator MI, const SILoadStoreOptimizer &LSO);

    // Compare by pointer order.
    bool operator<(const CombineInfo& Other) const {
      return (InstClass == MIMG) ? DMask < Other.DMask : Offset < Other.Offset;
    }
  };

```
<a name="block_2"></a>**File:** llvm/lib/Target/AMDGPU/SILoadStoreOptimizer.cpp (L917-932) [<sup>↩</sup>](#ref-block_2)
```cpp
bool SILoadStoreOptimizer::canSwapInstructions(
    const DenseSet<Register> &ARegDefs, const DenseSet<Register> &ARegUses,
    const MachineInstr &A, const MachineInstr &B) const {
  if (A.mayLoadOrStore() && B.mayLoadOrStore() &&
      (A.mayStore() || B.mayStore()) && A.mayAlias(AA, B, true))
    return false;
  for (const auto &BOp : B.operands()) {
    if (!BOp.isReg())
      continue;
    if ((BOp.isDef() || BOp.readsReg()) && ARegDefs.contains(BOp.getReg()))
      return false;
    if (BOp.isDef() && ARegUses.contains(BOp.getReg()))
      return false;
  }
  return true;
}
```
<a name="block_3"></a>**File:** llvm/lib/Target/AMDGPU/SILoadStoreOptimizer.cpp (L958-996) [<sup>↩</sup>](#ref-block_3)
```cpp
bool SILoadStoreOptimizer::dmasksCanBeCombined(const CombineInfo &CI,
                                               const SIInstrInfo &TII,
                                               const CombineInfo &Paired) {
  assert(CI.InstClass == MIMG);

  // Ignore instructions with tfe/lwe set.
  const auto *TFEOp = TII.getNamedOperand(*CI.I, AMDGPU::OpName::tfe);
  const auto *LWEOp = TII.getNamedOperand(*CI.I, AMDGPU::OpName::lwe);

  if ((TFEOp && TFEOp->getImm()) || (LWEOp && LWEOp->getImm()))
    return false;

  // Check other optional immediate operands for equality.
  AMDGPU::OpName OperandsToMatch[] = {
      AMDGPU::OpName::cpol, AMDGPU::OpName::d16,  AMDGPU::OpName::unorm,
      AMDGPU::OpName::da,   AMDGPU::OpName::r128, AMDGPU::OpName::a16};

  for (AMDGPU::OpName op : OperandsToMatch) {
    int Idx = AMDGPU::getNamedOperandIdx(CI.I->getOpcode(), op);
    if (AMDGPU::getNamedOperandIdx(Paired.I->getOpcode(), op) != Idx)
      return false;
    if (Idx != -1 &&
        CI.I->getOperand(Idx).getImm() != Paired.I->getOperand(Idx).getImm())
      return false;
  }

  // Check DMask for overlaps.
  unsigned MaxMask = std::max(CI.DMask, Paired.DMask);
  unsigned MinMask = std::min(CI.DMask, Paired.DMask);

  if (!MaxMask)
    return false;

  unsigned AllowedBitsForMin = llvm::countr_zero(MaxMask);
  if ((1u << AllowedBitsForMin) <= MinMask)
    return false;

  return true;
}
```
<a name="block_4"></a>**File:** llvm/lib/Target/AMDGPU/SILoadStoreOptimizer.cpp (L1033-1155) [<sup>↩</sup>](#ref-block_4)
```cpp
bool SILoadStoreOptimizer::offsetsCanBeCombined(CombineInfo &CI,
                                                const GCNSubtarget &STI,
                                                CombineInfo &Paired,
                                                bool Modify) {
  assert(CI.InstClass != MIMG);

  // XXX - Would the same offset be OK? Is there any reason this would happen or
  // be useful?
  if (CI.Offset == Paired.Offset)
    return false;

  // This won't be valid if the offset isn't aligned.
  if ((CI.Offset % CI.EltSize != 0) || (Paired.Offset % CI.EltSize != 0))
    return false;

  if (CI.InstClass == TBUFFER_LOAD || CI.InstClass == TBUFFER_STORE) {

    const llvm::AMDGPU::GcnBufferFormatInfo *Info0 =
        llvm::AMDGPU::getGcnBufferFormatInfo(CI.Format, STI);
    if (!Info0)
      return false;
    const llvm::AMDGPU::GcnBufferFormatInfo *Info1 =
        llvm::AMDGPU::getGcnBufferFormatInfo(Paired.Format, STI);
    if (!Info1)
      return false;

    if (Info0->BitsPerComp != Info1->BitsPerComp ||
        Info0->NumFormat != Info1->NumFormat)
      return false;

    // TODO: Should be possible to support more formats, but if format loads
    // are not dword-aligned, the merged load might not be valid.
    if (Info0->BitsPerComp != 32)
      return false;

    if (getBufferFormatWithCompCount(CI.Format, CI.Width + Paired.Width, STI) == 0)
      return false;
  }

  uint32_t EltOffset0 = CI.Offset / CI.EltSize;
  uint32_t EltOffset1 = Paired.Offset / CI.EltSize;
  CI.UseST64 = false;
  CI.BaseOff = 0;

  // Handle all non-DS instructions.
  if ((CI.InstClass != DS_READ) && (CI.InstClass != DS_WRITE)) {
    if (EltOffset0 + CI.Width != EltOffset1 &&
            EltOffset1 + Paired.Width != EltOffset0)
      return false;
    if (CI.CPol != Paired.CPol)
      return false;
    if (CI.InstClass == S_LOAD_IMM || CI.InstClass == S_BUFFER_LOAD_IMM ||
        CI.InstClass == S_BUFFER_LOAD_SGPR_IMM) {
      // Reject cases like:
      //   dword + dwordx2 -> dwordx3
      //   dword + dwordx3 -> dwordx4
      // If we tried to combine these cases, we would fail to extract a subreg
      // for the result of the second load due to SGPR alignment requirements.
      if (CI.Width != Paired.Width &&
          (CI.Width < Paired.Width) == (CI.Offset < Paired.Offset))
        return false;
    }
    return true;
  }

  // If the offset in elements doesn't fit in 8-bits, we might be able to use
  // the stride 64 versions.
  if ((EltOffset0 % 64 == 0) && (EltOffset1 % 64) == 0 &&
      isUInt<8>(EltOffset0 / 64) && isUInt<8>(EltOffset1 / 64)) {
    if (Modify) {
      CI.Offset = EltOffset0 / 64;
      Paired.Offset = EltOffset1 / 64;
      CI.UseST64 = true;
    }
    return true;
  }

  // Check if the new offsets fit in the reduced 8-bit range.
  if (isUInt<8>(EltOffset0) && isUInt<8>(EltOffset1)) {
    if (Modify) {
      CI.Offset = EltOffset0;
      Paired.Offset = EltOffset1;
    }
    return true;
  }

  // Try to shift base address to decrease offsets.
  uint32_t Min = std::min(EltOffset0, EltOffset1);
  uint32_t Max = std::max(EltOffset0, EltOffset1);

  const uint32_t Mask = maskTrailingOnes<uint32_t>(8) * 64;
  if (((Max - Min) & ~Mask) == 0) {
    if (Modify) {
      // From the range of values we could use for BaseOff, choose the one that
      // is aligned to the highest power of two, to maximise the chance that
      // the same offset can be reused for other load/store pairs.
      uint32_t BaseOff = mostAlignedValueInRange(Max - 0xff * 64, Min);
      // Copy the low bits of the offsets, so that when we adjust them by
      // subtracting BaseOff they will be multiples of 64.
      BaseOff |= Min & maskTrailingOnes<uint32_t>(6);
      CI.BaseOff = BaseOff * CI.EltSize;
      CI.Offset = (EltOffset0 - BaseOff) / 64;
      Paired.Offset = (EltOffset1 - BaseOff) / 64;
      CI.UseST64 = true;
    }
    return true;
  }

  if (isUInt<8>(Max - Min)) {
    if (Modify) {
      // From the range of values we could use for BaseOff, choose the one that
      // is aligned to the highest power of two, to maximise the chance that
      // the same offset can be reused for other load/store pairs.
      uint32_t BaseOff = mostAlignedValueInRange(Max - 0xff, Min);
      CI.BaseOff = BaseOff * CI.EltSize;
      CI.Offset = EltOffset0 - BaseOff;
      Paired.Offset = EltOffset1 - BaseOff;
    }
    return true;
  }

  return false;
}
```
<a name="block_5"></a>**File:** llvm/lib/Target/AMDGPU/SILoadStoreOptimizer.cpp (L1202-1253) [<sup>↩</sup>](#ref-block_5)
```cpp
SILoadStoreOptimizer::CombineInfo *
SILoadStoreOptimizer::checkAndPrepareMerge(CombineInfo &CI,
                                           CombineInfo &Paired) {
  // If another instruction has already been merged into CI, it may now be a
  // type that we can't do any further merging into.
  if (CI.InstClass == UNKNOWN || Paired.InstClass == UNKNOWN)
    return nullptr;
  assert(CI.InstClass == Paired.InstClass);

  if (getInstSubclass(CI.I->getOpcode(), *TII) !=
      getInstSubclass(Paired.I->getOpcode(), *TII))
    return nullptr;

  // Check both offsets (or masks for MIMG) can be combined and fit in the
  // reduced range.
  if (CI.InstClass == MIMG) {
    if (!dmasksCanBeCombined(CI, *TII, Paired))
      return nullptr;
  } else {
    if (!widthsFit(*STM, CI, Paired) || !offsetsCanBeCombined(CI, *STM, Paired))
      return nullptr;
  }

  DenseSet<Register> RegDefs;
  DenseSet<Register> RegUses;
  CombineInfo *Where;
  if (CI.I->mayLoad()) {
    // Try to hoist Paired up to CI.
    addDefsUsesToList(*Paired.I, RegDefs, RegUses);
    for (MachineBasicBlock::iterator MBBI = Paired.I; --MBBI != CI.I;) {
      if (!canSwapInstructions(RegDefs, RegUses, *Paired.I, *MBBI))
        return nullptr;
    }
    Where = &CI;
  } else {
    // Try to sink CI down to Paired.
    addDefsUsesToList(*CI.I, RegDefs, RegUses);
    for (MachineBasicBlock::iterator MBBI = CI.I; ++MBBI != Paired.I;) {
      if (!canSwapInstructions(RegDefs, RegUses, *CI.I, *MBBI))
        return nullptr;
    }
    Where = &Paired;
  }

  // Call offsetsCanBeCombined with modify = true so that the offsets are
  // correct for the new instruction.  This should return true, because
  // this function should only be called on CombineInfo objects that
  // have already been confirmed to be mergeable.
  if (CI.InstClass == DS_READ || CI.InstClass == DS_WRITE)
    offsetsCanBeCombined(CI, *STM, Paired, true);
  return Where;
}
```
<a name="block_6"></a>**File:** llvm/lib/Target/AMDGPU/SILoadStoreOptimizer.cpp (L1257-1295) [<sup>↩</sup>](#ref-block_6)
```cpp
void SILoadStoreOptimizer::copyToDestRegs(
    CombineInfo &CI, CombineInfo &Paired,
    MachineBasicBlock::iterator InsertBefore, AMDGPU::OpName OpName,
    Register DestReg, MachineInstr *NewMI) const {
  MachineBasicBlock *MBB = CI.I->getParent();
  MachineFunction *MF = MBB->getParent();
  DebugLoc DL = CI.I->getDebugLoc();

  auto [SubRegIdx0, SubRegIdx1] = getSubRegIdxs(CI, Paired);

  // Copy to the old destination registers.
  const MCInstrDesc &CopyDesc = TII->get(TargetOpcode::COPY);
  auto *Dest0 = TII->getNamedOperand(*CI.I, OpName);
  auto *Dest1 = TII->getNamedOperand(*Paired.I, OpName);

  // The constrained sload instructions in S_LOAD_IMM class will have
  // `early-clobber` flag in the dst operand. Remove the flag before using the
  // MOs in copies.
  Dest0->setIsEarlyClobber(false);
  Dest1->setIsEarlyClobber(false);

  BuildMI(*MBB, InsertBefore, DL, CopyDesc)
      .add(*Dest0) // Copy to same destination including flags and sub reg.
      .addReg(DestReg, 0, SubRegIdx0);
  BuildMI(*MBB, InsertBefore, DL, CopyDesc)
      .add(*Dest1)
      .addReg(DestReg, RegState::Kill, SubRegIdx1);

  if (unsigned DINum = CI.I->peekDebugInstrNum()) {
    unsigned NewDINum = NewMI->getDebugInstrNum();
    MF->makeDebugValueSubstitution(std::make_pair(DINum, 0),
                                   std::make_pair(NewDINum, 0), SubRegIdx0);
  }
  if (unsigned DINum = Paired.I->peekDebugInstrNum()) {
    unsigned NewDINum = NewMI->getDebugInstrNum();
    MF->makeDebugValueSubstitution(std::make_pair(DINum, 0),
                                   std::make_pair(NewDINum, 0), SubRegIdx1);
  }
}
```
<a name="block_7"></a>**File:** llvm/lib/Target/AMDGPU/SILoadStoreOptimizer.cpp (L1299-1322) [<sup>↩</sup>](#ref-block_7)
```cpp
Register
SILoadStoreOptimizer::copyFromSrcRegs(CombineInfo &CI, CombineInfo &Paired,
                                      MachineBasicBlock::iterator InsertBefore,
                                      AMDGPU::OpName OpName) const {
  MachineBasicBlock *MBB = CI.I->getParent();
  DebugLoc DL = CI.I->getDebugLoc();

  auto [SubRegIdx0, SubRegIdx1] = getSubRegIdxs(CI, Paired);

  // Copy to the new source register.
  const TargetRegisterClass *SuperRC = getTargetRegisterClass(CI, Paired);
  Register SrcReg = MRI->createVirtualRegister(SuperRC);

  const auto *Src0 = TII->getNamedOperand(*CI.I, OpName);
  const auto *Src1 = TII->getNamedOperand(*Paired.I, OpName);

  BuildMI(*MBB, InsertBefore, DL, TII->get(AMDGPU::REG_SEQUENCE), SrcReg)
      .add(*Src0)
      .addImm(SubRegIdx0)
      .add(*Src1)
      .addImm(SubRegIdx1);

  return SrcReg;
}
```
<a name="block_8"></a>**File:** llvm/lib/Target/AMDGPU/SILoadStoreOptimizer.cpp (L1338-1396) [<sup>↩</sup>](#ref-block_8)
```cpp
MachineBasicBlock::iterator
SILoadStoreOptimizer::mergeRead2Pair(CombineInfo &CI, CombineInfo &Paired,
                                     MachineBasicBlock::iterator InsertBefore) {
  MachineBasicBlock *MBB = CI.I->getParent();

  // Be careful, since the addresses could be subregisters themselves in weird
  // cases, like vectors of pointers.
  const auto *AddrReg = TII->getNamedOperand(*CI.I, AMDGPU::OpName::addr);

  unsigned NewOffset0 = std::min(CI.Offset, Paired.Offset);
  unsigned NewOffset1 = std::max(CI.Offset, Paired.Offset);
  unsigned Opc =
      CI.UseST64 ? read2ST64Opcode(CI.EltSize) : read2Opcode(CI.EltSize);

  assert((isUInt<8>(NewOffset0) && isUInt<8>(NewOffset1)) &&
         (NewOffset0 != NewOffset1) && "Computed offset doesn't fit");

  const MCInstrDesc &Read2Desc = TII->get(Opc);

  const TargetRegisterClass *SuperRC = getTargetRegisterClass(CI, Paired);
  Register DestReg = MRI->createVirtualRegister(SuperRC);

  DebugLoc DL = CI.I->getDebugLoc();

  Register BaseReg = AddrReg->getReg();
  unsigned BaseSubReg = AddrReg->getSubReg();
  unsigned BaseRegFlags = 0;
  if (CI.BaseOff) {
    Register ImmReg = MRI->createVirtualRegister(&AMDGPU::SReg_32RegClass);
    BuildMI(*MBB, InsertBefore, DL, TII->get(AMDGPU::S_MOV_B32), ImmReg)
        .addImm(CI.BaseOff);

    BaseReg = MRI->createVirtualRegister(&AMDGPU::VGPR_32RegClass);
    BaseRegFlags = RegState::Kill;

    TII->getAddNoCarry(*MBB, InsertBefore, DL, BaseReg)
        .addReg(ImmReg)
        .addReg(AddrReg->getReg(), 0, BaseSubReg)
        .addImm(0); // clamp bit
    BaseSubReg = 0;
  }

  MachineInstrBuilder Read2 =
      BuildMI(*MBB, InsertBefore, DL, Read2Desc, DestReg)
          .addReg(BaseReg, BaseRegFlags, BaseSubReg) // addr
          .addImm(NewOffset0)                        // offset0
          .addImm(NewOffset1)                        // offset1
          .addImm(0)                                 // gds
          .cloneMergedMemRefs({&*CI.I, &*Paired.I});

  copyToDestRegs(CI, Paired, InsertBefore, AMDGPU::OpName::vdst, DestReg,
                 Read2);

  CI.I->eraseFromParent();
  Paired.I->eraseFromParent();

  LLVM_DEBUG(dbgs() << "Inserted read2: " << *Read2 << '\n');
  return Read2;
}
```
<a name="block_9"></a>**File:** llvm/lib/Target/AMDGPU/SILoadStoreOptimizer.cpp (L1414-1478) [<sup>↩</sup>](#ref-block_9)
```cpp
MachineBasicBlock::iterator SILoadStoreOptimizer::mergeWrite2Pair(
    CombineInfo &CI, CombineInfo &Paired,
    MachineBasicBlock::iterator InsertBefore) {
  MachineBasicBlock *MBB = CI.I->getParent();

  // Be sure to use .addOperand(), and not .addReg() with these. We want to be
  // sure we preserve the subregister index and any register flags set on them.
  const MachineOperand *AddrReg =
      TII->getNamedOperand(*CI.I, AMDGPU::OpName::addr);
  const MachineOperand *Data0 =
      TII->getNamedOperand(*CI.I, AMDGPU::OpName::data0);
  const MachineOperand *Data1 =
      TII->getNamedOperand(*Paired.I, AMDGPU::OpName::data0);

  unsigned NewOffset0 = CI.Offset;
  unsigned NewOffset1 = Paired.Offset;
  unsigned Opc =
      CI.UseST64 ? write2ST64Opcode(CI.EltSize) : write2Opcode(CI.EltSize);

  if (NewOffset0 > NewOffset1) {
    // Canonicalize the merged instruction so the smaller offset comes first.
    std::swap(NewOffset0, NewOffset1);
    std::swap(Data0, Data1);
  }

  assert((isUInt<8>(NewOffset0) && isUInt<8>(NewOffset1)) &&
         (NewOffset0 != NewOffset1) && "Computed offset doesn't fit");

  const MCInstrDesc &Write2Desc = TII->get(Opc);
  DebugLoc DL = CI.I->getDebugLoc();

  Register BaseReg = AddrReg->getReg();
  unsigned BaseSubReg = AddrReg->getSubReg();
  unsigned BaseRegFlags = 0;
  if (CI.BaseOff) {
    Register ImmReg = MRI->createVirtualRegister(&AMDGPU::SReg_32RegClass);
    BuildMI(*MBB, InsertBefore, DL, TII->get(AMDGPU::S_MOV_B32), ImmReg)
        .addImm(CI.BaseOff);

    BaseReg = MRI->createVirtualRegister(&AMDGPU::VGPR_32RegClass);
    BaseRegFlags = RegState::Kill;

    TII->getAddNoCarry(*MBB, InsertBefore, DL, BaseReg)
        .addReg(ImmReg)
        .addReg(AddrReg->getReg(), 0, BaseSubReg)
        .addImm(0); // clamp bit
    BaseSubReg = 0;
  }

  MachineInstrBuilder Write2 =
      BuildMI(*MBB, InsertBefore, DL, Write2Desc)
          .addReg(BaseReg, BaseRegFlags, BaseSubReg) // addr
          .add(*Data0)                               // data0
          .add(*Data1)                               // data1
          .addImm(NewOffset0)                        // offset0
          .addImm(NewOffset1)                        // offset1
          .addImm(0)                                 // gds
          .cloneMergedMemRefs({&*CI.I, &*Paired.I});

  CI.I->eraseFromParent();
  Paired.I->eraseFromParent();

  LLVM_DEBUG(dbgs() << "Inserted write2 inst: " << *Write2 << '\n');
  return Write2;
}
```
<a name="block_10"></a>**File:** llvm/lib/Target/AMDGPU/SILoadStoreOptimizer.cpp (L1480-1514) [<sup>↩</sup>](#ref-block_10)
```cpp
MachineBasicBlock::iterator
SILoadStoreOptimizer::mergeImagePair(CombineInfo &CI, CombineInfo &Paired,
                                     MachineBasicBlock::iterator InsertBefore) {
  MachineBasicBlock *MBB = CI.I->getParent();
  DebugLoc DL = CI.I->getDebugLoc();
  const unsigned Opcode = getNewOpcode(CI, Paired);

  const TargetRegisterClass *SuperRC = getTargetRegisterClass(CI, Paired);

  Register DestReg = MRI->createVirtualRegister(SuperRC);
  unsigned MergedDMask = CI.DMask | Paired.DMask;
  unsigned DMaskIdx =
      AMDGPU::getNamedOperandIdx(CI.I->getOpcode(), AMDGPU::OpName::dmask);

  auto MIB = BuildMI(*MBB, InsertBefore, DL, TII->get(Opcode), DestReg);
  for (unsigned I = 1, E = (*CI.I).getNumOperands(); I != E; ++I) {
    if (I == DMaskIdx)
      MIB.addImm(MergedDMask);
    else
      MIB.add((*CI.I).getOperand(I));
  }

  // It shouldn't be possible to get this far if the two instructions
  // don't have a single memoperand, because MachineInstr::mayAlias()
  // will return true if this is the case.
  assert(CI.I->hasOneMemOperand() && Paired.I->hasOneMemOperand());

  MachineInstr *New = MIB.addMemOperand(combineKnownAdjacentMMOs(CI, Paired));

  copyToDestRegs(CI, Paired, InsertBefore, AMDGPU::OpName::vdata, DestReg, New);

  CI.I->eraseFromParent();
  Paired.I->eraseFromParent();
  return New;
}
```
<a name="block_11"></a>**File:** llvm/lib/Target/AMDGPU/SILoadStoreOptimizer.cpp (L1516-1546) [<sup>↩</sup>](#ref-block_11)
```cpp
MachineBasicBlock::iterator SILoadStoreOptimizer::mergeSMemLoadImmPair(
    CombineInfo &CI, CombineInfo &Paired,
    MachineBasicBlock::iterator InsertBefore) {
  MachineBasicBlock *MBB = CI.I->getParent();
  DebugLoc DL = CI.I->getDebugLoc();
  const unsigned Opcode = getNewOpcode(CI, Paired);

  const TargetRegisterClass *SuperRC = getTargetRegisterClass(CI, Paired);

  Register DestReg = MRI->createVirtualRegister(SuperRC);
  unsigned MergedOffset = std::min(CI.Offset, Paired.Offset);

  // It shouldn't be possible to get this far if the two instructions
  // don't have a single memoperand, because MachineInstr::mayAlias()
  // will return true if this is the case.
  assert(CI.I->hasOneMemOperand() && Paired.I->hasOneMemOperand());

  MachineInstrBuilder New =
      BuildMI(*MBB, InsertBefore, DL, TII->get(Opcode), DestReg)
          .add(*TII->getNamedOperand(*CI.I, AMDGPU::OpName::sbase));
  if (CI.InstClass == S_BUFFER_LOAD_SGPR_IMM)
    New.add(*TII->getNamedOperand(*CI.I, AMDGPU::OpName::soffset));
  New.addImm(MergedOffset);
  New.addImm(CI.CPol).addMemOperand(combineKnownAdjacentMMOs(CI, Paired));

  copyToDestRegs(CI, Paired, InsertBefore, AMDGPU::OpName::sdst, DestReg, New);

  CI.I->eraseFromParent();
  Paired.I->eraseFromParent();
  return New;
}
```
<a name="block_12"></a>**File:** llvm/lib/Target/AMDGPU/SILoadStoreOptimizer.cpp (L1548-1587) [<sup>↩</sup>](#ref-block_12)
```cpp
MachineBasicBlock::iterator SILoadStoreOptimizer::mergeBufferLoadPair(
    CombineInfo &CI, CombineInfo &Paired,
    MachineBasicBlock::iterator InsertBefore) {
  MachineBasicBlock *MBB = CI.I->getParent();
  DebugLoc DL = CI.I->getDebugLoc();

  const unsigned Opcode = getNewOpcode(CI, Paired);

  const TargetRegisterClass *SuperRC = getTargetRegisterClass(CI, Paired);

  // Copy to the new source register.
  Register DestReg = MRI->createVirtualRegister(SuperRC);
  unsigned MergedOffset = std::min(CI.Offset, Paired.Offset);

  auto MIB = BuildMI(*MBB, InsertBefore, DL, TII->get(Opcode), DestReg);

  AddressRegs Regs = getRegs(Opcode, *TII);

  if (Regs.VAddr)
    MIB.add(*TII->getNamedOperand(*CI.I, AMDGPU::OpName::vaddr));

  // It shouldn't be possible to get this far if the two instructions
  // don't have a single memoperand, because MachineInstr::mayAlias()
  // will return true if this is the case.
  assert(CI.I->hasOneMemOperand() && Paired.I->hasOneMemOperand());

  MachineInstr *New =
    MIB.add(*TII->getNamedOperand(*CI.I, AMDGPU::OpName::srsrc))
        .add(*TII->getNamedOperand(*CI.I, AMDGPU::OpName::soffset))
        .addImm(MergedOffset) // offset
        .addImm(CI.CPol)      // cpol
        .addImm(0)            // swz
        .addMemOperand(combineKnownAdjacentMMOs(CI, Paired));

  copyToDestRegs(CI, Paired, InsertBefore, AMDGPU::OpName::vdata, DestReg, New);

  CI.I->eraseFromParent();
  Paired.I->eraseFromParent();
  return New;
}
```
<a name="block_13"></a>**File:** llvm/lib/Target/AMDGPU/SILoadStoreOptimizer.cpp (L1740-1894) [<sup>↩</sup>](#ref-block_13)
```cpp
unsigned SILoadStoreOptimizer::getNewOpcode(const CombineInfo &CI,
                                            const CombineInfo &Paired) {
  const unsigned Width = CI.Width + Paired.Width;

  switch (getCommonInstClass(CI, Paired)) {
  default:
    assert(CI.InstClass == BUFFER_LOAD || CI.InstClass == BUFFER_STORE);
    // FIXME: Handle d16 correctly
    return AMDGPU::getMUBUFOpcode(AMDGPU::getMUBUFBaseOpcode(CI.I->getOpcode()),
                                  Width);
  case TBUFFER_LOAD:
  case TBUFFER_STORE:
    return AMDGPU::getMTBUFOpcode(AMDGPU::getMTBUFBaseOpcode(CI.I->getOpcode()),
                                  Width);

  case UNKNOWN:
    llvm_unreachable("Unknown instruction class");
  case S_BUFFER_LOAD_IMM: {
    // If XNACK is enabled, use the constrained opcodes when the first load is
    // under-aligned.
    bool NeedsConstrainedOpc =
        needsConstrainedOpcode(*STM, CI.I->memoperands(), Width);
    switch (Width) {
    default:
      return 0;
    case 2:
      return NeedsConstrainedOpc ? AMDGPU::S_BUFFER_LOAD_DWORDX2_IMM_ec
                                 : AMDGPU::S_BUFFER_LOAD_DWORDX2_IMM;
    case 3:
      return NeedsConstrainedOpc ? AMDGPU::S_BUFFER_LOAD_DWORDX3_IMM_ec
                                 : AMDGPU::S_BUFFER_LOAD_DWORDX3_IMM;
    case 4:
      return NeedsConstrainedOpc ? AMDGPU::S_BUFFER_LOAD_DWORDX4_IMM_ec
                                 : AMDGPU::S_BUFFER_LOAD_DWORDX4_IMM;
    case 8:
      return NeedsConstrainedOpc ? AMDGPU::S_BUFFER_LOAD_DWORDX8_IMM_ec
                                 : AMDGPU::S_BUFFER_LOAD_DWORDX8_IMM;
    }
  }
  case S_BUFFER_LOAD_SGPR_IMM: {
    // If XNACK is enabled, use the constrained opcodes when the first load is
    // under-aligned.
    bool NeedsConstrainedOpc =
        needsConstrainedOpcode(*STM, CI.I->memoperands(), Width);
    switch (Width) {
    default:
      return 0;
    case 2:
      return NeedsConstrainedOpc ? AMDGPU::S_BUFFER_LOAD_DWORDX2_SGPR_IMM_ec
                                 : AMDGPU::S_BUFFER_LOAD_DWORDX2_SGPR_IMM;
    case 3:
      return NeedsConstrainedOpc ? AMDGPU::S_BUFFER_LOAD_DWORDX3_SGPR_IMM_ec
                                 : AMDGPU::S_BUFFER_LOAD_DWORDX3_SGPR_IMM;
    case 4:
      return NeedsConstrainedOpc ? AMDGPU::S_BUFFER_LOAD_DWORDX4_SGPR_IMM_ec
                                 : AMDGPU::S_BUFFER_LOAD_DWORDX4_SGPR_IMM;
    case 8:
      return NeedsConstrainedOpc ? AMDGPU::S_BUFFER_LOAD_DWORDX8_SGPR_IMM_ec
                                 : AMDGPU::S_BUFFER_LOAD_DWORDX8_SGPR_IMM;
    }
  }
  case S_LOAD_IMM: {
    // If XNACK is enabled, use the constrained opcodes when the first load is
    // under-aligned.
    bool NeedsConstrainedOpc =
        needsConstrainedOpcode(*STM, CI.I->memoperands(), Width);
    switch (Width) {
    default:
      return 0;
    case 2:
      return NeedsConstrainedOpc ? AMDGPU::S_LOAD_DWORDX2_IMM_ec
                                 : AMDGPU::S_LOAD_DWORDX2_IMM;
    case 3:
      return NeedsConstrainedOpc ? AMDGPU::S_LOAD_DWORDX3_IMM_ec
                                 : AMDGPU::S_LOAD_DWORDX3_IMM;
    case 4:
      return NeedsConstrainedOpc ? AMDGPU::S_LOAD_DWORDX4_IMM_ec
                                 : AMDGPU::S_LOAD_DWORDX4_IMM;
    case 8:
      return NeedsConstrainedOpc ? AMDGPU::S_LOAD_DWORDX8_IMM_ec
                                 : AMDGPU::S_LOAD_DWORDX8_IMM;
    }
  }
  case GLOBAL_LOAD:
    switch (Width) {
    default:
      return 0;
    case 2:
      return AMDGPU::GLOBAL_LOAD_DWORDX2;
    case 3:
      return AMDGPU::GLOBAL_LOAD_DWORDX3;
    case 4:
      return AMDGPU::GLOBAL_LOAD_DWORDX4;
    }
  case GLOBAL_LOAD_SADDR:
    switch (Width) {
    default:
      return 0;
    case 2:
      return AMDGPU::GLOBAL_LOAD_DWORDX2_SADDR;
    case 3:
      return AMDGPU::GLOBAL_LOAD_DWORDX3_SADDR;
    case 4:
      return AMDGPU::GLOBAL_LOAD_DWORDX4_SADDR;
    }
  case GLOBAL_STORE:
    switch (Width) {
    default:
      return 0;
    case 2:
      return AMDGPU::GLOBAL_STORE_DWORDX2;
    case 3:
      return AMDGPU::GLOBAL_STORE_DWORDX3;
    case 4:
      return AMDGPU::GLOBAL_STORE_DWORDX4;
    }
  case GLOBAL_STORE_SADDR:
    switch (Width) {
    default:
      return 0;
    case 2:
      return AMDGPU::GLOBAL_STORE_DWORDX2_SADDR;
    case 3:
      return AMDGPU::GLOBAL_STORE_DWORDX3_SADDR;
    case 4:
      return AMDGPU::GLOBAL_STORE_DWORDX4_SADDR;
    }
  case FLAT_LOAD:
    switch (Width) {
    default:
      return 0;
    case 2:
      return AMDGPU::FLAT_LOAD_DWORDX2;
    case 3:
      return AMDGPU::FLAT_LOAD_DWORDX3;
    case 4:
      return AMDGPU::FLAT_LOAD_DWORDX4;
    }
  case FLAT_STORE:
    switch (Width) {
    default:
      return 0;
    case 2:
      return AMDGPU::FLAT_STORE_DWORDX2;
    case 3:
      return AMDGPU::FLAT_STORE_DWORDX3;
    case 4:
      return AMDGPU::FLAT_STORE_DWORDX4;
    }
  case MIMG:
    assert(((unsigned)llvm::popcount(CI.DMask | Paired.DMask) == Width) &&
           "No overlaps");
    return AMDGPU::getMaskedMIMGOp(CI.I->getOpcode(), Width);
  }
}
```
<a name="block_14"></a>**File:** llvm/lib/Target/AMDGPU/SILoadStoreOptimizer.cpp (L1896-1927) [<sup>↩</sup>](#ref-block_14)
```cpp
std::pair<unsigned, unsigned>
SILoadStoreOptimizer::getSubRegIdxs(const CombineInfo &CI,
                                    const CombineInfo &Paired) {
  assert((CI.InstClass != MIMG ||
          ((unsigned)llvm::popcount(CI.DMask | Paired.DMask) ==
           CI.Width + Paired.Width)) &&
         "No overlaps");

  unsigned Idx0;
  unsigned Idx1;

  static const unsigned Idxs[5][4] = {
      {AMDGPU::sub0, AMDGPU::sub0_sub1, AMDGPU::sub0_sub1_sub2, AMDGPU::sub0_sub1_sub2_sub3},
      {AMDGPU::sub1, AMDGPU::sub1_sub2, AMDGPU::sub1_sub2_sub3, AMDGPU::sub1_sub2_sub3_sub4},
      {AMDGPU::sub2, AMDGPU::sub2_sub3, AMDGPU::sub2_sub3_sub4, AMDGPU::sub2_sub3_sub4_sub5},
      {AMDGPU::sub3, AMDGPU::sub3_sub4, AMDGPU::sub3_sub4_sub5, AMDGPU::sub3_sub4_sub5_sub6},
      {AMDGPU::sub4, AMDGPU::sub4_sub5, AMDGPU::sub4_sub5_sub6, AMDGPU::sub4_sub5_sub6_sub7},
  };

  assert(CI.Width >= 1 && CI.Width <= 4);
  assert(Paired.Width >= 1 && Paired.Width <= 4);

  if (Paired < CI) {
    Idx1 = Idxs[0][Paired.Width - 1];
    Idx0 = Idxs[Paired.Width][CI.Width - 1];
  } else {
    Idx0 = Idxs[0][CI.Width - 1];
    Idx1 = Idxs[CI.Width][Paired.Width - 1];
  }

  return {Idx0, Idx1};
}
```
<a name="block_15"></a>**File:** llvm/lib/Target/AMDGPU/SILoadStoreOptimizer.cpp (L2011-2066) [<sup>↩</sup>](#ref-block_15)
```cpp
Register SILoadStoreOptimizer::computeBase(MachineInstr &MI,
                                           const MemAddress &Addr) const {
  MachineBasicBlock *MBB = MI.getParent();
  MachineBasicBlock::iterator MBBI = MI.getIterator();
  DebugLoc DL = MI.getDebugLoc();

  assert((TRI->getRegSizeInBits(Addr.Base.LoReg, *MRI) == 32 ||
          Addr.Base.LoSubReg) &&
         "Expected 32-bit Base-Register-Low!!");

  assert((TRI->getRegSizeInBits(Addr.Base.HiReg, *MRI) == 32 ||
          Addr.Base.HiSubReg) &&
         "Expected 32-bit Base-Register-Hi!!");

  LLVM_DEBUG(dbgs() << "  Re-Computed Anchor-Base:\n");
  MachineOperand OffsetLo = createRegOrImm(static_cast<int32_t>(Addr.Offset), MI);
  MachineOperand OffsetHi =
    createRegOrImm(static_cast<int32_t>(Addr.Offset >> 32), MI);

  const auto *CarryRC = TRI->getWaveMaskRegClass();
  Register CarryReg = MRI->createVirtualRegister(CarryRC);
  Register DeadCarryReg = MRI->createVirtualRegister(CarryRC);

  Register DestSub0 = MRI->createVirtualRegister(&AMDGPU::VGPR_32RegClass);
  Register DestSub1 = MRI->createVirtualRegister(&AMDGPU::VGPR_32RegClass);
  MachineInstr *LoHalf =
    BuildMI(*MBB, MBBI, DL, TII->get(AMDGPU::V_ADD_CO_U32_e64), DestSub0)
      .addReg(CarryReg, RegState::Define)
      .addReg(Addr.Base.LoReg, 0, Addr.Base.LoSubReg)
      .add(OffsetLo)
      .addImm(0); // clamp bit
  (void)LoHalf;
  LLVM_DEBUG(dbgs() << "    "; LoHalf->dump(););

  MachineInstr *HiHalf =
  BuildMI(*MBB, MBBI, DL, TII->get(AMDGPU::V_ADDC_U32_e64), DestSub1)
    .addReg(DeadCarryReg, RegState::Define | RegState::Dead)
    .addReg(Addr.Base.HiReg, 0, Addr.Base.HiSubReg)
    .add(OffsetHi)
    .addReg(CarryReg, RegState::Kill)
    .addImm(0); // clamp bit
  (void)HiHalf;
  LLVM_DEBUG(dbgs() << "    "; HiHalf->dump(););

  Register FullDestReg = MRI->createVirtualRegister(TRI->getVGPR64Class());
  MachineInstr *FullBase =
    BuildMI(*MBB, MBBI, DL, TII->get(TargetOpcode::REG_SEQUENCE), FullDestReg)
      .addReg(DestSub0)
      .addImm(AMDGPU::sub0)
      .addReg(DestSub1)
      .addImm(AMDGPU::sub1);
  (void)FullBase;
  LLVM_DEBUG(dbgs() << "    "; FullBase->dump(); dbgs() << "\n";);

  return FullDestReg;
}
```
<a name="block_16"></a>**File:** llvm/lib/Target/AMDGPU/SILoadStoreOptimizer.cpp (L2104-2161) [<sup>↩</sup>](#ref-block_16)
```cpp
void SILoadStoreOptimizer::processBaseWithConstOffset(const MachineOperand &Base,
                                                      MemAddress &Addr) const {
  if (!Base.isReg())
    return;

  MachineInstr *Def = MRI->getUniqueVRegDef(Base.getReg());
  if (!Def || Def->getOpcode() != AMDGPU::REG_SEQUENCE
      || Def->getNumOperands() != 5)
    return;

  MachineOperand BaseLo = Def->getOperand(1);
  MachineOperand BaseHi = Def->getOperand(3);
  if (!BaseLo.isReg() || !BaseHi.isReg())
    return;

  MachineInstr *BaseLoDef = MRI->getUniqueVRegDef(BaseLo.getReg());
  MachineInstr *BaseHiDef = MRI->getUniqueVRegDef(BaseHi.getReg());

  if (!BaseLoDef || BaseLoDef->getOpcode() != AMDGPU::V_ADD_CO_U32_e64 ||
      !BaseHiDef || BaseHiDef->getOpcode() != AMDGPU::V_ADDC_U32_e64)
    return;

  const auto *Src0 = TII->getNamedOperand(*BaseLoDef, AMDGPU::OpName::src0);
  const auto *Src1 = TII->getNamedOperand(*BaseLoDef, AMDGPU::OpName::src1);

  auto Offset0P = extractConstOffset(*Src0);
  if (Offset0P)
    BaseLo = *Src1;
  else {
    if (!(Offset0P = extractConstOffset(*Src1)))
      return;
    BaseLo = *Src0;
  }

  if (!BaseLo.isReg())
    return;

  Src0 = TII->getNamedOperand(*BaseHiDef, AMDGPU::OpName::src0);
  Src1 = TII->getNamedOperand(*BaseHiDef, AMDGPU::OpName::src1);

  if (Src0->isImm())
    std::swap(Src0, Src1);

  if (!Src1->isImm() || Src0->isImm())
    return;

  uint64_t Offset1 = Src1->getImm();
  BaseHi = *Src0;

  if (!BaseHi.isReg())
    return;

  Addr.Base.LoReg = BaseLo.getReg();
  Addr.Base.HiReg = BaseHi.getReg();
  Addr.Base.LoSubReg = BaseLo.getSubReg();
  Addr.Base.HiSubReg = BaseHi.getSubReg();
  Addr.Offset = (*Offset0P & 0x00000000ffffffff) | (Offset1 << 32);
}
```
<a name="block_17"></a>**File:** llvm/lib/Target/AMDGPU/SILoadStoreOptimizer.cpp (L2163-2313) [<sup>↩</sup>](#ref-block_17)
```cpp
bool SILoadStoreOptimizer::promoteConstantOffsetToImm(
    MachineInstr &MI,
    MemInfoMap &Visited,
    SmallPtrSet<MachineInstr *, 4> &AnchorList) const {

  if (!STM->hasFlatInstOffsets() || !SIInstrInfo::isFLAT(MI))
    return false;

  // TODO: Support FLAT_SCRATCH. Currently code expects 64-bit pointers.
  if (SIInstrInfo::isFLATScratch(MI))
    return false;

  unsigned AS = SIInstrInfo::isFLATGlobal(MI) ? AMDGPUAS::GLOBAL_ADDRESS
                                              : AMDGPUAS::FLAT_ADDRESS;

  if (AnchorList.count(&MI))
    return false;

  LLVM_DEBUG(dbgs() << "\nTryToPromoteConstantOffsetToImmFor "; MI.dump());

  if (TII->getNamedOperand(MI, AMDGPU::OpName::offset)->getImm()) {
    LLVM_DEBUG(dbgs() << "  Const-offset is already promoted.\n";);
    return false;
  }

  // Step1: Find the base-registers and a 64bit constant offset.
  MachineOperand &Base = *TII->getNamedOperand(MI, AMDGPU::OpName::vaddr);
  auto [It, Inserted] = Visited.try_emplace(&MI);
  MemAddress MAddr;
  if (Inserted) {
    processBaseWithConstOffset(Base, MAddr);
    It->second = MAddr;
  } else
    MAddr = It->second;

  if (MAddr.Offset == 0) {
    LLVM_DEBUG(dbgs() << "  Failed to extract constant-offset or there are no"
                         " constant offsets that can be promoted.\n";);
    return false;
  }

  LLVM_DEBUG(dbgs() << "  BASE: {" << printReg(MAddr.Base.HiReg, TRI) << ", "
                    << printReg(MAddr.Base.LoReg, TRI)
                    << "} Offset: " << MAddr.Offset << "\n\n";);

  // Step2: Traverse through MI's basic block and find an anchor(that has the
  // same base-registers) with the highest 13bit distance from MI's offset.
  // E.g. (64bit loads)
  // bb:
  //   addr1 = &a + 4096;   load1 = load(addr1,  0)
  //   addr2 = &a + 6144;   load2 = load(addr2,  0)
  //   addr3 = &a + 8192;   load3 = load(addr3,  0)
  //   addr4 = &a + 10240;  load4 = load(addr4,  0)
  //   addr5 = &a + 12288;  load5 = load(addr5,  0)
  //
  // Starting from the first load, the optimization will try to find a new base
  // from which (&a + 4096) has 13 bit distance. Both &a + 6144 and &a + 8192
  // has 13bit distance from &a + 4096. The heuristic considers &a + 8192
  // as the new-base(anchor) because of the maximum distance which can
  // accommodate more intermediate bases presumably.
  //
  // Step3: move (&a + 8192) above load1. Compute and promote offsets from
  // (&a + 8192) for load1, load2, load4.
  //   addr = &a + 8192
  //   load1 = load(addr,       -4096)
  //   load2 = load(addr,       -2048)
  //   load3 = load(addr,       0)
  //   load4 = load(addr,       2048)
  //   addr5 = &a + 12288;  load5 = load(addr5,  0)
  //
  MachineInstr *AnchorInst = nullptr;
  MemAddress AnchorAddr;
  uint32_t MaxDist = std::numeric_limits<uint32_t>::min();
  SmallVector<std::pair<MachineInstr *, int64_t>, 4> InstsWCommonBase;

  MachineBasicBlock *MBB = MI.getParent();
  MachineBasicBlock::iterator E = MBB->end();
  MachineBasicBlock::iterator MBBI = MI.getIterator();
  ++MBBI;
  const SITargetLowering *TLI =
    static_cast<const SITargetLowering *>(STM->getTargetLowering());

  for ( ; MBBI != E; ++MBBI) {
    MachineInstr &MINext = *MBBI;
    // TODO: Support finding an anchor(with same base) from store addresses or
    // any other load addresses where the opcodes are different.
    if (MINext.getOpcode() != MI.getOpcode() ||
        TII->getNamedOperand(MINext, AMDGPU::OpName::offset)->getImm())
      continue;

    const MachineOperand &BaseNext =
      *TII->getNamedOperand(MINext, AMDGPU::OpName::vaddr);
    MemAddress MAddrNext;
    auto [It, Inserted] = Visited.try_emplace(&MINext);
    if (Inserted) {
      processBaseWithConstOffset(BaseNext, MAddrNext);
      It->second = MAddrNext;
    } else
      MAddrNext = It->second;

    if (MAddrNext.Base.LoReg != MAddr.Base.LoReg ||
        MAddrNext.Base.HiReg != MAddr.Base.HiReg ||
        MAddrNext.Base.LoSubReg != MAddr.Base.LoSubReg ||
        MAddrNext.Base.HiSubReg != MAddr.Base.HiSubReg)
      continue;

    InstsWCommonBase.emplace_back(&MINext, MAddrNext.Offset);

    int64_t Dist = MAddr.Offset - MAddrNext.Offset;
    TargetLoweringBase::AddrMode AM;
    AM.HasBaseReg = true;
    AM.BaseOffs = Dist;
    if (TLI->isLegalFlatAddressingMode(AM, AS) &&
        (uint32_t)std::abs(Dist) > MaxDist) {
      MaxDist = std::abs(Dist);

      AnchorAddr = MAddrNext;
      AnchorInst = &MINext;
    }
  }

  if (AnchorInst) {
    LLVM_DEBUG(dbgs() << "  Anchor-Inst(with max-distance from Offset): ";
               AnchorInst->dump());
    LLVM_DEBUG(dbgs() << "  Anchor-Offset from BASE: "
               <<  AnchorAddr.Offset << "\n\n");

    // Instead of moving up, just re-compute anchor-instruction's base address.
    Register Base = computeBase(MI, AnchorAddr);

    updateBaseAndOffset(MI, Base, MAddr.Offset - AnchorAddr.Offset);
    LLVM_DEBUG(dbgs() << "  After promotion: "; MI.dump(););

    for (auto [OtherMI, OtherOffset] : InstsWCommonBase) {
      TargetLoweringBase::AddrMode AM;
      AM.HasBaseReg = true;
      AM.BaseOffs = OtherOffset - AnchorAddr.Offset;

      if (TLI->isLegalFlatAddressingMode(AM, AS)) {
        LLVM_DEBUG(dbgs() << "  Promote Offset(" << OtherOffset; dbgs() << ")";
                   OtherMI->dump());
        updateBaseAndOffset(*OtherMI, Base, OtherOffset - AnchorAddr.Offset);
        LLVM_DEBUG(dbgs() << "     After promotion: "; OtherMI->dump());
      }
    }
    AnchorList.insert(AnchorInst);
    return true;
  }

  return false;
}
```
<a name="block_18"></a>**File:** llvm/lib/Target/AMDGPU/SILoadStoreOptimizer.cpp (L2330-2420) [<sup>↩</sup>](#ref-block_18)
```cpp
std::pair<MachineBasicBlock::iterator, bool>
SILoadStoreOptimizer::collectMergeableInsts(
    MachineBasicBlock::iterator Begin, MachineBasicBlock::iterator End,
    MemInfoMap &Visited, SmallPtrSet<MachineInstr *, 4> &AnchorList,
    std::list<std::list<CombineInfo>> &MergeableInsts) const {
  bool Modified = false;

  // Sort potential mergeable instructions into lists.  One list per base address.
  unsigned Order = 0;
  MachineBasicBlock::iterator BlockI = Begin;
  for (; BlockI != End; ++BlockI) {
    MachineInstr &MI = *BlockI;

    // We run this before checking if an address is mergeable, because it can produce
    // better code even if the instructions aren't mergeable.
    if (promoteConstantOffsetToImm(MI, Visited, AnchorList))
      Modified = true;

    // Treat volatile accesses, ordered accesses and unmodeled side effects as
    // barriers. We can look after this barrier for separate merges.
    if (MI.hasOrderedMemoryRef() || MI.hasUnmodeledSideEffects()) {
      LLVM_DEBUG(dbgs() << "Breaking search on barrier: " << MI);

      // Search will resume after this instruction in a separate merge list.
      ++BlockI;
      break;
    }

    const InstClassEnum InstClass = getInstClass(MI.getOpcode(), *TII);
    if (InstClass == UNKNOWN)
      continue;

    // Do not merge VMEM buffer instructions with "swizzled" bit set.
    int Swizzled =
        AMDGPU::getNamedOperandIdx(MI.getOpcode(), AMDGPU::OpName::swz);
    if (Swizzled != -1 && MI.getOperand(Swizzled).getImm())
      continue;

    CombineInfo CI;
    CI.setMI(MI, *this);
    CI.Order = Order++;

    if (!CI.hasMergeableAddress(*MRI))
      continue;

    if (CI.InstClass == DS_WRITE && CI.IsAGPR) {
      // FIXME: nothing is illegal in a ds_write2 opcode with two AGPR data
      //        operands. However we are reporting that ds_write2 shall have
      //        only VGPR data so that machine copy propagation does not
      //        create an illegal instruction with a VGPR and AGPR sources.
      //        Consequenctially if we create such instruction the verifier
      //        will complain.
      continue;
    }

    LLVM_DEBUG(dbgs() << "Mergeable: " << MI);

    addInstToMergeableList(CI, MergeableInsts);
  }

  // At this point we have lists of Mergeable instructions.
  //
  // Part 2: Sort lists by offset and then for each CombineInfo object in the
  // list try to find an instruction that can be merged with I.  If an instruction
  // is found, it is stored in the Paired field.  If no instructions are found, then
  // the CombineInfo object is deleted from the list.

  for (std::list<std::list<CombineInfo>>::iterator I = MergeableInsts.begin(),
                                                   E = MergeableInsts.end(); I != E;) {

    std::list<CombineInfo> &MergeList = *I;
    if (MergeList.size() <= 1) {
      // This means we have found only one instruction with a given address
      // that can be merged, and we need at least 2 instructions to do a merge,
      // so this list can be discarded.
      I = MergeableInsts.erase(I);
      continue;
    }

    // Sort the lists by offsets, this way mergeable instructions will be
    // adjacent to each other in the list, which will make it easier to find
    // matches.
    MergeList.sort(
        [] (const CombineInfo &A, const CombineInfo &B) {
          return A.Offset < B.Offset;
        });
    ++I;
  }

  return {BlockI, Modified};
}
```
<a name="block_19"></a>**File:** llvm/lib/Target/AMDGPU/SILoadStoreOptimizer.cpp (L2425-2453) [<sup>↩</sup>](#ref-block_19)
```cpp
bool SILoadStoreOptimizer::optimizeBlock(
                       std::list<std::list<CombineInfo> > &MergeableInsts) {
  bool Modified = false;

  for (std::list<std::list<CombineInfo>>::iterator I = MergeableInsts.begin(),
                                                   E = MergeableInsts.end(); I != E;) {
    std::list<CombineInfo> &MergeList = *I;

    bool OptimizeListAgain = false;
    if (!optimizeInstsWithSameBaseAddr(MergeList, OptimizeListAgain)) {
      // We weren't able to make any changes, so delete the list so we don't
      // process the same instructions the next time we try to optimize this
      // block.
      I = MergeableInsts.erase(I);
      continue;
    }

    Modified = true;

    // We made changes, but also determined that there were no more optimization
    // opportunities, so we don't need to reprocess the list
    if (!OptimizeListAgain) {
      I = MergeableInsts.erase(I);
      continue;
    }
    OptimizeAgain = true;
  }
  return Modified;
}
```
<a name="block_20"></a>**File:** llvm/lib/Target/AMDGPU/SILoadStoreOptimizer.cpp (L2455-2544) [<sup>↩</sup>](#ref-block_20)
```cpp
bool
SILoadStoreOptimizer::optimizeInstsWithSameBaseAddr(
                                          std::list<CombineInfo> &MergeList,
                                          bool &OptimizeListAgain) {
  if (MergeList.empty())
    return false;

  bool Modified = false;

  for (auto I = MergeList.begin(), Next = std::next(I); Next != MergeList.end();
       Next = std::next(I)) {

    auto First = I;
    auto Second = Next;

    if ((*First).Order > (*Second).Order)
      std::swap(First, Second);
    CombineInfo &CI = *First;
    CombineInfo &Paired = *Second;

    CombineInfo *Where = checkAndPrepareMerge(CI, Paired);
    if (!Where) {
      ++I;
      continue;
    }

    Modified = true;

    LLVM_DEBUG(dbgs() << "Merging: " << *CI.I << "   with: " << *Paired.I);

    MachineBasicBlock::iterator NewMI;
    switch (CI.InstClass) {
    default:
      llvm_unreachable("unknown InstClass");
      break;
    case DS_READ:
      NewMI = mergeRead2Pair(CI, Paired, Where->I);
      break;
    case DS_WRITE:
      NewMI = mergeWrite2Pair(CI, Paired, Where->I);
      break;
    case S_BUFFER_LOAD_IMM:
    case S_BUFFER_LOAD_SGPR_IMM:
    case S_LOAD_IMM:
      NewMI = mergeSMemLoadImmPair(CI, Paired, Where->I);
      OptimizeListAgain |= CI.Width + Paired.Width < 8;
      break;
    case BUFFER_LOAD:
      NewMI = mergeBufferLoadPair(CI, Paired, Where->I);
      OptimizeListAgain |= CI.Width + Paired.Width < 4;
      break;
    case BUFFER_STORE:
      NewMI = mergeBufferStorePair(CI, Paired, Where->I);
      OptimizeListAgain |= CI.Width + Paired.Width < 4;
      break;
    case MIMG:
      NewMI = mergeImagePair(CI, Paired, Where->I);
      OptimizeListAgain |= CI.Width + Paired.Width < 4;
      break;
    case TBUFFER_LOAD:
      NewMI = mergeTBufferLoadPair(CI, Paired, Where->I);
      OptimizeListAgain |= CI.Width + Paired.Width < 4;
      break;
    case TBUFFER_STORE:
      NewMI = mergeTBufferStorePair(CI, Paired, Where->I);
      OptimizeListAgain |= CI.Width + Paired.Width < 4;
      break;
    case FLAT_LOAD:
    case GLOBAL_LOAD:
    case GLOBAL_LOAD_SADDR:
      NewMI = mergeFlatLoadPair(CI, Paired, Where->I);
      OptimizeListAgain |= CI.Width + Paired.Width < 4;
      break;
    case FLAT_STORE:
    case GLOBAL_STORE:
    case GLOBAL_STORE_SADDR:
      NewMI = mergeFlatStorePair(CI, Paired, Where->I);
      OptimizeListAgain |= CI.Width + Paired.Width < 4;
      break;
    }
    CI.setMI(NewMI, *this);
    CI.Order = Where->Order;
    if (I == Second)
      I = Next;

    MergeList.erase(Second);
  }

  return Modified;
}
```
<a name="block_21"></a>**File:** llvm/lib/Target/AMDGPU/SILoadStoreOptimizer.cpp (L2554-2598) [<sup>↩</sup>](#ref-block_21)
```cpp
bool SILoadStoreOptimizer::run(MachineFunction &MF) {
  STM = &MF.getSubtarget<GCNSubtarget>();
  if (!STM->loadStoreOptEnabled())
    return false;

  TII = STM->getInstrInfo();
  TRI = &TII->getRegisterInfo();

  MRI = &MF.getRegInfo();

  LLVM_DEBUG(dbgs() << "Running SILoadStoreOptimizer\n");

  bool Modified = false;

  // Contains the list of instructions for which constant offsets are being
  // promoted to the IMM. This is tracked for an entire block at time.
  SmallPtrSet<MachineInstr *, 4> AnchorList;
  MemInfoMap Visited;

  for (MachineBasicBlock &MBB : MF) {
    MachineBasicBlock::iterator SectionEnd;
    for (MachineBasicBlock::iterator I = MBB.begin(), E = MBB.end(); I != E;
         I = SectionEnd) {
      bool CollectModified;
      std::list<std::list<CombineInfo>> MergeableInsts;

      // First pass: Collect list of all instructions we know how to merge in a
      // subset of the block.
      std::tie(SectionEnd, CollectModified) =
          collectMergeableInsts(I, E, Visited, AnchorList, MergeableInsts);

      Modified |= CollectModified;

      do {
        OptimizeAgain = false;
        Modified |= optimizeBlock(MergeableInsts);
      } while (OptimizeAgain);
    }

    Visited.clear();
    AnchorList.clear();
  }

  return Modified;
}
```