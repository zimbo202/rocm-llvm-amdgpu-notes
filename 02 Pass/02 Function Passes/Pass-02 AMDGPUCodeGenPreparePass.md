# AMDGPUCodeGenPrepare.cpp 代码功能详解

<a name="ref-block_0"></a>## 1. Pass 的主要功能概述 llvm-project:8-12[<sup>↗</sup>](#block_0) 

**AMDGPUCodeGenPrepare** 是一个在指令选择前运行的优化 Pass，主要用于对 AMDGPU 目标架构进行 IR 级别的各种优化。

**作用及效果：**
- 将某些操作转换为更适合 AMDGPU 硬件的形式
- 优化除法、平方根等计算密集型操作
- 扩展或收缩数据类型以匹配硬件特性
- 改进代码生成质量，减少寄存器压力
- 为后续的指令选择阶段准备更优的 IR

## 2. 主要功能步骤和子功能提取

根据代码分析，该 Pass 包含以下主要子功能：

<a name="ref-block_2"></a>### 2.1 窄整数操作提升（Narrow Integer Promotion） llvm-project:470-515[<sup>↗</sup>](#block_2) llvm-project:517-542 llvm-project:544-571 

<a name="ref-block_5"></a>### 2.2 Mul24 内置函数替换（Mul24 Intrinsic Replacement） llvm-project:633-694[<sup>↗</sup>](#block_5) 

<a name="ref-block_12"></a>### 2.3 整数除法/取余扩展（Integer Division/Remainder Expansion） llvm-project:1236-1244[<sup>↗</sup>](#block_12) llvm-project:1389-1511 llvm-project:1513-1542 llvm-project:1544-1558 

<a name="ref-block_11"></a>### 2.4 浮点除法优化（Floating-Point Division Optimization） llvm-project:1078-1169[<sup>↗</sup>](#block_11) llvm-project:948-1003 llvm-project:906-940 llvm-project:1012-1035 

<a name="ref-block_18"></a>### 2.5 标量加载扩宽（Scalar Load Widening） llvm-project:1727-1771[<sup>↗</sup>](#block_18) 

<a name="ref-block_19"></a>### 2.6 大型 PHI 节点拆分（Large PHI Node Breaking） llvm-project:2033-2114[<sup>↗</sup>](#block_19) 

<a name="ref-block_6"></a>### 2.7 二元操作折叠到 Select（Binary Operation Folding into Select） llvm-project:711-770[<sup>↗</sup>](#block_6) 

<a name="ref-block_20"></a>### 2.8 地址空间转换优化（Address Space Cast Optimization） llvm-project:2160-2195[<sup>↗</sup>](#block_20) 

<a name="ref-block_23"></a>### 2.9 平方根优化（Square Root Optimization） llvm-project:2313-2369[<sup>↗</sup>](#block_23) 

<a name="ref-block_21"></a>### 2.10 Fract 模式匹配和优化（Fract Pattern Matching） llvm-project:2229-2266[<sup>↗</sup>](#block_21) llvm-project:2284-2305 

<a name="ref-block_16"></a>### 2.11 数学运算收窄（Math Operation Narrowing） llvm-project:1572-1632[<sup>↗</sup>](#block_16) 

## 3. 各子功能的具体描述分析

### 3.1 窄整数操作提升
**功能：** 将 uniform 的 16 位及以下的整数操作提升到 32 位
- 对于二元操作：先将操作数扩展（符号扩展或零扩展）到 32 位，执行 32 位操作，再截断回原类型
- 对于 icmp 指令：扩展操作数后直接比较
- 对于 select 指令：扩展 true/false 值，执行选择，再截断结果
- **原因：** AMDGPU 硬件对 32 位操作支持更好，16 位操作在某些情况下可能效率较低

### 3.2 Mul24 内置函数替换
**功能：** 将乘法指令替换为 `amdgcn.mul.u24` 或 `amdgcn.mul.s24` 内置函数
- 检测操作数是否可以用 24 位表示（无符号或有符号）
- 仅对非 uniform 的乘法进行替换（因为 uniform 值更适合使用标量 `s_mul_i32`）
- **效果：** 24 位乘法在 AMDGPU 上有专门的硬件支持，速度更快

### 3.3 整数除法/取余扩展
**功能：** 将除法和取余操作展开为更高效的指令序列
- **24 位展开：** 使用浮点运算实现，包括 rcp（倒数）、fmul、trunc 等操作
- **32 位展开：** 基于 Newton-Raphson 迭代方法，使用 rcp 估算初值，然后进行商/余数细化
- **64 位处理：** 尝试收缩到 32 位或 24 位处理；如果不可行，使用通用的 64 位展开
- **特殊优化检测：** 对常量除数或 2 的幂等特殊情况，保留给后续优化处理

### 3.4 浮点除法优化
**功能：** 根据精度要求和快速数学标志优化浮点除法
- **rsq 优化：** 将 `1.0/sqrt(x)` 转换为 `rsq(x)` 内置函数
- **rcp 优化：** 将 `1.0/x` 或 `a/b` 转换为使用 `rcp` 内置函数
- **fdiv.fast 优化：** 对于精度要求 ≥2.5 ULP 的情况，使用 `amdgcn.fdiv.fast` 内置函数
- **frexp 展开：** 使用 frexp 进行输入缩放以正确处理非正规数
- 包含对非正规数的特殊处理逻辑

### 3.5 标量加载扩宽
**功能：** 将常量地址空间的小于 32 位的加载扩宽到 32 位
- 仅对 uniform、简单加载、对齐 ≥4 字节的情况进行扩宽
- 加载 32 位后截断回原类型
- **效果：** 标量加载比向量加载更高效，避免不必要的向量化

### 3.6 大型 PHI 节点拆分
**功能：** 将大型向量 PHI 节点拆分成多个小型 PHI
- 主要针对 DAGISel，因为它对大型 PHI 处理不佳
- 将向量拆分为 32 位切片或标量元素
- 对每个切片创建独立的 PHI，然后重新组合
- 包含启发式判断，只有在可能折叠 extractelement 指令时才拆分
- **效果：** 减少寄存器压力，避免栈溢出

### 3.7 二元操作折叠到 Select
**功能：** 将形如 `binop(select(cond, a, b), c)` 的模式折叠为 `select(cond, binop(a, c), binop(b, c))`
- 仅当操作数都是常量且折叠后不产生 ConstantExpr 时进行
- 主要用于改进除法操作的优化机会
- **效果：** 减少运行时计算，将计算提前到编译时

### 3.8 地址空间转换优化
**功能：** 将 addrspacecast 指令替换为 `amdgcn.addrspacecast.nonnull` 内置函数
- 仅对 flat ↔ local/private 的转换进行优化
- 需要证明源指针永不为空
- **效果：** 使用非空版本可以避免空指针检查，提高效率

### 3.9 平方根优化
**功能：** 展开 `sqrt` 内置函数为更高效的指令序列
- 对于可以忽略非正规数的情况，直接使用 `amdgcn.sqrt` 内置函数
- 否则使用 ldexp 进行缩放以正确处理非正规数
- 精度要求 ≥1 ULP
- 避免与后续 rsq 优化冲突

### 3.10 Fract 模式匹配
**功能：** 识别并优化分数部分计算模式
- 匹配 `minnum(fsub(x, floor(x)), nextafter(1.0, -1.0))` 模式
- 替换为 `amdgcn.fract` 内置函数
- 需要硬件支持（某些芯片有 fract bug）
- **效果：** 使用单个硬件指令替代多个操作

### 3.11 数学运算收窄
**功能：** 将加法和乘法操作收窄到实际需要的位宽
- 基于已知位信息判断是否可以安全收窄
- 进行成本分析，只有在收窄后成本更低时才执行
- **效果：** 减少寄存器使用，提高效率

