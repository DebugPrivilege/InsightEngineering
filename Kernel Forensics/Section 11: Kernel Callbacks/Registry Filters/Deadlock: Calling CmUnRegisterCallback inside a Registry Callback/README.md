# Description

The call to **`CmUnRegisterCallback`** within the same callback tries to remove the registration of this callback. However, this operation needs to wait for the current execution of the callback to complete, as it must ensure that no more calls are made to this callback. 

Since the callback is actively executing and simultaneously attempting to unregister itself, it ends up waiting for its own execution to complete, leading to a deadlock. The registry subsystem gets locked up, waiting for resources that can't be released until the callback completes, which in turn can't complete because it's waiting to unregister itself.

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

    // Introduce a deadlock: Attempt to unregister the callback from within the callback routine.
// WARNING: This is for demonstration purposes only and should not be done in a real driver.
    NTSTATUS unregStatus = CmUnRegisterCallback(g_RegCallbackCookie);
    if (!NT_SUCCESS(unregStatus)) {
        DbgPrint("Failed to unregister callback within the callback routine: 0x%X\n", unregStatus);
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

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/d105f01d-8d5f-475d-96aa-baa84c0f5730)


The deadlock has now occurred in the driver.

# WinDbg Walk Through - Analysis

Load the Memory Dump in WinDbg:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/febae9d2-1f69-4a4a-9ec9-9115ca0758b7)


Let's start with the **`!mex.di`** command which stands for dump information. This command retrieves information about the machine from which the memory dump originated. It's especially handy when working with multiple memory dumps, as it helps distinguish the analysis related to each specific dump. This is an improved version of the **`vertarget`** command.

```
0: kd> !mex.di
Computer Name:  Not Found
Windows 10 Kernel Version 19041 MP (4 procs) Free x64
Product: WinNt, suite: TerminalServer SingleUserTS
Edition build lab: 19041.1.amd64fre.vb_release.191206-1406
Kernel base = 0xfffff806`0b200000 PsLoadedModuleList = 0xfffff806`0be2a770
Debug session time: Fri Dec 29 14:25:52.320 2023 (UTC - 8:00)
System Uptime: 0 days 0:36:44.995
SystemManufacturer = Microsoft Corporation
SystemProductName = Virtual Machine
Processor: 11th Gen Intel(R) Core(TM) i7-1185G7 @ 3.00GHz
Bugcheck: D1 (FFFFA48917A12010, 2, 0, FFFFF806203F12D0)
Bugcheck: This is a myfault.sys dump.
Bugcheck: 162 (FFFF910142869040, FFFF910145796F70, FFFFFFFF, 0)
```

Let's start with the **`!mex.tl -t`** command to discover threads of interest. It examines the state and wait reasons for every thread across all processes, presenting the results as statistics. This allows for a quick overview, such as determining how many processes have running, ready, and blocked threads for example.

