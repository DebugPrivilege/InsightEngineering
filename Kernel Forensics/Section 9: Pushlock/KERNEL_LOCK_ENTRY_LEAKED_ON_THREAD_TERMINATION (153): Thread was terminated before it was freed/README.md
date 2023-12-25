# Description

Not releasing a lock that has been acquired is typically considered bad practice in kernel programming, as it can lead to deadlocks or other synchronization issues.

Link to this Memory Dump: https://mega.nz/file/rxtx2LZa#-JnOYyKST2SMstIxIDjk7kC8EbEtvGW-h5KQdUnbwTQ

```c
#include <ntifs.h>
#include <ntstrsafe.h>

typedef struct _SYNC_RESOURCES {
    EX_PUSH_LOCK FilePushLock;
} SYNC_RESOURCES, * PSYNC_RESOURCES;


// Thread function for creating files.
NTSTATUS FileCreationThread(PVOID Context) {
    PSYNC_RESOURCES SyncRes = (PSYNC_RESOURCES)Context;
    UNICODE_STRING uniFilePath;
    WCHAR filePathBuffer[150];
    char dataToWrite[] = "Hello, World!\r\n";
    IO_STATUS_BLOCK ioStatusBlock;

    KeEnterCriticalRegion();  // Disable normal kernel APCs before acquiring Pushlock.
    ExAcquirePushLockExclusive(&SyncRes->FilePushLock); // Acquire Pushlock for exclusive access.

    // Loop to create a series of files.
    for (int i = 0; i < 1000; i++) {
        // Prepare file path.
        RtlStringCchPrintfW(filePathBuffer, sizeof(filePathBuffer) / sizeof(WCHAR), L"\\DosDevices\\C:\\Temp\\file_%d.txt", i);
        RtlInitUnicodeString(&uniFilePath, filePathBuffer);

        // Set up object attributes for the file.
        OBJECT_ATTRIBUTES objAttributes;
        InitializeObjectAttributes(&objAttributes, &uniFilePath, OBJ_CASE_INSENSITIVE | OBJ_KERNEL_HANDLE, NULL, NULL);

        HANDLE hFile;
        // Create the file.
        NTSTATUS status = ZwCreateFile(&hFile, GENERIC_WRITE, &objAttributes, &ioStatusBlock, NULL, FILE_ATTRIBUTE_NORMAL, 0, FILE_OVERWRITE_IF, FILE_SYNCHRONOUS_IO_NONALERT, NULL, 0);

        // Write data to the file if creation was successful.
        if (NT_SUCCESS(status)) {
            for (int j = 0; j < 100; j++) {
                ZwWriteFile(hFile, NULL, NULL, NULL, &ioStatusBlock, dataToWrite, sizeof(dataToWrite) - 1, NULL, NULL);
            }
            ZwClose(hFile); // Close the file handle.
        }
    }

    ExReleasePushLockExclusive(&SyncRes->FilePushLock); // Release the Pushlock.
    KeLeaveCriticalRegion(); // Re-enable normal kernel APCs.

    return STATUS_SUCCESS;
}

NTSTATUS FileMoveThread(PVOID Context) {
    PSYNC_RESOURCES SyncRes = (PSYNC_RESOURCES)Context;
    IO_STATUS_BLOCK ioStatusBlock;
    WCHAR oldFilePathBuffer[150], newFilePathBuffer[150];

    KeEnterCriticalRegion();  // Disable normal kernel APCs before acquiring the Pushlock.
    ExAcquirePushLockExclusive(&SyncRes->FilePushLock);  // Acquire Pushlock for exclusive access.

    // Loop to move a series of files.
    for (int i = 0; i < 1000; i++) {
        // Prepare old and new file paths.
        RtlStringCchPrintfW(oldFilePathBuffer, sizeof(oldFilePathBuffer) / sizeof(WCHAR), L"\\DosDevices\\C:\\Temp\\file_%d.txt", i);
        RtlStringCchPrintfW(newFilePathBuffer, sizeof(newFilePathBuffer) / sizeof(WCHAR), L"\\DosDevices\\C:\\Temp2\\NewFile_%d.txt", i);

        UNICODE_STRING oldFilePath, newFilePath;
        RtlInitUnicodeString(&oldFilePath, oldFilePathBuffer);
        RtlInitUnicodeString(&newFilePath, newFilePathBuffer);

        OBJECT_ATTRIBUTES objAttributes;
        InitializeObjectAttributes(&objAttributes, &oldFilePath, OBJ_CASE_INSENSITIVE | OBJ_KERNEL_HANDLE, NULL, NULL);

        HANDLE hFile;
        // Open the existing file for moving.
        NTSTATUS status = ZwOpenFile(&hFile, FILE_ALL_ACCESS, &objAttributes, &ioStatusBlock, FILE_SHARE_READ | FILE_SHARE_WRITE, FILE_SYNCHRONOUS_IO_NONALERT);

        if (NT_SUCCESS(status)) {
            // Allocate memory for rename information structure.
            FILE_RENAME_INFORMATION* renameInfo = ExAllocatePoolWithTag(NonPagedPool, sizeof(FILE_RENAME_INFORMATION) + newFilePath.Length, 'mvTg');
            if (renameInfo) {
                // Set up the rename information.
                renameInfo->ReplaceIfExists = TRUE;
                renameInfo->RootDirectory = NULL;
                renameInfo->FileNameLength = newFilePath.Length;
                RtlCopyMemory(renameInfo->FileName, newFilePath.Buffer, newFilePath.Length);

                // Rename the file using the ZwSetInformationFile function.
                status = ZwSetInformationFile(hFile, &ioStatusBlock, renameInfo, sizeof(FILE_RENAME_INFORMATION) + newFilePath.Length, FileRenameInformation);
                ExFreePoolWithTag(renameInfo, 'mvTg'); // Free the allocated memory.

                if (!NT_SUCCESS(status)) {
                    KdPrint(("ZwSetInformationFile failed with status 0x%X\n", status)); // Log if renaming fails.
                }
                ZwClose(hFile); // Close the file handle.
            }
        }
        else {
            KdPrint(("ZwOpenFile failed with status 0x%X\n", status)); // Log if opening the file fails.
        }

        // Terminate the thread before releasing the push lock.
        if (i == 500) { // Example condition to terminate the thread.
            KeLeaveCriticalRegion(); // It's important to leave the critical region before exiting.
            return STATUS_SUCCESS; // Return success or appropriate error status.
        }
    }

    // The following lines are now unreachable, but kept for reference.
    ExReleasePushLockExclusive(&SyncRes->FilePushLock); // Release the Pushlock.
    KeLeaveCriticalRegion();  // Re-enable normal kernel APCs.

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
    ExInitializePushLock(&SyncRes->FilePushLock);

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

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/cd1c9743-f355-4ba8-ac06-4c0fc5de7d74)

Start the kernel driver:

```
sc.exe start TestDrv
```

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/7ff6b86b-acf4-4cea-b6e6-8f686287d8bc)

This will bugcheck the system:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/1c20406b-682d-4cb9-88f4-1a4cbbbe402f)


# WinDbg Walk Through - Analysis

Load the Memory Dump in WinDbg:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/a875b69a-dd70-4eb0-a59f-81b3656244ac)


Let's start with the **`!mex.di`** command which stands for dump information. This command retrieves information about the machine from which the memory dump originated. It's especially handy when working with multiple memory dumps, as it helps distinguish the analysis related to each specific dump. This is an improved version of the **`vertarget`** command.

```
3: kd> !mex.di
Computer Name:  Not Found
Windows 10 Kernel Version 22621 MP (6 procs) Free x64
Product: WinNt, suite: TerminalServer SingleUserTS
Edition build lab: 22621.2506.amd64fre.ni_release_svc_prod3.231018-1809
Kernel base = 0xfffff807`26800000 PsLoadedModuleList = 0xfffff807`274134a0
Debug session time: Sun Dec 24 09:17:43.713 2023 (UTC - 8:00)
System Uptime: 0 days 0:51:44.311
SystemManufacturer = Microsoft Corporation
SystemProductName = Virtual Machine
Processor: 11th Gen Intel(R) Core(TM) i7-1185G7 @ 3.00GHz
Bugcheck: 153 (FFFFC78839108040, FFFFC788391086E0, 1, FFFFC7883A0F6F70)
```

Let's run the **`!analyze -v`** command to quickly triage the issue as it provides a quick and detailed overview of the state of the system at the time of the crash. It automates many aspects of crash dump analysis, providing a quick way to get valuable information. This output points to an issue where a thread in the system was terminated without releasing a lock it had acquired. The provided addresses and the stack trace are key in further investigating this issue. However, let's get back to that later.

```
3: kd> !analyze -v
*******************************************************************************
*                                                                             *
*                        Bugcheck Analysis                                    *
*                                                                             *
*******************************************************************************

KERNEL_LOCK_ENTRY_LEAKED_ON_THREAD_TERMINATION (153)
A thread was terminated before it had freed all its AutoBoost lock entries.
This is typically caused when a thread never released a lock it previously
acquired (e.g. by relying on another thread to release it), or if the thread
did not supply a consistent set of flags to lock package APIs.
Arguments:
Arg1: ffffc78839108040, The address of the thread
Arg2: ffffc788391086e0, The address of the entry that was not freed
Arg3: 0000000000000001, Lock pointer was not NULL
Arg4: ffffc7883a0f6f70, The address of the lock associated with the entry

Debugging Details:
------------------


KEY_VALUES_STRING: 1

    Key  : Analysis.CPU.mSec
    Value: 1859

    Key  : Analysis.Elapsed.mSec
    Value: 2067

    Key  : Analysis.IO.Other.Mb
    Value: 0

    Key  : Analysis.IO.Read.Mb
    Value: 1

    Key  : Analysis.IO.Write.Mb
    Value: 6

    Key  : Analysis.Init.CPU.mSec
    Value: 1046

    Key  : Analysis.Init.Elapsed.mSec
    Value: 469164

    Key  : Analysis.Memory.CommitPeak.Mb
    Value: 151

    Key  : Bugcheck.Code.KiBugCheckData
    Value: 0x153

    Key  : Bugcheck.Code.LegacyAPI
    Value: 0x153

    Key  : Dump.Attributes.AsUlong
    Value: 800

    Key  : Failure.Bucket
    Value: 0x153_KERNEL_LOCK_ENTRY_LEAKED_ON_THREAD_TERMINATION_LOCK_POINTER_NOT_NULL_nt!KeCleanupThreadState

    Key  : Failure.Hash
    Value: {c1f11abf-30b0-6f15-0afc-73c2b6a6a885}

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


