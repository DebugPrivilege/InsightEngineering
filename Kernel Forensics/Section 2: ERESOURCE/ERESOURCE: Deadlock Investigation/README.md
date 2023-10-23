# Summary

This kernel memory dump does not contain an actual crash instead, it was initiated with NotMyFault. I want to highlight two threads that are in a deadlock situation and explain how we reached that conclusion.

Link to this Crashdump: https://mega.nz/file/ek1W3SQR#MrzNpdujllvsFhgC2bs96it4PPLimLi4sL7IMHUDkAk

# Code Sample - Deadlock

In order to demonstrate this demo. Please follow the following steps:

- Create the following 2 folders:
  1. **C:\Temp**
  2. **C:\Temp2**

Compile the following code:

```c
#include <ntifs.h>
#include <ntstrsafe.h>

// Define a structure to hold our synchronization resources.
typedef struct _SYNC_RESOURCES {
    ERESOURCE FileResource1;    // First ERESOURCE object.
    ERESOURCE FileResource2;    // Second ERESOURCE object.
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

    // DEADLOCK INTRODUCTION STARTS HERE
    // Acquire FileResource1 first.
    ExAcquireResourceExclusiveLite(&SyncRes->FileResource1, TRUE);
    // Try to acquire FileResource2.
    ExAcquireResourceExclusiveLite(&SyncRes->FileResource2, TRUE);
    // DEADLOCK INTRODUCTION ENDS HERE

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

    // Release the ERESOURCEs.
    ExReleaseResourceLite(&SyncRes->FileResource2);
    ExReleaseResourceLite(&SyncRes->FileResource1);

    return STATUS_SUCCESS;
}

NTSTATUS FileDeletionThread(PVOID Context) {
    PSYNC_RESOURCES SyncRes = (PSYNC_RESOURCES)Context;
    UNICODE_STRING uniFilePath;
    WCHAR filePathBuffer[150];
    IO_STATUS_BLOCK ioStatusBlock;

    // DEADLOCK INTRODUCTION STARTS HERE
    // Acquire FileResource2 first.
    ExAcquireResourceExclusiveLite(&SyncRes->FileResource2, TRUE);
    // Try to acquire FileResource1.
    ExAcquireResourceExclusiveLite(&SyncRes->FileResource1, TRUE);
    // DEADLOCK INTRODUCTION ENDS HERE

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

    // Release the ERESOURCEs.
    ExReleaseResourceLite(&SyncRes->FileResource1);
    ExReleaseResourceLite(&SyncRes->FileResource2);

    return STATUS_SUCCESS;
}

VOID UnloadDriver(IN PDRIVER_OBJECT DriverObject) {
    PSYNC_RESOURCES SyncRes = (PSYNC_RESOURCES)DriverObject->DeviceObject->DeviceExtension;

    // Cleanup the ERESOURCEs.
    ExDeleteResourceLite(&SyncRes->FileResource1);
    ExDeleteResourceLite(&SyncRes->FileResource2);

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
    ExInitializeResourceLite(&SyncRes->FileResource2);

    HANDLE threadHandle1, threadHandle2, threadHandle3;

    status = PsCreateSystemThread(&threadHandle1, (ACCESS_MASK)0, NULL, (PHANDLE)0, NULL, FileCreationThread, SyncRes);
    if (NT_SUCCESS(status)) {
        ZwClose(threadHandle1);
    }

    status = PsCreateSystemThread(&threadHandle2, (ACCESS_MASK)0, NULL, (PHANDLE)0, NULL, FileMoveThread, SyncRes);
    if (NT_SUCCESS(status)) {
        ZwClose(threadHandle2);
    }

    status = PsCreateSystemThread(&threadHandle3, (ACCESS_MASK)0, NULL, (PHANDLE)0, NULL, FileDeletionThread, SyncRes);
    if (NT_SUCCESS(status)) {
        ZwClose(threadHandle3);
    }

    return STATUS_SUCCESS;
}
```

Start loading this kernel driver:

```
sc.exe create TestDrv type= kernel binPath=C:\Users\User\source\repos\TestDriver\x64\Release\TestDriver.sys
```

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/01a3df3d-c849-4991-b413-5285d6454ed1)


Start the kernel driver:

```
sc.exe start TestDrv
```

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/9c67efe2-b4fd-4c8e-a4cf-5ce92a968f7a)


The kernel driver is designed to create 1,000 files in the **C:\Temp** directory. However, it won't be able to move these files to **C:\Temp2** or delete them. This is because the files never actually make it to the C:\Temp2 folder where they were supposed to be moved to. This is because of a deadlock situation.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/387b32e0-5e4f-40e8-a35d-1c300dbd46a1)


