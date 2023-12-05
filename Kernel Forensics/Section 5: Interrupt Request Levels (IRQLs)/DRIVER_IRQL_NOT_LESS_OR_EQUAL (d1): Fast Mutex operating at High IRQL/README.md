# Description

**`DRIVER_IRQL_NOT_LESS_OR_EQUAL`** is a bugcheck with a stop code of **`0xD1`**. This type of bugcheck is typically caused by a driver attempting to access an invalid or pageable memory address at an IRQL that is too high.

Link to this CrashDump: https://mega.nz/file/XlEFGKSC#RPB67F0AesJBfkRWQlnbmOGHamhpFS2Jbu2kJ9E319I

# Code Sample - Fast Mutex operating at Higher IRQL

According to the Windows kernel rules, fast mutexes should only be used when the IRQL is **`APC_LEVEL`** or lower. Since **`DISPATCH_LEVEL`** is higher than **`APC_LEVEL`**, attempting to acquire a fast mutex at this level violates these rules and leads to a bugcheck.

This is documented by Microsoft here:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/ce5d45f3-5f7a-4963-9705-b807f5373a13)


```c
#include <ntifs.h>
#include <ntstrsafe.h>

// Structure to hold synchronization resources.
typedef struct _SYNC_RESOURCES {
    FAST_MUTEX FileMutex;  // FAST_MUTEX object for synchronized access to files.
} SYNC_RESOURCES, * PSYNC_RESOURCES;

// Function to log the current IRQL level to a file.
NTSTATUS LogCurrentIrql(const WCHAR* logFilePath) {
    UNICODE_STRING uniLogFilePath;
    RtlInitUnicodeString(&uniLogFilePath, logFilePath); // Initialize UNICODE_STRING for file path.

    OBJECT_ATTRIBUTES objAttributes;
    InitializeObjectAttributes(&objAttributes, &uniLogFilePath, OBJ_CASE_INSENSITIVE | OBJ_KERNEL_HANDLE, NULL, NULL); // Set up object attributes.

    HANDLE logFile;
    IO_STATUS_BLOCK ioStatusBlock;
    // Create or open a file to write the IRQL level.
    NTSTATUS status = ZwCreateFile(&logFile, GENERIC_WRITE, &objAttributes, &ioStatusBlock, NULL, FILE_ATTRIBUTE_NORMAL, 0, FILE_OVERWRITE_IF, FILE_SYNCHRONOUS_IO_NONALERT, NULL, 0);
    if (!NT_SUCCESS(status)) {
        return status;
    }

    KIRQL currentIrql = KeGetCurrentIrql(); // Get the current IRQL.
    WCHAR irqlName[30];
    // Determine the name of the current IRQL and copy it to the buffer.
    switch (currentIrql) {
    case PASSIVE_LEVEL:
        wcscpy_s(irqlName, 30, L"PASSIVE_LEVEL");
        break;
    case APC_LEVEL:
        wcscpy_s(irqlName, 30, L"APC_LEVEL");
        break;
    case DISPATCH_LEVEL:
        wcscpy_s(irqlName, 30, L"DISPATCH_LEVEL");
        break;
    default:
        swprintf_s(irqlName, 30, L"UNKNOWN_LEVEL (%u)", currentIrql);
        break;
    }

    // Write the IRQL name to the file.
    ZwWriteFile(logFile, NULL, NULL, NULL, &ioStatusBlock, irqlName, wcslen(irqlName) * sizeof(WCHAR), NULL, NULL);
    ZwClose(logFile); // Close the file handle.

    return STATUS_SUCCESS;
}

NTSTATUS FileCreationThread(PVOID Context) {
    PSYNC_RESOURCES SyncRes = (PSYNC_RESOURCES)Context;
    UNICODE_STRING uniFilePath;
    WCHAR filePathBuffer[150];
    char dataToWrite[] = "Hello, World!\r\n";
    IO_STATUS_BLOCK ioStatusBlock;
    KIRQL oldIrql;

    // Raise the IRQL to DISPATCH_LEVEL.
    KeRaiseIrql(DISPATCH_LEVEL, &oldIrql);

    // Log the current IRQL level to a file.
    LogCurrentIrql(L"\\DosDevices\\C:\\Temp\\FileCreationThreadIrql.txt");

    ExAcquireFastMutex(&SyncRes->FileMutex); // Acquire the fast mutex.

    // Create and write to multiple files in a loop.
    for (int i = 0; i < 1000; i++) {
        RtlStringCchPrintfW(filePathBuffer, sizeof(filePathBuffer) / sizeof(WCHAR), L"\\DosDevices\\C:\\Temp\\file_%d.txt", i);
        RtlInitUnicodeString(&uniFilePath, filePathBuffer);

        OBJECT_ATTRIBUTES objAttributes;
        InitializeObjectAttributes(&objAttributes, &uniFilePath, OBJ_CASE_INSENSITIVE | OBJ_KERNEL_HANDLE, NULL, NULL);

        HANDLE hFile;
        // Create or open the file.
        NTSTATUS status = ZwCreateFile(&hFile, GENERIC_WRITE, &objAttributes, &ioStatusBlock, NULL, FILE_ATTRIBUTE_NORMAL, 0, FILE_OVERWRITE_IF, FILE_SYNCHRONOUS_IO_NONALERT, NULL, 0);

        // Write data to the file.
        if (NT_SUCCESS(status)) {
            for (int j = 0; j < 100; j++) {
                ZwWriteFile(hFile, NULL, NULL, NULL, &ioStatusBlock, dataToWrite, sizeof(dataToWrite) - 1, NULL, NULL);
            }
            ZwClose(hFile); // Close the file handle.
        }
    }

    ExReleaseFastMutex(&SyncRes->FileMutex); // Release the fast mutex.

    return STATUS_SUCCESS;
}

// Thread function for moving files.
NTSTATUS FileMoveThread(PVOID Context) {
    PSYNC_RESOURCES SyncRes = (PSYNC_RESOURCES)Context;
    IO_STATUS_BLOCK ioStatusBlock;
    WCHAR oldFilePathBuffer[150], newFilePathBuffer[150];

    // Log the current IRQL level to a file.
    LogCurrentIrql(L"\\DosDevices\\C:\\Temp\\FileMoveThreadIrql.txt");

    ExAcquireFastMutex(&SyncRes->FileMutex); // Acquire the fast mutex.

    // Move files in a loop.
    for (int i = 0; i < 1000; i++) {
        RtlStringCchPrintfW(oldFilePathBuffer, sizeof(oldFilePathBuffer) / sizeof(WCHAR), L"\\DosDevices\\C:\\Temp\\file_%d.txt", i);
        RtlStringCchPrintfW(newFilePathBuffer, sizeof(newFilePathBuffer) / sizeof(WCHAR), L"\\DosDevices\\C:\\Temp2\\NewFile_%d.txt", i);

        UNICODE_STRING oldFilePath, newFilePath;
        RtlInitUnicodeString(&oldFilePath, oldFilePathBuffer);
        RtlInitUnicodeString(&newFilePath, newFilePathBuffer);

        OBJECT_ATTRIBUTES objAttributes;
        InitializeObjectAttributes(&objAttributes, &oldFilePath, OBJ_CASE_INSENSITIVE | OBJ_KERNEL_HANDLE, NULL, NULL);

        HANDLE hFile;
        // Open the file to be moved.
        NTSTATUS status = ZwOpenFile(&hFile, FILE_ALL_ACCESS, &objAttributes, &ioStatusBlock, FILE_SHARE_READ | FILE_SHARE_WRITE, FILE_SYNCHRONOUS_IO_NONALERT);

        // Rename the file.
        if (NT_SUCCESS(status)) {
            FILE_RENAME_INFORMATION renameInfo;
            RtlZeroMemory(&renameInfo, sizeof(FILE_RENAME_INFORMATION));
            renameInfo.ReplaceIfExists = TRUE;
            renameInfo.RootDirectory = NULL;
            renameInfo.FileNameLength = wcslen(newFilePathBuffer) * sizeof(WCHAR);
            RtlCopyMemory(renameInfo.FileName, newFilePathBuffer, renameInfo.FileNameLength);

            ZwSetInformationFile(hFile, &ioStatusBlock, &renameInfo, sizeof(FILE_RENAME_INFORMATION) + renameInfo.FileNameLength, FileRenameInformation);
            ZwClose(hFile); // Close the file handle.
        }
    }

    ExReleaseFastMutex(&SyncRes->FileMutex); // Release the fast mutex.

    return STATUS_SUCCESS;
}

// Driver unload routine.
VOID UnloadDriver(IN PDRIVER_OBJECT DriverObject) {
    IoDeleteDevice(DriverObject->DeviceObject); // Delete the device object.
}

// Driver entry point.
NTSTATUS DriverEntry(PDRIVER_OBJECT DriverObject, PUNICODE_STRING RegistryPath) {
    DriverObject->DriverUnload = UnloadDriver; // Set the unload routine.

    PDEVICE_OBJECT DeviceObject = NULL;
    // Create a device object.
    NTSTATUS status = IoCreateDevice(DriverObject, sizeof(SYNC_RESOURCES), NULL, FILE_DEVICE_UNKNOWN, FILE_DEVICE_SECURE_OPEN, FALSE, &DeviceObject);

    if (!NT_SUCCESS(status)) {
        return status;
    }

    PSYNC_RESOURCES SyncRes = (PSYNC_RESOURCES)DeviceObject->DeviceExtension;
    ExInitializeFastMutex(&SyncRes->FileMutex); // Initialize the fast mutex.

    HANDLE threadHandle1, threadHandle2;
    // Create two threads for file creation and moving.
    status = PsCreateSystemThread(&threadHandle1, (ACCESS_MASK)0, NULL, (PHANDLE)0, NULL, FileCreationThread, SyncRes);
    if (NT_SUCCESS(status)) {
        ZwClose(threadHandle1); // Close the thread handle.
    }

    status = PsCreateSystemThread(&threadHandle2, (ACCESS_MASK)0, NULL, (PHANDLE)0, NULL, FileMoveThread, SyncRes);
    if (NT_SUCCESS(status)) {
        ZwClose(threadHandle2); // Close the thread handle.
    }

    return STATUS_SUCCESS;
}
```