BUGCHECK_CODE:  153

BUGCHECK_P1: ffffc78839108040

BUGCHECK_P2: ffffc788391086e0

BUGCHECK_P3: 1

BUGCHECK_P4: ffffc7883a0f6f70

FILE_IN_CAB:  MEMORY2.DMP

VIRTUAL_MACHINE:  HyperV

DUMP_FILE_ATTRIBUTES: 0x800

BLACKBOXBSD: 1 (!blackboxbsd)


BLACKBOXNTFS: 1 (!blackboxntfs)


BLACKBOXPNP: 1 (!blackboxpnp)


BLACKBOXWINLOGON: 1

PROCESS_NAME:  System

STACK_TEXT:  
fffffb80`7c62f1d8 fffff807`26c79e00     : 00000000`00000153 ffffc788`39108040 ffffc788`391086e0 00000000`00000001 : nt!KeBugCheckEx
fffffb80`7c62f1e0 fffff807`26edb49e     : ffffc788`00000000 ffffc788`39108010 fffffb80`7c62f480 00000000`00000000 : nt!KeCleanupThreadState+0x2467a8
fffffb80`7c62f230 fffff807`26ef352e     : 00000000`00000000 ffffc788`39108010 fffffb80`7c62f480 ffffc788`39108010 : nt!PspThreadDelete+0x1e
fffffb80`7c62f2a0 fffff807`26aec627     : 00000000`00000000 00000000`00000000 fffffb80`7c62f480 ffffc788`39108040 : nt!ObpRemoveObjectRoutine+0x7e
fffffb80`7c62f300 fffff807`26b4056c     : ffffc788`39108040 ffffc788`39108430 ffffc788`31c95740 ffffc788`391084e8 : nt!ObfDereferenceObjectWithTag+0xc7
fffffb80`7c62f340 fffff807`26a34f85     : ffffc788`32aeb040 fffffb80`7c62f480 ffffc788`31c95740 ffffc788`00000000 : nt!PspReaper+0xac
fffffb80`7c62f380 fffff807`26b07167     : ffffc788`32aeb040 00000000`000007c8 ffffc788`32aeb040 fffff807`26a34e30 : nt!ExpWorkerThread+0x155
fffffb80`7c62f570 fffff807`26c1bb94     : ffffb680`0130d180 ffffc788`32aeb040 fffff807`26b07110 ffabc4da`ffabc4da : nt!PspSystemThreadStartup+0x57
fffffb80`7c62f5c0 00000000`00000000     : fffffb80`7c630000 fffffb80`7c629000 00000000`00000000 00000000`00000000 : nt!KiStartSystemThread+0x34


SYMBOL_NAME:  nt!KeCleanupThreadState+2467a8

MODULE_NAME: nt

IMAGE_NAME:  ntkrnlmp.exe

STACK_COMMAND:  .cxr; .ecxr ; kb

BUCKET_ID_FUNC_OFFSET:  2467a8

FAILURE_BUCKET_ID:  0x153_KERNEL_LOCK_ENTRY_LEAKED_ON_THREAD_TERMINATION_LOCK_POINTER_NOT_NULL_nt!KeCleanupThreadState

OS_VERSION:  10.0.22621.2506

BUILDLAB_STR:  ni_release_svc_prod3

OSPLATFORM_TYPE:  x64

OSNAME:  Windows 10

FAILURE_ID_HASH:  {c1f11abf-30b0-6f15-0afc-73c2b6a6a885}

