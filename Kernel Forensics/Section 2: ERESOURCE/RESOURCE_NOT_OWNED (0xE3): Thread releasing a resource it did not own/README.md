# Summary

The **`RESOURCE_NOT_OWNED`** bug check has a value of **`0x000000E3`**. This indicates that a thread tried to release a resource it did not own.

Link to this CrashDump: https://mega.nz/file/GoslSLob#V332hTfG15UnRpMz1d07LktJiMbz0QKmYAyoOstoTSc

# Code Sample - Thread releasing resource it did not own

The root cause of this code is the incorrect use of the **`ExReleaseResourceLite`** function within the **`FileReadThread`** function. It attempts to release an ERESOURCE object that was never acquired by the thread in the first place. This results to unpredictable behavior, such as a crash.

```c
#include <ntifs.h>
#include <ntstrsafe.h>

// Define a structure to hold our synchronization resources.
typedef struct _SYNC_RESOURCES {
    ERESOURCE FileResource1;    // ERESOURCE object.
} SYNC_RESOURCES, * PSYNC_RESOURCES;

NTSTATUS FileCreationThread(PVOID Context) {
    PSYNC_RESOURCES SyncRes = (PSYNC_RESOURCES)Context;
    UNICODE_STRING uniFilePath;
    WCHAR filePathBuffer[150];
    char dataToWrite[] = "Hello, World!\r\n";
    IO_STATUS_BLOCK ioStatusBlock;

    // Acquire the ERESOURCE in exclusive mode to ensure exclusive access to this section.
    ExAcquireResourceExclusiveLite(&SyncRes->FileResource1, TRUE);

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

    // Release the ERESOURCE.
    ExReleaseResourceLite(&SyncRes->FileResource1);

    return STATUS_SUCCESS;
}

NTSTATUS FileMoveThread(PVOID Context) {
    PSYNC_RESOURCES SyncRes = (PSYNC_RESOURCES)Context;
    IO_STATUS_BLOCK ioStatusBlock;
    WCHAR oldFilePathBuffer[150], newFilePathBuffer[150];

    // Acquire the ERESOURCE in exclusive mode.
    ExAcquireResourceExclusiveLite(&SyncRes->FileResource1, TRUE);

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

    // Release the ERESOURCE.
    ExReleaseResourceLite(&SyncRes->FileResource1);

    return STATUS_SUCCESS;
}

NTSTATUS FileReadThread(PVOID Context) {
    PSYNC_RESOURCES SyncRes = (PSYNC_RESOURCES)Context;
    UNICODE_STRING uniFilePath;
    WCHAR filePathBuffer[150];
    IO_STATUS_BLOCK ioStatusBlock;
    char readBuffer[1024];

    // Release the ERESOURCE without having acquired it
    ExReleaseResourceLite(&SyncRes->FileResource1);

    for (int i = 0; i < 1000; i++) {
        RtlStringCchPrintfW(filePathBuffer, sizeof(filePathBuffer) / sizeof(WCHAR), L"\\DosDevices\\C:\\Temp\\file_%d.txt", i);
        RtlInitUnicodeString(&uniFilePath, filePathBuffer);

        OBJECT_ATTRIBUTES objAttributes;
        InitializeObjectAttributes(&objAttributes, &uniFilePath, OBJ_CASE_INSENSITIVE | OBJ_KERNEL_HANDLE, NULL, NULL);

        HANDLE hFile;
        NTSTATUS status = ZwOpenFile(&hFile, GENERIC_READ, &objAttributes, &ioStatusBlock, FILE_SHARE_READ, FILE_SYNCHRONOUS_IO_NONALERT);

        if (NT_SUCCESS(status)) {
            ZwReadFile(hFile, NULL, NULL, NULL, &ioStatusBlock, readBuffer, sizeof(readBuffer) - 1, NULL, NULL);
            ZwClose(hFile);
        }
    }

    return STATUS_SUCCESS;
}

NTSTATUS FileDeletionThread(PVOID Context) {
    PSYNC_RESOURCES SyncRes = (PSYNC_RESOURCES)Context;
    UNICODE_STRING uniFilePath;
    WCHAR filePathBuffer[150];
    IO_STATUS_BLOCK ioStatusBlock;

    // Acquire the ERESOURCE in exclusive mode.
    ExAcquireResourceExclusiveLite(&SyncRes->FileResource1, TRUE);

    for (int i = 0; i < 1000; i++) {
        RtlStringCchPrintfW(filePathBuffer, sizeof(filePathBuffer) / sizeof(WCHAR), L"\\DosDevices\\C:\\Temp2\\NewFile_%d.txt", i);
        RtlInitUnicodeString(&uniFilePath, filePathBuffer);

        OBJECT_ATTRIBUTES objAttributes;
        InitializeObjectAttributes(&objAttributes, &uniFilePath, OBJ_CASE_INSENSITIVE | OBJ_KERNEL_HANDLE, NULL, NULL);

        HANDLE hFile;
        NTSTATUS status = ZwOpenFile(&hFile, DELETE, &objAttributes, &ioStatusBlock, FILE_SHARE_DELETE, FILE_DELETE_ON_CLOSE);

        if (NT_SUCCESS(status)) {
            ZwClose(hFile);  // This will delete the file due to FILE_DELETE_ON_CLOSE flag.
        }
    }

    // Release the ERESOURCE.
    ExReleaseResourceLite(&SyncRes->FileResource1);

    return STATUS_SUCCESS;
}

VOID UnloadDriver(IN PDRIVER_OBJECT DriverObject) {
    PSYNC_RESOURCES SyncRes = (PSYNC_RESOURCES)DriverObject->DeviceObject->DeviceExtension;

    // Cleanup the ERESOURCE.
    ExDeleteResourceLite(&SyncRes->FileResource1);

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
    ExInitializeResourceLite(&SyncRes->FileResource1);

    HANDLE threadHandle1, threadHandle2, threadHandle3, threadHandle4;

    status = PsCreateSystemThread(&threadHandle1, (ACCESS_MASK)0, NULL, (PHANDLE)0, NULL, FileCreationThread, SyncRes);
    if (NT_SUCCESS(status)) {
        ZwClose(threadHandle1);
    }

    status = PsCreateSystemThread(&threadHandle2, (ACCESS_MASK)0, NULL, (PHANDLE)0, NULL, FileMoveThread, SyncRes);
    if (NT_SUCCESS(status)) {
        ZwClose(threadHandle2);
    }

    status = PsCreateSystemThread(&threadHandle3, (ACCESS_MASK)0, NULL, (PHANDLE)0, NULL, FileReadThread, SyncRes);
    if (NT_SUCCESS(status)) {
        ZwClose(threadHandle3);
    }

    status = PsCreateSystemThread(&threadHandle4, (ACCESS_MASK)0, NULL, (PHANDLE)0, NULL, FileDeletionThread, SyncRes);
    if (NT_SUCCESS(status)) {
        ZwClose(threadHandle4);
    }

    return STATUS_SUCCESS;
}
```