Start loading this kernel driver:

```
sc.exe create TestDrv type= kernel binPath=C:\Users\User\source\repos\TestDriver\x64\Release\TestDriver.sys
```

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/9e9391f4-c533-4b55-8e5a-00e528291eb9)


Start the kernel driver:

```
sc.exe start TestDrv
```

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/fffc0fb3-0773-4f07-8371-477c074d1be8)


Here we can see that our machine has bugchecked:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/02698224-5d44-4b93-b84e-31e72d4d29b0)


# WinDbg Walk Through - Analysis

Load the Memory Dump in WinDbg:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/a8252068-4d18-408c-a53e-0e806aca561b)


Let's start with the **`!mex.di`** command which stands for dump information. This command retrieves information about the machine from which the memory dump originated. It's especially handy when working with multiple memory dumps, as it helps distinguish the analysis related to each specific dump. This is an improved version of the **`vertarget`** command.

```
4: kd> !mex.di
Computer Name:  Not Found
Windows 10 Kernel Version 22621 MP (6 procs) Free x64
Product: WinNt, suite: TerminalServer SingleUserTS
Edition build lab: 22621.2506.amd64fre.ni_release_svc_prod3.231018-1809
Kernel base = 0xfffff802`04000000 PsLoadedModuleList = 0xfffff802`04c134a0
Debug session time: Mon Dec  4 06:55:45.455 2023 (UTC - 8:00)
System Uptime: 0 days 0:07:30.911
SystemManufacturer = Microsoft Corporation
SystemProductName = Virtual Machine
Processor: 11th Gen Intel(R) Core(TM) i7-1185G7 @ 3.00GHz
Bugcheck: D1 (FFFF80091002AC08, 2, 1, FFFFF802084D9062)
```

