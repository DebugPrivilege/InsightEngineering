# Description

Holding a spinlock for more than 25 microseconds is discouraged because it can significantly consume CPU resources. Spinlocks are designed for very short-term synchronization. Holding a spinlock too long may even lead to a system bugcheck (crash), as it can disrupt essential system processes and prevent the execution of critical tasks. 

Link to this Memory Dump: https://mega.nz/file/7kFl2RxT#u4LdvplOxjO9BRnEhP3PrMxPSKh5_q5NXOGYrKn86Eg

# Code Sample - Holding a Spinlock for too long

The root cause of the bugcheck is that the spinlock is held for an extended period, while performing potentially time-consuming operations. This violates the guideline of holding a spinlock for a very short period, typically less than 25 microseconds. The **`ZwQueryInformationFile`** function can be problematic in this context because it's a time-consuming operation that interacts with the file system. File system operations can be slow due to various reasons such as disk latency or other I/O operations.

```c
#include <ntifs.h>
#include <ntstrsafe.h>

typedef struct _SYNC_RESOURCES {
    KSPIN_LOCK FileSpinLock;
} SYNC_RESOURCES, * PSYNC_RESOURCES;

NTSTATUS FileCreationThread(PVOID Context) {
    PSYNC_RESOURCES SyncRes = (PSYNC_RESOURCES)Context;
    UNICODE_STRING uniFilePath;
    WCHAR filePathBuffer[150];
    char dataToWrite[] = "Hello, World!\r\n";
    IO_STATUS_BLOCK ioStatusBlock;
    KIRQL oldIrql;

    KeAcquireSpinLock(&SyncRes->FileSpinLock, &oldIrql);

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

    KeReleaseSpinLock(&SyncRes->FileSpinLock, oldIrql);

    return STATUS_SUCCESS;
}

NTSTATUS FileMoveThread(PVOID Context) {
    PSYNC_RESOURCES SyncRes = (PSYNC_RESOURCES)Context;
    IO_STATUS_BLOCK ioStatusBlock;
    WCHAR oldFilePathBuffer[150], newFilePathBuffer[150];
    KIRQL oldIrql;

    KeAcquireSpinLock(&SyncRes->FileSpinLock, &oldIrql);

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

    KeReleaseSpinLock(&SyncRes->FileSpinLock, oldIrql);

    return STATUS_SUCCESS;
}

NTSTATUS FileQueryThread(PVOID Context) {
    PSYNC_RESOURCES SyncRes = (PSYNC_RESOURCES)Context;
    WCHAR filePathBuffer[150];
    IO_STATUS_BLOCK ioStatusBlock;
    KIRQL oldIrql;
    FILE_BASIC_INFORMATION fileInfo;

    KeAcquireSpinLock(&SyncRes->FileSpinLock, &oldIrql);  // Acquire lock here

    for (int i = 0; i < 1000; i++) {
        RtlStringCchPrintfW(filePathBuffer, sizeof(filePathBuffer) / sizeof(WCHAR), L"\\DosDevices\\C:\\Temp\\file_%d.txt", i);
        UNICODE_STRING uniFilePath;
        RtlInitUnicodeString(&uniFilePath, filePathBuffer);

        OBJECT_ATTRIBUTES objAttributes;
        InitializeObjectAttributes(&objAttributes, &uniFilePath, OBJ_CASE_INSENSITIVE | OBJ_KERNEL_HANDLE, NULL, NULL);

        HANDLE hFile;
        NTSTATUS status = ZwOpenFile(&hFile, FILE_READ_ATTRIBUTES, &objAttributes, &ioStatusBlock, FILE_SHARE_READ, FILE_SYNCHRONOUS_IO_NONALERT);

        if (NT_SUCCESS(status)) {
            ZwQueryInformationFile(hFile, &ioStatusBlock, &fileInfo, sizeof(FILE_BASIC_INFORMATION), FileBasicInformation);
            ZwClose(hFile);
        }
    }

    KeReleaseSpinLock(&SyncRes->FileSpinLock, oldIrql);  // Release lock here

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
    KeInitializeSpinLock(&SyncRes->FileSpinLock);

    HANDLE threadHandle1, threadHandle2, threadHandle3;

    status = PsCreateSystemThread(&threadHandle1, (ACCESS_MASK)0, NULL, (PHANDLE)0, NULL, FileCreationThread, SyncRes);
    if (NT_SUCCESS(status)) {
        ZwClose(threadHandle1);
    }

    status = PsCreateSystemThread(&threadHandle2, (ACCESS_MASK)0, NULL, (PHANDLE)0, NULL, FileMoveThread, SyncRes);
    if (NT_SUCCESS(status)) {
        ZwClose(threadHandle2);
    }

    // Adding the new FileQueryThread
    status = PsCreateSystemThread(&threadHandle3, (ACCESS_MASK)0, NULL, (PHANDLE)0, NULL, FileQueryThread, SyncRes);
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

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/cd1c9743-f355-4ba8-ac06-4c0fc5de7d74)

Start the kernel driver:

```
sc.exe start TestDrv
```

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/7ff6b86b-acf4-4cea-b6e6-8f686287d8bc)

This will bugcheck the system:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/4eefb623-d54b-4bc0-a420-08c6ad39c39e)


# WinDbg Walk Through - Analysis

Load the Memory Dump in WinDbg:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/e048fff9-6d59-4293-b27f-e9af10a13e75)


Let's start with the **`!mex.di`** command which stands for dump information. This command retrieves information about the machine from which the memory dump originated. It's especially handy when working with multiple memory dumps, as it helps distinguish the analysis related to each specific dump. This is an improved version of the **`vertarget`** command.

```
1: kd> !mex.di
Computer Name:  Not Found
Windows 10 Kernel Version 22621 MP (6 procs) Free x64
Product: WinNt, suite: TerminalServer SingleUserTS
Edition build lab: 22621.2506.amd64fre.ni_release_svc_prod3.231018-1809
Kernel base = 0xfffff801`49200000 PsLoadedModuleList = 0xfffff801`49e134a0
Debug session time: Tue Dec 26 06:06:43.498 2023 (UTC - 8:00)
System Uptime: 0 days 0:15:08.990
SystemManufacturer = Microsoft Corporation
SystemProductName = Virtual Machine
Processor: 11th Gen Intel(R) Core(TM) i7-1185G7 @ 3.00GHz
Bugcheck: D1 (FFFFE70BAFEA2AF0, 2, 0, FFFFF8014D54C7AE)
```

The **`!mex.tl -t`** command examines the state and wait reasons for every thread across all processes, presenting the results as statistics. This allows for a quick overview, such as determining how many processes have running threads, and so on. It's like having a satellite view of all the thread state that are associated with the processes.

```
1: kd> !mex.tl -t
PID            Address          Name                                   !! Rn Ry Bk Lc IO Er
============== ================ ====================================== == == == == == == ==
0x0    0n0     fffff80149f49f40 Idle                                    .  4  8  .  .  .  .
0x4    0n4     ffffc181e8eea040 System                                  6  3  1  .  .  1  .
0x80   0n128   ffffc181e8f17080 Registry                                .  .  .  .  .  .  .
0x234  0n564   ffffc181ec20b040 smss.exe                                .  .  .  1  .  .  .
0x2c4  0n708   ffffc181ed474140 csrss.exe                               .  .  .  .  1  .  .
0x30c  0n780   ffffc181ed737080 wininit.exe                             .  .  .  .  .  .  .
0x320  0n800   ffffc181ed73e140 csrss.exe                               .  .  1  .  1  .  .
0x374  0n884   ffffc181ed803080 winlogon.exe                            .  .  .  .  .  .  .
0x38c  0n908   ffffc181ed808080 services.exe                            .  .  .  .  .  .  .
0x3b8  0n952   ffffc181ed829080 LsaIso.exe                              .  .  .  .  .  .  .
0x3c4  0n964   ffffc181ed82a080 lsass.exe                               .  .  .  .  .  .  .
0x2c8  0n712   ffffc181ed883080 svchost.exe                             .  .  .  .  .  .  .
0x2b0  0n688   ffffc181ed88a080 fontdrvhost.exe                         .  .  .  .  .  .  .
0x3d8  0n984   ffffc181ed88d080 fontdrvhost.exe                         .  .  .  .  .  .  .
0x454  0n1108  ffffc181ed9440c0 svchost.exe                             .  .  .  .  .  .  .
0x484  0n1156  ffffc181ed96c080 svchost.exe                             .  .  .  .  .  .  .
0x4e4  0n1252  ffffc181eda68080 dwm.exe                                 .  .  .  .  .  .  .
0x53c  0n1340  ffffc181edaef0c0 svchost.exe                             .  .  .  .  .  .  .
0x57c  0n1404  ffffc181edb06080 svchost.exe                             .  .  .  .  .  .  .
0x5ec  0n1516  ffffc181edb440c0 svchost.exe                             .  .  .  .  .  .  .
0x5f8  0n1528  ffffc181edb45080 svchost.exe                             .  .  .  .  .  .  .
0x600  0n1536  ffffc181edb48080 svchost.exe                             .  .  .  .  .  .  .
0x608  0n1544  ffffc181edb4c080 svchost.exe                             .  .  .  .  .  .  .
0x610  0n1552  ffffc181edb39080 svchost.exe                             .  .  .  .  .  .  .
0x61c  0n1564  ffffc181edb46080 svchost.exe                             .  .  .  .  .  .  .
0x68c  0n1676  ffffc181edbbc080 svchost.exe                             .  .  .  .  1  .  .
0x6d8  0n1752  ffffc181edc1c0c0 svchost.exe                             .  .  .  .  .  .  .
0x788  0n1928  ffffc181edc78080 svchost.exe                             .  .  .  .  .  .  .
0x7d4  0n2004  ffffc181edce70c0 svchost.exe                             .  .  .  .  .  .  .
0x440  0n1088  ffffc181edd1d080 svchost.exe                             .  .  .  .  .  .  .
0x578  0n1400  ffffc181edd20080 svchost.exe                             .  .  .  .  .  .  .
0x588  0n1416  ffffc181edd240c0 svchost.exe                             .  .  .  .  .  .  .
0x808  0n2056  ffffc181edd2a080 svchost.exe                             .  .  .  .  .  .  .
0x810  0n2064  ffffc181edd2d080 svchost.exe                             .  .  .  .  .  .  .
0x830  0n2096  ffffc181edd460c0 svchost.exe                             .  .  .  .  .  .  .
0x858  0n2136  ffffc181edd48080 svchost.exe                             .  .  .  .  .  .  .
0x920  0n2336  ffffc181eddc5080 svchost.exe                             .  .  .  .  .  .  .
0x968  0n2408  ffffc181ede130c0 VSSVC.exe                               .  .  .  .  .  .  .
0x9dc  0n2524  ffffc181e8f2d080 svchost.exe                             .  .  .  .  .  .  .
0xa20  0n2592  ffffc181e8f0f080 svchost.exe                             .  .  .  .  .  .  .
0xa40  0n2624  ffffc181e8ea6080 svchost.exe                             .  .  .  .  .  .  .
0xa48  0n2632  ffffc181e8ea4080 svchost.exe                             .  .  .  .  .  .  .
0xa6c  0n2668  ffffc181e8fd3080 svchost.exe                             .  .  .  .  .  .  .
0xa78  0n2680  ffffc181e8fa2080 svchost.exe                             .  .  .  .  .  .  .
0xae0  0n2784  ffffc181ede22080 svchost.exe                             .  .  .  .  .  .  .
0xb00  0n2816  ffffc181e8f93080 svchost.exe                             .  .  .  .  .  .  .
0xb4c  0n2892  ffffc181e8f95080 svchost.exe                             .  .  .  .  .  .  .
0xb60  0n2912  ffffc181ede78240 MemCompression                          .  .  .  .  .  .  .
0xba0  0n2976  ffffc181e8e90080 svchost.exe                             .  .  .  .  .  .  .
0xbc8  0n3016  ffffc181edff4080 svchost.exe                             .  .  .  .  .  .  .
0xbe0  0n3040  ffffc181edff2080 svchost.exe                             .  .  .  .  .  .  .
0xc20  0n3104  ffffc181ee0b7080 svchost.exe                             .  .  .  .  .  .  .
0xc84  0n3204  ffffc181ee10f080 svchost.exe                             .  .  .  .  .  .  .
0xc8c  0n3212  ffffc181ee110080 svchost.exe                             .  .  .  .  .  .  .
0xd38  0n3384  ffffc181ee222140 svchost.exe                             .  .  .  .  .  .  .
0xd4c  0n3404  ffffc181ee229080 svchost.exe                             .  .  .  .  .  .  .
0xd60  0n3424  ffffc181ee22a080 svchost.exe                             .  .  .  .  .  .  .
0xd7c  0n3452  ffffc181ee22e080 svchost.exe                             .  .  .  .  .  .  .
0xdd0  0n3536  ffffc181ee2b7080 svchost.exe                             .  .  .  .  .  .  .
0xe60  0n3680  ffffc181ee4350c0 spoolsv.exe                             .  .  .  .  .  .  .
0xee4  0n3812  ffffc181ee2f90c0 svchost.exe                             .  .  .  .  .  .  .
0xf40  0n3904  ffffc181ee6240c0 svchost.exe                             .  .  .  .  .  .  .
0xf50  0n3920  ffffc181ee6130c0 svchost.exe                             .  .  .  .  .  .  .
0xf58  0n3928  ffffc181ee6020c0 svchost.exe                             .  .  .  .  .  .  .
0xf68  0n3944  ffffc181ee36f080 svchost.exe                             .  .  .  .  .  .  .
0xf74  0n3956  ffffc181ee5df0c0 IpOverUsbSvc.exe*32                     .  .  .  .  .  .  .
0xfa4  0n4004  ffffc181ee521080 sqlwriter.exe                           .  .  .  .  .  .  .
0xfc0  0n4032  ffffc181ee60f080 svchost.exe                             .  .  .  .  .  .  .
0xfd4  0n4052  ffffc181ee611080 svchost.exe                             .  .  .  .  .  .  .
0xff4  0n4084  ffffc181ee677080 MsMpEng.exe                             .  .  .  .  .  .  .
0xac0  0n2752  ffffc181ee6aa080 svchost.exe                             .  .  .  .  .  .  .
0xc68  0n3176  ffffc181ee6ca080 svchost.exe                             .  .  .  .  .  .  .
0xbf8  0n3064  ffffc181ee6cb080 wlms.exe                                .  .  .  .  .  .  .
0xc74  0n3188  ffffc181ee6dd080 svchost.exe                             .  .  .  .  .  .  .
0x11e4 0n4580  ffffc181ee8ca080 sppsvc.exe                              .  .  .  .  .  .  .
0x13a8 0n5032  ffffc181ee2af080 unsecapp.exe                            .  .  .  .  .  .  .
0xf3c  0n3900  ffffc181eead1080 svchost.exe                             .  .  .  .  .  .  .
0x14f4 0n5364  ffffc181eec550c0 svchost.exe                             .  .  .  .  .  .  .
0x157c 0n5500  ffffc181eecca0c0 svchost.exe                             .  .  .  .  .  .  .
0x1638 0n5688  ffffc181ec0d2080 SppExtComObj.Exe                        .  .  .  .  .  .  .
0x16e4 0n5860  ffffc181ec0d1080 svchost.exe                             .  .  .  .  .  .  .
0x1788 0n6024  ffffc181e9f0a080 AggregatorHost.exe                      .  .  .  .  .  .  .
0xfe0  0n4064  ffffc181eeccb080 sihost.exe                              .  .  .  .  .  .  .
0x530  0n1328  ffffc181ee709080 svchost.exe                             .  .  .  .  .  .  .
0x1820 0n6176  ffffc181ee708080 svchost.exe                             .  .  .  .  .  .  .
0x1870 0n6256  ffffc181edf88080 svchost.exe                             .  .  .  .  .  .  .
0x18a0 0n6304  ffffc181edf87080 svchost.exe                             .  .  .  .  .  .  .
0x18d0 0n6352  ffffc181e9de8080 taskhostw.exe                           .  .  .  .  .  .  .
0x1a88 0n6792  ffffc181ef043080 explorer.exe                            2  .  .  .  .  .  .
0x1aa8 0n6824  ffffc181ef041080 svchost.exe                             .  .  .  .  .  .  .
0x1a1c 0n6684  ffffc181ef03d080 svchost.exe                             .  .  .  .  .  .  .
0x1c18 0n7192  ffffc181e9f09080 svchost.exe                             .  .  .  .  .  .  .
0x1d88 0n7560  ffffc181ef3f5080 svchost.exe                             .  .  .  .  .  .  .
0x1e84 0n7812  ffffc181ef507080 SearchHost.exe                          9  .  .  .  2  .  .
0x1e8c 0n7820  ffffc181ef5020c0 StartMenuExperienceHost.exe             .  .  .  .  .  .  .
0x1f44 0n8004  ffffc181ef510080 RuntimeBroker.exe                       .  .  .  .  .  .  .
0x1fa8 0n8104  ffffc181efab7080 RuntimeBroker.exe                       .  .  .  .  .  .  .
0x1fbc 0n8124  ffffc181eff8c080 svchost.exe                             .  .  .  .  .  .  .
0x17c4 0n6084  ffffc181eff86080 svchost.exe                             .  .  .  .  .  .  .
0x20d4 0n8404  ffffc181efabd080 dllhost.exe                             .  .  .  .  .  .  .
0x1f08 0n7944  ffffc181f028e080 ctfmon.exe                              .  .  .  .  .  .  .
0x2358 0n9048  ffffc181efb4a080 SearchIndexer.exe                       .  .  .  .  .  .  .
0x1df8 0n7672  ffffc181f05ed080 svchost.exe                             .  .  .  .  .  .  .
0x258c 0n9612  ffffc181ef881080 svchost.exe                             .  .  .  .  .  .  .
0x2678 0n9848  ffffc181efcac080 MoUsoCoreWorker.exe                     .  .  .  .  .  .  .
0x279c 0n10140 ffffc181efcb5100 csrss.exe                               .  .  .  .  1  .  .
0x27e8 0n10216 ffffc181f04a7080 winlogon.exe                            .  .  .  .  1  .  .
0x1d64 0n7524  ffffc181efcb3080 fontdrvhost.exe                         .  .  .  .  .  .  .
0x2654 0n9812  ffffc181f01230c0 LogonUI.exe                             .  .  .  .  .  .  .
0x265c 0n9820  ffffc181f0299080 dwm.exe                                 .  .  .  .  .  .  .
0x548  0n1352  ffffc181f01d8080 rdpclip.exe                             .  .  .  .  .  .  .
0x1178 0n4472  ffffc181f01ba080 WUDFHost.exe                            .  .  .  .  .  .  .
0x22f8 0n8952  ffffc181efeaa0c0 rdpinput.exe                            .  .  .  .  .  .  .
0x275c 0n10076 ffffc181efe6e080 svchost.exe                             .  .  .  .  .  .  .
0x2320 0n8992  ffffc181efe81080 TextInputHost.exe                       .  .  .  .  .  .  .
0x2870 0n10352 ffffc181efaf50c0 svchost.exe                             .  .  .  .  .  .  .
0x2880 0n10368 ffffc181f05810c0 TabTip.exe                              .  .  .  .  .  .  .
0x2904 0n10500 ffffc181f03f5080 NisSrv.exe                              .  .  .  .  .  .  .
0x2d6c 0n11628 ffffc181f0b99080 GameBarFTServer.exe                     .  .  .  .  .  .  .
0x2f78 0n12152 ffffc181f0bad080 audiodg.exe                             .  .  .  .  .  .  .
0x2c0c 0n11276 ffffc181f0d88080 smartscreen.exe                         .  .  .  .  .  .  .
0x628  0n1576  ffffc181f0e98080 svchost.exe                             .  .  .  .  .  .  .
0x231c 0n8988  ffffc181f0efb080 svchost.exe                             .  .  .  .  .  .  .
0x2b08 0n11016 ffffc181f0f540c0 SecurityHealthSystray.exe               .  .  .  .  .  .  .
0x2a48 0n10824 ffffc181f0f46080 SecurityHealthService.exe               .  .  .  .  .  .  .
0x1f1c 0n7964  ffffc181f0fdc080 svchost.exe                             .  .  .  .  .  .  .
0x2d60 0n11616 ffffc181f11130c0 WmiPrvSE.exe                            .  .  .  .  .  .  .
0x23e0 0n9184  ffffc181f0e9d080 ShellExperienceHost.exe                 2  .  .  .  .  .  .
0x1d8  0n472   ffffc181f11180c0 RuntimeBroker.exe                       .  .  .  .  1  .  .
0x3180 0n12672 ffffc181ec6bd0c0 taskhostw.exe                           .  .  .  .  .  .  .
0x21c4 0n8644  ffffc181f11df0c0 svchost.exe                             .  .  .  .  .  .  .
0x334  0n820   ffffc181eeb650c0 svchost.exe                             .  .  .  .  .  .  .
0x318c 0n12684 ffffc181f2b350c0 svchost.exe                             .  .  .  .  .  .  .
0x3044 0n12356 ffffc181ec5e6080 svchost.exe                             .  .  .  .  .  .  .
0x2a20 0n10784 ffffc181f038b080 SystemSettingsBroker.exe                .  .  .  .  .  .  .
0x29f8 0n10744 ffffc181ef3020c0 svchost.exe                             .  .  .  .  .  .  .
0x27bc 0n10172 ffffc181f0b15140 svchost.exe                             .  .  .  .  .  .  .
0x30b8 0n12472 ffffc181f2a94140 svchost.exe                             .  .  .  .  .  .  .
0x2ab4 0n10932 ffffc181f2a8f080 WidgetService.exe                       .  .  .  .  .  .  .
0x2e44 0n11844 ffffc181ef968100 svchost.exe                             .  .  .  .  .  .  .
0x12bc 0n4796  ffffc181f1326080 svchost.exe                             .  .  .  .  .  .  .
0x1ecc 0n7884  ffffc181ec6d9080 svchost.exe                             .  .  .  .  .  .  .
0x31d8 0n12760 ffffc181ee2240c0 svchost.exe                             .  .  .  .  1  .  .
0x1a04 0n6660  ffffc181f24f40c0 svchost.exe                             .  .  .  .  .  .  .
0x3300 0n13056 ffffc181ecaf6080 WmiPrvSE.exe                            .  .  .  .  .  .  .
0x249c 0n9372  ffffc181f03aa080 svchost.exe                             .  .  .  .  .  .  .
0x2700 0n9984  ffffc181efa6d080 svchost.exe                             .  .  .  .  .  .  .
0x325c 0n12892 ffffc181ef96f080 msiexec.exe                             .  .  .  .  .  .  .
0x2cc0 0n11456 ffffc181ecc0e080 uhssvc.exe                              .  .  .  .  .  .  .
0x3310 0n13072 ffffc181f0b9c080 MoNotificationUx.exe                    .  .  .  .  .  .  .
0xa64  0n2660  ffffc181f0b77080 svchost.exe                             .  .  .  .  .  .  .
0x311c 0n12572 ffffc181efae60c0 devenv.exe                              .  .  .  2  .  .  .
0x10dc 0n4316  ffffc181f0aec0c0 PerfWatson2.exe                         .  .  .  2  .  .  .
0x2e38 0n11832 ffffc181ea7ef0c0 Microsoft.ServiceHub.Controller.exe     .  .  .  1  .  .  .
0xc2c  0n3116  ffffc181ea1dc0c0 ServiceHub.VSDetouredHost.exe           .  .  .  1  .  .  .
0x1988 0n6536  ffffc181f19ec0c0 ServiceHub.ThreadedWaitDialog.exe       .  .  .  1  .  .  .
0xe5c  0n3676  ffffc181f1aca0c0 vshost.exe                              1  .  .  .  .  .  .
0xe6c  0n3692  ffffc181ea5cf0c0 vcpkgsrv.exe*32                         .  .  .  .  .  .  .
0xd40  0n3392  ffffc181ea5f10c0 SearchProtocolHost.exe                  .  .  .  .  .  .  .
0x1038 0n4152  ffffc181f17ca0c0 ServiceHub.SettingsHost.exe             .  .  .  2  .  .  .
0x2670 0n9840  ffffc181ea9cf0c0 ServiceHub.Host.dotnet.x64.exe          .  .  .  2  .  .  .
0x1ad8 0n6872  ffffc181ee33d0c0 ServiceHub.IndexingService.exe          .  .  .  1  .  .  .
0xab4  0n2740  ffffc181f416c0c0 ServiceHub.Host.AnyCPU.exe              .  .  .  1  .  .  .
0x1eb0 0n7856  ffffc181ea4f10c0 SearchFilterHost.exe                    .  .  .  .  .  .  .
0x2704 0n9988  ffffc181f0ad40c0 MSBuild.exe                             .  .  .  .  .  .  .
0x2b3c 0n11068 ffffc181f0eb20c0 conhost.exe                             .  .  .  .  .  .  .
0x2de0 0n11744 ffffc181f2aab080 ServiceHub.TestWindowStoreHost.exe      .  .  .  1  .  .  .
0x2cc4 0n11460 ffffc181f17e50c0 ServiceHub.IntellicodeModelService.exe  .  .  .  .  .  .  .
0x20ac 0n8364  ffffc181efe4d080 ServiceHub.Host.netfx.x86.exe*32        .  .  .  4  .  .  .
0x1cbc 0n7356  ffffc181f21c50c0 Notepad.exe                             .  .  .  .  .  .  .
0x17c0 0n6080  ffffc181ecc0d0c0 dllhost.exe                             .  .  .  .  .  .  .
0x1af8 0n6904  ffffc181f0af3080 cmd.exe                                 .  .  .  .  .  .  .
0x1b60 0n7008  ffffc181f21ce080 conhost.exe                             .  1  .  .  .  .  .
============== ================ ====================================== == == == == == == ==
PID            Address          Name                                   !! Rn Ry Bk Lc IO Er

