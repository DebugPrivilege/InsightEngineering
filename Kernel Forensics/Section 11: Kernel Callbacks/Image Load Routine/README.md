# Description

Image Load Routines are mechanisms provided by the operating system to notify registered drivers or subsystems about image loading events. An "image" in this context typically refers to an executable file or a DLL (Dynamic Link Library). Whenever an image is loaded into the memory, either for execution or as a dependency of another executable, the registered Image Load Routine is called by the system.

The following APIs are used for Image Load Notify Routines:

| Function Type       | Purpose                                      | API for Registration                | Description                                                                                                          |
|---------------------|----------------------------------------------|-------------------------------------|----------------------------------------------------------------------------------------------------------------------|
| Image Load Routine  | To monitor the loading of executable images. | `PsSetLoadImageNotifyRoutine`       | Notified when an image (executable or DLL) is loaded into memory. Useful for security monitoring, software auditing, and system analysis. |

# Image Load Notify Routine

This code is for a driver that registers a callback function **`(ImageLoadCallback)`** to monitor image loading events in the Windows operating system. Whenever an executable file or DLL (Dynamic Link Library) is loaded into memory, the **`ImageLoadCallback`** function is invoked by the system.

When an image is loaded, the callback function logs various details to the debug output. These details include the name of the loaded image, the process ID (PID) of the process loading the image, and the name of that process.

```c
#include <Ntifs.h>
#include <ntddk.h>
#include <wdm.h>

void ImageLoadCallback(PUNICODE_STRING FullImageName, HANDLE ProcessId, PIMAGE_INFO ImageInfo);
void DriverUnloadRoutine(PDRIVER_OBJECT DriverObject);
VOID LogCurrentIrql(const char* action, const char* functionName, KIRQL irql);

void ImageLoadCallback(PUNICODE_STRING FullImageName, HANDLE ProcessId, PIMAGE_INFO ImageInfo) {
    PEPROCESS Process = NULL;
    PUNICODE_STRING FullProcessName = NULL;

    // Log the current IRQL
    LogCurrentIrql("ImageLoadCallback", "Handling event", KeGetCurrentIrql());

    // Retrieve the process name
    PsLookupProcessByProcessId(ProcessId, &Process);
    if (Process) {
        SeLocateProcessImageName(Process, &FullProcessName);
        if (FullProcessName) {
            WCHAR* ProcessName = wcsrchr(FullProcessName->Buffer, L'\\');
            if (ProcessName) {
                ProcessName++; // Skip the backslash
            }
            else {
                ProcessName = FullProcessName->Buffer;
            }

            // Print image load information with process name
            DbgPrint("Image loaded: %wZ, PID: %d, Process Name: %-20S, Image Size: %d\n",
                FullImageName, ProcessId, ProcessName, ImageInfo->ImageSize);

            ExFreePool(FullProcessName);
        }
        ObDereferenceObject(Process);
    }
}

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

void DriverUnloadRoutine(PDRIVER_OBJECT DriverObject) {
    PsRemoveLoadImageNotifyRoutine(ImageLoadCallback);
    DbgPrint("Driver unloaded\n");
}

NTSTATUS DriverEntry(PDRIVER_OBJECT DriverObject, PUNICODE_STRING RegistryPath) {
    UNREFERENCED_PARAMETER(RegistryPath);
    DriverObject->DriverUnload = DriverUnloadRoutine;

    NTSTATUS status = PsSetLoadImageNotifyRoutine(ImageLoadCallback);
    if (!NT_SUCCESS(status)) {
        DbgPrint("Failed to register image load notify routine: 0x%X\n", status);
        return status;
    }

    DbgPrint("Driver loaded and image load notify routine registered\n");
    return STATUS_SUCCESS;
}
```

Start loading this kernel driver:

```
sc.exe create TestDrv type= kernel binPath=C:\Users\User\source\repos\TestDriver\x64\Debug\TestDriver.sys
```

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/6af5cc23-a3ef-4e8c-8c85-d2767e3e3335)


Start the kernel driver:

```
sc.exe start TestDrv
```

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/7ff6b86b-acf4-4cea-b6e6-8f686287d8bc)

The DebugView output is the result of the  **`ImageLoadCallback`** function being invoked in response to the image load and unload events.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/fb2cf473-ac5e-4cad-93f1-f42180b66cc2)