The root cause of this deadlock is the inconsistent order in which the two resources, **`FileResource1`** and **`FileResource2`**, are acquired by the **`FileMoveThread`** and **`FileDeletionThread`**. 

In this graph:

1. **`FileMoveThread`** successfully acquires **`FileResource1`** but then gets stuck because it's waiting for **`FileResource2`**.
2. **`FileDeletionThread`** successfully acquires **`FileResource2`** but can't proceed because it's waiting for **`FileResource1`**.

Each thread is waiting for a resource held by the other, leading to a deadlock.

```
+------------------------------------------+
|                                          |
|           Deadlock Scenario              |
|                                          |
+------------------------------------------+
|                                          |
|   FileMoveThread                         |                       FileDeletionThread
|   +----------------------+               |               +----------------------+
|   |                      |               |               |                      |
|   | Acquires FileResource1 | --------------- | --------------- | Waiting for FileResource1 |
|   |                      |               |               |                      |
|   +----------------------+               |               +----------------------+
|                                          |
|             **DEADLOCK**                 |
|                                          |
|   +----------------------+               |               +----------------------+
|   |                      |               |               |                      |
|   | Waiting for FileResource2 | <-------------- | -------------- | Acquires FileResource2  |
|   |                      |               |               |                      |
|   +----------------------+               |               +----------------------+
|                                          |
+------------------------------------------+
```

To fix this issue, we need to make both threads lock the resources in the same order. This way, they won't get stuck waiting for each other.

# WinDbg Walk Through - Analyzing Crashdump

Load the Crashdump in WinDbg:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/7bd3d711-dd67-4eb3-a9b0-3534f4f28273)

Start with loading the **MEX** extension:

```
2: kd> .load mex
Mex External 3.0.0.7172 Loaded!
```

Let's start with the **`!di`** command which stands for dump information. This command retrieves information about the machine from which the memory dump originated. It's especially handy when working with multiple memory dumps, as it helps distinguish the analysis related to each specific dump. This is an improved version of the **`vertarget`** command.

```
0: kd> !di
Computer Name:  Not Found
Windows 10 Kernel Version 19041 MP (4 procs) Free x64
Product: WinNt, suite: TerminalServer SingleUserTS
Edition build lab: 19041.1.amd64fre.vb_release.191206-1406
Kernel base = 0xfffff802`80800000 PsLoadedModuleList = 0xfffff802`8142a270
Debug session time: Sun Oct 22 13:57:06.294 2023 (UTC - 7:00)
System Uptime: 0 days 0:04:58.167
SystemManufacturer = Microsoft Corporation
SystemProductName = Virtual Machine
Processor: 11th Gen Intel(R) Core(TM) i7-1185G7 @ 3.00GHz
Bugcheck: D1 (FFFF9B08ED297010, 2, 0, FFFFF8027D7612D0)
Bugcheck: This is a myfault.sys dump.
KernelMode Dump Path: C:\Windows\MEMORY_INCONSISTENT_LOCKORDERING.DMP
Share Path: \\DESKTOP-OQGRG4S\ADMIN$\MEMORY_INCONSISTENT_LOCKORDERING.DMP
```

Let's start with the **`!tl -t`** command to discover threads of interest. It examines the state and wait reasons for every thread across all processes, presenting the results as statistics. This allows for a quick overview, such as determining how many processes have 4 running threads, and so on. 