Warning! Zombie process(es) detected (not displayed). Count: 31 [zombie report]
```

From the **`!mex.tl -t`** command, we can see that there are a couple of running threads. Let's examine the running threads on the system. The output of the **`!mex.running`** command in provides a snapshot of the currently running threads on the system.

```
1: kd> !mex.running
Process      PID Thread             Id Pri Base Pri Next CPU CSwitches User    Kernel State         Time Reason
=========== ==== ================ ==== === ======== ======== ========= ==== ========= ======= ========== =============
Idle           0 fffff80149f4d700    0   0        0        0    619812    0 7m:30.000 Running          0 Executive
System         4 ffffc181f17e7080 1480   8        8        1        12    0         0 Running          0 WrDispatchInt
conhost.exe 1b60 ffffc181f06a8080  554  11        8        2       453    0      78ms Running          0 Executive
Idle           0 ffffc181e8f35080    0   0        0        3    389658    0 8m:58.797 Running      546ms Executive
System         4 ffffc181ef7ea540 297c   8        8        4         1    0         0 Running 15m:08.984 Executive
System         4 ffffc181f0960040 1d5c   8        8        5         1    0         0 Running 15m:08.984 Executive

Count: 6 | Show Unique Stacks
```

The **`!mex.us -cpu`** command is used to display unique call stacks of threads that are currently running on a CPU. It's specifically useful in kernel mode for analyzing CPU usage by showing call stacks in a CPU-focused context, which helps to identify which threads are actively utilizing the CPU. 

From the first look, there are two threads of interest. The call stack indicates that both of these threads, although are in a running state, are currently executing a spinlock acquisition routine. The presence of **`nt!KxWaitForSpinLockAndAcquire`** and **`nt!KeAcquireSpinLockRaiseToDpc`** indicates it's actively trying to acquire a spinlock and is possibly in a busy-wait loop waiting for the spinlock to become available.

```
1: kd> !mex.us -cpu
1 thread: ffffc181ef7ea540 --> Waiting to acquiring a Spinlock
    fffff8014944059c nt!KxWaitForSpinLockAndAcquire+0x1c
    fffff8014944056e nt!KeAcquireSpinLockRaiseToDpc+0x8e
    fffff8016d431153 TestDriver+0x1153
    fffff801495573d7 nt!PspSystemThreadStartup+0x57
    fffff8014961b8d4 nt!KiStartSystemThread+0x34