Followup:     MachineOwner
---------
```

From the output, we can conclude that the following arguments likely mean the following:

- **`Arg1:`** The address of the thread that was terminated. 
- **`Arg2:`** The address of the lock entry that was not freed. 
- **`Arg3:`** Indicates that the lock pointer was not NULL, confirming the presence of an unreleased lock.
- **`Arg4:`** The address of the lock associated with the entry. This helps in identifying the specific lock that was involved.

The **`!mex.tl -t`** command examines the state and wait reasons for every thread across all processes, presenting the results as statistics. This allows for a quick overview, such as determining how many processes have 4 running threads, and so on. It's like having a satellite view of all the thread state that are associated with the processes.

```
3: kd> !mex.tl -t
PID            Address          Name                                   !! Rn Ry Bk Lc IO Er
============== ================ ====================================== == == == == == == ==
0x0    0n0     fffff80727549f40 Idle                                    .  6  6  .  .  .  .
0x4    0n4     ffffc78831ceb040 System                                  6  1  .  1  .  1  .
0x78   0n120   ffffc78831d1a080 Registry                                .  .  .  .  .  .  .
0x230  0n560   ffffc78832ff9040 smss.exe                                .  .  .  1  .  .  .
0x2c4  0n708   ffffc7883624d140 csrss.exe                               .  .  .  .  1  .  .
0x30c  0n780   ffffc788364c8080 wininit.exe                             .  .  .  .  .  .  .
0x320  0n800   ffffc788364d5140 csrss.exe                               .  .  .  .  1  .  .
0x374  0n884   ffffc78836542080 winlogon.exe                            .  .  .  .  .  .  .
0x3a0  0n928   ffffc788363430c0 services.exe                            .  .  .  .  .  .  .
0x3cc  0n972   ffffc7883655e080 LsaIso.exe                              .  .  .  .  .  .  .
0x3d8  0n984   ffffc78836563080 lsass.exe                               .  .  .  .  .  .  .
0x308  0n776   ffffc788365bb080 svchost.exe                             .  .  .  .  .  .  .
0x404  0n1028  ffffc788365e50c0 fontdrvhost.exe                         .  .  .  .  .  .  .
0x408  0n1032  ffffc788365c5080 fontdrvhost.exe                         .  .  .  .  .  .  .
0x474  0n1140  ffffc788366950c0 svchost.exe                             .  .  .  .  .  .  .
0x4a0  0n1184  ffffc788366a3080 svchost.exe                             .  .  .  .  .  .  .
0x4f8  0n1272  ffffc788367cc0c0 dwm.exe                                 .  .  .  .  .  .  .
0x558  0n1368  ffffc78836814080 svchost.exe                             .  .  .  .  .  .  .
0x560  0n1376  ffffc78836822080 svchost.exe                             .  .  .  .  .  .  .
0x5f8  0n1528  ffffc7883688f080 svchost.exe                             .  .  .  .  .  .  .
0x60c  0n1548  ffffc78836895080 svchost.exe                             .  .  .  .  .  .  .
0x614  0n1556  ffffc78836898080 svchost.exe                             .  .  .  .  .  .  .
0x62c  0n1580  ffffc788368a8080 svchost.exe                             .  .  .  .  .  .  .
0x634  0n1588  ffffc788368ac080 svchost.exe                             .  .  .  .  .  .  .
0x63c  0n1596  ffffc788368a9080 svchost.exe                             .  .  .  .  .  .  .
0x654  0n1620  ffffc788368b0080 svchost.exe                             .  .  .  .  .  .  .
0x6e0  0n1760  ffffc78836924080 svchost.exe                             .  .  .  .  1  .  .
0x700  0n1792  ffffc7883692b080 svchost.exe                             .  .  .  .  .  .  .
0x794  0n1940  ffffc788369d20c0 svchost.exe                             .  .  .  .  .  .  .
0x7a0  0n1952  ffffc788369bf080 svchost.exe                             .  .  .  .  .  .  .
0x7b8  0n1976  ffffc788369dd080 svchost.exe                             .  .  .  .  .  .  .
0x35c  0n860   ffffc788369e30c0 svchost.exe                             .  .  .  .  .  .  .
0x80c  0n2060  ffffc78836a66080 svchost.exe                             .  .  .  .  .  .  .
0x830  0n2096  ffffc78836a79080 svchost.exe                             .  .  .  .  .  .  .
0x83c  0n2108  ffffc78836a460c0 svchost.exe                             .  .  .  .  .  .  .
0x874  0n2164  ffffc78836a50080 svchost.exe                             .  .  .  .  .  .  .
0x890  0n2192  ffffc78836a55080 svchost.exe                             .  .  .  .  .  .  .
0x8d8  0n2264  ffffc78836af5080 svchost.exe                             .  .  .  .  .  .  .
0x970  0n2416  ffffc78836b32080 VSSVC.exe                               .  .  .  .  .  .  .
0x99c  0n2460  ffffc78836ac9080 svchost.exe                             .  .  .  .  .  .  .
0x9e0  0n2528  ffffc78836bed080 svchost.exe                             .  .  .  .  .  .  .
0x9e8  0n2536  ffffc78836b350c0 svchost.exe                             .  .  .  .  .  .  .
0xa50  0n2640  ffffc78836c09080 svchost.exe                             .  .  .  .  .  .  .
0xa58  0n2648  ffffc78836c0f080 svchost.exe                             .  .  .  .  .  .  .
0xa78  0n2680  ffffc78836c17080 svchost.exe                             .  .  .  .  .  .  .
0xaa4  0n2724  ffffc78836c26080 svchost.exe                             .  .  .  .  .  .  .
0xac8  0n2760  ffffc78831dc1040 MemCompression                          .  .  .  .  .  .  .
0xb1c  0n2844  ffffc78831d79080 svchost.exe                             .  .  .  .  .  .  .
0xb6c  0n2924  ffffc78831d4e080 svchost.exe                             .  .  .  .  .  .  .
0xb74  0n2932  ffffc78831d48080 svchost.exe                             .  .  .  .  .  .  .
0xb80  0n2944  ffffc78831d72080 svchost.exe                             .  .  .  .  .  .  .
0xbc4  0n3012  ffffc78831d26080 svchost.exe                             .  .  .  .  .  .  .
0xc04  0n3076  ffffc78836dab080 svchost.exe                             .  .  .  .  .  .  .
0xc28  0n3112  ffffc78836de8080 svchost.exe                             .  .  .  .  .  .  .
0xca4  0n3236  ffffc78836e84080 svchost.exe                             .  .  .  .  .  .  .
0xd38  0n3384  ffffc78836f96080 svchost.exe                             .  .  .  .  .  .  .
0xd40  0n3392  ffffc78836eca080 svchost.exe                             .  .  .  .  .  .  .
0xd9c  0n3484  ffffc78838025080 svchost.exe                             .  .  .  .  .  .  .
0xe2c  0n3628  ffffc788380a80c0 spoolsv.exe                             .  .  .  .  .  .  .
0xe68  0n3688  ffffc788381ac0c0 svchost.exe                             .  .  .  .  .  .  .
0xf5c  0n3932  ffffc78838288080 svchost.exe                             .  .  .  .  .  .  .
0xf74  0n3956  ffffc788383a8080 svchost.exe                             .  .  .  .  .  .  .
0xf80  0n3968  ffffc78838342080 svchost.exe                             .  .  .  .  .  .  .
0xf98  0n3992  ffffc788382ec080 svchost.exe                             .  .  .  .  .  .  .
0xfa4  0n4004  ffffc788382ee080 svchost.exe                             .  .  .  .  .  .  .
0xfac  0n4012  ffffc788383fb080 svchost.exe                             .  .  .  .  .  .  .
0xfc4  0n4036  ffffc78838131080 svchost.exe                             .  .  .  .  .  .  .
0xfcc  0n4044  ffffc78838072080 IpOverUsbSvc.exe*32                     .  .  .  .  .  .  .
0xfd4  0n4052  ffffc788382aa080 sqlwriter.exe                           .  .  .  .  .  .  .
0xc44  0n3140  ffffc78838355080 svchost.exe                             .  .  .  .  .  .  .
0xc74  0n3188  ffffc788381bb080 svchost.exe                             .  .  .  .  .  .  .
0x1018 0n4120  ffffc78838188080 wlms.exe                                .  .  .  .  .  .  .
0x1044 0n4164  ffffc788382ba080 MsMpEng.exe                             .  1  .  .  .  .  .
0x1060 0n4192  ffffc788383b9080 svchost.exe                             .  .  .  .  .  .  .
0x111c 0n4380  ffffc788385a9080 sppsvc.exe                              .  .  .  .  .  .  .
0xfb8  0n4024  ffffc78838916080 svchost.exe                             .  .  .  .  .  .  .
0x1480 0n5248  ffffc78838993080 svchost.exe                             .  .  .  .  .  .  .
0x156c 0n5484  ffffc78838a1a080 svchost.exe                             .  .  .  .  .  .  .
0x1768 0n5992  ffffc78838c7d080 AggregatorHost.exe                      .  .  .  .  .  .  .
0x17ac 0n6060  ffffc78838bf1080 sihost.exe                              .  .  .  .  .  .  .
0x17e8 0n6120  ffffc78838c9e080 svchost.exe                             .  .  .  .  .  .  .
0x1518 0n5400  ffffc78838cb5080 svchost.exe                             .  .  .  .  .  .  .
0x13b8 0n5048  ffffc78838cde080 svchost.exe                             .  .  .  .  .  .  .
0x15c4 0n5572  ffffc78838cf9080 svchost.exe                             .  .  .  .  .  .  .
0x16c8 0n5832  ffffc78838cfa080 SppExtComObj.Exe                        .  .  .  .  .  .  .
0x186c 0n6252  ffffc78838dd6080 taskhostw.exe                           .  .  .  .  .  .  .
0x1898 0n6296  ffffc78838ddb0c0 svchost.exe                             .  .  .  .  .  .  .
0x18f8 0n6392  ffffc78838e430c0 explorer.exe                            5  .  .  .  1  .  .
0x1aec 0n6892  ffffc78838f650c0 svchost.exe                             .  .  .  .  .  .  .
0x1b38 0n6968  ffffc788362f1080 svchost.exe                             .  .  .  .  .  .  .
0x1838 0n6200  ffffc7883910d080 svchost.exe                             .  .  .  .  .  .  .
0x19bc 0n6588  ffffc78839117080 svchost.exe                             .  .  .  .  .  .  .
0x1714 0n5908  ffffc78839133080 unsecapp.exe                            .  .  .  .  .  .  .
0x1c88 0n7304  ffffc78839197080 SearchHost.exe                         10  .  .  .  2  .  .
0x1c90 0n7312  ffffc788391bd0c0 StartMenuExperienceHost.exe             .  .  .  .  .  .  .
0x1d64 0n7524  ffffc78839369080 Widgets.exe                             .  .  .  .  .  .  .
0x1de8 0n7656  ffffc7883936a080 RuntimeBroker.exe                       .  .  .  .  .  .  .
0x1e5c 0n7772  ffffc7883930f080 RuntimeBroker.exe                       .  .  .  .  .  .  .
0x1f24 0n7972  ffffc788387e4080 svchost.exe                             .  .  .  .  .  .  .
0x13c4 0n5060  ffffc78838d8e0c0 dllhost.exe                             .  .  .  .  .  .  .
0x220c 0n8716  ffffc78839268080 svchost.exe                             .  .  .  .  .  .  .
0x233c 0n9020  ffffc788394c80c0 ctfmon.exe                              .  .  .  .  .  .  .
0x23b4 0n9140  ffffc78838d020c0 SearchIndexer.exe                       .  .  .  .  .  .  .
0x514  0n1300  ffffc788399020c0 csrss.exe                               .  .  .  .  1  .  .
0x136c 0n4972  ffffc788399e5080 winlogon.exe                            .  .  .  .  1  .  .
0x20e8 0n8424  ffffc78839c450c0 fontdrvhost.exe                         .  .  .  .  .  .  .
0x2124 0n8484  ffffc78839c5e0c0 LogonUI.exe                             .  .  .  .  .  .  .
0x2120 0n8480  ffffc78839c4a080 dwm.exe                                 .  .  .  .  .  .  .
0x13d8 0n5080  ffffc78839d7a0c0 WUDFHost.exe                            .  .  .  .  .  .  .
0x3b0  0n944   ffffc78839d7d080 rdpclip.exe                             .  .  .  .  .  .  .
0x1fb4 0n8116  ffffc78839d9c0c0 rdpinput.exe                            .  .  .  .  .  .  .
0x1234 0n4660  ffffc78839b31080 svchost.exe                             .  .  .  .  .  .  .
0x24c0 0n9408  ffffc7883932a0c0 TabTip.exe                              .  .  .  .  .  .  .
0x255c 0n9564  ffffc78839dda080 TextInputHost.exe                       .  .  .  .  .  .  .
0x26b4 0n9908  ffffc7883914a0c0 NisSrv.exe                              .  .  .  .  .  .  .
0x11b0 0n4528  ffffc78838f240c0 SecurityHealthSystray.exe               .  .  .  .  .  .  .
0x1fbc 0n8124  ffffc7883973a080 SecurityHealthService.exe               .  .  .  .  .  .  .
0x2640 0n9792  ffffc78836df0080 taskhostw.exe                           .  .  .  .  .  .  .
0xd5c  0n3420  ffffc788362290c0 svchost.exe                             .  .  .  .  .  .  .
0xff8  0n4088  ffffc7883589b0c0 uhssvc.exe                              .  .  .  .  .  .  .
0x1068 0n4200  ffffc78835baa0c0 svchost.exe                             .  .  .  .  .  .  .
0x1540 0n5440  ffffc788393960c0 svchost.exe                             .  .  .  .  .  .  .
0x12e0 0n4832  ffffc78838ca80c0 svchost.exe                             .  .  .  .  .  .  .
0xe40  0n3648  ffffc78835ef40c0 svchost.exe                             .  .  .  .  .  .  .
0x5e4  0n1508  ffffc788393910c0 svchost.exe                             .  .  .  .  .  .  .
0x2398 0n9112  ffffc7883631d0c0 devenv.exe                              .  .  .  1  .  .  .
0xa10  0n2576  ffffc78835d380c0 PerfWatson2.exe                         .  .  .  2  .  .  .
0x6b4  0n1716  ffffc788358680c0 Microsoft.ServiceHub.Controller.exe     .  .  .  1  .  .  .
0x3c4  0n964   ffffc788395f10c0 ServiceHub.VSDetouredHost.exe           .  .  .  1  .  .  .
0x1430 0n5168  ffffc78839ef20c0 ServiceHub.IdentityHost.exe*32          .  .  .  3  .  .  .
0x8a0  0n2208  ffffc78835ed50c0 ServiceHub.SettingsHost.exe             .  .  .  2  .  .  .
0x10c0 0n4288  ffffc7883585e0c0 ServiceHub.Host.netfx.x86.exe*32        .  .  .  4  .  .  .
0x1944 0n6468  ffffc78835c8c0c0 svchost.exe                             .  .  .  .  .  .  .
0x1ae4 0n6884  ffffc788358630c0 ServiceHub.ThreadedWaitDialog.exe       .  .  .  1  .  .  .
0x2744 0n10052 ffffc788357390c0 ServiceHub.IndexingService.exe          .  .  .  1  .  .  .
0x1174 0n4468  ffffc78835ba50c0 vshost.exe                              1  .  .  .  .  .  .
0x1848 0n6216  ffffc78835ce50c0 ServiceHub.IntellicodeModelService.exe  .  .  .  .  .  .  .
0x2b0  0n688   ffffc788357320c0 vcpkgsrv.exe*32                         .  .  .  .  .  .  .
0x1ed0 0n7888  ffffc7883999e140 ServiceHub.Host.AnyCPU.exe              .  .  .  1  .  .  .
0x1334 0n4916  ffffc78835b020c0 ServiceHub.TestWindowStoreHost.exe      .  .  .  1  .  .  .
0xdc4  0n3524  ffffc78835daf080 vcpkgsrv.exe*32                         .  .  .  .  .  .  .
0xe48  0n3656  ffffc78838d8c080 ShellExperienceHost.exe                 2  .  .  .  .  .  .
0x5e8  0n1512  ffffc7883922f080 svchost.exe                             .  .  .  .  .  .  .
0x1440 0n5184  ffffc78832b7e080 RuntimeBroker.exe                       .  .  .  .  1  .  .
0x1700 0n5888  ffffc7883587a080 SystemSettingsBroker.exe                .  .  .  .  .  .  .
0x2164 0n8548  ffffc78839d81080 svchost.exe                             .  .  .  .  .  .  .
0x1b54 0n6996  ffffc7883940f080 svchost.exe                             .  .  .  .  .  .  .
0x1514 0n5396  ffffc78839ecf080 WidgetService.exe                       .  .  .  .  .  .  .
0xa04  0n2564  ffffc78839ef1080 svchost.exe                             .  .  .  .  .  .  .
0x1bd4 0n7124  ffffc78832bca180 vcpkgsrv.exe*32                         .  .  .  .  .  .  .
0x1c3c 0n7228  ffffc7883599e0c0 vcpkgsrv.exe*32                         .  .  .  .  .  .  .
0x2304 0n8964  ffffc7883682b080 dllhost.exe                             .  .  .  .  .  .  .
0x2a30 0n10800 ffffc78832acb080 cmd.exe                                 .  .  1  .  .  .  .
0x2a38 0n10808 ffffc78835ef9080 conhost.exe                             .  .  .  .  .  .  .
0x1cc0 0n7360  ffffc7883c8800c0 SearchProtocolHost.exe                  .  .  .  .  .  .  .
0x1a68 0n6760  ffffc7883becf0c0 MSBuild.exe                             .  .  .  .  .  .  .
0x2580 0n9600  ffffc7883b6eb0c0 conhost.exe                             .  .  .  .  .  .  .
0x81c  0n2076  ffffc7883bfcf0c0 SearchFilterHost.exe                    .  .  .  .  .  .  .
============== ================ ====================================== == == == == == == ==
PID            Address          Name                                   !! Rn Ry Bk Lc IO Er