```
0: kd> !tl -t
PID            Address          Name                                              !! Rn Ry Bk Lc IO Er
============== ================ ================================================= == == == == == == ==
0x0    0n0     fffff80281524a00 Idle                                               .  4  .  .  .  .  .
0x4    0n4     ffffdf8215882040 System                                             4  .  8  .  .  1  2
0x64   0n100   ffffdf82158c7080 Registry                                           .  .  .  .  .  .  .
0x1b4  0n436   ffffdf82177ba040 smss.exe                                           .  .  .  1  .  .  .
0x21c  0n540   ffffdf821b591080 csrss.exe                                          .  .  .  .  1  .  .
0x268  0n616   ffffdf821bcf3080 wininit.exe                                        .  .  .  .  .  .  .
0x278  0n632   ffffdf821bd15140 csrss.exe                                          .  .  .  .  1  .  .
0x2c8  0n712   ffffdf821bd67080 winlogon.exe                                       .  .  .  .  1  .  .
0x2f8  0n760   ffffdf821b277240 services.exe                                       .  .  .  .  .  .  .
0x30c  0n780   ffffdf821bd66080 lsass.exe                                          .  .  .  .  .  .  .
0x384  0n900   ffffdf821bddd080 svchost.exe                                        .  .  .  .  .  .  .
0x398  0n920   ffffdf821c4540c0 fontdrvhost.exe                                    .  .  .  .  .  .  .
0x3a0  0n928   ffffdf821c457300 fontdrvhost.exe                                    .  .  .  .  .  .  .
0x44   0n68    ffffdf821c4b9240 svchost.exe                                        .  .  .  .  2  .  .
0x1f4  0n500   ffffdf821c4e70c0 svchost.exe                                        .  .  .  .  .  .  .
0x2f4  0n756   ffffdf821c51a340 LogonUI.exe                                        .  .  .  .  .  .  .
0x3f8  0n1016  ffffdf821c5391c0 dwm.exe                                            .  .  .  .  .  .  .
0x430  0n1072  ffffdf821c571080 svchost.exe                                        .  .  .  .  .  .  .
0x438  0n1080  ffffdf821c572080 svchost.exe                                        .  .  .  .  .  .  .
0x46c  0n1132  ffffdf821c5bf240 svchost.exe                                        .  .  .  .  .  .  .
0x47c  0n1148  ffffdf821c5df080 svchost.exe                                        .  .  .  .  .  .  .
0x4e8  0n1256  ffffdf821c6020c0 svchost.exe                                        .  .  .  .  .  .  .
0x4f0  0n1264  ffffdf821c5e9080 svchost.exe                                        .  .  .  .  .  .  .
0x520  0n1312  ffffdf821c5ea080 svchost.exe                                        .  .  .  .  .  .  .
0x580  0n1408  ffffdf821c67c080 svchost.exe                                        .  .  .  .  .  .  .
0x5a0  0n1440  ffffdf821c687080 svchost.exe                                        .  .  .  .  .  .  .
0x5b0  0n1456  ffffdf821c6a70c0 svchost.exe                                        .  .  .  .  .  .  .
0x5c0  0n1472  ffffdf821c6c8080 svchost.exe                                        .  .  .  .  .  .  .
0x600  0n1536  ffffdf821af94300 svchost.exe                                        .  1  .  .  .  .  .
0x638  0n1592  ffffdf821b2720c0 svchost.exe                                        .  .  .  .  .  .  .
0x64c  0n1612  ffffdf821b26d0c0 svchost.exe                                        .  .  .  .  .  .  .
0x6ac  0n1708  ffffdf821c7820c0 svchost.exe                                        .  .  .  .  .  .  .
0x6f0  0n1776  ffffdf821c7ca080 svchost.exe                                        .  .  .  .  .  .  .
0x6f8  0n1784  ffffdf821c7cd080 svchost.exe                                        .  .  .  .  .  .  .
0x740  0n1856  ffffdf8215911080 svchost.exe                                        .  .  .  .  .  .  .
0x750  0n1872  ffffdf82158c6080 svchost.exe                                        .  .  .  .  .  .  .
0x78c  0n1932  ffffdf82158c0080 svchost.exe                                        .  .  .  .  .  .  .
0x7b8  0n1976  ffffdf821587b080 svchost.exe                                        .  .  .  .  .  .  .
0x814  0n2068  ffffdf8215947080 svchost.exe                                        .  .  .  .  .  .  .
0x870  0n2160  ffffdf821591c080 svchost.exe                                        .  .  .  .  1  .  .
0x88c  0n2188  ffffdf821c9020c0 svchost.exe                                        .  .  .  .  .  .  .
0x8a4  0n2212  ffffdf8215929080 svchost.exe                                        .  .  .  .  .  .  .
0x8bc  0n2236  ffffdf821592b080 svchost.exe                                        .  .  .  .  .  .  .
0x8f8  0n2296  ffffdf821c959080 svchost.exe                                        .  .  .  .  .  .  .
0x900  0n2304  ffffdf821c95a080 VSSVC.exe                                          .  .  .  .  .  .  .
0x988  0n2440  ffffdf821c9a7040 MemCompression                                     .  .  1  .  .  .  .
0x9d8  0n2520  ffffdf821c9a9240 svchost.exe                                        .  .  .  .  .  .  .
0x9e8  0n2536  ffffdf821c9c20c0 svchost.exe                                        .  .  .  .  .  .  .
0xa30  0n2608  ffffdf821cab70c0 svchost.exe                                        .  .  .  .  .  .  .
0xa38  0n2616  ffffdf821cab8080 svchost.exe                                        .  .  .  .  .  .  .
0xa60  0n2656  ffffdf821cabd080 svchost.exe                                        .  .  .  .  .  .  .
0xac8  0n2760  ffffdf821cadc080 svchost.exe                                        .  .  .  .  .  .  .
0xb04  0n2820  ffffdf821cb53080 svchost.exe                                        .  .  .  .  .  .  .
0xb48  0n2888  ffffdf821cb52080 svchost.exe                                        .  .  .  .  .  .  .
0xb50  0n2896  ffffdf821cb58080 svchost.exe                                        .  .  .  .  .  .  .
0xb58  0n2904  ffffdf821cb5b080 svchost.exe                                        .  .  .  .  .  .  .
0xba8  0n2984  ffffdf821cbb2080 svchost.exe                                        .  .  .  .  .  .  .
0x328  0n808   ffffdf821cc07080 svchost.exe                                        .  .  .  .  .  .  .
0xc14  0n3092  ffffdf821cc3c0c0 spoolsv.exe                                        .  .  .  .  .  .  .
0xc60  0n3168  ffffdf821cc7c0c0 svchost.exe                                        .  .  .  .  .  .  .
0xd08  0n3336  ffffdf821cc8d0c0 svchost.exe                                        .  .  .  .  .  .  .
0xd10  0n3344  ffffdf821cccc080 svchost.exe                                        .  .  .  .  .  .  .
0xd18  0n3352  ffffdf821ccce080 IpOverUsbSvc.exe*32                                .  .  .  .  .  .  .
0xd20  0n3360  ffffdf821ccd0080 svchost.exe                                        .  .  .  .  .  .  .
0xd2c  0n3372  ffffdf821ccd2080 svchost.exe                                        .  .  .  .  .  .  .
0xd38  0n3384  ffffdf821c95e080 sqlwriter.exe                                      .  .  .  .  .  .  .
0xd44  0n3396  ffffdf821cd3f080 svchost.exe                                        .  .  .  .  .  .  .
0xd74  0n3444  ffffdf821cd760c0 svchost.exe                                        .  .  .  .  .  .  .
0xd84  0n3460  ffffdf821cd43080 MsMpEng.exe                                        .  1  5  .  .  .  .
0xd90  0n3472  ffffdf821cd7a080 svchost.exe                                        .  .  .  .  .  .  .
0xdac  0n3500  ffffdf821cd83080 wlms.exe                                           .  .  .  .  .  .  .
0xe0c  0n3596  ffffdf821d048080 svchost.exe                                        .  .  .  .  .  .  .
0xe1c  0n3612  ffffdf821d049080 svchost.exe                                        .  .  .  .  .  .  .
0xea4  0n3748  ffffdf821d0b30c0 sppsvc.exe                                         .  .  .  .  .  .  .
0xf78  0n3960  ffffdf821cea0080 svchost.exe                                        .  .  .  .  .  .  .
0x1058 0n4184  ffffdf821d1e1080 csrss.exe                                          .  .  .  .  1  .  .
0x108c 0n4236  ffffdf821cfb40c0 winlogon.exe                                       .  .  .  .  .  .  .
0x1104 0n4356  ffffdf821d278300 WUDFHost.exe                                       .  .  1  .  .  .  .
0x113c 0n4412  ffffdf821d27c080 fontdrvhost.exe                                    .  .  .  .  .  .  .
0x1154 0n4436  ffffdf821d2eb080 dwm.exe                                            .  .  .  .  .  .  .
0x139c 0n5020  ffffdf821dab90c0 svchost.exe                                        .  .  .  .  .  .  .
0x1030 0n4144  ffffdf821ce9e340 svchost.exe                                        .  .  .  .  .  .  .
0x12dc 0n4828  ffffdf821dadb0c0 SppExtComObj.Exe                                   .  .  .  .  .  .  .
0x1288 0n4744  ffffdf821da8a080 svchost.exe                                        .  .  .  .  .  .  .
0xfec  0n4076  ffffdf821dbe0080 svchost.exe                                        .  .  .  .  .  .  .
0x1274 0n4724  ffffdf821dc4c340 svchost.exe                                        .  .  1  .  .  .  .
0x1468 0n5224  ffffdf821dc502c0 WmiPrvSE.exe                                       .  .  .  .  .  .  .
0x14c0 0n5312  ffffdf821dca3080 svchost.exe                                        .  .  .  .  .  .  .
0x1538 0n5432  ffffdf821dcd0080 svchost.exe                                        .  .  .  .  .  .  .
0x1570 0n5488  ffffdf821dcd7080 svchost.exe                                        .  .  .  .  .  .  .
0x1594 0n5524  ffffdf821dcf3080 svchost.exe                                        .  .  .  .  .  .  .
0x15d0 0n5584  ffffdf821dcd1080 svchost.exe                                        .  .  .  .  .  .  .
0x160c 0n5644  ffffdf821dcf0080 MoUsoCoreWorker.exe                                .  .  .  .  1  .  .
0x1788 0n6024  ffffdf821dda8080 svchost.exe                                        .  .  .  .  .  .  .
0x17bc 0n6076  ffffdf821e005080 ctfmon.exe                                         .  .  .  .  .  .  .
0x16b0 0n5808  ffffdf821dc9b080 rdpclip.exe                                        .  .  .  .  .  .  .
0xc4   0n196   ffffdf821e0e12c0 sihost.exe                                         .  .  .  .  .  .  .
0x1804 0n6148  ffffdf821e12e080 svchost.exe                                        .  .  .  .  .  .  .
0x1824 0n6180  ffffdf821e130080 svchost.exe                                        .  .  .  .  .  .  .
0x1888 0n6280  ffffdf821e138080 svchost.exe                                        .  .  .  .  .  .  .
0x1894 0n6292  ffffdf821e136080 taskhostw.exe                                      .  .  .  .  .  .  .
0x1988 0n6536  ffffdf821e24b080 svchost.exe                                        .  .  .  .  .  .  .
0x19cc 0n6604  ffffdf821e24f080 rdpinput.exe                                       .  .  .  .  .  .  .
0x1a7c 0n6780  ffffdf821e2b5080 explorer.exe                                       6  .  .  1  2  .  .
0x1ad0 0n6864  ffffdf821e165080 svchost.exe                                        .  .  1  .  .  .  .
0x1b7c 0n7036  ffffdf821e36a080 svchost.exe                                        .  .  .  .  .  .  .
0x18d4 0n6356  ffffdf821e41f080 dllhost.exe                                        .  .  .  .  .  .  .
0x1968 0n6504  ffffdf821e420080 dllhost.exe                                        .  .  .  .  .  .  .
0x1ab0 0n6832  ffffdf821e494080 TabTip.exe                                         .  .  .  .  .  .  .
0x1c44 0n7236  ffffdf821e4be080 TrustedInstaller.exe                               .  .  .  .  1  .  .
0x1c70 0n7280  ffffdf821e4bf080 NisSrv.exe                                         .  .  .  .  .  .  .
0x1c9c 0n7324  ffffdf821e4bd080 TiWorker.exe                                       .  .  .  1  .  .  .
0x1e24 0n7716  ffffdf821e590080 SearchApp.exe                                      7  .  .  .  .  .  .
0x1e70 0n7792  ffffdf821e5cd240 GenValObj.exe                                      .  .  .  .  .  .  .
0x1ea8 0n7848  ffffdf821e8550c0 RuntimeBroker.exe                                  .  .  .  .  1  .  .
0x1eb0 0n7856  ffffdf821e8470c0 SecurityHealthService.exe                          .  .  .  .  .  .  .
0x1f28 0n7976  ffffdf821b5a9080 svchost.exe                                        .  .  .  .  .  .  .
0x1fdc 0n8156  ffffdf821ea84080 StartMenuExperienceHost.exe                        .  .  .  .  .  .  .
0x14fc 0n5372  ffffdf821e495080 RuntimeBroker.exe                                  .  .  .  .  .  .  .
0x1a9c 0n6812  ffffdf821ecc5080 SearchApp.exe                                     15  .  .  .  1  .  .
0x202c 0n8236  ffffdf821dfb4080 SearchIndexer.exe                                  .  1  .  .  .  .  .
0x2388 0n9096  ffffdf821f02b2c0 ShellExperienceHost.exe                            .  .  .  .  .  .  .
0x2190 0n8592  ffffdf821eee8240 RuntimeBroker.exe                                  .  .  .  .  .  .  .
0x1d94 0n7572  ffffdf821f263080 audiodg.exe                                        .  .  .  .  .  .  .
0x105c 0n4188  ffffdf821f2be080 RuntimeBroker.exe                                  .  .  .  .  .  .  .
0x228  0n552   ffffdf821ea53080 GameBar.exe                                        .  .  .  .  .  .  .
0x810  0n2064  ffffdf821f3f2080 svchost.exe                                        .  .  .  .  .  .  .
0x630  0n1584  ffffdf821e890080 GameBarFTServer.exe                                .  .  .  .  .  .  .
0x240c 0n9228  ffffdf821e87b080 RuntimeBroker.exe                                  .  .  .  .  .  .  .
0x248c 0n9356  ffffdf821e7f9080 svchost.exe                                        .  .  .  .  .  .  .
0x2688 0n9864  ffffdf821f02d300 smartscreen.exe                                    .  .  .  .  .  .  .
0x26dc 0n9948  ffffdf821ea52080 SecurityHealthSystray.exe                          .  .  .  .  .  .  .
0x27f0 0n10224 ffffdf821f07e080 svchost.exe                                        .  .  .  .  1  .  .
0x2368 0n9064  ffffdf821f95c0c0 svchost.exe                                        .  .  .  .  .  .  .
0x1b90 0n7056  ffffdf821e691080 svchost.exe                                        .  .  .  .  .  .  .
0x210c 0n8460  ffffdf8217c11300 svchost.exe                                        .  .  .  .  .  .  .
0x284c 0n10316 ffffdf821fe760c0 PerfWatson2.exe                                    .  .  .  1  .  .  .
0x758  0n1880  ffffdf821f2540c0 MpCopyAccelerator.exe                              .  .  .  .  .  .  .
0x285c 0n10332 ffffdf821f2a70c0 OneDrive.exe                                       .  .  .  .  .  .  .
0x14d0 0n5328  ffffdf821e79b240 SearchProtocolHost.exe                             .  .  .  .  .  .  .
0x2b1c 0n11036 ffffdf821f7430c0 Microsoft.SharePoint.exe                           .  .  .  .  .  .  .
0x2294 0n8852  ffffdf821f4c4080 svchost.exe                                        .  .  .  .  .  .  .
0x2e24 0n11812 ffffdf82208a60c0 TextInputHost.exe                                  .  .  .  .  .  .  .
0x2974 0n10612 ffffdf821fef3240 SgrmBroker.exe                                     .  .  .  .  .  .  .
0xda8  0n3496  ffffdf821f487080 svchost.exe                                        .  .  .  .  1  .  .
0x2a54 0n10836 ffffdf821a10b080 uhssvc.exe                                         .  .  .  .  .  .  .
0x3028 0n12328 ffffdf822035d080 svchost.exe                                        .  .  .  .  .  .  .
0x317c 0n12668 ffffdf8220368080 svchost.exe                                        .  .  .  .  .  .  .
0x31c0 0n12736 ffffdf821e713080 svchost.exe                                        .  .  .  .  .  .  .
0x33d4 0n13268 ffffdf821f293080 wuauclt.exe                                        .  .  .  .  1  .  .
0x3244 0n12868 ffffdf8217a67080 SearchProtocolHost.exe                             .  .  .  .  .  .  .
0x3394 0n13204 ffffdf821e790240 dllhost.exe                                        .  .  .  .  .  .  .
0x31f0 0n12784 ffffdf821ea1a080 taskhostw.exe                                      .  .  .  .  1  .  .
0xa40  0n2624  ffffdf821f9442c0 MicrosoftEdgeUpdate.exe*32                         .  .  .  .  .  .  .
0xae8  0n2792  ffffdf8220ce32c0 MicrosoftEdgeUpdate.exe*32                         .  .  .  .  .  .  .
0xa08  0n2568  ffffdf821fcde2c0 CompatTelRunner.exe                                .  .  .  1  .  .  .
0x1ec8 0n7880  ffffdf821a00b2c0 VSIXAutoUpdate.exe*32                              .  .  1  .  .  .  .
0x3288 0n12936 ffffdf821f2942c0 WMIADAP.exe                                        .  .  .  .  .  .  .
0x3038 0n12344 ffffdf821f6e72c0 OneDriveStandaloneUpdater.exe                      .  .  .  .  .  .  .
0x3190 0n12688 ffffdf82219f00c0 WmiPrvSE.exe                                       .  .  .  .  .  .  .
0x2cc8 0n11464 ffffdf822082e080 conhost.exe                                        .  .  .  .  .  .  .
0x1938 0n6456  ffffdf821f957080 svchost.exe                                        .  .  .  .  .  .  .
0x10d8 0n4312  ffffdf8220350080 conhost.exe                                        .  .  .  .  .  .  .
0x2678 0n9848  ffffdf82200e0300 CompatTelRunner.exe                                .  .  1  1  .  .  .
0x2fa4 0n12196 ffffdf821f6c10c0 SearchFilterHost.exe                               .  .  .  .  .  .  .
0x2da8 0n11688 ffffdf821f3ea0c0 msedge.exe                                         .  .  3  .  1  .  .
0x2514 0n9492  ffffdf821f69f0c0 msedge.exe                                         .  .  .  .  .  .  .
0x748  0n1864  ffffdf821f6190c0 msedge.exe                                         .  .  .  .  .  .  .
0x29c  0n668   ffffdf821598d080 msedge.exe                                         .  .  .  .  .  .  .
0x8dc  0n2268  ffffdf821f284080 msedge.exe                                         .  .  .  .  .  .  .
0x18fc 0n6396  ffffdf8220390080 RuntimeBroker.exe                                  .  .  .  .  .  .  .
0x2240 0n8768  ffffdf821e8db080 cmd.exe                                            .  .  .  .  .  .  .
0x2874 0n10356 ffffdf821f3e0080 conhost.exe                                        .  .  .  .  .  .  .
0x2b7c 0n11132 ffffdf821e9c5340 MicrosoftEdgeUpdate.exe*32                         .  .  .  1  .  .  .
0x275c 0n10076 ffffdf821f2430c0 MicrosoftEdge_X64_118.0.2088.61_118.0.2088.46.exe  .  .  .  1  .  .  .
0x24d8 0n9432  ffffdf821e589080 setup.exe                                          .  .  .  .  .  .  .
0xb7c  0n2940  ffffdf821b33d080 msedge.exe                                         .  .  6  .  .  .  .
0x339c 0n13212 ffffdf821ea92080 msedge.exe                                         .  .  .  .  .  .  .
0x27fc 0n10236 ffffdf821fe2a080 msedge.exe                                         .  .  .  .  .  .  .
0x2b54 0n11092 ffffdf821ecb7080 notmyfault64.exe                                   .  1  .  .  .  .  .
0x1524 0n5412  ffffdf8217bbf080 msedge.exe                                         .  .  9  .  .  .  .
============== ================ ================================================= == == == == == == ==
PID            Address          Name                                              !! Rn Ry Bk Lc IO Er
```

