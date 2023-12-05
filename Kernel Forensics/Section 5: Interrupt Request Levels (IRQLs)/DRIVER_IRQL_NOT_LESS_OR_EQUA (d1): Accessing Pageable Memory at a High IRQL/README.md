# Description

**`DRIVER_IRQL_NOT_LESS_OR_EQUAL`** is a bugcheck with a stop code of **`0xD1`**. This type of bugcheck is typically caused by a driver attempting to access an invalid or pageable memory address at an IRQL that is too high.

Link to this CrashDump: https://mega.nz/file/Ls0ylBAI#R2IUcn9vWtvWFWkd3KFwWTXHiG7ggp3KoUslAPGAvIk

# Code Sample - Accessing Pageable Memory at High IRQL

The root cause of the crash in this driver lies in the attempt to access pageable memory at an inappropriate IRQL (Interrupt Request Level). Accessing pageable memory at **`DISPATCH_LEVEL`** or higher IRQLs is not allowed because at these high interrupt levels, the system cannot deal with the delay caused if the memory it needs has been moved to disk and needs to be retrieved. This retrieval process, called handling a page fault, requires time and resources that are not available at high IRQLs.

```c
#include <ntifs.h>
#include <ntstrsafe.h>

// Forward declaration of the function that will be placed in a pageable section.
void AccessPageableMemory();

// Place the AccessPageableMemory function in a pageable section.
#pragma alloc_text(PAGE, AccessPageableMemory)

// Definition of AccessPageableMemory, a function declared as pageable.
void AccessPageableMemory() {
    // Accessing pageable memory here (This is for demonstration only).
    // In practice, this operation is illegal at DISPATCH_LEVEL and above.
    DbgPrint("Accessing pageable memory...\n");
}

// Thread function that accesses pageable memory at DISPATCH_LEVEL.
NTSTATUS PageableMemoryAccessThread(PVOID Context) {
    UNREFERENCED_PARAMETER(Context);

    KIRQL oldIrql;

    // Raise the IRQL to DISPATCH_LEVEL.
    KeRaiseIrql(DISPATCH_LEVEL, &oldIrql);

    // Call the pageable function at DISPATCH_LEVEL (Illegal operation).
    AccessPageableMemory(); // This line is expected to cause a system crash.

    KeLowerIrql(oldIrql); // Lower the IRQL to its previous value.

    return STATUS_SUCCESS;
}

// Driver entry point.
NTSTATUS DriverEntry(PDRIVER_OBJECT DriverObject, PUNICODE_STRING RegistryPath) {
    UNREFERENCED_PARAMETER(RegistryPath);

    HANDLE threadHandle;
    NTSTATUS status = PsCreateSystemThread(&threadHandle, (ACCESS_MASK)0, NULL, (PHANDLE)0, NULL, PageableMemoryAccessThread, NULL);

    if (NT_SUCCESS(status)) {
        ZwClose(threadHandle); // Close the thread handle.
    }

    DriverObject->DriverUnload = NULL; // Unload routine not necessary for this example.

    return status;
}
```

Start loading this kernel driver:

```
sc.exe create TestDrv type= kernel binPath=C:\Users\User\source\repos\TestDriver\x64\Release\TestDriver.sys
```

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/6c404a29-f660-4d9c-9b91-8b90dd5d4c34)


Start the kernel driver:

```
sc.exe start TestDrv
```

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/d45974d7-4bc9-48fa-ade3-266b5289af53)


Here we can see that our machine has bugchecked:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/9a947f1f-0916-48e7-acd0-9edc5b85a0b0)

# WinDbg Walk Through - Analysis

Load the Memory Dump in WinDbg:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/9f694bc6-2b75-4ff5-a0ef-4f3b5988288a)


Let's start with the **`!mex.di`** command which stands for dump information. This command retrieves information about the machine from which the memory dump originated. It's especially handy when working with multiple memory dumps, as it helps distinguish the analysis related to each specific dump. This is an improved version of the **`vertarget`** command.

```
1: kd> !mex.di
Computer Name:  Not Found
Windows 10 Kernel Version 22621 MP (6 procs) Free x64
Product: WinNt, suite: TerminalServer SingleUserTS
Edition build lab: 22621.2506.amd64fre.ni_release_svc_prod3.231018-1809
Kernel base = 0xfffff803`44600000 PsLoadedModuleList = 0xfffff803`452134a0
Debug session time: Sat Dec  2 11:51:39.664 2023 (UTC - 8:00)
System Uptime: 0 days 0:25:05.774
SystemManufacturer = Microsoft Corporation
SystemProductName = Virtual Machine
Processor: 11th Gen Intel(R) Core(TM) i7-1185G7 @ 3.00GHz
Bugcheck: D1 (FFFFF803712E5000, 2, 8, FFFFF803712E5000)
```

