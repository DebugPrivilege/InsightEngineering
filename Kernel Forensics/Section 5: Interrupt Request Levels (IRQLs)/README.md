# What are Interrupt Request Levels?

Interrupt Request Levels (IRQLs) are a fundamental concept in kernel development and driver programming. They represent a prioritization scheme used by the Windows kernel to manage hardware interrupts, thread scheduling, and the execution of code at various priority levels. The purpose of IRQLs is to ensure that critical code sections are executed without interruption by less critical ones, maintaining system stability and efficiency. If less critical code sections were allowed to interrupt more critical ones in a computer system, it could lead to several problems affecting the system's stability and efficiency.

IRQLs work like a system of priority levels in a computer. Think of it as having a to-do list where some tasks are more important than others. The computer, using IRQLs, makes sure that the most important tasks (like responding to a click of your mouse or processing incoming network data) are done first. This way, less important tasks don't get in the way of these critical ones. By handling the most important tasks quickly, your computer runs smoothly and efficiently, just like how focusing on urgent tasks first helps you be more productive in your day.

IRQLs are organized into a hierarchy from lowest to highest priority. Here are the common levels but they are not limited to only **`PASSIVE_LEVEL`**, **`APC_LEVEL`**, and **`DISPATCH_LEVEL`**. The full list can be found here: https://view.officeapps.live.com/op/view.aspx?src=https%3A%2F%2Fdownload.microsoft.com%2Fdownload%2Fe%2Fb%2Fa%2Feba1050f-a31d-436b-9281-92cdfeae4b45%2FIRQL_thread.doc&wdOrigin=BROWSELINK

| IRQL Level       | Numeric Value | Description                                                                                                       | Example Use Case                                 |
|------------------|---------------|-------------------------------------------------------------------------------------------------------------------|--------------------------------------------------|
| `PASSIVE_LEVEL`  | 0             | The lowest IRQL, used for regular thread execution. Suitable for operations that might block or wait.             | A common use case is performing file I/O operations. For instance, reading data from a disk or writing to a file, where the thread may have to wait for the data to be ready.|
| `APC_LEVEL`      | 1             | Above `PASSIVE_LEVEL`, used for Asynchronous Procedure Calls (APCs) and handling page faults.                     | An example is executing I/O completion routines. After an I/O operation finishes, the completion routine can be executed at APC_LEVEL to process the results or to continue the next stage of processing. |
| `DISPATCH_LEVEL` | 2             | Higher than `APC_LEVEL`, used for thread scheduling and Deferred Procedure Calls (DPCs). Non-interruptible level. | Deferred Procedure Calls (DPCs) are often used at this level, especially in scenarios like quickly handling hardware interrupts. For example, processing network packets in a network driver, where the packets need to be processed quickly, and the operation cannot afford to be interrupted.|

Here are some of the common IRQL functions to raise, lower or check the current IRQL level.

| Function            | Description                                                                                                   |
|---------------------|--------------------------------------------------------------------------------------------------------------------------|
| `KeRaiseIrql`       | Increases the current priority level of the system. This helps to stop less important tasks from interrupting important ones. |
| `KeLowerIrql`       | Reduces the priority level back to what it was before. This lets the system handle regular, less urgent tasks again.       |
| `KeGetCurrentIrql`  | Checks the current priority level of the system. Useful for making sure the system is at the right level for certain tasks. |

**IMPORTANT:** It's not the thread itself that runs on a particular IRQL (Interrupt Request Level) level. The concept of IRQLs in Windows is specifically related to the CPU and how it handles interrupts, not directly to the execution of threads. To give an example:

"Thread A runs on CPU 4, and during its execution, the CPU may operate at various IRQLs, including DISPATCH_LEVEL for certain high-priority tasks."

# Code Sample - Get current IRQL level of each thread

In order to demonstrate this demo. Please follow the following steps:

- Create the following 2 folders:
  1. **C:\Temp**
  2. **C:\Temp2**