There are two threads that are waiting on an ERESOURCE as we can see from the output above. 

```
0: kd> !mex.lt ffffdf8215882040 -wr WrResource
Process PID Thread             Id State     Time Reason
======= === ================ ==== ======= ====== ==========
System    4 ffffdf821f4b8040 3304 Waiting 3s.218 WrResource
System    4 ffffdf8217a30040 19d8 Waiting 2s.343 WrResource

Thread Count: 2
```

This thread has been waiting 3s.218 on an ERESOURCE **`ffffdf821ce68448`** owned exclusively by System thread **`ffffdf8217a30040`**

```
0: kd> !mex.t ffffdf821f4b8040
Process                   Thread                       CID       UserTime KernelTime ContextSwitches Wait Reason   Time State
System (ffffdf8215882040) ffffdf821f4b8040 (E|K|W|R|V) 4.3304           0          0              18 WrResource  3s.218 Waiting

WaitBlockList:
    Object           Type                 Other Waiters
    ffffdc8b3e30f230 SynchronizationEvent             0

Priority:
    Current Base Decrement ForegroundBoost IO Page
    10      8    0         32              0  5

Thread at address 0000000000060001 is invalid. Not adding to waiter list.
# Child-SP         Return           Call Site                               Info                                                                             Source
0 ffffdc8b3e30edd0 fffff80280a0c800 nt!KiSwapContext+0x76                                                                                                    
1 ffffdc8b3e30ef10 fffff80280a0bd2f nt!KiSwapThread+0x500                                                                                                    
2 ffffdc8b3e30efc0 fffff80280a0b5d3 nt!KiCommitThreadWait+0x14f                                                                                              
3 ffffdc8b3e30f060 fffff80280a0e4ad nt!KeWaitForSingleObject+0x233                                                                                           
4 ffffdc8b3e30f150 fffff80280a08eee nt!ExpWaitForResource+0x6d                                                                                               
5 ffffdc8b3e30f1d0 fffff8027d7013c4 nt!ExAcquireResourceExclusiveLite+0x1fe Thread: ffffdf8217a30040 (System) exclusive owner of EResource: ffffdf821ce68448 
6 ffffdc8b3e30f260 fffff80280b55855 TestDriver!FileMoveThread+0x34                                                                                           C:\Users\Admin\source\repos\TestDriver\TestDriver\Driver.c @ 53
7 ffffdc8b3e30f590 fffff80280bfe808 nt!PspSystemThreadStartup+0x55                                                                                           
8 ffffdc8b3e30f5e0 0000000000000000 nt!KiStartSystemThread+0x28 
```