Start loading the kernel driver:

```
sc.exe create TestDrv type= kernel binPath=C:\Users\Admin\source\repos\TestDriver\x64\Release\TestDriver.sys
```

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/3d2fa34e-e433-49f1-9896-db640d81e2d6)


Start the kernel driver:

```
sc.exe start TestDrv
```

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/78b2a4c3-5169-4a85-834e-22d0ba8a3a07)


This kernel driver will bugcheck the system:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/803a6684-1fe2-4b6e-8781-d7f5a4e191d1)


# WinDbg Walk Through - Analyzing CrashDump

Load the CrashDump in WinDbg:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/79530ebe-9822-408f-b799-e6a659b89df3)


Start with loading the **MEX** extension:

```
2: kd> .load mex
Mex External 3.0.0.7172 Loaded!
```

Let's start with the **`!di`** command which stands for dump information. This command retrieves information about the machine from which the memory dump originated. It's especially handy when working with multiple memory dumps, as it helps distinguish the analysis related to each specific dump. This is an improved version of the vertarget command.

```
3: kd> !di
Computer Name:  Not Found
Windows 10 Kernel Version 19041 MP (4 procs) Free x64
Product: WinNt, suite: TerminalServer SingleUserTS
Edition build lab: 19041.1.amd64fre.vb_release.191206-1406
Kernel base = 0xfffff800`51800000 PsLoadedModuleList = 0xfffff800`5242a270
Debug session time: Sun Oct 15 14:10:25.931 2023 (UTC - 7:00)
System Uptime: 0 days 0:01:42.745
SystemManufacturer = Microsoft Corporation
SystemProductName = Virtual Machine
Processor: 11th Gen Intel(R) Core(TM) i7-1185G7 @ 3.00GHz
Bugcheck: E3 (FFFFB587DEF9DC80, FFFFB587DEA4E040, 0, 2)
KernelMode Dump Path: C:\Windows\MEMORY_RESOURCE_NOT_OWNED.DMP
Share Path: \\DESKTOP-OQGRG4S\ADMIN$\MEMORY_RESOURCE_NOT_OWNED.DMP
```