Warning! Zombie process(es) detected (not displayed). Count: 7 [zombie report]
```

From the **`!mex.tl -t`** command, we can see that there are a couple of running threads. Let's examine the running threads on the system. The output of the **`!mex.running`** command in provides a snapshot of the currently running threads on the system. 

```
0: kd> !mex.running
Process      PID Thread             Id Pri Base Pri Next CPU CSwitches  User     Kernel State      Time Reason
=========== ==== ================ ==== === ======== ======== ========= ===== ========== ======= ======= =============
MsMpEng.exe 1044 ffffc788386a4080 2b8c   8        8        0      1161 234ms       16ms Running       0 WrDispatchInt
Idle           0 ffffc78831d45080    0   0        0        1    520018     0 49m:35.906 Running    78ms Executive
Idle           0 ffffc78831d78080    0   0        0        2    943334     0 49m:20.719 Running  1s.531 Executive
System         4 ffffc78832aeb040 1cc4  15       15        3     29857     0     1s.109 Running       0 Executive
Idle           0 ffffc78831d89080    0  63        0        4    765504     0 49m:39.578 Running  4s.703 Executive
Idle           0 ffffc78831dde080    0  63        0        5    541371     0 50m:04.094 Running 15s.046 Executive

Count: 6 | Show Unique Stacks
```

This thread that is associated with the **System.exe** is running on **CPU 3**, but is also the crashing thread. The **`nt!PspThreadDelete`** function is involved in the deletion of a thread. The presence of this function suggests that the thread was in the process of being deleted or terminated when the issue occurred. Additionally, from the call stack, it's identical to the thread we identified earlier through the **`!analyze -v`** output.

```
3: kd> !mex.t ffffc78832aeb040
Process                   Thread                       CID       UserTime KernelTime ContextSwitches Wait Reason Time State
System (ffffc78831ceb040) ffffc78832aeb040 (E|K|W|R|V) 4.1cc4           0     1s.109           29857 Executive      0 Running on processor 3

Priority:
    Current Base Decrement ForegroundBoost IO Page
    15      15   0         0               0  5

# Child-SP         Return           Call Site
0 fffffb807c62f1d8 fffff80726c79e00 nt!KeBugCheckEx
1 fffffb807c62f1e0 fffff80726edb49e nt!KeCleanupThreadState+0x2467a8
2 fffffb807c62f230 fffff80726ef352e nt!PspThreadDelete+0x1e
3 fffffb807c62f2a0 fffff80726aec627 nt!ObpRemoveObjectRoutine+0x7e
4 fffffb807c62f300 fffff80726b4056c nt!ObfDereferenceObjectWithTag+0xc7
5 fffffb807c62f340 fffff80726a34f85 nt!PspReaper+0xac
6 fffffb807c62f380 fffff80726b07167 nt!ExpWorkerThread+0x155
7 fffffb807c62f570 fffff80726c1bb94 nt!PspSystemThreadStartup+0x57
8 fffffb807c62f5c0 0000000000000000 nt!KiStartSystemThread+0x34
```

The command **`.frame /r 0n2`** sets the context to the **`2th`** frame in the stack trace and displays the register values at that moment. By examining the register values and the current instruction, we can get insights into what the function is doing at that point. This is a snapshot of the CPU state at the **`2th`** frame of the stack, which is within the **`nt!PspThreadDelete`** function.

```
0: kd> .frame /r 0n2
02 fffffb80`7c62f230 fffff807`26ef352e     nt!PspThreadDelete+0x1e
rax=8000000000000000 rbx=ffffc78839108010 rcx=0000000000000153
rdx=ffffc78839108040 rsi=0000000000000000 rdi=ffffc78839108040
rip=fffff80726edb49e rsp=fffffb807c62f230 rbp=fffffb807c62f480
 r8=ffffc788391086e0  r9=0000000000000001 r10=fffff80726edb480
r11=0000000000000000 r12=0000000000000000 r13=0000000000000000
r14=0000000000000001 r15=0000000000000000
iopl=0         nv up ei ng nz na pe nc
cs=0010  ss=0018  ds=002b  es=002b  fs=0053  gs=002b             efl=00040282
nt!PspThreadDelete+0x1e:
fffff807`26edb49e 8b87ac050000    mov     eax,dword ptr [rdi+5ACh] ds:002b:ffffc788`391085ec=00000000
```

The **`nt!PspThreadDelete`** function is involved in the deletion of a thread, so which thread are we terminating or deleting? When looking around at the registers, we noticed that the **RDI** register is a thread.

```
0: kd> !object ffffc78839108040
Object: ffffc78839108040  Type: (ffffc78831cea140) Thread
    ObjectHeader: ffffc78839108010 (new version)
    HandleCount: 0  PointerCount: 0
```

This thread has been terminated as we can see here:

```
0: kd> !mex.t ffffc78839108040
Process                   Thread                       CID       UserTime KernelTime ContextSwitches Wait Reason  Time State
System (ffffc78831ceb040) ffffc78839108040 (E|K|W|R|V) 4.1928           0          0               2 WrTerminated    0 Terminated

# Child-SP         Return           Call Site
0 fffffb807af47120 fffff80726a6c9d5 nt!KiSwapContext+0x76
1 fffffb807af47260 fffff80726a342db nt!KiSwapThread+0xab5
2 fffffb807af473b0 fffff80726edc507 nt!KeTerminateThread+0x19f
3 fffffb807af47430 fffff80726f70613 nt!PspExitThread+0x4c3
4 fffffb807af47530 fffff80726b07181 nt!PspTerminateThreadByPointer+0x53
5 fffffb807af47570 fffff80726c1bb94 nt!PspSystemThreadStartup+0x71
6 fffffb807af475c0 0000000000000000 nt!KiStartSystemThread+0x34

Warning!!! Thread is marked as terminated
```

We can verify this by examining the **ETHREAD** structure, focusing on the **`Terminated`** field. This confirms that the thread in question was indeed terminated:

```
3: kd> !mex.ddt nt!_ETHREAD ffffc78839108040 -y Terminated

dt nt!_ETHREAD ffffc78839108040 -y Terminated () Recursive: [ -r1 -r2 -r ] Verbose Normal dt
==================================================================================
   +0x560 Terminated           : 0y1