Here we can see that thread **`ffffdf8217a30040`** is waiting to acquire this ERESOURCE object. 

```
0: kd> !mex.eresource ffffdf821ce68448
nt!_ERESOURCE @ ffffdf821ce68448 [verbose]
Contention count: 1

Exclusive owner:
Thread at address 0000000000060001 is invalid. Not adding to waiter list.
    Process Thread             Time Wait Function
    ======= ================ ====== ==================================
    System  ffffdf8217a30040 2s.343 TestDriver!FileDeletionThread+0x38

Exclusive Waiters: 0
Shared Waiters: 0
```

This thread has been waiting 2s.343 on an ERESOURCE **`ffffdf821ce683e0`** owned exclusively by System thread **`ffffdf821f4b8040`**

```
0: kd> !mex.t ffffdf8217a30040
Process                   Thread                       CID       UserTime KernelTime ContextSwitches Wait Reason   Time State
System (ffffdf8215882040) ffffdf8217a30040 (E|K|W|R|V) 4.19d8           0          0              18 WrResource  2s.343 Waiting

WaitBlockList:
    Object           Type                 Other Waiters
    ffffdc8b3e3473a0 SynchronizationEvent             0

Priority:
    Current Base Decrement ForegroundBoost IO Page
    10      8    0         32              0  5

Thread at address 0000000000060001 is invalid. Not adding to waiter list.
# Child-SP         Return           Call Site                               Info                                                                             Source
0 ffffdc8b3e346f40 fffff80280a0c800 nt!KiSwapContext+0x76                                                                                                    
1 ffffdc8b3e347080 fffff80280a0bd2f nt!KiSwapThread+0x500                                                                                                    
2 ffffdc8b3e347130 fffff80280a0b5d3 nt!KiCommitThreadWait+0x14f                                                                                              
3 ffffdc8b3e3471d0 fffff80280a0e4ad nt!KeWaitForSingleObject+0x233                                                                                           
4 ffffdc8b3e3472c0 fffff80280a08eee nt!ExpWaitForResource+0x6d                                                                                               
5 ffffdc8b3e347340 fffff8027d7012b8 nt!ExAcquireResourceExclusiveLite+0x1fe Thread: ffffdf821f4b8040 (System) exclusive owner of EResource: ffffdf821ce683e0 
6 ffffdc8b3e3473d0 fffff80280b55855 TestDriver!FileDeletionThread+0x38                                                                                       C:\Users\Admin\source\repos\TestDriver\TestDriver\Driver.c @ 100
7 ffffdc8b3e347590 fffff80280bfe808 nt!PspSystemThreadStartup+0x55                                                                                           
8 ffffdc8b3e3475e0 0000000000000000 nt!KiStartSystemThread+0x28
```

