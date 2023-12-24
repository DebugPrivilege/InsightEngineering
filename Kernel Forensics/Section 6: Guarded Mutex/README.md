# Description

A Guarded Mutex in Windows is a synchronization mechanism. It ensures that a specific code path is accessed by only one thread at a time, providing mutual exclusion with better performance compared to fast mutexes. It works by entering a guarded region, which is more efficient than the operation of acquiring a fast mutex. Guarded mutexes, which are available starting with Windows Server 2003, perform the same function as fast mutexes but with higher performance. Starting with Windows 8, guarded mutexes and fast mutexes are implemented identically.

Here are the common Guarded Mutex APIs:

| Function                          | Description |
| --------------------------------- | ----------- |
| `KeInitializeGuardedMutex()`      | Initializes a `KGUARDED_MUTEX` structure for use. Prepares the Guarded Mutex by setting it to the non-signaled, or free, state. |
| `KeAcquireGuardedMutex()`         | Acquires a `KGUARDED_MUTEX`. If the mutex is not immediately available, blocks the calling thread until the mutex becomes available. Used at IRQL <= APC_LEVEL. |
| `KeReleaseGuardedMutex()`         | Releases an acquired `KGUARDED_MUTEX`. Should be called at the same IRQL at which `KeAcquireGuardedMutex()` was called, allowing other threads to acquire the mutex. |
| `KeTryToAcquireGuardedMutex()`    | Tries to acquire a `KGUARDED_MUTEX` without blocking. Returns immediately with a success or failure status, suitable for scenarios where waiting is not desirable. Used at IRQL <= APC_LEVEL. |

Here are the characteristics of Guarded Mutexes:

- Ensures exclusive access is granted sequentially, with only one access at a time.
- Recursive acquisition attempts by the same thread are not supported and will lead to a deadlock scenario.
- Suitable for use in code operating at an Interrupt Request Level (IRQL) that is less than or equal to APC_LEVEL, applicable to both fast and guarded mutexes.

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

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/9f623c09-ac83-4eed-bc8c-df93617e07d0)


Start the kernel driver:

```
sc.exe start TestDrv
```

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/bc67020a-394c-4a07-bc8d-63ee128fa3d9)


We can see that 1000 files were created in the **C:\Temp** folder.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/a339f041-960c-44ce-8119-c9aae41efef3)

However, the files have not been renamed and moved to the **C:\Temp2** folder.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/be81cfcd-3202-4437-94fd-dd4b46ea3422)


Now delete all the files from the **C:\Temp** folder.

# Code Sample - Guarded Mutex

In order to demonstrate this demo. Please follow the following steps:

- Create the following 2 folders:
  1. **C:\Temp**
  2. **C:\Temp2**

The updated code uses a **`KGUARDED_MUTEX`** object for synchronization, minimizing race conditions during file operations. In the **`FileCreationThread`** function, the **`KGUARDED_MUTEX`** is acquired before initiating file creation and writing. This acquisition ensures that only one thread can access these shared resources at a time.

The **`FileMoveThread`** function acquires the same **`KGUARDED_MUTEX`** before starting file movements. This synchronization guarantees that while files are being moved, no other thread can perform file creation or movement concurrently. Upon completing its operations, each function releases the **`KGUARDED_MUTEX`** allowing other threads to safely access shared resources.

