# Description

The **`DriverEntry`** function includes a call to **`ExReleasePushLockExclusive`** at the end. This is problematic because this lock is supposed to be acquired and released by the same thread. **`DriverEntry`** runs in a different thread than the ones executing **`FileCreationThread`** and **`FileMoveThread`**. Releasing a lock in a different thread than the one that acquired it can lead to a BSOD due to a violation of the lock's usage contract.

Link to this Memory Dump: https://mega.nz/file/ihVVjKwY#eNYteblfHDwYJhsusYA6C2CLr-mmfi7yukzW-nOs3MQ

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

    ExReleasePushLockExclusive(&SyncRes->FilePushLock); // This is wrong

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

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/8508252c-729e-405d-82b8-7078c384bce0)


# WinDbg Walk Through - Analysis

Load the Memory Dump in WinDbg:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/a5957b62-bf15-4154-8555-902ce3af0618)


Let's start with the **`!mex.di`** command which stands for dump information. This command retrieves information about the machine from which the memory dump originated. It's especially handy when working with multiple memory dumps, as it helps distinguish the analysis related to each specific dump. This is an improved version of the **`vertarget`** command.

```
0: kd> !mex.di
Computer Name:  Not Found
Windows 10 Kernel Version 22621 MP (6 procs) Free x64
Product: WinNt, suite: TerminalServer SingleUserTS
Edition build lab: 22621.2506.amd64fre.ni_release_svc_prod3.231018-1809
Kernel base = 0xfffff802`53800000 PsLoadedModuleList = 0xfffff802`544134a0
Debug session time: Sun Dec 24 08:12:10.242 2023 (UTC - 8:00)
System Uptime: 0 days 0:30:18.084
SystemManufacturer = Microsoft Corporation
SystemProductName = Virtual Machine
Processor: 11th Gen Intel(R) Core(TM) i7-1185G7 @ 3.00GHz
Bugcheck: 162 (FFFF910142869040, FFFF910145796F70, FFFFFFFF, 0)
```

Let's start with the **`!tl -t`** command to discover threads of interest. It examines the state and wait reasons for every thread across all processes, presenting the results as statistics. This allows for a quick overview, such as determining how many processes have 4 running threads, and so on.