This thread **`ffffdf821f4b8040`** is waiting to acquire this ERESOURCE object.

```
0: kd> !mex.eresource ffffdf821ce683e0
nt!_ERESOURCE @ ffffdf821ce683e0 [verbose]
Contention count: 2

Exclusive owner:
Thread at address 0000000000060001 is invalid. Not adding to waiter list.
    Process Thread             Time Wait Function
    ======= ================ ====== ==============================
    System  ffffdf821f4b8040 3s.218 TestDriver!FileMoveThread+0x34

Exclusive Waiters: 0
Shared Waiters: 0
```

To summarize this:

1. Thread **`ffffdf821f4b8040`** is waiting to acquire the ERESOURCE at **`ffffdf821ce683e0`** and has been waiting for approximately 3.218 seconds. This resource is owned exclusively by thread **`ffffdf8217a30040`**.
2. Thread **`ffffdf8217a30040`** is waiting to acquire the ERESOURCE at **`ffffdf821ce68448`** and has been waiting for 2.343 seconds. This resource is owned exclusively by thread **`ffffdf821f4b8040`**.

Both threads **`ffffdf821f4b8040`** and **`ffffdf8217a30040`** are stuck in a waiting state due to inconsistent lock ordering, leading to a deadlock. Thread **`ffffdf821f4b8040`** holds the lock on ERESOURCE **`ffffdf821ce68448`** that thread **`ffffdf8217a30040`** needs, and has been waiting for approximately 3.218 seconds. 

Thread **`ffffdf8217a30040`** holds the lock on ERESOURCE **`ffffdf821ce683e0`** that thread **`ffffdf821f4b8040`** needs, and has been waiting for approximately 2.343 seconds. Neither thread can proceed until the other releases its lock, and neither will release its lock until it can proceed.