This sample code is a driver that creates and manages files using synchronized threads, and logs the current IRQL (Interrupt Request Level) to a file to ensure operations are performed at the appropriate execution levels. It demonstrates file creation, modification, and basic synchronization using fast mutexes.

```c
#include <ntifs.h>
#include <ntstrsafe.h>

// Structure to hold synchronization resources.
typedef struct _SYNC_RESOURCES {
    FAST_MUTEX FileMutex;  // FAST_MUTEX object for synchronized access to files.
} SYNC_RESOURCES, * PSYNC_RESOURCES;

// Function to log the current IRQL level to a file.
NTSTATUS LogCurrentIrql(const WCHAR* logFilePath) {
    UNICODE_STRING uniLogFilePath;
    RtlInitUnicodeString(&uniLogFilePath, logFilePath); // Initialize UNICODE_STRING for file path.

    OBJECT_ATTRIBUTES objAttributes;
    InitializeObjectAttributes(&objAttributes, &uniLogFilePath, OBJ_CASE_INSENSITIVE | OBJ_KERNEL_HANDLE, NULL, NULL); // Set up object attributes.

    HANDLE logFile;
    IO_STATUS_BLOCK ioStatusBlock;
    // Create or open a file to write the IRQL level.
    NTSTATUS status = ZwCreateFile(&logFile, GENERIC_WRITE, &objAttributes, &ioStatusBlock, NULL, FILE_ATTRIBUTE_NORMAL, 0, FILE_OVERWRITE_IF, FILE_SYNCHRONOUS_IO_NONALERT, NULL, 0);
    if (!NT_SUCCESS(status)) {
        return status;
    }

    KIRQL currentIrql = KeGetCurrentIrql(); // Get the current IRQL.
    WCHAR irqlName[30];
    // Determine the name of the current IRQL and copy it to the buffer.
    switch (currentIrql) {
    case PASSIVE_LEVEL:
        wcscpy_s(irqlName, 30, L"PASSIVE_LEVEL");
        break;
    case APC_LEVEL:
        wcscpy_s(irqlName, 30, L"APC_LEVEL");
        break;
    case DISPATCH_LEVEL:
        wcscpy_s(irqlName, 30, L"DISPATCH_LEVEL");
        break;
    default:
        swprintf_s(irqlName, 30, L"UNKNOWN_LEVEL (%u)", currentIrql);
        break;
    }

    // Write the IRQL name to the file.
    ZwWriteFile(logFile, NULL, NULL, NULL, &ioStatusBlock, irqlName, wcslen(irqlName) * sizeof(WCHAR), NULL, NULL);
    ZwClose(logFile); // Close the file handle.

    return STATUS_SUCCESS;
}

// Thread function for file creation.
NTSTATUS FileCreationThread(PVOID Context) {
    PSYNC_RESOURCES SyncRes = (PSYNC_RESOURCES)Context;
    UNICODE_STRING uniFilePath;
    WCHAR filePathBuffer[150];
    char dataToWrite[] = "Hello, World!\r\n";
    IO_STATUS_BLOCK ioStatusBlock;

    // Log the current IRQL level to a file.
    LogCurrentIrql(L"\\DosDevices\\C:\\Temp\\FileCreationThreadIrql.txt");

    ExAcquireFastMutex(&SyncRes->FileMutex); // Acquire the fast mutex.

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

    ExReleaseFastMutex(&SyncRes->FileMutex); // Release the fast mutex.

    return STATUS_SUCCESS;
}

// Thread function for moving files.
NTSTATUS FileMoveThread(PVOID Context) {
    PSYNC_RESOURCES SyncRes = (PSYNC_RESOURCES)Context;
    IO_STATUS_BLOCK ioStatusBlock;
    WCHAR oldFilePathBuffer[150], newFilePathBuffer[150];

    // Log the current IRQL level to a file.
    LogCurrentIrql(L"\\DosDevices\\C:\\Temp\\FileMoveThreadIrql.txt");

    ExAcquireFastMutex(&SyncRes->FileMutex); // Acquire the fast mutex.

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

    ExReleaseFastMutex(&SyncRes->FileMutex); // Release the fast mutex.

    return STATUS_SUCCESS;
}

// Driver unload routine.
VOID UnloadDriver(IN PDRIVER_OBJECT DriverObject) {
    IoDeleteDevice(DriverObject->DeviceObject); // Delete the device object.
}

// Driver entry point.
NTSTATUS DriverEntry(PDRIVER_OBJECT DriverObject, PUNICODE_STRING RegistryPath) {
    DriverObject->DriverUnload = UnloadDriver; // Set the unload routine.

    PDEVICE_OBJECT DeviceObject = NULL;
    // Create a device object.
    NTSTATUS status = IoCreateDevice(DriverObject, sizeof(SYNC_RESOURCES), NULL, FILE_DEVICE_UNKNOWN, FILE_DEVICE_SECURE_OPEN, FALSE, &DeviceObject);

    if (!NT_SUCCESS(status)) {
        return status;
    }

    PSYNC_RESOURCES SyncRes = (PSYNC_RESOURCES)DeviceObject->DeviceExtension;
    ExInitializeFastMutex(&SyncRes->FileMutex); // Initialize the fast mutex.

    HANDLE threadHandle1, threadHandle2;
    // Create two threads for file creation and moving.
    status = PsCreateSystemThread(&threadHandle1, (ACCESS_MASK)0, NULL, (PHANDLE)0, NULL, FileCreationThread, SyncRes);
    if (NT_SUCCESS(status)) {
        ZwClose(threadHandle1); // Close the thread handle.
    }

    status = PsCreateSystemThread(&threadHandle2, (ACCESS_MASK)0, NULL, (PHANDLE)0, NULL, FileMoveThread, SyncRes);
    if (NT_SUCCESS(status)) {
        ZwClose(threadHandle2); // Close the thread handle.
    }

    return STATUS_SUCCESS;
}
```