Let's run the **`!analyze -v`** command to quickly triage the issue as it provides a quick and detailed overview of the state of the system at the time of the crash. It automates many aspects of crash dump analysis, providing a quick way to get valuable information.

```
3: kd> !analyze -v
*******************************************************************************
*                                                                             *
*                        Bugcheck Analysis                                    *
*                                                                             *
*******************************************************************************

RESOURCE_NOT_OWNED (e3)
A thread tried to release a resource it did not own.
Arguments:
Arg1: ffffb587def9dc80, Address of resource
Arg2: ffffb587dea4e040, Address of thread
Arg3: 0000000000000000, Address of owner table if there is one
Arg4: 0000000000000002

Debugging Details:
------------------


KEY_VALUES_STRING: 1

    Key  : Analysis.CPU.mSec
    Value: 3218

    Key  : Analysis.Elapsed.mSec
    Value: 3264

    Key  : Analysis.IO.Other.Mb
    Value: 0

    Key  : Analysis.IO.Read.Mb
    Value: 0

    Key  : Analysis.IO.Write.Mb
    Value: 6

    Key  : Analysis.Init.CPU.mSec
    Value: 2280

    Key  : Analysis.Init.Elapsed.mSec
    Value: 334466

    Key  : Analysis.Memory.CommitPeak.Mb
    Value: 147

    Key  : Bugcheck.Code.KiBugCheckData
    Value: 0xe3

    Key  : Bugcheck.Code.LegacyAPI
    Value: 0xe3

    Key  : Failure.Bucket
    Value: 0xE3_TestDriver!FileCreationThread

    Key  : Failure.Hash
    Value: {b41d0cd9-1763-8d45-8d0a-0e6a6af02301}

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


BUGCHECK_CODE:  e3

BUGCHECK_P1: ffffb587def9dc80

BUGCHECK_P2: ffffb587dea4e040

BUGCHECK_P3: 0

BUGCHECK_P4: 2

FILE_IN_CAB:  MEMORY_RESOURCE_NOT_OWNED.DMP

VIRTUAL_MACHINE:  HyperV

BLACKBOXBSD: 1 (!blackboxbsd)


BLACKBOXNTFS: 1 (!blackboxntfs)


BLACKBOXPNP: 1 (!blackboxpnp)


BLACKBOXWINLOGON: 1

PROCESS_NAME:  System

STACK_TEXT:  
fffffc05`3c71f268 fffff800`51c1779b     : 00000000`000000e3 ffffb587`def9dc80 ffffb587`dea4e040 00000000`00000000 : nt!KeBugCheckEx
fffffc05`3c71f270 fffff800`51a08633     : 00000000`000003e7 ffffb587`def9dc80 ffffb587`dea4e040 00000000`00000000 : nt!ExpReleaseResourceSharedForThreadLite+0x20f08b
fffffc05`3c71f330 fffff800`5839128a     : 00000000`00000000 ffffb587`def9dc80 00000000`000003e8 00000000`000003e7 : nt!ExReleaseResourceLite+0xf3
fffffc05`3c71f390 fffff800`51b55855     : ffffffff`80003318 ffffb587`dea4e040 fffff800`58391150 ffffb587`def9dc80 : TestDriver!FileCreationThread+0x13a [C:\Users\Admin\source\repos\TestDriver\TestDriver\Driver.c @ 41] 
fffffc05`3c71f590 fffff800`51bfe808     : ffff8d01`bcbe7180 ffffb587`dea4e040 fffff800`51b55800 fffffc05`3c71f838 : nt!PspSystemThreadStartup+0x55
fffffc05`3c71f5e0 00000000`00000000     : fffffc05`3c720000 fffffc05`3c719000 00000000`00000000 00000000`00000000 : nt!KiStartSystemThread+0x28


FAULTING_SOURCE_LINE:  C:\Users\Admin\source\repos\TestDriver\TestDriver\Driver.c

FAULTING_SOURCE_FILE:  C:\Users\Admin\source\repos\TestDriver\TestDriver\Driver.c

FAULTING_SOURCE_LINE_NUMBER:  41

FAULTING_SOURCE_CODE:  
    37:     // Release the ERESOURCE.
    38:     ExReleaseResourceLite(&SyncRes->FileResource1);
    39: 
    40:     return STATUS_SUCCESS;
>   41: }
    42: 
    43: NTSTATUS FileMoveThread(PVOID Context) {
    44:     PSYNC_RESOURCES SyncRes = (PSYNC_RESOURCES)Context;
    45:     IO_STATUS_BLOCK ioStatusBlock;
    46:     WCHAR oldFilePathBuffer[150], newFilePathBuffer[150];


SYMBOL_NAME:  TestDriver!FileCreationThread+13a

MODULE_NAME: TestDriver

IMAGE_NAME:  TestDriver.sys

STACK_COMMAND:  .cxr; .ecxr ; kb

BUCKET_ID_FUNC_OFFSET:  13a

FAILURE_BUCKET_ID:  0xE3_TestDriver!FileCreationThread

OS_VERSION:  10.0.19041.1

BUILDLAB_STR:  vb_release

OSPLATFORM_TYPE:  x64

OSNAME:  Windows 10

FAILURE_ID_HASH:  {b41d0cd9-1763-8d45-8d0a-0e6a6af02301}

Followup:     MachineOwner
```