Let's run the **`!analyze -v`** command to quickly triage the issue as it provides a quick and detailed overview of the state of the system at the time of the crash. It automates many aspects of crash dump analysis, providing a quick way to get valuable information.

```
1: kd> !analyze -v
*******************************************************************************
*                                                                             *
*                        Bugcheck Analysis                                    *
*                                                                             *
*******************************************************************************

DRIVER_IRQL_NOT_LESS_OR_EQUAL (d1)
An attempt was made to access a pageable (or completely invalid) address at an
interrupt request level (IRQL) that is too high.  This is usually
caused by drivers using improper addresses.
If kernel debugger is available get stack backtrace.
Arguments:
Arg1: fffff803712e5000, memory referenced
Arg2: 0000000000000002, IRQL
Arg3: 0000000000000008, value 0 = read operation, 1 = write operation
Arg4: fffff803712e5000, address which referenced memory

Debugging Details:
------------------

Page 24d116 not present in the dump file. Type ".hh dbgerr004" for details

KEY_VALUES_STRING: 1

    Key  : Analysis.CPU.mSec
    Value: 5202

    Key  : Analysis.Elapsed.mSec
    Value: 6287

    Key  : Analysis.IO.Other.Mb
    Value: 0

    Key  : Analysis.IO.Read.Mb
    Value: 0

    Key  : Analysis.IO.Write.Mb
    Value: 6

    Key  : Analysis.Init.CPU.mSec
    Value: 3171

    Key  : Analysis.Init.Elapsed.mSec
    Value: 326673

    Key  : Analysis.Memory.CommitPeak.Mb
    Value: 155

    Key  : Bugcheck.Code.KiBugCheckData
    Value: 0xd1

    Key  : Bugcheck.Code.LegacyAPI
    Value: 0xd1

    Key  : Dump.Attributes.AsUlong
    Value: 800

    Key  : Failure.Bucket
    Value: AV_CODE_AV_PAGED_IP_TestDriver!AccessPageableMemory

    Key  : Failure.Hash
    Value: {2463ca34-5d68-cb7c-6807-dd2087a41e36}

    Key  : Hypervisor.Enlightenments.Value
    Value: 1113060

    Key  : Hypervisor.Enlightenments.ValueHex
    Value: 10fbe4

    Key  : Hypervisor.Flags.AnyHypervisorPresent
    Value: 1

    Key  : Hypervisor.Flags.ApicEnlightened
    Value: 0

    Key  : Hypervisor.Flags.ApicVirtualizationAvailable
    Value: 0

    Key  : Hypervisor.Flags.AsyncMemoryHint
    Value: 0

    Key  : Hypervisor.Flags.CoreSchedulerRequested
    Value: 0

    Key  : Hypervisor.Flags.CpuManager
    Value: 0

    Key  : Hypervisor.Flags.DeprecateAutoEoi
    Value: 1

    Key  : Hypervisor.Flags.DynamicCpuDisabled
    Value: 1

    Key  : Hypervisor.Flags.Epf
    Value: 0

    Key  : Hypervisor.Flags.ExtendedProcessorMasks
    Value: 1

    Key  : Hypervisor.Flags.HardwareMbecAvailable
    Value: 1

    Key  : Hypervisor.Flags.MaxBankNumber
    Value: 0

    Key  : Hypervisor.Flags.MemoryZeroingControl
    Value: 0

    Key  : Hypervisor.Flags.NoExtendedRangeFlush
    Value: 0

    Key  : Hypervisor.Flags.NoNonArchCoreSharing
    Value: 0

    Key  : Hypervisor.Flags.Phase0InitDone
    Value: 1

    Key  : Hypervisor.Flags.PowerSchedulerQos
    Value: 0

    Key  : Hypervisor.Flags.RootScheduler
    Value: 0

    Key  : Hypervisor.Flags.SynicAvailable
    Value: 1

    Key  : Hypervisor.Flags.UseQpcBias
    Value: 0

    Key  : Hypervisor.Flags.Value
    Value: 659708

    Key  : Hypervisor.Flags.ValueHex
    Value: a10fc

    Key  : Hypervisor.Flags.VpAssistPage
    Value: 1

    Key  : Hypervisor.Flags.VsmAvailable
    Value: 1

    Key  : Hypervisor.RootFlags.AccessStats
    Value: 0

    Key  : Hypervisor.RootFlags.CrashdumpEnlightened
    Value: 0

    Key  : Hypervisor.RootFlags.CreateVirtualProcessor
    Value: 0

    Key  : Hypervisor.RootFlags.DisableHyperthreading
    Value: 0

    Key  : Hypervisor.RootFlags.HostTimelineSync
    Value: 0

    Key  : Hypervisor.RootFlags.HypervisorDebuggingEnabled
    Value: 0

    Key  : Hypervisor.RootFlags.IsHyperV
    Value: 0

    Key  : Hypervisor.RootFlags.LivedumpEnlightened
    Value: 0

    Key  : Hypervisor.RootFlags.MapDeviceInterrupt
    Value: 0

    Key  : Hypervisor.RootFlags.MceEnlightened
    Value: 0

    Key  : Hypervisor.RootFlags.Nested
    Value: 0

    Key  : Hypervisor.RootFlags.StartLogicalProcessor
    Value: 0

    Key  : Hypervisor.RootFlags.Value
    Value: 0

    Key  : Hypervisor.RootFlags.ValueHex
    Value: 0

    Key  : SecureKernel.HalpHvciEnabled
    Value: 0

    Key  : WER.OS.Branch
    Value: ni_release_svc_prod3

    Key  : WER.OS.Version
    Value: 10.0.22621.2506


BUGCHECK_CODE:  d1

BUGCHECK_P1: fffff803712e5000

BUGCHECK_P2: 2

BUGCHECK_P3: 8

BUGCHECK_P4: fffff803712e5000

FILE_IN_CAB:  MEMORY.DMP

VIRTUAL_MACHINE:  HyperV

DUMP_FILE_ATTRIBUTES: 0x800

READ_ADDRESS:  fffff803712e5000 

IP_IN_PAGED_CODE: 
TestDriver!AccessPageableMemory+0 [C:\Users\User\source\repos\TestDriver\TestDriver\Driver.c @ 11]
Page 24d116 not present in the dump file. Type ".hh dbgerr004" for details
Page 24d116 not present in the dump file. Type ".hh dbgerr004" for details
fffff803`712e5000 ??              ???

