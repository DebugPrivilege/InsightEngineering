# What is a Kernel Driver?

A kernel driver is a piece of software that runs in the kernel space and interacts with the Windows operating system at a very low level. Kernel drivers have a **.sys** file extension and are important for enabling the OS to communicate with hardware and perform various system-level tasks.

The purpose of a kernel driver is to act as a bridge between the operating system and hardware or software components, facilitating low-level operations that can't be performed in user mode. Kernel drivers operate in a privileged state, allowing them direct access to system memory and hardware resources.

For example, the **kbdclass.sys** is a keyboard driver that manages the input from the keyboard, translating key presses into signals that the operating system can understand, while the **NTFS.sys** is a disk driver that enables the OS to read and write to NTFS file systems. It handles operations like file creation, deletion, and modification on an NTFS partition

# Code Sample - Basic Kernel Driver

This Windows Kernel Driver code creates a system thread when it loads, and the thread writes **`"Hello, World!"`** to a file at **`C:\Temp\HelloWorld.txt`**. When the driver is unloaded, it prints a **`"Driver Unloaded"`** message to the debug output.

```c
#include <ntddk.h>

// Thread function that creates a file and writes to it
VOID FileCreateThread(PVOID Context)
{
    UNREFERENCED_PARAMETER(Context);  // Context is not used, so mark it as unreferenced

    // Declare variables
    NTSTATUS status;  // Status for function calls
    HANDLE fileHandle; // Handle to the file
    IO_STATUS_BLOCK ioStatusBlock; // Status block for IO operations
    UNICODE_STRING fileName = RTL_CONSTANT_STRING(L"\\DosDevices\\C:\\Temp\\HelloWorld.txt"); // File path
    OBJECT_ATTRIBUTES attributes; // File attributes

    // Initialize object attributes
    InitializeObjectAttributes(&attributes, &fileName, OBJ_CASE_INSENSITIVE | OBJ_KERNEL_HANDLE, NULL, NULL);

    // Create the file
    status = ZwCreateFile(&fileHandle,
        GENERIC_WRITE,
        &attributes,
        &ioStatusBlock,
        0,
        FILE_ATTRIBUTE_NORMAL,
        0,
        FILE_OVERWRITE_IF,
        FILE_SYNCHRONOUS_IO_NONALERT,
        NULL,
        0);

    // Check if file creation succeeded
    if (NT_SUCCESS(status))
    {
        // Data to write to the file
        char data[] = "Hello, World!";

        // Write data to the file
        status = ZwWriteFile(fileHandle,
            NULL,
            NULL,
            NULL,
            &ioStatusBlock,
            data,
            sizeof(data) - 1,
            NULL,
            NULL);

        // Check if the write operation succeeded
        if (NT_SUCCESS(status))
        {
            KdPrint(("File successfully created and written from thread.\n"));
        }
        else
        {
            KdPrint(("Failed to write to file: %x\n", status));
        }

        // Close the file handle
        ZwClose(fileHandle);
    }
    else
    {
        KdPrint(("Failed to create file: %x\n", status));
    }

    // Terminate the system thread
    PsTerminateSystemThread(STATUS_SUCCESS);
}

// Driver Unload routine
VOID UnloadDriver(PDRIVER_OBJECT DriverObject)
{
    KdPrint(("Driver Unloaded\n"));
}

// Entry point for the driver
NTSTATUS DriverEntry(PDRIVER_OBJECT DriverObject, PUNICODE_STRING RegistryPath)
{
    UNREFERENCED_PARAMETER(RegistryPath);  // Not used, so mark as unreferenced

    // Set the Unload routine
    DriverObject->DriverUnload = UnloadDriver;

    // Variable to hold thread handle
    HANDLE threadHandle;

    // Create the system thread
    NTSTATUS status = PsCreateSystemThread(&threadHandle,
        GENERIC_EXECUTE,
        NULL,
        NULL,
        NULL,
        FileCreateThread,
        NULL);

    // Check if the thread was successfully created
    if (!NT_SUCCESS(status))
    {
        KdPrint(("Failed to create system thread: %x\n", status));
        return status;
    }

    // Variable for the thread object pointer
    PETHREAD threadObject;

    // Reference the thread object using the handle
    status = ObReferenceObjectByHandle(threadHandle,
        THREAD_ALL_ACCESS,
        NULL,
        KernelMode,
        (PVOID*)&threadObject,
        NULL);

    // Check for successful reference
    if (NT_SUCCESS(status))
    {
        // Dereference the thread object
        ObDereferenceObject(threadObject);
    }

    // Close the thread handle
    ZwClose(threadHandle);

    return STATUS_SUCCESS;
}
```

