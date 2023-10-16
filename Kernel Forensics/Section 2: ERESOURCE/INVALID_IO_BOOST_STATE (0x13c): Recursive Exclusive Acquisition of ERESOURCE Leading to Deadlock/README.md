# Summary

The **`INVALID_IO_BOOST_STATE`** bug check has a value of **`0x0000013C`**. This indicates that a thread exited with an invalid I/O boost state. This should be zero when a thread exits.

Link to this CrashDump: https://mega.nz/file/b9ERhL4T#vke_kmv4LeI7pjBZNrbNr5GgIJKTIck4WlN3GjPebg8

# Code Sample - Deadlock

The root cause of the deadlock in the code is the incorrect use of **ExAcquireResourceExclusiveLite** in the **`FileMoveThread`** function. The function tries to acquire the same ERESOURCE object twice consecutively without releasing it in between. This results in a deadlock.

```c
#include <ntifs.h>
#include <ntstrsafe.h>

// Define a structure to hold our synchronization resources.
typedef struct _SYNC_RESOURCES {
    ERESOURCE FileResource;    // ERESOURCE object for synchronized access to the files.
} SYNC_RESOURCES, * PSYNC_RESOURCES;

NTSTATUS FileCreationThread(PVOID Context) {
    PSYNC_RESOURCES SyncRes = (PSYNC_RESOURCES)Context;
    UNICODE_STRING uniFilePath;
    WCHAR filePathBuffer[150];
    char dataToWrite[] = "Hello, World!\r\n";
    IO_STATUS_BLOCK ioStatusBlock;

    ExAcquireResourceExclusiveLite(&SyncRes->FileResource, TRUE);

    for (int i = 0; i < 1000; i++) {
        RtlStringCchPrintfW(filePathBuffer, sizeof(filePathBuffer) / sizeof(WCHAR), L"\\DosDevices\\C:\\Temp\\file_%d.txt", i);
        RtlInitUnicodeString(&uniFilePath, filePathBuffer);

        OBJECT_ATTRIBUTES objAttributes;
        InitializeObjectAttributes(&objAttributes, &uniFilePath, OBJ_CASE_INSENSITIVE | OBJ_KERNEL_HANDLE, NULL, NULL);

        HANDLE hFile;
        NTSTATUS status = ZwCreateFile(&hFile, GENERIC_WRITE, &objAttributes, &ioStatusBlock, NULL, FILE_ATTRIBUTE_NORMAL, 0, FILE_OVERWRITE_IF, FILE_SYNCHRONOUS_IO_NONALERT, NULL, 0);

        if (NT_SUCCESS(status)) {
            for (int j = 0; j < 100; j++) {
                ZwWriteFile(hFile, NULL, NULL, NULL, &ioStatusBlock, dataToWrite, sizeof(dataToWrite) - 1, NULL, NULL);
            }
            ZwClose(hFile);
        }
    }

    ExReleaseResourceLite(&SyncRes->FileResource);

    return STATUS_SUCCESS;
}

NTSTATUS FileMoveThread(PVOID Context) {
    PSYNC_RESOURCES SyncRes = (PSYNC_RESOURCES)Context;
    IO_STATUS_BLOCK ioStatusBlock;
    WCHAR oldFilePathBuffer[150], newFilePathBuffer[150];

    // DEADLOCK: Acquire the ERESOURCE twice without releasing in between.
    // This will introduce a deadlock situation as the resource can't be re-acquired before it's released.
    ExAcquireResourceExclusiveLite(&SyncRes->FileResource, TRUE);
    ExAcquireResourceExclusiveLite(&SyncRes->FileResource, TRUE);

    for (int i = 0; i < 1000; i++) {
        RtlStringCchPrintfW(oldFilePathBuffer, sizeof(oldFilePathBuffer) / sizeof(WCHAR), L"\\DosDevices\\C:\\Temp\\file_%d.txt", i);
        RtlStringCchPrintfW(newFilePathBuffer, sizeof(newFilePathBuffer) / sizeof(WCHAR), L"\\DosDevices\\C:\\Temp2\\NewFile_%d.txt", i);

        UNICODE_STRING oldFilePath, newFilePath;
        RtlInitUnicodeString(&oldFilePath, oldFilePathBuffer);
        RtlInitUnicodeString(&newFilePath, newFilePathBuffer);

        OBJECT_ATTRIBUTES objAttributes;
        InitializeObjectAttributes(&objAttributes, &oldFilePath, OBJ_CASE_INSENSITIVE | OBJ_KERNEL_HANDLE, NULL, NULL);

        HANDLE hFile;
        NTSTATUS status = ZwOpenFile(&hFile, FILE_ALL_ACCESS, &objAttributes, &ioStatusBlock, FILE_SHARE_READ | FILE_SHARE_WRITE, FILE_SYNCHRONOUS_IO_NONALERT);

        if (NT_SUCCESS(status)) {
            FILE_RENAME_INFORMATION renameInfo;
            RtlZeroMemory(&renameInfo, sizeof(FILE_RENAME_INFORMATION));
            renameInfo.ReplaceIfExists = TRUE;
            renameInfo.RootDirectory = NULL;
            renameInfo.FileNameLength = wcslen(newFilePathBuffer) * sizeof(WCHAR);
            RtlCopyMemory(renameInfo.FileName, newFilePathBuffer, renameInfo.FileNameLength);

            ZwSetInformationFile(hFile, &ioStatusBlock, &renameInfo, sizeof(FILE_RENAME_INFORMATION) + renameInfo.FileNameLength, FileRenameInformation);
            ZwClose(hFile);
        }
    }

    // Only one release for the two acquires, leaving the resource still acquired.
    ExReleaseResourceLite(&SyncRes->FileResource);

    return STATUS_SUCCESS;
}

VOID UnloadDriver(IN PDRIVER_OBJECT DriverObject) {
    PSYNC_RESOURCES SyncRes = (PSYNC_RESOURCES)DriverObject->DeviceObject->DeviceExtension;

    ExDeleteResourceLite(&SyncRes->FileResource);
    IoDeleteDevice(DriverObject->DeviceObject);
}

NTSTATUS DriverEntry(PDRIVER_OBJECT DriverObject, PUNICODE_STRING RegistryPath) {
    DriverObject->DriverUnload = UnloadDriver;

    PDEVICE_OBJECT DeviceObject = NULL;
    NTSTATUS status = IoCreateDevice(DriverObject, sizeof(SYNC_RESOURCES), NULL, FILE_DEVICE_UNKNOWN, FILE_DEVICE_SECURE_OPEN, FALSE, &DeviceObject);

    if (!NT_SUCCESS(status)) {
        return status;
    }

    PSYNC_RESOURCES SyncRes = (PSYNC_RESOURCES)DeviceObject->DeviceExtension;

    ExInitializeResourceLite(&SyncRes->FileResource);

    HANDLE threadHandle1, threadHandle2;

    status = PsCreateSystemThread(&threadHandle1, (ACCESS_MASK)0, NULL, (PHANDLE)0, NULL, FileCreationThread, SyncRes);
    if (NT_SUCCESS(status)) {
        ZwClose(threadHandle1);
    }

    status = PsCreateSystemThread(&threadHandle2, (ACCESS_MASK)0, NULL, (PHANDLE)0, NULL, FileMoveThread, SyncRes);
    if (NT_SUCCESS(status)) {
        ZwClose(threadHandle2);
    }

    return STATUS_SUCCESS;
}
```