Let's look at the call stack a bit closer and see if we can examine something from it. The **`knL`** command in WinDbg is used to display the stack trace for the current thread with detailed frame information. We already know that **TestDriver** is the faulty driver, but what caused the bugcheck?

```
3: kd> knL
  *** Stack trace for last set context - .thread/.cxr resets it
 # Child-SP          RetAddr               Call Site
00 fffffc05`3c71f268 fffff800`51c1779b     nt!KeBugCheckEx
01 fffffc05`3c71f270 fffff800`51a08633     nt!ExpReleaseResourceSharedForThreadLite+0x20f08b
02 (Inline Function) --------`--------     nt!ExpReleaseResourceForThreadLite+0xbb
03 fffffc05`3c71f330 fffff800`5839128a     nt!ExReleaseResourceLite+0xf3
04 fffffc05`3c71f390 fffff800`51b55855     TestDriver+0x128a
05 fffffc05`3c71f590 fffff800`51bfe808     nt!PspSystemThreadStartup+0x55
06 fffffc05`3c71f5e0 00000000`00000000     nt!KiStartSystemThread+0x28
```

The command **`.frame /r 1`** sets the context to the **`1th`** frame in the stack trace and displays the register values at that moment. By examining the register values and the current instruction, we can get insights into what the function is doing at that point.

This is a snapshot of the CPU state at the 1th frame of the stack, which is within the **`nt!ExpReleaseResourceSharedForThreadLite`** function.

```
3: kd> .frame /r 1
01 fffffc05`3c71f270 fffff800`51a08633     nt!ExpReleaseResourceSharedForThreadLite+0x20f08b
rax=0000000000000000 rbx=0000000000000000 rcx=00000000000000e3
rdx=ffffb587def9dc80 rsi=0000000000000000 rdi=ffffb587def9dc80
rip=fffff80051c1779b rsp=fffffc053c71f270 rbp=0000000000000000
 r8=ffffb587dea4e040  r9=0000000000000000 r10=0000000000000000