# Breakdown of the code

This is the thread function (**`FileCreateThread`**) that will create and write to a file:

```c
// Thread function that creates a file and writes to it
VOID FileCreateThread(PVOID Context)
{
    UNREFERENCED_PARAMETER(Context);  // Context is not used, so mark it as unreferenced

    // Declare variables
    NTSTATUS status;  // Status for function calls
    HANDLE fileHandle; // Handle to the file
    IO_STATUS_BLOCK ioStatusBlock; // Status block for IO operations
    UNICODE_STRING fileName = RTL_CONSTANT_STRING(L"\\DosDevices\\C:\\Temp\\HelloWorld.txt"); // File path
    OBJECT_ATTRIBUTES attributes; // File attributes

    // Initialize object attributes
    InitializeObjectAttributes(&attributes, &fileName, OBJ_CASE_INSENSITIVE | OBJ_KERNEL_HANDLE, NULL, NULL);

    // Create the file
    status = ZwCreateFile(&fileHandle,
        GENERIC_WRITE,
        &attributes,
        &ioStatusBlock,
        0,
        FILE_ATTRIBUTE_NORMAL,
        0,
        FILE_OVERWRITE_IF,
        FILE_SYNCHRONOUS_IO_NONALERT,
        NULL,
        0);

    // Check if file creation succeeded
    if (NT_SUCCESS(status))
    {
        // Data to write to the file
        char data[] = "Hello, World!";

        // Write data to the file
        status = ZwWriteFile(fileHandle,
            NULL,
            NULL,
            NULL,
            &ioStatusBlock,
            data,
            sizeof(data) - 1,
            NULL,
            NULL);

        // Check if the write operation succeeded
        if (NT_SUCCESS(status))
        {
            KdPrint(("File successfully created and written from thread.\n"));
        }
        else
        {
            KdPrint(("Failed to write to file: %x\n", status));
        }

        // Close the file handle
        ZwClose(fileHandle);
    }
    else
    {
        KdPrint(("Failed to create file: %x\n", status));
    }

    // Terminate the system thread
    PsTerminateSystemThread(STATUS_SUCCESS);
}
```

This piece of code is declaring and initializing variables that will be used for file operations later in the function:

```c
    // Declare variables
    NTSTATUS status;  // Status for function calls
    HANDLE fileHandle; // Handle to the file
    IO_STATUS_BLOCK ioStatusBlock; // Status block for IO operations
    UNICODE_STRING fileName = RTL_CONSTANT_STRING(L"\\DosDevices\\C:\\Temp\\HelloWorld.txt"); // File path
    OBJECT_ATTRIBUTES attributes; // File attributes
```

This table summarizes each variable, its purpose, its data type, and any additional notes that might be helpful:

| Variable Name         | Purpose                                                                                       | Type               | Additional Notes                              |
|----------------------|-----------------------------------------------------------------------------------------------|--------------------|-----------------------------------------------|
| `NTSTATUS status;`    | Holds the status returned by system calls like `ZwCreateFile` and `ZwWriteFile`.               | `NTSTATUS`         | Enumeration type for operation completion status. |
| `HANDLE fileHandle;`  | Acts as a unique key for the file, allowing to perform further actions like writing or closing specifically on that file after it's been created or opened.                 | `HANDLE`           | Handle to an object.                           |
| `IO_STATUS_BLOCK ioStatusBlock;` | Receives status information from file I/O operations.                 | `IO_STATUS_BLOCK`  | Structure for I/O operation results.           |
| `UNICODE_STRING fileName;` | Specifies the path of the file that will be created.                                         | `UNICODE_STRING`   | Structure for Unicode strings. Initialized with `RTL_CONSTANT_STRING`. |
| `OBJECT_ATTRIBUTES attributes;` | Holds the attributes of the file object.                                                     | `OBJECT_ATTRIBUTES`| Structure for object attributes.                |

This line of code is setting up the "rules" for the file we're about to create or open

```c
InitializeObjectAttributes(&attributes, &fileName, OBJ_CASE_INSENSITIVE | OBJ_KERNEL_HANDLE, NULL, NULL);
```