Here is the issue of our code:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/1efb85d2-ca17-4140-853a-c1f46ffb7d85)


The **`FileMoveThread`** function is trying to acquire an exclusive lock on **`FileResource`** twice without releasing it after the first acquisition. Since it's an **exclusive lock**, no other thread or even the same thread can acquire it until it is released, leading to a deadlock.

Start loading the kernel driver:

```
sc.exe create TestDrv type= kernel binPath=C:\Users\Admin\source\repos\TestDriver\x64\Release\TestDriver.sys
```

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/e2cfeb43-4c4c-415b-9d41-a585c7289784)


Start the kernel driver:

```
sc.exe start TestDrv
```

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/a53eafd2-0c7a-4e21-b71a-c1f30b007d86)


This kernel driver will bugcheck the system:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/74153afe-b71d-4236-be00-58a0996d1386)


# WinDbg Walk Through - Analyzing CrashDump

Load the CrashDump in WinDbg:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/f15de286-59ec-46f9-bc04-b5c41c63b698)


Start with loading the **MEX** extension:

```
2: kd> .load mex
Mex External 3.0.0.7172 Loaded!
```

Let's start with the **`!di`** command which stands for dump information. This command retrieves information about the machine from which the memory dump originated. It's especially handy when working with multiple memory dumps, as it helps distinguish the analysis related to each specific dump. This is an improved version of the **`vertarget`** command.

```
2: kd> !di
Computer Name:  Not Found
Windows 10 Kernel Version 19041 MP (4 procs) Free x64
Product: WinNt, suite: TerminalServer SingleUserTS
Edition build lab: 19041.1.amd64fre.vb_release.191206-1406
Kernel base = 0xfffff804`5e200000 PsLoadedModuleList = 0xfffff804`5ee2a270
Debug session time: Sun Oct 15 02:43:30.919 2023 (UTC - 7:00)
System Uptime: 0 days 0:13:47.651
SystemManufacturer = Microsoft Corporation
SystemProductName = Virtual Machine
Processor: 11th Gen Intel(R) Core(TM) i7-1185G7 @ 3.00GHz
Bugcheck: 13C (FFFF8B0351756040, 1, 1, 0)
KernelMode Dump Path: C:\Windows\MEMORY_13C.DMP
Share Path: \\DESKTOP-OQGRG4S\ADMIN$\MEMORY_13C.DMP
```

Let's run the **`!analyze -v`** command to quickly triage the issue as it provides a quick and detailed overview of the state of the system at the time of the crash. It automates many aspects of crash dump analysis, providing a quick way to get valuable information.

