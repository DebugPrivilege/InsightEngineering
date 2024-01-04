# Parent of a Child Process

When a new process is created, the operating system records the **PID** of the **parent process** in the **`InheritedFromUniqueProcessId`** field of the child process's **`EPROCESS`** structure. This is how the operating system keeps track of the parent-child relationship between processes.

First, we'll begin by obtaining the memory address of the process we are focusing on:

```
2: kd> !mex.tl lsass
PID         Address          Name
=========== ================ =========
0x390 0n912 ffffa604f683b180 lsass.exe
=========== ================ =========
PID         Address          Name
```

This command displays the **`InheritedFromUniqueProcessId`** field of the **`_EPROCESS`** structure for a specific process, indicating that the parent process ID is **`0x00000000000002e0`**, which corresponds to the process **`wininit.exe`**

```
2: kd> !mex.ddt nt!_EPROCESS ffffa604f683b180 -y InheritedFromUniqueProcessId

dt nt!_EPROCESS ffffa604f683b180 -y InheritedFromUniqueProcessId () Recursive: [ -r1 -r2 -r ] Verbose Normal dt
==================================================================================
   +0x540 InheritedFromUniqueProcessId : 0x00000000`000002e0 Void  [process] (wininit.exe)
```

# Job Objects

A **Job Object** in Windows is for managing a group of processes together. Think of it as a "container" for processes, where you can apply certain rules and limits to all processes inside the container. 

Let's imagine we have several processes running as part of an program, and our goal is to make sure they collectively use no more than 1GB of RAM. We can achieve this by grouping all these processes into a **Job Object** and setting a memory limit of 1GB on it. This way, no matter the behavior of each individual process, the total memory usage for all processes in the **Job Object** will not exceed 1GB.

This example displays the properties of a **`WmiPrvSE.exe`** process, showing it's associated with a **Job Object** named **`WmiProviderSubSystemHostJob`**, which includes information on the process within the job and the job limits, such as memory restrictions and the number of active processes.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/02829d7e-15f2-457e-95a3-6635db52c35c)


The **`nt!_EJOB`** structure is a kernel object in Windows that contains information about a Job Object:

```
lkd> dt nt!_ejob ffffa604f683b180
+0x000 Event                : _KEVENT
+0x018 JobLinks             : _LIST_ENTRY [ 0xffffa604'f684e550 - 0xffffa604'f684e550 ]
+0x028 ProcessListHead      : _LIST_ENTRY [ 0xffffa604'f685a420 - 0xffffa604'f685a420 ]
+0x038 JobLock              : _ERESOURCE
+0x0a0 TotalUserTime        : _LARGE_INTEGER 0x1d4c58
+0x0a8 TotalKernelTime      : _LARGE_INTEGER 0x75bc0
+0x0b0 TotalCycleTime       : _LARGE_INTEGER 0xcfa3800
...
+0x0d4 TotalProcesses       : 5
+0x0d8 ActiveProcesses      : 2
+0x0dc TotalTerminatedProcesses : 3
...
+0x428 ParentJob            : (null)
+0x430 RootJob              : 0xffffa604'f683b180 _EJOB
...
+0x518 EnergyValues         : 0xffffa604'f683b988 _PROCESS_ENERGY_VALUES
+0x520 SharedCommitCharge   : 0x3e
```

# EPROCESS & KPROCESS

The **`EPROCESS`** structure, which represents a process in the Windows kernel, starts with an embedded **`KPROCESS`** block, also known as the Process Control Block (Pcb). The **`KPROCESS`** part of the **`EPROCESS`** structure contains data for scheduling and time accounting.

This output displays the **`nt!_EPROCESS`** structure for a specific process, showing detailed internal fields like the embedded **`_KPROCESS`** (Pcb), unique process ID (which in this case is the System process), and links to active process lists. It also includes extensive data on process state, flags, resource usage, session and job information, as well as performance and security-related attributes, such as the number of active threads, creation time, and memory usage statistics.

```
2: kd> !mex.ddt nt!_EPROCESS ffffa604f24ec040