```
0: kd> !mex.tl -t
PID            Address          Name                                   !! Rn Ry Bk Lc IO Er
============== ================ ====================================== == == == == == == ==
0x0    0n0     fffff80254549f40 Idle                                    .  6  6  .  .  .  .
0x4    0n4     ffff91013dcec040 System                                  6  1  .  4  .  1  .
0x84   0n132   ffff91013dd1f080 Registry                                .  .  .  .  .  .  .
0x238  0n568   ffff910141015040 smss.exe                                .  .  .  1  .  .  .
0x2c4  0n708   ffff9101422290c0 csrss.exe                               .  .  .  .  1  .  .
0x30c  0n780   ffff910142437080 wininit.exe                             .  .  .  .  .  .  .
0x320  0n800   ffff91014243e140 csrss.exe                               .  .  .  .  1  .  .
0x374  0n884   ffff910142474080 winlogon.exe                            .  .  .  .  .  .  .
0x39c  0n924   ffff9101422180c0 services.exe                            .  .  .  .  .  .  .
0x3cc  0n972   ffff9101424b5080 LsaIso.exe                              .  .  .  .  .  .  .
0x3d8  0n984   ffff9101424bb080 lsass.exe                               .  .  .  .  .  .  .
0x324  0n804   ffff910142530080 svchost.exe                             .  .  .  .  .  .  .
0x268  0n616   ffff910142537080 fontdrvhost.exe                         .  .  .  .  .  .  .
0x404  0n1028  ffff91014253a080 fontdrvhost.exe                         .  .  .  .  .  .  .
0x470  0n1136  ffff9101425ec0c0 svchost.exe                             .  .  .  .  .  .  .
0x49c  0n1180  ffff910142608080 svchost.exe                             .  .  .  .  .  .  .
0x508  0n1288  ffff9101427020c0 dwm.exe                                 .  .  .  .  .  .  .
0x55c  0n1372  ffff910142726080 svchost.exe                             .  .  .  .  .  .  .
0x56c  0n1388  ffff910142727080 svchost.exe                             .  .  .  .  .  .  .
0x5d0  0n1488  ffff91014278e080 svchost.exe                             .  .  .  .  .  .  .
0x614  0n1556  ffff9101427b1080 svchost.exe                             .  .  .  .  .  .  .
0x62c  0n1580  ffff9101427ba080 svchost.exe                             .  .  .  .  .  .  .
0x628  0n1576  ffff9101427b7080 svchost.exe                             .  .  .  .  .  .  .
0x63c  0n1596  ffff9101427ca0c0 svchost.exe                             .  .  .  .  .  .  .
0x644  0n1604  ffff9101427b8080 svchost.exe                             .  .  .  .  .  .  .
0x680  0n1664  ffff9101428230c0 svchost.exe                             .  .  .  .  .  .  .
0x6e0  0n1760  ffff91014286d080 svchost.exe                             .  .  .  .  .  .  .
0x70c  0n1804  ffff91014287e080 svchost.exe                             .  .  .  .  .  .  .
0x718  0n1816  ffff910142883080 svchost.exe                             .  .  .  .  .  .  .
0x720  0n1824  ffff910142887080 svchost.exe                             .  .  .  .  .  .  .
0x73c  0n1852  ffff910142898080 svchost.exe                             .  .  .  .  .  .  .
0x758  0n1880  ffff9101428b3080 svchost.exe                             .  .  .  .  .  .  .
0x768  0n1896  ffff9101428b8080 svchost.exe                             .  .  .  .  .  .  .
0x798  0n1944  ffff9101428c7080 svchost.exe                             .  .  .  .  .  .  .
0x7c0  0n1984  ffff91014290b080 svchost.exe                             .  .  .  .  .  .  .
0x878  0n2168  ffff9101429f5080 svchost.exe                             .  .  .  .  .  .  .
0x930  0n2352  ffff910142a4d080 VSSVC.exe                               .  .  .  .  .  .  .
0x960  0n2400  ffff910142a54080 svchost.exe                             .  .  .  .  .  .  .
0x974  0n2420  ffff910142a6b080 svchost.exe                             .  .  .  .  .  .  .
0x980  0n2432  ffff910142a6f080 svchost.exe                             .  .  .  .  .  .  .
0x9dc  0n2524  ffff910142a84080 svchost.exe                             .  .  .  .  .  .  .
0x9e8  0n2536  ffff910142a86080 svchost.exe                             .  .  .  .  .  .  .
0x9f4  0n2548  ffff910142a8b080 svchost.exe                             .  .  .  .  .  .  .
0xa44  0n2628  ffff91013dd53080 svchost.exe                             .  .  .  .  .  .  .
0xa60  0n2656  ffff91013dd03040 MemCompression                          .  .  .  .  .  .  .
0xae8  0n2792  ffff91013ddc2080 svchost.exe                             .  .  .  .  .  .  .
0xafc  0n2812  ffff91013dda2080 svchost.exe                             .  .  .  .  .  .  .
0xb2c  0n2860  ffff91013dd55080 svchost.exe                             .  .  .  .  .  .  .
0xb3c  0n2876  ffff91013dd95080 svchost.exe                             .  .  .  .  .  .  .
0xb68  0n2920  ffff91013dd69080 svchost.exe                             .  .  .  .  .  .  .
0xb9c  0n2972  ffff91013dd5c080 svchost.exe                             .  .  .  .  .  .  .
0x47c  0n1148  ffff910142c7a080 svchost.exe                             .  .  .  .  .  .  .
0xb28  0n2856  ffff910142cad080 svchost.exe                             .  .  .  .  .  .  .
0xc5c  0n3164  ffff910142ce7080 svchost.exe                             .  .  .  .  .  .  .
0xcf4  0n3316  ffff910142e63140 svchost.exe                             .  .  .  .  .  .  .
0xd1c  0n3356  ffff910142d9e0c0 svchost.exe                             .  .  .  .  .  .  .
0xd24  0n3364  ffff910142e6c080 svchost.exe                             .  .  .  .  .  .  .
0xd8c  0n3468  ffff910142ebc0c0 svchost.exe                             .  .  .  .  .  .  .
0xe4c  0n3660  ffff910142f520c0 spoolsv.exe                             .  .  .  .  .  .  .
0xe70  0n3696  ffff910142f560c0 svchost.exe                             .  .  .  .  .  .  .
0xf94  0n3988  ffff910142fd2080 svchost.exe                             .  .  .  .  .  .  .
0xf9c  0n3996  ffff910142ec0080 svchost.exe                             .  .  .  .  .  .  .
0xfb0  0n4016  ffff910142ff6080 svchost.exe                             .  .  .  .  .  .  .
0xfb8  0n4024  ffff910144098080 svchost.exe                             .  .  .  .  .  .  .
0xfc4  0n4036  ffff910142ff5080 svchost.exe                             .  .  .  .  .  .  .
0xc08  0n3080  ffff9101441cc080 IpOverUsbSvc.exe*32                     .  .  .  .  .  .  .
0xc48  0n3144  ffff910144277080 sqlwriter.exe                           .  .  .  .  .  .  .
0xe24  0n3620  ffff910144275080 svchost.exe                             .  .  .  .  .  .  .
0x988  0n2440  ffff9101442dd080 svchost.exe                             .  .  .  .  .  .  .
0x1014 0n4116  ffff910144111080 MsMpEng.exe                             .  1  .  .  .  .  .
0x102c 0n4140  ffff9101442ed080 svchost.exe                             .  .  .  .  .  .  .
0x1038 0n4152  ffff910144186080 wlms.exe                                .  .  .  .  .  .  .
0x10a0 0n4256  ffff9101443f6080 svchost.exe                             .  .  .  .  .  .  .
0x10b0 0n4272  ffff9101444840c0 svchost.exe                             .  .  .  .  .  .  .
0x11dc 0n4572  ffff910144541080 sppsvc.exe                              .  .  .  .  .  .  .
0x134c 0n4940  ffff910144a08080 svchost.exe                             .  .  .  .  .  .  .
0x14d8 0n5336  ffff910144a98080 svchost.exe                             .  .  .  .  .  .  .
0x15b0 0n5552  ffff910144b6a080 svchost.exe                             .  .  .  .  .  .  .
0x1710 0n5904  ffff910144b6d080 AggregatorHost.exe                      .  .  .  .  .  .  .
0x17a8 0n6056  ffff910144e09080 sihost.exe                              .  .  .  .  .  .  .
0x17dc 0n6108  ffff910144d61080 svchost.exe                             .  .  .  .  .  .  .
0x5cc  0n1484  ffff910144e1d080 svchost.exe                             .  .  .  .  .  .  .
0x13a0 0n5024  ffff910144e1c080 svchost.exe                             .  .  .  .  .  .  .
0x15c4 0n5572  ffff910144e40080 svchost.exe                             .  .  .  .  .  .  .
0x1624 0n5668  ffff910144e43080 taskhostw.exe                           .  .  .  .  .  .  .
0x1810 0n6160  ffff910144e9a0c0 unsecapp.exe                            .  .  .  .  .  .  .
0x192c 0n6444  ffff910144eed080 svchost.exe                             .  .  .  .  .  .  .
0x1974 0n6516  ffff910144f760c0 SppExtComObj.Exe                        .  .  .  .  .  .  .
0x1a5c 0n6748  ffff91014505a080 explorer.exe                            3  .  .  .  1  .  .
0x1604 0n5636  ffff9101450e1080 svchost.exe                             .  .  .  .  .  .  .
0x1174 0n4468  ffff9101450e4080 svchost.exe                             .  .  .  .  .  .  .
0x1ac8 0n6856  ffff910144e92080 svchost.exe                             .  .  .  .  .  .  .
0x16b0 0n5808  ffff910144d76080 svchost.exe                             .  .  .  .  .  .  .
0x1c94 0n7316  ffff910145247080 SearchHost.exe                          5  .  .  .  2  .  .
0x1cac 0n7340  ffff910145246080 StartMenuExperienceHost.exe             .  .  .  .  .  .  .
0x1d20 0n7456  ffff910145241080 Widgets.exe                             .  .  .  .  .  .  .
0x1d74 0n7540  ffff9101452450c0 RuntimeBroker.exe                       .  .  .  .  .  .  .
0x1dac 0n7596  ffff91014524a080 RuntimeBroker.exe                       .  .  .  .  1  .  .
0x1e4c 0n7756  ffff910145243080 svchost.exe                             .  .  .  .  .  .  .
0x1fd0 0n8144  ffff91014523d080 dllhost.exe                             .  .  .  .  .  .  .
0x1eb8 0n7864  ffff910145a09080 ctfmon.exe                              .  .  .  .  .  .  .
0x520  0n1312  ffff910145a11080 SearchIndexer.exe                       .  .  .  .  .  .  .
0x2188 0n8584  ffff910142216080 svchost.exe                             .  .  .  .  .  .  .
0x2348 0n9032  ffff9101448540c0 TextInputHost.exe                       .  .  .  .  .  .  .
0x23f0 0n9200  ffff910145e5c140 csrss.exe                               .  .  .  .  1  .  .
0x1b58 0n7000  ffff910145e55080 winlogon.exe                            .  .  .  .  1  .  .
0xe44  0n3652  ffff910145e8e080 LogonUI.exe                             .  .  .  .  .  .  .
0x1524 0n5412  ffff9101456500c0 dwm.exe                                 .  .  .  .  .  .  .
0x35c  0n860   ffff910145e8f080 fontdrvhost.exe                         .  .  .  .  .  .  .
0x12b8 0n4792  ffff910145d97080 WUDFHost.exe                            .  .  .  .  .  .  .
0x148c 0n5260  ffff910145b3a080 rdpclip.exe                             .  .  .  .  .  .  .
0x3b0  0n944   ffff91014583e0c0 rdpinput.exe                            .  .  .  .  .  .  .
0x1364 0n4964  ffff910144a03080 svchost.exe                             .  .  .  .  .  .  .
0x2160 0n8544  ffff9101458eb0c0 TabTip.exe                              .  .  .  .  .  .  .
0x1a24 0n6692  ffff9101453130c0 NisSrv.exe                              .  .  .  .  .  .  .
0x24f4 0n9460  ffff910145350080 SecurityHealthSystray.exe               .  .  .  .  .  .  .
0x2508 0n9480  ffff9101450dc0c0 SecurityHealthService.exe               .  .  .  .  .  .  .
0x27ac 0n10156 ffff910141fb90c0 svchost.exe                             .  .  .  .  .  .  .
0x25c0 0n9664  ffff91013ec480c0 svchost.exe                             .  .  .  .  .  .  .
0xd10  0n3344  ffff91014184f0c0 uhssvc.exe                              .  .  .  .  .  .  .
0xe98  0n3736  ffff9101471090c0 svchost.exe                             .  .  .  .  .  .  .
0xf4c  0n3916  ffff91014184a0c0 svchost.exe                             .  .  .  .  .  .  .
0x1088 0n4232  ffff91014535a0c0 svchost.exe                             .  .  .  .  .  .  .
0x1258 0n4696  ffff91014710e0c0 svchost.exe                             .  .  .  .  .  .  .
0x12a0 0n4768  ffff91014180a0c0 svchost.exe                             .  .  .  .  .  .  .
0x19a0 0n6560  ffff91014610c0c0 backgroundTaskHost.exe                 10  .  .  .  .  .  .
0x1b54 0n6996  ffff910141faf0c0 dllhost.exe                             .  .  .  .  .  .  .
0x1c08 0n7176  ffff910141ad20c0 ShellExperienceHost.exe                 3  .  .  .  .  .  .
0x10e0 0n4320  ffff91014534d0c0 svchost.exe                             .  .  .  .  .  .  .
0x2540 0n9536  ffff9101471020c0 svchost.exe                             .  .  .  .  .  .  .
0xd4c  0n3404  ffff910144dd50c0 WidgetService.exe                       .  .  .  .  .  .  .
0x21d8 0n8664  ffff91013ec430c0 smartscreen.exe                         .  .  .  .  .  .  .
0x13ec 0n5100  ffff9101450a2080 SearchProtocolHost.exe                  .  .  .  .  .  .  .
0x4f4  0n1268  ffff910141e59080 SearchFilterHost.exe                    .  .  .  .  .  .  .
0x2198 0n8600  ffff9101460d40c0 dllhost.exe                             .  .  .  .  .  .  .
0x1b30 0n6960  ffff91013f5f4080 audiodg.exe                             .  .  .  .  .  .  .
0x11c4 0n4548  ffff9101425c2140 cmd.exe                                 .  .  .  1  .  .  .
0x25fc 0n9724  ffff910141ad7080 conhost.exe                             .  1  .  .  .  .  .
0x2144 0n8516  ffff91013f78e080 devenv.exe                              .  .  .  2  .  .  .
0x6b0  0n1712  ffff91014235e140 PerfWatson2.exe                         .  .  .  2  .  .  .
0xca4  0n3236  ffff91013f726080 Microsoft.ServiceHub.Controller.exe     .  .  .  1  .  .  .
0x110  0n272   ffff91013f6ef080 ServiceHub.VSDetouredHost.exe           .  .  .  1  .  .  .
0xf88  0n3976  ffff910146072080 ServiceHub.ThreadedWaitDialog.exe       .  .  .  1  .  .  .
0x18cc 0n6348  ffff91014232d080 ServiceHub.IndexingService.exe          .  .  .  1  .  .  .
0x2d0  0n720   ffff910142b35080 ServiceHub.IdentityHost.exe*32          .  .  .  3  .  .  .
0x23a4 0n9124  ffff910141a1f080 vshost.exe                              1  .  .  .  .  .  .
0x7e4  0n2020  ffff91013fc52080 ServiceHub.SettingsHost.exe             .  .  .  2  .  .  .
0x1e70 0n7792  ffff91013eb41180 vcpkgsrv.exe*32                         .  .  .  .  .  .  .
0x18b4 0n6324  ffff91013ff3a080 ServiceHub.IntellicodeModelService.exe  .  .  .  .  .  .  .
0x258  0n600   ffff9101447ea0c0 ServiceHub.Host.netfx.x86.exe*32        .  .  .  4  .  .  .
0x850  0n2128  ffff91013fdef080 ServiceHub.Host.AnyCPU.exe              .  .  .  1  .  .  .
0x584  0n1412  ffff9101447d8080 ServiceHub.TestWindowStoreHost.exe      .  .  .  1  .  .  .
0xd84  0n3460  ffff91013fed9180 vcpkgsrv.exe*32                         .  .  .  .  .  .  .
0x138c 0n5004  ffff9101447d0080 vcpkgsrv.exe*32                         .  .  .  .  .  .  .
0x17e8 0n6120  ffff91013f776080 MSBuild.exe                             .  .  .  .  .  .  .
0x2258 0n8792  ffff910146456080 conhost.exe                             .  .  .  .  .  .  .
0x31c  0n796   ffff9101462430c0 vctip.exe                               .  .  .  2  .  .  .
0x22b4 0n8884  ffff910146458080 sc.exe                                  .  .  .  .  1  .  .
============== ================ ====================================== == == == == == == ==
PID            Address          Name                                   !! Rn Ry Bk Lc IO Er

Warning! Zombie process(es) detected (not displayed). Count: 10 [zombie report]
```