```
2: kd> !analyze -v
*******************************************************************************
*                                                                             *
*                        Bugcheck Analysis                                    *
*                                                                             *
*******************************************************************************

INVALID_IO_BOOST_STATE (13c)
A thread exited with an invalid I/O boost state.  This should be zero when
a thread exits.
Arguments:
Arg1: ffff8b0351756040, Pointer to the thread which had the invalid boost state.
Arg2: 0000000000000001, Current boost state.
Arg3: 0000000000000001
Arg4: 0000000000000000

Debugging Details:
------------------


KEY_VALUES_STRING: 1

    Key  : Analysis.CPU.mSec
    Value: 3921

    Key  : Analysis.Elapsed.mSec
    Value: 4001

    Key  : Analysis.IO.Other.Mb
    Value: 0

    Key  : Analysis.IO.Read.Mb
    Value: 0

    Key  : Analysis.IO.Write.Mb
    Value: 10

    Key  : Analysis.Init.CPU.mSec
    Value: 3578

    Key  : Analysis.Init.Elapsed.mSec
    Value: 169834

    Key  : Analysis.Memory.CommitPeak.Mb
    Value: 136

    Key  : Bugcheck.Code.KiBugCheckData
    Value: 0x13c

    Key  : Bugcheck.Code.LegacyAPI
    Value: 0x13c

    Key  : Failure.Bucket
    Value: 0x13C_nt!PspThreadDelete

    Key  : Failure.Hash
    Value: {6fe88179-1572-f8e2-aeff-cd9c600ecf3a}

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


BUGCHECK_CODE:  13c

BUGCHECK_P1: ffff8b0351756040

BUGCHECK_P2: 1

BUGCHECK_P3: 1

BUGCHECK_P4: 0

FILE_IN_CAB:  MEMORY_13C.DMP

VIRTUAL_MACHINE:  HyperV

BLACKBOXBSD: 1 (!blackboxbsd)


BLACKBOXNTFS: 1 (!blackboxntfs)


BLACKBOXPNP: 1 (!blackboxpnp)


BLACKBOXWINLOGON: 1

PROCESS_NAME:  System

STACK_TEXT:  
ffff8287`1f44f3a8 fffff804`5ea0b0f7     : 00000000`0000013c ffff8b03`51756040 00000000`00000001 00000000`00000001 : nt!KeBugCheckEx
ffff8287`1f44f3b0 fffff804`5e8732d0     : ffff8b03`51756010 ffff8b03`51756010 fffff804`5e4e5370 00000000`00000000 : nt!PspThreadDelete+0x1706e7
ffff8287`1f44f420 fffff804`5e408357     : 00000000`00000000 00000000`00000000 fffff804`5e4e5370 ffff8b03`51756040 : nt!ObpRemoveObjectRoutine+0x80
ffff8287`1f44f480 fffff804`5e4e53e2     : 00000000`00000000 00000000`00000000 00000000`00000000 ffff8b03`51756498 : nt!ObfDereferenceObjectWithTag+0xc7
ffff8287`1f44f4c0 fffff804`5e4b8515     : ffff8b03`52ded080 ffff8b03`4a07ea30 ffff8b03`4a07ea30 00000000`00000000 : nt!PspReaper+0x72
ffff8287`1f44f4f0 fffff804`5e555855     : ffff8b03`52ded080 00000000`00000080 ffff8b03`4a09a040 00000000`00000000 : nt!ExpWorkerThread+0x105
ffff8287`1f44f590 fffff804`5e5fe808     : ffffd080`ee1e7180 ffff8b03`52ded080 fffff804`5e555800 000003cf`00000057 : nt!PspSystemThreadStartup+0x55
ffff8287`1f44f5e0 00000000`00000000     : ffff8287`1f450000 ffff8287`1f449000 00000000`00000000 00000000`00000000 : nt!KiStartSystemThread+0x28


SYMBOL_NAME:  nt!PspThreadDelete+1706e7

MODULE_NAME: nt

IMAGE_NAME:  ntkrnlmp.exe

STACK_COMMAND:  .cxr; .ecxr ; kb

BUCKET_ID_FUNC_OFFSET:  1706e7

FAILURE_BUCKET_ID:  0x13C_nt!PspThreadDelete

OS_VERSION:  10.0.19041.1

BUILDLAB_STR:  vb_release

OSPLATFORM_TYPE:  x64

OSNAME:  Windows 10

FAILURE_ID_HASH:  {6fe88179-1572-f8e2-aeff-cd9c600ecf3a}

Followup:     MachineOwner
---------
```

This call stack doesn't reveal any faulty drivers, making it unclear from this output what could be the cause of the crash. 

Let's look at the call stack a bit closer and see if we can examine something from it. The **`knL`** command in WinDbg is used to display the stack trace for the current thread with detailed frame information.

```
2: kd> knL
  *** Stack trace for last set context - .thread/.cxr resets it
 # Child-SP          RetAddr               Call Site
00 ffff8287`1fbd71d0 fffff804`5e40c800     nt!KiSwapContext+0x76
01 ffff8287`1fbd7310 fffff804`5e4f9aff     nt!KiSwapThread+0x500
02 ffff8287`1fbd73c0 fffff804`5e8b0949     nt!KeTerminateThread+0x1a7
03 ffff8287`1fbd7450 fffff804`5e8b3973     nt!PspExitThread+0x489
04 ffff8287`1fbd7550 fffff804`5e55586f     nt!PspTerminateThreadByPointer+0x53
05 ffff8287`1fbd7590 fffff804`5e5fe808     nt!PspSystemThreadStartup+0x6f
06 ffff8287`1fbd75e0 00000000`00000000     nt!KiStartSystemThread+0x28
```

The command **`.frame /r 2`** sets the context to the 2th frame in the stack trace and displays the register values at that moment. By examining the register values and the current instruction, we can get insights into what the function is doing at that point.

This is a snapshot of the CPU state at the **2th** frame of the stack, which is within the **`nt!KeTerminateThread`** function.

