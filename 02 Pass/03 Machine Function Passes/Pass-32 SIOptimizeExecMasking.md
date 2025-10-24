# SIOptimizeExecMasking.cpp 代码功能详解

## 1. Pass 主要功能概述

<a name="ref-block_0"></a>`SIOptimizeExecMasking` 是一个针对 AMD GPU (AMDGPU) 的机器码优化 Pass，主要用于**优化 exec 寄存器掩码操作序列**，以减少指令数量和流水线停顿。 llvm-project:79-81[<sup>↗</sup>](#block_0) 

该 Pass 的核心作用是：
- **合并冗余的 exec 寄存器操作**：将多条指令序列优化为更高效的单条指令
- **减少流水线停顿**：特别是在 GFX10.3 及更高版本上，通过使用 `v_cmpx` 指令替代 `v_cmp + s_and_saveexec` 序列
<a name="ref-block_7"></a>- **保持控制流正确性**：在不改变程序语义的前提下进行优化 llvm-project:428-437[<sup>↗</sup>](#block_7) 

## 2. 主要实现步骤和子功能提取

该 Pass 包含以下核心子功能：

<a name="ref-block_8"></a>### 2.1 优化 Exec 序列 (`optimizeExecSequence`) llvm-project:438-438[<sup>↗</sup>](#block_8) 

### 2.2 记录和优化 VCmpx-SaveExec 序列
<a name="ref-block_20"></a>- `tryRecordVCmpxAndSaveexecSequence` llvm-project:659-660[<sup>↗</sup>](#block_20) 
<a name="ref-block_14"></a>- `optimizeVCMPSaveExecSequence` llvm-project:595-596[<sup>↗</sup>](#block_14) 

### 2.3 记录和优化 Or-SaveExec-Xor 序列
<a name="ref-block_26"></a>- `tryRecordOrSaveexecXorSequence` llvm-project:750-750[<sup>↗</sup>](#block_26) 
<a name="ref-block_30"></a>- `optimizeOrSaveexecXorSequences` llvm-project:784-784[<sup>↗</sup>](#block_30) 

### 2.4 辅助功能函数
<a name="ref-block_4"></a>- **伪终止符处理**：`fixTerminators`, `removeTerminatorBit` llvm-project:300-301[<sup>↗</sup>](#block_4) 
<a name="ref-block_1"></a>- **寄存器分析**：`isCopyFromExec`, `isCopyToExec` llvm-project:115-115[<sup>↗</sup>](#block_1) llvm-project:132-132 
- **活跃性分析**：`isRegisterInUseBetween`, `isRegisterInUseAfter` llvm-project:400-404 
- **反向搜索**：`findInstrBackwards`, `findExecCopy` llvm-project:352-356 

## 3. 各子功能的详细描述分析

### 3.1 optimizeExecSequence - Exec 序列优化

**功能**：优化控制流降低时生成的指令序列，将以下模式：
```
x = copy exec
z = s_<op>_b64 x, y
exec = copy z
```
优化为：
```
x = s_<op>_saveexec_b64 y
<a name="ref-block_7"></a>``` llvm-project:428-437[<sup>↗</sup>](#block_7) 

**实现细节**：
<a name="ref-block_9"></a>- 首先调用 `fixTerminators` 将伪终止符转换为普通指令 llvm-project:441-441[<sup>↗</sup>](#block_9) 
<a name="ref-block_10"></a>- 查找 `copy to exec` 指令 llvm-project:453-454[<sup>↗</sup>](#block_10) 
<a name="ref-block_11"></a>- 反向查找对应的 `copy from exec` 指令 llvm-project:464-464[<sup>↗</sup>](#block_11) 
<a name="ref-block_12"></a>- 检查寄存器活跃性，确保优化安全 llvm-project:485-489[<sup>↗</sup>](#block_12) 
<a name="ref-block_13"></a>- 查找可以转换为 saveexec 指令的逻辑操作 llvm-project:517-519[<sup>↗</sup>](#block_13) 

### 3.2 VCmpx-SaveExec 序列优化

**功能**：在 GFX10.3 及更高版本上，将以下模式：
```
v_cmp_* SGPR, IMM, VGPR
s_and_saveexec_b32 EXEC_SGPR_DEST, SGPR
```
优化为：
```
s_mov_b32 EXEC_SGPR_DEST, exec_lo
v_cmpx_* IMM, VGPR
<a name="ref-block_19"></a>``` llvm-project:652-658[<sup>↗</sup>](#block_19) 

**记录阶段** (`tryRecordVCmpxAndSaveexecSequence`)：
<a name="ref-block_21"></a>- 仅在 GFX10.3+ 硬件上启用 llvm-project:661-662[<sup>↗</sup>](#block_21) 
<a name="ref-block_22"></a>- 识别 `s_and_saveexec` 指令 llvm-project:664-668[<sup>↗</sup>](#block_22) 
<a name="ref-block_23"></a>- 反向查找匹配的 `v_cmp` 指令 llvm-project:687-693[<sup>↗</sup>](#block_23) 
<a name="ref-block_24"></a>- 进行多项安全性检查，包括操作数依赖性和活跃性分析 llvm-project:701-723[<sup>↗</sup>](#block_24) 

**优化阶段** (`optimizeVCMPSaveExecSequence`)：
<a name="ref-block_15"></a>- 获取 `v_cmpx` 对应的操作码 llvm-project:597-600[<sup>↗</sup>](#block_15) 
<a name="ref-block_16"></a>- 如果 saveexec 的结果被使用，插入 `s_mov` 指令保存 exec 值 llvm-project:608-614[<sup>↗</sup>](#block_16) 
<a name="ref-block_17"></a>- 构建新的 `v_cmpx` 指令，包含必要的修饰符 llvm-project:618-634[<sup>↗</sup>](#block_17) 
<a name="ref-block_18"></a>- 清理 kill 标志并删除原指令 llvm-project:637-647[<sup>↗</sup>](#block_18) 

### 3.3 Or-SaveExec-Xor 序列优化

**功能**：将以下模式：
```
s_or_saveexec s_o, s_i
s_xor exec, exec, s_o
```
优化为：
```
s_andn2_saveexec s_o, s_i
<a name="ref-block_25"></a>``` llvm-project:745-749[<sup>↗</sup>](#block_25) 

**记录阶段** (`tryRecordOrSaveexecXorSequence`)：
<a name="ref-block_27"></a>- 识别 `s_xor` 指令，检查其操作数是否包含 exec llvm-project:754-761[<sup>↗</sup>](#block_27) 
<a name="ref-block_28"></a>- 检查前一条指令是否为 `s_or_saveexec` llvm-project:768-770[<sup>↗</sup>](#block_28) 
<a name="ref-block_29"></a>- 验证操作数匹配关系 llvm-project:775-778[<sup>↗</sup>](#block_29) 

**优化阶段** (`optimizeOrSaveexecXorSequences`)：
<a name="ref-block_31"></a>- 遍历记录的指令对 llvm-project:793-796[<sup>↗</sup>](#block_31) 
<a name="ref-block_32"></a>- 构建 `s_andn2_saveexec` 指令替换原序列 llvm-project:797-802[<sup>↗</sup>](#block_32) 

### 3.4 辅助功能详解

<a name="ref-block_3"></a>**fixTerminators**：将基本块末尾的伪终止符转换为普通指令。这些指令最初被标记为终止符是为了在寄存器分配期间获得正确的溢出代码位置。 llvm-project:297-299[<sup>↗</sup>](#block_3) 

<a name="ref-block_5"></a>**findInstrBackwards**：从给定指令反向搜索满足条件的指令，同时确保指定寄存器未被修改。支持记录 kill 标志候选，用于后续清理。 llvm-project:347-356[<sup>↗</sup>](#block_5) 

<a name="ref-block_6"></a>**isRegisterInUseBetween**：通过反向活跃性分析判断寄存器在指定范围内是否仍在使用且未被重新定义。 llvm-project:394-404[<sup>↗</sup>](#block_6) 

## 4. 步骤和子功能之间的关系

### 执行流程

<a name="ref-block_33"></a>主函数 `run` 协调所有优化步骤： llvm-project:817-862[<sup>↗</sup>](#block_33) 

1. **初始化阶段**：设置目标信息（ST, TRI, TII, MRI）和 Exec 寄存器 llvm-project:818-823 

2. **第一轮优化**：执行 `optimizeExecSequence()`，优化基本的 exec 拷贝序列 llvm-project:825-825 

3. **模式识别阶段**：反向遍历所有基本块，在有限搜索窗口内记录优化候选 llvm-project:831-851 
   - 调用 `tryRecordOrSaveexecXorSequence` 记录 Or-Xor 模式
   - 调用 `tryRecordVCmpxAndSaveexecSequence` 记录 VCmp-SaveExec 模式
   - 遇到修改 exec 的指令时停止搜索

4. **第二轮优化**：应用记录的优化 llvm-project:853-859 
   - `optimizeOrSaveexecXorSequences()` 处理 Or-Xor 序列
   - 遍历 `SaveExecVCmpMapping` 处理 VCmp-SaveExec 序列

### 依赖关系

- **辅助函数为核心优化提供支持**：
  - `isCopyFromExec` 和 `isCopyToExec` 被 `optimizeExecSequence` 使用以识别拷贝指令
  - `findInstrBackwards` 被 `tryRecordVCmpxAndSaveexecSequence` 使用以定位匹配的 v_cmp 指令
  - `isRegisterInUseBetween` 确保优化不会破坏数据依赖关系

- **两阶段设计（记录-应用）**：分离模式识别和优化应用，确保在修改代码前完成所有分析，避免迭代器失效问题

- **保守的安全检查**：所有优化都包含严格的前置条件检查，如活跃性分析、寄存器依赖检查和 live-out 检测，确保优化的正确性

## Notes

该 Pass 是 AMDGPU 后端的关键优化组件，专注于减少控制流操作的开销。优化主要发生在**寄存器分配之后**，因此需要特别注意处理 kill 标志和寄存器依赖关系。Pass 的设计采用了保守策略，只在确保安全的情况下才进行优化，这对于维护程序正确性至关重要。

<a name="ref-block_19"></a>对于 GFX10.3+ 架构，VCmpx 优化特别重要，因为它可以显著减少流水线停顿。 llvm-project:652-658[<sup>↗</sup>](#block_19)
### Citations
<a name="block_0"></a>**File:** llvm/lib/Target/AMDGPU/SIOptimizeExecMasking.cpp (L79-81) [<sup>↩</sup>](#ref-block_0)
```cpp
  StringRef getPassName() const override {
    return "SI optimize exec mask operations";
  }
```
<a name="block_1"></a>**File:** llvm/lib/Target/AMDGPU/SIOptimizeExecMasking.cpp (L115-115) [<sup>↩</sup>](#ref-block_1)
```cpp
Register SIOptimizeExecMasking::isCopyFromExec(const MachineInstr &MI) const {
```
<a name="block_2"></a>**File:** llvm/lib/Target/AMDGPU/SIOptimizeExecMasking.cpp (L132-132)
```cpp
Register SIOptimizeExecMasking::isCopyToExec(const MachineInstr &MI) const {
```
<a name="block_3"></a>**File:** llvm/lib/Target/AMDGPU/SIOptimizeExecMasking.cpp (L297-299) [<sup>↩</sup>](#ref-block_3)
```cpp
// Turn all pseudoterminators in the block into their equivalent non-terminator
// instructions. Returns the reverse iterator to the first non-terminator
// instruction in the block.
```
<a name="block_4"></a>**File:** llvm/lib/Target/AMDGPU/SIOptimizeExecMasking.cpp (L300-301) [<sup>↩</sup>](#ref-block_4)
```cpp
MachineBasicBlock::reverse_iterator
SIOptimizeExecMasking::fixTerminators(MachineBasicBlock &MBB) const {
```
<a name="block_5"></a>**File:** llvm/lib/Target/AMDGPU/SIOptimizeExecMasking.cpp (L347-356) [<sup>↩</sup>](#ref-block_5)
```cpp
// Backwards-iterate from Origin (for n=MaxInstructions iterations) until either
// the beginning of the BB is reached or Pred evaluates to true - which can be
// an arbitrary condition based on the current MachineInstr, for instance an
// target instruction. Breaks prematurely by returning nullptr if one of the
// registers given in NonModifiableRegs is modified by the current instruction.
MachineInstr *SIOptimizeExecMasking::findInstrBackwards(
    MachineInstr &Origin, std::function<bool(MachineInstr *)> Pred,
    ArrayRef<MCRegister> NonModifiableRegs, MachineInstr *Terminator,
    SmallVectorImpl<MachineOperand *> *KillFlagCandidates,
    unsigned MaxInstructions) const {
```
<a name="block_6"></a>**File:** llvm/lib/Target/AMDGPU/SIOptimizeExecMasking.cpp (L394-404) [<sup>↩</sup>](#ref-block_6)
```cpp
// Determine if a register Reg is not re-defined and still in use
// in the range (Stop..Start].
// It does so by backwards calculating liveness from the end of the BB until
// either Stop or the beginning of the BB is reached.
// After liveness is calculated, we can determine if Reg is still in use and not
// defined inbetween the instructions.
bool SIOptimizeExecMasking::isRegisterInUseBetween(MachineInstr &Stop,
                                                   MachineInstr &Start,
                                                   MCRegister Reg,
                                                   bool UseLiveOuts,
                                                   bool IgnoreStart) const {
```
<a name="block_7"></a>**File:** llvm/lib/Target/AMDGPU/SIOptimizeExecMasking.cpp (L428-437) [<sup>↩</sup>](#ref-block_7) [<sup>↩</sup>](#ref-block_7)
```cpp
// Optimize sequences emitted for control flow lowering. They are originally
// emitted as the separate operations because spill code may need to be
// inserted for the saved copy of exec.
//
//     x = copy exec
//     z = s_<op>_b64 x, y
//     exec = copy z
// =>
//     x = s_<op>_saveexec_b64 y
//
```
<a name="block_8"></a>**File:** llvm/lib/Target/AMDGPU/SIOptimizeExecMasking.cpp (L438-438) [<sup>↩</sup>](#ref-block_8)
```cpp
bool SIOptimizeExecMasking::optimizeExecSequence() {
```
<a name="block_9"></a>**File:** llvm/lib/Target/AMDGPU/SIOptimizeExecMasking.cpp (L441-441) [<sup>↩</sup>](#ref-block_9)
```cpp
    MachineBasicBlock::reverse_iterator I = fixTerminators(MBB);
```
<a name="block_10"></a>**File:** llvm/lib/Target/AMDGPU/SIOptimizeExecMasking.cpp (L453-454) [<sup>↩</sup>](#ref-block_10)
```cpp
      CopyToExec = isCopyToExec(*I);
      if (CopyToExec)
```
<a name="block_11"></a>**File:** llvm/lib/Target/AMDGPU/SIOptimizeExecMasking.cpp (L464-464) [<sup>↩</sup>](#ref-block_11)
```cpp
    auto CopyFromExecInst = findExecCopy(MBB, I);
```
<a name="block_12"></a>**File:** llvm/lib/Target/AMDGPU/SIOptimizeExecMasking.cpp (L485-489) [<sup>↩</sup>](#ref-block_12)
```cpp
    if (isLiveOut(MBB, CopyToExec)) {
      // The copied register is live out and has a second use in another block.
      LLVM_DEBUG(dbgs() << "Exec copy source register is live out\n");
      continue;
    }
```
<a name="block_13"></a>**File:** llvm/lib/Target/AMDGPU/SIOptimizeExecMasking.cpp (L517-519) [<sup>↩</sup>](#ref-block_13)
```cpp
        unsigned SaveExecOp = getSaveExecOp(J->getOpcode());
        if (SaveExecOp == AMDGPU::INSTRUCTION_LIST_END)
          break;
```
<a name="block_14"></a>**File:** llvm/lib/Target/AMDGPU/SIOptimizeExecMasking.cpp (L595-596) [<sup>↩</sup>](#ref-block_14)
```cpp
bool SIOptimizeExecMasking::optimizeVCMPSaveExecSequence(
    MachineInstr &SaveExecInstr, MachineInstr &VCmp, MCRegister Exec) const {
```
<a name="block_15"></a>**File:** llvm/lib/Target/AMDGPU/SIOptimizeExecMasking.cpp (L597-600) [<sup>↩</sup>](#ref-block_15)
```cpp
  const int NewOpcode = AMDGPU::getVCMPXOpFromVCMP(VCmp.getOpcode());

  if (NewOpcode == -1)
    return false;
```
<a name="block_16"></a>**File:** llvm/lib/Target/AMDGPU/SIOptimizeExecMasking.cpp (L608-614) [<sup>↩</sup>](#ref-block_16)
```cpp
  if (!SaveExecInstr.uses().empty()) {
    bool IsSGPR32 = TRI->getRegSizeInBits(MoveDest, *MRI) == 32;
    unsigned MovOpcode = IsSGPR32 ? AMDGPU::S_MOV_B32 : AMDGPU::S_MOV_B64;
    BuildMI(*SaveExecInstr.getParent(), InsertPosIt,
            SaveExecInstr.getDebugLoc(), TII->get(MovOpcode), MoveDest)
        .addReg(Exec);
  }
```
<a name="block_17"></a>**File:** llvm/lib/Target/AMDGPU/SIOptimizeExecMasking.cpp (L618-634) [<sup>↩</sup>](#ref-block_17)
```cpp
  auto Builder = BuildMI(*VCmp.getParent(), std::next(InsertPosIt),
                         VCmp.getDebugLoc(), TII->get(NewOpcode));

  auto TryAddImmediateValueFromNamedOperand =
      [&](AMDGPU::OpName OperandName) -> void {
    if (auto *Mod = TII->getNamedOperand(VCmp, OperandName))
      Builder.addImm(Mod->getImm());
  };

  TryAddImmediateValueFromNamedOperand(AMDGPU::OpName::src0_modifiers);
  Builder.add(*Src0);

  TryAddImmediateValueFromNamedOperand(AMDGPU::OpName::src1_modifiers);
  Builder.add(*Src1);

  TryAddImmediateValueFromNamedOperand(AMDGPU::OpName::clamp);

```
<a name="block_18"></a>**File:** llvm/lib/Target/AMDGPU/SIOptimizeExecMasking.cpp (L637-647) [<sup>↩</sup>](#ref-block_18)
```cpp
  // The kill flags may no longer be correct.
  if (Src0->isReg())
    MRI->clearKillFlags(Src0->getReg());
  if (Src1->isReg())
    MRI->clearKillFlags(Src1->getReg());

  for (MachineOperand *MO : KillFlagCandidates)
    MO->setIsKill(false);

  SaveExecInstr.eraseFromParent();
  VCmp.eraseFromParent();
```
<a name="block_19"></a>**File:** llvm/lib/Target/AMDGPU/SIOptimizeExecMasking.cpp (L652-658) [<sup>↩</sup>](#ref-block_19) [<sup>↩</sup>](#ref-block_19)
```cpp
// Record (on GFX10.3 and later) occurences of
// v_cmp_* SGPR, IMM, VGPR
// s_and_saveexec_b32 EXEC_SGPR_DEST, SGPR
// to be replaced with
// s_mov_b32 EXEC_SGPR_DEST, exec_lo
// v_cmpx_* IMM, VGPR
// to reduce pipeline stalls.
```
<a name="block_20"></a>**File:** llvm/lib/Target/AMDGPU/SIOptimizeExecMasking.cpp (L659-660) [<sup>↩</sup>](#ref-block_20)
```cpp
void SIOptimizeExecMasking::tryRecordVCmpxAndSaveexecSequence(
    MachineInstr &MI) {
```
<a name="block_21"></a>**File:** llvm/lib/Target/AMDGPU/SIOptimizeExecMasking.cpp (L661-662) [<sup>↩</sup>](#ref-block_21)
```cpp
  if (!ST->hasGFX10_3Insts())
    return;
```
<a name="block_22"></a>**File:** llvm/lib/Target/AMDGPU/SIOptimizeExecMasking.cpp (L664-668) [<sup>↩</sup>](#ref-block_22)
```cpp
  const unsigned AndSaveExecOpcode =
      ST->isWave32() ? AMDGPU::S_AND_SAVEEXEC_B32 : AMDGPU::S_AND_SAVEEXEC_B64;

  if (MI.getOpcode() != AndSaveExecOpcode)
    return;
```
<a name="block_23"></a>**File:** llvm/lib/Target/AMDGPU/SIOptimizeExecMasking.cpp (L687-693) [<sup>↩</sup>](#ref-block_23)
```cpp
  VCmp = findInstrBackwards(
      MI,
      [&](MachineInstr *Check) {
        return AMDGPU::getVCMPXOpFromVCMP(Check->getOpcode()) != -1 &&
               Check->modifiesRegister(SaveExecSrc0->getReg(), TRI);
      },
      {Exec, SaveExecSrc0->getReg()});
```
<a name="block_24"></a>**File:** llvm/lib/Target/AMDGPU/SIOptimizeExecMasking.cpp (L701-723) [<sup>↩</sup>](#ref-block_24)
```cpp
  // Check if any of the v_cmp source operands is written by the saveexec.
  MachineOperand *Src0 = TII->getNamedOperand(*VCmp, AMDGPU::OpName::src0);
  if (Src0->isReg() && TRI->isSGPRReg(*MRI, Src0->getReg()) &&
      MI.modifiesRegister(Src0->getReg(), TRI))
    return;

  MachineOperand *Src1 = TII->getNamedOperand(*VCmp, AMDGPU::OpName::src1);
  if (Src1->isReg() && TRI->isSGPRReg(*MRI, Src1->getReg()) &&
      MI.modifiesRegister(Src1->getReg(), TRI))
    return;

  // Don't do the transformation if the destination operand is included in
  // it's MBB Live-outs, meaning it's used in any of its successors, leading
  // to incorrect code if the v_cmp and therefore the def of
  // the dest operand is removed.
  if (isLiveOut(*VCmp->getParent(), VCmpDest->getReg()))
    return;

  // If the v_cmp target is in use between v_cmp and s_and_saveexec or after the
  // s_and_saveexec, skip the optimization.
  if (isRegisterInUseBetween(*VCmp, MI, VCmpDest->getReg(), false, true) ||
      isRegisterInUseAfter(MI, VCmpDest->getReg()))
    return;
```
<a name="block_25"></a>**File:** llvm/lib/Target/AMDGPU/SIOptimizeExecMasking.cpp (L745-749) [<sup>↩</sup>](#ref-block_25)
```cpp
// Record occurences of
// s_or_saveexec s_o, s_i
// s_xor exec, exec, s_o
// to be replaced with
// s_andn2_saveexec s_o, s_i.
```
<a name="block_26"></a>**File:** llvm/lib/Target/AMDGPU/SIOptimizeExecMasking.cpp (L750-750) [<sup>↩</sup>](#ref-block_26)
```cpp
void SIOptimizeExecMasking::tryRecordOrSaveexecXorSequence(MachineInstr &MI) {
```
<a name="block_27"></a>**File:** llvm/lib/Target/AMDGPU/SIOptimizeExecMasking.cpp (L754-761) [<sup>↩</sup>](#ref-block_27)
```cpp
  if (MI.getOpcode() == XorOpcode && &MI != &MI.getParent()->front()) {
    const MachineOperand &XorDst = MI.getOperand(0);
    const MachineOperand &XorSrc0 = MI.getOperand(1);
    const MachineOperand &XorSrc1 = MI.getOperand(2);

    if (XorDst.isReg() && XorDst.getReg() == Exec && XorSrc0.isReg() &&
        XorSrc1.isReg() &&
        (XorSrc0.getReg() == Exec || XorSrc1.getReg() == Exec)) {
```
<a name="block_28"></a>**File:** llvm/lib/Target/AMDGPU/SIOptimizeExecMasking.cpp (L768-770) [<sup>↩</sup>](#ref-block_28)
```cpp
      MachineInstr &PossibleOrSaveexec = *MI.getPrevNode();
      if (PossibleOrSaveexec.getOpcode() != OrSaveexecOpcode)
        return;
```
<a name="block_29"></a>**File:** llvm/lib/Target/AMDGPU/SIOptimizeExecMasking.cpp (L775-778) [<sup>↩</sup>](#ref-block_29)
```cpp
        if ((XorSrc0.getReg() == Exec && XorSrc1.getReg() == OrDst.getReg()) ||
            (XorSrc0.getReg() == OrDst.getReg() && XorSrc1.getReg() == Exec)) {
          OrXors.emplace_back(&PossibleOrSaveexec, &MI);
        }
```
<a name="block_30"></a>**File:** llvm/lib/Target/AMDGPU/SIOptimizeExecMasking.cpp (L784-784) [<sup>↩</sup>](#ref-block_30)
```cpp
bool SIOptimizeExecMasking::optimizeOrSaveexecXorSequences() {
```
<a name="block_31"></a>**File:** llvm/lib/Target/AMDGPU/SIOptimizeExecMasking.cpp (L793-796) [<sup>↩</sup>](#ref-block_31)
```cpp
  for (const auto &Pair : OrXors) {
    MachineInstr *Or = nullptr;
    MachineInstr *Xor = nullptr;
    std::tie(Or, Xor) = Pair;
```
<a name="block_32"></a>**File:** llvm/lib/Target/AMDGPU/SIOptimizeExecMasking.cpp (L797-802) [<sup>↩</sup>](#ref-block_32)
```cpp
    BuildMI(*Or->getParent(), Or->getIterator(), Or->getDebugLoc(),
            TII->get(Andn2Opcode), Or->getOperand(0).getReg())
        .addReg(Or->getOperand(1).getReg());

    Or->eraseFromParent();
    Xor->eraseFromParent();
```
<a name="block_33"></a>**File:** llvm/lib/Target/AMDGPU/SIOptimizeExecMasking.cpp (L817-862) [<sup>↩</sup>](#ref-block_33)
```cpp
bool SIOptimizeExecMasking::run(MachineFunction &MF) {
  this->MF = &MF;
  ST = &MF.getSubtarget<GCNSubtarget>();
  TRI = ST->getRegisterInfo();
  TII = ST->getInstrInfo();
  MRI = &MF.getRegInfo();
  Exec = TRI->getExec();

  bool Changed = optimizeExecSequence();

  OrXors.clear();
  SaveExecVCmpMapping.clear();
  KillFlagCandidates.clear();
  static unsigned SearchWindow = 10;
  for (MachineBasicBlock &MBB : MF) {
    unsigned SearchCount = 0;

    for (auto &MI : llvm::reverse(MBB)) {
      if (MI.isDebugInstr())
        continue;

      if (SearchCount >= SearchWindow) {
        break;
      }

      tryRecordOrSaveexecXorSequence(MI);
      tryRecordVCmpxAndSaveexecSequence(MI);

      if (MI.modifiesRegister(Exec, TRI)) {
        break;
      }

      ++SearchCount;
    }
  }

  Changed |= optimizeOrSaveexecXorSequences();
  for (const auto &Entry : SaveExecVCmpMapping) {
    MachineInstr *SaveExecInstr = Entry.getFirst();
    MachineInstr *VCmpInstr = Entry.getSecond();

    Changed |= optimizeVCMPSaveExecSequence(*SaveExecInstr, *VCmpInstr, Exec);
  }

  return Changed;
}
```