```
0: kd> !mex.tl -t
PID           Address          Name                                   !! Rn Ry Bk Lc IO Er
============= ================ ====================================== == == == == == == ==
0x0    0n0    fffff8060bf24a00 Idle                                    .  4  .  .  .  .  .
0x4    0n4    ffff928d660a6040 System                                  4  .  .  1  .  1  .
0x64   0n100  ffff928d660ec080 Registry                                .  .  .  .  .  .  .
0x1b4  0n436  ffff928d6a131040 smss.exe                                .  .  .  1  .  .  .
0x224  0n548  ffff928d6af870c0 csrss.exe                               .  .  .  .  1  .  .
0x270  0n624  ffff928d6b0c1080 wininit.exe                             .  .  .  .  .  .  .
0x278  0n632  ffff928d6b0de140 csrss.exe                               .  .  .  .  1  .  .
0x2d0  0n720  ffff928d6b123080 winlogon.exe                            .  .  .  .  1  .  .
0x300  0n768  ffff928d6b13e2c0 services.exe                            .  .  .  .  .  .  .
0x32c  0n812  ffff928d6b153280 lsass.exe                               .  .  .  .  .  .  .
0x3a0  0n928  ffff928d6b1a1080 svchost.exe                             .  .  .  .  .  .  .
0x3b4  0n948  ffff928d6b1a2080 fontdrvhost.exe                         .  .  .  .  .  .  .
0x3c0  0n960  ffff928d6b15f080 fontdrvhost.exe                         .  .  .  .  .  .  .
0x204  0n516  ffff928d6b1f0240 svchost.exe                             .  .  .  .  .  .  .
0x228  0n552  ffff928d6b264080 svchost.exe                             .  .  .  .  .  .  .
0x3c8  0n968  ffff928d6b28b2c0 LogonUI.exe                             .  .  .  .  .  .  .
0x408  0n1032 ffff928d6b291200 dwm.exe                                 .  .  .  .  .  .  .
0x474  0n1140 ffff928d6b2ef080 svchost.exe                             .  .  .  .  .  .  .
0x518  0n1304 ffff928d6b37a0c0 svchost.exe                             .  .  .  .  .  .  .
0x520  0n1312 ffff928d6b370080 svchost.exe                             .  .  .  .  .  .  .
0x52c  0n1324 ffff928d6b37d080 svchost.exe                             .  .  .  .  .  .  .
0x538  0n1336 ffff928d6b380080 svchost.exe                             .  .  .  .  .  .  .
0x540  0n1344 ffff928d6b383080 svchost.exe                             .  .  .  .  .  .  .
0x550  0n1360 ffff928d6b37e080 svchost.exe                             .  .  .  .  .  .  .
0x59c  0n1436 ffff928d6b3c1080 svchost.exe                             .  .  .  .  .  .  .
0x608  0n1544 ffff928d6c083080 svchost.exe                             .  .  .  .  .  .  .
0x630  0n1584 ffff928d6c090080 svchost.exe                             .  .  .  .  .  .  .
0x640  0n1600 ffff928d6c091080 svchost.exe                             .  .  .  .  .  .  .
0x660  0n1632 ffff928d6c0c1080 svchost.exe                             .  .  .  .  .  .  .
0x688  0n1672 ffff928d6c0d5080 svchost.exe                             .  .  .  .  .  .  .
0x694  0n1684 ffff928d6c0d4080 svchost.exe                             .  .  .  .  .  .  .
0x7a0  0n1952 ffff928d6c217080 svchost.exe                             .  .  .  .  .  .  .
0x7b4  0n1972 ffff928d6c2560c0 svchost.exe                             .  .  .  .  1  .  .
0x7c8  0n1992 ffff928d6c25c080 svchost.exe                             .  .  .  .  .  .  .
0x7d0  0n2000 ffff928d6c259080 svchost.exe                             .  .  .  .  .  .  .
0x7e4  0n2020 ffff928d6c282080 svchost.exe                             .  .  .  .  .  .  .
0x7ec  0n2028 ffff928d6c285080 svchost.exe                             .  .  .  .  .  .  .
0x808  0n2056 ffff928d66133080 svchost.exe                             .  .  .  .  .  .  .
0x810  0n2064 ffff928d66131080 svchost.exe                             .  .  .  .  .  .  .
0x928  0n2344 ffff928d66158080 VSSVC.exe                               .  .  .  .  .  .  .
0x940  0n2368 ffff928d660da080 svchost.exe                             .  .  .  .  .  .  .
0x948  0n2376 ffff928d66153080 svchost.exe                             .  .  .  .  .  .  .
0x958  0n2392 ffff928d6c287080 svchost.exe                             .  .  .  .  .  .  .
0x960  0n2400 ffff928d6c2ec080 svchost.exe                             .  .  .  .  .  .  .
0x96c  0n2412 ffff928d6c3130c0 svchost.exe                             .  .  .  .  .  .  .
0x9e4  0n2532 ffff928d6c364080 svchost.exe                             .  .  .  .  .  .  .
0xa14  0n2580 ffff928d6c37d040 MemCompression                          .  .  .  .  .  .  .
0xa20  0n2592 ffff928d6c37e080 svchost.exe                             .  .  .  .  .  .  .
0xa30  0n2608 ffff928d6c3bf080 svchost.exe                             .  .  .  .  .  .  .
0xa3c  0n2620 ffff928d6c3c0080 svchost.exe                             .  .  .  .  .  .  .
0xa9c  0n2716 ffff928d6c3eb080 svchost.exe                             .  .  .  .  .  .  .
0xaf8  0n2808 ffff928d6c43f080 svchost.exe                             .  .  .  .  .  .  .
0xb3c  0n2876 ffff928d6c474080 svchost.exe                             .  .  .  .  .  .  .
0xb44  0n2884 ffff928d6c475080 svchost.exe                             .  .  .  .  .  .  .
0xb50  0n2896 ffff928d6c476080 svchost.exe                             .  .  .  .  .  .  .
0xbc0  0n3008 ffff928d6c4f2080 svchost.exe                             .  .  .  .  .  .  .
0xbc8  0n3016 ffff928d6c5130c0 svchost.exe                             .  .  .  .  .  .  .
0xbf0  0n3056 ffff928d6c515080 svchost.exe                             .  .  .  .  .  .  .
0xc0c  0n3084 ffff928d6c5a5080 spoolsv.exe                             .  .  .  .  .  .  .
0xc60  0n3168 ffff928d6c5b9080 svchost.exe                             .  .  .  .  .  .  .
0xd40  0n3392 ffff928d6c66a080 svchost.exe                             .  .  .  .  .  .  .
0xd4c  0n3404 ffff928d6c76b080 svchost.exe                             .  .  .  .  .  .  .
0xd54  0n3412 ffff928d6c768080 svchost.exe                             .  .  .  .  .  .  .
0xd5c  0n3420 ffff928d6c755080 IpOverUsbSvc.exe*32                     .  .  .  .  .  .  .
0xd6c  0n3436 ffff928d6c785080 svchost.exe                             .  .  .  .  .  .  .
0xda8  0n3496 ffff928d6c674080 sqlwriter.exe                           .  .  .  .  .  .  .
0xdbc  0n3516 ffff928d6c7bd080 Sysmon.exe                              .  1  .  .  .  .  .
0xdd4  0n3540 ffff928d6c7cf0c0 svchost.exe                             .  .  .  .  .  .  .
0xde4  0n3556 ffff928d6c7ca0c0 MsMpEng.exe                             .  1  .  .  .  .  .
0xdf8  0n3576 ffff928d6c7ec0c0 wlms.exe                                .  .  .  .  .  .  .
0xe00  0n3584 ffff928d6c7ef080 svchost.exe                             .  .  .  .  .  .  .
0xe08  0n3592 ffff928d6c7f0080 svchost.exe                             .  .  .  .  .  .  .
0xe94  0n3732 ffff928d6c874080 svchost.exe                             .  .  .  .  .  .  .
0xea8  0n3752 ffff928d6c877080 svchost.exe                             .  .  .  .  .  .  .
0xf7c  0n3964 ffff928d6c8ec080 sppsvc.exe                              .  .  .  .  .  .  .
0xfc0  0n4032 ffff928d6c94a0c0 svchost.exe                             .  .  .  .  .  .  .
0x58c  0n1420 ffff928d6c9be140 csrss.exe                               .  .  .  .  1  .  .
0xd64  0n3428 ffff928d6c9c7080 winlogon.exe                            .  .  .  .  .  .  .
0x10d4 0n4308 ffff928d6d070080 WUDFHost.exe                            .  .  .  .  .  .  .
0x1124 0n4388 ffff928d6c9a3080 dwm.exe                                 .  .  .  .  .  .  .
0x1204 0n4612 ffff928d6d2570c0 svchost.exe                             .  .  .  .  .  .  .
0x1238 0n4664 ffff928d6d2f7280 svchost.exe                             .  .  .  .  .  .  .
0x12a8 0n4776 ffff928d6d1e0300 fontdrvhost.exe                         .  .  .  .  .  .  .
0x1300 0n4864 ffff928d6d37a080 unsecapp.exe                            .  .  .  .  .  .  .
0xd30  0n3376 ffff928d6d6b9080 SearchIndexer.exe                       .  .  .  .  .  .  .
0x17bc 0n6076 ffff928d6d80f080 ctfmon.exe                              .  .  .  .  .  .  .
0xce0  0n3296 ffff928d6dac40c0 NisSrv.exe                              .  .  .  .  .  .  .
0x158c 0n5516 ffff928d6dbb2080 rdpclip.exe                             .  .  .  .  .  .  .
0x15c8 0n5576 ffff928d6dbe6080 sihost.exe                              .  .  .  .  .  .  .
0x15e0 0n5600 ffff928d6dbe80c0 svchost.exe                             .  .  .  .  .  .  .
0x15f8 0n5624 ffff928d6dc230c0 svchost.exe                             .  .  .  .  .  .  .
0x166c 0n5740 ffff928d6dc7a080 taskhostw.exe                           .  .  .  .  .  .  .
0x1680 0n5760 ffff928d6dc27080 svchost.exe                             .  .  .  .  .  .  .
0x1718 0n5912 ffff928d6dc9c080 svchost.exe                             .  .  .  .  .  .  .
0x1818 0n6168 ffff928d6dc98080 rdpinput.exe                            .  .  .  .  .  .  .
0x1848 0n6216 ffff928d6dd2d080 svchost.exe                             .  .  .  .  .  .  .
0x18b4 0n6324 ffff928d6ddbd080 explorer.exe                            4  .  .  1  1  .  .
0x1a14 0n6676 ffff928d6deee080 TabTip.exe                              .  .  .  .  .  .  .
0x1a54 0n6740 ffff928d6db2c080 svchost.exe                             .  .  .  .  .  .  .
0x1b34 0n6964 ffff928d6d7cd080 SppExtComObj.Exe                        .  .  .  .  .  .  .
0x1b98 0n7064 ffff928d6dc860c0 dllhost.exe                             .  .  .  .  .  .  .
0x1460 0n5216 ffff928d6dac8080 svchost.exe                             .  .  .  .  .  .  .
0x1c10 0n7184 ffff928d6e19b080 svchost.exe                             .  .  .  .  .  .  .
0x1c28 0n7208 ffff928d6e194080 StartMenuExperienceHost.exe             .  .  .  .  .  .  .
0x1cf4 0n7412 ffff928d6e3a9300 RuntimeBroker.exe                       .  .  .  .  .  .  .
0x1dd0 0n7632 ffff928d66f7a080 SearchApp.exe                          23  .  .  .  1  .  .
0x1e4c 0n7756 ffff928d6e513240 RuntimeBroker.exe                       .  .  .  .  1  .  .
0x1b14 0n6932 ffff928d6d76c080 RuntimeBroker.exe                       .  .  .  .  .  .  .
0x21e0 0n8672 ffff928d6e2a4080 svchost.exe                             .  .  .  .  .  .  .
0x227c 0n8828 ffff928d6e103080 PhoneExperienceHost.exe                 .  .  .  .  .  .  .
0x2308 0n8968 ffff928d6d067080 RuntimeBroker.exe                       .  .  .  .  .  .  .
0x2374 0n9076 ffff928d6e636300 SecurityHealthSystray.exe               .  .  .  .  .  .  .
0x239c 0n9116 ffff928d6e1b5300 SecurityHealthService.exe               .  .  .  .  .  .  .
0x23e4 0n9188 ffff928d6aeea0c0 OneDrive.exe                            .  .  .  .  .  .  .
0x23fc 0n9212 ffff928d6e10c340 msedge.exe                              .  .  .  .  .  .  .
0x16a0 0n5792 ffff928d6f6180c0 msedge.exe                              .  .  .  .  .  .  .
0x19f0 0n6640 ffff928d6f61d0c0 msedge.exe                              .  .  .  .  .  .  .
0x13d0 0n5072 ffff928d6aee50c0 msedge.exe                              .  .  .  .  .  .  .
0x1214 0n4628 ffff928d66c810c0 msedge.exe                              .  .  .  .  .  .  .
0x1fa0 0n8096 ffff928d6ded5080 msedge.exe                              .  .  .  .  .  .  .
0x1f18 0n7960 ffff928d6f607080 msedge.exe                              .  .  .  .  .  .  .
0x6e4  0n1764 ffff928d6e3aa080 svchost.exe                             .  .  .  .  .  .  .
0xb60  0n2912 ffff928d6dddc080 svchost.exe                             .  .  .  .  .  .  .
0x1d5c 0n7516 ffff928d6daca080 SgrmBroker.exe                          .  .  .  .  .  .  .
0x564  0n1380 ffff928d6a2be300 uhssvc.exe                              .  .  .  .  .  .  .
0xc7c  0n3196 ffff928d6b2d8340 svchost.exe                             .  .  .  .  .  .  .
0x1670 0n5744 ffff928d6ded62c0 svchost.exe                             .  .  .  .  .  .  .
0x16ec 0n5868 ffff928d6e642240 svchost.exe                             .  .  .  .  .  .  .
0x7d8  0n2008 ffff928d6e8b50c0 svchost.exe                             .  .  .  .  .  .  .
0x1b5c 0n7004 ffff928d6d6fb080 FileCoAuth.exe                          .  .  .  .  .  .  .
0x4cc  0n1228 ffff928d6b37c340 svchost.exe                             .  .  .  .  .  .  .
0x134c 0n4940 ffff928d6c4f1300 TextInputHost.exe                       .  .  .  .  .  .  .
0x1d28 0n7464 ffff928d6a335080 svchost.exe                             .  .  .  .  .  .  .
0x1f98 0n8088 ffff928d6db59080 SearchApp.exe                           5  .  .  .  1  .  .
0x185c 0n6236 ffff928d6e2b9300 dllhost.exe                             .  .  .  .  .  .  .
0x8c4  0n2244 ffff928d6e26f080 devenv.exe                              .  .  .  1  .  .  .
0x1af0 0n6896 ffff928d6d57d300 PerfWatson2.exe                         .  .  .  1  .  .  .
0x194  0n404  ffff928d6e9fa2c0 Microsoft.ServiceHub.Controller.exe     .  .  .  1  .  .  .
0x2330 0n9008 ffff928d66c722c0 ServiceHub.VSDetouredHost.exe           .  .  .  1  .  .  .
0x217c 0n8572 ffff928d6db5d080 audiodg.exe                             .  .  .  .  .  .  .
0xcf4  0n3316 ffff928d7003f0c0 ServiceHub.ThreadedWaitDialog.exe       .  .  .  1  .  .  .
0x1364 0n4964 ffff928d7028e280 ServiceHub.IndexingService.exe          .  .  .  .  .  .  .
0x167c 0n5756 ffff928d6e2212c0 vshost.exe                              1  .  .  .  .  .  .
0x1078 0n4216 ffff928d6ff0c0c0 ServiceHub.SettingsHost.exe             .  .  .  2  .  .  .
0x13ac 0n5036 ffff928d6ec81080 ServiceHub.IdentityHost.exe*32          .  .  .  2  .  .  .
0x1d48 0n7496 ffff928d6f703080 ServiceHub.IntellicodeModelService.exe  .  .  .  .  .  .  .
0x2350 0n9040 ffff928d6eb6a080 ServiceHub.Host.netfx.x86.exe*32        .  .  .  3  .  .  .
0x1938 0n6456 ffff928d6fe10340 MSBuild.exe                             .  .  .  .  .  .  .
0x65c  0n1628 ffff928d701e8340 conhost.exe                             .  .  .  .  .  .  .
0x97c  0n2428 ffff928d6dd16080 vcpkgsrv.exe*32                         .  .  .  .  .  .  .
0xf68  0n3944 ffff928d6eb22080 ServiceHub.Host.AnyCPU.exe              .  .  .  1  .  .  .
0x1958 0n6488 ffff928d700cb340 ServiceHub.TestWindowStoreHost.exe      .  .  .  1  .  .  .
0x224c 0n8780 ffff928d6e1b6080 cmd.exe                                 .  .  .  1  .  .  .
0x1a94 0n6804 ffff928d66c7d080 conhost.exe                             .  .  .  .  .  .  .
0x9d4  0n2516 ffff928d6aa0f080 vcpkgsrv.exe*32                         .  .  .  .  .  .  .
0x1008 0n4104 ffff928d6fdf9300 svchost.exe                             .  .  .  .  .  .  .
0x25a4 0n9636 ffff928d700db080 vcpkgsrv.exe*32                         .  .  .  .  .  .  .
0xa2c  0n2604 ffff928d6eb0d080 SearchProtocolHost.exe                  .  .  .  .  .  .  .
0x1508 0n5384 ffff928d6dd41080 SearchFilterHost.exe                    .  .  .  .  .  .  .
0x19b4 0n6580 ffff928d6f705080 vcpkgsrv.exe*32                         .  .  .  .  .  .  .
0x1f58 0n8024 ffff928d66e12080 sc.exe                                  .  .  .  .  1  .  .
0x514  0n1300 ffff928d6ffaa380 svchost.exe                             .  .  .  .  .  .  .
0x260c 0n9740 ffff928d6f614080 smartscreen.exe                         .  .  .  .  .  .  .
0x20b0 0n8368 ffff928d6e6ec080 cmd.exe                                 .  .  .  .  .  .  .
0x830  0n2096 ffff928d6e6ea080 conhost.exe                             .  .  .  .  .  .  .
0x247c 0n9340 ffff928d6e57a080 notmyfault64.exe                        .  1  .  .  .  .  .
============= ================ ====================================== == == == == == == ==
PID           Address          Name                                   !! Rn Ry Bk Lc IO Er

Warning! Zombie process(es) detected (not displayed). Count: 10 [zombie report]
```