dt nt!_EPROCESS ffffa604f24ec040 () Recursive: [ -r1 -r2 -r ] Verbose Normal dt
==================================================================================
   +0x000 Pcb                  : _KPROCESS
   +0x438 ProcessLock          : _EX_PUSH_LOCK
   +0x440 UniqueProcessId      : 0x00000000`00000004 Void  [process] (System)
   +0x448 ActiveProcessLinks   : _LIST_ENTRY [ 0xffffa604`f25df4c8 - 0xfffff805`2ba37ec0 ]
   +0x458 RundownProtect       : _EX_RUNDOWN_REF
   +0x460 Flags2               : 0xd000 (0n53248)
   +0x460 JobNotReallyActive   : 0y0
   +0x460 AccountingFolded     : 0y0
   +0x460 NewProcessReported   : 0y0
   +0x460 ExitProcessReported  : 0y0
   +0x460 ReportCommitChanges  : 0y0
   +0x460 LastReportMemory     : 0y0
   +0x460 ForceWakeCharge      : 0y0
   +0x460 CrossSessionCreate   : 0y0
   +0x460 NeedsHandleRundown   : 0y0
   +0x460 RefTraceEnabled      : 0y0
   +0x460 PicoCreated          : 0y0
   +0x460 EmptyJobEvaluated    : 0y0
   +0x460 DefaultPagePriority  : 0y101 (0n5)
   +0x460 PrimaryTokenFrozen   : 0y1
   +0x460 ProcessVerifierTarget : 0y0
   +0x460 RestrictSetThreadContext : 0y0
   +0x460 AffinityPermanent    : 0y0
   +0x460 AffinityUpdateEnable : 0y0
   +0x460 PropagateNode        : 0y0
   +0x460 ExplicitAffinity     : 0y0
   +0x460 ProcessExecutionState : 0y00 (0n0)
   +0x460 EnableReadVmLogging  : 0y0
   +0x460 EnableWriteVmLogging : 0y0
   +0x460 FatalAccessTerminationRequested : 0y0
   +0x460 DisableSystemAllowedCpuSet : 0y0
   +0x460 ProcessStateChangeRequest : 0y00 (0n0)
   +0x460 ProcessStateChangeInProgress : 0y0
   +0x460 InPrivate            : 0y0
   +0x464 Flags                : 0x14840c00 (0n344198144)
   +0x464 CreateReported       : 0y0
   +0x464 NoDebugInherit       : 0y0
   +0x464 ProcessExiting       : 0y0
   +0x464 ProcessDelete        : 0y0
   +0x464 ManageExecutableMemoryWrites : 0y0
   +0x464 VmDeleted            : 0y0
   +0x464 OutswapEnabled       : 0y0
   +0x464 Outswapped           : 0y0
   +0x464 FailFastOnCommitFail : 0y0
   +0x464 Wow64VaSpace4Gb      : 0y0
   +0x464 AddressSpaceInitialized : 0y11 (0n3)
   +0x464 SetTimerResolution   : 0y0
   +0x464 BreakOnTermination   : 0y0
   +0x464 DeprioritizeViews    : 0y0
   +0x464 WriteWatch           : 0y0
   +0x464 ProcessInSession     : 0y0
   +0x464 OverrideAddressSpace : 0y0
   +0x464 HasAddressSpace      : 0y1
   +0x464 LaunchPrefetched     : 0y0
   +0x464 Reserved             : 0y0
   +0x464 VmTopDown            : 0y0
   +0x464 ImageNotifyDone      : 0y0
   +0x464 PdeUpdateNeeded      : 0y1
   +0x464 VdmAllowed           : 0y0
   +0x464 ProcessRundown       : 0y0
   +0x464 ProcessInserted      : 0y1
   +0x464 DefaultIoPriority    : 0y010 (0n2)
   +0x464 ProcessSelfDelete    : 0y0
   +0x464 SetTimerResolutionLink : 0y0
   +0x468 CreateTime           : _LARGE_INTEGER 0x01d99594`d1ee8f11 (06/02/2023 20:57:19 UTC)
   +0x470 ProcessQuotaUsage    : [2] 0x110
   +0x480 ProcessQuotaPeak     : [2] 0x110
   +0x490 PeakVirtualSize      : 0x3049000 (0n50630656)
   +0x498 VirtualSize          : 0x3fb000 (0n4173824)
   +0x4a0 SessionProcessLinks  : _LIST_ENTRY [ 0x00000000`00000000 - 0x00000000`00000000 ] [INVALID]
   +0x4b0 ExceptionPortData    : (null)
   +0x4b0 ExceptionPortValue   : 0
   +0x4b0 ExceptionPortState   : 0y000 (0n0)
   +0x4b8 Token                : _EX_FAST_REF
   +0x4c0 MmReserved           : 0
   +0x4c8 AddressCreationLock  : _EX_PUSH_LOCK
   +0x4d0 PageTableCommitmentLock : _EX_PUSH_LOCK
   +0x4d8 RotateInProgress     : (null)
   +0x4e0 ForkInProgress       : (null)
   +0x4e8 CommitChargeJob      : (null)
   +0x4f0 CloneRoot            : _RTL_AVL_TREE
   +0x4f8 NumberOfPrivatePages : 0x12 (0n18)
   +0x500 NumberOfLockedPages  : 0
   +0x508 Win32Process         : (null)
   +0x510 Job                  : (null)
   +0x518 SectionObject        : (null)
   +0x520 SectionBaseAddress   : (null)
   +0x528 Cookie               : 0x57d21f0 (0n92086768)
   +0x530 WorkingSetWatch      : (null)
   +0x538 Win32WindowStation   : (null)
   +0x540 InheritedFromUniqueProcessId : (null)
   +0x548 OwnerProcessId       : 2
   +0x550 Peb                  : (null)
   +0x558 Session              : (null)
   +0x560 Spare1               : (null)
   +0x568 QuotaBlock           : 0xfffff805`2ba6ad00 _EPROCESS_QUOTA_BLOCK
   +0x570 ObjectTable          : 0xffff8000`ab28ae80 _HANDLE_TABLE
   +0x578 DebugPort            : (null)
   +0x580 WoW64Process         : (null)
   +0x588 DeviceMap            : _EX_FAST_REF
   +0x590 EtwDataSource        : (null)
   +0x598 PageDirectoryPte     : 0
   +0x5a0 ImageFilePointer     : (null)
   +0x5a8 ImageFileName        : [15] "System"
   +0x5b7 PriorityClass        : 0x2 ''
   +0x5b8 SecurityPort         : (null)
   +0x5c0 SeAuditProcessCreationInfo : _SE_AUDIT_PROCESS_CREATION_INFO
   +0x5c8 JobLinks             : _LIST_ENTRY [ 0x00000000`00000000 - 0x00000000`00000000 ] [INVALID]
   +0x5d8 HighestUserAddress   : 0x00007fff`ffff0000 Void  [generic address]
   +0x5e0 ThreadListHead       : _LIST_ENTRY [ 0xffffa604`f248d578 - 0xffffa604`f76c25b8 ]
   +0x5f0 ActiveThreads        : 0xbf (0n191)
   +0x5f4 ImagePathHash        : 0
   +0x5f8 DefaultHardErrorProcessing : 5
   +0x5fc LastThreadExitStatus : 0n0
   +0x600 PrefetchTrace        : _EX_FAST_REF
   +0x608 LockedPagesList      : (null)
   +0x610 ReadOperationCount   : _LARGE_INTEGER 0x1d (0n29)
   +0x618 WriteOperationCount  : _LARGE_INTEGER 0xd (0n13)
   +0x620 OtherOperationCount  : _LARGE_INTEGER 0xe66 (0n3686)
   +0x628 ReadTransferCount    : _LARGE_INTEGER 0x3eac (0n16044)
   +0x630 WriteTransferCount   : _LARGE_INTEGER 0x34e0c (0n216588)
   +0x638 OtherTransferCount   : _LARGE_INTEGER 0x3721 (0n14113)
   +0x640 CommitChargeLimit    : 0
   +0x648 CommitCharge         : 0xa (0n10)
   +0x650 CommitChargePeak     : 0x20 (0n32)
   +0x680 Vm                   : _MMSUPPORT_FULL
   +0x7c0 MmProcessLinks       : _LIST_ENTRY [ 0xffffa604`f25df840 - 0xfffff805`2bb49700 ]
   +0x7d0 ModifiedPageCount    : 0x5f94 (0n24468)
   +0x7d4 ExitStatus           : 0n259 (0x103) !error
   +0x7d8 VadRoot              : _RTL_AVL_TREE
   +0x7e0 VadHint              : 0xffffa604`f24a9790 Void  PoolTag: VadS
   +0x7e8 VadCount             : 6
   +0x7f0 VadPhysicalPages     : 0
   +0x7f8 VadPhysicalPagesLimit : 0
   +0x800 AlpcContext          : _ALPC_PROCESS_CONTEXT
   +0x820 TimerResolutionLink  : _LIST_ENTRY [ 0x00000000`00000000 - 0x00000000`00000000 ] [INVALID]
   +0x830 TimerResolutionStackRecord : (null)
   +0x838 RequestedTimerResolution : 0
   +0x83c SmallestTimerResolution : 0
   +0x840 ExitTime             : _LARGE_INTEGER 0x0 (01/01/1601 00:00:00 UTC)
   +0x848 InvertedFunctionTable : (null)
   +0x850 InvertedFunctionTableLock : _EX_PUSH_LOCK
   +0x858 ActiveThreadsHighWatermark : 0xc2 (0n194)
   +0x85c LargePrivateVadCount : 0
   +0x860 ThreadListLock       : _EX_PUSH_LOCK
   +0x868 WnfContext           : 0xffff8000`ab2c9060 Void  PoolTag: Wnf
   +0x870 ServerSilo           : (null)
   +0x878 SignatureLevel       : 0x1e ''
   +0x879 SectionSignatureLevel : 0x1c ''
   +0x87a Protection           : _PS_PROTECTION
   +0x87b HangCount            : 0y000 (0n0)
   +0x87b GhostCount           : 0y000 (0n0)
   +0x87b PrefilterException   : 0y0
   +0x87c Flags3               : 0x405080 (0n4214912)
   +0x87c Minimal              : 0y0
   +0x87c ReplacingPageRoot    : 0y0
   +0x87c Crashed              : 0y0
   +0x87c JobVadsAreTracked    : 0y0
   +0x87c VadTrackingDisabled  : 0y0
   +0x87c AuxiliaryProcess     : 0y0
   +0x87c SubsystemProcess     : 0y0
   +0x87c IndirectCpuSets      : 0y1
   +0x87c RelinquishedCommit   : 0y0
   +0x87c HighGraphicsPriority : 0y0
   +0x87c CommitFailLogged     : 0y0
   +0x87c ReserveFailLogged    : 0y0
   +0x87c SystemProcess        : 0y1
   +0x87c HideImageBaseAddresses : 0y0
   +0x87c AddressPolicyFrozen  : 0y1
   +0x87c ProcessFirstResume   : 0y0
   +0x87c ForegroundExternal   : 0y0
   +0x87c ForegroundSystem     : 0y0
   +0x87c HighMemoryPriority   : 0y0
   +0x87c EnableProcessSuspendResumeLogging : 0y0
   +0x87c EnableThreadSuspendResumeLogging : 0y0
   +0x87c SecurityDomainChanged : 0y0
   +0x87c SecurityFreezeComplete : 0y1
   +0x87c VmProcessorHost      : 0y0
   +0x87c VmProcessorHostTransition : 0y0
   +0x87c AltSyscall           : 0y0
   +0x87c TimerResolutionIgnore : 0y0
   +0x87c DisallowUserTerminate : 0y0
   +0x87c EnableProcessRemoteExecProtectVmLogging : 0y0
   +0x87c EnableProcessLocalExecProtectVmLogging : 0y0
   +0x87c MemoryCompressionProcess : 0y0
   +0x880 DeviceAsid           : 0n0
   +0x888 SvmData              : (null)
   +0x890 SvmProcessLock       : _EX_PUSH_LOCK
   +0x898 SvmLock              : 0
   +0x8a0 SvmProcessDeviceListHead : _LIST_ENTRY [ 0xffffa604`f24ec8e0 - 0xffffa604`f24ec8e0 ] [EMPTY OR 1 ELEMENT]
   +0x8b0 LastFreezeInterruptTime : 0
   +0x8b8 DiskCounters         : 0xffffa604`f24ecbc0 _PROCESS_DISK_COUNTERS
   +0x8c0 PicoContext          : (null)
   +0x8c8 EnclaveTable         : (null)
   +0x8d0 EnclaveNumber        : 0
   +0x8d8 EnclaveLock          : _EX_PUSH_LOCK
   +0x8e0 HighPriorityFaultsAllowed : 0
   +0x8e8 EnergyContext        : 0xffffa604`f24ecbe8 _PO_PROCESS_ENERGY_CONTEXT
   +0x8f0 VmContext            : (null)
   +0x8f8 SequenceNumber       : 1
   +0x900 CreateInterruptTime  : 0x9c9973 (0n10262899)
   +0x908 CreateUnbiasedInterruptTime : 0x9c9973 (0n10262899)
   +0x910 TotalUnbiasedFrozenTime : 0
   +0x918 LastAppStateUpdateTime : 0
   +0x920 LastAppStateUptime   : 0y0000000000000000000000000000000000000000000000000000000000000 (0)
   +0x920 LastAppState         : 0y000 (0n0)
   +0x928 SharedCommitCharge   : 0x4a (0n74)
   +0x930 SharedCommitLock     : _EX_PUSH_LOCK
   +0x938 SharedCommitLinks    : _LIST_ENTRY [ 0xffff8000`abb1dd58 - 0xffff8000`adbfa5c8 ]
   +0x948 AllowedCpuSets       : 0xffffa604`f24ecdc8 (0n-98934801379896)
   +0x950 DefaultCpuSets       : 0xffffa604`f24ecec8 (0n-98934801379640)
   +0x948 AllowedCpuSetsIndirect : 0xffffa604`f24ecdc8  -> 0xf
   +0x950 DefaultCpuSetsIndirect : 0xffffa604`f24ecec8  -> 0
   +0x958 DiskIoAttribution    : (null)
   +0x960 DxgProcess           : 0xffff8000`ab92c4e0 Void  [process] (¥bÂ«‚Ùþ,QÐƒÙ *32)
   +0x968 Win32KFilterSet      : 0
   +0x96c Machine              : 0x8664 (0n34404)
   +0x96e Spare0               : 0
   +0x970 ProcessTimerDelay    : _PS_INTERLOCKED_TIMER_DELAY_VALUES
   +0x978 KTimerSets           : 0
   +0x97c KTimer2Sets          : 0
   +0x980 ThreadTimerSets      : 0
   +0x988 VirtualTimerListLock : 0
   +0x990 VirtualTimerListHead : _LIST_ENTRY [ 0xffffa604`f24ec9d0 - 0xffffa604`f24ec9d0 ] [EMPTY OR 1 ELEMENT]
   +0x9a0 WakeChannel          : _WNF_STATE_NAME
   +0x9a0 WakeInfo             : _PS_PROCESS_WAKE_INFORMATION
   +0x9d0 MitigationFlags      : 0x40000060 (0n1073741920)
   +0x9d0 MitigationFlagsValues : <unnamed-tag>
   +0x9d4 MitigationFlags2     : 0x40002000 (0n1073750016)
   +0x9d4 MitigationFlags2Values : <unnamed-tag>
   +0x9d8 PartitionObject      : 0xffffa604`f24aaeb0 Void  PoolTag: Part
   +0x9e0 SecurityDomain       : 0
   +0x9e8 ParentSecurityDomain : 0
   +0x9f0 CoverageSamplerContext : (null)
   +0x9f8 MmHotPatchContext    : (null)
   +0xa00 IdealProcessorAssignmentBlock : _KE_IDEAL_PROCESSOR_ASSIGNMENT_BLOCK
   +0xb18 DynamicEHContinuationTargetsTree : _RTL_AVL_TREE
   +0xb20 DynamicEHContinuationTargetsLock : _EX_PUSH_LOCK
   +0xb28 DynamicEnforcedCetCompatibleRanges : _PS_DYNAMIC_ENFORCED_ADDRESS_RANGES
   +0xb38 DisabledComponentFlags : 0
   +0xb3c PageCombineSequence  : 0n1
   +0xb40 EnableOptionalXStateFeaturesLock : _EX_PUSH_LOCK
   +0xb48 PathRedirectionHashes : (null)
   +0xb50 SyscallProvider      : (null)
   +0xb58 SyscallProviderProcessLinks : _LIST_ENTRY [ 0x00000000`00000000 - 0x00000000`00000000 ] [INVALID]
   +0xb68 SyscallProviderDispatchContext : _PSP_SYSCALL_PROVIDER_DISPATCH_CONTEXT
   +0xb70 MitigationFlags3     : 0
   +0xb70 MitigationFlags3Values : <unnamed-tag>
```