1 thread: ffffc181f06a8080
    00007ffec1adfdc2 0x7ffec1adfdc2

1 thread: ffffc181f0960040 
    fffff8014944059c nt!KxWaitForSpinLockAndAcquire+0x1c --> Waiting to acquiring a Spinlock
    fffff8014944056e nt!KeAcquireSpinLockRaiseToDpc+0x8e
    fffff8016d431468 TestDriver+0x1468
    fffff801495573d7 nt!PspSystemThreadStartup+0x57
    fffff8014961b8d4 nt!KiStartSystemThread+0x34

1 thread: ffffc181f17e7080
    fffff80149616970 nt!KeBugCheckEx
    fffff8014962bfa9 nt!KiBugCheckDispatch+0x69
    fffff80149627634 nt!KiPageFault+0x474
    fffff8014d54c7ae Ntfs!NtfsFindPrefixHashEntry+0x1de
    fffff8014d54e016 Ntfs!NtfsFindStartingNode+0x886
    fffff8014d54fa0d Ntfs!NtfsCommonCreate+0x11dd
    fffff8014d54af00 Ntfs!NtfsFsdCreate+0x200
    fffff80149453075 nt!IofCallDriver+0x55
    fffff8014c63a1db FLTMGR!FltpLegacyProcessingAfterPreCallbacksCompleted+0x15b
    fffff8014c671f83 FLTMGR!FltpCreate+0x323
    fffff80149453075 nt!IofCallDriver+0x55
    fffff801498fd7de nt!IopParseDevice+0x8be
    fffff801498f81f1 nt!ObpLookupObjectName+0x7e1
    fffff801498f74c2 nt!ObOpenObjectByNameEx+0x1f2
    fffff801498f4f51 nt!IopCreateFile+0x431
    fffff8014992e738 nt!NtOpenFile+0x58
    fffff8014962b6e5 nt!KiSystemServiceCopyEnd+0x25
    fffff8014961c200 nt!KiServiceLinkage
    fffff8016d431375 TestDriver+0x1375
    fffff801495573d7 nt!PspSystemThreadStartup+0x57
    fffff8014961b8d4 nt!KiStartSystemThread+0x34