This table provides a detailed breakdown of each argument used in the **InitializeObjectAttributes** function call. It outlines the purpose of each argument, explaining what role it plays in setting up the attributes for the file operation.

| Argument              | Purpose                                                                                           | Additional Notes                             |
|----------------------|---------------------------------------------------------------------------------------------------|----------------------------------------------|
| `&attributes`         | Storing the attributes for the file operation in this variable.                                    | Will be used in `ZwCreateFile` later.        |
| `&fileName`           | Telling the system the name and path of the file we want to create or open.                        | Specifies where the file will be located.    |
| `OBJ_CASE_INSENSITIVE \| OBJ_KERNEL_HANDLE` | Setting flags for how we want the object to behave.                 | - `OBJ_CASE_INSENSITIVE`: Makes file name case-insensitive. <br> - `OBJ_KERNEL_HANDLE`: Allows handle access only in Kernel mode.|
| `NULL, NULL`          | Leaving optional parameters for root directory and security settings empty, as we're not using them.| These could be used for more specific control.|

This piece of code is using the **ZwCreateFile** function to create a new file. The function takes multiple arguments that we've previously set up, like **`fileHandle`** and **`attributes`**.

```c
    // Create the file
    status = ZwCreateFile(&fileHandle,
        GENERIC_WRITE,
        &attributes,
        &ioStatusBlock,
        0,
        FILE_ATTRIBUTE_NORMAL,
        0,
        FILE_OVERWRITE_IF,
        FILE_SYNCHRONOUS_IO_NONALERT,
        NULL,
        0);
```

This table summarizes each argument used in the **ZwCreateFile** function call, outlining their purpose and any additional notes that might be helpful:

| Argument                    | Purpose                                                                           | Additional Notes                                    |
|-----------------------------|-----------------------------------------------------------------------------------|-----------------------------------------------------|
| `&fileHandle`                | To store the handle of the new or opened file, which will be used for future operations on this file.     | Acts as a unique key for the file.                  |
| `GENERIC_WRITE`              | To specify that we want to open the file with write access.                       |                                                     |
| `&attributes`                | To provide the attributes we've set up for the file, like its name and other object settings.               | Initialized using `InitializeObjectAttributes`.     |
| `&ioStatusBlock`             | To get status information about the operation, such as whether it succeeded or failed.                      | Acts like a report card for the operation.          |
| `FILE_ATTRIBUTE_NORMAL`      | To indicate that the file should have normal attributes.                          | No special attributes like 'read-only' or 'hidden'. |
| `FILE_OVERWRITE_IF`          | To specify that if the file already exists, it should be overwritten.             |                                                     |
| `FILE_SYNCHRONOUS_IO_NONALERT`| To indicate that file operations should be synchronous and non-alertable.        | Function won't return until operation is complete.  |

At this piece of code, we're performing a series of operations to write data to the file and then close it.

```c
        // Data to write to the file
        char data[] = "Hello, World!";

        // Write data to the file
        status = ZwWriteFile(fileHandle,
            NULL,
            NULL,
            NULL,
            &ioStatusBlock,
            data,
            sizeof(data) - 1,
            NULL,
            NULL);

        // Check if the write operation succeeded
        if (NT_SUCCESS(status))
        {
            KdPrint(("File successfully created and written from thread.\n"));
        }
        else
        {
            KdPrint(("Failed to write to file: %x\n", status));
        }

        // Close the file handle
        ZwClose(fileHandle);
    }
    else
    {
        KdPrint(("Failed to create file: %x\n", status));
    }
```

This table summarizes each part of the code that is involved in writing data to the file, checking the success of the operation, and closing the file:

| Code Snippet                 | Purpose                                                                                     | Additional Notes                                                |
|------------------------------|---------------------------------------------------------------------------------------------|----------------------------------------------------------------|
| `char data[] = "Hello, World!";` | Declaring the data to be written to the file.                                               | A character array containing the string "Hello, World!".       |
| `ZwWriteFile(...)`           | Writing the declared data to the file.                                                      | Uses `fileHandle` to identify the file, and `&ioStatusBlock` to receive status information. |
| `sizeof(data) - 1`           | Specifying the number of bytes to write.                                                    | `-1` is to exclude the null-terminator from the write operation.|
| `if (NT_SUCCESS(status))`    | Checking if the write operation succeeded.                                                  | `NT_SUCCESS` is a macro that evaluates the `status` variable.   |
| `KdPrint(...)`               | Printing a message to the debug output based on the success or failure of the operation.     | Messages vary depending on the success or failure of the write operation. |
| `ZwClose(fileHandle);`       | Closing the file handle.                                                                    | Releases the resources associated with `fileHandle`.            |
| `else { KdPrint(...); }`     | Executed if the file creation failed.                                                        | Prints an error message with the status code if file creation failed. |