From this output, it seems that there are four threads in the System process that are waiting for Pushlocks to be released. The relatively long wait times, especially for the first two threads, might indicate a potential deadlock or some performance issues. The time it takes to release a Pushlock is generally very short, usually on the order of microseconds or less in typical scenarios.

```
0: kd> !mex.lt ffff91013dcec040 -bk -wr WrFastMutex,WrGuardedMutex,WrMutex,WrPushLock
Process PID Thread             Id State        Time Reason
======= === ================ ==== ======= ========= ==========
System    4 ffff9101457f2040  d7c Waiting 1m:25.171 WrPushLock
System    4 ffff910142868040 1808 Waiting 1m:26.843 WrPushLock
System    4 ffff9101464f6040  580 Waiting         0 WrPushLock
System    4 ffff910141e36040 2110 Waiting         0 WrPushLock

Thread Count: 4
```

The **`!mex.us`** command shows the call stacks of threads that are currently waiting on a **WrPushLock**. Each thread's call stack is provided to show the sequence of function calls leading up to the point where the thread started waiting on the Pushlock.

```
0: kd> !mex.us -p ffff91013dcec040 -wr WrPushLock
1 thread [stats]: ffff910141e36040
    fffff80253c201c6 nt!KiSwapContext+0x76
    fffff80253a6c9d5 nt!KiSwapThread+0xab5
    fffff80253a6ebb7 nt!KiCommitThreadWait+0x137
    fffff80253a70ad6 nt!KeWaitForSingleObject+0x256
    fffff80253a2aea0 nt!ExfAcquirePushLockExclusiveEx+0x1a0
    fffff80253aee012 nt!ExAcquirePushLockExclusiveEx+0x112
    fffff8025d8412a3 TestDriver+0x12a3
    fffff80253b07167 nt!PspSystemThreadStartup+0x57
    fffff80253c1bb94 nt!KiStartSystemThread+0x34

1 thread [stats]: ffff910142868040
    fffff80253c201c6 nt!KiSwapContext+0x76
    fffff80253a6c9d5 nt!KiSwapThread+0xab5
    fffff80253a6ebb7 nt!KiCommitThreadWait+0x137
    fffff80253a70ad6 nt!KeWaitForSingleObject+0x256
    fffff80253a2aea0 nt!ExfAcquirePushLockExclusiveEx+0x1a0
    fffff80253aee012 nt!ExAcquirePushLockExclusiveEx+0x112
    fffff8025e5a12b3 <Unloaded_TestDriver.sys>+0x12b3
    ffff910142c9cb70 0xffff910142c9cb70
    ffff910100000000 0xffff910100000000
    ffff910100000000 0xffff910100000000
    ffff910142868040 0xffff910142868040
    ffff91013dd82000 0xffff91013dd82000
    ffff808222dfe3d0 0xffff808222dfe3d0

1 thread [stats]: ffff9101457f2040
    fffff80253c201c6 nt!KiSwapContext+0x76
    fffff80253a6c9d5 nt!KiSwapThread+0xab5
    fffff80253a6ebb7 nt!KiCommitThreadWait+0x137
    fffff80253a70ad6 nt!KeWaitForSingleObject+0x256
    fffff80253a2b0ea nt!ExfAcquirePushLockSharedEx+0x1ba
    fffff80253aedd7c nt!ExAcquirePushLockSharedEx+0x11c
    fffff8025e5a1230 <Unloaded_TestDriver.sys>+0x1230
    fffff6840aecf450 0xfffff6840aecf450
    ffff910100000001 0xffff910100000001
    fffff8025e5a1e90 <Unloaded_TestDriver.sys>+0x1e90
    00000000000003e7 0x3e7
    fffff6840aecf3c8 0xfffff6840aecf3c8
    fffff6840aecf408 0xfffff6840aecf408
    000000000000000f 0xf

1 thread [stats]: ffff9101464f6040
    fffff80253c201c6 nt!KiSwapContext+0x76
    fffff80253a6c9d5 nt!KiSwapThread+0xab5
    fffff80253a6ebb7 nt!KiCommitThreadWait+0x137
    fffff80253a70ad6 nt!KeWaitForSingleObject+0x256
    fffff80253a2aea0 nt!ExfAcquirePushLockExclusiveEx+0x1a0
    fffff80253aee012 nt!ExAcquirePushLockExclusiveEx+0x112
    fffff8025d84112f TestDriver+0x112f
    fffff80253b07167 nt!PspSystemThreadStartup+0x57
    fffff80253c1bb94 nt!KiStartSystemThread+0x34

4 stack(s) with 4 threads displayed (4 Total threads)
```

