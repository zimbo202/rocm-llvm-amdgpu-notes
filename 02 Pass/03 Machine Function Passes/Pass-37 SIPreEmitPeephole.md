# SIPreEmitPeephole.cpp 代码功能详解

## 1. Pass的主要功能概述

<a name="ref-block_0"></a>**SIPreEmitPeephole** 是一个在代码生成（code emission）之前执行的窥孔优化（peephole optimization）pass llvm-project:10-10[<sup>↗</sup>](#block_0) 。

**主要作用与效果：**
- 在AMDGPU目标架构上，对机器指令进行局部优化
- 优化分支指令，减少不必要的条件判断
- 消除冗余的GPR索引设置指令
- 移除不必要的EXECZ分支，提升性能

## 2. 主要功能步骤与子功能提取

该pass包含以下核心子功能：

1. **`optimizeVccBranch`** - VCC分支优化
2. **`optimizeSetGPR`** - GPR索引模式优化
3. **`removeExeczBranch`** - EXECZ分支消除
4. **`getBlockDestinations`** - 分支目标获取（辅助函数）
5. **`mustRetainExeczBranch`** - 分支保留判断
6. **`run`** - 主执行入口

## 3. 各子功能的详细描述分析

### 3.1 optimizeVccBranch - VCC分支优化

<a name="ref-block_1"></a>该函数执行两种主要的分支优化模式 llvm-project:69-87[<sup>↗</sup>](#block_1) ：

**优化模式一：**
将以下模式：
```
sreg = -1 or 0
vcc = S_AND_B64 exec, sreg 或 S_ANDN2_B64 exec, sreg
S_CBRANCH_VCC[N]Z
```
转换为：
```
S_CBRANCH_EXEC[N]Z
```

**优化模式二：**
将以下模式：
```
vcc = V_CMP
vcc = S_AND exec, vcc
S_CBRANCH_VCC[N]Z
```
转换为：
```
vcc = V_CMP
S_CBRANCH_VCC[N]Z
```

**核心优化逻辑：**
<a name="ref-block_2"></a>- 向后扫描最多5条指令，查找VCC的定义 llvm-project:98-116[<sup>↗</sup>](#block_2) 
<a name="ref-block_3"></a>- 检查是否为S_AND或S_ANDN2指令，且操作数之一为EXEC寄存器 llvm-project:118-128[<sup>↗</sup>](#block_3) 
<a name="ref-block_4"></a>- 如果第二个操作数是寄存器，继续向后查找其定义，确认是否为-1或0的立即数移动 llvm-project:131-169[<sup>↗</sup>](#block_4) 
- 对于VALU比较指令（V_CMP），可以直接消除冗余的S_AND指令 llvm-project:146-154 
<a name="ref-block_5"></a>- 根据掩码值（MaskValue）和分支类型，将分支转换为EXEC分支或无条件分支 llvm-project:191-241[<sup>↗</sup>](#block_5) 

### 3.2 optimizeSetGPR - GPR索引模式优化

<a name="ref-block_6"></a>该函数优化重复的`S_SET_GPR_IDX_ON`指令 llvm-project:246-247[<sup>↗</sup>](#block_6) 。

**优化原理：**
- 当两个相同的`S_SET_GPR_IDX_ON`指令之间没有修改M0寄存器或索引寄存器的指令时，第二个可以被删除
<a name="ref-block_7"></a>- 扫描两个`S_SET_GPR_IDX_ON`指令之间的所有指令 llvm-project:260-289[<sup>↗</sup>](#block_7) 
- 检查是否存在以下情况：
  - `S_SET_GPR_IDX_MODE`指令（返回false，不优化）
  - `S_SET_GPR_IDX_OFF`指令（记录为IdxOn=false）
  - 修改M0或索引寄存器的指令（返回false）
  - 使用向量寄存器的指令（除非是间接向量移动指令）
<a name="ref-block_8"></a>- 如果检查通过，删除第二个`S_SET_GPR_IDX_ON`指令及中间的`S_SET_GPR_IDX_OFF`指令 llvm-project:291-294[<sup>↗</sup>](#block_8) 

### 3.3 removeExeczBranch - EXECZ分支消除

<a name="ref-block_18"></a>该函数尝试移除`S_CBRANCH_EXECZ`分支指令以提升性能 llvm-project:392-393[<sup>↗</sup>](#block_18) 。

**执行条件：**
<a name="ref-block_19"></a>- 需要指令调度模型支持 llvm-project:395-396[<sup>↗</sup>](#block_19) 
<a name="ref-block_21"></a>- 只考虑前向分支（源基本块号小于目标基本块号） llvm-project:405-407[<sup>↗</sup>](#block_21) 
<a name="ref-block_22"></a>- 必须满足合法性和收益性判断 llvm-project:409-411[<sup>↗</sup>](#block_22) 

**优化效果：**
- 移除EXECZ分支指令
<a name="ref-block_23"></a>- 更新基本块的后继关系 llvm-project:413-415[<sup>↗</sup>](#block_23) 

### 3.4 mustRetainExeczBranch - 分支保留判断

<a name="ref-block_11"></a>该函数使用成本模型判断是否必须保留EXECZ分支 llvm-project:354-357[<sup>↗</sup>](#block_11) 。

**判断依据：**
<a name="ref-block_13"></a>1. **控制流分析**：检查从源基本块到目标基本块之间的所有指令 llvm-project:361-363[<sup>↗</sup>](#block_13) 
<a name="ref-block_14"></a>2. **循环检测**：如果发现条件分支，说明可能存在循环，必须保留分支以避免无限循环 llvm-project:366-370[<sup>↗</sup>](#block_14) 
<a name="ref-block_15"></a>3. **非顺序跳转**：如果有跳转到非下一个基本块的无条件分支，保留分支 llvm-project:372-374[<sup>↗</sup>](#block_15) 
<a name="ref-block_16"></a>4. **副作用检测**：如果指令在EXEC=0时有不良副作用，保留分支 llvm-project:379-380[<sup>↗</sup>](#block_16) 
<a name="ref-block_17"></a>5. **成本模型**：使用`BranchWeightCostModel`进行收益性分析 llvm-project:382-383[<sup>↗</sup>](#block_17) 

<a name="ref-block_10"></a>**成本模型计算** llvm-project:310-352[<sup>↗</sup>](#block_10) ：
- 基于分支概率和指令延迟计算总成本
- 比较始终执行then-block与条件执行的成本
- 公式：`(1-P) * ThenCost <= (1-P)*BranchTakenCost + P*BranchNotTakenCost`

### 3.5 getBlockDestinations - 分支目标获取

<a name="ref-block_9"></a>该辅助函数获取基本块的真假分支目标 llvm-project:297-307[<sup>↗</sup>](#block_9) ：
- 使用`analyzeBranch`分析分支结构
- 如果没有假分支，将下一个基本块作为假分支目标

### 3.6 run - 主执行函数

<a name="ref-block_24"></a>这是pass的主入口函数，协调所有优化操作 llvm-project:429-487[<sup>↗</sup>](#block_24) 。

**执行流程：**
1. 初始化目标特定的指令信息和寄存器信息 llvm-project:430-432 
2. 重新编号基本块 llvm-project:435-435 
3. 遍历所有基本块 llvm-project:437-437 
4. **分支优化阶段**：检查终结符指令，针对不同类型的分支调用相应的优化函数 llvm-project:438-451 
   - `S_CBRANCH_VCCZ/VCCNZ` → `optimizeVccBranch`
   - `S_CBRANCH_EXECZ` → `removeExeczBranch`
5. **GPR索引优化阶段**：如果硬件支持VGPR索引模式，扫描基本块查找重复的`S_SET_GPR_IDX_ON`指令 llvm-project:453-484 
   - 限制扫描距离为20条指令以控制编译时间
   - 需要处理指令bundle的情况

## 4. 步骤/子功能之间的关系

### 执行顺序关系
```
run (主入口)
├── 遍历所有基本块
│   ├── 阶段1: 分支优化
│   │   ├── optimizeVccBranch (优化VCC分支)
│   │   └── removeExeczBranch (移除EXECZ分支)
│   │       ├── getBlockDestinations (获取分支目标)
│   │       └── mustRetainExeczBranch (判断是否保留)
│   │           └── BranchWeightCostModel (成本分析)
│   └── 阶段2: GPR索引优化
│       └── optimizeSetGPR (优化GPR索引设置)
```

### 功能协作关系

1. **`run` 作为调度中心**：协调所有优化操作，确保按正确顺序执行 llvm-project:429-429 

2. **分支优化子系统**：
<a name="ref-block_20"></a>   - `removeExeczBranch`依赖`getBlockDestinations`获取分支信息 llvm-project:402-403[<sup>↗</sup>](#block_20) 
   - `removeExeczBranch`依赖`mustRetainExeczBranch`进行收益性判断 llvm-project:410-410 
<a name="ref-block_12"></a>   - `mustRetainExeczBranch`使用`BranchWeightCostModel`进行成本分析 llvm-project:358-358[<sup>↗</sup>](#block_12) 

3. **优化的独立性**：
   - `optimizeVccBranch`和`optimizeSetGPR`是相对独立的优化
   - 它们不直接相互依赖，但都通过`run`函数统一调度

4. **两阶段设计**：
   - 第一阶段处理终结符（分支指令） llvm-project:438-451 
   - 第二阶段处理非终结符（GPR索引指令） llvm-project:453-484 
   - 这种分离确保不同类型的优化不会相互干扰

## Notes

该pass是AMDGPU后端在代码生成最后阶段的重要优化pass，主要特点：

- **针对性强**：专门针对AMDGPU架构的特定指令模式进行优化
- **局部性**：使用窥孔优化技术，在局部指令序列中寻找优化机会
- **保守性**：通过多重检查确保优化的正确性和有效性
- **成本感知**：在移除分支时考虑性能收益，避免负优化
- **硬件相关**：部分优化（如`optimizeSetGPR`）依赖于硬件特性（`hasVGPRIndexMode`）

这些优化对于减少AMDGPU上的分支开销和消除冗余指令非常重要，能够提升最终生成代码的执行效率。
### Citations
<a name="block_0"></a>**File:** llvm/lib/Target/AMDGPU/SIPreEmitPeephole.cpp (L10-10) [<sup>↩</sup>](#ref-block_0)
```cpp
/// This pass performs the peephole optimizations before code emission.
```
<a name="block_1"></a>**File:** llvm/lib/Target/AMDGPU/SIPreEmitPeephole.cpp (L69-87) [<sup>↩</sup>](#ref-block_1)
```cpp
bool SIPreEmitPeephole::optimizeVccBranch(MachineInstr &MI) const {
  // Match:
  // sreg = -1 or 0
  // vcc = S_AND_B64 exec, sreg or S_ANDN2_B64 exec, sreg
  // S_CBRANCH_VCC[N]Z
  // =>
  // S_CBRANCH_EXEC[N]Z
  // We end up with this pattern sometimes after basic block placement.
  // It happens while combining a block which assigns -1 or 0 to a saved mask
  // and another block which consumes that saved mask and then a branch.
  //
  // While searching this also performs the following substitution:
  // vcc = V_CMP
  // vcc = S_AND exec, vcc
  // S_CBRANCH_VCC[N]Z
  // =>
  // vcc = V_CMP
  // S_CBRANCH_VCC[N]Z

```
<a name="block_2"></a>**File:** llvm/lib/Target/AMDGPU/SIPreEmitPeephole.cpp (L98-116) [<sup>↩</sup>](#ref-block_2)
```cpp
  MachineBasicBlock::reverse_iterator A = MI.getReverseIterator(),
                                      E = MBB.rend();
  bool ReadsCond = false;
  unsigned Threshold = 5;
  for (++A; A != E; ++A) {
    if (!--Threshold)
      return false;
    if (A->modifiesRegister(ExecReg, TRI))
      return false;
    if (A->modifiesRegister(CondReg, TRI)) {
      if (!A->definesRegister(CondReg, TRI) ||
          (A->getOpcode() != And && A->getOpcode() != AndN2))
        return false;
      break;
    }
    ReadsCond |= A->readsRegister(CondReg, TRI);
  }
  if (A == E)
    return false;
```
<a name="block_3"></a>**File:** llvm/lib/Target/AMDGPU/SIPreEmitPeephole.cpp (L118-128) [<sup>↩</sup>](#ref-block_3)
```cpp
  MachineOperand &Op1 = A->getOperand(1);
  MachineOperand &Op2 = A->getOperand(2);
  if (Op1.getReg() != ExecReg && Op2.isReg() && Op2.getReg() == ExecReg) {
    TII->commuteInstruction(*A);
    Changed = true;
  }
  if (Op1.getReg() != ExecReg)
    return Changed;
  if (Op2.isImm() && !(Op2.getImm() == -1 || Op2.getImm() == 0))
    return Changed;

```
<a name="block_4"></a>**File:** llvm/lib/Target/AMDGPU/SIPreEmitPeephole.cpp (L131-169) [<sup>↩</sup>](#ref-block_4)
```cpp
  if (Op2.isReg()) {
    SReg = Op2.getReg();
    auto M = std::next(A);
    bool ReadsSreg = false;
    bool ModifiesExec = false;
    for (; M != E; ++M) {
      if (M->definesRegister(SReg, TRI))
        break;
      if (M->modifiesRegister(SReg, TRI))
        return Changed;
      ReadsSreg |= M->readsRegister(SReg, TRI);
      ModifiesExec |= M->modifiesRegister(ExecReg, TRI);
    }
    if (M == E)
      return Changed;
    // If SReg is VCC and SReg definition is a VALU comparison.
    // This means S_AND with EXEC is not required.
    // Erase the S_AND and return.
    // Note: isVOPC is used instead of isCompare to catch V_CMP_CLASS
    if (A->getOpcode() == And && SReg == CondReg && !ModifiesExec &&
        TII->isVOPC(*M)) {
      A->eraseFromParent();
      return true;
    }
    if (!M->isMoveImmediate() || !M->getOperand(1).isImm() ||
        (M->getOperand(1).getImm() != -1 && M->getOperand(1).getImm() != 0))
      return Changed;
    MaskValue = M->getOperand(1).getImm();
    // First if sreg is only used in the AND instruction fold the immediate
    // into the AND.
    if (!ReadsSreg && Op2.isKill()) {
      A->getOperand(2).ChangeToImmediate(MaskValue);
      M->eraseFromParent();
    }
  } else if (Op2.isImm()) {
    MaskValue = Op2.getImm();
  } else {
    llvm_unreachable("Op2 must be register or immediate");
  }
```
<a name="block_5"></a>**File:** llvm/lib/Target/AMDGPU/SIPreEmitPeephole.cpp (L191-241) [<sup>↩</sup>](#ref-block_5)
```cpp
  bool IsVCCZ = MI.getOpcode() == AMDGPU::S_CBRANCH_VCCZ;
  if (SReg == ExecReg) {
    // EXEC is updated directly
    if (IsVCCZ) {
      MI.eraseFromParent();
      return true;
    }
    MI.setDesc(TII->get(AMDGPU::S_BRANCH));
  } else if (IsVCCZ && MaskValue == 0) {
    // Will always branch
    // Remove all successors shadowed by new unconditional branch
    MachineBasicBlock *Parent = MI.getParent();
    SmallVector<MachineInstr *, 4> ToRemove;
    bool Found = false;
    for (MachineInstr &Term : Parent->terminators()) {
      if (Found) {
        if (Term.isBranch())
          ToRemove.push_back(&Term);
      } else {
        Found = Term.isIdenticalTo(MI);
      }
    }
    assert(Found && "conditional branch is not terminator");
    for (auto *BranchMI : ToRemove) {
      MachineOperand &Dst = BranchMI->getOperand(0);
      assert(Dst.isMBB() && "destination is not basic block");
      Parent->removeSuccessor(Dst.getMBB());
      BranchMI->eraseFromParent();
    }

    if (MachineBasicBlock *Succ = Parent->getFallThrough()) {
      Parent->removeSuccessor(Succ);
    }

    // Rewrite to unconditional branch
    MI.setDesc(TII->get(AMDGPU::S_BRANCH));
  } else if (!IsVCCZ && MaskValue == 0) {
    // Will never branch
    MachineOperand &Dst = MI.getOperand(0);
    assert(Dst.isMBB() && "destination is not basic block");
    MI.getParent()->removeSuccessor(Dst.getMBB());
    MI.eraseFromParent();
    return true;
  } else if (MaskValue == -1) {
    // Depends only on EXEC
    MI.setDesc(
        TII->get(IsVCCZ ? AMDGPU::S_CBRANCH_EXECZ : AMDGPU::S_CBRANCH_EXECNZ));
  }

  MI.removeOperand(MI.findRegisterUseOperandIdx(CondReg, TRI, false /*Kill*/));
  MI.addImplicitDefUseOperands(*MBB.getParent());
```
<a name="block_6"></a>**File:** llvm/lib/Target/AMDGPU/SIPreEmitPeephole.cpp (L246-247) [<sup>↩</sup>](#ref-block_6)
```cpp
bool SIPreEmitPeephole::optimizeSetGPR(MachineInstr &First,
                                       MachineInstr &MI) const {
```
<a name="block_7"></a>**File:** llvm/lib/Target/AMDGPU/SIPreEmitPeephole.cpp (L260-289) [<sup>↩</sup>](#ref-block_7)
```cpp
  for (MachineBasicBlock::instr_iterator I = std::next(First.getIterator()),
                                         E = MI.getIterator();
       I != E; ++I) {
    if (I->isBundle())
      continue;
    switch (I->getOpcode()) {
    case AMDGPU::S_SET_GPR_IDX_MODE:
      return false;
    case AMDGPU::S_SET_GPR_IDX_OFF:
      IdxOn = false;
      ToRemove.push_back(&*I);
      break;
    default:
      if (I->modifiesRegister(AMDGPU::M0, TRI))
        return false;
      if (IdxReg && I->modifiesRegister(IdxReg, TRI))
        return false;
      if (llvm::any_of(I->operands(),
                       [&MRI, this](const MachineOperand &MO) {
                         return MO.isReg() &&
                                TRI->isVectorRegister(MRI, MO.getReg());
                       })) {
        // The only exception allowed here is another indirect vector move
        // with the same mode.
        if (!IdxOn || !(I->getOpcode() == AMDGPU::V_MOV_B32_indirect_write ||
                        I->getOpcode() == AMDGPU::V_MOV_B32_indirect_read))
          return false;
      }
    }
  }
```
<a name="block_8"></a>**File:** llvm/lib/Target/AMDGPU/SIPreEmitPeephole.cpp (L291-294) [<sup>↩</sup>](#ref-block_8)
```cpp
  MI.eraseFromBundle();
  for (MachineInstr *RI : ToRemove)
    RI->eraseFromBundle();
  return true;
```
<a name="block_9"></a>**File:** llvm/lib/Target/AMDGPU/SIPreEmitPeephole.cpp (L297-307) [<sup>↩</sup>](#ref-block_9)
```cpp
bool SIPreEmitPeephole::getBlockDestinations(
    MachineBasicBlock &SrcMBB, MachineBasicBlock *&TrueMBB,
    MachineBasicBlock *&FalseMBB, SmallVectorImpl<MachineOperand> &Cond) {
  if (TII->analyzeBranch(SrcMBB, TrueMBB, FalseMBB, Cond))
    return false;

  if (!FalseMBB)
    FalseMBB = SrcMBB.getNextNode();

  return true;
}
```
<a name="block_10"></a>**File:** llvm/lib/Target/AMDGPU/SIPreEmitPeephole.cpp (L310-352) [<sup>↩</sup>](#ref-block_10)
```cpp
class BranchWeightCostModel {
  const SIInstrInfo &TII;
  const TargetSchedModel &SchedModel;
  BranchProbability BranchProb;
  static constexpr uint64_t BranchNotTakenCost = 1;
  uint64_t BranchTakenCost;
  uint64_t ThenCyclesCost = 0;

public:
  BranchWeightCostModel(const SIInstrInfo &TII, const MachineInstr &Branch,
                        const MachineBasicBlock &Succ)
      : TII(TII), SchedModel(TII.getSchedModel()) {
    const MachineBasicBlock &Head = *Branch.getParent();
    const auto *FromIt = find(Head.successors(), &Succ);
    assert(FromIt != Head.succ_end());

    BranchProb = Head.getSuccProbability(FromIt);
    if (BranchProb.isUnknown())
      BranchProb = BranchProbability::getZero();
    BranchTakenCost = SchedModel.computeInstrLatency(&Branch);
  }

  bool isProfitable(const MachineInstr &MI) {
    if (TII.isWaitcnt(MI.getOpcode()))
      return false;

    ThenCyclesCost += SchedModel.computeInstrLatency(&MI);

    // Consider `P = N/D` to be the probability of execz being false (skipping
    // the then-block) The transformation is profitable if always executing the
    // 'then' block is cheaper than executing sometimes 'then' and always
    // executing s_cbranch_execz:
    // * ThenCost <= P*ThenCost + (1-P)*BranchTakenCost + P*BranchNotTakenCost
    // * (1-P) * ThenCost <= (1-P)*BranchTakenCost + P*BranchNotTakenCost
    // * (D-N)/D * ThenCost <= (D-N)/D * BranchTakenCost + N/D *
    // BranchNotTakenCost
    uint64_t Numerator = BranchProb.getNumerator();
    uint64_t Denominator = BranchProb.getDenominator();
    return (Denominator - Numerator) * ThenCyclesCost <=
           ((Denominator - Numerator) * BranchTakenCost +
            Numerator * BranchNotTakenCost);
  }
};
```
<a name="block_11"></a>**File:** llvm/lib/Target/AMDGPU/SIPreEmitPeephole.cpp (L354-357) [<sup>↩</sup>](#ref-block_11)
```cpp
bool SIPreEmitPeephole::mustRetainExeczBranch(
    const MachineInstr &Branch, const MachineBasicBlock &From,
    const MachineBasicBlock &To) const {
  assert(is_contained(Branch.getParent()->successors(), &From));
```
<a name="block_12"></a>**File:** llvm/lib/Target/AMDGPU/SIPreEmitPeephole.cpp (L358-358) [<sup>↩</sup>](#ref-block_12)
```cpp
  BranchWeightCostModel CostModel{*TII, Branch, From};
```
<a name="block_13"></a>**File:** llvm/lib/Target/AMDGPU/SIPreEmitPeephole.cpp (L361-363) [<sup>↩</sup>](#ref-block_13)
```cpp
  for (MachineFunction::const_iterator MBBI(&From), ToI(&To), End = MF->end();
       MBBI != End && MBBI != ToI; ++MBBI) {
    const MachineBasicBlock &MBB = *MBBI;
```
<a name="block_14"></a>**File:** llvm/lib/Target/AMDGPU/SIPreEmitPeephole.cpp (L366-370) [<sup>↩</sup>](#ref-block_14)
```cpp
      // When a uniform loop is inside non-uniform control flow, the branch
      // leaving the loop might never be taken when EXEC = 0.
      // Hence we should retain cbranch out of the loop lest it become infinite.
      if (MI.isConditionalBranch())
        return true;
```
<a name="block_15"></a>**File:** llvm/lib/Target/AMDGPU/SIPreEmitPeephole.cpp (L372-374) [<sup>↩</sup>](#ref-block_15)
```cpp
      if (MI.isUnconditionalBranch() &&
          TII->getBranchDestBlock(MI) != MBB.getNextNode())
        return true;
```
<a name="block_16"></a>**File:** llvm/lib/Target/AMDGPU/SIPreEmitPeephole.cpp (L379-380) [<sup>↩</sup>](#ref-block_16)
```cpp
      if (TII->hasUnwantedEffectsWhenEXECEmpty(MI))
        return true;
```
<a name="block_17"></a>**File:** llvm/lib/Target/AMDGPU/SIPreEmitPeephole.cpp (L382-383) [<sup>↩</sup>](#ref-block_17)
```cpp
      if (!CostModel.isProfitable(MI))
        return true;
```
<a name="block_18"></a>**File:** llvm/lib/Target/AMDGPU/SIPreEmitPeephole.cpp (L392-393) [<sup>↩</sup>](#ref-block_18)
```cpp
bool SIPreEmitPeephole::removeExeczBranch(MachineInstr &MI,
                                          MachineBasicBlock &SrcMBB) {
```
<a name="block_19"></a>**File:** llvm/lib/Target/AMDGPU/SIPreEmitPeephole.cpp (L395-396) [<sup>↩</sup>](#ref-block_19)
```cpp
  if (!TII->getSchedModel().hasInstrSchedModel())
    return false;
```
<a name="block_20"></a>**File:** llvm/lib/Target/AMDGPU/SIPreEmitPeephole.cpp (L402-403) [<sup>↩</sup>](#ref-block_20)
```cpp
  if (!getBlockDestinations(SrcMBB, TrueMBB, FalseMBB, Cond))
    return false;
```
<a name="block_21"></a>**File:** llvm/lib/Target/AMDGPU/SIPreEmitPeephole.cpp (L405-407) [<sup>↩</sup>](#ref-block_21)
```cpp
  // Consider only the forward branches.
  if (SrcMBB.getNumber() >= TrueMBB->getNumber())
    return false;
```
<a name="block_22"></a>**File:** llvm/lib/Target/AMDGPU/SIPreEmitPeephole.cpp (L409-411) [<sup>↩</sup>](#ref-block_22)
```cpp
  // Consider only when it is legal and profitable
  if (mustRetainExeczBranch(MI, *FalseMBB, *TrueMBB))
    return false;
```
<a name="block_23"></a>**File:** llvm/lib/Target/AMDGPU/SIPreEmitPeephole.cpp (L413-415) [<sup>↩</sup>](#ref-block_23)
```cpp
  LLVM_DEBUG(dbgs() << "Removing the execz branch: " << MI);
  MI.eraseFromParent();
  SrcMBB.removeSuccessor(TrueMBB);
```
<a name="block_24"></a>**File:** llvm/lib/Target/AMDGPU/SIPreEmitPeephole.cpp (L429-487) [<sup>↩</sup>](#ref-block_24)
```cpp
bool SIPreEmitPeephole::run(MachineFunction &MF) {
  const GCNSubtarget &ST = MF.getSubtarget<GCNSubtarget>();
  TII = ST.getInstrInfo();
  TRI = &TII->getRegisterInfo();
  bool Changed = false;

  MF.RenumberBlocks();

  for (MachineBasicBlock &MBB : MF) {
    MachineBasicBlock::iterator TermI = MBB.getFirstTerminator();
    // Check first terminator for branches to optimize
    if (TermI != MBB.end()) {
      MachineInstr &MI = *TermI;
      switch (MI.getOpcode()) {
      case AMDGPU::S_CBRANCH_VCCZ:
      case AMDGPU::S_CBRANCH_VCCNZ:
        Changed |= optimizeVccBranch(MI);
        break;
      case AMDGPU::S_CBRANCH_EXECZ:
        Changed |= removeExeczBranch(MI, MBB);
        break;
      }
    }

    if (!ST.hasVGPRIndexMode())
      continue;

    MachineInstr *SetGPRMI = nullptr;
    const unsigned Threshold = 20;
    unsigned Count = 0;
    // Scan the block for two S_SET_GPR_IDX_ON instructions to see if a
    // second is not needed. Do expensive checks in the optimizeSetGPR()
    // and limit the distance to 20 instructions for compile time purposes.
    // Note: this needs to work on bundles as S_SET_GPR_IDX* instructions
    // may be bundled with the instructions they modify.
    for (auto &MI : make_early_inc_range(MBB.instrs())) {
      if (Count == Threshold)
        SetGPRMI = nullptr;
      else
        ++Count;

      if (MI.getOpcode() != AMDGPU::S_SET_GPR_IDX_ON)
        continue;

      Count = 0;
      if (!SetGPRMI) {
        SetGPRMI = &MI;
        continue;
      }

      if (optimizeSetGPR(*SetGPRMI, MI))
        Changed = true;
      else
        SetGPRMI = &MI;
    }
  }

  return Changed;
}
```