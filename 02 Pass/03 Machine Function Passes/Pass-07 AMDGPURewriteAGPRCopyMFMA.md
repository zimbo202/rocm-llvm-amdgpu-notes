# AMDGPURewriteAGPRCopyMFMA Pass 功能详解

## 1. 主要功能概述

<a name="ref-block_0"></a>该 Pass 的主要功能是**优化 MFMA（Matrix Fused Multiply-Add，矩阵融合乘加）指令的寄存器使用**，尝试将使用 VGPR（Vector General Purpose Registers）的 MFMA 指令替换为使用 AGPR（Accumulation General Purpose Registers）的版本。 llvm-project:9-14[<sup>↗</sup>](#block_0) 

**主要作用：**
- 消除 AGPR 和 VGPR 之间不必要的跨寄存器组拷贝
- 当寄存器分配器已经将虚拟寄存器分配到 AGPR 时，直接使用 AGPR 版本的 MFMA 指令
- 避免因寄存器溢出而产生的性能损失

**主要效果：**
- 减少寄存器拷贝指令，提高代码效率
- 更有效地利用 AGPR 资源
- 优化矩阵运算的性能

## 2. 主要功能步骤提取

该 Pass 的实现包含以下主要步骤和子功能：

<a name="ref-block_3"></a>### 步骤 A: 初始化和前置检查 llvm-project:92-100[<sup>↗</sup>](#block_3) 

<a name="ref-block_4"></a>### 步骤 B: 虚拟寄存器遍历和筛选 llvm-project:104-118[<sup>↗</sup>](#block_4) 

<a name="ref-block_5"></a>### 步骤 C: 生命周期分析和拷贝指令识别 llvm-project:119-142[<sup>↗</sup>](#block_5) 

<a name="ref-block_7"></a>### 步骤 D: 寄存器约束验证 llvm-project:176-203[<sup>↗</sup>](#block_7) 

<a name="ref-block_8"></a>### 步骤 E: 指令替换和寄存器重写 llvm-project:205-218[<sup>↗</sup>](#block_8) 

<a name="ref-block_9"></a>### 步骤 F: 清理和生命周期更新 llvm-project:219-227[<sup>↗</sup>](#block_9) 

<a name="ref-block_2"></a>### 辅助功能: 寄存器类重新计算 llvm-project:71-90[<sup>↗</sup>](#block_2) 

## 3. 各步骤详细分析

### A. 初始化和前置检查
**功能描述：**
- 检查目标架构是否支持可配置的 AGPR vs VGPR 分配（需要 GFX90A 指令集）
- 验证是否有 AGPR 被实际使用（检查 `AGPR0` 是否被使用）
- 如果不满足条件，提前退出以避免不必要的计算

**关键检查：**
- `ST.hasGFX90AInsts()`: 确保硬件支持
- `LRM.isPhysRegUsed(AMDGPU::AGPR0)`: 确保有 AGPR 被分配

### B. 虚拟寄存器遍历和筛选
**功能描述：**
- 遍历所有虚拟寄存器
- 筛选出已分配物理寄存器的虚拟寄存器
- 进一步筛选出 AV_* 寄存器类（向量超类）
- 只关注那些被分配到 AGPR 的寄存器

**筛选条件：**
- 虚拟寄存器必须有对应的物理寄存器分配
- 虚拟寄存器类必须是向量超类（`isVectorSuperClass`）
- 分配的物理寄存器必须属于 AGPR 类（`isAGPRClass`）

### C. 生命周期分析和拷贝指令识别
**功能描述：**
- 对每个符合条件的虚拟寄存器，分析其生命周期信息
- 识别定义该寄存器的指令是否为拷贝指令（`isFullCopy`）
- 追踪拷贝指令的源寄存器
- 查找源寄存器的定义指令，判断是否为 MFMA 指令
- 使用 `getMFMASrcCVDstAGPROp` 检查是否存在对应的 AGPR 版本 MFMA 指令

**关键模式识别：**
寻找形如 `AGPR = COPY (MFMA_VGPR)` 的模式

### D. 寄存器约束验证
**功能描述：**
- 检查 MFMA 指令的 `src2` 操作数是否与拷贝源寄存器相同（tied 情况）
- 使用 `recomputeRegClassExcept` 重新计算除 MFMA 指令外的其他使用对 `src2` 的寄存器类约束
- 验证新的 AGPR 版本 MFMA 指令的寄存器类约束
- 检查是否可以找到一个兼容的寄存器类，同时满足：
  - 其他使用的约束
  - 新 MFMA 指令的约束

**约束检查逻辑：**
- 计算排除 MFMA 指令后的寄存器类约束
- 获取 AGPR 版本 MFMA 指令对 `src2` 的约束
- 使用 `getCommonSubClass` 找到公共子类
- 如果找不到公共子类，说明不能安全替换

### E. 指令替换和寄存器重写
**功能描述：**
- 更新虚拟寄存器的寄存器类为实际分配的 AGPR 类
- 更新 `src2` 寄存器的寄存器类
- 将 MFMA 指令的操作码替换为 AGPR 版本
- 使用 `replaceRegWith` 将拷贝源寄存器的所有使用替换为目标 AGPR 寄存器

**替换操作：**
- `setRegClass`: 更新寄存器类信息
- `setDesc`: 更换指令描述符（操作码）
- `replaceRegWith`: 全局替换寄存器引用

### F. 清理和生命周期更新
**功能描述：**
- 删除已经变成恒等拷贝的原始拷贝指令
- 从 LiveIntervals 映射中移除该指令
- 从物理寄存器中取消分配拷贝源寄存器的生命周期
- 删除过时的生命周期区间信息
- 为更新后的寄存器重新计算生命周期信息

**清理步骤：**
- `RemoveMachineInstrFromMaps`: 从生命周期分析中移除指令
- `eraseFromParent`: 从基本块中删除指令
- `unassign`: 取消物理寄存器分配
- `removeInterval`: 删除旧的生命周期区间
- `createAndComputeVirtRegInterval`: 重新计算生命周期

### 辅助功能: `recomputeRegClassExcept`
**功能描述：**
- 计算一个虚拟寄存器的寄存器类约束，但排除指定指令的影响
- 遍历寄存器的所有使用（排除调试信息）
- 对每个使用应用其对寄存器类的约束效果
- 返回满足所有使用约束的寄存器类

**应用场景：**
用于验证如果移除某条指令（如将要被替换的 MFMA 指令），剩余的使用是否能够兼容新的寄存器类。

## 4. 步骤之间的关系

### 顺序依赖关系：
```
A (初始化检查) 
    ↓
B (寄存器遍历筛选) 
    ↓
C (模式识别) ← 使用辅助功能
    ↓
D (约束验证) ← 使用辅助功能
    ↓
E (执行替换)
    ↓
F (清理更新)
```

### 详细关系描述：

1. **A → B**: 初始化检查确保环境满足要求后，才开始遍历寄存器。这是性能优化，避免在不支持的架构上做无用功。

2. **B → C**: 寄存器筛选确定了候选集合，然后对每个候选进行生命周期分析和模式识别。B 提供的 AGPR 分配信息是 C 中判断是否需要优化的前提。

3. **C → D**: 识别出 `COPY-MFMA` 模式后，需要验证替换的可行性。C 找到的 MFMA 指令和相关寄存器是 D 中进行约束检查的输入。

4. **辅助功能与 D 的关系**: `recomputeRegClassExcept` 是 D 的核心工具，用于计算排除 MFMA 指令后其他使用对寄存器的约束。这确保替换后不会违反其他指令的要求。

5. **D → E**: 只有通过所有约束验证后，才会执行实际的替换操作。D 的成功是 E 执行的必要条件。

6. **E → F**: 替换操作完成后，必须清理遗留的拷贝指令和更新生命周期信息。E 的修改使得原拷贝指令变成恒等拷贝，F 负责删除它并维护数据结构一致性。

### 循环迭代：
<a name="ref-block_10"></a>步骤 B-F 在一个循环中对所有虚拟寄存器执行，每次成功替换后设置 `MadeChange = true`。 llvm-project:228-229[<sup>↗</sup>](#block_10) 

## Notes

**当前限制：**
<a name="ref-block_6"></a>该 Pass 目前只处理 tied 情况（即 MFMA 的 `dst` 和 `src2` 使用相同寄存器）。 llvm-project:162-174[<sup>↗</sup>](#block_6) 

**未来改进方向：**
文件开头的 TODO 列出了三个改进方向：
1. 处理非 tied 的 dst+src2 情况
2. 处理同一寄存器被多个 MFMA 使用的情况（如链式 MFMA）
<a name="ref-block_1"></a>3. 增量更新 LiveIntervals 而不是从头重新计算 llvm-project:16-25[<sup>↗</sup>](#block_1) 

该 Pass 专门针对 GFX90A 架构及更新的 AMD GPU，利用其可配置的 AGPR/VGPR 分配特性来优化矩阵运算性能。
### Citations
<a name="block_0"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPURewriteAGPRCopyMFMA.cpp (L9-14) [<sup>↩</sup>](#ref-block_0)
```cpp
/// \file \brief Try to replace MFMA instructions using VGPRs with MFMA
/// instructions using AGPRs. We expect MFMAs to be selected using VGPRs, and
/// only use AGPRs if it helps avoid spilling. In this case, the MFMA will have
/// copies between AGPRs and VGPRs and the AGPR variant of an MFMA pseudo. This
/// pass will attempt to delete the cross register bank copy and replace the
/// MFMA opcode.
```
<a name="block_1"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPURewriteAGPRCopyMFMA.cpp (L16-25) [<sup>↩</sup>](#ref-block_1)
```cpp
/// TODO:
///  - Handle non-tied dst+src2 cases. We need to try to find a copy from an
///    AGPR from src2, or reassign src2 to an available AGPR (which should work
///    in the common case of a load).
///
///  - Handle multiple MFMA uses of the same register. e.g. chained MFMAs that
///    can be rewritten as a set
///
///  - Update LiveIntervals incrementally instead of recomputing from scratch
///
```
<a name="block_2"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPURewriteAGPRCopyMFMA.cpp (L71-90) [<sup>↩</sup>](#ref-block_2)
```cpp
const TargetRegisterClass *
AMDGPURewriteAGPRCopyMFMAImpl::recomputeRegClassExcept(
    Register Reg, const TargetRegisterClass *OldRC,
    const TargetRegisterClass *NewRC, const MachineInstr *ExceptMI) const {

  // Accumulate constraints from all uses.
  for (MachineOperand &MO : MRI.reg_nodbg_operands(Reg)) {
    // Apply the effect of the given operand to NewRC.
    MachineInstr *MI = MO.getParent();
    if (MI == ExceptMI)
      continue;

    unsigned OpNo = &MO - &MI->getOperand(0);
    NewRC = MI->getRegClassConstraintEffect(OpNo, NewRC, &TII, &TRI);
    if (!NewRC || NewRC == OldRC)
      return nullptr;
  }

  return NewRC;
}
```
<a name="block_3"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPURewriteAGPRCopyMFMA.cpp (L92-100) [<sup>↩</sup>](#ref-block_3)
```cpp
bool AMDGPURewriteAGPRCopyMFMAImpl::run(MachineFunction &MF) const {
  // This only applies on subtargets that have a configurable AGPR vs. VGPR
  // allocation.
  if (!ST.hasGFX90AInsts())
    return false;

  // Early exit if no AGPRs were assigned.
  if (!LRM.isPhysRegUsed(AMDGPU::AGPR0))
    return false;
```
<a name="block_4"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPURewriteAGPRCopyMFMA.cpp (L104-118) [<sup>↩</sup>](#ref-block_4)
```cpp
  for (unsigned I = 0, E = MRI.getNumVirtRegs(); I != E; ++I) {
    Register VReg = Register::index2VirtReg(I);
    Register PhysReg = VRM.getPhys(VReg);
    if (!PhysReg)
      continue;

    // Find AV_* registers assigned to AGPRs.
    const TargetRegisterClass *VirtRegRC = MRI.getRegClass(VReg);
    if (!TRI.isVectorSuperClass(VirtRegRC))
      continue;

    const TargetRegisterClass *AssignedRC = TRI.getPhysRegBaseClass(PhysReg);
    if (!TRI.isAGPRClass(AssignedRC))
      continue;

```
<a name="block_5"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPURewriteAGPRCopyMFMA.cpp (L119-142) [<sup>↩</sup>](#ref-block_5)
```cpp
    LiveInterval &LI = LIS.getInterval(VReg);

    // TODO: Test multiple uses
    for (VNInfo *VNI : LI.vnis()) {
      MachineInstr *DefMI = LIS.getInstructionFromIndex(VNI->def);

      // TODO: Handle SplitKit produced copy bundles for partially defined
      // registers.
      if (!DefMI || !DefMI->isFullCopy())
        continue;

      Register CopySrcReg = DefMI->getOperand(1).getReg();
      if (!CopySrcReg.isVirtual())
        continue;

      LiveInterval &CopySrcLI = LIS.getInterval(CopySrcReg);
      LiveQueryResult LRQ = CopySrcLI.Query(VNI->def.getRegSlot());
      MachineInstr *CopySrcMI = LIS.getInstructionFromIndex(LRQ.valueIn()->def);
      if (!CopySrcMI)
        continue;

      int AGPROp = AMDGPU::getMFMASrcCVDstAGPROp(CopySrcMI->getOpcode());
      if (AGPROp == -1)
        continue;
```
<a name="block_6"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPURewriteAGPRCopyMFMA.cpp (L162-174) [<sup>↩</sup>](#ref-block_6)
```cpp
      if (Src2->getReg() != CopySrcReg) {
        LLVM_DEBUG(
            dbgs()
            << "Replacing untied VGPR MFMAs with AGPR form not yet handled\n");
        // TODO: Only handles the tied case for now. If the input operand is a
        // different register, we need to also reassign it (either by looking
        // for a compatible copy-from-AGPR, or by seeing if an available AGPR is
        // compatible with all other uses.

        // If we can't reassign it, we'd need to introduce a different copy
        // which is likely worse than the copy we'd be saving.
        continue;
      }
```
<a name="block_7"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPURewriteAGPRCopyMFMA.cpp (L176-203) [<sup>↩</sup>](#ref-block_7)
```cpp
      const TargetRegisterClass *Src2VirtRegRC =
          MRI.getRegClass(Src2->getReg());

      // We've found av = COPY (MFMA), and need to verify that we can trivially
      // rewrite src2 to use the new AGPR. If we can't trivially replace it,
      // we're going to induce as many copies as we would have emitted in the
      // first place, as well as need to assign another register, and need to
      // figure out where to put them. The live range splitting is smarter than
      // anything we're doing here, so trust it did something reasonable.
      const TargetRegisterClass *Src2ExceptRC = recomputeRegClassExcept(
          Src2->getReg(), Src2VirtRegRC, VirtRegRC, CopySrcMI);
      if (!Src2ExceptRC)
        continue;

      const TargetRegisterClass *NewSrc2ConstraintRC =
          TII.getRegClass(TII.get(AGPROp), Src2->getOperandNo(), &TRI, MF);

      // Try to constrain src2 to the replacement instruction candidate's
      // register class.
      const TargetRegisterClass *NewSrc2RC =
          TRI.getCommonSubClass(Src2ExceptRC, NewSrc2ConstraintRC);
      if (!NewSrc2RC) {
        // TODO: This is ignoring ther rewritable uses. e.g. a rewritable MFMA
        // using a rewritable MFMA can be rewritten as a pair.
        LLVM_DEBUG(dbgs() << "Other uses of " << printReg(Src2->getReg(), &TRI)
                          << " are incompatible with replacement class\n");
        continue;
      }
```
<a name="block_8"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPURewriteAGPRCopyMFMA.cpp (L205-218) [<sup>↩</sup>](#ref-block_8)
```cpp
      MRI.setRegClass(VReg, AssignedRC);
      MRI.setRegClass(Src2->getReg(), NewSrc2RC);

      CopySrcMI->setDesc(TII.get(AGPROp));

      // TODO: Is replacing too aggressive, fixup these instructions only?
      MRI.replaceRegWith(CopySrcReg, VReg);

      LLVM_DEBUG(dbgs() << "Replaced VGPR MFMA with AGPR: " << *CopySrcMI);

      // We left behind an identity copy, so delete it.
      LIS.RemoveMachineInstrFromMaps(*DefMI);
      DefMI->eraseFromParent();

```
<a name="block_9"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPURewriteAGPRCopyMFMA.cpp (L219-227) [<sup>↩</sup>](#ref-block_9)
```cpp
      LRM.unassign(CopySrcLI);

      // We don't need the liveness information anymore, so don't bother
      // updating the intervals. Just delete the stale information.
      // TODO: Is it worth preserving these?
      LIS.removeInterval(CopySrcReg);
      LIS.removeInterval(VReg);
      LIS.createAndComputeVirtRegInterval(VReg);

```
<a name="block_10"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPURewriteAGPRCopyMFMA.cpp (L228-229) [<sup>↩</sup>](#ref-block_10)
```cpp
      MadeChange = true;
    }
```