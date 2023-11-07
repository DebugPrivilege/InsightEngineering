# Summary

The **`KERNEL_APC_PENDING_DURING_EXIT`** bug check has a value of **`0x00000020`**. This indicates that an asynchronous procedure call (APC) was still pending when a thread exited.

The key data item is the APC disable count **(Parameter 2)** for the thread. **If the count is nonzero**, it will indicate the source of the problem. 

Link to this CrashDump: https://mega.nz/file/GwUwUD7R#ApTCMpMxCFC1lL85JSEG3aDxiAHv5wGKgHfp65uB0M0

# Code Sample - Thread Starvation by not releasing a Mutex

In order to demonstrate this demo. Please follow the following steps:

- Create the following 2 folders:
  1. **C:\Temp**
  2. **C:\Temp2**

     
```c
#include <ntifs.h>
#include <ntstrsafe.h>

// Define a structure to hold our synchronization resources.
typedef struct _SYNC_RESOURCES {
    KMUTEX FileMutex;  // KMUTEX object for synchronized access to the files.
} SYNC_RESOURCES, * PSYNC_RESOURCES;

NTSTATUS FileCreationThread(PVOID Context) {
    PSYNC_RESOURCES SyncRes = (PSYNC_RESOURCES)Context;
    UNICODE_STRING uniFilePath;
    WCHAR filePathBuffer[150];
    char dataToWrite[] = "Hello, World!\r\n";
    IO_STATUS_BLOCK ioStatusBlock;

    // Acquire the KMUTEX
    KeWaitForSingleObject(&SyncRes->FileMutex, Executive, KernelMode, FALSE, NULL);

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

    // NOT RELEASING MUTEX
    //KeReleaseMutex(&SyncRes->FileMutex, FALSE);

    return STATUS_SUCCESS;
}

NTSTATUS FileMoveThread(PVOID Context) {
    PSYNC_RESOURCES SyncRes = (PSYNC_RESOURCES)Context;
    IO_STATUS_BLOCK ioStatusBlock;
    WCHAR oldFilePathBuffer[150], newFilePathBuffer[150];

    // Acquire the KMUTEX
    KeWaitForSingleObject(&SyncRes->FileMutex, Executive, KernelMode, FALSE, NULL);

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

    // Release the KMUTEX
    KeReleaseMutex(&SyncRes->FileMutex, FALSE);

    return STATUS_SUCCESS;
}

VOID UnloadDriver(IN PDRIVER_OBJECT DriverObject) {
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

    // Initialize the KMUTEX
    KeInitializeMutex(&SyncRes->FileMutex, 0);

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

Start loading this kernel driver:

```
sc.exe create TestDrv type= kernel binPath=C:\Users\User\source\repos\TestDriver\x64\Release\TestDriver.sys
```

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/8669d574-de44-4285-b98c-2fd22321ffcd)

Start the kernel driver:

```
sc.exe start TestDrv
```

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/098dff94-96d1-4ea5-9afe-1d13ef93a7b2)


Here we can see that the machine has experienced a bugcheck:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/154f67ee-9000-4064-8111-f07f8210e4a1)

# WinDbg Walk Through - Analyzing Crash Dump

Load the Crash Dump in WinDbg:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/d8d4dd4b-7a19-477c-bcef-9f593b4081a2)


Start with loading the **MEX** extension:

```
2: kd> .load mex
Mex External 3.0.0.7172 Loaded!
```

Let's start with the **`!di`** command which stands for dump information. This command retrieves information about the machine from which the memory dump originated. It's especially handy when working with multiple memory dumps, as it helps distinguish the analysis related to each specific dump. This is an improved version of the **`vertarget`** command.

```
4: kd> !di
Computer Name:  Not Found
Windows 10 Kernel Version 22621 MP (6 procs) Free x64
Product: WinNt, suite: TerminalServer SingleUserTS
Edition build lab: 22621.2506.amd64fre.ni_release_svc_prod3.231018-1809
Kernel base = 0xfffff806`22c00000 PsLoadedModuleList = 0xfffff806`238134a0
Debug session time: Thu Nov  2 06:04:38.568 2023 (UTC - 7:00)
System Uptime: 0 days 0:13:45.249
SystemManufacturer = Microsoft Corporation
SystemProductName = Virtual Machine
Processor: 11th Gen Intel(R) Core(TM) i7-1185G7 @ 3.00GHz
Bugcheck: 20 (0, FFFF, 0, 1)
KernelMode Dump Path: C:\Windows\MEMORY.DMP
Share Path: \\WINDEV2305EVAL\ADMIN$\MEMORY.DMP
```

Let's run the **`!analyze -v`** command to quickly triage the issue as it provides a quick and detailed overview of the state of the system at the time of the crash. It automates many aspects of crash dump analysis, providing a quick way to get valuable information.

```
4: kd> !analyze -v
*******************************************************************************
*                                                                             *
*                        Bugcheck Analysis                                    *
*                                                                             *
*******************************************************************************

