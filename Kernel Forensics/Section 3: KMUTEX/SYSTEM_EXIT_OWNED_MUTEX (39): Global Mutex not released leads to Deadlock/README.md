# Description

The deadlock in the code occurs because the **`DriverEntry`** function acquires a global mutex **`(GlobalProcessMutex)`** and never releases it. When the **`ProcessNotifyCallback`** function, registered for process notifications, tries to acquire the same mutex, it gets blocked. This results in a deadlock situation since the **`ProcessNotifyCallback`** will always wait for a mutex that is held by **`DriverEntry`**

```c
#include <Ntifs.h>
#include <ntddk.h>
#include <wdm.h>

void ProcessNotifyCallback(HANDLE ParentProcessId, HANDLE ProcessId, BOOLEAN Create);
void DriverUnloadRoutine(PDRIVER_OBJECT DriverObject);
void WaitForGlobalMutex();

// Global mutex object for process synchronization
KMUTEX GlobalProcessMutex;

void ProcessNotifyCallback(HANDLE ParentProcessId, HANDLE ProcessId, BOOLEAN Create) {
    PEPROCESS Process = NULL;
    PUNICODE_STRING FullProcessName = NULL;

    // Wait for the global mutex
    WaitForGlobalMutex();

    if (Create) {
        PsLookupProcessByProcessId(ProcessId, &Process);
        if (Process) {
            SeLocateProcessImageName(Process, &FullProcessName);
            if (FullProcessName) {
                WCHAR* ProcessName = wcsrchr(FullProcessName->Buffer, L'\\');
                if (ProcessName) ProcessName++; // Skip the backslash
                else ProcessName = FullProcessName->Buffer;

                DbgPrint("| %10d | %8d | %-20S |\n", ParentProcessId, ProcessId, ProcessName);
                ExFreePool(FullProcessName);
            }
            ObDereferenceObject(Process);
        }
    }
    else {
        DbgPrint("| %10d | %8d | -                    |\n", ParentProcessId, ProcessId);
    }
}

void WaitForGlobalMutex() {
    NTSTATUS status = KeWaitForSingleObject(&GlobalProcessMutex, Executive, KernelMode, FALSE, NULL);
    if (status != STATUS_SUCCESS) {
        // Handle error or failed acquisition
        DbgPrint("Failed to wait for global mutex: 0x%X\n", status);
    }
    // Note: In this scenario, the mutex is not released after acquisition
}

void DriverUnloadRoutine(PDRIVER_OBJECT DriverObject) {
    PsSetCreateProcessNotifyRoutine(ProcessNotifyCallback, TRUE);
    DbgPrint("Driver unloaded\n");
}

NTSTATUS DriverEntry(PDRIVER_OBJECT DriverObject, PUNICODE_STRING RegistryPath) {
    UNREFERENCED_PARAMETER(RegistryPath);

    // Initialize the global mutex
    KeInitializeMutex(&GlobalProcessMutex, 0);

    // Acquire the mutex and intentionally do not release it
    KeWaitForSingleObject(&GlobalProcessMutex, Executive, KernelMode, FALSE, NULL);

    DriverObject->DriverUnload = DriverUnloadRoutine;

    NTSTATUS status = PsSetCreateProcessNotifyRoutine(ProcessNotifyCallback, FALSE);
    if (!NT_SUCCESS(status)) {
        DbgPrint("Failed to register process notify routine: 0x%X\n", status);
        return status;
    }

    DbgPrint("Driver loaded and process notify routine registered\n");
    return STATUS_SUCCESS;
}
```

Start loading this kernel driver:

```
sc.exe create TestDrv type= kernel binPath=C:\Users\User\source\repos\TestDriver\x64\Release\TestDriver.sys
```

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/bd7d3f88-d0c7-47e6-b733-af647c96f53c)

Start the kernel driver:

```
sc.exe start TestDrv
```

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/b07a0142-1c0b-4e45-b6f5-71f728fcfcfe)

Here we can see that the machine has experienced a bugcheck:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/cd5e5a4b-c446-4ffb-b0d1-5b2705d08e9e)


# WinDbg Walk Through - Analyzing Crash Dump

Load the CrashDump in WinDbg:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/3beb68e1-f4ca-44cc-9f57-4d6891b91b11)


Start with loading the **MEX** extension:

```
2: kd> .load mex
Mex External 3.0.0.7172 Loaded!
```

Let's start with the **`!di`** command which stands for dump information. This command retrieves information about the machine from which the memory dump originated. It's especially handy when working with multiple memory dumps, as it helps distinguish the analysis related to each specific dump. This is an improved version of the vertarget command.

```
1: kd> !mex.di
Computer Name:  Not Found
Windows 10 Kernel Version 19041 MP (4 procs) Free x64
Product: WinNt, suite: TerminalServer SingleUserTS
Edition build lab: 19041.1.amd64fre.vb_release.191206-1406
Kernel base = 0xfffff804`06000000 PsLoadedModuleList = 0xfffff804`06c2a770
Debug session time: Fri Dec 29 15:06:10.787 2023 (UTC - 8:00)
System Uptime: 0 days 0:40:05.871
SystemManufacturer = Microsoft Corporation
SystemProductName = Virtual Machine
Processor: 11th Gen Intel(R) Core(TM) i7-1185G7 @ 3.00GHz
Bugcheck: 39 (FFFFF8040677FE00, FFFFC80F43C8F3F0, FFFFC80F43C8F3F0, 0)
```