BLACKBOXBSD: 1 (!blackboxbsd)


BLACKBOXNTFS: 1 (!blackboxntfs)


BLACKBOXPNP: 1 (!blackboxpnp)


BLACKBOXWINLOGON: 1

PROCESS_NAME:  System

TRAP_FRAME:  ffffae03936f73a0 -- (.trap 0xffffae03936f73a0)
NOTE: The trap frame does not contain all registers.
Some register values may be zeroed or incorrect.
rax=0000000000000000 rbx=0000000000000000 rcx=0000000000000000
rdx=0000000000000000 rsi=0000000000000000 rdi=0000000000000000
rip=fffff803712e5000 rsp=ffffae03936f7538 rbp=0000000000000080
 r8=0000000000000002  r9=00000410000002ff r10=fffff803712e1060
r11=0000000000000000 r12=0000000000000000 r13=0000000000000000
r14=0000000000000000 r15=0000000000000000
iopl=0         nv up ei pl zr na po nc
TestDriver!AccessPageableMemory:
Page 24d116 not present in the dump file. Type ".hh dbgerr004" for details
Page 24d116 not present in the dump file. Type ".hh dbgerr004" for details
fffff803`712e5000 ??              ???
Resetting default scope

FAILED_INSTRUCTION_ADDRESS: 
TestDriver!AccessPageableMemory+0 [C:\Users\User\source\repos\TestDriver\TestDriver\Driver.c @ 11]
Page 24d116 not present in the dump file. Type ".hh dbgerr004" for details
Page 24d116 not present in the dump file. Type ".hh dbgerr004" for details
fffff803`712e5000 ??              ???

