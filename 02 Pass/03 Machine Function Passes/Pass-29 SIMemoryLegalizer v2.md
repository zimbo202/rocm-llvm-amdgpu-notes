# SIMemoryLegalizer.cpp 代码功能详解

## 1. 主要功能概括

<a name="ref-block_0"></a>SIMemoryLegalizer 是 AMDGPU 后端的一个机器函数级 Pass，其主要功能是**实现内存模型合法化（Memory Model Legalization）**。 llvm-project:10-12[<sup>↗</sup>](#block_0) 

该 Pass 的作用和效果包括：
- **插入必要的内存同步指令**（如 waitcnt、barrier 等）来确保内存操作按照指定的原子顺序（atomic ordering）执行
- **设置缓存控制位**（如 GLC、SLC、DLC、SC0/SC1 等）来控制各级缓存的行为
- **插入缓存刷新和失效指令**（如 BUFFER_WBINVL1、BUFFER_INV、GLOBAL_INV 等）确保内存一致性
- **支持多种同步作用域**（system、agent、workgroup、wavefront、singlethread）和地址空间（global、LDS、scratch、GDS）

## 2. 主要功能的步骤与子功能

### 2.1 内存操作信息提取（SIMemOpAccess 类）

<a name="ref-block_2"></a>SIMemOpAccess 负责从机器指令中提取内存操作的语义信息： llvm-project:220-266[<sup>↗</sup>](#block_2) 

主要方法包括：
- `getLoadInfo()` - 提取 load 指令信息
- `getStoreInfo()` - 提取 store 指令信息
- `getAtomicFenceInfo()` - 提取原子 fence 指令信息
- `getAtomicCmpxchgOrRmwInfo()` - 提取原子 compare-exchange 或 read-modify-write 指令信息

<a name="ref-block_1"></a>这些方法会构造 `SIMemOpInfo` 对象，包含： llvm-project:92-118[<sup>↗</sup>](#block_1) 
- 原子顺序（Ordering）
- 同步作用域（Scope）
- 地址空间（AddrSpace）
- volatile、nontemporal、last-use 等属性

### 2.2 缓存控制（SICacheControl 类层次结构）

<a name="ref-block_3"></a>SICacheControl 是一个抽象基类，针对不同 GPU 架构（GFX6/7/90A/940/10/11/12）提供特化实现： llvm-project:268-364[<sup>↗</sup>](#block_3) 

核心功能方法包括：
- `enableLoadCacheBypass()` - 为 load 指令设置缓存绕过策略
- `enableStoreCacheBypass()` - 为 store 指令设置缓存绕过策略
- `enableRMWCacheBypass()` - 为原子 RMW 指令设置缓存绕过策略
- `enableVolatileAndOrNonTemporal()` - 处理 volatile 和 nontemporal 属性
- `insertWait()` - 插入等待指令（waitcnt）
- `insertAcquire()` - 插入 acquire 语义所需的缓存失效指令
- `insertRelease()` - 插入 release 语义所需的缓存刷新指令

### 2.3 指令展开（SIMemoryLegalizer 类）

<a name="ref-block_4"></a>SIMemoryLegalizer 是主要的 Pass 实现类，包含四个核心展开方法： llvm-project:630-669[<sup>↗</sup>](#block_4) 

#### 2.3.1 Load 指令展开

<a name="ref-block_16"></a>`expandLoad()` 处理原子和非原子 load 操作： llvm-project:2602-2644[<sup>↗</sup>](#block_16) 

对于原子 load：
- Monotonic/Acquire/SeqCst ordering 需要启用缓存绕过
- SeqCst ordering 需要在 load 前插入 wait 指令
- Acquire/SeqCst ordering 需要在 load 后插入 wait 和 cache invalidate 指令

#### 2.3.2 Store 指令展开

<a name="ref-block_17"></a>`expandStore()` 处理原子和非原子 store 操作： llvm-project:2646-2681[<sup>↗</sup>](#block_17) 

对于原子 store：
- Monotonic/Release/SeqCst ordering 需要启用缓存绕过
- Release/SeqCst ordering 需要在 store 前插入 release 操作（cache flush + wait）

#### 2.3.3 Atomic Fence 展开

<a name="ref-block_18"></a>`expandAtomicFence()` 处理内存屏障指令： llvm-project:2683-2733[<sup>↗</sup>](#block_18) 

根据 fence 的 ordering：
- Acquire：插入 wait 指令 + cache invalidate
- Release/AcquireRelease/SeqCst：插入 release 操作
- 支持通过 MMRA（Memory Model Relaxation Annotations）细化 fence 作用的地址空间

#### 2.3.4 Atomic RMW/Cmpxchg 展开

<a name="ref-block_19"></a>`expandAtomicCmpxchgOrRmw()` 处理原子读-改-写操作： llvm-project:2735-2778[<sup>↗</sup>](#block_19) 

根据 ordering 和 failure ordering：
- 启用 RMW 缓存绕过
- 根据需要插入 release 操作（在指令前）
- 根据需要插入 acquire 操作（在指令后）

### 2.4 主执行流程

<a name="ref-block_20"></a>`run()` 方法是 Pass 的主入口： llvm-project:2798-2838[<sup>↗</sup>](#block_20) 

执行流程：
1. 创建 SIMemOpAccess 和 SICacheControl 对象
2. 遍历函数中的所有基本块和指令
3. 解绑 bundle 指令（post-RA scheduler 后）
4. 对标记为 `maybeAtomic` 的指令，调用相应的展开方法
5. 清理原子伪指令

## 3. 各子功能的具体描述分析

### 3.1 同步作用域转换

<a name="ref-block_5"></a>`toSIAtomicScope()` 方法将 LLVM 的 SyncScope::ID 转换为 AMDGPU 特定的作用域： llvm-project:741-773[<sup>↗</sup>](#block_5) 

支持的作用域从细到粗：
- SINGLETHREAD：单线程
- WAVEFRONT：波前（warp/wavefront）
- WORKGROUP：工作组
- AGENT：代理（GPU 设备）
- SYSTEM：系统级别

### 3.2 地址空间处理

<a name="ref-block_6"></a>`toSIAtomicAddrSpace()` 方法将地址空间 ID 映射到 AMDGPU 的原子地址空间： llvm-project:775-788[<sup>↗</sup>](#block_6) 

支持的地址空间：
- GLOBAL：全局内存
- LDS：本地数据共享（Local Data Share）
- SCRATCH：私有内存
- GDS：全局数据共享（Global Data Share）
- FLAT：可访问 GLOBAL、LDS、SCRATCH 的平面地址空间

### 3.3 架构特定的缓存控制

不同 GPU 架构的缓存层次结构差异：

<a name="ref-block_7"></a>**GFX6/7**： llvm-project:964-997[<sup>↗</sup>](#block_7) 
- 使用 GLC 和 SLC 位控制 L1 缓存策略
- 使用 BUFFER_WBINVL1 失效 L1 缓存

<a name="ref-block_9"></a>**GFX90A**： llvm-project:1274-1314[<sup>↗</sup>](#block_9) 
- 支持 threadgroup split 模式
- 使用 BUFFER_INVL2 进行 system scope 失效
- 使用 BUFFER_WBL2 进行 system scope 刷新

<a name="ref-block_10"></a>**GFX940**： llvm-project:1568-1610[<sup>↗</sup>](#block_10) 
- 使用 SC0/SC1 位表示缓存一致性作用域
- 使用 NT 位表示 nontemporal

<a name="ref-block_11"></a>**GFX10**： llvm-project:1870-1911[<sup>↗</sup>](#block_11) 
- 引入 DLC 位和 L0 缓存
- 支持 WGP/CU 模式
- 使用 VSCNT 计数器跟踪 store 操作

<a name="ref-block_13"></a>**GFX11**： llvm-project:2142-2181[<sup>↗</sup>](#block_13) 
- 简化了 GLC 位的使用（不再需要 DLC for loads）
- 为 volatile 操作添加 MALL NOALLOC 支持

<a name="ref-block_15"></a>**GFX12**： llvm-project:2553-2589[<sup>↗</sup>](#block_15) 
- 使用 TH（Temporal Hint）策略字段
- 使用 SCOPE 字段指定一致性域
- 引入 GLOBAL_INV 和 GLOBAL_WB 指令
- 对 system scope store 需要插入额外的 wait 指令

### 3.4 Wait 指令插入策略

Wait 指令用于确保内存操作完成。不同架构使用不同的计数器：

<a name="ref-block_8"></a>**GFX6-9**：使用统一的 S_WAITCNT 指令： llvm-project:1072-1167[<sup>↗</sup>](#block_8) 
- VMCNT：等待 VMEM（global/scratch）操作
- LGKMCNT：等待 LDS/GDS/标量内存操作

<a name="ref-block_12"></a>**GFX10+**：分离的计数器： llvm-project:1964-2082[<sup>↗</sup>](#block_12) 
- VMCNT：等待 load 操作
- VSCNT：等待 store 操作
- LGKMCNT：等待 LDS/GDS 操作

<a name="ref-block_14"></a>**GFX12**：更细粒度的计数器： llvm-project:2285-2392[<sup>↗</sup>](#block_14) 
- LOADCNT、SAMPLECNT、BVHCNT：不同类型的 load
- STORECNT：store 操作
- DSCNT：LDS 操作
- KMCNT：标量内存操作

## 4. 步骤或子功能之间的关系

### 4.1 分层架构关系

```
SIMemoryLegalizer (Pass 主类)
    ↓ 使用
SIMemOpAccess (信息提取)
    ↓ 产生
SIMemOpInfo (操作信息)
    ↓ 传递给
SICacheControl (缓存控制抽象层)
    ↓ 多态实现
[GFX6/7/90A/940/10/11/12 CacheControl] (架构特定实现)
```

### 4.2 执行流程关系 llvm-project:2804-2834 

1. **信息收集阶段**：SIMemOpAccess 从机器指令和 MMO（Machine Memory Operand）中提取原子语义信息
2. **决策阶段**：根据 ordering、scope、address space 决定需要的同步操作
3. **代码生成阶段**：SICacheControl 插入具体的缓存控制指令和同步指令
4. **清理阶段**：删除伪指令（如 ATOMIC_FENCE）

### 4.3 Acquire-Release 语义实现

Acquire-Release 语义通过配对的操作实现：

**Release 操作**（在 store/fence 前）：
1. 插入 wait 指令确保之前的内存操作完成
2. 插入 cache writeback 指令刷新脏数据
3. 设置适当的缓存策略位

**Acquire 操作**（在 load/fence 后）：
1. 插入 wait 指令确保 load 完成
2. 插入 cache invalidate 指令失效旧数据
3. 确保后续操作看到最新数据

### 4.4 跨地址空间同步

`IsCrossAddressSpaceOrdering` 标志指示是否需要跨地址空间同步： llvm-project:1106-1127 

当涉及多个地址空间时，需要额外的 LGKMCNT wait 来确保 LDS/GDS 操作与 global 内存操作之间的顺序。

## Notes

- 该 Pass 实现了 AMDGPU 内存模型规范，详细信息可参考：http://llvm.org/docs/AMDGPUUsage.html#memory-model
- 不同 GPU 架构的缓存层次结构和一致性协议差异很大，因此需要架构特定的实现
- 该 Pass 在 post-RA（寄存器分配后）阶段运行，因此操作的是机器指令而非 LLVM IR
- 可通过 `-amdgcn-skip-cache-invalidations` 选项跳过缓存失效指令的插入（用于调试）
- GFX12 引入了显式的 SCOPE 和 TH 字段，使得缓存控制更加直观和细粒度
### Citations
<a name="block_0"></a>**File:** llvm/lib/Target/AMDGPU/SIMemoryLegalizer.cpp (L10-12) [<sup>↩</sup>](#ref-block_0)
```cpp
/// Memory legalizer - implements memory model. More information can be
/// found here:
///   http://llvm.org/docs/AMDGPUUsage.html#memory-model
```
<a name="block_1"></a>**File:** llvm/lib/Target/AMDGPU/SIMemoryLegalizer.cpp (L92-118) [<sup>↩</sup>](#ref-block_1)
```cpp
class SIMemOpInfo final {
private:

  friend class SIMemOpAccess;

  AtomicOrdering Ordering = AtomicOrdering::NotAtomic;
  AtomicOrdering FailureOrdering = AtomicOrdering::NotAtomic;
  SIAtomicScope Scope = SIAtomicScope::SYSTEM;
  SIAtomicAddrSpace OrderingAddrSpace = SIAtomicAddrSpace::NONE;
  SIAtomicAddrSpace InstrAddrSpace = SIAtomicAddrSpace::NONE;
  bool IsCrossAddressSpaceOrdering = false;
  bool IsVolatile = false;
  bool IsNonTemporal = false;
  bool IsLastUse = false;

  SIMemOpInfo(
      AtomicOrdering Ordering = AtomicOrdering::SequentiallyConsistent,
      SIAtomicScope Scope = SIAtomicScope::SYSTEM,
      SIAtomicAddrSpace OrderingAddrSpace = SIAtomicAddrSpace::ATOMIC,
      SIAtomicAddrSpace InstrAddrSpace = SIAtomicAddrSpace::ALL,
      bool IsCrossAddressSpaceOrdering = true,
      AtomicOrdering FailureOrdering = AtomicOrdering::SequentiallyConsistent,
      bool IsVolatile = false, bool IsNonTemporal = false,
      bool IsLastUse = false)
      : Ordering(Ordering), FailureOrdering(FailureOrdering), Scope(Scope),
        OrderingAddrSpace(OrderingAddrSpace), InstrAddrSpace(InstrAddrSpace),
        IsCrossAddressSpaceOrdering(IsCrossAddressSpaceOrdering),
```
<a name="block_2"></a>**File:** llvm/lib/Target/AMDGPU/SIMemoryLegalizer.cpp (L220-266) [<sup>↩</sup>](#ref-block_2)
```cpp
class SIMemOpAccess final {
private:
  const AMDGPUMachineModuleInfo *MMI = nullptr;

  /// Reports unsupported message \p Msg for \p MI to LLVM context.
  void reportUnsupported(const MachineBasicBlock::iterator &MI,
                         const char *Msg) const;

  /// Inspects the target synchronization scope \p SSID and determines
  /// the SI atomic scope it corresponds to, the address spaces it
  /// covers, and whether the memory ordering applies between address
  /// spaces.
  std::optional<std::tuple<SIAtomicScope, SIAtomicAddrSpace, bool>>
  toSIAtomicScope(SyncScope::ID SSID, SIAtomicAddrSpace InstrAddrSpace) const;

  /// \return Return a bit set of the address spaces accessed by \p AS.
  SIAtomicAddrSpace toSIAtomicAddrSpace(unsigned AS) const;

  /// \returns Info constructed from \p MI, which has at least machine memory
  /// operand.
  std::optional<SIMemOpInfo>
  constructFromMIWithMMO(const MachineBasicBlock::iterator &MI) const;

public:
  /// Construct class to support accessing the machine memory operands
  /// of instructions in the machine function \p MF.
  SIMemOpAccess(const AMDGPUMachineModuleInfo &MMI);

  /// \returns Load info if \p MI is a load operation, "std::nullopt" otherwise.
  std::optional<SIMemOpInfo>
  getLoadInfo(const MachineBasicBlock::iterator &MI) const;

  /// \returns Store info if \p MI is a store operation, "std::nullopt"
  /// otherwise.
  std::optional<SIMemOpInfo>
  getStoreInfo(const MachineBasicBlock::iterator &MI) const;

  /// \returns Atomic fence info if \p MI is an atomic fence operation,
  /// "std::nullopt" otherwise.
  std::optional<SIMemOpInfo>
  getAtomicFenceInfo(const MachineBasicBlock::iterator &MI) const;

  /// \returns Atomic cmpxchg/rmw info if \p MI is an atomic cmpxchg or
  /// rmw operation, "std::nullopt" otherwise.
  std::optional<SIMemOpInfo>
  getAtomicCmpxchgOrRmwInfo(const MachineBasicBlock::iterator &MI) const;
};
```
<a name="block_3"></a>**File:** llvm/lib/Target/AMDGPU/SIMemoryLegalizer.cpp (L268-364) [<sup>↩</sup>](#ref-block_3)
```cpp
class SICacheControl {
protected:

  /// AMDGPU subtarget info.
  const GCNSubtarget &ST;

  /// Instruction info.
  const SIInstrInfo *TII = nullptr;

  IsaVersion IV;

  /// Whether to insert cache invalidating instructions.
  bool InsertCacheInv;

  SICacheControl(const GCNSubtarget &ST);

  /// Sets named bit \p BitName to "true" if present in instruction \p MI.
  /// \returns Returns true if \p MI is modified, false otherwise.
  bool enableNamedBit(const MachineBasicBlock::iterator MI,
                      AMDGPU::CPol::CPol Bit) const;

public:

  /// Create a cache control for the subtarget \p ST.
  static std::unique_ptr<SICacheControl> create(const GCNSubtarget &ST);

  /// Update \p MI memory load instruction to bypass any caches up to
  /// the \p Scope memory scope for address spaces \p
  /// AddrSpace. Return true iff the instruction was modified.
  virtual bool enableLoadCacheBypass(const MachineBasicBlock::iterator &MI,
                                     SIAtomicScope Scope,
                                     SIAtomicAddrSpace AddrSpace) const = 0;

  /// Update \p MI memory store instruction to bypass any caches up to
  /// the \p Scope memory scope for address spaces \p
  /// AddrSpace. Return true iff the instruction was modified.
  virtual bool enableStoreCacheBypass(const MachineBasicBlock::iterator &MI,
                                      SIAtomicScope Scope,
                                      SIAtomicAddrSpace AddrSpace) const = 0;

  /// Update \p MI memory read-modify-write instruction to bypass any caches up
  /// to the \p Scope memory scope for address spaces \p AddrSpace. Return true
  /// iff the instruction was modified.
  virtual bool enableRMWCacheBypass(const MachineBasicBlock::iterator &MI,
                                    SIAtomicScope Scope,
                                    SIAtomicAddrSpace AddrSpace) const = 0;

  /// Update \p MI memory instruction of kind \p Op associated with address
  /// spaces \p AddrSpace to indicate it is volatile and/or
  /// nontemporal/last-use. Return true iff the instruction was modified.
  virtual bool enableVolatileAndOrNonTemporal(MachineBasicBlock::iterator &MI,
                                              SIAtomicAddrSpace AddrSpace,
                                              SIMemOp Op, bool IsVolatile,
                                              bool IsNonTemporal,
                                              bool IsLastUse = false) const = 0;

  virtual bool expandSystemScopeStore(MachineBasicBlock::iterator &MI) const {
    return false;
  };

  /// Inserts any necessary instructions at position \p Pos relative
  /// to instruction \p MI to ensure memory instructions before \p Pos of kind
  /// \p Op associated with address spaces \p AddrSpace have completed. Used
  /// between memory instructions to enforce the order they become visible as
  /// observed by other memory instructions executing in memory scope \p Scope.
  /// \p IsCrossAddrSpaceOrdering indicates if the memory ordering is between
  /// address spaces. Returns true iff any instructions inserted.
  virtual bool insertWait(MachineBasicBlock::iterator &MI, SIAtomicScope Scope,
                          SIAtomicAddrSpace AddrSpace, SIMemOp Op,
                          bool IsCrossAddrSpaceOrdering, Position Pos,
                          AtomicOrdering Order) const = 0;

  /// Inserts any necessary instructions at position \p Pos relative to
  /// instruction \p MI to ensure any subsequent memory instructions of this
  /// thread with address spaces \p AddrSpace will observe the previous memory
  /// operations by any thread for memory scopes up to memory scope \p Scope .
  /// Returns true iff any instructions inserted.
  virtual bool insertAcquire(MachineBasicBlock::iterator &MI,
                             SIAtomicScope Scope,
                             SIAtomicAddrSpace AddrSpace,
                             Position Pos) const = 0;

  /// Inserts any necessary instructions at position \p Pos relative to
  /// instruction \p MI to ensure previous memory instructions by this thread
  /// with address spaces \p AddrSpace have completed and can be observed by
  /// subsequent memory instructions by any thread executing in memory scope \p
  /// Scope. \p IsCrossAddrSpaceOrdering indicates if the memory ordering is
  /// between address spaces. Returns true iff any instructions inserted.
  virtual bool insertRelease(MachineBasicBlock::iterator &MI,
                             SIAtomicScope Scope,
                             SIAtomicAddrSpace AddrSpace,
                             bool IsCrossAddrSpaceOrdering,
                             Position Pos) const = 0;

  /// Virtual destructor to allow derivations to be deleted.
  virtual ~SICacheControl() = default;
};
```
<a name="block_4"></a>**File:** llvm/lib/Target/AMDGPU/SIMemoryLegalizer.cpp (L630-669) [<sup>↩</sup>](#ref-block_4)
```cpp
class SIMemoryLegalizer final {
private:
  const MachineModuleInfo &MMI;
  /// Cache Control.
  std::unique_ptr<SICacheControl> CC = nullptr;

  /// List of atomic pseudo instructions.
  std::list<MachineBasicBlock::iterator> AtomicPseudoMIs;

  /// Return true iff instruction \p MI is a atomic instruction that
  /// returns a result.
  bool isAtomicRet(const MachineInstr &MI) const {
    return SIInstrInfo::isAtomicRet(MI);
  }

  /// Removes all processed atomic pseudo instructions from the current
  /// function. Returns true if current function is modified, false otherwise.
  bool removeAtomicPseudoMIs();

  /// Expands load operation \p MI. Returns true if instructions are
  /// added/deleted or \p MI is modified, false otherwise.
  bool expandLoad(const SIMemOpInfo &MOI,
                  MachineBasicBlock::iterator &MI);
  /// Expands store operation \p MI. Returns true if instructions are
  /// added/deleted or \p MI is modified, false otherwise.
  bool expandStore(const SIMemOpInfo &MOI,
                   MachineBasicBlock::iterator &MI);
  /// Expands atomic fence operation \p MI. Returns true if
  /// instructions are added/deleted or \p MI is modified, false otherwise.
  bool expandAtomicFence(const SIMemOpInfo &MOI,
                         MachineBasicBlock::iterator &MI);
  /// Expands atomic cmpxchg or rmw operation \p MI. Returns true if
  /// instructions are added/deleted or \p MI is modified, false otherwise.
  bool expandAtomicCmpxchgOrRmw(const SIMemOpInfo &MOI,
                                MachineBasicBlock::iterator &MI);

public:
  SIMemoryLegalizer(const MachineModuleInfo &MMI) : MMI(MMI) {};
  bool run(MachineFunction &MF);
};
```
<a name="block_5"></a>**File:** llvm/lib/Target/AMDGPU/SIMemoryLegalizer.cpp (L741-773) [<sup>↩</sup>](#ref-block_5)
```cpp
std::optional<std::tuple<SIAtomicScope, SIAtomicAddrSpace, bool>>
SIMemOpAccess::toSIAtomicScope(SyncScope::ID SSID,
                               SIAtomicAddrSpace InstrAddrSpace) const {
  if (SSID == SyncScope::System)
    return std::tuple(SIAtomicScope::SYSTEM, SIAtomicAddrSpace::ATOMIC, true);
  if (SSID == MMI->getAgentSSID())
    return std::tuple(SIAtomicScope::AGENT, SIAtomicAddrSpace::ATOMIC, true);
  if (SSID == MMI->getWorkgroupSSID())
    return std::tuple(SIAtomicScope::WORKGROUP, SIAtomicAddrSpace::ATOMIC,
                      true);
  if (SSID == MMI->getWavefrontSSID())
    return std::tuple(SIAtomicScope::WAVEFRONT, SIAtomicAddrSpace::ATOMIC,
                      true);
  if (SSID == SyncScope::SingleThread)
    return std::tuple(SIAtomicScope::SINGLETHREAD, SIAtomicAddrSpace::ATOMIC,
                      true);
  if (SSID == MMI->getSystemOneAddressSpaceSSID())
    return std::tuple(SIAtomicScope::SYSTEM,
                      SIAtomicAddrSpace::ATOMIC & InstrAddrSpace, false);
  if (SSID == MMI->getAgentOneAddressSpaceSSID())
    return std::tuple(SIAtomicScope::AGENT,
                      SIAtomicAddrSpace::ATOMIC & InstrAddrSpace, false);
  if (SSID == MMI->getWorkgroupOneAddressSpaceSSID())
    return std::tuple(SIAtomicScope::WORKGROUP,
                      SIAtomicAddrSpace::ATOMIC & InstrAddrSpace, false);
  if (SSID == MMI->getWavefrontOneAddressSpaceSSID())
    return std::tuple(SIAtomicScope::WAVEFRONT,
                      SIAtomicAddrSpace::ATOMIC & InstrAddrSpace, false);
  if (SSID == MMI->getSingleThreadOneAddressSpaceSSID())
    return std::tuple(SIAtomicScope::SINGLETHREAD,
                      SIAtomicAddrSpace::ATOMIC & InstrAddrSpace, false);
  return std::nullopt;
}
```
<a name="block_6"></a>**File:** llvm/lib/Target/AMDGPU/SIMemoryLegalizer.cpp (L775-788) [<sup>↩</sup>](#ref-block_6)
```cpp
SIAtomicAddrSpace SIMemOpAccess::toSIAtomicAddrSpace(unsigned AS) const {
  if (AS == AMDGPUAS::FLAT_ADDRESS)
    return SIAtomicAddrSpace::FLAT;
  if (AS == AMDGPUAS::GLOBAL_ADDRESS)
    return SIAtomicAddrSpace::GLOBAL;
  if (AS == AMDGPUAS::LOCAL_ADDRESS)
    return SIAtomicAddrSpace::LDS;
  if (AS == AMDGPUAS::PRIVATE_ADDRESS)
    return SIAtomicAddrSpace::SCRATCH;
  if (AS == AMDGPUAS::REGION_ADDRESS)
    return SIAtomicAddrSpace::GDS;

  return SIAtomicAddrSpace::OTHER;
}
```
<a name="block_7"></a>**File:** llvm/lib/Target/AMDGPU/SIMemoryLegalizer.cpp (L964-997) [<sup>↩</sup>](#ref-block_7)
```cpp
bool SIGfx6CacheControl::enableLoadCacheBypass(
    const MachineBasicBlock::iterator &MI,
    SIAtomicScope Scope,
    SIAtomicAddrSpace AddrSpace) const {
  assert(MI->mayLoad() && !MI->mayStore());
  bool Changed = false;

  if ((AddrSpace & SIAtomicAddrSpace::GLOBAL) != SIAtomicAddrSpace::NONE) {
    switch (Scope) {
    case SIAtomicScope::SYSTEM:
    case SIAtomicScope::AGENT:
      // Set L1 cache policy to MISS_EVICT.
      // Note: there is no L2 cache bypass policy at the ISA level.
      Changed |= enableGLCBit(MI);
      break;
    case SIAtomicScope::WORKGROUP:
    case SIAtomicScope::WAVEFRONT:
    case SIAtomicScope::SINGLETHREAD:
      // No cache to bypass.
      break;
    default:
      llvm_unreachable("Unsupported synchronization scope");
    }
  }

  /// The scratch address space does not need the global memory caches
  /// to be bypassed as all memory operations by the same thread are
  /// sequentially consistent, and no other thread can access scratch
  /// memory.

  /// Other address spaces do not have a cache.

  return Changed;
}
```
<a name="block_8"></a>**File:** llvm/lib/Target/AMDGPU/SIMemoryLegalizer.cpp (L1072-1167) [<sup>↩</sup>](#ref-block_8)
```cpp
bool SIGfx6CacheControl::insertWait(MachineBasicBlock::iterator &MI,
                                    SIAtomicScope Scope,
                                    SIAtomicAddrSpace AddrSpace, SIMemOp Op,
                                    bool IsCrossAddrSpaceOrdering, Position Pos,
                                    AtomicOrdering Order) const {
  bool Changed = false;

  MachineBasicBlock &MBB = *MI->getParent();
  DebugLoc DL = MI->getDebugLoc();

  if (Pos == Position::AFTER)
    ++MI;

  bool VMCnt = false;
  bool LGKMCnt = false;

  if ((AddrSpace & (SIAtomicAddrSpace::GLOBAL | SIAtomicAddrSpace::SCRATCH)) !=
      SIAtomicAddrSpace::NONE) {
    switch (Scope) {
    case SIAtomicScope::SYSTEM:
    case SIAtomicScope::AGENT:
      VMCnt |= true;
      break;
    case SIAtomicScope::WORKGROUP:
    case SIAtomicScope::WAVEFRONT:
    case SIAtomicScope::SINGLETHREAD:
      // The L1 cache keeps all memory operations in order for
      // wavefronts in the same work-group.
      break;
    default:
      llvm_unreachable("Unsupported synchronization scope");
    }
  }

  if ((AddrSpace & SIAtomicAddrSpace::LDS) != SIAtomicAddrSpace::NONE) {
    switch (Scope) {
    case SIAtomicScope::SYSTEM:
    case SIAtomicScope::AGENT:
    case SIAtomicScope::WORKGROUP:
      // If no cross address space ordering then an "S_WAITCNT lgkmcnt(0)" is
      // not needed as LDS operations for all waves are executed in a total
      // global ordering as observed by all waves. Required if also
      // synchronizing with global/GDS memory as LDS operations could be
      // reordered with respect to later global/GDS memory operations of the
      // same wave.
      LGKMCnt |= IsCrossAddrSpaceOrdering;
      break;
    case SIAtomicScope::WAVEFRONT:
    case SIAtomicScope::SINGLETHREAD:
      // The LDS keeps all memory operations in order for
      // the same wavefront.
      break;
    default:
      llvm_unreachable("Unsupported synchronization scope");
    }
  }

  if ((AddrSpace & SIAtomicAddrSpace::GDS) != SIAtomicAddrSpace::NONE) {
    switch (Scope) {
    case SIAtomicScope::SYSTEM:
    case SIAtomicScope::AGENT:
      // If no cross address space ordering then an GDS "S_WAITCNT lgkmcnt(0)"
      // is not needed as GDS operations for all waves are executed in a total
      // global ordering as observed by all waves. Required if also
      // synchronizing with global/LDS memory as GDS operations could be
      // reordered with respect to later global/LDS memory operations of the
      // same wave.
      LGKMCnt |= IsCrossAddrSpaceOrdering;
      break;
    case SIAtomicScope::WORKGROUP:
    case SIAtomicScope::WAVEFRONT:
    case SIAtomicScope::SINGLETHREAD:
      // The GDS keeps all memory operations in order for
      // the same work-group.
      break;
    default:
      llvm_unreachable("Unsupported synchronization scope");
    }
  }

  if (VMCnt || LGKMCnt) {
    unsigned WaitCntImmediate =
      AMDGPU::encodeWaitcnt(IV,
                            VMCnt ? 0 : getVmcntBitMask(IV),
                            getExpcntBitMask(IV),
                            LGKMCnt ? 0 : getLgkmcntBitMask(IV));
    BuildMI(MBB, MI, DL, TII->get(AMDGPU::S_WAITCNT_soft))
        .addImm(WaitCntImmediate);
    Changed = true;
  }

  if (Pos == Position::AFTER)
    --MI;

  return Changed;
}
```
<a name="block_9"></a>**File:** llvm/lib/Target/AMDGPU/SIMemoryLegalizer.cpp (L1274-1314) [<sup>↩</sup>](#ref-block_9)
```cpp
bool SIGfx90ACacheControl::enableLoadCacheBypass(
    const MachineBasicBlock::iterator &MI,
    SIAtomicScope Scope,
    SIAtomicAddrSpace AddrSpace) const {
  assert(MI->mayLoad() && !MI->mayStore());
  bool Changed = false;

  if ((AddrSpace & SIAtomicAddrSpace::GLOBAL) != SIAtomicAddrSpace::NONE) {
    switch (Scope) {
    case SIAtomicScope::SYSTEM:
    case SIAtomicScope::AGENT:
      // Set the L1 cache policy to MISS_LRU.
      // Note: there is no L2 cache bypass policy at the ISA level.
      Changed |= enableGLCBit(MI);
      break;
    case SIAtomicScope::WORKGROUP:
      // In threadgroup split mode the waves of a work-group can be executing on
      // different CUs. Therefore need to bypass the L1 which is per CU.
      // Otherwise in non-threadgroup split mode all waves of a work-group are
      // on the same CU, and so the L1 does not need to be bypassed.
      if (ST.isTgSplitEnabled())
        Changed |= enableGLCBit(MI);
      break;
    case SIAtomicScope::WAVEFRONT:
    case SIAtomicScope::SINGLETHREAD:
      // No cache to bypass.
      break;
    default:
      llvm_unreachable("Unsupported synchronization scope");
    }
  }

  /// The scratch address space does not need the global memory caches
  /// to be bypassed as all memory operations by the same thread are
  /// sequentially consistent, and no other thread can access scratch
  /// memory.

  /// Other address spaces do not have a cache.

  return Changed;
}
```
<a name="block_10"></a>**File:** llvm/lib/Target/AMDGPU/SIMemoryLegalizer.cpp (L1568-1610) [<sup>↩</sup>](#ref-block_10)
```cpp
bool SIGfx940CacheControl::enableLoadCacheBypass(
    const MachineBasicBlock::iterator &MI, SIAtomicScope Scope,
    SIAtomicAddrSpace AddrSpace) const {
  assert(MI->mayLoad() && !MI->mayStore());
  bool Changed = false;

  if ((AddrSpace & SIAtomicAddrSpace::GLOBAL) != SIAtomicAddrSpace::NONE) {
    switch (Scope) {
    case SIAtomicScope::SYSTEM:
      // Set SC bits to indicate system scope.
      Changed |= enableSC0Bit(MI);
      Changed |= enableSC1Bit(MI);
      break;
    case SIAtomicScope::AGENT:
      // Set SC bits to indicate agent scope.
      Changed |= enableSC1Bit(MI);
      break;
    case SIAtomicScope::WORKGROUP:
      // In threadgroup split mode the waves of a work-group can be executing on
      // different CUs. Therefore need to bypass the L1 which is per CU.
      // Otherwise in non-threadgroup split mode all waves of a work-group are
      // on the same CU, and so the L1 does not need to be bypassed. Setting SC
      // bits to indicate work-group scope will do this automatically.
      Changed |= enableSC0Bit(MI);
      break;
    case SIAtomicScope::WAVEFRONT:
    case SIAtomicScope::SINGLETHREAD:
      // Leave SC bits unset to indicate wavefront scope.
      break;
    default:
      llvm_unreachable("Unsupported synchronization scope");
    }
  }

  /// The scratch address space does not need the global memory caches
  /// to be bypassed as all memory operations by the same thread are
  /// sequentially consistent, and no other thread can access scratch
  /// memory.

  /// Other address spaces do not have a cache.

  return Changed;
}
```
<a name="block_11"></a>**File:** llvm/lib/Target/AMDGPU/SIMemoryLegalizer.cpp (L1870-1911) [<sup>↩</sup>](#ref-block_11)
```cpp
bool SIGfx10CacheControl::enableLoadCacheBypass(
    const MachineBasicBlock::iterator &MI,
    SIAtomicScope Scope,
    SIAtomicAddrSpace AddrSpace) const {
  assert(MI->mayLoad() && !MI->mayStore());
  bool Changed = false;

  if ((AddrSpace & SIAtomicAddrSpace::GLOBAL) != SIAtomicAddrSpace::NONE) {
    switch (Scope) {
    case SIAtomicScope::SYSTEM:
    case SIAtomicScope::AGENT:
      // Set the L0 and L1 cache policies to MISS_EVICT.
      // Note: there is no L2 cache coherent bypass control at the ISA level.
      Changed |= enableGLCBit(MI);
      Changed |= enableDLCBit(MI);
      break;
    case SIAtomicScope::WORKGROUP:
      // In WGP mode the waves of a work-group can be executing on either CU of
      // the WGP. Therefore need to bypass the L0 which is per CU. Otherwise in
      // CU mode all waves of a work-group are on the same CU, and so the L0
      // does not need to be bypassed.
      if (!ST.isCuModeEnabled())
        Changed |= enableGLCBit(MI);
      break;
    case SIAtomicScope::WAVEFRONT:
    case SIAtomicScope::SINGLETHREAD:
      // No cache to bypass.
      break;
    default:
      llvm_unreachable("Unsupported synchronization scope");
    }
  }

  /// The scratch address space does not need the global memory caches
  /// to be bypassed as all memory operations by the same thread are
  /// sequentially consistent, and no other thread can access scratch
  /// memory.

  /// Other address spaces do not have a cache.

  return Changed;
}
```
<a name="block_12"></a>**File:** llvm/lib/Target/AMDGPU/SIMemoryLegalizer.cpp (L1964-2082) [<sup>↩</sup>](#ref-block_12)
```cpp
bool SIGfx10CacheControl::insertWait(MachineBasicBlock::iterator &MI,
                                     SIAtomicScope Scope,
                                     SIAtomicAddrSpace AddrSpace, SIMemOp Op,
                                     bool IsCrossAddrSpaceOrdering,
                                     Position Pos, AtomicOrdering Order) const {
  bool Changed = false;

  MachineBasicBlock &MBB = *MI->getParent();
  DebugLoc DL = MI->getDebugLoc();

  if (Pos == Position::AFTER)
    ++MI;

  bool VMCnt = false;
  bool VSCnt = false;
  bool LGKMCnt = false;

  if ((AddrSpace & (SIAtomicAddrSpace::GLOBAL | SIAtomicAddrSpace::SCRATCH)) !=
      SIAtomicAddrSpace::NONE) {
    switch (Scope) {
    case SIAtomicScope::SYSTEM:
    case SIAtomicScope::AGENT:
      if ((Op & SIMemOp::LOAD) != SIMemOp::NONE)
        VMCnt |= true;
      if ((Op & SIMemOp::STORE) != SIMemOp::NONE)
        VSCnt |= true;
      break;
    case SIAtomicScope::WORKGROUP:
      // In WGP mode the waves of a work-group can be executing on either CU of
      // the WGP. Therefore need to wait for operations to complete to ensure
      // they are visible to waves in the other CU as the L0 is per CU.
      // Otherwise in CU mode and all waves of a work-group are on the same CU
      // which shares the same L0.
      if (!ST.isCuModeEnabled()) {
        if ((Op & SIMemOp::LOAD) != SIMemOp::NONE)
          VMCnt |= true;
        if ((Op & SIMemOp::STORE) != SIMemOp::NONE)
          VSCnt |= true;
      }
      break;
    case SIAtomicScope::WAVEFRONT:
    case SIAtomicScope::SINGLETHREAD:
      // The L0 cache keeps all memory operations in order for
      // work-items in the same wavefront.
      break;
    default:
      llvm_unreachable("Unsupported synchronization scope");
    }
  }

  if ((AddrSpace & SIAtomicAddrSpace::LDS) != SIAtomicAddrSpace::NONE) {
    switch (Scope) {
    case SIAtomicScope::SYSTEM:
    case SIAtomicScope::AGENT:
    case SIAtomicScope::WORKGROUP:
      // If no cross address space ordering then an "S_WAITCNT lgkmcnt(0)" is
      // not needed as LDS operations for all waves are executed in a total
      // global ordering as observed by all waves. Required if also
      // synchronizing with global/GDS memory as LDS operations could be
      // reordered with respect to later global/GDS memory operations of the
      // same wave.
      LGKMCnt |= IsCrossAddrSpaceOrdering;
      break;
    case SIAtomicScope::WAVEFRONT:
    case SIAtomicScope::SINGLETHREAD:
      // The LDS keeps all memory operations in order for
      // the same wavefront.
      break;
    default:
      llvm_unreachable("Unsupported synchronization scope");
    }
  }

  if ((AddrSpace & SIAtomicAddrSpace::GDS) != SIAtomicAddrSpace::NONE) {
    switch (Scope) {
    case SIAtomicScope::SYSTEM:
    case SIAtomicScope::AGENT:
      // If no cross address space ordering then an GDS "S_WAITCNT lgkmcnt(0)"
      // is not needed as GDS operations for all waves are executed in a total
      // global ordering as observed by all waves. Required if also
      // synchronizing with global/LDS memory as GDS operations could be
      // reordered with respect to later global/LDS memory operations of the
      // same wave.
      LGKMCnt |= IsCrossAddrSpaceOrdering;
      break;
    case SIAtomicScope::WORKGROUP:
    case SIAtomicScope::WAVEFRONT:
    case SIAtomicScope::SINGLETHREAD:
      // The GDS keeps all memory operations in order for
      // the same work-group.
      break;
    default:
      llvm_unreachable("Unsupported synchronization scope");
    }
  }

  if (VMCnt || LGKMCnt) {
    unsigned WaitCntImmediate =
      AMDGPU::encodeWaitcnt(IV,
                            VMCnt ? 0 : getVmcntBitMask(IV),
                            getExpcntBitMask(IV),
                            LGKMCnt ? 0 : getLgkmcntBitMask(IV));
    BuildMI(MBB, MI, DL, TII->get(AMDGPU::S_WAITCNT_soft))
        .addImm(WaitCntImmediate);
    Changed = true;
  }

  if (VSCnt) {
    BuildMI(MBB, MI, DL, TII->get(AMDGPU::S_WAITCNT_VSCNT_soft))
        .addReg(AMDGPU::SGPR_NULL, RegState::Undef)
        .addImm(0);
    Changed = true;
  }

  if (Pos == Position::AFTER)
    --MI;

  return Changed;
}
```
<a name="block_13"></a>**File:** llvm/lib/Target/AMDGPU/SIMemoryLegalizer.cpp (L2142-2181) [<sup>↩</sup>](#ref-block_13)
```cpp
bool SIGfx11CacheControl::enableLoadCacheBypass(
    const MachineBasicBlock::iterator &MI, SIAtomicScope Scope,
    SIAtomicAddrSpace AddrSpace) const {
  assert(MI->mayLoad() && !MI->mayStore());
  bool Changed = false;

  if ((AddrSpace & SIAtomicAddrSpace::GLOBAL) != SIAtomicAddrSpace::NONE) {
    switch (Scope) {
    case SIAtomicScope::SYSTEM:
    case SIAtomicScope::AGENT:
      // Set the L0 and L1 cache policies to MISS_EVICT.
      // Note: there is no L2 cache coherent bypass control at the ISA level.
      Changed |= enableGLCBit(MI);
      break;
    case SIAtomicScope::WORKGROUP:
      // In WGP mode the waves of a work-group can be executing on either CU of
      // the WGP. Therefore need to bypass the L0 which is per CU. Otherwise in
      // CU mode all waves of a work-group are on the same CU, and so the L0
      // does not need to be bypassed.
      if (!ST.isCuModeEnabled())
        Changed |= enableGLCBit(MI);
      break;
    case SIAtomicScope::WAVEFRONT:
    case SIAtomicScope::SINGLETHREAD:
      // No cache to bypass.
      break;
    default:
      llvm_unreachable("Unsupported synchronization scope");
    }
  }

  /// The scratch address space does not need the global memory caches
  /// to be bypassed as all memory operations by the same thread are
  /// sequentially consistent, and no other thread can access scratch
  /// memory.

  /// Other address spaces do not have a cache.

  return Changed;
}
```
<a name="block_14"></a>**File:** llvm/lib/Target/AMDGPU/SIMemoryLegalizer.cpp (L2285-2392) [<sup>↩</sup>](#ref-block_14)
```cpp
bool SIGfx12CacheControl::insertWait(MachineBasicBlock::iterator &MI,
                                     SIAtomicScope Scope,
                                     SIAtomicAddrSpace AddrSpace, SIMemOp Op,
                                     bool IsCrossAddrSpaceOrdering,
                                     Position Pos, AtomicOrdering Order) const {
  bool Changed = false;

  MachineBasicBlock &MBB = *MI->getParent();
  DebugLoc DL = MI->getDebugLoc();

  bool LOADCnt = false;
  bool DSCnt = false;
  bool STORECnt = false;

  if (Pos == Position::AFTER)
    ++MI;

  if ((AddrSpace & (SIAtomicAddrSpace::GLOBAL | SIAtomicAddrSpace::SCRATCH)) !=
      SIAtomicAddrSpace::NONE) {
    switch (Scope) {
    case SIAtomicScope::SYSTEM:
    case SIAtomicScope::AGENT:
      if ((Op & SIMemOp::LOAD) != SIMemOp::NONE)
        LOADCnt |= true;
      if ((Op & SIMemOp::STORE) != SIMemOp::NONE)
        STORECnt |= true;
      break;
    case SIAtomicScope::WORKGROUP:
      // In WGP mode the waves of a work-group can be executing on either CU of
      // the WGP. Therefore need to wait for operations to complete to ensure
      // they are visible to waves in the other CU as the L0 is per CU.
      // Otherwise in CU mode and all waves of a work-group are on the same CU
      // which shares the same L0.
      if (!ST.isCuModeEnabled()) {
        if ((Op & SIMemOp::LOAD) != SIMemOp::NONE)
          LOADCnt |= true;
        if ((Op & SIMemOp::STORE) != SIMemOp::NONE)
          STORECnt |= true;
      }
      break;
    case SIAtomicScope::WAVEFRONT:
    case SIAtomicScope::SINGLETHREAD:
      // The L0 cache keeps all memory operations in order for
      // work-items in the same wavefront.
      break;
    default:
      llvm_unreachable("Unsupported synchronization scope");
    }
  }

  if ((AddrSpace & SIAtomicAddrSpace::LDS) != SIAtomicAddrSpace::NONE) {
    switch (Scope) {
    case SIAtomicScope::SYSTEM:
    case SIAtomicScope::AGENT:
    case SIAtomicScope::WORKGROUP:
      // If no cross address space ordering then an "S_WAITCNT lgkmcnt(0)" is
      // not needed as LDS operations for all waves are executed in a total
      // global ordering as observed by all waves. Required if also
      // synchronizing with global/GDS memory as LDS operations could be
      // reordered with respect to later global/GDS memory operations of the
      // same wave.
      DSCnt |= IsCrossAddrSpaceOrdering;
      break;
    case SIAtomicScope::WAVEFRONT:
    case SIAtomicScope::SINGLETHREAD:
      // The LDS keeps all memory operations in order for
      // the same wavefront.
      break;
    default:
      llvm_unreachable("Unsupported synchronization scope");
    }
  }

  if (LOADCnt) {
    // Acquire sequences only need to wait on the previous atomic operation.
    // e.g. a typical sequence looks like
    //    atomic load
    //    (wait)
    //    global_inv
    //
    // We do not have BVH or SAMPLE atomics, so the atomic load is always going
    // to be tracked using loadcnt.
    //
    // This also applies to fences. Fences cannot pair with an instruction
    // tracked with bvh/samplecnt as we don't have any atomics that do that.
    if (Order != AtomicOrdering::Acquire) {
      BuildMI(MBB, MI, DL, TII->get(AMDGPU::S_WAIT_BVHCNT_soft)).addImm(0);
      BuildMI(MBB, MI, DL, TII->get(AMDGPU::S_WAIT_SAMPLECNT_soft)).addImm(0);
    }
    BuildMI(MBB, MI, DL, TII->get(AMDGPU::S_WAIT_LOADCNT_soft)).addImm(0);
    Changed = true;
  }

  if (STORECnt) {
    BuildMI(MBB, MI, DL, TII->get(AMDGPU::S_WAIT_STORECNT_soft)).addImm(0);
    Changed = true;
  }

  if (DSCnt) {
    BuildMI(MBB, MI, DL, TII->get(AMDGPU::S_WAIT_DSCNT_soft)).addImm(0);
    Changed = true;
  }

  if (Pos == Position::AFTER)
    --MI;

  return Changed;
}
```
<a name="block_15"></a>**File:** llvm/lib/Target/AMDGPU/SIMemoryLegalizer.cpp (L2553-2589) [<sup>↩</sup>](#ref-block_15)
```cpp
bool SIGfx12CacheControl::setAtomicScope(const MachineBasicBlock::iterator &MI,
                                         SIAtomicScope Scope,
                                         SIAtomicAddrSpace AddrSpace) const {
  bool Changed = false;

  if ((AddrSpace & SIAtomicAddrSpace::GLOBAL) != SIAtomicAddrSpace::NONE) {
    switch (Scope) {
    case SIAtomicScope::SYSTEM:
      Changed |= setScope(MI, AMDGPU::CPol::SCOPE_SYS);
      break;
    case SIAtomicScope::AGENT:
      Changed |= setScope(MI, AMDGPU::CPol::SCOPE_DEV);
      break;
    case SIAtomicScope::WORKGROUP:
      // In workgroup mode, SCOPE_SE is needed as waves can executes on
      // different CUs that access different L0s.
      if (!ST.isCuModeEnabled())
        Changed |= setScope(MI, AMDGPU::CPol::SCOPE_SE);
      break;
    case SIAtomicScope::WAVEFRONT:
    case SIAtomicScope::SINGLETHREAD:
      // No cache to bypass.
      break;
    default:
      llvm_unreachable("Unsupported synchronization scope");
    }
  }

  // The scratch address space does not need the global memory caches
  // to be bypassed as all memory operations by the same thread are
  // sequentially consistent, and no other thread can access scratch
  // memory.

  // Other address spaces do not have a cache.

  return Changed;
}
```
<a name="block_16"></a>**File:** llvm/lib/Target/AMDGPU/SIMemoryLegalizer.cpp (L2602-2644) [<sup>↩</sup>](#ref-block_16)
```cpp
bool SIMemoryLegalizer::expandLoad(const SIMemOpInfo &MOI,
                                   MachineBasicBlock::iterator &MI) {
  assert(MI->mayLoad() && !MI->mayStore());

  bool Changed = false;

  if (MOI.isAtomic()) {
    const AtomicOrdering Order = MOI.getOrdering();
    if (Order == AtomicOrdering::Monotonic ||
        Order == AtomicOrdering::Acquire ||
        Order == AtomicOrdering::SequentiallyConsistent) {
      Changed |= CC->enableLoadCacheBypass(MI, MOI.getScope(),
                                           MOI.getOrderingAddrSpace());
    }

    if (Order == AtomicOrdering::SequentiallyConsistent)
      Changed |= CC->insertWait(MI, MOI.getScope(), MOI.getOrderingAddrSpace(),
                                SIMemOp::LOAD | SIMemOp::STORE,
                                MOI.getIsCrossAddressSpaceOrdering(),
                                Position::BEFORE, Order);

    if (Order == AtomicOrdering::Acquire ||
        Order == AtomicOrdering::SequentiallyConsistent) {
      Changed |= CC->insertWait(
          MI, MOI.getScope(), MOI.getInstrAddrSpace(), SIMemOp::LOAD,
          MOI.getIsCrossAddressSpaceOrdering(), Position::AFTER, Order);
      Changed |= CC->insertAcquire(MI, MOI.getScope(),
                                   MOI.getOrderingAddrSpace(),
                                   Position::AFTER);
    }

    return Changed;
  }

  // Atomic instructions already bypass caches to the scope specified by the
  // SyncScope operand. Only non-atomic volatile and nontemporal/last-use
  // instructions need additional treatment.
  Changed |= CC->enableVolatileAndOrNonTemporal(
      MI, MOI.getInstrAddrSpace(), SIMemOp::LOAD, MOI.isVolatile(),
      MOI.isNonTemporal(), MOI.isLastUse());

  return Changed;
}
```
<a name="block_17"></a>**File:** llvm/lib/Target/AMDGPU/SIMemoryLegalizer.cpp (L2646-2681) [<sup>↩</sup>](#ref-block_17)
```cpp
bool SIMemoryLegalizer::expandStore(const SIMemOpInfo &MOI,
                                    MachineBasicBlock::iterator &MI) {
  assert(!MI->mayLoad() && MI->mayStore());

  bool Changed = false;

  if (MOI.isAtomic()) {
    if (MOI.getOrdering() == AtomicOrdering::Monotonic ||
        MOI.getOrdering() == AtomicOrdering::Release ||
        MOI.getOrdering() == AtomicOrdering::SequentiallyConsistent) {
      Changed |= CC->enableStoreCacheBypass(MI, MOI.getScope(),
                                            MOI.getOrderingAddrSpace());
    }

    if (MOI.getOrdering() == AtomicOrdering::Release ||
        MOI.getOrdering() == AtomicOrdering::SequentiallyConsistent)
      Changed |= CC->insertRelease(MI, MOI.getScope(),
                                   MOI.getOrderingAddrSpace(),
                                   MOI.getIsCrossAddressSpaceOrdering(),
                                   Position::BEFORE);

    return Changed;
  }

  // Atomic instructions already bypass caches to the scope specified by the
  // SyncScope operand. Only non-atomic volatile and nontemporal instructions
  // need additional treatment.
  Changed |= CC->enableVolatileAndOrNonTemporal(
      MI, MOI.getInstrAddrSpace(), SIMemOp::STORE, MOI.isVolatile(),
      MOI.isNonTemporal());

  // GFX12 specific, scope(desired coherence domain in cache hierarchy) is
  // instruction field, do not confuse it with atomic scope.
  Changed |= CC->expandSystemScopeStore(MI);
  return Changed;
}
```
<a name="block_18"></a>**File:** llvm/lib/Target/AMDGPU/SIMemoryLegalizer.cpp (L2683-2733) [<sup>↩</sup>](#ref-block_18)
```cpp
bool SIMemoryLegalizer::expandAtomicFence(const SIMemOpInfo &MOI,
                                          MachineBasicBlock::iterator &MI) {
  assert(MI->getOpcode() == AMDGPU::ATOMIC_FENCE);

  AtomicPseudoMIs.push_back(MI);
  bool Changed = false;

  // Refine fenced address space based on MMRAs.
  //
  // TODO: Should we support this MMRA on other atomic operations?
  auto OrderingAddrSpace =
      getFenceAddrSpaceMMRA(*MI, MOI.getOrderingAddrSpace());

  if (MOI.isAtomic()) {
    const AtomicOrdering Order = MOI.getOrdering();
    if (Order == AtomicOrdering::Acquire) {
      Changed |= CC->insertWait(
          MI, MOI.getScope(), OrderingAddrSpace, SIMemOp::LOAD | SIMemOp::STORE,
          MOI.getIsCrossAddressSpaceOrdering(), Position::BEFORE, Order);
    }

    if (Order == AtomicOrdering::Release ||
        Order == AtomicOrdering::AcquireRelease ||
        Order == AtomicOrdering::SequentiallyConsistent)
      /// TODO: This relies on a barrier always generating a waitcnt
      /// for LDS to ensure it is not reordered with the completion of
      /// the proceeding LDS operations. If barrier had a memory
      /// ordering and memory scope, then library does not need to
      /// generate a fence. Could add support in this file for
      /// barrier. SIInsertWaitcnt.cpp could then stop unconditionally
      /// adding S_WAITCNT before a S_BARRIER.
      Changed |= CC->insertRelease(MI, MOI.getScope(), OrderingAddrSpace,
                                   MOI.getIsCrossAddressSpaceOrdering(),
                                   Position::BEFORE);

    // TODO: If both release and invalidate are happening they could be combined
    // to use the single "BUFFER_WBINV*" instruction. This could be done by
    // reorganizing this code or as part of optimizing SIInsertWaitcnt pass to
    // track cache invalidate and write back instructions.

    if (Order == AtomicOrdering::Acquire ||
        Order == AtomicOrdering::AcquireRelease ||
        Order == AtomicOrdering::SequentiallyConsistent)
      Changed |= CC->insertAcquire(MI, MOI.getScope(), OrderingAddrSpace,
                                   Position::BEFORE);

    return Changed;
  }

  return Changed;
}
```
<a name="block_19"></a>**File:** llvm/lib/Target/AMDGPU/SIMemoryLegalizer.cpp (L2735-2778) [<sup>↩</sup>](#ref-block_19)
```cpp
bool SIMemoryLegalizer::expandAtomicCmpxchgOrRmw(const SIMemOpInfo &MOI,
  MachineBasicBlock::iterator &MI) {
  assert(MI->mayLoad() && MI->mayStore());

  bool Changed = false;

  if (MOI.isAtomic()) {
    const AtomicOrdering Order = MOI.getOrdering();
    if (Order == AtomicOrdering::Monotonic ||
        Order == AtomicOrdering::Acquire || Order == AtomicOrdering::Release ||
        Order == AtomicOrdering::AcquireRelease ||
        Order == AtomicOrdering::SequentiallyConsistent) {
      Changed |= CC->enableRMWCacheBypass(MI, MOI.getScope(),
                                          MOI.getInstrAddrSpace());
    }

    if (Order == AtomicOrdering::Release ||
        Order == AtomicOrdering::AcquireRelease ||
        Order == AtomicOrdering::SequentiallyConsistent ||
        MOI.getFailureOrdering() == AtomicOrdering::SequentiallyConsistent)
      Changed |= CC->insertRelease(MI, MOI.getScope(),
                                   MOI.getOrderingAddrSpace(),
                                   MOI.getIsCrossAddressSpaceOrdering(),
                                   Position::BEFORE);

    if (Order == AtomicOrdering::Acquire ||
        Order == AtomicOrdering::AcquireRelease ||
        Order == AtomicOrdering::SequentiallyConsistent ||
        MOI.getFailureOrdering() == AtomicOrdering::Acquire ||
        MOI.getFailureOrdering() == AtomicOrdering::SequentiallyConsistent) {
      Changed |= CC->insertWait(
          MI, MOI.getScope(), MOI.getInstrAddrSpace(),
          isAtomicRet(*MI) ? SIMemOp::LOAD : SIMemOp::STORE,
          MOI.getIsCrossAddressSpaceOrdering(), Position::AFTER, Order);
      Changed |= CC->insertAcquire(MI, MOI.getScope(),
                                   MOI.getOrderingAddrSpace(),
                                   Position::AFTER);
    }

    return Changed;
  }

  return Changed;
}
```
<a name="block_20"></a>**File:** llvm/lib/Target/AMDGPU/SIMemoryLegalizer.cpp (L2798-2838) [<sup>↩</sup>](#ref-block_20)
```cpp
bool SIMemoryLegalizer::run(MachineFunction &MF) {
  bool Changed = false;

  SIMemOpAccess MOA(MMI.getObjFileInfo<AMDGPUMachineModuleInfo>());
  CC = SICacheControl::create(MF.getSubtarget<GCNSubtarget>());

  for (auto &MBB : MF) {
    for (auto MI = MBB.begin(); MI != MBB.end(); ++MI) {

      // Unbundle instructions after the post-RA scheduler.
      if (MI->isBundle() && MI->mayLoadOrStore()) {
        MachineBasicBlock::instr_iterator II(MI->getIterator());
        for (MachineBasicBlock::instr_iterator I = ++II, E = MBB.instr_end();
             I != E && I->isBundledWithPred(); ++I) {
          I->unbundleFromPred();
          for (MachineOperand &MO : I->operands())
            if (MO.isReg())
              MO.setIsInternalRead(false);
        }

        MI->eraseFromParent();
        MI = II->getIterator();
      }

      if (!(MI->getDesc().TSFlags & SIInstrFlags::maybeAtomic))
        continue;

      if (const auto &MOI = MOA.getLoadInfo(MI))
        Changed |= expandLoad(*MOI, MI);
      else if (const auto &MOI = MOA.getStoreInfo(MI)) {
        Changed |= expandStore(*MOI, MI);
      } else if (const auto &MOI = MOA.getAtomicFenceInfo(MI))
        Changed |= expandAtomicFence(*MOI, MI);
      else if (const auto &MOI = MOA.getAtomicCmpxchgOrRmwInfo(MI))
        Changed |= expandAtomicCmpxchgOrRmw(*MOI, MI);
    }
  }

  Changed |= removeAtomicPseudoMIs();
  return Changed;
}
```