r11=ffffb587dea4e040 r12=000000000000028e r13=0000000000000000
r14=0000000000000000 r15=fffffc053c71f360
iopl=0         nv up ei pl zr na po nc
cs=0010  ss=0018  ds=002b  es=002b  fs=0053  gs=002b             efl=00040246
nt!ExpReleaseResourceSharedForThreadLite+0x20f08b:
fffff800`51c1779b cc              int     3
```

This **`int 3`** value is interesting since it's used to set a breakpoint or something related to that.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/8fcd7209-ae06-424a-b5ac-f29bc611356f)


Let's try to poke around and look at what this **`RDI`** register is.

```
3: kd> !object ffffb587def9dc80
ffffb587def9dc80: Not a valid object (ObjectType invalid)
```

The output indicates that it's an invalid object, which is interesting. However, when we examine the arguments from the bugcheck, we notice something even more intriguing: the **`Arg1`** memory address overlaps with the registers in the 1st frame.

```
3: kd> !analyze -show
RESOURCE_NOT_OWNED (e3)
A thread tried to release a resource it did not own.
Arguments:
Arg1: ffffb587def9dc80, Address of resource
Arg2: ffffb587dea4e040, Address of thread
Arg3: 0000000000000000, Address of owner table if there is one
Arg4: 0000000000000002
```

The state of this ERESOURCE suggests that it is not currently in use **(Unowned Resource)**, either because it attempts to release an ERESOURCE object that was never acquired by the thread in the first.

```
3: kd> dx -r1 -nv (*((ntkrnlmp!_ERESOURCE *)0xffffb587def9dc80))
(*((ntkrnlmp!_ERESOURCE *)0xffffb587def9dc80))                 : Unowned Resource [Type: _ERESOURCE]
    [+0x000] SystemResourcesList [Type: _LIST_ENTRY]
    [+0x010] OwnerTable       : 0x0 [Type: _OWNER_ENTRY *]
    [+0x018] ActiveCount      : 0 [Type: short]
    [+0x01a] Flag             : 0x0 [Type: unsigned short]
    [+0x01a] ReservedLowFlags : 0x0 [Type: unsigned char]
    [+0x01b] WaiterPriority   : 0x0 [Type: unsigned char]
    [+0x020] SharedWaiters    : 0x0 [Type: void *]
    [+0x028] ExclusiveWaiters : 0x0 [Type: void *]
    [+0x030] OwnerEntry       [Type: _OWNER_ENTRY]
    [+0x040] ActiveEntries    : 0x0 [Type: unsigned long]
    [+0x044] ContentionCount  : 0x1 [Type: unsigned long]
    [+0x048] NumberOfSharedWaiters : 0x0 [Type: unsigned long]
    [+0x04c] NumberOfExclusiveWaiters : 0x0 [Type: unsigned long]
    [+0x050] Reserved2        : 0x0 [Type: void *]
    [+0x058] Address          : 0x0 [Type: void *]
    [+0x058] CreatorBackTraceIndex : 0x0 [Type: unsigned __int64]
    [+0x060] SpinLock         : 0xfffffc053c71f360 [Type: unsigned __int64]
```