```
2: kd> .frame /r 2
02 ffff8287`1fbd73c0 fffff804`5e8b0949     nt!KeTerminateThread+0x1a7
rax=0000000000000000 rbx=ffff8b0351756040 rcx=0000000000000000
rdx=0000000000000000 rsi=ffff8b0351756338 rdi=ffff8b0351756040
rip=fffff8045e4f9aff rsp=ffff82871fbd73c0 rbp=ffffd080ee556180
 r8=0000000000000000  r9=0000000000000000 r10=0000000000000000
r11=0000000000000000 r12=0000000000000002 r13=0000000000000001
r14=ffff8b0351756048 r15=ffff8b0351756048
iopl=0         nv up di pl nz na pe nc
cs=0000  ss=0000  ds=0000  es=0000  fs=0000  gs=0000             efl=00000000
nt!KeTerminateThread+0x1a7:
fffff804`5e4f9aff 4883c450        add     rsp,50h
```

The **`!object`** command in WinDbg is used to display information about a kernel object. In this particular case, the **RDI** register that holds the address **`ffff8b0351756040`** is a thread.

```
2: kd> !object ffff8b0351756040
Object: ffff8b0351756040  Type: (ffff8b034a0b6220) Thread
    ObjectHeader: ffff8b0351756010 (new version)
    HandleCount: 0  PointerCount: 0
```

Interesting, we can see that this thread has been terminated but we don't have much context yet on what happened though.

```
0: kd> !mex.t ffff8b0351756040
Process                   Thread                       CID       UserTime KernelTime ContextSwitches Wait Reason        Time State
System (ffff8b034a09a040) ffff8b0351756040 (E|K|W|R|V) 4.1070           0       16ms               1 WrTerminated 13m:47.640 Terminated

# Child-SP         Return           Call Site
0 ffff82871fbd71d0 fffff8045e40c800 nt!KiSwapContext+0x76
1 ffff82871fbd7310 fffff8045e4f9aff nt!KiSwapThread+0x500
2 ffff82871fbd73c0 fffff8045e8b0949 nt!KeTerminateThread+0x1a7
3 ffff82871fbd7450 fffff8045e8b3973 nt!PspExitThread+0x489
4 ffff82871fbd7550 fffff8045e55586f nt!PspTerminateThreadByPointer+0x53
5 ffff82871fbd7590 fffff8045e5fe808 nt!PspSystemThreadStartup+0x6f
6 ffff82871fbd75e0 0000000000000000 nt!KiStartSystemThread+0x28