Let's run the **`!analyze -v`** command to quickly triage the issue as it provides a quick and detailed overview of the state of the system at the time of the crash. It automates many aspects of crash dump analysis, providing a quick way to get valuable information.

```
4: kd> !analyze -v
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
Arg1: ffff80091002ac08, memory referenced
Arg2: 0000000000000002, IRQL
Arg3: 0000000000000001, value 0 = read operation, 1 = write operation
Arg4: fffff802084d9062, address which referenced memory

Debugging Details:
------------------


KEY_VALUES_STRING: 1

    Key  : Analysis.CPU.mSec
    Value: 3406

    Key  : Analysis.Elapsed.mSec
    Value: 4750

    Key  : Analysis.IO.Other.Mb
    Value: 0

    Key  : Analysis.IO.Read.Mb
    Value: 1

    Key  : Analysis.IO.Write.Mb
    Value: 6

    Key  : Analysis.Init.CPU.mSec
    Value: 3046

    Key  : Analysis.Init.Elapsed.mSec
    Value: 159347

    Key  : Analysis.Memory.CommitPeak.Mb
    Value: 164

    Key  : Bugcheck.Code.KiBugCheckData
    Value: 0xd1

    Key  : Bugcheck.Code.LegacyAPI
    Value: 0xd1

    Key  : Dump.Attributes.AsUlong
    Value: 800

    Key  : Failure.Bucket
    Value: AV_fileinfo!FIPfFileFSOpPostCreate

    Key  : Failure.Hash
    Value: {b4048e5d-9388-9a34-9958-ec36d87bbe12}

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

BUGCHECK_P1: ffff80091002ac08

BUGCHECK_P2: 2

BUGCHECK_P3: 1

BUGCHECK_P4: fffff802084d9062

FILE_IN_CAB:  MEMORY.DMP

VIRTUAL_MACHINE:  HyperV

DUMP_FILE_ATTRIBUTES: 0x800

WRITE_ADDRESS:  ffff80091002ac08 

BLACKBOXBSD: 1 (!blackboxbsd)


BLACKBOXNTFS: 1 (!blackboxntfs)


BLACKBOXPNP: 1 (!blackboxpnp)


BLACKBOXWINLOGON: 1

PROCESS_NAME:  System

TRAP_FRAME:  ffffed0ee77dce90 -- (.trap 0xffffed0ee77dce90)
NOTE: The trap frame does not contain all registers.
Some register values may be zeroed or incorrect.
rax=00000006991b441c rbx=0000000000000000 rcx=0b4df3b13fee0000
rdx=0000000000000000 rsi=0000000000000000 rdi=0000000000000000
rip=fffff802084d9062 rsp=ffffed0ee77dd020 rbp=ffffed0ee77ddb39
 r8=0000000000000000  r9=7ffffffffffffffc r10=ffffb5076d140d70
r11=0000000000000001 r12=0000000000000000 r13=0000000000000000
r14=0000000000000000 r15=0000000000000000
iopl=0         nv up ei ng nz na pe nc
Ntfs!NtfsChangeAttributeValue+0x392:
fffff802`084d9062 49894708        mov     qword ptr [r15+8],rax ds:00000000`00000008=????????????????
Resetting default scope

STACK_TEXT:  
ffffed0e`e77dcd48 fffff802`0442bfa9     : 00000000`0000000a ffff8009`1002ac08 00000000`00000002 00000000`00000001 : nt!KeBugCheckEx
ffffed0e`e77dcd50 fffff802`04427634     : 00000000`00000007 ffffed0e`e77dcf68 ffffb507`72616018 ffffb507`7276b500 : nt!KiBugCheckDispatch+0x69
ffffed0e`e77dce90 fffff802`084d9062     : 00000000`00000000 00000000`0fb6b000 ffffb507`00000400 ffff8009`1002ac88 : nt!KiPageFault+0x474
ffffed0e`e77dd020 fffff802`0848980c     : 00000000`00000000 ffffb507`00000000 00000000`00000000 ffff8009`1002aca8 : Ntfs!NtfsChangeAttributeValue+0x392
ffffed0e`e77dd210 fffff802`08467ce3     : ffffb507`72616018 ffffc708`6dff6a40 ffffed0e`e77ddb39 fffff802`08475301 : Ntfs!NtfsAllocateRecord+0x42c
ffffed0e`e77dd390 fffff802`084660ee     : ffffb507`72616018 ffffc708`6dff6800 ffffed0e`e77dd880 ffffc708`6dff6800 : Ntfs!GetIndexBuffer+0x6f
ffffed0e`e77dd460 fffff802`084cc842     : 00000000`00000000 fffff802`08472973 ffffc708`6dff6800 ffffed0e`e77dd840 : Ntfs!PushIndexRoot+0x2ee
ffffed0e`e77dd660 fffff802`084cc489     : 00000000`00010000 ffffc708`6dff6800 ffffed0e`e77dd7b0 ffffed0e`e77dd880 : Ntfs!AddToIndex+0x25e
ffffed0e`e77dd720 fffff802`084b2ca3     : ffffc708`00000000 ffffed0e`e77dd9d0 00080000`eec06801 ffffc708`00000018 : Ntfs!NtfsAddIndexEntry+0x165
ffffed0e`e77dd990 fffff802`084cbdf6     : ffffb507`72616018 ffffc708`6dff6800 ffffc708`65115560 00000000`00000000 : Ntfs!NtfsAddDosOnlyName+0x113
ffffed0e`e77dda60 fffff802`084877f4     : 00000000`00000002 ffffc708`651156c0 ffffc708`651159c0 00000000`00000000 : Ntfs!NtfsAddLink+0x28a
ffffed0e`e77ddb80 fffff802`08460bc3     : ffffb507`72616018 ffffc708`6dff68b0 ffffed0e`e77de330 ffffb507`72616018 : Ntfs!NtfsCreateNewFile+0x1db4
ffffed0e`e77ddf60 fffff802`0845af00     : 00000000`00000000 ffffed0e`e77de376 ffffed0e`e77de3e8 ffffb507`6d202da0 : Ntfs!NtfsCommonCreate+0x2393
ffffed0e`e77de220 fffff802`042ec015     : ffffb507`6d32a030 ffffb507`72ddaa20 00000000`00000000 fffff802`042818a3 : Ntfs!NtfsFsdCreate+0x200
ffffed0e`e77de4a0 fffff802`0754a1db     : 00000000`00000000 ffffb507`71fe64e0 ffffb507`71fe6590 fffff802`07549128 : nt!IofCallDriver+0x55
ffffed0e`e77de4e0 fffff802`0758a0ba     : ffffed0e`e77de580 ffffed0e`e77de5b0 ffffb507`71fe6500 00000000`00000000 : FLTMGR!FltpLegacyProcessingAfterPreCallbacksCompleted+0x15b
ffffed0e`e77de550 fffff802`0758a13c     : 00000000`00000000 ffffb507`71fe65c8 ffffed0e`e77de710 ffffb507`6d1b5d50 : FLTMGR!FltpReissueSynchronousIo+0x1b6
ffffed0e`e77de5d0 fffff802`0827e75f     : 00000000`00000000 00000000`00000000 00000000`00000000 ffffed0e`e77de710 : FLTMGR!FltReissueSynchronousIo+0xc
ffffed0e`e77de600 fffff802`0827e16f     : ffffb507`6de940f8 00000000`00000001 ffffb507`6d1b5d50 00000000`00000000 : fileinfo!FIPfFileFSOpPostCreate+0x2ff
ffffed0e`e77de6c0 fffff802`07548cb5     : ffffb507`6d329c28 ffffb507`6de940f8 fffe0000`00000000 00000040`1fb1fea7 : fileinfo!FIPostCreateCallback+0x1cf
ffffed0e`e77de770 fffff802`075485c0     : ffffb507`6de94000 00000000`00000000 00000000`00000000 00000000`00000000 : FLTMGR!FltpPerformPostCallbacksWorker+0x345
ffffed0e`e77de840 fffff802`0754a273     : ffffed0e`e77d9000 ffffed0e`e77e0000 ffffb507`6de94010 fffff802`075490a1 : FLTMGR!FltpPassThroughCompletionWorker+0x120
ffffed0e`e77de8f0 fffff802`07581f83     : ffffed0e`e77de9a0 ffffed0e`e77d9000 ffffb507`6d202d00 fffff802`046f8ce3 : FLTMGR!FltpLegacyProcessingAfterPreCallbacksCompleted+0x1f3
ffffed0e`e77de960 fffff802`042ec015     : ffffb507`6e9c0700 ffffb507`6d1868f0 00000000`00000000 00000000`00000000 : FLTMGR!FltpCreate+0x323
ffffed0e`e77dea10 fffff802`046f754e     : ffffb507`6d1868f0 ffffed0e`e77ded20 ffffb507`72ddaa20 ffffb507`71b09c00 : nt!IofCallDriver+0x55
ffffed0e`e77dea50 fffff802`046f1f61     : ffffed0e`e77dee48 ffffc708`6124aca0 ffffed0e`e77ded50 ffffed0e`e77dee40 : nt!IopParseDevice+0x8be
ffffed0e`e77dec20 fffff802`046f1232     : ffffb507`72769001 ffffed0e`e77dee40 00000000`00000240 ffffb507`6a4fdae0 : nt!ObpLookupObjectName+0x7e1
ffffed0e`e77dedb0 fffff802`046eecc1     : 00000000`00000000 ffffb507`6e9c0730 ffffed0e`e77df2c8 ffffb507`6e9c0748 : nt!ObOpenObjectByNameEx+0x1f2
ffffed0e`e77deee0 fffff802`046ee5b9     : ffffed0e`e77df2a0 00000000`00000000 ffffed0e`e77df2c8 ffffed0e`e77df2b8 : nt!IopCreateFile+0x431
ffffed0e`e77defa0 fffff802`0442b6e5     : ffffb507`6ed12080 ffffb507`6ed12378 ffffb507`714b90c0 fffff802`007e3180 : nt!NtCreateFile+0x79
ffffed0e`e77df030 fffff802`0441c210     : fffff802`25da14b2 00000000`00000000 00000000`00000000 ffffb507`6ed12080 : nt!KiSystemServiceCopyEnd+0x25
ffffed0e`e77df238 fffff802`25da14b2     : 00000000`00000000 00000000`00000000 ffffb507`6ed12080 00000000`00000001 : nt!KiServiceLinkage
ffffed0e`e77df240 fffff802`25da1142     : 00000000`00000000 ffffb507`729e3040 00000000`00000000 00000000`00000000 : TestDriver!LogCurrentIrql+0x92 [C:\Users\User\source\repos\TestDriver\TestDriver\Driver.c @ 21] 
ffffed0e`e77df350 fffff802`04307287     : ffffb507`729e3040 ffffb507`729e3040 fffff802`25da10f0 ffffb507`71e13f20 : TestDriver!FileCreationThread+0x52 [C:\Users\User\source\repos\TestDriver\TestDriver\Driver.c @ 64] 
ffffed0e`e77df570 fffff802`0441b8e4     : ffffa300`54ea8180 ffffb507`729e3040 fffff802`04307230 00000000`00000246 : nt!PspSystemThreadStartup+0x57
ffffed0e`e77df5c0 00000000`00000000     : ffffed0e`e77e0000 ffffed0e`e77d9000 00000000`00000000 00000000`00000000 : nt!KiStartSystemThread+0x34


SYMBOL_NAME:  fileinfo!FIPfFileFSOpPostCreate+2ff

MODULE_NAME: fileinfo

IMAGE_NAME:  fileinfo.sys

STACK_COMMAND:  .cxr; .ecxr ; kb

BUCKET_ID_FUNC_OFFSET:  2ff

FAILURE_BUCKET_ID:  AV_fileinfo!FIPfFileFSOpPostCreate

OS_VERSION:  10.0.22621.2506

BUILDLAB_STR:  ni_release_svc_prod3

OSPLATFORM_TYPE:  x64

OSNAME:  Windows 10

FAILURE_ID_HASH:  {b4048e5d-9388-9a34-9958-ec36d87bbe12}

Followup:     MachineOwner
---------
```

