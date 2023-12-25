# Description

This kind of lock acquisition pattern is a logic error and should be avoided. A thread should not attempt to acquire a lock it already holds, as this can lead to deadlocks, particularly with locks that do not support recursive acquisition. In the case of Pushlocks, as used here, the lock is not designed to support recursive acquisition in different modes (exclusive and shared) within the same thread.

Link to this Memory Dump: https://mega.nz/file/XsVwjSaS#GCeXuN0-t0STKkXS8RfaxqF8A3yfwuFqEh2wHuYGHZI

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

    // Try to acquire the Pushlock recursively in shared mode.
    ExAcquirePushLockShared(&SyncRes->FilePushLock); // This should not be allowed

    ExReleasePushLockExclusive(&SyncRes->FilePushLock); // Release the Pushlock.
    KeLeaveCriticalRegion(); // Re-enable normal kernel APCs.

    return STATUS_SUCCESS;
}

// Thread function for moving files.
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
    }

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

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/07282fed-5afe-447a-944e-3b3d5736543e)


Start the kernel driver:

```
sc.exe start TestDrv
```

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/1ff0e04a-f6d7-4ea5-ac36-25076a9c13a1)


We can see that 1000 files were created in the **C:\Temp** folder.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/8272742e-51fe-47e3-9da1-f24ddacad235)


However, the files have not been renamed and moved to the **C:\Temp2** folder.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/c102bd11-4539-441a-bad3-e135c8e455e1)


Now delete all the files from the **C:\Temp** folder. While the driver is running, start obtaining a kernel memory dump. Once done, we can start stop and delete the driver.

# WinDbg Walk Through - Analysis

Load the Memory Dump in WinDbg:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/6448c0da-d28a-4bb0-a2bc-76c1e4c36d51)


Let's start with the **`!mex.di`** command which stands for dump information. This command retrieves information about the machine from which the memory dump originated. It's especially handy when working with multiple memory dumps, as it helps distinguish the analysis related to each specific dump. This is an improved version of the **`vertarget`** command.

```
0: kd> !mex.di
Computer Name:  Not Found
Windows 10 Kernel Version 22621 MP (6 procs) Free x64
Product: WinNt, suite: TerminalServer SingleUserTS
Edition build lab: 22621.2506.amd64fre.ni_release_svc_prod3.231018-1809
Kernel base = 0xfffff802`54c00000 PsLoadedModuleList = 0xfffff802`558134a0
Debug session time: Sun Dec 24 03:50:20.398 2023 (UTC - 8:00)
System Uptime: 0 days 0:53:47.161
SystemManufacturer = Microsoft Corporation
SystemProductName = Virtual Machine
Processor: 11th Gen Intel(R) Core(TM) i7-1185G7 @ 3.00GHz
Bugcheck: 161 (5461736B6D6772, 0, 0, 0)
```

Let's start with the **`!tl -t`** command to discover threads of interest. It examines the state and wait reasons for every thread across all processes, presenting the results as statistics. This allows for a quick overview, such as determining how many processes have 4 running threads, and so on.

