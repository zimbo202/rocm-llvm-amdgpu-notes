# AMDGPULowerBufferFatPointers.cpp 代码功能详解

## 1）主要功能概括

<a name="ref-block_0"></a>该pass的主要功能是**将buffer fat pointer（地址空间7）的操作降低（lower）为buffer resource（地址空间8）的操作**，这是AMDGPU后端正确代码生成所必需的转换。 llvm-project:9-11[<sup>↗</sup>](#block_0) 

**核心作用**：
<a name="ref-block_1"></a>- Buffer fat pointer是一个160位的指针，由128位的buffer descriptor和32位的偏移量组成 llvm-project:15-19[<sup>↗</sup>](#block_1) 
<a name="ref-block_2"></a>- 由于其非2的幂次大小，这些fat pointer无法在转换到MIR阶段时存在，因此需要此pass进行转换 llvm-project:26-29[<sup>↗</sup>](#block_2) 

<a name="ref-block_4"></a>**转换效果**：将 `ptr addrspace(7) %x` 转换为 `ptr addrspace(8) %x.rsrc` 和 `i32 %x.off` 两个独立的部分，并在需要时组合为结构体 `{ptr addrspace(8), i32}`。 llvm-project:39-42[<sup>↗</sup>](#block_4) 

## 2）主要功能步骤提取

<a name="ref-block_5"></a>该pass按以下**三个主要阶段**顺序执行： llvm-project:44-46[<sup>↗</sup>](#block_5) 

### 阶段1：重写loads/stores并处理memcpy
- **类名**：`StoreFatPtrsAsIntsAndExpandMemcpyVisitor`
<a name="ref-block_6"></a>- **功能**：将所有 `ptr addrspace(7)` 的加载和存储重写为使用 `i160` 的操作 llvm-project:48-52[<sup>↗</sup>](#block_6) 

### 阶段2：Buffer内容类型合法化
- **类名**：`LegalizeBufferContentTypesVisitor`
<a name="ref-block_9"></a>- **功能**：将buffer内容的类型转换为buffer intrinsics能够支持的合法类型（最大128位） llvm-project:73-82[<sup>↗</sup>](#block_9) 

### 阶段3：类型重映射和指针结构分割
<a name="ref-block_11"></a>- **类型重映射**：使用 `ValueMapper` 将buffer fat pointer类型转换为对应的结构体类型 `{ptr addrspace(8), i32}` llvm-project:95-99[<sup>↗</sup>](#block_11) 
<a name="ref-block_12"></a>- **分割指针结构**：通过 `SplitPtrStructs` visitor实现，将操作分解为资源部分和偏移部分的操作 llvm-project:115-120[<sup>↗</sup>](#block_12) 

## 3）各步骤/子功能的具体描述分析

### 阶段1详解：StoreFatPtrsAsIntsAndExpandMemcpyVisitor

**核心转换逻辑**：
- 将包含 `ptr addrspace(7)` 的类型重写为用 `i160` 替代的类型
<a name="ref-block_7"></a>- 存储时使用 `ptrtoint` 将指针转为i160，加载时使用 `inttoptr` 转回 llvm-project:53-57[<sup>↗</sup>](#block_7) 

**关键方法**：
<a name="ref-block_16"></a>- `fatPtrsToInts()`: 将buffer fat pointer递归转换为整数以便存储 llvm-project:466-502[<sup>↗</sup>](#block_16) 
<a name="ref-block_17"></a>- `intsToFatPtrs()`: 将整数转换回buffer fat pointer llvm-project:504-535[<sup>↗</sup>](#block_17) 
<a name="ref-block_15"></a>- 处理alloca、load、store、GEP等指令 llvm-project:454-462[<sup>↗</sup>](#block_15) 

**memcpy处理**：
<a name="ref-block_18"></a>- 将涉及buffer fat pointer的memcpy/memmove/memset展开为循环 llvm-project:611-650[<sup>↗</sup>](#block_18) 

### 阶段2详解：LegalizeBufferContentTypesVisitor

<a name="ref-block_10"></a>**合法化策略**： llvm-project:83-93[<sup>↗</sup>](#block_10) 
1. 将数组转换为向量（如果可能）
2. 将聚合类型的加载/存储拆分为各组件的操作
3. 零扩展填充至完整字节数
4. 将不支持的类型（如i96、i256）转换为支持的类型（如`<3 x i32>`、`<8 x i32>`）
5. 将过长的值拆分为多个操作

**核心转换方法**：
<a name="ref-block_19"></a>- `scalarArrayTypeAsVector()`: 将标量数组转换为向量 llvm-project:737-749[<sup>↗</sup>](#block_19) 
<a name="ref-block_20"></a>- `legalNonAggregateFor()`: 确定非聚合类型的合法表示 llvm-project:779-808[<sup>↗</sup>](#block_20) 
<a name="ref-block_21"></a>- `getVecSlices()`: 将向量切片以满足128位限制 llvm-project:871-907[<sup>↗</sup>](#block_21) 

### 阶段3详解：类型重映射与指针分割

**类型重映射系统**：
<a name="ref-block_14"></a>- `BufferFatPtrToStructTypeMap`: 将 `ptr addrspace(7)` 映射为 `{ptr addrspace(8), i32}` llvm-project:296-305[<sup>↗</sup>](#block_14) 
<a name="ref-block_22"></a>- `FatPtrConstMaterializer`: 处理常量的重映射 llvm-project:1198-1220[<sup>↗</sup>](#block_22) 

<a name="ref-block_23"></a>**SplitPtrStructs核心功能**： llvm-project:1287-1382[<sup>↗</sup>](#block_23) 

<a name="ref-block_24"></a>1. **指针部分获取**：`getPtrParts()` 方法提取或创建资源和偏移部分 llvm-project:1394-1424[<sup>↗</sup>](#block_24) 

2. **指令处理**：为各种指令实现分割逻辑
<a name="ref-block_28"></a>   - **内存操作**：load/store/atomic转换为buffer intrinsic调用 llvm-project:1677-1795[<sup>↗</sup>](#block_28) 
<a name="ref-block_29"></a>   - **GEP**：将偏移累加到offset部分，资源部分保持不变 llvm-project:1871-1923[<sup>↗</sup>](#block_29) 
<a name="ref-block_30"></a>   - **ptrtoint/inttoptr**：在整数和分割表示之间转换 llvm-project:1925-2000[<sup>↗</sup>](#block_30) 
<a name="ref-block_31"></a>   - **向量操作**：extractelement、insertelement、shufflevector分别处理两个部分 llvm-project:2094-2152[<sup>↗</sup>](#block_31) 

<a name="ref-block_13"></a>3. **条件优化**：特殊处理PHI和select指令 llvm-project:143-175[<sup>↗</sup>](#block_13) 
<a name="ref-block_25"></a>   - `getPossibleRsrcRoots()`: 收集可能的资源来源 llvm-project:1442-1468[<sup>↗</sup>](#block_25) 
<a name="ref-block_26"></a>   - `processConditionals()`: 如果所有路径使用相同资源，则避免不必要的PHI节点 llvm-project:1470-1578[<sup>↗</sup>](#block_26) 

<a name="ref-block_27"></a>4. **清理替换**：`killAndReplaceSplitInstructions()` 删除原指令并构建必要的结构体 llvm-project:1580-1644[<sup>↗</sup>](#block_27) 

## 4）步骤/子功能之间的关系

**执行顺序和依赖关系**：

<a name="ref-block_8"></a>1. **阶段1 → 阶段2**：首先处理内存操作确保fat pointer不会以原始形式在内存中传递，为后续类型合法化做准备 llvm-project:59-64[<sup>↗</sup>](#block_8) 

2. **阶段2 → 阶段3**：类型合法化确保所有buffer操作的数据类型符合intrinsic要求，然后才能进行类型重映射 llvm-project:2515-2527 

3. **阶段3内部关系**：
   - 类型重映射先执行，创建新函数签名 llvm-project:2536-2560 
   - 指针分割visitor遍历所有指令，构建资源和偏移映射 llvm-project:2561-2563 

<a name="ref-block_32"></a>**整体流程**： llvm-project:2455-2576[<sup>↗</sup>](#block_32) 
```
常量表达式展开 → 内存操作重写 → 类型合法化 → 类型重映射 → 指针分割 → 条件优化 → 指令清理
```

**关键设计决策**：
<a name="ref-block_3"></a>- 分离资源和偏移部分能提高性能，因为很多操作只修改偏移而保持资源不变 llvm-project:31-37[<sup>↗</sup>](#block_3) 
- PHI优化避免了不必要的寄存器压力和可能的waterfall循环 llvm-project:143-152 

## Notes

该pass是AMDGPU后端处理SPIR-V kernel中buffer资源的关键组件，通过三阶段转换实现了从高级的fat pointer抽象到硬件支持的buffer intrinsic的降低过程。整个设计平衡了正确性、性能和IR有效性要求。


### Citations


<a name="block_0"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPULowerBufferFatPointers.cpp (L9-11) [<sup>↩</sup>](#ref-block_0)

```cpp
// This pass lowers operations on buffer fat pointers (addrspace 7) to
// operations on buffer resources (addrspace 8) and is needed for correct
// codegen.
```

<a name="block_1"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPULowerBufferFatPointers.cpp (L15-19) [<sup>↩</sup>](#ref-block_1)

```cpp
// Address space 7 (the buffer fat pointer) is a 160-bit pointer that consists
// of a 128-bit buffer descriptor and a 32-bit offset into that descriptor.
// The buffer resource part needs to be it needs to be a "raw" buffer resource
// (it must have a stride of 0 and bounds checks must be in raw buffer mode
// or disabled).
```

<a name="block_2"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPULowerBufferFatPointers.cpp (L26-29) [<sup>↩</sup>](#ref-block_2)

```cpp
// However, because of their non-power-of-2 size, these fat pointers cannot be
// present during translation to MIR (though this restriction may be lifted
// during the transition to GlobalISel). Therefore, this pass is needed in order
// to correctly implement these fat pointers.
```

<a name="block_3"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPULowerBufferFatPointers.cpp (L31-37) [<sup>↩</sup>](#ref-block_3)

```cpp
// The resource intrinsics take the resource part (the address space 8 pointer)
// and the offset part (the 32-bit integer) as separate arguments. In addition,
// many users of these buffers manipulate the offset while leaving the resource
// part alone. For these reasons, we want to typically separate the resource
// and offset parts into separate variables, but combine them together when
// encountering cases where this is required, such as by inserting these values
// into aggretates or moving them to memory.
```

<a name="block_4"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPULowerBufferFatPointers.cpp (L39-42) [<sup>↩</sup>](#ref-block_4)

```cpp
// Therefore, at a high level, `ptr addrspace(7) %x` becomes `ptr addrspace(8)
// %x.rsrc` and `i32 %x.off`, which will be combined into `{ptr addrspace(8),
// i32} %x = {%x.rsrc, %x.off}` if needed. Similarly, `vector<Nxp7>` becomes
// `{vector<Nxp8>, vector<Nxi32 >}` and its component parts.
```

<a name="block_5"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPULowerBufferFatPointers.cpp (L44-46) [<sup>↩</sup>](#ref-block_5)

```cpp
// # Implementation
//
// This pass proceeds in three main phases:
```

<a name="block_6"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPULowerBufferFatPointers.cpp (L48-52) [<sup>↩</sup>](#ref-block_6)

```cpp
// ## Rewriting loads and stores of p7 and memcpy()-like handling
//
// The first phase is to rewrite away all loads and stors of `ptr addrspace(7)`,
// including aggregates containing such pointers, to ones that use `i160`. This
// is handled by `StoreFatPtrsAsIntsAndExpandMemcpyVisitor` , which visits
```

<a name="block_7"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPULowerBufferFatPointers.cpp (L53-57) [<sup>↩</sup>](#ref-block_7)

```cpp
// loads, stores, and allocas and, if the loaded or stored type contains `ptr
// addrspace(7)`, rewrites that type to one where the p7s are replaced by i160s,
// copying other parts of aggregates as needed. In the case of a store, each
// pointer is `ptrtoint`d to i160 before storing, and load integers are
// `inttoptr`d back. This same transformation is applied to vectors of pointers.
```

<a name="block_8"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPULowerBufferFatPointers.cpp (L59-64) [<sup>↩</sup>](#ref-block_8)

```cpp
// Such a transformation allows the later phases of the pass to not need
// to handle buffer fat pointers moving to and from memory, where we load
// have to handle the incompatibility between a `{Nxp8, Nxi32}` representation
// and `Nxi60` directly. Instead, that transposing action (where the vectors
// of resources and vectors of offsets are concatentated before being stored to
// memory) are handled through implementing `inttoptr` and `ptrtoint` only.
```

<a name="block_9"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPULowerBufferFatPointers.cpp (L73-82) [<sup>↩</sup>](#ref-block_9)

```cpp
// ## Buffer contents type legalization
//
// The underlying buffer intrinsics only support types up to 128 bits long,
// and don't support complex types. If buffer operations were
// standard pointer operations that could be represented as MIR-level loads,
// this would be handled by the various legalization schemes in instruction
// selection. However, because we have to do the conversion from `load` and
// `store` to intrinsics at LLVM IR level, we must perform that legalization
// ourselves.
//
```

<a name="block_10"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPULowerBufferFatPointers.cpp (L83-93) [<sup>↩</sup>](#ref-block_10)

```cpp
// This involves a combination of
// - Converting arrays to vectors where possible
// - Otherwise, splitting loads and stores of aggregates into loads/stores of
//   each component.
// - Zero-extending things to fill a whole number of bytes
// - Casting values of types that don't neatly correspond to supported machine
// value
//   (for example, an i96 or i256) into ones that would work (
//    like <3 x i32> and <8 x i32>, respectively)
// - Splitting values that are too long (such as aforementioned <8 x i32>) into
//   multiple operations.
```

<a name="block_11"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPULowerBufferFatPointers.cpp (L95-99) [<sup>↩</sup>](#ref-block_11)

```cpp
// ## Type remapping
//
// We use a `ValueMapper` to mangle uses of [vectors of] buffer fat pointers
// to the corresponding struct type, which has a resource part and an offset
// part.
```

<a name="block_12"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPULowerBufferFatPointers.cpp (L115-120) [<sup>↩</sup>](#ref-block_12)

```cpp
// ## Splitting pointer structs
//
// The meat of this pass consists of defining semantics for operations that
// produce or consume [vectors of] buffer fat pointers in terms of their
// resource and offset parts. This is accomplished throgh the `SplitPtrStructs`
// visitor.
```

<a name="block_13"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPULowerBufferFatPointers.cpp (L143-175) [<sup>↩</sup>](#ref-block_13)

```cpp
// ### Conditionals
//
// PHI nodes are initially given resource parts via `extractvalue`. However,
// this is not an efficient rewrite of such nodes, as, in most cases, the
// resource part in a conditional or loop remains constant throughout the loop
// and only the offset varies. Failing to optimize away these constant resources
// would cause additional registers to be sent around loops and might lead to
// waterfall loops being generated for buffer operations due to the
// "non-uniform" resource argument.
//
// Therefore, after all instructions have been visited, the pointer splitter
// post-processes all encountered conditionals. Given a PHI node or select,
// getPossibleRsrcRoots() collects all values that the resource parts of that
// conditional's input could come from as well as collecting all conditional
// instructions encountered during the search. If, after filtering out the
// initial node itself, the set of encountered conditionals is a subset of the
// potential roots and there is a single potential resource that isn't in the
// conditional set, that value is the only possible value the resource argument
// could have throughout the control flow.
//
// If that condition is met, then a PHI node can have its resource part changed
// to the singleton value and then be replaced by a PHI on the offsets.
// Otherwise, each PHI node is split into two, one for the resource part and one
// for the offset part, which replace the temporary `extractvalue` instructions
// that were added during the first pass.
//
// Similar logic applies to `select`, where
// `%z = select i1 %cond, %cond, ptr addrspace(7) %x, ptr addrspace(7) %y`
// can be split into `%z.rsrc = %x.rsrc` and
// `%z.off = select i1 %cond, ptr i32 %x.off, i32 %y.off`
// if both `%x` and `%y` have the same resource part, but two `select`
// operations will be needed if they do not.
//
```

<a name="block_14"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPULowerBufferFatPointers.cpp (L296-305) [<sup>↩</sup>](#ref-block_14)

```cpp
/// Remap ptr addrspace(7) to {ptr addrspace(8), i32} (the resource and offset
/// parts of the pointer) so that we can easily rewrite operations on these
/// values that aren't loading them from or storing them to memory.
class BufferFatPtrToStructTypeMap : public BufferFatPtrTypeLoweringBase {
  using BufferFatPtrTypeLoweringBase::BufferFatPtrTypeLoweringBase;

protected:
  Type *remapScalar(PointerType *PT) override;
  Type *remapVector(VectorType *VT) override;
};
```

<a name="block_15"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPULowerBufferFatPointers.cpp (L454-462) [<sup>↩</sup>](#ref-block_15)

```cpp
  bool visitAllocaInst(AllocaInst &I);
  bool visitLoadInst(LoadInst &LI);
  bool visitStoreInst(StoreInst &SI);
  bool visitGetElementPtrInst(GetElementPtrInst &I);

  bool visitMemCpyInst(MemCpyInst &MCI);
  bool visitMemMoveInst(MemMoveInst &MMI);
  bool visitMemSetInst(MemSetInst &MSI);
  bool visitMemSetPatternInst(MemSetPatternInst &MSPI);
```

<a name="block_16"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPULowerBufferFatPointers.cpp (L466-502) [<sup>↩</sup>](#ref-block_16)

```cpp
Value *StoreFatPtrsAsIntsAndExpandMemcpyVisitor::fatPtrsToInts(
    Value *V, Type *From, Type *To, const Twine &Name) {
  if (From == To)
    return V;
  ValueToValueMapTy::iterator Find = ConvertedForStore.find(V);
  if (Find != ConvertedForStore.end())
    return Find->second;
  if (isBufferFatPtrOrVector(From)) {
    Value *Cast = IRB.CreatePtrToInt(V, To, Name + ".int");
    ConvertedForStore[V] = Cast;
    return Cast;
  }
  if (From->getNumContainedTypes() == 0)
    return V;
  // Structs, arrays, and other compound types.
  Value *Ret = PoisonValue::get(To);
  if (auto *AT = dyn_cast<ArrayType>(From)) {
    Type *FromPart = AT->getArrayElementType();
    Type *ToPart = cast<ArrayType>(To)->getElementType();
    for (uint64_t I = 0, E = AT->getArrayNumElements(); I < E; ++I) {
      Value *Field = IRB.CreateExtractValue(V, I);
      Value *NewField =
          fatPtrsToInts(Field, FromPart, ToPart, Name + "." + Twine(I));
      Ret = IRB.CreateInsertValue(Ret, NewField, I);
    }
  } else {
    for (auto [Idx, FromPart, ToPart] :
         enumerate(From->subtypes(), To->subtypes())) {
      Value *Field = IRB.CreateExtractValue(V, Idx);
      Value *NewField =
          fatPtrsToInts(Field, FromPart, ToPart, Name + "." + Twine(Idx));
      Ret = IRB.CreateInsertValue(Ret, NewField, Idx);
    }
  }
  ConvertedForStore[V] = Ret;
  return Ret;
}
```

<a name="block_17"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPULowerBufferFatPointers.cpp (L504-535) [<sup>↩</sup>](#ref-block_17)

```cpp
Value *StoreFatPtrsAsIntsAndExpandMemcpyVisitor::intsToFatPtrs(
    Value *V, Type *From, Type *To, const Twine &Name) {
  if (From == To)
    return V;
  if (isBufferFatPtrOrVector(To)) {
    Value *Cast = IRB.CreateIntToPtr(V, To, Name + ".ptr");
    return Cast;
  }
  if (From->getNumContainedTypes() == 0)
    return V;
  // Structs, arrays, and other compound types.
  Value *Ret = PoisonValue::get(To);
  if (auto *AT = dyn_cast<ArrayType>(From)) {
    Type *FromPart = AT->getArrayElementType();
    Type *ToPart = cast<ArrayType>(To)->getElementType();
    for (uint64_t I = 0, E = AT->getArrayNumElements(); I < E; ++I) {
      Value *Field = IRB.CreateExtractValue(V, I);
      Value *NewField =
          intsToFatPtrs(Field, FromPart, ToPart, Name + "." + Twine(I));
      Ret = IRB.CreateInsertValue(Ret, NewField, I);
    }
  } else {
    for (auto [Idx, FromPart, ToPart] :
         enumerate(From->subtypes(), To->subtypes())) {
      Value *Field = IRB.CreateExtractValue(V, Idx);
      Value *NewField =
          intsToFatPtrs(Field, FromPart, ToPart, Name + "." + Twine(Idx));
      Ret = IRB.CreateInsertValue(Ret, NewField, Idx);
    }
  }
  return Ret;
}
```

<a name="block_18"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPULowerBufferFatPointers.cpp (L611-650) [<sup>↩</sup>](#ref-block_18)

```cpp
bool StoreFatPtrsAsIntsAndExpandMemcpyVisitor::visitMemCpyInst(
    MemCpyInst &MCI) {
  // TODO: Allow memcpy.p7.p3 as a synonym for the direct-to-LDS copy, which'll
  // need loop expansion here.
  if (MCI.getSourceAddressSpace() != AMDGPUAS::BUFFER_FAT_POINTER &&
      MCI.getDestAddressSpace() != AMDGPUAS::BUFFER_FAT_POINTER)
    return false;
  llvm::expandMemCpyAsLoop(&MCI,
                           TM->getTargetTransformInfo(*MCI.getFunction()));
  MCI.eraseFromParent();
  return true;
}

bool StoreFatPtrsAsIntsAndExpandMemcpyVisitor::visitMemMoveInst(
    MemMoveInst &MMI) {
  if (MMI.getSourceAddressSpace() != AMDGPUAS::BUFFER_FAT_POINTER &&
      MMI.getDestAddressSpace() != AMDGPUAS::BUFFER_FAT_POINTER)
    return false;
  reportFatalUsageError(
      "memmove() on buffer descriptors is not implemented because pointer "
      "comparison on buffer descriptors isn't implemented\n");
}

bool StoreFatPtrsAsIntsAndExpandMemcpyVisitor::visitMemSetInst(
    MemSetInst &MSI) {
  if (MSI.getDestAddressSpace() != AMDGPUAS::BUFFER_FAT_POINTER)
    return false;
  llvm::expandMemSetAsLoop(&MSI);
  MSI.eraseFromParent();
  return true;
}

bool StoreFatPtrsAsIntsAndExpandMemcpyVisitor::visitMemSetPatternInst(
    MemSetPatternInst &MSPI) {
  if (MSPI.getDestAddressSpace() != AMDGPUAS::BUFFER_FAT_POINTER)
    return false;
  llvm::expandMemSetPatternAsLoop(&MSPI);
  MSPI.eraseFromParent();
  return true;
}
```

<a name="block_19"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPULowerBufferFatPointers.cpp (L737-749) [<sup>↩</sup>](#ref-block_19)

```cpp
Type *LegalizeBufferContentTypesVisitor::scalarArrayTypeAsVector(Type *T) {
  ArrayType *AT = dyn_cast<ArrayType>(T);
  if (!AT)
    return T;
  Type *ET = AT->getElementType();
  if (!ET->isSingleValueType() || isa<VectorType>(ET))
    reportFatalUsageError("loading non-scalar arrays from buffer fat pointers "
                          "should have recursed");
  if (!DL.typeSizeEqualsStoreSize(AT))
    reportFatalUsageError(
        "loading padded arrays from buffer fat pinters should have recursed");
  return FixedVectorType::get(ET, AT->getNumElements());
}
```

<a name="block_20"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPULowerBufferFatPointers.cpp (L779-808) [<sup>↩</sup>](#ref-block_20)

```cpp
Type *LegalizeBufferContentTypesVisitor::legalNonAggregateFor(Type *T) {
  TypeSize Size = DL.getTypeStoreSizeInBits(T);
  // Implicitly zero-extend to the next byte if needed
  if (!DL.typeSizeEqualsStoreSize(T))
    T = IRB.getIntNTy(Size.getFixedValue());
  Type *ElemTy = T->getScalarType();
  if (isa<PointerType, ScalableVectorType>(ElemTy)) {
    // Pointers are always big enough, and we'll let scalable vectors through to
    // fail in codegen.
    return T;
  }
  unsigned ElemSize = DL.getTypeSizeInBits(ElemTy).getFixedValue();
  if (isPowerOf2_32(ElemSize) && ElemSize >= 16 && ElemSize <= 128) {
    // [vectors of] anything that's 16/32/64/128 bits can be cast and split into
    // legal buffer operations.
    return T;
  }
  Type *BestVectorElemType = nullptr;
  if (Size.isKnownMultipleOf(32))
    BestVectorElemType = IRB.getInt32Ty();
  else if (Size.isKnownMultipleOf(16))
    BestVectorElemType = IRB.getInt16Ty();
  else
    BestVectorElemType = IRB.getInt8Ty();
  unsigned NumCastElems =
      Size.getFixedValue() / BestVectorElemType->getIntegerBitWidth();
  if (NumCastElems == 1)
    return BestVectorElemType;
  return FixedVectorType::get(BestVectorElemType, NumCastElems);
}
```

<a name="block_21"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPULowerBufferFatPointers.cpp (L871-907) [<sup>↩</sup>](#ref-block_21)

```cpp
void LegalizeBufferContentTypesVisitor::getVecSlices(
    Type *T, SmallVectorImpl<VecSlice> &Slices) {
  Slices.clear();
  auto *VT = dyn_cast<FixedVectorType>(T);
  if (!VT)
    return;

  uint64_t ElemBitWidth =
      DL.getTypeSizeInBits(VT->getElementType()).getFixedValue();

  uint64_t ElemsPer4Words = 128 / ElemBitWidth;
  uint64_t ElemsPer2Words = ElemsPer4Words / 2;
  uint64_t ElemsPerWord = ElemsPer2Words / 2;
  uint64_t ElemsPerShort = ElemsPerWord / 2;
  uint64_t ElemsPerByte = ElemsPerShort / 2;
  // If the elements evenly pack into 32-bit words, we can use 3-word stores,
  // such as for <6 x bfloat> or <3 x i32>, but we can't dot his for, for
  // example, <3 x i64>, since that's not slicing.
  uint64_t ElemsPer3Words = ElemsPerWord * 3;

  uint64_t TotalElems = VT->getNumElements();
  uint64_t Index = 0;
  auto TrySlice = [&](unsigned MaybeLen) {
    if (MaybeLen > 0 && Index + MaybeLen <= TotalElems) {
      VecSlice Slice{/*Index=*/Index, /*Length=*/MaybeLen};
      Slices.push_back(Slice);
      Index += MaybeLen;
      return true;
    }
    return false;
  };
  while (Index < TotalElems) {
    TrySlice(ElemsPer4Words) || TrySlice(ElemsPer3Words) ||
        TrySlice(ElemsPer2Words) || TrySlice(ElemsPerWord) ||
        TrySlice(ElemsPerShort) || TrySlice(ElemsPerByte);
  }
}
```

<a name="block_22"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPULowerBufferFatPointers.cpp (L1198-1220) [<sup>↩</sup>](#ref-block_22)

```cpp
/// Handle the remapping of ptr addrspace(7) constants.
class FatPtrConstMaterializer final : public ValueMaterializer {
  BufferFatPtrToStructTypeMap *TypeMap;
  // An internal mapper that is used to recurse into the arguments of constants.
  // While the documentation for `ValueMapper` specifies not to use it
  // recursively, examination of the logic in mapValue() shows that it can
  // safely be used recursively when handling constants, like it does in its own
  // logic.
  ValueMapper InternalMapper;

  Constant *materializeBufferFatPtrConst(Constant *C);

public:
  // UnderlyingMap is the value map this materializer will be filling.
  FatPtrConstMaterializer(BufferFatPtrToStructTypeMap *TypeMap,
                          ValueToValueMapTy &UnderlyingMap)
      : TypeMap(TypeMap),
        InternalMapper(UnderlyingMap, RF_None, TypeMap, this) {}
  ~FatPtrConstMaterializer() = default;

  Value *materialize(Value *V) override;
};
} // namespace
```

<a name="block_23"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPULowerBufferFatPointers.cpp (L1287-1382) [<sup>↩</sup>](#ref-block_23)

```cpp
namespace {
// The visitor returns the resource and offset parts for an instruction if they
// can be computed, or (nullptr, nullptr) for cases that don't have a meaningful
// value mapping.
class SplitPtrStructs : public InstVisitor<SplitPtrStructs, PtrParts> {
  ValueToValueMapTy RsrcParts;
  ValueToValueMapTy OffParts;

  // Track instructions that have been rewritten into a user of the component
  // parts of their ptr addrspace(7) input. Instructions that produced
  // ptr addrspace(7) parts should **not** be RAUW'd before being added to this
  // set, as that replacement will be handled in a post-visit step. However,
  // instructions that yield values that aren't fat pointers (ex. ptrtoint)
  // should RAUW themselves with new instructions that use the split parts
  // of their arguments during processing.
  DenseSet<Instruction *> SplitUsers;

  // Nodes that need a second look once we've computed the parts for all other
  // instructions to see if, for example, we really need to phi on the resource
  // part.
  SmallVector<Instruction *> Conditionals;
  // Temporary instructions produced while lowering conditionals that should be
  // killed.
  SmallVector<Instruction *> ConditionalTemps;

  // Subtarget info, needed for determining what cache control bits to set.
  const TargetMachine *TM;
  const GCNSubtarget *ST = nullptr;

  IRBuilder<InstSimplifyFolder> IRB;

  // Copy metadata between instructions if applicable.
  void copyMetadata(Value *Dest, Value *Src);

  // Get the resource and offset parts of the value V, inserting appropriate
  // extractvalue calls if needed.
  PtrParts getPtrParts(Value *V);

  // Given an instruction that could produce multiple resource parts (a PHI or
  // select), collect the set of possible instructions that could have provided
  // its resource parts  that it could have (the `Roots`) and the set of
  // conditional instructions visited during the search (`Seen`). If, after
  // removing the root of the search from `Seen` and `Roots`, `Seen` is a subset
  // of `Roots` and `Roots - Seen` contains one element, the resource part of
  // that element can replace the resource part of all other elements in `Seen`.
  void getPossibleRsrcRoots(Instruction *I, SmallPtrSetImpl<Value *> &Roots,
                            SmallPtrSetImpl<Value *> &Seen);
  void processConditionals();

  // If an instruction hav been split into resource and offset parts,
  // delete that instruction. If any of its uses have not themselves been split
  // into parts (for example, an insertvalue), construct the structure
  // that the type rewrites declared should be produced by the dying instruction
  // and use that.
  // Also, kill the temporary extractvalue operations produced by the two-stage
  // lowering of PHIs and conditionals.
  void killAndReplaceSplitInstructions(SmallVectorImpl<Instruction *> &Origs);

  void setAlign(CallInst *Intr, Align A, unsigned RsrcArgIdx);
  void insertPreMemOpFence(AtomicOrdering Order, SyncScope::ID SSID);
  void insertPostMemOpFence(AtomicOrdering Order, SyncScope::ID SSID);
  Value *handleMemoryInst(Instruction *I, Value *Arg, Value *Ptr, Type *Ty,
                          Align Alignment, AtomicOrdering Order,
                          bool IsVolatile, SyncScope::ID SSID);

public:
  SplitPtrStructs(const DataLayout &DL, LLVMContext &Ctx,
                  const TargetMachine *TM)
      : TM(TM), IRB(Ctx, InstSimplifyFolder(DL)) {}

  void processFunction(Function &F);

  PtrParts visitInstruction(Instruction &I);
  PtrParts visitLoadInst(LoadInst &LI);
  PtrParts visitStoreInst(StoreInst &SI);
  PtrParts visitAtomicRMWInst(AtomicRMWInst &AI);
  PtrParts visitAtomicCmpXchgInst(AtomicCmpXchgInst &AI);
  PtrParts visitGetElementPtrInst(GetElementPtrInst &GEP);

  PtrParts visitPtrToAddrInst(PtrToAddrInst &PA);
  PtrParts visitPtrToIntInst(PtrToIntInst &PI);
  PtrParts visitIntToPtrInst(IntToPtrInst &IP);
  PtrParts visitAddrSpaceCastInst(AddrSpaceCastInst &I);
  PtrParts visitICmpInst(ICmpInst &Cmp);
  PtrParts visitFreezeInst(FreezeInst &I);

  PtrParts visitExtractElementInst(ExtractElementInst &I);
  PtrParts visitInsertElementInst(InsertElementInst &I);
  PtrParts visitShuffleVectorInst(ShuffleVectorInst &I);

  PtrParts visitPHINode(PHINode &PHI);
  PtrParts visitSelectInst(SelectInst &SI);

  PtrParts visitIntrinsicInst(IntrinsicInst &II);
};
} // namespace
```

<a name="block_24"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPULowerBufferFatPointers.cpp (L1394-1424) [<sup>↩</sup>](#ref-block_24)

```cpp
PtrParts SplitPtrStructs::getPtrParts(Value *V) {
  assert(isSplitFatPtr(V->getType()) && "it's not meaningful to get the parts "
                                        "of something that wasn't rewritten");
  auto *RsrcEntry = &RsrcParts[V];
  auto *OffEntry = &OffParts[V];
  if (*RsrcEntry && *OffEntry)
    return {*RsrcEntry, *OffEntry};

  if (auto *C = dyn_cast<Constant>(V)) {
    auto [Rsrc, Off] = splitLoweredFatBufferConst(C);
    return {*RsrcEntry = Rsrc, *OffEntry = Off};
  }

  IRBuilder<InstSimplifyFolder>::InsertPointGuard Guard(IRB);
  if (auto *I = dyn_cast<Instruction>(V)) {
    LLVM_DEBUG(dbgs() << "Recursing to split parts of " << *I << "\n");
    auto [Rsrc, Off] = visit(*I);
    if (Rsrc && Off)
      return {*RsrcEntry = Rsrc, *OffEntry = Off};
    // We'll be creating the new values after the relevant instruction.
    // This instruction generates a value and so isn't a terminator.
    IRB.SetInsertPoint(*I->getInsertionPointAfterDef());
    IRB.SetCurrentDebugLocation(I->getDebugLoc());
  } else if (auto *A = dyn_cast<Argument>(V)) {
    IRB.SetInsertPointPastAllocas(A->getParent());
    IRB.SetCurrentDebugLocation(DebugLoc());
  }
  Value *Rsrc = IRB.CreateExtractValue(V, 0, V->getName() + ".rsrc");
  Value *Off = IRB.CreateExtractValue(V, 1, V->getName() + ".off");
  return {*RsrcEntry = Rsrc, *OffEntry = Off};
}
```

<a name="block_25"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPULowerBufferFatPointers.cpp (L1442-1468) [<sup>↩</sup>](#ref-block_25)

```cpp
void SplitPtrStructs::getPossibleRsrcRoots(Instruction *I,
                                           SmallPtrSetImpl<Value *> &Roots,
                                           SmallPtrSetImpl<Value *> &Seen) {
  if (auto *PHI = dyn_cast<PHINode>(I)) {
    if (!Seen.insert(I).second)
      return;
    for (Value *In : PHI->incoming_values()) {
      In = rsrcPartRoot(In);
      Roots.insert(In);
      if (isa<PHINode, SelectInst>(In))
        getPossibleRsrcRoots(cast<Instruction>(In), Roots, Seen);
    }
  } else if (auto *SI = dyn_cast<SelectInst>(I)) {
    if (!Seen.insert(SI).second)
      return;
    Value *TrueVal = rsrcPartRoot(SI->getTrueValue());
    Value *FalseVal = rsrcPartRoot(SI->getFalseValue());
    Roots.insert(TrueVal);
    Roots.insert(FalseVal);
    if (isa<PHINode, SelectInst>(TrueVal))
      getPossibleRsrcRoots(cast<Instruction>(TrueVal), Roots, Seen);
    if (isa<PHINode, SelectInst>(FalseVal))
      getPossibleRsrcRoots(cast<Instruction>(FalseVal), Roots, Seen);
  } else {
    llvm_unreachable("getPossibleRsrcParts() only works on phi and select");
  }
}
```

<a name="block_26"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPULowerBufferFatPointers.cpp (L1470-1578) [<sup>↩</sup>](#ref-block_26)

```cpp
void SplitPtrStructs::processConditionals() {
  SmallDenseMap<Value *, Value *> FoundRsrcs;
  SmallPtrSet<Value *, 4> Roots;
  SmallPtrSet<Value *, 4> Seen;
  for (Instruction *I : Conditionals) {
    // These have to exist by now because we've visited these nodes.
    Value *Rsrc = RsrcParts[I];
    Value *Off = OffParts[I];
    assert(Rsrc && Off && "must have visited conditionals by now");

    std::optional<Value *> MaybeRsrc;
    auto MaybeFoundRsrc = FoundRsrcs.find(I);
    if (MaybeFoundRsrc != FoundRsrcs.end()) {
      MaybeRsrc = MaybeFoundRsrc->second;
    } else {
      IRBuilder<InstSimplifyFolder>::InsertPointGuard Guard(IRB);
      Roots.clear();
      Seen.clear();
      getPossibleRsrcRoots(I, Roots, Seen);
      LLVM_DEBUG(dbgs() << "Processing conditional: " << *I << "\n");
#ifndef NDEBUG
      for (Value *V : Roots)
        LLVM_DEBUG(dbgs() << "Root: " << *V << "\n");
      for (Value *V : Seen)
        LLVM_DEBUG(dbgs() << "Seen: " << *V << "\n");
#endif
      // If we are our own possible root, then we shouldn't block our
      // replacement with a valid incoming value.
      Roots.erase(I);
      // We don't want to block the optimization for conditionals that don't
      // refer to themselves but did see themselves during the traversal.
      Seen.erase(I);

      if (set_is_subset(Seen, Roots)) {
        auto Diff = set_difference(Roots, Seen);
        if (Diff.size() == 1) {
          Value *RootVal = *Diff.begin();
          // Handle the case where previous loops already looked through
          // an addrspacecast.
          if (isSplitFatPtr(RootVal->getType()))
            MaybeRsrc = std::get<0>(getPtrParts(RootVal));
          else
            MaybeRsrc = RootVal;
        }
      }
    }

    if (auto *PHI = dyn_cast<PHINode>(I)) {
      Value *NewRsrc;
      StructType *PHITy = cast<StructType>(PHI->getType());
      IRB.SetInsertPoint(*PHI->getInsertionPointAfterDef());
      IRB.SetCurrentDebugLocation(PHI->getDebugLoc());
      if (MaybeRsrc) {
        NewRsrc = *MaybeRsrc;
      } else {
        Type *RsrcTy = PHITy->getElementType(0);
        auto *RsrcPHI = IRB.CreatePHI(RsrcTy, PHI->getNumIncomingValues());
        RsrcPHI->takeName(Rsrc);
        for (auto [V, BB] : llvm::zip(PHI->incoming_values(), PHI->blocks())) {
          Value *VRsrc = std::get<0>(getPtrParts(V));
          RsrcPHI->addIncoming(VRsrc, BB);
        }
        copyMetadata(RsrcPHI, PHI);
        NewRsrc = RsrcPHI;
      }

      Type *OffTy = PHITy->getElementType(1);
      auto *NewOff = IRB.CreatePHI(OffTy, PHI->getNumIncomingValues());
      NewOff->takeName(Off);
      for (auto [V, BB] : llvm::zip(PHI->incoming_values(), PHI->blocks())) {
        assert(OffParts.count(V) && "An offset part had to be created by now");
        Value *VOff = std::get<1>(getPtrParts(V));
        NewOff->addIncoming(VOff, BB);
      }
      copyMetadata(NewOff, PHI);

      // Note: We don't eraseFromParent() the temporaries because we don't want
      // to put the corrections maps in an inconstent state. That'll be handed
      // during the rest of the killing. Also, `ValueToValueMapTy` guarantees
      // that references in that map will be updated as well.
      // Note that if the temporary instruction got `InstSimplify`'d away, it
      // might be something like a block argument.
      if (auto *RsrcInst = dyn_cast<Instruction>(Rsrc)) {
        ConditionalTemps.push_back(RsrcInst);
        RsrcInst->replaceAllUsesWith(NewRsrc);
      }
      if (auto *OffInst = dyn_cast<Instruction>(Off)) {
        ConditionalTemps.push_back(OffInst);
        OffInst->replaceAllUsesWith(NewOff);
      }

      // Save on recomputing the cycle traversals in known-root cases.
      if (MaybeRsrc)
        for (Value *V : Seen)
          FoundRsrcs[V] = NewRsrc;
    } else if (isa<SelectInst>(I)) {
      if (MaybeRsrc) {
        if (auto *RsrcInst = dyn_cast<Instruction>(Rsrc)) {
          ConditionalTemps.push_back(RsrcInst);
          RsrcInst->replaceAllUsesWith(*MaybeRsrc);
        }
        for (Value *V : Seen)
          FoundRsrcs[V] = *MaybeRsrc;
      }
    } else {
      llvm_unreachable("Only PHIs and selects go in the conditionals list");
    }
  }
}
```

<a name="block_27"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPULowerBufferFatPointers.cpp (L1580-1644) [<sup>↩</sup>](#ref-block_27)

```cpp
void SplitPtrStructs::killAndReplaceSplitInstructions(
    SmallVectorImpl<Instruction *> &Origs) {
  for (Instruction *I : ConditionalTemps)
    I->eraseFromParent();

  for (Instruction *I : Origs) {
    if (!SplitUsers.contains(I))
      continue;

    SmallVector<DbgVariableRecord *> Dbgs;
    findDbgValues(I, Dbgs);
    for (DbgVariableRecord *Dbg : Dbgs) {
      auto &DL = I->getDataLayout();
      assert(isSplitFatPtr(I->getType()) &&
             "We should've RAUW'd away loads, stores, etc. at this point");
      DbgVariableRecord *OffDbg = Dbg->clone();
      auto [Rsrc, Off] = getPtrParts(I);

      int64_t RsrcSz = DL.getTypeSizeInBits(Rsrc->getType());
      int64_t OffSz = DL.getTypeSizeInBits(Off->getType());

      std::optional<DIExpression *> RsrcExpr =
          DIExpression::createFragmentExpression(Dbg->getExpression(), 0,
                                                 RsrcSz);
      std::optional<DIExpression *> OffExpr =
          DIExpression::createFragmentExpression(Dbg->getExpression(), RsrcSz,
                                                 OffSz);
      if (OffExpr) {
        OffDbg->setExpression(*OffExpr);
        OffDbg->replaceVariableLocationOp(I, Off);
        OffDbg->insertBefore(Dbg);
      } else {
        OffDbg->eraseFromParent();
      }
      if (RsrcExpr) {
        Dbg->setExpression(*RsrcExpr);
        Dbg->replaceVariableLocationOp(I, Rsrc);
      } else {
        Dbg->replaceVariableLocationOp(I, PoisonValue::get(I->getType()));
      }
    }

    Value *Poison = PoisonValue::get(I->getType());
    I->replaceUsesWithIf(Poison, [&](const Use &U) -> bool {
      if (const auto *UI = dyn_cast<Instruction>(U.getUser()))
        return SplitUsers.contains(UI);
      return false;
    });

    if (I->use_empty()) {
      I->eraseFromParent();
      continue;
    }
    IRB.SetInsertPoint(*I->getInsertionPointAfterDef());
    IRB.SetCurrentDebugLocation(I->getDebugLoc());
    auto [Rsrc, Off] = getPtrParts(I);
    Value *Struct = PoisonValue::get(I->getType());
    Struct = IRB.CreateInsertValue(Struct, Rsrc, 0);
    Struct = IRB.CreateInsertValue(Struct, Off, 1);
    copyMetadata(Struct, I);
    Struct->takeName(I);
    I->replaceAllUsesWith(Struct);
    I->eraseFromParent();
  }
}
```

<a name="block_28"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPULowerBufferFatPointers.cpp (L1677-1795) [<sup>↩</sup>](#ref-block_28)

```cpp
Value *SplitPtrStructs::handleMemoryInst(Instruction *I, Value *Arg, Value *Ptr,
                                         Type *Ty, Align Alignment,
                                         AtomicOrdering Order, bool IsVolatile,
                                         SyncScope::ID SSID) {
  IRB.SetInsertPoint(I);

  auto [Rsrc, Off] = getPtrParts(Ptr);
  SmallVector<Value *, 5> Args;
  if (Arg)
    Args.push_back(Arg);
  Args.push_back(Rsrc);
  Args.push_back(Off);
  insertPreMemOpFence(Order, SSID);
  // soffset is always 0 for these cases, where we always want any offset to be
  // part of bounds checking and we don't know which parts of the GEPs is
  // uniform.
  Args.push_back(IRB.getInt32(0));

  uint32_t Aux = 0;
  if (IsVolatile)
    Aux |= AMDGPU::CPol::VOLATILE;
  Args.push_back(IRB.getInt32(Aux));

  Intrinsic::ID IID = Intrinsic::not_intrinsic;
  if (isa<LoadInst>(I))
    IID = Order == AtomicOrdering::NotAtomic
              ? Intrinsic::amdgcn_raw_ptr_buffer_load
              : Intrinsic::amdgcn_raw_ptr_atomic_buffer_load;
  else if (isa<StoreInst>(I))
    IID = Intrinsic::amdgcn_raw_ptr_buffer_store;
  else if (auto *RMW = dyn_cast<AtomicRMWInst>(I)) {
    switch (RMW->getOperation()) {
    case AtomicRMWInst::Xchg:
      IID = Intrinsic::amdgcn_raw_ptr_buffer_atomic_swap;
      break;
    case AtomicRMWInst::Add:
      IID = Intrinsic::amdgcn_raw_ptr_buffer_atomic_add;
      break;
    case AtomicRMWInst::Sub:
      IID = Intrinsic::amdgcn_raw_ptr_buffer_atomic_sub;
      break;
    case AtomicRMWInst::And:
      IID = Intrinsic::amdgcn_raw_ptr_buffer_atomic_and;
      break;
    case AtomicRMWInst::Or:
      IID = Intrinsic::amdgcn_raw_ptr_buffer_atomic_or;
      break;
    case AtomicRMWInst::Xor:
      IID = Intrinsic::amdgcn_raw_ptr_buffer_atomic_xor;
      break;
    case AtomicRMWInst::Max:
      IID = Intrinsic::amdgcn_raw_ptr_buffer_atomic_smax;
      break;
    case AtomicRMWInst::Min:
      IID = Intrinsic::amdgcn_raw_ptr_buffer_atomic_smin;
      break;
    case AtomicRMWInst::UMax:
      IID = Intrinsic::amdgcn_raw_ptr_buffer_atomic_umax;
      break;
    case AtomicRMWInst::UMin:
      IID = Intrinsic::amdgcn_raw_ptr_buffer_atomic_umin;
      break;
    case AtomicRMWInst::FAdd:
      IID = Intrinsic::amdgcn_raw_ptr_buffer_atomic_fadd;
      break;
    case AtomicRMWInst::FMax:
      IID = Intrinsic::amdgcn_raw_ptr_buffer_atomic_fmax;
      break;
    case AtomicRMWInst::FMin:
      IID = Intrinsic::amdgcn_raw_ptr_buffer_atomic_fmin;
      break;
    case AtomicRMWInst::FSub: {
      reportFatalUsageError(
          "atomic floating point subtraction not supported for "
          "buffer resources and should've been expanded away");
      break;
    }
    case AtomicRMWInst::FMaximum: {
      reportFatalUsageError(
          "atomic floating point fmaximum not supported for "
          "buffer resources and should've been expanded away");
      break;
    }
    case AtomicRMWInst::FMinimum: {
      reportFatalUsageError(
          "atomic floating point fminimum not supported for "
          "buffer resources and should've been expanded away");
      break;
    }
    case AtomicRMWInst::Nand:
      reportFatalUsageError(
          "atomic nand not supported for buffer resources and "
          "should've been expanded away");
      break;
    case AtomicRMWInst::UIncWrap:
    case AtomicRMWInst::UDecWrap:
      reportFatalUsageError("wrapping increment/decrement not supported for "
                            "buffer resources and should've ben expanded away");
      break;
    case AtomicRMWInst::BAD_BINOP:
      llvm_unreachable("Not sure how we got a bad binop");
    case AtomicRMWInst::USubCond:
    case AtomicRMWInst::USubSat:
      break;
    }
  }

  auto *Call = IRB.CreateIntrinsic(IID, Ty, Args);
  copyMetadata(Call, I);
  setAlign(Call, Alignment, Arg ? 1 : 0);
  Call->takeName(I);

  insertPostMemOpFence(Order, SSID);
  // The "no moving p7 directly" rewrites ensure that this load or store won't
  // itself need to be split into parts.
  SplitUsers.insert(I);
  I->replaceAllUsesWith(Call);
  return Call;
}
```

<a name="block_29"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPULowerBufferFatPointers.cpp (L1871-1923) [<sup>↩</sup>](#ref-block_29)

```cpp
PtrParts SplitPtrStructs::visitGetElementPtrInst(GetElementPtrInst &GEP) {
  using namespace llvm::PatternMatch;
  Value *Ptr = GEP.getPointerOperand();
  if (!isSplitFatPtr(Ptr->getType()))
    return {nullptr, nullptr};
  IRB.SetInsertPoint(&GEP);

  auto [Rsrc, Off] = getPtrParts(Ptr);
  const DataLayout &DL = GEP.getDataLayout();
  bool IsNUW = GEP.hasNoUnsignedWrap();
  bool IsNUSW = GEP.hasNoUnsignedSignedWrap();

  StructType *ResTy = cast<StructType>(GEP.getType());
  Type *ResRsrcTy = ResTy->getElementType(0);
  VectorType *ResRsrcVecTy = dyn_cast<VectorType>(ResRsrcTy);
  bool BroadcastsPtr = ResRsrcVecTy && !isa<VectorType>(Off->getType());

  // In order to call emitGEPOffset() and thus not have to reimplement it,
  // we need the GEP result to have ptr addrspace(7) type.
  Type *FatPtrTy =
      ResRsrcTy->getWithNewType(IRB.getPtrTy(AMDGPUAS::BUFFER_FAT_POINTER));
  GEP.mutateType(FatPtrTy);
  Value *OffAccum = emitGEPOffset(&IRB, DL, &GEP);
  GEP.mutateType(ResTy);

  if (BroadcastsPtr) {
    Rsrc = IRB.CreateVectorSplat(ResRsrcVecTy->getElementCount(), Rsrc,
                                 Rsrc->getName());
    Off = IRB.CreateVectorSplat(ResRsrcVecTy->getElementCount(), Off,
                                Off->getName());
  }
  if (match(OffAccum, m_Zero())) { // Constant-zero offset
    SplitUsers.insert(&GEP);
    return {Rsrc, Off};
  }

  bool HasNonNegativeOff = false;
  if (auto *CI = dyn_cast<ConstantInt>(OffAccum)) {
    HasNonNegativeOff = !CI->isNegative();
  }
  Value *NewOff;
  if (match(Off, m_Zero())) {
    NewOff = OffAccum;
  } else {
    NewOff = IRB.CreateAdd(Off, OffAccum, "",
                           /*hasNUW=*/IsNUW || (IsNUSW && HasNonNegativeOff),
                           /*hasNSW=*/false);
  }
  copyMetadata(NewOff, &GEP);
  NewOff->takeName(&GEP);
  SplitUsers.insert(&GEP);
  return {Rsrc, NewOff};
}
```

<a name="block_30"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPULowerBufferFatPointers.cpp (L1925-2000) [<sup>↩</sup>](#ref-block_30)

```cpp
PtrParts SplitPtrStructs::visitPtrToIntInst(PtrToIntInst &PI) {
  Value *Ptr = PI.getPointerOperand();
  if (!isSplitFatPtr(Ptr->getType()))
    return {nullptr, nullptr};
  IRB.SetInsertPoint(&PI);

  Type *ResTy = PI.getType();
  unsigned Width = ResTy->getScalarSizeInBits();

  auto [Rsrc, Off] = getPtrParts(Ptr);
  const DataLayout &DL = PI.getDataLayout();
  unsigned FatPtrWidth = DL.getPointerSizeInBits(AMDGPUAS::BUFFER_FAT_POINTER);

  Value *Res;
  if (Width <= BufferOffsetWidth) {
    Res = IRB.CreateIntCast(Off, ResTy, /*isSigned=*/false,
                            PI.getName() + ".off");
  } else {
    Value *RsrcInt = IRB.CreatePtrToInt(Rsrc, ResTy, PI.getName() + ".rsrc");
    Value *Shl = IRB.CreateShl(
        RsrcInt,
        ConstantExpr::getIntegerValue(ResTy, APInt(Width, BufferOffsetWidth)),
        "", Width >= FatPtrWidth, Width > FatPtrWidth);
    Value *OffCast = IRB.CreateIntCast(Off, ResTy, /*isSigned=*/false,
                                       PI.getName() + ".off");
    Res = IRB.CreateOr(Shl, OffCast);
  }

  copyMetadata(Res, &PI);
  Res->takeName(&PI);
  SplitUsers.insert(&PI);
  PI.replaceAllUsesWith(Res);
  return {nullptr, nullptr};
}

PtrParts SplitPtrStructs::visitPtrToAddrInst(PtrToAddrInst &PA) {
  Value *Ptr = PA.getPointerOperand();
  if (!isSplitFatPtr(Ptr->getType()))
    return {nullptr, nullptr};
  IRB.SetInsertPoint(&PA);

  auto [Rsrc, Off] = getPtrParts(Ptr);
  Value *Res = IRB.CreateIntCast(Off, PA.getType(), /*isSigned=*/false);
  copyMetadata(Res, &PA);
  Res->takeName(&PA);
  SplitUsers.insert(&PA);
  PA.replaceAllUsesWith(Res);
  return {nullptr, nullptr};
}

PtrParts SplitPtrStructs::visitIntToPtrInst(IntToPtrInst &IP) {
  if (!isSplitFatPtr(IP.getType()))
    return {nullptr, nullptr};
  IRB.SetInsertPoint(&IP);
  const DataLayout &DL = IP.getDataLayout();
  unsigned RsrcPtrWidth = DL.getPointerSizeInBits(AMDGPUAS::BUFFER_RESOURCE);
  Value *Int = IP.getOperand(0);
  Type *IntTy = Int->getType();
  Type *RsrcIntTy = IntTy->getWithNewBitWidth(RsrcPtrWidth);
  unsigned Width = IntTy->getScalarSizeInBits();

  auto *RetTy = cast<StructType>(IP.getType());
  Type *RsrcTy = RetTy->getElementType(0);
  Type *OffTy = RetTy->getElementType(1);
  Value *RsrcPart = IRB.CreateLShr(
      Int,
      ConstantExpr::getIntegerValue(IntTy, APInt(Width, BufferOffsetWidth)));
  Value *RsrcInt = IRB.CreateIntCast(RsrcPart, RsrcIntTy, /*isSigned=*/false);
  Value *Rsrc = IRB.CreateIntToPtr(RsrcInt, RsrcTy, IP.getName() + ".rsrc");
  Value *Off =
      IRB.CreateIntCast(Int, OffTy, /*IsSigned=*/false, IP.getName() + ".off");

  copyMetadata(Rsrc, &IP);
  SplitUsers.insert(&IP);
  return {Rsrc, Off};
}
```

<a name="block_31"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPULowerBufferFatPointers.cpp (L2094-2152) [<sup>↩</sup>](#ref-block_31)

```cpp
PtrParts SplitPtrStructs::visitExtractElementInst(ExtractElementInst &I) {
  if (!isSplitFatPtr(I.getType()))
    return {nullptr, nullptr};
  IRB.SetInsertPoint(&I);
  Value *Vec = I.getVectorOperand();
  Value *Idx = I.getIndexOperand();
  auto [Rsrc, Off] = getPtrParts(Vec);

  Value *RsrcRes = IRB.CreateExtractElement(Rsrc, Idx, I.getName() + ".rsrc");
  copyMetadata(RsrcRes, &I);
  Value *OffRes = IRB.CreateExtractElement(Off, Idx, I.getName() + ".off");
  copyMetadata(OffRes, &I);
  SplitUsers.insert(&I);
  return {RsrcRes, OffRes};
}

PtrParts SplitPtrStructs::visitInsertElementInst(InsertElementInst &I) {
  // The mutated instructions temporarily don't return vectors, and so
  // we need the generic getType() here to avoid crashes.
  if (!isSplitFatPtr(cast<Instruction>(I).getType()))
    return {nullptr, nullptr};
  IRB.SetInsertPoint(&I);
  Value *Vec = I.getOperand(0);
  Value *Elem = I.getOperand(1);
  Value *Idx = I.getOperand(2);
  auto [VecRsrc, VecOff] = getPtrParts(Vec);
  auto [ElemRsrc, ElemOff] = getPtrParts(Elem);

  Value *RsrcRes =
      IRB.CreateInsertElement(VecRsrc, ElemRsrc, Idx, I.getName() + ".rsrc");
  copyMetadata(RsrcRes, &I);
  Value *OffRes =
      IRB.CreateInsertElement(VecOff, ElemOff, Idx, I.getName() + ".off");
  copyMetadata(OffRes, &I);
  SplitUsers.insert(&I);
  return {RsrcRes, OffRes};
}

PtrParts SplitPtrStructs::visitShuffleVectorInst(ShuffleVectorInst &I) {
  // Cast is needed for the same reason as insertelement's.
  if (!isSplitFatPtr(cast<Instruction>(I).getType()))
    return {nullptr, nullptr};
  IRB.SetInsertPoint(&I);

  Value *V1 = I.getOperand(0);
  Value *V2 = I.getOperand(1);
  ArrayRef<int> Mask = I.getShuffleMask();
  auto [V1Rsrc, V1Off] = getPtrParts(V1);
  auto [V2Rsrc, V2Off] = getPtrParts(V2);

  Value *RsrcRes =
      IRB.CreateShuffleVector(V1Rsrc, V2Rsrc, Mask, I.getName() + ".rsrc");
  copyMetadata(RsrcRes, &I);
  Value *OffRes =
      IRB.CreateShuffleVector(V1Off, V2Off, Mask, I.getName() + ".off");
  copyMetadata(OffRes, &I);
  SplitUsers.insert(&I);
  return {RsrcRes, OffRes};
}
```

<a name="block_32"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPULowerBufferFatPointers.cpp (L2455-2576) [<sup>↩</sup>](#ref-block_32)

```cpp
bool AMDGPULowerBufferFatPointers::run(Module &M, const TargetMachine &TM) {
  bool Changed = false;
  const DataLayout &DL = M.getDataLayout();
  // Record the functions which need to be remapped.
  // The second element of the pair indicates whether the function has to have
  // its arguments or return types adjusted.
  SmallVector<std::pair<Function *, bool>> NeedsRemap;

  LLVMContext &Ctx = M.getContext();

  BufferFatPtrToStructTypeMap StructTM(DL);
  BufferFatPtrToIntTypeMap IntTM(DL);
  for (const GlobalVariable &GV : M.globals()) {
    if (GV.getAddressSpace() == AMDGPUAS::BUFFER_FAT_POINTER) {
      // FIXME: Use DiagnosticInfo unsupported but it requires a Function
      Ctx.emitError("global variables with a buffer fat pointer address "
                    "space (7) are not supported");
      continue;
    }

    Type *VT = GV.getValueType();
    if (VT != StructTM.remapType(VT)) {
      // FIXME: Use DiagnosticInfo unsupported but it requires a Function
      Ctx.emitError("global variables that contain buffer fat pointers "
                    "(address space 7 pointers) are unsupported. Use "
                    "buffer resource pointers (address space 8) instead");
      continue;
    }
  }

  {
    // Collect all constant exprs and aggregates referenced by any function.
    SmallVector<Constant *, 8> Worklist;
    for (Function &F : M.functions())
      for (Instruction &I : instructions(F))
        for (Value *Op : I.operands())
          if (isa<ConstantExpr, ConstantAggregate>(Op))
            Worklist.push_back(cast<Constant>(Op));

    // Recursively look for any referenced buffer pointer constants.
    SmallPtrSet<Constant *, 8> Visited;
    SetVector<Constant *> BufferFatPtrConsts;
    while (!Worklist.empty()) {
      Constant *C = Worklist.pop_back_val();
      if (!Visited.insert(C).second)
        continue;
      if (isBufferFatPtrOrVector(C->getType()))
        BufferFatPtrConsts.insert(C);
      for (Value *Op : C->operands())
        if (isa<ConstantExpr, ConstantAggregate>(Op))
          Worklist.push_back(cast<Constant>(Op));
    }

    // Expand all constant expressions using fat buffer pointers to
    // instructions.
    Changed |= convertUsersOfConstantsToInstructions(
        BufferFatPtrConsts.getArrayRef(), /*RestrictToFunc=*/nullptr,
        /*RemoveDeadConstants=*/false, /*IncludeSelf=*/true);
  }

  StoreFatPtrsAsIntsAndExpandMemcpyVisitor MemOpsRewrite(&IntTM, DL,
                                                         M.getContext(), &TM);
  LegalizeBufferContentTypesVisitor BufferContentsTypeRewrite(DL,
                                                              M.getContext());
  for (Function &F : M.functions()) {
    bool InterfaceChange = hasFatPointerInterface(F, &StructTM);
    bool BodyChanges = containsBufferFatPointers(F, &StructTM);
    Changed |= MemOpsRewrite.processFunction(F);
    if (InterfaceChange || BodyChanges) {
      NeedsRemap.push_back(std::make_pair(&F, InterfaceChange));
      Changed |= BufferContentsTypeRewrite.processFunction(F);
    }
  }
  if (NeedsRemap.empty())
    return Changed;

  SmallVector<Function *> NeedsPostProcess;
  SmallVector<Function *> Intrinsics;
  // Keep one big map so as to memoize constants across functions.
  ValueToValueMapTy CloneMap;
  FatPtrConstMaterializer Materializer(&StructTM, CloneMap);

  ValueMapper LowerInFuncs(CloneMap, RF_None, &StructTM, &Materializer);
  for (auto [F, InterfaceChange] : NeedsRemap) {
    Function *NewF = F;
    if (InterfaceChange)
      NewF = moveFunctionAdaptingType(
          F, cast<FunctionType>(StructTM.remapType(F->getFunctionType())),
          CloneMap);
    else
      makeCloneInPraceMap(F, CloneMap);
    LowerInFuncs.remapFunction(*NewF);
    if (NewF->isIntrinsic())
      Intrinsics.push_back(NewF);
    else
      NeedsPostProcess.push_back(NewF);
    if (InterfaceChange) {
      F->replaceAllUsesWith(NewF);
      F->eraseFromParent();
    }
    Changed = true;
  }
  StructTM.clear();
  IntTM.clear();
  CloneMap.clear();

  SplitPtrStructs Splitter(DL, M.getContext(), &TM);
  for (Function *F : NeedsPostProcess)
    Splitter.processFunction(*F);
  for (Function *F : Intrinsics) {
    // use_empty() can also occur with cases like masked load, which will
    // have been rewritten out of the module by now but not erased.
    if (F->use_empty() || isRemovablePointerIntrinsic(F->getIntrinsicID())) {
      F->eraseFromParent();
    } else {
      std::optional<Function *> NewF = Intrinsic::remangleIntrinsicFunction(F);
      if (NewF)
        F->replaceAllUsesWith(*NewF);
    }
  }
  return Changed;
}
```

