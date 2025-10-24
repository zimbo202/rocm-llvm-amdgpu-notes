# SIOptimizeExecMaskingPreRA.cpp 代码功能详细分析

## 1. Pass的主要功能概括

<a name="ref-block_0"></a>**`SIOptimizeExecMaskingPreRA`** 是一个在寄存器分配前执行的机器码优化pass，专门针对AMDGPU架构的exec mask操作进行窥孔优化。 llvm-project:8-12[<sup>↗</sup>](#block_0) 

**主要作用：**
- 优化exec mask相关的指令序列模式
- 消除冗余的条件掩码计算和分支准备代码
- 简化控制流lowering过程中引入的保守性代码

**主要效果：**
- **降低寄存器压力**：通过消除中间临时寄存器的使用
- **减少指令数量**：将多条指令序列简化为更少的指令
- **提高代码质量**：移除死代码和无副作用的指令

## 2. 主要功能步骤和子功能

<a name="ref-block_16"></a>该pass的主要执行流程在`run`函数中，包含四个核心优化步骤： llvm-project:368-511[<sup>↗</sup>](#block_16) 

### 核心子功能列表：

1. **`optimizeVcndVcmpPair`** - V_CNDMASK/V_CMP/S_AND序列优化
2. **`optimizeElseBranch`** - Else分支的exec mask优化
3. **s_endpgm前的死代码消除** - 移除程序退出前的无效指令
4. **冗余exec拷贝折叠** - 消除不必要的exec寄存器拷贝

## 3. 各步骤/子功能的详细分析

### 子功能1: `optimizeVcndVcmpPair` - 条件掩码序列优化

**功能描述：**

<a name="ref-block_1"></a>此函数优化由DAGCombiner在处理分支条件时产生的否定模式。它将一个4指令序列简化为2指令序列： llvm-project:115-128[<sup>↗</sup>](#block_1) 

**优化模式：**
```
优化前：
  %sel = V_CNDMASK_B32_e64 0, 1, %cc      // 条件选择0或1
  %cmp = V_CMP_NE_U32 1, %sel              // 比较是否不等于1
  $vcc = S_AND_B64 $exec, %cmp             // 与exec掩码做AND
  S_CBRANCH_VCC[N]Z                        // 基于VCC分支

优化后：
  $vcc = S_ANDN2_B64 $exec, %cc            // 直接用ANDN2计算
  S_CBRANCH_VCC[N]Z                        // 基于VCC分支
```

**实现细节：**

<a name="ref-block_2"></a>1. **模式匹配**：从终结指令`S_CBRANCH_VCCZ/VCCNZ`开始反向查找 llvm-project:132-137[<sup>↗</sup>](#block_2) 

<a name="ref-block_3"></a>2. **查找S_AND指令**：使用`findReachingDef`找到定义VCC的AND指令，并验证它与exec进行AND操作 llvm-project:139-154[<sup>↗</sup>](#block_3) 

<a name="ref-block_4"></a>3. **查找V_CMP指令**：继续反向查找CMP指令，验证它比较值是否为1 llvm-project:156-167[<sup>↗</sup>](#block_4) 

<a name="ref-block_5"></a>4. **查找V_CNDMASK指令**：最后找到CNDMASK指令，验证它选择0和1两个立即数 llvm-project:173-186[<sup>↗</sup>](#block_5) 

<a name="ref-block_6"></a>5. **活跃性检查**：确保条件寄存器在select和and之间没有被重新定义 llvm-project:190-203[<sup>↗</sup>](#block_6) 

6. **指令替换和清理**：
<a name="ref-block_7"></a>   - 用`S_ANDN2`替换`S_AND` llvm-project:209-221[<sup>↗</sup>](#block_7) 
<a name="ref-block_8"></a>   - 尝试移除V_CMP指令 llvm-project:239-251[<sup>↗</sup>](#block_8) 
<a name="ref-block_9"></a>   - 尝试移除V_CNDMASK指令 llvm-project:253-270[<sup>↗</sup>](#block_9) 

### 子功能2: `optimizeElseBranch` - Else分支exec优化

**功能描述：**

<a name="ref-block_10"></a>此函数清理控制流lowering过程中为安全性添加的潜在不必要代码。它优化else块中的exec mask恢复序列： llvm-project:276-288[<sup>↗</sup>](#block_10) 

**优化模式：**
```
优化前：
  %dst = S_OR_SAVEEXEC %src              // 保存exec并更新
  ... 不修改exec的指令 ...
  %tmp = S_AND $exec, %dst               // 中间AND操作
  $exec = S_XOR_term $exec, %tmp         // 用tmp恢复exec

优化后：
  %dst = S_OR_SAVEEXEC %src              // 保存exec并更新
  ... 不修改exec的指令 ...
  $exec = S_XOR_term $exec, %dst         // 直接用dst恢复exec
```

**实现细节：**

<a name="ref-block_11"></a>1. **识别else块**：检查基本块是否以`S_OR_SAVEEXEC`开始 llvm-project:294-298[<sup>↗</sup>](#block_11) 

<a name="ref-block_12"></a>2. **查找XOR_term**：在终结指令中找到`S_XOR_term` llvm-project:300-308[<sup>↗</sup>](#block_12) 

<a name="ref-block_13"></a>3. **查找中间AND**：反向搜索找到不必要的`S_AND`指令 llvm-project:313-323[<sup>↗</sup>](#block_13) 

<a name="ref-block_14"></a>4. **验证exec未被修改**：检查在SaveExec和And之间exec没有被修改 llvm-project:325-335[<sup>↗</sup>](#block_14) 

<a name="ref-block_15"></a>5. **移除冗余AND**：删除AND指令，直接使用SaveExec的结果 llvm-project:337-347[<sup>↗</sup>](#block_15) 

### 子功能3: s_endpgm前的死代码消除

**功能描述：**

对于没有后继块（程序退出块）的基本块，在`S_ENDPGM`指令前移除所有无副作用的指令。 llvm-project:400-411 

**实现细节：**

1. **识别退出块**：检查基本块是否没有后继且以`S_ENDPGM`结束 llvm-project:401-411 

2. **反向遍历指令**：从endpgm向前扫描指令 llvm-project:415-423 

3. **检查副作用**：跳过有存储、屏障、调用、副作用或有序内存引用的指令 llvm-project:430-433 

4. **移除无效指令**：删除没有副作用的指令并记录相关寄存器 llvm-project:435-448 

5. **前驱块扩展**：对于只有一个后继的前驱块，继续向上清理 llvm-project:454-459 

### 子功能4: 冗余exec拷贝折叠

**功能描述：**

如果逻辑操作的唯一使用者是移动到exec，则立即折叠它以防止形成saveexec模式。 llvm-project:463-469 

**优化模式：**
```
优化前：
  %0 = COPY $exec
  %1 = S_AND_B64 %0, %2

优化后：
  %1 = S_AND_B64 $exec, %2
```

**实现细节：**

1. **查找exec拷贝**：反向扫描找到完整的exec拷贝指令 llvm-project:471-475 

2. **检查单一使用**：验证保存的exec寄存器只有一个非调试使用 llvm-project:478-481 

3. **验证合法性**：检查使用者在同一基本块且操作数可以合法使用exec llvm-project:483-485 

4. **替换和清理**：移除拷贝指令，直接使用exec寄存器 llvm-project:486-492 

## 4. 步骤/子功能之间的关系

### 执行顺序和依赖关系：

```mermaid
graph TD
    A["run函数入口"] --> B["初始化目标相关信息"]
    B --> C["遍历每个基本块"]
    C --> D["优化1: optimizeElseBranch"]
    C --> E["优化2: optimizeVcndVcmpPair"]
    C --> F["优化3: s_endpgm前死代码消除"]
    C --> G["优化4: 冗余exec拷贝折叠"]
    D --> H["记录需要重新计算的寄存器"]
    E --> H
    F --> H
    G --> H
    H --> I["重新计算活跃区间"]
    I --> J["返回是否有修改"]
``` llvm-project:386-396 

### 功能间的协同关系：

1. **独立性**：四个优化步骤相互独立，可以在同一基本块中同时应用 llvm-project:386-398 

2. **寄存器活跃性维护**：所有优化都使用`LiveIntervals`分析来确保转换的正确性，并在修改后更新活跃区间 llvm-project:498-508 

3. **统一的寄存器重计算**：使用`RecalcRegs`集合统一收集所有需要重新计算活跃区间的寄存器 llvm-project:383-390 

4. **Wave32/Wave64适配**：所有优化都根据GPU的wave大小选择正确的指令操作码 llvm-project:374-381 

### 优化目标的协同：

- **优化1和2**主要针对控制流相关的exec mask操作
- **优化3**专注于程序退出路径的清理
- **优化4**防止后续产生不必要的saveexec模式

所有优化共同作用，确保在寄存器分配前生成更紧凑、更高效的代码，减轻寄存器分配器的压力。

## Notes

这个pass是AMDGPU后端优化流水线的重要组成部分，它利用了以下关键特性：

1. **在寄存器分配前执行**：这使得优化可以更自由地消除临时寄存器，直接降低寄存器压力
2. **使用LiveIntervals分析**：确保所有转换都保持程序语义的正确性
3. **针对特定模式**：识别由前端lowering产生的特定指令序列模式
4. **渐进式优化**：通过反向搜索和活跃性检查，只在安全时进行优化

该pass的设计体现了LLVM优化框架的典型模式：在机器码层面进行目标特定的窥孔优化，利用精确的数据流分析确保正确性。
### Citations
<a name="block_0"></a>**File:** llvm/lib/Target/AMDGPU/SIOptimizeExecMaskingPreRA.cpp (L8-12) [<sup>↩</sup>](#ref-block_0)
```cpp
//
/// \file
/// This pass performs exec mask handling peephole optimizations which needs
/// to be done before register allocation to reduce register pressure.
///
```
<a name="block_1"></a>**File:** llvm/lib/Target/AMDGPU/SIOptimizeExecMaskingPreRA.cpp (L115-128) [<sup>↩</sup>](#ref-block_1)
```cpp
// Optimize sequence
//    %sel = V_CNDMASK_B32_e64 0, 1, %cc
//    %cmp = V_CMP_NE_U32 1, %sel
//    $vcc = S_AND_B64 $exec, %cmp
//    S_CBRANCH_VCC[N]Z
// =>
//    $vcc = S_ANDN2_B64 $exec, %cc
//    S_CBRANCH_VCC[N]Z
//
// It is the negation pattern inserted by DAGCombiner::visitBRCOND() in the
// rebuildSetCC(). We start with S_CBRANCH to avoid exhaustive search, but
// only 3 first instructions are really needed. S_AND_B64 with exec is a
// required part of the pattern since V_CNDMASK_B32 writes zeroes for inactive
// lanes.
```
<a name="block_2"></a>**File:** llvm/lib/Target/AMDGPU/SIOptimizeExecMaskingPreRA.cpp (L132-137) [<sup>↩</sup>](#ref-block_2)
```cpp
  auto I = llvm::find_if(MBB.terminators(), [](const MachineInstr &MI) {
                           unsigned Opc = MI.getOpcode();
                           return Opc == AMDGPU::S_CBRANCH_VCCZ ||
                                  Opc == AMDGPU::S_CBRANCH_VCCNZ; });
  if (I == MBB.terminators().end())
    return false;
```
<a name="block_3"></a>**File:** llvm/lib/Target/AMDGPU/SIOptimizeExecMaskingPreRA.cpp (L139-154) [<sup>↩</sup>](#ref-block_3)
```cpp
  auto *And =
      TRI->findReachingDef(CondReg, AMDGPU::NoSubRegister, *I, *MRI, LIS);
  if (!And || And->getOpcode() != AndOpc ||
      !And->getOperand(1).isReg() || !And->getOperand(2).isReg())
    return false;

  MachineOperand *AndCC = &And->getOperand(1);
  Register CmpReg = AndCC->getReg();
  unsigned CmpSubReg = AndCC->getSubReg();
  if (CmpReg == Register(ExecReg)) {
    AndCC = &And->getOperand(2);
    CmpReg = AndCC->getReg();
    CmpSubReg = AndCC->getSubReg();
  } else if (And->getOperand(2).getReg() != Register(ExecReg)) {
    return false;
  }
```
<a name="block_4"></a>**File:** llvm/lib/Target/AMDGPU/SIOptimizeExecMaskingPreRA.cpp (L156-167) [<sup>↩</sup>](#ref-block_4)
```cpp
  auto *Cmp = TRI->findReachingDef(CmpReg, CmpSubReg, *And, *MRI, LIS);
  if (!Cmp || !(Cmp->getOpcode() == AMDGPU::V_CMP_NE_U32_e32 ||
                Cmp->getOpcode() == AMDGPU::V_CMP_NE_U32_e64) ||
      Cmp->getParent() != And->getParent())
    return false;

  MachineOperand *Op1 = TII->getNamedOperand(*Cmp, AMDGPU::OpName::src0);
  MachineOperand *Op2 = TII->getNamedOperand(*Cmp, AMDGPU::OpName::src1);
  if (Op1->isImm() && Op2->isReg())
    std::swap(Op1, Op2);
  if (!Op1->isReg() || !Op2->isImm() || Op2->getImm() != 1)
    return false;
```
<a name="block_5"></a>**File:** llvm/lib/Target/AMDGPU/SIOptimizeExecMaskingPreRA.cpp (L173-186) [<sup>↩</sup>](#ref-block_5)
```cpp
  auto *Sel = TRI->findReachingDef(SelReg, Op1->getSubReg(), *Cmp, *MRI, LIS);
  if (!Sel || Sel->getOpcode() != AMDGPU::V_CNDMASK_B32_e64)
    return false;

  if (TII->hasModifiersSet(*Sel, AMDGPU::OpName::src0_modifiers) ||
      TII->hasModifiersSet(*Sel, AMDGPU::OpName::src1_modifiers))
    return false;

  Op1 = TII->getNamedOperand(*Sel, AMDGPU::OpName::src0);
  Op2 = TII->getNamedOperand(*Sel, AMDGPU::OpName::src1);
  MachineOperand *CC = TII->getNamedOperand(*Sel, AMDGPU::OpName::src2);
  if (!Op1->isImm() || !Op2->isImm() || !CC->isReg() ||
      Op1->getImm() != 0 || Op2->getImm() != 1)
    return false;
```
<a name="block_6"></a>**File:** llvm/lib/Target/AMDGPU/SIOptimizeExecMaskingPreRA.cpp (L190-203) [<sup>↩</sup>](#ref-block_6)
```cpp
  // If there was a def between the select and the and, we would need to move it
  // to fold this.
  if (isDefBetween(*TRI, LIS, CCReg, *Sel, *And))
    return false;

  // Cannot safely mirror live intervals with PHI nodes, so check for these
  // before optimization.
  SlotIndex SelIdx = LIS->getInstructionIndex(*Sel);
  LiveInterval *SelLI = &LIS->getInterval(SelReg);
  if (llvm::any_of(SelLI->vnis(),
                    [](const VNInfo *VNI) {
                      return VNI->isPHIDef();
                    }))
    return false;
```
<a name="block_7"></a>**File:** llvm/lib/Target/AMDGPU/SIOptimizeExecMaskingPreRA.cpp (L209-221) [<sup>↩</sup>](#ref-block_7)
```cpp
  MachineInstr *Andn2 =
      BuildMI(MBB, *And, And->getDebugLoc(), TII->get(Andn2Opc),
              And->getOperand(0).getReg())
          .addReg(ExecReg)
          .addReg(CCReg, getUndefRegState(CC->isUndef()), CC->getSubReg());
  MachineOperand &AndSCC = And->getOperand(3);
  assert(AndSCC.getReg() == AMDGPU::SCC);
  MachineOperand &Andn2SCC = Andn2->getOperand(3);
  assert(Andn2SCC.getReg() == AMDGPU::SCC);
  Andn2SCC.setIsDead(AndSCC.isDead());

  SlotIndex AndIdx = LIS->ReplaceMachineInstrInMaps(*And, *Andn2);
  And->eraseFromParent();
```
<a name="block_8"></a>**File:** llvm/lib/Target/AMDGPU/SIOptimizeExecMaskingPreRA.cpp (L239-251) [<sup>↩</sup>](#ref-block_8)
```cpp
  // and s_and_b64 if VCC or just unused if any other register.
  LiveInterval *CmpLI = CmpReg.isVirtual() ? &LIS->getInterval(CmpReg) : nullptr;
  if ((CmpLI && CmpLI->Query(AndIdx.getRegSlot()).isKill()) ||
      (CmpReg == Register(CondReg) &&
       std::none_of(std::next(Cmp->getIterator()), Andn2->getIterator(),
                    [&](const MachineInstr &MI) {
                      return MI.readsRegister(CondReg, TRI);
                    }))) {
    LLVM_DEBUG(dbgs() << "Erasing: " << *Cmp << '\n');
    if (CmpLI)
      LIS->removeVRegDefAt(*CmpLI, CmpIdx.getRegSlot());
    LIS->RemoveMachineInstrFromMaps(*Cmp);
    Cmp->eraseFromParent();
```
<a name="block_9"></a>**File:** llvm/lib/Target/AMDGPU/SIOptimizeExecMaskingPreRA.cpp (L253-270) [<sup>↩</sup>](#ref-block_9)
```cpp
    // Try to remove v_cndmask_b32.
    // Kill status must be checked before shrinking the live range.
    bool IsKill = SelLI->Query(CmpIdx.getRegSlot()).isKill();
    LIS->shrinkToUses(SelLI);
    bool IsDead = SelLI->Query(SelIdx.getRegSlot()).isDeadDef();
    if (MRI->use_nodbg_empty(SelReg) && (IsKill || IsDead)) {
      LLVM_DEBUG(dbgs() << "Erasing: " << *Sel << '\n');

      LIS->removeVRegDefAt(*SelLI, SelIdx.getRegSlot());
      LIS->RemoveMachineInstrFromMaps(*Sel);
      bool ShrinkSel = Sel->getOperand(0).readsReg();
      Sel->eraseFromParent();
      if (ShrinkSel) {
        // The result of the V_CNDMASK was a subreg def which counted as a read
        // from the other parts of the reg. Shrink their live ranges.
        LIS->shrinkToUses(SelLI);
      }
    }
```
<a name="block_10"></a>**File:** llvm/lib/Target/AMDGPU/SIOptimizeExecMaskingPreRA.cpp (L276-288) [<sup>↩</sup>](#ref-block_10)
```cpp
// Optimize sequence
//    %dst = S_OR_SAVEEXEC %src
//    ... instructions not modifying exec ...
//    %tmp = S_AND $exec, %dst
//    $exec = S_XOR_term $exec, %tmp
// =>
//    %dst = S_OR_SAVEEXEC %src
//    ... instructions not modifying exec ...
//    $exec = S_XOR_term $exec, %dst
//
// Clean up potentially unnecessary code added for safety during
// control flow lowering.
//
```
<a name="block_11"></a>**File:** llvm/lib/Target/AMDGPU/SIOptimizeExecMaskingPreRA.cpp (L294-298) [<sup>↩</sup>](#ref-block_11)
```cpp
  // Check this is an else block.
  auto First = MBB.begin();
  MachineInstr &SaveExecMI = *First;
  if (SaveExecMI.getOpcode() != OrSaveExecOpc)
    return false;
```
<a name="block_12"></a>**File:** llvm/lib/Target/AMDGPU/SIOptimizeExecMaskingPreRA.cpp (L300-308) [<sup>↩</sup>](#ref-block_12)
```cpp
  auto I = llvm::find_if(MBB.terminators(), [this](const MachineInstr &MI) {
    return MI.getOpcode() == XorTermrOpc;
  });
  if (I == MBB.terminators().end())
    return false;

  MachineInstr &XorTermMI = *I;
  if (XorTermMI.getOperand(1).getReg() != Register(ExecReg))
    return false;
```
<a name="block_13"></a>**File:** llvm/lib/Target/AMDGPU/SIOptimizeExecMaskingPreRA.cpp (L313-323) [<sup>↩</sup>](#ref-block_13)
```cpp
  // Find potentially unnecessary S_AND
  MachineInstr *AndExecMI = nullptr;
  I--;
  while (I != First && !AndExecMI) {
    if (I->getOpcode() == AndOpc && I->getOperand(0).getReg() == DstReg &&
        I->getOperand(1).getReg() == Register(ExecReg))
      AndExecMI = &*I;
    I--;
  }
  if (!AndExecMI)
    return false;
```
<a name="block_14"></a>**File:** llvm/lib/Target/AMDGPU/SIOptimizeExecMaskingPreRA.cpp (L325-335) [<sup>↩</sup>](#ref-block_14)
```cpp
  // Check for exec modifying instructions.
  // Note: exec defs do not create live ranges beyond the
  // instruction so isDefBetween cannot be used.
  // Instead just check that the def segments are adjacent.
  SlotIndex StartIdx = LIS->getInstructionIndex(SaveExecMI);
  SlotIndex EndIdx = LIS->getInstructionIndex(*AndExecMI);
  for (MCRegUnit Unit : TRI->regunits(ExecReg)) {
    LiveRange &RegUnit = LIS->getRegUnit(Unit);
    if (RegUnit.find(StartIdx) != std::prev(RegUnit.find(EndIdx)))
      return false;
  }
```
<a name="block_15"></a>**File:** llvm/lib/Target/AMDGPU/SIOptimizeExecMaskingPreRA.cpp (L337-347) [<sup>↩</sup>](#ref-block_15)
```cpp
  // Remove unnecessary S_AND
  LIS->removeInterval(SavedExecReg);
  LIS->removeInterval(DstReg);

  SaveExecMI.getOperand(0).setReg(DstReg);

  LIS->RemoveMachineInstrFromMaps(*AndExecMI);
  AndExecMI->eraseFromParent();

  LIS->createAndComputeVirtRegInterval(DstReg);

```
<a name="block_16"></a>**File:** llvm/lib/Target/AMDGPU/SIOptimizeExecMaskingPreRA.cpp (L368-511) [<sup>↩</sup>](#ref-block_16)
```cpp
bool SIOptimizeExecMaskingPreRA::run(MachineFunction &MF) {
  const GCNSubtarget &ST = MF.getSubtarget<GCNSubtarget>();
  TRI = ST.getRegisterInfo();
  TII = ST.getInstrInfo();
  MRI = &MF.getRegInfo();

  const bool Wave32 = ST.isWave32();
  AndOpc = Wave32 ? AMDGPU::S_AND_B32 : AMDGPU::S_AND_B64;
  Andn2Opc = Wave32 ? AMDGPU::S_ANDN2_B32 : AMDGPU::S_ANDN2_B64;
  OrSaveExecOpc =
      Wave32 ? AMDGPU::S_OR_SAVEEXEC_B32 : AMDGPU::S_OR_SAVEEXEC_B64;
  XorTermrOpc = Wave32 ? AMDGPU::S_XOR_B32_term : AMDGPU::S_XOR_B64_term;
  CondReg = MCRegister::from(Wave32 ? AMDGPU::VCC_LO : AMDGPU::VCC);
  ExecReg = MCRegister::from(Wave32 ? AMDGPU::EXEC_LO : AMDGPU::EXEC);

  DenseSet<Register> RecalcRegs({AMDGPU::EXEC_LO, AMDGPU::EXEC_HI});
  bool Changed = false;

  for (MachineBasicBlock &MBB : MF) {

    if (optimizeElseBranch(MBB)) {
      RecalcRegs.insert(AMDGPU::SCC);
      Changed = true;
    }

    if (optimizeVcndVcmpPair(MBB)) {
      RecalcRegs.insert(AMDGPU::VCC_LO);
      RecalcRegs.insert(AMDGPU::VCC_HI);
      RecalcRegs.insert(AMDGPU::SCC);
      Changed = true;
    }

    // Try to remove unneeded instructions before s_endpgm.
    if (MBB.succ_empty()) {
      if (MBB.empty())
        continue;

      // Skip this if the endpgm has any implicit uses, otherwise we would need
      // to be careful to update / remove them.
      // S_ENDPGM always has a single imm operand that is not used other than to
      // end up in the encoding
      MachineInstr &Term = MBB.back();
      if (Term.getOpcode() != AMDGPU::S_ENDPGM || Term.getNumOperands() != 1)
        continue;

      SmallVector<MachineBasicBlock*, 4> Blocks({&MBB});

      while (!Blocks.empty()) {
        auto *CurBB = Blocks.pop_back_val();
        auto I = CurBB->rbegin(), E = CurBB->rend();
        if (I != E) {
          if (I->isUnconditionalBranch() || I->getOpcode() == AMDGPU::S_ENDPGM)
            ++I;
          else if (I->isBranch())
            continue;
        }

        while (I != E) {
          if (I->isDebugInstr()) {
            I = std::next(I);
            continue;
          }

          if (I->mayStore() || I->isBarrier() || I->isCall() ||
              I->hasUnmodeledSideEffects() || I->hasOrderedMemoryRef())
            break;

          LLVM_DEBUG(dbgs()
                     << "Removing no effect instruction: " << *I << '\n');

          for (auto &Op : I->operands()) {
            if (Op.isReg())
              RecalcRegs.insert(Op.getReg());
          }

          auto Next = std::next(I);
          LIS->RemoveMachineInstrFromMaps(*I);
          I->eraseFromParent();
          I = Next;

          Changed = true;
        }

        if (I != E)
          continue;

        // Try to ascend predecessors.
        for (auto *Pred : CurBB->predecessors()) {
          if (Pred->succ_size() == 1)
            Blocks.push_back(Pred);
        }
      }
      continue;
    }

    // If the only user of a logical operation is move to exec, fold it now
    // to prevent forming of saveexec. I.e.:
    //
    //    %0:sreg_64 = COPY $exec
    //    %1:sreg_64 = S_AND_B64 %0:sreg_64, %2:sreg_64
    // =>
    //    %1 = S_AND_B64 $exec, %2:sreg_64
    unsigned ScanThreshold = 10;
    for (auto I = MBB.rbegin(), E = MBB.rend(); I != E
         && ScanThreshold--; ++I) {
      // Continue scanning if this is not a full exec copy
      if (!(I->isFullCopy() && I->getOperand(1).getReg() == Register(ExecReg)))
        continue;

      Register SavedExec = I->getOperand(0).getReg();
      if (SavedExec.isVirtual() && MRI->hasOneNonDBGUse(SavedExec)) {
        MachineInstr *SingleExecUser = &*MRI->use_instr_nodbg_begin(SavedExec);
        int Idx = SingleExecUser->findRegisterUseOperandIdx(SavedExec,
                                                            /*TRI=*/nullptr);
        assert(Idx != -1);
        if (SingleExecUser->getParent() == I->getParent() &&
            !SingleExecUser->getOperand(Idx).isImplicit() &&
            TII->isOperandLegal(*SingleExecUser, Idx, &I->getOperand(1))) {
          LLVM_DEBUG(dbgs() << "Redundant EXEC COPY: " << *I << '\n');
          LIS->RemoveMachineInstrFromMaps(*I);
          I->eraseFromParent();
          MRI->replaceRegWith(SavedExec, ExecReg);
          LIS->removeInterval(SavedExec);
          Changed = true;
        }
      }
      break;
    }
  }

  if (Changed) {
    for (auto Reg : RecalcRegs) {
      if (Reg.isVirtual()) {
        LIS->removeInterval(Reg);
        if (!MRI->reg_empty(Reg))
          LIS->createAndComputeVirtRegInterval(Reg);
      } else {
        LIS->removeAllRegUnitsForPhysReg(Reg);
      }
    }
  }

  return Changed;
}
```