On the other hand, this output details the **`_KPROCESS`** structure for a process, revealing internal fields such as scheduling information **`ActiveProcessors`**, **`BasePriority`**, **`QuantumReset`**, process performance data **`CycleTime`**, **`ContextSwitches`**, and various flags and counters related to process state and execution, such as **`KernelTime`**, **`UserTime`**

```
2: kd> !mex.ddt ffffa604f24ec040+0x000 nt!_KPROCESS

dt ffffa604f24ec040+0x000 nt!_KPROCESS () Recursive: [ -r1 -r2 -r ] Verbose Normal dt
==================================================================================
   +0x000 Header               : _DISPATCHER_HEADER
   +0x018 ProfileListHead      : _LIST_ENTRY [ 0xffffa604`f24ec058 - 0xffffa604`f24ec058 ] [EMPTY OR 1 ELEMENT]
   +0x028 DirectoryTableBase   : 0x7d5000 (0n8212480)
   +0x030 ThreadListHead       : _LIST_ENTRY [ 0xffffa604`f248d338 - 0xffffa604`f76c2378 ]
   +0x040 ProcessLock          : 0
   +0x044 ProcessTimerDelay    : 0
   +0x048 DeepFreezeStartTime  : 0
   +0x050 Affinity             : _KAFFINITY_EX
   +0x158 ReadyListHead        : _LIST_ENTRY [ 0xffffa604`f24ec198 - 0xffffa604`f24ec198 ] [EMPTY OR 1 ELEMENT]
   +0x168 SwapListEntry        : _SINGLE_LIST_ENTRY
   +0x170 ActiveProcessors     : _KAFFINITY_EX
   +0x278 AutoAlignment        : 0y1
   +0x278 DisableBoost         : 0y0
   +0x278 DisableQuantum       : 0y0
   +0x278 DeepFreeze           : 0y0
   +0x278 TimerVirtualization  : 0y0
   +0x278 CheckStackExtents    : 0y0
   +0x278 CacheIsolationEnabled : 0y0
   +0x278 PpmPolicy            : 0y0000 (0n0)
   +0x278 VaSpaceDeleted       : 0y0
   +0x278 MultiGroup           : 0y0
   +0x278 ReservedFlags        : 0y0000000000000000000 (0)
   +0x278 ProcessFlags         : 0n1
   +0x27c ActiveGroupsMask     : 1
   +0x280 BasePriority         : 8 ''
   +0x281 QuantumReset         : 6 ''
   +0x282 Visited              : 0 ''
   +0x283 Flags                : _KEXECUTE_OPTIONS
   +0x284 ThreadSeed           : [32] 3
   +0x2c4 IdealProcessor       : [32] 0
   +0x304 IdealNode            : [32] 0
   +0x344 IdealGlobalNode      : 0
   +0x346 Spare1               : 0
   +0x348 StackCount           : _KSTACK_COUNT
   +0x350 ProcessListEntry     : _LIST_ENTRY [ 0xffffa604`f250c3d0 - 0xfffff805`2ba421c0 ]
   +0x360 CycleTime            : 0x00000002`c1f27ecf (01/01/1601 00:19:44 UTC)
   +0x368 ContextSwitches      : 0x9888 (0n39048)
   +0x370 SchedulingGroup      : (null)
   +0x378 FreezeCount          : 0
   +0x37c KernelTime           : 0xf9 (0n249)
   +0x380 UserTime             : 0
   +0x384 ReadyTime            : 0x1a (0n26)
   +0x388 UserDirectoryTableBase : 0
   +0x390 AddressPolicy        : 0x1 ''
   +0x391 Spare2               : [71] ""
   +0x3d8 InstrumentationCallback : (null)
   +0x3e0 SecureState          : <unnamed-tag>
   +0x3e8 KernelWaitTime       : 0x661f (0n26143)
   +0x3f0 UserWaitTime         : 0
   +0x3f8 LastRebalanceQpc     : 0
   +0x400 PerProcessorCycleTimes : (null)
   +0x408 ExtendedFeatureDisableMask : 0
   +0x410 PrimaryGroup         : 0
   +0x412 Spare3               : [3] 0
   +0x418 UserCetLogging       : (null)
   +0x420 CpuPartitionList     : _LIST_ENTRY [ 0xffffa604`f24ec460 - 0xffffa604`f24ec460 ] [EMPTY OR 1 ELEMENT]
   +0x430 EndPadding           : [1] 0