The **UnloadDriver** function serves as the driver's unload routine. This function is called when the driver is being unloaded from memory. Unloading a driver is important for freeing up system resources and ensuring system stability.

```c
// Driver Unload routine
VOID UnloadDriver(PDRIVER_OBJECT DriverObject)
{
    KdPrint(("Driver Unloaded\n"));
}
```

This table summarizes the **UnloadDriver** function, which is the driver's unload routine:

| Code Snippet                  | Purpose                                                                                     | Additional Notes                                        |
|-------------------------------|---------------------------------------------------------------------------------------------|--------------------------------------------------------|
| `VOID UnloadDriver(PDRIVER_OBJECT DriverObject)` | Defines the function for the driver's unload routine.                                       | Takes a pointer to `DRIVER_OBJECT` as an argument.      |
| `KdPrint(("Driver Unloaded\n"));` | Prints a debug message indicating that the driver has been unloaded.                       | Useful for debugging or logging purposes.               |


**`DriverEntry`** is the entry point function for a Windows kernel-mode driver, similar to the **`main()`** function in a standard C program. When the driver is loaded into the system, **`DriverEntry`** is the first function that gets called, and it's responsible for initializing the driver, setting up any required resources, and defining the driver's behavior.

```c
// Entry point for the driver
NTSTATUS DriverEntry(PDRIVER_OBJECT DriverObject, PUNICODE_STRING RegistryPath)
{
    UNREFERENCED_PARAMETER(RegistryPath);  // Not used, so mark as unreferenced

    // Set the Unload routine
    DriverObject->DriverUnload = UnloadDriver;

    // Variable to hold thread handle
    HANDLE threadHandle;

    // Create the system thread
    NTSTATUS status = PsCreateSystemThread(&threadHandle,
        GENERIC_EXECUTE,
        NULL,
        NULL,
        NULL,
        FileCreateThread,
        NULL);

    // Check if the thread was successfully created
    if (!NT_SUCCESS(status))
    {
        KdPrint(("Failed to create system thread: %x\n", status));
        return status;
    }

    // Variable for the thread object pointer
    PETHREAD threadObject;

    // Reference the thread object using the handle
    status = ObReferenceObjectByHandle(threadHandle,
        THREAD_ALL_ACCESS,
        NULL,
        KernelMode,
        (PVOID*)&threadObject,
        NULL);

    // Check for successful reference
    if (NT_SUCCESS(status))
    {
        // Dereference the thread object
        ObDereferenceObject(threadObject);
    }

    // Close the thread handle
    ZwClose(threadHandle);

    return STATUS_SUCCESS;
}
```

This piece of code is responsible for creating and managing a system thread:

```c
    // Create the system thread
    NTSTATUS status = PsCreateSystemThread(&threadHandle,
        GENERIC_EXECUTE,
        NULL,
        NULL,
        NULL,
        FileCreateThread,
        NULL);

    // Check if the thread was successfully created
    if (!NT_SUCCESS(status))
    {
        KdPrint(("Failed to create system thread: %x\n", status));
        return status;
    }
```

This table summarizes each part of the code involved in creating and managing a system thread:

| Code Snippet                          | Purpose                                                                                     | Additional Notes                                                  |
|---------------------------------------|---------------------------------------------------------------------------------------------|------------------------------------------------------------------|
| `NTSTATUS status = PsCreateSystemThread(&threadHandle, ..., FileCreateThread, NULL);` | Creates a new system thread running the `FileCreateThread` function. | The handle to this thread is stored in `threadHandle`.           |
| `if (!NT_SUCCESS(status)) { ... }`    | Checks if the system thread was successfully created.                                       | Prints a debug message and returns the status code if it fails.   |
| `PETHREAD threadObject;`              | Declares a variable to hold the thread object.                                               | Will hold the thread object pointer later.                        |
| `status = ObReferenceObjectByHandle(threadHandle, ..., (PVOID*)&threadObject, NULL);` | Retrieves a pointer to the thread object using its handle.         | Allows us to interact directly with the thread object.           |
| `if (NT_SUCCESS(status)) { ... }`     | Checks if the thread object was successfully referenced.                                     |          |
| `ZwClose(threadHandle);`              | Closes the thread handle.                                                                    | Releases the resources associated with the thread handle.         |
| `return STATUS_SUCCESS;`              | Returns a success status code.                                                               | Indicates that all operations were successful.                    |

