# Description

Pushlocks, first seen in Windows Server 2003, are a type of lock in the kernel that allows many readers but only one writer at a time. They are quite similar to what the kernel's EResource does. You can find a list of the benefits and drawbacks of Pushlocks compared to EResources in the remarks of the ExInitializePushLock function's documentation. See: https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/wdm/nf-wdm-exinitializepushlock

Here are the characteristics of Pushlocks:

- Pushlocks can be obtained for either shared use or exclusive access.
- They are designed not to allow recursive acquisition.
- Pushlocks don't keep track of who currently owns them.
- To acquire a Pushlock, the caller must first disable user and normal kernel APCs. This is done by calling **KeEnterCriticalRegion**.

Here are the common Pushlock APIs:

| Function                              | Description                                                                                                                                    |
|---------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------|
| `ExInitializePushLock`                | Initializes a Pushlock. Prepares the Pushlock for use by setting it to an unlocked and unowned state.                                          |
| `ExAcquirePushLockExclusive`          | Acquires a Pushlock for exclusive access. Only one thread can hold the lock exclusively, blocking other threads from access.                   |
| `ExReleasePushLockExclusive`          | Releases a Pushlock from exclusive access. Allows other threads to acquire the lock.                                                          |
| `ExAcquirePushLockShared`             | Acquires a Pushlock for shared access. Multiple threads can hold the lock in shared mode simultaneously.                                       |
| `ExReleasePushLockShared`             | Releases a Pushlock from shared access. Decrements the count of threads holding the lock in shared mode.                                       |
| `ExTryConvertPushLockSharedToExclusive` | Attempts to convert a Pushlock from shared to exclusive access, without releasing and re-acquiring the lock.                                   |

The choice between Shared and Exclusive access depends on the nature of the operations we intend to perform on the shared resource.

- **Shared Access:** Used for read-only operations where concurrent access by multiple threads is safe and desirable.
- **Exclusive Access:** Used for write or modify operations where concurrent access could lead to data corruption or inconsistencies.

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

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/c8dd026f-8c61-4475-a1ca-7c80241709b4)


Start the kernel driver:

```
sc.exe start TestDrv
```

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/b2c8ef6f-68c9-4552-ae84-7492261a3f01)


We can see that 1000 files were created in the **C:\Temp** folder.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/ae9d7b73-55cd-423b-add7-cb4a32135201)


However, the files have not been renamed and moved to the **C:\Temp2** folder.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/99774c74-9266-4d7f-96d7-eb5967939afe)


Now delete all the files from the **C:\Temp** folder.

# Code Sample - Pushlock

In order to demonstrate this demo. Please follow the following steps:

- Create the following 2 folders:
  1. **C:\Temp**
  2. **C:\Temp2**

This code utilizes a Pushlock for effective synchronization in file operations for avoiding access conflicts when multiple tasks operate on the same files concurrently. In the **`FileCreationThread`** function, before initiating file creation and writing, the Pushlock is acquired to ensure exclusive access to the files. Additionally, this function calls **`KeEnterCriticalRegion`** to disable normal kernel APCs before acquiring the Pushlock.

When we hold a lock in our code, it's important to disable normal kernel APCs to avoid deadlocks. APCs can interrupt what a thread is doing and might try to acquire the same lock that the thread already holds. If an APC tries to get a lock that's already taken by the thread it interrupted, the thread can't continue — it's stuck waiting for itself to release the lock. 

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

    return STATUS_SUCCESS;
}
```

Start loading this kernel driver:

```
sc.exe create TestDrv type= kernel binPath=C:\Users\User\source\repos\TestDriver\x64\Release\TestDriver.sys
```

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/9d244018-cd5a-4c8d-b54c-e3c06822836d)


Start the kernel driver:

```
sc.exe start TestDrv
```

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/a28d2763-6791-47d1-b912-41c6abf331bd)

We can see that 1000 files were created in the **C:\Temp** folder.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/01d861e2-7ca9-4f2b-b637-49af7af56d68)


However, we can also see that those 1000 files have now been renamed and moved to **C:\Temp2** folder.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/c3cde17d-56a5-4fc9-a387-376fb0fdda43)

