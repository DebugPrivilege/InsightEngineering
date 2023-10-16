# What is an Executive Resource?

An ERESOURCE is a synchronization primitive used in the Windows kernel for managing access to shared resources. It functions as a multi-reader, single-writer lock, which means it allows multiple threads to read a resource simultaneously while ensuring that only one thread can write to it at any given time. 

ERESOURCE is set up so that a single thread can safely lock a resource more than once in the same way, either for reading or writing, without running into issues. But if that thread wants to change from a read-lock to a write-lock, or the other way around, it has to first unlock the resource.

Here are the common ERESOURCE Kernel Mode APIs:

| Function                               | Description  |
| -------------------------------------- | ------------ |
| `ExInitializeResourceLite()`           | Initializes an `ERESOURCE` object for use. |
| `ExAcquireResourceSharedLite()`        | Acquires an `ERESOURCE` object for shared access. |
| `ExReleaseResourceLite()`              | Releases an acquired `ERESOURCE` object. |
| `ExReleaseResourceForThreadLite()`     | Releases an acquired `ERESOURCE` object for a specific thread. |
| `ExAcquireResourceExclusiveLite()`     | Acquires an `ERESOURCE` object for exclusive access. |
| `ExConvertExclusiveToSharedLite()`     | Converts an exclusive lock to a shared lock on an `ERESOURCE` object. |

ERESOURCE is frequently utilized by both third-party and Microsoft drivers.