Start loading this kernel driver:

```
sc.exe create TestDrv type= kernel binPath=C:\Users\User\source\repos\TestDriver\x64\Release\TestDriver.sys
```

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/3373a314-42c0-4f25-80ff-6f1cf202cd1a)

Start the kernel driver:

```
sc.exe start TestDrv
```

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/768bcdf4-8d51-4b99-92bc-f3a8e6529e09)


Two files will be created in the **C:\Temp** folder that contains the IRQL level of each thread. In this case, it happens to be **`PASSIVE_LEVEL`**.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/a190c404-102a-45ba-94da-2c7bd99709e7)


We can also see that 1000 files have now been renamed and moved to **C:\Temp2** folder.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/c5502c85-1801-4785-a717-8ad34f10d56d)


Now delete all the files from both **C:\Temp** and **C:\Temp2** folder.

# Code Sample - Raising IRQL level to APC_LEVEL

In order to demonstrate this demo. Please follow the following steps:

- Create the following 2 folders:
  1. **C:\Temp**
  2. **C:\Temp2**
  3. Download **DebugView** from SysInternals: https://learn.microsoft.com/en-us/sysinternals/downloads/debugview

Let's use an example that is related to fast mutex. A code path that is protected by a fast mutex runs at IRQL = **`APC_LEVEL`**. **`ExAcquireFastMutex`** and **`ExTryToAcquireFastMutex`** raise the current IRQL to **`APC_LEVEL`**, and **`ExReleaseFastMutex`** restores the original IRQL. Thus, all APCs are disabled while the thread holds a fast mutex.

This is something interesting that is documented by Microsoft, but I wanted to verify by myself from a code perspective. Compile the following code in **Debug** mode:

```c
#include <ntifs.h>
#include <ntstrsafe.h>

// Structure to hold synchronization resources.
typedef struct _SYNC_RESOURCES {
    FAST_MUTEX FileMutex;  // FAST_MUTEX object for synchronized access to files.
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

    ExAcquireFastMutex(&SyncRes->FileMutex); // Acquire the fast mutex.
    LogCurrentIrql("Acquiring Fast Mutex", __FUNCTION__, KeGetCurrentIrql());

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

    ExReleaseFastMutex(&SyncRes->FileMutex); // Release the fast mutex.
    LogCurrentIrql("Releasing Fast Mutex", __FUNCTION__, KeGetCurrentIrql());

    return STATUS_SUCCESS;
}

// Thread function for file movement.
NTSTATUS FileMoveThread(PVOID Context) {
    PSYNC_RESOURCES SyncRes = (PSYNC_RESOURCES)Context;
    IO_STATUS_BLOCK ioStatusBlock;
    WCHAR oldFilePathBuffer[150], newFilePathBuffer[150];

    ExAcquireFastMutex(&SyncRes->FileMutex); // Acquire the fast mutex.
    LogCurrentIrql("Acquiring Fast Mutex", __FUNCTION__, KeGetCurrentIrql());

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

    ExReleaseFastMutex(&SyncRes->FileMutex); // Release the fast mutex.
    LogCurrentIrql("Releasing Fast Mutex", __FUNCTION__, KeGetCurrentIrql());

    return STATUS_SUCCESS;
}

// Driver unload routine.
VOID UnloadDriver(IN PDRIVER_OBJECT DriverObject) {
    IoDeleteDevice(DriverObject->DeviceObject); // Delete the device object.
}

// Driver entry point.
NTSTATUS DriverEntry(PDRIVER_OBJECT DriverObject, PUNICODE_STRING RegistryPath) {
    DriverObject->DriverUnload = UnloadDriver; // Set the unload routine.

    PDEVICE_OBJECT DeviceObject = NULL;
    // Create a device object.
    NTSTATUS status = IoCreateDevice(DriverObject, sizeof(SYNC_RESOURCES), NULL, FILE_DEVICE_UNKNOWN, FILE_DEVICE_SECURE_OPEN, FALSE, &DeviceObject);

    if (!NT_SUCCESS(status)) {
        return status;
    }

    PSYNC_RESOURCES SyncRes = (PSYNC_RESOURCES)DeviceObject->DeviceExtension;
    ExInitializeFastMutex(&SyncRes->FileMutex); // Initialize the fast mutex.

    HANDLE threadHandle1, threadHandle2;
    // Create two threads for file creation and moving.
    status = PsCreateSystemThread(&threadHandle1, (ACCESS_MASK)0, NULL, (PHANDLE)0, NULL, FileCreationThread, SyncRes);
    if (NT_SUCCESS(status)) {
        ZwClose(threadHandle1); // Close the thread handle.
    }

    status = PsCreateSystemThread(&threadHandle2, (ACCESS_MASK)0, NULL, (PHANDLE)0, NULL, FileMoveThread, SyncRes);
    if (NT_SUCCESS(status)) {
        ZwClose(threadHandle2); // Close the thread handle.
    }

    return STATUS_SUCCESS;
}
```

Run **DebugView** as an administrator and start capturing:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/d7bad82a-5084-4a41-b1c1-419eda92e9ba)

Start loading this kernel driver:

```
sc.exe create TestDrv type= kernel binPath=C:\Users\User\source\repos\TestDriver\x64\Release\TestDriver.sys
```

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/ee18f500-3cfa-4937-b32b-798448084625)

Start the kernel driver:

```
sc.exe start TestDrv
```

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/f5beb1b6-d984-4c15-a105-74c4ce85a71b)

When **`ExAcquireFastMutex`** is called, the IRQL is raised to **`APC_LEVEL`**. This is evident from the log entries **00000003** and **00000004**, where both **`FileCreationThread`** and **`FileMoveThread`** report acquiring a Fast Mutex at **`APC_LEVEL`**. Upon releasing the fast mutex with **`ExReleaseFastMutex`**, the IRQL is restored to its original level. In our case, it's **`PASSIVE_LEVEL`**, as shown in log entries **00000005** and **00000007**.

This aligns well with what has been documented here by Microsoft: https://learn.microsoft.com/en-us/windows-hardware/drivers/kernel/fast-mutexes-and-guarded-mutexes

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/b031274c-d54f-4003-921e-e97c2d4a9633)

