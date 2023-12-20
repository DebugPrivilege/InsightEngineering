# Description

Event objects are key to controlling how multiple threads run and interact. These objects work like signals that let threads stop and start again, depending on specific situations or events. There are two main kinds of event objects, and each one has its own way of working and specific uses.

- **Manual-Reset Event:** This type of event remains in a signaled state until it is manually reset to a non-signaled state. Ideal for situations where multiple threads need to be released simultaneously or where the event needs to stay active until explicitly reset. Think of it like a manually operated traffic signal. It stays green (signaled) allowing continuous flow (threads to proceed) until someone manually changes it to red (non-signaled).

- **Auto-Reset Event:** This type of event is designed to automatically revert to a non-signaled state as soon as it allows a single waiting thread to proceed. In situations where no threads are currently waiting, the event maintains its signaled state. Once a thread does come to wait on it, the event will then release this thread and immediately reset itself to non-signaled. Similar to a self-regulating traffic signal, which turns red (non-signaled) automatically after allowing a single car (thread) to pass.

Here are some of the common Event Signaling APIs in the Windows Kernel:

| Function                          | Description |
| --------------------------------- | ----------- |
| `KeInitializeEvent()`             | Initializes a `KEVENT` structure. Sets up the event object as either a Notification Event (Manual Reset) or a Synchronization Event (Auto Reset), and initially non-signaled. |
| `KeSetEvent()`                    | Signals the event object. Releases one or more waiting threads depending on the event type. Usable at any IRQL for Synchronization events and at IRQL <= DISPATCH_LEVEL for Notification events. |
| `KeResetEvent()`                  | Resets a Notification (Manual Reset) event to non-signaled. Not applicable to Synchronization (Auto Reset) events. Usable at any IRQL. |
| `KeClearEvent()`                  | Clears an event object, setting it to non-signaled. Applicable to both event types. Usable at any IRQL. |
| `KeWaitForSingleObject()`         | Waits for an object, such as an event, to become signaled. Blocks the calling thread until the condition is met or timeout occurs. Usable at IRQL PASSIVE_LEVEL. |
| `KePulseEvent()`                  | Temporarily signals an event and then immediately resets it, releasing any waiting threads at that moment. Use is generally discouraged in favor of `KeSetEvent()`. Usable at IRQL <= DISPATCH_LEVEL. |

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

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/d4289bca-c496-4b9f-b8c0-3b14d1ffa5df)


Start the kernel driver:

```
sc.exe start TestDrv
```

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/3eca3179-8b39-4cb0-9e2d-a660af776404)


We can see that 1000 files were created in the **C:\Temp** folder.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/99f5f4ee-a620-429a-8d97-9e127949c649)


However, the files have not been renamed and moved to the **C:\Temp2** folder.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/609acc19-66db-44c9-93eb-3f92bb2b4c91)


Now delete all the files from the **C:\Temp** folder.

# Code Sample - Event Signaling Synchronization

In order to demonstrate this demo. Please follow the following steps:

- Create the following 2 folders:
  1. **C:\Temp**
  2. **C:\Temp2**
     

The updated code uses a **`KEVENT`** object for synchronization, reducing race conditions during file operations. In the **`FileCreationThread`** function, after it finishes creating and writing to files, it signals the **`KEVENT`**. This signal indicates that the file creation part is done.

The **`FileMoveThread`** function waits for this **`KEVENT`** to be signaled before it starts moving files. This way of synchronizing makes sure that the moving of files only starts after all the files have been created and written to. Once the file movement task is finished, there's no further need for synchronization in this function, as the main point is to ensure file creation is complete before moving files.