```

# Protected Process Light (PPL)

Protected Process Light (PPL) is a security feature in Windows designed to provide an enhanced level of protection for critical system processes. PPL is typically used for system services and other critical processes that require a high level of integrity and protection from tampering. This includes security services like antivirus engines or Windows security services.

Processes marked as PPL have strict limitations on who can interact with them. This includes restrictions on code injection, reading or writing memory, and other forms of interference.

The screenshot displays a list of processes running on a Windows system, with details such as memory usage and process IDs. It indicates the protection status of each process, with labels like **`PsProtectedSignerWinTcb-Light`** and **`PsProtectedSignerAntimalware-Light`**, showing that certain processes are running under the Protected Process Light (PPL) security feature, which restricts access to these processes to prevent tampering by unauthorized code.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/7b3481e3-cb91-47ec-a2da-b88a28f2b10f)


# Thread Priorities

Thread priorities is a component of the operating system's scheduling mechanism, which dictates the sequence in which threads are allocated CPU time for execution. Each thread is assigned a specific priority value, within a range from 0 (the lowest priority) to 31 (the highest priority). 

In Windows, the scheduler leverages these priority values to determine the next thread to execute. Threads with higher priority values are typically selected before those with lower values, provided they are in a ready state to run. 

The **`_KTHREAD`** structure is a kernel structure that represents a thread within the system. The **`Priority`** field within this **`_KTHREAD`** structure specifies the current scheduling priority of the thread.

Here are some examples of threads from the **System.exe** process with the associated priority:

```
0: kd> !mex.ddt nt!_KTHREAD ffff970c8cf13080 -y Priority