KERNEL_APC_PENDING_DURING_EXIT (20)
The key data item is the thread's APC disable count.
If this is non-zero, then this is the source of the problem.
The APC disable count is decremented each time a driver calls
KeEnterCriticalRegion, FsRtlEnterFileSystem, or acquires a mutex.  The APC
disable count is incremented each time a driver calls KeLeaveCriticalRegion,
FsRtlExitFileSystem, or KeReleaseMutex.  Since these calls should always be in
pairs, this value should be zero when a thread exits.  A negative value
indicates that a driver has disabled APC calls without re-enabling them.  A
positive value indicates that the reverse is true.
If you ever see this error, be very suspicious of all drivers installed on the
machine -- especially unusual or non-standard drivers.  Third party file
system redirectors are especially suspicious since they do not generally
receive the heavy duty testing that NTFS, FAT, RDR, etc receive.
This current IRQL should also be 0.  If it is not, that a driver's
cancellation routine can cause this BugCheck by returning at an elevated
IRQL.  Always attempt to note what you were doing/closing at the
time of the crash, and note all of the installed drivers at the time of
the crash.  This symptom is usually a severe bug in a third party
driver.
Arguments:
Arg1: 0000000000000000, The address of the APC found pending during exit.
Arg2: 000000000000ffff, The thread's APC disable count
Arg3: 0000000000000000, The current IRQL
Arg4: 0000000000000001

Debugging Details:
------------------


KEY_VALUES_STRING: 1

    Key  : Analysis.CPU.mSec
    Value: 3702

    Key  : Analysis.Elapsed.mSec
    Value: 4094

    Key  : Analysis.IO.Other.Mb
    Value: 1

    Key  : Analysis.IO.Read.Mb
    Value: 0

    Key  : Analysis.IO.Write.Mb
    Value: 8

    Key  : Analysis.Init.CPU.mSec
    Value: 3499

    Key  : Analysis.Init.Elapsed.mSec
    Value: 397593

    Key  : Analysis.Memory.CommitPeak.Mb
    Value: 143

    Key  : Bugcheck.Code.KiBugCheckData
    Value: 0x20

    Key  : Bugcheck.Code.LegacyAPI
    Value: 0x20

    Key  : Dump.Attributes.AsUlong
    Value: 800

    Key  : Failure.Bucket
    Value: 0x20_NULLAPC_KAPC_NEGATIVE_nt!PspExitThread

    Key  : Failure.Hash
    Value: {fcbbc787-b7cc-0ca4-c274-c788b86a9951}

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


BUGCHECK_CODE:  20

BUGCHECK_P1: 0

BUGCHECK_P2: ffff

BUGCHECK_P3: 0

BUGCHECK_P4: 1

FILE_IN_CAB:  MEMORY.DMP

VIRTUAL_MACHINE:  HyperV

DUMP_FILE_ATTRIBUTES: 0x800

BLACKBOXBSD: 1 (!blackboxbsd)


BLACKBOXNTFS: 1 (!blackboxntfs)


BLACKBOXPNP: 1 (!blackboxpnp)


BLACKBOXWINLOGON: 1

PROCESS_NAME:  System

STACK_TEXT:  
ffffd98c`3ff17428 fffff806`234b36a8     : 00000000`00000020 00000000`00000000 00000000`0000ffff 00000000`00000000 : nt!KeBugCheckEx
ffffd98c`3ff17430 fffff806`232b9703     : 00000161`00000000 00000000`00000000 00000000`00000000 00000000`00000000 : nt!PspExitThread+0x1f8f14
ffffd98c`3ff17530 fffff806`22f573f1     : ffffdd07`b96c3040 00000000`00000080 fffff806`4a1d10e0 00000000`00000000 : nt!PspTerminateThreadByPointer+0x53
ffffd98c`3ff17570 fffff806`2301b8d4     : ffffb601`40ddb180 ffffdd07`b96c3040 fffff806`22f57380 00000000`00000246 : nt!PspSystemThreadStartup+0x71
ffffd98c`3ff175c0 00000000`00000000     : ffffd98c`3ff18000 ffffd98c`3ff11000 00000000`00000000 00000000`00000000 : nt!KiStartSystemThread+0x34


SYMBOL_NAME:  nt!PspExitThread+1f8f14

MODULE_NAME: nt

IMAGE_NAME:  ntkrnlmp.exe

STACK_COMMAND:  .cxr; .ecxr ; kb

BUCKET_ID_FUNC_OFFSET:  1f8f14

FAILURE_BUCKET_ID:  0x20_NULLAPC_KAPC_NEGATIVE_nt!PspExitThread

OS_VERSION:  10.0.22621.2506

BUILDLAB_STR:  ni_release_svc_prod3

OSPLATFORM_TYPE:  x64

OSNAME:  Windows 10

FAILURE_ID_HASH:  {fcbbc787-b7cc-0ca4-c274-c788b86a9951}

Followup:     MachineOwner
---------
```