```

This is the same thread that showed up in the **`!analyze -v`** results:

```
3: kd> !analyze -show
KERNEL_LOCK_ENTRY_LEAKED_ON_THREAD_TERMINATION (153)
A thread was terminated before it had freed all its AutoBoost lock entries.
This is typically caused when a thread never released a lock it previously
acquired (e.g. by relying on another thread to release it), or if the thread
did not supply a consistent set of flags to lock package APIs.
Arguments:
Arg1: ffffc78839108040, The address of the thread
Arg2: ffffc788391086e0, The address of the entry that was not freed
Arg3: 0000000000000001, Lock pointer was not NULL
Arg4: ffffc7883a0f6f70, The address of the lock associated with the entry
```

Returning to the **`!mex.tl -t`** command, which offers an overview of the state of threads associated with processes, we've already reviewed the running threads and identified one of particular interest. Now, we'll shift our focus to other threads. Within the **System.exe** process, there's a thread in a blocked state (Bk). 

```
3: kd> !mex.tl -t
PID            Address          Name                                   !! Rn Ry Bk Lc IO Er
============== ================ ====================================== == == == == == == ==
0x0    0n0     fffff80727549f40 Idle                                    .  6  6  .  .  .  .
0x4    0n4     ffffc78831ceb040 System                                  6  1  .  1  .  1  .
0x78   0n120   ffffc78831d1a080 Registry                                .  .  .  .  .  .  .
0x230  0n560   ffffc78832ff9040 smss.exe                                .  .  .  1  .  .  .
0x2c4  0n708   ffffc7883624d140 csrss.exe                               .  .  .  .  1  .  .
0x30c  0n780   ffffc788364c8080 wininit.exe                             .  .  .  .  .  .  .
0x320  0n800   ffffc788364d5140 csrss.exe                               .  .  .  .  1  .  .
0x374  0n884   ffffc78836542080 winlogon.exe                            .  .  .  .  .  .  .
0x3a0  0n928   ffffc788363430c0 services.exe                            .  .  .  .  .  .  .
0x3cc  0n972   ffffc7883655e080 LsaIso.exe                              .  .  .  .  .  .  .
0x3d8  0n984   ffffc78836563080 lsass.exe                               .  .  .  .  .  .  .
0x308  0n776   ffffc788365bb080 svchost.exe                             .  .  .  .  .  .  .
0x404  0n1028  ffffc788365e50c0 fontdrvhost.exe                         .  .  .  .  .  .  .
0x408  0n1032  ffffc788365c5080 fontdrvhost.exe                         .  .  .  .  .  .  .
0x474  0n1140  ffffc788366950c0 svchost.exe                             .  .  .  .  .  .  .
0x4a0  0n1184  ffffc788366a3080 svchost.exe                             .  .  .  .  .  .  .
0x4f8  0n1272  ffffc788367cc0c0 dwm.exe                                 .  .  .  .  .  .  .
0x558  0n1368  ffffc78836814080 svchost.exe                             .  .  .  .  .  .  .
0x560  0n1376  ffffc78836822080 svchost.exe                             .  .  .  .  .  .  .
0x5f8  0n1528  ffffc7883688f080 svchost.exe                             .  .  .  .  .  .  .
0x60c  0n1548  ffffc78836895080 svchost.exe                             .  .  .  .  .  .  .
0x614  0n1556  ffffc78836898080 svchost.exe                             .  .  .  .  .  .  .
0x62c  0n1580  ffffc788368a8080 svchost.exe                             .  .  .  .  .  .  .
0x634  0n1588  ffffc788368ac080 svchost.exe                             .  .  .  .  .  .  .
0x63c  0n1596  ffffc788368a9080 svchost.exe                             .  .  .  .  .  .  .
0x654  0n1620  ffffc788368b0080 svchost.exe                             .  .  .  .  .  .  .
0x6e0  0n1760  ffffc78836924080 svchost.exe                             .  .  .  .  1  .  .
0x700  0n1792  ffffc7883692b080 svchost.exe                             .  .  .  .  .  .  .
0x794  0n1940  ffffc788369d20c0 svchost.exe                             .  .  .  .  .  .  .
0x7a0  0n1952  ffffc788369bf080 svchost.exe                             .  .  .  .  .  .  .
0x7b8  0n1976  ffffc788369dd080 svchost.exe                             .  .  .  .  .  .  .
0x35c  0n860   ffffc788369e30c0 svchost.exe                             .  .  .  .  .  .  .
0x80c  0n2060  ffffc78836a66080 svchost.exe                             .  .  .  .  .  .  .
0x830  0n2096  ffffc78836a79080 svchost.exe                             .  .  .  .  .  .  .
0x83c  0n2108  ffffc78836a460c0 svchost.exe                             .  .  .  .  .  .  .
0x874  0n2164  ffffc78836a50080 svchost.exe                             .  .  .  .  .  .  .
0x890  0n2192  ffffc78836a55080 svchost.exe                             .  .  .  .  .  .  .
0x8d8  0n2264  ffffc78836af5080 svchost.exe                             .  .  .  .  .  .  .
0x970  0n2416  ffffc78836b32080 VSSVC.exe                               .  .  .  .  .  .  .
0x99c  0n2460  ffffc78836ac9080 svchost.exe                             .  .  .  .  .  .  .
0x9e0  0n2528  ffffc78836bed080 svchost.exe                             .  .  .  .  .  .  .
0x9e8  0n2536  ffffc78836b350c0 svchost.exe                             .  .  .  .  .  .  .
0xa50  0n2640  ffffc78836c09080 svchost.exe                             .  .  .  .  .  .  .
0xa58  0n2648  ffffc78836c0f080 svchost.exe                             .  .  .  .  .  .  .
0xa78  0n2680  ffffc78836c17080 svchost.exe                             .  .  .  .  .  .  .
0xaa4  0n2724  ffffc78836c26080 svchost.exe                             .  .  .  .  .  .  .
0xac8  0n2760  ffffc78831dc1040 MemCompression                          .  .  .  .  .  .  .
0xb1c  0n2844  ffffc78831d79080 svchost.exe                             .  .  .  .  .  .  .
0xb6c  0n2924  ffffc78831d4e080 svchost.exe                             .  .  .  .  .  .  .
0xb74  0n2932  ffffc78831d48080 svchost.exe                             .  .  .  .  .  .  .
0xb80  0n2944  ffffc78831d72080 svchost.exe                             .  .  .  .  .  .  .
0xbc4  0n3012  ffffc78831d26080 svchost.exe                             .  .  .  .  .  .  .
0xc04  0n3076  ffffc78836dab080 svchost.exe                             .  .  .  .  .  .  .
0xc28  0n3112  ffffc78836de8080 svchost.exe                             .  .  .  .  .  .  .
0xca4  0n3236  ffffc78836e84080 svchost.exe                             .  .  .  .  .  .  .
0xd38  0n3384  ffffc78836f96080 svchost.exe                             .  .  .  .  .  .  .
0xd40  0n3392  ffffc78836eca080 svchost.exe                             .  .  .  .  .  .  .
0xd9c  0n3484  ffffc78838025080 svchost.exe                             .  .  .  .  .  .  .
0xe2c  0n3628  ffffc788380a80c0 spoolsv.exe                             .  .  .  .  .  .  .
0xe68  0n3688  ffffc788381ac0c0 svchost.exe                             .  .  .  .  .  .  .
0xf5c  0n3932  ffffc78838288080 svchost.exe                             .  .  .  .  .  .  .
0xf74  0n3956  ffffc788383a8080 svchost.exe                             .  .  .  .  .  .  .
0xf80  0n3968  ffffc78838342080 svchost.exe                             .  .  .  .  .  .  .
0xf98  0n3992  ffffc788382ec080 svchost.exe                             .  .  .  .  .  .  .
0xfa4  0n4004  ffffc788382ee080 svchost.exe                             .  .  .  .  .  .  .
0xfac  0n4012  ffffc788383fb080 svchost.exe                             .  .  .  .  .  .  .
0xfc4  0n4036  ffffc78838131080 svchost.exe                             .  .  .  .  .  .  .
0xfcc  0n4044  ffffc78838072080 IpOverUsbSvc.exe*32                     .  .  .  .  .  .  .
0xfd4  0n4052  ffffc788382aa080 sqlwriter.exe                           .  .  .  .  .  .  .
0xc44  0n3140  ffffc78838355080 svchost.exe                             .  .  .  .  .  .  .
0xc74  0n3188  ffffc788381bb080 svchost.exe                             .  .  .  .  .  .  .
0x1018 0n4120  ffffc78838188080 wlms.exe                                .  .  .  .  .  .  .
0x1044 0n4164  ffffc788382ba080 MsMpEng.exe                             .  1  .  .  .  .  .
0x1060 0n4192  ffffc788383b9080 svchost.exe                             .  .  .  .  .  .  .
0x111c 0n4380  ffffc788385a9080 sppsvc.exe                              .  .  .  .  .  .  .
0xfb8  0n4024  ffffc78838916080 svchost.exe                             .  .  .  .  .  .  .
0x1480 0n5248  ffffc78838993080 svchost.exe                             .  .  .  .  .  .  .
0x156c 0n5484  ffffc78838a1a080 svchost.exe                             .  .  .  .  .  .  .
0x1768 0n5992  ffffc78838c7d080 AggregatorHost.exe                      .  .  .  .  .  .  .
0x17ac 0n6060  ffffc78838bf1080 sihost.exe                              .  .  .  .  .  .  .
0x17e8 0n6120  ffffc78838c9e080 svchost.exe                             .  .  .  .  .  .  .
0x1518 0n5400  ffffc78838cb5080 svchost.exe                             .  .  .  .  .  .  .
0x13b8 0n5048  ffffc78838cde080 svchost.exe                             .  .  .  .  .  .  .
0x15c4 0n5572  ffffc78838cf9080 svchost.exe                             .  .  .  .  .  .  .
0x16c8 0n5832  ffffc78838cfa080 SppExtComObj.Exe                        .  .  .  .  .  .  .
0x186c 0n6252  ffffc78838dd6080 taskhostw.exe                           .  .  .  .  .  .  .
0x1898 0n6296  ffffc78838ddb0c0 svchost.exe                             .  .  .  .  .  .  .
0x18f8 0n6392  ffffc78838e430c0 explorer.exe                            5  .  .  .  1  .  .
0x1aec 0n6892  ffffc78838f650c0 svchost.exe                             .  .  .  .  .  .  .
0x1b38 0n6968  ffffc788362f1080 svchost.exe                             .  .  .  .  .  .  .
0x1838 0n6200  ffffc7883910d080 svchost.exe                             .  .  .  .  .  .  .
0x19bc 0n6588  ffffc78839117080 svchost.exe                             .  .  .  .  .  .  .
0x1714 0n5908  ffffc78839133080 unsecapp.exe                            .  .  .  .  .  .  .
0x1c88 0n7304  ffffc78839197080 SearchHost.exe                         10  .  .  .  2  .  .
0x1c90 0n7312  ffffc788391bd0c0 StartMenuExperienceHost.exe             .  .  .  .  .  .  .
0x1d64 0n7524  ffffc78839369080 Widgets.exe                             .  .  .  .  .  .  .
0x1de8 0n7656  ffffc7883936a080 RuntimeBroker.exe                       .  .  .  .  .  .  .
0x1e5c 0n7772  ffffc7883930f080 RuntimeBroker.exe                       .  .  .  .  .  .  .
0x1f24 0n7972  ffffc788387e4080 svchost.exe                             .  .  .  .  .  .  .
0x13c4 0n5060  ffffc78838d8e0c0 dllhost.exe                             .  .  .  .  .  .  .
0x220c 0n8716  ffffc78839268080 svchost.exe                             .  .  .  .  .  .  .
0x233c 0n9020  ffffc788394c80c0 ctfmon.exe                              .  .  .  .  .  .  .
0x23b4 0n9140  ffffc78838d020c0 SearchIndexer.exe                       .  .  .  .  .  .  .
0x514  0n1300  ffffc788399020c0 csrss.exe                               .  .  .  .  1  .  .
0x136c 0n4972  ffffc788399e5080 winlogon.exe                            .  .  .  .  1  .  .
0x20e8 0n8424  ffffc78839c450c0 fontdrvhost.exe                         .  .  .  .  .  .  .
0x2124 0n8484  ffffc78839c5e0c0 LogonUI.exe                             .  .  .  .  .  .  .
0x2120 0n8480  ffffc78839c4a080 dwm.exe                                 .  .  .  .  .  .  .
0x13d8 0n5080  ffffc78839d7a0c0 WUDFHost.exe                            .  .  .  .  .  .  .
0x3b0  0n944   ffffc78839d7d080 rdpclip.exe                             .  .  .  .  .  .  .
0x1fb4 0n8116  ffffc78839d9c0c0 rdpinput.exe                            .  .  .  .  .  .  .
0x1234 0n4660  ffffc78839b31080 svchost.exe                             .  .  .  .  .  .  .
0x24c0 0n9408  ffffc7883932a0c0 TabTip.exe                              .  .  .  .  .  .  .
0x255c 0n9564  ffffc78839dda080 TextInputHost.exe                       .  .  .  .  .  .  .
0x26b4 0n9908  ffffc7883914a0c0 NisSrv.exe                              .  .  .  .  .  .  .
0x11b0 0n4528  ffffc78838f240c0 SecurityHealthSystray.exe               .  .  .  .  .  .  .
0x1fbc 0n8124  ffffc7883973a080 SecurityHealthService.exe               .  .  .  .  .  .  .
0x2640 0n9792  ffffc78836df0080 taskhostw.exe                           .  .  .  .  .  .  .
0xd5c  0n3420  ffffc788362290c0 svchost.exe                             .  .  .  .  .  .  .
0xff8  0n4088  ffffc7883589b0c0 uhssvc.exe                              .  .  .  .  .  .  .
0x1068 0n4200  ffffc78835baa0c0 svchost.exe                             .  .  .  .  .  .  .
0x1540 0n5440  ffffc788393960c0 svchost.exe                             .  .  .  .  .  .  .
0x12e0 0n4832  ffffc78838ca80c0 svchost.exe                             .  .  .  .  .  .  .
0xe40  0n3648  ffffc78835ef40c0 svchost.exe                             .  .  .  .  .  .  .
0x5e4  0n1508  ffffc788393910c0 svchost.exe                             .  .  .  .  .  .  .
0x2398 0n9112  ffffc7883631d0c0 devenv.exe                              .  .  .  1  .  .  .
0xa10  0n2576  ffffc78835d380c0 PerfWatson2.exe                         .  .  .  2  .  .  .
0x6b4  0n1716  ffffc788358680c0 Microsoft.ServiceHub.Controller.exe     .  .  .  1  .  .  .
0x3c4  0n964   ffffc788395f10c0 ServiceHub.VSDetouredHost.exe           .  .  .  1  .  .  .
0x1430 0n5168  ffffc78839ef20c0 ServiceHub.IdentityHost.exe*32          .  .  .  3  .  .  .
0x8a0  0n2208  ffffc78835ed50c0 ServiceHub.SettingsHost.exe             .  .  .  2  .  .  .
0x10c0 0n4288  ffffc7883585e0c0 ServiceHub.Host.netfx.x86.exe*32        .  .  .  4  .  .  .
0x1944 0n6468  ffffc78835c8c0c0 svchost.exe                             .  .  .  .  .  .  .
0x1ae4 0n6884  ffffc788358630c0 ServiceHub.ThreadedWaitDialog.exe       .  .  .  1  .  .  .
0x2744 0n10052 ffffc788357390c0 ServiceHub.IndexingService.exe          .  .  .  1  .  .  .
0x1174 0n4468  ffffc78835ba50c0 vshost.exe                              1  .  .  .  .  .  .
0x1848 0n6216  ffffc78835ce50c0 ServiceHub.IntellicodeModelService.exe  .  .  .  .  .  .  .
0x2b0  0n688   ffffc788357320c0 vcpkgsrv.exe*32                         .  .  .  .  .  .  .
0x1ed0 0n7888  ffffc7883999e140 ServiceHub.Host.AnyCPU.exe              .  .  .  1  .  .  .
0x1334 0n4916  ffffc78835b020c0 ServiceHub.TestWindowStoreHost.exe      .  .  .  1  .  .  .
0xdc4  0n3524  ffffc78835daf080 vcpkgsrv.exe*32                         .  .  .  .  .  .  .
0xe48  0n3656  ffffc78838d8c080 ShellExperienceHost.exe                 2  .  .  .  .  .  .
0x5e8  0n1512  ffffc7883922f080 svchost.exe                             .  .  .  .  .  .  .
0x1440 0n5184  ffffc78832b7e080 RuntimeBroker.exe                       .  .  .  .  1  .  .
0x1700 0n5888  ffffc7883587a080 SystemSettingsBroker.exe                .  .  .  .  .  .  .
0x2164 0n8548  ffffc78839d81080 svchost.exe                             .  .  .  .  .  .  .
0x1b54 0n6996  ffffc7883940f080 svchost.exe                             .  .  .  .  .  .  .
0x1514 0n5396  ffffc78839ecf080 WidgetService.exe                       .  .  .  .  .  .  .
0xa04  0n2564  ffffc78839ef1080 svchost.exe                             .  .  .  .  .  .  .
0x1bd4 0n7124  ffffc78832bca180 vcpkgsrv.exe*32                         .  .  .  .  .  .  .
0x1c3c 0n7228  ffffc7883599e0c0 vcpkgsrv.exe*32                         .  .  .  .  .  .  .
0x2304 0n8964  ffffc7883682b080 dllhost.exe                             .  .  .  .  .  .  .
0x2a30 0n10800 ffffc78832acb080 cmd.exe                                 .  .  1  .  .  .  .
0x2a38 0n10808 ffffc78835ef9080 conhost.exe                             .  .  .  .  .  .  .
0x1cc0 0n7360  ffffc7883c8800c0 SearchProtocolHost.exe                  .  .  .  .  .  .  .
0x1a68 0n6760  ffffc7883becf0c0 MSBuild.exe                             .  .  .  .  .  .  .
0x2580 0n9600  ffffc7883b6eb0c0 conhost.exe                             .  .  .  .  .  .  .
0x81c  0n2076  ffffc7883bfcf0c0 SearchFilterHost.exe                    .  .  .  .  .  .  .
============== ================ ====================================== == == == == == == ==
PID            Address          Name                                   !! Rn Ry Bk Lc IO Er

