# AMDGPUSwLowerLDS.cpp 代码功能详解

## 1. Pass 主要功能概括

<a name="ref-block_0"></a>这个 pass 的核心功能是将 AMDGPU 的本地数据存储（LDS, Local Data Store）访问降低为动态分配的全局内存访问，主要目的是支持地址消毒器（Address Sanitizer）的检测功能。 llvm-project:9-16[<sup>↗</sup>](#block_0) 

**主要作用和效果：**
- 将内核和非内核函数中的 LDS 使用转换为全局内存操作
- 在全局内存中模拟 Packed LDS 布局
- 对降低后的内存指令进行地址消毒器插桩，以捕获寻址错误
<a name="ref-block_12"></a>- 仅在启用地址消毒器并通过 "nosanitize_address" 模块标志检测到已插桩的 IR 时才工作 llvm-project:1322-1324[<sup>↗</sup>](#block_12) 

## 2. 主要实现步骤和子功能

基于代码文件的完整分析，该 pass 包含以下主要步骤和子功能：

### 2.1 初始化和分析阶段
- **消除常量表达式使用**
- **获取 LDS 传递使用信息**（`getTransitiveUsesOfLDS`）
- **分类 LDS 访问类型**（静态/动态、直接/间接）
- **初始化地址消毒器信息**（`initAsanInfo`）

### 2.2 内核 LDS 访问处理
- **构建 SW LDS 全局变量**（`buildSwLDSGlobal`）
- **构建 SW 动态 LDS 全局变量**（`buildSwDynLDSGlobal`）
- **填充元数据全局变量**（`populateSwMetadataGlobal`）
- **填充 LDS 属性和元数据**（`populateSwLDSAttributeAndMetadata`）
- **填充 LDS 替换索引映射**（`populateLDSToReplacementIndicesMap`）
- **降低内核 LDS 访问**（`lowerKernelLDSAccesses`）
- **替换内核 LDS 访问**（`replaceKernelLDSAccesses`）

### 2.3 内存操作转换
- **获取 LDS 内存指令**（`getLDSMemoryInstructions`）
- **转换 LDS 内存操作为全局内存**（`translateLDSMemoryOperationsToGlobalMemory`）
- **获取转换后的全局内存指针**（`getTranslatedGlobalMemoryPtrOfLDS`）
- **投毒红区**（`poisonRedzones`）

### 2.4 非内核 LDS 访问处理
- **获取非内核的 LDS 使用**（`getUsesOfLDSByNonKernels`）
- **获取带 LDS 参数的非内核函数**（`getNonKernelsWithLDSArguments`）
- **构建非内核 LDS 基表**（`buildNonKernelLDSBaseTable`）
- **构建非内核 LDS 偏移表**（`buildNonKernelLDSOffsetTable`）
- **降低非内核 LDS 访问**（`lowerNonKernelLDSAccesses`）

### 2.5 地址消毒器插桩
- **对降低后的内存操作进行插桩**

## 3. 各步骤/子功能的具体描述分析

### 3.1 构建 SW LDS 全局变量 (`buildSwLDSGlobal`)

<a name="ref-block_1"></a>为每个内核创建一个新的 LDS 全局变量，用于存储设备全局内存指针。这个变量的名称格式为 `llvm.amdgcn.sw.lds.<kernel-name>`，是一个指向全局地址空间的指针类型。 llvm-project:345-357[<sup>↗</sup>](#block_1) 

### 3.2 构建 SW 动态 LDS 全局变量 (`buildSwDynLDSGlobal`)

<a name="ref-block_2"></a>如果内核访问动态 LDS，则创建一个动态 LDS 全局变量。这个变量用于标记动态 LDS 的使用，并通过绝对符号元数据记录静态 LDS 的总大小。 llvm-project:359-375[<sup>↗</sup>](#block_2) 

### 3.3 填充元数据全局变量 (`populateSwMetadataGlobal`)

<a name="ref-block_3"></a>创建元数据全局变量，存储所有 LDS 访问对应的偏移量和大小信息。元数据结构为 `{i32, i32, i32}`，分别表示偏移量、大小和对齐后的总大小。该函数还计算包括红区（redzone）在内的总分配大小。 llvm-project:387-493[<sup>↗</sup>](#block_3) 

### 3.4 降低内核 LDS 访问 (`lowerKernelLDSAccesses`)

这是内核 LDS 访问降低的核心函数，执行以下操作：
- 创建基本块结构（Malloc 块、WId 块、CondFree 块、Free 块、End 块）
- 只有工作组中的 {0,0,0} 工作项执行 malloc 和 free 操作
- 插入工作组屏障同步
- 调用 `__asan_malloc_impl` 分配全局内存
- 调用 `__asan_poison_region` 投毒红区
<a name="ref-block_7"></a>- 在所有返回点之前插入 free 操作 llvm-project:769-952[<sup>↗</sup>](#block_7) 

### 3.5 转换 LDS 内存操作 (`translateLDSMemoryOperationsToGlobalMemory`)

<a name="ref-block_5"></a>将 LDS 地址空间的内存操作（load、store、atomic 等）转换为全局地址空间的相应操作。每个转换后的指令都会被添加到地址消毒器插桩列表中。 llvm-project:681-745[<sup>↗</sup>](#block_5) 

### 3.6 更新动态 LDS 的 Malloc 大小 (`updateMallocSizeForDynamicLDS`)

<a name="ref-block_4"></a>对于动态 LDS，从内核的隐藏参数 `hidden_dynamic_lds_size` 读取运行时大小，并更新元数据全局变量中的偏移量、大小和对齐大小字段。 llvm-project:581-627[<sup>↗</sup>](#block_4) 

### 3.7 构建非内核 LDS 基表 (`buildNonKernelLDSBaseTable`)

<a name="ref-block_8"></a>为非内核函数创建基表，该表是一个单行数组，每个元素对应一个内核的 SW LDS 全局变量的地址。元素按内核 ID 排列。 llvm-project:985-1017[<sup>↗</sup>](#block_8) 

### 3.8 构建非内核 LDS 偏移表 (`buildNonKernelLDSOffsetTable`)

创建偏移表，这是一个二维数组：
- 行数 = 通过非内核访问 LDS 的内核数量
- 列数 = 所有非内核访问的唯一 LDS 全局变量数量
<a name="ref-block_9"></a>- 每个元素是特定内核对特定 LDS 全局变量的替换地址 llvm-project:1019-1055[<sup>↗</sup>](#block_9) 

### 3.9 降低非内核 LDS 访问 (`lowerNonKernelLDSAccesses`)

<a name="ref-block_10"></a>在非内核函数中，使用内核 ID 内部函数查询当前内核，然后从基表和偏移表中获取相应的 SW LDS 指针和偏移量，完成 LDS 访问的替换。 llvm-project:1057-1101[<sup>↗</sup>](#block_10) 

### 3.10 投毒红区 (`poisonRedzones`)

<a name="ref-block_6"></a>为每个 LDS 全局变量的红区调用 `__asan_poison_region` 函数，将红区标记为不可访问，以便地址消毒器可以检测到越界访问。 llvm-project:747-767[<sup>↗</sup>](#block_6) 

## 4. 步骤和子功能之间的关系

### 4.1 整体流程关系

<a name="ref-block_11"></a>主函数 `run()` 协调整个转换过程： llvm-project:1146-1296[<sup>↗</sup>](#block_11) 

**流程依赖关系：**

1. **分析阶段** → **内核处理阶段** → **非内核处理阶段** → **清理和插桩阶段**

2. **内核 LDS 访问处理的内部依赖：**
   - `buildSwLDSGlobal` 必须在 `populateSwMetadataGlobal` 之前执行
   - `populateSwMetadataGlobal` 依赖于所有 LDS 全局变量的分类（静态/动态、直接/间接）
   - `populateLDSToReplacementIndicesMap` 依赖于 `populateSwMetadataGlobal` 建立的索引结构
   - `replaceKernelLDSAccesses` 使用 `populateLDSToReplacementIndicesMap` 创建的映射
   - `lowerKernelLDSAccesses` 是最后执行的，它调用 `replaceKernelLDSAccesses` 和 `translateLDSMemoryOperationsToGlobalMemory`

3. **非内核 LDS 访问处理的内部依赖：**
   - `getUsesOfLDSByNonKernels` 和 `getNonKernelsWithLDSArguments` 必须在构建表之前执行
   - `buildNonKernelLDSBaseTable` 和 `buildNonKernelLDSOffsetTable` 依赖于所有内核的 SW LDS 已创建
   - `lowerNonKernelLDSAccesses` 依赖于基表和偏移表已构建

### 4.2 数据流关系

- **SW LDS 全局变量** 是内核和非内核访问的中心枢纽
- **元数据全局变量** 提供偏移量信息，被 `replaceKernelLDSAccesses` 使用
- **基表和偏移表** 连接内核的 SW LDS 和非内核的 LDS 访问
- **LDS 替换索引映射** 将原始 LDS 全局变量映射到元数据结构中的索引

### 4.3 同步和屏障关系

在内核执行期间：
- **Malloc 块**：{0,0,0} 工作项执行内存分配和红区投毒
- **屏障同步**：确保所有工作项在使用分配的内存前等待分配完成
- **CondFree 块和 Free 块**：{0,0,0} 工作项在内核结束时释放内存 llvm-project:897-898 

## Notes

这个 pass 是 AMDGPU 后端支持地址消毒器的关键组件。通过将 LDS（本地数据存储）转换为全局内存访问，它使得原本无法被地址消毒器检测的 LDS 访问错误变得可检测。转换过程涉及复杂的元数据管理、内存布局计算和红区插入，以确保既能正确模拟原始 LDS 行为，又能支持完整的消毒器功能。

内核和非内核的处理方式不同，主要是因为非内核函数可能被多个内核调用，需要通过查表机制动态确定使用哪个内核的 SW LDS。这种设计保证了代码的正确性和灵活性。


### Citations


<a name="block_0"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUSwLowerLDS.cpp (L9-16) [<sup>↩</sup>](#ref-block_0)

```cpp
// This pass lowers the local data store, LDS, uses in kernel and non-kernel
// functions in module to use dynamically allocated global memory.
// Packed LDS Layout is emulated in the global memory.
// The lowered memory instructions from LDS to global memory are then
// instrumented for address sanitizer, to catch addressing errors.
// This pass only work when address sanitizer has been enabled and has
// instrumented the IR. It identifies that IR has been instrumented using
// "nosanitize_address" module flag.
```

<a name="block_1"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUSwLowerLDS.cpp (L345-357) [<sup>↩</sup>](#ref-block_1)

```cpp
void AMDGPUSwLowerLDS::buildSwLDSGlobal(Function *Func) {
  // Create new LDS global required for each kernel to store
  // device global memory pointer.
  auto &LDSParams = FuncLDSAccessInfo.KernelToLDSParametersMap[Func];
  // Create new global pointer variable
  LDSParams.SwLDS = new GlobalVariable(
      M, IRB.getPtrTy(), false, GlobalValue::InternalLinkage,
      PoisonValue::get(IRB.getPtrTy()), "llvm.amdgcn.sw.lds." + Func->getName(),
      nullptr, GlobalValue::NotThreadLocal, AMDGPUAS::LOCAL_ADDRESS, false);
  GlobalValue::SanitizerMetadata MD;
  MD.NoAddress = true;
  LDSParams.SwLDS->setSanitizerMetadata(MD);
}
```

<a name="block_2"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUSwLowerLDS.cpp (L359-375) [<sup>↩</sup>](#ref-block_2)

```cpp
void AMDGPUSwLowerLDS::buildSwDynLDSGlobal(Function *Func) {
  // Create new Dyn LDS global if kernel accesses dyn LDS.
  auto &LDSParams = FuncLDSAccessInfo.KernelToLDSParametersMap[Func];
  if (LDSParams.DirectAccess.DynamicLDSGlobals.empty() &&
      LDSParams.IndirectAccess.DynamicLDSGlobals.empty())
    return;
  // Create new global pointer variable
  auto *emptyCharArray = ArrayType::get(IRB.getInt8Ty(), 0);
  LDSParams.SwDynLDS = new GlobalVariable(
      M, emptyCharArray, false, GlobalValue::ExternalLinkage, nullptr,
      "llvm.amdgcn." + Func->getName() + ".dynlds", nullptr,
      GlobalValue::NotThreadLocal, AMDGPUAS::LOCAL_ADDRESS, false);
  markUsedByKernel(Func, LDSParams.SwDynLDS);
  GlobalValue::SanitizerMetadata MD;
  MD.NoAddress = true;
  LDSParams.SwDynLDS->setSanitizerMetadata(MD);
}
```

<a name="block_3"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUSwLowerLDS.cpp (L387-493) [<sup>↩</sup>](#ref-block_3)

```cpp
void AMDGPUSwLowerLDS::populateSwMetadataGlobal(Function *Func) {
  // Create new metadata global for every kernel and initialize the
  // start offsets and sizes corresponding to each LDS accesses.
  auto &LDSParams = FuncLDSAccessInfo.KernelToLDSParametersMap[Func];
  auto &Ctx = M.getContext();
  auto &DL = M.getDataLayout();
  std::vector<Type *> Items;
  Type *Int32Ty = IRB.getInt32Ty();
  std::vector<Constant *> Initializers;
  Align MaxAlignment(1);
  auto UpdateMaxAlignment = [&MaxAlignment, &DL](GlobalVariable *GV) {
    Align GVAlign = AMDGPU::getAlign(DL, GV);
    MaxAlignment = std::max(MaxAlignment, GVAlign);
  };

  for (GlobalVariable *GV : LDSParams.DirectAccess.StaticLDSGlobals)
    UpdateMaxAlignment(GV);

  for (GlobalVariable *GV : LDSParams.DirectAccess.DynamicLDSGlobals)
    UpdateMaxAlignment(GV);

  for (GlobalVariable *GV : LDSParams.IndirectAccess.StaticLDSGlobals)
    UpdateMaxAlignment(GV);

  for (GlobalVariable *GV : LDSParams.IndirectAccess.DynamicLDSGlobals)
    UpdateMaxAlignment(GV);

  //{StartOffset, AlignedSizeInBytes}
  SmallString<128> MDItemStr;
  raw_svector_ostream MDItemOS(MDItemStr);
  MDItemOS << "llvm.amdgcn.sw.lds." << Func->getName() << ".md.item";

  StructType *LDSItemTy =
      StructType::create(Ctx, {Int32Ty, Int32Ty, Int32Ty}, MDItemOS.str());
  uint32_t &MallocSize = LDSParams.MallocSize;
  SetVector<GlobalVariable *> UniqueLDSGlobals;
  int AsanScale = AsanInfo.Scale;
  auto buildInitializerForSwLDSMD =
      [&](SetVector<GlobalVariable *> &LDSGlobals) {
        for (auto &GV : LDSGlobals) {
          if (is_contained(UniqueLDSGlobals, GV))
            continue;
          UniqueLDSGlobals.insert(GV);

          Type *Ty = GV->getValueType();
          const uint64_t SizeInBytes = DL.getTypeAllocSize(Ty);
          Items.push_back(LDSItemTy);
          Constant *ItemStartOffset = ConstantInt::get(Int32Ty, MallocSize);
          Constant *SizeInBytesConst = ConstantInt::get(Int32Ty, SizeInBytes);
          // Get redzone size corresponding a size.
          const uint64_t RightRedzoneSize =
              AMDGPU::getRedzoneSizeForGlobal(AsanScale, SizeInBytes);
          // Update MallocSize with current size and redzone size.
          MallocSize += SizeInBytes;
          if (!AMDGPU::isDynamicLDS(*GV))
            LDSParams.RedzoneOffsetAndSizeVector.emplace_back(MallocSize,
                                                              RightRedzoneSize);
          MallocSize += RightRedzoneSize;
          // Align current size plus redzone.
          uint64_t AlignedSize =
              alignTo(SizeInBytes + RightRedzoneSize, MaxAlignment);
          Constant *AlignedSizeInBytesConst =
              ConstantInt::get(Int32Ty, AlignedSize);
          // Align MallocSize
          MallocSize = alignTo(MallocSize, MaxAlignment);
          Constant *InitItem =
              ConstantStruct::get(LDSItemTy, {ItemStartOffset, SizeInBytesConst,
                                              AlignedSizeInBytesConst});
          Initializers.push_back(InitItem);
        }
      };
  SetVector<GlobalVariable *> SwLDSVector;
  SwLDSVector.insert(LDSParams.SwLDS);
  buildInitializerForSwLDSMD(SwLDSVector);
  buildInitializerForSwLDSMD(LDSParams.DirectAccess.StaticLDSGlobals);
  buildInitializerForSwLDSMD(LDSParams.IndirectAccess.StaticLDSGlobals);
  buildInitializerForSwLDSMD(LDSParams.DirectAccess.DynamicLDSGlobals);
  buildInitializerForSwLDSMD(LDSParams.IndirectAccess.DynamicLDSGlobals);

  // Update the LDS size used by the kernel.
  Type *Ty = LDSParams.SwLDS->getValueType();
  const uint64_t SizeInBytes = DL.getTypeAllocSize(Ty);
  uint64_t AlignedSize = alignTo(SizeInBytes, MaxAlignment);
  LDSParams.LDSSize = AlignedSize;
  SmallString<128> MDTypeStr;
  raw_svector_ostream MDTypeOS(MDTypeStr);
  MDTypeOS << "llvm.amdgcn.sw.lds." << Func->getName() << ".md.type";
  StructType *MetadataStructType =
      StructType::create(Ctx, Items, MDTypeOS.str());
  SmallString<128> MDStr;
  raw_svector_ostream MDOS(MDStr);
  MDOS << "llvm.amdgcn.sw.lds." << Func->getName() << ".md";
  LDSParams.SwLDSMetadata = new GlobalVariable(
      M, MetadataStructType, false, GlobalValue::InternalLinkage,
      PoisonValue::get(MetadataStructType), MDOS.str(), nullptr,
      GlobalValue::NotThreadLocal, AMDGPUAS::GLOBAL_ADDRESS, false);
  Constant *data = ConstantStruct::get(MetadataStructType, Initializers);
  LDSParams.SwLDSMetadata->setInitializer(data);
  assert(LDSParams.SwLDS);
  // Set the alignment to MaxAlignment for SwLDS.
  LDSParams.SwLDS->setAlignment(MaxAlignment);
  if (LDSParams.SwDynLDS)
    LDSParams.SwDynLDS->setAlignment(MaxAlignment);
  GlobalValue::SanitizerMetadata MD;
  MD.NoAddress = true;
  LDSParams.SwLDSMetadata->setSanitizerMetadata(MD);
}
```

<a name="block_4"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUSwLowerLDS.cpp (L581-627) [<sup>↩</sup>](#ref-block_4)

```cpp
void AMDGPUSwLowerLDS::updateMallocSizeForDynamicLDS(
    Function *Func, Value **CurrMallocSize, Value *HiddenDynLDSSize,
    SetVector<GlobalVariable *> &DynamicLDSGlobals) {
  auto &LDSParams = FuncLDSAccessInfo.KernelToLDSParametersMap[Func];
  Type *Int32Ty = IRB.getInt32Ty();

  GlobalVariable *SwLDS = LDSParams.SwLDS;
  GlobalVariable *SwLDSMetadata = LDSParams.SwLDSMetadata;
  assert(SwLDS && SwLDSMetadata);
  StructType *MetadataStructType =
      cast<StructType>(SwLDSMetadata->getValueType());
  unsigned MaxAlignment = SwLDS->getAlignment();
  Value *MaxAlignValue = IRB.getInt32(MaxAlignment);
  Value *MaxAlignValueMinusOne = IRB.getInt32(MaxAlignment - 1);

  for (GlobalVariable *DynGV : DynamicLDSGlobals) {
    auto &Indices = LDSParams.LDSToReplacementIndicesMap[DynGV];
    // Update the Offset metadata.
    Constant *Index0 = ConstantInt::get(Int32Ty, 0);
    Constant *Index1 = ConstantInt::get(Int32Ty, Indices[1]);

    Constant *Index2Offset = ConstantInt::get(Int32Ty, 0);
    auto *GEPForOffset = IRB.CreateInBoundsGEP(
        MetadataStructType, SwLDSMetadata, {Index0, Index1, Index2Offset});

    IRB.CreateStore(*CurrMallocSize, GEPForOffset);
    // Update the size and Aligned Size metadata.
    Constant *Index2Size = ConstantInt::get(Int32Ty, 1);
    auto *GEPForSize = IRB.CreateInBoundsGEP(MetadataStructType, SwLDSMetadata,
                                             {Index0, Index1, Index2Size});

    Value *CurrDynLDSSize = IRB.CreateLoad(Int32Ty, HiddenDynLDSSize);
    IRB.CreateStore(CurrDynLDSSize, GEPForSize);
    Constant *Index2AlignedSize = ConstantInt::get(Int32Ty, 2);
    auto *GEPForAlignedSize = IRB.CreateInBoundsGEP(
        MetadataStructType, SwLDSMetadata, {Index0, Index1, Index2AlignedSize});

    Value *AlignedDynLDSSize =
        IRB.CreateAdd(CurrDynLDSSize, MaxAlignValueMinusOne);
    AlignedDynLDSSize = IRB.CreateUDiv(AlignedDynLDSSize, MaxAlignValue);
    AlignedDynLDSSize = IRB.CreateMul(AlignedDynLDSSize, MaxAlignValue);
    IRB.CreateStore(AlignedDynLDSSize, GEPForAlignedSize);

    // Update the Current Malloc Size
    *CurrMallocSize = IRB.CreateAdd(*CurrMallocSize, AlignedDynLDSSize);
  }
}
```

<a name="block_5"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUSwLowerLDS.cpp (L681-745) [<sup>↩</sup>](#ref-block_5)

```cpp
void AMDGPUSwLowerLDS::translateLDSMemoryOperationsToGlobalMemory(
    Function *Func, Value *LoadMallocPtr,
    SetVector<Instruction *> &LDSInstructions) {
  LLVM_DEBUG(dbgs() << "Translating LDS memory operations to global memory : "
                    << Func->getName());
  for (Instruction *Inst : LDSInstructions) {
    IRB.SetInsertPoint(Inst);
    if (LoadInst *LI = dyn_cast<LoadInst>(Inst)) {
      Value *LIOperand = LI->getPointerOperand();
      Value *Replacement =
          getTranslatedGlobalMemoryPtrOfLDS(LoadMallocPtr, LIOperand);
      LoadInst *NewLI = IRB.CreateAlignedLoad(LI->getType(), Replacement,
                                              LI->getAlign(), LI->isVolatile());
      NewLI->setAtomic(LI->getOrdering(), LI->getSyncScopeID());
      AsanInfo.Instructions.insert(NewLI);
      LI->replaceAllUsesWith(NewLI);
      LI->eraseFromParent();
    } else if (StoreInst *SI = dyn_cast<StoreInst>(Inst)) {
      Value *SIOperand = SI->getPointerOperand();
      Value *Replacement =
          getTranslatedGlobalMemoryPtrOfLDS(LoadMallocPtr, SIOperand);
      StoreInst *NewSI = IRB.CreateAlignedStore(
          SI->getValueOperand(), Replacement, SI->getAlign(), SI->isVolatile());
      NewSI->setAtomic(SI->getOrdering(), SI->getSyncScopeID());
      AsanInfo.Instructions.insert(NewSI);
      SI->replaceAllUsesWith(NewSI);
      SI->eraseFromParent();
    } else if (AtomicRMWInst *RMW = dyn_cast<AtomicRMWInst>(Inst)) {
      Value *RMWPtrOperand = RMW->getPointerOperand();
      Value *RMWValOperand = RMW->getValOperand();
      Value *Replacement =
          getTranslatedGlobalMemoryPtrOfLDS(LoadMallocPtr, RMWPtrOperand);
      AtomicRMWInst *NewRMW = IRB.CreateAtomicRMW(
          RMW->getOperation(), Replacement, RMWValOperand, RMW->getAlign(),
          RMW->getOrdering(), RMW->getSyncScopeID());
      NewRMW->setVolatile(RMW->isVolatile());
      AsanInfo.Instructions.insert(NewRMW);
      RMW->replaceAllUsesWith(NewRMW);
      RMW->eraseFromParent();
    } else if (AtomicCmpXchgInst *XCHG = dyn_cast<AtomicCmpXchgInst>(Inst)) {
      Value *XCHGPtrOperand = XCHG->getPointerOperand();
      Value *Replacement =
          getTranslatedGlobalMemoryPtrOfLDS(LoadMallocPtr, XCHGPtrOperand);
      AtomicCmpXchgInst *NewXCHG = IRB.CreateAtomicCmpXchg(
          Replacement, XCHG->getCompareOperand(), XCHG->getNewValOperand(),
          XCHG->getAlign(), XCHG->getSuccessOrdering(),
          XCHG->getFailureOrdering(), XCHG->getSyncScopeID());
      NewXCHG->setVolatile(XCHG->isVolatile());
      AsanInfo.Instructions.insert(NewXCHG);
      XCHG->replaceAllUsesWith(NewXCHG);
      XCHG->eraseFromParent();
    } else if (AddrSpaceCastInst *ASC = dyn_cast<AddrSpaceCastInst>(Inst)) {
      Value *AIOperand = ASC->getPointerOperand();
      Value *Replacement =
          getTranslatedGlobalMemoryPtrOfLDS(LoadMallocPtr, AIOperand);
      Value *NewAI = IRB.CreateAddrSpaceCast(Replacement, ASC->getType());
      // Note: No need to add the instruction to AsanInfo instructions to be
      // instrumented list. FLAT_ADDRESS ptr would have been already
      // instrumented by asan pass prior to this pass.
      ASC->replaceAllUsesWith(NewAI);
      ASC->eraseFromParent();
    } else
      report_fatal_error("Unimplemented LDS lowering instruction");
  }
}
```

<a name="block_6"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUSwLowerLDS.cpp (L747-767) [<sup>↩</sup>](#ref-block_6)

```cpp
void AMDGPUSwLowerLDS::poisonRedzones(Function *Func, Value *MallocPtr) {
  auto &LDSParams = FuncLDSAccessInfo.KernelToLDSParametersMap[Func];
  Type *Int64Ty = IRB.getInt64Ty();
  Type *VoidTy = IRB.getVoidTy();
  FunctionCallee AsanPoisonRegion = M.getOrInsertFunction(
      "__asan_poison_region",
      FunctionType::get(VoidTy, {Int64Ty, Int64Ty}, false));

  auto RedzonesVec = LDSParams.RedzoneOffsetAndSizeVector;
  size_t VecSize = RedzonesVec.size();
  for (unsigned i = 0; i < VecSize; i++) {
    auto &RedzonePair = RedzonesVec[i];
    uint64_t RedzoneOffset = RedzonePair.first;
    uint64_t RedzoneSize = RedzonePair.second;
    Value *RedzoneAddrOffset = IRB.CreateInBoundsGEP(
        IRB.getInt8Ty(), MallocPtr, {IRB.getInt64(RedzoneOffset)});
    Value *RedzoneAddress = IRB.CreatePtrToInt(RedzoneAddrOffset, Int64Ty);
    IRB.CreateCall(AsanPoisonRegion,
                   {RedzoneAddress, IRB.getInt64(RedzoneSize)});
  }
}
```

<a name="block_7"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUSwLowerLDS.cpp (L769-952) [<sup>↩</sup>](#ref-block_7)

```cpp
void AMDGPUSwLowerLDS::lowerKernelLDSAccesses(Function *Func,
                                              DomTreeUpdater &DTU) {
  LLVM_DEBUG(dbgs() << "Sw Lowering Kernel LDS for : " << Func->getName());
  auto &LDSParams = FuncLDSAccessInfo.KernelToLDSParametersMap[Func];
  auto &Ctx = M.getContext();
  auto *PrevEntryBlock = &Func->getEntryBlock();
  SetVector<Instruction *> LDSInstructions;
  getLDSMemoryInstructions(Func, LDSInstructions);

  // Create malloc block.
  auto *MallocBlock = BasicBlock::Create(Ctx, "Malloc", Func, PrevEntryBlock);

  // Create WIdBlock block which has instructions related to selection of
  // {0,0,0} indiex work item in the work group.
  auto *WIdBlock = BasicBlock::Create(Ctx, "WId", Func, MallocBlock);
  IRB.SetInsertPoint(WIdBlock, WIdBlock->begin());
  DebugLoc FirstDL =
      getOrCreateDebugLoc(&*PrevEntryBlock->begin(), Func->getSubprogram());
  IRB.SetCurrentDebugLocation(FirstDL);
  Value *WIdx = IRB.CreateIntrinsic(Intrinsic::amdgcn_workitem_id_x, {});
  Value *WIdy = IRB.CreateIntrinsic(Intrinsic::amdgcn_workitem_id_y, {});
  Value *WIdz = IRB.CreateIntrinsic(Intrinsic::amdgcn_workitem_id_z, {});
  Value *XYOr = IRB.CreateOr(WIdx, WIdy);
  Value *XYZOr = IRB.CreateOr(XYOr, WIdz);
  Value *WIdzCond = IRB.CreateICmpEQ(XYZOr, IRB.getInt32(0));

  // All work items will branch to PrevEntryBlock except {0,0,0} index
  // work item which will branch to malloc block.
  IRB.CreateCondBr(WIdzCond, MallocBlock, PrevEntryBlock);

  // Malloc block
  IRB.SetInsertPoint(MallocBlock, MallocBlock->begin());

  // If Dynamic LDS globals are accessed by the kernel,
  // Get the size of dyn lds from hidden dyn_lds_size kernel arg.
  // Update the corresponding metadata global entries for this dyn lds global.
  GlobalVariable *SwLDS = LDSParams.SwLDS;
  GlobalVariable *SwLDSMetadata = LDSParams.SwLDSMetadata;
  assert(SwLDS && SwLDSMetadata);
  StructType *MetadataStructType =
      cast<StructType>(SwLDSMetadata->getValueType());
  uint32_t MallocSize = 0;
  Value *CurrMallocSize;
  Type *Int32Ty = IRB.getInt32Ty();
  Type *Int64Ty = IRB.getInt64Ty();

  SetVector<GlobalVariable *> UniqueLDSGlobals;
  auto GetUniqueLDSGlobals = [&](SetVector<GlobalVariable *> &LDSGlobals) {
    for (auto &GV : LDSGlobals) {
      if (is_contained(UniqueLDSGlobals, GV))
        continue;
      UniqueLDSGlobals.insert(GV);
    }
  };

  GetUniqueLDSGlobals(LDSParams.DirectAccess.StaticLDSGlobals);
  GetUniqueLDSGlobals(LDSParams.IndirectAccess.StaticLDSGlobals);
  unsigned NumStaticLDS = 1 + UniqueLDSGlobals.size();
  UniqueLDSGlobals.clear();

  if (NumStaticLDS) {
    auto *GEPForEndStaticLDSOffset =
        IRB.CreateInBoundsGEP(MetadataStructType, SwLDSMetadata,
                              {ConstantInt::get(Int32Ty, 0),
                               ConstantInt::get(Int32Ty, NumStaticLDS - 1),
                               ConstantInt::get(Int32Ty, 0)});

    auto *GEPForEndStaticLDSSize =
        IRB.CreateInBoundsGEP(MetadataStructType, SwLDSMetadata,
                              {ConstantInt::get(Int32Ty, 0),
                               ConstantInt::get(Int32Ty, NumStaticLDS - 1),
                               ConstantInt::get(Int32Ty, 2)});

    Value *EndStaticLDSOffset =
        IRB.CreateLoad(Int32Ty, GEPForEndStaticLDSOffset);
    Value *EndStaticLDSSize = IRB.CreateLoad(Int32Ty, GEPForEndStaticLDSSize);
    CurrMallocSize = IRB.CreateAdd(EndStaticLDSOffset, EndStaticLDSSize);
  } else
    CurrMallocSize = IRB.getInt32(MallocSize);

  if (LDSParams.SwDynLDS) {
    if (!(AMDGPU::getAMDHSACodeObjectVersion(M) >= AMDGPU::AMDHSA_COV5))
      report_fatal_error(
          "Dynamic LDS size query is only supported for CO V5 and later.");
    // Get size from hidden dyn_lds_size argument of kernel
    Value *ImplicitArg =
        IRB.CreateIntrinsic(Intrinsic::amdgcn_implicitarg_ptr, {});
    Value *HiddenDynLDSSize = IRB.CreateInBoundsGEP(
        ImplicitArg->getType(), ImplicitArg,
        {ConstantInt::get(Int64Ty, COV5_HIDDEN_DYN_LDS_SIZE_ARG)});
    UniqueLDSGlobals.clear();
    GetUniqueLDSGlobals(LDSParams.DirectAccess.DynamicLDSGlobals);
    GetUniqueLDSGlobals(LDSParams.IndirectAccess.DynamicLDSGlobals);
    updateMallocSizeForDynamicLDS(Func, &CurrMallocSize, HiddenDynLDSSize,
                                  UniqueLDSGlobals);
  }

  CurrMallocSize = IRB.CreateZExt(CurrMallocSize, Int64Ty);

  // Create a call to malloc function which does device global memory allocation
  // with size equals to all LDS global accesses size in this kernel.
  Value *ReturnAddress =
      IRB.CreateIntrinsic(Intrinsic::returnaddress, {IRB.getInt32(0)});
  FunctionCallee MallocFunc = M.getOrInsertFunction(
      StringRef("__asan_malloc_impl"),
      FunctionType::get(Int64Ty, {Int64Ty, Int64Ty}, false));
  Value *RAPtrToInt = IRB.CreatePtrToInt(ReturnAddress, Int64Ty);
  Value *MallocCall = IRB.CreateCall(MallocFunc, {CurrMallocSize, RAPtrToInt});

  Value *MallocPtr =
      IRB.CreateIntToPtr(MallocCall, IRB.getPtrTy(AMDGPUAS::GLOBAL_ADDRESS));

  // Create store of malloc to new global
  IRB.CreateStore(MallocPtr, SwLDS);

  // Create calls to __asan_poison_region to poison redzones.
  poisonRedzones(Func, MallocPtr);

  // Create branch to PrevEntryBlock
  IRB.CreateBr(PrevEntryBlock);

  // Create wave-group barrier at the starting of Previous entry block
  Type *Int1Ty = IRB.getInt1Ty();
  IRB.SetInsertPoint(PrevEntryBlock, PrevEntryBlock->begin());
  auto *XYZCondPhi = IRB.CreatePHI(Int1Ty, 2, "xyzCond");
  XYZCondPhi->addIncoming(IRB.getInt1(0), WIdBlock);
  XYZCondPhi->addIncoming(IRB.getInt1(1), MallocBlock);

  IRB.CreateIntrinsic(Intrinsic::amdgcn_s_barrier, {});

  // Load malloc pointer from Sw LDS.
  Value *LoadMallocPtr =
      IRB.CreateLoad(IRB.getPtrTy(AMDGPUAS::GLOBAL_ADDRESS), SwLDS);

  // Replace All uses of LDS globals with new LDS pointers.
  replaceKernelLDSAccesses(Func);

  // Replace Memory Operations on LDS with corresponding
  // global memory pointers.
  translateLDSMemoryOperationsToGlobalMemory(Func, LoadMallocPtr,
                                             LDSInstructions);

  auto *CondFreeBlock = BasicBlock::Create(Ctx, "CondFree", Func);
  auto *FreeBlock = BasicBlock::Create(Ctx, "Free", Func);
  auto *EndBlock = BasicBlock::Create(Ctx, "End", Func);
  for (BasicBlock &BB : *Func) {
    if (!BB.empty()) {
      if (ReturnInst *RI = dyn_cast<ReturnInst>(&BB.back())) {
        RI->eraseFromParent();
        IRB.SetInsertPoint(&BB, BB.end());
        IRB.CreateBr(CondFreeBlock);
      }
    }
  }

  // Cond Free Block
  IRB.SetInsertPoint(CondFreeBlock, CondFreeBlock->begin());
  IRB.CreateIntrinsic(Intrinsic::amdgcn_s_barrier, {});
  IRB.CreateCondBr(XYZCondPhi, FreeBlock, EndBlock);

  // Free Block
  IRB.SetInsertPoint(FreeBlock, FreeBlock->begin());

  // Free the previously allocate device global memory.
  FunctionCallee AsanFreeFunc = M.getOrInsertFunction(
      StringRef("__asan_free_impl"),
      FunctionType::get(IRB.getVoidTy(), {Int64Ty, Int64Ty}, false));
  Value *ReturnAddr =
      IRB.CreateIntrinsic(Intrinsic::returnaddress, IRB.getInt32(0));
  Value *RAPToInt = IRB.CreatePtrToInt(ReturnAddr, Int64Ty);
  Value *MallocPtrToInt = IRB.CreatePtrToInt(LoadMallocPtr, Int64Ty);
  IRB.CreateCall(AsanFreeFunc, {MallocPtrToInt, RAPToInt});

  IRB.CreateBr(EndBlock);

  // End Block
  IRB.SetInsertPoint(EndBlock, EndBlock->begin());
  IRB.CreateRetVoid();
  // Update the DomTree with corresponding links to basic blocks.
  DTU.applyUpdates({{DominatorTree::Insert, WIdBlock, MallocBlock},
                    {DominatorTree::Insert, MallocBlock, PrevEntryBlock},
                    {DominatorTree::Insert, CondFreeBlock, FreeBlock},
                    {DominatorTree::Insert, FreeBlock, EndBlock}});
}
```

<a name="block_8"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUSwLowerLDS.cpp (L985-1017) [<sup>↩</sup>](#ref-block_8)

```cpp
void AMDGPUSwLowerLDS::buildNonKernelLDSBaseTable(
    NonKernelLDSParameters &NKLDSParams) {
  // Base table will have single row, with elements of the row
  // placed as per kernel ID. Each element in the row corresponds
  // to addresss of "SW LDS" global of the kernel.
  auto &Kernels = NKLDSParams.OrderedKernels;
  if (Kernels.empty())
    return;
  Type *Int32Ty = IRB.getInt32Ty();
  const size_t NumberKernels = Kernels.size();
  ArrayType *AllKernelsOffsetsType =
      ArrayType::get(IRB.getPtrTy(AMDGPUAS::LOCAL_ADDRESS), NumberKernels);
  std::vector<Constant *> OverallConstantExprElts(NumberKernels);
  for (size_t i = 0; i < NumberKernels; i++) {
    Function *Func = Kernels[i];
    auto &LDSParams = FuncLDSAccessInfo.KernelToLDSParametersMap[Func];
    GlobalVariable *SwLDS = LDSParams.SwLDS;
    assert(SwLDS);
    Constant *GEPIdx[] = {ConstantInt::get(Int32Ty, 0)};
    Constant *GEP =
        ConstantExpr::getGetElementPtr(SwLDS->getType(), SwLDS, GEPIdx, true);
    OverallConstantExprElts[i] = GEP;
  }
  Constant *init =
      ConstantArray::get(AllKernelsOffsetsType, OverallConstantExprElts);
  NKLDSParams.LDSBaseTable = new GlobalVariable(
      M, AllKernelsOffsetsType, true, GlobalValue::InternalLinkage, init,
      "llvm.amdgcn.sw.lds.base.table", nullptr, GlobalValue::NotThreadLocal,
      AMDGPUAS::GLOBAL_ADDRESS);
  GlobalValue::SanitizerMetadata MD;
  MD.NoAddress = true;
  NKLDSParams.LDSBaseTable->setSanitizerMetadata(MD);
}
```

<a name="block_9"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUSwLowerLDS.cpp (L1019-1055) [<sup>↩</sup>](#ref-block_9)

```cpp
void AMDGPUSwLowerLDS::buildNonKernelLDSOffsetTable(
    NonKernelLDSParameters &NKLDSParams) {
  // Offset table will have multiple rows and columns.
  // Rows are assumed to be from 0 to (n-1). n is total number
  // of kernels accessing the LDS through non-kernels.
  // Each row will have m elements. m is the total number of
  // unique LDS globals accessed by non-kernels.
  // Each element in the row correspond to the address of
  // the replacement of LDS global done by that particular kernel.
  auto &Variables = NKLDSParams.OrdereLDSGlobals;
  auto &Kernels = NKLDSParams.OrderedKernels;
  if (Variables.empty() || Kernels.empty())
    return;
  const size_t NumberVariables = Variables.size();
  const size_t NumberKernels = Kernels.size();

  ArrayType *KernelOffsetsType =
      ArrayType::get(IRB.getPtrTy(AMDGPUAS::GLOBAL_ADDRESS), NumberVariables);

  ArrayType *AllKernelsOffsetsType =
      ArrayType::get(KernelOffsetsType, NumberKernels);
  std::vector<Constant *> overallConstantExprElts(NumberKernels);
  for (size_t i = 0; i < NumberKernels; i++) {
    Function *Func = Kernels[i];
    overallConstantExprElts[i] =
        getAddressesOfVariablesInKernel(Func, Variables);
  }
  Constant *Init =
      ConstantArray::get(AllKernelsOffsetsType, overallConstantExprElts);
  NKLDSParams.LDSOffsetTable = new GlobalVariable(
      M, AllKernelsOffsetsType, true, GlobalValue::InternalLinkage, Init,
      "llvm.amdgcn.sw.lds.offset.table", nullptr, GlobalValue::NotThreadLocal,
      AMDGPUAS::GLOBAL_ADDRESS);
  GlobalValue::SanitizerMetadata MD;
  MD.NoAddress = true;
  NKLDSParams.LDSOffsetTable->setSanitizerMetadata(MD);
}
```

<a name="block_10"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUSwLowerLDS.cpp (L1057-1101) [<sup>↩</sup>](#ref-block_10)

```cpp
void AMDGPUSwLowerLDS::lowerNonKernelLDSAccesses(
    Function *Func, SetVector<GlobalVariable *> &LDSGlobals,
    NonKernelLDSParameters &NKLDSParams) {
  // Replace LDS access in non-kernel with replacement queried from
  // Base table and offset from offset table.
  LLVM_DEBUG(dbgs() << "Sw LDS lowering, lower non-kernel access for : "
                    << Func->getName());
  auto InsertAt = Func->getEntryBlock().getFirstNonPHIOrDbgOrAlloca();
  IRB.SetInsertPoint(InsertAt);

  // Get LDS memory instructions.
  SetVector<Instruction *> LDSInstructions;
  getLDSMemoryInstructions(Func, LDSInstructions);

  auto *KernelId = IRB.CreateIntrinsic(Intrinsic::amdgcn_lds_kernel_id, {});
  GlobalVariable *LDSBaseTable = NKLDSParams.LDSBaseTable;
  GlobalVariable *LDSOffsetTable = NKLDSParams.LDSOffsetTable;
  auto &OrdereLDSGlobals = NKLDSParams.OrdereLDSGlobals;
  Value *BaseGEP = IRB.CreateInBoundsGEP(
      LDSBaseTable->getValueType(), LDSBaseTable, {IRB.getInt32(0), KernelId});
  Value *BaseLoad =
      IRB.CreateLoad(IRB.getPtrTy(AMDGPUAS::LOCAL_ADDRESS), BaseGEP);
  Value *LoadMallocPtr =
      IRB.CreateLoad(IRB.getPtrTy(AMDGPUAS::GLOBAL_ADDRESS), BaseLoad);

  for (GlobalVariable *GV : LDSGlobals) {
    const auto *GVIt = llvm::find(OrdereLDSGlobals, GV);
    assert(GVIt != OrdereLDSGlobals.end());
    uint32_t GVOffset = std::distance(OrdereLDSGlobals.begin(), GVIt);

    Value *OffsetGEP = IRB.CreateInBoundsGEP(
        LDSOffsetTable->getValueType(), LDSOffsetTable,
        {IRB.getInt32(0), KernelId, IRB.getInt32(GVOffset)});
    Value *OffsetLoad =
        IRB.CreateLoad(IRB.getPtrTy(AMDGPUAS::GLOBAL_ADDRESS), OffsetGEP);
    Value *Offset = IRB.CreateLoad(IRB.getInt32Ty(), OffsetLoad);
    Value *BasePlusOffset =
        IRB.CreateInBoundsGEP(IRB.getInt8Ty(), BaseLoad, {Offset});
    LLVM_DEBUG(dbgs() << "Sw LDS Lowering, Replace non-kernel LDS for "
                      << GV->getName());
    replacesUsesOfGlobalInFunction(Func, GV, BasePlusOffset);
  }
  translateLDSMemoryOperationsToGlobalMemory(Func, LoadMallocPtr,
                                             LDSInstructions);
}
```

<a name="block_11"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUSwLowerLDS.cpp (L1146-1296) [<sup>↩</sup>](#ref-block_11)

```cpp
bool AMDGPUSwLowerLDS::run() {
  bool Changed = false;

  CallGraph CG = CallGraph(M);

  Changed |= eliminateConstantExprUsesOfLDSFromAllInstructions(M);

  // Get all the direct and indirect access of LDS for all the kernels.
  LDSUsesInfoTy LDSUsesInfo = getTransitiveUsesOfLDS(CG, M);

  // Flag to decide whether to lower all the LDS accesses
  // based on sanitize_address attribute.
  bool LowerAllLDS = hasFnWithSanitizeAddressAttr(LDSUsesInfo.direct_access) ||
                     hasFnWithSanitizeAddressAttr(LDSUsesInfo.indirect_access);

  if (!LowerAllLDS)
    return Changed;

  // Utility to group LDS access into direct, indirect, static and dynamic.
  auto PopulateKernelStaticDynamicLDS = [&](FunctionVariableMap &LDSAccesses,
                                            bool DirectAccess) {
    for (auto &K : LDSAccesses) {
      Function *F = K.first;
      if (!F || K.second.empty())
        continue;

      assert(isKernelLDS(F));

      // Only inserts if key isn't already in the map.
      FuncLDSAccessInfo.KernelToLDSParametersMap.insert(
          {F, KernelLDSParameters()});

      auto &LDSParams = FuncLDSAccessInfo.KernelToLDSParametersMap[F];
      if (!DirectAccess)
        FuncLDSAccessInfo.KernelsWithIndirectLDSAccess.insert(F);
      for (GlobalVariable *GV : K.second) {
        if (!DirectAccess) {
          if (AMDGPU::isDynamicLDS(*GV))
            LDSParams.IndirectAccess.DynamicLDSGlobals.insert(GV);
          else
            LDSParams.IndirectAccess.StaticLDSGlobals.insert(GV);
          FuncLDSAccessInfo.AllNonKernelLDSAccess.insert(GV);
        } else {
          if (AMDGPU::isDynamicLDS(*GV))
            LDSParams.DirectAccess.DynamicLDSGlobals.insert(GV);
          else
            LDSParams.DirectAccess.StaticLDSGlobals.insert(GV);
        }
      }
    }
  };

  PopulateKernelStaticDynamicLDS(LDSUsesInfo.direct_access, true);
  PopulateKernelStaticDynamicLDS(LDSUsesInfo.indirect_access, false);

  // Get address sanitizer scale.
  initAsanInfo();

  for (auto &K : FuncLDSAccessInfo.KernelToLDSParametersMap) {
    Function *Func = K.first;
    auto &LDSParams = FuncLDSAccessInfo.KernelToLDSParametersMap[Func];
    if (LDSParams.DirectAccess.StaticLDSGlobals.empty() &&
        LDSParams.DirectAccess.DynamicLDSGlobals.empty() &&
        LDSParams.IndirectAccess.StaticLDSGlobals.empty() &&
        LDSParams.IndirectAccess.DynamicLDSGlobals.empty()) {
      Changed = false;
    } else {
      removeFnAttrFromReachable(
          CG, Func,
          {"amdgpu-no-workitem-id-x", "amdgpu-no-workitem-id-y",
           "amdgpu-no-workitem-id-z", "amdgpu-no-heap-ptr"});
      if (!LDSParams.IndirectAccess.StaticLDSGlobals.empty() ||
          !LDSParams.IndirectAccess.DynamicLDSGlobals.empty())
        removeFnAttrFromReachable(CG, Func, {"amdgpu-no-lds-kernel-id"});
      reorderStaticDynamicIndirectLDSSet(LDSParams);
      buildSwLDSGlobal(Func);
      buildSwDynLDSGlobal(Func);
      populateSwMetadataGlobal(Func);
      populateSwLDSAttributeAndMetadata(Func);
      populateLDSToReplacementIndicesMap(Func);
      DomTreeUpdater DTU(DTCallback(*Func),
                         DomTreeUpdater::UpdateStrategy::Lazy);
      lowerKernelLDSAccesses(Func, DTU);
      Changed = true;
    }
  }

  // Get the Uses of LDS from non-kernels.
  getUsesOfLDSByNonKernels();

  // Get non-kernels with LDS ptr as argument and called by kernels.
  getNonKernelsWithLDSArguments(CG);

  // Lower LDS accesses in non-kernels.
  if (!FuncLDSAccessInfo.NonKernelToLDSAccessMap.empty() ||
      !FuncLDSAccessInfo.NonKernelsWithLDSArgument.empty()) {
    NonKernelLDSParameters NKLDSParams;
    NKLDSParams.OrderedKernels = getOrderedIndirectLDSAccessingKernels(
        FuncLDSAccessInfo.KernelsWithIndirectLDSAccess);
    NKLDSParams.OrdereLDSGlobals = getOrderedNonKernelAllLDSGlobals(
        FuncLDSAccessInfo.AllNonKernelLDSAccess);
    buildNonKernelLDSBaseTable(NKLDSParams);
    buildNonKernelLDSOffsetTable(NKLDSParams);
    for (auto &K : FuncLDSAccessInfo.NonKernelToLDSAccessMap) {
      Function *Func = K.first;
      DenseSet<GlobalVariable *> &LDSGlobals = K.second;
      SetVector<GlobalVariable *> OrderedLDSGlobals = sortByName(
          std::vector<GlobalVariable *>(LDSGlobals.begin(), LDSGlobals.end()));
      lowerNonKernelLDSAccesses(Func, OrderedLDSGlobals, NKLDSParams);
    }
    for (Function *Func : FuncLDSAccessInfo.NonKernelsWithLDSArgument) {
      auto &K = FuncLDSAccessInfo.NonKernelToLDSAccessMap;
      if (K.contains(Func))
        continue;
      SetVector<llvm::GlobalVariable *> Vec;
      lowerNonKernelLDSAccesses(Func, Vec, NKLDSParams);
    }
    Changed = true;
  }

  if (!Changed)
    return Changed;

  for (auto &GV : make_early_inc_range(M.globals())) {
    if (AMDGPU::isLDSVariableToLower(GV)) {
      // probably want to remove from used lists
      GV.removeDeadConstantUsers();
      if (GV.use_empty())
        GV.eraseFromParent();
    }
  }

  if (AsanInstrumentLDS) {
    SmallVector<InterestingMemoryOperand, 16> OperandsToInstrument;
    for (Instruction *Inst : AsanInfo.Instructions) {
      SmallVector<InterestingMemoryOperand, 1> InterestingOperands;
      getInterestingMemoryOperands(M, Inst, InterestingOperands);
      llvm::append_range(OperandsToInstrument, InterestingOperands);
    }
    for (auto &Operand : OperandsToInstrument) {
      Value *Addr = Operand.getPtr();
      instrumentAddress(M, IRB, Operand.getInsn(), Operand.getInsn(), Addr,
                        Operand.Alignment.valueOrOne(), Operand.TypeStoreSize,
                        Operand.IsWrite, nullptr, false, false, AsanInfo.Scale,
                        AsanInfo.Offset);
      Changed = true;
    }
  }

  return Changed;
}
```

<a name="block_12"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUSwLowerLDS.cpp (L1322-1324) [<sup>↩</sup>](#ref-block_12)

```cpp
  // instrumented the IR. Return early if the flag is not present.
  if (!M.getModuleFlag("nosanitize_address"))
    return false;
```