2 threads: ffffc181e8f35080 fffff80149f4d700
    fffff801495c86ad nt!PpmIdleGuestExecute+0x1d
    fffff8014947665f nt!PpmIdleExecuteTransition+0x41f
    fffff80149476011 nt!PoIdle+0x361
    fffff8014961b724 nt!KiIdleLoop+0x54
```

The thread **`ffffc181ef7ea540`** from the **System** process is currently running for **15m:08.984** on **CPU 4** and is engaged in acquiring a spinlock. The call stack, showing **`nt!KxWaitForSpinLockAndAcquire`** and **`nt!KeAcquireSpinLockRaiseToDpc`**, indicates that the thread is in the process of obtaining a spinlock and waiting in a busy loop for the spinlock to become available. This already violates the rules of Spinlock, since Spinlock should not be hold for longer than 25 microseconds.

```
0: kd> !mex.t ffffc181f0960040
Process                   Thread                       CID       UserTime KernelTime ContextSwitches Wait Reason       Time State
System (ffffc181e8eea040) ffffc181f0960040 (E|K|W|R|V) 4.1d5c           0          0               1 Executive   15m:08.984 Running on processor 5

# Child-SP         Return           Call Site
0 fffffb0289e67310 fffff8014944056e nt!KxWaitForSpinLockAndAcquire+0x1c
1 fffffb0289e67340 fffff8016d431468 nt!KeAcquireSpinLockRaiseToDpc+0x8e
2 fffffb0289e67370 fffff801495573d7 TestDriver+0x1468
3 fffffb0289e67570 fffff8014961b8d4 nt!PspSystemThreadStartup+0x57
4 fffffb0289e675c0 0000000000000000 nt!KiStartSystemThread+0x34
```

The thread **`ffffc181f0960040`** in the **System** process, running for **15m:08.984** on **CPU 5** is also involved in a spinlock acquisition. The call stack, beginning with **`nt!KxWaitForSpinLockAndAcquire`**, indicates it's actively attempting to acquire a spinlock, similar to the previous thread. 

```
0: kd> !mex.t ffffc181f0960040
Process                   Thread                       CID       UserTime KernelTime ContextSwitches Wait Reason       Time State
System (ffffc181e8eea040) ffffc181f0960040 (E|K|W|R|V) 4.1d5c           0          0               1 Executive   15m:08.984 Running on processor 5