Warning! Zombie process(es) detected (not displayed). Count: 7 [zombie report]
```

Here we can see that this particular thread is waiting on a pushlock.

```
3: kd> !mex.lt ffffc78831ceb040 -bk -wr WrFastMutex,WrGuardedMutex,WrMutex,WrPushLock
Process PID Thread            Id State   Time Reason
======= === ================ === ======= ==== ==========
System    4 ffffc78838a32040 780 Waiting    0 WrPushLock

Thread Count: 1
```

This thread is waiting on acquiring a pushlock because the lock it requires is not currently available. 

```
3: kd> !mex.t ffffc78838a32040
Process                   Thread                       CID       UserTime KernelTime ContextSwitches Wait Reason Time State
System (ffffc78831ceb040) ffffc78838a32040 (E|K|W|R|V) 4.780            0          0               1 WrPushLock     0 Waiting

WaitBlockList:
    Object           Type                 Other Waiters
    fffffb807d617290 SynchronizationEvent             0

# Child-SP         Return           Call Site
0 fffffb807d616b80 fffff80726a6c9d5 nt!KiSwapContext+0x76
1 fffffb807d616cc0 fffff80726a6ebb7 nt!KiSwapThread+0xab5
2 fffffb807d616e10 fffff80726a70ad6 nt!KiCommitThreadWait+0x137
3 fffffb807d616ec0 fffff80726a2aea0 nt!KeWaitForSingleObject+0x256
4 fffffb807d617260 fffff80726aee012 nt!ExfAcquirePushLockExclusiveEx+0x1a0
5 fffffb807d617310 fffff8074596112f nt!ExAcquirePushLockExclusiveEx+0x112
6 fffffb807d617350 fffff80726b07167 TestDriver+0x112f
7 fffffb807d617570 fffff80726c1bb94 nt!PspSystemThreadStartup+0x57
8 fffffb807d6175c0 0000000000000000 nt!KiStartSystemThread+0x34
```

We are going to examine the **ETHREAD** structure for this specific thread:

```
0: kd> !mex.ddt nt!_ETHREAD ffffc78838a32040

