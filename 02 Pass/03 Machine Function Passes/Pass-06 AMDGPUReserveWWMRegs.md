# AMDGPUReserveWWMRegs.cpp 代码功能解析

## 1. Pass的主要功能概述

<a name="ref-block_0"></a>`AMDGPUReserveWWMRegs` pass 是 AMDGPU 后端在 WWM（Whole Wave Mode，全波模式）寄存器分配流水线末尾执行的一个关键 pass。 llvm-project:10-13[<sup>↗</sup>](#block_0) 

**主要作用：**
- 识别在 WWM 寄存器分配流水线中已分配的 WWM 寄存器
- 将这些寄存器添加到保留寄存器列表中
- 确保这些寄存器在后续的常规 VGPR 分配流水线中不会被重新分配

**产生的效果：**
- 保护 WWM 寄存器不被常规寄存器分配器使用
- 确保需要全波模式执行的操作所使用的寄存器不会被覆盖
- 维护 WWM 代码和非 WWM 代码之间的寄存器隔离

## 2. 主要功能步骤提取

该 pass 的核心实现在 `AMDGPUReserveWWMRegs::run()` 方法中，包含以下主要步骤：

1. **遍历所有机器指令**
2. **识别 WWM 寄存器溢出/恢复指令**
3. **提取物理寄存器并标记为保留**
4. **清除可重命名标志**
5. **清理 NonWWMRegMask**

## 3. 各步骤的具体描述分析

### 步骤 1：遍历所有机器指令 llvm-project:80-85 

该步骤通过双重循环遍历函数中的所有基本块和指令，为后续的 WWM 寄存器识别做准备。

### 步骤 2：识别 WWM 寄存器溢出/恢复指令 llvm-project:82-89 

这一步检查指令操作码是否为 `SI_SPILL_S32_TO_VGPR`（SGPR 溢出到 VGPR）或 `SI_RESTORE_S32_FROM_VGPR`（从 VGPR 恢复 SGPR）。这两个伪指令专门用于 SGPR 到 VGPR lane 的溢出/恢复操作。

**关键点：**
- `SI_SPILL_S32_TO_VGPR`：目标操作数（operand 0）是被写入的 VGPR
- `SI_RESTORE_S32_FROM_VGPR`：源操作数（operand 1）是被读取的 VGPR

### 步骤 3：提取物理寄存器并标记为保留 llvm-project:87-95 

从识别出的指令中提取实际使用的物理寄存器，并通过 `MFI->reserveWWMRegister(Reg)` 将其添加到保留列表。代码断言这些寄存器必须是物理寄存器，因为此时所有 WWM 寄存器应该已经被分配。

<a name="ref-block_5"></a>**reserveWWMRegister 的作用：** llvm-project:643-643[<sup>↗</sup>](#block_5) 

该方法将寄存器插入到 `WWMReservedRegs` 集合中，这个集合在后续的寄存器分配中会被查询以确定哪些寄存器不可用。

<a name="ref-block_2"></a>### 步骤 4：清除可重命名标志 llvm-project:99-106[<sup>↗</sup>](#block_2) 

遍历所有被保留的 WWM 寄存器，对于这些寄存器的所有使用点（MachineOperand），将其 `IsRenamable` 标志设置为 false。这是因为保留寄存器不能被重命名。

**原因：**
保留寄存器在寄存器分配器看来是"固定的"，不能参与寄存器重命名优化，因此需要明确标记。

<a name="ref-block_3"></a>### 步骤 5：清理 NonWWMRegMask llvm-project:108-109[<sup>↗</sup>](#block_3) 

调用 `clearNonWWMRegAllocMask()` 清除之前在 WWM 寄存器分配期间设置的 NonWWMRegMask。

<a name="ref-block_4"></a>**NonWWMRegMask 的作用：** llvm-project:556-559[<sup>↗</sup>](#block_4) 

在分配之前，VGPR 被划分为两个不同的集合：WWM 值集合和非 WWM 值集合。NonWWMRegMask 用于在 WWM 寄存器分配期间保留非 WWM 寄存器。一旦 WWM 寄存器分配完成，这个掩码就不再需要了。

## 4. 步骤之间的关系

这些步骤形成了一个严格的顺序执行流程：

```
[遍历指令] 
    ↓
[识别 WWM 溢出/恢复指令] → 如果不是，继续遍历
    ↓ 如果是
[提取并保留物理寄存器] → 更新 Changed 标志
    ↓
[全部识别完成后]
    ↓
[清除可重命名标志] → 防止保留寄存器被重命名
    ↓
[清理 NonWWMRegMask] → 为常规寄存器分配做准备
```

**依赖关系：**
- 步骤 2-3 在循环中重复执行，逐个处理每条指令
- 步骤 4-5 只在所有指令处理完后执行一次
- 步骤 4 依赖于步骤 3 收集的 WWM 保留寄存器列表
- 步骤 5 标志着从 WWM 寄存器分配阶段到常规寄存器分配阶段的转换

<a name="ref-block_6"></a>**与寄存器分配流水线的关系：** llvm-project:797-798[<sup>↗</sup>](#block_6) 

该 pass 的输出（`WWMReservedRegs`）会在 `SIRegisterInfo::getReservedRegs()` 中被使用，确保这些寄存器在后续的寄存器分配中被标记为保留状态。

## Notes

- 该 pass 支持新的 Pass Manager（`AMDGPUReserveWWMRegsPass`）和旧的 Pass Manager（`AMDGPUReserveWWMRegsLegacy`）两种形式
- WWM（Whole Wave Mode）是 AMDGPU 的一个特殊执行模式，在该模式下所有 wavefront 的 lane 都处于活动状态，无论 EXEC 掩码如何设置
- 这个 pass 是 WWM 寄存器分配流水线的最后一个步骤，确保 WWM 和非 WWM 代码之间的寄存器隔离
### Citations
<a name="block_0"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUReserveWWMRegs.cpp (L10-13) [<sup>↩</sup>](#ref-block_0)
```cpp
/// This pass should be invoked at the end of wwm-regalloc pipeline.
/// It identifies the WWM regs allocated during this pipeline and add
/// them to the list of reserved registers so that they won't be available for
/// regular VGPR allocation in the subsequent regalloc pipeline.
```
<a name="block_1"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUReserveWWMRegs.cpp (L80-95)
```cpp
  for (MachineBasicBlock &MBB : MF) {
    for (MachineInstr &MI : MBB) {
      unsigned Opc = MI.getOpcode();
      if (Opc != AMDGPU::SI_SPILL_S32_TO_VGPR &&
          Opc != AMDGPU::SI_RESTORE_S32_FROM_VGPR)
        continue;

      Register Reg = Opc == AMDGPU::SI_SPILL_S32_TO_VGPR
                         ? MI.getOperand(0).getReg()
                         : MI.getOperand(1).getReg();

      assert(Reg.isPhysical() &&
             "All WWM registers should have been allocated by now.");

      MFI->reserveWWMRegister(Reg);
      Changed |= true;
```
<a name="block_2"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUReserveWWMRegs.cpp (L99-106) [<sup>↩</sup>](#ref-block_2)
```cpp
  // The renamable flag can't be set for reserved registers. Reset the flag for
  // MOs involving wwm-regs as they will be reserved during vgpr-regalloc
  // pipeline.
  const MachineRegisterInfo &MRI = MF.getRegInfo();
  for (Register Reg : MFI->getWWMReservedRegs()) {
    for (MachineOperand &MO : MRI.reg_operands(Reg))
      MO.setIsRenamable(false);
  }
```
<a name="block_3"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUReserveWWMRegs.cpp (L108-109) [<sup>↩</sup>](#ref-block_3)
```cpp
  // Now clear the NonWWMRegMask earlier set during wwm-regalloc.
  MFI->clearNonWWMRegAllocMask();
```
<a name="block_4"></a>**File:** llvm/lib/Target/AMDGPU/SIMachineFunctionInfo.h (L556-559) [<sup>↩</sup>](#ref-block_4)
```text
  // Before allocation, the VGPR registers are partitioned into two distinct
  // sets, the first one for WWM-values and the second set for non-WWM values.
  // The latter set should be reserved during WWM-regalloc.
  BitVector NonWWMRegMask;
```
<a name="block_5"></a>**File:** llvm/lib/Target/AMDGPU/SIMachineFunctionInfo.h (L643-643) [<sup>↩</sup>](#ref-block_5)
```text
  void reserveWWMRegister(Register Reg) { WWMReservedRegs.insert(Reg); }
```
<a name="block_6"></a>**File:** llvm/lib/Target/AMDGPU/SIRegisterInfo.cpp (L797-798) [<sup>↩</sup>](#ref-block_6)
```cpp
  for (Register Reg : MFI->getWWMReservedRegs())
    reserveRegisterTuples(Reserved, Reg);
```