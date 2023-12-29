# Description

Process Notify Routines in Windows Kernel are mechanisms provided by the operating system to notify registered drivers or subsystems about process lifecycle events. When a process is created or exits, the registered process notify routine is called by the system. It is commonly used in security software to maintain a log of process activity.

The following APIs are used for Process Notify Routines:

| Function Type                  | Purpose                                                         | API for Registration                        | Description                                                                                                       |
|--------------------------------|-----------------------------------------------------------------|---------------------------------------------|-------------------------------------------------------------------------------------------------------------------|
| Process Notify Routine         | To monitor process creation and termination.                    | `PsSetCreateProcessNotifyRoutine`           | Notified when a process is created or exits. Useful for security monitoring or managing system resources.        |


# Process Notify Routine

This code is for a driver that registers a callback function **`(ProcessNotifyCallback)`** to monitor process creation and termination events. When a process is created or terminated, the callback function logs details such as the process ID, parent process ID, and process name (for creation events) to the debug output.

```c
#include <Ntifs.h>
#include <ntddk.h>
#include <wdm.h>

void ProcessNotifyCallback(HANDLE ParentProcessId, HANDLE ProcessId, BOOLEAN IsCreate);
void DriverUnloadRoutine(PDRIVER_OBJECT DriverObject);
VOID LogCurrentIrql(const char* action, const char* functionName, KIRQL irql);

void ProcessNotifyCallback(HANDLE ParentProcessId, HANDLE ProcessId, BOOLEAN IsCreate) {
    PEPROCESS Process = NULL;
    PUNICODE_STRING FullProcessName = NULL;
    
    // Log the current IRQL
    LogCurrentIrql("ProcessNotifyCallback", "Handling event", KeGetCurrentIrql());

    // Print header for the table
    DbgPrint("+--------------------------------------------+\n");
    DbgPrint("| Parent PID | PID      | Process Name       |\n");
    DbgPrint("+--------------------------------------------+\n");

    if (IsCreate) {
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

                // Print process creation information in table format
                DbgPrint("| %10d | %8d | %-20S |\n", ParentProcessId, ProcessId, ProcessName);
                ExFreePool(FullProcessName);
            }
            ObDereferenceObject(Process);
        }
    }
    else {
        // Since we don't have the process name at termination, we'll just display the PIDs
        DbgPrint("| %10d | %8d | -                    |\n", ParentProcessId, ProcessId);
    }

    // Print footer for the table
    DbgPrint("+--------------------------------------------+\n");
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
    PsSetCreateProcessNotifyRoutine(ProcessNotifyCallback, TRUE);
    DbgPrint("Driver unloaded\n");
}

NTSTATUS DriverEntry(PDRIVER_OBJECT DriverObject, PUNICODE_STRING RegistryPath) {
    UNREFERENCED_PARAMETER(RegistryPath);
    DriverObject->DriverUnload = DriverUnloadRoutine;

    NTSTATUS status = PsSetCreateProcessNotifyRoutine(ProcessNotifyCallback, FALSE);
    if (!NT_SUCCESS(status)) {
        DbgPrint("Failed to register process notify routine: 0x%X\n", status);
        return status;
    }

    DbgPrint("Driver loaded and process notify routine registered\n");
    return STATUS_SUCCESS;
}
```

Start loading this kernel driver:

```
sc.exe create TestDrv type= kernel binPath=C:\Users\User\source\repos\TestDriver\x64\Debug\TestDriver.sys
```

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/adea9f6f-395b-4faf-95e7-de4116e39a46)



Start the kernel driver:

```
sc.exe start TestDrv
```

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/896dcf15-6933-4329-968d-46bde7c6cec4)


The DebugView output is the result of the **`ProcessNotifyCallback`** function being invoked in response to process creation and termination events.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/fb9b0c68-7fdb-48ae-942e-69dd2697a1ae)


# Process Notify Routine - ASCII

This ASCII diagram outlines on how the Process Notify functionality is implemented and executed within the driver:

```
+-------------------------------+
|          DriverEntry          |
+-------------------------------+
                |
                | (Start) Initializes the driver and registers
                |         ProcessNotifyCallback using 
                |         PsSetCreateProcessNotifyRoutine.
                |         This callback will be invoked for
                |         process creation and termination.
                ↓
+---------------------------------------------+
|       PsSetCreateProcessNotifyRoutine       |
+---------------------------------------------+
                |
                | (Register) The system registers ProcessNotifyCallback.
                |           It will be called for process creation or
                |           termination events.
                ↓
+---------------------------------+
|      ProcessNotifyCallback      |
+---------------------------------+
                |
                | (Callback Execution) Automatically called by the
                |                     Windows Kernel on process events.
                |                     Determines if the event is process
                |                     creation and fetches details for logging.
                |
                |                     If a process is created:
                ↓
+------------------------------------+
|    PsLookupProcessByProcessId      |
+------------------------------------+
                |
                | (Fetch Process Handle) Retrieves a handle to the process
                |                        using its ID, which is then used to
                |                        obtain more information about the process.
                ↓
+---------------------------------+
|    SeLocateProcessImageName     |
+---------------------------------+
                |
                | (Retrieve Process Name) Gets the full image name of the
                |                         process and logs information.
                ↓
+-------------------------------+
|           DbgPrint            |
+-------------------------------+
                |
                | (Logging) Debug print statements log process information
                |           and other relevant details.
                |
                |           Logs ID and parent ID when a process terminates.
                ↓
+---------------------------------+
|      DriverUnloadRoutine        |
+---------------------------------+
                |
                | (Cleanup) Invoked during driver unloading. Unregisters
                |           ProcessNotifyCallback using 
                |           PsSetCreateProcessNotifyRoutine.
                ↓
+---------------------------------------------+
|       PsSetCreateProcessNotifyRoutine       |
+---------------------------------------------+
                |
                | (Unregister Callback) Unregisters the callback, signaling
                |                      the system to stop invoking this callback
                |                      for process events.
                ↓
```