## 4. 子功能之间的关系

<a name="ref-block_1"></a>### 4.1 主执行流程 llvm-project:354-381[<sup>↗</sup>](#block_1) 

主函数 `run()` 遍历所有基本块和指令，使用访问者模式调用相应的处理函数。

<a name="ref-block_17"></a>### 4.2 访问者模式分发 llvm-project:1634-1725[<sup>↗</sup>](#block_17) 

`visitBinaryOperator` 协调多个优化步骤的执行顺序：
1. 先尝试 foldBinOpIntoSelect
2. 然后检查是否需要提升到 32 位
3. 接着尝试 mul24 替换
4. 最后处理除法/取余扩展

### 4.3 优化优先级和互斥关系

<a name="ref-block_10"></a>**浮点除法优化的优先级链：** llvm-project:1037-1061[<sup>↗</sup>](#block_10) 

1. 首先尝试 rsq 优化（`1/sqrt(x)` 模式）
2. 其次尝试 rcp 优化（`1/x` 或 `a*rcp(b)` 模式）
3. 然后考虑 fdiv.fast
4. 最后使用 frexp 展开

**整数除法的分层处理：**
- 24 位优先，速度最快
- 32 位次之，使用 Newton-Raphson
- 64 位最后，可能收缩到小位宽或使用通用展开

### 4.4 条件依赖关系

- **窄整数提升：** 依赖 uniformity 分析和硬件特性（`has16BitInsts()`）
- **Mul24 替换：** 排除 uniform 值（留给标量乘法）和已有 16 位指令支持的情况
- **PHI 拆分：** 仅在 DAGISel 模式下启用，GlobalISel 不需要
- **加载扩宽：** 需要 uniform、特定地址空间、对齐要求

### 4.5 共享辅助功能

多个优化共享以下辅助函数：
- `extractValues` / `insertValues`：向量元素提取和插入
- `computeKnownBits` / `ComputeNumSignBits`：位信息分析
- `getFrexpResults`：frexp 结果获取（用于除法和平方根优化）
- `canIgnoreDenormalInput`：非正规数检测（用于浮点优化）

<a name="ref-block_24"></a>### 4.6 Pass 的整体协调 llvm-project:2371-2389[<sup>↗</sup>](#block_24) 

Pass 的入口点协调所有分析结果（UniformityAnalysis、DominatorTree、AssumptionCache 等），传递给实现类并执行优化。

**总结：** AMDGPUCodeGenPrepare 是一个复杂的多功能 Pass，通过访问者模式协调多个优化步骤，这些优化步骤之间有明确的优先级和依赖关系，共同作用于 IR 以生成更适合 AMDGPU 架构的代码。
### Citations
<a name="block_0"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUCodeGenPrepare.cpp (L8-12) [<sup>↩</sup>](#ref-block_0)
```cpp
//
/// \file
/// This pass does misc. AMDGPU optimizations on IR before instruction
/// selection.
//
```
<a name="block_1"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUCodeGenPrepare.cpp (L354-381) [<sup>↩</sup>](#ref-block_1)
```cpp
bool AMDGPUCodeGenPrepareImpl::run() {
  BreakPhiNodesCache.clear();
  bool MadeChange = false;

  Function::iterator NextBB;
  for (Function::iterator FI = F.begin(), FE = F.end(); FI != FE; FI = NextBB) {
    BasicBlock *BB = &*FI;
    NextBB = std::next(FI);

    BasicBlock::iterator Next;
    for (BasicBlock::iterator I = BB->begin(), E = BB->end(); I != E;
         I = Next) {
      Next = std::next(I);

      MadeChange |= visit(*I);

      if (Next != E) { // Control flow changed
        BasicBlock *NextInstBB = Next->getParent();
        if (NextInstBB != BB) {
          BB = NextInstBB;
          E = BB->end();
          FE = F.end();
        }
      }
    }
  }
  return MadeChange;
}
```
<a name="block_2"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUCodeGenPrepare.cpp (L470-515) [<sup>↩</sup>](#ref-block_2)
```cpp
bool AMDGPUCodeGenPrepareImpl::promoteUniformOpToI32(BinaryOperator &I) const {
  assert(needsPromotionToI32(I.getType()) &&
         "I does not need promotion to i32");

  if (I.getOpcode() == Instruction::SDiv ||
      I.getOpcode() == Instruction::UDiv ||
      I.getOpcode() == Instruction::SRem ||
      I.getOpcode() == Instruction::URem)
    return false;

  IRBuilder<> Builder(&I);
  Builder.SetCurrentDebugLocation(I.getDebugLoc());

  Type *I32Ty = getI32Ty(Builder, I.getType());
  Value *ExtOp0 = nullptr;
  Value *ExtOp1 = nullptr;
  Value *ExtRes = nullptr;
  Value *TruncRes = nullptr;

  if (isSigned(I)) {
    ExtOp0 = Builder.CreateSExt(I.getOperand(0), I32Ty);
    ExtOp1 = Builder.CreateSExt(I.getOperand(1), I32Ty);
  } else {
    ExtOp0 = Builder.CreateZExt(I.getOperand(0), I32Ty);
    ExtOp1 = Builder.CreateZExt(I.getOperand(1), I32Ty);
  }

  ExtRes = Builder.CreateBinOp(I.getOpcode(), ExtOp0, ExtOp1);
  if (Instruction *Inst = dyn_cast<Instruction>(ExtRes)) {
    if (promotedOpIsNSW(cast<Instruction>(I)))
      Inst->setHasNoSignedWrap();

    if (promotedOpIsNUW(cast<Instruction>(I)))
      Inst->setHasNoUnsignedWrap();

    if (const auto *ExactOp = dyn_cast<PossiblyExactOperator>(&I))
      Inst->setIsExact(ExactOp->isExact());
  }

  TruncRes = Builder.CreateTrunc(ExtRes, I.getType());

  I.replaceAllUsesWith(TruncRes);
  I.eraseFromParent();

  return true;
}
```
<a name="block_3"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUCodeGenPrepare.cpp (L517-542)
```cpp
bool AMDGPUCodeGenPrepareImpl::promoteUniformOpToI32(ICmpInst &I) const {
  assert(needsPromotionToI32(I.getOperand(0)->getType()) &&
         "I does not need promotion to i32");

  IRBuilder<> Builder(&I);
  Builder.SetCurrentDebugLocation(I.getDebugLoc());

  Type *I32Ty = getI32Ty(Builder, I.getOperand(0)->getType());
  Value *ExtOp0 = nullptr;
  Value *ExtOp1 = nullptr;
  Value *NewICmp  = nullptr;

  if (I.isSigned()) {
    ExtOp0 = Builder.CreateSExt(I.getOperand(0), I32Ty);
    ExtOp1 = Builder.CreateSExt(I.getOperand(1), I32Ty);
  } else {
    ExtOp0 = Builder.CreateZExt(I.getOperand(0), I32Ty);
    ExtOp1 = Builder.CreateZExt(I.getOperand(1), I32Ty);
  }
  NewICmp = Builder.CreateICmp(I.getPredicate(), ExtOp0, ExtOp1);

  I.replaceAllUsesWith(NewICmp);
  I.eraseFromParent();

  return true;
}
```
<a name="block_4"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUCodeGenPrepare.cpp (L544-571)
```cpp
bool AMDGPUCodeGenPrepareImpl::promoteUniformOpToI32(SelectInst &I) const {
  assert(needsPromotionToI32(I.getType()) &&
         "I does not need promotion to i32");

  IRBuilder<> Builder(&I);
  Builder.SetCurrentDebugLocation(I.getDebugLoc());

  Type *I32Ty = getI32Ty(Builder, I.getType());
  Value *ExtOp1 = nullptr;
  Value *ExtOp2 = nullptr;
  Value *ExtRes = nullptr;
  Value *TruncRes = nullptr;

  if (isSigned(I)) {
    ExtOp1 = Builder.CreateSExt(I.getOperand(1), I32Ty);
    ExtOp2 = Builder.CreateSExt(I.getOperand(2), I32Ty);
  } else {
    ExtOp1 = Builder.CreateZExt(I.getOperand(1), I32Ty);
    ExtOp2 = Builder.CreateZExt(I.getOperand(2), I32Ty);
  }
  ExtRes = Builder.CreateSelect(I.getOperand(0), ExtOp1, ExtOp2);
  TruncRes = Builder.CreateTrunc(ExtRes, I.getType());

  I.replaceAllUsesWith(TruncRes);
  I.eraseFromParent();

  return true;
}
```
<a name="block_5"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUCodeGenPrepare.cpp (L633-694) [<sup>↩</sup>](#ref-block_5)
```cpp
bool AMDGPUCodeGenPrepareImpl::replaceMulWithMul24(BinaryOperator &I) const {
  if (I.getOpcode() != Instruction::Mul)
    return false;

  Type *Ty = I.getType();
  unsigned Size = Ty->getScalarSizeInBits();
  if (Size <= 16 && ST.has16BitInsts())
    return false;

  // Prefer scalar if this could be s_mul_i32
  if (UA.isUniform(&I))
    return false;

  Value *LHS = I.getOperand(0);
  Value *RHS = I.getOperand(1);
  IRBuilder<> Builder(&I);
  Builder.SetCurrentDebugLocation(I.getDebugLoc());

  unsigned LHSBits = 0, RHSBits = 0;
  bool IsSigned = false;

  if (ST.hasMulU24() && (LHSBits = numBitsUnsigned(LHS)) <= 24 &&
      (RHSBits = numBitsUnsigned(RHS)) <= 24) {
    IsSigned = false;

  } else if (ST.hasMulI24() && (LHSBits = numBitsSigned(LHS)) <= 24 &&
             (RHSBits = numBitsSigned(RHS)) <= 24) {
    IsSigned = true;

  } else
    return false;

  SmallVector<Value *, 4> LHSVals;
  SmallVector<Value *, 4> RHSVals;
  SmallVector<Value *, 4> ResultVals;
  extractValues(Builder, LHSVals, LHS);
  extractValues(Builder, RHSVals, RHS);

  IntegerType *I32Ty = Builder.getInt32Ty();
  IntegerType *IntrinTy = Size > 32 ? Builder.getInt64Ty() : I32Ty;
  Type *DstTy = LHSVals[0]->getType();

  for (int I = 0, E = LHSVals.size(); I != E; ++I) {
    Value *LHS = IsSigned ? Builder.CreateSExtOrTrunc(LHSVals[I], I32Ty)
                          : Builder.CreateZExtOrTrunc(LHSVals[I], I32Ty);
    Value *RHS = IsSigned ? Builder.CreateSExtOrTrunc(RHSVals[I], I32Ty)
                          : Builder.CreateZExtOrTrunc(RHSVals[I], I32Ty);
    Intrinsic::ID ID =
        IsSigned ? Intrinsic::amdgcn_mul_i24 : Intrinsic::amdgcn_mul_u24;
    Value *Result = Builder.CreateIntrinsic(ID, {IntrinTy}, {LHS, RHS});
    Result = IsSigned ? Builder.CreateSExtOrTrunc(Result, DstTy)
                      : Builder.CreateZExtOrTrunc(Result, DstTy);
    ResultVals.push_back(Result);
  }

  Value *NewVal = insertValues(Builder, Ty, ResultVals);
  NewVal->takeName(&I);
  I.replaceAllUsesWith(NewVal);
  I.eraseFromParent();

  return true;
}
```
<a name="block_6"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUCodeGenPrepare.cpp (L711-770) [<sup>↩</sup>](#ref-block_6)
```cpp
bool AMDGPUCodeGenPrepareImpl::foldBinOpIntoSelect(BinaryOperator &BO) const {
  // Don't do this unless the old select is going away. We want to eliminate the
  // binary operator, not replace a binop with a select.
  int SelOpNo = 0;

  CastInst *CastOp;

  // TODO: Should probably try to handle some cases with multiple
  // users. Duplicating the select may be profitable for division.
  SelectInst *Sel = findSelectThroughCast(BO.getOperand(0), CastOp);
  if (!Sel || !Sel->hasOneUse()) {
    SelOpNo = 1;
    Sel = findSelectThroughCast(BO.getOperand(1), CastOp);
  }

  if (!Sel || !Sel->hasOneUse())
    return false;

  Constant *CT = dyn_cast<Constant>(Sel->getTrueValue());
  Constant *CF = dyn_cast<Constant>(Sel->getFalseValue());
  Constant *CBO = dyn_cast<Constant>(BO.getOperand(SelOpNo ^ 1));
  if (!CBO || !CT || !CF)
    return false;

  if (CastOp) {
    if (!CastOp->hasOneUse())
      return false;
    CT = ConstantFoldCastOperand(CastOp->getOpcode(), CT, BO.getType(), DL);
    CF = ConstantFoldCastOperand(CastOp->getOpcode(), CF, BO.getType(), DL);
  }

  // TODO: Handle special 0/-1 cases DAG combine does, although we only really
  // need to handle divisions here.
  Constant *FoldedT =
      SelOpNo ? ConstantFoldBinaryOpOperands(BO.getOpcode(), CBO, CT, DL)
              : ConstantFoldBinaryOpOperands(BO.getOpcode(), CT, CBO, DL);
  if (!FoldedT || isa<ConstantExpr>(FoldedT))
    return false;

  Constant *FoldedF =
      SelOpNo ? ConstantFoldBinaryOpOperands(BO.getOpcode(), CBO, CF, DL)
              : ConstantFoldBinaryOpOperands(BO.getOpcode(), CF, CBO, DL);
  if (!FoldedF || isa<ConstantExpr>(FoldedF))
    return false;

  IRBuilder<> Builder(&BO);
  Builder.SetCurrentDebugLocation(BO.getDebugLoc());
  if (const FPMathOperator *FPOp = dyn_cast<const FPMathOperator>(&BO))
    Builder.setFastMathFlags(FPOp->getFastMathFlags());

  Value *NewSelect = Builder.CreateSelect(Sel->getCondition(),
                                          FoldedT, FoldedF);
  NewSelect->takeName(&BO);
  BO.replaceAllUsesWith(NewSelect);
  BO.eraseFromParent();
  if (CastOp)
    CastOp->eraseFromParent();
  Sel->eraseFromParent();
  return true;
}
```
<a name="block_7"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUCodeGenPrepare.cpp (L906-940)
```cpp
Value *AMDGPUCodeGenPrepareImpl::optimizeWithRsq(
    IRBuilder<> &Builder, Value *Num, Value *Den, const FastMathFlags DivFMF,
    const FastMathFlags SqrtFMF, const Instruction *CtxI) const {
  // The rsqrt contraction increases accuracy from ~2ulp to ~1ulp.
  assert(DivFMF.allowContract() && SqrtFMF.allowContract());

  // rsq_f16 is accurate to 0.51 ulp.
  // rsq_f32 is accurate for !fpmath >= 1.0ulp and denormals are flushed.
  // rsq_f64 is never accurate.
  const ConstantFP *CLHS = dyn_cast<ConstantFP>(Num);
  if (!CLHS)
    return nullptr;

  assert(Den->getType()->isFloatTy());

  bool IsNegative = false;

  // TODO: Handle other numerator values with arcp.
  if (CLHS->isExactlyValue(1.0) || (IsNegative = CLHS->isExactlyValue(-1.0))) {
    // Add in the sqrt flags.
    IRBuilder<>::FastMathFlagGuard Guard(Builder);
    Builder.setFastMathFlags(DivFMF | SqrtFMF);

    if ((DivFMF.approxFunc() && SqrtFMF.approxFunc()) || HasUnsafeFPMath ||
        canIgnoreDenormalInput(Den, CtxI)) {
      Value *Result = Builder.CreateUnaryIntrinsic(Intrinsic::amdgcn_rsq, Den);
      // -1.0 / sqrt(x) -> fneg(rsq(x))
      return IsNegative ? Builder.CreateFNeg(Result) : Result;
    }

    return emitRsqIEEE1ULP(Builder, Den, IsNegative);
  }

  return nullptr;
}
```
<a name="block_8"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUCodeGenPrepare.cpp (L948-1003)
```cpp
Value *
AMDGPUCodeGenPrepareImpl::optimizeWithRcp(IRBuilder<> &Builder, Value *Num,
                                          Value *Den, FastMathFlags FMF,
                                          const Instruction *CtxI) const {
  // rcp_f16 is accurate to 0.51 ulp.
  // rcp_f32 is accurate for !fpmath >= 1.0ulp and denormals are flushed.
  // rcp_f64 is never accurate.
  assert(Den->getType()->isFloatTy());

  if (const ConstantFP *CLHS = dyn_cast<ConstantFP>(Num)) {
    bool IsNegative = false;
    if (CLHS->isExactlyValue(1.0) ||
        (IsNegative = CLHS->isExactlyValue(-1.0))) {
      Value *Src = Den;

      if (HasFP32DenormalFlush || FMF.approxFunc()) {
        // -1.0 / x -> 1.0 / fneg(x)
        if (IsNegative)
          Src = Builder.CreateFNeg(Src);

        // v_rcp_f32 and v_rsq_f32 do not support denormals, and according to
        // the CI documentation has a worst case error of 1 ulp.
        // OpenCL requires <= 2.5 ulp for 1.0 / x, so it should always be OK
        // to use it as long as we aren't trying to use denormals.
        //
        // v_rcp_f16 and v_rsq_f16 DO support denormals.

        // NOTE: v_sqrt and v_rcp will be combined to v_rsq later. So we don't
        //       insert rsq intrinsic here.

        // 1.0 / x -> rcp(x)
        return Builder.CreateUnaryIntrinsic(Intrinsic::amdgcn_rcp, Src);
      }

      // TODO: If the input isn't denormal, and we know the input exponent isn't
      // big enough to introduce a denormal we can avoid the scaling.
      return emitRcpIEEE1ULP(Builder, Src, IsNegative);
    }
  }

  if (FMF.allowReciprocal()) {
    // x / y -> x * (1.0 / y)

    // TODO: Could avoid denormal scaling and use raw rcp if we knew the output
    // will never underflow.
    if (HasFP32DenormalFlush || FMF.approxFunc()) {
      Value *Recip = Builder.CreateUnaryIntrinsic(Intrinsic::amdgcn_rcp, Den);
      return Builder.CreateFMul(Num, Recip);
    }

    Value *Recip = emitRcpIEEE1ULP(Builder, Den, false);
    return Builder.CreateFMul(Num, Recip);
  }

  return nullptr;
}
```
<a name="block_9"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUCodeGenPrepare.cpp (L1012-1035)
```cpp
Value *AMDGPUCodeGenPrepareImpl::optimizeWithFDivFast(
    IRBuilder<> &Builder, Value *Num, Value *Den, float ReqdAccuracy) const {
  // fdiv.fast can achieve 2.5 ULP accuracy.
  if (ReqdAccuracy < 2.5f)
    return nullptr;

  // Only have fdiv.fast for f32.
  assert(Den->getType()->isFloatTy());

  bool NumIsOne = false;
  if (const ConstantFP *CNum = dyn_cast<ConstantFP>(Num)) {
    if (CNum->isExactlyValue(+1.0) || CNum->isExactlyValue(-1.0))
      NumIsOne = true;
  }

  // fdiv does not support denormals. But 1.0/x is always fine to use it.
  //
  // TODO: This works for any value with a specific known exponent range, don't
  // just limit to constant 1.
  if (!HasFP32DenormalFlush && !NumIsOne)
    return nullptr;

  return Builder.CreateIntrinsic(Intrinsic::amdgcn_fdiv_fast, {Num, Den});
}
```
<a name="block_10"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUCodeGenPrepare.cpp (L1037-1061) [<sup>↩</sup>](#ref-block_10)
```cpp
Value *AMDGPUCodeGenPrepareImpl::visitFDivElement(
    IRBuilder<> &Builder, Value *Num, Value *Den, FastMathFlags DivFMF,
    FastMathFlags SqrtFMF, Value *RsqOp, const Instruction *FDivInst,
    float ReqdDivAccuracy) const {
  if (RsqOp) {
    Value *Rsq =
        optimizeWithRsq(Builder, Num, RsqOp, DivFMF, SqrtFMF, FDivInst);
    if (Rsq)
      return Rsq;
  }

  Value *Rcp = optimizeWithRcp(Builder, Num, Den, DivFMF, FDivInst);
  if (Rcp)
    return Rcp;

  // In the basic case fdiv_fast has the same instruction count as the frexp div
  // expansion. Slightly prefer fdiv_fast since it ends in an fmul that can
  // potentially be fused into a user. Also, materialization of the constants
  // can be reused for multiple instances.
  Value *FDivFast = optimizeWithFDivFast(Builder, Num, Den, ReqdDivAccuracy);
  if (FDivFast)
    return FDivFast;

  return emitFrexpDiv(Builder, Num, Den, DivFMF);
}
```
<a name="block_11"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUCodeGenPrepare.cpp (L1078-1169) [<sup>↩</sup>](#ref-block_11)
```cpp
bool AMDGPUCodeGenPrepareImpl::visitFDiv(BinaryOperator &FDiv) {
  if (DisableFDivExpand)
    return false;

  Type *Ty = FDiv.getType()->getScalarType();
  if (!Ty->isFloatTy())
    return false;

  // The f64 rcp/rsq approximations are pretty inaccurate. We can do an
  // expansion around them in codegen. f16 is good enough to always use.

  const FPMathOperator *FPOp = cast<const FPMathOperator>(&FDiv);
  const FastMathFlags DivFMF = FPOp->getFastMathFlags();
  const float ReqdAccuracy = FPOp->getFPAccuracy();

  FastMathFlags SqrtFMF;

  Value *Num = FDiv.getOperand(0);
  Value *Den = FDiv.getOperand(1);

  Value *RsqOp = nullptr;
  auto *DenII = dyn_cast<IntrinsicInst>(Den);
  if (DenII && DenII->getIntrinsicID() == Intrinsic::sqrt &&
      DenII->hasOneUse()) {
    const auto *SqrtOp = cast<FPMathOperator>(DenII);
    SqrtFMF = SqrtOp->getFastMathFlags();
    if (canOptimizeWithRsq(SqrtOp, DivFMF, SqrtFMF))
      RsqOp = SqrtOp->getOperand(0);
  }

  // Inaccurate rcp is allowed with unsafe-fp-math or afn.
  //
  // Defer to codegen to handle this.
  //
  // TODO: Decide on an interpretation for interactions between afn + arcp +
  // !fpmath, and make it consistent between here and codegen. For now, defer
  // expansion of afn to codegen. The current interpretation is so aggressive we
  // don't need any pre-consideration here when we have better information. A
  // more conservative interpretation could use handling here.
  const bool AllowInaccurateRcp = HasUnsafeFPMath || DivFMF.approxFunc();
  if (!RsqOp && AllowInaccurateRcp)
    return false;

  // Defer the correct implementations to codegen.
  if (ReqdAccuracy < 1.0f)
    return false;

  IRBuilder<> Builder(FDiv.getParent(), std::next(FDiv.getIterator()));
  Builder.setFastMathFlags(DivFMF);
  Builder.SetCurrentDebugLocation(FDiv.getDebugLoc());

  SmallVector<Value *, 4> NumVals;
  SmallVector<Value *, 4> DenVals;
  SmallVector<Value *, 4> RsqDenVals;
  extractValues(Builder, NumVals, Num);
  extractValues(Builder, DenVals, Den);

  if (RsqOp)
    extractValues(Builder, RsqDenVals, RsqOp);

  SmallVector<Value *, 4> ResultVals(NumVals.size());
  for (int I = 0, E = NumVals.size(); I != E; ++I) {
    Value *NumElt = NumVals[I];
    Value *DenElt = DenVals[I];
    Value *RsqDenElt = RsqOp ? RsqDenVals[I] : nullptr;

    Value *NewElt =
        visitFDivElement(Builder, NumElt, DenElt, DivFMF, SqrtFMF, RsqDenElt,
                         cast<Instruction>(FPOp), ReqdAccuracy);
    if (!NewElt) {
      // Keep the original, but scalarized.

      // This has the unfortunate side effect of sometimes scalarizing when
      // we're not going to do anything.
      NewElt = Builder.CreateFDiv(NumElt, DenElt);
      if (auto *NewEltInst = dyn_cast<Instruction>(NewElt))
        NewEltInst->copyMetadata(FDiv);
    }

    ResultVals[I] = NewElt;
  }

  Value *NewVal = insertValues(Builder, FDiv.getType(), ResultVals);

  if (NewVal) {
    FDiv.replaceAllUsesWith(NewVal);
    NewVal->takeName(&FDiv);
    RecursivelyDeleteTriviallyDeadInstructions(&FDiv, TLI);
  }

  return true;
}
```
<a name="block_12"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUCodeGenPrepare.cpp (L1236-1244) [<sup>↩</sup>](#ref-block_12)
```cpp
Value *AMDGPUCodeGenPrepareImpl::expandDivRem24(IRBuilder<> &Builder,
                                                BinaryOperator &I, Value *Num,
                                                Value *Den, bool IsDiv,
                                                bool IsSigned) const {
  unsigned DivBits = getDivNumBits(I, Num, Den, 24, IsSigned);
  if (DivBits > 24)
    return nullptr;
  return expandDivRem24Impl(Builder, I, Num, Den, DivBits, IsDiv, IsSigned);
}
```
<a name="block_13"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUCodeGenPrepare.cpp (L1389-1511)
```cpp
Value *AMDGPUCodeGenPrepareImpl::expandDivRem32(IRBuilder<> &Builder,
                                                BinaryOperator &I, Value *X,
                                                Value *Y) const {
  Instruction::BinaryOps Opc = I.getOpcode();
  assert(Opc == Instruction::URem || Opc == Instruction::UDiv ||
         Opc == Instruction::SRem || Opc == Instruction::SDiv);

  FastMathFlags FMF;
  FMF.setFast();
  Builder.setFastMathFlags(FMF);

  if (divHasSpecialOptimization(I, X, Y))
    return nullptr;  // Keep it for later optimization.

  bool IsDiv = Opc == Instruction::UDiv || Opc == Instruction::SDiv;
  bool IsSigned = Opc == Instruction::SRem || Opc == Instruction::SDiv;

  Type *Ty = X->getType();
  Type *I32Ty = Builder.getInt32Ty();
  Type *F32Ty = Builder.getFloatTy();

  if (Ty->getScalarSizeInBits() != 32) {
    if (IsSigned) {
      X = Builder.CreateSExtOrTrunc(X, I32Ty);
      Y = Builder.CreateSExtOrTrunc(Y, I32Ty);
    } else {
      X = Builder.CreateZExtOrTrunc(X, I32Ty);
      Y = Builder.CreateZExtOrTrunc(Y, I32Ty);
    }
  }

  if (Value *Res = expandDivRem24(Builder, I, X, Y, IsDiv, IsSigned)) {
    return IsSigned ? Builder.CreateSExtOrTrunc(Res, Ty) :
                      Builder.CreateZExtOrTrunc(Res, Ty);
  }

  ConstantInt *Zero = Builder.getInt32(0);
  ConstantInt *One = Builder.getInt32(1);

  Value *Sign = nullptr;
  if (IsSigned) {
    Value *SignX = getSign32(X, Builder, DL);
    Value *SignY = getSign32(Y, Builder, DL);
    // Remainder sign is the same as LHS
    Sign = IsDiv ? Builder.CreateXor(SignX, SignY) : SignX;

    X = Builder.CreateAdd(X, SignX);
    Y = Builder.CreateAdd(Y, SignY);

    X = Builder.CreateXor(X, SignX);
    Y = Builder.CreateXor(Y, SignY);
  }

  // The algorithm here is based on ideas from "Software Integer Division", Tom
  // Rodeheffer, August 2008.
  //
  // unsigned udiv(unsigned x, unsigned y) {
  //   // Initial estimate of inv(y). The constant is less than 2^32 to ensure
  //   // that this is a lower bound on inv(y), even if some of the calculations
  //   // round up.
  //   unsigned z = (unsigned)((4294967296.0 - 512.0) * v_rcp_f32((float)y));
  //
  //   // One round of UNR (Unsigned integer Newton-Raphson) to improve z.
  //   // Empirically this is guaranteed to give a "two-y" lower bound on
  //   // inv(y).
  //   z += umulh(z, -y * z);
  //
  //   // Quotient/remainder estimate.
  //   unsigned q = umulh(x, z);
  //   unsigned r = x - q * y;
  //
  //   // Two rounds of quotient/remainder refinement.
  //   if (r >= y) {
  //     ++q;
  //     r -= y;
  //   }
  //   if (r >= y) {
  //     ++q;
  //     r -= y;
  //   }
  //
  //   return q;
  // }

  // Initial estimate of inv(y).
  Value *FloatY = Builder.CreateUIToFP(Y, F32Ty);
  Value *RcpY = Builder.CreateIntrinsic(Intrinsic::amdgcn_rcp, F32Ty, {FloatY});
  Constant *Scale = ConstantFP::get(F32Ty, llvm::bit_cast<float>(0x4F7FFFFE));
  Value *ScaledY = Builder.CreateFMul(RcpY, Scale);
  Value *Z = Builder.CreateFPToUI(ScaledY, I32Ty);

  // One round of UNR.
  Value *NegY = Builder.CreateSub(Zero, Y);
  Value *NegYZ = Builder.CreateMul(NegY, Z);
  Z = Builder.CreateAdd(Z, getMulHu(Builder, Z, NegYZ));

  // Quotient/remainder estimate.
  Value *Q = getMulHu(Builder, X, Z);
  Value *R = Builder.CreateSub(X, Builder.CreateMul(Q, Y));

  // First quotient/remainder refinement.
  Value *Cond = Builder.CreateICmpUGE(R, Y);
  if (IsDiv)
    Q = Builder.CreateSelect(Cond, Builder.CreateAdd(Q, One), Q);
  R = Builder.CreateSelect(Cond, Builder.CreateSub(R, Y), R);

  // Second quotient/remainder refinement.
  Cond = Builder.CreateICmpUGE(R, Y);
  Value *Res;
  if (IsDiv)
    Res = Builder.CreateSelect(Cond, Builder.CreateAdd(Q, One), Q);
  else
    Res = Builder.CreateSelect(Cond, Builder.CreateSub(R, Y), R);

  if (IsSigned) {
    Res = Builder.CreateXor(Res, Sign);
    Res = Builder.CreateSub(Res, Sign);
    Res = Builder.CreateSExtOrTrunc(Res, Ty);
  } else {
    Res = Builder.CreateZExtOrTrunc(Res, Ty);
  }
  return Res;
}
```
<a name="block_14"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUCodeGenPrepare.cpp (L1513-1542)
```cpp
Value *AMDGPUCodeGenPrepareImpl::shrinkDivRem64(IRBuilder<> &Builder,
                                                BinaryOperator &I, Value *Num,
                                                Value *Den) const {
  if (!ExpandDiv64InIR && divHasSpecialOptimization(I, Num, Den))
    return nullptr;  // Keep it for later optimization.

  Instruction::BinaryOps Opc = I.getOpcode();

  bool IsDiv = Opc == Instruction::SDiv || Opc == Instruction::UDiv;
  bool IsSigned = Opc == Instruction::SDiv || Opc == Instruction::SRem;

  unsigned NumDivBits = getDivNumBits(I, Num, Den, 32, IsSigned);
  if (NumDivBits > 32)
    return nullptr;

  Value *Narrowed = nullptr;
  if (NumDivBits <= 24) {
    Narrowed = expandDivRem24Impl(Builder, I, Num, Den, NumDivBits,
                                  IsDiv, IsSigned);
  } else if (NumDivBits <= 32) {
    Narrowed = expandDivRem32(Builder, I, Num, Den);
  }

  if (Narrowed) {
    return IsSigned ? Builder.CreateSExt(Narrowed, Num->getType()) :
                      Builder.CreateZExt(Narrowed, Num->getType());
  }

  return nullptr;
}
```
<a name="block_15"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUCodeGenPrepare.cpp (L1544-1558)
```cpp
void AMDGPUCodeGenPrepareImpl::expandDivRem64(BinaryOperator &I) const {
  Instruction::BinaryOps Opc = I.getOpcode();
  // Do the general expansion.
  if (Opc == Instruction::UDiv || Opc == Instruction::SDiv) {
    expandDivisionUpTo64Bits(&I);
    return;
  }

  if (Opc == Instruction::URem || Opc == Instruction::SRem) {
    expandRemainderUpTo64Bits(&I);
    return;
  }

  llvm_unreachable("not a division");
}
```
<a name="block_16"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUCodeGenPrepare.cpp (L1572-1632) [<sup>↩</sup>](#ref-block_16)
```cpp
static bool tryNarrowMathIfNoOverflow(Instruction *I,
                                      const SITargetLowering *TLI,
                                      const TargetTransformInfo &TTI,
                                      const DataLayout &DL) {
  unsigned Opc = I->getOpcode();
  Type *OldType = I->getType();

  if (Opc != Instruction::Add && Opc != Instruction::Mul)
    return false;

  unsigned OrigBit = OldType->getScalarSizeInBits();

  if (Opc != Instruction::Add && Opc != Instruction::Mul)
    llvm_unreachable("Unexpected opcode, only valid for Instruction::Add and "
                     "Instruction::Mul.");

  unsigned MaxBitsNeeded = computeKnownBits(I, DL).countMaxActiveBits();

  MaxBitsNeeded = std::max<unsigned>(bit_ceil(MaxBitsNeeded), 8);
  Type *NewType = DL.getSmallestLegalIntType(I->getContext(), MaxBitsNeeded);
  if (!NewType)
    return false;
  unsigned NewBit = NewType->getIntegerBitWidth();
  if (NewBit >= OrigBit)
    return false;
  NewType = I->getType()->getWithNewBitWidth(NewBit);

  // Old cost
  InstructionCost OldCost =
      TTI.getArithmeticInstrCost(Opc, OldType, TTI::TCK_RecipThroughput);
  // New cost of new op
  InstructionCost NewCost =
      TTI.getArithmeticInstrCost(Opc, NewType, TTI::TCK_RecipThroughput);
  // New cost of narrowing 2 operands (use trunc)
  int NumOfNonConstOps = 2;
  if (isa<Constant>(I->getOperand(0)) || isa<Constant>(I->getOperand(1))) {
    // Cannot be both constant, should be propagated
    NumOfNonConstOps = 1;
  }
  NewCost += NumOfNonConstOps * TTI.getCastInstrCost(Instruction::Trunc,
                                                     NewType, OldType,
                                                     TTI.getCastContextHint(I),
                                                     TTI::TCK_RecipThroughput);
  // New cost of zext narrowed result to original type
  NewCost +=
      TTI.getCastInstrCost(Instruction::ZExt, OldType, NewType,
                           TTI.getCastContextHint(I), TTI::TCK_RecipThroughput);
  if (NewCost >= OldCost)
    return false;

  IRBuilder<> Builder(I);
  Value *Trunc0 = Builder.CreateTrunc(I->getOperand(0), NewType);
  Value *Trunc1 = Builder.CreateTrunc(I->getOperand(1), NewType);
  Value *Arith =
      Builder.CreateBinOp((Instruction::BinaryOps)Opc, Trunc0, Trunc1);

  Value *Zext = Builder.CreateZExt(Arith, OldType);
  I->replaceAllUsesWith(Zext);
  I->eraseFromParent();
  return true;
}
```
<a name="block_17"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUCodeGenPrepare.cpp (L1634-1725) [<sup>↩</sup>](#ref-block_17)
```cpp
bool AMDGPUCodeGenPrepareImpl::visitBinaryOperator(BinaryOperator &I) {
  if (foldBinOpIntoSelect(I))
    return true;

  if (ST.has16BitInsts() && needsPromotionToI32(I.getType()) &&
      UA.isUniform(&I) && promoteUniformOpToI32(I))
    return true;

  if (UseMul24Intrin && replaceMulWithMul24(I))
    return true;
  if (tryNarrowMathIfNoOverflow(&I, ST.getTargetLowering(),
                                TM.getTargetTransformInfo(F), DL))
    return true;

  bool Changed = false;
  Instruction::BinaryOps Opc = I.getOpcode();
  Type *Ty = I.getType();
  Value *NewDiv = nullptr;
  unsigned ScalarSize = Ty->getScalarSizeInBits();

  SmallVector<BinaryOperator *, 8> Div64ToExpand;

  if ((Opc == Instruction::URem || Opc == Instruction::UDiv ||
       Opc == Instruction::SRem || Opc == Instruction::SDiv) &&
      ScalarSize <= 64 &&
      !DisableIDivExpand) {
    Value *Num = I.getOperand(0);
    Value *Den = I.getOperand(1);
    IRBuilder<> Builder(&I);
    Builder.SetCurrentDebugLocation(I.getDebugLoc());

    if (auto *VT = dyn_cast<FixedVectorType>(Ty)) {
      NewDiv = PoisonValue::get(VT);

      for (unsigned N = 0, E = VT->getNumElements(); N != E; ++N) {
        Value *NumEltN = Builder.CreateExtractElement(Num, N);
        Value *DenEltN = Builder.CreateExtractElement(Den, N);

        Value *NewElt;
        if (ScalarSize <= 32) {
          NewElt = expandDivRem32(Builder, I, NumEltN, DenEltN);
          if (!NewElt)
            NewElt = Builder.CreateBinOp(Opc, NumEltN, DenEltN);
        } else {
          // See if this 64-bit division can be shrunk to 32/24-bits before
          // producing the general expansion.
          NewElt = shrinkDivRem64(Builder, I, NumEltN, DenEltN);
          if (!NewElt) {
            // The general 64-bit expansion introduces control flow and doesn't
            // return the new value. Just insert a scalar copy and defer
            // expanding it.
            NewElt = Builder.CreateBinOp(Opc, NumEltN, DenEltN);
            // CreateBinOp does constant folding. If the operands are constant,
            // it will return a Constant instead of a BinaryOperator.
            if (auto *NewEltBO = dyn_cast<BinaryOperator>(NewElt))
              Div64ToExpand.push_back(NewEltBO);
          }
        }

        if (auto *NewEltI = dyn_cast<Instruction>(NewElt))
          NewEltI->copyIRFlags(&I);

        NewDiv = Builder.CreateInsertElement(NewDiv, NewElt, N);
      }
    } else {
      if (ScalarSize <= 32)
        NewDiv = expandDivRem32(Builder, I, Num, Den);
      else {
        NewDiv = shrinkDivRem64(Builder, I, Num, Den);
        if (!NewDiv)
          Div64ToExpand.push_back(&I);
      }
    }

    if (NewDiv) {
      I.replaceAllUsesWith(NewDiv);
      I.eraseFromParent();
      Changed = true;
    }
  }

  if (ExpandDiv64InIR) {
    // TODO: We get much worse code in specially handled constant cases.
    for (BinaryOperator *Div : Div64ToExpand) {
      expandDivRem64(*Div);
      FlowChanged = true;
      Changed = true;
    }
  }

  return Changed;
}
```
<a name="block_18"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUCodeGenPrepare.cpp (L1727-1771) [<sup>↩</sup>](#ref-block_18)
```cpp
bool AMDGPUCodeGenPrepareImpl::visitLoadInst(LoadInst &I) {
  if (!WidenLoads)
    return false;

  if ((I.getPointerAddressSpace() == AMDGPUAS::CONSTANT_ADDRESS ||
       I.getPointerAddressSpace() == AMDGPUAS::CONSTANT_ADDRESS_32BIT) &&
      canWidenScalarExtLoad(I)) {
    IRBuilder<> Builder(&I);
    Builder.SetCurrentDebugLocation(I.getDebugLoc());

    Type *I32Ty = Builder.getInt32Ty();
    LoadInst *WidenLoad = Builder.CreateLoad(I32Ty, I.getPointerOperand());
    WidenLoad->copyMetadata(I);

    // If we have range metadata, we need to convert the type, and not make
    // assumptions about the high bits.
    if (auto *Range = WidenLoad->getMetadata(LLVMContext::MD_range)) {
      ConstantInt *Lower =
        mdconst::extract<ConstantInt>(Range->getOperand(0));

      if (Lower->isNullValue()) {
        WidenLoad->setMetadata(LLVMContext::MD_range, nullptr);
      } else {
        Metadata *LowAndHigh[] = {
          ConstantAsMetadata::get(ConstantInt::get(I32Ty, Lower->getValue().zext(32))),
          // Don't make assumptions about the high bits.
          ConstantAsMetadata::get(ConstantInt::get(I32Ty, 0))
        };

        WidenLoad->setMetadata(LLVMContext::MD_range,
                               MDNode::get(F.getContext(), LowAndHigh));
      }
    }

    int TySize = DL.getTypeSizeInBits(I.getType());
    Type *IntNTy = Builder.getIntNTy(TySize);
    Value *ValTrunc = Builder.CreateTrunc(WidenLoad, IntNTy);
    Value *ValOrig = Builder.CreateBitCast(ValTrunc, I.getType());
    I.replaceAllUsesWith(ValOrig);
    I.eraseFromParent();
    return true;
  }

  return false;
}
```
<a name="block_19"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUCodeGenPrepare.cpp (L2033-2114) [<sup>↩</sup>](#ref-block_19)
```cpp
bool AMDGPUCodeGenPrepareImpl::visitPHINode(PHINode &I) {
  // Break-up fixed-vector PHIs into smaller pieces.
  // Default threshold is 32, so it breaks up any vector that's >32 bits into
  // its elements, or into 32-bit pieces (for 8/16 bit elts).
  //
  // This is only helpful for DAGISel because it doesn't handle large PHIs as
  // well as GlobalISel. DAGISel lowers PHIs by using CopyToReg/CopyFromReg.
  // With large, odd-sized PHIs we may end up needing many `build_vector`
  // operations with most elements being "undef". This inhibits a lot of
  // optimization opportunities and can result in unreasonably high register
  // pressure and the inevitable stack spilling.
  if (!BreakLargePHIs || getCGPassBuilderOption().EnableGlobalISelOption)
    return false;

  FixedVectorType *FVT = dyn_cast<FixedVectorType>(I.getType());
  if (!FVT || FVT->getNumElements() == 1 ||
      DL.getTypeSizeInBits(FVT) <= BreakLargePHIsThreshold)
    return false;

  if (!ForceBreakLargePHIs && !canBreakPHINode(I))
    return false;

  std::vector<VectorSlice> Slices;

  Type *EltTy = FVT->getElementType();
  {
    unsigned Idx = 0;
    // For 8/16 bits type, don't scalarize fully but break it up into as many
    // 32-bit slices as we can, and scalarize the tail.
    const unsigned EltSize = DL.getTypeSizeInBits(EltTy);
    const unsigned NumElts = FVT->getNumElements();
    if (EltSize == 8 || EltSize == 16) {
      const unsigned SubVecSize = (32 / EltSize);
      Type *SubVecTy = FixedVectorType::get(EltTy, SubVecSize);
      for (unsigned End = alignDown(NumElts, SubVecSize); Idx < End;
           Idx += SubVecSize)
        Slices.emplace_back(SubVecTy, Idx, SubVecSize);
    }

    // Scalarize all remaining elements.
    for (; Idx < NumElts; ++Idx)
      Slices.emplace_back(EltTy, Idx, 1);
  }

  assert(Slices.size() > 1);

  // Create one PHI per vector piece. The "VectorSlice" class takes care of
  // creating the necessary instruction to extract the relevant slices of each
  // incoming value.
  IRBuilder<> B(I.getParent());
  B.SetCurrentDebugLocation(I.getDebugLoc());

  unsigned IncNameSuffix = 0;
  for (VectorSlice &S : Slices) {
    // We need to reset the build on each iteration, because getSlicedVal may
    // have inserted something into I's BB.
    B.SetInsertPoint(I.getParent()->getFirstNonPHIIt());
    S.NewPHI = B.CreatePHI(S.Ty, I.getNumIncomingValues());

    for (const auto &[Idx, BB] : enumerate(I.blocks())) {
      S.NewPHI->addIncoming(S.getSlicedVal(BB, I.getIncomingValue(Idx),
                                           "largephi.extractslice" +
                                               std::to_string(IncNameSuffix++)),
                            BB);
    }
  }

  // And replace this PHI with a vector of all the previous PHI values.
  Value *Vec = PoisonValue::get(FVT);
  unsigned NameSuffix = 0;
  for (VectorSlice &S : Slices) {
    const auto ValName = "largephi.insertslice" + std::to_string(NameSuffix++);
    if (S.NumElts > 1)
      Vec = B.CreateInsertVector(FVT, Vec, S.NewPHI, S.Idx, ValName);
    else
      Vec = B.CreateInsertElement(Vec, S.NewPHI, S.Idx, ValName);
  }

  I.replaceAllUsesWith(Vec);
  I.eraseFromParent();
  return true;
}
```
<a name="block_20"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUCodeGenPrepare.cpp (L2160-2195) [<sup>↩</sup>](#ref-block_20)
```cpp
bool AMDGPUCodeGenPrepareImpl::visitAddrSpaceCastInst(AddrSpaceCastInst &I) {
  // Intrinsic doesn't support vectors, also it seems that it's often difficult
  // to prove that a vector cannot have any nulls in it so it's unclear if it's
  // worth supporting.
  if (I.getType()->isVectorTy())
    return false;

  // Check if this can be lowered to a amdgcn.addrspacecast.nonnull.
  // This is only worthwhile for casts from/to priv/local to flat.
  const unsigned SrcAS = I.getSrcAddressSpace();
  const unsigned DstAS = I.getDestAddressSpace();

  bool CanLower = false;
  if (SrcAS == AMDGPUAS::FLAT_ADDRESS)
    CanLower = (DstAS == AMDGPUAS::LOCAL_ADDRESS ||
                DstAS == AMDGPUAS::PRIVATE_ADDRESS);
  else if (DstAS == AMDGPUAS::FLAT_ADDRESS)
    CanLower = (SrcAS == AMDGPUAS::LOCAL_ADDRESS ||
                SrcAS == AMDGPUAS::PRIVATE_ADDRESS);
  if (!CanLower)
    return false;

  SmallVector<const Value *, 4> WorkList;
  getUnderlyingObjects(I.getOperand(0), WorkList);
  if (!all_of(WorkList, [&](const Value *V) {
        return isPtrKnownNeverNull(V, DL, TM, SrcAS);
      }))
    return false;

  IRBuilder<> B(&I);
  auto *Intrin = B.CreateIntrinsic(
      I.getType(), Intrinsic::amdgcn_addrspacecast_nonnull, {I.getOperand(0)});
  I.replaceAllUsesWith(Intrin);
  I.eraseFromParent();
  return true;
}
```
<a name="block_21"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUCodeGenPrepare.cpp (L2229-2266) [<sup>↩</sup>](#ref-block_21)
```cpp
Value *AMDGPUCodeGenPrepareImpl::matchFractPat(IntrinsicInst &I) {
  if (ST.hasFractBug())
    return nullptr;

  Intrinsic::ID IID = I.getIntrinsicID();

  // The value is only used in contexts where we know the input isn't a nan, so
  // any of the fmin variants are fine.
  if (IID != Intrinsic::minnum && IID != Intrinsic::minimum &&
      IID != Intrinsic::minimumnum)
    return nullptr;

  Type *Ty = I.getType();
  if (!isLegalFloatingTy(Ty->getScalarType()))
    return nullptr;

  Value *Arg0 = I.getArgOperand(0);
  Value *Arg1 = I.getArgOperand(1);

  const APFloat *C;
  if (!match(Arg1, m_APFloat(C)))
    return nullptr;

  APFloat One(1.0);
  bool LosesInfo;
  One.convert(C->getSemantics(), APFloat::rmNearestTiesToEven, &LosesInfo);

  // Match nextafter(1.0, -1)
  One.next(true);
  if (One != *C)
    return nullptr;

  Value *FloorSrc;
  if (match(Arg0, m_FSub(m_Value(FloorSrc),
                         m_Intrinsic<Intrinsic::floor>(m_Deferred(FloorSrc)))))
    return FloorSrc;
  return nullptr;
}
```
<a name="block_22"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUCodeGenPrepare.cpp (L2284-2305)
```cpp
bool AMDGPUCodeGenPrepareImpl::visitFMinLike(IntrinsicInst &I) {
  Value *FractArg = matchFractPat(I);
  if (!FractArg)
    return false;

  // Match pattern for fract intrinsic in contexts where the nan check has been
  // optimized out (and hope the knowledge the source can't be nan wasn't lost).
  if (!I.hasNoNaNs() && !isKnownNeverNaN(FractArg, SimplifyQuery(DL, TLI)))
    return false;

  IRBuilder<> Builder(&I);
  FastMathFlags FMF = I.getFastMathFlags();
  FMF.setNoNaNs();
  Builder.setFastMathFlags(FMF);

  Value *Fract = applyFractPat(Builder, FractArg);
  Fract->takeName(&I);
  I.replaceAllUsesWith(Fract);

  RecursivelyDeleteTriviallyDeadInstructions(&I, TLI);
  return true;
}
```
<a name="block_23"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUCodeGenPrepare.cpp (L2313-2369) [<sup>↩</sup>](#ref-block_23)
```cpp
bool AMDGPUCodeGenPrepareImpl::visitSqrt(IntrinsicInst &Sqrt) {
  Type *Ty = Sqrt.getType()->getScalarType();
  if (!Ty->isFloatTy() && (!Ty->isHalfTy() || ST.has16BitInsts()))
    return false;

  const FPMathOperator *FPOp = cast<const FPMathOperator>(&Sqrt);
  FastMathFlags SqrtFMF = FPOp->getFastMathFlags();

  // We're trying to handle the fast-but-not-that-fast case only. The lowering
  // of fast llvm.sqrt will give the raw instruction anyway.
  if (SqrtFMF.approxFunc() || HasUnsafeFPMath)
    return false;

  const float ReqdAccuracy = FPOp->getFPAccuracy();

  // Defer correctly rounded expansion to codegen.
  if (ReqdAccuracy < 1.0f)
    return false;

  // FIXME: This is an ugly hack for this pass using forward iteration instead
  // of reverse. If it worked like a normal combiner, the rsq would form before
  // we saw a sqrt call.
  auto *FDiv =
      dyn_cast_or_null<FPMathOperator>(Sqrt.getUniqueUndroppableUser());
  if (FDiv && FDiv->getOpcode() == Instruction::FDiv &&
      FDiv->getFPAccuracy() >= 1.0f &&
      canOptimizeWithRsq(FPOp, FDiv->getFastMathFlags(), SqrtFMF) &&
      // TODO: We should also handle the arcp case for the fdiv with non-1 value
      isOneOrNegOne(FDiv->getOperand(0)))
    return false;

  Value *SrcVal = Sqrt.getOperand(0);
  bool CanTreatAsDAZ = canIgnoreDenormalInput(SrcVal, &Sqrt);

  // The raw instruction is 1 ulp, but the correction for denormal handling
  // brings it to 2.
  if (!CanTreatAsDAZ && ReqdAccuracy < 2.0f)
    return false;

  IRBuilder<> Builder(&Sqrt);
  SmallVector<Value *, 4> SrcVals;
  extractValues(Builder, SrcVals, SrcVal);

  SmallVector<Value *, 4> ResultVals(SrcVals.size());
  for (int I = 0, E = SrcVals.size(); I != E; ++I) {
    if (CanTreatAsDAZ)
      ResultVals[I] = Builder.CreateCall(getSqrtF32(), SrcVals[I]);
    else
      ResultVals[I] = emitSqrtIEEE2ULP(Builder, SrcVals[I], SqrtFMF);
  }

  Value *NewSqrt = insertValues(Builder, Sqrt.getType(), ResultVals);
  NewSqrt->takeName(&Sqrt);
  Sqrt.replaceAllUsesWith(NewSqrt);
  Sqrt.eraseFromParent();
  return true;
}
```
<a name="block_24"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUCodeGenPrepare.cpp (L2371-2389) [<sup>↩</sup>](#ref-block_24)
```cpp
bool AMDGPUCodeGenPrepare::runOnFunction(Function &F) {
  if (skipFunction(F))
    return false;

  auto *TPC = getAnalysisIfAvailable<TargetPassConfig>();
  if (!TPC)
    return false;

  const AMDGPUTargetMachine &TM = TPC->getTM<AMDGPUTargetMachine>();
  const TargetLibraryInfo *TLI =
      &getAnalysis<TargetLibraryInfoWrapperPass>().getTLI(F);
  AssumptionCache *AC =
      &getAnalysis<AssumptionCacheTracker>().getAssumptionCache(F);
  auto *DTWP = getAnalysisIfAvailable<DominatorTreeWrapperPass>();
  const DominatorTree *DT = DTWP ? &DTWP->getDomTree() : nullptr;
  const UniformityInfo &UA =
      getAnalysis<UniformityInfoWrapperPass>().getUniformityInfo();
  return AMDGPUCodeGenPrepareImpl(F, TM, TLI, AC, DT, UA).run();
}
```