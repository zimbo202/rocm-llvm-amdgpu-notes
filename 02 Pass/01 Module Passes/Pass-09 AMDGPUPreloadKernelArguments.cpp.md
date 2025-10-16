# AMDGPUPreloadKernelArguments.cpp 代码功能分析

## 1. Pass 主要功能概括

<a name="ref-block_0"></a>该 pass 的主要功能是**将 kernel 参数预加载到 user_data SGPR（Scalar General Purpose Registers）寄存器中**，以在 kernel 执行开始前提前准备好参数数据，从而提升执行性能。 llvm-project:9-17[<sup>↗</sup>](#block_0) 

**作用与效果：**
- 将 kernel 的显式参数和隐藏参数预加载到 SGPR 中
- 预加载的参数数量取决于可用的 free user SGPR 数量，受硬件最大限制
- 预加载的参数会被标记为 `inreg` 属性
- 隐藏参数（如果被预加载）会被追加到 kernel 签名的显式参数之后

## 2. 主要功能的步骤和子功能

该 pass 的实现包含以下核心步骤/子功能：

### 步骤 1: 主入口函数 `markKernelArgsAsInreg`
<a name="ref-block_8"></a>这是整个 pass 的核心处理函数。 llvm-project:282-351[<sup>↗</sup>](#block_8) 

### 步骤 2: 初始化预加载信息类 `PreloadKernelArgInfo`
<a name="ref-block_2"></a>用于管理和跟踪 kernel 参数预加载的相关信息。 llvm-project:62-79[<sup>↗</sup>](#block_2) 

### 步骤 3: 计算可用 SGPR 数量 `setInitialFreeUserSGPRsCount`
<a name="ref-block_5"></a>确定有多少 user SGPR 可用于参数预加载。 llvm-project:176-179[<sup>↗</sup>](#block_5) 

### 步骤 4: 检查预加载可行性 `canPreloadKernArgAtOffset`
<a name="ref-block_6"></a>判断在特定偏移量处的参数是否可以预加载。 llvm-project:181-183[<sup>↗</sup>](#block_6) 

### 步骤 5: 预加载显式参数
遍历函数的显式参数，为符合条件的参数添加 `InReg` 属性。 llvm-project:298-329 

### 步骤 6: 尝试预加载隐藏参数 `tryAllocHiddenArgPreloadSGPRs`
<a name="ref-block_7"></a>处理隐藏的 kernel 参数（如 block count、group size 等）的预加载。 llvm-project:186-262[<sup>↗</sup>](#block_7) 

### 步骤 7: 克隆函数 `cloneFunctionWithPreloadImplicitArgs`
<a name="ref-block_4"></a>当需要预加载隐藏参数时，创建新的函数签名并克隆函数。 llvm-project:128-167[<sup>↗</sup>](#block_4) 

## 3. 各步骤的具体描述分析

### 3.1 主入口函数 `markKernelArgsAsInreg`

**功能描述：**
- 检查是否启用了 kernarg preload 功能
- 遍历模块中的所有函数
- 筛选出满足条件的 AMDGPU kernel（具有 kernarg preload 支持且调用约定为 AMDGPU_KERNEL）
- 对每个符合条件的 kernel 执行参数预加载处理
- 清理需要删除的旧函数（因签名变化而被克隆替换的函数）

**关键检查点：**
- `EnableKernargPreload` 标志必须为 true
- Subtarget 必须支持 `hasKernargPreload()`
- 函数的调用约定必须是 `CallingConv::AMDGPU_KERNEL`

### 3.2 `PreloadKernelArgInfo` 类

**功能描述：**
这是一个辅助类，封装了预加载参数所需的所有信息和操作：

**包含的数据成员：**
- `Function &F`: 要处理的函数引用
- `const GCNSubtarget &ST`: 目标平台的 subtarget 信息
- `NumFreeUserSGPRs`: 可用的 free user SGPR 数量

**隐藏参数定义：**
<a name="ref-block_3"></a>定义了 9 种隐藏参数类型（block count x/y/z, group size x/y/z, remainder x/y/z），每个都有对应的偏移量、大小和名称。 llvm-project:92-97[<sup>↗</sup>](#block_3) 

### 3.3 计算可用 SGPR 数量

**功能描述：**
调用 `GCNUserSGPRUsageInfo` 来计算当前函数还有多少 user SGPR 可用于预加载参数。这个数量会受到已被其他用途占用的 SGPR 的影响（如 implicit 参数占用的寄存器）。

### 3.4 检查预加载可行性

**功能描述：**
根据参数的偏移量判断是否有足够的 SGPR 来预加载该参数。计算公式为：参数偏移量（字节）≤ 可用 SGPR 数量 × 4（每个 SGPR 4 字节）。

### 3.5 预加载显式参数处理

**功能描述：**
这是处理显式（用户定义的）kernel 参数的核心逻辑：

**处理流程：**
1. 遍历函数的每个参数
2. 检查参数兼容性（跳过 byref、nest、已标记为 hidden 的参数）
3. 检查是否还需要预加载更多参数（基于 `KernargPreloadCount`）
4. 跳过聚合类型（aggregate types）
5. 计算参数的对齐和大小，累计偏移量
6. 检查是否有足够的 SGPR 预加载该参数
7. 为参数添加 `InReg` 属性
8. 更新预加载计数

**重要约束：**
- 不支持 byref 参数
- 不支持聚合类型
- 按参数顺序连续预加载，不能跳过中间的参数

### 3.6 预加载隐藏参数处理

**功能描述：**
处理隐藏的 kernel 参数预加载，这些参数来自 implicit argument pointer：

**处理流程：**
1. 查找 `amdgcn_implicitarg_ptr` intrinsic 的使用
2. 识别从 implicit arg pointer 的加载指令
3. 匹配加载的偏移量和类型是否对应已知的隐藏参数
4. 按偏移量排序所有可预加载的隐藏参数 load
5. 找到第一个无法预加载的参数位置（SGPR 不足）
6. 对可以预加载的隐藏参数，克隆函数并修改签名
7. 用新的函数参数替换原来的 load 指令

**关键机制：**
隐藏参数通过模式匹配识别，必须满足：
- 加载来自正确的偏移量
- 加载的类型与预期的隐藏参数类型匹配
- 有足够的 SGPR 空间

### 3.7 函数克隆

**功能描述：**
当需要预加载隐藏参数时，必须修改函数签名来接收这些额外的参数：

**克隆流程：**
1. 构建新的参数类型列表（原参数 + 隐藏参数）
2. 创建新的函数类型和函数对象
3. 复制原函数的属性和元数据
4. 将新函数插入到模块中
5. 转移函数体（basic blocks）到新函数
6. 重新映射原参数到新函数的参数
7. 为新添加的隐藏参数设置属性（`InReg` 和 `amdgpu-hidden-argument`）
8. 用新函数替换所有对旧函数的引用

## 4. 步骤/子功能之间的关系

整个 pass 的执行流程呈现**层次化的调用关系**：

```
runOnModule / run (Pass 入口)
    └─> markKernelArgsAsInreg (主处理函数)
         ├─> 遍历每个 kernel 函数
         │    ├─> PreloadKernelArgInfo 构造
         │    │    └─> setInitialFreeUserSGPRsCount (初始化)
         │    │
         │    ├─> 预加载显式参数循环
         │    │    └─> canPreloadKernArgAtOffset (可行性检查)
         │    │
         │    └─> tryAllocHiddenArgPreloadSGPRs (预加载隐藏参数)
         │         ├─> 识别和收集隐藏参数 loads
         │         ├─> 排序并检查可行性 (canPreloadKernArgAtOffset)
         │         └─> cloneFunctionWithPreloadImplicitArgs (修改签名)
         │
         └─> 清理被克隆替换的旧函数
```

**关键依赖关系：**

1. **顺序依赖**: 必须先处理显式参数，只有当所有显式参数都能成功预加载时，才尝试预加载隐藏参数。 llvm-project:331-339 

2. **资源约束**: `canPreloadKernArgAtOffset` 被多处调用，作为统一的资源检查点，确保不超出可用 SGPR 数量。

3. **函数变换**: 预加载隐藏参数需要修改函数签名，因此触发 `cloneFunctionWithPreloadImplicitArgs`，这会产生新函数并将旧函数加入删除列表。

4. **连续性要求**: 预加载是连续的，从第一个参数开始，到某个参数无法预加载时停止，不支持跳跃式预加载。这适用于显式参数和隐藏参数。 llvm-project:236-248 

## Notes

<a name="ref-block_1"></a>- 该 pass 支持通过命令行选项控制：`-amdgpu-kernarg-preload` 启用/禁用预加载，`-amdgpu-kernarg-preload-count` 指定预加载的参数数量。 llvm-project:36-43[<sup>↗</sup>](#block_1) 

- 预加载机制假设 kernel 参数按顺序存储在 kernarg 段中，从基址偏移量开始。隐藏参数位于显式参数之后，通过 implicit argument pointer 访问。

- 目前不支持预加载聚合类型和 byref 参数，这是未来可能的改进方向。 llvm-project:304-315 

<a name="ref-block_9"></a>- 该实现包含 Legacy Pass Manager 和新 Pass Manager 两种版本的接口。 llvm-project:360-364[<sup>↗</sup>](#block_9)


### Citations


<a name="block_0"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUPreloadKernelArguments.cpp (L9-17) [<sup>↩</sup>](#ref-block_0)

```cpp
/// \file This pass preloads kernel arguments into user_data SGPRs before kernel
/// execution begins. The number of registers available for preloading depends
/// on the number of free user SGPRs, up to the hardware's maximum limit.
/// Implicit arguments enabled in the kernel descriptor are allocated first,
/// followed by SGPRs used for preloaded kernel arguments. (Reference:
/// https://llvm.org/docs/AMDGPUUsage.html#initial-kernel-execution-state)
/// Additionally, hidden kernel arguments may be preloaded, in which case they
/// are appended to the kernel signature after explicit arguments. Preloaded
/// arguments will be marked with `inreg`.
```

<a name="block_1"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUPreloadKernelArguments.cpp (L36-43) [<sup>↩</sup>](#ref-block_1)

```cpp
static cl::opt<unsigned> KernargPreloadCount(
    "amdgpu-kernarg-preload-count",
    cl::desc("How many kernel arguments to preload onto SGPRs"), cl::init(0));

static cl::opt<bool>
    EnableKernargPreload("amdgpu-kernarg-preload",
                         cl::desc("Enable preload kernel arguments to SGPRs"),
                         cl::init(true));
```

<a name="block_2"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUPreloadKernelArguments.cpp (L62-79) [<sup>↩</sup>](#ref-block_2)

```cpp
class PreloadKernelArgInfo {
private:
  Function &F;
  const GCNSubtarget &ST;
  unsigned NumFreeUserSGPRs;

  enum HiddenArg : unsigned {
    HIDDEN_BLOCK_COUNT_X,
    HIDDEN_BLOCK_COUNT_Y,
    HIDDEN_BLOCK_COUNT_Z,
    HIDDEN_GROUP_SIZE_X,
    HIDDEN_GROUP_SIZE_Y,
    HIDDEN_GROUP_SIZE_Z,
    HIDDEN_REMAINDER_X,
    HIDDEN_REMAINDER_Y,
    HIDDEN_REMAINDER_Z,
    END_HIDDEN_ARGS
  };
```

<a name="block_3"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUPreloadKernelArguments.cpp (L92-97) [<sup>↩</sup>](#ref-block_3)

```cpp
  static constexpr HiddenArgInfo HiddenArgs[END_HIDDEN_ARGS] = {
      {0, 4, "_hidden_block_count_x"}, {4, 4, "_hidden_block_count_y"},
      {8, 4, "_hidden_block_count_z"}, {12, 2, "_hidden_group_size_x"},
      {14, 2, "_hidden_group_size_y"}, {16, 2, "_hidden_group_size_z"},
      {18, 2, "_hidden_remainder_x"},  {20, 2, "_hidden_remainder_y"},
      {22, 2, "_hidden_remainder_z"}};
```

<a name="block_4"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUPreloadKernelArguments.cpp (L128-167) [<sup>↩</sup>](#ref-block_4)

```cpp
  Function *cloneFunctionWithPreloadImplicitArgs(unsigned LastPreloadIndex) {
    FunctionType *FT = F.getFunctionType();
    LLVMContext &Ctx = F.getParent()->getContext();
    SmallVector<Type *, 16> FTypes(FT->param_begin(), FT->param_end());
    for (unsigned I = 0; I <= LastPreloadIndex; ++I)
      FTypes.push_back(getHiddenArgType(Ctx, HiddenArg(I)));

    FunctionType *NFT =
        FunctionType::get(FT->getReturnType(), FTypes, FT->isVarArg());
    Function *NF =
        Function::Create(NFT, F.getLinkage(), F.getAddressSpace(), F.getName());

    NF->copyAttributesFrom(&F);
    NF->copyMetadata(&F, 0);

    F.getParent()->getFunctionList().insert(F.getIterator(), NF);
    NF->takeName(&F);
    NF->splice(NF->begin(), &F);

    Function::arg_iterator NFArg = NF->arg_begin();
    for (Argument &Arg : F.args()) {
      Arg.replaceAllUsesWith(&*NFArg);
      NFArg->takeName(&Arg);
      ++NFArg;
    }

    AttrBuilder AB(Ctx);
    AB.addAttribute(Attribute::InReg);
    AB.addAttribute("amdgpu-hidden-argument");
    AttributeList AL = NF->getAttributes();
    for (unsigned I = 0; I <= LastPreloadIndex; ++I) {
      AL = AL.addParamAttributes(Ctx, NFArg->getArgNo(), AB);
      NFArg++->setName(getHiddenArgName(HiddenArg(I)));
    }

    NF->setAttributes(AL);
    F.replaceAllUsesWith(NF);

    return NF;
  }
```

<a name="block_5"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUPreloadKernelArguments.cpp (L176-179) [<sup>↩</sup>](#ref-block_5)

```cpp
  void setInitialFreeUserSGPRsCount() {
    GCNUserSGPRUsageInfo UserSGPRInfo(F, ST);
    NumFreeUserSGPRs = UserSGPRInfo.getNumFreeUserSGPRs();
  }
```

<a name="block_6"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUPreloadKernelArguments.cpp (L181-183) [<sup>↩</sup>](#ref-block_6)

```cpp
  bool canPreloadKernArgAtOffset(uint64_t ExplicitArgOffset) {
    return ExplicitArgOffset <= NumFreeUserSGPRs * 4;
  }
```

<a name="block_7"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUPreloadKernelArguments.cpp (L186-262) [<sup>↩</sup>](#ref-block_7)

```cpp
  void
  tryAllocHiddenArgPreloadSGPRs(uint64_t ImplicitArgsBaseOffset,
                                SmallVectorImpl<Function *> &FunctionsToErase) {
    Function *ImplicitArgPtr = Intrinsic::getDeclarationIfExists(
        F.getParent(), Intrinsic::amdgcn_implicitarg_ptr);
    if (!ImplicitArgPtr)
      return;

    const DataLayout &DL = F.getParent()->getDataLayout();
    // Pair is the load and the load offset.
    SmallVector<std::pair<LoadInst *, unsigned>, 4> ImplicitArgLoads;
    for (auto *U : ImplicitArgPtr->users()) {
      Instruction *CI = dyn_cast<Instruction>(U);
      if (!CI || CI->getParent()->getParent() != &F)
        continue;

      for (auto *U : CI->users()) {
        int64_t Offset = 0;
        auto *Load = dyn_cast<LoadInst>(U); // Load from ImplicitArgPtr?
        if (!Load) {
          if (GetPointerBaseWithConstantOffset(U, Offset, DL) != CI)
            continue;

          Load = dyn_cast<LoadInst>(*U->user_begin()); // Load from GEP?
        }

        if (!Load || !Load->isSimple())
          continue;

        // FIXME: Expand handle merged loads.
        LLVMContext &Ctx = F.getParent()->getContext();
        Type *LoadTy = Load->getType();
        HiddenArg HA = getHiddenArgFromOffset(Offset);
        if (HA == END_HIDDEN_ARGS || LoadTy != getHiddenArgType(Ctx, HA))
          continue;

        ImplicitArgLoads.push_back(std::make_pair(Load, Offset));
      }
    }

    if (ImplicitArgLoads.empty())
      return;

    // Allocate loads in order of offset. We need to be sure that the implicit
    // argument can actually be preloaded.
    std::sort(ImplicitArgLoads.begin(), ImplicitArgLoads.end(), less_second());

    // If we fail to preload any implicit argument we know we don't have SGPRs
    // to preload any subsequent ones with larger offsets. Find the first
    // argument that we cannot preload.
    auto *PreloadEnd = llvm::find_if(
        ImplicitArgLoads, [&](const std::pair<LoadInst *, unsigned> &Load) {
          unsigned LoadSize = DL.getTypeStoreSize(Load.first->getType());
          unsigned LoadOffset = Load.second;
          if (!canPreloadKernArgAtOffset(LoadOffset + LoadSize +
                                         ImplicitArgsBaseOffset))
            return true;

          return false;
        });

    if (PreloadEnd == ImplicitArgLoads.begin())
      return;

    unsigned LastHiddenArgIndex = getHiddenArgFromOffset(PreloadEnd[-1].second);
    Function *NF = cloneFunctionWithPreloadImplicitArgs(LastHiddenArgIndex);
    assert(NF);
    FunctionsToErase.push_back(&F);
    for (const auto *I = ImplicitArgLoads.begin(); I != PreloadEnd; ++I) {
      LoadInst *LoadInst = I->first;
      unsigned LoadOffset = I->second;
      unsigned HiddenArgIndex = getHiddenArgFromOffset(LoadOffset);
      unsigned Index = NF->arg_size() - LastHiddenArgIndex + HiddenArgIndex - 1;
      Argument *Arg = NF->getArg(Index);
      LoadInst->replaceAllUsesWith(Arg);
    }
  }
```

<a name="block_8"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUPreloadKernelArguments.cpp (L282-351) [<sup>↩</sup>](#ref-block_8)

```cpp
static bool markKernelArgsAsInreg(Module &M, const TargetMachine &TM) {
  if (!EnableKernargPreload)
    return false;

  SmallVector<Function *, 4> FunctionsToErase;
  bool Changed = false;
  for (auto &F : M) {
    const GCNSubtarget &ST = TM.getSubtarget<GCNSubtarget>(F);
    if (!ST.hasKernargPreload() ||
        F.getCallingConv() != CallingConv::AMDGPU_KERNEL)
      continue;

    PreloadKernelArgInfo PreloadInfo(F, ST);
    uint64_t ExplicitArgOffset = 0;
    const DataLayout &DL = F.getDataLayout();
    const uint64_t BaseOffset = ST.getExplicitKernelArgOffset();
    unsigned NumPreloadsRequested = KernargPreloadCount;
    unsigned NumPreloadedExplicitArgs = 0;
    for (Argument &Arg : F.args()) {
      // Avoid incompatible attributes and guard against running this pass
      // twice.
      //
      // TODO: Preload byref kernel arguments
      if (Arg.hasByRefAttr() || Arg.hasNestAttr() ||
          Arg.hasAttribute("amdgpu-hidden-argument"))
        break;

      // Inreg may be pre-existing on some arguments, try to preload these.
      if (NumPreloadsRequested == 0 && !Arg.hasInRegAttr())
        break;

      // FIXME: Preload aggregates.
      if (Arg.getType()->isAggregateType())
        break;

      Type *ArgTy = Arg.getType();
      Align ABITypeAlign = DL.getABITypeAlign(ArgTy);
      uint64_t AllocSize = DL.getTypeAllocSize(ArgTy);
      ExplicitArgOffset = alignTo(ExplicitArgOffset, ABITypeAlign) + AllocSize;

      if (!PreloadInfo.canPreloadKernArgAtOffset(ExplicitArgOffset))
        break;

      Arg.addAttr(Attribute::InReg);
      NumPreloadedExplicitArgs++;
      if (NumPreloadsRequested > 0)
        NumPreloadsRequested--;
    }

    // Only try preloading hidden arguments if we can successfully preload the
    // last explicit argument.
    if (NumPreloadedExplicitArgs == F.arg_size()) {
      uint64_t ImplicitArgsBaseOffset =
          alignTo(ExplicitArgOffset, ST.getAlignmentForImplicitArgPtr()) +
          BaseOffset;
      PreloadInfo.tryAllocHiddenArgPreloadSGPRs(ImplicitArgsBaseOffset,
                                                FunctionsToErase);
    }

    Changed |= NumPreloadedExplicitArgs > 0;
  }

  Changed |= !FunctionsToErase.empty();
  // Erase cloned functions if we needed to update the kernel signature to
  // support preloading hidden kernel arguments.
  for (auto *F : FunctionsToErase)
    F->eraseFromParent();

  return Changed;
}
```

<a name="block_9"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUPreloadKernelArguments.cpp (L360-364) [<sup>↩</sup>](#ref-block_9)

```cpp
PreservedAnalyses
AMDGPUPreloadKernelArgumentsPass::run(Module &M, ModuleAnalysisManager &AM) {
  bool Changed = markKernelArgsAsInreg(M, TM);
  return Changed ? PreservedAnalyses::none() : PreservedAnalyses::all();
}
```