**PsCreateSystemThread** is often considered the kernel-mode equivalent of the user-mode **CreateThread** function in Windows. 

# How to list Kernel Drivers?

The **`driverquery`** command is a built-in Windows command-line utility that allows you to list all installed device drivers on a system. 

```
C:\Users\User>driverquery /v

Module Name  Display Name           Description            Driver Type   Start Mode State      Status     Accept Stop Accept Pause Paged Pool(bytes) Code(bytes) BSS(bytes) Link Date              Path                                             Init(bytes)
============ ====================== ====================== ============= ========== ========== ========== =========== ============ ================= =========== ========== ====================== ================================================ ===========
1394ohci     1394 OHCI Compliant Ho 1394 OHCI Compliant Ho Kernel        Manual     Stopped    OK         FALSE       FALSE        4,096             233,472     0                                 C:\Windows\system32\drivers\1394ohci.sys         4,096
3ware        3ware                  3ware                  Kernel        Manual     Stopped    OK         FALSE       FALSE        0                 81,920      0          5/18/2015 3:28:03 PM   C:\Windows\system32\drivers\3ware.sys            4,096
ACPI         Microsoft ACPI Driver  Microsoft ACPI Driver  Kernel        Boot       Running    OK         TRUE        FALSE        176,128           393,216     0                                 C:\Windows\system32\drivers\ACPI.sys             24,576
AcpiDev      ACPI Devices driver    ACPI Devices driver    Kernel        Manual     Stopped    OK         FALSE       FALSE        8,192             12,288      0                                 C:\Windows\system32\drivers\AcpiDev.sys          4,096
acpiex       Microsoft ACPIEx Drive Microsoft ACPIEx Drive Kernel        Boot       Running    OK         TRUE        FALSE        40,960            69,632      0                                 C:\Windows\system32\Drivers\acpiex.sys           4,096
acpipagr     ACPI Processor Aggrega ACPI Processor Aggrega Kernel        Manual     Stopped    OK         FALSE       FALSE        4,096             8,192       0                                 C:\Windows\system32\drivers\acpipagr.sys         4,096
AcpiPmi      ACPI Power Meter Drive ACPI Power Meter Drive Kernel        Manual     Stopped    OK         FALSE       FALSE        8,192             8,192       0                                 C:\Windows\system32\drivers\acpipmi.sys          4,096
acpitime     ACPI Wake Alarm Driver ACPI Wake Alarm Driver Kernel        Manual     Stopped    OK         FALSE       FALSE        8,192             12,288      0                                 C:\Windows\system32\drivers\acpitime.sys         4,096
Acx01000     Acx01000               Acx01000               Kernel        Manual     Stopped    OK         FALSE       FALSE        499,712           135,168     0                                 C:\Windows\system32\drivers\Acx01000.sys         4,096
ADP80XX      ADP80XX                ADP80XX                Kernel        Manual     Stopped    OK         FALSE       FALSE        0                 1,101,824   0          4/9/2015 1:49:48 PM    C:\Windows\system32\drivers\ADP80XX.SYS          4,096
AFD          Ancillary Function Dri Ancillary Function Dri Kernel        System     Running    OK         TRUE        FALSE        147,456           323,584     0                                 C:\Windows\system32\drivers\afd.sys
    8,192
afunix       afunix                 afunix                 Kernel        System     Running    OK         TRUE        FALSE        24,576            12,288      0                                 C:\Windows\system32\drivers\afunix.sys
    4,096

<< SNIPPET >>     
```

# Load a Kernel Driver

1. Open CMD as an administrator and run the following command:

```
sc.exe create MyDriver type= kernel binPath=C:\Users\User\source\repos\Driver\x64\Release\Driver.sys
```
![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/ea52c61b-07e4-4752-b021-4ab4530fcfc8)

2. Start the service:

```
sc start MyDriver
```

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/80926b34-0bba-4ac9-adf0-d471f2887967)

3. Confirm that our kernel driver is running:

```
driverquery /v | find "MyDriver"
```

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/8d606042-1291-410b-b6f0-dcf8bf40ebc4)