dt nt!_KTHREAD ffff970c8cf13080 -y Priority () Recursive: [ -r1 -r2 -r ] Verbose Normal dt
==================================================================================
   +0x0c3 Priority             : 15 ''
   +0x234 PriorityDecrement    : 32 ' '
   +0x338 PriorityFloorCounts  : [16] ""
   +0x348 PriorityFloorCountsReserved : [16] ""
   +0x358 PriorityFloorSummary : 0

0: kd> !mex.ddt nt!_KTHREAD ffff970c8cf9b080 -y Priority

dt nt!_KTHREAD ffff970c8cf9b080 -y Priority () Recursive: [ -r1 -r2 -r ] Verbose Normal dt
==================================================================================
   +0x0c3 Priority             : 16 ''
   +0x234 PriorityDecrement    : 0 ''
   +0x338 PriorityFloorCounts  : [16] ""
   +0x348 PriorityFloorCountsReserved : [16] ""
   +0x358 PriorityFloorSummary : 0

0: kd> !mex.ddt nt!_KTHREAD ffff970c91fcf040 -y Priority

dt nt!_KTHREAD ffff970c91fcf040 -y Priority () Recursive: [ -r1 -r2 -r ] Verbose Normal dt
==================================================================================
   +0x0c3 Priority             : 10 ''
   +0x234 PriorityDecrement    : 0 ''
   +0x338 PriorityFloorCounts  : [16] ""
   +0x348 PriorityFloorCountsReserved : [16] ""
   +0x358 PriorityFloorSummary : 0

0: kd> !mex.ddt nt!_KTHREAD ffff970c93d2e080 -y Priority

dt nt!_KTHREAD ffff970c93d2e080 -y Priority () Recursive: [ -r1 -r2 -r ] Verbose Normal dt
==================================================================================
   +0x0c3 Priority             : 8 ''
   +0x234 PriorityDecrement    : 0 ''
   +0x338 PriorityFloorCounts  : [16] ""
   +0x348 PriorityFloorCountsReserved : [16] ""
   +0x358 PriorityFloorSummary : 0

0: kd> !mex.ddt nt!_KTHREAD ffff970c8def8040 -y Priority

dt nt!_KTHREAD ffff970c8def8040 -y Priority () Recursive: [ -r1 -r2 -r ] Verbose Normal dt
==================================================================================
   +0x0c3 Priority             : 9 ''
   +0x234 PriorityDecrement    : 0 ''
   +0x338 PriorityFloorCounts  : [16] ""
   +0x348 PriorityFloorCountsReserved : [16] ""
   +0x358 PriorityFloorSummary : 0

0: kd> !mex.ddt nt!_KTHREAD ffff970c939e3040 -y Priority

dt nt!_KTHREAD ffff970c939e3040 -y Priority () Recursive: [ -r1 -r2 -r ] Verbose Normal dt
==================================================================================
   +0x0c3 Priority             : 9 ''
   +0x234 PriorityDecrement    : 0 ''
   +0x338 PriorityFloorCounts  : [16] ""
   +0x348 PriorityFloorCountsReserved : [16] ""
   +0x358 PriorityFloorSummary : 0

0: kd> !mex.ddt nt!_KTHREAD ffff970c9179e0c0 -y Priority

dt nt!_KTHREAD ffff970c9179e0c0 -y Priority () Recursive: [ -r1 -r2 -r ] Verbose Normal dt
==================================================================================
   +0x0c3 Priority             : 15 ''
   +0x234 PriorityDecrement    : 0 ''
   +0x338 PriorityFloorCounts  : [16] ""
   +0x348 PriorityFloorCountsReserved : [16] ""
   +0x358 PriorityFloorSummary : 0

0: kd> !mex.ddt nt!_KTHREAD ffff970c95cb7040 -y Priority

dt nt!_KTHREAD ffff970c95cb7040 -y Priority () Recursive: [ -r1 -r2 -r ] Verbose Normal dt
==================================================================================
   +0x0c3 Priority             : 13 ''
   +0x234 PriorityDecrement    : 0 ''
   +0x338 PriorityFloorCounts  : [16] ""
   +0x348 PriorityFloorCountsReserved : [16] ""
   +0x358 PriorityFloorSummary : 0