Warning!!! Thread is marked as terminated
```

Let's start with the **`!tl -t`** command to discover threads of interest. It examines the state and wait reasons for every thread across all processes, presenting the results as statistics. This allows for a quick overview, such as determining how many processes have 4 running threads, and so on. 

```
2: kd> !tl -t
PID            Address          Name                                   !! Rn Ry Bk Lc IO Er
============== ================ ====================================== == == == == == == ==
0x0    0n0     fffff8045ef24a00 Idle                                    .  4  .  .  .  .  .
0x4    0n4     ffff8b034a09a040 System                                  4  1  .  .  .  1  1
0x64   0n100   ffff8b034a0ea080 Registry                                .  .  .  .  .  .  .
0x1b0  0n432   ffff8b034a8301c0 smss.exe                                .  .  .  1  .  .  .
0x220  0n544   ffff8b034fdc8240 csrss.exe                               .  .  .  .  1  .  .
0x26c  0n620   ffff8b034ff0f080 wininit.exe                             .  .  .  .  .  .  .
0x278  0n632   ffff8b034ff20140 csrss.exe                               .  .  .  .  1  .  .
0x2c8  0n712   ffff8b034ff44080 winlogon.exe                            .  .  .  .  1  .  .
0x2f8  0n760   ffff8b034ff73240 services.exe                            .  .  .  .  .  .  .
0x320  0n800   ffff8b034ff710c0 lsass.exe                               .  .  .  .  .  .  .
0x3a4  0n932   ffff8b034ffab080 svchost.exe                             .  .  .  .  .  .  .
0x3bc  0n956   ffff8b0350c0c300 fontdrvhost.exe                         .  .  .  .  .  .  .
0x3c4  0n964   ffff8b0350c0f080 fontdrvhost.exe                         .  .  .  .  .  .  .
0x1fc  0n508   ffff8b0350c980c0 svchost.exe                             .  .  .  .  .  .  .
0x224  0n548   ffff8b0350cc9080 svchost.exe                             .  .  .  .  .  .  .
0x408  0n1032  ffff8b0350d9a1c0 dwm.exe                                 .  .  .  .  .  .  .
0x410  0n1040  ffff8b0350d98240 LogonUI.exe                             .  .  .  .  .  .  .
0x450  0n1104  ffff8b0350dc9080 svchost.exe                             .  .  .  .  .  .  .
0x4b4  0n1204  ffff8b0350e25080 svchost.exe                             .  .  .  .  .  .  .
0x51c  0n1308  ffff8b0350e3c080 svchost.exe                             .  .  .  .  .  .  .
0x52c  0n1324  ffff8b0350eee0c0 svchost.exe                             .  .  .  .  .  .  .
0x534  0n1332  ffff8b0350ef1080 svchost.exe                             .  .  .  .  .  .  .
0x53c  0n1340  ffff8b0350eef080 svchost.exe                             .  .  .  .  .  .  .
0x564  0n1380  ffff8b0350e56080 svchost.exe                             .  .  .  .  .  .  .
0x57c  0n1404  ffff8b0350e57080 svchost.exe                             .  .  .  .  .  .  .
0x5d8  0n1496  ffff8b0350fe00c0 svchost.exe                             .  .  .  .  .  .  .
0x60c  0n1548  ffff8b0351009080 svchost.exe                             .  .  .  .  .  .  .
0x614  0n1556  ffff8b035100c080 svchost.exe                             .  .  .  .  .  .  .
0x624  0n1572  ffff8b0351008080 svchost.exe                             .  .  .  .  .  .  .
0x654  0n1620  ffff8b035100f080 svchost.exe                             .  .  .  .  .  .  .
0x668  0n1640  ffff8b03510590c0 svchost.exe                             .  .  .  .  .  .  .
0x688  0n1672  ffff8b035105a080 svchost.exe                             .  .  .  .  .  .  .
0x6b0  0n1712  ffff8b03510b1080 svchost.exe                             .  .  .  .  .  .  .
0x6c0  0n1728  ffff8b03510d7080 svchost.exe                             .  .  .  .  .  .  .
0x6fc  0n1788  ffff8b035108f080 svchost.exe                             .  .  .  .  .  .  .
0x79c  0n1948  ffff8b035116e0c0 svchost.exe                             .  .  .  .  .  .  .
0x7a4  0n1956  ffff8b0351171080 svchost.exe                             .  .  .  .  .  .  .
0x7ac  0n1964  ffff8b035113a080 svchost.exe                             .  .  .  .  .  .  .
0x7bc  0n1980  ffff8b03511ab0c0 svchost.exe                             .  .  .  .  .  .  .
0x8cc  0n2252  ffff8b0351131080 VSSVC.exe                               .  .  .  .  .  .  .
0x8d4  0n2260  ffff8b0351134080 svchost.exe                             .  .  .  .  .  .  .
0x8e0  0n2272  ffff8b035120d080 svchost.exe                             .  .  .  .  .  .  .
0x8f4  0n2292  ffff8b0351135080 svchost.exe                             .  .  .  .  .  .  .
0x91c  0n2332  ffff8b03512ea040 MemCompression                          .  .  .  .  .  .  .
0x988  0n2440  ffff8b0351358080 svchost.exe                             .  .  .  .  .  .  .
0x998  0n2456  ffff8b0351313300 svchost.exe                             .  .  .  .  .  .  .
0x9d8  0n2520  ffff8b034a0770c0 svchost.exe                             .  .  .  .  .  .  .
0x9e8  0n2536  ffff8b034a103080 svchost.exe                             .  .  .  .  1  .  .
0xa24  0n2596  ffff8b034a0c0080 svchost.exe                             .  .  .  .  .  .  .
0xa90  0n2704  ffff8b034a1c2080 svchost.exe                             .  .  .  .  .  .  .
0xad0  0n2768  ffff8b034a15e080 svchost.exe                             .  .  .  .  .  .  .
0xaf4  0n2804  ffff8b034a107080 svchost.exe                             .  .  .  .  .  .  .
0xb40  0n2880  ffff8b03513b0080 svchost.exe                             .  .  .  .  .  .  .
0xb48  0n2888  ffff8b03513b1080 svchost.exe                             .  .  .  .  .  .  .
0xb58  0n2904  ffff8b03513b2080 svchost.exe                             .  .  .  .  .  .  .
0xbf4  0n3060  ffff8b035155c080 svchost.exe                             .  .  .  .  .  .  .
0x7c8  0n1992  ffff8b035155b080 svchost.exe                             .  .  .  .  .  .  .
0xc08  0n3080  ffff8b0351592080 spoolsv.exe                             .  .  .  .  .  .  .
0xc38  0n3128  ffff8b03515b0080 svchost.exe                             .  .  .  .  .  .  .
0xce4  0n3300  ffff8b0351628080 svchost.exe                             .  .  .  .  .  .  .
0xcec  0n3308  ffff8b035162a080 svchost.exe                             .  .  .  .  .  .  .
0xcf4  0n3316  ffff8b035162c080 svchost.exe                             .  .  .  .  .  .  .
0xd1c  0n3356  ffff8b035165e0c0 IpOverUsbSvc.exe*32                     .  .  .  .  .  .  .
0xd24  0n3364  ffff8b0351631080 svchost.exe                             .  .  .  .  .  .  .
0xd80  0n3456  ffff8b03517f1080 svchost.exe                             .  .  .  .  .  .  .
0xd88  0n3464  ffff8b03517f4080 svchost.exe                             .  .  .  .  .  .  .
0xd94  0n3476  ffff8b03517f8080 wlms.exe                                .  .  .  .  .  .  .
0xd9c  0n3484  ffff8b03517f5080 sqlwriter.exe                           .  .  .  .  .  .  .
0xdb0  0n3504  ffff8b03517f6080 svchost.exe                             .  .  .  .  .  .  .
0xdc4  0n3524  ffff8b03516e10c0 MsMpEng.exe                             .  .  .  .  .  .  .
0xde0  0n3552  ffff8b03517ef340 svchost.exe                             .  .  .  .  .  .  .
0xe24  0n3620  ffff8b0351779080 svchost.exe                             .  .  .  .  .  .  .
0xef4  0n3828  ffff8b03518020c0 sppsvc.exe                              .  .  .  .  .  .  .
0xf64  0n3940  ffff8b03518e6140 csrss.exe                               .  .  .  .  1  .  .
0xfac  0n4012  ffff8b035184b080 winlogon.exe                            .  .  .  .  .  .  .
0xf48  0n3912  ffff8b0351898080 WUDFHost.exe                            .  .  .  .  .  .  .
0x37c  0n892   ffff8b03520371c0 dwm.exe                                 .  .  .  .  .  .  .
0x101c 0n4124  ffff8b0352039300 fontdrvhost.exe                         .  .  .  .  .  .  .
0x11ac 0n4524  ffff8b0352291080 svchost.exe                             .  .  .  .  .  .  .
0x1234 0n4660  ffff8b03522f8080 svchost.exe                             .  .  .  .  .  .  .
0x131c 0n4892  ffff8b03523eb080 NisSrv.exe                              .  .  .  .  .  .  .
0xf6c  0n3948  ffff8b03523f7080 svchost.exe                             .  .  .  .  .  .  .
0x6d8  0n1752  ffff8b034f8e70c0 ctfmon.exe                              .  .  .  .  .  .  .
0x11a0 0n4512  ffff8b034fd140c0 rdpclip.exe                             .  .  .  .  .  .  .
0xb08  0n2824  ffff8b034f92b380 sihost.exe                              .  .  .  .  .  .  .
0x7f4  0n2036  ffff8b034f9b80c0 svchost.exe                             .  .  .  .  .  .  .
0x4f0  0n1264  ffff8b034f7ad0c0 svchost.exe                             .  .  .  .  .  .  .
0x6e0  0n1760  ffff8b034fcfa0c0 taskhostw.exe                           .  .  .  .  .  .  .
0x1410 0n5136  ffff8b034fb0d0c0 svchost.exe                             .  .  .  .  .  .  .
0x141c 0n5148  ffff8b034fb020c0 svchost.exe                             .  .  .  .  .  .  .
0x14f8 0n5368  ffff8b034f588300 svchost.exe                             .  .  .  .  .  .  .
0x1550 0n5456  ffff8b034af92080 rdpinput.exe                            .  .  .  .  .  .  .
0x15b4 0n5556  ffff8b034f510080 svchost.exe                             .  .  .  .  .  .  .
0x1604 0n5636  ffff8b034abca080 explorer.exe                            4  .  .  1  1  .  .
0x173c 0n5948  ffff8b034f935080 svchost.exe                             .  .  .  .  .  .  .
0x588  0n1416  ffff8b034f611080 dllhost.exe                             .  .  .  .  .  .  .
0x14c8 0n5320  ffff8b035269d080 SppExtComObj.Exe                        .  .  .  .  .  .  .
0x177c 0n6012  ffff8b03526d80c0 TabTip.exe                              .  .  .  .  .  .  .
0xf10  0n3856  ffff8b03526a0080 StartMenuExperienceHost.exe             .  .  .  .  .  .  .
0x16c8 0n5832  ffff8b034f4ae080 RuntimeBroker.exe                       .  .  .  .  .  .  .
0x1870 0n6256  ffff8b034f7de080 SearchApp.exe                          11  .  .  .  1  .  .
0x1878 0n6264  ffff8b0352810240 SearchIndexer.exe                       .  .  .  .  .  .  .
0x18ec 0n6380  ffff8b03528e90c0 RuntimeBroker.exe                       .  .  .  .  1  .  .
0x1a90 0n6800  ffff8b034f2eb300 RuntimeBroker.exe                       .  .  .  .  .  .  .
0x1260 0n4704  ffff8b034e115080 SecurityHealthSystray.exe               .  .  .  .  .  .  .
0x72c  0n1836  ffff8b034f2d5080 SecurityHealthService.exe               .  .  .  .  .  .  .
0xe58  0n3672  ffff8b034f2bc080 OneDrive.exe                            .  .  .  .  .  .  .
0x1dd0 0n7632  ffff8b034f0d60c0 svchost.exe                             .  .  .  .  .  .  .
0x1004 0n4100  ffff8b034f932080 svchost.exe                             .  .  .  .  .  .  .
0x1820 0n6176  ffff8b035332e0c0 svchost.exe                             .  .  .  .  .  .  .
0x1458 0n5208  ffff8b0353157300 SgrmBroker.exe                          .  .  .  .  .  .  .
0x1c70 0n7280  ffff8b035339b0c0 uhssvc.exe                              .  .  .  .  .  .  .
0x1880 0n6272  ffff8b0352ba8080 svchost.exe                             .  .  .  .  .  .  .
0x1c6c 0n7276  ffff8b0352d55080 MoUsoCoreWorker.exe                     .  .  .  .  .  .  .
0xd90  0n3472  ffff8b0352d58080 svchost.exe                             .  .  .  .  .  .  .
0x203c 0n8252  ffff8b034fac5080 svchost.exe                             .  .  .  .  .  .  .
0x20dc 0n8412  ffff8b0353cae0c0 svchost.exe                             .  .  .  .  .  .  .
0x2110 0n8464  ffff8b0352dad080 svchost.exe                             .  .  .  .  .  .  .
0x1b88 0n7048  ffff8b0352d98340 MusNotifyIcon.exe                       .  .  .  .  .  .  .
0x4a0  0n1184  ffff8b0352e10080 svchost.exe                             .  .  .  .  .  .  .
0x2070 0n8304  ffff8b0353405080 svchost.exe                             .  .  .  .  .  .  .
0x1e38 0n7736  ffff8b035360c0c0 WmiPrvSE.exe                            .  .  .  .  .  .  .
0x1674 0n5748  ffff8b03552e70c0 svchost.exe                             .  .  .  .  .  .  .
0x261c 0n9756  ffff8b03531db080 Microsoft.SharePoint.exe                .  .  .  .  .  .  .
0x14c0 0n5312  ffff8b0354745080 TrustedInstaller.exe                    .  .  .  .  .  .  .
0x17b8 0n6072  ffff8b0350dc8340 TiWorker.exe                            .  .  .  .  .  .  .
0x15a4 0n5540  ffff8b035361a080 svchost.exe                             .  .  .  .  .  .  .
0x2324 0n8996  ffff8b03555c5340 smartscreen.exe                         .  .  .  .  .  .  .
0x279c 0n10140 ffff8b0352d78080 devenv.exe                              .  .  .  1  .  .  .
0x2494 0n9364  ffff8b034fae1080 PerfWatson2.exe                         .  .  .  1  .  .  .
0x2528 0n9512  ffff8b034e118080 Microsoft.ServiceHub.Controller.exe     .  .  .  1  .  .  .
0x2248 0n8776  ffff8b0353dee300 ServiceHub.VSDetouredHost.exe           .  .  .  1  .  .  .
0x12c4 0n4804  ffff8b034fcdc300 ServiceHub.SettingsHost.exe             .  .  .  2  .  .  .
0x1cb4 0n7348  ffff8b034f0e1080 TextInputHost.exe                       .  .  .  .  .  .  .
0x2718 0n10008 ffff8b0352d7b300 ServiceHub.IdentityHost.exe             .  .  .  2  .  .  .
0x2708 0n9992  ffff8b034e25a080 ServiceHub.Host.netfx.x86.exe*32        .  .  .  3  .  .  .
0xe08  0n3592  ffff8b034f8b6300 ServiceHub.ThreadedWaitDialog.exe       .  .  .  1  .  .  .
0xa58  0n2648  ffff8b0352ba4300 ServiceHub.IndexingService.exe          .  .  .  .  .  .  .
0xa04  0n2564  ffff8b034abeb080 vshost.exe                              1  .  .  .  .  .  .
0x2650 0n9808  ffff8b03533bd300 vcpkgsrv.exe*32                         .  .  .  .  .  .  .
0x1210 0n4624  ffff8b0353196080 ServiceHub.IntellicodeModelService.exe  .  .  .  .  .  .  .
0x2d8  0n728   ffff8b0353247080 vcpkgsrv.exe*32                         .  .  .  .  .  .  .
0x27f4 0n10228 ffff8b034aed8340 vcpkgsrv.exe*32                         .  .  .  .  .  .  .
0x1024 0n4132  ffff8b034e0e6080 MSBuild.exe                             .  .  .  .  .  .  .
0x22dc 0n8924  ffff8b0352e1e080 conhost.exe                             .  .  .  .  .  .  .
0x2880 0n10368 ffff8b034f7f0080 svchost.exe                             .  .  .  .  .  .  .
0x2b28 0n11048 ffff8b03519ec080 svchost.exe                             .  .  .  .  .  .  .
0x1f6c 0n8044  ffff8b034af25080 dllhost.exe                             .  .  .  .  .  .  .
0x2928 0n10536 ffff8b0353f71080 audiodg.exe                             .  .  .  .  .  .  .
0x2b1c 0n11036 ffff8b0353c17080 cmd.exe                                 .  .  .  .  .  .  .
0x2510 0n9488  ffff8b0352bd2080 conhost.exe                             .  1  .  .  .  .  .
0x2290 0n8848  ffff8b03524d9300 notepad.exe                             .  .  .  .  .  .  .
0x116c 0n4460  ffff8b03532a0300 svchost.exe                             .  .  .  .  .  .  .
0xc68  0n3176  ffff8b0353355080 svchost.exe                             .  .  .  .  .  .  .
0x494  0n1172  ffff8b03550f4080 svchost.exe                             .  .  .  .  .  .  .
============== ================ ====================================== == == == == == == ==
PID            Address          Name                                   !! Rn Ry Bk Lc IO Er

