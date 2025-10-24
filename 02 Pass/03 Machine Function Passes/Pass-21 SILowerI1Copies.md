# SILowerI1Copies.cpp 代码功能分析

## 1. Pass的主要功能概述

<a name="ref-block_0"></a>该Pass的主要作用是将所有i1值（使用vreg_1寄存器类）降低（lower）为lane masks（车道掩码，即32位或64位标量寄存器）。 llvm-project:9-11[<sup>↗</sup>](#block_0) 

**核心目标和效果：**
- 该Pass假设机器处于SSA形式且具有wave级别的控制流图
- 在此Pass之前，语义上为i1且在同一基本块内定义和使用的值已经被表示为标量寄存器中的lane masks
<a name="ref-block_1"></a>- 该Pass专门处理跨基本块传递的i1值，这些值在基本块之间通过vreg_1虚拟寄存器传递 llvm-project:13-17[<sup>↗</sup>](#block_1) 
<a name="ref-block_2"></a>- 仅有COPY、PHI和IMPLICIT_DEF指令会使用或定义vreg_1虚拟寄存器 llvm-project:19-20[<sup>↗</sup>](#block_2) 

## 2. 主要功能步骤和子功能提取

该Pass通过以下主要步骤实现功能：

### 步骤1：lowerCopiesFromI1
<a name="ref-block_6"></a>处理从vreg_1到向量寄存器的COPY指令 llvm-project:399-440[<sup>↗</sup>](#block_6) 

### 步骤2：lowerPhis  
<a name="ref-block_7"></a>降低使用vreg_1的PHI节点 llvm-project:471-584[<sup>↗</sup>](#block_7) 

### 步骤3：lowerCopiesToI1
<a name="ref-block_8"></a>降低到vreg_1的COPY和IMPLICIT_DEF指令 llvm-project:586-661[<sup>↗</sup>](#block_8) 

### 辅助类和子功能：

#### PhiIncomingAnalysis类
<a name="ref-block_4"></a>用于确定phi的incoming值在控制流图中的关系，判断何时可以直接使用标量lane mask，何时需要与之前定义的lane mask合并 llvm-project:84-102[<sup>↗</sup>](#block_4) 

#### LoopFinder类
<a name="ref-block_5"></a>检测需要将i1 COPY降低为按位操作的循环。它判断是否需要按位降低的规则是：如果从块B的定义可以到达返回B的后向边，而无需经过B和所有使用该定义的块的最近公共后支配者 llvm-project:180-204[<sup>↗</sup>](#block_5) 

#### Vreg1LoweringHelper类
<a name="ref-block_3"></a>主要的辅助类，继承自PhiLoweringHelper，协调整个降低过程 llvm-project:39-68[<sup>↗</sup>](#block_3) 

## 3. 各步骤的具体描述分析

### lowerCopiesFromI1 详细分析
该函数遍历所有基本块和指令，查找从vreg_1寄存器复制到其他寄存器的COPY指令。对于每个这样的COPY：
- 使用V_CNDMASK_B32_e64指令将i1值转换为32位值（0或-1）
- 将源寄存器标记为需要约束到SReg_1_XEXEC寄存器类
- 删除原始COPY指令 llvm-project:419-432 

### lowerPhis 详细分析
这是最复杂的降低过程，处理PHI节点：
- 首先收集所有vreg_1的PHI节点
- 对每个PHI，将目标寄存器标记为lane mask类型
<a name="ref-block_26"></a>- 收集并排序incoming值（按支配关系排序以便进行常量折叠） llvm-project:497-506[<sup>↗</sup>](#block_26) 
- 使用LoopFinder检测是否在循环中观察到PHI
- 根据是否在循环中采用不同的降低策略：
  - 在循环中：为所有incoming值创建新寄存器，并使用SSAUpdater合并
  - 不在循环中：使用PhiIncomingAnalysis进行更精确的降低 llvm-project:529-573 

### lowerCopiesToI1 详细分析
处理目标为vreg_1的COPY和IMPLICIT_DEF指令：
- 对于IMPLICIT_DEF，仅标记为lane mask即可
- 对于COPY，需要转换源操作数：
<a name="ref-block_31"></a>  - 如果源不是lane mask寄存器，使用V_CMP_NE_U32_e64进行比较转换 llvm-project:623-630[<sup>↗</sup>](#block_31) 
<a name="ref-block_32"></a>  - 检测循环并在必要时使用按位操作合并值 llvm-project:637-653[<sup>↗</sup>](#block_32) 

### buildMergeLaneMasks 功能
合并两个lane mask值（PrevReg和CurReg）到目标寄存器：
- 处理常量优化情况
- 使用EXEC寄存器屏蔽当前活动的lanes
<a name="ref-block_9"></a>- 使用按位操作（AND、OR、XOR等）组合值 llvm-project:784-846[<sup>↗</sup>](#block_9) 

## 4. 步骤和子功能之间的关系

整个Pass的执行流程由`runFixI1Copies`函数协调，严格按照以下顺序执行：

**执行顺序和依赖关系：**
```
1. lowerCopiesFromI1() - 第一阶段
2. lowerPhis()         - 第二阶段  
3. lowerCopiesToI1()   - 第三阶段
4. cleanConstrainRegs() - 清理阶段
<a name="ref-block_13"></a>``` llvm-project:866-870[<sup>↗</sup>](#block_13) 

**执行顺序的原因：**
<a name="ref-block_10"></a>- 首先处理从vreg_1到向量寄存器的COPY，因为这发生在vreg_1寄存器被改为标量mask寄存器之前 llvm-project:852-854[<sup>↗</sup>](#block_10) 
<a name="ref-block_11"></a>- PHI节点在其他指令之前降低，因为PHI降低会查看COPY指令，并且通常能使COPY降低变得不必要 llvm-project:856-858[<sup>↗</sup>](#block_11) 

**辅助类的协作关系：**
- `PhiIncomingAnalysis`和`LoopFinder`为`lowerPhis`和`lowerCopiesToI1`提供控制流分析
- `MachineSSAUpdater`用于在PHI和循环降低中维护SSA形式
- 所有降低操作都通过`Vreg1LoweringHelper`类协调，该类维护共享状态和提供通用功能

## Notes

<a name="ref-block_12"></a>该Pass是AMDGPU目标特定的优化Pass，专门处理GPU架构中的SIMT（单指令多线程）执行模型。Lane mask表示在一个wave（wavefront）中哪些线程是活动的，这对于正确处理分支和合并执行至关重要。Pass只在SelectionDAG路径中运行，而不在GlobalISel路径中运行。 llvm-project:861-863[<sup>↗</sup>](#block_12)
### Citations
<a name="block_0"></a>**File:** llvm/lib/Target/AMDGPU/SILowerI1Copies.cpp (L9-11) [<sup>↩</sup>](#ref-block_0) [<sup>↩</sup>](#ref-block_0)
```cpp
// This pass lowers all occurrences of i1 values (with a vreg_1 register class)
// to lane masks (32 / 64-bit scalar registers). The pass assumes machine SSA
// form and a wave-level control flow graph.
```
<a name="block_1"></a>**File:** llvm/lib/Target/AMDGPU/SILowerI1Copies.cpp (L13-17) [<sup>↩</sup>](#ref-block_1) [<sup>↩</sup>](#ref-block_1)
```cpp
// Before this pass, values that are semantically i1 and are defined and used
// within the same basic block are already represented as lane masks in scalar
// registers. However, values that cross basic blocks are always transferred
// between basic blocks in vreg_1 virtual registers and are lowered by this
// pass.
```
<a name="block_2"></a>**File:** llvm/lib/Target/AMDGPU/SILowerI1Copies.cpp (L19-20) [<sup>↩</sup>](#ref-block_2) [<sup>↩</sup>](#ref-block_2)
```cpp
// The only instructions that use or define vreg_1 virtual registers are COPY,
// PHI, and IMPLICIT_DEF.
```
<a name="block_3"></a>**File:** llvm/lib/Target/AMDGPU/SILowerI1Copies.cpp (L39-68) [<sup>↩</sup>](#ref-block_3)
```cpp
class Vreg1LoweringHelper : public PhiLoweringHelper {
public:
  Vreg1LoweringHelper(MachineFunction *MF, MachineDominatorTree *DT,
                      MachinePostDominatorTree *PDT);

private:
  DenseSet<Register> ConstrainRegs;

public:
  void markAsLaneMask(Register DstReg) const override;
  void getCandidatesForLowering(
      SmallVectorImpl<MachineInstr *> &Vreg1Phis) const override;
  void collectIncomingValuesFromPhi(
      const MachineInstr *MI,
      SmallVectorImpl<Incoming> &Incomings) const override;
  void replaceDstReg(Register NewReg, Register OldReg,
                     MachineBasicBlock *MBB) override;
  void buildMergeLaneMasks(MachineBasicBlock &MBB,
                           MachineBasicBlock::iterator I, const DebugLoc &DL,
                           Register DstReg, Register PrevReg,
                           Register CurReg) override;
  void constrainAsLaneMask(Incoming &In) override;

  bool lowerCopiesFromI1();
  bool lowerCopiesToI1();
  bool cleanConstrainRegs(bool Changed);
  bool isVreg1(Register Reg) const {
    return Reg.isVirtual() && MRI->getRegClass(Reg) == &AMDGPU::VReg_1RegClass;
  }
};
```
<a name="block_4"></a>**File:** llvm/lib/Target/AMDGPU/SILowerI1Copies.cpp (L84-102) [<sup>↩</sup>](#ref-block_4) [<sup>↩</sup>](#ref-block_4)
```cpp
/// Helper class that determines the relationship between incoming values of a
/// phi in the control flow graph to determine where an incoming value can
/// simply be taken as a scalar lane mask as-is, and where it needs to be
/// merged with another, previously defined lane mask.
///
/// The approach is as follows:
///  - Determine all basic blocks which, starting from the incoming blocks,
///    a wave may reach before entering the def block (the block containing the
///    phi).
///  - If an incoming block has no predecessors in this set, we can take the
///    incoming value as a scalar lane mask as-is.
///  -- A special case of this is when the def block has a self-loop.
///  - Otherwise, the incoming value needs to be merged with a previously
///    defined lane mask.
///  - If there is a path into the set of reachable blocks that does _not_ go
///    through an incoming block where we can take the scalar lane mask as-is,
///    we need to invent an available value for the SSAUpdater. Choices are
///    0 and undef, with differing consequences for how to merge values etc.
///
```
<a name="block_5"></a>**File:** llvm/lib/Target/AMDGPU/SILowerI1Copies.cpp (L180-204) [<sup>↩</sup>](#ref-block_5) [<sup>↩</sup>](#ref-block_5)
```cpp
/// Helper class that detects loops which require us to lower an i1 COPY into
/// bitwise manipulation.
///
/// Unfortunately, we cannot use LoopInfo because LoopInfo does not distinguish
/// between loops with the same header. Consider this example:
///
///  A-+-+
///  | | |
///  B-+ |
///  |   |
///  C---+
///
/// A is the header of a loop containing A, B, and C as far as LoopInfo is
/// concerned. However, an i1 COPY in B that is used in C must be lowered to
/// bitwise operations to combine results from different loop iterations when
/// B has a divergent branch (since by default we will compile this code such
/// that threads in a wave are merged at the entry of C).
///
/// The following rule is implemented to determine whether bitwise operations
/// are required: use the bitwise lowering for a def in block B if a backward
/// edge to B is reachable without going through the nearest common
/// post-dominator of B and all uses of the def.
///
/// TODO: This rule is conservative because it does not check whether the
///       relevant branches are actually divergent.
```
<a name="block_6"></a>**File:** llvm/lib/Target/AMDGPU/SILowerI1Copies.cpp (L399-440) [<sup>↩</sup>](#ref-block_6)
```cpp
bool Vreg1LoweringHelper::lowerCopiesFromI1() {
  bool Changed = false;
  SmallVector<MachineInstr *, 4> DeadCopies;

  for (MachineBasicBlock &MBB : *MF) {
    for (MachineInstr &MI : MBB) {
      if (MI.getOpcode() != AMDGPU::COPY)
        continue;

      Register DstReg = MI.getOperand(0).getReg();
      Register SrcReg = MI.getOperand(1).getReg();
      if (!isVreg1(SrcReg))
        continue;

      if (isLaneMaskReg(DstReg) || isVreg1(DstReg))
        continue;

      Changed = true;

      // Copy into a 32-bit vector register.
      LLVM_DEBUG(dbgs() << "Lower copy from i1: " << MI);
      DebugLoc DL = MI.getDebugLoc();

      assert(isVRegCompatibleReg(TII->getRegisterInfo(), *MRI, DstReg));
      assert(!MI.getOperand(0).getSubReg());

      ConstrainRegs.insert(SrcReg);
      BuildMI(MBB, MI, DL, TII->get(AMDGPU::V_CNDMASK_B32_e64), DstReg)
          .addImm(0)
          .addImm(0)
          .addImm(0)
          .addImm(-1)
          .addReg(SrcReg);
      DeadCopies.push_back(&MI);
    }

    for (MachineInstr *MI : DeadCopies)
      MI->eraseFromParent();
    DeadCopies.clear();
  }
  return Changed;
}
```
<a name="block_7"></a>**File:** llvm/lib/Target/AMDGPU/SILowerI1Copies.cpp (L471-584) [<sup>↩</sup>](#ref-block_7)
```cpp
bool PhiLoweringHelper::lowerPhis() {
  MachineSSAUpdater SSAUpdater(*MF);
  LoopFinder LF(*DT, *PDT);
  PhiIncomingAnalysis PIA(*PDT, TII);
  SmallVector<MachineInstr *, 4> Vreg1Phis;
  SmallVector<Incoming, 4> Incomings;

  getCandidatesForLowering(Vreg1Phis);
  if (Vreg1Phis.empty())
    return false;

  DT->updateDFSNumbers();
  MachineBasicBlock *PrevMBB = nullptr;
  for (MachineInstr *MI : Vreg1Phis) {
    MachineBasicBlock &MBB = *MI->getParent();
    if (&MBB != PrevMBB) {
      LF.initialize(MBB);
      PrevMBB = &MBB;
    }

    LLVM_DEBUG(dbgs() << "Lower PHI: " << *MI);

    Register DstReg = MI->getOperand(0).getReg();
    markAsLaneMask(DstReg);
    initializeLaneMaskRegisterAttributes(DstReg);

    collectIncomingValuesFromPhi(MI, Incomings);

    // Sort the incomings such that incoming values that dominate other incoming
    // values are sorted earlier. This allows us to do some amount of on-the-fly
    // constant folding.
    // Incoming with smaller DFSNumIn goes first, DFSNumIn is 0 for entry block.
    llvm::sort(Incomings, [this](Incoming LHS, Incoming RHS) {
      return DT->getNode(LHS.Block)->getDFSNumIn() <
             DT->getNode(RHS.Block)->getDFSNumIn();
    });

#ifndef NDEBUG
    PhiRegisters.insert(DstReg);
#endif

    // Phis in a loop that are observed outside the loop receive a simple but
    // conservatively correct treatment.
    std::vector<MachineBasicBlock *> DomBlocks = {&MBB};
    for (MachineInstr &Use : MRI->use_instructions(DstReg))
      DomBlocks.push_back(Use.getParent());

    MachineBasicBlock *PostDomBound =
        PDT->findNearestCommonDominator(DomBlocks);

    // FIXME: This fails to find irreducible cycles. If we have a def (other
    // than a constant) in a pair of blocks that end up looping back to each
    // other, it will be mishandle. Due to structurization this shouldn't occur
    // in practice.
    unsigned FoundLoopLevel = LF.findLoop(PostDomBound);

    SSAUpdater.Initialize(DstReg);

    if (FoundLoopLevel) {
      LF.addLoopEntries(FoundLoopLevel, SSAUpdater, *MRI, LaneMaskRegAttrs,
                        Incomings);

      for (auto &Incoming : Incomings) {
        Incoming.UpdatedReg = createLaneMaskReg(MRI, LaneMaskRegAttrs);
        SSAUpdater.AddAvailableValue(Incoming.Block, Incoming.UpdatedReg);
      }

      for (auto &Incoming : Incomings) {
        MachineBasicBlock &IMBB = *Incoming.Block;
        buildMergeLaneMasks(
            IMBB, getSaluInsertionAtEnd(IMBB), {}, Incoming.UpdatedReg,
            SSAUpdater.GetValueInMiddleOfBlock(&IMBB), Incoming.Reg);
      }
    } else {
      // The phi is not observed from outside a loop. Use a more accurate
      // lowering.
      PIA.analyze(MBB, Incomings);

      for (MachineBasicBlock *MBB : PIA.predecessors())
        SSAUpdater.AddAvailableValue(
            MBB, insertUndefLaneMask(MBB, MRI, LaneMaskRegAttrs));

      for (auto &Incoming : Incomings) {
        MachineBasicBlock &IMBB = *Incoming.Block;
        if (PIA.isSource(IMBB)) {
          constrainAsLaneMask(Incoming);
          SSAUpdater.AddAvailableValue(&IMBB, Incoming.Reg);
        } else {
          Incoming.UpdatedReg = createLaneMaskReg(MRI, LaneMaskRegAttrs);
          SSAUpdater.AddAvailableValue(&IMBB, Incoming.UpdatedReg);
        }
      }

      for (auto &Incoming : Incomings) {
        if (!Incoming.UpdatedReg.isValid())
          continue;

        MachineBasicBlock &IMBB = *Incoming.Block;
        buildMergeLaneMasks(
            IMBB, getSaluInsertionAtEnd(IMBB), {}, Incoming.UpdatedReg,
            SSAUpdater.GetValueInMiddleOfBlock(&IMBB), Incoming.Reg);
      }
    }

    Register NewReg = SSAUpdater.GetValueInMiddleOfBlock(&MBB);
    if (NewReg != DstReg) {
      replaceDstReg(NewReg, DstReg, &MBB);
      MI->eraseFromParent();
    }

    Incomings.clear();
  }
  return true;
}
```
<a name="block_8"></a>**File:** llvm/lib/Target/AMDGPU/SILowerI1Copies.cpp (L586-661) [<sup>↩</sup>](#ref-block_8)
```cpp
bool Vreg1LoweringHelper::lowerCopiesToI1() {
  bool Changed = false;
  MachineSSAUpdater SSAUpdater(*MF);
  LoopFinder LF(*DT, *PDT);
  SmallVector<MachineInstr *, 4> DeadCopies;

  for (MachineBasicBlock &MBB : *MF) {
    LF.initialize(MBB);

    for (MachineInstr &MI : MBB) {
      if (MI.getOpcode() != AMDGPU::IMPLICIT_DEF &&
          MI.getOpcode() != AMDGPU::COPY)
        continue;

      Register DstReg = MI.getOperand(0).getReg();
      if (!isVreg1(DstReg))
        continue;

      Changed = true;

      if (MRI->use_empty(DstReg)) {
        DeadCopies.push_back(&MI);
        continue;
      }

      LLVM_DEBUG(dbgs() << "Lower Other: " << MI);

      markAsLaneMask(DstReg);
      initializeLaneMaskRegisterAttributes(DstReg);

      if (MI.getOpcode() == AMDGPU::IMPLICIT_DEF)
        continue;

      DebugLoc DL = MI.getDebugLoc();
      Register SrcReg = MI.getOperand(1).getReg();
      assert(!MI.getOperand(1).getSubReg());

      if (!SrcReg.isVirtual() || (!isLaneMaskReg(SrcReg) && !isVreg1(SrcReg))) {
        assert(TII->getRegisterInfo().getRegSizeInBits(SrcReg, *MRI) == 32);
        Register TmpReg = createLaneMaskReg(MRI, LaneMaskRegAttrs);
        BuildMI(MBB, MI, DL, TII->get(AMDGPU::V_CMP_NE_U32_e64), TmpReg)
            .addReg(SrcReg)
            .addImm(0);
        MI.getOperand(1).setReg(TmpReg);
        SrcReg = TmpReg;
      } else {
        // SrcReg needs to be live beyond copy.
        MI.getOperand(1).setIsKill(false);
      }

      // Defs in a loop that are observed outside the loop must be transformed
      // into appropriate bit manipulation.
      std::vector<MachineBasicBlock *> DomBlocks = {&MBB};
      for (MachineInstr &Use : MRI->use_instructions(DstReg))
        DomBlocks.push_back(Use.getParent());

      MachineBasicBlock *PostDomBound =
          PDT->findNearestCommonDominator(DomBlocks);
      unsigned FoundLoopLevel = LF.findLoop(PostDomBound);
      if (FoundLoopLevel) {
        SSAUpdater.Initialize(DstReg);
        SSAUpdater.AddAvailableValue(&MBB, DstReg);
        LF.addLoopEntries(FoundLoopLevel, SSAUpdater, *MRI, LaneMaskRegAttrs);

        buildMergeLaneMasks(MBB, MI, DL, DstReg,
                            SSAUpdater.GetValueInMiddleOfBlock(&MBB), SrcReg);
        DeadCopies.push_back(&MI);
      }
    }

    for (MachineInstr *MI : DeadCopies)
      MI->eraseFromParent();
    DeadCopies.clear();
  }
  return Changed;
}
```
<a name="block_9"></a>**File:** llvm/lib/Target/AMDGPU/SILowerI1Copies.cpp (L784-846) [<sup>↩</sup>](#ref-block_9) [<sup>↩</sup>](#ref-block_9)
```cpp
void Vreg1LoweringHelper::buildMergeLaneMasks(MachineBasicBlock &MBB,
                                              MachineBasicBlock::iterator I,
                                              const DebugLoc &DL,
                                              Register DstReg, Register PrevReg,
                                              Register CurReg) {
  bool PrevVal = false;
  bool PrevConstant = isConstantLaneMask(PrevReg, PrevVal);
  bool CurVal = false;
  bool CurConstant = isConstantLaneMask(CurReg, CurVal);

  if (PrevConstant && CurConstant) {
    if (PrevVal == CurVal) {
      BuildMI(MBB, I, DL, TII->get(AMDGPU::COPY), DstReg).addReg(CurReg);
    } else if (CurVal) {
      BuildMI(MBB, I, DL, TII->get(AMDGPU::COPY), DstReg).addReg(ExecReg);
    } else {
      BuildMI(MBB, I, DL, TII->get(XorOp), DstReg)
          .addReg(ExecReg)
          .addImm(-1);
    }
    return;
  }

  Register PrevMaskedReg;
  Register CurMaskedReg;
  if (!PrevConstant) {
    if (CurConstant && CurVal) {
      PrevMaskedReg = PrevReg;
    } else {
      PrevMaskedReg = createLaneMaskReg(MRI, LaneMaskRegAttrs);
      BuildMI(MBB, I, DL, TII->get(AndN2Op), PrevMaskedReg)
          .addReg(PrevReg)
          .addReg(ExecReg);
    }
  }
  if (!CurConstant) {
    // TODO: check whether CurReg is already masked by EXEC
    if (PrevConstant && PrevVal) {
      CurMaskedReg = CurReg;
    } else {
      CurMaskedReg = createLaneMaskReg(MRI, LaneMaskRegAttrs);
      BuildMI(MBB, I, DL, TII->get(AndOp), CurMaskedReg)
          .addReg(CurReg)
          .addReg(ExecReg);
    }
  }

  if (PrevConstant && !PrevVal) {
    BuildMI(MBB, I, DL, TII->get(AMDGPU::COPY), DstReg)
        .addReg(CurMaskedReg);
  } else if (CurConstant && !CurVal) {
    BuildMI(MBB, I, DL, TII->get(AMDGPU::COPY), DstReg)
        .addReg(PrevMaskedReg);
  } else if (PrevConstant && PrevVal) {
    BuildMI(MBB, I, DL, TII->get(OrN2Op), DstReg)
        .addReg(CurMaskedReg)
        .addReg(ExecReg);
  } else {
    BuildMI(MBB, I, DL, TII->get(OrOp), DstReg)
        .addReg(PrevMaskedReg)
        .addReg(CurMaskedReg ? CurMaskedReg : ExecReg);
  }
}
```
<a name="block_10"></a>**File:** llvm/lib/Target/AMDGPU/SILowerI1Copies.cpp (L852-854) [<sup>↩</sup>](#ref-block_10)
```cpp
/// In a first pass, we lower COPYs from vreg_1 to vector registers, as can
/// occur around inline assembly. We do this first, before vreg_1 registers
/// are changed to scalar mask registers.
```
<a name="block_11"></a>**File:** llvm/lib/Target/AMDGPU/SILowerI1Copies.cpp (L856-858) [<sup>↩</sup>](#ref-block_11) [<sup>↩</sup>](#ref-block_11)
```cpp
/// Then we lower all defs of vreg_1 registers. Phi nodes are lowered before
/// all others, because phi lowering looks through copies and can therefore
/// often make copy lowering unnecessary.
```
<a name="block_12"></a>**File:** llvm/lib/Target/AMDGPU/SILowerI1Copies.cpp (L861-863) [<sup>↩</sup>](#ref-block_12) [<sup>↩</sup>](#ref-block_12)
```cpp
  // Only need to run this in SelectionDAG path.
  if (MF.getProperties().hasSelected())
    return false;
```
<a name="block_13"></a>**File:** llvm/lib/Target/AMDGPU/SILowerI1Copies.cpp (L866-870) [<sup>↩</sup>](#ref-block_13)
```cpp
  bool Changed = false;
  Changed |= Helper.lowerCopiesFromI1();
  Changed |= Helper.lowerPhis();
  Changed |= Helper.lowerCopiesToI1();
  return Helper.cleanConstrainRegs(Changed);
```


# SILowerI1Copies.cpp 代码功能详解 V2

## 1. Pass的主要功能概述

<a name="ref-block_0"></a>`SILowerI1Copies` pass的核心功能是将所有i1值（使用vreg_1寄存器类）降低为lane masks（32位或64位标量寄存器）。 llvm-project:9-11[<sup>↗</sup>](#block_0) 

<a name="ref-block_1"></a>该pass假设机器代码处于SSA形式，并工作在wave级别的控制流图上。在此pass之前，在同一基本块内定义和使用的i1值已经被表示为标量寄存器中的lane masks，但跨基本块传递的i1值仍使用vreg_1虚拟寄存器，需要通过此pass进行降低。 llvm-project:13-17[<sup>↗</sup>](#block_1) 

<a name="ref-block_2"></a>只有COPY、PHI和IMPLICIT_DEF指令会使用或定义vreg_1虚拟寄存器。 llvm-project:19-20[<sup>↗</sup>](#block_2) 

## 2. 主要功能的实现步骤

该pass通过以下主要步骤实现其功能：

### 步骤1：`lowerCopiesFromI1()`
降低从vreg_1到向量寄存器的COPY指令

### 步骤2：`lowerPhis()`
降低PHI节点中的vreg_1操作数

### 步骤3：`lowerCopiesToI1()`
降低到vreg_1的COPY和IMPLICIT_DEF指令

### 步骤4：`cleanConstrainRegs()`
清理并约束寄存器类

### 辅助类和功能：
- `PhiIncomingAnalysis`：分析PHI节点的incoming值
- `LoopFinder`：检测循环
- `Vreg1LoweringHelper`：主要的降低辅助类
- `buildMergeLaneMasks()`：合并不同控制流路径的lane masks

## 3. 各步骤的具体功能分析

### 3.1 `lowerCopiesFromI1()`

<a name="ref-block_22"></a>此函数遍历所有基本块和指令，查找源操作数为vreg_1的COPY指令。 llvm-project:399-411[<sup>↗</sup>](#block_22) 

<a name="ref-block_23"></a>对于每个这样的COPY，如果目标寄存器不是lane mask寄存器或vreg_1，则使用`V_CNDMASK_B32_e64`指令将i1值转换为32位向量寄存器。这个指令根据i1条件选择0或-1。 llvm-project:426-431[<sup>↗</sup>](#block_23) 

### 3.2 `lowerPhis()`

这是最复杂的降低步骤。对于每个vreg_1的PHI节点：

<a name="ref-block_25"></a>1. **初始化**：标记目标寄存器为lane mask并初始化SSA更新器 llvm-project:493-495[<sup>↗</sup>](#block_25) 

<a name="ref-block_26"></a>2. **收集incoming值**：从PHI节点收集所有incoming值和对应的基本块 llvm-project:497-506[<sup>↗</sup>](#block_26) 

<a name="ref-block_27"></a>3. **循环检测**：使用`LoopFinder`检测PHI是否在循环中被观察到 llvm-project:514-525[<sup>↗</sup>](#block_27) 

<a name="ref-block_28"></a>4. **处理循环情况**：如果检测到循环，为每个incoming值创建新寄存器，并使用`buildMergeLaneMasks`合并来自不同迭代的值 llvm-project:529-543[<sup>↗</sup>](#block_28) 

<a name="ref-block_29"></a>5. **处理非循环情况**：使用`PhiIncomingAnalysis`进行更精确的降低，区分源块和需要合并的块 llvm-project:544-573[<sup>↗</sup>](#block_29) 

### 3.3 `lowerCopiesToI1()`

<a name="ref-block_30"></a>此函数处理目标为vreg_1的COPY和IMPLICIT_DEF指令。 llvm-project:586-602[<sup>↗</sup>](#block_30) 

<a name="ref-block_31"></a>对于非lane mask的源寄存器，使用`V_CMP_NE_U32_e64`指令将32位值转换为i1值（通过与0比较）。 llvm-project:623-630[<sup>↗</sup>](#block_31) 

<a name="ref-block_32"></a>如果定义在循环中且在循环外被使用，需要使用位操作合并来自不同循环迭代的值。 llvm-project:637-653[<sup>↗</sup>](#block_32) 

### 3.4 `PhiIncomingAnalysis`

此辅助类分析PHI节点incoming值在控制流图中的关系，确定：
- 哪些incoming值可以直接作为标量lane mask使用
<a name="ref-block_4"></a>- 哪些需要与之前定义的lane mask合并 llvm-project:84-102[<sup>↗</sup>](#block_4) 

<a name="ref-block_19"></a>分析方法通过遍历从incoming块到def块的所有可达块来实现。 llvm-project:128-177[<sup>↗</sup>](#block_19) 

### 3.5 `LoopFinder`

<a name="ref-block_5"></a>此类检测需要使用位操作降低i1 COPY的循环。它实现了一个规则：如果从定义块B可以到达B的后向边，而不需要经过B和所有使用def的块的最近公共后支配者，则需要位操作降低。 llvm-project:180-204[<sup>↗</sup>](#block_5) 

<a name="ref-block_21"></a>该类通过层级遍历CFG来检测循环，并缓存遍历结果以供同一基本块内多个定义重用。 llvm-project:256-273[<sup>↗</sup>](#block_21) 

### 3.6 `buildMergeLaneMasks()`

<a name="ref-block_9"></a>此函数负责合并来自不同控制流路径的lane masks。它考虑了常量优化和EXEC寄存器的掩码。 llvm-project:784-846[<sup>↗</sup>](#block_9) 

根据前一个值和当前值是否为常量，生成不同的合并代码：
- 两者都是常量：直接COPY或使用XOR
- 一个是常量：使用AND/OR操作优化
- 都不是常量：使用ANDN2、AND和OR操作组合

## 4. 步骤之间的关系

<a name="ref-block_36"></a>整个pass的执行流程由`runFixI1Copies()`函数协调： llvm-project:859-871[<sup>↗</sup>](#block_36) 

**执行顺序及其原因**：

<a name="ref-block_34"></a>1. **首先执行`lowerCopiesFromI1()`**：在vreg_1寄存器被改变为标量mask寄存器之前，先处理从vreg_1到向量寄存器的COPY（常见于内联汇编周围）。 llvm-project:851-854[<sup>↗</sup>](#block_34) 

<a name="ref-block_11"></a>2. **然后执行`lowerPhis()`**：PHI节点优先于其他指令降低，因为PHI降低会透过COPY查看，这样可以使后续的COPY降低变得不必要。 llvm-project:856-858[<sup>↗</sup>](#block_11) 

3. **接着执行`lowerCopiesToI1()`**：处理所有剩余的到vreg_1的COPY和IMPLICIT_DEF。

<a name="ref-block_17"></a>4. **最后执行`cleanConstrainRegs()`**：约束所有标记的寄存器到正确的寄存器类。 llvm-project:75-82[<sup>↗</sup>](#block_17) 

**辅助类的协作关系**：

- `PhiIncomingAnalysis`和`LoopFinder`为主降低逻辑提供控制流分析支持
- `MachineSSAUpdater`在所有降低步骤中用于维护SSA形式
- `buildMergeLaneMasks()`被PHI降低和COPY降低调用，用于生成实际的合并代码

## Notes

<a name="ref-block_12"></a>该pass仅在SelectionDAG路径中运行，GlobalISel路径不需要此pass。 llvm-project:861-863[<sup>↗</sup>](#block_12) 

<a name="ref-block_24"></a>Pass会根据wave大小（32位或64位）选择不同的操作码和EXEC寄存器。 llvm-project:452-468[<sup>↗</sup>](#block_24) 

循环检测和处理是保守的，因为它不检查相关分支是否真正发散，这可能导致在某些情况下进行不必要的位操作。 llvm-project:202-204
### Citations
<a name="block_14"></a>**File:** llvm/lib/Target/AMDGPU/SILowerI1Copies.cpp (L9-11)
```cpp
// This pass lowers all occurrences of i1 values (with a vreg_1 register class)
// to lane masks (32 / 64-bit scalar registers). The pass assumes machine SSA
// form and a wave-level control flow graph.
```
<a name="block_15"></a>**File:** llvm/lib/Target/AMDGPU/SILowerI1Copies.cpp (L13-17)
```cpp
// Before this pass, values that are semantically i1 and are defined and used
// within the same basic block are already represented as lane masks in scalar
// registers. However, values that cross basic blocks are always transferred
// between basic blocks in vreg_1 virtual registers and are lowered by this
// pass.
```
<a name="block_16"></a>**File:** llvm/lib/Target/AMDGPU/SILowerI1Copies.cpp (L19-20)
```cpp
// The only instructions that use or define vreg_1 virtual registers are COPY,
// PHI, and IMPLICIT_DEF.
```
<a name="block_17"></a>**File:** llvm/lib/Target/AMDGPU/SILowerI1Copies.cpp (L75-82) [<sup>↩</sup>](#ref-block_17)
```cpp
bool Vreg1LoweringHelper::cleanConstrainRegs(bool Changed) {
  assert(Changed || ConstrainRegs.empty());
  for (Register Reg : ConstrainRegs)
    MRI->constrainRegClass(Reg, &AMDGPU::SReg_1_XEXECRegClass);
  ConstrainRegs.clear();

  return Changed;
}
```
<a name="block_18"></a>**File:** llvm/lib/Target/AMDGPU/SILowerI1Copies.cpp (L84-102)
```cpp
/// Helper class that determines the relationship between incoming values of a
/// phi in the control flow graph to determine where an incoming value can
/// simply be taken as a scalar lane mask as-is, and where it needs to be
/// merged with another, previously defined lane mask.
///
/// The approach is as follows:
///  - Determine all basic blocks which, starting from the incoming blocks,
///    a wave may reach before entering the def block (the block containing the
///    phi).
///  - If an incoming block has no predecessors in this set, we can take the
///    incoming value as a scalar lane mask as-is.
///  -- A special case of this is when the def block has a self-loop.
///  - Otherwise, the incoming value needs to be merged with a previously
///    defined lane mask.
///  - If there is a path into the set of reachable blocks that does _not_ go
///    through an incoming block where we can take the scalar lane mask as-is,
///    we need to invent an available value for the SSAUpdater. Choices are
///    0 and undef, with differing consequences for how to merge values etc.
///
```
<a name="block_19"></a>**File:** llvm/lib/Target/AMDGPU/SILowerI1Copies.cpp (L128-177) [<sup>↩</sup>](#ref-block_19)
```cpp
  void analyze(MachineBasicBlock &DefBlock, ArrayRef<Incoming> Incomings) {
    assert(Stack.empty());
    ReachableMap.clear();
    Predecessors.clear();

    // Insert the def block first, so that it acts as an end point for the
    // traversal.
    ReachableMap.try_emplace(&DefBlock, false);

    for (auto Incoming : Incomings) {
      MachineBasicBlock *MBB = Incoming.Block;
      if (MBB == &DefBlock) {
        ReachableMap[&DefBlock] = true; // self-loop on DefBlock
        continue;
      }

      ReachableMap.try_emplace(MBB, false);

      // If this block has a divergent terminator and the def block is its
      // post-dominator, the wave may first visit the other successors.
      if (TII->hasDivergentBranch(MBB) && PDT.dominates(&DefBlock, MBB))
        append_range(Stack, MBB->successors());
    }

    while (!Stack.empty()) {
      MachineBasicBlock *MBB = Stack.pop_back_val();
      if (ReachableMap.try_emplace(MBB, false).second)
        append_range(Stack, MBB->successors());
    }

    for (auto &[MBB, Reachable] : ReachableMap) {
      bool HaveReachablePred = false;
      for (MachineBasicBlock *Pred : MBB->predecessors()) {
        if (ReachableMap.count(Pred)) {
          HaveReachablePred = true;
        } else {
          Stack.push_back(Pred);
        }
      }
      if (!HaveReachablePred)
        Reachable = true;
      if (HaveReachablePred) {
        for (MachineBasicBlock *UnreachablePred : Stack) {
          if (!llvm::is_contained(Predecessors, UnreachablePred))
            Predecessors.push_back(UnreachablePred);
        }
      }
      Stack.clear();
    }
  }
```
<a name="block_20"></a>**File:** llvm/lib/Target/AMDGPU/SILowerI1Copies.cpp (L180-204)
```cpp
/// Helper class that detects loops which require us to lower an i1 COPY into
/// bitwise manipulation.
///
/// Unfortunately, we cannot use LoopInfo because LoopInfo does not distinguish
/// between loops with the same header. Consider this example:
///
///  A-+-+
///  | | |
///  B-+ |
///  |   |
///  C---+
///
/// A is the header of a loop containing A, B, and C as far as LoopInfo is
/// concerned. However, an i1 COPY in B that is used in C must be lowered to
/// bitwise operations to combine results from different loop iterations when
/// B has a divergent branch (since by default we will compile this code such
/// that threads in a wave are merged at the entry of C).
///
/// The following rule is implemented to determine whether bitwise operations
/// are required: use the bitwise lowering for a def in block B if a backward
/// edge to B is reachable without going through the nearest common
/// post-dominator of B and all uses of the def.
///
/// TODO: This rule is conservative because it does not check whether the
///       relevant branches are actually divergent.
```
<a name="block_21"></a>**File:** llvm/lib/Target/AMDGPU/SILowerI1Copies.cpp (L256-273) [<sup>↩</sup>](#ref-block_21)
```cpp
  unsigned findLoop(MachineBasicBlock *PostDom) {
    MachineDomTreeNode *PDNode = PDT.getNode(DefBlock);

    if (!VisitedPostDom)
      advanceLevel();

    unsigned Level = 0;
    while (PDNode->getBlock() != PostDom) {
      if (PDNode->getBlock() == VisitedPostDom)
        advanceLevel();
      PDNode = PDNode->getIDom();
      Level++;
      if (FoundLoopLevel == Level)
        return Level;
    }

    return 0;
  }
```
<a name="block_22"></a>**File:** llvm/lib/Target/AMDGPU/SILowerI1Copies.cpp (L399-411) [<sup>↩</sup>](#ref-block_22)
```cpp
bool Vreg1LoweringHelper::lowerCopiesFromI1() {
  bool Changed = false;
  SmallVector<MachineInstr *, 4> DeadCopies;

  for (MachineBasicBlock &MBB : *MF) {
    for (MachineInstr &MI : MBB) {
      if (MI.getOpcode() != AMDGPU::COPY)
        continue;

      Register DstReg = MI.getOperand(0).getReg();
      Register SrcReg = MI.getOperand(1).getReg();
      if (!isVreg1(SrcReg))
        continue;
```
<a name="block_23"></a>**File:** llvm/lib/Target/AMDGPU/SILowerI1Copies.cpp (L426-431) [<sup>↩</sup>](#ref-block_23)
```cpp
      BuildMI(MBB, MI, DL, TII->get(AMDGPU::V_CNDMASK_B32_e64), DstReg)
          .addImm(0)
          .addImm(0)
          .addImm(0)
          .addImm(-1)
          .addReg(SrcReg);
```
<a name="block_24"></a>**File:** llvm/lib/Target/AMDGPU/SILowerI1Copies.cpp (L452-468) [<sup>↩</sup>](#ref-block_24)
```cpp
  if (IsWave32) {
    ExecReg = AMDGPU::EXEC_LO;
    MovOp = AMDGPU::S_MOV_B32;
    AndOp = AMDGPU::S_AND_B32;
    OrOp = AMDGPU::S_OR_B32;
    XorOp = AMDGPU::S_XOR_B32;
    AndN2Op = AMDGPU::S_ANDN2_B32;
    OrN2Op = AMDGPU::S_ORN2_B32;
  } else {
    ExecReg = AMDGPU::EXEC;
    MovOp = AMDGPU::S_MOV_B64;
    AndOp = AMDGPU::S_AND_B64;
    OrOp = AMDGPU::S_OR_B64;
    XorOp = AMDGPU::S_XOR_B64;
    AndN2Op = AMDGPU::S_ANDN2_B64;
    OrN2Op = AMDGPU::S_ORN2_B64;
  }
```
<a name="block_25"></a>**File:** llvm/lib/Target/AMDGPU/SILowerI1Copies.cpp (L493-495) [<sup>↩</sup>](#ref-block_25)
```cpp
    Register DstReg = MI->getOperand(0).getReg();
    markAsLaneMask(DstReg);
    initializeLaneMaskRegisterAttributes(DstReg);
```
<a name="block_26"></a>**File:** llvm/lib/Target/AMDGPU/SILowerI1Copies.cpp (L497-506) [<sup>↩</sup>](#ref-block_26) [<sup>↩</sup>](#ref-block_26)
```cpp
    collectIncomingValuesFromPhi(MI, Incomings);

    // Sort the incomings such that incoming values that dominate other incoming
    // values are sorted earlier. This allows us to do some amount of on-the-fly
    // constant folding.
    // Incoming with smaller DFSNumIn goes first, DFSNumIn is 0 for entry block.
    llvm::sort(Incomings, [this](Incoming LHS, Incoming RHS) {
      return DT->getNode(LHS.Block)->getDFSNumIn() <
             DT->getNode(RHS.Block)->getDFSNumIn();
    });
```
<a name="block_27"></a>**File:** llvm/lib/Target/AMDGPU/SILowerI1Copies.cpp (L514-525) [<sup>↩</sup>](#ref-block_27)
```cpp
    std::vector<MachineBasicBlock *> DomBlocks = {&MBB};
    for (MachineInstr &Use : MRI->use_instructions(DstReg))
      DomBlocks.push_back(Use.getParent());

    MachineBasicBlock *PostDomBound =
        PDT->findNearestCommonDominator(DomBlocks);

    // FIXME: This fails to find irreducible cycles. If we have a def (other
    // than a constant) in a pair of blocks that end up looping back to each
    // other, it will be mishandle. Due to structurization this shouldn't occur
    // in practice.
    unsigned FoundLoopLevel = LF.findLoop(PostDomBound);
```
<a name="block_28"></a>**File:** llvm/lib/Target/AMDGPU/SILowerI1Copies.cpp (L529-543) [<sup>↩</sup>](#ref-block_28)
```cpp
    if (FoundLoopLevel) {
      LF.addLoopEntries(FoundLoopLevel, SSAUpdater, *MRI, LaneMaskRegAttrs,
                        Incomings);

      for (auto &Incoming : Incomings) {
        Incoming.UpdatedReg = createLaneMaskReg(MRI, LaneMaskRegAttrs);
        SSAUpdater.AddAvailableValue(Incoming.Block, Incoming.UpdatedReg);
      }

      for (auto &Incoming : Incomings) {
        MachineBasicBlock &IMBB = *Incoming.Block;
        buildMergeLaneMasks(
            IMBB, getSaluInsertionAtEnd(IMBB), {}, Incoming.UpdatedReg,
            SSAUpdater.GetValueInMiddleOfBlock(&IMBB), Incoming.Reg);
      }
```
<a name="block_29"></a>**File:** llvm/lib/Target/AMDGPU/SILowerI1Copies.cpp (L544-573) [<sup>↩</sup>](#ref-block_29)
```cpp
    } else {
      // The phi is not observed from outside a loop. Use a more accurate
      // lowering.
      PIA.analyze(MBB, Incomings);

      for (MachineBasicBlock *MBB : PIA.predecessors())
        SSAUpdater.AddAvailableValue(
            MBB, insertUndefLaneMask(MBB, MRI, LaneMaskRegAttrs));

      for (auto &Incoming : Incomings) {
        MachineBasicBlock &IMBB = *Incoming.Block;
        if (PIA.isSource(IMBB)) {
          constrainAsLaneMask(Incoming);
          SSAUpdater.AddAvailableValue(&IMBB, Incoming.Reg);
        } else {
          Incoming.UpdatedReg = createLaneMaskReg(MRI, LaneMaskRegAttrs);
          SSAUpdater.AddAvailableValue(&IMBB, Incoming.UpdatedReg);
        }
      }

      for (auto &Incoming : Incomings) {
        if (!Incoming.UpdatedReg.isValid())
          continue;

        MachineBasicBlock &IMBB = *Incoming.Block;
        buildMergeLaneMasks(
            IMBB, getSaluInsertionAtEnd(IMBB), {}, Incoming.UpdatedReg,
            SSAUpdater.GetValueInMiddleOfBlock(&IMBB), Incoming.Reg);
      }
    }
```
<a name="block_30"></a>**File:** llvm/lib/Target/AMDGPU/SILowerI1Copies.cpp (L586-602) [<sup>↩</sup>](#ref-block_30)
```cpp
bool Vreg1LoweringHelper::lowerCopiesToI1() {
  bool Changed = false;
  MachineSSAUpdater SSAUpdater(*MF);
  LoopFinder LF(*DT, *PDT);
  SmallVector<MachineInstr *, 4> DeadCopies;

  for (MachineBasicBlock &MBB : *MF) {
    LF.initialize(MBB);

    for (MachineInstr &MI : MBB) {
      if (MI.getOpcode() != AMDGPU::IMPLICIT_DEF &&
          MI.getOpcode() != AMDGPU::COPY)
        continue;

      Register DstReg = MI.getOperand(0).getReg();
      if (!isVreg1(DstReg))
        continue;
```
<a name="block_31"></a>**File:** llvm/lib/Target/AMDGPU/SILowerI1Copies.cpp (L623-630) [<sup>↩</sup>](#ref-block_31) [<sup>↩</sup>](#ref-block_31)
```cpp
      if (!SrcReg.isVirtual() || (!isLaneMaskReg(SrcReg) && !isVreg1(SrcReg))) {
        assert(TII->getRegisterInfo().getRegSizeInBits(SrcReg, *MRI) == 32);
        Register TmpReg = createLaneMaskReg(MRI, LaneMaskRegAttrs);
        BuildMI(MBB, MI, DL, TII->get(AMDGPU::V_CMP_NE_U32_e64), TmpReg)
            .addReg(SrcReg)
            .addImm(0);
        MI.getOperand(1).setReg(TmpReg);
        SrcReg = TmpReg;
```
<a name="block_32"></a>**File:** llvm/lib/Target/AMDGPU/SILowerI1Copies.cpp (L637-653) [<sup>↩</sup>](#ref-block_32) [<sup>↩</sup>](#ref-block_32)
```cpp
      // into appropriate bit manipulation.
      std::vector<MachineBasicBlock *> DomBlocks = {&MBB};
      for (MachineInstr &Use : MRI->use_instructions(DstReg))
        DomBlocks.push_back(Use.getParent());

      MachineBasicBlock *PostDomBound =
          PDT->findNearestCommonDominator(DomBlocks);
      unsigned FoundLoopLevel = LF.findLoop(PostDomBound);
      if (FoundLoopLevel) {
        SSAUpdater.Initialize(DstReg);
        SSAUpdater.AddAvailableValue(&MBB, DstReg);
        LF.addLoopEntries(FoundLoopLevel, SSAUpdater, *MRI, LaneMaskRegAttrs);

        buildMergeLaneMasks(MBB, MI, DL, DstReg,
                            SSAUpdater.GetValueInMiddleOfBlock(&MBB), SrcReg);
        DeadCopies.push_back(&MI);
      }
```
<a name="block_33"></a>**File:** llvm/lib/Target/AMDGPU/SILowerI1Copies.cpp (L784-846)
```cpp
void Vreg1LoweringHelper::buildMergeLaneMasks(MachineBasicBlock &MBB,
                                              MachineBasicBlock::iterator I,
                                              const DebugLoc &DL,
                                              Register DstReg, Register PrevReg,
                                              Register CurReg) {
  bool PrevVal = false;
  bool PrevConstant = isConstantLaneMask(PrevReg, PrevVal);
  bool CurVal = false;
  bool CurConstant = isConstantLaneMask(CurReg, CurVal);

  if (PrevConstant && CurConstant) {
    if (PrevVal == CurVal) {
      BuildMI(MBB, I, DL, TII->get(AMDGPU::COPY), DstReg).addReg(CurReg);
    } else if (CurVal) {
      BuildMI(MBB, I, DL, TII->get(AMDGPU::COPY), DstReg).addReg(ExecReg);
    } else {
      BuildMI(MBB, I, DL, TII->get(XorOp), DstReg)
          .addReg(ExecReg)
          .addImm(-1);
    }
    return;
  }

  Register PrevMaskedReg;
  Register CurMaskedReg;
  if (!PrevConstant) {
    if (CurConstant && CurVal) {
      PrevMaskedReg = PrevReg;
    } else {
      PrevMaskedReg = createLaneMaskReg(MRI, LaneMaskRegAttrs);
      BuildMI(MBB, I, DL, TII->get(AndN2Op), PrevMaskedReg)
          .addReg(PrevReg)
          .addReg(ExecReg);
    }
  }
  if (!CurConstant) {
    // TODO: check whether CurReg is already masked by EXEC
    if (PrevConstant && PrevVal) {
      CurMaskedReg = CurReg;
    } else {
      CurMaskedReg = createLaneMaskReg(MRI, LaneMaskRegAttrs);
      BuildMI(MBB, I, DL, TII->get(AndOp), CurMaskedReg)
          .addReg(CurReg)
          .addReg(ExecReg);
    }
  }

  if (PrevConstant && !PrevVal) {
    BuildMI(MBB, I, DL, TII->get(AMDGPU::COPY), DstReg)
        .addReg(CurMaskedReg);
  } else if (CurConstant && !CurVal) {
    BuildMI(MBB, I, DL, TII->get(AMDGPU::COPY), DstReg)
        .addReg(PrevMaskedReg);
  } else if (PrevConstant && PrevVal) {
    BuildMI(MBB, I, DL, TII->get(OrN2Op), DstReg)
        .addReg(CurMaskedReg)
        .addReg(ExecReg);
  } else {
    BuildMI(MBB, I, DL, TII->get(OrOp), DstReg)
        .addReg(PrevMaskedReg)
        .addReg(CurMaskedReg ? CurMaskedReg : ExecReg);
  }
}
```
<a name="block_34"></a>**File:** llvm/lib/Target/AMDGPU/SILowerI1Copies.cpp (L851-854) [<sup>↩</sup>](#ref-block_34)
```cpp
///
/// In a first pass, we lower COPYs from vreg_1 to vector registers, as can
/// occur around inline assembly. We do this first, before vreg_1 registers
/// are changed to scalar mask registers.
```
<a name="block_35"></a>**File:** llvm/lib/Target/AMDGPU/SILowerI1Copies.cpp (L856-858)
```cpp
/// Then we lower all defs of vreg_1 registers. Phi nodes are lowered before
/// all others, because phi lowering looks through copies and can therefore
/// often make copy lowering unnecessary.
```
<a name="block_36"></a>**File:** llvm/lib/Target/AMDGPU/SILowerI1Copies.cpp (L859-871) [<sup>↩</sup>](#ref-block_36)
```cpp
static bool runFixI1Copies(MachineFunction &MF, MachineDominatorTree &MDT,
                           MachinePostDominatorTree &MPDT) {
  // Only need to run this in SelectionDAG path.
  if (MF.getProperties().hasSelected())
    return false;

  Vreg1LoweringHelper Helper(&MF, &MDT, &MPDT);
  bool Changed = false;
  Changed |= Helper.lowerCopiesFromI1();
  Changed |= Helper.lowerPhis();
  Changed |= Helper.lowerCopiesToI1();
  return Helper.cleanConstrainRegs(Changed);
}
```