From this output, it seems that there is a thread in the **System** process that are waiting for a Pushlock to be released.

```
0: kd> !mex.lt ffff928d660a6040 -bk -wr WrFastMutex,WrGuardedMutex,WrMutex,WrPushLock
Process PID Thread             Id State   Time Reason
======= === ================ ==== ======= ==== ==========
System    4 ffff928d6e52a040 1df8 Waiting 46ms WrPushLock

Thread Count: 1
```

This thread has been waiting 46ms on a push lock, but there is a bug. Since calling **`CmUnRegisterCallback`** inside a registry callback is not allowed.

```
0: kd> !mex.t ffff928d6e52a040
Process                   Thread                       CID       UserTime KernelTime ContextSwitches Wait Reason Time State
System (ffff928d660a6040) ffff928d6e52a040 (E|K|W|R|V) 4.1df8           0     2s.344           19014 WrPushLock  46ms Waiting

WaitBlockList:
    Object                  Type              Other Waiters
    ffff838739789d10(stack) NotificationEvent             0

Priority:
    Current Base Decrement ForegroundBoost IO Page
    13      12   0         0               0  5

 # Child-SP         Return           Call Site                                Source
 0 ffff838739789930 fffff8060b441330 nt!KiSwapContext+0x76                    
 1 ffff838739789a70 fffff8060b44085f nt!KiSwapThread+0x500                    
 2 ffff838739789b20 fffff8060b440103 nt!KiCommitThreadWait+0x14f              
 3 ffff838739789bc0 fffff8060b523be4 nt!KeWaitForSingleObject+0x233           
 4 ffff838739789cb0 fffff8060b523ab2 nt!ExTimedWaitForUnblockPushLock+0xb4    
 5 ffff838739789cf0 fffff8060ba6c56c nt!ExBlockOnAddressPushLock+0x62         
 6 ffff838739789d60 fffff806203e135a nt!CmUnRegisterCallback+0xfc             
 7 ffff838739789e20 fffff8060b848a98 TestDriver!RegistryCallbackRoutine+0x112 C:\Users\Admin\source\repos\TestDriver\TestDriver\Driver.c @ 99
 8 ffff838739789e70 fffff8060b8549c4 nt!CmpCallCallBacksEx+0x1c8              
 9 ffff838739789f80 fffff8060b8545d7 nt!CmpCallCallBacks+0x28                 
 a ffff838739789fd0 fffff8060b845950 nt!CmpDeleteKeyObject+0xa7               
 b ffff83873978a060 fffff8060b445c07 nt!ObpRemoveObjectRoutine+0x80           
 c ffff83873978a0c0 fffff8060b84b1b9 nt!ObfDereferenceObjectWithTag+0xc7      
 d ffff83873978a100 fffff8060b863beb nt!ObCloseHandleTableEntry+0x6c9         
 e ffff83873978a240 fffff8060b863b4b nt!ObpCloseHandle+0x8b                   
 f ffff83873978a2b0 fffff8060b93be15 nt!ObCloseHandle+0x2b                    
10 ffff83873978a2e0 fffff8060b97fe57 nt!IopLoadDriver+0x625                   
11 ffff83873978a4b0 fffff8060b4c46b5 nt!IopLoadUnloadDriver+0x57              
12 ffff83873978a4f0 fffff8060b5078e5 nt!ExpWorkerThread+0x105                 
13 ffff83873978a590 fffff8060b6064b8 nt!PspSystemThreadStartup+0x55           
14 ffff83873978a5e0 0000000000000000 nt!KiStartSystemThread+0x28
```

# Reference

- https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/wdm/nf-wdm-cmunregistercallback
