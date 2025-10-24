# SIFoldOperands.cpp 代码功能详解

## 1. Pass的主要功能概述

<a name="ref-block_0"></a>`SIFoldOperands` 是AMDGPU后端的一个机器指令优化Pass，其**主要功能是将操作数折叠（fold）到使用它们的指令中**，从而减少中间寄存器的使用，优化代码质量。 llvm-project:288-289[<sup>↗</sup>](#block_0) 

该Pass的**核心作用**包括：
- 将立即数、帧索引、全局地址等直接折叠到使用指令中
- 消除冗余的COPY指令
- 优化寄存器类之间的转换（如SGPR到VGPR，AGPR到VGPR）
- 应用特殊的指令优化模式（如clamp、omod）

<a name="ref-block_28"></a>**主要效果**：减少寄存器压力，减少指令数量，提高执行效率。 llvm-project:2726-2786[<sup>↗</sup>](#block_28) 

## 2. 主要功能步骤/子功能提取

通过遍历代码文件，可以提取以下主要子功能：

### 核心折叠功能
- `foldInstOperand` - 指令操作数折叠的主入口
- `foldOperand` - 将可折叠定义折叠到使用指令
- `tryAddToFoldList` - 将折叠候选添加到列表
- `updateOperand` - 实际更新操作数

### 特定模式折叠
- `tryFoldFoldableCopy` - 折叠可折叠的COPY指令
- `tryConstantFoldOp` - 常量折叠优化
- `tryFoldCndMask` - 折叠条件掩码指令
- `tryFoldZeroHighBits` - 折叠高位为零的模式
- `tryFoldClamp` - 折叠clamp操作
- `tryFoldOMod` - 折叠输出修饰符

### AGPR相关优化
- `tryFoldPhiAGPR` - 提升AGPR到VGPR的拷贝跨越PHI节点
- `tryFoldLoad` - 将VGPR加载转换为AGPR加载
- `foldCopyToAGPRRegSequence` - 折叠到AGPR寄存器序列的拷贝
- `tryOptimizeAGPRPhis` - 优化AGPR PHI节点

### 寄存器序列处理
- `tryFoldRegSequence` - 折叠寄存器序列
- `getRegSeqInit` - 获取寄存器序列的初始化器
- `isRegSeqSplat` - 检查寄存器序列是否为splat模式
- `tryFoldRegSeqSplat` - 折叠splat寄存器序列

### 立即数和特殊操作数处理
- `tryFoldImmWithOpSel` - 使用op_sel折叠立即数
- `canUseImmWithOpSel` - 检查是否可以使用op_sel折叠立即数
- `tryToFoldACImm` - 折叠累加器立即数
- `frameIndexMayFold` - 检查帧索引是否可折叠

### 辅助功能
- `isUseSafeToFold` - 检查折叠是否安全
- `isClamp` - 识别clamp模式
- `isOMod` - 识别输出修饰符模式
- `getImmOrMaterializedImm` - 获取立即数或物化的立即数

## 3. 各子功能的具体描述分析

### 3.1 核心折叠机制

<a name="ref-block_17"></a>**`foldInstOperand`** 是操作数折叠的主入口函数： llvm-project:1722-1801[<sup>↗</sup>](#block_17) 

该函数遍历目标寄存器的所有使用，尝试将可折叠的定义折叠到这些使用中。它首先尝试常量折叠，然后处理每个使用，最后更新所有成功的折叠候选。

<a name="ref-block_12"></a>**`foldOperand`** 处理单个操作数的折叠： llvm-project:1137-1445[<sup>↗</sup>](#block_12) 

它处理多种特殊情况：REG_SEQUENCE指令、帧索引折叠、COPY指令转换、以及一般的操作数折叠。

<a name="ref-block_6"></a>**`tryAddToFoldList`** 验证折叠的合法性： llvm-project:740-895[<sup>↗</sup>](#block_6) 

该函数检查操作数是否可以合法折叠到目标指令，包括特殊处理MAC到MAD转换、FMAC到FMAAK/FMAMK转换、以及指令交换以实现折叠。

<a name="ref-block_5"></a>**`updateOperand`** 执行实际的操作数更新： llvm-project:613-719[<sup>↗</sup>](#block_5) 

它处理立即数、帧索引、全局地址和寄存器的更新，还处理指令收缩（shrinking）以满足VCC活跃性要求。

### 3.2 特定模式折叠

<a name="ref-block_19"></a>**`tryFoldFoldableCopy`** 处理可折叠的COPY指令： llvm-project:1935-2050[<sup>↗</sup>](#block_19) 

该函数特别处理M0寄存器的重定义、COPY到MOV的转换、以及标量加法帧索引的特殊折叠。

<a name="ref-block_14"></a>**`tryConstantFoldOp`** 实现常量折叠： llvm-project:1552-1655[<sup>↗</sup>](#block_14) 

它识别可以在编译时计算的操作（如AND、OR、XOR、位移等），并将它们替换为立即数或COPY指令。

<a name="ref-block_15"></a>**`tryFoldCndMask`** 简化条件掩码指令： llvm-project:1658-1698[<sup>↗</sup>](#block_15) 

当V_CNDMASK的两个源操作数相同时，将其转换为简单的COPY或MOV指令。

<a name="ref-block_16"></a>**`tryFoldZeroHighBits`** 优化高位清零模式： llvm-project:1700-1720[<sup>↗</sup>](#block_16) 

识别`V_AND_B32 x, 0xffff`模式，如果源指令已经将高16位清零，则消除AND操作。

<a name="ref-block_21"></a>**`tryFoldClamp`** 折叠clamp操作： llvm-project:2103-2152[<sup>↗</sup>](#block_21) 

将独立的clamp指令（v_max with clamp）折叠到产生值的源指令中。

<a name="ref-block_23"></a>**`tryFoldOMod`** 折叠输出修饰符： llvm-project:2282-2320[<sup>↗</sup>](#block_23) 

识别乘以0.5、2.0或4.0的模式，将其折叠为源指令的omod字段。

### 3.3 AGPR优化

<a name="ref-block_25"></a>**`tryFoldPhiAGPR`** 提升AGPR拷贝： llvm-project:2471-2572[<sup>↗</sup>](#block_25) 

将AGPR到VGPR的拷贝提升到PHI节点之后，允许后续折叠优化。这对于循环中的AGPR使用特别重要。

<a name="ref-block_26"></a>**`tryFoldLoad`** 转换加载指令： llvm-project:2575-2627[<sup>↗</sup>](#block_26) 

当VGPR加载的所有使用最终都到达AGPR时，直接将加载目标改为AGPR类，避免额外的拷贝。

<a name="ref-block_18"></a>**`foldCopyToAGPRRegSequence`** 优化AGPR寄存器序列： llvm-project:1805-1933[<sup>↗</sup>](#block_18) 

将`COPY (REG_SEQUENCE)`模式重写为直接的REG_SEQUENCE with AGPR输入，使用V_ACCVGPR_WRITE处理立即数和SGPR。

<a name="ref-block_27"></a>**`tryOptimizeAGPRPhis`** 缓存AGPR PHI操作数： llvm-project:2661-2724[<sup>↗</sup>](#block_27) 

对于GFX908（没有V_ACCVGPR_MOV），识别多个PHI共享的AGPR子寄存器，并缓存到VGPR中以减少昂贵的AGPR-AGPR拷贝。

### 3.4 寄存器序列处理

<a name="ref-block_24"></a>**`tryFoldRegSequence`** 折叠寄存器序列到指令： llvm-project:2324-2400[<sup>↗</sup>](#block_24) 

尝试将VGPR REG_SEQUENCE（包含AGPR输入）直接折叠到可以接受AGPR的使用指令中。

<a name="ref-block_8"></a>**`getRegSeqInit`** 追踪寄存器序列来源： llvm-project:924-972[<sup>↗</sup>](#block_8) 

递归追踪REG_SEQUENCE的每个输入，通过COPY链查找最终的定义。

<a name="ref-block_9"></a>**`isRegSeqSplat`** 识别splat模式： llvm-project:974-1040[<sup>↗</sup>](#block_9) 

检查REG_SEQUENCE是否所有元素都是相同的立即数值（包括32位和64位splat）。

<a name="ref-block_10"></a>**`tryFoldRegSeqSplat`** 折叠splat立即数： llvm-project:1042-1088[<sup>↗</sup>](#block_10) 

将splat REG_SEQUENCE折叠为单个立即数操作数，适用于支持accumulator寄存器的指令。

### 3.5 立即数处理

<a name="ref-block_4"></a>**`tryFoldImmWithOpSel`** 使用op_sel优化立即数： llvm-project:490-611[<sup>↗</sup>](#block_4) 

对于packed指令，通过调整op_sel位来使非内联立即数变成可内联的，或者在add/sub之间转换以实现内联。

<a name="ref-block_3"></a>**`canUseImmWithOpSel`** 检查op_sel可用性： llvm-project:455-488[<sup>↗</sup>](#block_3) 

验证指令是否支持带op_sel的立即数折叠，排除不支持的指令类型（MAI、WMMA、DOT等）。

<a name="ref-block_11"></a>**`tryToFoldACImm`** 折叠累加器立即数： llvm-project:1090-1135[<sup>↗</sup>](#block_11) 

尝试将立即数折叠到接受内联常量的累加器操作数位置。

### 3.6 帧索引处理

<a name="ref-block_1"></a>**`frameIndexMayFold`** 判断帧索引折叠可行性： llvm-project:344-379[<sup>↗</sup>](#block_1) 

检查帧索引是否可以折叠到特定指令中，主要针对标量加法、MUBUF和FLAT_SCRATCH指令。

<a name="ref-block_2"></a>**`foldCopyToVGPROfScalarAddOfFrameIndex`** 优化帧索引加法： llvm-project:384-449[<sup>↗</sup>](#block_2) 

将`COPY (S_ADD frameindex, x)`转换为向量加法`V_ADD frameindex, x`，避免通过标量寄存器。

### 3.7 辅助和验证功能

<a name="ref-block_7"></a>**`isUseSafeToFold`** 检查折叠安全性： llvm-project:897-901[<sup>↗</sup>](#block_7) 

验证操作数是否可以安全折叠，特别是SDWA指令要求寄存器操作数。

<a name="ref-block_20"></a>**`isClamp`** 识别clamp模式： llvm-project:2054-2100[<sup>↗</sup>](#block_20) 

识别`v_max(x, x) with clamp`模式，这是规范化的clamp表示。

<a name="ref-block_22"></a>**`isOMod`** 识别输出修饰符模式： llvm-project:2203-2279[<sup>↗</sup>](#block_22) 

识别乘法或加法指令可以用omod表示的模式。

<a name="ref-block_13"></a>**`getImmOrMaterializedImm`** 获取实际立即数值： llvm-project:1531-1547[<sup>↗</sup>](#block_13) 

从操作数或其定义的MOV指令中提取立即数值。

## 4. 步骤/子功能之间的关系

### 4.1 主控制流关系

Pass的执行从`run`函数开始，它按深度优先顺序遍历基本块： llvm-project:2741-2783 

在每个基本块中，按顺序尝试不同的优化：

```
run()
├─> tryFoldCndMask()           [简化条件掩码]
├─> tryFoldZeroHighBits()      [消除冗余AND]
├─> tryFoldRegSequence()       [折叠寄存器序列]
├─> tryFoldPhiAGPR()          [优化AGPR PHI]
├─> tryFoldLoad()              [转换加载指令]
├─> tryFoldFoldableCopy()      [折叠COPY指令]
│   └─> foldInstOperand()      [主折叠逻辑]
│       └─> foldOperand()      [处理单个操作数]
│           └─> tryAddToFoldList() [验证并添加候选]
├─> tryFoldOMod()              [折叠输出修饰符]
└─> tryFoldClamp()             [折叠clamp]
    └─> tryOptimizeAGPRPhis()  [优化AGPR PHI]
```

### 4.2 折叠候选处理流程

折叠操作遵循以下流程：

1. **识别阶段**：`tryFoldFoldableCopy`识别可折叠的COPY指令
2. **候选收集**：`foldOperand`遍历使用，调用`tryAddToFoldList`收集候选
3. **合法性检查**：`tryAddToFoldList`验证折叠合法性，可能尝试指令交换
4. **执行更新**：`updateOperand`实际修改指令操作数
5. **后续优化**：`tryConstantFoldOp`可能进一步简化结果

关系链： llvm-project:2014-2015 

### 4.3 特殊模式识别的层次关系

特殊模式优化形成优先级关系：

- **高优先级**：`tryFoldCndMask`、`tryFoldZeroHighBits`（先执行，更简单）
- **中优先级**：`tryFoldFoldableCopy`、`foldInstOperand`（主要折叠逻辑）
- **低优先级**：`tryFoldClamp`、`tryFoldOMod`（更复杂的模式，可能依赖前面的折叠结果）

`isClamp`和`isOMod`作为识别辅助函数，被对应的折叠函数调用： llvm-project:2104-2105 llvm-project:2283-2285 

### 4.4 AGPR优化的协同关系

AGPR相关优化相互配合：

1. **`tryFoldPhiAGPR`**：提升拷贝，创建AGPR PHI
2. **`tryOptimizeAGPRPhis`**：缓存共享的AGPR操作数（仅GFX908）
3. **`tryFoldLoad`**：直接加载到AGPR
4. **`foldCopyToAGPRRegSequence`**：优化到AGPR的批量拷贝

这些优化共同目标是减少AGPR-VGPR-AGPR的往返拷贝： llvm-project:2662-2665 

### 4.5 寄存器序列处理的依赖关系

寄存器序列处理存在调用依赖：

```
tryFoldRegSequence()
└─> getRegSeqInit()
    └─> lookUpCopyChain()

foldOperand() [for REG_SEQUENCE]
├─> isRegSeqSplat()
│   └─> getRegSeqInit()
└─> tryFoldRegSeqSplat()
``` llvm-project:1160-1194 

### 4.6 立即数折叠的层次关系

立即数折叠有多个层次：

1. **直接折叠**：`tryAddToFoldList`检查`isOperandLegal`
2. **op_sel优化**：`canUseImmWithOpSel` + `tryFoldImmWithOpSel`
3. **累加器折叠**：`tryToFoldACImm`
4. **常量折叠**：`tryConstantFoldOp`（折叠后的后处理） llvm-project:779-783 

### 4.7 帧索引处理的特殊路径

帧索引有专门的处理路径：

```
foldOperand()
├─> frameIndexMayFold()        [检查可行性]
└─> [直接修改为FrameIndex]

tryFoldFoldableCopy()
└─> foldCopyToVGPROfScalarAddOfFrameIndex() [特殊优化]
``` llvm-project:1199-1228 

## Notes

该Pass是AMDGPU后端优化的关键组成部分，它在寄存器分配后运行，利用机器指令的特定编码能力（如内联常量、op_sel、omod、clamp等）来优化代码。代码针对不同GPU架构（GFX908、GFX90A等）的特性进行了特殊处理，特别是AGPR（累加器寄存器）的管理。整体设计采用多遍扫描和模式匹配的策略，既保证了优化的正确性，又最大化了优化机会。
### Citations
<a name="block_0"></a>**File:** llvm/lib/Target/AMDGPU/SIFoldOperands.cpp (L288-289) [<sup>↩</sup>](#ref-block_0)
```cpp
  StringRef getPassName() const override { return "SI Fold Operands"; }

```
<a name="block_1"></a>**File:** llvm/lib/Target/AMDGPU/SIFoldOperands.cpp (L344-379) [<sup>↩</sup>](#ref-block_1)
```cpp
bool SIFoldOperandsImpl::frameIndexMayFold(const MachineInstr &UseMI, int OpNo,
                                           const FoldableDef &OpToFold) const {
  if (!OpToFold.isFI())
    return false;

  const unsigned Opc = UseMI.getOpcode();
  switch (Opc) {
  case AMDGPU::S_ADD_I32:
  case AMDGPU::S_ADD_U32:
  case AMDGPU::V_ADD_U32_e32:
  case AMDGPU::V_ADD_CO_U32_e32:
    // TODO: Possibly relax hasOneUse. It matters more for mubuf, since we have
    // to insert the wave size shift at every point we use the index.
    // TODO: Fix depending on visit order to fold immediates into the operand
    return UseMI.getOperand(OpNo == 1 ? 2 : 1).isImm() &&
           MRI->hasOneNonDBGUse(UseMI.getOperand(OpNo).getReg());
  case AMDGPU::V_ADD_U32_e64:
  case AMDGPU::V_ADD_CO_U32_e64:
    return UseMI.getOperand(OpNo == 2 ? 3 : 2).isImm() &&
           MRI->hasOneNonDBGUse(UseMI.getOperand(OpNo).getReg());
  default:
    break;
  }

  if (TII->isMUBUF(UseMI))
    return OpNo == AMDGPU::getNamedOperandIdx(Opc, AMDGPU::OpName::vaddr);
  if (!TII->isFLATScratch(UseMI))
    return false;

  int SIdx = AMDGPU::getNamedOperandIdx(Opc, AMDGPU::OpName::saddr);
  if (OpNo == SIdx)
    return true;

  int VIdx = AMDGPU::getNamedOperandIdx(Opc, AMDGPU::OpName::vaddr);
  return OpNo == VIdx && SIdx == -1;
}
```
<a name="block_2"></a>**File:** llvm/lib/Target/AMDGPU/SIFoldOperands.cpp (L384-449) [<sup>↩</sup>](#ref-block_2) [<sup>↩</sup>](#ref-block_2)
```cpp
bool SIFoldOperandsImpl::foldCopyToVGPROfScalarAddOfFrameIndex(
    Register DstReg, Register SrcReg, MachineInstr &MI) const {
  if (TRI->isVGPR(*MRI, DstReg) && TRI->isSGPRReg(*MRI, SrcReg) &&
      MRI->hasOneNonDBGUse(SrcReg)) {
    MachineInstr *Def = MRI->getVRegDef(SrcReg);
    if (!Def || Def->getNumOperands() != 4)
      return false;

    MachineOperand *Src0 = &Def->getOperand(1);
    MachineOperand *Src1 = &Def->getOperand(2);

    // TODO: This is profitable with more operand types, and for more
    // opcodes. But ultimately this is working around poor / nonexistent
    // regbankselect.
    if (!Src0->isFI() && !Src1->isFI())
      return false;

    if (Src0->isFI())
      std::swap(Src0, Src1);

    const bool UseVOP3 = !Src0->isImm() || TII->isInlineConstant(*Src0);
    unsigned NewOp = convertToVALUOp(Def->getOpcode(), UseVOP3);
    if (NewOp == AMDGPU::INSTRUCTION_LIST_END ||
        !Def->getOperand(3).isDead()) // Check if scc is dead
      return false;

    MachineBasicBlock *MBB = Def->getParent();
    const DebugLoc &DL = Def->getDebugLoc();
    if (NewOp != AMDGPU::V_ADD_CO_U32_e32) {
      MachineInstrBuilder Add =
          BuildMI(*MBB, *Def, DL, TII->get(NewOp), DstReg);

      if (Add->getDesc().getNumDefs() == 2) {
        Register CarryOutReg = MRI->createVirtualRegister(TRI->getBoolRC());
        Add.addDef(CarryOutReg, RegState::Dead);
        MRI->setRegAllocationHint(CarryOutReg, 0, TRI->getVCC());
      }

      Add.add(*Src0).add(*Src1).setMIFlags(Def->getFlags());
      if (AMDGPU::hasNamedOperand(NewOp, AMDGPU::OpName::clamp))
        Add.addImm(0);

      Def->eraseFromParent();
      MI.eraseFromParent();
      return true;
    }

    assert(NewOp == AMDGPU::V_ADD_CO_U32_e32);

    MachineBasicBlock::LivenessQueryResult Liveness =
        MBB->computeRegisterLiveness(TRI, AMDGPU::VCC, *Def, 16);
    if (Liveness == MachineBasicBlock::LQR_Dead) {
      // TODO: If src1 satisfies operand constraints, use vop3 version.
      BuildMI(*MBB, *Def, DL, TII->get(NewOp), DstReg)
          .add(*Src0)
          .add(*Src1)
          .setOperandDead(3) // implicit-def $vcc
          .setMIFlags(Def->getFlags());
      Def->eraseFromParent();
      MI.eraseFromParent();
      return true;
    }
  }

  return false;
}
```
<a name="block_3"></a>**File:** llvm/lib/Target/AMDGPU/SIFoldOperands.cpp (L455-488) [<sup>↩</sup>](#ref-block_3)
```cpp
bool SIFoldOperandsImpl::canUseImmWithOpSel(const MachineInstr *MI,
                                            unsigned UseOpNo,
                                            int64_t ImmVal) const {
  const uint64_t TSFlags = MI->getDesc().TSFlags;

  if (!(TSFlags & SIInstrFlags::IsPacked) || (TSFlags & SIInstrFlags::IsMAI) ||
      (TSFlags & SIInstrFlags::IsWMMA) || (TSFlags & SIInstrFlags::IsSWMMAC) ||
      (ST->hasDOTOpSelHazard() && (TSFlags & SIInstrFlags::IsDOT)))
    return false;

  const MachineOperand &Old = MI->getOperand(UseOpNo);
  int OpNo = MI->getOperandNo(&Old);

  unsigned Opcode = MI->getOpcode();
  uint8_t OpType = TII->get(Opcode).operands()[OpNo].OperandType;
  switch (OpType) {
  default:
    return false;
  case AMDGPU::OPERAND_REG_IMM_V2FP16:
  case AMDGPU::OPERAND_REG_IMM_V2BF16:
  case AMDGPU::OPERAND_REG_IMM_V2INT16:
  case AMDGPU::OPERAND_REG_INLINE_C_V2FP16:
  case AMDGPU::OPERAND_REG_INLINE_C_V2BF16:
  case AMDGPU::OPERAND_REG_INLINE_C_V2INT16:
    // VOP3 packed instructions ignore op_sel source modifiers, we cannot encode
    // two different constants.
    if ((TSFlags & SIInstrFlags::VOP3) && !(TSFlags & SIInstrFlags::VOP3P) &&
        static_cast<uint16_t>(ImmVal) != static_cast<uint16_t>(ImmVal >> 16))
      return false;
    break;
  }

  return true;
}
```
<a name="block_4"></a>**File:** llvm/lib/Target/AMDGPU/SIFoldOperands.cpp (L490-611) [<sup>↩</sup>](#ref-block_4)
```cpp
bool SIFoldOperandsImpl::tryFoldImmWithOpSel(MachineInstr *MI, unsigned UseOpNo,
                                             int64_t ImmVal) const {
  MachineOperand &Old = MI->getOperand(UseOpNo);
  unsigned Opcode = MI->getOpcode();
  int OpNo = MI->getOperandNo(&Old);
  uint8_t OpType = TII->get(Opcode).operands()[OpNo].OperandType;

  // If the literal can be inlined as-is, apply it and short-circuit the
  // tests below. The main motivation for this is to avoid unintuitive
  // uses of opsel.
  if (AMDGPU::isInlinableLiteralV216(ImmVal, OpType)) {
    Old.ChangeToImmediate(ImmVal);
    return true;
  }

  // Refer to op_sel/op_sel_hi and check if we can change the immediate and
  // op_sel in a way that allows an inline constant.
  AMDGPU::OpName ModName = AMDGPU::OpName::NUM_OPERAND_NAMES;
  unsigned SrcIdx = ~0;
  if (OpNo == AMDGPU::getNamedOperandIdx(Opcode, AMDGPU::OpName::src0)) {
    ModName = AMDGPU::OpName::src0_modifiers;
    SrcIdx = 0;
  } else if (OpNo == AMDGPU::getNamedOperandIdx(Opcode, AMDGPU::OpName::src1)) {
    ModName = AMDGPU::OpName::src1_modifiers;
    SrcIdx = 1;
  } else if (OpNo == AMDGPU::getNamedOperandIdx(Opcode, AMDGPU::OpName::src2)) {
    ModName = AMDGPU::OpName::src2_modifiers;
    SrcIdx = 2;
  }
  assert(ModName != AMDGPU::OpName::NUM_OPERAND_NAMES);
  int ModIdx = AMDGPU::getNamedOperandIdx(Opcode, ModName);
  MachineOperand &Mod = MI->getOperand(ModIdx);
  unsigned ModVal = Mod.getImm();

  uint16_t ImmLo =
      static_cast<uint16_t>(ImmVal >> (ModVal & SISrcMods::OP_SEL_0 ? 16 : 0));
  uint16_t ImmHi =
      static_cast<uint16_t>(ImmVal >> (ModVal & SISrcMods::OP_SEL_1 ? 16 : 0));
  uint32_t Imm = (static_cast<uint32_t>(ImmHi) << 16) | ImmLo;
  unsigned NewModVal = ModVal & ~(SISrcMods::OP_SEL_0 | SISrcMods::OP_SEL_1);

  // Helper function that attempts to inline the given value with a newly
  // chosen opsel pattern.
  auto tryFoldToInline = [&](uint32_t Imm) -> bool {
    if (AMDGPU::isInlinableLiteralV216(Imm, OpType)) {
      Mod.setImm(NewModVal | SISrcMods::OP_SEL_1);
      Old.ChangeToImmediate(Imm);
      return true;
    }

    // Try to shuffle the halves around and leverage opsel to get an inline
    // constant.
    uint16_t Lo = static_cast<uint16_t>(Imm);
    uint16_t Hi = static_cast<uint16_t>(Imm >> 16);
    if (Lo == Hi) {
      if (AMDGPU::isInlinableLiteralV216(Lo, OpType)) {
        Mod.setImm(NewModVal);
        Old.ChangeToImmediate(Lo);
        return true;
      }

      if (static_cast<int16_t>(Lo) < 0) {
        int32_t SExt = static_cast<int16_t>(Lo);
        if (AMDGPU::isInlinableLiteralV216(SExt, OpType)) {
          Mod.setImm(NewModVal);
          Old.ChangeToImmediate(SExt);
          return true;
        }
      }

      // This check is only useful for integer instructions
      if (OpType == AMDGPU::OPERAND_REG_IMM_V2INT16) {
        if (AMDGPU::isInlinableLiteralV216(Lo << 16, OpType)) {
          Mod.setImm(NewModVal | SISrcMods::OP_SEL_0 | SISrcMods::OP_SEL_1);
          Old.ChangeToImmediate(static_cast<uint32_t>(Lo) << 16);
          return true;
        }
      }
    } else {
      uint32_t Swapped = (static_cast<uint32_t>(Lo) << 16) | Hi;
      if (AMDGPU::isInlinableLiteralV216(Swapped, OpType)) {
        Mod.setImm(NewModVal | SISrcMods::OP_SEL_0);
        Old.ChangeToImmediate(Swapped);
        return true;
      }
    }

    return false;
  };

  if (tryFoldToInline(Imm))
    return true;

  // Replace integer addition by subtraction and vice versa if it allows
  // folding the immediate to an inline constant.
  //
  // We should only ever get here for SrcIdx == 1 due to canonicalization
  // earlier in the pipeline, but we double-check here to be safe / fully
  // general.
  bool IsUAdd = Opcode == AMDGPU::V_PK_ADD_U16;
  bool IsUSub = Opcode == AMDGPU::V_PK_SUB_U16;
  if (SrcIdx == 1 && (IsUAdd || IsUSub)) {
    unsigned ClampIdx =
        AMDGPU::getNamedOperandIdx(Opcode, AMDGPU::OpName::clamp);
    bool Clamp = MI->getOperand(ClampIdx).getImm() != 0;

    if (!Clamp) {
      uint16_t NegLo = -static_cast<uint16_t>(Imm);
      uint16_t NegHi = -static_cast<uint16_t>(Imm >> 16);
      uint32_t NegImm = (static_cast<uint32_t>(NegHi) << 16) | NegLo;

      if (tryFoldToInline(NegImm)) {
        unsigned NegOpcode =
            IsUAdd ? AMDGPU::V_PK_SUB_U16 : AMDGPU::V_PK_ADD_U16;
        MI->setDesc(TII->get(NegOpcode));
        return true;
      }
    }
  }

  return false;
}
```
<a name="block_5"></a>**File:** llvm/lib/Target/AMDGPU/SIFoldOperands.cpp (L613-719) [<sup>↩</sup>](#ref-block_5)
```cpp
bool SIFoldOperandsImpl::updateOperand(FoldCandidate &Fold) const {
  MachineInstr *MI = Fold.UseMI;
  MachineOperand &Old = MI->getOperand(Fold.UseOpNo);
  assert(Old.isReg());

  std::optional<int64_t> ImmVal;
  if (Fold.isImm())
    ImmVal = Fold.Def.getEffectiveImmVal();

  if (ImmVal && canUseImmWithOpSel(Fold.UseMI, Fold.UseOpNo, *ImmVal)) {
    if (tryFoldImmWithOpSel(Fold.UseMI, Fold.UseOpNo, *ImmVal))
      return true;

    // We can't represent the candidate as an inline constant. Try as a literal
    // with the original opsel, checking constant bus limitations.
    MachineOperand New = MachineOperand::CreateImm(*ImmVal);
    int OpNo = MI->getOperandNo(&Old);
    if (!TII->isOperandLegal(*MI, OpNo, &New))
      return false;
    Old.ChangeToImmediate(*ImmVal);
    return true;
  }

  if ((Fold.isImm() || Fold.isFI() || Fold.isGlobal()) && Fold.needsShrink()) {
    MachineBasicBlock *MBB = MI->getParent();
    auto Liveness = MBB->computeRegisterLiveness(TRI, AMDGPU::VCC, MI, 16);
    if (Liveness != MachineBasicBlock::LQR_Dead) {
      LLVM_DEBUG(dbgs() << "Not shrinking " << MI << " due to vcc liveness\n");
      return false;
    }

    int Op32 = Fold.ShrinkOpcode;
    MachineOperand &Dst0 = MI->getOperand(0);
    MachineOperand &Dst1 = MI->getOperand(1);
    assert(Dst0.isDef() && Dst1.isDef());

    bool HaveNonDbgCarryUse = !MRI->use_nodbg_empty(Dst1.getReg());

    const TargetRegisterClass *Dst0RC = MRI->getRegClass(Dst0.getReg());
    Register NewReg0 = MRI->createVirtualRegister(Dst0RC);

    MachineInstr *Inst32 = TII->buildShrunkInst(*MI, Op32);

    if (HaveNonDbgCarryUse) {
      BuildMI(*MBB, MI, MI->getDebugLoc(), TII->get(AMDGPU::COPY),
              Dst1.getReg())
        .addReg(AMDGPU::VCC, RegState::Kill);
    }

    // Keep the old instruction around to avoid breaking iterators, but
    // replace it with a dummy instruction to remove uses.
    //
    // FIXME: We should not invert how this pass looks at operands to avoid
    // this. Should track set of foldable movs instead of looking for uses
    // when looking at a use.
    Dst0.setReg(NewReg0);
    for (unsigned I = MI->getNumOperands() - 1; I > 0; --I)
      MI->removeOperand(I);
    MI->setDesc(TII->get(AMDGPU::IMPLICIT_DEF));

    if (Fold.Commuted)
      TII->commuteInstruction(*Inst32, false);
    return true;
  }

  assert(!Fold.needsShrink() && "not handled");

  if (ImmVal) {
    if (Old.isTied()) {
      int NewMFMAOpc = AMDGPU::getMFMAEarlyClobberOp(MI->getOpcode());
      if (NewMFMAOpc == -1)
        return false;
      MI->setDesc(TII->get(NewMFMAOpc));
      MI->untieRegOperand(0);
    }

    // TODO: Should we try to avoid adding this to the candidate list?
    MachineOperand New = MachineOperand::CreateImm(*ImmVal);
    int OpNo = MI->getOperandNo(&Old);
    if (!TII->isOperandLegal(*MI, OpNo, &New))
      return false;

    Old.ChangeToImmediate(*ImmVal);
    return true;
  }

  if (Fold.isGlobal()) {
    Old.ChangeToGA(Fold.Def.OpToFold->getGlobal(),
                   Fold.Def.OpToFold->getOffset(),
                   Fold.Def.OpToFold->getTargetFlags());
    return true;
  }

  if (Fold.isFI()) {
    Old.ChangeToFrameIndex(Fold.getFI());
    return true;
  }

  MachineOperand *New = Fold.Def.OpToFold;
  // Rework once the VS_16 register class is updated to include proper
  // 16-bit SGPRs instead of 32-bit ones.
  if (Old.getSubReg() == AMDGPU::lo16 && TRI->isSGPRReg(*MRI, New->getReg()))
    Old.setSubReg(AMDGPU::NoSubRegister);
  Old.substVirtReg(New->getReg(), New->getSubReg(), *TRI);
  Old.setIsUndef(New->isUndef());
  return true;
}
```
<a name="block_6"></a>**File:** llvm/lib/Target/AMDGPU/SIFoldOperands.cpp (L740-895) [<sup>↩</sup>](#ref-block_6) [<sup>↩</sup>](#ref-block_6)
```cpp
bool SIFoldOperandsImpl::tryAddToFoldList(
    SmallVectorImpl<FoldCandidate> &FoldList, MachineInstr *MI, unsigned OpNo,
    const FoldableDef &OpToFold) const {
  const unsigned Opc = MI->getOpcode();

  auto tryToFoldAsFMAAKorMK = [&]() {
    if (!OpToFold.isImm())
      return false;

    const bool TryAK = OpNo == 3;
    const unsigned NewOpc = TryAK ? AMDGPU::S_FMAAK_F32 : AMDGPU::S_FMAMK_F32;
    MI->setDesc(TII->get(NewOpc));

    // We have to fold into operand which would be Imm not into OpNo.
    bool FoldAsFMAAKorMK =
        tryAddToFoldList(FoldList, MI, TryAK ? 3 : 2, OpToFold);
    if (FoldAsFMAAKorMK) {
      // Untie Src2 of fmac.
      MI->untieRegOperand(3);
      // For fmamk swap operands 1 and 2 if OpToFold was meant for operand 1.
      if (OpNo == 1) {
        MachineOperand &Op1 = MI->getOperand(1);
        MachineOperand &Op2 = MI->getOperand(2);
        Register OldReg = Op1.getReg();
        // Operand 2 might be an inlinable constant
        if (Op2.isImm()) {
          Op1.ChangeToImmediate(Op2.getImm());
          Op2.ChangeToRegister(OldReg, false);
        } else {
          Op1.setReg(Op2.getReg());
          Op2.setReg(OldReg);
        }
      }
      return true;
    }
    MI->setDesc(TII->get(Opc));
    return false;
  };

  bool IsLegal = OpToFold.isOperandLegal(*TII, *MI, OpNo);
  if (!IsLegal && OpToFold.isImm()) {
    if (std::optional<int64_t> ImmVal = OpToFold.getEffectiveImmVal())
      IsLegal = canUseImmWithOpSel(MI, OpNo, *ImmVal);
  }

  if (!IsLegal) {
    // Special case for v_mac_{f16, f32}_e64 if we are trying to fold into src2
    unsigned NewOpc = macToMad(Opc);
    if (NewOpc != AMDGPU::INSTRUCTION_LIST_END) {
      // Check if changing this to a v_mad_{f16, f32} instruction will allow us
      // to fold the operand.
      MI->setDesc(TII->get(NewOpc));
      bool AddOpSel = !AMDGPU::hasNamedOperand(Opc, AMDGPU::OpName::op_sel) &&
                      AMDGPU::hasNamedOperand(NewOpc, AMDGPU::OpName::op_sel);
      if (AddOpSel)
        MI->addOperand(MachineOperand::CreateImm(0));
      bool FoldAsMAD = tryAddToFoldList(FoldList, MI, OpNo, OpToFold);
      if (FoldAsMAD) {
        MI->untieRegOperand(OpNo);
        return true;
      }
      if (AddOpSel)
        MI->removeOperand(MI->getNumExplicitOperands() - 1);
      MI->setDesc(TII->get(Opc));
    }

    // Special case for s_fmac_f32 if we are trying to fold into Src2.
    // By transforming into fmaak we can untie Src2 and make folding legal.
    if (Opc == AMDGPU::S_FMAC_F32 && OpNo == 3) {
      if (tryToFoldAsFMAAKorMK())
        return true;
    }

    // Special case for s_setreg_b32
    if (OpToFold.isImm()) {
      unsigned ImmOpc = 0;
      if (Opc == AMDGPU::S_SETREG_B32)
        ImmOpc = AMDGPU::S_SETREG_IMM32_B32;
      else if (Opc == AMDGPU::S_SETREG_B32_mode)
        ImmOpc = AMDGPU::S_SETREG_IMM32_B32_mode;
      if (ImmOpc) {
        MI->setDesc(TII->get(ImmOpc));
        appendFoldCandidate(FoldList, MI, OpNo, OpToFold);
        return true;
      }
    }

    // Operand is not legal, so try to commute the instruction to
    // see if this makes it possible to fold.
    unsigned CommuteOpNo = TargetInstrInfo::CommuteAnyOperandIndex;
    bool CanCommute = TII->findCommutedOpIndices(*MI, OpNo, CommuteOpNo);
    if (!CanCommute)
      return false;

    MachineOperand &Op = MI->getOperand(OpNo);
    MachineOperand &CommutedOp = MI->getOperand(CommuteOpNo);

    // One of operands might be an Imm operand, and OpNo may refer to it after
    // the call of commuteInstruction() below. Such situations are avoided
    // here explicitly as OpNo must be a register operand to be a candidate
    // for memory folding.
    if (!Op.isReg() || !CommutedOp.isReg())
      return false;

    // The same situation with an immediate could reproduce if both inputs are
    // the same register.
    if (Op.isReg() && CommutedOp.isReg() &&
        (Op.getReg() == CommutedOp.getReg() &&
         Op.getSubReg() == CommutedOp.getSubReg()))
      return false;

    if (!TII->commuteInstruction(*MI, false, OpNo, CommuteOpNo))
      return false;

    int Op32 = -1;
    if (!OpToFold.isOperandLegal(*TII, *MI, CommuteOpNo)) {
      if ((Opc != AMDGPU::V_ADD_CO_U32_e64 && Opc != AMDGPU::V_SUB_CO_U32_e64 &&
           Opc != AMDGPU::V_SUBREV_CO_U32_e64) || // FIXME
          (!OpToFold.isImm() && !OpToFold.isFI() && !OpToFold.isGlobal())) {
        TII->commuteInstruction(*MI, false, OpNo, CommuteOpNo);
        return false;
      }

      // Verify the other operand is a VGPR, otherwise we would violate the
      // constant bus restriction.
      MachineOperand &OtherOp = MI->getOperand(OpNo);
      if (!OtherOp.isReg() ||
          !TII->getRegisterInfo().isVGPR(*MRI, OtherOp.getReg()))
        return false;

      assert(MI->getOperand(1).isDef());

      // Make sure to get the 32-bit version of the commuted opcode.
      unsigned MaybeCommutedOpc = MI->getOpcode();
      Op32 = AMDGPU::getVOPe32(MaybeCommutedOpc);
    }

    appendFoldCandidate(FoldList, MI, CommuteOpNo, OpToFold, /*Commuted=*/true,
                        Op32);
    return true;
  }

  // Special case for s_fmac_f32 if we are trying to fold into Src0 or Src1.
  // By changing into fmamk we can untie Src2.
  // If folding for Src0 happens first and it is identical operand to Src1 we
  // should avoid transforming into fmamk which requires commuting as it would
  // cause folding into Src1 to fail later on due to wrong OpNo used.
  if (Opc == AMDGPU::S_FMAC_F32 &&
      (OpNo != 1 || !MI->getOperand(1).isIdenticalTo(MI->getOperand(2)))) {
    if (tryToFoldAsFMAAKorMK())
      return true;
  }

  appendFoldCandidate(FoldList, MI, OpNo, OpToFold);
  return true;
}
```
<a name="block_7"></a>**File:** llvm/lib/Target/AMDGPU/SIFoldOperands.cpp (L897-901) [<sup>↩</sup>](#ref-block_7)
```cpp
bool SIFoldOperandsImpl::isUseSafeToFold(const MachineInstr &MI,
                                         const MachineOperand &UseMO) const {
  // Operands of SDWA instructions must be registers.
  return !TII->isSDWA(MI);
}
```
<a name="block_8"></a>**File:** llvm/lib/Target/AMDGPU/SIFoldOperands.cpp (L924-972) [<sup>↩</sup>](#ref-block_8)
```cpp
const TargetRegisterClass *SIFoldOperandsImpl::getRegSeqInit(
    MachineInstr &RegSeq,
    SmallVectorImpl<std::pair<MachineOperand *, unsigned>> &Defs) const {

  assert(RegSeq.isRegSequence());

  const TargetRegisterClass *RC = nullptr;

  for (unsigned I = 1, E = RegSeq.getNumExplicitOperands(); I != E; I += 2) {
    MachineOperand &SrcOp = RegSeq.getOperand(I);
    unsigned SubRegIdx = RegSeq.getOperand(I + 1).getImm();

    // Only accept reg_sequence with uniform reg class inputs for simplicity.
    const TargetRegisterClass *OpRC = getRegOpRC(*MRI, *TRI, SrcOp);
    if (!RC)
      RC = OpRC;
    else if (!TRI->getCommonSubClass(RC, OpRC))
      return nullptr;

    if (SrcOp.getSubReg()) {
      // TODO: Handle subregister compose
      Defs.emplace_back(&SrcOp, SubRegIdx);
      continue;
    }

    MachineOperand *DefSrc = lookUpCopyChain(*TII, *MRI, SrcOp.getReg());
    if (DefSrc && (DefSrc->isReg() || DefSrc->isImm())) {
      Defs.emplace_back(DefSrc, SubRegIdx);
      continue;
    }

    Defs.emplace_back(&SrcOp, SubRegIdx);
  }

  return RC;
}

// Find a def of the UseReg, check if it is a reg_sequence and find initializers
// for each subreg, tracking it to an immediate if possible. Returns the
// register class of the inputs on success.
const TargetRegisterClass *SIFoldOperandsImpl::getRegSeqInit(
    SmallVectorImpl<std::pair<MachineOperand *, unsigned>> &Defs,
    Register UseReg) const {
  MachineInstr *Def = MRI->getVRegDef(UseReg);
  if (!Def || !Def->isRegSequence())
    return nullptr;

  return getRegSeqInit(*Def, Defs);
}
```
<a name="block_9"></a>**File:** llvm/lib/Target/AMDGPU/SIFoldOperands.cpp (L974-1040) [<sup>↩</sup>](#ref-block_9)
```cpp
std::pair<int64_t, const TargetRegisterClass *>
SIFoldOperandsImpl::isRegSeqSplat(MachineInstr &RegSeq) const {
  SmallVector<std::pair<MachineOperand *, unsigned>, 32> Defs;
  const TargetRegisterClass *SrcRC = getRegSeqInit(RegSeq, Defs);
  if (!SrcRC)
    return {};

  bool TryToMatchSplat64 = false;

  int64_t Imm;
  for (unsigned I = 0, E = Defs.size(); I != E; ++I) {
    const MachineOperand *Op = Defs[I].first;
    if (!Op->isImm())
      return {};

    int64_t SubImm = Op->getImm();
    if (!I) {
      Imm = SubImm;
      continue;
    }

    if (Imm != SubImm) {
      if (I == 1 && (E & 1) == 0) {
        // If we have an even number of inputs, there's a chance this is a
        // 64-bit element splat broken into 32-bit pieces.
        TryToMatchSplat64 = true;
        break;
      }

      return {}; // Can only fold splat constants
    }
  }

  if (!TryToMatchSplat64)
    return {Defs[0].first->getImm(), SrcRC};

  // Fallback to recognizing 64-bit splats broken into 32-bit pieces
  // (i.e. recognize every other other element is 0 for 64-bit immediates)
  int64_t SplatVal64;
  for (unsigned I = 0, E = Defs.size(); I != E; I += 2) {
    const MachineOperand *Op0 = Defs[I].first;
    const MachineOperand *Op1 = Defs[I + 1].first;

    if (!Op0->isImm() || !Op1->isImm())
      return {};

    unsigned SubReg0 = Defs[I].second;
    unsigned SubReg1 = Defs[I + 1].second;

    // Assume we're going to generally encounter reg_sequences with sorted
    // subreg indexes, so reject any that aren't consecutive.
    if (TRI->getChannelFromSubReg(SubReg0) + 1 !=
        TRI->getChannelFromSubReg(SubReg1))
      return {};

    int64_t MergedVal = Make_64(Op1->getImm(), Op0->getImm());
    if (I == 0)
      SplatVal64 = MergedVal;
    else if (SplatVal64 != MergedVal)
      return {};
  }

  const TargetRegisterClass *RC64 = TRI->getSubRegisterClass(
      MRI->getRegClass(RegSeq.getOperand(0).getReg()), AMDGPU::sub0_sub1);

  return {SplatVal64, RC64};
}
```
<a name="block_10"></a>**File:** llvm/lib/Target/AMDGPU/SIFoldOperands.cpp (L1042-1088) [<sup>↩</sup>](#ref-block_10)
```cpp
bool SIFoldOperandsImpl::tryFoldRegSeqSplat(
    MachineInstr *UseMI, unsigned UseOpIdx, int64_t SplatVal,
    const TargetRegisterClass *SplatRC) const {
  const MCInstrDesc &Desc = UseMI->getDesc();
  if (UseOpIdx >= Desc.getNumOperands())
    return false;

  // Filter out unhandled pseudos.
  if (!AMDGPU::isSISrcOperand(Desc, UseOpIdx))
    return false;

  int16_t RCID = Desc.operands()[UseOpIdx].RegClass;
  if (RCID == -1)
    return false;

  const TargetRegisterClass *OpRC = TRI->getRegClass(RCID);

  // Special case 0/-1, since when interpreted as a 64-bit element both halves
  // have the same bits. These are the only cases where a splat has the same
  // interpretation for 32-bit and 64-bit splats.
  if (SplatVal != 0 && SplatVal != -1) {
    // We need to figure out the scalar type read by the operand. e.g. the MFMA
    // operand will be AReg_128, and we want to check if it's compatible with an
    // AReg_32 constant.
    uint8_t OpTy = Desc.operands()[UseOpIdx].OperandType;
    switch (OpTy) {
    case AMDGPU::OPERAND_REG_INLINE_AC_INT32:
    case AMDGPU::OPERAND_REG_INLINE_AC_FP32:
      OpRC = TRI->getSubRegisterClass(OpRC, AMDGPU::sub0);
      break;
    case AMDGPU::OPERAND_REG_INLINE_AC_FP64:
      OpRC = TRI->getSubRegisterClass(OpRC, AMDGPU::sub0_sub1);
      break;
    default:
      return false;
    }

    if (!TRI->getCommonSubClass(OpRC, SplatRC))
      return false;
  }

  MachineOperand TmpOp = MachineOperand::CreateImm(SplatVal);
  if (!TII->isOperandLegal(*UseMI, UseOpIdx, &TmpOp))
    return false;

  return true;
}
```
<a name="block_11"></a>**File:** llvm/lib/Target/AMDGPU/SIFoldOperands.cpp (L1090-1135) [<sup>↩</sup>](#ref-block_11)
```cpp
bool SIFoldOperandsImpl::tryToFoldACImm(
    const FoldableDef &OpToFold, MachineInstr *UseMI, unsigned UseOpIdx,
    SmallVectorImpl<FoldCandidate> &FoldList) const {
  const MCInstrDesc &Desc = UseMI->getDesc();
  if (UseOpIdx >= Desc.getNumOperands())
    return false;

  if (!AMDGPU::isSISrcInlinableOperand(Desc, UseOpIdx))
    return false;

  MachineOperand &UseOp = UseMI->getOperand(UseOpIdx);
  if (OpToFold.isImm() && OpToFold.isOperandLegal(*TII, *UseMI, UseOpIdx)) {
    appendFoldCandidate(FoldList, UseMI, UseOpIdx, OpToFold);
    return true;
  }

  // TODO: Verify the following code handles subregisters correctly.
  // TODO: Handle extract of global reference
  if (UseOp.getSubReg())
    return false;

  if (!OpToFold.isReg())
    return false;

  Register UseReg = OpToFold.getReg();
  if (!UseReg.isVirtual())
    return false;

  // Maybe it is just a COPY of an immediate itself.

  // FIXME: Remove this handling. There is already special case folding of
  // immediate into copy in foldOperand. This is looking for the def of the
  // value the folding started from in the first place.
  MachineInstr *Def = MRI->getVRegDef(UseReg);
  if (Def && TII->isFoldableCopy(*Def)) {
    MachineOperand &DefOp = Def->getOperand(1);
    if (DefOp.isImm() && TII->isOperandLegal(*UseMI, UseOpIdx, &DefOp)) {
      FoldableDef FoldableImm(DefOp.getImm(), OpToFold.DefRC,
                              OpToFold.DefSubReg);
      appendFoldCandidate(FoldList, UseMI, UseOpIdx, FoldableImm);
      return true;
    }
  }

  return false;
}
```
<a name="block_12"></a>**File:** llvm/lib/Target/AMDGPU/SIFoldOperands.cpp (L1137-1445) [<sup>↩</sup>](#ref-block_12) [<sup>↩</sup>](#ref-block_12)
```cpp
void SIFoldOperandsImpl::foldOperand(
    FoldableDef OpToFold, MachineInstr *UseMI, int UseOpIdx,
    SmallVectorImpl<FoldCandidate> &FoldList,
    SmallVectorImpl<MachineInstr *> &CopiesToReplace) const {
  const MachineOperand *UseOp = &UseMI->getOperand(UseOpIdx);

  if (!isUseSafeToFold(*UseMI, *UseOp))
    return;

  // FIXME: Fold operands with subregs.
  if (UseOp->isReg() && OpToFold.isReg()) {
    if (UseOp->isImplicit())
      return;
    // Allow folding from SGPRs to 16-bit VGPRs.
    if (UseOp->getSubReg() != AMDGPU::NoSubRegister &&
        (UseOp->getSubReg() != AMDGPU::lo16 ||
         !TRI->isSGPRReg(*MRI, OpToFold.getReg())))
      return;
  }

  // Special case for REG_SEQUENCE: We can't fold literals into
  // REG_SEQUENCE instructions, so we have to fold them into the
  // uses of REG_SEQUENCE.
  if (UseMI->isRegSequence()) {
    Register RegSeqDstReg = UseMI->getOperand(0).getReg();
    unsigned RegSeqDstSubReg = UseMI->getOperand(UseOpIdx + 1).getImm();

    int64_t SplatVal;
    const TargetRegisterClass *SplatRC;
    std::tie(SplatVal, SplatRC) = isRegSeqSplat(*UseMI);

    // Grab the use operands first
    SmallVector<MachineOperand *, 4> UsesToProcess(
        llvm::make_pointer_range(MRI->use_nodbg_operands(RegSeqDstReg)));
    for (auto *RSUse : UsesToProcess) {
      MachineInstr *RSUseMI = RSUse->getParent();
      unsigned OpNo = RSUseMI->getOperandNo(RSUse);

      if (SplatRC) {
        if (tryFoldRegSeqSplat(RSUseMI, OpNo, SplatVal, SplatRC)) {
          FoldableDef SplatDef(SplatVal, SplatRC);
          appendFoldCandidate(FoldList, RSUseMI, OpNo, SplatDef);
          continue;
        }
      }

      // TODO: Handle general compose
      if (RSUse->getSubReg() != RegSeqDstSubReg)
        continue;

      // FIXME: We should avoid recursing here. There should be a cleaner split
      // between the in-place mutations and adding to the fold list.
      foldOperand(OpToFold, RSUseMI, RSUseMI->getOperandNo(RSUse), FoldList,
                  CopiesToReplace);
    }

    return;
  }

  if (tryToFoldACImm(OpToFold, UseMI, UseOpIdx, FoldList))
    return;

  if (frameIndexMayFold(*UseMI, UseOpIdx, OpToFold)) {
    // Verify that this is a stack access.
    // FIXME: Should probably use stack pseudos before frame lowering.

    if (TII->isMUBUF(*UseMI)) {
      if (TII->getNamedOperand(*UseMI, AMDGPU::OpName::srsrc)->getReg() !=
          MFI->getScratchRSrcReg())
        return;

      // Ensure this is either relative to the current frame or the current
      // wave.
      MachineOperand &SOff =
          *TII->getNamedOperand(*UseMI, AMDGPU::OpName::soffset);
      if (!SOff.isImm() || SOff.getImm() != 0)
        return;
    }

    // A frame index will resolve to a positive constant, so it should always be
    // safe to fold the addressing mode, even pre-GFX9.
    UseMI->getOperand(UseOpIdx).ChangeToFrameIndex(OpToFold.getFI());

    const unsigned Opc = UseMI->getOpcode();
    if (TII->isFLATScratch(*UseMI) &&
        AMDGPU::hasNamedOperand(Opc, AMDGPU::OpName::vaddr) &&
        !AMDGPU::hasNamedOperand(Opc, AMDGPU::OpName::saddr)) {
      unsigned NewOpc = AMDGPU::getFlatScratchInstSSfromSV(Opc);
      UseMI->setDesc(TII->get(NewOpc));
    }

    return;
  }

  bool FoldingImmLike =
      OpToFold.isImm() || OpToFold.isFI() || OpToFold.isGlobal();

  if (FoldingImmLike && UseMI->isCopy()) {
    Register DestReg = UseMI->getOperand(0).getReg();
    Register SrcReg = UseMI->getOperand(1).getReg();
    assert(SrcReg.isVirtual());

    const TargetRegisterClass *SrcRC = MRI->getRegClass(SrcReg);

    // Don't fold into a copy to a physical register with the same class. Doing
    // so would interfere with the register coalescer's logic which would avoid
    // redundant initializations.
    if (DestReg.isPhysical() && SrcRC->contains(DestReg))
      return;

    const TargetRegisterClass *DestRC = TRI->getRegClassForReg(*MRI, DestReg);
    if (!DestReg.isPhysical() && DestRC == &AMDGPU::AGPR_32RegClass) {
      std::optional<int64_t> UseImmVal = OpToFold.getEffectiveImmVal();
      if (UseImmVal && TII->isInlineConstant(
                           *UseImmVal, AMDGPU::OPERAND_REG_INLINE_C_INT32)) {
        UseMI->setDesc(TII->get(AMDGPU::V_ACCVGPR_WRITE_B32_e64));
        UseMI->getOperand(1).ChangeToImmediate(*UseImmVal);
        CopiesToReplace.push_back(UseMI);
        return;
      }
    }

    // Allow immediates COPYd into sgpr_lo16 to be further folded while
    // still being legal if not further folded
    if (DestRC == &AMDGPU::SGPR_LO16RegClass) {
      assert(ST->useRealTrue16Insts());
      MRI->setRegClass(DestReg, &AMDGPU::SGPR_32RegClass);
      DestRC = &AMDGPU::SGPR_32RegClass;
    }

    // In order to fold immediates into copies, we need to change the
    // copy to a MOV.

    unsigned MovOp = TII->getMovOpcode(DestRC);
    if (MovOp == AMDGPU::COPY)
      return;

    // Fold if the destination register class of the MOV instruction (ResRC)
    // is a superclass of (or equal to) the destination register class of the
    // COPY (DestRC). If this condition fails, folding would be illegal.
    const MCInstrDesc &MovDesc = TII->get(MovOp);
    assert(MovDesc.getNumDefs() > 0 && MovDesc.operands()[0].RegClass != -1);
    const TargetRegisterClass *ResRC =
        TRI->getRegClass(MovDesc.operands()[0].RegClass);
    if (!DestRC->hasSuperClassEq(ResRC))
      return;

    MachineInstr::mop_iterator ImpOpI = UseMI->implicit_operands().begin();
    MachineInstr::mop_iterator ImpOpE = UseMI->implicit_operands().end();
    while (ImpOpI != ImpOpE) {
      MachineInstr::mop_iterator Tmp = ImpOpI;
      ImpOpI++;
      UseMI->removeOperand(UseMI->getOperandNo(Tmp));
    }
    UseMI->setDesc(TII->get(MovOp));

    if (MovOp == AMDGPU::V_MOV_B16_t16_e64) {
      const auto &SrcOp = UseMI->getOperand(UseOpIdx);
      MachineOperand NewSrcOp(SrcOp);
      MachineFunction *MF = UseMI->getParent()->getParent();
      UseMI->removeOperand(1);
      UseMI->addOperand(*MF, MachineOperand::CreateImm(0)); // src0_modifiers
      UseMI->addOperand(NewSrcOp);                          // src0
      UseMI->addOperand(*MF, MachineOperand::CreateImm(0)); // op_sel
      UseOpIdx = 2;
      UseOp = &UseMI->getOperand(UseOpIdx);
    }
    CopiesToReplace.push_back(UseMI);
  } else {
    if (UseMI->isCopy() && OpToFold.isReg() &&
        UseMI->getOperand(0).getReg().isVirtual() &&
        !UseMI->getOperand(1).getSubReg() &&
        OpToFold.DefMI->implicit_operands().empty()) {
      LLVM_DEBUG(dbgs() << "Folding " << OpToFold.OpToFold << "\n into "
                        << *UseMI);
      unsigned Size = TII->getOpSize(*UseMI, 1);
      Register UseReg = OpToFold.getReg();
      UseMI->getOperand(1).setReg(UseReg);
      unsigned SubRegIdx = OpToFold.getSubReg();
      // Hack to allow 32-bit SGPRs to be folded into True16 instructions
      // Remove this if 16-bit SGPRs (i.e. SGPR_LO16) are added to the
      // VS_16RegClass
      //
      // Excerpt from AMDGPUGenRegisterInfo.inc
      // NoSubRegister, //0
      // hi16, // 1
      // lo16, // 2
      // sub0, // 3
      // ...
      // sub1, // 11
      // sub1_hi16, // 12
      // sub1_lo16, // 13
      static_assert(AMDGPU::sub1_hi16 == 12, "Subregister layout has changed");
      if (Size == 2 && TRI->isVGPR(*MRI, UseMI->getOperand(0).getReg()) &&
          TRI->isSGPRReg(*MRI, UseReg)) {
        // Produce the 32 bit subregister index to which the 16-bit subregister
        // is aligned.
        if (SubRegIdx > AMDGPU::sub1) {
          LaneBitmask M = TRI->getSubRegIndexLaneMask(SubRegIdx);
          M |= M.getLane(M.getHighestLane() - 1);
          SmallVector<unsigned, 4> Indexes;
          TRI->getCoveringSubRegIndexes(TRI->getRegClassForReg(*MRI, UseReg), M,
                                        Indexes);
          assert(Indexes.size() == 1 && "Expected one 32-bit subreg to cover");
          SubRegIdx = Indexes[0];
          // 32-bit registers do not have a sub0 index
        } else if (TII->getOpSize(*UseMI, 1) == 4)
          SubRegIdx = 0;
        else
          SubRegIdx = AMDGPU::sub0;
      }
      UseMI->getOperand(1).setSubReg(SubRegIdx);
      UseMI->getOperand(1).setIsKill(false);
      CopiesToReplace.push_back(UseMI);
      OpToFold.OpToFold->setIsKill(false);

      // Remove kill flags as kills may now be out of order with uses.
      MRI->clearKillFlags(UseReg);
      if (foldCopyToAGPRRegSequence(UseMI))
        return;
    }

    unsigned UseOpc = UseMI->getOpcode();
    if (UseOpc == AMDGPU::V_READFIRSTLANE_B32 ||
        (UseOpc == AMDGPU::V_READLANE_B32 &&
         (int)UseOpIdx ==
         AMDGPU::getNamedOperandIdx(UseOpc, AMDGPU::OpName::src0))) {
      // %vgpr = V_MOV_B32 imm
      // %sgpr = V_READFIRSTLANE_B32 %vgpr
      // =>
      // %sgpr = S_MOV_B32 imm
      if (FoldingImmLike) {
        if (execMayBeModifiedBeforeUse(*MRI,
                                       UseMI->getOperand(UseOpIdx).getReg(),
                                       *OpToFold.DefMI, *UseMI))
          return;

        UseMI->setDesc(TII->get(AMDGPU::S_MOV_B32));

        if (OpToFold.isImm()) {
          UseMI->getOperand(1).ChangeToImmediate(
              *OpToFold.getEffectiveImmVal());
        } else if (OpToFold.isFI())
          UseMI->getOperand(1).ChangeToFrameIndex(OpToFold.getFI());
        else {
          assert(OpToFold.isGlobal());
          UseMI->getOperand(1).ChangeToGA(OpToFold.OpToFold->getGlobal(),
                                          OpToFold.OpToFold->getOffset(),
                                          OpToFold.OpToFold->getTargetFlags());
        }
        UseMI->removeOperand(2); // Remove exec read (or src1 for readlane)
        return;
      }

      if (OpToFold.isReg() && TRI->isSGPRReg(*MRI, OpToFold.getReg())) {
        if (checkIfExecMayBeModifiedBeforeUseAcrossBB(
                *MRI, UseMI->getOperand(UseOpIdx).getReg(),
                *OpToFold.DefMI, *UseMI, SIFoldOperandsPreheaderThreshold))
          return;

        // %vgpr = COPY %sgpr0
        // %sgpr1 = V_READFIRSTLANE_B32 %vgpr
        // =>
        // %sgpr1 = COPY %sgpr0
        UseMI->setDesc(TII->get(AMDGPU::COPY));
        UseMI->getOperand(1).setReg(OpToFold.getReg());
        UseMI->getOperand(1).setSubReg(OpToFold.getSubReg());
        UseMI->getOperand(1).setIsKill(false);
        UseMI->removeOperand(2); // Remove exec read (or src1 for readlane)
        return;
      }
    }

    const MCInstrDesc &UseDesc = UseMI->getDesc();

    // Don't fold into target independent nodes.  Target independent opcodes
    // don't have defined register classes.
    if (UseDesc.isVariadic() || UseOp->isImplicit() ||
        UseDesc.operands()[UseOpIdx].RegClass == -1)
      return;
  }

  if (!FoldingImmLike) {
    if (OpToFold.isReg() && ST->needsAlignedVGPRs()) {
      // Don't fold if OpToFold doesn't hold an aligned register.
      const TargetRegisterClass *RC =
          TRI->getRegClassForReg(*MRI, OpToFold.getReg());
      assert(RC);
      if (TRI->hasVectorRegisters(RC) && OpToFold.getSubReg()) {
        unsigned SubReg = OpToFold.getSubReg();
        if (const TargetRegisterClass *SubRC =
                TRI->getSubRegisterClass(RC, SubReg))
          RC = SubRC;
      }

      if (!RC || !TRI->isProperlyAlignedRC(*RC))
        return;
    }

    tryAddToFoldList(FoldList, UseMI, UseOpIdx, OpToFold);

    // FIXME: We could try to change the instruction from 64-bit to 32-bit
    // to enable more folding opportunities.  The shrink operands pass
    // already does this.
    return;
  }

  tryAddToFoldList(FoldList, UseMI, UseOpIdx, OpToFold);
}
```
<a name="block_13"></a>**File:** llvm/lib/Target/AMDGPU/SIFoldOperands.cpp (L1531-1547) [<sup>↩</sup>](#ref-block_13)
```cpp
std::optional<int64_t>
SIFoldOperandsImpl::getImmOrMaterializedImm(MachineOperand &Op) const {
  if (Op.isImm())
    return Op.getImm();

  if (!Op.isReg() || !Op.getReg().isVirtual())
    return std::nullopt;

  const MachineInstr *Def = MRI->getVRegDef(Op.getReg());
  if (Def && Def->isMoveImmediate()) {
    const MachineOperand &ImmSrc = Def->getOperand(1);
    if (ImmSrc.isImm())
      return TII->extractSubregFromImm(ImmSrc.getImm(), Op.getSubReg());
  }

  return std::nullopt;
}
```
<a name="block_14"></a>**File:** llvm/lib/Target/AMDGPU/SIFoldOperands.cpp (L1552-1655) [<sup>↩</sup>](#ref-block_14) [<sup>↩</sup>](#ref-block_14)
```cpp
bool SIFoldOperandsImpl::tryConstantFoldOp(MachineInstr *MI) const {
  if (!MI->allImplicitDefsAreDead())
    return false;

  unsigned Opc = MI->getOpcode();

  int Src0Idx = AMDGPU::getNamedOperandIdx(Opc, AMDGPU::OpName::src0);
  if (Src0Idx == -1)
    return false;

  MachineOperand *Src0 = &MI->getOperand(Src0Idx);
  std::optional<int64_t> Src0Imm = getImmOrMaterializedImm(*Src0);

  if ((Opc == AMDGPU::V_NOT_B32_e64 || Opc == AMDGPU::V_NOT_B32_e32 ||
       Opc == AMDGPU::S_NOT_B32) &&
      Src0Imm) {
    MI->getOperand(1).ChangeToImmediate(~*Src0Imm);
    mutateCopyOp(*MI, TII->get(getMovOpc(Opc == AMDGPU::S_NOT_B32)));
    return true;
  }

  int Src1Idx = AMDGPU::getNamedOperandIdx(Opc, AMDGPU::OpName::src1);
  if (Src1Idx == -1)
    return false;

  MachineOperand *Src1 = &MI->getOperand(Src1Idx);
  std::optional<int64_t> Src1Imm = getImmOrMaterializedImm(*Src1);

  if (!Src0Imm && !Src1Imm)
    return false;

  // and k0, k1 -> v_mov_b32 (k0 & k1)
  // or k0, k1 -> v_mov_b32 (k0 | k1)
  // xor k0, k1 -> v_mov_b32 (k0 ^ k1)
  if (Src0Imm && Src1Imm) {
    int32_t NewImm;
    if (!evalBinaryInstruction(Opc, NewImm, *Src0Imm, *Src1Imm))
      return false;

    bool IsSGPR = TRI->isSGPRReg(*MRI, MI->getOperand(0).getReg());

    // Be careful to change the right operand, src0 may belong to a different
    // instruction.
    MI->getOperand(Src0Idx).ChangeToImmediate(NewImm);
    MI->removeOperand(Src1Idx);
    mutateCopyOp(*MI, TII->get(getMovOpc(IsSGPR)));
    return true;
  }

  if (!MI->isCommutable())
    return false;

  if (Src0Imm && !Src1Imm) {
    std::swap(Src0, Src1);
    std::swap(Src0Idx, Src1Idx);
    std::swap(Src0Imm, Src1Imm);
  }

  int32_t Src1Val = static_cast<int32_t>(*Src1Imm);
  if (Opc == AMDGPU::V_OR_B32_e64 ||
      Opc == AMDGPU::V_OR_B32_e32 ||
      Opc == AMDGPU::S_OR_B32) {
    if (Src1Val == 0) {
      // y = or x, 0 => y = copy x
      MI->removeOperand(Src1Idx);
      mutateCopyOp(*MI, TII->get(AMDGPU::COPY));
    } else if (Src1Val == -1) {
      // y = or x, -1 => y = v_mov_b32 -1
      MI->removeOperand(Src1Idx);
      mutateCopyOp(*MI, TII->get(getMovOpc(Opc == AMDGPU::S_OR_B32)));
    } else
      return false;

    return true;
  }

  if (Opc == AMDGPU::V_AND_B32_e64 || Opc == AMDGPU::V_AND_B32_e32 ||
      Opc == AMDGPU::S_AND_B32) {
    if (Src1Val == 0) {
      // y = and x, 0 => y = v_mov_b32 0
      MI->removeOperand(Src0Idx);
      mutateCopyOp(*MI, TII->get(getMovOpc(Opc == AMDGPU::S_AND_B32)));
    } else if (Src1Val == -1) {
      // y = and x, -1 => y = copy x
      MI->removeOperand(Src1Idx);
      mutateCopyOp(*MI, TII->get(AMDGPU::COPY));
    } else
      return false;

    return true;
  }

  if (Opc == AMDGPU::V_XOR_B32_e64 || Opc == AMDGPU::V_XOR_B32_e32 ||
      Opc == AMDGPU::S_XOR_B32) {
    if (Src1Val == 0) {
      // y = xor x, 0 => y = copy x
      MI->removeOperand(Src1Idx);
      mutateCopyOp(*MI, TII->get(AMDGPU::COPY));
      return true;
    }
  }

  return false;
}
```
<a name="block_15"></a>**File:** llvm/lib/Target/AMDGPU/SIFoldOperands.cpp (L1658-1698) [<sup>↩</sup>](#ref-block_15) [<sup>↩</sup>](#ref-block_15)
```cpp
bool SIFoldOperandsImpl::tryFoldCndMask(MachineInstr &MI) const {
  unsigned Opc = MI.getOpcode();
  if (Opc != AMDGPU::V_CNDMASK_B32_e32 && Opc != AMDGPU::V_CNDMASK_B32_e64 &&
      Opc != AMDGPU::V_CNDMASK_B64_PSEUDO)
    return false;

  MachineOperand *Src0 = TII->getNamedOperand(MI, AMDGPU::OpName::src0);
  MachineOperand *Src1 = TII->getNamedOperand(MI, AMDGPU::OpName::src1);
  if (!Src1->isIdenticalTo(*Src0)) {
    std::optional<int64_t> Src1Imm = getImmOrMaterializedImm(*Src1);
    if (!Src1Imm)
      return false;

    std::optional<int64_t> Src0Imm = getImmOrMaterializedImm(*Src0);
    if (!Src0Imm || *Src0Imm != *Src1Imm)
      return false;
  }

  int Src1ModIdx =
      AMDGPU::getNamedOperandIdx(Opc, AMDGPU::OpName::src1_modifiers);
  int Src0ModIdx =
      AMDGPU::getNamedOperandIdx(Opc, AMDGPU::OpName::src0_modifiers);
  if ((Src1ModIdx != -1 && MI.getOperand(Src1ModIdx).getImm() != 0) ||
      (Src0ModIdx != -1 && MI.getOperand(Src0ModIdx).getImm() != 0))
    return false;

  LLVM_DEBUG(dbgs() << "Folded " << MI << " into ");
  auto &NewDesc =
      TII->get(Src0->isReg() ? (unsigned)AMDGPU::COPY : getMovOpc(false));
  int Src2Idx = AMDGPU::getNamedOperandIdx(Opc, AMDGPU::OpName::src2);
  if (Src2Idx != -1)
    MI.removeOperand(Src2Idx);
  MI.removeOperand(AMDGPU::getNamedOperandIdx(Opc, AMDGPU::OpName::src1));
  if (Src1ModIdx != -1)
    MI.removeOperand(Src1ModIdx);
  if (Src0ModIdx != -1)
    MI.removeOperand(Src0ModIdx);
  mutateCopyOp(MI, NewDesc);
  LLVM_DEBUG(dbgs() << MI);
  return true;
}
```
<a name="block_16"></a>**File:** llvm/lib/Target/AMDGPU/SIFoldOperands.cpp (L1700-1720) [<sup>↩</sup>](#ref-block_16) [<sup>↩</sup>](#ref-block_16)
```cpp
bool SIFoldOperandsImpl::tryFoldZeroHighBits(MachineInstr &MI) const {
  if (MI.getOpcode() != AMDGPU::V_AND_B32_e64 &&
      MI.getOpcode() != AMDGPU::V_AND_B32_e32)
    return false;

  std::optional<int64_t> Src0Imm = getImmOrMaterializedImm(MI.getOperand(1));
  if (!Src0Imm || *Src0Imm != 0xffff || !MI.getOperand(2).isReg())
    return false;

  Register Src1 = MI.getOperand(2).getReg();
  MachineInstr *SrcDef = MRI->getVRegDef(Src1);
  if (!ST->zeroesHigh16BitsOfDest(SrcDef->getOpcode()))
    return false;

  Register Dst = MI.getOperand(0).getReg();
  MRI->replaceRegWith(Dst, Src1);
  if (!MI.getOperand(2).isKill())
    MRI->clearKillFlags(Src1);
  MI.eraseFromParent();
  return true;
}
```
<a name="block_17"></a>**File:** llvm/lib/Target/AMDGPU/SIFoldOperands.cpp (L1722-1801) [<sup>↩</sup>](#ref-block_17) [<sup>↩</sup>](#ref-block_17)
```cpp
bool SIFoldOperandsImpl::foldInstOperand(MachineInstr &MI,
                                         const FoldableDef &OpToFold) const {
  // We need mutate the operands of new mov instructions to add implicit
  // uses of EXEC, but adding them invalidates the use_iterator, so defer
  // this.
  SmallVector<MachineInstr *, 4> CopiesToReplace;
  SmallVector<FoldCandidate, 4> FoldList;
  MachineOperand &Dst = MI.getOperand(0);
  bool Changed = false;

  if (OpToFold.isImm()) {
    for (auto &UseMI :
         make_early_inc_range(MRI->use_nodbg_instructions(Dst.getReg()))) {
      // Folding the immediate may reveal operations that can be constant
      // folded or replaced with a copy. This can happen for example after
      // frame indices are lowered to constants or from splitting 64-bit
      // constants.
      //
      // We may also encounter cases where one or both operands are
      // immediates materialized into a register, which would ordinarily not
      // be folded due to multiple uses or operand constraints.
      if (tryConstantFoldOp(&UseMI)) {
        LLVM_DEBUG(dbgs() << "Constant folded " << UseMI);
        Changed = true;
      }
    }
  }

  SmallVector<MachineOperand *, 4> UsesToProcess(
      llvm::make_pointer_range(MRI->use_nodbg_operands(Dst.getReg())));
  for (auto *U : UsesToProcess) {
    MachineInstr *UseMI = U->getParent();

    FoldableDef SubOpToFold = OpToFold.getWithSubReg(*TRI, U->getSubReg());
    foldOperand(SubOpToFold, UseMI, UseMI->getOperandNo(U), FoldList,
                CopiesToReplace);
  }

  if (CopiesToReplace.empty() && FoldList.empty())
    return Changed;

  MachineFunction *MF = MI.getParent()->getParent();
  // Make sure we add EXEC uses to any new v_mov instructions created.
  for (MachineInstr *Copy : CopiesToReplace)
    Copy->addImplicitDefUseOperands(*MF);

  for (FoldCandidate &Fold : FoldList) {
    assert(!Fold.isReg() || Fold.Def.OpToFold);
    if (Fold.isReg() && Fold.getReg().isVirtual()) {
      Register Reg = Fold.getReg();
      const MachineInstr *DefMI = Fold.Def.DefMI;
      if (DefMI->readsRegister(AMDGPU::EXEC, TRI) &&
          execMayBeModifiedBeforeUse(*MRI, Reg, *DefMI, *Fold.UseMI))
        continue;
    }
    if (updateOperand(Fold)) {
      // Clear kill flags.
      if (Fold.isReg()) {
        assert(Fold.Def.OpToFold && Fold.isReg());
        // FIXME: Probably shouldn't bother trying to fold if not an
        // SGPR. PeepholeOptimizer can eliminate redundant VGPR->VGPR
        // copies.
        MRI->clearKillFlags(Fold.getReg());
      }
      LLVM_DEBUG(dbgs() << "Folded source from " << MI << " into OpNo "
                        << static_cast<int>(Fold.UseOpNo) << " of "
                        << *Fold.UseMI);

      if (Fold.isImm() && tryConstantFoldOp(Fold.UseMI)) {
        LLVM_DEBUG(dbgs() << "Constant folded " << *Fold.UseMI);
        Changed = true;
      }

    } else if (Fold.Commuted) {
      // Restoring instruction's original operand order if fold has failed.
      TII->commuteInstruction(*Fold.UseMI, false);
    }
  }
  return true;
}
```
<a name="block_18"></a>**File:** llvm/lib/Target/AMDGPU/SIFoldOperands.cpp (L1805-1933) [<sup>↩</sup>](#ref-block_18)
```cpp
bool SIFoldOperandsImpl::foldCopyToAGPRRegSequence(MachineInstr *CopyMI) const {
  // It is very tricky to store a value into an AGPR. v_accvgpr_write_b32 can
  // only accept VGPR or inline immediate. Recreate a reg_sequence with its
  // initializers right here, so we will rematerialize immediates and avoid
  // copies via different reg classes.
  const TargetRegisterClass *DefRC =
      MRI->getRegClass(CopyMI->getOperand(0).getReg());
  if (!TRI->isAGPRClass(DefRC))
    return false;

  Register UseReg = CopyMI->getOperand(1).getReg();
  MachineInstr *RegSeq = MRI->getVRegDef(UseReg);
  if (!RegSeq || !RegSeq->isRegSequence())
    return false;

  const DebugLoc &DL = CopyMI->getDebugLoc();
  MachineBasicBlock &MBB = *CopyMI->getParent();

  MachineInstrBuilder B(*MBB.getParent(), CopyMI);
  DenseMap<TargetInstrInfo::RegSubRegPair, Register> VGPRCopies;

  const TargetRegisterClass *UseRC =
      MRI->getRegClass(CopyMI->getOperand(1).getReg());

  // Value, subregindex for new REG_SEQUENCE
  SmallVector<std::pair<MachineOperand *, unsigned>, 32> NewDefs;

  unsigned NumRegSeqOperands = RegSeq->getNumOperands();
  unsigned NumFoldable = 0;

  for (unsigned I = 1; I != NumRegSeqOperands; I += 2) {
    MachineOperand &RegOp = RegSeq->getOperand(I);
    unsigned SubRegIdx = RegSeq->getOperand(I + 1).getImm();

    if (RegOp.getSubReg()) {
      // TODO: Handle subregister compose
      NewDefs.emplace_back(&RegOp, SubRegIdx);
      continue;
    }

    MachineOperand *Lookup = lookUpCopyChain(*TII, *MRI, RegOp.getReg());
    if (!Lookup)
      Lookup = &RegOp;

    if (Lookup->isImm()) {
      // Check if this is an agpr_32 subregister.
      const TargetRegisterClass *DestSuperRC = TRI->getMatchingSuperRegClass(
          DefRC, &AMDGPU::AGPR_32RegClass, SubRegIdx);
      if (DestSuperRC &&
          TII->isInlineConstant(*Lookup, AMDGPU::OPERAND_REG_INLINE_C_INT32)) {
        ++NumFoldable;
        NewDefs.emplace_back(Lookup, SubRegIdx);
        continue;
      }
    }

    const TargetRegisterClass *InputRC =
        Lookup->isReg() ? MRI->getRegClass(Lookup->getReg())
                        : MRI->getRegClass(RegOp.getReg());

    // TODO: Account for Lookup->getSubReg()

    // If we can't find a matching super class, this is an SGPR->AGPR or
    // VGPR->AGPR subreg copy (or something constant-like we have to materialize
    // in the AGPR). We can't directly copy from SGPR to AGPR on gfx908, so we
    // want to rewrite to copy to an intermediate VGPR class.
    const TargetRegisterClass *MatchRC =
        TRI->getMatchingSuperRegClass(DefRC, InputRC, SubRegIdx);
    if (!MatchRC) {
      ++NumFoldable;
      NewDefs.emplace_back(&RegOp, SubRegIdx);
      continue;
    }

    NewDefs.emplace_back(&RegOp, SubRegIdx);
  }

  // Do not clone a reg_sequence and merely change the result register class.
  if (NumFoldable == 0)
    return false;

  CopyMI->setDesc(TII->get(AMDGPU::REG_SEQUENCE));
  for (unsigned I = CopyMI->getNumOperands() - 1; I > 0; --I)
    CopyMI->removeOperand(I);

  for (auto [Def, DestSubIdx] : NewDefs) {
    if (!Def->isReg()) {
      // TODO: Should we use single write for each repeated value like in
      // register case?
      Register Tmp = MRI->createVirtualRegister(&AMDGPU::AGPR_32RegClass);
      BuildMI(MBB, CopyMI, DL, TII->get(AMDGPU::V_ACCVGPR_WRITE_B32_e64), Tmp)
          .add(*Def);
      B.addReg(Tmp);
    } else {
      TargetInstrInfo::RegSubRegPair Src = getRegSubRegPair(*Def);
      Def->setIsKill(false);

      Register &VGPRCopy = VGPRCopies[Src];
      if (!VGPRCopy) {
        const TargetRegisterClass *VGPRUseSubRC =
            TRI->getSubRegisterClass(UseRC, DestSubIdx);

        // We cannot build a reg_sequence out of the same registers, they
        // must be copied. Better do it here before copyPhysReg() created
        // several reads to do the AGPR->VGPR->AGPR copy.

        // Direct copy from SGPR to AGPR is not possible on gfx908. To avoid
        // creation of exploded copies SGPR->VGPR->AGPR in the copyPhysReg()
        // later, create a copy here and track if we already have such a copy.
        if (TRI->getSubRegisterClass(MRI->getRegClass(Src.Reg), Src.SubReg) !=
            VGPRUseSubRC) {
          VGPRCopy = MRI->createVirtualRegister(VGPRUseSubRC);
          BuildMI(MBB, CopyMI, DL, TII->get(AMDGPU::COPY), VGPRCopy).add(*Def);
          B.addReg(VGPRCopy);
        } else {
          // If it is already a VGPR, do not copy the register.
          B.add(*Def);
        }
      } else {
        B.addReg(VGPRCopy);
      }
    }

    B.addImm(DestSubIdx);
  }

  LLVM_DEBUG(dbgs() << "Folded " << *CopyMI);
  return true;
}
```
<a name="block_19"></a>**File:** llvm/lib/Target/AMDGPU/SIFoldOperands.cpp (L1935-2050) [<sup>↩</sup>](#ref-block_19) [<sup>↩</sup>](#ref-block_19)
```cpp
bool SIFoldOperandsImpl::tryFoldFoldableCopy(
    MachineInstr &MI, MachineOperand *&CurrentKnownM0Val) const {
  Register DstReg = MI.getOperand(0).getReg();
  // Specially track simple redefs of m0 to the same value in a block, so we
  // can erase the later ones.
  if (DstReg == AMDGPU::M0) {
    MachineOperand &NewM0Val = MI.getOperand(1);
    if (CurrentKnownM0Val && CurrentKnownM0Val->isIdenticalTo(NewM0Val)) {
      MI.eraseFromParent();
      return true;
    }

    // We aren't tracking other physical registers
    CurrentKnownM0Val = (NewM0Val.isReg() && NewM0Val.getReg().isPhysical())
                            ? nullptr
                            : &NewM0Val;
    return false;
  }

  MachineOperand *OpToFoldPtr;
  if (MI.getOpcode() == AMDGPU::V_MOV_B16_t16_e64) {
    // Folding when any src_modifiers are non-zero is unsupported
    if (TII->hasAnyModifiersSet(MI))
      return false;
    OpToFoldPtr = &MI.getOperand(2);
  } else
    OpToFoldPtr = &MI.getOperand(1);
  MachineOperand &OpToFold = *OpToFoldPtr;
  bool FoldingImm = OpToFold.isImm() || OpToFold.isFI() || OpToFold.isGlobal();

  // FIXME: We could also be folding things like TargetIndexes.
  if (!FoldingImm && !OpToFold.isReg())
    return false;

  if (OpToFold.isReg() && !OpToFold.getReg().isVirtual())
    return false;

  // Prevent folding operands backwards in the function. For example,
  // the COPY opcode must not be replaced by 1 in this example:
  //
  //    %3 = COPY %vgpr0; VGPR_32:%3
  //    ...
  //    %vgpr0 = V_MOV_B32_e32 1, implicit %exec
  if (!DstReg.isVirtual())
    return false;

  const TargetRegisterClass *DstRC =
      MRI->getRegClass(MI.getOperand(0).getReg());

  // True16: Fix malformed 16-bit sgpr COPY produced by peephole-opt
  // Can remove this code if proper 16-bit SGPRs are implemented
  // Example: Pre-peephole-opt
  // %29:sgpr_lo16 = COPY %16.lo16:sreg_32
  // %32:sreg_32 = COPY %29:sgpr_lo16
  // %30:sreg_32 = S_PACK_LL_B32_B16 killed %31:sreg_32, killed %32:sreg_32
  // Post-peephole-opt and DCE
  // %32:sreg_32 = COPY %16.lo16:sreg_32
  // %30:sreg_32 = S_PACK_LL_B32_B16 killed %31:sreg_32, killed %32:sreg_32
  // After this transform
  // %32:sreg_32 = COPY %16:sreg_32
  // %30:sreg_32 = S_PACK_LL_B32_B16 killed %31:sreg_32, killed %32:sreg_32
  // After the fold operands pass
  // %30:sreg_32 = S_PACK_LL_B32_B16 killed %31:sreg_32, killed %16:sreg_32
  if (MI.getOpcode() == AMDGPU::COPY && OpToFold.isReg() &&
      OpToFold.getSubReg()) {
    if (DstRC == &AMDGPU::SReg_32RegClass &&
        DstRC == MRI->getRegClass(OpToFold.getReg())) {
      assert(OpToFold.getSubReg() == AMDGPU::lo16);
      OpToFold.setSubReg(0);
    }
  }

  // Fold copy to AGPR through reg_sequence
  // TODO: Handle with subregister extract
  if (OpToFold.isReg() && MI.isCopy() && !MI.getOperand(1).getSubReg()) {
    if (foldCopyToAGPRRegSequence(&MI))
      return true;
  }

  FoldableDef Def(OpToFold, DstRC);
  bool Changed = foldInstOperand(MI, Def);

  // If we managed to fold all uses of this copy then we might as well
  // delete it now.
  // The only reason we need to follow chains of copies here is that
  // tryFoldRegSequence looks forward through copies before folding a
  // REG_SEQUENCE into its eventual users.
  auto *InstToErase = &MI;
  while (MRI->use_nodbg_empty(InstToErase->getOperand(0).getReg())) {
    auto &SrcOp = InstToErase->getOperand(1);
    auto SrcReg = SrcOp.isReg() ? SrcOp.getReg() : Register();
    InstToErase->eraseFromParent();
    Changed = true;
    InstToErase = nullptr;
    if (!SrcReg || SrcReg.isPhysical())
      break;
    InstToErase = MRI->getVRegDef(SrcReg);
    if (!InstToErase || !TII->isFoldableCopy(*InstToErase))
      break;
  }

  if (InstToErase && InstToErase->isRegSequence() &&
      MRI->use_nodbg_empty(InstToErase->getOperand(0).getReg())) {
    InstToErase->eraseFromParent();
    Changed = true;
  }

  if (Changed)
    return true;

  // Run this after foldInstOperand to avoid turning scalar additions into
  // vector additions when the result scalar result could just be folded into
  // the user(s).
  return OpToFold.isReg() &&
         foldCopyToVGPROfScalarAddOfFrameIndex(DstReg, OpToFold.getReg(), MI);
}
```
<a name="block_20"></a>**File:** llvm/lib/Target/AMDGPU/SIFoldOperands.cpp (L2054-2100) [<sup>↩</sup>](#ref-block_20) [<sup>↩</sup>](#ref-block_20)
```cpp
const MachineOperand *
SIFoldOperandsImpl::isClamp(const MachineInstr &MI) const {
  unsigned Op = MI.getOpcode();
  switch (Op) {
  case AMDGPU::V_MAX_F32_e64:
  case AMDGPU::V_MAX_F16_e64:
  case AMDGPU::V_MAX_F16_t16_e64:
  case AMDGPU::V_MAX_F16_fake16_e64:
  case AMDGPU::V_MAX_F64_e64:
  case AMDGPU::V_MAX_NUM_F64_e64:
  case AMDGPU::V_PK_MAX_F16: {
    if (MI.mayRaiseFPException())
      return nullptr;

    if (!TII->getNamedOperand(MI, AMDGPU::OpName::clamp)->getImm())
      return nullptr;

    // Make sure sources are identical.
    const MachineOperand *Src0 = TII->getNamedOperand(MI, AMDGPU::OpName::src0);
    const MachineOperand *Src1 = TII->getNamedOperand(MI, AMDGPU::OpName::src1);
    if (!Src0->isReg() || !Src1->isReg() ||
        Src0->getReg() != Src1->getReg() ||
        Src0->getSubReg() != Src1->getSubReg() ||
        Src0->getSubReg() != AMDGPU::NoSubRegister)
      return nullptr;

    // Can't fold up if we have modifiers.
    if (TII->hasModifiersSet(MI, AMDGPU::OpName::omod))
      return nullptr;

    unsigned Src0Mods
      = TII->getNamedOperand(MI, AMDGPU::OpName::src0_modifiers)->getImm();
    unsigned Src1Mods
      = TII->getNamedOperand(MI, AMDGPU::OpName::src1_modifiers)->getImm();

    // Having a 0 op_sel_hi would require swizzling the output in the source
    // instruction, which we can't do.
    unsigned UnsetMods = (Op == AMDGPU::V_PK_MAX_F16) ? SISrcMods::OP_SEL_1
                                                      : 0u;
    if (Src0Mods != UnsetMods && Src1Mods != UnsetMods)
      return nullptr;
    return Src0;
  }
  default:
    return nullptr;
  }
}
```
<a name="block_21"></a>**File:** llvm/lib/Target/AMDGPU/SIFoldOperands.cpp (L2103-2152) [<sup>↩</sup>](#ref-block_21) [<sup>↩</sup>](#ref-block_21)
```cpp
bool SIFoldOperandsImpl::tryFoldClamp(MachineInstr &MI) {
  const MachineOperand *ClampSrc = isClamp(MI);
  if (!ClampSrc || !MRI->hasOneNonDBGUser(ClampSrc->getReg()))
    return false;

  if (!ClampSrc->getReg().isVirtual())
    return false;

  // Look through COPY. COPY only observed with True16.
  Register DefSrcReg = TRI->lookThruCopyLike(ClampSrc->getReg(), MRI);
  MachineInstr *Def =
      MRI->getVRegDef(DefSrcReg.isVirtual() ? DefSrcReg : ClampSrc->getReg());

  // The type of clamp must be compatible.
  if (TII->getClampMask(*Def) != TII->getClampMask(MI))
    return false;

  if (Def->mayRaiseFPException())
    return false;

  MachineOperand *DefClamp = TII->getNamedOperand(*Def, AMDGPU::OpName::clamp);
  if (!DefClamp)
    return false;

  LLVM_DEBUG(dbgs() << "Folding clamp " << *DefClamp << " into " << *Def);

  // Clamp is applied after omod, so it is OK if omod is set.
  DefClamp->setImm(1);

  Register DefReg = Def->getOperand(0).getReg();
  Register MIDstReg = MI.getOperand(0).getReg();
  if (TRI->isSGPRReg(*MRI, DefReg)) {
    // Pseudo scalar instructions have a SGPR for dst and clamp is a v_max*
    // instruction with a VGPR dst.
    BuildMI(*MI.getParent(), MI, MI.getDebugLoc(), TII->get(AMDGPU::COPY),
            MIDstReg)
        .addReg(DefReg);
  } else {
    MRI->replaceRegWith(MIDstReg, DefReg);
  }
  MI.eraseFromParent();

  // Use of output modifiers forces VOP3 encoding for a VOP2 mac/fmac
  // instruction, so we might as well convert it to the more flexible VOP3-only
  // mad/fma form.
  if (TII->convertToThreeAddress(*Def, nullptr, nullptr))
    Def->eraseFromParent();

  return true;
}
```
<a name="block_22"></a>**File:** llvm/lib/Target/AMDGPU/SIFoldOperands.cpp (L2203-2279) [<sup>↩</sup>](#ref-block_22) [<sup>↩</sup>](#ref-block_22)
```cpp
std::pair<const MachineOperand *, int>
SIFoldOperandsImpl::isOMod(const MachineInstr &MI) const {
  unsigned Op = MI.getOpcode();
  switch (Op) {
  case AMDGPU::V_MUL_F64_e64:
  case AMDGPU::V_MUL_F64_pseudo_e64:
  case AMDGPU::V_MUL_F32_e64:
  case AMDGPU::V_MUL_F16_t16_e64:
  case AMDGPU::V_MUL_F16_fake16_e64:
  case AMDGPU::V_MUL_F16_e64: {
    // If output denormals are enabled, omod is ignored.
    if ((Op == AMDGPU::V_MUL_F32_e64 &&
         MFI->getMode().FP32Denormals.Output != DenormalMode::PreserveSign) ||
        ((Op == AMDGPU::V_MUL_F64_e64 || Op == AMDGPU::V_MUL_F64_pseudo_e64 ||
          Op == AMDGPU::V_MUL_F16_e64 || Op == AMDGPU::V_MUL_F16_t16_e64 ||
          Op == AMDGPU::V_MUL_F16_fake16_e64) &&
         MFI->getMode().FP64FP16Denormals.Output !=
             DenormalMode::PreserveSign) ||
        MI.mayRaiseFPException())
      return std::pair(nullptr, SIOutMods::NONE);

    const MachineOperand *RegOp = nullptr;
    const MachineOperand *ImmOp = nullptr;
    const MachineOperand *Src0 = TII->getNamedOperand(MI, AMDGPU::OpName::src0);
    const MachineOperand *Src1 = TII->getNamedOperand(MI, AMDGPU::OpName::src1);
    if (Src0->isImm()) {
      ImmOp = Src0;
      RegOp = Src1;
    } else if (Src1->isImm()) {
      ImmOp = Src1;
      RegOp = Src0;
    } else
      return std::pair(nullptr, SIOutMods::NONE);

    int OMod = getOModValue(Op, ImmOp->getImm());
    if (OMod == SIOutMods::NONE ||
        TII->hasModifiersSet(MI, AMDGPU::OpName::src0_modifiers) ||
        TII->hasModifiersSet(MI, AMDGPU::OpName::src1_modifiers) ||
        TII->hasModifiersSet(MI, AMDGPU::OpName::omod) ||
        TII->hasModifiersSet(MI, AMDGPU::OpName::clamp))
      return std::pair(nullptr, SIOutMods::NONE);

    return std::pair(RegOp, OMod);
  }
  case AMDGPU::V_ADD_F64_e64:
  case AMDGPU::V_ADD_F64_pseudo_e64:
  case AMDGPU::V_ADD_F32_e64:
  case AMDGPU::V_ADD_F16_e64:
  case AMDGPU::V_ADD_F16_t16_e64:
  case AMDGPU::V_ADD_F16_fake16_e64: {
    // If output denormals are enabled, omod is ignored.
    if ((Op == AMDGPU::V_ADD_F32_e64 &&
         MFI->getMode().FP32Denormals.Output != DenormalMode::PreserveSign) ||
        ((Op == AMDGPU::V_ADD_F64_e64 || Op == AMDGPU::V_ADD_F64_pseudo_e64 ||
          Op == AMDGPU::V_ADD_F16_e64 || Op == AMDGPU::V_ADD_F16_t16_e64 ||
          Op == AMDGPU::V_ADD_F16_fake16_e64) &&
         MFI->getMode().FP64FP16Denormals.Output != DenormalMode::PreserveSign))
      return std::pair(nullptr, SIOutMods::NONE);

    // Look through the DAGCombiner canonicalization fmul x, 2 -> fadd x, x
    const MachineOperand *Src0 = TII->getNamedOperand(MI, AMDGPU::OpName::src0);
    const MachineOperand *Src1 = TII->getNamedOperand(MI, AMDGPU::OpName::src1);

    if (Src0->isReg() && Src1->isReg() && Src0->getReg() == Src1->getReg() &&
        Src0->getSubReg() == Src1->getSubReg() &&
        !TII->hasModifiersSet(MI, AMDGPU::OpName::src0_modifiers) &&
        !TII->hasModifiersSet(MI, AMDGPU::OpName::src1_modifiers) &&
        !TII->hasModifiersSet(MI, AMDGPU::OpName::clamp) &&
        !TII->hasModifiersSet(MI, AMDGPU::OpName::omod))
      return std::pair(Src0, SIOutMods::MUL2);

    return std::pair(nullptr, SIOutMods::NONE);
  }
  default:
    return std::pair(nullptr, SIOutMods::NONE);
  }
}
```
<a name="block_23"></a>**File:** llvm/lib/Target/AMDGPU/SIFoldOperands.cpp (L2282-2320) [<sup>↩</sup>](#ref-block_23) [<sup>↩</sup>](#ref-block_23)
```cpp
bool SIFoldOperandsImpl::tryFoldOMod(MachineInstr &MI) {
  const MachineOperand *RegOp;
  int OMod;
  std::tie(RegOp, OMod) = isOMod(MI);
  if (OMod == SIOutMods::NONE || !RegOp->isReg() ||
      RegOp->getSubReg() != AMDGPU::NoSubRegister ||
      !MRI->hasOneNonDBGUser(RegOp->getReg()))
    return false;

  MachineInstr *Def = MRI->getVRegDef(RegOp->getReg());
  MachineOperand *DefOMod = TII->getNamedOperand(*Def, AMDGPU::OpName::omod);
  if (!DefOMod || DefOMod->getImm() != SIOutMods::NONE)
    return false;

  if (Def->mayRaiseFPException())
    return false;

  // Clamp is applied after omod. If the source already has clamp set, don't
  // fold it.
  if (TII->hasModifiersSet(*Def, AMDGPU::OpName::clamp))
    return false;

  LLVM_DEBUG(dbgs() << "Folding omod " << MI << " into " << *Def);

  DefOMod->setImm(OMod);
  MRI->replaceRegWith(MI.getOperand(0).getReg(), Def->getOperand(0).getReg());
  // Kill flags can be wrong if we replaced a def inside a loop with a def
  // outside the loop.
  MRI->clearKillFlags(Def->getOperand(0).getReg());
  MI.eraseFromParent();

  // Use of output modifiers forces VOP3 encoding for a VOP2 mac/fmac
  // instruction, so we might as well convert it to the more flexible VOP3-only
  // mad/fma form.
  if (TII->convertToThreeAddress(*Def, nullptr, nullptr))
    Def->eraseFromParent();

  return true;
}
```
<a name="block_24"></a>**File:** llvm/lib/Target/AMDGPU/SIFoldOperands.cpp (L2324-2400) [<sup>↩</sup>](#ref-block_24) [<sup>↩</sup>](#ref-block_24)
```cpp
bool SIFoldOperandsImpl::tryFoldRegSequence(MachineInstr &MI) {
  assert(MI.isRegSequence());
  auto Reg = MI.getOperand(0).getReg();

  if (!ST->hasGFX90AInsts() || !TRI->isVGPR(*MRI, Reg) ||
      !MRI->hasOneNonDBGUse(Reg))
    return false;

  SmallVector<std::pair<MachineOperand*, unsigned>, 32> Defs;
  if (!getRegSeqInit(Defs, Reg))
    return false;

  for (auto &[Op, SubIdx] : Defs) {
    if (!Op->isReg())
      return false;
    if (TRI->isAGPR(*MRI, Op->getReg()))
      continue;
    // Maybe this is a COPY from AREG
    const MachineInstr *SubDef = MRI->getVRegDef(Op->getReg());
    if (!SubDef || !SubDef->isCopy() || SubDef->getOperand(1).getSubReg())
      return false;
    if (!TRI->isAGPR(*MRI, SubDef->getOperand(1).getReg()))
      return false;
  }

  MachineOperand *Op = &*MRI->use_nodbg_begin(Reg);
  MachineInstr *UseMI = Op->getParent();
  while (UseMI->isCopy() && !Op->getSubReg()) {
    Reg = UseMI->getOperand(0).getReg();
    if (!TRI->isVGPR(*MRI, Reg) || !MRI->hasOneNonDBGUse(Reg))
      return false;
    Op = &*MRI->use_nodbg_begin(Reg);
    UseMI = Op->getParent();
  }

  if (Op->getSubReg())
    return false;

  unsigned OpIdx = Op - &UseMI->getOperand(0);
  const MCInstrDesc &InstDesc = UseMI->getDesc();
  const TargetRegisterClass *OpRC =
      TII->getRegClass(InstDesc, OpIdx, TRI, *MI.getMF());
  if (!OpRC || !TRI->isVectorSuperClass(OpRC))
    return false;

  const auto *NewDstRC = TRI->getEquivalentAGPRClass(MRI->getRegClass(Reg));
  auto Dst = MRI->createVirtualRegister(NewDstRC);
  auto RS = BuildMI(*MI.getParent(), MI, MI.getDebugLoc(),
                    TII->get(AMDGPU::REG_SEQUENCE), Dst);

  for (auto &[Def, SubIdx] : Defs) {
    Def->setIsKill(false);
    if (TRI->isAGPR(*MRI, Def->getReg())) {
      RS.add(*Def);
    } else { // This is a copy
      MachineInstr *SubDef = MRI->getVRegDef(Def->getReg());
      SubDef->getOperand(1).setIsKill(false);
      RS.addReg(SubDef->getOperand(1).getReg(), 0, Def->getSubReg());
    }
    RS.addImm(SubIdx);
  }

  Op->setReg(Dst);
  if (!TII->isOperandLegal(*UseMI, OpIdx, Op)) {
    Op->setReg(Reg);
    RS->eraseFromParent();
    return false;
  }

  LLVM_DEBUG(dbgs() << "Folded " << *RS << " into " << *UseMI);

  // Erase the REG_SEQUENCE eagerly, unless we followed a chain of COPY users,
  // in which case we can erase them all later in runOnMachineFunction.
  if (MRI->use_nodbg_empty(MI.getOperand(0).getReg()))
    MI.eraseFromParent();
  return true;
}
```
<a name="block_25"></a>**File:** llvm/lib/Target/AMDGPU/SIFoldOperands.cpp (L2471-2572) [<sup>↩</sup>](#ref-block_25) [<sup>↩</sup>](#ref-block_25)
```cpp
bool SIFoldOperandsImpl::tryFoldPhiAGPR(MachineInstr &PHI) {
  assert(PHI.isPHI());

  Register PhiOut = PHI.getOperand(0).getReg();
  if (!TRI->isVGPR(*MRI, PhiOut))
    return false;

  // Iterate once over all incoming values of the PHI to check if this PHI is
  // eligible, and determine the exact AGPR RC we'll target.
  const TargetRegisterClass *ARC = nullptr;
  for (unsigned K = 1; K < PHI.getNumExplicitOperands(); K += 2) {
    MachineOperand &MO = PHI.getOperand(K);
    MachineInstr *Copy = MRI->getVRegDef(MO.getReg());
    if (!Copy || !Copy->isCopy())
      continue;

    Register AGPRSrc;
    unsigned AGPRRegMask = AMDGPU::NoSubRegister;
    if (!isAGPRCopy(*TRI, *MRI, *Copy, AGPRSrc, AGPRRegMask))
      continue;

    const TargetRegisterClass *CopyInRC = MRI->getRegClass(AGPRSrc);
    if (const auto *SubRC = TRI->getSubRegisterClass(CopyInRC, AGPRRegMask))
      CopyInRC = SubRC;

    if (ARC && !ARC->hasSubClassEq(CopyInRC))
      return false;
    ARC = CopyInRC;
  }

  if (!ARC)
    return false;

  bool IsAGPR32 = (ARC == &AMDGPU::AGPR_32RegClass);

  // Rewrite the PHI's incoming values to ARC.
  LLVM_DEBUG(dbgs() << "Folding AGPR copies into: " << PHI);
  for (unsigned K = 1; K < PHI.getNumExplicitOperands(); K += 2) {
    MachineOperand &MO = PHI.getOperand(K);
    Register Reg = MO.getReg();

    MachineBasicBlock::iterator InsertPt;
    MachineBasicBlock *InsertMBB = nullptr;

    // Look at the def of Reg, ignoring all copies.
    unsigned CopyOpc = AMDGPU::COPY;
    if (MachineInstr *Def = MRI->getVRegDef(Reg)) {

      // Look at pre-existing COPY instructions from ARC: Steal the operand. If
      // the copy was single-use, it will be removed by DCE later.
      if (Def->isCopy()) {
        Register AGPRSrc;
        unsigned AGPRSubReg = AMDGPU::NoSubRegister;
        if (isAGPRCopy(*TRI, *MRI, *Def, AGPRSrc, AGPRSubReg)) {
          MO.setReg(AGPRSrc);
          MO.setSubReg(AGPRSubReg);
          continue;
        }

        // If this is a multi-use SGPR -> VGPR copy, use V_ACCVGPR_WRITE on
        // GFX908 directly instead of a COPY. Otherwise, SIFoldOperand may try
        // to fold the sgpr -> vgpr -> agpr copy into a sgpr -> agpr copy which
        // is unlikely to be profitable.
        //
        // Note that V_ACCVGPR_WRITE is only used for AGPR_32.
        MachineOperand &CopyIn = Def->getOperand(1);
        if (IsAGPR32 && !ST->hasGFX90AInsts() && !MRI->hasOneNonDBGUse(Reg) &&
            TRI->isSGPRReg(*MRI, CopyIn.getReg()))
          CopyOpc = AMDGPU::V_ACCVGPR_WRITE_B32_e64;
      }

      InsertMBB = Def->getParent();
      InsertPt = InsertMBB->SkipPHIsLabelsAndDebug(++Def->getIterator());
    } else {
      InsertMBB = PHI.getOperand(MO.getOperandNo() + 1).getMBB();
      InsertPt = InsertMBB->getFirstTerminator();
    }

    Register NewReg = MRI->createVirtualRegister(ARC);
    MachineInstr *MI = BuildMI(*InsertMBB, InsertPt, PHI.getDebugLoc(),
                               TII->get(CopyOpc), NewReg)
                           .addReg(Reg);
    MO.setReg(NewReg);

    (void)MI;
    LLVM_DEBUG(dbgs() << "  Created COPY: " << *MI);
  }

  // Replace the PHI's result with a new register.
  Register NewReg = MRI->createVirtualRegister(ARC);
  PHI.getOperand(0).setReg(NewReg);

  // COPY that new register back to the original PhiOut register. This COPY will
  // usually be folded out later.
  MachineBasicBlock *MBB = PHI.getParent();
  BuildMI(*MBB, MBB->getFirstNonPHI(), PHI.getDebugLoc(),
          TII->get(AMDGPU::COPY), PhiOut)
      .addReg(NewReg);

  LLVM_DEBUG(dbgs() << "  Done: Folded " << PHI);
  return true;
}
```
<a name="block_26"></a>**File:** llvm/lib/Target/AMDGPU/SIFoldOperands.cpp (L2575-2627) [<sup>↩</sup>](#ref-block_26) [<sup>↩</sup>](#ref-block_26)
```cpp
bool SIFoldOperandsImpl::tryFoldLoad(MachineInstr &MI) {
  assert(MI.mayLoad());
  if (!ST->hasGFX90AInsts() || MI.getNumExplicitDefs() != 1)
    return false;

  MachineOperand &Def = MI.getOperand(0);
  if (!Def.isDef())
    return false;

  Register DefReg = Def.getReg();

  if (DefReg.isPhysical() || !TRI->isVGPR(*MRI, DefReg))
    return false;

  SmallVector<const MachineInstr *, 8> Users(
      llvm::make_pointer_range(MRI->use_nodbg_instructions(DefReg)));
  SmallVector<Register, 8> MoveRegs;

  if (Users.empty())
    return false;

  // Check that all uses a copy to an agpr or a reg_sequence producing an agpr.
  while (!Users.empty()) {
    const MachineInstr *I = Users.pop_back_val();
    if (!I->isCopy() && !I->isRegSequence())
      return false;
    Register DstReg = I->getOperand(0).getReg();
    // Physical registers may have more than one instruction definitions
    if (DstReg.isPhysical())
      return false;
    if (TRI->isAGPR(*MRI, DstReg))
      continue;
    MoveRegs.push_back(DstReg);
    for (const MachineInstr &U : MRI->use_nodbg_instructions(DstReg))
      Users.push_back(&U);
  }

  const TargetRegisterClass *RC = MRI->getRegClass(DefReg);
  MRI->setRegClass(DefReg, TRI->getEquivalentAGPRClass(RC));
  if (!TII->isOperandLegal(MI, 0, &Def)) {
    MRI->setRegClass(DefReg, RC);
    return false;
  }

  while (!MoveRegs.empty()) {
    Register Reg = MoveRegs.pop_back_val();
    MRI->setRegClass(Reg, TRI->getEquivalentAGPRClass(MRI->getRegClass(Reg)));
  }

  LLVM_DEBUG(dbgs() << "Folded " << MI);

  return true;
}
```
<a name="block_27"></a>**File:** llvm/lib/Target/AMDGPU/SIFoldOperands.cpp (L2661-2724) [<sup>↩</sup>](#ref-block_27) [<sup>↩</sup>](#ref-block_27)
```cpp
bool SIFoldOperandsImpl::tryOptimizeAGPRPhis(MachineBasicBlock &MBB) {
  // This is only really needed on GFX908 where AGPR-AGPR copies are
  // unreasonably difficult.
  if (ST->hasGFX90AInsts())
    return false;

  // Look at all AGPR Phis and collect the register + subregister used.
  DenseMap<std::pair<Register, unsigned>, std::vector<MachineOperand *>>
      RegToMO;

  for (auto &MI : MBB) {
    if (!MI.isPHI())
      break;

    if (!TRI->isAGPR(*MRI, MI.getOperand(0).getReg()))
      continue;

    for (unsigned K = 1; K < MI.getNumOperands(); K += 2) {
      MachineOperand &PhiMO = MI.getOperand(K);
      if (!PhiMO.getSubReg())
        continue;
      RegToMO[{PhiMO.getReg(), PhiMO.getSubReg()}].push_back(&PhiMO);
    }
  }

  // For all (Reg, SubReg) pair that are used more than once, cache the value in
  // a VGPR.
  bool Changed = false;
  for (const auto &[Entry, MOs] : RegToMO) {
    if (MOs.size() == 1)
      continue;

    const auto [Reg, SubReg] = Entry;
    MachineInstr *Def = MRI->getVRegDef(Reg);
    MachineBasicBlock *DefMBB = Def->getParent();

    // Create a copy in a VGPR using V_ACCVGPR_READ_B32_e64 so it's not folded
    // out.
    const TargetRegisterClass *ARC = getRegOpRC(*MRI, *TRI, *MOs.front());
    Register TempVGPR =
        MRI->createVirtualRegister(TRI->getEquivalentVGPRClass(ARC));
    MachineInstr *VGPRCopy =
        BuildMI(*DefMBB, ++Def->getIterator(), Def->getDebugLoc(),
                TII->get(AMDGPU::V_ACCVGPR_READ_B32_e64), TempVGPR)
            .addReg(Reg, /* flags */ 0, SubReg);

    // Copy back to an AGPR and use that instead of the AGPR subreg in all MOs.
    Register TempAGPR = MRI->createVirtualRegister(ARC);
    BuildMI(*DefMBB, ++VGPRCopy->getIterator(), Def->getDebugLoc(),
            TII->get(AMDGPU::COPY), TempAGPR)
        .addReg(TempVGPR);

    LLVM_DEBUG(dbgs() << "Caching AGPR into VGPR: " << *VGPRCopy);
    for (MachineOperand *MO : MOs) {
      MO->setReg(TempAGPR);
      MO->setSubReg(AMDGPU::NoSubRegister);
      LLVM_DEBUG(dbgs() << "  Changed PHI Operand: " << *MO << "\n");
    }

    Changed = true;
  }

  return Changed;
}
```
<a name="block_28"></a>**File:** llvm/lib/Target/AMDGPU/SIFoldOperands.cpp (L2726-2786) [<sup>↩</sup>](#ref-block_28) [<sup>↩</sup>](#ref-block_28)
```cpp
bool SIFoldOperandsImpl::run(MachineFunction &MF) {
  MRI = &MF.getRegInfo();
  ST = &MF.getSubtarget<GCNSubtarget>();
  TII = ST->getInstrInfo();
  TRI = &TII->getRegisterInfo();
  MFI = MF.getInfo<SIMachineFunctionInfo>();

  // omod is ignored by hardware if IEEE bit is enabled. omod also does not
  // correctly handle signed zeros.
  //
  // FIXME: Also need to check strictfp
  bool IsIEEEMode = MFI->getMode().IEEE;
  bool HasNSZ = MFI->hasNoSignedZerosFPMath();

  bool Changed = false;
  for (MachineBasicBlock *MBB : depth_first(&MF)) {
    MachineOperand *CurrentKnownM0Val = nullptr;
    for (auto &MI : make_early_inc_range(*MBB)) {
      Changed |= tryFoldCndMask(MI);

      if (tryFoldZeroHighBits(MI)) {
        Changed = true;
        continue;
      }

      if (MI.isRegSequence() && tryFoldRegSequence(MI)) {
        Changed = true;
        continue;
      }

      if (MI.isPHI() && tryFoldPhiAGPR(MI)) {
        Changed = true;
        continue;
      }

      if (MI.mayLoad() && tryFoldLoad(MI)) {
        Changed = true;
        continue;
      }

      if (TII->isFoldableCopy(MI)) {
        Changed |= tryFoldFoldableCopy(MI, CurrentKnownM0Val);
        continue;
      }

      // Saw an unknown clobber of m0, so we no longer know what it is.
      if (CurrentKnownM0Val && MI.modifiesRegister(AMDGPU::M0, TRI))
        CurrentKnownM0Val = nullptr;

      // TODO: Omod might be OK if there is NSZ only on the source
      // instruction, and not the omod multiply.
      if (IsIEEEMode || (!HasNSZ && !MI.getFlag(MachineInstr::FmNsz)) ||
          !tryFoldOMod(MI))
        Changed |= tryFoldClamp(MI);
    }

    Changed |= tryOptimizeAGPRPhis(*MBB);
  }

  return Changed;
}
```


# SIFoldOperands.cpp 代码功能详解 v2

## 1. Pass的主要功能概述

<a name="ref-block_31"></a>`SIFoldOperands` 是一个针对 AMD GPU 架构的机器函数优化 Pass，其主要功能是**操作数折叠（Operand Folding）优化**。 llvm-project:276-298[<sup>↗</sup>](#block_31) 

**作用与效果：**
- 将立即数、帧索引、寄存器值等折叠到使用这些值的指令中
- 消除冗余的 COPY 和 MOV 指令
- 减少寄存器压力和指令数量
- 针对 AMDGPU 的特定硬件特性进行优化（如 AGPR/VGPR/SGPR 之间的转换）
- 利用硬件的输出修饰符（omod）和钳位（clamp）特性

## 2. 主要功能步骤和子功能提取

<a name="ref-block_29"></a>### 核心数据结构 llvm-project:33-147[<sup>↗</sup>](#block_29) llvm-project:149-177 

### 主要子功能模块

<a name="ref-block_28"></a>1. **主运行函数** - `run()` llvm-project:2726-2786[<sup>↗</sup>](#block_28) 

<a name="ref-block_19"></a>2. **可折叠拷贝处理** - `tryFoldFoldableCopy()` llvm-project:1935-2050[<sup>↗</sup>](#block_19) 

<a name="ref-block_17"></a>3. **指令操作数折叠** - `foldInstOperand()` llvm-project:1722-1801[<sup>↗</sup>](#block_17) 

<a name="ref-block_12"></a>4. **操作数折叠核心** - `foldOperand()` llvm-project:1137-1445[<sup>↗</sup>](#block_12) 

<a name="ref-block_14"></a>5. **常量折叠** - `tryConstantFoldOp()` llvm-project:1552-1655[<sup>↗</sup>](#block_14) 

<a name="ref-block_15"></a>6. **条件掩码折叠** - `tryFoldCndMask()` llvm-project:1658-1698[<sup>↗</sup>](#block_15) 

<a name="ref-block_16"></a>7. **高位清零折叠** - `tryFoldZeroHighBits()` llvm-project:1700-1720[<sup>↗</sup>](#block_16) 

<a name="ref-block_21"></a>8. **钳位操作折叠** - `tryFoldClamp()` llvm-project:2103-2152[<sup>↗</sup>](#block_21) 

<a name="ref-block_23"></a>9. **输出修饰符折叠** - `tryFoldOMod()` llvm-project:2282-2320[<sup>↗</sup>](#block_23) 

<a name="ref-block_24"></a>10. **寄存器序列折叠** - `tryFoldRegSequence()` llvm-project:2324-2400[<sup>↗</sup>](#block_24) 

<a name="ref-block_25"></a>11. **AGPR PHI 折叠** - `tryFoldPhiAGPR()` llvm-project:2471-2572[<sup>↗</sup>](#block_25) 

<a name="ref-block_26"></a>12. **加载指令折叠** - `tryFoldLoad()` llvm-project:2575-2627[<sup>↗</sup>](#block_26) 

<a name="ref-block_27"></a>13. **AGPR PHI 优化** - `tryOptimizeAGPRPhis()` llvm-project:2661-2724[<sup>↗</sup>](#block_27) 

## 3. 各子功能的具体描述分析

### 3.1 主运行函数 (run)
按深度优先遍历基本块，对每条指令尝试各种折叠优化，包括条件掩码、高位清零、寄存器序列、PHI 节点、加载指令和可折叠拷贝。还会根据 IEEE 模式和 NSZ 标志决定是否应用输出修饰符和钳位优化。

### 3.2 可折叠拷贝处理 (tryFoldFoldableCopy)
处理 COPY 和 MOV 指令，将立即数、帧索引或寄存器折叠到使用点。特殊处理包括：
- M0 寄存器的冗余定义消除
- AGPR 寄存器序列的折叠
- True16 指令的特殊处理
<a name="ref-block_2"></a>- 标量加法与帧索引的折叠 llvm-project:384-449[<sup>↗</sup>](#block_2) 

### 3.3 指令操作数折叠 (foldInstOperand)
主要折叠逻辑，遍历目标寄存器的所有使用点，尝试将源操作数折叠进去。如果是立即数，还会尝试常量折叠。处理子寄存器的情况。

### 3.4 操作数折叠核心 (foldOperand)
根据操作数类型（立即数、帧索引、寄存器）进行不同的折叠策略：
- REG_SEQUENCE 的特殊处理
- 内联常量的折叠
- 帧索引在 MUBUF/FLAT 指令中的折叠
- COPY 指令的转换为 MOV
- READFIRSTLANE/READLANE 的优化 llvm-project:1360-1408 

### 3.5 常量折叠 (tryConstantFoldOp)
<a name="ref-block_35"></a>对二元运算指令进行常量折叠，支持的操作包括 AND、OR、XOR、逻辑移位等。特殊处理恒等元素（如 `x OR 0 = x`，`x AND -1 = x`）。 llvm-project:1447-1511[<sup>↗</sup>](#block_35) 

### 3.6 条件掩码折叠 (tryFoldCndMask)
将 `V_CNDMASK` 指令在两个源操作数相同时优化为简单的 COPY 或 MOV。

### 3.7 高位清零折叠 (tryFoldZeroHighBits)
识别 `V_AND_B32` 与 `0xffff` 的模式，如果源指令已经将高 16 位清零，则消除 AND 操作。

### 3.8 钳位操作折叠 (tryFoldClamp)
<a name="ref-block_20"></a>识别钳位模式（`v_max_* x, x` with clamp），将钳位属性折叠到源指令中，消除额外的 max 指令。 llvm-project:2054-2100[<sup>↗</sup>](#block_20) 

### 3.9 输出修饰符折叠 (tryFoldOMod)
<a name="ref-block_22"></a>识别乘法或加法模式（如 `x * 2.0`、`x + x`），将其折叠为源指令的输出修饰符（omod），减少一条乘法/加法指令。 llvm-project:2203-2279[<sup>↗</sup>](#block_22) 

### 3.10 寄存器序列折叠 (tryFoldRegSequence)
在 GFX90A+ 上，将 VGPR 的 REG_SEQUENCE（包含 AGPR 输入）折叠到可以直接接受 AGPR 的指令（如存储指令）中。

### 3.11 AGPR PHI 折叠 (tryFoldPhiAGPR)
将 AGPR 到 VGPR 的拷贝提升到 PHI 节点之上，使 PHI 直接操作 AGPR，允许后续更好的折叠机会。 llvm-project:2444-2471 

### 3.12 加载指令折叠 (tryFoldLoad)
在 GFX90A+ 上，如果 VGPR 加载的结果只被拷贝到 AGPR 使用，则直接将加载转换为 AGPR 加载。

### 3.13 AGPR PHI 优化 (tryOptimizeAGPRPhis)
针对 GFX908 的特殊优化，缓存在多个 AGPR PHI 中重复使用的 AGPR 子寄存器到 VGPR，避免产生大量昂贵的 AGPR-AGPR 拷贝。 llvm-project:2629-2661 

## 4. 步骤/子功能之间的关系

### 层次关系
```
run() [主入口]
├── tryFoldCndMask()
├── tryFoldZeroHighBits()
├── tryFoldRegSequence()
├── tryFoldPhiAGPR()
├── tryFoldLoad()
├── tryFoldFoldableCopy()
│   ├── foldCopyToVGPROfScalarAddOfFrameIndex()
│   ├── foldCopyToAGPRRegSequence()
│   └── foldInstOperand()
│       ├── tryConstantFoldOp()
│       └── foldOperand()
│           ├── tryToFoldACImm()
│           ├── frameIndexMayFold()
│           └── tryAddToFoldList()
│               └── updateOperand()
│                   └── tryFoldImmWithOpSel()
├── tryFoldOMod()
├── tryFoldClamp()
└── tryOptimizeAGPRPhis()
```

### 执行顺序关系 llvm-project:2743-2783 

1. **优先级高的优化**（能直接修改/删除指令）：
   - `tryFoldCndMask` - 简化条件选择
   - `tryFoldZeroHighBits` - 消除冗余的 AND
   - `tryFoldRegSequence` - 寄存器序列优化
   - `tryFoldPhiAGPR` - PHI 节点提升
   - `tryFoldLoad` - 加载类型转换

2. **中间优化**（可折叠拷贝处理）：
   - `tryFoldFoldableCopy` - 这是核心的折叠入口，会递归调用其他折叠函数

3. **后续优化**（利用硬件特性）：
   - `tryFoldOMod` / `tryFoldClamp` - 输出修饰符优化（受 IEEE 模式和 NSZ 标志控制）

4. **基本块级优化**：
   - `tryOptimizeAGPRPhis` - 整个基本块的 PHI 优化

<a name="ref-block_6"></a>### 数据流关系 llvm-project:740-895[<sup>↗</sup>](#block_6) 

- `FoldableDef` 结构体封装了可折叠的定义（立即数、寄存器、帧索引等）
- `FoldCandidate` 结构体记录折叠候选（使用指令、操作数索引、是否需要缩减等）
- 折叠过程通过 `FoldList` 累积候选，最后批量应用
- `CopiesToReplace` 跟踪需要添加隐式使用的新 MOV 指令

## Notes

该 Pass 是 AMDGPU 后端优化的关键组件，针对 AMD GPU 的寄存器架构（SGPR/VGPR/AGPR）和指令特性进行了深度定制。特别关注：
- GFX908 和 GFX90A 之间的差异（AGPR 处理不同）
- 帧索引折叠到内存访问指令的地址计算中
- 利用硬件的输出修饰符避免额外的算术运算
- PHI 节点的 AGPR 优化以减少寄存器拷贝开销
### Citations
<a name="block_29"></a>**File:** llvm/lib/Target/AMDGPU/SIFoldOperands.cpp (L33-147) [<sup>↩</sup>](#ref-block_29)
```cpp
/// Track a value we may want to fold into downstream users, applying
/// subregister extracts along the way.
struct FoldableDef {
  union {
    MachineOperand *OpToFold = nullptr;
    uint64_t ImmToFold;
    int FrameIndexToFold;
  };

  /// Register class of the originally defined value.
  const TargetRegisterClass *DefRC = nullptr;

  /// Track the original defining instruction for the value.
  const MachineInstr *DefMI = nullptr;

  /// Subregister to apply to the value at the use point.
  unsigned DefSubReg = AMDGPU::NoSubRegister;

  /// Kind of value stored in the union.
  MachineOperand::MachineOperandType Kind;

  FoldableDef() = delete;
  FoldableDef(MachineOperand &FoldOp, const TargetRegisterClass *DefRC,
              unsigned DefSubReg = AMDGPU::NoSubRegister)
      : DefRC(DefRC), DefSubReg(DefSubReg), Kind(FoldOp.getType()) {

    if (FoldOp.isImm()) {
      ImmToFold = FoldOp.getImm();
    } else if (FoldOp.isFI()) {
      FrameIndexToFold = FoldOp.getIndex();
    } else {
      assert(FoldOp.isReg() || FoldOp.isGlobal());
      OpToFold = &FoldOp;
    }

    DefMI = FoldOp.getParent();
  }

  FoldableDef(int64_t FoldImm, const TargetRegisterClass *DefRC,
              unsigned DefSubReg = AMDGPU::NoSubRegister)
      : ImmToFold(FoldImm), DefRC(DefRC), DefSubReg(DefSubReg),
        Kind(MachineOperand::MO_Immediate) {}

  /// Copy the current def and apply \p SubReg to the value.
  FoldableDef getWithSubReg(const SIRegisterInfo &TRI, unsigned SubReg) const {
    FoldableDef Copy(*this);
    Copy.DefSubReg = TRI.composeSubRegIndices(DefSubReg, SubReg);
    return Copy;
  }

  bool isReg() const { return Kind == MachineOperand::MO_Register; }

  Register getReg() const {
    assert(isReg());
    return OpToFold->getReg();
  }

  unsigned getSubReg() const {
    assert(isReg());
    return OpToFold->getSubReg();
  }

  bool isImm() const { return Kind == MachineOperand::MO_Immediate; }

  bool isFI() const {
    return Kind == MachineOperand::MO_FrameIndex;
  }

  int getFI() const {
    assert(isFI());
    return FrameIndexToFold;
  }

  bool isGlobal() const { return Kind == MachineOperand::MO_GlobalAddress; }

  /// Return the effective immediate value defined by this instruction, after
  /// application of any subregister extracts which may exist between the use
  /// and def instruction.
  std::optional<int64_t> getEffectiveImmVal() const {
    assert(isImm());
    return SIInstrInfo::extractSubregFromImm(ImmToFold, DefSubReg);
  }

  /// Check if it is legal to fold this effective value into \p MI's \p OpNo
  /// operand.
  bool isOperandLegal(const SIInstrInfo &TII, const MachineInstr &MI,
                      unsigned OpIdx) const {
    switch (Kind) {
    case MachineOperand::MO_Immediate: {
      std::optional<int64_t> ImmToFold = getEffectiveImmVal();
      if (!ImmToFold)
        return false;

      // TODO: Should verify the subregister index is supported by the class
      // TODO: Avoid the temporary MachineOperand
      MachineOperand TmpOp = MachineOperand::CreateImm(*ImmToFold);
      return TII.isOperandLegal(MI, OpIdx, &TmpOp);
    }
    case MachineOperand::MO_FrameIndex: {
      if (DefSubReg != AMDGPU::NoSubRegister)
        return false;
      MachineOperand TmpOp = MachineOperand::CreateFI(FrameIndexToFold);
      return TII.isOperandLegal(MI, OpIdx, &TmpOp);
    }
    default:
      // TODO: Try to apply DefSubReg, for global address we can extract
      // low/high.
      if (DefSubReg != AMDGPU::NoSubRegister)
        return false;
      return TII.isOperandLegal(MI, OpIdx, OpToFold);
    }

    llvm_unreachable("covered MachineOperand kind switch");
  }
};
```
<a name="block_30"></a>**File:** llvm/lib/Target/AMDGPU/SIFoldOperands.cpp (L149-177)
```cpp
struct FoldCandidate {
  MachineInstr *UseMI;
  FoldableDef Def;
  int ShrinkOpcode;
  unsigned UseOpNo;
  bool Commuted;

  FoldCandidate(MachineInstr *MI, unsigned OpNo, FoldableDef Def,
                bool Commuted = false, int ShrinkOp = -1)
      : UseMI(MI), Def(Def), ShrinkOpcode(ShrinkOp), UseOpNo(OpNo),
        Commuted(Commuted) {}

  bool isFI() const { return Def.isFI(); }

  int getFI() const {
    assert(isFI());
    return Def.FrameIndexToFold;
  }

  bool isImm() const { return Def.isImm(); }

  bool isReg() const { return Def.isReg(); }

  Register getReg() const { return Def.getReg(); }

  bool isGlobal() const { return Def.isGlobal(); }

  bool needsShrink() const { return ShrinkOpcode != -1; }
};
```
<a name="block_31"></a>**File:** llvm/lib/Target/AMDGPU/SIFoldOperands.cpp (L276-298) [<sup>↩</sup>](#ref-block_31)
```cpp
class SIFoldOperandsLegacy : public MachineFunctionPass {
public:
  static char ID;

  SIFoldOperandsLegacy() : MachineFunctionPass(ID) {}

  bool runOnMachineFunction(MachineFunction &MF) override {
    if (skipFunction(MF.getFunction()))
      return false;
    return SIFoldOperandsImpl().run(MF);
  }

  StringRef getPassName() const override { return "SI Fold Operands"; }

  void getAnalysisUsage(AnalysisUsage &AU) const override {
    AU.setPreservesCFG();
    MachineFunctionPass::getAnalysisUsage(AU);
  }

  MachineFunctionProperties getRequiredProperties() const override {
    return MachineFunctionProperties().setIsSSA();
  }
};
```
<a name="block_32"></a>**File:** llvm/lib/Target/AMDGPU/SIFoldOperands.cpp (L384-449)
```cpp
bool SIFoldOperandsImpl::foldCopyToVGPROfScalarAddOfFrameIndex(
    Register DstReg, Register SrcReg, MachineInstr &MI) const {
  if (TRI->isVGPR(*MRI, DstReg) && TRI->isSGPRReg(*MRI, SrcReg) &&
      MRI->hasOneNonDBGUse(SrcReg)) {
    MachineInstr *Def = MRI->getVRegDef(SrcReg);
    if (!Def || Def->getNumOperands() != 4)
      return false;

    MachineOperand *Src0 = &Def->getOperand(1);
    MachineOperand *Src1 = &Def->getOperand(2);

    // TODO: This is profitable with more operand types, and for more
    // opcodes. But ultimately this is working around poor / nonexistent
    // regbankselect.
    if (!Src0->isFI() && !Src1->isFI())
      return false;

    if (Src0->isFI())
      std::swap(Src0, Src1);

    const bool UseVOP3 = !Src0->isImm() || TII->isInlineConstant(*Src0);
    unsigned NewOp = convertToVALUOp(Def->getOpcode(), UseVOP3);
    if (NewOp == AMDGPU::INSTRUCTION_LIST_END ||
        !Def->getOperand(3).isDead()) // Check if scc is dead
      return false;

    MachineBasicBlock *MBB = Def->getParent();
    const DebugLoc &DL = Def->getDebugLoc();
    if (NewOp != AMDGPU::V_ADD_CO_U32_e32) {
      MachineInstrBuilder Add =
          BuildMI(*MBB, *Def, DL, TII->get(NewOp), DstReg);

      if (Add->getDesc().getNumDefs() == 2) {
        Register CarryOutReg = MRI->createVirtualRegister(TRI->getBoolRC());
        Add.addDef(CarryOutReg, RegState::Dead);
        MRI->setRegAllocationHint(CarryOutReg, 0, TRI->getVCC());
      }

      Add.add(*Src0).add(*Src1).setMIFlags(Def->getFlags());
      if (AMDGPU::hasNamedOperand(NewOp, AMDGPU::OpName::clamp))
        Add.addImm(0);

      Def->eraseFromParent();
      MI.eraseFromParent();
      return true;
    }

    assert(NewOp == AMDGPU::V_ADD_CO_U32_e32);

    MachineBasicBlock::LivenessQueryResult Liveness =
        MBB->computeRegisterLiveness(TRI, AMDGPU::VCC, *Def, 16);
    if (Liveness == MachineBasicBlock::LQR_Dead) {
      // TODO: If src1 satisfies operand constraints, use vop3 version.
      BuildMI(*MBB, *Def, DL, TII->get(NewOp), DstReg)
          .add(*Src0)
          .add(*Src1)
          .setOperandDead(3) // implicit-def $vcc
          .setMIFlags(Def->getFlags());
      Def->eraseFromParent();
      MI.eraseFromParent();
      return true;
    }
  }

  return false;
}
```
<a name="block_33"></a>**File:** llvm/lib/Target/AMDGPU/SIFoldOperands.cpp (L740-895)
```cpp
bool SIFoldOperandsImpl::tryAddToFoldList(
    SmallVectorImpl<FoldCandidate> &FoldList, MachineInstr *MI, unsigned OpNo,
    const FoldableDef &OpToFold) const {
  const unsigned Opc = MI->getOpcode();

  auto tryToFoldAsFMAAKorMK = [&]() {
    if (!OpToFold.isImm())
      return false;

    const bool TryAK = OpNo == 3;
    const unsigned NewOpc = TryAK ? AMDGPU::S_FMAAK_F32 : AMDGPU::S_FMAMK_F32;
    MI->setDesc(TII->get(NewOpc));

    // We have to fold into operand which would be Imm not into OpNo.
    bool FoldAsFMAAKorMK =
        tryAddToFoldList(FoldList, MI, TryAK ? 3 : 2, OpToFold);
    if (FoldAsFMAAKorMK) {
      // Untie Src2 of fmac.
      MI->untieRegOperand(3);
      // For fmamk swap operands 1 and 2 if OpToFold was meant for operand 1.
      if (OpNo == 1) {
        MachineOperand &Op1 = MI->getOperand(1);
        MachineOperand &Op2 = MI->getOperand(2);
        Register OldReg = Op1.getReg();
        // Operand 2 might be an inlinable constant
        if (Op2.isImm()) {
          Op1.ChangeToImmediate(Op2.getImm());
          Op2.ChangeToRegister(OldReg, false);
        } else {
          Op1.setReg(Op2.getReg());
          Op2.setReg(OldReg);
        }
      }
      return true;
    }
    MI->setDesc(TII->get(Opc));
    return false;
  };

  bool IsLegal = OpToFold.isOperandLegal(*TII, *MI, OpNo);
  if (!IsLegal && OpToFold.isImm()) {
    if (std::optional<int64_t> ImmVal = OpToFold.getEffectiveImmVal())
      IsLegal = canUseImmWithOpSel(MI, OpNo, *ImmVal);
  }

  if (!IsLegal) {
    // Special case for v_mac_{f16, f32}_e64 if we are trying to fold into src2
    unsigned NewOpc = macToMad(Opc);
    if (NewOpc != AMDGPU::INSTRUCTION_LIST_END) {
      // Check if changing this to a v_mad_{f16, f32} instruction will allow us
      // to fold the operand.
      MI->setDesc(TII->get(NewOpc));
      bool AddOpSel = !AMDGPU::hasNamedOperand(Opc, AMDGPU::OpName::op_sel) &&
                      AMDGPU::hasNamedOperand(NewOpc, AMDGPU::OpName::op_sel);
      if (AddOpSel)
        MI->addOperand(MachineOperand::CreateImm(0));
      bool FoldAsMAD = tryAddToFoldList(FoldList, MI, OpNo, OpToFold);
      if (FoldAsMAD) {
        MI->untieRegOperand(OpNo);
        return true;
      }
      if (AddOpSel)
        MI->removeOperand(MI->getNumExplicitOperands() - 1);
      MI->setDesc(TII->get(Opc));
    }

    // Special case for s_fmac_f32 if we are trying to fold into Src2.
    // By transforming into fmaak we can untie Src2 and make folding legal.
    if (Opc == AMDGPU::S_FMAC_F32 && OpNo == 3) {
      if (tryToFoldAsFMAAKorMK())
        return true;
    }

    // Special case for s_setreg_b32
    if (OpToFold.isImm()) {
      unsigned ImmOpc = 0;
      if (Opc == AMDGPU::S_SETREG_B32)
        ImmOpc = AMDGPU::S_SETREG_IMM32_B32;
      else if (Opc == AMDGPU::S_SETREG_B32_mode)
        ImmOpc = AMDGPU::S_SETREG_IMM32_B32_mode;
      if (ImmOpc) {
        MI->setDesc(TII->get(ImmOpc));
        appendFoldCandidate(FoldList, MI, OpNo, OpToFold);
        return true;
      }
    }

    // Operand is not legal, so try to commute the instruction to
    // see if this makes it possible to fold.
    unsigned CommuteOpNo = TargetInstrInfo::CommuteAnyOperandIndex;
    bool CanCommute = TII->findCommutedOpIndices(*MI, OpNo, CommuteOpNo);
    if (!CanCommute)
      return false;

    MachineOperand &Op = MI->getOperand(OpNo);
    MachineOperand &CommutedOp = MI->getOperand(CommuteOpNo);

    // One of operands might be an Imm operand, and OpNo may refer to it after
    // the call of commuteInstruction() below. Such situations are avoided
    // here explicitly as OpNo must be a register operand to be a candidate
    // for memory folding.
    if (!Op.isReg() || !CommutedOp.isReg())
      return false;

    // The same situation with an immediate could reproduce if both inputs are
    // the same register.
    if (Op.isReg() && CommutedOp.isReg() &&
        (Op.getReg() == CommutedOp.getReg() &&
         Op.getSubReg() == CommutedOp.getSubReg()))
      return false;

    if (!TII->commuteInstruction(*MI, false, OpNo, CommuteOpNo))
      return false;

    int Op32 = -1;
    if (!OpToFold.isOperandLegal(*TII, *MI, CommuteOpNo)) {
      if ((Opc != AMDGPU::V_ADD_CO_U32_e64 && Opc != AMDGPU::V_SUB_CO_U32_e64 &&
           Opc != AMDGPU::V_SUBREV_CO_U32_e64) || // FIXME
          (!OpToFold.isImm() && !OpToFold.isFI() && !OpToFold.isGlobal())) {
        TII->commuteInstruction(*MI, false, OpNo, CommuteOpNo);
        return false;
      }

      // Verify the other operand is a VGPR, otherwise we would violate the
      // constant bus restriction.
      MachineOperand &OtherOp = MI->getOperand(OpNo);
      if (!OtherOp.isReg() ||
          !TII->getRegisterInfo().isVGPR(*MRI, OtherOp.getReg()))
        return false;

      assert(MI->getOperand(1).isDef());

      // Make sure to get the 32-bit version of the commuted opcode.
      unsigned MaybeCommutedOpc = MI->getOpcode();
      Op32 = AMDGPU::getVOPe32(MaybeCommutedOpc);
    }

    appendFoldCandidate(FoldList, MI, CommuteOpNo, OpToFold, /*Commuted=*/true,
                        Op32);
    return true;
  }

  // Special case for s_fmac_f32 if we are trying to fold into Src0 or Src1.
  // By changing into fmamk we can untie Src2.
  // If folding for Src0 happens first and it is identical operand to Src1 we
  // should avoid transforming into fmamk which requires commuting as it would
  // cause folding into Src1 to fail later on due to wrong OpNo used.
  if (Opc == AMDGPU::S_FMAC_F32 &&
      (OpNo != 1 || !MI->getOperand(1).isIdenticalTo(MI->getOperand(2)))) {
    if (tryToFoldAsFMAAKorMK())
      return true;
  }

  appendFoldCandidate(FoldList, MI, OpNo, OpToFold);
  return true;
}
```
<a name="block_34"></a>**File:** llvm/lib/Target/AMDGPU/SIFoldOperands.cpp (L1137-1445)
```cpp
void SIFoldOperandsImpl::foldOperand(
    FoldableDef OpToFold, MachineInstr *UseMI, int UseOpIdx,
    SmallVectorImpl<FoldCandidate> &FoldList,
    SmallVectorImpl<MachineInstr *> &CopiesToReplace) const {
  const MachineOperand *UseOp = &UseMI->getOperand(UseOpIdx);

  if (!isUseSafeToFold(*UseMI, *UseOp))
    return;

  // FIXME: Fold operands with subregs.
  if (UseOp->isReg() && OpToFold.isReg()) {
    if (UseOp->isImplicit())
      return;
    // Allow folding from SGPRs to 16-bit VGPRs.
    if (UseOp->getSubReg() != AMDGPU::NoSubRegister &&
        (UseOp->getSubReg() != AMDGPU::lo16 ||
         !TRI->isSGPRReg(*MRI, OpToFold.getReg())))
      return;
  }

  // Special case for REG_SEQUENCE: We can't fold literals into
  // REG_SEQUENCE instructions, so we have to fold them into the
  // uses of REG_SEQUENCE.
  if (UseMI->isRegSequence()) {
    Register RegSeqDstReg = UseMI->getOperand(0).getReg();
    unsigned RegSeqDstSubReg = UseMI->getOperand(UseOpIdx + 1).getImm();

    int64_t SplatVal;
    const TargetRegisterClass *SplatRC;
    std::tie(SplatVal, SplatRC) = isRegSeqSplat(*UseMI);

    // Grab the use operands first
    SmallVector<MachineOperand *, 4> UsesToProcess(
        llvm::make_pointer_range(MRI->use_nodbg_operands(RegSeqDstReg)));
    for (auto *RSUse : UsesToProcess) {
      MachineInstr *RSUseMI = RSUse->getParent();
      unsigned OpNo = RSUseMI->getOperandNo(RSUse);

      if (SplatRC) {
        if (tryFoldRegSeqSplat(RSUseMI, OpNo, SplatVal, SplatRC)) {
          FoldableDef SplatDef(SplatVal, SplatRC);
          appendFoldCandidate(FoldList, RSUseMI, OpNo, SplatDef);
          continue;
        }
      }

      // TODO: Handle general compose
      if (RSUse->getSubReg() != RegSeqDstSubReg)
        continue;

      // FIXME: We should avoid recursing here. There should be a cleaner split
      // between the in-place mutations and adding to the fold list.
      foldOperand(OpToFold, RSUseMI, RSUseMI->getOperandNo(RSUse), FoldList,
                  CopiesToReplace);
    }

    return;
  }

  if (tryToFoldACImm(OpToFold, UseMI, UseOpIdx, FoldList))
    return;

  if (frameIndexMayFold(*UseMI, UseOpIdx, OpToFold)) {
    // Verify that this is a stack access.
    // FIXME: Should probably use stack pseudos before frame lowering.

    if (TII->isMUBUF(*UseMI)) {
      if (TII->getNamedOperand(*UseMI, AMDGPU::OpName::srsrc)->getReg() !=
          MFI->getScratchRSrcReg())
        return;

      // Ensure this is either relative to the current frame or the current
      // wave.
      MachineOperand &SOff =
          *TII->getNamedOperand(*UseMI, AMDGPU::OpName::soffset);
      if (!SOff.isImm() || SOff.getImm() != 0)
        return;
    }

    // A frame index will resolve to a positive constant, so it should always be
    // safe to fold the addressing mode, even pre-GFX9.
    UseMI->getOperand(UseOpIdx).ChangeToFrameIndex(OpToFold.getFI());

    const unsigned Opc = UseMI->getOpcode();
    if (TII->isFLATScratch(*UseMI) &&
        AMDGPU::hasNamedOperand(Opc, AMDGPU::OpName::vaddr) &&
        !AMDGPU::hasNamedOperand(Opc, AMDGPU::OpName::saddr)) {
      unsigned NewOpc = AMDGPU::getFlatScratchInstSSfromSV(Opc);
      UseMI->setDesc(TII->get(NewOpc));
    }

    return;
  }

  bool FoldingImmLike =
      OpToFold.isImm() || OpToFold.isFI() || OpToFold.isGlobal();

  if (FoldingImmLike && UseMI->isCopy()) {
    Register DestReg = UseMI->getOperand(0).getReg();
    Register SrcReg = UseMI->getOperand(1).getReg();
    assert(SrcReg.isVirtual());

    const TargetRegisterClass *SrcRC = MRI->getRegClass(SrcReg);

    // Don't fold into a copy to a physical register with the same class. Doing
    // so would interfere with the register coalescer's logic which would avoid
    // redundant initializations.
    if (DestReg.isPhysical() && SrcRC->contains(DestReg))
      return;

    const TargetRegisterClass *DestRC = TRI->getRegClassForReg(*MRI, DestReg);
    if (!DestReg.isPhysical() && DestRC == &AMDGPU::AGPR_32RegClass) {
      std::optional<int64_t> UseImmVal = OpToFold.getEffectiveImmVal();
      if (UseImmVal && TII->isInlineConstant(
                           *UseImmVal, AMDGPU::OPERAND_REG_INLINE_C_INT32)) {
        UseMI->setDesc(TII->get(AMDGPU::V_ACCVGPR_WRITE_B32_e64));
        UseMI->getOperand(1).ChangeToImmediate(*UseImmVal);
        CopiesToReplace.push_back(UseMI);
        return;
      }
    }

    // Allow immediates COPYd into sgpr_lo16 to be further folded while
    // still being legal if not further folded
    if (DestRC == &AMDGPU::SGPR_LO16RegClass) {
      assert(ST->useRealTrue16Insts());
      MRI->setRegClass(DestReg, &AMDGPU::SGPR_32RegClass);
      DestRC = &AMDGPU::SGPR_32RegClass;
    }

    // In order to fold immediates into copies, we need to change the
    // copy to a MOV.

    unsigned MovOp = TII->getMovOpcode(DestRC);
    if (MovOp == AMDGPU::COPY)
      return;

    // Fold if the destination register class of the MOV instruction (ResRC)
    // is a superclass of (or equal to) the destination register class of the
    // COPY (DestRC). If this condition fails, folding would be illegal.
    const MCInstrDesc &MovDesc = TII->get(MovOp);
    assert(MovDesc.getNumDefs() > 0 && MovDesc.operands()[0].RegClass != -1);
    const TargetRegisterClass *ResRC =
        TRI->getRegClass(MovDesc.operands()[0].RegClass);
    if (!DestRC->hasSuperClassEq(ResRC))
      return;

    MachineInstr::mop_iterator ImpOpI = UseMI->implicit_operands().begin();
    MachineInstr::mop_iterator ImpOpE = UseMI->implicit_operands().end();
    while (ImpOpI != ImpOpE) {
      MachineInstr::mop_iterator Tmp = ImpOpI;
      ImpOpI++;
      UseMI->removeOperand(UseMI->getOperandNo(Tmp));
    }
    UseMI->setDesc(TII->get(MovOp));

    if (MovOp == AMDGPU::V_MOV_B16_t16_e64) {
      const auto &SrcOp = UseMI->getOperand(UseOpIdx);
      MachineOperand NewSrcOp(SrcOp);
      MachineFunction *MF = UseMI->getParent()->getParent();
      UseMI->removeOperand(1);
      UseMI->addOperand(*MF, MachineOperand::CreateImm(0)); // src0_modifiers
      UseMI->addOperand(NewSrcOp);                          // src0
      UseMI->addOperand(*MF, MachineOperand::CreateImm(0)); // op_sel
      UseOpIdx = 2;
      UseOp = &UseMI->getOperand(UseOpIdx);
    }
    CopiesToReplace.push_back(UseMI);
  } else {
    if (UseMI->isCopy() && OpToFold.isReg() &&
        UseMI->getOperand(0).getReg().isVirtual() &&
        !UseMI->getOperand(1).getSubReg() &&
        OpToFold.DefMI->implicit_operands().empty()) {
      LLVM_DEBUG(dbgs() << "Folding " << OpToFold.OpToFold << "\n into "
                        << *UseMI);
      unsigned Size = TII->getOpSize(*UseMI, 1);
      Register UseReg = OpToFold.getReg();
      UseMI->getOperand(1).setReg(UseReg);
      unsigned SubRegIdx = OpToFold.getSubReg();
      // Hack to allow 32-bit SGPRs to be folded into True16 instructions
      // Remove this if 16-bit SGPRs (i.e. SGPR_LO16) are added to the
      // VS_16RegClass
      //
      // Excerpt from AMDGPUGenRegisterInfo.inc
      // NoSubRegister, //0
      // hi16, // 1
      // lo16, // 2
      // sub0, // 3
      // ...
      // sub1, // 11
      // sub1_hi16, // 12
      // sub1_lo16, // 13
      static_assert(AMDGPU::sub1_hi16 == 12, "Subregister layout has changed");
      if (Size == 2 && TRI->isVGPR(*MRI, UseMI->getOperand(0).getReg()) &&
          TRI->isSGPRReg(*MRI, UseReg)) {
        // Produce the 32 bit subregister index to which the 16-bit subregister
        // is aligned.
        if (SubRegIdx > AMDGPU::sub1) {
          LaneBitmask M = TRI->getSubRegIndexLaneMask(SubRegIdx);
          M |= M.getLane(M.getHighestLane() - 1);
          SmallVector<unsigned, 4> Indexes;
          TRI->getCoveringSubRegIndexes(TRI->getRegClassForReg(*MRI, UseReg), M,
                                        Indexes);
          assert(Indexes.size() == 1 && "Expected one 32-bit subreg to cover");
          SubRegIdx = Indexes[0];
          // 32-bit registers do not have a sub0 index
        } else if (TII->getOpSize(*UseMI, 1) == 4)
          SubRegIdx = 0;
        else
          SubRegIdx = AMDGPU::sub0;
      }
      UseMI->getOperand(1).setSubReg(SubRegIdx);
      UseMI->getOperand(1).setIsKill(false);
      CopiesToReplace.push_back(UseMI);
      OpToFold.OpToFold->setIsKill(false);

      // Remove kill flags as kills may now be out of order with uses.
      MRI->clearKillFlags(UseReg);
      if (foldCopyToAGPRRegSequence(UseMI))
        return;
    }

    unsigned UseOpc = UseMI->getOpcode();
    if (UseOpc == AMDGPU::V_READFIRSTLANE_B32 ||
        (UseOpc == AMDGPU::V_READLANE_B32 &&
         (int)UseOpIdx ==
         AMDGPU::getNamedOperandIdx(UseOpc, AMDGPU::OpName::src0))) {
      // %vgpr = V_MOV_B32 imm
      // %sgpr = V_READFIRSTLANE_B32 %vgpr
      // =>
      // %sgpr = S_MOV_B32 imm
      if (FoldingImmLike) {
        if (execMayBeModifiedBeforeUse(*MRI,
                                       UseMI->getOperand(UseOpIdx).getReg(),
                                       *OpToFold.DefMI, *UseMI))
          return;

        UseMI->setDesc(TII->get(AMDGPU::S_MOV_B32));

        if (OpToFold.isImm()) {
          UseMI->getOperand(1).ChangeToImmediate(
              *OpToFold.getEffectiveImmVal());
        } else if (OpToFold.isFI())
          UseMI->getOperand(1).ChangeToFrameIndex(OpToFold.getFI());
        else {
          assert(OpToFold.isGlobal());
          UseMI->getOperand(1).ChangeToGA(OpToFold.OpToFold->getGlobal(),
                                          OpToFold.OpToFold->getOffset(),
                                          OpToFold.OpToFold->getTargetFlags());
        }
        UseMI->removeOperand(2); // Remove exec read (or src1 for readlane)
        return;
      }

      if (OpToFold.isReg() && TRI->isSGPRReg(*MRI, OpToFold.getReg())) {
        if (checkIfExecMayBeModifiedBeforeUseAcrossBB(
                *MRI, UseMI->getOperand(UseOpIdx).getReg(),
                *OpToFold.DefMI, *UseMI, SIFoldOperandsPreheaderThreshold))
          return;

        // %vgpr = COPY %sgpr0
        // %sgpr1 = V_READFIRSTLANE_B32 %vgpr
        // =>
        // %sgpr1 = COPY %sgpr0
        UseMI->setDesc(TII->get(AMDGPU::COPY));
        UseMI->getOperand(1).setReg(OpToFold.getReg());
        UseMI->getOperand(1).setSubReg(OpToFold.getSubReg());
        UseMI->getOperand(1).setIsKill(false);
        UseMI->removeOperand(2); // Remove exec read (or src1 for readlane)
        return;
      }
    }

    const MCInstrDesc &UseDesc = UseMI->getDesc();

    // Don't fold into target independent nodes.  Target independent opcodes
    // don't have defined register classes.
    if (UseDesc.isVariadic() || UseOp->isImplicit() ||
        UseDesc.operands()[UseOpIdx].RegClass == -1)
      return;
  }

  if (!FoldingImmLike) {
    if (OpToFold.isReg() && ST->needsAlignedVGPRs()) {
      // Don't fold if OpToFold doesn't hold an aligned register.
      const TargetRegisterClass *RC =
          TRI->getRegClassForReg(*MRI, OpToFold.getReg());
      assert(RC);
      if (TRI->hasVectorRegisters(RC) && OpToFold.getSubReg()) {
        unsigned SubReg = OpToFold.getSubReg();
        if (const TargetRegisterClass *SubRC =
                TRI->getSubRegisterClass(RC, SubReg))
          RC = SubRC;
      }

      if (!RC || !TRI->isProperlyAlignedRC(*RC))
        return;
    }

    tryAddToFoldList(FoldList, UseMI, UseOpIdx, OpToFold);

    // FIXME: We could try to change the instruction from 64-bit to 32-bit
    // to enable more folding opportunities.  The shrink operands pass
    // already does this.
    return;
  }

  tryAddToFoldList(FoldList, UseMI, UseOpIdx, OpToFold);
}
```
<a name="block_35"></a>**File:** llvm/lib/Target/AMDGPU/SIFoldOperands.cpp (L1447-1511) [<sup>↩</sup>](#ref-block_35)
```cpp
static bool evalBinaryInstruction(unsigned Opcode, int32_t &Result,
                                  uint32_t LHS, uint32_t RHS) {
  switch (Opcode) {
  case AMDGPU::V_AND_B32_e64:
  case AMDGPU::V_AND_B32_e32:
  case AMDGPU::S_AND_B32:
    Result = LHS & RHS;
    return true;
  case AMDGPU::V_OR_B32_e64:
  case AMDGPU::V_OR_B32_e32:
  case AMDGPU::S_OR_B32:
    Result = LHS | RHS;
    return true;
  case AMDGPU::V_XOR_B32_e64:
  case AMDGPU::V_XOR_B32_e32:
  case AMDGPU::S_XOR_B32:
    Result = LHS ^ RHS;
    return true;
  case AMDGPU::S_XNOR_B32:
    Result = ~(LHS ^ RHS);
    return true;
  case AMDGPU::S_NAND_B32:
    Result = ~(LHS & RHS);
    return true;
  case AMDGPU::S_NOR_B32:
    Result = ~(LHS | RHS);
    return true;
  case AMDGPU::S_ANDN2_B32:
    Result = LHS & ~RHS;
    return true;
  case AMDGPU::S_ORN2_B32:
    Result = LHS | ~RHS;
    return true;
  case AMDGPU::V_LSHL_B32_e64:
  case AMDGPU::V_LSHL_B32_e32:
  case AMDGPU::S_LSHL_B32:
    // The instruction ignores the high bits for out of bounds shifts.
    Result = LHS << (RHS & 31);
    return true;
  case AMDGPU::V_LSHLREV_B32_e64:
  case AMDGPU::V_LSHLREV_B32_e32:
    Result = RHS << (LHS & 31);
    return true;
  case AMDGPU::V_LSHR_B32_e64:
  case AMDGPU::V_LSHR_B32_e32:
  case AMDGPU::S_LSHR_B32:
    Result = LHS >> (RHS & 31);
    return true;
  case AMDGPU::V_LSHRREV_B32_e64:
  case AMDGPU::V_LSHRREV_B32_e32:
    Result = RHS >> (LHS & 31);
    return true;
  case AMDGPU::V_ASHR_I32_e64:
  case AMDGPU::V_ASHR_I32_e32:
  case AMDGPU::S_ASHR_I32:
    Result = static_cast<int32_t>(LHS) >> (RHS & 31);
    return true;
  case AMDGPU::V_ASHRREV_I32_e64:
  case AMDGPU::V_ASHRREV_I32_e32:
    Result = static_cast<int32_t>(RHS) >> (LHS & 31);
    return true;
  default:
    return false;
  }
}
```
<a name="block_36"></a>**File:** llvm/lib/Target/AMDGPU/SIFoldOperands.cpp (L1552-1655)
```cpp
bool SIFoldOperandsImpl::tryConstantFoldOp(MachineInstr *MI) const {
  if (!MI->allImplicitDefsAreDead())
    return false;

  unsigned Opc = MI->getOpcode();

  int Src0Idx = AMDGPU::getNamedOperandIdx(Opc, AMDGPU::OpName::src0);
  if (Src0Idx == -1)
    return false;

  MachineOperand *Src0 = &MI->getOperand(Src0Idx);
  std::optional<int64_t> Src0Imm = getImmOrMaterializedImm(*Src0);

  if ((Opc == AMDGPU::V_NOT_B32_e64 || Opc == AMDGPU::V_NOT_B32_e32 ||
       Opc == AMDGPU::S_NOT_B32) &&
      Src0Imm) {
    MI->getOperand(1).ChangeToImmediate(~*Src0Imm);
    mutateCopyOp(*MI, TII->get(getMovOpc(Opc == AMDGPU::S_NOT_B32)));
    return true;
  }

  int Src1Idx = AMDGPU::getNamedOperandIdx(Opc, AMDGPU::OpName::src1);
  if (Src1Idx == -1)
    return false;

  MachineOperand *Src1 = &MI->getOperand(Src1Idx);
  std::optional<int64_t> Src1Imm = getImmOrMaterializedImm(*Src1);

  if (!Src0Imm && !Src1Imm)
    return false;

  // and k0, k1 -> v_mov_b32 (k0 & k1)
  // or k0, k1 -> v_mov_b32 (k0 | k1)
  // xor k0, k1 -> v_mov_b32 (k0 ^ k1)
  if (Src0Imm && Src1Imm) {
    int32_t NewImm;
    if (!evalBinaryInstruction(Opc, NewImm, *Src0Imm, *Src1Imm))
      return false;

    bool IsSGPR = TRI->isSGPRReg(*MRI, MI->getOperand(0).getReg());

    // Be careful to change the right operand, src0 may belong to a different
    // instruction.
    MI->getOperand(Src0Idx).ChangeToImmediate(NewImm);
    MI->removeOperand(Src1Idx);
    mutateCopyOp(*MI, TII->get(getMovOpc(IsSGPR)));
    return true;
  }

  if (!MI->isCommutable())
    return false;

  if (Src0Imm && !Src1Imm) {
    std::swap(Src0, Src1);
    std::swap(Src0Idx, Src1Idx);
    std::swap(Src0Imm, Src1Imm);
  }

  int32_t Src1Val = static_cast<int32_t>(*Src1Imm);
  if (Opc == AMDGPU::V_OR_B32_e64 ||
      Opc == AMDGPU::V_OR_B32_e32 ||
      Opc == AMDGPU::S_OR_B32) {
    if (Src1Val == 0) {
      // y = or x, 0 => y = copy x
      MI->removeOperand(Src1Idx);
      mutateCopyOp(*MI, TII->get(AMDGPU::COPY));
    } else if (Src1Val == -1) {
      // y = or x, -1 => y = v_mov_b32 -1
      MI->removeOperand(Src1Idx);
      mutateCopyOp(*MI, TII->get(getMovOpc(Opc == AMDGPU::S_OR_B32)));
    } else
      return false;

    return true;
  }

  if (Opc == AMDGPU::V_AND_B32_e64 || Opc == AMDGPU::V_AND_B32_e32 ||
      Opc == AMDGPU::S_AND_B32) {
    if (Src1Val == 0) {
      // y = and x, 0 => y = v_mov_b32 0
      MI->removeOperand(Src0Idx);
      mutateCopyOp(*MI, TII->get(getMovOpc(Opc == AMDGPU::S_AND_B32)));
    } else if (Src1Val == -1) {
      // y = and x, -1 => y = copy x
      MI->removeOperand(Src1Idx);
      mutateCopyOp(*MI, TII->get(AMDGPU::COPY));
    } else
      return false;

    return true;
  }

  if (Opc == AMDGPU::V_XOR_B32_e64 || Opc == AMDGPU::V_XOR_B32_e32 ||
      Opc == AMDGPU::S_XOR_B32) {
    if (Src1Val == 0) {
      // y = xor x, 0 => y = copy x
      MI->removeOperand(Src1Idx);
      mutateCopyOp(*MI, TII->get(AMDGPU::COPY));
      return true;
    }
  }

  return false;
}
```
<a name="block_37"></a>**File:** llvm/lib/Target/AMDGPU/SIFoldOperands.cpp (L1658-1698)
```cpp
bool SIFoldOperandsImpl::tryFoldCndMask(MachineInstr &MI) const {
  unsigned Opc = MI.getOpcode();
  if (Opc != AMDGPU::V_CNDMASK_B32_e32 && Opc != AMDGPU::V_CNDMASK_B32_e64 &&
      Opc != AMDGPU::V_CNDMASK_B64_PSEUDO)
    return false;

  MachineOperand *Src0 = TII->getNamedOperand(MI, AMDGPU::OpName::src0);
  MachineOperand *Src1 = TII->getNamedOperand(MI, AMDGPU::OpName::src1);
  if (!Src1->isIdenticalTo(*Src0)) {
    std::optional<int64_t> Src1Imm = getImmOrMaterializedImm(*Src1);
    if (!Src1Imm)
      return false;

    std::optional<int64_t> Src0Imm = getImmOrMaterializedImm(*Src0);
    if (!Src0Imm || *Src0Imm != *Src1Imm)
      return false;
  }

  int Src1ModIdx =
      AMDGPU::getNamedOperandIdx(Opc, AMDGPU::OpName::src1_modifiers);
  int Src0ModIdx =
      AMDGPU::getNamedOperandIdx(Opc, AMDGPU::OpName::src0_modifiers);
  if ((Src1ModIdx != -1 && MI.getOperand(Src1ModIdx).getImm() != 0) ||
      (Src0ModIdx != -1 && MI.getOperand(Src0ModIdx).getImm() != 0))
    return false;

  LLVM_DEBUG(dbgs() << "Folded " << MI << " into ");
  auto &NewDesc =
      TII->get(Src0->isReg() ? (unsigned)AMDGPU::COPY : getMovOpc(false));
  int Src2Idx = AMDGPU::getNamedOperandIdx(Opc, AMDGPU::OpName::src2);
  if (Src2Idx != -1)
    MI.removeOperand(Src2Idx);
  MI.removeOperand(AMDGPU::getNamedOperandIdx(Opc, AMDGPU::OpName::src1));
  if (Src1ModIdx != -1)
    MI.removeOperand(Src1ModIdx);
  if (Src0ModIdx != -1)
    MI.removeOperand(Src0ModIdx);
  mutateCopyOp(MI, NewDesc);
  LLVM_DEBUG(dbgs() << MI);
  return true;
}
```
<a name="block_38"></a>**File:** llvm/lib/Target/AMDGPU/SIFoldOperands.cpp (L1700-1720)
```cpp
bool SIFoldOperandsImpl::tryFoldZeroHighBits(MachineInstr &MI) const {
  if (MI.getOpcode() != AMDGPU::V_AND_B32_e64 &&
      MI.getOpcode() != AMDGPU::V_AND_B32_e32)
    return false;

  std::optional<int64_t> Src0Imm = getImmOrMaterializedImm(MI.getOperand(1));
  if (!Src0Imm || *Src0Imm != 0xffff || !MI.getOperand(2).isReg())
    return false;

  Register Src1 = MI.getOperand(2).getReg();
  MachineInstr *SrcDef = MRI->getVRegDef(Src1);
  if (!ST->zeroesHigh16BitsOfDest(SrcDef->getOpcode()))
    return false;

  Register Dst = MI.getOperand(0).getReg();
  MRI->replaceRegWith(Dst, Src1);
  if (!MI.getOperand(2).isKill())
    MRI->clearKillFlags(Src1);
  MI.eraseFromParent();
  return true;
}
```
<a name="block_39"></a>**File:** llvm/lib/Target/AMDGPU/SIFoldOperands.cpp (L1722-1801)
```cpp
bool SIFoldOperandsImpl::foldInstOperand(MachineInstr &MI,
                                         const FoldableDef &OpToFold) const {
  // We need mutate the operands of new mov instructions to add implicit
  // uses of EXEC, but adding them invalidates the use_iterator, so defer
  // this.
  SmallVector<MachineInstr *, 4> CopiesToReplace;
  SmallVector<FoldCandidate, 4> FoldList;
  MachineOperand &Dst = MI.getOperand(0);
  bool Changed = false;

  if (OpToFold.isImm()) {
    for (auto &UseMI :
         make_early_inc_range(MRI->use_nodbg_instructions(Dst.getReg()))) {
      // Folding the immediate may reveal operations that can be constant
      // folded or replaced with a copy. This can happen for example after
      // frame indices are lowered to constants or from splitting 64-bit
      // constants.
      //
      // We may also encounter cases where one or both operands are
      // immediates materialized into a register, which would ordinarily not
      // be folded due to multiple uses or operand constraints.
      if (tryConstantFoldOp(&UseMI)) {
        LLVM_DEBUG(dbgs() << "Constant folded " << UseMI);
        Changed = true;
      }
    }
  }

  SmallVector<MachineOperand *, 4> UsesToProcess(
      llvm::make_pointer_range(MRI->use_nodbg_operands(Dst.getReg())));
  for (auto *U : UsesToProcess) {
    MachineInstr *UseMI = U->getParent();

    FoldableDef SubOpToFold = OpToFold.getWithSubReg(*TRI, U->getSubReg());
    foldOperand(SubOpToFold, UseMI, UseMI->getOperandNo(U), FoldList,
                CopiesToReplace);
  }

  if (CopiesToReplace.empty() && FoldList.empty())
    return Changed;

  MachineFunction *MF = MI.getParent()->getParent();
  // Make sure we add EXEC uses to any new v_mov instructions created.
  for (MachineInstr *Copy : CopiesToReplace)
    Copy->addImplicitDefUseOperands(*MF);

  for (FoldCandidate &Fold : FoldList) {
    assert(!Fold.isReg() || Fold.Def.OpToFold);
    if (Fold.isReg() && Fold.getReg().isVirtual()) {
      Register Reg = Fold.getReg();
      const MachineInstr *DefMI = Fold.Def.DefMI;
      if (DefMI->readsRegister(AMDGPU::EXEC, TRI) &&
          execMayBeModifiedBeforeUse(*MRI, Reg, *DefMI, *Fold.UseMI))
        continue;
    }
    if (updateOperand(Fold)) {
      // Clear kill flags.
      if (Fold.isReg()) {
        assert(Fold.Def.OpToFold && Fold.isReg());
        // FIXME: Probably shouldn't bother trying to fold if not an
        // SGPR. PeepholeOptimizer can eliminate redundant VGPR->VGPR
        // copies.
        MRI->clearKillFlags(Fold.getReg());
      }
      LLVM_DEBUG(dbgs() << "Folded source from " << MI << " into OpNo "
                        << static_cast<int>(Fold.UseOpNo) << " of "
                        << *Fold.UseMI);

      if (Fold.isImm() && tryConstantFoldOp(Fold.UseMI)) {
        LLVM_DEBUG(dbgs() << "Constant folded " << *Fold.UseMI);
        Changed = true;
      }

    } else if (Fold.Commuted) {
      // Restoring instruction's original operand order if fold has failed.
      TII->commuteInstruction(*Fold.UseMI, false);
    }
  }
  return true;
}
```
<a name="block_40"></a>**File:** llvm/lib/Target/AMDGPU/SIFoldOperands.cpp (L1935-2050)
```cpp
bool SIFoldOperandsImpl::tryFoldFoldableCopy(
    MachineInstr &MI, MachineOperand *&CurrentKnownM0Val) const {
  Register DstReg = MI.getOperand(0).getReg();
  // Specially track simple redefs of m0 to the same value in a block, so we
  // can erase the later ones.
  if (DstReg == AMDGPU::M0) {
    MachineOperand &NewM0Val = MI.getOperand(1);
    if (CurrentKnownM0Val && CurrentKnownM0Val->isIdenticalTo(NewM0Val)) {
      MI.eraseFromParent();
      return true;
    }

    // We aren't tracking other physical registers
    CurrentKnownM0Val = (NewM0Val.isReg() && NewM0Val.getReg().isPhysical())
                            ? nullptr
                            : &NewM0Val;
    return false;
  }

  MachineOperand *OpToFoldPtr;
  if (MI.getOpcode() == AMDGPU::V_MOV_B16_t16_e64) {
    // Folding when any src_modifiers are non-zero is unsupported
    if (TII->hasAnyModifiersSet(MI))
      return false;
    OpToFoldPtr = &MI.getOperand(2);
  } else
    OpToFoldPtr = &MI.getOperand(1);
  MachineOperand &OpToFold = *OpToFoldPtr;
  bool FoldingImm = OpToFold.isImm() || OpToFold.isFI() || OpToFold.isGlobal();

  // FIXME: We could also be folding things like TargetIndexes.
  if (!FoldingImm && !OpToFold.isReg())
    return false;

  if (OpToFold.isReg() && !OpToFold.getReg().isVirtual())
    return false;

  // Prevent folding operands backwards in the function. For example,
  // the COPY opcode must not be replaced by 1 in this example:
  //
  //    %3 = COPY %vgpr0; VGPR_32:%3
  //    ...
  //    %vgpr0 = V_MOV_B32_e32 1, implicit %exec
  if (!DstReg.isVirtual())
    return false;

  const TargetRegisterClass *DstRC =
      MRI->getRegClass(MI.getOperand(0).getReg());

  // True16: Fix malformed 16-bit sgpr COPY produced by peephole-opt
  // Can remove this code if proper 16-bit SGPRs are implemented
  // Example: Pre-peephole-opt
  // %29:sgpr_lo16 = COPY %16.lo16:sreg_32
  // %32:sreg_32 = COPY %29:sgpr_lo16
  // %30:sreg_32 = S_PACK_LL_B32_B16 killed %31:sreg_32, killed %32:sreg_32
  // Post-peephole-opt and DCE
  // %32:sreg_32 = COPY %16.lo16:sreg_32
  // %30:sreg_32 = S_PACK_LL_B32_B16 killed %31:sreg_32, killed %32:sreg_32
  // After this transform
  // %32:sreg_32 = COPY %16:sreg_32
  // %30:sreg_32 = S_PACK_LL_B32_B16 killed %31:sreg_32, killed %32:sreg_32
  // After the fold operands pass
  // %30:sreg_32 = S_PACK_LL_B32_B16 killed %31:sreg_32, killed %16:sreg_32
  if (MI.getOpcode() == AMDGPU::COPY && OpToFold.isReg() &&
      OpToFold.getSubReg()) {
    if (DstRC == &AMDGPU::SReg_32RegClass &&
        DstRC == MRI->getRegClass(OpToFold.getReg())) {
      assert(OpToFold.getSubReg() == AMDGPU::lo16);
      OpToFold.setSubReg(0);
    }
  }

  // Fold copy to AGPR through reg_sequence
  // TODO: Handle with subregister extract
  if (OpToFold.isReg() && MI.isCopy() && !MI.getOperand(1).getSubReg()) {
    if (foldCopyToAGPRRegSequence(&MI))
      return true;
  }

  FoldableDef Def(OpToFold, DstRC);
  bool Changed = foldInstOperand(MI, Def);

  // If we managed to fold all uses of this copy then we might as well
  // delete it now.
  // The only reason we need to follow chains of copies here is that
  // tryFoldRegSequence looks forward through copies before folding a
  // REG_SEQUENCE into its eventual users.
  auto *InstToErase = &MI;
  while (MRI->use_nodbg_empty(InstToErase->getOperand(0).getReg())) {
    auto &SrcOp = InstToErase->getOperand(1);
    auto SrcReg = SrcOp.isReg() ? SrcOp.getReg() : Register();
    InstToErase->eraseFromParent();
    Changed = true;
    InstToErase = nullptr;
    if (!SrcReg || SrcReg.isPhysical())
      break;
    InstToErase = MRI->getVRegDef(SrcReg);
    if (!InstToErase || !TII->isFoldableCopy(*InstToErase))
      break;
  }

  if (InstToErase && InstToErase->isRegSequence() &&
      MRI->use_nodbg_empty(InstToErase->getOperand(0).getReg())) {
    InstToErase->eraseFromParent();
    Changed = true;
  }

  if (Changed)
    return true;

  // Run this after foldInstOperand to avoid turning scalar additions into
  // vector additions when the result scalar result could just be folded into
  // the user(s).
  return OpToFold.isReg() &&
         foldCopyToVGPROfScalarAddOfFrameIndex(DstReg, OpToFold.getReg(), MI);
}
```
<a name="block_41"></a>**File:** llvm/lib/Target/AMDGPU/SIFoldOperands.cpp (L2054-2100)
```cpp
const MachineOperand *
SIFoldOperandsImpl::isClamp(const MachineInstr &MI) const {
  unsigned Op = MI.getOpcode();
  switch (Op) {
  case AMDGPU::V_MAX_F32_e64:
  case AMDGPU::V_MAX_F16_e64:
  case AMDGPU::V_MAX_F16_t16_e64:
  case AMDGPU::V_MAX_F16_fake16_e64:
  case AMDGPU::V_MAX_F64_e64:
  case AMDGPU::V_MAX_NUM_F64_e64:
  case AMDGPU::V_PK_MAX_F16: {
    if (MI.mayRaiseFPException())
      return nullptr;

    if (!TII->getNamedOperand(MI, AMDGPU::OpName::clamp)->getImm())
      return nullptr;

    // Make sure sources are identical.
    const MachineOperand *Src0 = TII->getNamedOperand(MI, AMDGPU::OpName::src0);
    const MachineOperand *Src1 = TII->getNamedOperand(MI, AMDGPU::OpName::src1);
    if (!Src0->isReg() || !Src1->isReg() ||
        Src0->getReg() != Src1->getReg() ||
        Src0->getSubReg() != Src1->getSubReg() ||
        Src0->getSubReg() != AMDGPU::NoSubRegister)
      return nullptr;

    // Can't fold up if we have modifiers.
    if (TII->hasModifiersSet(MI, AMDGPU::OpName::omod))
      return nullptr;

    unsigned Src0Mods
      = TII->getNamedOperand(MI, AMDGPU::OpName::src0_modifiers)->getImm();
    unsigned Src1Mods
      = TII->getNamedOperand(MI, AMDGPU::OpName::src1_modifiers)->getImm();

    // Having a 0 op_sel_hi would require swizzling the output in the source
    // instruction, which we can't do.
    unsigned UnsetMods = (Op == AMDGPU::V_PK_MAX_F16) ? SISrcMods::OP_SEL_1
                                                      : 0u;
    if (Src0Mods != UnsetMods && Src1Mods != UnsetMods)
      return nullptr;
    return Src0;
  }
  default:
    return nullptr;
  }
}
```
<a name="block_42"></a>**File:** llvm/lib/Target/AMDGPU/SIFoldOperands.cpp (L2103-2152)
```cpp
bool SIFoldOperandsImpl::tryFoldClamp(MachineInstr &MI) {
  const MachineOperand *ClampSrc = isClamp(MI);
  if (!ClampSrc || !MRI->hasOneNonDBGUser(ClampSrc->getReg()))
    return false;

  if (!ClampSrc->getReg().isVirtual())
    return false;

  // Look through COPY. COPY only observed with True16.
  Register DefSrcReg = TRI->lookThruCopyLike(ClampSrc->getReg(), MRI);
  MachineInstr *Def =
      MRI->getVRegDef(DefSrcReg.isVirtual() ? DefSrcReg : ClampSrc->getReg());

  // The type of clamp must be compatible.
  if (TII->getClampMask(*Def) != TII->getClampMask(MI))
    return false;

  if (Def->mayRaiseFPException())
    return false;

  MachineOperand *DefClamp = TII->getNamedOperand(*Def, AMDGPU::OpName::clamp);
  if (!DefClamp)
    return false;

  LLVM_DEBUG(dbgs() << "Folding clamp " << *DefClamp << " into " << *Def);

  // Clamp is applied after omod, so it is OK if omod is set.
  DefClamp->setImm(1);

  Register DefReg = Def->getOperand(0).getReg();
  Register MIDstReg = MI.getOperand(0).getReg();
  if (TRI->isSGPRReg(*MRI, DefReg)) {
    // Pseudo scalar instructions have a SGPR for dst and clamp is a v_max*
    // instruction with a VGPR dst.
    BuildMI(*MI.getParent(), MI, MI.getDebugLoc(), TII->get(AMDGPU::COPY),
            MIDstReg)
        .addReg(DefReg);
  } else {
    MRI->replaceRegWith(MIDstReg, DefReg);
  }
  MI.eraseFromParent();

  // Use of output modifiers forces VOP3 encoding for a VOP2 mac/fmac
  // instruction, so we might as well convert it to the more flexible VOP3-only
  // mad/fma form.
  if (TII->convertToThreeAddress(*Def, nullptr, nullptr))
    Def->eraseFromParent();

  return true;
}
```
<a name="block_43"></a>**File:** llvm/lib/Target/AMDGPU/SIFoldOperands.cpp (L2203-2279)
```cpp
std::pair<const MachineOperand *, int>
SIFoldOperandsImpl::isOMod(const MachineInstr &MI) const {
  unsigned Op = MI.getOpcode();
  switch (Op) {
  case AMDGPU::V_MUL_F64_e64:
  case AMDGPU::V_MUL_F64_pseudo_e64:
  case AMDGPU::V_MUL_F32_e64:
  case AMDGPU::V_MUL_F16_t16_e64:
  case AMDGPU::V_MUL_F16_fake16_e64:
  case AMDGPU::V_MUL_F16_e64: {
    // If output denormals are enabled, omod is ignored.
    if ((Op == AMDGPU::V_MUL_F32_e64 &&
         MFI->getMode().FP32Denormals.Output != DenormalMode::PreserveSign) ||
        ((Op == AMDGPU::V_MUL_F64_e64 || Op == AMDGPU::V_MUL_F64_pseudo_e64 ||
          Op == AMDGPU::V_MUL_F16_e64 || Op == AMDGPU::V_MUL_F16_t16_e64 ||
          Op == AMDGPU::V_MUL_F16_fake16_e64) &&
         MFI->getMode().FP64FP16Denormals.Output !=
             DenormalMode::PreserveSign) ||
        MI.mayRaiseFPException())
      return std::pair(nullptr, SIOutMods::NONE);

    const MachineOperand *RegOp = nullptr;
    const MachineOperand *ImmOp = nullptr;
    const MachineOperand *Src0 = TII->getNamedOperand(MI, AMDGPU::OpName::src0);
    const MachineOperand *Src1 = TII->getNamedOperand(MI, AMDGPU::OpName::src1);
    if (Src0->isImm()) {
      ImmOp = Src0;
      RegOp = Src1;
    } else if (Src1->isImm()) {
      ImmOp = Src1;
      RegOp = Src0;
    } else
      return std::pair(nullptr, SIOutMods::NONE);

    int OMod = getOModValue(Op, ImmOp->getImm());
    if (OMod == SIOutMods::NONE ||
        TII->hasModifiersSet(MI, AMDGPU::OpName::src0_modifiers) ||
        TII->hasModifiersSet(MI, AMDGPU::OpName::src1_modifiers) ||
        TII->hasModifiersSet(MI, AMDGPU::OpName::omod) ||
        TII->hasModifiersSet(MI, AMDGPU::OpName::clamp))
      return std::pair(nullptr, SIOutMods::NONE);

    return std::pair(RegOp, OMod);
  }
  case AMDGPU::V_ADD_F64_e64:
  case AMDGPU::V_ADD_F64_pseudo_e64:
  case AMDGPU::V_ADD_F32_e64:
  case AMDGPU::V_ADD_F16_e64:
  case AMDGPU::V_ADD_F16_t16_e64:
  case AMDGPU::V_ADD_F16_fake16_e64: {
    // If output denormals are enabled, omod is ignored.
    if ((Op == AMDGPU::V_ADD_F32_e64 &&
         MFI->getMode().FP32Denormals.Output != DenormalMode::PreserveSign) ||
        ((Op == AMDGPU::V_ADD_F64_e64 || Op == AMDGPU::V_ADD_F64_pseudo_e64 ||
          Op == AMDGPU::V_ADD_F16_e64 || Op == AMDGPU::V_ADD_F16_t16_e64 ||
          Op == AMDGPU::V_ADD_F16_fake16_e64) &&
         MFI->getMode().FP64FP16Denormals.Output != DenormalMode::PreserveSign))
      return std::pair(nullptr, SIOutMods::NONE);

    // Look through the DAGCombiner canonicalization fmul x, 2 -> fadd x, x
    const MachineOperand *Src0 = TII->getNamedOperand(MI, AMDGPU::OpName::src0);
    const MachineOperand *Src1 = TII->getNamedOperand(MI, AMDGPU::OpName::src1);

    if (Src0->isReg() && Src1->isReg() && Src0->getReg() == Src1->getReg() &&
        Src0->getSubReg() == Src1->getSubReg() &&
        !TII->hasModifiersSet(MI, AMDGPU::OpName::src0_modifiers) &&
        !TII->hasModifiersSet(MI, AMDGPU::OpName::src1_modifiers) &&
        !TII->hasModifiersSet(MI, AMDGPU::OpName::clamp) &&
        !TII->hasModifiersSet(MI, AMDGPU::OpName::omod))
      return std::pair(Src0, SIOutMods::MUL2);

    return std::pair(nullptr, SIOutMods::NONE);
  }
  default:
    return std::pair(nullptr, SIOutMods::NONE);
  }
}
```
<a name="block_44"></a>**File:** llvm/lib/Target/AMDGPU/SIFoldOperands.cpp (L2282-2320)
```cpp
bool SIFoldOperandsImpl::tryFoldOMod(MachineInstr &MI) {
  const MachineOperand *RegOp;
  int OMod;
  std::tie(RegOp, OMod) = isOMod(MI);
  if (OMod == SIOutMods::NONE || !RegOp->isReg() ||
      RegOp->getSubReg() != AMDGPU::NoSubRegister ||
      !MRI->hasOneNonDBGUser(RegOp->getReg()))
    return false;

  MachineInstr *Def = MRI->getVRegDef(RegOp->getReg());
  MachineOperand *DefOMod = TII->getNamedOperand(*Def, AMDGPU::OpName::omod);
  if (!DefOMod || DefOMod->getImm() != SIOutMods::NONE)
    return false;

  if (Def->mayRaiseFPException())
    return false;

  // Clamp is applied after omod. If the source already has clamp set, don't
  // fold it.
  if (TII->hasModifiersSet(*Def, AMDGPU::OpName::clamp))
    return false;

  LLVM_DEBUG(dbgs() << "Folding omod " << MI << " into " << *Def);

  DefOMod->setImm(OMod);
  MRI->replaceRegWith(MI.getOperand(0).getReg(), Def->getOperand(0).getReg());
  // Kill flags can be wrong if we replaced a def inside a loop with a def
  // outside the loop.
  MRI->clearKillFlags(Def->getOperand(0).getReg());
  MI.eraseFromParent();

  // Use of output modifiers forces VOP3 encoding for a VOP2 mac/fmac
  // instruction, so we might as well convert it to the more flexible VOP3-only
  // mad/fma form.
  if (TII->convertToThreeAddress(*Def, nullptr, nullptr))
    Def->eraseFromParent();

  return true;
}
```
<a name="block_45"></a>**File:** llvm/lib/Target/AMDGPU/SIFoldOperands.cpp (L2324-2400)
```cpp
bool SIFoldOperandsImpl::tryFoldRegSequence(MachineInstr &MI) {
  assert(MI.isRegSequence());
  auto Reg = MI.getOperand(0).getReg();

  if (!ST->hasGFX90AInsts() || !TRI->isVGPR(*MRI, Reg) ||
      !MRI->hasOneNonDBGUse(Reg))
    return false;

  SmallVector<std::pair<MachineOperand*, unsigned>, 32> Defs;
  if (!getRegSeqInit(Defs, Reg))
    return false;

  for (auto &[Op, SubIdx] : Defs) {
    if (!Op->isReg())
      return false;
    if (TRI->isAGPR(*MRI, Op->getReg()))
      continue;
    // Maybe this is a COPY from AREG
    const MachineInstr *SubDef = MRI->getVRegDef(Op->getReg());
    if (!SubDef || !SubDef->isCopy() || SubDef->getOperand(1).getSubReg())
      return false;
    if (!TRI->isAGPR(*MRI, SubDef->getOperand(1).getReg()))
      return false;
  }

  MachineOperand *Op = &*MRI->use_nodbg_begin(Reg);
  MachineInstr *UseMI = Op->getParent();
  while (UseMI->isCopy() && !Op->getSubReg()) {
    Reg = UseMI->getOperand(0).getReg();
    if (!TRI->isVGPR(*MRI, Reg) || !MRI->hasOneNonDBGUse(Reg))
      return false;
    Op = &*MRI->use_nodbg_begin(Reg);
    UseMI = Op->getParent();
  }

  if (Op->getSubReg())
    return false;

  unsigned OpIdx = Op - &UseMI->getOperand(0);
  const MCInstrDesc &InstDesc = UseMI->getDesc();
  const TargetRegisterClass *OpRC =
      TII->getRegClass(InstDesc, OpIdx, TRI, *MI.getMF());
  if (!OpRC || !TRI->isVectorSuperClass(OpRC))
    return false;

  const auto *NewDstRC = TRI->getEquivalentAGPRClass(MRI->getRegClass(Reg));
  auto Dst = MRI->createVirtualRegister(NewDstRC);
  auto RS = BuildMI(*MI.getParent(), MI, MI.getDebugLoc(),
                    TII->get(AMDGPU::REG_SEQUENCE), Dst);

  for (auto &[Def, SubIdx] : Defs) {
    Def->setIsKill(false);
    if (TRI->isAGPR(*MRI, Def->getReg())) {
      RS.add(*Def);
    } else { // This is a copy
      MachineInstr *SubDef = MRI->getVRegDef(Def->getReg());
      SubDef->getOperand(1).setIsKill(false);
      RS.addReg(SubDef->getOperand(1).getReg(), 0, Def->getSubReg());
    }
    RS.addImm(SubIdx);
  }

  Op->setReg(Dst);
  if (!TII->isOperandLegal(*UseMI, OpIdx, Op)) {
    Op->setReg(Reg);
    RS->eraseFromParent();
    return false;
  }

  LLVM_DEBUG(dbgs() << "Folded " << *RS << " into " << *UseMI);

  // Erase the REG_SEQUENCE eagerly, unless we followed a chain of COPY users,
  // in which case we can erase them all later in runOnMachineFunction.
  if (MRI->use_nodbg_empty(MI.getOperand(0).getReg()))
    MI.eraseFromParent();
  return true;
}
```
<a name="block_46"></a>**File:** llvm/lib/Target/AMDGPU/SIFoldOperands.cpp (L2444-2572)
```cpp
//
// Example 1: LCSSA PHI
//      loop:
//        %1:vreg = COPY %0:areg
//      exit:
//        %2:vreg = PHI %1:vreg, %loop
//  =>
//      loop:
//      exit:
//        %1:areg = PHI %0:areg, %loop
//        %2:vreg = COPY %1:areg
//
// Example 2: PHI with multiple incoming values:
//      entry:
//        %1:vreg = GLOBAL_LOAD(..)
//      loop:
//        %2:vreg = PHI %1:vreg, %entry, %5:vreg, %loop
//        %3:areg = COPY %2:vreg
//        %4:areg = (instr using %3:areg)
//        %5:vreg = COPY %4:areg
//  =>
//      entry:
//        %1:vreg = GLOBAL_LOAD(..)
//        %2:areg = COPY %1:vreg
//      loop:
//        %3:areg = PHI %2:areg, %entry, %X:areg,
//        %4:areg = (instr using %3:areg)
bool SIFoldOperandsImpl::tryFoldPhiAGPR(MachineInstr &PHI) {
  assert(PHI.isPHI());

  Register PhiOut = PHI.getOperand(0).getReg();
  if (!TRI->isVGPR(*MRI, PhiOut))
    return false;

  // Iterate once over all incoming values of the PHI to check if this PHI is
  // eligible, and determine the exact AGPR RC we'll target.
  const TargetRegisterClass *ARC = nullptr;
  for (unsigned K = 1; K < PHI.getNumExplicitOperands(); K += 2) {
    MachineOperand &MO = PHI.getOperand(K);
    MachineInstr *Copy = MRI->getVRegDef(MO.getReg());
    if (!Copy || !Copy->isCopy())
      continue;

    Register AGPRSrc;
    unsigned AGPRRegMask = AMDGPU::NoSubRegister;
    if (!isAGPRCopy(*TRI, *MRI, *Copy, AGPRSrc, AGPRRegMask))
      continue;

    const TargetRegisterClass *CopyInRC = MRI->getRegClass(AGPRSrc);
    if (const auto *SubRC = TRI->getSubRegisterClass(CopyInRC, AGPRRegMask))
      CopyInRC = SubRC;

    if (ARC && !ARC->hasSubClassEq(CopyInRC))
      return false;
    ARC = CopyInRC;
  }

  if (!ARC)
    return false;

  bool IsAGPR32 = (ARC == &AMDGPU::AGPR_32RegClass);

  // Rewrite the PHI's incoming values to ARC.
  LLVM_DEBUG(dbgs() << "Folding AGPR copies into: " << PHI);
  for (unsigned K = 1; K < PHI.getNumExplicitOperands(); K += 2) {
    MachineOperand &MO = PHI.getOperand(K);
    Register Reg = MO.getReg();

    MachineBasicBlock::iterator InsertPt;
    MachineBasicBlock *InsertMBB = nullptr;

    // Look at the def of Reg, ignoring all copies.
    unsigned CopyOpc = AMDGPU::COPY;
    if (MachineInstr *Def = MRI->getVRegDef(Reg)) {

      // Look at pre-existing COPY instructions from ARC: Steal the operand. If
      // the copy was single-use, it will be removed by DCE later.
      if (Def->isCopy()) {
        Register AGPRSrc;
        unsigned AGPRSubReg = AMDGPU::NoSubRegister;
        if (isAGPRCopy(*TRI, *MRI, *Def, AGPRSrc, AGPRSubReg)) {
          MO.setReg(AGPRSrc);
          MO.setSubReg(AGPRSubReg);
          continue;
        }

        // If this is a multi-use SGPR -> VGPR copy, use V_ACCVGPR_WRITE on
        // GFX908 directly instead of a COPY. Otherwise, SIFoldOperand may try
        // to fold the sgpr -> vgpr -> agpr copy into a sgpr -> agpr copy which
        // is unlikely to be profitable.
        //
        // Note that V_ACCVGPR_WRITE is only used for AGPR_32.
        MachineOperand &CopyIn = Def->getOperand(1);
        if (IsAGPR32 && !ST->hasGFX90AInsts() && !MRI->hasOneNonDBGUse(Reg) &&
            TRI->isSGPRReg(*MRI, CopyIn.getReg()))
          CopyOpc = AMDGPU::V_ACCVGPR_WRITE_B32_e64;
      }

      InsertMBB = Def->getParent();
      InsertPt = InsertMBB->SkipPHIsLabelsAndDebug(++Def->getIterator());
    } else {
      InsertMBB = PHI.getOperand(MO.getOperandNo() + 1).getMBB();
      InsertPt = InsertMBB->getFirstTerminator();
    }

    Register NewReg = MRI->createVirtualRegister(ARC);
    MachineInstr *MI = BuildMI(*InsertMBB, InsertPt, PHI.getDebugLoc(),
                               TII->get(CopyOpc), NewReg)
                           .addReg(Reg);
    MO.setReg(NewReg);

    (void)MI;
    LLVM_DEBUG(dbgs() << "  Created COPY: " << *MI);
  }

  // Replace the PHI's result with a new register.
  Register NewReg = MRI->createVirtualRegister(ARC);
  PHI.getOperand(0).setReg(NewReg);

  // COPY that new register back to the original PhiOut register. This COPY will
  // usually be folded out later.
  MachineBasicBlock *MBB = PHI.getParent();
  BuildMI(*MBB, MBB->getFirstNonPHI(), PHI.getDebugLoc(),
          TII->get(AMDGPU::COPY), PhiOut)
      .addReg(NewReg);

  LLVM_DEBUG(dbgs() << "  Done: Folded " << PHI);
  return true;
}
```
<a name="block_47"></a>**File:** llvm/lib/Target/AMDGPU/SIFoldOperands.cpp (L2575-2627)
```cpp
bool SIFoldOperandsImpl::tryFoldLoad(MachineInstr &MI) {
  assert(MI.mayLoad());
  if (!ST->hasGFX90AInsts() || MI.getNumExplicitDefs() != 1)
    return false;

  MachineOperand &Def = MI.getOperand(0);
  if (!Def.isDef())
    return false;

  Register DefReg = Def.getReg();

  if (DefReg.isPhysical() || !TRI->isVGPR(*MRI, DefReg))
    return false;

  SmallVector<const MachineInstr *, 8> Users(
      llvm::make_pointer_range(MRI->use_nodbg_instructions(DefReg)));
  SmallVector<Register, 8> MoveRegs;

  if (Users.empty())
    return false;

  // Check that all uses a copy to an agpr or a reg_sequence producing an agpr.
  while (!Users.empty()) {
    const MachineInstr *I = Users.pop_back_val();
    if (!I->isCopy() && !I->isRegSequence())
      return false;
    Register DstReg = I->getOperand(0).getReg();
    // Physical registers may have more than one instruction definitions
    if (DstReg.isPhysical())
      return false;
    if (TRI->isAGPR(*MRI, DstReg))
      continue;
    MoveRegs.push_back(DstReg);
    for (const MachineInstr &U : MRI->use_nodbg_instructions(DstReg))
      Users.push_back(&U);
  }

  const TargetRegisterClass *RC = MRI->getRegClass(DefReg);
  MRI->setRegClass(DefReg, TRI->getEquivalentAGPRClass(RC));
  if (!TII->isOperandLegal(MI, 0, &Def)) {
    MRI->setRegClass(DefReg, RC);
    return false;
  }

  while (!MoveRegs.empty()) {
    Register Reg = MoveRegs.pop_back_val();
    MRI->setRegClass(Reg, TRI->getEquivalentAGPRClass(MRI->getRegClass(Reg)));
  }

  LLVM_DEBUG(dbgs() << "Folded " << MI);

  return true;
}
```
<a name="block_48"></a>**File:** llvm/lib/Target/AMDGPU/SIFoldOperands.cpp (L2629-2724)
```cpp
// tryFoldPhiAGPR will aggressively try to create AGPR PHIs.
// For GFX90A and later, this is pretty much always a good thing, but for GFX908
// there's cases where it can create a lot more AGPR-AGPR copies, which are
// expensive on this architecture due to the lack of V_ACCVGPR_MOV.
//
// This function looks at all AGPR PHIs in a basic block and collects their
// operands. Then, it checks for register that are used more than once across
// all PHIs and caches them in a VGPR. This prevents ExpandPostRAPseudo from
// having to create one VGPR temporary per use, which can get very messy if
// these PHIs come from a broken-up large PHI (e.g. 32 AGPR phis, one per vector
// element).
//
// Example
//      a:
//        %in:agpr_256 = COPY %foo:vgpr_256
//      c:
//        %x:agpr_32 = ..
//      b:
//        %0:areg = PHI %in.sub0:agpr_32, %a, %x, %c
//        %1:areg = PHI %in.sub0:agpr_32, %a, %y, %c
//        %2:areg = PHI %in.sub0:agpr_32, %a, %z, %c
//  =>
//      a:
//        %in:agpr_256 = COPY %foo:vgpr_256
//        %tmp:vgpr_32 = V_ACCVGPR_READ_B32_e64 %in.sub0:agpr_32
//        %tmp_agpr:agpr_32 = COPY %tmp
//      c:
//        %x:agpr_32 = ..
//      b:
//        %0:areg = PHI %tmp_agpr, %a, %x, %c
//        %1:areg = PHI %tmp_agpr, %a, %y, %c
//        %2:areg = PHI %tmp_agpr, %a, %z, %c
bool SIFoldOperandsImpl::tryOptimizeAGPRPhis(MachineBasicBlock &MBB) {
  // This is only really needed on GFX908 where AGPR-AGPR copies are
  // unreasonably difficult.
  if (ST->hasGFX90AInsts())
    return false;

  // Look at all AGPR Phis and collect the register + subregister used.
  DenseMap<std::pair<Register, unsigned>, std::vector<MachineOperand *>>
      RegToMO;

  for (auto &MI : MBB) {
    if (!MI.isPHI())
      break;

    if (!TRI->isAGPR(*MRI, MI.getOperand(0).getReg()))
      continue;

    for (unsigned K = 1; K < MI.getNumOperands(); K += 2) {
      MachineOperand &PhiMO = MI.getOperand(K);
      if (!PhiMO.getSubReg())
        continue;
      RegToMO[{PhiMO.getReg(), PhiMO.getSubReg()}].push_back(&PhiMO);
    }
  }

  // For all (Reg, SubReg) pair that are used more than once, cache the value in
  // a VGPR.
  bool Changed = false;
  for (const auto &[Entry, MOs] : RegToMO) {
    if (MOs.size() == 1)
      continue;

    const auto [Reg, SubReg] = Entry;
    MachineInstr *Def = MRI->getVRegDef(Reg);
    MachineBasicBlock *DefMBB = Def->getParent();

    // Create a copy in a VGPR using V_ACCVGPR_READ_B32_e64 so it's not folded
    // out.
    const TargetRegisterClass *ARC = getRegOpRC(*MRI, *TRI, *MOs.front());
    Register TempVGPR =
        MRI->createVirtualRegister(TRI->getEquivalentVGPRClass(ARC));
    MachineInstr *VGPRCopy =
        BuildMI(*DefMBB, ++Def->getIterator(), Def->getDebugLoc(),
                TII->get(AMDGPU::V_ACCVGPR_READ_B32_e64), TempVGPR)
            .addReg(Reg, /* flags */ 0, SubReg);

    // Copy back to an AGPR and use that instead of the AGPR subreg in all MOs.
    Register TempAGPR = MRI->createVirtualRegister(ARC);
    BuildMI(*DefMBB, ++VGPRCopy->getIterator(), Def->getDebugLoc(),
            TII->get(AMDGPU::COPY), TempAGPR)
        .addReg(TempVGPR);

    LLVM_DEBUG(dbgs() << "Caching AGPR into VGPR: " << *VGPRCopy);
    for (MachineOperand *MO : MOs) {
      MO->setReg(TempAGPR);
      MO->setSubReg(AMDGPU::NoSubRegister);
      LLVM_DEBUG(dbgs() << "  Changed PHI Operand: " << *MO << "\n");
    }

    Changed = true;
  }

  return Changed;
}
```
<a name="block_49"></a>**File:** llvm/lib/Target/AMDGPU/SIFoldOperands.cpp (L2726-2786)
```cpp
bool SIFoldOperandsImpl::run(MachineFunction &MF) {
  MRI = &MF.getRegInfo();
  ST = &MF.getSubtarget<GCNSubtarget>();
  TII = ST->getInstrInfo();
  TRI = &TII->getRegisterInfo();
  MFI = MF.getInfo<SIMachineFunctionInfo>();

  // omod is ignored by hardware if IEEE bit is enabled. omod also does not
  // correctly handle signed zeros.
  //
  // FIXME: Also need to check strictfp
  bool IsIEEEMode = MFI->getMode().IEEE;
  bool HasNSZ = MFI->hasNoSignedZerosFPMath();

  bool Changed = false;
  for (MachineBasicBlock *MBB : depth_first(&MF)) {
    MachineOperand *CurrentKnownM0Val = nullptr;
    for (auto &MI : make_early_inc_range(*MBB)) {
      Changed |= tryFoldCndMask(MI);

      if (tryFoldZeroHighBits(MI)) {
        Changed = true;
        continue;
      }

      if (MI.isRegSequence() && tryFoldRegSequence(MI)) {
        Changed = true;
        continue;
      }

      if (MI.isPHI() && tryFoldPhiAGPR(MI)) {
        Changed = true;
        continue;
      }

      if (MI.mayLoad() && tryFoldLoad(MI)) {
        Changed = true;
        continue;
      }

      if (TII->isFoldableCopy(MI)) {
        Changed |= tryFoldFoldableCopy(MI, CurrentKnownM0Val);
        continue;
      }

      // Saw an unknown clobber of m0, so we no longer know what it is.
      if (CurrentKnownM0Val && MI.modifiesRegister(AMDGPU::M0, TRI))
        CurrentKnownM0Val = nullptr;

      // TODO: Omod might be OK if there is NSZ only on the source
      // instruction, and not the omod multiply.
      if (IsIEEEMode || (!HasNSZ && !MI.getFlag(MachineInstr::FmNsz)) ||
          !tryFoldOMod(MI))
        Changed |= tryFoldClamp(MI);
    }

    Changed |= tryOptimizeAGPRPhis(*MBB);
  }

  return Changed;
}
```