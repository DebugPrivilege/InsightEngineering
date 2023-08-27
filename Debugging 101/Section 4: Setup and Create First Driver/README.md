# Setting Up Windows Driver Kit (WDK)

This README provides a step-by-step guide for setting up the Windows Driver Kit (WDK) to start developing Windows kernel drivers.

**Prerequisites**
- Windows 10 or later
- Visual Studio 2022 or later

# How to setup everything?

"Spectre-mitigated libs" refers to libraries that have been compiled with protections against the Spectre vulnerability. These libraries are intended to help developers create more secure applications by providing built-in defenses against Spectre attacks. When developing Windows drivers, using Spectre-mitigated libraries adds an extra layer of security to the driver code.

1. Open Visual Studio
2. Go to **Tools**
3. Click on **Get Tools and Features**

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/375c79a8-4194-49ca-a1cf-cf98b2870cbc)

4. Click on **Individual components** and install **MSVC v143 - VS 2022 C++ x64/x86 Spectre-mitigated libs (Latest)**

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/ee6dae4e-f3cc-4e6d-af35-ec6a9141d75e)

The Windows Software Development Kit (SDK) workload is a collection of tools, libraries, and headers that developers can install to create applications that run on Windows. This workload provides the core tools for building Windows applications and drivers, including the headers, libraries, and so on.

5. Open **Workloads**
6. Install **Windows SDK** (Make sure to select the right version, if you use Windows 10, select 10. If you use Windows 11, select 11)

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/c0969cee-17c8-4fb4-b783-1e95cb27e601)

7. Install **Windows Driver Kit (WDK)** at the following link: https://go.microsoft.com/fwlink/?linkid=2196230

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/fa9e2533-e1cf-46d4-a30b-fae50922bc10)

8. Create a new project and select **Kernel Mode Driver (KMDF)**

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/a3a956a0-f771-465e-b523-7830eb4a70e9)

9. Let's create a simple Driver. This Driver spawns a system thread called **`FileCreateThread`**, which then creates a text file at **`C:\Temp\HelloWorld.txt`** and writes the string **`"Hello, World!"`** to it.
Edit the **Driver.c** file and paste the following code:

```c
#include <ntddk.h>

// Thread function that creates a file and writes to it
VOID FileCreateThread(PVOID Context)
{
    UNREFERENCED_PARAMETER(Context);  // Context is not used, so mark it as unreferenced

    // Variable declarations
    NTSTATUS status;
    HANDLE fileHandle;
    IO_STATUS_BLOCK ioStatusBlock;
    UNICODE_STRING fileName = RTL_CONSTANT_STRING(L"\\DosDevices\\C:\\Temp\\HelloWorld.txt");
    OBJECT_ATTRIBUTES attributes;

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

When we are trying to compile this code, it will throw a compiler error:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/1f25cb5f-1a52-4421-a085-9483297a6b11)

10. We can avoid these compiler errors by doing the following:
11. Go to Properties -> Under C/C++ -> General -> **Threat Warning As Errors: No (/WX-)**

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/886dd2a7-e240-4fba-88ce-bc86e6c84008)

# How to load a Kernel Driver?

1. Open CMD as an administrator
2. Run the following command:

```
sc create DrverTest type= kernel binPath=C:\Users\Admin\source\repos\Drver\x64\Release\Drver.sys
```

3. Start the Kernel Driver:

```
sc start DrverTest
```

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/dc11ce17-ee18-4dec-ab28-9a2213690c07)

We can use **WinObj** from SysInternals to verify that our driver has been loaded as well:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/61a6b292-0641-438c-a694-d2893c329e15)





   