dt nt!_ETHREAD ffffc78838a32040 () Recursive: [ -r1 -r2 -r ] Verbose Normal dt
==================================================================================
   +0x000 Tcb                  : _KTHREAD
   +0x480 CreateTime           : _LARGE_INTEGER 0x01da368d`1b4b470d (12/24/2023 17:17:43 UTC)
   +0x488 ExitTime             : _LARGE_INTEGER 0xffffc788`38a324c8 (0n-62087097015096)
   +0x488 KeyedWaitChain       : _LIST_ENTRY [ 0xffffc788`38a324c8 - 0xffffc788`38a324c8 ] [EMPTY OR 1 ELEMENT]
   +0x498 PostBlockList        : _LIST_ENTRY [ 0x00000000`00000000 - 0xfffff807`459610e0 ] [INVALID]
   +0x498 ForwardLinkShadow    : (null)
   +0x4a0 StartAddress         : 0xfffff807`459610e0 Void  [generic address]
   +0x4a8 TerminationPort      : (null)
   +0x4a8 ReaperLink           : (null)
   +0x4a8 KeyedWaitValue       : (null)
   +0x4b0 ActiveTimerListLock  : 0
   +0x4b8 ActiveTimerListHead  : _LIST_ENTRY [ 0xffffc788`38a324f8 - 0xffffc788`38a324f8 ] [EMPTY OR 1 ELEMENT]
   +0x4c8 Cid                  : _CLIENT_ID
   +0x4d8 KeyedWaitSemaphore   : _KSEMAPHORE
   +0x4d8 AlpcWaitSemaphore    : _KSEMAPHORE
   +0x4f8 ClientSecurity       : _PS_CLIENT_SECURITY_CONTEXT
   +0x500 IrpList              : _LIST_ENTRY [ 0xffffc788`38a32540 - 0xffffc788`38a32540 ] [EMPTY OR 1 ELEMENT]
   +0x510 TopLevelIrp          : 0
   +0x518 DeviceToVerify       : (null)
   +0x520 Win32StartAddress    : 0xfffff807`459610e0 Void  [generic address]
   +0x528 ChargeOnlySession    : (null)
   +0x530 LegacyPowerObject    : (null)
   +0x538 ThreadListEntry      : _LIST_ENTRY [ 0xffffc788`39108578 - 0xffffc788`396517f8 ]
   +0x548 RundownProtect       : _EX_RUNDOWN_REF
   +0x550 ThreadLock           : _EX_PUSH_LOCK
   +0x558 ReadClusterSize      : 7
   +0x55c MmLockOrdering       : 0n0
   +0x560 CrossThreadFlags     : 0x5402 (0n21506)
   +0x560 Terminated           : 0y0
   +0x560 ThreadInserted       : 0y1
   +0x560 HideFromDebugger     : 0y0
   +0x560 ActiveImpersonationInfo : 0y0
   +0x560 HardErrorsAreDisabled : 0y0
   +0x560 BreakOnTermination   : 0y0
   +0x560 SkipCreationMsg      : 0y0
   +0x560 SkipTerminationMsg   : 0y0
   +0x560 CopyTokenOnOpen      : 0y0
   +0x560 ThreadIoPriority     : 0y010 (0n2)
   +0x560 ThreadPagePriority   : 0y101 (0n5)
   +0x560 RundownFail          : 0y0
   +0x560 UmsForceQueueTermination : 0y0
   +0x560 IndirectCpuSets      : 0y0
   +0x560 DisableDynamicCodeOptOut : 0y0
   +0x560 ExplicitCaseSensitivity : 0y0
   +0x560 PicoNotifyExit       : 0y0
   +0x560 DbgWerUserReportActive : 0y0
   +0x560 ForcedSelfTrimActive : 0y0
   +0x560 SamplingCoverage     : 0y0
   +0x560 ReservedCrossThreadFlags : 0y00000000 (0)
   +0x564 SameThreadPassiveFlags : 0
   +0x564 ActiveExWorker       : 0y0
   +0x564 MemoryMaker          : 0y0
   +0x564 StoreLockThread      : 0y00 (0n0)
   +0x564 ClonedThread         : 0y0
   +0x564 KeyedEventInUse      : 0y0
   +0x564 SelfTerminate        : 0y0
   +0x564 RespectIoPriority    : 0y0
   +0x564 ActivePageLists      : 0y0
   +0x564 SecureContext        : 0y0
   +0x564 ZeroPageThread       : 0y0
   +0x564 WorkloadClass        : 0y0
   +0x564 GenerateDumpOnBadHandleAccess : 0y0
   +0x564 ReservedSameThreadPassiveFlags : 0y0000000000000000000 (0)
   +0x568 SameThreadApcFlags   : 0
   +0x568 OwnsProcessAddressSpaceExclusive : 0y0
   +0x568 OwnsProcessAddressSpaceShared : 0y0
   +0x568 HardFaultBehavior    : 0y0
   +0x568 StartAddressInvalid  : 0y0
   +0x568 EtwCalloutActive     : 0y0
   +0x568 SuppressSymbolLoad   : 0y0
   +0x568 Prefetching          : 0y0
   +0x568 OwnsVadExclusive     : 0y0
   +0x569 SystemPagePriorityActive : 0y0
   +0x569 SystemPagePriority   : 0y000 (0n0)
   +0x569 AllowUserWritesToExecutableMemory : 0y0
   +0x569 AllowKernelWritesToExecutableMemory : 0y0
   +0x569 OwnsVadShared        : 0y0
   +0x569 SessionAttachActive  : 0y0
   +0x56a PasidMsrValid        : 0y0
   +0x56c CacheManagerActive   : 0 ''
   +0x56d DisablePageFaultClustering : 0 ''
   +0x56e ActiveFaultCount     : 0 ''
   +0x56f LockOrderState       : 0 ''
   +0x570 PerformanceCountLowReserved : 0
   +0x574 PerformanceCountHighReserved : 0n0
   +0x578 AlpcMessageId        : 0
   +0x580 AlpcMessage          : (null)
   +0x580 AlpcReceiveAttributeSet : 0
   +0x588 AlpcWaitListEntry    : _LIST_ENTRY [ 0x00000000`00000000 - 0x00000000`00000000 ] [INVALID]
   +0x598 ExitStatus           : 0n0
   +0x59c CacheManagerCount    : 0
   +0x5a0 IoBoostCount         : 0
   +0x5a4 IoQoSBoostCount      : 0
   +0x5a8 IoQoSThrottleCount   : 0
   +0x5ac KernelStackReference : 1
   +0x5b0 BoostList            : _LIST_ENTRY [ 0xffffc788`38a325f0 - 0xffffc788`38a325f0 ] [EMPTY OR 1 ELEMENT]
   +0x5c0 DeboostList          : _LIST_ENTRY [ 0xffffc788`38a32600 - 0xffffc788`38a32600 ] [EMPTY OR 1 ELEMENT]
   +0x5d0 BoostListLock        : 0
   +0x5d8 IrpListLock          : 0
   +0x5e0 ReservedForSynchTracking : (null)
   +0x5e8 CmCallbackListHead   : _SINGLE_LIST_ENTRY
   +0x5f0 ActivityId           : (null)
   +0x5f8 SeLearningModeListHead : _SINGLE_LIST_ENTRY
   +0x600 VerifierContext      : (null)
   +0x608 AdjustedClientToken  : (null)
   +0x610 WorkOnBehalfThread   : (null)
   +0x618 PropertySet          : _PS_PROPERTY_SET
   +0x630 PicoContext          : (null)
   +0x638 UserFsBase           : 0
   +0x640 UserGsBase           : 0
   +0x648 EnergyValues         : 0xffffc788`38a32950 _THREAD_ENERGY_VALUES
   +0x650 SelectedCpuSets      : 0
   +0x650 SelectedCpuSetsIndirect : (null)
   +0x658 Silo                 : 0xffffffff`fffffffd _EJOB
   +0x660 ThreadName           : (null)
   +0x668 SetContextState      : (null)
   +0x670 LastSoftParkElectionQos : 0 ''
   +0x671 LastSoftParkElectionWorkloadType : 0 ''
   +0x672 LastSoftParkElectionRunningType : 0 ''
   +0x673 Spare1               : 0 ''
   +0x674 HeapData             : 0
   +0x678 OwnerEntryListHead   : _LIST_ENTRY [ 0xffffc788`38a326b8 - 0xffffc788`38a326b8 ] [EMPTY OR 1 ELEMENT]
   +0x688 DisownedOwnerEntryListLock : 0
   +0x690 DisownedOwnerEntryListHead : _LIST_ENTRY [ 0xffffc788`38a326d0 - 0xffffc788`38a326d0 ] [EMPTY OR 1 ELEMENT]
   +0x6a0 LockEntries          : [6] _KLOCK_ENTRY
   +0x8e0 CmThreadInfo         : (null)
   +0x8e8 FlsData              : (null)
   +0x8f0 LastExpectedRunTime  : 0
   +0x8f4 LastSoftParkElectionRunTime : 0
   +0x8f8 LastSoftParkElectionGeneration : 0
   +0x900 LastSoftParkElectionGroupAffinity : _GROUP_AFFINITY
```

Every thread maintains a fixed list of lock entries. These entries are used to record the locks that the thread is currently holding or waiting on. This list is found in the **ETHREAD** structure's **`LockEntries`** field. To be precise, an array of **`KLOCK_ENTRY`** structures used by the thread for lock management.

```
0: kd> !mex.ddt -a6 ffffc78838a32040+0x6a0 nt!_KLOCK_ENTRY