```
0: kd> !mex.tl -t
PID           Address          Name                                   !! Rn Ry Bk Lc IO Er
============= ================ ====================================== == == == == == == ==
0x0    0n0    fffff80255949f40 Idle                                    .  6  6  .  .  .  .
0x4    0n4    ffff9b84742ea040 System                                  6  .  .  4  .  1  .
0x78   0n120  ffff9b8474315080 Registry                                .  .  .  .  .  .  .
0x234  0n564  ffff9b84776070c0 smss.exe                                .  .  .  1  .  .  .
0x2c0  0n704  ffff9b84770ec140 csrss.exe                               .  .  .  .  1  .  .
0x308  0n776  ffff9b8478b25080 wininit.exe                             .  .  .  .  .  .  .
0x31c  0n796  ffff9b8478b32140 csrss.exe                               .  .  .  .  1  .  .
0x370  0n880  ffff9b8478ba9080 winlogon.exe                            .  .  .  .  .  .  .
0x38c  0n908  ffff9b8478be5080 services.exe                            .  .  .  .  .  .  .
0x3b4  0n948  ffff9b8478b3b0c0 LsaIso.exe                              .  .  .  .  .  .  .
0x3c0  0n960  ffff9b8478ba8080 lsass.exe                               .  .  .  .  .  .  .
0x320  0n800  ffff9b8478d3e080 svchost.exe                             .  .  .  .  .  .  .
0x3e4  0n996  ffff9b8478d370c0 fontdrvhost.exe                         .  .  .  .  .  .  .
0x3d8  0n984  ffff9b8478d3f080 fontdrvhost.exe                         .  .  .  .  .  .  .
0x450  0n1104 ffff9b8478e230c0 svchost.exe                             .  .  .  .  .  .  .
0x480  0n1152 ffff9b8478e3b080 svchost.exe                             .  .  .  .  .  .  .
0x4e4  0n1252 ffff9b8478f440c0 dwm.exe                                 .  .  .  .  .  .  .
0x520  0n1312 ffff9b8478f53080 svchost.exe                             .  1  .  .  .  .  .
0x57c  0n1404 ffff9b8478fb4080 svchost.exe                             .  .  .  .  .  .  .
0x5a8  0n1448 ffff9b8478fbd080 svchost.exe                             .  .  .  .  .  .  .
0x5b0  0n1456 ffff9b8478f51080 svchost.exe                             .  .  .  .  .  .  .
0x600  0n1536 ffff9b8479011080 svchost.exe                             .  .  .  .  .  .  .
0x610  0n1552 ffff9b8479016080 svchost.exe                             .  .  .  .  .  .  .
0x67c  0n1660 ffff9b84790a0080 svchost.exe                             .  .  .  .  .  .  .
0x684  0n1668 ffff9b84790a3080 svchost.exe                             .  .  .  .  .  .  .
0x6d0  0n1744 ffff9b84790f5080 svchost.exe                             .  .  .  .  .  .  .
0x6f8  0n1784 ffff9b847908c080 svchost.exe                             .  .  .  .  .  .  .
0x720  0n1824 ffff9b847914c080 svchost.exe                             .  .  .  .  .  .  .
0x72c  0n1836 ffff9b847914a080 svchost.exe                             .  .  .  .  .  .  .
0x738  0n1848 ffff9b8479151080 svchost.exe                             .  .  .  .  .  .  .
0x74c  0n1868 ffff9b8479166080 svchost.exe                             .  .  .  .  .  .  .
0x788  0n1928 ffff9b847916d0c0 svchost.exe                             .  .  .  .  .  .  .
0x7ac  0n1964 ffff9b84791e00c0 svchost.exe                             .  .  .  .  .  .  .
0x7c4  0n1988 ffff9b84791ea080 svchost.exe                             .  .  .  .  .  .  .
0x7cc  0n1996 ffff9b84791eb080 svchost.exe                             .  .  .  .  .  .  .
0x940  0n2368 ffff9b84792a0080 svchost.exe                             .  .  .  .  .  .  .
0x958  0n2392 ffff9b847935b0c0 VSSVC.exe                               .  .  .  .  .  .  .
0x968  0n2408 ffff9b847935f080 svchost.exe                             .  .  .  .  .  .  .
0x990  0n2448 ffff9b8479330080 svchost.exe                             .  .  .  .  .  .  .
0x9e8  0n2536 ffff9b847934b080 svchost.exe                             .  .  .  .  .  .  .
0x9f8  0n2552 ffff9b847936c0c0 svchost.exe                             .  .  .  .  .  .  .
0xa00  0n2560 ffff9b847934a080 svchost.exe                             .  .  .  .  .  .  .
0xa14  0n2580 ffff9b8479374080 svchost.exe                             .  .  .  .  .  .  .
0xa84  0n2692 ffff9b8474309040 MemCompression                          .  .  .  .  .  .  .
0xae4  0n2788 ffff9b84743e0080 svchost.exe                             .  .  .  .  .  .  .
0xaec  0n2796 ffff9b84743e4080 svchost.exe                             .  .  .  .  .  .  .
0xaf8  0n2808 ffff9b84743d1080 svchost.exe                             .  .  .  .  .  .  .
0xb48  0n2888 ffff9b8474386080 svchost.exe                             .  .  .  .  .  .  .
0xb60  0n2912 ffff9b847439e080 svchost.exe                             .  .  .  .  .  .  .
0xb68  0n2920 ffff9b84743a2080 svchost.exe                             .  .  .  .  .  .  .
0x980  0n2432 ffff9b84795cf080 svchost.exe                             .  .  .  .  .  .  .
0xbcc  0n3020 ffff9b84795df080 svchost.exe                             .  .  .  .  .  .  .
0xc34  0n3124 ffff9b84796b9080 svchost.exe                             .  .  .  .  .  .  .
0xc78  0n3192 ffff9b84796440c0 svchost.exe                             .  .  .  .  .  .  .
0xd00  0n3328 ffff9b84796240c0 svchost.exe                             .  .  .  .  .  .  .
0xd54  0n3412 ffff9b8479773080 svchost.exe                             .  .  .  .  .  .  .
0xd5c  0n3420 ffff9b8479772080 svchost.exe                             .  .  .  .  .  .  .
0xdb0  0n3504 ffff9b847976e080 svchost.exe                             .  .  .  .  .  .  .
0xe2c  0n3628 ffff9b84798660c0 spoolsv.exe                             .  .  .  .  .  .  .
0xea8  0n3752 ffff9b84798cf0c0 svchost.exe                             .  .  .  .  .  .  .
0xf80  0n3968 ffff9b8479314080 svchost.exe                             .  .  .  .  .  .  .
0xf90  0n3984 ffff9b8479920080 svchost.exe                             .  .  .  .  .  .  .
0xf98  0n3992 ffff9b8479921080 svchost.exe                             .  .  .  .  .  .  .
0xfa0  0n4000 ffff9b8479c53080 IpOverUsbSvc.exe*32                     .  .  .  .  .  .  .
0xfa8  0n4008 ffff9b8479942080 svchost.exe                             .  .  .  .  .  .  .
0xfb4  0n4020 ffff9b847990f080 svchost.exe                             .  .  .  .  .  .  .
0xff4  0n4084 ffff9b84799dd080 svchost.exe                             .  .  .  .  .  .  .
0xdac  0n3500 ffff9b8479a87080 svchost.exe                             .  .  .  .  .  .  .
0x99c  0n2460 ffff9b8479a88080 sqlwriter.exe                           .  .  .  .  .  .  .
0xd78  0n3448 ffff9b8479aa8080 MsMpEng.exe                             .  .  .  .  .  .  .
0x9f4  0n2548 ffff9b8479b10080 svchost.exe                             .  .  .  .  .  .  .
0xf1c  0n3868 ffff9b8479b11080 wlms.exe                                .  .  .  .  .  .  .
0x1024 0n4132 ffff9b8479b22080 svchost.exe                             .  .  .  .  .  .  .
0x1128 0n4392 ffff9b8479dd1080 sppsvc.exe                              .  .  .  .  .  .  .
0x13f0 0n5104 ffff9b84799bb080 unsecapp.exe                            .  .  .  .  .  .  .
0x1578 0n5496 ffff9b847b259080 svchost.exe                             .  .  .  .  .  .  .
0x16ac 0n5804 ffff9b847b2f2080 AggregatorHost.exe                      .  .  .  .  .  .  .
0x16cc 0n5836 ffff9b847b3020c0 svchost.exe                             .  .  .  .  .  .  .
0x12a8 0n4776 ffff9b847b4a9080 sihost.exe                              .  .  .  .  .  .  .
0x1638 0n5688 ffff9b8479fc50c0 svchost.exe                             .  .  .  .  .  .  .
0x16f8 0n5880 ffff9b847b2da0c0 svchost.exe                             .  .  .  .  .  .  .
0x1688 0n5768 ffff9b847b561080 svchost.exe                             .  .  .  .  .  .  .
0x1760 0n5984 ffff9b847b560080 svchost.exe                             .  .  .  .  .  .  .
0x1830 0n6192 ffff9b847b5ed080 taskhostw.exe                           .  .  .  .  .  .  .
0x1920 0n6432 ffff9b847b5ea080 SppExtComObj.Exe                        .  .  .  .  .  .  .
0x1998 0n6552 ffff9b847b5e7080 svchost.exe                             .  .  .  .  .  .  .
0x19bc 0n6588 ffff9b847b5e6080 explorer.exe                            2  .  .  .  1  .  .
0x1b30 0n6960 ffff9b847b5ee080 svchost.exe                             .  .  .  .  .  .  .
0x1b98 0n7064 ffff9b847b5eb080 svchost.exe                             .  .  .  .  .  .  .
0xe78  0n3704 ffff9b847b5e5080 svchost.exe                             .  .  .  .  .  .  .
0x1aa4 0n6820 ffff9b847b5e4080 svchost.exe                             .  .  .  .  .  .  .
0x1c24 0n7204 ffff9b847b9ce0c0 SearchHost.exe                          5  .  .  .  3  .  .
0x1c3c 0n7228 ffff9b847ba020c0 StartMenuExperienceHost.exe             .  .  .  .  .  .  .
0x1c80 0n7296 ffff9b847caf0080 Widgets.exe                             .  .  .  .  .  .  .
0x1ccc 0n7372 ffff9b847caef0c0 RuntimeBroker.exe                       .  .  .  .  .  .  .
0x1d48 0n7496 ffff9b847bfbd080 RuntimeBroker.exe                       .  .  .  .  1  .  .
0x1dd0 0n7632 ffff9b847bac9080 svchost.exe                             .  .  .  .  .  .  .
0x1fac 0n8108 ffff9b847bad6080 dllhost.exe                             .  .  .  .  .  .  .
0x2180 0n8576 ffff9b847bf25080 svchost.exe                             .  .  .  .  .  .  .
0x2304 0n8964 ffff9b847c1ac080 ctfmon.exe                              .  .  .  .  .  .  .
0x2368 0n9064 ffff9b847c1b8080 SearchIndexer.exe                       .  .  .  .  .  .  .
0x2214 0n8724 ffff9b847bf240c0 SecurityHealthSystray.exe               .  .  .  .  .  .  .
0x176c 0n5996 ffff9b847c7dd080 SecurityHealthService.exe               .  .  .  .  .  .  .
0x168c 0n5772 ffff9b847b0d70c0 TextInputHost.exe                       .  .  .  .  .  .  .
0xcdc  0n3292 ffff9b847c0b70c0 NisSrv.exe                              .  .  .  .  .  .  .
0x21a0 0n8608 ffff9b847bff20c0 svchost.exe                             .  .  .  .  .  .  .
0x1dbc 0n7612 ffff9b847ba350c0 uhssvc.exe                              .  .  .  .  .  .  .
0x1d0c 0n7436 ffff9b8477fdc0c0 svchost.exe                             .  .  .  .  .  .  .
0x29c  0n668  ffff9b84784380c0 svchost.exe                             .  .  .  .  .  .  .
0x5e8  0n1512 ffff9b84789840c0 svchost.exe                             .  .  .  .  .  .  .
0x808  0n2056 ffff9b847897f0c0 svchost.exe                             .  .  .  .  .  .  .
0xb0c  0n2828 ffff9b847c1ee0c0 svchost.exe                             .  .  .  .  .  .  .
0x784  0n1924 ffff9b847854d0c0 ShellExperienceHost.exe                 2  .  .  .  .  .  .
0xef4  0n3828 ffff9b8479e360c0 svchost.exe                             .  .  .  .  .  .  .
0xb38  0n2872 ffff9b847bfed0c0 svchost.exe                             .  .  .  .  .  .  .
0x1f24 0n7972 ffff9b84785480c0 WidgetService.exe                       .  .  .  .  .  .  .
0x6dc  0n1756 ffff9b8477ff00c0 svchost.exe                             .  .  .  .  .  .  .
0x1968 0n6504 ffff9b84785a40c0 svchost.exe                             .  .  .  .  .  .  .
0x22d8 0n8920 ffff9b847bbc90c0 csrss.exe                               .  .  .  .  1  .  .
0x175c 0n5980 ffff9b847ba300c0 winlogon.exe                            .  .  .  .  1  .  .
0x1694 0n5780 ffff9b8478b2d140 fontdrvhost.exe                         .  .  .  .  .  .  .
0xe0   0n224  ffff9b847c1690c0 LogonUI.exe                             .  .  .  .  .  .  .
0x1914 0n6420 ffff9b8477fd70c0 dwm.exe                                 .  .  .  .  .  .  .
0xb84  0n2948 ffff9b8477fd20c0 WUDFHost.exe                            .  .  .  .  .  .  .
0x17e8 0n6120 ffff9b847b2020c0 rdpclip.exe                             .  .  .  .  .  .  .
0xc04  0n3076 ffff9b847c0020c0 svchost.exe                             .  .  .  .  .  .  .
0x1bdc 0n7132 ffff9b8479e8d080 svchost.exe                             .  .  .  .  .  .  .
0x13a4 0n5028 ffff9b8478f550c0 rdpinput.exe                            .  .  .  .  .  .  .
0x2150 0n8528 ffff9b8478a1d0c0 TabTip.exe                              .  .  .  .  .  .  .
0x24c  0n588  ffff9b847ba2b0c0 devenv.exe                              .  .  .  2  .  .  .
0x7a4  0n1956 ffff9b84771ca0c0 PerfWatson2.exe                         .  .  .  2  .  .  .
0x1390 0n5008 ffff9b847c67e0c0 Microsoft.ServiceHub.Controller.exe     .  .  .  1  .  .  .
0xce4  0n3300 ffff9b84786ed0c0 ServiceHub.IdentityHost.exe*32          .  .  .  3  .  .  .
0x114c 0n4428 ffff9b847c6df0c0 ServiceHub.SettingsHost.exe             .  .  .  2  .  .  .
0x21a8 0n8616 ffff9b84783830c0 ServiceHub.VSDetouredHost.exe           .  .  .  1  .  .  .
0x178c 0n6028 ffff9b847bc8a0c0 ServiceHub.Host.netfx.x86.exe*32        .  .  .  4  .  .  .
0x954  0n2388 ffff9b84780d30c0 ServiceHub.ThreadedWaitDialog.exe       .  .  .  1  .  .  .
0x8fc  0n2300 ffff9b847c6bd0c0 ServiceHub.IndexingService.exe          .  .  .  1  .  .  .
0x18e4 0n6372 ffff9b8475a350c0 ServiceHub.IntellicodeModelService.exe  .  .  .  .  .  .  .
0x17a8 0n6056 ffff9b84756460c0 vshost.exe                              1  .  .  .  .  .  .
0x1930 0n6448 ffff9b8475abd0c0 vcpkgsrv.exe*32                         .  .  .  .  .  .  .
0x2360 0n9056 ffff9b84756ce0c0 ServiceHub.Host.AnyCPU.exe              .  .  .  1  .  .  .
0xbd4  0n3028 ffff9b8475b130c0 ServiceHub.TestWindowStoreHost.exe      .  .  .  1  .  .  .
0x2290 0n8848 ffff9b847c6020c0 MSBuild.exe                             .  .  .  .  .  .  .
0x22a0 0n8864 ffff9b84755570c0 conhost.exe                             .  .  .  .  .  .  .
0x1978 0n6520 ffff9b847593e080 cmd.exe                                 .  .  .  .  .  .  .
0x1794 0n6036 ffff9b847589f080 conhost.exe                             .  .  .  .  .  .  .
0x1568 0n5480 ffff9b847bbcd080 DbgX.Shell.exe                          .  .  .  .  .  .  .
0x14b4 0n5300 ffff9b8475a5b080 EngHost.exe                             .  .  .  .  .  .  .
0x2540 0n9536 ffff9b847c6660c0 SearchProtocolHost.exe                  .  .  .  .  .  .  .
0x1e58 0n7768 ffff9b8477578080 SearchFilterHost.exe                    .  .  .  .  .  .  .
0x528  0n1320 ffff9b847ca9f080 vcpkgsrv.exe*32                         .  .  .  .  .  .  .
0x11c4 0n4548 ffff9b84786620c0 vcpkgsrv.exe*32                         .  .  .  .  .  .  .
0x10cc 0n4300 ffff9b847578a0c0 Taskmgr.exe                             .  1  .  .  .  .  .
0x13bc 0n5052 ffff9b847c0df080 svchost.exe                             .  .  .  .  .  .  .
============= ================ ====================================== == == == == == == ==
PID           Address          Name                                   !! Rn Ry Bk Lc IO Er

Warning! Zombie process(es) detected (not displayed). Count: 9 [zombie report]
```

