# Description

Spinlocks are a synchronization mechanism used for protecting shared resources in multi-processor environments. They are utilized when routines run at or above **`DISPATCH_LEVEL`**, preventing simultaneous access to shared data. **Spinlocks are efficient for short-term locking where waiting time is minimal**. However, they must be used cautiously, as holding them for extended periods (more than 25 microseconds) can lead to system performance issues, such as BSOD.

In this scenario, we initially obtain a Spinlock while operating at **`DISPATCH_LEVEL`** in terms of Interrupt Request Level (IRQL). Even after we release the Spinlock, the IRQL remains unchanged at **`DISPATCH_LEVEL`**.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/45fade25-f716-4376-a13b-f967ee80a2be)



Here are some of the characteristics:

- **Non-Blocking:** Unlike other synchronization primitives, spinlocks do not block or put the current thread to sleep if the lock is not available. Instead, they keep the thread actively waiting.
- **High IRQL Usage:** Spinlocks often operate at high IRQLs (Interrupt Request Levels), such as **`DISPATCH_LEVEL`**, which prevents preemption and ensures that the code holding the lock runs uninterrupted.
- **No Ownership Information:** Spinlocks do not keep track of the thread or processor that holds them, meaning they can't be used recursively.

Here are the common Spinlock APIs:

| Function                        | Description                                                                                                              |
|---------------------------------|--------------------------------------------------------------------------------------------------------------------------|
| `KeInitializeSpinLock`          | Initializes a spinlock. Prepares the spinlock for use by setting its initial state to unlocked.                          |
| `KeAcquireSpinLock`             | Acquires a spinlock for exclusive access, raising the IRQL to `DISPATCH_LEVEL`. Blocks other threads from access.        |
| `KeReleaseSpinLock`             | Releases a spinlock from exclusive access and restores the IRQL to its previous value.                                   |
| `KeAcquireSpinLockAtDpcLevel`   | Acquires a spinlock without changing the IRQL, assuming the current IRQL is already `DISPATCH_LEVEL` or higher.         |
| `KeReleaseSpinLockFromDpcLevel` | Releases a spinlock without changing the IRQL, to be used when the current IRQL is `DISPATCH_LEVEL` or higher.          |
| `KeTryToAcquireSpinLockAtDpcLevel` | Tries to acquire a spinlock at `DISPATCH_LEVEL` without waiting. Returns immediately if the lock is not available.  |

# Code Sample - File I/O should not use Spinlocks

In order to demonstrate this demo. Please follow the following steps:

- Create the following 2 folders:
  1. **C:\Temp**
  2. **C:\Temp2**

Using Spinlocks for File I/O can lead to system crashes because spinlocks are designed for short, quick operations at high CPU levels. File I/O, however, often involves longer, waiting processes. If a spinlock is held during such operations, it can disrupt the system's ability to manage other critical tasks, potentially leading to a crash.

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

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/190e0f94-54ac-4bd3-aaf2-ccb44316ebeb)


# Debug Details

Load the Memory Dump in WinDbg:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/0032e36d-4f20-4dbc-a368-5464d9560e52)


Let's start with the **`!mex.di`** command which stands for dump information. This command retrieves information about the machine from which the memory dump originated. It's especially handy when working with multiple memory dumps, as it helps distinguish the analysis related to each specific dump. This is an improved version of the **`vertarget`** command.

```
0: kd> !mex.di
Computer Name:  Not Found
Windows 10 Kernel Version 22621 MP (6 procs) Free x64
Product: WinNt, suite: TerminalServer SingleUserTS
Edition build lab: 22621.2506.amd64fre.ni_release_svc_prod3.231018-1809
Kernel base = 0xfffff802`31600000 PsLoadedModuleList = 0xfffff802`322134a0
Debug session time: Tue Dec 26 07:57:39.921 2023 (UTC - 8:00)
System Uptime: 0 days 0:05:49.516
SystemManufacturer = Microsoft Corporation
SystemProductName = Virtual Machine
Processor: 11th Gen Intel(R) Core(TM) i7-1185G7 @ 3.00GHz
Bugcheck: D1 (FFFFF802364E5641, 2, 8, FFFFF802364E5641)
```

