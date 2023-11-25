# What is a Fast Mutex?

A Fast Mutex is a lightweight synchronization primitive in the Windows kernel, used in kernel-mode code for quick and efficient protection of resources with low contention. It ensures that only one thread accesses a specific resource or code section at a time, minimizing race conditions. Low contention means that there is rarely a situation where multiple threads are trying to access the same resource at the same time.

Here are some characteristics of a Fast Mutex

- **Efficiency in Low Contention:** Optimized for scenarios where the likelihood of multiple threads competing for the same resource simultaneously is low, making it efficient and fast in these situations.
- **Non-Recursive:** A thread cannot acquire a Fast Mutex it already holds; attempting to do so will lead to a deadlock.

Here are the common Fast Mutex APIs:

| Function                      | Description |
| ----------------------------- | ----------- |
| `ExInitializeFastMutex()`     | Initializes a `FAST_MUTEX` object for use. Prepares the Fast Mutex by setting it to the non-signaled, or free, state. |
| `ExAcquireFastMutex()`        | Acquires a `FAST_MUTEX` object. Blocks the calling thread until the mutex becomes available if it's not immediately available. Intended for use at IRQL = PASSIVE_LEVEL. |
| `ExReleaseFastMutex()`        | Releases an acquired `FAST_MUTEX` object. Should be called at the same IRQL at which `ExAcquireFastMutex()` was called. Allows other threads to acquire the mutex. |
| `ExTryToAcquireFastMutex()`   | Tries to acquire a `FAST_MUTEX` object without blocking. Returns immediately with a success or failure status, suitable for scenarios where waiting is not desirable. Can be called at IRQL = PASSIVE_LEVEL. |

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

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/cd1c9743-f355-4ba8-ac06-4c0fc5de7d74)

Start the kernel driver:

```
sc.exe start TestDrv
```

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/7ff6b86b-acf4-4cea-b6e6-8f686287d8bc)

We can see that 1000 files were created in the **C:\Temp** folder.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/4d28f1b8-cc4c-4357-8d2d-c3b8caa0d52a)

However, the files have not been renamed and moved to the **C:\Temp2** folder.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/34b71733-9cee-46e6-be14-a0b2f1f2ed8b)

Now delete all the files from the **C:\Temp** folder.

# Code Sample - Fast Mutex

In order to demonstrate this demo. Please follow the following steps:

- Create the following 2 folders:
  1. **C:\Temp**
  2. **C:\Temp2**

The improved code eliminates potential race conditions by using a **FAST_MUTEX** object for synchronization. This ensures that file creation, writing, and moving operations do not occur at the same time across different threads.

In the **`FileCreationThread`** function, the **FAST_MUTEX** is acquired before entering the loop for creating and writing to files. This mutex acquisition guarantees exclusive access to the shared resources during these operations. Once the file creation and writing loop is complete, the function releases the **FAST_MUTEX**, allowing other threads to acquire it.

The **`FileMoveThread`** function acquires the same **FAST_MUTEX** before it starts the loop for moving files. This ensures that while one thread is moving files, no other thread can create or move files concurrently. After completing the file-moving operations, the **FAST_MUTEX** is released.