# Child-SP         Return           Call Site
0 fffffb0289e67310 fffff8014944056e nt!KxWaitForSpinLockAndAcquire+0x1c
1 fffffb0289e67340 fffff8016d431468 nt!KeAcquireSpinLockRaiseToDpc+0x8e
2 fffffb0289e67370 fffff801495573d7 TestDriver+0x1468
3 fffffb0289e67570 fffff8014961b8d4 nt!PspSystemThreadStartup+0x57
4 fffffb0289e675c0 0000000000000000 nt!KiStartSystemThread+0x34
```

From the **`!mex.tl -t`** command, we can see that there is one thread of the **System** process in a readyish state.

```
0: kd> !mex.lt ffffc181e8eea040 -state ready,deferredready,standby,rescheduled,transition
Process PID Thread           Id State   Time Reason
======= === ================ == ======= ==== =======
System    4 ffffc181e90c3400 f8 Standby    0 WrQueue

Thread Count: 1
```

The thread **`ffffc181e90c3400`** from the **System** process is currently in a **standby** state, waiting for its turn to execute on **CPU 4**. The wait reason being **WrQueue** indicates it's waiting in a scheduler queue. The thread has experienced a significant number of context switches. A context switch is like a CPU pausing one task to start or resume another. It's necessary for multitasking, but too many switches can slow things down. 

Imagine a chef cooking multiple dishes; constantly switching between them isn't efficient. High numbers of context switches might mean the system is juggling too many tasks at once, leading to reduced efficiency and performance.

```
0: kd> !mex.t ffffc181e90c3400
Process                   Thread                       CID       UserTime KernelTime ContextSwitches Wait Reason Time State
System (ffffc181e8eea040) ffffc181e90c3400 (E|K|W|R|V) 4.f8             0     4s.219          116759 WrQueue        0 Standby on processor 4