The **`!mex.tl -t`** command examines the state and wait reasons for every thread across all processes, presenting the results as statistics. This allows for a quick overview, such as determining how many processes have 2 running threads, and so on. It's like having a satellite view of all the thread state that are associated with the processes.

```
0: kd> !mex.tl -t
PID            Address          Name                        !! Rn Ry Bk Lc IO Er
============== ================ =========================== == == == == == == ==
0x0    0n0     fffff80232349f40 Idle                         .  6  6  .  .  .  .
0x4    0n4     ffffab026f2b4040 System                       6  2  .  .  .  1  .
0x84   0n132   ffffab026f31d080 Registry                     .  .  .  .  .  .  .
0x238  0n568   ffffab02726ae040 smss.exe                     .  .  .  1  .  .  .
0x2c4  0n708   ffffab02724ce0c0 csrss.exe                    .  .  .  .  1  .  .
0x30c  0n780   ffffab0273bc8080 wininit.exe                  .  .  .  .  .  .  .
0x39c  0n924   ffffab0273c20080 services.exe                 .  .  1  .  .  .  .
0x3cc  0n972   ffffab0273c510c0 LsaIso.exe                   .  .  .  .  .  .  .
0x3d8  0n984   ffffab0273c4f080 lsass.exe                    .  .  .  .  .  .  .
0x324  0n804   ffffab0273ce00c0 svchost.exe                  .  .  .  .  .  .  .
0x26c  0n620   ffffab0273ce6080 fontdrvhost.exe              .  .  .  .  .  .  .
0x46c  0n1132  ffffab0273ddf0c0 svchost.exe                  .  .  .  .  .  .  .
0x494  0n1172  ffffab0273e230c0 svchost.exe                  .  .  .  .  .  .  .
0x534  0n1332  ffffab0273f34080 svchost.exe                  .  .  .  .  .  .  .
0x588  0n1416  ffffab0273f5d080 svchost.exe                  .  .  .  .  .  .  .
0x5b8  0n1464  ffffab0273f8b080 svchost.exe                  .  .  .  .  .  .  .
0x5c0  0n1472  ffffab0273f8e080 svchost.exe                  .  .  .  .  .  .  .
0x5c8  0n1480  ffffab0273f9a0c0 svchost.exe                  .  .  .  .  .  .  .
0x5d0  0n1488  ffffab0273f89080 svchost.exe                  .  .  .  .  .  .  .
0x608  0n1544  ffffab0273fa9080 svchost.exe                  .  .  .  .  .  .  .
0x634  0n1588  ffffab0274009080 svchost.exe                  .  .  .  .  .  .  .
0x644  0n1604  ffffab027400a080 svchost.exe                  .  .  .  .  .  .  .
0x650  0n1616  ffffab0274007080 svchost.exe                  .  .  .  .  .  .  .
0x674  0n1652  ffffab0274010080 svchost.exe                  .  .  .  .  .  .  .
0x68c  0n1676  ffffab02740130c0 svchost.exe                  .  .  .  .  .  .  .
0x6c0  0n1728  ffffab0274020080 svchost.exe                  .  .  .  .  1  .  .
0x7a4  0n1956  ffffab02740af080 svchost.exe                  .  .  .  .  .  .  .
0x7b0  0n1968  ffffab02740b2080 VSSVC.exe                    .  .  .  .  .  .  .
0x354  0n852   ffffab027413e0c0 svchost.exe                  .  .  .  .  .  .  .
0x7fc  0n2044  ffffab027412d0c0 svchost.exe                  .  .  .  .  .  .  .
0x3e0  0n992   ffffab0274132080 svchost.exe                  .  .  .  .  .  .  .
0x4a4  0n1188  ffffab0274130080 svchost.exe                  .  .  .  .  .  .  .
0x4ec  0n1260  ffffab0274131080 svchost.exe                  .  .  .  .  .  .  .
0x570  0n1392  ffffab0274135080 svchost.exe                  .  .  .  .  .  .  .
0x904  0n2308  ffffab0274242080 svchost.exe                  .  .  .  .  .  .  .
0x90c  0n2316  ffffab0274243040 MemCompression               .  .  .  .  .  .  .
0x928  0n2344  ffffab027425a080 svchost.exe                  .  .  .  .  .  .  .
0x96c  0n2412  ffffab0274276080 svchost.exe                  .  .  .  .  .  .  .
0x988  0n2440  ffffab027427b0c0 svchost.exe                  .  .  .  .  .  .  .
0x9a4  0n2468  ffffab02742ab080 svchost.exe                  .  .  .  .  .  .  .
0x9ac  0n2476  ffffab02742ae0c0 svchost.exe                  .  .  .  .  .  .  .
0xa18  0n2584  ffffab02743cf0c0 svchost.exe                  .  .  .  .  .  .  .
0xa50  0n2640  ffffab027436b080 svchost.exe                  .  .  .  .  .  .  .
0xa64  0n2660  ffffab02743680c0 svchost.exe                  .  .  .  .  .  .  .
0xad4  0n2772  ffffab026f27a080 svchost.exe                  .  .  .  .  .  .  .
0xb20  0n2848  ffffab026f3bb080 svchost.exe                  .  .  .  .  .  .  .
0xb68  0n2920  ffffab026f391080 svchost.exe                  .  .  .  .  .  .  .
0xba4  0n2980  ffffab026f358080 svchost.exe                  .  .  .  .  .  .  .
0xb74  0n2932  ffffab026f403080 svchost.exe                  .  .  .  .  .  .  .
0xbb4  0n2996  ffffab026f318080 svchost.exe                  .  .  .  .  .  .  .
0xaa4  0n2724  ffffab026f3ea080 svchost.exe                  .  .  .  .  .  .  .
0xc40  0n3136  ffffab027452c080 svchost.exe                  .  .  .  .  .  .  .
0xcac  0n3244  ffffab02745eb080 svchost.exe                  .  .  .  .  .  .  .
0xcb8  0n3256  ffffab02745d60c0 svchost.exe                  .  .  .  .  .  .  .
0xd60  0n3424  ffffab027466b080 svchost.exe                  .  .  .  .  .  .  .
0xdd0  0n3536  ffffab02746d90c0 spoolsv.exe                  .  .  .  .  .  .  .
0xea4  0n3748  ffffab0274776140 csrss.exe                    .  .  .  .  1  .  .
0xec0  0n3776  ffffab0274787140 svchost.exe                  .  .  .  .  .  .  .
0xf10  0n3856  ffffab0274811080 IpOverUsbSvc.exe*32          .  .  .  .  .  .  .
0xf1c  0n3868  ffffab0274821080 sqlwriter.exe                .  .  .  .  .  .  .
0xf30  0n3888  ffffab0274866080 MsMpEng.exe                  .  1  .  .  .  .  .
0xf6c  0n3948  ffffab02748a8080 winlogon.exe                 .  .  .  .  .  .  .
0xf7c  0n3964  ffffab02748cc080 wlms.exe                     .  .  .  .  .  .  .
0xf88  0n3976  ffffab02748db080 svchost.exe                  .  .  .  .  .  .  .
0xf90  0n3984  ffffab02748ec080 svchost.exe                  .  .  .  .  .  .  .
0xf98  0n3992  ffffab027490f080 svchost.exe                  .  .  .  .  .  .  .
0xfa0  0n4000  ffffab02748dc080 svchost.exe                  .  .  .  .  .  .  .
0xfb0  0n4016  ffffab0274b620c0 svchost.exe                  .  .  .  .  .  .  .
0xfb8  0n4024  ffffab02748ca080 svchost.exe                  .  .  .  .  .  .  .
0xfc0  0n4032  ffffab0274976080 svchost.exe                  .  .  .  .  .  .  .
0xfe4  0n4068  ffffab02749db080 svchost.exe                  .  .  .  .  .  .  .
0xd58  0n3416  ffffab0274a77080 svchost.exe                  .  .  .  .  .  .  .
0x100c 0n4108  ffffab0274a54080 sppsvc.exe                   .  .  .  .  .  .  .
0x1350 0n4944  ffffab0274eb6080 dwm.exe                      .  .  .  .  .  .  .
0x1460 0n5216  ffffab02760440c0 AggregatorHost.exe           .  .  .  .  .  .  .
0x14d8 0n5336  ffffab0276081080 unsecapp.exe                 .  .  .  .  .  .  .
0x1778 0n6008  ffffab02747ce0c0 fontdrvhost.exe              .  .  .  .  .  .  .
0x1608 0n5640  ffffab02762b1080 svchost.exe                  .  .  .  .  .  .  .
0x160c 0n5644  ffffab02762cf080 svchost.exe                  .  .  .  .  .  .  .
0x1604 0n5636  ffffab02762ce080 svchost.exe                  .  .  .  .  1  .  .
0x15f4 0n5620  ffffab02762cd080 svchost.exe                  .  .  .  .  .  .  .
0x1890 0n6288  ffffab02760ea0c0 rdpclip.exe                  .  .  .  .  .  .  .
0x18e8 0n6376  ffffab02762cc080 sihost.exe                   .  .  .  .  .  .  .
0x1920 0n6432  ffffab02762ca0c0 svchost.exe                  .  .  .  .  .  .  .
0x1974 0n6516  ffffab02762c8080 svchost.exe                  .  .  .  .  .  .  .
0x19d4 0n6612  ffffab02762c7080 svchost.exe                  .  .  .  .  .  .  .
0x19dc 0n6620  ffffab02762c6080 svchost.exe                  .  .  .  .  .  .  .
0x1a54 0n6740  ffffab02762c4080 svchost.exe                  .  .  .  .  .  .  .
0x1acc 0n6860  ffffab02762c1080 taskhostw.exe                .  .  .  .  .  .  .
0x1af0 0n6896  ffffab02762bf080 svchost.exe                  .  .  .  .  .  .  .
0x1b80 0n7040  ffffab0276941080 svchost.exe                  .  .  .  .  .  .  .
0x1b3c 0n6972  ffffab027693e080 SppExtComObj.Exe             .  .  .  .  .  .  .
0x1bbc 0n7100  ffffab027693f080 svchost.exe                  .  .  .  .  .  .  .
0x1c74 0n7284  ffffab027693b080 explorer.exe                 3  .  .  .  1  .  .
0x1df0 0n7664  ffffab0276937080 svchost.exe                  .  .  .  .  .  .  .
0x1eb4 0n7860  ffffab0276936080 svchost.exe                  .  .  .  .  .  .  .
0x1ef8 0n7928  ffffab02768f5080 svchost.exe                  .  .  .  .  .  .  .
0x1ae8 0n6888  ffffab0274dda0c0 rdpinput.exe                 .  .  .  .  .  .  .
0x13e4 0n5092  ffffab027693a080 ctfmon.exe                   .  .  .  .  .  .  .
0x1cb4 0n7348  ffffab02762530c0 svchost.exe                  .  .  .  .  .  .  .
0x1a1c 0n6684  ffffab02766e6080 svchost.exe                  .  .  .  .  .  .  .
0x1928 0n6440  ffffab0276e6d140 csrss.exe                    .  .  .  .  1  .  .
0x16ac 0n5804  ffffab0276e6e080 winlogon.exe                 .  .  .  .  1  .  .
0x378  0n888   ffffab0276eb10c0 fontdrvhost.exe              .  .  .  .  .  .  .
0xa3c  0n2620  ffffab0276f020c0 LogonUI.exe                  .  .  .  .  .  .  .
0x155c 0n5468  ffffab0276db8080 dwm.exe                      .  .  .  .  .  .  .
0xbfc  0n3068  ffffab0276f1f080 WUDFHost.exe                 .  .  .  .  .  .  .
0x1314 0n4884  ffffab0276d750c0 TabTip.exe                   .  .  .  .  .  .  .
0x16a8 0n5800  ffffab0274eb3080 svchost.exe                  .  .  .  .  .  .  .
0x2034 0n8244  ffffab0276fa5080 SearchHost.exe               6  .  .  .  2  .  .
0x2058 0n8280  ffffab027260f080 StartMenuExperienceHost.exe  .  .  .  .  .  .  .
0x20a0 0n8352  ffffab027670f080 Widgets.exe                  .  .  .  .  .  .  .
0x20f4 0n8436  ffffab02774df0c0 RuntimeBroker.exe            .  .  .  .  .  .  .
0x2144 0n8516  ffffab02766ae080 RuntimeBroker.exe            .  .  .  .  1  .  .
0x21b0 0n8624  ffffab0276f980c0 svchost.exe                  .  .  .  .  .  .  .
0x22a0 0n8864  ffffab0276bdd080 dllhost.exe                  .  .  .  .  .  .  .
0x23b8 0n9144  ffffab02774ad080 NisSrv.exe                   .  .  .  .  .  .  .
0x2524 0n9508  ffffab02700430c0 TextInputHost.exe            .  .  .  .  .  .  .
0x2668 0n9832  ffffab0274106080 svchost.exe                  .  .  .  .  .  .  .
0x27b8 0n10168 ffffab027726f080 smartscreen.exe              .  .  .  .  .  .  .
0x27e4 0n10212 ffffab02774020c0 SecurityHealthSystray.exe    .  .  .  .  .  .  .
0x27f8 0n10232 ffffab0272608080 SecurityHealthService.exe    .  .  .  .  .  .  .
0x554  0n1364  ffffab02720eb080 msedge.exe                   .  .  .  .  .  .  .
0x1014 0n4116  ffffab0277bc70c0 msedge.exe                   .  .  .  .  .  .  .
0x24a0 0n9376  ffffab0277b990c0 msedge.exe                   .  .  .  .  .  .  .
0x2494 0n9364  ffffab0277a110c0 msedge.exe                   .  .  .  .  .  .  .
0x1be4 0n7140  ffffab0277aaa0c0 msedge.exe                   .  .  .  .  .  .  .
0x2890 0n10384 ffffab0277ac20c0 SearchIndexer.exe            .  .  .  .  .  .  .
0x29c0 0n10688 ffffab02748240c0 taskhostw.exe                .  .  .  .  .  .  .
0x25ec 0n9708  ffffab027040e0c0 svchost.exe                  .  .  .  .  1  .  .
0x1fa4 0n8100  ffffab0277a460c0 svchost.exe                  .  .  .  .  .  .  .
0x2af8 0n11000 ffffab02779c80c0 uhssvc.exe                   .  .  .  .  .  .  .
0x296c 0n10604 ffffab02779b70c0 svchost.exe                  .  .  .  .  .  .  .
0x23d8 0n9176  ffffab0276d7c080 MoUsoCoreWorker.exe          .  .  .  .  .  .  .
0x1334 0n4916  ffffab02766e0080 svchost.exe                  .  .  .  .  .  .  .
0x5f8  0n1528  ffffab027029a080 svchost.exe                  .  .  .  .  .  .  .
0x7ac  0n1964  ffffab02720e8080 svchost.exe                  .  .  .  .  .  .  .
0x29c  0n668   ffffab027799f0c0 WmiPrvSE.exe                 .  .  .  .  .  .  .
0x1b2c 0n6956  ffffab02766c00c0 svchost.exe                  .  .  .  .  .  .  .
0xbec  0n3052  ffffab02779c30c0 svchost.exe                  .  .  .  .  .  .  .
0x53c  0n1340  ffffab0277adf0c0 dllhost.exe                  .  .  .  .  .  .  .
0x18a8 0n6312  ffffab0276db7080 audiodg.exe                  .  .  .  .  .  .  .
0xc50  0n3152  ffffab02703460c0 svchost.exe                  .  .  .  .  .  .  .
0x19c0 0n6592  ffffab02774130c0 WMIADAP.exe                  .  .  .  .  .  .  .
0x1970 0n6512  ffffab0276ab70c0 WmiPrvSE.exe                 .  .  .  .  .  .  .
0x430  0n1072  ffffab02700750c0 WmiApSrv.exe                 .  .  .  .  .  .  .
0x1ee8 0n7912  ffffab02702ec0c0 svchost.exe                  .  .  .  .  .  .  .
0x19a0 0n6560  ffffab0274153080 wuaucltcore.exe              .  .  .  .  1  .  .
0x16a4 0n5796  ffffab0277bdc0c0 TrustedInstaller.exe         .  .  .  .  1  .  .
0x1680 0n5760  ffffab02772db0c0 TiWorker.exe                 .  .  .  .  .  .  .
0x394  0n916   ffffab0277b680c0 SearchProtocolHost.exe       .  .  .  .  .  .  .
0x2a90 0n10896 ffffab0274f130c0 WmiPrvSE.exe                 .  .  .  .  .  .  .
0x2100 0n8448  ffffab02779b10c0 cmd.exe                      .  .  .  1  .  .  .
0x24b8 0n9400  ffffab02702d70c0 conhost.exe                  .  1  .  .  .  .  .
0x76c  0n1900  ffffab02772350c0 ShellExperienceHost.exe      3  .  .  .  .  .  .
0x1518 0n5400  ffffab0272ea90c0 svchost.exe                  .  .  .  .  .  .  .
0x23f8 0n9208  ffffab02741ae080 svchost.exe                  .  .  .  .  .  .  .
0x2224 0n8740  ffffab0273fce080 WidgetService.exe            .  .  .  .  .  .  .
0x2ba4 0n11172 ffffab0274d20080 sc.exe                       .  .  .  .  1  .  .
============== ================ =========================== == == == == == == ==
PID            Address          Name                        !! Rn Ry Bk Lc IO Er

Warning! Zombie process(es) detected (not displayed). Count: 16 [zombie report]
```