In this analysis, our initial focus is on the thread identified by **`ffff9101457f2040`**. We've chosen to scrutinize this particular thread as it has been in a wait state for 1 minute and 25.171 seconds on a pushlock. This duration stands out, especially when compared to other threads in the system that have significantly shorter wait times, such as 0 seconds.

```
0: kd> !mex.t ffff9101457f2040
Process                   Thread                       CID       UserTime KernelTime ContextSwitches Wait Reason      Time State
System (ffff91013dcec040) ffff9101457f2040 (E|K|W|R|V) 4.d7c            0      234ms            3187 WrPushLock  1m:25.171 Waiting

WaitBlockList:
    Object           Type                 Other Waiters
    fffff6840aecf280 SynchronizationEvent             0

Priority:
    Current Base Decrement ForegroundBoost IO Page
    9       8    0         0               0  5

# Child-SP         Return           Call Site
0 fffff6840aeceb70 fffff80253a6c9d5 nt!KiSwapContext+0x76
1 fffff6840aececb0 fffff80253a6ebb7 nt!KiSwapThread+0xab5
2 fffff6840aecee00 fffff80253a70ad6 nt!KiCommitThreadWait+0x137
3 fffff6840aeceeb0 fffff80253a2b0ea nt!KeWaitForSingleObject+0x256
4 fffff6840aecf250 fffff80253aedd7c nt!ExfAcquirePushLockSharedEx+0x1ba
5 fffff6840aecf300 fffff8025e5a1230 nt!ExAcquirePushLockSharedEx+0x11c
6 fffff6840aecf350 fffff6840aecf450 <Unloaded_TestDriver.sys>+0x1230
7 fffff6840aecf358 ffff910100000001 0xfffff6840aecf450
8 fffff6840aecf360 fffff8025e5a1e90 0xffff910100000001
9 fffff6840aecf368 00000000000003e7 <Unloaded_TestDriver.sys>+0x1e90
a fffff6840aecf370 fffff6840aecf3c8 0x3e7
b fffff6840aecf378 fffff6840aecf408 0xfffff6840aecf3c8
c fffff6840aecf380 000000000000000f 0xfffff6840aecf408
d fffff6840aecf388 0000000000000000 0xf
```

