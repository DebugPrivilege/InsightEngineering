# What is a Deadlock?

Deadlocks are situations where two or more threads are unable to proceed with their execution because they are each waiting for the other to release a resource. Deadlocks typically involve a circular wait condition, where Thread A is waiting for a resource held by Thread B, and Thread B is waiting for a resource held by Thread A.

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

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/012612d7-b22c-4e2a-ab59-a3b7b670f8e8)

Start the kernel driver:

```
sc.exe start TestDrv
```

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/5d25f0cd-dc34-4ce6-9cdb-b05f58d7f2cf)

The kernel driver is designed to create 1,000 files in the **C:\Temp** directory. However, it won't be able to move these files to **C:\Temp2** or delete them. This is because the files never actually make it to the C:\Temp2 folder where they were supposed to be moved to. This is because of a deadlock situation.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/24bd496d-4193-4f0b-a785-0586208e29a5)

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

# WPA - Walk Through

1. Open the **.etl** file in **Windows Performance Analyzer**
2. Drag the **CPU Usage (Precise)** to the **Analysis** view

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/b6ea86e8-5cea-4668-aa7e-29ff951a8bc6)

3. Click on **System** and select "Filter to Selection"

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/57e2c600-bd21-4f9e-ba84-705b183d4f79)

4. Press "CTRL+F" and type in the driver name. Once done, click on "Find All"

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/f06f50cd-33fa-4901-bf7d-8774b1bb99dd)

The **New Thread Stack** column typically indicates threads that are in a waiting state. However, in this example, that is not the case, as evidenced by the call stack. Threads that are in a waiting state usually display specific wait routines in their call stack, such as, but not limited to, **`KeWaitForSingleObject`** or **`KeWaitForMultipleObjects`**. 

This means that thread ID **2304** is not in a waiting state.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/3c55fa1c-c67e-4e47-9257-d58359e1b274)

The following thread ID **6928** is waiting for a resource. The presence of functions like **`KeWaitForSingleObject`** and **`ExpWaitForResource`** suggests that the thread is in a waiting state. The **`ExpWaitForResource`** suggests that the thread is waiting to acquire an exclusive lock on a resource.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/91085d41-250f-4484-b3eb-34eae3d17c9d)

Similar to the previous call stack, this one also suggests that thread ID **1276** is in a waiting state, trying to acquire a resource exclusively but hasn't been able to yet.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/345ae846-8ee0-4919-897e-8606bdec261d)

Both call stacks show threads waiting on resources, and if these resources are being held by each other's corresponding threads, then we might have a deadlock situation. But the call stacks alone can't confirm that. However, it can give us some clues though that it is related about ERESOURCE. Let's take a kernel memory dump for further diagnostics.









