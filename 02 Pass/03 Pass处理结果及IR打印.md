# 在AMDGPU后端各个Pass执行后打印IR的方法

有两种主要方式，取决于你使用的是Legacy Pass Manager还是New Pass Manager。

## 方法一：使用命令行选项（最简单）

直接使用LLVM提供的命令行选项：

- **打印所有pass之后的IR**：`-print-after-all`
- **打印特定pass之后的IR**：`-print-after=<pass-name>`
- **打印所有pass之前的IR**：`-print-before-all`
- **过滤特定函数**：`-filter-print-funcs=<function-name>`

<a name="ref-block_0"></a>这些选项定义在： llvm-project:20-34[<sup>↗</sup>](#block_0) 

## 方法二：Legacy Pass Manager代码实现

<a name="ref-block_2"></a>Legacy Pass Manager会自动在pass pipeline中插入打印pass。核心逻辑在 `schedulePass` 函数中： llvm-project:739-757[<sup>↗</sup>](#block_2) 

<a name="ref-block_1"></a>判断是否需要打印的函数： llvm-project:136-138[<sup>↗</sup>](#block_1) 

## 方法三：New Pass Manager代码实现

New Pass Manager使用 `StandardInstrumentations` 类来处理IR打印。参考实现位置：

<a name="ref-block_4"></a>**StandardInstrumentations的定义**： llvm-project:599-630[<sup>↗</sup>](#block_4) 

<a name="ref-block_3"></a>**PrintPassOptions配置结构**： llvm-project:124-131[<sup>↗</sup>](#block_3) 

<a name="ref-block_5"></a>**实际使用示例**（在Clang后端工具中）： llvm-project:916-926[<sup>↗</sup>](#block_5) 

## AMDGPU特定信息

<a name="ref-block_6"></a>AMDGPU的所有passes注册在： llvm-project:1-150[<sup>↗</sup>](#block_6) 

<a name="ref-block_7"></a>AMDGPU Target Machine的pass pipeline配置： llvm-project:842-845[<sup>↗</sup>](#block_7) 

## Notes

1. **命令行使用示例**：
   ```bash
   opt -print-after-all <input.ll> -o <output.bc>
   clang -mllvm -print-after-all input.c
   ```

2. **Pass名称**：AMDGPU的pass名称可以在 `AMDGPUPassRegistry.def` 中找到，例如 `amdgpu-codegenprepare`、`si-annotate-control-flow` 等。

3. **New Pass Manager** 是LLVM的新架构，推荐用于新代码。Legacy Pass Manager已经被逐步淘汰。

4. 如果你需要在自己的工具中实现类似功能，参考 `BackendUtil.cpp` 中的 `StandardInstrumentations` 使用方式，创建 `PassInstrumentationCallbacks` 并配置 `PrintPassOptions`。


### Citations


<a name="block_0"></a>**File:** llvm/lib/IR/PrintPasses.cpp (L20-34) [<sup>↩</sup>](#ref-block_0)

```cpp
static cl::list<std::string>
    PrintBefore("print-before",
                llvm::cl::desc("Print IR before specified passes"),
                cl::CommaSeparated, cl::Hidden);

static cl::list<std::string>
    PrintAfter("print-after", llvm::cl::desc("Print IR after specified passes"),
               cl::CommaSeparated, cl::Hidden);

static cl::opt<bool> PrintBeforeAll("print-before-all",
                                    llvm::cl::desc("Print IR before each pass"),
                                    cl::init(false), cl::Hidden);
static cl::opt<bool> PrintAfterAll("print-after-all",
                                   llvm::cl::desc("Print IR after each pass"),
                                   cl::init(false), cl::Hidden);
```

<a name="block_1"></a>**File:** llvm/lib/IR/PrintPasses.cpp (L136-138) [<sup>↩</sup>](#ref-block_1)

```cpp
bool llvm::shouldPrintAfterPass(StringRef PassID) {
  return PrintAfterAll || shouldPrintBeforeOrAfterPass(PassID, PrintAfter);
}
```

<a name="block_2"></a>**File:** llvm/lib/IR/LegacyPassManager.cpp (L739-757) [<sup>↩</sup>](#ref-block_2)

```cpp
  if (PI && !PI->isAnalysis() && shouldPrintBeforePass(PI->getPassArgument())) {
    Pass *PP =
        P->createPrinterPass(dbgs(), ("*** IR Dump Before " + P->getPassName() +
                                      " (" + PI->getPassArgument() + ") ***")
                                         .str());
    PP->assignPassManager(activeStack, getTopLevelPassManagerType());
  }

  // Add the requested pass to the best available pass manager.
  P->assignPassManager(activeStack, getTopLevelPassManagerType());

  if (PI && !PI->isAnalysis() && shouldPrintAfterPass(PI->getPassArgument())) {
    Pass *PP =
        P->createPrinterPass(dbgs(), ("*** IR Dump After " + P->getPassName() +
                                      " (" + PI->getPassArgument() + ") ***")
                                         .str());
    PP->assignPassManager(activeStack, getTopLevelPassManagerType());
  }
}
```

<a name="block_3"></a>**File:** llvm/include/llvm/Passes/StandardInstrumentations.h (L124-131) [<sup>↩</sup>](#ref-block_3)

```text
struct PrintPassOptions {
  /// Print adaptors and pass managers.
  bool Verbose = false;
  /// Don't print information for analyses.
  bool SkipAnalyses = false;
  /// Indent based on hierarchy.
  bool Indent = false;
};
```

<a name="block_4"></a>**File:** llvm/include/llvm/Passes/StandardInstrumentations.h (L599-630) [<sup>↩</sup>](#ref-block_4)

```text
class StandardInstrumentations {
  PrintIRInstrumentation PrintIR;
  PrintPassInstrumentation PrintPass;
  TimePassesHandler TimePasses;
  TimeProfilingPassesHandler TimeProfilingPasses;
  OptNoneInstrumentation OptNone;
  OptPassGateInstrumentation OptPassGate;
  PreservedCFGCheckerInstrumentation PreservedCFGChecker;
  IRChangedPrinter PrintChangedIR;
  PseudoProbeVerifier PseudoProbeVerification;
  InLineChangePrinter PrintChangedDiff;
  DotCfgChangeReporter WebsiteChangeReporter;
  PrintCrashIRInstrumentation PrintCrashIR;
  IRChangedTester ChangeTester;
  VerifyInstrumentation Verify;
  DroppedVariableStatsIR DroppedStatsIR;

  bool VerifyEach;

public:
  LLVM_ABI
  StandardInstrumentations(LLVMContext &Context, bool DebugLogging,
                           bool VerifyEach = false,
                           PrintPassOptions PrintPassOpts = PrintPassOptions());

  // Register all the standard instrumentation callbacks. If \p FAM is nullptr
  // then PreservedCFGChecker is not enabled.
  LLVM_ABI void registerCallbacks(PassInstrumentationCallbacks &PIC,
                                  ModuleAnalysisManager *MAM = nullptr);

  TimePassesHandler &getTimePasses() { return TimePasses; }
};
```

<a name="block_5"></a>**File:** clang/lib/CodeGen/BackendUtil.cpp (L916-926) [<sup>↩</sup>](#ref-block_5)

```cpp
  bool DebugPassStructure = CodeGenOpts.DebugPass == "Structure";
  PassInstrumentationCallbacks PIC;
  PrintPassOptions PrintPassOpts;
  PrintPassOpts.Indent = DebugPassStructure;
  PrintPassOpts.SkipAnalyses = DebugPassStructure;
  StandardInstrumentations SI(
      TheModule->getContext(),
      (CodeGenOpts.DebugPassManager || DebugPassStructure),
      CodeGenOpts.VerifyEach, PrintPassOpts);
  SI.registerCallbacks(PIC, &MAM);
  PassBuilder PB(TM.get(), PTO, PGOOpt, &PIC);
```

<a name="block_6"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUPassRegistry.def (L1-150) [<sup>↩</sup>](#ref-block_6)

```text
//===- AMDGPUPassRegistry.def - Registry of AMDGPU passes -------*- C++ -*-===//
//
// Part of the LLVM Project, under the Apache License v2.0 with LLVM Exceptions.
// See https://llvm.org/LICENSE.txt for license information.
// SPDX-License-Identifier: Apache-2.0 WITH LLVM-exception
//
//===----------------------------------------------------------------------===//
//
// This file is used as the registry of passes that are part of the
// AMDGPU backend.
//
//===----------------------------------------------------------------------===//

// NOTE: NO INCLUDE GUARD DESIRED!

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

#define DUMMY_MACHINE_FUNCTION_PASS(NAME, CREATE_PASS)
// Global ISel passes
DUMMY_MACHINE_FUNCTION_PASS("amdgpu-prelegalizer-combiner", AMDGPUPreLegalizerCombinerPass())
DUMMY_MACHINE_FUNCTION_PASS("amdgpu-postlegalizer-combiner", AMDGPUPostLegalizerCombinerPass())
DUMMY_MACHINE_FUNCTION_PASS("amdgpu-global-isel-divergence-lowering", AMDGPUGlobalISelDivergenceLoweringPass())
DUMMY_MACHINE_FUNCTION_PASS("amdgpu-regbankselect", AMDGPURegBankSelectPass())
DUMMY_MACHINE_FUNCTION_PASS("amdgpu-regbanklegalize", AMDGPURegBankLegalizePass())
DUMMY_MACHINE_FUNCTION_PASS("amdgpu-regbank-combiner", AMDGPURegBankCombinerPass())

```

<a name="block_7"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUTargetMachine.cpp (L842-845) [<sup>↩</sup>](#ref-block_7)

```cpp
void AMDGPUTargetMachine::registerPassBuilderCallbacks(PassBuilder &PB) {

#define GET_PASS_REGISTRY "AMDGPUPassRegistry.def"
#include "llvm/Passes/TargetPassRegistry.inc"
```

