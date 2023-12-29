# Description

Thread Notify Routines serve a similar purpose to Process Notify Routines, but they focus specifically on thread lifecycle events. These routines are used to monitor the creation and termination of threads within the system. When a thread is created or exits in any process, the registered thread notify routine is called by the operating system.

| Function Type           | Purpose                                      | API for Registration              | Description                                                                                                         |
|-------------------------|----------------------------------------------|----------------------------------|---------------------------------------------------------------------------------------------------------------------|
| Thread Notify Routine   | To monitor thread creation and termination.  | `PsSetCreateThreadNotifyRoutine` | Notified when a thread is created or exits. Useful for security monitoring, system analysis, or managing thread-specific resources. |

# Thread Notify Routine

This code is for a driver that registers a callback function, **`ThreadNotifyCallback`**, to monitor thread creation and termination events. This is accomplished using the **`PsSetCreateThreadNotifyRoutine`** API, which sets up the callback to be invoked whenever a thread is created or terminated in the system.

```c
#include <Ntifs.h>
#include <ntddk.h>
#include <wdm.h>

void ThreadNotifyCallback(HANDLE ProcessId, HANDLE ThreadId, BOOLEAN Create);
void DriverUnloadRoutine(PDRIVER_OBJECT DriverObject);
VOID LogCurrentIrql(const char* action, const char* functionName, KIRQL irql);

void ThreadNotifyCallback(HANDLE ProcessId, HANDLE ThreadId, BOOLEAN Create) {
    // Log the current IRQL
    LogCurrentIrql("ThreadNotifyCallback", "Handling event", KeGetCurrentIrql());

    // Print header for the table
    DbgPrint("+--------------------------------------+\n");
    DbgPrint("| PID      | TID      | Thread Event   |\n");
    DbgPrint("+--------------------------------------+\n");

    // Check if thread is created or terminated
    if (Create) {
        DbgPrint("| %8d | %8d | Create          |\n", ProcessId, ThreadId);
    }
    else {
        DbgPrint("| %8d | %8d | Terminate       |\n", ProcessId, ThreadId);
    }

    // Print footer for the table
    DbgPrint("+--------------------------------------+\n");
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
    PsRemoveCreateThreadNotifyRoutine(ThreadNotifyCallback);
    DbgPrint("Driver unloaded\n");
}

NTSTATUS DriverEntry(PDRIVER_OBJECT DriverObject, PUNICODE_STRING RegistryPath) {
    UNREFERENCED_PARAMETER(RegistryPath);
    DriverObject->DriverUnload = DriverUnloadRoutine;

    NTSTATUS status = PsSetCreateThreadNotifyRoutine(ThreadNotifyCallback);
    if (!NT_SUCCESS(status)) {
        DbgPrint("Failed to register thread notify routine: 0x%X\n", status);
        return status;
    }

    DbgPrint("Driver loaded and thread notify routine registered\n");
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

The DebugView output is the result of the **`ThreadNotifyCallback`** function being invoked in response to thread creation and termination events.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/9e9d88e2-4591-4437-89f9-1ca7c4264056)