```c
0: kd> !imports -m -f ExInitializeResourceLite
The following Modules import a function that contains the string: ExInitializeResourceLite
fffff80053200000 (srv2)               [Show Matches !imports]
    srv2 imports ntoskrnl.exe!ExInitializeResourceLite
    fffff800`53245198  fffff800`560766d0 nt!ExInitializeResourceLite

fffff800532e0000 (WUDFRd)             [Show Matches !imports]
    WUDFRd imports ntoskrnl.exe!ExInitializeResourceLite
    fffff800`5332a628  fffff800`560766d0 nt!ExInitializeResourceLite

fffff80053340000 (peauth)             [Show Matches !imports]
    peauth imports ntoskrnl.exe!ExInitializeResourceLite
    fffff800`5337a100  fffff800`560766d0 nt!ExInitializeResourceLite

fffff80053420000 (IndirectKmd)        [Show Matches !imports]
    IndirectKmd imports ntoskrnl.exe!ExInitializeResourceLite
    fffff800`53427040  fffff800`560766d0 nt!ExInitializeResourceLite

fffff80053460000 (wtd)                [Show Matches !imports]
    wtd imports ntoskrnl.exe!ExInitializeResourceLite
    fffff800`5346e188  fffff800`560766d0 nt!ExInitializeResourceLite

fffff80053510000 (EvilDriver)         [Show Matches !imports]
    EvilDriver imports ntoskrnl.exe!ExInitializeResourceLite
    fffff800`535130a0  fffff800`560766d0 nt!ExInitializeResourceLite

fffff80054330000 (srvnet)             [Show Matches !imports]
    srvnet imports ntoskrnl.exe!ExInitializeResourceLite
    fffff800`543678f0  fffff800`560766d0 nt!ExInitializeResourceLite

fffff80058a00000 (tm)                 [Show Matches !imports]
    tm imports ntoskrnl.exe!ExInitializeResourceLite
    fffff800`58a09338  fffff800`560766d0 nt!ExInitializeResourceLite

fffff80058a30000 (CLFS)               [Show Matches !imports]
    CLFS imports ntoskrnl.exe!ExInitializeResourceLite
    fffff800`58a54098  fffff800`560766d0 nt!ExInitializeResourceLite

fffff80058bf0000 (FLTMGR)             [Show Matches !imports]
    FLTMGR imports ntoskrnl.exe!ExInitializeResourceLite
    fffff800`58c23440  fffff800`560766d0 nt!ExInitializeResourceLite

fffff80058d50000 (CI)                 [Show Matches !imports]
    CI imports ntoskrnl.exe!ExInitializeResourceLite
    fffff800`58d903a8  fffff800`560766d0 nt!ExInitializeResourceLite

fffff80058e50000 (cng)                [Show Matches !imports]
    cng imports ntoskrnl.exe!ExInitializeResourceLite
    fffff800`58f00298  fffff800`560766d0 nt!ExInitializeResourceLite

fffff80058fe0000 (WDFLDR)             [Show Matches !imports]
    WDFLDR imports ntoskrnl.exe!ExInitializeResourceLite
    fffff800`58fee0b8  fffff800`560766d0 nt!ExInitializeResourceLite

fffff800591c0000 (ACPI)               [Show Matches !imports]
    ACPI imports ntoskrnl.exe!ExInitializeResourceLite
    fffff800`59237398  fffff800`560766d0 nt!ExInitializeResourceLite

fffff800593c0000 (pdc)                [Show Matches !imports]
    pdc imports ntoskrnl.exe!ExInitializeResourceLite
    fffff800`593d3238  fffff800`560766d0 nt!ExInitializeResourceLite

fffff80059460000 (spaceport)          [Show Matches !imports]
    spaceport imports ntoskrnl.exe!ExInitializeResourceLite
    fffff800`594d4508  fffff800`560766d0 nt!ExInitializeResourceLite

fffff800596a0000 (NETIO)              [Show Matches !imports]
    NETIO imports ntoskrnl.exe!ExInitializeResourceLite
    fffff800`59728570  fffff800`560766d0 nt!ExInitializeResourceLite

fffff80059750000 (NDIS)               [Show Matches !imports]
    NDIS imports ntoskrnl.exe!ExInitializeResourceLite
    fffff800`59855318  fffff800`560766d0 nt!ExInitializeResourceLite

fffff800599a0000 (WdFilter)           [Show Matches !imports]
    WdFilter imports ntoskrnl.exe!ExInitializeResourceLite
    fffff800`599c46f8  fffff800`560766d0 nt!ExInitializeResourceLite

fffff80059a40000 (Ntfs)               [Show Matches !imports]
    Ntfs imports ntoskrnl.exe!ExInitializeResourceLite
    fffff800`59affb98  fffff800`560766d0 nt!ExInitializeResourceLite

fffff80059da0000 (storport)           [Show Matches !imports]
    storport imports ntoskrnl.exe!ExInitializeResourceLite
    fffff800`59e3b4e0  fffff800`560766d0 nt!ExInitializeResourceLite

fffff80059fa0000 (ksecpkg)            [Show Matches !imports]
    ksecpkg imports ntoskrnl.exe!ExInitializeResourceLite
    fffff800`59fbb288  fffff800`560766d0 nt!ExInitializeResourceLite

fffff80059fe0000 (tcpip)              [Show Matches !imports]
    tcpip imports ntoskrnl.exe!ExInitializeResourceLite
    fffff800`5a220618  fffff800`560766d0 nt!ExInitializeResourceLite

fffff8005a380000 (fvevol)             [Show Matches !imports]
    fvevol imports ntoskrnl.exe!ExInitializeResourceLite
    fffff800`5a3b87d8  fffff800`560766d0 nt!ExInitializeResourceLite

fffff8005a470000 (volsnap)            [Show Matches !imports]
    volsnap imports ntoskrnl.exe!ExInitializeResourceLite
    fffff800`5a4895d0  fffff800`560766d0 nt!ExInitializeResourceLite

fffff8005a4f0000 (SysmonDrv)          [Show Matches !imports]
    fffff800`5a50f290  fffff800`560766d0 nt!ExInitializeResourceLite

fffff8005a580000 (mup)                [Show Matches !imports]
    mup imports ntoskrnl.exe!ExInitializeResourceLite
    fffff800`5a58b388  fffff800`560766d0 nt!ExInitializeResourceLite

fffff8005aec0000 (dxgkrnl)            [Show Matches !imports]
    dxgkrnl imports ntoskrnl.exe!ExInitializeResourceLite
    fffff800`5b016208  fffff800`560766d0 nt!ExInitializeResourceLite

fffff8005b470000 (dfsc)               [Show Matches !imports]
    dfsc imports ntoskrnl.exe!ExInitializeResourceLite
    fffff800`5b47e4e0  fffff800`560766d0 nt!ExInitializeResourceLite

fffff8005d5f0000 (netbios)            [Show Matches !imports]
    netbios imports ntoskrnl.exe!ExInitializeResourceLite
    fffff800`5d5fa148  fffff800`560766d0 nt!ExInitializeResourceLite

fffff8005d710000 (rdbss)              [Show Matches !imports]
    rdbss imports ntoskrnl.exe!ExInitializeResourceLite
    fffff800`5d7454c0  fffff800`560766d0 nt!ExInitializeResourceLite

fffff8005d790000 (csc)                [Show Matches !imports]
    csc imports ntoskrnl.exe!ExInitializeResourceLite
    fffff800`5d7cf1b0  fffff800`560766d0 nt!ExInitializeResourceLite

fffff8005d8b0000 (netbt)              [Show Matches !imports]
    netbt imports ntoskrnl.exe!ExInitializeResourceLite
    fffff800`5d8e7438  fffff800`560766d0 nt!ExInitializeResourceLite

fffff8005d930000 (afd)                [Show Matches !imports]
    afd imports ntoskrnl.exe!ExInitializeResourceLite
    fffff800`5d98f3e0  fffff800`560766d0 nt!ExInitializeResourceLite

fffff8005da20000 (ahcache)            [Show Matches !imports]
    ahcache imports ntoskrnl.exe!ExInitializeResourceLite
    fffff800`5da40050  fffff800`560766d0 nt!ExInitializeResourceLite

fffff8005ddb0000 (storvsp)            [Show Matches !imports]
    storvsp imports ntoskrnl.exe!ExInitializeResourceLite
    fffff800`5ddcc520  fffff800`560766d0 nt!ExInitializeResourceLite

fffff8005de50000 (ks)                 [Show Matches !imports]
    ks imports ntoskrnl.exe!ExInitializeResourceLite
    fffff800`5de7d2b0  fffff800`560766d0 nt!ExInitializeResourceLite

fffff8005e060000 (dxgmms2)            [Show Matches !imports]
    dxgmms2 imports ntoskrnl.exe!ExInitializeResourceLite
    fffff800`5e0e1388  fffff800`560766d0 nt!ExInitializeResourceLite

fffff8005e180000 (rdpdr)              [Show Matches !imports]
    rdpdr imports ntoskrnl.exe!ExInitializeResourceLite
    fffff800`5e18e3f8  fffff800`560766d0 nt!ExInitializeResourceLite

fffff8005e1c0000 (tsusbhub)           [Show Matches !imports]
    tsusbhub imports ntoskrnl.exe!ExInitializeResourceLite
    fffff800`5e1de100  fffff800`560766d0 nt!ExInitializeResourceLite

fffff8005e250000 (wcifs)              [Show Matches !imports]
    wcifs imports ntoskrnl.exe!ExInitializeResourceLite
    fffff800`5e261668  fffff800`560766d0 nt!ExInitializeResourceLite

fffff8005e290000 (bowser)             [Show Matches !imports]
    bowser imports ntoskrnl.exe!ExInitializeResourceLite
    fffff800`5e29c1b8  fffff800`560766d0 nt!ExInitializeResourceLite

fffff8005e330000 (cldflt)             [Show Matches !imports]
    cldflt imports ntoskrnl.exe!ExInitializeResourceLite
    fffff800`5e35c9a0  fffff800`560766d0 nt!ExInitializeResourceLite

fffff8005e3c0000 (mrxsmb)             [Show Matches !imports]
    mrxsmb imports ntoskrnl.exe!ExInitializeResourceLite
    fffff800`5e421348  fffff800`560766d0 nt!ExInitializeResourceLite

fffff8005e490000 (mrxsmb20)           [Show Matches !imports]
    mrxsmb20 imports ntoskrnl.exe!ExInitializeResourceLite
    fffff800`5e4cd470  fffff800`560766d0 nt!ExInitializeResourceLite

fffff8005e4e0000 (bindflt)            [Show Matches !imports]
    bindflt imports ntoskrnl.exe!ExInitializeResourceLite
    fffff800`5e4eb470  fffff800`560766d0 nt!ExInitializeResourceLite

fffff8005e980000 (fastfat)            [Show Matches !imports]
    fastfat imports ntoskrnl.exe!ExInitializeResourceLite
    fffff800`5e9a15f8  fffff800`560766d0 nt!ExInitializeResourceLite
```

