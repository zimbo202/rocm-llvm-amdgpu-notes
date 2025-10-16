# AMDGPU后端软件功能详细描述

## 1. AMDGPU后端的主要功能模块

### 1.1 目标机器与编译流程配置

<a name="ref-block_1"></a>AMDGPU后端通过 `GCNTargetMachine` 和 `AMDGPUCodeGenPassBuilder` 类来管理整个代码生成流程。 llvm-project:1144-1150[<sup>↗</sup>](#block_1) 

<a name="ref-block_4"></a>编译流程配置类负责设置和组织各个编译阶段的Pass执行顺序，支持传统Pass Manager和新的Pass Manager两种模式。 llvm-project:2072-2083[<sup>↗</sup>](#block_4) 

### 1.2 Pass注册与管理

AMDGPU后端定义了完整的Pass注册表，包括三类Pass：
- **模块级Pass（Module Pass）**：如LDS lowering、特性谓词展开、元数据统一等
- **函数级Pass（Function Pass）**：如CodeGenPrepare、内核参数lowering、Promote Alloca等
<a name="ref-block_16"></a>- **机器函数级Pass（Machine Function Pass）**：如指令折叠、寄存器复制修复、内存合并等 llvm-project:16-140[<sup>↗</sup>](#block_16) 

### 1.3 指令选择子系统

AMDGPU支持两种指令选择路径：

**（1）SelectionDAG路径（主要）**：
- `AMDGPUISelLowering`：基础IR操作lowering到DAG节点
- `SIISelLowering`：SI架构特定的指令选择lowering
<a name="ref-block_18"></a>- `AMDGPUDAGToDAGISel`：SelectionDAG模式匹配与指令选择 llvm-project:63-86[<sup>↗</sup>](#block_18) 

**（2）Global ISel路径（备选）**：
- `AMDGPUInstructionSelector`：全局指令选择器
- `AMDGPULegalizerInfo`：类型合法化
<a name="ref-block_17"></a>- `AMDGPURegisterBankInfo`：寄存器bank分配 llvm-project:143-149[<sup>↗</sup>](#block_17) 

### 1.4 寄存器分配子系统

AMDGPU具有独特的多寄存器bank架构，需要分阶段分配：
- **SGPR分配器**：分配标量寄存器（Scalar GPRs）
- **VGPR分配器**：分配矢量寄存器（Vector GPRs）
- **WWM寄存器分配器**：分配whole wave mode寄存器
<a name="ref-block_3"></a>- **AGPR支持**：累加器寄存器（用于矩阵运算） llvm-project:1657-1700[<sup>↗</sup>](#block_3) 

<a name="ref-block_12"></a>寄存器分配采用Greedy或Fast策略，针对不同寄存器类型使用独立的分配器实例。 llvm-project:2313-2352[<sup>↗</sup>](#block_12) 

### 1.5 IR转换与优化Pass

**模块级转换**：
- `AMDGPULowerModuleLDS`：LDS（Local Data Share）内存lowering
- `AMDGPUSwLowerLDS`：软件LDS lowering（用于地址清理）
- `AMDGPUPrintfRuntimeBinding`：printf函数运行时绑定
- `AMDGPURemoveIncompatibleFunctions`：移除不兼容函数 llvm-project:19-34 

**函数级转换**：
- `AMDGPUCodeGenPrepare`：代码生成准备（地址空间优化、指令lowering）
- `AMDGPUPromoteAlloca`：提升alloca到寄存器或LDS
- `AMDGPUAtomicOptimizer`：原子操作优化
- `AMDGPULowerKernelArguments`：内核参数lowering
- `SIAnnotateControlFlow`：控制流标注 llvm-project:53-74 

### 1.6 机器级优化Pass

**指令优化**：
- `SIFoldOperands`：操作数折叠（消除冗余拷贝）
- `SIShrinkInstructions`：指令缩减（使用更紧凑的编码）
- `SILoadStoreOptimizer`：load/store合并优化
- `SIPeepholeSDWA`：SDWA（Sub-Dword Addressing）窥孔优化
<a name="ref-block_10"></a>- `GCNDPPCombine`：DPP（Data Parallel Primitives）合并 llvm-project:2249-2266[<sup>↗</sup>](#block_10) 

**控制流与寄存器优化**：
- `SIWholeQuadMode`：whole quad mode插入与优化
- `SILowerControlFlow`：控制流lowering
- `SIOptimizeExecMasking`：exec mask优化
- `SIFixSGPRCopies`/`SIFixVGPRCopies`：寄存器拷贝修复 llvm-project:117-139 

### 1.7 指令调度器

AMDGPU提供多种调度策略：
- **GCN Max Occupancy Scheduler**：最大化occupancy
- **GCN Max ILP Scheduler**：最大化指令级并行
- **GCN Max Memory Clause Scheduler**：最大化内存子句
<a name="ref-block_0"></a>- **Iterative Schedulers**：迭代式调度器（用于寄存器压力优化） llvm-project:628-728[<sup>↗</sup>](#block_0) 

### 1.8 代码发射与元数据生成

**元数据流式传输**：
- `AMDGPUHSAMetadataStreamer`：HSA元数据生成
- `AMDGPUPALMetadata`：PAL元数据管理
- 内核描述符生成

**汇编与反汇编**：
- `AMDGPUAsmPrinter`：机器码到汇编的打印
- `AMDGPUAsmParser`：汇编语法解析
- `AMDGPUDisassembler`：二进制指令反汇编
- `AMDGPUAsmBackend`：fixup和relaxation处理

### 1.9 Subtarget特性管理

`GCNSubtarget` 管理不同GPU架构的特性：
- 指令集特性检测（wave32/wave64、DPP、SDWA等）
- 寄存器文件大小配置
- 内存层次结构参数
<a name="ref-block_2"></a>- 调度策略选择 llvm-project:1153-1172[<sup>↗</sup>](#block_2) 

## 2. AMDGPU后端的端到端执行步骤

### 步骤1：IR级Pass处理

**执行时机**：最早阶段，在IR层面进行转换

**主要操作**：
- 移除不兼容函数
- Printf运行时绑定
- 构造函数/析构函数lowering
- 图像内联优化
- 强制内联（总是内联pass）
- LDS lowering（模块级和软件级）
- 原子操作优化
- 原子扩展
- Alloca提升到LDS或寄存器
- 标量IR优化（GEP分离、SLSR、GVN/EarlyCSE）
<a name="ref-block_5"></a>- CodeGenPrepare（地址空间推断、指令lowering） llvm-project:2085-2153[<sup>↗</sup>](#block_5) 

**传递内容**：LLVM IR模块 → 优化后的LLVM IR模块

### 步骤2：CodeGenPrepare阶段

**执行时机**：IR到MachineIR转换前的最后准备

**主要操作**：
- 内核参数预加载（优化参数访问）
- 内核参数lowering（load展开）
- Load/Store向量化
- Buffer fat pointers lowering（拆分为组件）
- Intrinsic lowering
<a name="ref-block_6"></a>- Switch lowering llvm-project:2155-2184[<sup>↗</sup>](#block_6) 

**传递内容**：IR函数 → 准备好进行指令选择的IR

### 步骤3：PreISel转换

**执行时机**：指令选择前的最后IR转换

**主要操作**：
- CFG扁平化（FlattenCFG）
- Sinking优化
- Late CodeGenPrepare
- 统一分歧退出节点
- 修复不可归约控制流
- 统一循环出口
- 结构化CFG（StructurizeCFG）- 关键步骤！
- 标注uniform值
- 控制流标注（插入if/else/loop等intrinsics）
- 重写PHI的undef
<a name="ref-block_7"></a>- LCSSA形式转换 llvm-project:2186-2222[<sup>↗</sup>](#block_7) 

**传递内容**：IR函数（结构化的CFG）→ 准备进入SelectionDAG的IR

**关键点**：StructurizeCFG是AMDGPU最重要的转换之一，它将任意控制流转换为结构化形式（只使用if-then-else和循环），这对于GPU的SIMT执行模型至关重要。

### 步骤4：指令选择（Instruction Selection）

**执行时机**：IR到MachineIR的转换点

**主要操作**：

**SelectionDAG路径**：
1. DAG构建：将IR转换为SelectionDAG节点
2. Combine优化：DAG节点合并与简化
3. 合法化：类型合法化和操作合法化
4. DAG到DAG指令选择：模式匹配选择目标指令
5. 调度与发射：生成MachineInstr

**后处理**：
- SGPR拷贝修复（SIFixSGPRCopies）
<a name="ref-block_8"></a>- i1类型拷贝lowering（SILowerI1Copies） llvm-project:2236-2241[<sup>↗</sup>](#block_8) 

**传递内容**：LLVM IR → SelectionDAG → MachineInstr（机器指令，SSA形式）

### 步骤5：Machine SSA优化

**执行时机**：生成MachineInstr后的SSA优化阶段

**主要操作**：
- 操作数折叠（SIFoldOperands）- 消除中间拷贝
- DPP合并（GCNDPPCombine）- 使用数据并行原语
- Load/Store优化（SILoadStoreOptimizer）- 合并内存访问
- SDWA窥孔优化（SIPeepholeSDWA）
- 死指令消除（DeadMachineInstructionElim）
<a name="ref-block_10"></a>- 指令缩减（SIShrinkInstructions） llvm-project:2249-2266[<sup>↗</sup>](#block_10) 

**传递内容**：MachineInstr（初始SSA形式）→ 优化后的MachineInstr（仍是SSA形式）

### 步骤6：寄存器分配前优化

**执行时机**：寄存器分配前的准备和优化

**主要操作**：
- AGPR分配准备（AMDGPUPrepareAGPRAlloc）
- VGPR活跃范围优化（SIOptimizeVGPRLiveRange）
- PHI消除和控制流lowering（SILowerControlFlow）
- 部分寄存器使用重写（GCNRewritePartialRegUses）
- Pre-RA优化（GCNPreRAOptimizations）
- Whole Quad Mode插入（SIWholeQuadMode）
- Exec masking优化（SIOptimizeExecMaskingPreRA）
<a name="ref-block_11"></a>- 内存子句形成（SIFormMemoryClauses） llvm-project:2268-2305[<sup>↗</sup>](#block_11) 

**传递内容**：SSA形式的MachineInstr → 准备分配寄存器的MachineInstr

### 步骤7：寄存器分配

**执行时机**：核心的寄存器分配阶段

**分配顺序**（三阶段分配）：

**阶段1 - SGPR分配**：
- 预留长分支寄存器（GCNPreRALongBranchReg）
- SGPR贪婪分配（RAGreedy with onlyAllocateSGPRs）
- 虚拟寄存器重写（VirtRegRewriter）
- 栈槽着色（StackSlotColoring）
- SGPR溢出lowering（SILowerSGPRSpills）- 自定义SGPR溢出处理

**阶段2 - WWM寄存器分配**：
- WWM寄存器预分配（SIPreAllocateWWMRegs）
- WWM寄存器分配（RAGreedy with onlyAllocateWWMRegs）
- WWM拷贝lowering（SILowerWWMCopies）
- WWM寄存器预留（AMDGPUReserveWWMRegs）

**阶段3 - VGPR分配**：
<a name="ref-block_12"></a>- VGPR贪婪分配（RAGreedy with onlyAllocateVGPRs） llvm-project:2313-2352[<sup>↗</sup>](#block_12) 

**传递内容**：虚拟寄存器 → 物理寄存器（SGPR、VGPR、WWM寄存器）

**关键设计**：AMDGPU的分阶段寄存器分配是其最独特的设计之一，因为不同寄存器类型有不同的约束和使用场景。

### 步骤8：寄存器分配后优化

**执行时机**：完成寄存器分配后

**主要操作**：
- NSA寄存器重分配（GCNNSAReassign）- 针对GFX10+的NSA指令
- AGPR拷贝重写（AMDGPURewriteAGPRCopyMFMA）
- VGPR拷贝修复（SIFixVGPRCopies）
- Exec masking优化（SIOptimizeExecMasking）
<a name="ref-block_9"></a>- 最后scratch load标记（AMDGPUMarkLastScratchLoad） llvm-project:2243-2248[<sup>↗</sup>](#block_9) llvm-project:2354-2359 

**传递内容**：分配物理寄存器的MachineInstr → 修复后的MachineInstr

### 步骤9：Pre-Sched2优化

**执行时机**：后调度前

**主要操作**：
- 指令缩减（SIShrinkInstructions）
<a name="ref-block_14"></a>- Post-RA bundler（SIPostRABundler）- 将相关指令打包 llvm-project:2361-2365[<sup>↗</sup>](#block_14) 

**传递内容**：MachineInstr → 打包的指令束

### 步骤10：Pre-Emit优化与代码发射准备

**执行时机**：代码发射前的最后转换

**主要操作**：
- VOPD创建（GCNCreateVOPD）- GFX11+的双发射VALU指令
- 内存合法化（SIMemoryLegalizer）- 插入必要的内存fence
- Waitcnt插入（SIInsertWaitcnts）- 插入等待计数器指令
- Mode寄存器管理（SIModeRegister）
- Hard子句插入（SIInsertHardClauses）
- 分支lowering（SILateBranchLowering）
- Wave优先级设置（AMDGPUSetWavePriority）
- Pre-emit窥孔优化（SIPreEmitPeephole）
- 后RA hazard识别（PostRAHazardRecognizer）
- SGPR hazard等待（AMDGPUWaitSGPRHazards）
- VGPR编码lowering（AMDGPULowerVGPREncoding）
- Delay ALU插入（AMDGPUInsertDelayAlu）- GFX11+指令延迟
<a name="ref-block_15"></a>- 分支松弛（BranchRelaxation） llvm-project:2367-2406[<sup>↗</sup>](#block_15) 

**传递内容**：MachineInstr → 完全合法化的机器码（带同步和hazard处理）

### 步骤11：代码发射

**执行时机**：最终阶段

**主要操作**：
- 指令编码：将MachineInstr转换为二进制编码
- 元数据生成：生成HSA/PAL元数据
- 符号表构建
- 重定位信息生成
- 目标文件写出（ELF格式）

**传递内容**：MachineInstr → 二进制目标文件（.o）+ 元数据

## Notes

AMDGPU后端的设计充分考虑了GPU架构的特殊性：

1. **结构化控制流**：通过StructurizeCFG将任意控制流转换为结构化形式，适配SIMT执行模型

2. **分阶段寄存器分配**：SGPR、WWM和VGPR的独立分配，每种寄存器有不同的约束和溢出策略

3. **显式同步**：通过插入waitcnt、s_barrier等指令实现内存一致性和hazard避免

4. **特殊优化**：DPP、SDWA、VOPD等GPU特有的指令格式优化

5. **Occupancy管理**：通过调度器策略和寄存器分配平衡occupancy和性能

整个流程是一个典型的编译器后端pipeline，但在每个阶段都针对GPU的SIMT架构和硬件约束进行了深度定制。


### Citations


<a name="block_0"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUTargetMachine.cpp (L628-728) [<sup>↩</sup>](#ref-block_0)

```cpp
static ScheduleDAGInstrs *createSIMachineScheduler(MachineSchedContext *C) {
  return new SIScheduleDAGMI(C);
}

static ScheduleDAGInstrs *
createGCNMaxOccupancyMachineScheduler(MachineSchedContext *C) {
  const GCNSubtarget &ST = C->MF->getSubtarget<GCNSubtarget>();
  ScheduleDAGMILive *DAG =
    new GCNScheduleDAGMILive(C, std::make_unique<GCNMaxOccupancySchedStrategy>(C));
  DAG->addMutation(createLoadClusterDAGMutation(DAG->TII, DAG->TRI));
  if (ST.shouldClusterStores())
    DAG->addMutation(createStoreClusterDAGMutation(DAG->TII, DAG->TRI));
  DAG->addMutation(createIGroupLPDAGMutation(AMDGPU::SchedulingPhase::Initial));
  DAG->addMutation(createAMDGPUMacroFusionDAGMutation());
  DAG->addMutation(createAMDGPUExportClusteringDAGMutation());
  return DAG;
}

static ScheduleDAGInstrs *
createGCNMaxILPMachineScheduler(MachineSchedContext *C) {
  ScheduleDAGMILive *DAG =
      new GCNScheduleDAGMILive(C, std::make_unique<GCNMaxILPSchedStrategy>(C));
  DAG->addMutation(createIGroupLPDAGMutation(AMDGPU::SchedulingPhase::Initial));
  return DAG;
}

static ScheduleDAGInstrs *
createGCNMaxMemoryClauseMachineScheduler(MachineSchedContext *C) {
  const GCNSubtarget &ST = C->MF->getSubtarget<GCNSubtarget>();
  ScheduleDAGMILive *DAG = new GCNScheduleDAGMILive(
      C, std::make_unique<GCNMaxMemoryClauseSchedStrategy>(C));
  DAG->addMutation(createLoadClusterDAGMutation(DAG->TII, DAG->TRI));
  if (ST.shouldClusterStores())
    DAG->addMutation(createStoreClusterDAGMutation(DAG->TII, DAG->TRI));
  DAG->addMutation(createAMDGPUExportClusteringDAGMutation());
  return DAG;
}

static ScheduleDAGInstrs *
createIterativeGCNMaxOccupancyMachineScheduler(MachineSchedContext *C) {
  const GCNSubtarget &ST = C->MF->getSubtarget<GCNSubtarget>();
  auto *DAG = new GCNIterativeScheduler(
      C, GCNIterativeScheduler::SCHEDULE_LEGACYMAXOCCUPANCY);
  DAG->addMutation(createLoadClusterDAGMutation(DAG->TII, DAG->TRI));
  if (ST.shouldClusterStores())
    DAG->addMutation(createStoreClusterDAGMutation(DAG->TII, DAG->TRI));
  DAG->addMutation(createIGroupLPDAGMutation(AMDGPU::SchedulingPhase::Initial));
  return DAG;
}

static ScheduleDAGInstrs *createMinRegScheduler(MachineSchedContext *C) {
  auto *DAG = new GCNIterativeScheduler(
      C, GCNIterativeScheduler::SCHEDULE_MINREGFORCED);
  DAG->addMutation(createIGroupLPDAGMutation(AMDGPU::SchedulingPhase::Initial));
  return DAG;
}

static ScheduleDAGInstrs *
createIterativeILPMachineScheduler(MachineSchedContext *C) {
  const GCNSubtarget &ST = C->MF->getSubtarget<GCNSubtarget>();
  auto *DAG = new GCNIterativeScheduler(C, GCNIterativeScheduler::SCHEDULE_ILP);
  DAG->addMutation(createLoadClusterDAGMutation(DAG->TII, DAG->TRI));
  if (ST.shouldClusterStores())
    DAG->addMutation(createStoreClusterDAGMutation(DAG->TII, DAG->TRI));
  DAG->addMutation(createAMDGPUMacroFusionDAGMutation());
  DAG->addMutation(createIGroupLPDAGMutation(AMDGPU::SchedulingPhase::Initial));
  return DAG;
}

static MachineSchedRegistry
SISchedRegistry("si", "Run SI's custom scheduler",
                createSIMachineScheduler);

static MachineSchedRegistry
GCNMaxOccupancySchedRegistry("gcn-max-occupancy",
                             "Run GCN scheduler to maximize occupancy",
                             createGCNMaxOccupancyMachineScheduler);

static MachineSchedRegistry
    GCNMaxILPSchedRegistry("gcn-max-ilp", "Run GCN scheduler to maximize ilp",
                           createGCNMaxILPMachineScheduler);

static MachineSchedRegistry GCNMaxMemoryClauseSchedRegistry(
    "gcn-max-memory-clause", "Run GCN scheduler to maximize memory clause",
    createGCNMaxMemoryClauseMachineScheduler);

static MachineSchedRegistry IterativeGCNMaxOccupancySchedRegistry(
    "gcn-iterative-max-occupancy-experimental",
    "Run GCN scheduler to maximize occupancy (experimental)",
    createIterativeGCNMaxOccupancyMachineScheduler);

static MachineSchedRegistry GCNMinRegSchedRegistry(
    "gcn-iterative-minreg",
    "Run GCN iterative scheduler for minimal register usage (experimental)",
    createMinRegScheduler);

static MachineSchedRegistry GCNILPSchedRegistry(
    "gcn-iterative-ilp",
    "Run GCN iterative scheduler for ILP scheduling (experimental)",
    createIterativeILPMachineScheduler);

```

<a name="block_1"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUTargetMachine.cpp (L1144-1150) [<sup>↩</sup>](#ref-block_1)

```cpp
GCNTargetMachine::GCNTargetMachine(const Target &T, const Triple &TT,
                                   StringRef CPU, StringRef FS,
                                   const TargetOptions &Options,
                                   std::optional<Reloc::Model> RM,
                                   std::optional<CodeModel::Model> CM,
                                   CodeGenOptLevel OL, bool JIT)
    : AMDGPUTargetMachine(T, TT, CPU, FS, Options, RM, CM, OL) {}
```

<a name="block_2"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUTargetMachine.cpp (L1153-1172) [<sup>↩</sup>](#ref-block_2)

```cpp
GCNTargetMachine::getSubtargetImpl(const Function &F) const {
  StringRef GPU = getGPUName(F);
  StringRef FS = getFeatureString(F);

  SmallString<128> SubtargetKey(GPU);
  SubtargetKey.append(FS);

  auto &I = SubtargetMap[SubtargetKey];
  if (!I) {
    // This needs to be done before we create a new subtarget since any
    // creation will depend on the TM and the code generation flags on the
    // function that reside in TargetOptions.
    resetTargetOptions(F);
    I = std::make_unique<GCNSubtarget>(TargetTriple, GPU, FS, *this);
  }

  I->setScalarizeGlobalBehavior(ScalarizeGlobal);

  return I.get();
}
```

<a name="block_3"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUTargetMachine.cpp (L1657-1700) [<sup>↩</sup>](#ref-block_3)

```cpp
FunctionPass *GCNPassConfig::createSGPRAllocPass(bool Optimized) {
  // Initialize the global default.
  llvm::call_once(InitializeDefaultSGPRRegisterAllocatorFlag,
                  initializeDefaultSGPRRegisterAllocatorOnce);

  RegisterRegAlloc::FunctionPassCtor Ctor = SGPRRegisterRegAlloc::getDefault();
  if (Ctor != useDefaultRegisterAllocator)
    return Ctor();

  if (Optimized)
    return createGreedyRegisterAllocator(onlyAllocateSGPRs);

  return createFastRegisterAllocator(onlyAllocateSGPRs, false);
}

FunctionPass *GCNPassConfig::createVGPRAllocPass(bool Optimized) {
  // Initialize the global default.
  llvm::call_once(InitializeDefaultVGPRRegisterAllocatorFlag,
                  initializeDefaultVGPRRegisterAllocatorOnce);

  RegisterRegAlloc::FunctionPassCtor Ctor = VGPRRegisterRegAlloc::getDefault();
  if (Ctor != useDefaultRegisterAllocator)
    return Ctor();

  if (Optimized)
    return createGreedyVGPRRegisterAllocator();

  return createFastVGPRRegisterAllocator();
}

FunctionPass *GCNPassConfig::createWWMRegAllocPass(bool Optimized) {
  // Initialize the global default.
  llvm::call_once(InitializeDefaultWWMRegisterAllocatorFlag,
                  initializeDefaultWWMRegisterAllocatorOnce);

  RegisterRegAlloc::FunctionPassCtor Ctor = WWMRegisterRegAlloc::getDefault();
  if (Ctor != useDefaultRegisterAllocator)
    return Ctor();

  if (Optimized)
    return createGreedyWWMRegisterAllocator();

  return createFastWWMRegisterAllocator();
}
```

<a name="block_4"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUTargetMachine.cpp (L2072-2083) [<sup>↩</sup>](#ref-block_4)

```cpp
AMDGPUCodeGenPassBuilder::AMDGPUCodeGenPassBuilder(
    GCNTargetMachine &TM, const CGPassBuilderOption &Opts,
    PassInstrumentationCallbacks *PIC)
    : CodeGenPassBuilder(TM, Opts, PIC) {
  Opt.MISchedPostRA = true;
  Opt.RequiresCodeGenSCCOrder = true;
  // Exceptions and StackMaps are not supported, so these passes will never do
  // anything.
  // Garbage collection is not supported.
  disablePass<StackMapLivenessPass, FuncletLayoutPass,
              ShadowStackGCLoweringPass>();
}
```

<a name="block_5"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUTargetMachine.cpp (L2085-2153) [<sup>↩</sup>](#ref-block_5)

```cpp
void AMDGPUCodeGenPassBuilder::addIRPasses(AddIRPass &addPass) const {
  if (RemoveIncompatibleFunctions && TM.getTargetTriple().isAMDGCN())
    addPass(AMDGPURemoveIncompatibleFunctionsPass(TM));

  addPass(AMDGPUPrintfRuntimeBindingPass());
  if (LowerCtorDtor)
    addPass(AMDGPUCtorDtorLoweringPass());

  if (isPassEnabled(EnableImageIntrinsicOptimizer))
    addPass(AMDGPUImageIntrinsicOptimizerPass(TM));

  // This can be disabled by passing ::Disable here or on the command line
  // with --expand-variadics-override=disable.
  addPass(ExpandVariadicsPass(ExpandVariadicsMode::Lowering));

  addPass(AMDGPUAlwaysInlinePass());
  addPass(AlwaysInlinerPass());

  addPass(AMDGPUExportKernelRuntimeHandlesPass());

  if (EnableSwLowerLDS)
    addPass(AMDGPUSwLowerLDSPass(TM));

  // Runs before PromoteAlloca so the latter can account for function uses
  if (EnableLowerModuleLDS)
    addPass(AMDGPULowerModuleLDSPass(TM));

  // Run atomic optimizer before Atomic Expand
  if (TM.getOptLevel() >= CodeGenOptLevel::Less &&
      (AMDGPUAtomicOptimizerStrategy != ScanOptions::None))
    addPass(AMDGPUAtomicOptimizerPass(TM, AMDGPUAtomicOptimizerStrategy));

  addPass(AtomicExpandPass(&TM));

  if (TM.getOptLevel() > CodeGenOptLevel::None) {
    addPass(AMDGPUPromoteAllocaPass(TM));
    if (isPassEnabled(EnableScalarIRPasses))
      addStraightLineScalarOptimizationPasses(addPass);

    // TODO: Handle EnableAMDGPUAliasAnalysis

    // TODO: May want to move later or split into an early and late one.
    addPass(AMDGPUCodeGenPreparePass(TM));

    // Try to hoist loop invariant parts of divisions AMDGPUCodeGenPrepare may
    // have expanded.
    if (TM.getOptLevel() > CodeGenOptLevel::Less) {
      addPass(createFunctionToLoopPassAdaptor(LICMPass(LICMOptions()),
                                              /*UseMemorySSA=*/true));
    }
  }

  Base::addIRPasses(addPass);

  // EarlyCSE is not always strong enough to clean up what LSR produces. For
  // example, GVN can combine
  //
  //   %0 = add %a, %b
  //   %1 = add %b, %a
  //
  // and
  //
  //   %0 = shl nsw %a, 2
  //   %1 = shl %a, 2
  //
  // but EarlyCSE can do neither of them.
  if (isPassEnabled(EnableScalarIRPasses))
    addEarlyCSEOrGVNPass(addPass);
}
```

<a name="block_6"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUTargetMachine.cpp (L2155-2184) [<sup>↩</sup>](#ref-block_6)

```cpp
void AMDGPUCodeGenPassBuilder::addCodeGenPrepare(AddIRPass &addPass) const {
  if (TM.getOptLevel() > CodeGenOptLevel::None)
    addPass(AMDGPUPreloadKernelArgumentsPass(TM));

  if (EnableLowerKernelArguments)
    addPass(AMDGPULowerKernelArgumentsPass(TM));

  Base::addCodeGenPrepare(addPass);

  if (isPassEnabled(EnableLoadStoreVectorizer))
    addPass(LoadStoreVectorizerPass());

  // This lowering has been placed after codegenprepare to take advantage of
  // address mode matching (which is why it isn't put with the LDS lowerings).
  // It could be placed anywhere before uniformity annotations (an analysis
  // that it changes by splitting up fat pointers into their components)
  // but has been put before switch lowering and CFG flattening so that those
  // passes can run on the more optimized control flow this pass creates in
  // many cases.
  addPass(AMDGPULowerBufferFatPointersPass(TM));
  addPass.requireCGSCCOrder();

  addPass(AMDGPULowerIntrinsicsPass(TM));

  // LowerSwitch pass may introduce unreachable blocks that can cause unexpected
  // behavior for subsequent passes. Placing it here seems better that these
  // blocks would get cleaned up by UnreachableBlockElim inserted next in the
  // pass flow.
  addPass(LowerSwitchPass());
}
```

<a name="block_7"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUTargetMachine.cpp (L2186-2222) [<sup>↩</sup>](#ref-block_7)

```cpp
void AMDGPUCodeGenPassBuilder::addPreISel(AddIRPass &addPass) const {

  if (TM.getOptLevel() > CodeGenOptLevel::None) {
    addPass(FlattenCFGPass());
    addPass(SinkingPass());
    addPass(AMDGPULateCodeGenPreparePass(TM));
  }

  // Merge divergent exit nodes. StructurizeCFG won't recognize the multi-exit
  // regions formed by them.

  addPass(AMDGPUUnifyDivergentExitNodesPass());
  addPass(FixIrreduciblePass());
  addPass(UnifyLoopExitsPass());
  addPass(StructurizeCFGPass(/*SkipUniformRegions=*/false));

  addPass(AMDGPUAnnotateUniformValuesPass());

  addPass(SIAnnotateControlFlowPass(TM));

  // TODO: Move this right after structurizeCFG to avoid extra divergence
  // analysis. This depends on stopping SIAnnotateControlFlow from making
  // control flow modifications.
  addPass(AMDGPURewriteUndefForPHIPass());

  if (!getCGPassBuilderOption().EnableGlobalISelOption ||
      !isGlobalISelAbortEnabled() || !NewRegBankSelect)
    addPass(LCSSAPass());

  if (TM.getOptLevel() > CodeGenOptLevel::Less)
    addPass(AMDGPUPerfHintAnalysisPass(TM));

  // FIXME: Why isn't this queried as required from AMDGPUISelDAGToDAG, and why
  // isn't this in addInstSelector?
  addPass(RequireAnalysisPass<UniformityInfoAnalysis, Function>(),
          /*Force=*/true);
}
```

<a name="block_8"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUTargetMachine.cpp (L2236-2241) [<sup>↩</sup>](#ref-block_8)

```cpp
Error AMDGPUCodeGenPassBuilder::addInstSelector(AddMachinePass &addPass) const {
  addPass(AMDGPUISelDAGToDAGPass(TM));
  addPass(SIFixSGPRCopiesPass());
  addPass(SILowerI1CopiesPass());
  return Error::success();
}
```

<a name="block_9"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUTargetMachine.cpp (L2243-2248) [<sup>↩</sup>](#ref-block_9)

```cpp
void AMDGPUCodeGenPassBuilder::addPreRewrite(AddMachinePass &addPass) const {
  if (EnableRegReassign) {
    addPass(GCNNSAReassignPass());
  }
}

```

<a name="block_10"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUTargetMachine.cpp (L2249-2266) [<sup>↩</sup>](#ref-block_10) [<sup>↩</sup>](#ref-block_10)

```cpp
void AMDGPUCodeGenPassBuilder::addMachineSSAOptimization(
    AddMachinePass &addPass) const {
  Base::addMachineSSAOptimization(addPass);

  addPass(SIFoldOperandsPass());
  if (EnableDPPCombine) {
    addPass(GCNDPPCombinePass());
  }
  addPass(SILoadStoreOptimizerPass());
  if (isPassEnabled(EnableSDWAPeephole)) {
    addPass(SIPeepholeSDWAPass());
    addPass(EarlyMachineLICMPass());
    addPass(MachineCSEPass());
    addPass(SIFoldOperandsPass());
  }
  addPass(DeadMachineInstructionElimPass());
  addPass(SIShrinkInstructionsPass());
}
```

<a name="block_11"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUTargetMachine.cpp (L2268-2305) [<sup>↩</sup>](#ref-block_11)

```cpp
void AMDGPUCodeGenPassBuilder::addOptimizedRegAlloc(
    AddMachinePass &addPass) const {
  if (EnableDCEInRA)
    insertPass<DetectDeadLanesPass>(DeadMachineInstructionElimPass());

  // FIXME: when an instruction has a Killed operand, and the instruction is
  // inside a bundle, seems only the BUNDLE instruction appears as the Kills of
  // the register in LiveVariables, this would trigger a failure in verifier,
  // we should fix it and enable the verifier.
  if (OptVGPRLiveRange)
    insertPass<RequireAnalysisPass<LiveVariablesAnalysis, MachineFunction>>(
        SIOptimizeVGPRLiveRangePass());

  // This must be run immediately after phi elimination and before
  // TwoAddressInstructions, otherwise the processing of the tied operand of
  // SI_ELSE will introduce a copy of the tied operand source after the else.
  insertPass<PHIEliminationPass>(SILowerControlFlowPass());

  if (EnableRewritePartialRegUses)
    insertPass<RenameIndependentSubregsPass>(GCNRewritePartialRegUsesPass());

  if (isPassEnabled(EnablePreRAOptimizations))
    insertPass<MachineSchedulerPass>(GCNPreRAOptimizationsPass());

  // Allow the scheduler to run before SIWholeQuadMode inserts exec manipulation
  // instructions that cause scheduling barriers.
  insertPass<MachineSchedulerPass>(SIWholeQuadModePass());

  if (OptExecMaskPreRA)
    insertPass<MachineSchedulerPass>(SIOptimizeExecMaskingPreRAPass());

  // This is not an essential optimization and it has a noticeable impact on
  // compilation time, so we only enable it from O2.
  if (TM.getOptLevel() > CodeGenOptLevel::Less)
    insertPass<MachineSchedulerPass>(SIFormMemoryClausesPass());

  Base::addOptimizedRegAlloc(addPass);
}
```

<a name="block_12"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUTargetMachine.cpp (L2313-2352) [<sup>↩</sup>](#ref-block_12) [<sup>↩</sup>](#ref-block_12)

```cpp
    AddMachinePass &addPass) const {
  // TODO: Check --regalloc-npm option

  addPass(GCNPreRALongBranchRegPass());

  addPass(RAGreedyPass({onlyAllocateSGPRs, "sgpr"}));

  // Commit allocated register changes. This is mostly necessary because too
  // many things rely on the use lists of the physical registers, such as the
  // verifier. This is only necessary with allocators which use LiveIntervals,
  // since FastRegAlloc does the replacements itself.
  addPass(VirtRegRewriterPass(false));

  // At this point, the sgpr-regalloc has been done and it is good to have the
  // stack slot coloring to try to optimize the SGPR spill stack indices before
  // attempting the custom SGPR spill lowering.
  addPass(StackSlotColoringPass());

  // Equivalent of PEI for SGPRs.
  addPass(SILowerSGPRSpillsPass());

  // To Allocate wwm registers used in whole quad mode operations (for shaders).
  addPass(SIPreAllocateWWMRegsPass());

  // For allocating other wwm register operands.
  addPass(RAGreedyPass({onlyAllocateWWMRegs, "wwm"}));
  addPass(SILowerWWMCopiesPass());
  addPass(VirtRegRewriterPass(false));
  addPass(AMDGPUReserveWWMRegsPass());

  // For allocating per-thread VGPRs.
  addPass(RAGreedyPass({onlyAllocateVGPRs, "vgpr"}));


  addPreRewrite(addPass);
  addPass(VirtRegRewriterPass(true));

  addPass(AMDGPUMarkLastScratchLoadPass());
  return Error::success();
}
```

<a name="block_13"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUTargetMachine.cpp (L2354-2359)

```cpp
void AMDGPUCodeGenPassBuilder::addPostRegAlloc(AddMachinePass &addPass) const {
  addPass(SIFixVGPRCopiesPass());
  if (TM.getOptLevel() > CodeGenOptLevel::None)
    addPass(SIOptimizeExecMaskingPass());
  Base::addPostRegAlloc(addPass);
}
```

<a name="block_14"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUTargetMachine.cpp (L2361-2365) [<sup>↩</sup>](#ref-block_14)

```cpp
void AMDGPUCodeGenPassBuilder::addPreSched2(AddMachinePass &addPass) const {
  if (TM.getOptLevel() > CodeGenOptLevel::None)
    addPass(SIShrinkInstructionsPass());
  addPass(SIPostRABundlerPass());
}
```

<a name="block_15"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUTargetMachine.cpp (L2367-2406) [<sup>↩</sup>](#ref-block_15)

```cpp
void AMDGPUCodeGenPassBuilder::addPreEmitPass(AddMachinePass &addPass) const {
  if (isPassEnabled(EnableVOPD, CodeGenOptLevel::Less)) {
    addPass(GCNCreateVOPDPass());
  }

  addPass(SIMemoryLegalizerPass());
  addPass(SIInsertWaitcntsPass());

  // TODO: addPass(SIModeRegisterPass());

  if (TM.getOptLevel() > CodeGenOptLevel::None) {
    // TODO: addPass(SIInsertHardClausesPass());
  }

  addPass(SILateBranchLoweringPass());

  if (isPassEnabled(EnableSetWavePriority, CodeGenOptLevel::Less))
    addPass(AMDGPUSetWavePriorityPass());

  if (TM.getOptLevel() > CodeGenOptLevel::None)
    addPass(SIPreEmitPeepholePass());

  // The hazard recognizer that runs as part of the post-ra scheduler does not
  // guarantee to be able handle all hazards correctly. This is because if there
  // are multiple scheduling regions in a basic block, the regions are scheduled
  // bottom up, so when we begin to schedule a region we don't know what
  // instructions were emitted directly before it.
  //
  // Here we add a stand-alone hazard recognizer pass which can handle all
  // cases.
  addPass(PostRAHazardRecognizerPass());
  addPass(AMDGPUWaitSGPRHazardsPass());
  addPass(AMDGPULowerVGPREncodingPass());

  if (isPassEnabled(EnableInsertDelayAlu, CodeGenOptLevel::Less)) {
    addPass(AMDGPUInsertDelayAluPass());
  }

  addPass(BranchRelaxationPass());
}
```

<a name="block_16"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUPassRegistry.def (L16-140) [<sup>↩</sup>](#ref-block_16)

```text
#ifndef MODULE_PASS
#define MODULE_PASS(NAME, CREATE_PASS)
#endif
MODULE_PASS("amdgpu-expand-feature-predicates",
            AMDGPUExpandFeaturePredicatesPass(*this))
MODULE_PASS("amdgpu-always-inline", AMDGPUAlwaysInlinePass())
MODULE_PASS("amdgpu-export-kernel-runtime-handles", AMDGPUExportKernelRuntimeHandlesPass())
MODULE_PASS("amdgpu-lower-buffer-fat-pointers",
            AMDGPULowerBufferFatPointersPass(*this))
MODULE_PASS("amdgpu-lower-ctor-dtor", AMDGPUCtorDtorLoweringPass())
MODULE_PASS("amdgpu-lower-module-lds", AMDGPULowerModuleLDSPass(*this))
MODULE_PASS("amdgpu-perf-hint",
            AMDGPUPerfHintAnalysisPass(
              *static_cast<const GCNTargetMachine *>(this)))
MODULE_PASS("amdgpu-preload-kernel-arguments", AMDGPUPreloadKernelArgumentsPass(*this))
MODULE_PASS("amdgpu-printf-runtime-binding", AMDGPUPrintfRuntimeBindingPass())
MODULE_PASS("amdgpu-remove-incompatible-functions", AMDGPURemoveIncompatibleFunctionsPass(*this))
MODULE_PASS("amdgpu-sw-lower-lds", AMDGPUSwLowerLDSPass(*this))
MODULE_PASS("amdgpu-unify-metadata", AMDGPUUnifyMetadataPass())
MODULE_PASS("amdgpu-expand-feature-predicates",
            AMDGPUExpandFeaturePredicatesPass(*this))
#undef MODULE_PASS

#ifndef MODULE_PASS_WITH_PARAMS
#define MODULE_PASS_WITH_PARAMS(NAME, CLASS, CREATE_PASS, PARSER, PARAMS)
#endif
MODULE_PASS_WITH_PARAMS(
    "amdgpu-attributor", "AMDGPUAttributorPass",
    [=](AMDGPUAttributorOptions Options) {
      return AMDGPUAttributorPass(*this, Options);
    },
    parseAMDGPUAttributorPassOptions, "closed-world")
#undef MODULE_PASS_WITH_PARAMS

#ifndef FUNCTION_PASS
#define FUNCTION_PASS(NAME, CREATE_PASS)
#endif
FUNCTION_PASS("amdgpu-annotate-uniform", AMDGPUAnnotateUniformValuesPass())
FUNCTION_PASS("amdgpu-codegenprepare", AMDGPUCodeGenPreparePass(*this))
FUNCTION_PASS("amdgpu-image-intrinsic-opt",
              AMDGPUImageIntrinsicOptimizerPass(*this))
FUNCTION_PASS("amdgpu-late-codegenprepare",
              AMDGPULateCodeGenPreparePass(
                *static_cast<const GCNTargetMachine *>(this)))
FUNCTION_PASS("amdgpu-lower-kernel-arguments",
              AMDGPULowerKernelArgumentsPass(*this))
FUNCTION_PASS("amdgpu-lower-kernel-attributes",
              AMDGPULowerKernelAttributesPass())
FUNCTION_PASS("amdgpu-promote-alloca", AMDGPUPromoteAllocaPass(*this))
FUNCTION_PASS("amdgpu-promote-alloca-to-vector",
              AMDGPUPromoteAllocaToVectorPass(*this))
FUNCTION_PASS("amdgpu-promote-kernel-arguments",
              AMDGPUPromoteKernelArgumentsPass())
FUNCTION_PASS("amdgpu-rewrite-undef-for-phi", AMDGPURewriteUndefForPHIPass())
FUNCTION_PASS("amdgpu-simplifylib", AMDGPUSimplifyLibCallsPass())
FUNCTION_PASS("amdgpu-unify-divergent-exit-nodes",
              AMDGPUUnifyDivergentExitNodesPass())
FUNCTION_PASS("amdgpu-usenative", AMDGPUUseNativeCallsPass())
FUNCTION_PASS("si-annotate-control-flow", SIAnnotateControlFlowPass(*static_cast<const GCNTargetMachine *>(this)))
#undef FUNCTION_PASS

#ifndef FUNCTION_ANALYSIS
#define FUNCTION_ANALYSIS(NAME, CREATE_PASS)
#endif

#ifndef FUNCTION_ALIAS_ANALYSIS
#define FUNCTION_ALIAS_ANALYSIS(NAME, CREATE_PASS)                             \
  FUNCTION_ANALYSIS(NAME, CREATE_PASS)
#endif
FUNCTION_ALIAS_ANALYSIS("amdgpu-aa", AMDGPUAA())
#undef FUNCTION_ALIAS_ANALYSIS
#undef FUNCTION_ANALYSIS

#ifndef FUNCTION_PASS_WITH_PARAMS
#define FUNCTION_PASS_WITH_PARAMS(NAME, CLASS, CREATE_PASS, PARSER, PARAMS)
#endif
FUNCTION_PASS_WITH_PARAMS(
    "amdgpu-atomic-optimizer",
    "AMDGPUAtomicOptimizerPass",
    [=](ScanOptions Strategy) {
      return AMDGPUAtomicOptimizerPass(*this, Strategy);
    },
    parseAMDGPUAtomicOptimizerStrategy, "strategy=dpp|iterative|none")
#undef FUNCTION_PASS_WITH_PARAMS

#ifndef MACHINE_FUNCTION_PASS
#define MACHINE_FUNCTION_PASS(NAME, CREATE_PASS)
#endif
MACHINE_FUNCTION_PASS("amdgpu-insert-delay-alu", AMDGPUInsertDelayAluPass())
MACHINE_FUNCTION_PASS("amdgpu-isel", AMDGPUISelDAGToDAGPass(*this))
MACHINE_FUNCTION_PASS("amdgpu-mark-last-scratch-load", AMDGPUMarkLastScratchLoadPass())
MACHINE_FUNCTION_PASS("amdgpu-pre-ra-long-branch-reg", GCNPreRALongBranchRegPass())
MACHINE_FUNCTION_PASS("amdgpu-reserve-wwm-regs", AMDGPUReserveWWMRegsPass())
MACHINE_FUNCTION_PASS("amdgpu-rewrite-agpr-copy-mfma", AMDGPURewriteAGPRCopyMFMAPass())
MACHINE_FUNCTION_PASS("amdgpu-rewrite-partial-reg-uses", GCNRewritePartialRegUsesPass())
MACHINE_FUNCTION_PASS("amdgpu-set-wave-priority", AMDGPUSetWavePriorityPass())
MACHINE_FUNCTION_PASS("amdgpu-pre-ra-optimizations", GCNPreRAOptimizationsPass())
MACHINE_FUNCTION_PASS("amdgpu-preload-kern-arg-prolog", AMDGPUPreloadKernArgPrologPass())
MACHINE_FUNCTION_PASS("amdgpu-nsa-reassign", GCNNSAReassignPass())
MACHINE_FUNCTION_PASS("gcn-create-vopd", GCNCreateVOPDPass())
MACHINE_FUNCTION_PASS("gcn-dpp-combine", GCNDPPCombinePass())
MACHINE_FUNCTION_PASS("si-fix-sgpr-copies", SIFixSGPRCopiesPass())
MACHINE_FUNCTION_PASS("si-fix-vgpr-copies", SIFixVGPRCopiesPass())
MACHINE_FUNCTION_PASS("si-fold-operands", SIFoldOperandsPass());
MACHINE_FUNCTION_PASS("si-form-memory-clauses", SIFormMemoryClausesPass())
MACHINE_FUNCTION_PASS("si-i1-copies", SILowerI1CopiesPass())
MACHINE_FUNCTION_PASS("si-insert-hard-clauses", SIInsertHardClausesPass())
MACHINE_FUNCTION_PASS("si-insert-waitcnts", SIInsertWaitcntsPass())
MACHINE_FUNCTION_PASS("si-late-branch-lowering", SILateBranchLoweringPass())
MACHINE_FUNCTION_PASS("si-load-store-opt", SILoadStoreOptimizerPass())
MACHINE_FUNCTION_PASS("si-lower-control-flow", SILowerControlFlowPass())
MACHINE_FUNCTION_PASS("si-lower-sgpr-spills", SILowerSGPRSpillsPass())
MACHINE_FUNCTION_PASS("si-lower-wwm-copies", SILowerWWMCopiesPass())
MACHINE_FUNCTION_PASS("si-memory-legalizer", SIMemoryLegalizerPass())
MACHINE_FUNCTION_PASS("si-mode-register", SIModeRegisterPass())
MACHINE_FUNCTION_PASS("si-opt-vgpr-liverange", SIOptimizeVGPRLiveRangePass())
MACHINE_FUNCTION_PASS("si-optimize-exec-masking", SIOptimizeExecMaskingPass())
MACHINE_FUNCTION_PASS("si-optimize-exec-masking-pre-ra", SIOptimizeExecMaskingPreRAPass())
MACHINE_FUNCTION_PASS("si-peephole-sdwa", SIPeepholeSDWAPass())
MACHINE_FUNCTION_PASS("si-post-ra-bundler", SIPostRABundlerPass())
MACHINE_FUNCTION_PASS("si-pre-allocate-wwm-regs", SIPreAllocateWWMRegsPass())
MACHINE_FUNCTION_PASS("si-pre-emit-peephole", SIPreEmitPeepholePass())
MACHINE_FUNCTION_PASS("si-shrink-instructions", SIShrinkInstructionsPass())
MACHINE_FUNCTION_PASS("si-wqm", SIWholeQuadModePass())
#undef MACHINE_FUNCTION_PASS
```

<a name="block_17"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUPassRegistry.def (L143-149) [<sup>↩</sup>](#ref-block_17)

```text
// Global ISel passes
DUMMY_MACHINE_FUNCTION_PASS("amdgpu-prelegalizer-combiner", AMDGPUPreLegalizerCombinerPass())
DUMMY_MACHINE_FUNCTION_PASS("amdgpu-postlegalizer-combiner", AMDGPUPostLegalizerCombinerPass())
DUMMY_MACHINE_FUNCTION_PASS("amdgpu-global-isel-divergence-lowering", AMDGPUGlobalISelDivergenceLoweringPass())
DUMMY_MACHINE_FUNCTION_PASS("amdgpu-regbankselect", AMDGPURegBankSelectPass())
DUMMY_MACHINE_FUNCTION_PASS("amdgpu-regbanklegalize", AMDGPURegBankLegalizePass())
DUMMY_MACHINE_FUNCTION_PASS("amdgpu-regbank-combiner", AMDGPURegBankCombinerPass())
```

<a name="block_18"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUISelDAGToDAG.h (L63-86) [<sup>↩</sup>](#ref-block_18)

```text
/// AMDGPU specific code to select AMDGPU machine instructions for
/// SelectionDAG operations.
class AMDGPUDAGToDAGISel : public SelectionDAGISel {
  // Subtarget - Keep a pointer to the AMDGPU Subtarget around so that we can
  // make the right decision when generating code for different targets.
  const GCNSubtarget *Subtarget;

  // Default FP mode for the current function.
  SIModeRegisterDefaults Mode;

  // Instructions that will be lowered with a final instruction that zeros the
  // high result bits.
  bool fp16SrcZerosHighBits(unsigned Opc) const;

public:
  AMDGPUDAGToDAGISel() = delete;

  explicit AMDGPUDAGToDAGISel(TargetMachine &TM, CodeGenOptLevel OptLevel);

  bool runOnMachineFunction(MachineFunction &MF) override;
  bool matchLoadD16FromBuildVector(SDNode *N) const;
  void PreprocessISelDAG() override;
  void Select(SDNode *N) override;
  void PostprocessISelDAG() override;
```