Let's revisit the address **`ffff9101457f2040`**. Remember, this is a thread address.

```
0: kd> !object ffff9101457f2040
Object: ffff9101457f2040  Type: (ffff91013dcea900) Thread
    ObjectHeader: ffff9101457f2010 (new version)
    HandleCount: 0  PointerCount: 1
```

This output displays detailed information about the **ETHREAD** structure for a specific thread.

```
0: kd> !mex.ddt nt!_ETHREAD ffff910142868040

dt nt!_ETHREAD ffff910142868040 () Recursive: [ -r1 -r2 -r ] Verbose Normal dt
==================================================================================
   +0x000 Tcb                  : _KTHREAD
   +0x480 CreateTime           : _LARGE_INTEGER 0x01da3683`bf03ee04 (12/24/2023 16:10:43 UTC)
   +0x488 ExitTime             : _LARGE_INTEGER 0xffff9101`428684c8 (0n-122040379603768)
   +0x488 KeyedWaitChain       : _LIST_ENTRY [ 0xffff9101`428684c8 - 0xffff9101`428684c8 ] [EMPTY OR 1 ELEMENT]
   +0x498 PostBlockList        : _LIST_ENTRY [ 0x00000000`00000000 - 0xfffff802`5e5a1270 ] [INVALID]
   +0x498 ForwardLinkShadow    : (null)
   +0x4a0 StartAddress         : 0xfffff802`5e5a1270 Void  [generic address]
   +0x4a8 TerminationPort      : (null)
   +0x4a8 ReaperLink           : (null)
   +0x4a8 KeyedWaitValue       : (null)
   +0x4b0 ActiveTimerListLock  : 0
   +0x4b8 ActiveTimerListHead  : _LIST_ENTRY [ 0xffff9101`428684f8 - 0xffff9101`428684f8 ] [EMPTY OR 1 ELEMENT]
   +0x4c8 Cid                  : _CLIENT_ID
   +0x4d8 KeyedWaitSemaphore   : _KSEMAPHORE
   +0x4d8 AlpcWaitSemaphore    : _KSEMAPHORE
   +0x4f8 ClientSecurity       : _PS_CLIENT_SECURITY_CONTEXT
   +0x500 IrpList              : _LIST_ENTRY [ 0xffff9101`42868540 - 0xffff9101`42868540 ] [EMPTY OR 1 ELEMENT]
   +0x510 TopLevelIrp          : 0
   +0x518 DeviceToVerify       : (null)
   +0x520 Win32StartAddress    : 0xfffff802`5e5a1270 Void  [generic address]
   +0x528 ChargeOnlySession    : (null)
   +0x530 LegacyPowerObject    : (null)
   +0x538 ThreadListEntry      : _LIST_ENTRY [ 0xffff9101`44e3a5b8 - 0xffff9101`457f2578 ]
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
   +0x5b0 BoostList            : _LIST_ENTRY [ 0xffff9101`428685f0 - 0xffff9101`428685f0 ] [EMPTY OR 1 ELEMENT]
   +0x5c0 DeboostList          : _LIST_ENTRY [ 0xffff9101`42868600 - 0xffff9101`42868600 ] [EMPTY OR 1 ELEMENT]
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
   +0x648 EnergyValues         : 0xffff9101`42868950 _THREAD_ENERGY_VALUES
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
   +0x678 OwnerEntryListHead   : _LIST_ENTRY [ 0xffff9101`428686b8 - 0xffff9101`428686b8 ] [EMPTY OR 1 ELEMENT]
   +0x688 DisownedOwnerEntryListLock : 0
   +0x690 DisownedOwnerEntryListHead : _LIST_ENTRY [ 0xffff9101`428686d0 - 0xffff9101`428686d0 ] [EMPTY OR 1 ELEMENT]
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
0: kd> !mex.ddt -a6 ffff910142868040+0x6a0 nt!_KLOCK_ENTRY