# What is the difference between Exclusive Mode and Shared Mode?

When an ERESOURCE is acquired in **Exclusive Mode**, only one thread can hold the resource at a given time. No other thread can acquire the resource, either in **Exclusive Mode** or Shared Mode, until the thread that holds the resource releases it. This is suitable for read-write operations where you want to ensure that only one thread modifies the resource at a time.

On the other hand, when an ERESOURCE is acquired in **Shared Mode**, multiple threads can hold the resource simultaneously, but all must acquire it in **Shared Mode**. A thread that wants to acquire the resource in Exclusive Mode has to wait until all threads that have acquired the resource in **Shared Mode** release it. Shared Mode is commonly used for **read-only** operations where multiple threads need to read from a shared resource but no modifications are being made.

# Code Sample - Race Condition

In order to demonstrate this demo. Please follow the following steps:

- Create the following 2 folders:
  1. **C:\Temp**
  2. **C:\Temp2**

A race condition is a situation in programming where two or more threads access and manipulate shared data simultaneously, leading to unpredictable and incorrect results.

The root cause of the race condition in this code is the lack of synchronization primitive between the **`FileCreationThread`** and the **`FileMoveThread`**. Both threads operate on files located in the **`C:\Temp`** directory, but they do so concurrently. This unsynchronized access means that **`FileMoveThread`** might attempt to move a file that **`FileCreationThread`** is still writing to, or hasn't even created yet. While, **`FileCreationThread`** could potentially overwrite a file just as **`FileMoveThread`** is moving it. The result could be incomplete, missing, or corrupted files.