Warning! Zombie process(es) detected (not displayed). Count: 5 [zombie report]
```

There is a thread that is waiting on an **ERESOURCE** as we can see from the output above. 

```
0: kd> !mex.lt ffff8b034a09a040 -wr WrResource
Process PID Thread             Id State   Time Reason
======= === ================ ==== ======= ==== ==========
System    4 ffff8b03515d5040 2018 Waiting 15ms WrResource

Thread Count: 1
```

This thread has been waiting **15ms** on an **ERESOURCE** **`ffff8b0352014a90`** owned exclusively by System thread **`ffff8b0351756040`**

```
0: kd> !mex.t ffff8b03515d5040
Process                   Thread                       CID       UserTime KernelTime ContextSwitches Wait Reason Time State
System (ffff8b034a09a040) ffff8b03515d5040 (E|K|W|R|V) 4.2018           0          0               1 WrResource  15ms Waiting

WaitBlockList:
    Object           Type                 Other Waiters
    ffff82871d947360 SynchronizationEvent             0

Thread at address 0000000000060001 is invalid. Not adding to waiter list.
# Child-SP         Return           Call Site                               Info                                                                             Source
0 ffff82871d946f00 fffff8045e40c800 nt!KiSwapContext+0x76                                                                                                    
1 ffff82871d947040 fffff8045e40bd2f nt!KiSwapThread+0x500                                                                                                    
2 ffff82871d9470f0 fffff8045e40b5d3 nt!KiCommitThreadWait+0x14f                                                                                              
3 ffff82871d947190 fffff8045e40e4ad nt!KeWaitForSingleObject+0x233                                                                                           
4 ffff82871d947280 fffff8045e408eee nt!ExpWaitForResource+0x6d                                                                                               
5 ffff82871d947300 fffff804653c1116 nt!ExAcquireResourceExclusiveLite+0x1fe Thread: ffff8b0351756040 (System) exclusive owner of EResource: ffff8b0352014a90 
6 ffff82871d947390 fffff8045e555855 TestDriver!FileCreationThread+0x36                                                                                       C:\Users\Admin\source\repos\TestDriver\TestDriver\Driver.c @ 16
7 ffff82871d947590 fffff8045e5fe808 nt!PspSystemThreadStartup+0x55                                                                                           
8 ffff82871d9475e0 0000000000000000 nt!KiStartSystemThread+0x28
```

The **ERESOURCE** is currently owned by a thread in the **System** process, and it has been held for a significantly long time (over 13 minutes).

```
0: kd> !mex.eresource ffff8b0352014a90
nt!_ERESOURCE @ ffff8b0352014a90 [verbose]
Contention count: 1