0: kd> !mex.ddt nt!_KTHREAD ffff970c94c81040 -y Priority

dt nt!_KTHREAD ffff970c94c81040 -y Priority () Recursive: [ -r1 -r2 -r ] Verbose Normal dt
==================================================================================
   +0x0c3 Priority             : 20 ''
   +0x234 PriorityDecrement    : 0 ''
   +0x338 PriorityFloorCounts  : [16] ""
   +0x348 PriorityFloorCountsReserved : [16] ""
   +0x358 PriorityFloorSummary : 0

0: kd> !mex.ddt nt!_KTHREAD ffff970c91775040 -y Priority

dt nt!_KTHREAD ffff970c91775040 -y Priority () Recursive: [ -r1 -r2 -r ] Verbose Normal dt
==================================================================================
   +0x0c3 Priority             : 12 ''
   +0x234 PriorityDecrement    : 0 ''
   +0x338 PriorityFloorCounts  : [16] ""
   +0x348 PriorityFloorCountsReserved : [16] ""
   +0x358 PriorityFloorSummary : 0
```

# Thread State

A thread state refers to the current condition of a thread in a process, indicating its activity or readiness level. It can be one of several values, such as **Ready**, **Running**, **Waiting**, **Standby**, **Transition**, or **Terminated**, each reflecting a different stage in the thread's lifecycle.


| Thread State | Description |
|--------------|-------------|
| Ready | The thread is prepared to run and is waiting for an available processor. The system's scheduler determines which ready threads get to use the processor based on their priority levels. |
| Standby | The thread is selected to run on a processor the next time the processor becomes available. This is the thread that the system will run next on that processor. |
| Running | The thread is currently executing instructions on a processor. |
| Waiting | The thread is not running, and is waiting for some resource to become available or for some event to occur. For example, a thread might be waiting for disk I/O to complete, for data from a network connection, or for a mutex to be released by another thread. This is the most common one that you will see. |
| Transition | The thread is waiting for resources to become available so that it can transition to the "Ready" state and then to the "Running" state. For example, it might be waiting for a page to be read from disk into memory. |
| Terminated | The thread has finished executing and has exited. A thread in this state is effectively dead and cannot be scheduled to run again. |
| Initialized | A state that indicates the thread has been initialized, but has not yet started. |

The difference between **Standby** and **Ready** is the following:

A thread in the **Standby** state has been selected by the scheduler to run next on a specific CPU, while a thread in the **Ready** state is prepared to run but has not yet been chosen by the scheduler. These threads are in a queue, waiting for the scheduler to select them for execution.

# Thread Wait Reasons

A thread wait reason is a status that indicates why a thread is not currently executing:

| Wait Reason        | Description |
| ------------------ | ----------- |
| DelayExecution     | The thread is in a "sleep" mode, possibly because of a call to `Sleep`, `SleepEx`, or `KeDelayExecutionThread` in the kernel. |
| Executive          | This is a generic placeholder. It means some kernel component (like a driver) asked the thread to wait. |
| WrQueue            | The thread is waiting for some incoming work, usually from an I/O operation. |
| Suspended          | The thread is on hold. It might be because it decided to suspend itself, or another thread told it to. |
| UserRequest        | This is the wait reason you'll see the most. It's often because the program asked the thread to wait, but it could also be because a kernel component (like a driver) made the request. |
| WrAlertByThreadId  | This happens when a thread is waiting for things like Critical Sections, Conditional Variables, or Slim Reader/Writer locks. |
| WrLpcReceive       | The thread is waiting for an incoming Local Procedure Call (LPC) message. |
| WrLpcReply         | The thread is waiting for a reply to a local procedure call to arrive. |
| WrPreempted        | This refers to a thread that was paused on a CPU because a more important thread needed to use that CPU. |
| WrFreePage         | The thread is waiting for a free virtual memory page. |
| WrVirtualMemory    | The thread is waiting for the system to allocate virtual memory. |
| WrResource         | The thread is waiting on an ERESOURCE. |
| WrPushlock         | The thread is waiting for a push lock. |
| WrFastMutex        | The thread is waiting on a fast mutex. |


# Priority Boost Scenarios

A Priority Boost in Windows operating systems is a temporary elevation of a thread's priority by the system scheduler to improve performance and responsiveness. Here are some scenarios:

- **Scheduler/Dispatcher Boosts:** Increases priority to reduce latency in thread scheduling.
- **I/O Completion Boosts:** Elevates priority post-I/O tasks to decrease latency.
- **UI Input Boosts:** Prioritizes UI threads for better responsiveness and lower latency.
- **Extended ERESOURCE Waiting Boosts:** Boosts priority to prevent resource waiting starvation.
- **Dormant Thread Boosts:** Increases priority of inactive threads to avoid starvation and priority inversion.

**Note:** Windows doesn't boost priorities for real-time range threads (16-31) to ensure predictable scheduling.

# Context Switch

A context switch is the process of the CPU switching from one thread or process to another, requiring the saving and reloading of the current state to ensure each process or thread resumes exactly where it left off. The following data is needed:

- **Instruction Pointer:** Saved and reloaded during a context switch, indicating the next instruction.
- **Kernel Stack Pointer:** Saved, points to the thread's kernel mode stack, crucial for execution.
- **Address Space Pointer:** Noted for managing the memory space of the thread's process.
- **Saving Method:** Detailed as pushing onto the old thread's kernel stack, then updating and saving this in the KTHREAD structure.
- **Loading New Thread Context:** Involves setting the kernel stack pointer to the new thread's stack and loading its context.
- **Handling Different Processes:** For a new thread in a different process, the system loads its page table directory into a register for address 

Frequent context switches can cause significant CPU overhead, reducing overall system performance. More CPU resources are spent on the process of switching contexts, rather than on executing actual process or thread workloads.

# How does a thread get CPU time?

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/d1459a38-8483-4162-a11b-4fc6869be953)

A thread gets CPU time through the coordination of the CPU scheduler and the dispatcher.

- **CPU Scheduler:** This component is responsible for determining which threads should be executed by the CPU. It prioritizes threads based on a variety of factors including their priority level, whether they are I/O bound or CPU bound, and other scheduling policies.

Threads are typically assigned priorities. The CPU Scheduler will look at these threads and decide the order in which they should be served based on their priority and other rules. Higher priority threads are typically given CPU time before lower priority ones.

- **Dispatcher:** Once the CPU Scheduler has decided which thread should be executed next, the dispatcher takes over. It switches the context of the CPU to that of the chosen thread. Context switching involves saving the state of the current thread and loading the state of the thread that is about to be executed. This includes updating various registers with the thread's information, such as the instruction pointer and stack pointer, which tell the CPU where to resume execution in the thread's code.

If a thread with a higher priority becomes ready to run or if the running thread has exhausted its time quantum, the CPU Scheduler may decide to preempt the current thread. The dispatcher will perform a context switch, saving the state of the current thread and loading the state of the new thread to be run.

Preempting the current thread means that the operating system's scheduler forcibly takes away the CPU from a running thread before it has finished its assigned time. This is done to allow another thread, usually one with a higher priority or one that has become runnable after waiting for a resource, to execute.

# Parameter of Process Startup

How to get the command-line of a user-mode dump of a running process?

```
1. Dump the PEB