From this output, it seems that there are four threads in the System process that are waiting for Pushlocks to be released. The relatively long wait times, especially for the first two threads, might indicate a potential deadlock or some performance issues. The time it takes to release a Pushlock is generally very short, usually on the order of microseconds or less in typical scenarios.

```
0: kd> !mex.lt ffff9b84742ea040 -bk -wr WrFastMutex,WrGuardedMutex,WrMutex,WrPushLock
Process PID Thread             Id State        Time Reason
======= === ================ ==== ======= ========= ==========
System    4 ffff9b847c60f040 26a4 Waiting 4m:03.203 WrPushLock
System    4 ffff9b8475886040 2778 Waiting 4m:03.703 WrPushLock
System    4 ffff9b84771c0580  b8c Waiting   26s.406 WrPushLock
System    4 ffff9b8479441580 19e8 Waiting   32s.578 WrPushLock

Thread Count: 4
```

The **`!mex.us`** command shows the call stacks of threads that are currently waiting on a **WrPushLock**. Each thread's call stack is provided to show the sequence of function calls leading up to the point where the thread started waiting on the Pushlock.

```
0: kd> !mex.us -p ffff9b84742ea040 -wr WrPushLock
1 thread [stats]: ffff9b8475886040
    fffff802550201c6 nt!KiSwapContext+0x76
    fffff80254e6c9d5 nt!KiSwapThread+0xab5
    fffff80254e6ebb7 nt!KiCommitThreadWait+0x137
    fffff80254e70ad6 nt!KeWaitForSingleObject+0x256
    fffff80254e2aea0 nt!ExfAcquirePushLockExclusiveEx+0x1a0
    fffff80254eee012 nt!ExAcquirePushLockExclusiveEx+0x112
    fffff80260da1172 <Unloaded_TestDriver.sys>+0x1172
    fffff80260da1140 <Unloaded_TestDriver.sys>+0x1140

1 thread [stats]: ffff9b84771c0580
    fffff802550201c6 nt!KiSwapContext+0x76               
    fffff80254e6c9d5 nt!KiSwapThread+0xab5               
    fffff80254e6ebb7 nt!KiCommitThreadWait+0x137         
    fffff80254e70ad6 nt!KeWaitForSingleObject+0x256      
    fffff80254e2b0ea nt!ExfAcquirePushLockSharedEx+0x1ba 
    fffff80254eedd7c nt!ExAcquirePushLockSharedEx+0x11c  
    fffff80260ff1230 TestDriver!FileCreationThread+0x150 (C:\Users\User\source\repos\TestDriver\TestDriver\Driver.c @ 45)
    fffff80254f07167 nt!PspSystemThreadStartup+0x57      
    fffff8025501bb94 nt!KiStartSystemThread+0x34         

1 thread [stats]: ffff9b8479441580
    fffff802550201c6 nt!KiSwapContext+0x76                  
    fffff80254e6c9d5 nt!KiSwapThread+0xab5                  
    fffff80254e6ebb7 nt!KiCommitThreadWait+0x137            
    fffff80254e70ad6 nt!KeWaitForSingleObject+0x256         
    fffff80254e2aea0 nt!ExfAcquirePushLockExclusiveEx+0x1a0 
    fffff80254eee012 nt!ExAcquirePushLockExclusiveEx+0x112  
    fffff80260ff12b3 TestDriver!FileMoveThread+0x43         (C:\Users\User\source\repos\TestDriver\TestDriver\Driver.c @ 58)
    fffff80254f07167 nt!PspSystemThreadStartup+0x57         
    fffff8025501bb94 nt!KiStartSystemThread+0x34            

1 thread [stats]: ffff9b847c60f040
    fffff802550201c6 nt!KiSwapContext+0x76
    fffff80254e6c9d5 nt!KiSwapThread+0xab5
    fffff80254e6ebb7 nt!KiCommitThreadWait+0x137
    fffff80254e70ad6 nt!KeWaitForSingleObject+0x256
    fffff80254e2b0ea nt!ExfAcquirePushLockSharedEx+0x1ba
    fffff80254eedd7c nt!ExAcquirePushLockSharedEx+0x11c
    fffff80260da111d <Unloaded_TestDriver.sys>+0x111d
    0000000000000080 0x80
    0000000000000001 0x1

4 stack(s) with 4 threads displayed (4 Total threads)
```

