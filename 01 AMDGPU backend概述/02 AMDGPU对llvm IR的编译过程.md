
# 基于LLVM IR生成到AMDGPU可执行文件的LLVM后端编译过程

## 1. 模块级优化阶段（Module-Level Optimization）

这一阶段对整个LLVM模块进行优化，主要涉及AMDGPU特定的模块级转换。关键的优化pass包括：

<a name="ref-block_1"></a>- **特性谓词展开**：`amdgpu-expand-feature-predicates` pass用于展开GPU特定的特性谓词 llvm-project:19-20[<sup>↗</sup>](#block_1) 
<a name="ref-block_2"></a>- **内联优化**：`amdgpu-always-inline` pass强制内联所有函数调用 llvm-project:21-21[<sup>↗</sup>](#block_2) 
<a name="ref-block_4"></a>- **LDS（Local Data Share）处理**：`amdgpu-lower-module-lds` pass用于处理模块级的本地数据共享 llvm-project:26-26[<sup>↗</sup>](#block_4) 
<a name="ref-block_3"></a>- **缓冲区指针处理**：`amdgpu-lower-buffer-fat-pointers` pass处理缓冲区fat指针的lowering llvm-project:23-24[<sup>↗</sup>](#block_3) 

## 2. 函数级优化阶段（Function-Level Optimization）

这一阶段对每个函数进行AMDGPU特定的优化：

<a name="ref-block_5"></a>- **统一值标注**：`amdgpu-annotate-uniform` pass标注统一值 llvm-project:53-53[<sup>↗</sup>](#block_5) 
<a name="ref-block_6"></a>- **代码生成准备**：`amdgpu-codegenprepare` pass为代码生成做准备 llvm-project:54-54[<sup>↗</sup>](#block_6) 
<a name="ref-block_8"></a>- **内存分配提升**：`amdgpu-promote-alloca` pass将栈分配提升到寄存器或向量 llvm-project:64-64[<sup>↗</sup>](#block_8) 
<a name="ref-block_7"></a>- **内核参数处理**：`amdgpu-lower-kernel-arguments` pass处理内核参数的lowering llvm-project:60-61[<sup>↗</sup>](#block_7) 
<a name="ref-block_9"></a>- **控制流标注**：`si-annotate-control-flow` pass为Southern Islands架构标注控制流 llvm-project:74-74[<sup>↗</sup>](#block_9) 

## 3. 指令选择阶段（Instruction Selection）

LLVM为AMDGPU提供两种指令选择路径：

<a name="ref-block_16"></a>- **传统SelectionDAG路径**：通过SIISelLowering.cpp实现指令选择和lowering llvm-project:1-1[<sup>↗</sup>](#block_16) 
<a name="ref-block_16"></a>- **现代GlobalISel路径**：通过AMDGPUInstructionSelector实现指令选择 llvm-project:1-1[<sup>↗</sup>](#block_16) 
<a name="ref-block_10"></a>- **指令选择pass**：`amdgpu-isel` pass执行DAG到DAG的指令选择 llvm-project:105-105[<sup>↗</sup>](#block_10) 

## 4. 机器函数级优化阶段（Machine Function-Level Optimization）

这一阶段对机器指令进行AMDGPU特定的优化：

<a name="ref-block_14"></a>- **等待指令插入**：`si-insert-waitcnts` pass插入内存和执行同步的等待指令 llvm-project:123-123[<sup>↗</sup>](#block_14) 
<a name="ref-block_11"></a>- **寄存器复制优化**：`si-fix-sgpr-copies` 和 `si-fix-vgpr-copies` 优化标量和向量寄存器复制 llvm-project:117-118[<sup>↗</sup>](#block_11) 
<a name="ref-block_12"></a>- **指令折叠**：`si-fold-operands` pass进行操作数折叠优化 llvm-project:119-119[<sup>↗</sup>](#block_12) 
<a name="ref-block_13"></a>- **内存子句形成**：`si-form-memory-clauses` pass形成内存访问子句 llvm-project:120-120[<sup>↗</sup>](#block_13) 
<a name="ref-block_15"></a>- **控制流处理**：`si-lower-control-flow` pass降低控制流结构 llvm-project:126-126[<sup>↗</sup>](#block_15) 

## 5. 代码生成和发射阶段（Code Generation and Emission）

最终阶段将优化后的机器指令转换为汇编代码或目标文件：

<a name="ref-block_18"></a>- **后端编译驱动**：BackendUtil.cpp中的`addPassesToEmitFile`方法协调整个代码生成流程 llvm-project:632-633[<sup>↗</sup>](#block_18) 
<a name="ref-block_19"></a>- **目标定义**：AMDGPU.td定义了AMDGPU目标架构的基本信息和特性 llvm-project:1-100[<sup>↗</sup>](#block_19) 

## Notes

<a name="ref-block_0"></a>整个编译流程是高度模块化的，每个阶段都有明确的职责分工。AMDGPU后端通过AMDGPUPassRegistry.def统一管理所有的编译pass llvm-project:1-12[<sup>↗</sup>](#block_0) ，确保编译流程的完整性和正确性。这种设计使得LLVM能够针对AMD GPU的特殊架构特点（如wavefront执行模型、分层内存系统、SIMD执行单元等）进行深度优化。


### Citations


<a name="block_0"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUPassRegistry.def (L1-12) [<sup>↩</sup>](#ref-block_0)

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
```

<a name="block_1"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUPassRegistry.def (L19-20) [<sup>↩</sup>](#ref-block_1)

```text
MODULE_PASS("amdgpu-expand-feature-predicates",
            AMDGPUExpandFeaturePredicatesPass(*this))
```

<a name="block_2"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUPassRegistry.def (L21-21) [<sup>↩</sup>](#ref-block_2)

```text
MODULE_PASS("amdgpu-always-inline", AMDGPUAlwaysInlinePass())
```

<a name="block_3"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUPassRegistry.def (L23-24) [<sup>↩</sup>](#ref-block_3)

```text
MODULE_PASS("amdgpu-lower-buffer-fat-pointers",
            AMDGPULowerBufferFatPointersPass(*this))
```

<a name="block_4"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUPassRegistry.def (L26-26) [<sup>↩</sup>](#ref-block_4)

```text
MODULE_PASS("amdgpu-lower-module-lds", AMDGPULowerModuleLDSPass(*this))
```

<a name="block_5"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUPassRegistry.def (L53-53) [<sup>↩</sup>](#ref-block_5)

```text
FUNCTION_PASS("amdgpu-annotate-uniform", AMDGPUAnnotateUniformValuesPass())
```

<a name="block_6"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUPassRegistry.def (L54-54) [<sup>↩</sup>](#ref-block_6)

```text
FUNCTION_PASS("amdgpu-codegenprepare", AMDGPUCodeGenPreparePass(*this))
```

<a name="block_7"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUPassRegistry.def (L60-61) [<sup>↩</sup>](#ref-block_7)

```text
FUNCTION_PASS("amdgpu-lower-kernel-arguments",
              AMDGPULowerKernelArgumentsPass(*this))
```

<a name="block_8"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUPassRegistry.def (L64-64) [<sup>↩</sup>](#ref-block_8)

```text
FUNCTION_PASS("amdgpu-promote-alloca", AMDGPUPromoteAllocaPass(*this))
```

<a name="block_9"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUPassRegistry.def (L74-74) [<sup>↩</sup>](#ref-block_9)

```text
FUNCTION_PASS("si-annotate-control-flow", SIAnnotateControlFlowPass(*static_cast<const GCNTargetMachine *>(this)))
```

<a name="block_10"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUPassRegistry.def (L105-105) [<sup>↩</sup>](#ref-block_10)

```text
MACHINE_FUNCTION_PASS("amdgpu-isel", AMDGPUISelDAGToDAGPass(*this))
```

<a name="block_11"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUPassRegistry.def (L117-118) [<sup>↩</sup>](#ref-block_11)

```text
MACHINE_FUNCTION_PASS("si-fix-sgpr-copies", SIFixSGPRCopiesPass())
MACHINE_FUNCTION_PASS("si-fix-vgpr-copies", SIFixVGPRCopiesPass())
```

<a name="block_12"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUPassRegistry.def (L119-119) [<sup>↩</sup>](#ref-block_12)

```text
MACHINE_FUNCTION_PASS("si-fold-operands", SIFoldOperandsPass());
```

<a name="block_13"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUPassRegistry.def (L120-120) [<sup>↩</sup>](#ref-block_13)

```text
MACHINE_FUNCTION_PASS("si-form-memory-clauses", SIFormMemoryClausesPass())
```

<a name="block_14"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUPassRegistry.def (L123-123) [<sup>↩</sup>](#ref-block_14)

```text
MACHINE_FUNCTION_PASS("si-insert-waitcnts", SIInsertWaitcntsPass())
```

<a name="block_15"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUPassRegistry.def (L126-126) [<sup>↩</sup>](#ref-block_15)

```text
MACHINE_FUNCTION_PASS("si-lower-control-flow", SILowerControlFlowPass())
```

<a name="block_16"></a>**File:** llvm/lib/Target/AMDGPU/SIISelLowering.cpp (L1-1) [<sup>↩</sup>](#ref-block_16) [<sup>↩</sup>](#ref-block_16)

```cpp
//===-- SIISelLowering.cpp - SI DAG Lowering Implementation ---------------===//
```

<a name="block_17"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPUInstructionSelector.cpp (L1-1)

```cpp
//===- AMDGPUInstructionSelector.cpp ----------------------------*- C++ -*-==//
```

<a name="block_18"></a>**File:** clang/lib/CodeGen/BackendUtil.cpp (L632-633) [<sup>↩</sup>](#ref-block_18)

```cpp
  if (TM->addPassesToEmitFile(CodeGenPasses, OS, DwoOS, CGFT,
                              /*DisableVerify=*/!CodeGenOpts.VerifyModule)) {
```

<a name="block_19"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPU.td (L1-100) [<sup>↩</sup>](#ref-block_19)

```text
//===-- AMDGPU.td - AMDGPU Tablegen files --------*- tablegen -*-===//
//
// Part of the LLVM Project, under the Apache License v2.0 with LLVM Exceptions.
// See https://llvm.org/LICENSE.txt for license information.
// SPDX-License-Identifier: Apache-2.0 WITH LLVM-exception
//
//===------------------------------------------------------------===//

include "llvm/TableGen/SearchableTable.td"
include "llvm/Target/Target.td"
include "AMDGPUFeatures.td"
include "AMDGPUPredicateControl.td"

def p0 : PtrValueType<i64, 0>;
def p1 : PtrValueType<i64, 1>;
def p2 : PtrValueType<i32, 2>;
def p3 : PtrValueType<i32, 3>;
def p4 : PtrValueType<i64, 4>;
def p5 : PtrValueType<i32, 5>;
def p6 : PtrValueType<i32, 6>;

//===------------------------------------------------------------===//
// Subtarget Features (device properties)
//===------------------------------------------------------------===//

def FeatureFastFMAF32 : SubtargetFeature<"fast-fmaf",
  "FastFMAF32",
  "true",
  "Assuming f32 fma is at least as fast as mul + add"
>;

def FeatureFastDenormalF32 : SubtargetFeature<"fast-denormal-f32",
  "FastDenormalF32",
  "true",
  "Enabling denormals does not cause f32 instructions to run at f64 rates"
>;

def FeatureMIMG_R128 : SubtargetFeature<"mimg-r128",
  "MIMG_R128",
  "true",
  "Support 128-bit texture resources"
>;

def HalfRate64Ops : SubtargetFeature<"half-rate-64-ops",
  "HalfRate64Ops",
  "true",
  "Most fp64 instructions are half rate instead of quarter"
>;

def FullRate64Ops : SubtargetFeature<"full-rate-64-ops",
  "FullRate64Ops",
  "true",
  "Most fp64 instructions are full rate"
>;

def FeatureFlatAddressSpace : SubtargetFeature<"flat-address-space",
  "FlatAddressSpace",
  "true",
  "Support flat address space"
>;

def FeatureFlatInstOffsets : SubtargetFeature<"flat-inst-offsets",
  "FlatInstOffsets",
  "true",
  "Flat instructions have immediate offset addressing mode"
>;

def FeatureFlatGlobalInsts : SubtargetFeature<"flat-global-insts",
  "FlatGlobalInsts",
  "true",
  "Have global_* flat memory instructions"
>;

def FeatureFlatScratchInsts : SubtargetFeature<"flat-scratch-insts",
  "FlatScratchInsts",
  "true",
  "Have scratch_* flat memory instructions"
>;

def FeatureScalarFlatScratchInsts : SubtargetFeature<"scalar-flat-scratch-insts",
  "ScalarFlatScratchInsts",
  "true",
  "Have s_scratch_* flat memory instructions"
>;

def FeatureEnableFlatScratch : SubtargetFeature<"enable-flat-scratch",
  "EnableFlatScratch",
  "true",
  "Use scratch_* flat memory instructions to access scratch"
>;

def FeatureAddNoCarryInsts : SubtargetFeature<"add-no-carry-insts",
  "AddNoCarryInsts",
  "true",
  "Have VALU add/sub instructions without carry out"
>;

def FeatureUnalignedBufferAccess : SubtargetFeature<"unaligned-buffer-access",
  "UnalignedBufferAccess",
  "true",
```

## Q2

为何没有包含寄存器分配阶段？修正并补充相关内容


### Answer

您说得对，我之前的回答确实遗漏了寄存器分配这个关键阶段。让我修正并补充完整的AMDGPU后端编译流程。

基于LLVM IR生成到AMDGPU可执行文件的LLVM后端编译过程应该包括以下主要环节：

## 1. 模块级优化阶段（Module-Level Optimization）
<cite/>

## 2. 函数级优化阶段（Function-Level Optimization）
<cite/>

## 3. 指令选择阶段（Instruction Selection）
<cite/>

## 4. 寄存器分配阶段（Register Allocation）

这是我之前遗漏的关键阶段，AMDGPU后端有其独特的寄存器分配策略：

### 寄存器类定义
<a name="ref-block_20"></a>AMDGPU定义了多种寄存器类来处理不同类型的数据和操作： llvm-project:371-378[<sup>↗</sup>](#block_20) 

### 寄存器银行分配
<a name="ref-block_21"></a>AMDGPU有独特的寄存器银行约束，需要特殊的分配策略： llvm-project:14-26[<sup>↗</sup>](#block_21) 

### 寄存器溢出处理
<a name="ref-block_22"></a>当寄存器不足时，AMDGPU需要特殊的溢出策略： llvm-project:1837-1869[<sup>↗</sup>](#block_22) 

### 寄存器预留机制
<a name="ref-block_23"></a>AMDGPU需要预留特定寄存器用于特殊用途： llvm-project:765-777[<sup>↗</sup>](#block_23) 

## 5. 机器函数级优化阶段（Machine Function-Level Optimization）
<cite/>

## 6. 代码生成和发射阶段（Code Generation and Emission）
<cite/>

## Notes

AMDGPU的寄存器分配阶段特别复杂，因为它需要处理：
1. **多种寄存器银行**：SGPR（标量）、VGPR（向量）、AGPR（累加器）等不同类型的寄存器
2. **寄存器约束**：某些指令只能使用特定类型的寄存器
3. **溢出策略**：由于GPU寄存器资源有限，需要高效的溢出到内存的策略
<a name="ref-block_24"></a>4. **寄存器对齐**：某些操作需要寄存器对齐，如测试用例所示： llvm-project:2-5[<sup>↗</sup>](#block_24) 

这个阶段在指令选择之后、机器函数级优化之前进行，是AMDGPU后端编译流程中不可缺少的关键环节。

Wiki pages you might want to explore:
- [AMDGPU Backend (ROCm/llvm-project)](/wiki/ROCm/llvm-project#4.5)


### Citations


<a name="block_20"></a>**File:** llvm/lib/Target/AMDGPU/SIRegisterInfo.td (L371-378) [<sup>↩</sup>](#ref-block_20)

```text
def SGPR_32 : SIRegisterClass<"AMDGPU", [i32, f32, i16, f16, bf16, v2i16, v2f16, v2bf16], 32,
                            (add (sequence "SGPR%u", 0, 105))> {
  // Give all SGPR classes higher priority than VGPR classes, because
  // we want to spill SGPRs to VGPRs.
  let AllocationPriority = 0;
  let GeneratePressureSet = 0;
  let HasSGPR = 1;
}
```

<a name="block_21"></a>**File:** llvm/lib/Target/AMDGPU/AMDGPURegisterBankInfo.cpp (L14-26) [<sup>↩</sup>](#ref-block_21)

```cpp
/// AMDGPU has unique register bank constraints that require special high level
/// strategies to deal with. There are two main true physical register banks
/// VGPR (vector), and SGPR (scalar). Additionally the VCC register bank is a
/// sort of pseudo-register bank needed to represent SGPRs used in a vector
/// boolean context. There is also the AGPR bank, which is a special purpose
/// physical register bank present on some subtargets.
///
/// Copying from VGPR to SGPR is generally illegal, unless the value is known to
/// be uniform. It is generally not valid to legalize operands by inserting
/// copies as on other targets. Operations which require uniform, SGPR operands
/// generally require scalarization by repeatedly executing the instruction,
/// activating each set of lanes using a unique set of input values. This is
/// referred to as a waterfall loop.
```

<a name="block_22"></a>**File:** llvm/lib/Target/AMDGPU/SIInstrInfo.cpp (L1837-1869) [<sup>↩</sup>](#ref-block_22)

```cpp
static unsigned getSGPRSpillRestoreOpcode(unsigned Size) {
  switch (Size) {
  case 4:
    return AMDGPU::SI_SPILL_S32_RESTORE;
  case 8:
    return AMDGPU::SI_SPILL_S64_RESTORE;
  case 12:
    return AMDGPU::SI_SPILL_S96_RESTORE;
  case 16:
    return AMDGPU::SI_SPILL_S128_RESTORE;
  case 20:
    return AMDGPU::SI_SPILL_S160_RESTORE;
  case 24:
    return AMDGPU::SI_SPILL_S192_RESTORE;
  case 28:
    return AMDGPU::SI_SPILL_S224_RESTORE;
  case 32:
    return AMDGPU::SI_SPILL_S256_RESTORE;
  case 36:
    return AMDGPU::SI_SPILL_S288_RESTORE;
  case 40:
    return AMDGPU::SI_SPILL_S320_RESTORE;
  case 44:
    return AMDGPU::SI_SPILL_S352_RESTORE;
  case 48:
    return AMDGPU::SI_SPILL_S384_RESTORE;
  case 64:
    return AMDGPU::SI_SPILL_S512_RESTORE;
  case 128:
    return AMDGPU::SI_SPILL_S1024_RESTORE;
  default:
    llvm_unreachable("unknown register size");
  }
```

<a name="block_23"></a>**File:** llvm/lib/Target/AMDGPU/SIRegisterInfo.cpp (L765-777) [<sup>↩</sup>](#ref-block_23)

```cpp
  // Reserve all the AGPRs if there are no instructions to use it.
  if (!ST.hasMAIInsts())
    MaxNumAGPRs = 0;
  for (const TargetRegisterClass *RC : regclasses()) {
    if (RC->isBaseClass() && isAGPRClass(RC)) {
      unsigned NumRegs = divideCeil(getRegSizeInBits(*RC), 32);
      for (MCPhysReg Reg : *RC) {
        unsigned Index = getHWRegIndex(Reg);
        if (Index + NumRegs > MaxNumAGPRs)
          Reserved.set(Reg);
      }
    }
  }
```

<a name="block_24"></a>**File:** llvm/test/CodeGen/AMDGPU/partial-regcopy-and-spill-missed-at-regalloc.ll (L2-5) [<sup>↩</sup>](#ref-block_24)

```text
;RUN: llc -mtriple=amdgcn-amd-amdhsa -mcpu=gfx908 --stop-after=greedy,1 -verify-machineinstrs < %s | FileCheck -check-prefix=REGALLOC-GFX908 %s
;RUN: llc -mtriple=amdgcn-amd-amdhsa -mcpu=gfx908 --stop-after=prologepilog -verify-machineinstrs < %s | FileCheck -check-prefix=PEI-GFX908 %s
;RUN: llc -mtriple=amdgcn-amd-amdhsa -mcpu=gfx90a --stop-after=greedy,1 -verify-machineinstrs < %s | FileCheck -check-prefix=REGALLOC-GFX90A %s
;RUN: llc -mtriple=amdgcn-amd-amdhsa -mcpu=gfx90a --stop-after=prologepilog -verify-machineinstrs < %s | FileCheck -check-prefix=PEI-GFX90A %s
```