Let's run the **`!analyze -v`** command to quickly triage the issue as it provides a quick and detailed overview of the state of the system at the time of the crash. It automates many aspects of crash dump analysis, providing a quick way to get valuable information. This output points to an issue where a thread in the system was terminated without releasing a lock it had acquired. The provided addresses and the stack trace are key in further investigating this issue. However, let's get back to that later.

```
1: kd> !analyze -v
*******************************************************************************
*                                                                             *
*                        Bugcheck Analysis                                    *
*                                                                             *
*******************************************************************************

SYSTEM_EXIT_OWNED_MUTEX (39)
Arguments:
Arg1: fffff8040677fe00
Arg2: ffffc80f43c8f3f0
Arg3: ffffc80f43c8f3f0
Arg4: 0000000000000000

Debugging Details:
------------------


KEY_VALUES_STRING: 1

    Key  : Analysis.CPU.mSec
    Value: 1953

    Key  : Analysis.Elapsed.mSec
    Value: 2340

    Key  : Analysis.IO.Other.Mb
    Value: 0

    Key  : Analysis.IO.Read.Mb
    Value: 1

    Key  : Analysis.IO.Write.Mb
    Value: 7

    Key  : Analysis.Init.CPU.mSec
    Value: 5015

    Key  : Analysis.Init.Elapsed.mSec
    Value: 577716

    Key  : Analysis.Memory.CommitPeak.Mb
    Value: 138

    Key  : Bugcheck.Code.KiBugCheckData
    Value: 0x39

    Key  : Bugcheck.Code.LegacyAPI
    Value: 0x39

    Key  : Failure.Bucket
    Value: 0x39_nt!IopLoadUnloadDriver

    Key  : Failure.Hash
    Value: {040db381-bc8d-17b6-aa75-a2fbc31557a4}

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
    Value: vb_release

    Key  : WER.OS.Version
    Value: 10.0.19041.1


BUGCHECK_CODE:  39

BUGCHECK_P1: fffff8040677fe00

BUGCHECK_P2: ffffc80f43c8f3f0

BUGCHECK_P3: ffffc80f43c8f3f0

BUGCHECK_P4: 0

FILE_IN_CAB:  MEMORY.DMP

VIRTUAL_MACHINE:  HyperV

BLACKBOXBSD: 1 (!blackboxbsd)


BLACKBOXNTFS: 1 (!blackboxntfs)


BLACKBOXPNP: 1 (!blackboxpnp)


BLACKBOXWINLOGON: 1

PROCESS_NAME:  System

STACK_TEXT:  
ffffc80f`44ab74e8 fffff804`06456b56     : 00000000`00000039 fffff804`0677fe00 ffffc80f`43c8f3f0 ffffc80f`43c8f3f0 : nt!KeBugCheckEx
ffffc80f`44ab74f0 fffff804`063078e5     : ffffde8b`b9363040 00000000`00000080 ffffde8b`b3861040 ffffffff`feccf0fc : nt!ExpWorkerThread+0x1925a6
ffffc80f`44ab7590 fffff804`064064b8     : ffffc980`d3756180 ffffde8b`b9363040 fffff804`06307890 feafdff6`fe2c2c60 : nt!PspSystemThreadStartup+0x55
ffffc80f`44ab75e0 00000000`00000000     : ffffc80f`44ab8000 ffffc80f`44ab1000 00000000`00000000 00000000`00000000 : nt!KiStartSystemThread+0x28


SYMBOL_NAME:  nt!IopLoadUnloadDriver+0

MODULE_NAME: nt

IMAGE_NAME:  ntkrnlmp.exe

STACK_COMMAND:  .cxr; .ecxr ; kb

BUCKET_ID_FUNC_OFFSET:  0

FAILURE_BUCKET_ID:  0x39_nt!IopLoadUnloadDriver

OS_VERSION:  10.0.19041.1

BUILDLAB_STR:  vb_release

OSPLATFORM_TYPE:  x64

OSNAME:  Windows 10

FAILURE_ID_HASH:  {040db381-bc8d-17b6-aa75-a2fbc31557a4}

Followup:     MachineOwner
```

The **`!mex.mut`** command's output suggests the mutex has an issue with its **`MutantListEntry`** and does not have a valid owner thread at the time of the crash, which is consistent with the bugcheck's nature. The absence of an owning thread **`(OwnerThread: (null)`** could imply that the thread has already terminated or there's some inconsistency in the state of the mutex.

```
1: kd> !mex.mut ffffc80f43c8f3f0

dt nt!_KMUTANT ffffc80f43c8f3f0 () Recursive: [ -r1 -r2 -r ] Verbose Normal dt
==================================================================================
   +0x000 Header               : _DISPATCHER_HEADER
   +0x018 MutantListEntry      : _LIST_ENTRY [ 0xfffff804`06411235 - 0x00000000`00000000 ] [INVALID]
   +0x028 OwnerThread          : (null)
   +0x030 MutantFlags          : 0 ''
   +0x030 Abandoned            : 0y0
   +0x030 Spare1               : 0y0000000 (0)
   +0x030 Abandoned2           : 0y0
   +0x030 AbEnabled            : 0y0
   +0x030 Spare2               : 0y000000 (0)
   +0x031 ApcDisable           : 0 ''


Number of waiters: 1
No owning thread
```
