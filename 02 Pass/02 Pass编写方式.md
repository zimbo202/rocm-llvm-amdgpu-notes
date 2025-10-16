# LLVM Pass的编写方式

## Pass的基本定义和结构

<a name="ref-block_0"></a>这个Pass采用新Pass管理器的编写模式，定义为一个继承自`PassInfoMixin`的结构体： llvm-project:448-455[<sup>↗</sup>](#block_0) 

## 输入输出

### 输入
- **操作对象**: `Module &M` - 整个LLVM模块
- **分析管理器**: `ModuleAnalysisManager &MAM` - 用于获取分析结果
- **目标机器**: 通过构造函数传入的`AMDGPUTargetMachine &TM`

### 输出  
<a name="ref-block_6"></a>- **返回值**: `PreservedAnalyses` - 表示哪些分析结果被保留 llvm-project:130-133[<sup>↗</sup>](#block_6) 

## 操作对象

Pass主要操作以下对象：

1. **全局变量**: 查找以`llvm.amdgcn.`开头的特征谓词全局变量
2. **指令**: 通过常量折叠处理使用这些全局变量的指令
<a name="ref-block_7"></a>3. **函数**: 清理优化后产生的无用代码块 llvm-project:135-144[<sup>↗</sup>](#block_7) 

## 主要操作方法

### 1. 收集处理目标
首先扫描模块中的全局变量，找到需要处理的特征谓词： llvm-project:135-141 

### 2. 设置谓词值
<a name="ref-block_2"></a>根据目标架构的特性，将抽象的特征谓词转换为具体的布尔值： llvm-project:50-74[<sup>↗</sup>](#block_2) 

### 3. 常量折叠
<a name="ref-block_5"></a>对使用特征谓词的指令进行常量折叠，将运行时检查转换为编译时常量： llvm-project:111-124[<sup>↗</sup>](#block_5) 

### 4. 清理和错误处理
<a name="ref-block_8"></a>删除处理完的全局变量，清理无用代码块，对无法折叠的情况报告错误： llvm-project:161-166[<sup>↗</sup>](#block_8) 

## 共有的编写模式

### 1. Pass结构模式
- 继承自`PassInfoMixin<PassType>`
- 实现`run()`方法作为入口点
- 使用`PreservedAnalyses`返回类型

### 2. 工具函数模式
<a name="ref-block_1"></a>使用辅助函数来组织代码逻辑，如用于收集用户的模板函数： llvm-project:42-48[<sup>↗</sup>](#block_1) 

### 3. 错误处理模式
<a name="ref-block_3"></a>对于无法处理的情况，使用诊断接口报告详细错误信息： llvm-project:76-88[<sup>↗</sup>](#block_3) 

### 4. 分析管理器使用模式
<a name="ref-block_4"></a>通过分析管理器获取需要的分析结果，如支配树分析： llvm-project:108-109[<sup>↗</sup>](#block_4) 

### 5. 保守的分析保留策略
根据Pass的修改程度返回适当的`PreservedAnalyses`值，确保后续Pass的正确性。

## Notes

这个Pass体现了现代LLVM Pass编写的几个关键特点：使用新Pass管理器框架、模块化的函数组织、完善的错误处理机制，以及与目标特定信息的紧密集成。它是一个典型的目标特定优化Pass，展示了如何将前端生成的抽象代码转换为针对具体硬件优化的代码。


### Citations


<a name="block_0"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPU.h (L448-455) [<sup>↩</sup>](#ref-block_0)

```text
struct AMDGPUExpandFeaturePredicatesPass
    : PassInfoMixin<AMDGPUExpandFeaturePredicatesPass> {
  const AMDGPUTargetMachine &TM;
  AMDGPUExpandFeaturePredicatesPass(const AMDGPUTargetMachine &ATM) : TM(ATM) {}
  PreservedAnalyses run(Module &M, ModuleAnalysisManager &AM);

  static bool isRequired() { return true; }
};
```

<a name="block_1"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUExpandFeaturePredicates.cpp (L42-48) [<sup>↩</sup>](#ref-block_1)

```cpp
template <typename C> void collectUsers(Value *V, C &Container) {
  assert(V && "Must pass an existing Value!");

  for (auto &&U : V->users())
    if (auto *I = dyn_cast<Instruction>(U))
      Container.insert(Container.end(), I);
}
```

<a name="block_2"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUExpandFeaturePredicates.cpp (L50-74) [<sup>↩</sup>](#ref-block_2)

```cpp
inline void setPredicate(const GCNSubtarget &ST, GlobalVariable *P) {
  const bool IsFeature = P->getName().starts_with("llvm.amdgcn.has");
  const size_t Offset =
      IsFeature ? sizeof("llvm.amdgcn.has") : sizeof("llvm.amdgcn.is");

  std::string PV = P->getName().substr(Offset).str();
  if (IsFeature) {
    size_t Dx = PV.find(',');
    while (Dx != std::string::npos) {
      PV.insert(++Dx, {'+'});

      Dx = PV.find(',', Dx);
    }
    PV.insert(PV.cbegin(), '+');
  }

  Type *PTy = P->getValueType();
  P->setLinkage(GlobalValue::PrivateLinkage);
  P->setExternallyInitialized(false);

  if (IsFeature)
    P->setInitializer(ConstantInt::getBool(PTy, ST.checkFeatures(PV)));
  else
    P->setInitializer(ConstantInt::getBool(PTy, PV == ST.getCPU()));
}
```

<a name="block_3"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUExpandFeaturePredicates.cpp (L76-88) [<sup>↩</sup>](#ref-block_3)

```cpp
std::pair<PreservedAnalyses, bool>
unfoldableFound(Function *Caller, GlobalVariable *P, Instruction *NoFold) {
  std::string W;
  raw_string_ostream OS(W);

  OS << "Impossible to constant fold feature predicate: " << *P << " used by "
     << *NoFold << ", please simplify.\n";

  Caller->getContext().diagnose(
      DiagnosticInfoUnsupported(*Caller, W, NoFold->getDebugLoc(), DS_Error));

  return {PreservedAnalyses::none(), false};
}
```

<a name="block_4"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUExpandFeaturePredicates.cpp (L108-109) [<sup>↩</sup>](#ref-block_4)

```cpp
    auto &DT = FAM.getResult<DominatorTreeAnalysis>(*F);
    DomTreeUpdater DTU(DT, DomTreeUpdater::UpdateStrategy::Lazy);
```

<a name="block_5"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUExpandFeaturePredicates.cpp (L111-124) [<sup>↩</sup>](#ref-block_5)

```cpp
    if (auto *C = ConstantFoldInstruction(I, P->getDataLayout())) {
      collectUsers(I, ToFold);
      I->replaceAllUsesWith(C);
      I->eraseFromParent();
      continue;
    } else if (I->isTerminator() &&
               ConstantFoldTerminator(I->getParent(), true, nullptr, &DTU)) {
        Predicated.insert(F);

        continue;
    }

    return unfoldableFound(I->getParent()->getParent(), P, I);
  } while (!ToFold.empty());
```

<a name="block_6"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUExpandFeaturePredicates.cpp (L130-133) [<sup>↩</sup>](#ref-block_6)

```cpp
PreservedAnalyses
AMDGPUExpandFeaturePredicatesPass::run(Module &M, ModuleAnalysisManager &MAM) {
  if (M.empty())
    return PreservedAnalyses::all();
```

<a name="block_7"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUExpandFeaturePredicates.cpp (L135-144) [<sup>↩</sup>](#ref-block_7)

```cpp
  SmallVector<GlobalVariable *> Predicates;
  for (auto &&G : M.globals()) {
    if (!G.isDeclaration() || !G.hasName())
      continue;
    if (G.getName().starts_with("llvm.amdgcn."))
      Predicates.push_back(&G);
  }

  if (Predicates.empty())
    return PreservedAnalyses::all();
```

<a name="block_8"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUExpandFeaturePredicates.cpp (L161-166) [<sup>↩</sup>](#ref-block_8)

```cpp
  for (auto &&P : Predicates)
    P->eraseFromParent();
  for (auto &&F : Predicated)
    removeUnreachableBlocks(*F);

  return Ret;
```