STACK_TEXT:  
ffffae03`936f7258 fffff803`44a2bfa9     : 00000000`0000000a fffff803`712e5000 00000000`00000002 00000000`00000008 : nt!KeBugCheckEx
ffffae03`936f7260 fffff803`44a27634     : ffff908b`9ea1bf9b 00000000`00000000 0000024a`d7402440 0000024a`d743ebc0 : nt!KiBugCheckDispatch+0x69
ffffae03`936f73a0 fffff803`712e5000     : fffff803`712e1075 00000000`00000000 00000000`00000000 00000000`00000000 : nt!KiPageFault+0x474
ffffae03`936f7538 fffff803`712e1075     : 00000000`00000000 00000000`00000000 00000000`00000000 00000000`00000000 : TestDriver!AccessPageableMemory [C:\Users\User\source\repos\TestDriver\TestDriver\Driver.c @ 11] 
ffffae03`936f7540 fffff803`449573d7     : ffffe682`e8e7f040 fffff803`712e1060 00000000`00000000 00000000`00000000 : TestDriver!PageableMemoryAccessThread+0x15 [C:\Users\User\source\repos\TestDriver\TestDriver\Driver.c @ 29] 
ffffae03`936f7570 fffff803`44a1b8d4     : ffffb100`572a8180 ffffe682`e8e7f040 fffff803`44957380 00000000`00000246 : nt!PspSystemThreadStartup+0x57
ffffae03`936f75c0 00000000`00000000     : ffffae03`936f8000 ffffae03`936f1000 00000000`00000000 00000000`00000000 : nt!KiStartSystemThread+0x34


FAULTING_SOURCE_LINE:  C:\Users\User\source\repos\TestDriver\TestDriver\Driver.c

FAULTING_SOURCE_FILE:  C:\Users\User\source\repos\TestDriver\TestDriver\Driver.c

FAULTING_SOURCE_LINE_NUMBER:  11

FAULTING_SOURCE_CODE:  
     7: // Place the AccessPageableMemory function in a pageable section.
     8: #pragma alloc_text(PAGE, AccessPageableMemory)
     9: 
    10: // Definition of AccessPageableMemory, a function declared as pageable.
>   11: void AccessPageableMemory() {
    12:     // Accessing pageable memory here (This is for demonstration only).
    13:     // In practice, this operation is illegal at DISPATCH_LEVEL and above.
    14:     DbgPrint("Accessing pageable memory...\n");
    15: }
    16: 


SYMBOL_NAME:  TestDriver!AccessPageableMemory+0

MODULE_NAME: TestDriver

IMAGE_NAME:  TestDriver.sys

STACK_COMMAND:  .cxr; .ecxr ; kb

BUCKET_ID_FUNC_OFFSET:  0

FAILURE_BUCKET_ID:  AV_CODE_AV_PAGED_IP_TestDriver!AccessPageableMemory

OS_VERSION:  10.0.22621.2506

BUILDLAB_STR:  ni_release_svc_prod3

OSPLATFORM_TYPE:  x64

OSNAME:  Windows 10

FAILURE_ID_HASH:  {2463ca34-5d68-cb7c-6807-dd2087a41e36}

Followup:     MachineOwner
---------
```

From this analysis, it is clear that the driver tried to read from a pageable memory address while operating at **`DISPATCH_LEVEL`**, which is not allowed. Let's use the following arguments of this bugcheck code as a placeholder:

```
1: kd> !analyze -show
DRIVER_IRQL_NOT_LESS_OR_EQUAL (d1)
An attempt was made to access a pageable (or completely invalid) address at an
interrupt request level (IRQL) that is too high.  This is usually
caused by drivers using improper addresses.
If kernel debugger is available get stack backtrace.
Arguments:
Arg1: fffff803712e5000, memory referenced
Arg2: 0000000000000002, IRQL
Arg3: 0000000000000008, value 0 = read operation, 1 = write operation
Arg4: fffff803712e5000, address which referenced memory
```

Microsoft Documentation is giving us some tips what to do next:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/d2643705-a023-40ad-ac95-6bd6db7c4f8a)


The output from the **`!pool`** command is used to analyze memory pool issues. This output indicates that the memory address **`fffff803712e5000`** is not part of the recognized kernel memory pool regions. It could potentially be a freed, invalid, or paged-out memory address.

```
1: kd> !pool fffff803712e5000
Pool page fffff803712e5000 region is Unknown
fffff803712e5000 is not a valid large pool allocation, checking large session pool...
Unable to read large session pool table (Session data is not present in mini and kernel-only dumps)
fffff803712e5000 is not valid pool. Checking for freed (or corrupt) pool
Address fffff803712e5000 could not be read. It may be a freed, invalid or paged out page
```

The **`!pte`** output indicates that the virtual address **`fffff803712e5000`** does not have a currently valid mapping to a physical address in memory, as evidenced by the non-valid final PTE. 

```
1: kd> !pte fffff803712e5000
                                           VA fffff803712e5000