0:000> !peb
PEB at 00000079616db000
    InheritedAddressSpace:    No
    ReadImageFileExecOptions: No
    BeingDebugged:            No
    ImageBaseAddress:         00007ff7f0d40000
    NtGlobalFlag:             0
    NtGlobalFlag2:            0
    Ldr                       00007ffb1be1c4c0
    Ldr.Initialized:          Yes
    Ldr.InInitializationOrderModuleList: 000001be88612aa0 . 000001be88626b00
    Ldr.InLoadOrderModuleList:           000001be88612c10 . 000001be88626ae0
    Ldr.InMemoryOrderModuleList:         000001be88612c20 . 000001be88626af0
                    Base TimeStamp                     Module
            7ff7f0d40000 d7ee190d Oct 18 04:03:41 2084 C:\Windows\system32\cmd.exe
            7ffb1bcb0000 9b64aa6f Aug 12 01:55:11 2052 C:\Windows\SYSTEM32\ntdll.dll
            7ffb1bb10000 9ec9da27 Jun 02 08:58:31 2054 C:\Windows\System32\KERNEL32.DLL
            7ffb196b0000 d80f8f12 Nov 12 12:12:02 2084 C:\Windows\System32\KERNELBASE.dll
            7ffb1bbd0000 9bf60e04 Nov 30 07:38:44 2052 C:\Windows\System32\msvcrt.dll
            7ffb19f80000 613e7d3d Sep 12 15:20:45 2021 C:\Windows\System32\combase.dll
            7ffb195b0000 81cf5d89 Jan 05 06:32:41 2039 C:\Windows\System32\ucrtbase.dll
            7ffb1a640000 a250f40f Apr 17 09:25:51 2056 C:\Windows\System32\RPCRT4.dll
            7ffb09fd0000 469f83fc Jul 19 08:32:12 2007 C:\Windows\SYSTEM32\winbrand.dll
            7ffb1a3c0000 9dfab69a Dec 27 05:07:38 2053 C:\Windows\System32\sechost.dll
    SubSystemData:     0000000000000000
    ProcessHeap:       000001be88610000
    ProcessParameters: 000001be886121c0
    CurrentDirectory:  'C:\Users\Admin\'
    WindowTitle:  'C:\Users\Admin\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\System Tools\Command Prompt.lnk'
    ImageFile:    'C:\Windows\system32\cmd.exe'
    CommandLine:  '"C:\Windows\system32\cmd.exe" '
    DllPath:      '< Name not readable >'
    Environment:  000001be8863d700
        =::=::\
        =C:=C:\Users\Admin
        =ExitCode=00000000
        ALLUSERSPROFILE=C:\ProgramData
        APPDATA=C:\Users\Admin\AppData\Roaming
        CLIENTNAME=LAPTOP-4AMAH4I8
        CommonProgramFiles=C:\Program Files\Common Files
        CommonProgramFiles(x86)=C:\Program Files (x86)\Common Files
        CommonProgramW6432=C:\Program Files\Common Files
        COMPUTERNAME=DESKTOP-OQGRG4S
        ComSpec=C:\Windows\system32\cmd.exe
        DriverData=C:\Windows\System32\Drivers\DriverData
        FPS_BROWSER_APP_PROFILE_STRING=Internet Explorer
        FPS_BROWSER_USER_PROFILE_STRING=Default
        HOMEDRIVE=C:
        HOMEPATH=\Users\Admin
        LOCALAPPDATA=C:\Users\Admin\AppData\Local
        LOGONSERVER=\\DESKTOP-OQGRG4S
        NUMBER_OF_PROCESSORS=4
        OneDrive=C:\Users\Admin\OneDrive
        OS=Windows_NT
        Path=C:\Windows\system32;C:\Windows;C:\Windows\System32\Wbem;C:\Windows\System32\WindowsPowerShell\v1.0\;C:\Windows\System32\OpenSSH\;C:\Program Files\Microsoft SQL Server\150\Tools\Binn\;C:\Program Files\Microsoft SQL Server\Client SDK\ODBC\170\Tools\Binn\;C:\Program Files\dotnet\;C:\Program Files (x86)\Windows Kits\10\Windows Performance Toolkit\;C:\Program Files\010 Editor;C:\Program Files\Git\cmd;C:\Users\Admin\AppData\Local\Programs\Python\Python312\Scripts\;C:\Users\Admin\AppData\Local\Programs\Python\Python312\;C:\Users\Admin\AppData\Local\Programs\Python\Launcher\;C:\Users\Admin\AppData\Local\Microsoft\WindowsApps;C:\Users\Admin\.dotnet\tools;C:\Program Files (x86)\Windows Kits\10\Debuggers\x64;
        PATHEXT=.COM;.EXE;.BAT;.CMD;.VBS;.VBE;.JS;.JSE;.WSF;.WSH;.MSC
        PROCESSOR_ARCHITECTURE=AMD64
        PROCESSOR_IDENTIFIER=Intel64 Family 6 Model 140 Stepping 1, GenuineIntel
        PROCESSOR_LEVEL=6
        PROCESSOR_REVISION=8c01
        ProgramData=C:\ProgramData
        ProgramFiles=C:\Program Files
        ProgramFiles(x86)=C:\Program Files (x86)
        ProgramW6432=C:\Program Files
        PROMPT=$P$G
        PSModulePath=C:\Program Files\WindowsPowerShell\Modules;C:\Windows\system32\WindowsPowerShell\v1.0\Modules
        PUBLIC=C:\Users\Public
        SESSIONNAME=31C5CE94259D4006A9E4#0
        SystemDrive=C:
        SystemRoot=C:\Windows
        TEMP=C:\Users\Admin\AppData\Local\Temp
        TMP=C:\Users\Admin\AppData\Local\Temp
        USERDOMAIN=DESKTOP-OQGRG4S
        USERDOMAIN_ROAMINGPROFILE=DESKTOP-OQGRG4S
        USERNAME=Admin
        USERPROFILE=C:\Users\Admin
        windir=C:\Windows
        _NT_SOURCE_PATH=SRV*C:\Symbols*https://msdl.microsoft.com/download/symbols
        _NT_SYMBOL_PATH=SRV*C:\Symbols*https://msdl.microsoft.com/download/symbols
        _NT_SYMCACHE_PATH=C:\SymCache
