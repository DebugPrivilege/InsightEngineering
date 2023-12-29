# Description

Object Manager Callbacks are a mechanism that allows kernel-mode drivers to monitor certain system operations related to object handles. Especially, **`ObRegisterCallbacks`** is the function used to set up these callbacks. When a driver registers a callback using **`ObRegisterCallbacks`**, it can specify routines to be called before and after operations such as creating, duplicating, or closing handles to processes, threads, and other objects.

In short, Object Manager Callbacks provide a way for drivers to intercept and react to key system operations involving object handles. This includes operations on processes, threads, and other kernel objects.

| Function Type            | Purpose                                                      | API for Registration  | Description                                                                                                                                                       |
|--------------------------|--------------------------------------------------------------|-----------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Object Manager Callback  | Monitor and control handle operations for objects.           | `ObRegisterCallbacks` | Enables kernel-mode drivers to register callbacks for operations like handle creation or duplication on objects like processes, threads, and files. Useful for implementing security policies, access control, auditing, and system behavior analysis. Pre-operation callbacks can modify operations, while post-operation callbacks are used for monitoring and logging. |


# Object Manager Callback

This code is for a driver that registers a callback function **`(LsassProcessHandleCallback)`** to monitor handle operations specifically for the lsass.exe process.

```c
#include <ntddk.h>
#include <ntstrsafe.h>

// Kernel API declarations
NTKERNELAPI char* PsGetProcessImageFileName(PEPROCESS Process);

// Global handle for callback registration
PVOID GlobalCallbackHandle = NULL;

// Function to bypass signature enforcement (not recommended for production)
void BypassSignatureEnforcement(PDRIVER_OBJECT DriverObject)
{
    typedef struct _LDR_DATA
    {
        // Structure members
        struct _LIST_ENTRY InLoadOrderLinks;
        struct _LIST_ENTRY InMemoryOrderLinks;
        struct _LIST_ENTRY InInitializationOrderLinks;
        VOID* DllBase;
        VOID* EntryPoint;
        ULONG32 SizeOfImage;
        UINT8 _PADDING0_[0x4];
        struct _UNICODE_STRING FullDllName;
        struct _UNICODE_STRING BaseDllName;
        ULONG32 Flags;
    } LDR_DATA, * PLDR_DATA;

    PLDR_DATA ldrData = (PLDR_DATA)(DriverObject->DriverSection);
    ldrData->Flags |= 0x20; // Modify flag to potentially bypass signature check
}

// Check if the process is "lsass.exe"
BOOLEAN IsTargetProcess(PEPROCESS Process)
{
    char* processName = PsGetProcessImageFileName(Process);
    return _stricmp("lsass.exe", processName) == 0;
}

// Callback for process handle operations
OB_PREOP_CALLBACK_STATUS LsassProcessHandleCallback(PVOID RegistrationContext, POB_PRE_OPERATION_INFORMATION OperationInformation)
{
    // Only handle process callbacks
    if (OperationInformation->ObjectType != *PsProcessType)
    {
        return OB_PREOP_SUCCESS;
    }

    // Check if the process is "lsass.exe"
    if (IsTargetProcess((PEPROCESS)OperationInformation->Object))
    {
        // Identify the type of operation (Create or Duplicate)
        const char* operationType = (OperationInformation->Operation == OB_OPERATION_HANDLE_CREATE) ? "Create" : "Duplicate";

        // Get access rights requested
        ACCESS_MASK requestedAccess = OperationInformation->Parameters->CreateHandleInformation.OriginalDesiredAccess;

        // Log detailed handle operation event
        DbgPrint("[lsass.exe] Handle %s operation detected. Requested Access: 0x%X\n", operationType, requestedAccess);
    }

    return OB_PREOP_SUCCESS;
}


// Driver unload function
VOID UnloadDriver(PDRIVER_OBJECT DriverObject)
{
    ObUnRegisterCallbacks(GlobalCallbackHandle);
    DbgPrint("Driver callback unregistered successfully.\n");
}

// Driver entry point
NTSTATUS DriverEntry(IN PDRIVER_OBJECT DriverObject, IN PUNICODE_STRING RegistryPath)
{
    DbgPrint("Driver for lsass.exe protection loaded.\n");

    // Optionally bypass signature enforcement
    BypassSignatureEnforcement(DriverObject);

    OB_OPERATION_REGISTRATION CallbackRegistration;
    OB_CALLBACK_REGISTRATION CallbackConfiguration;

    // Setup process callback for "lsass.exe"
    memset(&CallbackRegistration, 0, sizeof(CallbackRegistration));
    CallbackRegistration.ObjectType = PsProcessType;
    CallbackRegistration.Operations = OB_OPERATION_HANDLE_CREATE | OB_OPERATION_HANDLE_DUPLICATE;
    CallbackRegistration.PreOperation = LsassProcessHandleCallback;
    CallbackRegistration.PostOperation = NULL;

    // Configure the callback registration
    RtlUnicodeStringInit(&CallbackConfiguration.Altitude, L"600000");
    CallbackConfiguration.RegistrationContext = NULL;
    CallbackConfiguration.Version = OB_FLT_REGISTRATION_VERSION;
    CallbackConfiguration.OperationRegistration = &CallbackRegistration;
    CallbackConfiguration.OperationRegistrationCount = 1;

    // Register the process callback
    NTSTATUS status = ObRegisterCallbacks(&CallbackConfiguration, &GlobalCallbackHandle);
    if (NT_SUCCESS(status))
    {
        DbgPrint("Process callback for lsass.exe registered successfully.\n");
    }
    else
    {
        DbgPrint("Failed to register process callback for lsass.exe \n");
    }

    // Set the unload routine
    DriverObject->DriverUnload = UnloadDriver;

    return status;
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

The DebugView output is the result of the **`(LsassProcessHandleCallback)`** function being invoked in response to handle creation and duplication events.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/20dfb516-8344-44dc-b51a-4e18c550bc20)



# Object Manager Callback - Prevent process from being terminated

This code is for a driver that registers a callback function **`(LsassProcessObjectCallback)`** to monitor handle operations for the **`lsass.exe`** process. When a handle to the **`lsass.exe`** process is created or duplicated, the callback function checks the requested access rights and, if it includes **termination** rights, modifies them to prevent the process from being terminated. 

```c
#include <ntddk.h>
#include <ntstrsafe.h>