PXE at FFFFFA7D3E9F4F80    PPE at FFFFFA7D3E9F0068    PDE at FFFFFA7D3E00DC48    PTE at FFFFFA7C01B89728
contains 00000000F6C0B063  contains 0000000010017063  contains 0A0000013BFBC863  contains C182CC1FFDF80400
pfn f6c0b     ---DA--KWEV  pfn 10017     ---DA--KWEV  pfn 13bfbc    ---DA--KWEV  not valid
                                                                                  Proto: FFFFC182CC1FFDF8
```

The **`!address`** command is providing information about a virtual address. From this information, it's clear that the virtual address **`fffff803712e5000`** is part of the memory allocated to TestDriver.sys, a driver module loaded during the boot process. 

```
1: kd> !address fffff803712e5000
Mapping user range ...
Mapping system range ...
Mapping non addressable range ...
Mapping page tables...
Mapping hyperspace...
Mapping HAL reserved range...
Mapping User Probe Area...
Mapping system shared page...
Mapping system cache working set...
Mapping loader mappings...
Mapping system PTEs...
Mapping system paged pool...
Mapping session space...
Mapping dynamic system space...
Mapping PFN database...
Mapping non paged pool...
Mapping VAD regions...
Mapping module regions...
Mapping process, thread, and stack regions...
Mapping system cache regions...


Usage:                  Module
Base Address:           fffff803`712e0000
End Address:            fffff803`712e8000
Region Size:            00000000`00008000
VA Type:                BootLoaded
Module name:            TestDriver.sys
Module path:            [\??\C:\Users\User\source\repos\TestDriver\x64\Release\TestDriver.sys]
```

Given this information, the error suggests that the driver was trying to access memory that is either pageable or completely invalid while the system was operating at **`DISPATCH_LEVEL`**. 

The **`.trap`** output shows the trap frame, which is a data structure saved by the operating system when an exception occurs. The trap frame contains the state of the CPU registers at the time of the exception.

```
1: kd> .trap 0xffffae03936f73a0
NOTE: The trap frame does not contain all registers.
Some register values may be zeroed or incorrect.
rax=0000000000000000 rbx=0000000000000000 rcx=0000000000000000
rdx=0000000000000000 rsi=0000000000000000 rdi=0000000000000000
rip=fffff803712e5000 rsp=ffffae03936f7538 rbp=0000000000000080
 r8=0000000000000002  r9=00000410000002ff r10=fffff803712e1060
r11=0000000000000000 r12=0000000000000000 r13=0000000000000000
r14=0000000000000000 r15=0000000000000000
iopl=0         nv up ei pl zr na po nc
TestDriver!AccessPageableMemory:
Page 24d116 not present in the dump file. Type ".hh dbgerr004" for details
Page 24d116 not present in the dump file. Type ".hh dbgerr004" for details
fffff803`712e5000 ??              ???
```

The **`rip`** register holds the address of the next instruction to be executed. Here, it points to **`fffff803712e5000`**. This suggests that the fault occurred in or near the **`TestDriver!AccessPageableMemory`** function. This can be confirmed as well if we go to the crashing thread:

```
0: kd> !mex.t
Process                   Thread                       CID       UserTime KernelTime ContextSwitches Wait Reason       Time State
System (ffffe682e1aeb040) ffffe682e8e7f040 (E|K|W|R|V) 4.1944           0          0               1 Executive   25m:05.765 Running on processor 1

# Child-SP         Return           Call Site                                  Source
0 ffffae03936f7258 fffff80344a2bfa9 nt!KeBugCheckEx                            
1 ffffae03936f7260 fffff80344a27634 nt!KiBugCheckDispatch+0x69                 
2 ffffae03936f73a0 fffff803712e5000 nt!KiPageFault+0x474                       
3 ffffae03936f7538 fffff803712e1075 TestDriver!AccessPageableMemory            C:\Users\User\source\repos\TestDriver\TestDriver\Driver.c @ 11
4 ffffae03936f7540 fffff803449573d7 TestDriver!PageableMemoryAccessThread+0x15 C:\Users\User\source\repos\TestDriver\TestDriver\Driver.c @ 29
5 ffffae03936f7570 fffff80344a1b8d4 nt!PspSystemThreadStartup+0x57             
6 ffffae03936f75c0 0000000000000000 nt!KiStartSystemThread+0x34
```           

The **`!irql 1`** command output indicates that processor **1** was operating at **`DISPATCH_LEVEL`** (IRQL 2) at the time the debugger captured the state.

```
0: kd> !irql 1
Debugger saved IRQL for processor 0x1 -- 2 (DISPATCH_LEVEL)
```