From the **`!mex.tl -t`** command, we can see that there are a couple of running threads. Let's examine the running threads on the system. The output of the **`!mex.running`** command in provides a snapshot of the currently running threads on the system.

```
0: kd> !mex.running
Process      PID Thread             Id Pri Base Pri Next CPU CSwitches  User    Kernel State        Time Reason
=========== ==== ================ ==== === ======== ======== ========= ===== ========= ======= ========= ==============
MsMpEng.exe  f30 ffffab0274a31080 22e8   9        8        0      4407 453ms     172ms Running      15ms UserRequest
System         4 ffffab027465f040  b48   8        8        1         1     0         0 Running 5m:49.515 Executive
Idle           0 ffffab026f3ac080    0   0        0        2    200204     0 3m:32.359 Running 1m:09.343 Executive
conhost.exe 24b8 ffffab02739de080 1514  11        8        3       348     0      78ms Running      31ms DelayExecution
Idle           0 ffffab026f313080    0   0        0        4    205730     0 4m:20.547 Running   21s.890 WrCalloutStack
System         4 ffffab02745f3040  ab0   8        8        5         1     0         0 Running 5m:49.515 Executive
```

Here we can see a situation where a thread in the **System** process is spending an unusually long time trying to acquire a spinlock. This thread has been in a running state state for approximately **5 minutes** and **49.515 seconds**. The call stack shows that the thread is currently in **`nt!KxWaitForSpinLockAndAcquire`**, indicating it is waiting to acquire a spinlock.

```
5: kd> !mex.t ffffab027465f040
Process                   Thread                       CID       UserTime KernelTime ContextSwitches Wait Reason      Time State
System (ffffab026f2b4040) ffffab027465f040 (E|K|W|R|V) 4.b48            0          0               1 Executive   5m:49.515 Running on processor 1

# Child-SP         Return           Call Site
0 fffffb00851ef1d0 fffff8023184056e nt!KxWaitForSpinLockAndAcquire+0x1c
1 fffffb00851ef200 fffff802516a1297 nt!KeAcquireSpinLockRaiseToDpc+0x8e
2 fffffb00851ef230 fffff802319573d7 TestDriver+0x1297
3 fffffb00851ef570 fffff80231a1b8d4 nt!PspSystemThreadStartup+0x57
4 fffffb00851ef5c0 0000000000000000 nt!KiStartSystemThread+0x34
```

This documentation of Microsoft emphasizes that routines holding a spin lock should execute quickly and should not hold the spin lock for longer than **25 microseconds**, while our thread is holding a spinlock for **5m:49.515**.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/c40024d8-de6e-4a5b-8527-3d2c18044b31)