#define PROCESS_TERMINATE 0x1  // Define flag for process termination permission

// Declare functions from the NT kernel for process and thread operations
NTKERNELAPI PEPROCESS IoThreadToProcess(PETHREAD Thread);
NTKERNELAPI char* PsGetProcessImageFileName(PEPROCESS Process);

PVOID GlobalObjectHandle = NULL; // Global handle for the registered callback

// Function to bypass signature enforcement (not recommended for production)
void BypassSignatureChecks(PDRIVER_OBJECT DriverObject)
{
    // Define the structure representing the driver's load data
    typedef struct _LDR_DATA
    {
        struct _LIST_ENTRY InLoadOrderLinks;
        struct _LIST_ENTRY InMemoryOrderLinks;
        struct _LIST_ENTRY InInitializationOrderLinks;
        VOID* DllBase;
        VOID* EntryPoint;
        ULONG32 SizeOfImage;
        UINT8 _PADDING0_[0x4];
        struct _UNICODE_STRING FullDllName;
        struct _UNICODE_STRING BaseDllName;
        ULONG32 Flags;
    } LDR_DATA, * PLDR_DATA;

    // Modify the Flags field to bypass signature check
    PLDR_DATA ldrData = (PLDR_DATA)(DriverObject->DriverSection);
    ldrData->Flags |= 0x20;
}

// Function to check if a given process is "lsass.exe"
BOOLEAN CheckLsassProcess(PEPROCESS Process)
{
    char* processName = PsGetProcessImageFileName(Process);
    // Compare process name with "lsass.exe"
    return _stricmp("lsass.exe", processName) == 0;
}

// Callback function for process object operations
OB_PREOP_CALLBACK_STATUS LsassProcessObjectCallback(PVOID RegistrationContext, POB_PRE_OPERATION_INFORMATION OperationInformation)
{
    // Filter out non-process objects
    if (OperationInformation->ObjectType != *PsProcessType)
    {
        return OB_PREOP_SUCCESS;
    }

    // Check if the target process is "lsass.exe"
    if (CheckLsassProcess((PEPROCESS)OperationInformation->Object))
    {
        // Prevent termination of "lsass.exe" by removing termination permission
        if ((OperationInformation->Parameters->CreateHandleInformation.OriginalDesiredAccess & PROCESS_TERMINATE) == PROCESS_TERMINATE)
        {
            OperationInformation->Parameters->CreateHandleInformation.DesiredAccess &= ~PROCESS_TERMINATE;
        }
    }

    return OB_PREOP_SUCCESS;
}

// Unload function for the driver
VOID UnloadDriver(PDRIVER_OBJECT DriverObject)
{
    // Unregister the callback
    ObUnRegisterCallbacks(GlobalObjectHandle);
    DbgPrint("Callback unregistered successfully.\n");
}

// Entry point for the driver
NTSTATUS DriverEntry(IN PDRIVER_OBJECT DriverObject, IN PUNICODE_STRING RegistryPath)
{
    DbgPrint("Driver for lsass.exe protection loaded.\n");

    // Optionally bypass signature checks
    BypassSignatureChecks(DriverObject);

    // Set up the callback registration structure
    OB_OPERATION_REGISTRATION CallbackRegistration;
    OB_CALLBACK_REGISTRATION CallbackConfiguration;

    // Initialize the callback registration structure
    memset(&CallbackRegistration, 0, sizeof(CallbackRegistration));
    CallbackRegistration.ObjectType = PsProcessType;
    CallbackRegistration.Operations = OB_OPERATION_HANDLE_CREATE | OB_OPERATION_HANDLE_DUPLICATE;
    CallbackRegistration.PreOperation = LsassProcessObjectCallback;
    CallbackRegistration.PostOperation = NULL;

    // Configure the callback registration details
    RtlUnicodeStringInit(&CallbackConfiguration.Altitude, L"600000");
    CallbackConfiguration.RegistrationContext = NULL;
    CallbackConfiguration.Version = OB_FLT_REGISTRATION_VERSION;
    CallbackConfiguration.OperationRegistration = &CallbackRegistration;
    CallbackConfiguration.OperationRegistrationCount = 1;

    // Register the process callback
    NTSTATUS status = ObRegisterCallbacks(&CallbackConfiguration, &GlobalObjectHandle);
    if (NT_SUCCESS(status))
    {
        DbgPrint("Process callback for lsass.exe protection registered successfully.\n");
    }
    else
    {
        DbgPrint("Failed to register process callback for lsass.exe protection.\n");
    }

    // Set the driver's unload routine
    DriverObject->DriverUnload = UnloadDriver;

    return status;
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

The DebugView output confirms that our callback has been successfully registered:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/ec74cdab-87e1-4170-9bce-877a8ca40ddd)


When we are trying to terminate **lsass.exe** process, it won't work anymore:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/4077dd3e-5ffa-4666-827a-1aa7174a478d)

# Reference

- https://www.cnblogs.com/LyShark/p/16818453.html