From the provided **`!analyze -v`** output, the crash occurred because the driver attempted to perform a write operation to a pageable memory address at **`DISPATCH_LEVEL`** IRQL. 

```
4: kd> !analyze -show
DRIVER_IRQL_NOT_LESS_OR_EQUAL (d1)
An attempt was made to access a pageable (or completely invalid) address at an
interrupt request level (IRQL) that is too high.  This is usually
caused by drivers using improper addresses.
If kernel debugger is available get stack backtrace.
Arguments:
Arg1: ffff80091002ac08, memory referenced
Arg2: 0000000000000002, IRQL
Arg3: 0000000000000001, value 0 = read operation, 1 = write operation
Arg4: fffff802084d9062, address which referenced memory
```

The output from the **`!pool`** command is used to analyze memory pool issues. This output indicates that the memory address **`ffff80091002ac08`** is not part of the recognized kernel memory pool regions. It could potentially be a freed, invalid, or paged-out memory address.

```
4: kd> !pool ffff80091002ac08
Pool page ffff80091002ac08 region is Unknown
ffff80091002a000 is not a valid large pool allocation, checking large session pool...
Unable to read large session pool table (Session data is not present in mini and kernel-only dumps)
ffff80091002a000 is not valid pool. Checking for freed (or corrupt) pool
Address ffff80091002a000 could not be read. It may be a freed, invalid or paged out page
```