```c
#include <ntifs.h>
#include <ntstrsafe.h>

// Define a structure to hold our synchronization resources.
typedef struct _SYNC_RESOURCES {
    KEVENT FileCreationEvent;  // Synchronization event for signaling completion of file creation.
} SYNC_RESOURCES, * PSYNC_RESOURCES;

NTSTATUS FileCreationThread(PVOID Context) {
    PSYNC_RESOURCES SyncRes = (PSYNC_RESOURCES)Context;
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

    // Signal completion of file creation
    KeSetEvent(&SyncRes->FileCreationEvent, IO_NO_INCREMENT, FALSE);

    return STATUS_SUCCESS;
}

NTSTATUS FileMoveThread(PVOID Context) {
    PSYNC_RESOURCES SyncRes = (PSYNC_RESOURCES)Context;
    IO_STATUS_BLOCK ioStatusBlock;
    WCHAR oldFilePathBuffer[150], newFilePathBuffer[150];

    // Wait for the file creation to complete
    KeWaitForSingleObject(&SyncRes->FileCreationEvent, Executive, KernelMode, FALSE, NULL);

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
    NTSTATUS status = IoCreateDevice(DriverObject, sizeof(SYNC_RESOURCES), NULL, FILE_DEVICE_UNKNOWN, FILE_DEVICE_SECURE_OPEN, FALSE, &DeviceObject);

    if (!NT_SUCCESS(status)) {
        return status;
    }

    PSYNC_RESOURCES SyncRes = (PSYNC_RESOURCES)DeviceObject->DeviceExtension;

    // Initialize the synchronization event
    KeInitializeEvent(&SyncRes->FileCreationEvent, SynchronizationEvent, FALSE);

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

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/b504d240-f1d2-4e7b-9e29-9de11a437285)


Start the kernel driver:

```
sc.exe start TestDrv
```

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/eb34a484-9555-4e72-891e-28fba29fc81d)


We can see that 1000 files were created in the **C:\Temp** folder.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/79e1ac83-be40-4b38-a6a9-5768e878530f)


However, we can also see that those 1000 files have now been renamed and moved to **C:\Temp2** folder.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/2030dd0e-ae4a-44e3-ba75-f6daa0a49446)


# Code Sample - Event Signaling Notification Event

In order to demonstrate this demo. Please follow the following steps:

- Create the following 2 folders:
  1. **C:\Temp**
  2. **C:\Temp2**

In our earlier code with the auto-reset event, when we completed the file creation task in the **`FileCreationThread`** and signaled the event, the **`FileMoveThread`** would begin its operation as soon as it detected this signal. Then, the event would automatically reset itself. This automatic reset is like a one-time notification; as soon as one thread responds to it, the notification turns off.

Now, in our current code with the manual-reset event, the process is slightly different. After we finish creating files in the **`FileCreationThread`** and signal the event, the **`FileMoveThread`** starts once it recognizes this signal. However, in this scenario, the event does not reset itself automatically. It remains in the signaled state until we manually reset it. This is akin to an ongoing alert that stays active until we manually turn it off. We have more control over when the event resets, but it also means we need to actively manage it and remember to reset it once our task is complete.

  ```c
#include <ntifs.h>
#include <ntstrsafe.h>

// Define a structure to hold our synchronization resources.
typedef struct _SYNC_RESOURCES {
    KEVENT FileOperationEvent;  // Notification event for synchronizing file operations.
} SYNC_RESOURCES, *PSYNC_RESOURCES;

NTSTATUS FileCreationThread(PVOID Context) {
    PSYNC_RESOURCES SyncRes = (PSYNC_RESOURCES)Context;
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

    // Signal completion of file creation and allow the file move operation to proceed
    KeSetEvent(&SyncRes->FileOperationEvent, IO_NO_INCREMENT, FALSE);

    return STATUS_SUCCESS;
}

NTSTATUS FileMoveThread(PVOID Context) {
    PSYNC_RESOURCES SyncRes = (PSYNC_RESOURCES)Context;
    IO_STATUS_BLOCK ioStatusBlock;
    WCHAR oldFilePathBuffer[150], newFilePathBuffer[150];

    // Wait for the file creation to complete
    KeWaitForSingleObject(&SyncRes->FileOperationEvent, Executive, KernelMode, FALSE, NULL);

    // Manually reset the event before starting the file move operation
    KeResetEvent(&SyncRes->FileOperationEvent);

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
    NTSTATUS status = IoCreateDevice(DriverObject, sizeof(SYNC_RESOURCES), NULL, FILE_DEVICE_UNKNOWN, FILE_DEVICE_SECURE_OPEN, FALSE, &DeviceObject);

    if (!NT_SUCCESS(status)) {
        return status;
    }

    PSYNC_RESOURCES SyncRes = (PSYNC_RESOURCES)DeviceObject->DeviceExtension;

    // Initialize the notification event (manual-reset event)
    KeInitializeEvent(&SyncRes->FileOperationEvent, NotificationEvent, FALSE);

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

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/b504d240-f1d2-4e7b-9e29-9de11a437285)


Start the kernel driver:

```
sc.exe start TestDrv
```

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/eb34a484-9555-4e72-891e-28fba29fc81d)


We can see that 1000 files were created in the **C:\Temp** folder.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/79e1ac83-be40-4b38-a6a9-5768e878530f)


However, we can also see that those 1000 files have now been renamed and moved to **C:\Temp2** folder.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/2030dd0e-ae4a-44e3-ba75-f6daa0a49446)