Let's examine a random thread. This thread, identified by its address **`ffff9b84771c0580`**, has been in a state of waiting for 26 seconds and 406 milliseconds. It is waiting to acquire a pushlock.

```
0: kd> !mex.t ffff9b84771c0580
Process                   Thread                       CID       UserTime KernelTime ContextSwitches Wait Reason    Time State
System (ffff9b84742ea040) ffff9b84771c0580 (E|K|W|R|V) 4.b8c            0     2s.031            5361 WrPushLock  26s.406 Waiting

WaitBlockList:
    Object           Type                 Other Waiters
    ffffc1881ca0f280 SynchronizationEvent             0

Priority:
    Current Base Decrement ForegroundBoost IO Page
    9       8    0         0               0  5

# Child-SP         Return           Call Site                           Source
0 ffffc1881ca0eb70 fffff80254e6c9d5 nt!KiSwapContext+0x76               
1 ffffc1881ca0ecb0 fffff80254e6ebb7 nt!KiSwapThread+0xab5               
2 ffffc1881ca0ee00 fffff80254e70ad6 nt!KiCommitThreadWait+0x137         
3 ffffc1881ca0eeb0 fffff80254e2b0ea nt!KeWaitForSingleObject+0x256      
4 ffffc1881ca0f250 fffff80254eedd7c nt!ExfAcquirePushLockSharedEx+0x1ba 
5 ffffc1881ca0f300 fffff80260ff1230 nt!ExAcquirePushLockSharedEx+0x11c --> This function contains the memory address of a Pushlock 
6 ffffc1881ca0f350 fffff80254f07167 TestDriver!FileCreationThread+0x150 C:\Users\User\source\repos\TestDriver\TestDriver\Driver.c @ 45
7 ffffc1881ca0f570 fffff8025501bb94 nt!PspSystemThreadStartup+0x57      
8 ffffc1881ca0f5c0 0000000000000000 nt!KiStartSystemThread+0x34
```