```

The output was used to display the **`_RTL_USER_PROCESS_PARAMETERS`** structure, which is pointed to by the **`ProcessParameters`** field in the **`ntdll!_PEB`** structure.

```
0:000> !mex.ddt nt!_PEB 00000079616db000 -y ProcessParameters

dt nt!_PEB 00000079616db000 -y ProcessParameters () Recursive: [ -r1 -r2 -r ] Verbose Normal dt
==================================================================================
ntdll!_PEB
   +0x020 ProcessParameters    : 0x000001be`886121c0 _RTL_USER_PROCESS_PARAMETERS
```

From here, we can now identify the content of the **`_RTL_USER_PROCESS_PARAMETERS`** structure, with the **`CommandLine`** field at offset +0x070 displaying the command line used to start **cmd.exe**

```
0:000> !mex.ddt 0x000001be`886121c0 ntdll!_RTL_USER_PROCESS_PARAMETERS

dt 0x000001be`886121c0 ntdll!_RTL_USER_PROCESS_PARAMETERS () Recursive: [ -r1 -r2 -r ] Verbose Normal dt
==================================================================================
   +0x000 MaximumLength        : 0x7aa (0n1962)
   +0x004 Length               : 0x7aa (0n1962)
   +0x008 Flags                : 0x6001 (0n24577)
   +0x00c DebugFlags           : 0
   +0x010 ConsoleHandle        : 0x00000000`00000044 Void  [handle]
   +0x018 ConsoleFlags         : 0
   +0x020 StandardInput        : 0x00000000`00000050 Void    [ !ndao dps dc !handle ln ? ]
   +0x028 StandardOutput       : 0x00000000`00000054 Void    [ !ndao dps dc !handle ln ? ]
   +0x030 StandardError        : 0x00000000`00000058 Void    [ !ndao dps dc !handle ln ? ]
   +0x038 CurrentDirectory     : _CURDIR
   +0x050 DllPath              : _UNICODE_STRING  ""
   +0x060 ImagePathName        : _UNICODE_STRING  "C:\Windows\system32\cmd.exe"
   +0x070 CommandLine          : _UNICODE_STRING  ""C:\Windows\system32\cmd.exe" "
   +0x080 Environment          : 0x000001be`8863d700 Void    [ !ndao dps dc !handle ln ? ]
   +0x088 StartingX            : 0
   +0x08c StartingY            : 0
   +0x090 CountX               : 0
   +0x094 CountY               : 0
   +0x098 CountCharsX          : 0
   +0x09c CountCharsY          : 0
   +0x0a0 FillAttribute        : 0
   +0x0a4 WindowFlags          : 0x801 (0n2049)
   +0x0a8 ShowWindowFlags      : 1
   +0x0b0 WindowTitle          : _UNICODE_STRING  "C:\Users\Admin\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\System Tools\Command Prompt.lnk"
   +0x0c0 DesktopInfo          : _UNICODE_STRING  "Winsta0\Default"
   +0x0d0 ShellInfo            : _UNICODE_STRING  ""
   +0x0e0 RuntimeData          : _UNICODE_STRING  ""
   +0x0f0 CurrentDirectores    : [32] _RTL_DRIVE_LETTER_CURDIR
   +0x3f0 EnvironmentSize      : 0x1234 (0n4660)
   +0x3f8 EnvironmentVersion   : 0xb (0n11)
   +0x400 PackageDependencyData : (null)
   +0x408 ProcessGroupId       : 0x15b8 (0n5560)
   +0x40c LoaderThreads        : 0
   +0x410 RedirectionDllName   : _UNICODE_STRING  ""
   +0x420 HeapPartitionName    : _UNICODE_STRING  ""
   +0x430 DefaultThreadpoolCpuSetMasks : (null)
   +0x438 DefaultThreadpoolCpuSetMaskCount : 0
   +0x43c DefaultThreadpoolThreadMaximum : 0
```

# CPU Affinity

CPU affinity dictates on which CPU cores the thread is allowed to be scheduled. There are Windows APIs that can be used to achieve this:

- **SetThreadAffinityMask:** This function is used to set the affinity mask for an individual thread. It specifies the CPU cores that the thread is allowed to run on.
- **SetProcessAffinityMask:** This function sets the affinity mask for all threads within a specific process. This way, we can control on which CPU cores the entire process is allowed to execute.
- **Process Explorer:**  Locate the process for which we want to set the affinity. Choose 'Set Affinity' from the context menu. A window will open where we can select the CPU cores that the process is allowed to run on.

When compiling a program, we can also specify an affinity mask in the image header. This sets the default affinity for the process when it starts.

In this example, the process **LogonUI.exe** is configured to exclusively run on **CPU 3**.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/beb2f646-2ad8-4de7-bd02-55ddb11536be)

Observe that the ideal CPU for all threads has now been updated to 3 too.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/7332eb59-6f70-46be-a979-b5c74a67fe74)

# Reference

- https://youtu.be/6IXx7xx8t2Y?si=snp7fSDIh4jW7xdS