The **`!pte`** command output is indicating issues with the memory mapping of the address **`ffff80091002ac08`**

```
4: kd> !pte ffff80091002ac08
                                           VA ffff80091002ac08
PXE at FFFFD7EBF5FAF800    PPE at FFFFD7EBF5F00120    PDE at FFFFD7EBE0024400    PTE at FFFFD7C004880150
Page 19e511 not present in the dump file. Type ".hh dbgerr004" for details
Unable to get PXE FFFFD7EBF5F00120
```

Here are some documentation from Microsoft on what we could do next:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/c2d6382d-1739-4630-8f35-0fec9fa0c00c)


The disassembled output doesn't really tell us much.

```
0: kd> ub fffff802084d9062
Ntfs!NtfsChangeAttributeValue+0x363:
fffff802`084d9033 48897c2420      mov     qword ptr [rsp+20h],rdi
fffff802`084d9038 41b907000000    mov     r9d,7
fffff802`084d903e 4c8b8424e8000000 mov     r8,qword ptr [rsp+0E8h]
fffff802`084d9046 4d8b00          mov     r8,qword ptr [r8]
fffff802`084d9049 488b5130        mov     rdx,qword ptr [rcx+30h]
fffff802`084d904d 488b8c24d0000000 mov     rcx,qword ptr [rsp+0D0h]
fffff802`084d9055 e8c6daffff      call    Ntfs!NtfsWriteLog (fffff802`084d6b20)
fffff802`084d905a 4c8bbc24a8000000 mov     r15,qword ptr [rsp+0A8h]
```

Let's take a look at the crashing thread again, the crash seems to be related to a file operation handle by a NTFS driver from the first sight. However, the presence of **`TestDriver`** in the call stack just before the crash suggests that a third-party driver's operations at an elevated IRQL.

```
0: kd> !mex.t ffffb507729e3040
Process                   Thread                       CID       UserTime KernelTime ContextSwitches Wait Reason Time State
System (ffffb5076a4eb040) ffffb507729e3040 (E|K|W|R|V) 4.26ec           0          0               3 WrResource  15ms Running on processor 4

Irp List:
    IRP              File                             Driver
    ffffb50772ddaa20 \Temp\FileCreationThreadIrql.txt Ntfs

 # Child-SP         Return           Call Site                                                   Source
 0 ffffed0ee77dcd48 fffff8020442bfa9 nt!KeBugCheckEx                                             
 1 ffffed0ee77dcd50 fffff80204427634 nt!KiBugCheckDispatch+0x69                                  
 2 ffffed0ee77dce90 fffff802084d9062 nt!KiPageFault+0x474                                        
 3 ffffed0ee77dd020 fffff8020848980c Ntfs!NtfsChangeAttributeValue+0x392                         
 4 ffffed0ee77dd210 fffff80208467ce3 Ntfs!NtfsAllocateRecord+0x42c                               
 5 ffffed0ee77dd390 fffff802084660ee Ntfs!GetIndexBuffer+0x6f                                    
 6 ffffed0ee77dd460 fffff802084cc842 Ntfs!PushIndexRoot+0x2ee                                    
 7 ffffed0ee77dd660 fffff802084cc489 Ntfs!AddToIndex+0x25e                                       
 8 ffffed0ee77dd720 fffff802084b2ca3 Ntfs!NtfsAddIndexEntry+0x165                                
 9 ffffed0ee77dd990 fffff802084cbdf6 Ntfs!NtfsAddDosOnlyName+0x113                               
 a ffffed0ee77dda60 fffff802084877f4 Ntfs!NtfsAddLink+0x28a                                      
 b ffffed0ee77ddb80 fffff80208460bc3 Ntfs!NtfsCreateNewFile+0x1db4                               
 c ffffed0ee77ddf60 fffff8020845af00 Ntfs!NtfsCommonCreate+0x2393                                
 d ffffed0ee77de220 fffff802042ec015 Ntfs!NtfsFsdCreate+0x200                                    
 e ffffed0ee77de4a0 fffff8020754a1db nt!IofCallDriver+0x55                                       
 f ffffed0ee77de4e0 fffff8020758a0ba FLTMGR!FltpLegacyProcessingAfterPreCallbacksCompleted+0x15b 
10 ffffed0ee77de550 fffff8020758a13c FLTMGR!FltpReissueSynchronousIo+0x1b6                       
11 ffffed0ee77de5d0 fffff8020827e75f FLTMGR!FltReissueSynchronousIo+0xc                          
12 ffffed0ee77de600 fffff8020827e16f fileinfo!FIPfFileFSOpPostCreate+0x2ff                       
13 ffffed0ee77de6c0 fffff80207548cb5 fileinfo!FIPostCreateCallback+0x1cf                         
14 ffffed0ee77de770 fffff802075485c0 FLTMGR!FltpPerformPostCallbacksWorker+0x345                 
15 ffffed0ee77de840 fffff8020754a273 FLTMGR!FltpPassThroughCompletionWorker+0x120                
16 ffffed0ee77de8f0 fffff80207581f83 FLTMGR!FltpLegacyProcessingAfterPreCallbacksCompleted+0x1f3 
17 ffffed0ee77de960 fffff802042ec015 FLTMGR!FltpCreate+0x323                                     
18 ffffed0ee77dea10 fffff802046f754e nt!IofCallDriver+0x55                                       
19 ffffed0ee77dea50 fffff802046f1f61 nt!IopParseDevice+0x8be                                     
1a ffffed0ee77dec20 fffff802046f1232 nt!ObpLookupObjectName+0x7e1                                
1b ffffed0ee77dedb0 fffff802046eecc1 nt!ObOpenObjectByNameEx+0x1f2                               
1c ffffed0ee77deee0 fffff802046ee5b9 nt!IopCreateFile+0x431                                      
1d ffffed0ee77defa0 fffff8020442b6e5 nt!NtCreateFile+0x79                                        
1e ffffed0ee77df030 fffff8020441c210 nt!KiSystemServiceCopyEnd+0x25                              
1f ffffed0ee77df238 fffff80225da14b2 nt!KiServiceLinkage                                         
20 ffffed0ee77df240 fffff80225da1142 TestDriver!LogCurrentIrql+0x92                              C:\Users\User\source\repos\TestDriver\TestDriver\Driver.c @ 21
21 ffffed0ee77df350 fffff80204307287 TestDriver!FileCreationThread+0x52                          C:\Users\User\source\repos\TestDriver\TestDriver\Driver.c @ 64
22 ffffed0ee77df570 fffff8020441b8e4 nt!PspSystemThreadStartup+0x57                              
23 ffffed0ee77df5c0 0000000000000000 nt!KiStartSystemThread+0x34
```

The **`!mex.imports`** command output reveals the Import Address Table (IAT) of the **`TestDriver`** module, showing that it imports functions like **`nt!ExAcquireFastMutex`**, which is used for acquiring a Fast Mutex.

```
4: kd> !mex.imports -a TestDriver
Dumping Import table at fffff80225da0000 + 6030 
ModuleImageName: TestDriver
fffff802`25da3000  fffff802`07940420 WDFLDR!WdfVersionUnbind
fffff802`25da3008  fffff802`07938bf0 WDFLDR!WdfLdrQueryInterface 
fffff802`25da3010  fffff802`0793fdd0 WDFLDR!WdfVersionBind
fffff802`25da3018  fffff802`07938d50 WDFLDR!WdfVersionUnbindClass 
fffff802`25da3020  fffff802`079315c0 WDFLDR!WdfVersionBindClass 
fffff802`25da3030  fffff802`04728670 nt!IoCreateDevice
fffff802`25da3038  fffff802`04207d10 nt!IoDeleteDevice
fffff802`25da3040  fffff802`04413730 nt!ZwCreateFile
fffff802`25da3048  fffff802`044132f0 nt!ZwOpenFile
fffff802`25da3050  fffff802`047220a0 nt!PsCreateSystemThread
fffff802`25da3058  fffff802`042ed710 nt!ExAcquireFastMutex
fffff802`25da3060  fffff802`04412e70 nt!ZwClose
fffff802`25da3068  fffff802`043d9f00 nt!swprintf_s
fffff802`25da3070  fffff802`043d44f0 nt!vsnwprintf
fffff802`25da3078  fffff802`04282100 nt!KeInitializeEvent
fffff802`25da3080  fffff802`04413170 nt!ZwSetInformationFile
fffff802`25da3088  fffff802`042ed850 nt!ExReleaseFastMutex
fffff802`25da3090  fffff802`04281860 nt!RtlCopyUnicodeString
fffff802`25da3098  fffff802`0431af70 nt!DbgPrintEx
fffff802`25da30a0  fffff802`04279a80 nt!KzRaiseIrql
fffff802`25da30a8  fffff802`04279ab0 nt!KeGetCurrentIrql
fffff802`25da30b0  fffff802`042e9d40 nt!RtlInitUnicodeString
fffff802`25da30b8  fffff802`04412d90 nt!ZwWriteFile
fffff802`25da30c0  fffff802`043db8d0 nt!wcscpy_s
fffff802`25da30d0  fffff802`25da1630 TestDriver!_guard_check_icall_nop
```

The **`!mex.strings`** command displays a list of strings extracted from the **`TestDriver`** module.

```
4: kd> !mex.strings -m TestDriver
fffff80225da004d !This program cannot be run in DOS mode.\r

$
fffff80225da01f8 .text
fffff80225da021f h.rdata
fffff80225da0247 H.data
fffff80225da0270 .pdata
fffff80225da0297 HINIT
fffff80225da02bf b.reloc
fffff80225da101f D$PE3
fffff80225da11fd L$`E3
fffff80225da1394 f94Au
fffff80225da152a \$@E3
fffff80225da1532 \$8E3
fffff80225da1587 L$ SVWAVH
fffff80225da15e4 (A^_^[
fffff80225da17bd x AVH
fffff80225da18c0 @8h0t&H
fffff80225da19c4 9(u"H
fffff80225da1c05 ;PuHH
fffff80225da1dd2 -fffffff
fffff80225da1de2 fffffff
fffff80225da1df1 fffffff
fffff80225da1e6a fffffff
fffff80225da1e91 .fffffff
fffff80225da1ea1 fffffff
fffff80225da1eb0 fffffff
fffff80225da1fc0 PASSIVE_LEVEL
fffff80225da1fe0 APC_LEVEL
fffff80225da2000 DISPATCH_LEVEL
fffff80225da2020 UNKNOWN_LEVEL (%u)
fffff80225da2050 Hello, World!
fffff80225da2060 \DosDevices\C:\Temp\FileCreationThreadIrql.txt
fffff80225da20c0 \DosDevices\C:\Temp\file_%d.txt
fffff80225da2100 \DosDevices\C:\Temp\FileMoveThreadIrql.txt
fffff80225da2160 \DosDevices\C:\Temp2\NewFile_%d.txt
fffff80225da3130 KmdfLibrary
fffff80225da3148 DriverEntry failed 0x%x for driver %wZ
fffff80225da3180 FxStubInitTypes: invalid driver image, the address of symbol __KMDF_TYPE_INIT_START 0x%p is greater than the address of symbol __KMDF_TYPE_INIT_END 0x%p, status 0x%x
fffff80225da3228 FxGetNextObjectContextTypeInfo failed
fffff80225da3250 FxStubBindClasses: invalid driver image, the address of symbol __KMDF_CLASS_BIND_START 0x%p is greater than the address of symbol __KMDF_CLASS_BIND_END 0x%p, status 0x%x
fffff80225da3300 FxGetNextClassBindInfo failed
fffff80225da3320 FxStubBindClasses: ClientBindClass %p, WDF_CLASS_BIND_INFO 0x%p, class %S, returned status 0x%x
fffff80225da3390 FxStubBindClasses: VersionBindClass WDF_CLASS_BIND_INFO 0x%p, class %S, returned status 0x%x
fffff80225da3580 RSDS
fffff80225da3598 C:\Users\User\source\repos\TestDriver\x64\Release\TestDriver.pdb
fffff80225da35e8 .text$mn
fffff80225da35fc .text$mn$00
fffff80225da3610 .text$mn$21
fffff80225da3624 .text$s
fffff80225da3634 .idata$5
fffff80225da3648 .00cfg
fffff80225da3658 .gfids
fffff80225da3668 .rdata
fffff80225da3678 .rdata$zzzdbg
fffff80225da3690 .xdata
fffff80225da36a0 .data
fffff80225da36b0 .kmdfclassbind$a
fffff80225da36cc .kmdfclassbind$c
fffff80225da36e8 .kmdfclassbind$d
fffff80225da3704 .kmdftypeinit$a
fffff80225da371c .kmdftypeinit$c
fffff80225da3744 .pdata
fffff80225da3764 .idata$2
fffff80225da3778 .idata$3
fffff80225da378c .idata$4
fffff80225da37a0 .idata$6
fffff80225da4080 \REGISTRY\MACHINE\SYSTEM\ControlSet001\Services\TestDrv
```

The crashing thread, running on **CPU 4**, was involved in a third-party driver's operation that incorrectly attempted to acquire a Fast Mutex while operating at **`DISPATCH_LEVEL`**, an action that is not supported within the Windows kernel environment.

```
4: kd> !irql 4
Debugger saved IRQL for processor 0x4 -- 2 (DISPATCH_LEVEL)
```