Priority:
    Current Base Decrement ForegroundBoost IO Page
    13      13   0         0               0  5

# Child-SP         Return           Call Site
0 fffffb0288a41fa0 fffff801494cb3f5 nt!KiSwapContext+0x76
1 fffffb0288a420e0 fffff801494cd5d7 nt!KiSwapThread+0xab5
2 fffffb0288a42230 fffff8014941d84f nt!KiCommitThreadWait+0x137
3 fffffb0288a422e0 fffff8014941d203 nt!KeRemovePriQueue+0x1ff
4 fffffb0288a42380 fffff801495573d7 nt!ExpWorkerThread+0xd3
5 fffffb0288a42570 fffff8014961b8d4 nt!PspSystemThreadStartup+0x57
6 fffffb0288a425c0 0000000000000000 nt!KiStartSystemThread+0x34
```


The Processor Control Region (PCR) is a data structure in Windows operating systems that provides each processor with quick access to important data specific to that processor, like the current thread and processor control block.

 The **`CurrentThread`** field **`(ffffc181ef7ea540)`** points to the Thread Control Block of the thread currently running on this CPU. The **`NextThread`** field **`(ffffc181e90c3400)`** indicates the next thread scheduled to run. These addresses are for identifying and analyzing the active and upcoming thread execution in a debugging session.

```
 0: kd> !pcr 4
KPCR for Processor 4 at ffffa001290c6000:
    Major 1 Minor 1
	NtTib.ExceptionList: ffffa001290d6fb0
	    NtTib.StackBase: ffffa001290d5000
	   NtTib.StackLimit: 0000003f6737f968
	 NtTib.SubSystemTib: ffffa001290c6000
	      NtTib.Version: 00000000290c6180
	  NtTib.UserPointer: ffffa001290c6870
	      NtTib.SelfTib: 0000003f66fd3000

	            SelfPcr: 0000000000000000
	               Prcb: ffffa001290c6180
	               Irql: 0000000000000000
	                IRR: 0000000000000000
	                IDR: 0000000000000000
	      InterruptMode: 0000000000000000
	                IDT: 0000000000000000
	                GDT: 0000000000000000
	                TSS: 0000000000000000

	      CurrentThread: ffffc181ef7ea540
	         NextThread: ffffc181e90c3400
	         IdleThread: ffffc181e8ec7080

	          DpcQueue:  0xffffc181e9baf728 0xfffff8014da1cee0 [Normal] tcpip!TcpPeriodicTimeoutHandler
	                     0xffffa001290ce5d8 0xfffff8014941e4e0 [Normal] nt!PpmPerfAction
```