The command **`.frame /r 5`** sets the context to the 5th frame in the stack trace and displays the register values at that moment. By examining the register values and the current instruction, we can get insights into what the function is doing at that point.

```
0: kd> .frame /r 0n5
05 ffffc188`1ca0f300 fffff802`60ff1230     nt!ExAcquirePushLockSharedEx+0x11c
rax=0000000000000000 rbx=ffff9b84771c0c80 rcx=0000000000000000
rdx=0000000000000000 rsi=0000000000000000 rdi=ffff9b847433ef70
rip=fffff80254eedd7c rsp=ffffc1881ca0f300 rbp=ffffc1881ca0f450
 r8=0000000000000000  r9=0000000000000000 r10=0000000000000000
r11=0000000000000000 r12=0000000000000266 r13=ffff9b8477678080
r14=ffff9b84742ea040 r15=ffffe500547ea000
iopl=0         nv up di pl nz na pe nc
cs=0000  ss=0000  ds=0000  es=0000  fs=0000  gs=0000             efl=00000000
nt!ExAcquirePushLockSharedEx+0x11c:
fffff802`54eedd7c ebb4            jmp     nt!ExAcquirePushLockSharedEx+0xd2 (fffff802`54eedd32)
```
The push lock at ffff9b847433ef70 (RDI) is currently locked and has threads waiting for it. 

```
0: kd> !mex.ddt nt!_EX_PUSH_LOCK ffff9b847433ef70

dt nt!_EX_PUSH_LOCK ffff9b847433ef70 () Recursive: [ -r1 -r2 -r ] Verbose Normal dt
==================================================================================
   +0x000 Locked               : 0y1 --> This bit is set to 1, indicating that the push lock is currently acquired or locked
   +0x000 Waiting              : 0y1 --> This bit is also set to 1, suggesting that there are one or more threads waiting for this push lock to be released
   +0x000 Waking               : 0y0
   +0x000 MultipleShared       : 0y0
   +0x000 Shared               : 0y111111111111111111000001100010000001110010100000111100101000 (0xffffc1881ca0f28)
   +0x000 Value                : 0xffffc188`1ca0f283 (0n-68684636687741)
   +0x000 Ptr                  : 0xffffc188`1ca0f283 Void    [ !ndao dps dc !handle ln ? ]
```

Let's now start to examine a second thread that is also waiting on a pushlock. This output indicates that a thread in the **System** process is waiting to acquire an exclusive pushlock and has been in this waiting state for over 32 seconds.

```
0: kd> !mex.t ffff9b8479441580
Process                   Thread                       CID       UserTime KernelTime ContextSwitches Wait Reason    Time State
System (ffff9b84742ea040) ffff9b8479441580 (E|K|W|R|V) 4.19e8           0          0               1 WrPushLock  32s.578 Waiting

WaitBlockList:
    Object           Type                 Other Waiters
    ffffc1881cb1f190 SynchronizationEvent             0

# Child-SP         Return           Call Site                              Source
0 ffffc1881cb1ea80 fffff80254e6c9d5 nt!KiSwapContext+0x76                  
1 ffffc1881cb1ebc0 fffff80254e6ebb7 nt!KiSwapThread+0xab5                  
2 ffffc1881cb1ed10 fffff80254e70ad6 nt!KiCommitThreadWait+0x137            
3 ffffc1881cb1edc0 fffff80254e2aea0 nt!KeWaitForSingleObject+0x256         
4 ffffc1881cb1f160 fffff80254eee012 nt!ExfAcquirePushLockExclusiveEx+0x1a0 
5 ffffc1881cb1f210 fffff80260ff12b3 nt!ExAcquirePushLockExclusiveEx+0x112  
6 ffffc1881cb1f250 fffff80254f07167 TestDriver!FileMoveThread+0x43         C:\Users\User\source\repos\TestDriver\TestDriver\Driver.c @ 58
7 ffffc1881cb1f570 fffff8025501bb94 nt!PspSystemThreadStartup+0x57         
8 ffffc1881cb1f5c0 0000000000000000 nt!KiStartSystemThread+0x34
```

This thread has a similar call stack as the previous one. Let's examine the **5th** frame, which contains the memory address of the pushlock. 

```
0: kd> .frame /r 0n5
05 ffffc188`1cb1f210 fffff802`60ff12b3     nt!ExAcquirePushLockExclusiveEx+0x112
rax=0000000000000000 rbx=ffff9b8479441c20 rcx=0000000000000000
rdx=0000000000000000 rsi=ffff9b847433ef70 rdi=ffff9b847433ef70
rip=fffff80254eee012 rsp=ffffc1881cb1f210 rbp=ffffc1881cb1f350
 r8=0000000000000000  r9=0000000000000000 r10=0000000000000000
r11=0000000000000000 r12=0000000000002268 r13=ffff9b84759660c0
r14=ffff9b84742ea040 r15=ffffe50054751000
iopl=0         nv up di pl nz na pe nc
cs=0000  ss=0000  ds=0000  es=0000  fs=0000  gs=0000             efl=00000000
nt!ExAcquirePushLockExclusiveEx+0x112:
fffff802`54eee012 ebb3            jmp     nt!ExAcquirePushLockExclusiveEx+0xc7 (fffff802`54eedfc7)
```

The **RDI** register holds the pushlock address **`ffff9b847433ef70`**, which is also referenced by other threads. This indicates that multiple threads are concurrently waiting on this specific pushlock address. The deadlock scenario arises because one of these threads, which presumably holds the lock, has not released it yet. This prevents the other threads from acquiring the lock and proceeding with their execution, leading to a deadlock condition where none of the involved threads can progress.

```
0: kd> !mex.ddt nt!_EX_PUSH_LOCK ffff9b847433ef70

dt nt!_EX_PUSH_LOCK ffff9b847433ef70 () Recursive: [ -r1 -r2 -r ] Verbose Normal dt
==================================================================================
   +0x000 Locked               : 0y1
   +0x000 Waiting              : 0y1
   +0x000 Waking               : 0y0
   +0x000 MultipleShared       : 0y0
   +0x000 Shared               : 0y111111111111111111000001100010000001110010100000111100101000 (0xffffc1881ca0f28)
   +0x000 Value                : 0xffffc188`1ca0f283 (0n-68684636687741)
   +0x000 Ptr                  : 0xffffc188`1ca0f283 Void    [ !ndao dps dc !handle ln ? ]
```