```c
#include <ntifs.h>
#include <ntstrsafe.h>

NTSTATUS FileCreationThread(PVOID Context) {
    UNICODE_STRING uniFilePath;
    WCHAR filePathBuffer[150];
    char dataToWrite[] = "Hello, World!\r\n";  
    IO_STATUS_BLOCK ioStatusBlock;

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

    return STATUS_SUCCESS;
}

NTSTATUS FileMoveThread(PVOID Context) {
    IO_STATUS_BLOCK ioStatusBlock;
    WCHAR oldFilePathBuffer[150], newFilePathBuffer[150];

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

    return STATUS_SUCCESS;
}

VOID UnloadDriver(IN PDRIVER_OBJECT DriverObject) {
    IoDeleteDevice(DriverObject->DeviceObject);
}

NTSTATUS DriverEntry(PDRIVER_OBJECT DriverObject, PUNICODE_STRING RegistryPath) {
    DriverObject->DriverUnload = UnloadDriver;

    PDEVICE_OBJECT DeviceObject = NULL;
    NTSTATUS status = IoCreateDevice(DriverObject, 0, NULL, FILE_DEVICE_UNKNOWN, FILE_DEVICE_SECURE_OPEN, FALSE, &DeviceObject);

    if (!NT_SUCCESS(status)) {
        return status;
    }

    HANDLE threadHandle1, threadHandle2;

    status = PsCreateSystemThread(&threadHandle1, (ACCESS_MASK)0, NULL, (PHANDLE)0, NULL, FileCreationThread, NULL);
    if (NT_SUCCESS(status)) {
        ZwClose(threadHandle1);
    }

    status = PsCreateSystemThread(&threadHandle2, (ACCESS_MASK)0, NULL, (PHANDLE)0, NULL, FileMoveThread, NULL);
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

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/541524ca-0b1b-4797-8b77-c1cc78b37035)


Start the kernel driver:

```
sc.exe start TestDrv
```

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/d979280f-840d-4d62-92c1-6fd0b3c7baa7)


We can see that 1000 files were created in the **C:\Temp** folder.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/2aa35c63-6e61-47cb-8edc-048aad73ffdd)


However, the files have not been renamed and moved to the **C:\Temp2** folder.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/da7837de-6557-4c2e-b75d-da212c789e6b)


Now delete all the files from the **C:\Temp** folder.

# Code Sample - ERESOURCE

In order to demonstrate this demo. Please follow the following steps:

- Create the following 2 folders:
  1. **C:\Temp**
  2. **C:\Temp2**

This improved code eliminates the race condition by using an ERESOURCE object for synchronization. It ensures that the file creation and file moving operations are not happening simultaneously across different threads.

The **`FileCreationThread`** function acquires the ERESOURCE in exclusive mode before creating and writing to files and releases it afterwards. Likewise, the **`FileMoveThread`** function acquires the same ERESOURCE in exclusive mode before it moves files and releases it when it's finished. This ensures that only one of these operations can occur at any given time.

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
    char dataToWrite[] = "Hello, World!\r\n";  // Data to write, with a newline for each "Hello, World!"
    IO_STATUS_BLOCK ioStatusBlock;

    // Acquire the ERESOURCE in exclusive mode to ensure exclusive access to this section.
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

    // Release the ERESOURCE, allowing other threads to acquire it.
    ExReleaseResourceLite(&SyncRes->FileResource);

    return STATUS_SUCCESS;
}

NTSTATUS FileMoveThread(PVOID Context) {
    PSYNC_RESOURCES SyncRes = (PSYNC_RESOURCES)Context;
    IO_STATUS_BLOCK ioStatusBlock;
    WCHAR oldFilePathBuffer[150], newFilePathBuffer[150];  

    // Acquire the ERESOURCE in exclusive mode to ensure exclusive access to this section.
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

    // Release the ERESOURCE, allowing other threads to acquire it.
    ExReleaseResourceLite(&SyncRes->FileResource);

    return STATUS_SUCCESS;
}

VOID UnloadDriver(IN PDRIVER_OBJECT DriverObject) {
    PSYNC_RESOURCES SyncRes = (PSYNC_RESOURCES)DriverObject->DeviceObject->DeviceExtension;

    // Clean up the ERESOURCE.
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

    // Initialize the ERESOURCE.
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

Start loading this kernel driver:

```
sc.exe create TestDrv type= kernel binPath=C:\Users\User\source\repos\TestDriver\x64\Release\TestDriver.sys
```

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/39a01e99-efa0-4a7d-8295-c2649185f34b)


Start the kernel driver:

```
sc.exe start TestDrv
```

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/d6c26034-9bb2-49cb-b3a0-90fd4fed159f)


We can see that 1000 files were created in the **C:\Temp** folder.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/a031a3bf-3e70-4759-848e-a9270b295ebc)


However, we can also see that those 1000 files have now been renamed and moved to **C:\Temp2** folder.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/bf7903f8-d178-422f-9c3a-71d7daabb4c1)


# Code Sample - ERESOURCE Shared Mode

The **`FileReaderThread`** is designed to read from files and write some information about the files to a "README.txt" file. It doesn't modify the files it reads which makes this makes it a good candidate for acquiring the ERESOURCE in **Shared Mode**.

By using **Shared Mode** for **`FileReaderThread`**, we're optimizing for a scenario where reading can happen alongside other reading operations, but will make way when a write operation needs to happen. This results in a more efficient result.

```c
#include <ntifs.h>
#include <ntstrsafe.h>