```c
#include <ntifs.h>
#include <ntstrsafe.h>

// Define a structure to hold our synchronization resources.
typedef struct _SYNC_RESOURCES {
    KGUARDED_MUTEX FileMutex;  // KGUARDED_MUTEX object for synchronized access to the files.
} SYNC_RESOURCES, * PSYNC_RESOURCES;

NTSTATUS FileCreationThread(PVOID Context) {
    PSYNC_RESOURCES SyncRes = (PSYNC_RESOURCES)Context;
    UNICODE_STRING uniFilePath;
    WCHAR filePathBuffer[150];
    char dataToWrite[] = "Hello, World!\r\n";
    IO_STATUS_BLOCK ioStatusBlock;

    // Acquire the Guarded Mutex
    KeAcquireGuardedMutex(&SyncRes->FileMutex);

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

    // Release the Guarded Mutex
    KeReleaseGuardedMutex(&SyncRes->FileMutex);

    return STATUS_SUCCESS;
}

NTSTATUS FileMoveThread(PVOID Context) {
    PSYNC_RESOURCES SyncRes = (PSYNC_RESOURCES)Context;
    IO_STATUS_BLOCK ioStatusBlock;
    WCHAR oldFilePathBuffer[150], newFilePathBuffer[150];

    // Acquire the Guarded Mutex
    KeAcquireGuardedMutex(&SyncRes->FileMutex);

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

    // Release the Guarded Mutex
    KeReleaseGuardedMutex(&SyncRes->FileMutex);

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

    // Initialize the Guarded Mutex
    KeInitializeGuardedMutex(&SyncRes->FileMutex);

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

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/7010eaae-70a9-432d-8c15-856ca52706c7)


Start the kernel driver:

```
sc.exe start TestDrv
```

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/cff3c7fe-a270-4c5c-bca0-249566487495)


We can see that 1000 files were created in the **C:\Temp** folder.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/8dc1cf79-48bf-4c3e-9491-ca5573687b98)


However, we can also see that those 1000 files have now been renamed and moved to **C:\Temp2** folder.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/9698c587-ae05-4c58-8cca-7c215d5971be)


# Code Sample - Guarded Mutex IRQL Handeling

In Windows 8 and later versions, Guarded Mutexes are implemented identically to Fast Mutexes. In the newer versions, the IRQL handling for both Guarded Mutexes and Fast Mutexes are the same. Compile the following code in **Debug** mode:

```c
#include <ntifs.h>
#include <ntstrsafe.h>

// Structure to hold synchronization resources.
typedef struct _SYNC_RESOURCES {
    KGUARDED_MUTEX FileMutex;  // KGUARDED_MUTEX object for synchronized access to files.
} SYNC_RESOURCES, * PSYNC_RESOURCES;

// Function to log the current IRQL level to DebugView.
VOID LogCurrentIrql(const char* action, const char* functionName, KIRQL irql) {
    const char* irqlName = "UNKNOWN_LEVEL";
    switch (irql) {
    case PASSIVE_LEVEL:
        irqlName = "PASSIVE_LEVEL";
        break;
    case APC_LEVEL:
        irqlName = "APC_LEVEL";
        break;
    case DISPATCH_LEVEL:
        irqlName = "DISPATCH_LEVEL";
        break;
    }
    DbgPrint("%s (%s) IRQL: %s\n", action, functionName, irqlName);
}


NTSTATUS FileCreationThread(PVOID Context) {
    PSYNC_RESOURCES SyncRes = (PSYNC_RESOURCES)Context;
    UNICODE_STRING uniFilePath;
    WCHAR filePathBuffer[150];
    char dataToWrite[] = "Hello, World!\r\n";
    IO_STATUS_BLOCK ioStatusBlock;
    KIRQL oldIrql;

    KeAcquireGuardedMutex(&SyncRes->FileMutex); // Acquire the guarded mutex.
    LogCurrentIrql("Acquiring Guarded Mutex", __FUNCTION__, KeGetCurrentIrql());

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

    KeReleaseGuardedMutex(&SyncRes->FileMutex); // Release the guarded mutex.
    LogCurrentIrql("Releasing Guarded Mutex", __FUNCTION__, KeGetCurrentIrql());

    return STATUS_SUCCESS;
}

// Thread function for file movement.
NTSTATUS FileMoveThread(PVOID Context) {
    PSYNC_RESOURCES SyncRes = (PSYNC_RESOURCES)Context;
    IO_STATUS_BLOCK ioStatusBlock;
    WCHAR oldFilePathBuffer[150], newFilePathBuffer[150];

    KeAcquireGuardedMutex(&SyncRes->FileMutex); // Acquire the guarded mutex.
    LogCurrentIrql("Acquiring Guarded Mutex", __FUNCTION__, KeGetCurrentIrql());

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

    KeReleaseGuardedMutex(&SyncRes->FileMutex); // Release the guarded mutex.
    LogCurrentIrql("Releasing Guarded Mutex", __FUNCTION__, KeGetCurrentIrql());

    return STATUS_SUCCESS;
}

// Driver unload routine.
VOID UnloadDriver(IN PDRIVER_OBJECT DriverObject) {
    IoDeleteDevice(DriverObject->DeviceObject); // Delete the device object.
}

