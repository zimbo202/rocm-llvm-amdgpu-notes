# SILowerControlFlow.cpp 代码功能详解

## 1. Pass 的主要功能概括

<a name="ref-block_0"></a>这个 pass 的核心作用是**将 AMDGPU 的伪控制流指令降低（lower）为真实的机器指令**。 llvm-project:10-11[<sup>↗</sup>](#block_0) 

### 作用与效果：

- **使用谓词执行模型**：所有控制流通过谓词指令和谓词栈来处理，而不是传统的跳转指令 llvm-project:13-15 

- **EXEC 寄存器控制**：每个标量 ALU 控制 64 个向量 ALU 的操作，通过写入 64 位 EXEC 寄存器来更新任意向量 ALU 的谓词（每个位对应一个向量 ALU） llvm-project:14-17 

<a name="ref-block_2"></a>- **转换示例**：将高级伪指令（如 SI_IF、SI_ELSE、SI_END_CF）转换为实际的标量 ALU 指令（如 S_AND_SAVEEXEC_B64、S_OR_SAVEEXEC_B64、S_CBRANCH_EXECZ 等） llvm-project:22-48[<sup>↗</sup>](#block_2) 

## 2. 实现主要功能的步骤或子功能提取

通过遍历代码，可以提取出以下关键子功能：

1. **emitIf** - 处理 SI_IF 指令 llvm-project:100-100 
2. **emitElse** - 处理 SI_ELSE 指令 llvm-project:101-101 
3. **emitIfBreak** - 处理 SI_IF_BREAK 指令 llvm-project:102-102 
4. **emitLoop** - 处理 SI_LOOP 指令 llvm-project:103-103 
5. **emitEndCf** - 处理 SI_END_CF 指令 llvm-project:105-105 
<a name="ref-block_5"></a>6. **combineMasks** - 组合和优化掩码操作 llvm-project:110-110[<sup>↗</sup>](#block_5) 
<a name="ref-block_7"></a>7. **optimizeEndCf** - 优化移除冗余的 END_CF 指令 llvm-project:137-137[<sup>↗</sup>](#block_7) 
<a name="ref-block_6"></a>8. **process** - 主处理分发函数 llvm-project:114-114[<sup>↗</sup>](#block_6) 
<a name="ref-block_8"></a>9. **run** - Pass 的主入口函数 llvm-project:143-143[<sup>↗</sup>](#block_8) 
<a name="ref-block_3"></a>10. **hasKill** - 检查是否存在 kill 指令 llvm-project:98-98[<sup>↗</sup>](#block_3) 

## 3. 各步骤/子功能的具体描述分析

### 3.1 emitIf（处理条件分支的 IF 部分）

**功能**：将 SI_IF 伪指令转换为实际的机器指令序列，保存 EXEC 掩码并设置新的执行掩码。

**关键操作**：
<a name="ref-block_11"></a>- 检查是否为简单 IF（只有一个使用且直接连接 SI_END_CF） llvm-project:227-230[<sup>↗</sup>](#block_11) 
<a name="ref-block_13"></a>- 保存当前 EXEC 寄存器 llvm-project:243-246[<sup>↗</sup>](#block_13) 
<a name="ref-block_14"></a>- 执行 AND 操作计算新的 EXEC 掩码 llvm-project:251-254[<sup>↗</sup>](#block_14) 
<a name="ref-block_15"></a>- 对于非简单 IF，执行 XOR 操作保存被清除的位 llvm-project:260-267[<sup>↗</sup>](#block_15) 
<a name="ref-block_16"></a>- 插入条件分支指令 S_CBRANCH_EXECZ llvm-project:283-284[<sup>↗</sup>](#block_16) 

### 3.2 emitElse（处理条件分支的 ELSE 部分）

**功能**：处理 SI_ELSE，恢复之前保存的 EXEC 掩码并反转执行路径。

**关键操作**：
<a name="ref-block_17"></a>- 使用 S_OR_SAVEEXEC 恢复保存的 EXEC 掩码 llvm-project:326-330[<sup>↗</sup>](#block_17) 
<a name="ref-block_18"></a>- 执行 AND 操作获取当前激活的线程 llvm-project:338-340[<sup>↗</sup>](#block_18) 
<a name="ref-block_19"></a>- 使用 XOR 反转 EXEC 掩码以执行 else 分支 llvm-project:342-345[<sup>↗</sup>](#block_19) 
<a name="ref-block_20"></a>- 插入条件分支 S_CBRANCH_EXECZ llvm-project:351-353[<sup>↗</sup>](#block_20) 

### 3.3 emitIfBreak（处理循环中的 break 条件）

**功能**：处理 SI_IF_BREAK 指令，用于循环中的条件退出。

**关键操作**：
<a name="ref-block_21"></a>- 检查是否可以跳过与 EXEC 的 AND 操作（如果条件已经被 EXEC 掩码） llvm-project:382-392[<sup>↗</sup>](#block_21) 
<a name="ref-block_22"></a>- 将 break 条件与 EXEC 进行 AND 操作 llvm-project:399-403[<sup>↗</sup>](#block_22) 
<a name="ref-block_23"></a>- 将结果 OR 到循环退出掩码中 llvm-project:405-407[<sup>↗</sup>](#block_23) 

### 3.4 emitLoop（处理循环回跳）

**功能**：处理 SI_LOOP 指令，生成循环回跳逻辑。

**关键操作**：
<a name="ref-block_24"></a>- 使用 S_ANDN2 从 EXEC 中清除退出的线程 llvm-project:435-438[<sup>↗</sup>](#block_24) 
<a name="ref-block_25"></a>- 插入条件分支 S_CBRANCH_EXECNZ 回到循环头 llvm-project:443-445[<sup>↗</sup>](#block_25) 

### 3.5 emitEndCf（结束控制流区域）

**功能**：处理 SI_END_CF 指令，恢复之前保存的 EXEC 掩码。

**关键操作**：
<a name="ref-block_26"></a>- 检查是否需要分割基本块（如果有指令修改了数据寄存器） llvm-project:495-503[<sup>↗</sup>](#block_26) 
<a name="ref-block_27"></a>- 如果需要分割，则分割基本块并更新支配树 llvm-project:507-519[<sup>↗</sup>](#block_27) 
<a name="ref-block_28"></a>- 使用 S_OR 恢复保存的 EXEC 位 llvm-project:521-524[<sup>↗</sup>](#block_28) 

### 3.6 combineMasks（优化掩码操作）

**功能**：搜索并组合等价的指令对，优化 EXEC 掩码上的位操作。

**关键操作**：
<a name="ref-block_29"></a>- 查找可以组合的 S_AND_B64 或 S_OR_B64 操作 llvm-project:601-604[<sup>↗</sup>](#block_29) 
<a name="ref-block_30"></a>- 递归查找操作数，检查是否可以简化嵌套的相同操作 llvm-project:607-612[<sup>↗</sup>](#block_30) 
<a name="ref-block_31"></a>- 识别重复操作数并消除冗余计算 llvm-project:614-624[<sup>↗</sup>](#block_31) 

### 3.7 optimizeEndCf（优化 END_CF 指令）

**功能**：移除冗余的 SI_END_CF 指令以优化代码。

**关键操作**：
- 检查 END_CF 之后是否紧跟另一个 END_CF llvm-project:633-638 
- 验证外层 END_CF 属于 SI_IF（而非 SI_ELSE） llvm-project:639-646 
- 如果满足条件，移除内层冗余的 END_CF 指令 llvm-project:647-657 

### 3.8 process（主处理分发函数）

**功能**：根据指令类型分发到相应的处理函数。

**关键操作**：
- 使用 switch 语句匹配不同的伪控制流指令 llvm-project:668-696 
- 调用相应的 emit 函数进行转换 llvm-project:669-691 
- 处理完成后执行 combineMasks 进行后续优化 llvm-project:698-714 

### 3.9 run（Pass 主入口）

**功能**：Pass 的主执行函数，遍历整个函数并处理所有控制流伪指令。

**关键操作**：
- 初始化目标相关的操作码（根据 wave32 或 wave64） llvm-project:775-795 
- 计算包含 kill 指令的基本块集合 llvm-project:797-817 
- 遍历所有基本块和指令，查找并处理控制流伪指令 llvm-project:820-850 
- 调用 optimizeEndCf 进行最终优化 llvm-project:852-852 
- 重新计算活跃区间（LiveIntervals） llvm-project:854-859 

### 3.10 hasKill（检查 kill 指令）

**功能**：检查从起始块到结束块的路径中是否存在 kill 指令。

**关键操作**：
<a name="ref-block_9"></a>- 使用工作列表算法遍历所有可达的基本块 llvm-project:185-202[<sup>↗</sup>](#block_9) 
- 检查每个块是否在 KillBlocks 集合中 llvm-project:195-196 

## 4. 步骤/子功能之间的关系

### 层次关系：

<a name="ref-block_34"></a>1. **顶层入口**：`run` 函数是整个 pass 的入口点 llvm-project:765-867[<sup>↗</sup>](#block_34) 

<a name="ref-block_33"></a>2. **中间层分发**：`process` 函数作为分发器，根据指令类型调用具体的处理函数 llvm-project:661-717[<sup>↗</sup>](#block_33) 

<a name="ref-block_4"></a>3. **底层处理**：`emitIf`、`emitElse`、`emitIfBreak`、`emitLoop`、`emitEndCf` 等函数执行具体的指令转换 llvm-project:100-105[<sup>↗</sup>](#block_4) 

### 执行流程关系：

**初始化阶段** → **遍历处理阶段** → **优化阶段** → **清理阶段**

- `run` 首先设置操作码和收集 kill 块
- 然后遍历所有指令，调用 `process` 处理每个控制流伪指令
- `process` 调用相应的 emit 函数，并执行 `combineMasks` 进行局部优化
- 最后调用 `optimizeEndCf` 进行全局优化

### 数据依赖关系：

<a name="ref-block_10"></a>- **emitIf** 生成的保存寄存器被 **emitElse** 和 **emitEndCf** 使用 llvm-project:220-220[<sup>↗</sup>](#block_10) 
<a name="ref-block_12"></a>- **hasKill** 辅助 **emitIf** 判断是否可以进行简单 IF 优化 llvm-project:233-236[<sup>↗</sup>](#block_12) 
- **combineMasks** 在每次 `process` 调用后执行，优化生成的指令 llvm-project:698-714 
<a name="ref-block_32"></a>- **optimizeEndCf** 依赖于 `LoweredEndCf` 和 `LoweredIf` 集合来识别可优化的模式 llvm-project:633-658[<sup>↗</sup>](#block_32) 

### 协作关系：

所有 emit 函数共同工作以支持完整的控制流转换，它们遵循相同的模式：保存/恢复 EXEC 寄存器，更新活跃变量信息（LiveVariables）和活跃区间（LiveIntervals），并正确处理支配树（MachineDominatorTree）的更新。

## Notes

- 该 pass 是 AMDGPU 后端代码生成的关键步骤，它将高级控制流表示转换为 GPU 硬件能够直接执行的基于谓词的执行模型
- 代码支持 wave32 和 wave64 两种模式，自动选择相应的指令操作码
- Pass 保持了多个可选分析的有效性，包括 LiveIntervals、LiveVariables 和 MachineDominatorTree
- 优化功能（如 `optimizeEndCf` 和 `combineMasks`）可以显著减少生成代码的冗余，提高执行效率
### Citations
<a name="block_0"></a>**File:** llvm/lib/Target/AMDGPU/SILowerControlFlow.cpp (L10-11) [<sup>↩</sup>](#ref-block_0)
```cpp
/// This pass lowers the pseudo control flow instructions to real
/// machine instructions.
```
<a name="block_1"></a>**File:** llvm/lib/Target/AMDGPU/SILowerControlFlow.cpp (L13-17)
```cpp
/// All control flow is handled using predicated instructions and
/// a predicate stack.  Each Scalar ALU controls the operations of 64 Vector
/// ALUs.  The Scalar ALU can update the predicate for any of the Vector ALUs
/// by writing to the 64-bit EXEC register (each bit corresponds to a
/// single vector ALU).  Typically, for predicates, a vector ALU will write
```
<a name="block_2"></a>**File:** llvm/lib/Target/AMDGPU/SILowerControlFlow.cpp (L22-48) [<sup>↩</sup>](#ref-block_2)
```cpp
/// For example:
/// %vcc = V_CMP_GT_F32 %vgpr1, %vgpr2
/// %sgpr0 = SI_IF %vcc
///   %vgpr0 = V_ADD_F32 %vgpr0, %vgpr0
/// %sgpr0 = SI_ELSE %sgpr0
///   %vgpr0 = V_SUB_F32 %vgpr0, %vgpr0
/// SI_END_CF %sgpr0
///
/// becomes:
///
/// %sgpr0 = S_AND_SAVEEXEC_B64 %vcc  // Save and update the exec mask
/// %sgpr0 = S_XOR_B64 %sgpr0, %exec  // Clear live bits from saved exec mask
/// S_CBRANCH_EXECZ label0            // This instruction is an optional
///                                   // optimization which allows us to
///                                   // branch if all the bits of
///                                   // EXEC are zero.
/// %vgpr0 = V_ADD_F32 %vgpr0, %vgpr0 // Do the IF block of the branch
///
/// label0:
/// %sgpr0 = S_OR_SAVEEXEC_B64 %sgpr0  // Restore the exec mask for the Then
///                                    // block
/// %exec = S_XOR_B64 %sgpr0, %exec    // Update the exec mask
/// S_CBRANCH_EXECZ label1             // Use our branch optimization
///                                    // instruction again.
/// %vgpr0 = V_SUB_F32 %vgpr0, %vgpr   // Do the ELSE block
/// label1:
/// %exec = S_OR_B64 %exec, %sgpr0     // Re-enable saved exec mask bits
```
<a name="block_3"></a>**File:** llvm/lib/Target/AMDGPU/SILowerControlFlow.cpp (L98-98) [<sup>↩</sup>](#ref-block_3)
```cpp
  bool hasKill(const MachineBasicBlock *Begin, const MachineBasicBlock *End);
```
<a name="block_4"></a>**File:** llvm/lib/Target/AMDGPU/SILowerControlFlow.cpp (L100-105) [<sup>↩</sup>](#ref-block_4)
```cpp
  void emitIf(MachineInstr &MI);
  void emitElse(MachineInstr &MI);
  void emitIfBreak(MachineInstr &MI);
  void emitLoop(MachineInstr &MI);

  MachineBasicBlock *emitEndCf(MachineInstr &MI);
```
<a name="block_5"></a>**File:** llvm/lib/Target/AMDGPU/SILowerControlFlow.cpp (L110-110) [<sup>↩</sup>](#ref-block_5)
```cpp
  void combineMasks(MachineInstr &MI);
```
<a name="block_6"></a>**File:** llvm/lib/Target/AMDGPU/SILowerControlFlow.cpp (L114-114) [<sup>↩</sup>](#ref-block_6)
```cpp
  MachineBasicBlock *process(MachineInstr &MI);
```
<a name="block_7"></a>**File:** llvm/lib/Target/AMDGPU/SILowerControlFlow.cpp (L137-137) [<sup>↩</sup>](#ref-block_7)
```cpp
  void optimizeEndCf();
```
<a name="block_8"></a>**File:** llvm/lib/Target/AMDGPU/SILowerControlFlow.cpp (L143-143) [<sup>↩</sup>](#ref-block_8)
```cpp
  bool run(MachineFunction &MF);
```
<a name="block_9"></a>**File:** llvm/lib/Target/AMDGPU/SILowerControlFlow.cpp (L185-202) [<sup>↩</sup>](#ref-block_9)
```cpp
bool SILowerControlFlow::hasKill(const MachineBasicBlock *Begin,
                                 const MachineBasicBlock *End) {
  DenseSet<const MachineBasicBlock*> Visited;
  SmallVector<MachineBasicBlock *, 4> Worklist(Begin->successors());

  while (!Worklist.empty()) {
    MachineBasicBlock *MBB = Worklist.pop_back_val();

    if (MBB == End || !Visited.insert(MBB).second)
      continue;
    if (KillBlocks.contains(MBB))
      return true;

    Worklist.append(MBB->succ_begin(), MBB->succ_end());
  }

  return false;
}
```
<a name="block_10"></a>**File:** llvm/lib/Target/AMDGPU/SILowerControlFlow.cpp (L220-220) [<sup>↩</sup>](#ref-block_10)
```cpp
  Register SaveExecReg = MI.getOperand(0).getReg();
```
<a name="block_11"></a>**File:** llvm/lib/Target/AMDGPU/SILowerControlFlow.cpp (L227-230) [<sup>↩</sup>](#ref-block_11)
```cpp
  // If there is only one use of save exec register and that use is SI_END_CF,
  // we can optimize SI_IF by returning the full saved exec mask instead of
  // just cleared bits.
  bool SimpleIf = isSimpleIf(MI, MRI);
```
<a name="block_12"></a>**File:** llvm/lib/Target/AMDGPU/SILowerControlFlow.cpp (L233-236) [<sup>↩</sup>](#ref-block_12)
```cpp
    // Check for SI_KILL_*_TERMINATOR on path from if to endif.
    // if there is any such terminator simplifications are not safe.
    auto UseMI = MRI->use_instr_nodbg_begin(SaveExecReg);
    SimpleIf = !hasKill(MI.getParent(), UseMI->getParent());
```
<a name="block_13"></a>**File:** llvm/lib/Target/AMDGPU/SILowerControlFlow.cpp (L243-246) [<sup>↩</sup>](#ref-block_13)
```cpp
  MachineInstr *CopyExec =
    BuildMI(MBB, I, DL, TII->get(AMDGPU::COPY), CopyReg)
    .addReg(Exec)
    .addReg(Exec, RegState::ImplicitDefine);
```
<a name="block_14"></a>**File:** llvm/lib/Target/AMDGPU/SILowerControlFlow.cpp (L251-254) [<sup>↩</sup>](#ref-block_14)
```cpp
  MachineInstr *And =
    BuildMI(MBB, I, DL, TII->get(AndOpc), Tmp)
    .addReg(CopyReg)
    .add(Cond);
```
<a name="block_15"></a>**File:** llvm/lib/Target/AMDGPU/SILowerControlFlow.cpp (L260-267) [<sup>↩</sup>](#ref-block_15)
```cpp
  MachineInstr *Xor = nullptr;
  if (!SimpleIf) {
    Xor =
      BuildMI(MBB, I, DL, TII->get(XorOpc), SaveExecReg)
      .addReg(Tmp)
      .addReg(CopyReg);
    setImpSCCDefDead(*Xor, ImpDefSCC.isDead());
  }
```
<a name="block_16"></a>**File:** llvm/lib/Target/AMDGPU/SILowerControlFlow.cpp (L283-284) [<sup>↩</sup>](#ref-block_16)
```cpp
  MachineInstr *NewBr = BuildMI(MBB, I, DL, TII->get(AMDGPU::S_CBRANCH_EXECZ))
                            .add(MI.getOperand(2));
```
<a name="block_17"></a>**File:** llvm/lib/Target/AMDGPU/SILowerControlFlow.cpp (L326-330) [<sup>↩</sup>](#ref-block_17)
```cpp
  MachineInstr *OrSaveExec =
    BuildMI(MBB, Start, DL, TII->get(OrSaveExecOpc), SaveReg)
    .add(MI.getOperand(1)); // Saved EXEC
  if (LV)
    LV->replaceKillInstruction(SrcReg, MI, *OrSaveExec);
```
<a name="block_18"></a>**File:** llvm/lib/Target/AMDGPU/SILowerControlFlow.cpp (L338-340) [<sup>↩</sup>](#ref-block_18)
```cpp
  MachineInstr *And = BuildMI(MBB, ElsePt, DL, TII->get(AndOpc), DstReg)
                          .addReg(Exec)
                          .addReg(SaveReg);
```
<a name="block_19"></a>**File:** llvm/lib/Target/AMDGPU/SILowerControlFlow.cpp (L342-345) [<sup>↩</sup>](#ref-block_19)
```cpp
  MachineInstr *Xor =
    BuildMI(MBB, ElsePt, DL, TII->get(XorTermrOpc), Exec)
    .addReg(Exec)
    .addReg(DstReg);
```
<a name="block_20"></a>**File:** llvm/lib/Target/AMDGPU/SILowerControlFlow.cpp (L351-353) [<sup>↩</sup>](#ref-block_20)
```cpp
  MachineInstr *Branch =
      BuildMI(MBB, ElsePt, DL, TII->get(AMDGPU::S_CBRANCH_EXECZ))
          .addMBB(DestBB);
```
<a name="block_21"></a>**File:** llvm/lib/Target/AMDGPU/SILowerControlFlow.cpp (L382-392) [<sup>↩</sup>](#ref-block_21)
```cpp
  // Skip ANDing with exec if the break condition is already masked by exec
  // because it is a V_CMP in the same basic block. (We know the break
  // condition operand was an i1 in IR, so if it is a VALU instruction it must
  // be one with a carry-out.)
  bool SkipAnding = false;
  if (MI.getOperand(1).isReg()) {
    if (MachineInstr *Def = MRI->getUniqueVRegDef(MI.getOperand(1).getReg())) {
      SkipAnding = Def->getParent() == MI.getParent()
          && SIInstrInfo::isVALU(*Def);
    }
  }
```
<a name="block_22"></a>**File:** llvm/lib/Target/AMDGPU/SILowerControlFlow.cpp (L399-403) [<sup>↩</sup>](#ref-block_22)
```cpp
    AndReg = MRI->createVirtualRegister(BoolRC);
    And = BuildMI(MBB, &MI, DL, TII->get(AndOpc), AndReg)
             .addReg(Exec)
             .add(MI.getOperand(1));
    if (LV)
```
<a name="block_23"></a>**File:** llvm/lib/Target/AMDGPU/SILowerControlFlow.cpp (L405-407) [<sup>↩</sup>](#ref-block_23)
```cpp
    Or = BuildMI(MBB, &MI, DL, TII->get(OrOpc), Dst)
             .addReg(AndReg)
             .add(MI.getOperand(2));
```
<a name="block_24"></a>**File:** llvm/lib/Target/AMDGPU/SILowerControlFlow.cpp (L435-438) [<sup>↩</sup>](#ref-block_24)
```cpp
  MachineInstr *AndN2 =
      BuildMI(MBB, &MI, DL, TII->get(Andn2TermOpc), Exec)
          .addReg(Exec)
          .add(MI.getOperand(0));
```
<a name="block_25"></a>**File:** llvm/lib/Target/AMDGPU/SILowerControlFlow.cpp (L443-445) [<sup>↩</sup>](#ref-block_25)
```cpp
  MachineInstr *Branch =
      BuildMI(MBB, BranchPt, DL, TII->get(AMDGPU::S_CBRANCH_EXECNZ))
          .add(MI.getOperand(1));
```
<a name="block_26"></a>**File:** llvm/lib/Target/AMDGPU/SILowerControlFlow.cpp (L495-503) [<sup>↩</sup>](#ref-block_26)
```cpp
  bool NeedBlockSplit = false;
  Register DataReg = MI.getOperand(0).getReg();
  for (MachineBasicBlock::iterator I = InsPt, E = MI.getIterator();
       I != E; ++I) {
    if (I->modifiesRegister(DataReg, TRI)) {
      NeedBlockSplit = true;
      break;
    }
  }
```
<a name="block_27"></a>**File:** llvm/lib/Target/AMDGPU/SILowerControlFlow.cpp (L507-519) [<sup>↩</sup>](#ref-block_27)
```cpp
  if (NeedBlockSplit) {
    SplitBB = MBB.splitAt(MI, /*UpdateLiveIns*/true, LIS);
    if (MDT && SplitBB != &MBB) {
      MachineDomTreeNode *MBBNode = (*MDT)[&MBB];
      SmallVector<MachineDomTreeNode *> Children(MBBNode->begin(),
                                                 MBBNode->end());
      MachineDomTreeNode *SplitBBNode = MDT->addNewBlock(SplitBB, &MBB);
      for (MachineDomTreeNode *Child : Children)
        MDT->changeImmediateDominator(Child, SplitBBNode);
    }
    Opcode = OrTermrOpc;
    InsPt = MI;
  }
```
<a name="block_28"></a>**File:** llvm/lib/Target/AMDGPU/SILowerControlFlow.cpp (L521-524) [<sup>↩</sup>](#ref-block_28)
```cpp
  MachineInstr *NewMI =
    BuildMI(MBB, InsPt, DL, TII->get(Opcode), Exec)
    .addReg(Exec)
    .add(MI.getOperand(0));
```
<a name="block_29"></a>**File:** llvm/lib/Target/AMDGPU/SILowerControlFlow.cpp (L601-604) [<sup>↩</sup>](#ref-block_29)
```cpp
// Search and combine pairs of equivalent instructions, like
// S_AND_B64 x, (S_AND_B64 x, y) => S_AND_B64 x, y
// S_OR_B64  x, (S_OR_B64  x, y) => S_OR_B64  x, y
// One of the operands is exec mask.
```
<a name="block_30"></a>**File:** llvm/lib/Target/AMDGPU/SILowerControlFlow.cpp (L607-612) [<sup>↩</sup>](#ref-block_30)
```cpp
  SmallVector<MachineOperand, 4> Ops;
  unsigned OpToReplace = 1;
  findMaskOperands(MI, 1, Ops);
  if (Ops.size() == 1) OpToReplace = 2; // First operand can be exec or its copy
  findMaskOperands(MI, 2, Ops);
  if (Ops.size() != 3) return;
```
<a name="block_31"></a>**File:** llvm/lib/Target/AMDGPU/SILowerControlFlow.cpp (L614-624) [<sup>↩</sup>](#ref-block_31)
```cpp
  unsigned UniqueOpndIdx;
  if (Ops[0].isIdenticalTo(Ops[1])) UniqueOpndIdx = 2;
  else if (Ops[0].isIdenticalTo(Ops[2])) UniqueOpndIdx = 1;
  else if (Ops[1].isIdenticalTo(Ops[2])) UniqueOpndIdx = 1;
  else return;

  Register Reg = MI.getOperand(OpToReplace).getReg();
  MI.removeOperand(OpToReplace);
  MI.addOperand(Ops[UniqueOpndIdx]);
  if (MRI->use_empty(Reg))
    MRI->getUniqueVRegDef(Reg)->eraseFromParent();
```
<a name="block_32"></a>**File:** llvm/lib/Target/AMDGPU/SILowerControlFlow.cpp (L633-658) [<sup>↩</sup>](#ref-block_32)
```cpp
  for (MachineInstr *MI : reverse(LoweredEndCf)) {
    MachineBasicBlock &MBB = *MI->getParent();
    auto Next =
      skipIgnoreExecInstsTrivialSucc(MBB, std::next(MI->getIterator()));
    if (Next == MBB.end() || !LoweredEndCf.count(&*Next))
      continue;
    // Only skip inner END_CF if outer ENDCF belongs to SI_IF.
    // If that belongs to SI_ELSE then saved mask has an inverted value.
    Register SavedExec
      = TII->getNamedOperand(*Next, AMDGPU::OpName::src1)->getReg();
    assert(SavedExec.isVirtual() && "Expected saved exec to be src1!");

    const MachineInstr *Def = MRI->getUniqueVRegDef(SavedExec);
    if (Def && LoweredIf.count(SavedExec)) {
      LLVM_DEBUG(dbgs() << "Skip redundant "; MI->dump());
      if (LIS)
        LIS->RemoveMachineInstrFromMaps(*MI);
      Register Reg;
      if (LV)
        Reg = TII->getNamedOperand(*MI, AMDGPU::OpName::src1)->getReg();
      MI->eraseFromParent();
      if (LV)
        LV->recomputeForSingleDefVirtReg(Reg);
      removeMBBifRedundant(MBB);
    }
  }
```
<a name="block_33"></a>**File:** llvm/lib/Target/AMDGPU/SILowerControlFlow.cpp (L661-717) [<sup>↩</sup>](#ref-block_33)
```cpp
MachineBasicBlock *SILowerControlFlow::process(MachineInstr &MI) {
  MachineBasicBlock &MBB = *MI.getParent();
  MachineBasicBlock::iterator I(MI);
  MachineInstr *Prev = (I != MBB.begin()) ? &*(std::prev(I)) : nullptr;

  MachineBasicBlock *SplitBB = &MBB;

  switch (MI.getOpcode()) {
  case AMDGPU::SI_IF:
    emitIf(MI);
    break;

  case AMDGPU::SI_ELSE:
    emitElse(MI);
    break;

  case AMDGPU::SI_IF_BREAK:
    emitIfBreak(MI);
    break;

  case AMDGPU::SI_LOOP:
    emitLoop(MI);
    break;

  case AMDGPU::SI_WATERFALL_LOOP:
    MI.setDesc(TII->get(AMDGPU::S_CBRANCH_EXECNZ));
    break;

  case AMDGPU::SI_END_CF:
    SplitBB = emitEndCf(MI);
    break;

  default:
    assert(false && "Attempt to process unsupported instruction");
    break;
  }

  MachineBasicBlock::iterator Next;
  for (I = Prev ? Prev->getIterator() : MBB.begin(); I != MBB.end(); I = Next) {
    Next = std::next(I);
    MachineInstr &MaskMI = *I;
    switch (MaskMI.getOpcode()) {
    case AMDGPU::S_AND_B64:
    case AMDGPU::S_OR_B64:
    case AMDGPU::S_AND_B32:
    case AMDGPU::S_OR_B32:
      // Cleanup bit manipulations on exec mask
      combineMasks(MaskMI);
      break;
    default:
      I = MBB.end();
      break;
    }
  }

  return SplitBB;
}
```
<a name="block_34"></a>**File:** llvm/lib/Target/AMDGPU/SILowerControlFlow.cpp (L765-867) [<sup>↩</sup>](#ref-block_34)
```cpp
bool SILowerControlFlow::run(MachineFunction &MF) {
  const GCNSubtarget &ST = MF.getSubtarget<GCNSubtarget>();
  TII = ST.getInstrInfo();
  TRI = &TII->getRegisterInfo();
  EnableOptimizeEndCf = RemoveRedundantEndcf &&
                        MF.getTarget().getOptLevel() > CodeGenOptLevel::None;

  MRI = &MF.getRegInfo();
  BoolRC = TRI->getBoolRC();

  if (ST.isWave32()) {
    AndOpc = AMDGPU::S_AND_B32;
    OrOpc = AMDGPU::S_OR_B32;
    XorOpc = AMDGPU::S_XOR_B32;
    MovTermOpc = AMDGPU::S_MOV_B32_term;
    Andn2TermOpc = AMDGPU::S_ANDN2_B32_term;
    XorTermrOpc = AMDGPU::S_XOR_B32_term;
    OrTermrOpc = AMDGPU::S_OR_B32_term;
    OrSaveExecOpc = AMDGPU::S_OR_SAVEEXEC_B32;
    Exec = AMDGPU::EXEC_LO;
  } else {
    AndOpc = AMDGPU::S_AND_B64;
    OrOpc = AMDGPU::S_OR_B64;
    XorOpc = AMDGPU::S_XOR_B64;
    MovTermOpc = AMDGPU::S_MOV_B64_term;
    Andn2TermOpc = AMDGPU::S_ANDN2_B64_term;
    XorTermrOpc = AMDGPU::S_XOR_B64_term;
    OrTermrOpc = AMDGPU::S_OR_B64_term;
    OrSaveExecOpc = AMDGPU::S_OR_SAVEEXEC_B64;
    Exec = AMDGPU::EXEC;
  }

  // Compute set of blocks with kills
  const bool CanDemote =
      MF.getFunction().getCallingConv() == CallingConv::AMDGPU_PS;
  for (auto &MBB : MF) {
    bool IsKillBlock = false;
    for (auto &Term : MBB.terminators()) {
      if (TII->isKillTerminator(Term.getOpcode())) {
        KillBlocks.insert(&MBB);
        IsKillBlock = true;
        break;
      }
    }
    if (CanDemote && !IsKillBlock) {
      for (auto &MI : MBB) {
        if (MI.getOpcode() == AMDGPU::SI_DEMOTE_I1) {
          KillBlocks.insert(&MBB);
          break;
        }
      }
    }
  }

  bool Changed = false;
  MachineFunction::iterator NextBB;
  for (MachineFunction::iterator BI = MF.begin();
       BI != MF.end(); BI = NextBB) {
    NextBB = std::next(BI);
    MachineBasicBlock *MBB = &*BI;

    MachineBasicBlock::iterator I, E, Next;
    E = MBB->end();
    for (I = MBB->begin(); I != E; I = Next) {
      Next = std::next(I);
      MachineInstr &MI = *I;
      MachineBasicBlock *SplitMBB = MBB;

      switch (MI.getOpcode()) {
      case AMDGPU::SI_IF:
      case AMDGPU::SI_ELSE:
      case AMDGPU::SI_IF_BREAK:
      case AMDGPU::SI_WATERFALL_LOOP:
      case AMDGPU::SI_LOOP:
      case AMDGPU::SI_END_CF:
        SplitMBB = process(MI);
        Changed = true;
        break;
      }

      if (SplitMBB != MBB) {
        MBB = Next->getParent();
        E = MBB->end();
      }
    }
  }

  optimizeEndCf();

  if (LIS) {
    for (Register Reg : RecomputeRegs) {
      LIS->removeInterval(Reg);
      LIS->createAndComputeVirtRegInterval(Reg);
    }
  }

  RecomputeRegs.clear();
  LoweredEndCf.clear();
  LoweredIf.clear();
  KillBlocks.clear();

  return Changed;
}
```