Exclusive owner:
Thread at address 0000000000060001 is invalid. Not adding to waiter list.
    Process Thread                 Time
    ======= ================ ==========
    System  ffff8b0351756040 13m:47.640
```

The warning message **Warning!!! Thread is marked as terminated** is a good indicator that something bad is going on. It means the thread has been terminated, but it appears to still hold an ERESOURCE.

```
0: kd> !mex.t ffff8b0351756040
Process                   Thread                       CID       UserTime KernelTime ContextSwitches Wait Reason        Time State
System (ffff8b034a09a040) ffff8b0351756040 (E|K|W|R|V) 4.1070           0       16ms               1 WrTerminated 13m:47.640 Terminated

# Child-SP         Return           Call Site
0 ffff82871fbd71d0 fffff8045e40c800 nt!KiSwapContext+0x76
1 ffff82871fbd7310 fffff8045e4f9aff nt!KiSwapThread+0x500
2 ffff82871fbd73c0 fffff8045e8b0949 nt!KeTerminateThread+0x1a7
3 ffff82871fbd7450 fffff8045e8b3973 nt!PspExitThread+0x489
4 ffff82871fbd7550 fffff8045e55586f nt!PspTerminateThreadByPointer+0x53
5 ffff82871fbd7590 fffff8045e5fe808 nt!PspSystemThreadStartup+0x6f
6 ffff82871fbd75e0 0000000000000000 nt!KiStartSystemThread+0x28