dt -a6 ffff910142868040+0x6a0 nt!_KLOCK_ENTRY () Recursive: [ -r1 -r2 -r ] Verbose Normal dt
==================================================================================
[0] @ ffff9101`428686e0 
---------------------------------------------
   +0x000 LockState            : _KLOCK_ENTRY_LOCK_STATE
   +0x000 LockUnsafe           : 0xffff9101`42c9cb70 Void  PoolTag: }..{
   +0x000 CrossThreadReleasableAndBusyByte : 0x70 'p'
   +0x001 Reserved             : [6] "???"
   +0x007 InTreeByte           : 0xff ''
   +0x008 SessionState         : 0x00000000`ffffffff Void    [ !ndao dps dc !handle ln ? ]
   +0x008 SessionId            : 0xffffffff (0n4294967295)
   +0x00c SessionPad           : 0
   +0x010 EntryFlags           : 0x7000100 (0n117440768)
   +0x010 EntryIndex           : 0 ''
   +0x011 WaitingByte          : 0x1 ''
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

[1] @ ffff9101`42868740 
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

[2] @ ffff9101`428687a0 
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

[3] @ ffff9101`42868800 
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

[4] @ ffff9101`42868860 
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

[5] @ ffff9101`428688c0 
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

In the first lock entry, **`LockUnsafe`** is set to a non-null value **`(0xffff910142c9cb70)`**. This indicates that this particular lock entry is in use with some unsafe conditions. This output shows that the **`_EX_PUSH_LOCK`** at the specified address is currently not locked (neither exclusively nor shared), no threads are waiting for it.

```
0: kd> !mex.ddt nt!_EX_PUSH_LOCK 0xffff9101`42c9cb70

dt nt!_EX_PUSH_LOCK 0xffff9101`42c9cb70 () Recursive: [ -r1 -r2 -r ] Verbose Normal dt
==================================================================================
   +0x000 Locked                   : 0y0
   +0x000 Waiting                  : 0y0
   +0x000 Waking                   : 0y0
   +0x000 MultipleShared           : 0y0
   +0x000 Shared                   : 0y000000000000000000000000000000000000000000000000000000000000 (0)
   +0x000 Value                    : 0
   +0x000 Ptr                      : (null)
```

Let's run the **`!analyze -v`** command to quickly triage the issue as it provides a quick and detailed overview of the state of the system at the time of the crash. It automates many aspects of crash dump analysis, providing a quick way to get valuable information.

```
0: kd> !ext.analyze -v
*******************************************************************************
*                                                                             *
*                        Bugcheck Analysis                                    *
*                                                                             *
*******************************************************************************

KERNEL_AUTO_BOOST_INVALID_LOCK_RELEASE (162)
A lock tracked by AutoBoost was released by a thread that did not own the lock.
This is typically caused when some thread releases a lock on behalf of another
thread (which is not legal with AutoBoost tracking enabled) or when some thread
tries to release a lock it no longer owns.
Arguments:
Arg1: ffff910142869040, The address of the thread
Arg2: ffff910145796f70, The lock address
Arg3: 00000000ffffffff, The session ID of the thread
Arg4: 0000000000000000, Reserved

Debugging Details:
------------------


KEY_VALUES_STRING: 1

    Key  : Analysis.CPU.mSec
    Value: 4452

    Key  : Analysis.Elapsed.mSec
    Value: 5497

    Key  : Analysis.IO.Other.Mb
    Value: 0

    Key  : Analysis.IO.Read.Mb
    Value: 1

    Key  : Analysis.IO.Write.Mb
    Value: 7

    Key  : Analysis.Init.CPU.mSec
    Value: 16530

    Key  : Analysis.Init.Elapsed.mSec
    Value: 814604

    Key  : Analysis.Memory.CommitPeak.Mb
    Value: 148

    Key  : Bugcheck.Code.KiBugCheckData
    Value: 0x162

    Key  : Bugcheck.Code.LegacyAPI
    Value: 0x162

    Key  : Dump.Attributes.AsUlong
    Value: 800

    Key  : Failure.Bucket
    Value: 0x162_TestDriver!unknown_function

    Key  : Failure.Hash
    Value: {419d12bb-5d5f-512c-e75c-1fd193415869}

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


BUGCHECK_CODE:  162

BUGCHECK_P1: ffff910142869040

BUGCHECK_P2: ffff910145796f70

BUGCHECK_P3: ffffffff

BUGCHECK_P4: 0

FILE_IN_CAB:  MEMORY.DMP

VIRTUAL_MACHINE:  HyperV

DUMP_FILE_ATTRIBUTES: 0x800

BLACKBOXBSD: 1 (!blackboxbsd)


BLACKBOXNTFS: 1 (!blackboxntfs)


BLACKBOXPNP: 1 (!blackboxpnp)


BLACKBOXWINLOGON: 1

PROCESS_NAME:  System

STACK_TEXT:  
fffff684`0d4d6fe8 fffff802`53ca9e56     : 00000000`00000162 ffff9101`42869040 ffff9101`45796f70 00000000`ffffffff : nt!KeBugCheckEx
fffff684`0d4d6ff0 fffff802`53aee10b     : 00000000`00000000 00000000`00000000 ffff9101`42eff000 00000000`00000000 : nt!KeAbPostRelease+0x1bbd26
fffff684`0d4d7030 fffff802`5d8410d7     : ffff9101`45796f70 fffff802`5d843298 ffff9101`44b49380 fffff802`53a81c13 : nt!ExReleasePushLockExclusiveEx+0x3b
fffff684`0d4d7070 fffff802`5d8416fb     : ffff9101`45796e20 00000000`00000000 ffffffff`800046d8 ffffffff`800046d8 : TestDriver+0x10d7
fffff684`0d4d70c0 fffff802`5d841630     : ffff9101`42eff000 fffff684`0d4d7280 ffff9101`45dae9cd ffff9101`44b49380 : TestDriver+0x16fb
fffff684`0d4d7100 fffff802`53fe2a60     : ffff9101`42eff000 00000000`00000000 ffff9101`44b49380 fffff802`53b14088 : TestDriver+0x1630
fffff684`0d4d7130 fffff802`53e9ad0b     : ffff9101`42eff000 00000000`00000000 00000000`00000000 ffff8082`245552d0 : nt!PnpCallDriverEntry+0x54
fffff684`0d4d7180 fffff802`53fcff17     : 00000000`00000000 00000000`00000000 ffff9101`45791000 fffff802`5454aac0 : nt!IopLoadDriver+0x523
fffff684`0d4d7340 fffff802`53a34f85     : ffff9101`00000000 ffffffff`800044dc ffff9101`42869040 ffff9101`00000000 : nt!IopLoadUnloadDriver+0x57
fffff684`0d4d7380 fffff802`53b07167     : ffff9101`42869040 00000000`00000178 ffff9101`42869040 fffff802`53a34e30 : nt!ExpWorkerThread+0x155
fffff684`0d4d7570 fffff802`53c1bb94     : ffffa481`b4cc6180 ffff9101`42869040 fffff802`53b07110 00000000`00000246 : nt!PspSystemThreadStartup+0x57
fffff684`0d4d75c0 00000000`00000000     : fffff684`0d4d8000 fffff684`0d4d1000 00000000`00000000 00000000`00000000 : nt!KiStartSystemThread+0x34


SYMBOL_NAME:  TestDriver+10d7

MODULE_NAME: TestDriver

IMAGE_NAME:  TestDriver.sys

STACK_COMMAND:  .cxr; .ecxr ; kb

BUCKET_ID_FUNC_OFFSET:  10d7

FAILURE_BUCKET_ID:  0x162_TestDriver!unknown_function

OS_VERSION:  10.0.22621.2506

BUILDLAB_STR:  ni_release_svc_prod3

OSPLATFORM_TYPE:  x64

OSNAME:  Windows 10

FAILURE_ID_HASH:  {419d12bb-5d5f-512c-e75c-1fd193415869}

Followup:     MachineOwner
---------
```

This is the thread **`(ffff910142869040)`** that is releasing a lock that it does not own. This is not allowed when **AutoBoost** is enabled, as each thread is responsible for releasing the locks it acquires.

```
0: kd> !mex.t ffff910142869040
Process                   Thread                       CID       UserTime KernelTime ContextSwitches Wait Reason Time State
System (ffff91013dcec040) ffff910142869040 (E|K|W|R|V) 4.b70            0      109ms            5686 Executive      0 Running on processor 0

Priority:
    Current Base Decrement ForegroundBoost IO Page
    13      12   0         0               0  5

# Child-SP         Return           Call Site
0 fffff6840d4d6fe8 fffff80253ca9e56 nt!KeBugCheckEx
1 fffff6840d4d6ff0 fffff80253aee10b nt!KeAbPostRelease+0x1bbd26
2 fffff6840d4d7030 fffff8025d8410d7 nt!ExReleasePushLockExclusiveEx+0x3b
3 fffff6840d4d7070 fffff8025d8416fb TestDriver+0x10d7
4 fffff6840d4d70c0 fffff8025d841630 TestDriver+0x16fb
5 fffff6840d4d7100 fffff80253fe2a60 TestDriver+0x1630
6 fffff6840d4d7130 fffff80253e9ad0b nt!PnpCallDriverEntry+0x54
7 fffff6840d4d7180 fffff80253fcff17 nt!IopLoadDriver+0x523
8 fffff6840d4d7340 fffff80253a34f85 nt!IopLoadUnloadDriver+0x57
9 fffff6840d4d7380 fffff80253b07167 nt!ExpWorkerThread+0x155
a fffff6840d4d7570 fffff80253c1bb94 nt!PspSystemThreadStartup+0x57
b fffff6840d4d75c0 0000000000000000 nt!KiStartSystemThread+0x34
```

This output indicates a push lock that is currently in a locked state and other threads waiting to acquire it. 

```
0: kd> !mex.ddt nt!_EX_PUSH_LOCK ffff910145796f70

dt nt!_EX_PUSH_LOCK ffff910145796f70 () Recursive: [ -r1 -r2 -r ] Verbose Normal dt
==================================================================================
   +0x000 Locked               : 0y1
   +0x000 Waiting              : 0y1
   +0x000 Waking               : 0y1
   +0x000 MultipleShared       : 0y1
   +0x000 Shared               : 0y111111111111111111110110100001000000110100110011011100101001 (0xfffff6840d33729)
   +0x000 Value                : 0xfffff684`0d33729f (0n-10427959119201)
   +0x000 Ptr                  : 0xfffff684`0d33729f Void    [ !ndao dps dc !handle ln ? ]
```

The root cause of this BSOD in this case is due to the thread identified by **`ffff910142869040`** releasing a lock that it does not own. This inappropriate lock release has led to other threads indefinitely waiting on the push lock, as they expect it to be released by the owning thread. This situation violates the fundamental principles of lock ownership and synchronization in kernel mode.

```
0: kd> !mex.lt ffff91013dcec040 -bk -wr WrPushLock
Process PID  TID Thread           State        Time Reason
======= === ==== ================ ======= ========= ==========
System    4  d7c ffff9101457f2040 Waiting 1m:25.171 WrPushLock
System    4 1808 ffff910142868040 Waiting 1m:26.843 WrPushLock
System    4  580 ffff9101464f6040 Waiting        0s WrPushLock
System    4 2110 ffff910141e36040 Waiting        0s WrPushLock
======= === ==== ================ ======= ========= ==========
Process PID  TID Thread           State        Time Reason

Thread Count: 4
```