typedef struct _SYNC_RESOURCES {
    ERESOURCE FileResource;
} SYNC_RESOURCES, * PSYNC_RESOURCES;

NTSTATUS FileCreationThread(PVOID Context) {
    PSYNC_RESOURCES SyncRes = (PSYNC_RESOURCES)Context;
    UNICODE_STRING uniFilePath;
    WCHAR filePathBuffer[150];
    char dataToWrite[] = "Hello, World!\r\n";
    IO_STATUS_BLOCK ioStatusBlock;

    if (!ExAcquireResourceExclusiveLite(&SyncRes->FileResource, TRUE)) {
        return STATUS_UNSUCCESSFUL;
    }

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

    if (!ExAcquireResourceExclusiveLite(&SyncRes->FileResource, TRUE)) {
        return STATUS_UNSUCCESSFUL;
    }

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

    ExReleaseResourceLite(&SyncRes->FileResource);

    return STATUS_SUCCESS;
}

// Thread function to read from files and write to a README file.
NTSTATUS FileReaderThread(PVOID Context) {
    PSYNC_RESOURCES SyncRes = (PSYNC_RESOURCES)Context;  // Access our sync resources.
    IO_STATUS_BLOCK ioStatusBlock;
    WCHAR filePathBuffer[150];
    WCHAR readMeBuffer[150];
    UNICODE_STRING uniReadMePath;
    HANDLE hReadMeFile;

    // Acquire the ERESOURCE in shared mode, allowing other threads to also acquire it in shared mode.
    if (!ExAcquireResourceSharedLite(&SyncRes->FileResource, TRUE)) {
        return STATUS_UNSUCCESSFUL;
    }

    // Initialize the README file path.
    RtlStringCchPrintfW(readMeBuffer, sizeof(readMeBuffer) / sizeof(WCHAR), L"\\DosDevices\\C:\\Temp2\\README.txt");
    RtlInitUnicodeString(&uniReadMePath, readMeBuffer);

    // Prepare object attributes for the README file.
    OBJECT_ATTRIBUTES objAttributes;
    InitializeObjectAttributes(&objAttributes, &uniReadMePath, OBJ_CASE_INSENSITIVE | OBJ_KERNEL_HANDLE, NULL, NULL);

    // Create or overwrite the README file.
    NTSTATUS status = ZwCreateFile(&hReadMeFile, GENERIC_WRITE, &objAttributes, &ioStatusBlock, NULL, FILE_ATTRIBUTE_NORMAL, 0, FILE_OVERWRITE_IF, FILE_SYNCHRONOUS_IO_NONALERT, NULL, 0);

    // Loop through and write the names of the files we are "reading" to the README file.
    for (int i = 0; i < 1000; i++) {
        // Generate the file path for the file we are "reading".
        RtlStringCchPrintfW(filePathBuffer, sizeof(filePathBuffer) / sizeof(WCHAR), L"\\DosDevices\\C:\\Temp2\\NewFile_%d.txt", i);

        // If the README file was successfully created, write to it.
        if (NT_SUCCESS(status)) {
            char outputBuffer[200];
            sprintf(outputBuffer, "Read File: NewFile_%d.txt\r\n", i);
            ZwWriteFile(hReadMeFile, NULL, NULL, NULL, &ioStatusBlock, outputBuffer, strlen(outputBuffer), NULL, NULL);
        }
    }

    // Close the README file if it was successfully created.
    if (NT_SUCCESS(status)) {
        ZwClose(hReadMeFile);
    }

    // Release the ERESOURCE, allowing other threads to acquire it.
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

    HANDLE threadHandle1, threadHandle2, threadHandle3;

    status = PsCreateSystemThread(&threadHandle1, (ACCESS_MASK)0, NULL, (PHANDLE)0, NULL, FileCreationThread, SyncRes);
    if (NT_SUCCESS(status)) {
        ZwClose(threadHandle1);
    }

    status = PsCreateSystemThread(&threadHandle2, (ACCESS_MASK)0, NULL, (PHANDLE)0, NULL, FileMoveThread, SyncRes);
    if (NT_SUCCESS(status)) {
        ZwClose(threadHandle2);
    }

    status = PsCreateSystemThread(&threadHandle3, (ACCESS_MASK)0, NULL, (PHANDLE)0, NULL, FileReaderThread, SyncRes);
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

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/af6e0f41-93ee-4f1b-88c3-25959bb5910b)


Start the kernel driver:

```
sc.exe start TestDrv
```

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/b7116f2b-64e1-4451-b46c-ea2ba95b8644)


The code is functioning as expected. Now, we can find a "README.txt" file in the **C:\Temp2** directory, which contains the names of all the files that our thread has read.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/be73f1f7-9216-4e7b-b405-13f828d99c0b)