Warning!!! Thread is marked as terminated
```

We can further confirm that this thread is indeed terminated by examining its **`ETHREAD`** structure.

```
0: kd> !mex.ddt nt!_ETHREAD ffff8b0351756040 Terminated

dt nt!_ETHREAD ffff8b0351756040 Terminated () Recursive: [ -r1 -r2 -r ] Verbose Normal dt
==================================================================================
   +0x510 Terminated           : 0y1 --> Terminated
```

What happens when the **`TerminateThread`** function is called and why is it considered bad?

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/4f62a7e1-f7b3-4f7a-9e65-69cb4421e311)



The **`!locks`** command is used to display information about all the Executive Resource locks that are currently held. This helps in identifying potential deadlock situations. The resource at **`0xffff8b0352014a90`** is exclusively owned and has a thread **`(ffff8b03515d5040)`** waiting for exclusive access. 

```
0: kd> !locks
**** DUMP OF ALL RESOURCE OBJECTS ****
KD: Scanning for held locks...............................................................................................................................................

Resource @ 0xffff8b03516cae70    Shared 1 owning threads
    Contention Count = 39
     Threads: ffff8b034f13e080-01<*> 
KD: Scanning for held locks...............................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................................

Resource @ 0xffff8b035731cea0    Exclusively owned
     Threads: ffff8b034f13e080-01<*> 
KD: Scanning for held locks.....................................................................................................................................................................................................................

Resource @ 0xffff8b0352014a90    Exclusively owned
    Contention Count = 1
    NumberOfExclusiveWaiters = 1
     Threads: ffff8b0351756040-01<*> 

     Threads Waiting On Exclusive Access:
              ffff8b03515d5040       
27708 total locks, 3 locks currently held
```

This thread **`ffff8b03515d5040`** is part of the **System** process and is currently waiting **15 ms** for an ERESOURCE. With all this information, we can confirm that this thread is in a deadlock situation. It is stuck waiting for an ERESOURCE that is exclusively owned by another thread, and there's no indication that this ownership will be released soon. 

```
0: kd> !t ffff8b03515d5040
Process                   Thread                       CID       UserTime KernelTime ContextSwitches Wait Reason Time State
System (ffff8b034a09a040) ffff8b03515d5040 (E|K|W|R|V) 4.2018           0          0               1 WrResource  15ms Waiting

WaitBlockList:
    Object           Type                 Other Waiters
    ffff82871d947360 SynchronizationEvent             0

Thread at address 0000000000060001 is invalid. Not adding to waiter list.
# Child-SP         Return           Call Site                               Info                                                                             Source
0 ffff82871d946f00 fffff8045e40c800 nt!KiSwapContext+0x76                                                                                                    
1 ffff82871d947040 fffff8045e40bd2f nt!KiSwapThread+0x500                                                                                                    
2 ffff82871d9470f0 fffff8045e40b5d3 nt!KiCommitThreadWait+0x14f                                                                                              
3 ffff82871d947190 fffff8045e40e4ad nt!KeWaitForSingleObject+0x233                                                                                           
4 ffff82871d947280 fffff8045e408eee nt!ExpWaitForResource+0x6d                                                                                               
5 ffff82871d947300 fffff804653c1116 nt!ExAcquireResourceExclusiveLite+0x1fe Thread: ffff8b0351756040 (System) exclusive owner of EResource: ffff8b0352014a90 
6 ffff82871d947390 fffff8045e555855 TestDriver!FileCreationThread+0x36                                                                                       C:\Users\Admin\source\repos\TestDriver\TestDriver\Driver.c @ 16
7 ffff82871d947590 fffff8045e5fe808 nt!PspSystemThreadStartup+0x55                                                                                           
8 ffff82871d9475e0 0000000000000000 nt!KiStartSystemThread+0x28
``` 