NTSTATUS DriverEntry(PDRIVER_OBJECT DriverObject, PUNICODE_STRING RegistryPath) {
    DriverObject->DriverUnload = UnloadDriver;

    PDEVICE_OBJECT DeviceObject = NULL;
    NTSTATUS status = IoCreateDevice(DriverObject, sizeof(SYNC_RESOURCES), NULL, FILE_DEVICE_UNKNOWN, FILE_DEVICE_SECURE_OPEN, FALSE, &DeviceObject);

    if (!NT_SUCCESS(status)) {
        return status;
    }

    PSYNC_RESOURCES SyncRes = (PSYNC_RESOURCES)DeviceObject->DeviceExtension;
    KeInitializeGuardedMutex(&SyncRes->FileMutex); // Initialize the guarded mutex.

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
sc.exe create TestDrv type= kernel binPath=C:\Users\User\source\repos\TestDriver\x64\Debug\TestDriver.sy
```

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/22f448ac-e65c-4bb0-8dc1-36714137d075)



Start the kernel driver:

```
sc.exe start TestDrv
```

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/c37b5723-aab0-4fc4-a850-da3403968cdb)


When the Guarded Mutex is acquired in both the **`FileCreationThread`** and **`FileMoveThread`**, the IRQL is at **`APC_LEVEL`**. This suggests that the code executing these mutex acquisitions is running at **`APC_LEVEL`**. It's important to note that acquiring the Guarded Mutex itself doesn't raise the IRQL to **`APC_LEVEL`**. It requires that the IRQL be at or below **`APC_LEVEL`** when acquiring the mutex. After releasing the Guarded Mutex in both threads, the IRQL is at **`PASSIVE_LEVEL`**. 

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/a8300abd-3875-4c5b-9b47-230f02abd5fc)


# Guarded Mutex - Data Structure


Fast Mutexes and Guarded Mutexes are functionally equivalent in their implementation. For instance, invoking **`KeAcquireGuardedMutex`** effectively leads to the execution of **`KeAcquireFastMutex`**.

The **`!WrFastMutex`** displays detailed information about threads that are waiting for a Fast Mutex. There are two threads that are currently in a waiting state for a Fast Mutex.

```
0: kd> !WrFastMutex
Process PID Thread             Id CSwitches User Kernel State      Time Reason      Waiting On                      Wait Function
======= === ================ ==== ========= ==== ====== ======= ======= =========== =============================== =================
System    4 ffffd18ca1f07040 249c         1    0      0 Waiting 34s.593 WrFastMutex Owning Thread: ffffd18ca1f07040 TestDriver+0x112d
System    4 ffffd18ca2c43040 1c10         1    0      0 Waiting 34s.593 WrFastMutex Owning Thread: ffffd18ca1f07040 TestDriver+0x1298
Count: 2
```

Even though we're using a Guarded Mutex, the internal kernel call stack may refer to Fast Mutex mechanisms due to their shared implementation in the kernel. Note the wait reason is set to **`WrFastMutex`**.

```
0: kd> !mex.t ffffd18ca1f07040
Process                   Thread                       CID       UserTime KernelTime ContextSwitches Wait Reason    Time State
System (ffffd18c9d2ea040) ffffd18ca1f07040 (E|K|W|R|V) 4.249c           0          0               1 WrFastMutex 34s.593 Waiting

WaitBlockList:
    Object           Type                 Other Waiters Info
    ffffd18ca2fa9cb8 SynchronizationEvent             1 Fast Mutex Owner: ffffd18ca2fa9ca8

# Child-SP         Return           Call Site
0 ffffce04643bec00 fffff8075a26c9d5 nt!KiSwapContext+0x76
1 ffffce04643bed40 fffff8075a26ebb7 nt!KiSwapThread+0xab5
2 ffffce04643bee90 fffff8075a270ad6 nt!KiCommitThreadWait+0x137
3 ffffce04643bef40 fffff8075a22acb6 nt!KeWaitForSingleObject+0x256
4 ffffce04643bf2e0 fffff8075a22abbc nt!ExpAcquireFastMutexContended+0x7a
5 ffffce04643bf320 fffff807815f112d nt!KeAcquireGuardedMutex+0x6c --> Attempt to acquire a Guarded Mutex
6 ffffce04643bf350 fffff8075a307167 TestDriver+0x112d
7 ffffce04643bf570 fffff8075a41ba44 nt!PspSystemThreadStartup+0x57
8 ffffce04643bf5c0 0000000000000000 nt!KiStartSystemThread+0x34
```