```c
#include <ntifs.h>
#include <ntstrsafe.h>

// Define a structure to hold our synchronization resources.
typedef struct _SYNC_RESOURCES {
    FAST_MUTEX FileMutex;  // FAST_MUTEX object for synchronized access to the files.
} SYNC_RESOURCES, * PSYNC_RESOURCES;

NTSTATUS FileCreationThread(PVOID Context) {
    PSYNC_RESOURCES SyncRes = (PSYNC_RESOURCES)Context;
    UNICODE_STRING uniFilePath;
    WCHAR filePathBuffer[150];
    char dataToWrite[] = "Hello, World!\r\n";
    IO_STATUS_BLOCK ioStatusBlock;

    // Acquire the FAST_MUTEX
    ExAcquireFastMutex(&SyncRes->FileMutex);

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

    // Release the FAST_MUTEX
    ExReleaseFastMutex(&SyncRes->FileMutex);

    return STATUS_SUCCESS;
}

NTSTATUS FileMoveThread(PVOID Context) {
    PSYNC_RESOURCES SyncRes = (PSYNC_RESOURCES)Context;
    IO_STATUS_BLOCK ioStatusBlock;
    WCHAR oldFilePathBuffer[150], newFilePathBuffer[150];

    // Acquire the FAST_MUTEX
    ExAcquireFastMutex(&SyncRes->FileMutex);

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

    // Release the FAST_MUTEX
    ExReleaseFastMutex(&SyncRes->FileMutex);

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

    // Initialize the FAST_MUTEX
    ExInitializeFastMutex(&SyncRes->FileMutex);

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

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/d827b0c5-cae6-46be-8fb5-0efad3868578)

Start the kernel driver:

```
sc.exe start TestDrv
```

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/a28d2763-6791-47d1-b912-41c6abf331bd)

We can see that 1000 files were created in the **C:\Temp** folder.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/613fbc97-b8fb-47d5-b682-176d64c30771)

However, we can also see that those 1000 files have now been renamed and moved to **C:\Temp2** folder.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/ae428ae9-120f-481b-ab8c-f62d7ad20972)

# Fast Mutex

This kernel memory dump can be download here: https://mega.nz/file/GgVGARhA#UiJjF7oH5wnp_dCvYiQ7R0r62pfS0jmyq8A3G-DymiQ

This structure is a part of the Windows kernel and is used to implement fast mutexes.

```
0: kd> !mex.ddt nt!_FAST_MUTEX

dt nt!_FAST_MUTEX () Recursive: [ -r1 -r2 -r ] Verbose Normal dt
==================================================================================
   +0x000 Count                : Int4B
   +0x008 Owner                : Ptr64 Void
   +0x010 Contention           : Uint4B
   +0x018 Event                : _KEVENT
   +0x030 OldIrql              : Uint4B
```

The **`!WrFastMutex`** command displays details of a thread that is waiting on a Fast Mutex.

```
0: kd> !WrFastMutex
Process PID Thread             Id CSwitches User Kernel State      Time Reason      Waiting On                      Wait Function
======= === ================ ==== ========= ==== ====== ======= ======= =========== =============================== ===========================
System    4 ffffe10b186e9040 249c         2    0      0 Waiting 36s.515 WrFastMutex Owning Thread: ffffe10b186bc5c0 nt!ExAcquireFastMutex+0x122
Count: 1
```

The **`Contention`** field specifically records the number of times a thread had to wait for the mutex because it was already owned by another thread. In other words, it's a count of how many times there was contention for this mutex. 

```
0: kd> dt nt!_FAST_MUTEX 0xffffe10b187ac3f0
   +0x000 Count            : 0n4
   +0x008 Owner            : 0xffffe10b`186bc5c0 Void
   +0x010 Contention       : 1
   +0x018 Event            : _KEVENT
   +0x030 OldIrql          : 0
```

Here is an example of a thread that has been waiting **36s.515** on a fast mutex owned by System thread **`ffffe10b186bc5c0`**

```
0: kd> !mex.t ffffe10b186e9040
Process                   Thread                       CID       UserTime KernelTime ContextSwitches Wait Reason    Time State
System (ffffe10b11aa7040) ffffe10b186e9040 (E|K|W|R|V) 4.249c           0          0               2 WrFastMutex 36s.515 Waiting

WaitBlockList:
    Object           Type                 Other Waiters Info
    ffffe10b187ac408 SynchronizationEvent             0 Fast Mutex Owner: ffffe10b187ac3f8

# Child-SP         Return           Call Site                            Source
0 ffffb781a82d4af0 fffff8041426d645 nt!KiSwapContext+0x76                
1 ffffb781a82d4c30 fffff8041426f827 nt!KiSwapThread+0xab5                
2 ffffb781a82d4d80 fffff80414271746 nt!KiCommitThreadWait+0x137          
3 ffffb781a82d4e30 fffff8041422acb6 nt!KeWaitForSingleObject+0x256       
4 ffffb781a82d51d0 fffff804142ed832 nt!ExpAcquireFastMutexContended+0x7a 
5 ffffb781a82d5210 fffff804163f12b8 nt!ExAcquireFastMutex+0x122          
6 ffffb781a82d5240 fffff80414307287 TestDriver!FileMoveThread+0x38       C:\Users\User\source\repos\TestDriver\TestDriver\Driver.c @ 54
7 ffffb781a82d5570 fffff8041441b8e4 nt!PspSystemThreadStartup+0x57       
8 ffffb781a82d55c0 0000000000000000 nt!KiStartSystemThread+0x34
```
