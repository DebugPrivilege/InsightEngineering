# Description

Registry Filters are components that monitor the behavior of registry operations. They are a part of the Windows kernel and provide a mechanism for software to intercept and process actions taken on the Windows Registry. For example; Security software (e.g. EDR) might use a Registry Filter to monitor changes to specific registry keys that are known to be targeted. When a change is detected, the filter can log the action, alert the user, or block the change.

The following APIs are used for Registry Filters:

| Function Type                | Purpose                                                         | API for Registration      | Description                                                                                                              |
|------------------------------|-----------------------------------------------------------------|---------------------------|--------------------------------------------------------------------------------------------------------------------------|
| Registry Filter              | To monitor and intercept registry operations.                   | `CmRegisterCallback`      | Notified before certain registry operations (like key or value creation, deletion, modification) are completed.          |
| Registry Filter (Ex version) | Enhanced monitoring and interception of registry operations.    | `CmRegisterCallbackEx`    | Provides additional capabilities over `CmRegisterCallback`, such as context passing and support for altitude specification in filter drivers. |

# Registry Filters

This code implements a driver that registers a callback function **`(RegistryCallbackRoutine)`** to monitor and log registry value set operations. The callback function is triggered by the system when a registry key's value is being set or modified. When this happens, the callback function retrieves the full path of the registry key and logs details such as the process ID of the process making the change, the path of the registry key, the name of the value being set, and the new value itself (if it's a DWORD).

This specific example focuses on **`RegNtSetValueKey`** operations, where a registry value is being set or modified.

```c
#include <ntifs.h>
#include <windef.h>

// Global variable for storing the registry callback cookie.
LARGE_INTEGER g_RegCallbackCookie;

// Function to retrieve the full path of a registry key object.
BOOLEAN GetRegistryKeyFullPath(PUNICODE_STRING pRegistryPath, PVOID pRegistryObject) {
    // Check if the registry object pointer is valid.
    if (!MmIsAddressValid(pRegistryObject) || pRegistryObject == NULL) {
        return FALSE;
    }

    // Allocate memory for storing the object name information.
    ULONG ulSize = 512;
    PVOID lpObjectNameInfo = ExAllocatePool(NonPagedPool, ulSize);
    if (lpObjectNameInfo == NULL) {
        return FALSE; // Memory allocation failed.
    }

    // Query the object's name.
    ULONG ulRetLen = 0;
    NTSTATUS status = ObQueryNameString(pRegistryObject, (POBJECT_NAME_INFORMATION)lpObjectNameInfo, ulSize, &ulRetLen);
    if (!NT_SUCCESS(status)) {
        ExFreePool(lpObjectNameInfo); // Free allocated memory on failure.
        return FALSE;
    }

    // Copy the retrieved name to the provided UNICODE_STRING.
    RtlCopyUnicodeString(pRegistryPath, (PUNICODE_STRING)lpObjectNameInfo);

    // Free the allocated memory and return success.
    ExFreePool(lpObjectNameInfo);
    return TRUE;
}

NTSTATUS RegistryCallbackRoutine(_In_ PVOID CallbackContext, _In_opt_ PVOID Argument1, _In_opt_ PVOID Argument2) {
    NTSTATUS status = STATUS_SUCCESS;
    UNICODE_STRING registryPath;

    // Cast the operation type from Argument1 to REG_NOTIFY_CLASS.
    REG_NOTIFY_CLASS operationType = (REG_NOTIFY_CLASS)Argument1;

    // Allocate memory for the registry path using ExAllocatePool2
    registryPath.Length = 0;
    registryPath.MaximumLength = 1024 * sizeof(WCHAR);
    registryPath.Buffer = ExAllocatePool2(POOL_FLAG_NON_PAGED, registryPath.MaximumLength, 'RegT');
    if (registryPath.Buffer == NULL) {
        return status; // Allocation failed, exit early.
    }
    RtlZeroMemory(registryPath.Buffer, registryPath.MaximumLength);

    // Get the current process ID
    PEPROCESS currentProcess = PsGetCurrentProcess(); // Current process
    HANDLE currentPID = PsGetProcessId(currentProcess); // Current process ID

    // Check if Argument2 (operation-specific data) is NULL.
    if (Argument2 == NULL) {
        if (registryPath.Buffer) {
            ExFreePool(registryPath.Buffer);
        }
        return status; // Argument2 is NULL, exit early.
    }

    // Handle specific registry operations.
    switch (operationType) {
    case RegNtSetValueKey: {
        // Cast Argument2 to the appropriate structure for the operation.
        PREG_SET_VALUE_KEY_INFORMATION SetValueInfo = (PREG_SET_VALUE_KEY_INFORMATION)Argument2;
        // If successful in retrieving the key path, log the information.
        if (GetRegistryKeyFullPath(&registryPath, SetValueInfo->Object)) {
            // Check if the value type is DWORD and log accordingly.
            if (SetValueInfo->Type == REG_DWORD && SetValueInfo->DataSize == sizeof(ULONG)) {
                ULONG valueData = *(PULONG)(SetValueInfo->Data);
                DbgPrint("[SetValue][PID: %lu][Key: %wZ][Value: %wZ][New DWORD Value: %lu]\n",
                    (ULONG)(ULONG_PTR)currentPID, &registryPath, SetValueInfo->ValueName, valueData);
            }
            else {
                DbgPrint("[SetValue][PID: %lu][Key: %wZ][Value: %wZ]\n",
                    (ULONG)(ULONG_PTR)currentPID, &registryPath, SetValueInfo->ValueName);
            }
        }
        else {
            // Log an error if the key path couldn't be retrieved.
            DbgPrint("[SetValue][PID: %lu] Unable to retrieve key path\n", (ULONG)(ULONG_PTR)currentPID);
        }
        break;
    }
    }

    // Free allocated memory for the registry path.
    if (registryPath.Buffer) {
        ExFreePool(registryPath.Buffer);
    }

    return status;
}

// Driver unload routine.
VOID DriverUnloadRoutine(PDRIVER_OBJECT driver) {
    DbgPrint("Unloading Registry Monitoring Driver\n");

    // Unregister the registry callback using the stored cookie.
    if (g_RegCallbackCookie.QuadPart > 0) {
        CmUnRegisterCallback(g_RegCallbackCookie);
    }
}

// Driver entry point.
NTSTATUS DriverEntry(IN PDRIVER_OBJECT Driver, PUNICODE_STRING RegistryPath) {
    DbgPrint("Initializing Registry Monitoring Driver\n");

    // Register the registry callback function.
    NTSTATUS status = CmRegisterCallback(RegistryCallbackRoutine, NULL, &g_RegCallbackCookie);
    if (!NT_SUCCESS(status)) {
        g_RegCallbackCookie.QuadPart = 0; // Reset the cookie on failure.
        return status;
    }

    // Set the driver's unload routine.
    Driver->DriverUnload = DriverUnloadRoutine;
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

The DebugView output is the result of the **`RegistryCallbackRoutine`** function being invoked in response to registry operations, particularly when a registry value is set or modified.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/5373f4e4-e9f0-4978-b002-86f608926f58)


Here we see an instance where a specific registry value is being modified, and a new value is assigned to it:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/55ef4fe0-30db-406d-92b4-14bb7530e143)