dt -a6 ffffc78838a32040+0x6a0 nt!_KLOCK_ENTRY () Recursive: [ -r1 -r2 -r ] Verbose Normal dt
==================================================================================
[0] @ ffffc788`38a326e0 
---------------------------------------------
   +0x000 LockState            : _KLOCK_ENTRY_LOCK_STATE
   +0x000 LockUnsafe           : 0xffffc788`3a0f6f70 Void  PoolTag: Devi
   +0x000 CrossThreadReleasableAndBusyByte : 0x70 'p'
   +0x001 Reserved             : [6] "o???"
   +0x007 InTreeByte           : 0xff ''
   +0x008 SessionState         : 0x00000000`ffffffff Void    [ !ndao dps dc !handle ln ? ]
   +0x008 SessionId            : 0xffffffff (0n4294967295)
   +0x00c SessionPad           : 0
   +0x010 EntryFlags           : 0x7000100 (0n117440768)
   +0x010 EntryIndex           : 0 ''
   +0x011 WaitingByte          : 0x1 '' --> Indicates whether a thread is waiting to acquire the associated lock
   +0x012 AcquiredByte         : 0 ''
   +0x013 CrossThreadFlags     : 0x7 ''
   +0x013 HeadNodeBit          : 0y1
   +0x013 IoPriorityBit        : 0y1
   +0x013 IoQoSWaiter          : 0y1
   +0x013 Spare1               : 0y00000 (0)
   +0x010 StaticState          : 0y00000000 (0)
   +0x010 AllFlags             : 0y000001110000000000000001 (0x70001)
   +0x014 SpareFlags           : 0
   +0x018 TreeNode             : _RTL_BALANCED_NODE
   +0x030 OwnerTree            : _RTL_RB_TREE
   +0x040 WaiterTree           : _RTL_RB_TREE
   +0x030 CpuPriorityKey       : -8 ''
   +0x050 EntryLock            : 0
   +0x058 BoostBitmap          : _KLOCK_ENTRY_BOOST_BITMAP

[1] @ ffffc788`38a32740 
---------------------------------------------
   +0x000 LockState            : _KLOCK_ENTRY_LOCK_STATE
   +0x000 LockUnsafe           : (null)
   +0x000 CrossThreadReleasableAndBusyByte : 0 ''
   +0x001 Reserved             : [6] ""
   +0x007 InTreeByte           : 0 ''
   +0x008 SessionState         : (null)
   +0x008 SessionId            : 0
   +0x00c SessionPad           : 0
   +0x010 EntryFlags           : 1
   +0x010 EntryIndex           : 0x1 ''
   +0x011 WaitingByte          : 0 ''
   +0x012 AcquiredByte         : 0 ''
   +0x013 CrossThreadFlags     : 0 ''
   +0x013 HeadNodeBit          : 0y0
   +0x013 IoPriorityBit        : 0y0
   +0x013 IoQoSWaiter          : 0y0
   +0x013 Spare1               : 0y00000 (0)
   +0x010 StaticState          : 0y00000001 (0x1)
   +0x010 AllFlags             : 0y000000000000000000000000 (0)
   +0x014 SpareFlags           : 0
   +0x018 TreeNode             : _RTL_BALANCED_NODE
   +0x030 OwnerTree            : _RTL_RB_TREE
   +0x040 WaiterTree           : _RTL_RB_TREE
   +0x030 CpuPriorityKey       : 0 ''
   +0x050 EntryLock            : 0
   +0x058 BoostBitmap          : _KLOCK_ENTRY_BOOST_BITMAP

[2] @ ffffc788`38a327a0 
---------------------------------------------
   +0x000 LockState            : _KLOCK_ENTRY_LOCK_STATE
   +0x000 LockUnsafe           : (null)
   +0x000 CrossThreadReleasableAndBusyByte : 0 ''
   +0x001 Reserved             : [6] ""
   +0x007 InTreeByte           : 0 ''
   +0x008 SessionState         : (null)
   +0x008 SessionId            : 0
   +0x00c SessionPad           : 0
   +0x010 EntryFlags           : 2
   +0x010 EntryIndex           : 0x2 ''
   +0x011 WaitingByte          : 0 ''
   +0x012 AcquiredByte         : 0 ''
   +0x013 CrossThreadFlags     : 0 ''
   +0x013 HeadNodeBit          : 0y0
   +0x013 IoPriorityBit        : 0y0
   +0x013 IoQoSWaiter          : 0y0
   +0x013 Spare1               : 0y00000 (0)
   +0x010 StaticState          : 0y00000010 (0x2)
   +0x010 AllFlags             : 0y000000000000000000000000 (0)
   +0x014 SpareFlags           : 0
   +0x018 TreeNode             : _RTL_BALANCED_NODE
   +0x030 OwnerTree            : _RTL_RB_TREE
   +0x040 WaiterTree           : _RTL_RB_TREE
   +0x030 CpuPriorityKey       : 0 ''
   +0x050 EntryLock            : 0
   +0x058 BoostBitmap          : _KLOCK_ENTRY_BOOST_BITMAP

[3] @ ffffc788`38a32800 
---------------------------------------------
   +0x000 LockState            : _KLOCK_ENTRY_LOCK_STATE
   +0x000 LockUnsafe           : (null)
   +0x000 CrossThreadReleasableAndBusyByte : 0 ''
   +0x001 Reserved             : [6] ""
   +0x007 InTreeByte           : 0 ''
   +0x008 SessionState         : (null)
   +0x008 SessionId            : 0
   +0x00c SessionPad           : 0
   +0x010 EntryFlags           : 3
   +0x010 EntryIndex           : 0x3 ''
   +0x011 WaitingByte          : 0 ''
   +0x012 AcquiredByte         : 0 ''
   +0x013 CrossThreadFlags     : 0 ''
   +0x013 HeadNodeBit          : 0y0
   +0x013 IoPriorityBit        : 0y0
   +0x013 IoQoSWaiter          : 0y0
   +0x013 Spare1               : 0y00000 (0)
   +0x010 StaticState          : 0y00000011 (0x3)
   +0x010 AllFlags             : 0y000000000000000000000000 (0)
   +0x014 SpareFlags           : 0
   +0x018 TreeNode             : _RTL_BALANCED_NODE
   +0x030 OwnerTree            : _RTL_RB_TREE
   +0x040 WaiterTree           : _RTL_RB_TREE
   +0x030 CpuPriorityKey       : 0 ''
   +0x050 EntryLock            : 0
   +0x058 BoostBitmap          : _KLOCK_ENTRY_BOOST_BITMAP

[4] @ ffffc788`38a32860 
---------------------------------------------
   +0x000 LockState            : _KLOCK_ENTRY_LOCK_STATE
   +0x000 LockUnsafe           : (null)
   +0x000 CrossThreadReleasableAndBusyByte : 0 ''
   +0x001 Reserved             : [6] ""
   +0x007 InTreeByte           : 0 ''
   +0x008 SessionState         : (null)
   +0x008 SessionId            : 0
   +0x00c SessionPad           : 0
   +0x010 EntryFlags           : 4
   +0x010 EntryIndex           : 0x4 ''
   +0x011 WaitingByte          : 0 ''
   +0x012 AcquiredByte         : 0 ''
   +0x013 CrossThreadFlags     : 0 ''
   +0x013 HeadNodeBit          : 0y0
   +0x013 IoPriorityBit        : 0y0
   +0x013 IoQoSWaiter          : 0y0
   +0x013 Spare1               : 0y00000 (0)
   +0x010 StaticState          : 0y00000100 (0x4)
   +0x010 AllFlags             : 0y000000000000000000000000 (0)
   +0x014 SpareFlags           : 0
   +0x018 TreeNode             : _RTL_BALANCED_NODE
   +0x030 OwnerTree            : _RTL_RB_TREE
   +0x040 WaiterTree           : _RTL_RB_TREE
   +0x030 CpuPriorityKey       : 0 ''
   +0x050 EntryLock            : 0
   +0x058 BoostBitmap          : _KLOCK_ENTRY_BOOST_BITMAP

[5] @ ffffc788`38a328c0 
---------------------------------------------
   +0x000 LockState            : _KLOCK_ENTRY_LOCK_STATE
   +0x000 LockUnsafe           : (null)
   +0x000 CrossThreadReleasableAndBusyByte : 0 ''
   +0x001 Reserved             : [6] ""
   +0x007 InTreeByte           : 0 ''
   +0x008 SessionState         : (null)
   +0x008 SessionId            : 0
   +0x00c SessionPad           : 0
   +0x010 EntryFlags           : 5
   +0x010 EntryIndex           : 0x5 ''
   +0x011 WaitingByte          : 0 ''
   +0x012 AcquiredByte         : 0 ''
   +0x013 CrossThreadFlags     : 0 ''
   +0x013 HeadNodeBit          : 0y0
   +0x013 IoPriorityBit        : 0y0
   +0x013 IoQoSWaiter          : 0y0
   +0x013 Spare1               : 0y00000 (0)
   +0x010 StaticState          : 0y00000101 (0x5)
   +0x010 AllFlags             : 0y000000000000000000000000 (0)
   +0x014 SpareFlags           : 0
   +0x018 TreeNode             : _RTL_BALANCED_NODE
   +0x030 OwnerTree            : _RTL_RB_TREE
   +0x040 WaiterTree           : _RTL_RB_TREE
   +0x030 CpuPriorityKey       : 0 ''
   +0x050 EntryLock            : 0
   +0x058 BoostBitmap          : _KLOCK_ENTRY_BOOST_BITMAP
```

The **`WaitingByte`** indicates whether a thread is waiting to acquire the associated lock. If the **`WaitingByte`** is set (e.g., a non-zero value), it means that at least one thread is waiting for the lock to be released.

```
3: kd> !mex.ddt ffffc78838a32040+0x6a0 nt!_KLOCK_ENTRY -y WaitingByte

dt ffffc78838a32040+0x6a0 nt!_KLOCK_ENTRY -y WaitingByte () Recursive: [ -r1 -r2 -r ] Verbose Normal dt
==================================================================================
   +0x011 WaitingByte              : 0x1 '' --> Indicates whether a thread is waiting to acquire the associated lock
```

In the first lock entry, **`LockUnsafe`** is set to a non-null value **`0xffffc7883a0f6f70`**. This indicates that this particular lock entry is in use with some unsafe conditions.

This output shows that the lock at address **`0xffffc7883a0f6f70`** is **currently held by a thread** with other threads waiting to acquire it. This explains why the wait reason of this thread **`ffffc78838a32040`** is set to **`WrPushlock`**

```
3: kd> !mex.ddt nt!_EX_PUSH_LOCK 0xffffc788`3a0f6f70

dt nt!_EX_PUSH_LOCK 0xffffc788`3a0f6f70 () Recursive: [ -r1 -r2 -r ] Verbose Normal dt
==================================================================================
   +0x000 Locked               : 0y1
   +0x000 Waiting              : 0y1
   +0x000 Waking               : 0y0
   +0x000 MultipleShared       : 0y0
   +0x000 Shared               : 0y111111111111111111111011100000000111110101100001011100101001 (0xfffffb807d61729)
   +0x000 Value                : 0xfffffb80`7d617293 (0n-4945698786669)
   +0x000 Ptr                  : 0xfffffb80`7d617293 Void    [ !ndao dps dc !handle ln ? ]
```

To come up with a reasonable conclusion. It seems that this thread **`ffffc78839108040`** has terminated before it has released the pushlock, which puts **`ffffc78838a32040`** in a waiting state. When a lock is acquired and not released, the resources it protects cannot be accessed by other threads. Within the kernel, this (usually) leads to a bugcheck.
