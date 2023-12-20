# Description

The threads are waiting on the **`FileCreationEvent`** as a synchronization mechanism to coordinate the completion of file creation and file moving tasks. However, the way these waits and signals are implemented in the code is likely to result in a deadlock, as both threads end up waiting for each other, preventing any further progress.

# Code Sample - Deadlock

In order to demonstrate this demo. Please follow the following steps:

- Create the following 2 folders:
  1. **C:\Temp**
  2. **C:\Temp2**

```c
#include <ntifs.h>
#include <ntstrsafe.h>

// Define a structure to hold our synchronization resources.
typedef struct _SYNC_RESOURCES {
    KEVENT FileCreationEvent;  // Synchronization event for signaling completion of file creation.
} SYNC_RESOURCES, * PSYNC_RESOURCES;

NTSTATUS FileCreationThread(PVOID Context) {
    PSYNC_RESOURCES SyncRes = (PSYNC_RESOURCES)Context;
    UNICODE_STRING uniFilePath;
    WCHAR filePathBuffer[150];
    char dataToWrite[] = "Hello, World!\r\n";
    IO_STATUS_BLOCK ioStatusBlock;

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

    // Wait for the file move to complete before signaling
    KeWaitForSingleObject(&SyncRes->FileCreationEvent, Executive, KernelMode, FALSE, NULL);

    // Signal completion of file creation
    KeSetEvent(&SyncRes->FileCreationEvent, IO_NO_INCREMENT, FALSE);

    return STATUS_SUCCESS;
}

NTSTATUS FileMoveThread(PVOID Context) {
    PSYNC_RESOURCES SyncRes = (PSYNC_RESOURCES)Context;
    IO_STATUS_BLOCK ioStatusBlock;
    WCHAR oldFilePathBuffer[150], newFilePathBuffer[150];

    // Wait for the file creation to complete
    KeWaitForSingleObject(&SyncRes->FileCreationEvent, Executive, KernelMode, FALSE, NULL);

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

    // Signal that file move is done
    KeSetEvent(&SyncRes->FileCreationEvent, IO_NO_INCREMENT, FALSE);

    // Wait for the file creation to signal completion
    KeWaitForSingleObject(&SyncRes->FileCreationEvent, Executive, KernelMode, FALSE, NULL);

    return STATUS_SUCCESS;
}

VOID UnloadDriver(IN PDRIVER_OBJECT DriverObject) {
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

    // Initialize the synchronization event
    KeInitializeEvent(&SyncRes->FileCreationEvent, SynchronizationEvent, FALSE);

    HANDLE threadHandle1, threadHandle2;

    status = PsCreateSystemThread(&threadHandle1, (ACCESS_MASK)0, NULL, (PHANDLE)0, NULL, FileCreationThread, SyncRes);
    if (NT_SUCCESS(status)) {
        ZwClose(threadHandle1);
    }

    status = PsCreateSystemThread(&threadHandle2, (ACCESS_MASK)0, NULL, (PHANDLE)0, NULL, FileMoveThread, SyncRes);
    if (NT_SUCCESS(status)) {
        ZwClose(threadHandle2);
    }

    return STATUS_SUCCESS;
}
```

Start loading this kernel driver:

```
sc.exe create TestDrv type= kernel binPath=C:\Users\User\source\repos\TestDriver\x64\Release\TestDriver.sys
```

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/c3cf27f4-8fb1-425e-a06f-dd55b94a533c)


Start the kernel driver:

```
sc.exe start TestDrv
```

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/ecc4f916-13df-44ae-b11f-a876b856dfc2)


We can see that 1000 files were created in the **C:\Temp** folder.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/a76b1199-c568-4f4e-a9fd-9c645ffdac7a)


However, the files have not been renamed and moved to the **C:\Temp2** folder.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/325cb5ad-89cf-44b3-90c0-dcb98bf1cb1b)


Now delete all the files from the **C:\Temp** folder. While the driver is running, start obtaining a kernel memory dump. Once done, we can start stop and delete the driver.

# Event Signaling - Structure

Load the Memory Dump in WinDbg:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/25df258c-4d62-4fd9-a9a6-4fc93b4feecf)


Let's start with the **`!mex.di`** command which stands for dump information. This command retrieves information about the machine from which the memory dump originated. It's especially handy when working with multiple memory dumps, as it helps distinguish the analysis related to each specific dump. This is an improved version of the **`vertarget`** command.

```
0: kd> !mex.di
Computer Name:  Not Found
Windows 10 Kernel Version 22621 MP (7 procs) Free x64
Product: WinNt, suite: TerminalServer SingleUserTS
Edition build lab: 22621.2506.amd64fre.ni_release_svc_prod3.231018-1809
Kernel base = 0xfffff803`27a00000 PsLoadedModuleList = 0xfffff803`286134a0
Debug session time: Sat Nov 25 04:31:01.857 2023 (UTC - 8:00)
System Uptime: 0 days 0:05:50.322
SystemManufacturer = Microsoft Corporation
SystemProductName = Virtual Machine
Processor: 11th Gen Intel(R) Core(TM) i7-1185G7 @ 3.00GHz
Bugcheck: 161 (5461736B6D6772, 0, 0, 0)
```

The **`!mex.tl`** command enumerates all the running processes present in the system at the time the memory dump was taken. Its intent is to provide a concise overview of all active processes, rather than detailed information. We can also just specify a process name within the command, which we will be doing to get the memory address of the **System.exe** process.

```
0: kd> !mex.tl system
PID           Address          Name
============= ================ ========================
0x4    0n4    ffff8004ce0ec040 System
0x10b4 0n4276 ffff8004cf6ad0c0 SystemSettingsBroker.exe
============= ================ ========================
PID           Address          Name

Warning! Zombie process(es) detected (not displayed). Count: 11 [zombie report]
```

We can for example switch to a different process context like **System.exe** and then examine this process by using the **`!mex.p`** command.

```
0: kd> !mex.p ffff8004ce0ec040
Name   Address                  Ses PID     Parent  PEB              Create Time                Mods Handle Thrd User Name
====== ======================== === ======= ======= ================ ========================== ==== ====== ==== =========================
System ffff8004ce0ec040 (E|K|O)   0 4 (0n4) 0 (0n0) 0000000000000000 12/20/2023 07:23:57.634 AM    0      0  241 WORKGROUP\WINDEV2305EVAL$

Memory Details:

    VM   Peak      Commit Size NPP Quota
    ==== ========= =========== =========
    4 MB 652.32 MB       40 KB     272 B

File Server Threads (Active threads running srv or srv2 driver)
Exective Work Queues

Show LPC Port information for process

Show Threads: Unique Stacks    !mex.listthreads (!lt) ffff8004ce0ec040    !process ffff8004ce0ec040 7
```

The **`!mex.us`** command allows us to display unique stacks within threads. First, we need to navigate to a specific process of interest. In this case, we are in the context of the **System.exe** process. With the **`!mex.us`** command we can also filter on a specific string, so it will only display the the call stacks that contains a specific keyword.

```
0: kd> !us TestDriver
1 thread [stats]: ffff8004cf5ed2c0
    fffff8034501fe86 nt!KiSwapContext+0x76
    fffff80344e6d645 nt!KiSwapThread+0xab5
    fffff80344e6f827 nt!KiCommitThreadWait+0x137
    fffff80344e71746 nt!KeWaitForSingleObject+0x256
    fffff803526a122b <Unloaded_TestDriver.sys>+0x122b

1 thread [stats]: ffff8004d2e7d040
    fffff8034501fe86 nt!KiSwapContext+0x76          
    fffff80344e6d645 nt!KiSwapThread+0xab5          
    fffff80344e6f827 nt!KiCommitThreadWait+0x137    
    fffff80344e71746 nt!KeWaitForSingleObject+0x256 
    fffff803528012b7 TestDriver!FileMoveThread+0x47 (C:\Users\User\source\repos\TestDriver\TestDriver\Driver.c @ 49)
    fffff80344f07287 nt!PspSystemThreadStartup+0x57 
    fffff8034501b8e4 nt!KiStartSystemThread+0x34    

1 thread [stats]: ffff8004d4a1b540
    fffff8034501fe86 nt!KiSwapContext+0x76
    fffff80344e6d645 nt!KiSwapThread+0xab5
    fffff80344e6f827 nt!KiCommitThreadWait+0x137
    fffff80344e71746 nt!KeWaitForSingleObject+0x256
    fffff803526a12b7 <Unloaded_TestDriver.sys>+0x12b7
    ffff8004d52d4080 0xffff8004d52d4080
    ffff8004d4a1b540 0xffff8004d4a1b540
    ffff8004d22e4080 0xffff8004d22e4080
    ffff8004d22e4378 0xffff8004d22e4378

1 thread [stats]: ffff8004d5773040
    fffff8034501fe86 nt!KiSwapContext+0x76               
    fffff80344e6d645 nt!KiSwapThread+0xab5               
    fffff80344e6f827 nt!KiCommitThreadWait+0x137         
    fffff80344e71746 nt!KeWaitForSingleObject+0x256      
    fffff8035280122b TestDriver!FileCreationThread+0x14b (C:\Users\User\source\repos\TestDriver\TestDriver\Driver.c @ 38)
    fffff80344f07287 nt!PspSystemThreadStartup+0x57      
    fffff8034501b8e4 nt!KiStartSystemThread+0x34         

Threads matching filter: 4 out of 241
```

The thread identified by **`ffff8004d5773040`** in the **System** process is waiting on a synchronization event object, identified by **`ffff8004d569df40`**. This waiting state is part of its normal operation, as implemented in the **`FileCreationThread`** function. The thread will remain in the waiting state until the synchronization event it is waiting on is signaled by another thread or process.

```
0: kd> !mex.t ffff8004d5773040
Process                   Thread                       CID       UserTime KernelTime ContextSwitches Wait Reason    Time State
System (ffff8004ce0ec040) ffff8004d5773040 (E|K|W|R|V) 4.24dc           0      453ms            3723 Executive   20s.296 Waiting

WaitBlockList:
    Object           Type                 Other Waiters
    ffff8004d569df40 SynchronizationEvent             1

Priority:
    Current Base Decrement ForegroundBoost IO Page
    9       8    0         0               0  5

# Child-SP         Return           Call Site                           Source
0 ffffcf0110c7ec70 fffff80344e6d645 nt!KiSwapContext+0x76               
1 ffffcf0110c7edb0 fffff80344e6f827 nt!KiSwapThread+0xab5               
2 ffffcf0110c7ef00 fffff80344e71746 nt!KiCommitThreadWait+0x137         
3 ffffcf0110c7efb0 fffff8035280122b nt!KeWaitForSingleObject+0x256      
4 ffffcf0110c7f350 fffff80344f07287 TestDriver!FileCreationThread+0x14b C:\Users\User\source\repos\TestDriver\TestDriver\Driver.c @ 38
5 ffffcf0110c7f570 fffff8034501b8e4 nt!PspSystemThreadStartup+0x57      
6 ffffcf0110c7f5c0 0000000000000000 nt!KiStartSystemThread+0x34
```

The event object **`ffff8004d569df40`** is currently in a non-signaled state, with two threads waiting on it.

```
0: kd> !mex.evt ffff8004d569df40

dt -r1 nt!_KEVENT ffff8004d569df40 () Recursive: [ -r1 -r2 -r ] Verbose Normal dt
==================================================================================
   +0x000 Header               : _DISPATCHER_HEADER
      +0x000 Lock                 : 0n393217 (0x60001)
      +0x000 LockNV               : 0n393217 (0x60001)
      +0x000 Type                 : 0x1 ''
      +0x001 Signalling           : 0 ''
      +0x002 Size                 : 0x6 ''
      +0x003 Reserved1            : 0 ''
      +0x000 TimerType            : 0x1 ''
      +0x001 TimerControlFlags    : 0 ''
      +0x001 Absolute             : 0y0
      +0x001 Wake                 : 0y0
      +0x001 EncodedTolerableDelay : 0y000000 (0)
      +0x002 Hand                 : 0x6 ''
      +0x003 TimerMiscFlags       : 0 ''
      +0x003 Index                : 0y000000 (0)
      +0x003 Inserted             : 0y0
      +0x003 Expired              : 0y0
      +0x000 Timer2Type           : 0x1 ''
      +0x001 Timer2Flags          : 0 ''
      +0x001 Timer2Inserted       : 0y0
      +0x001 Timer2Expiring       : 0y0
      +0x001 Timer2CancelPending  : 0y0
      +0x001 Timer2SetPending     : 0y0
      +0x001 Timer2Running        : 0y0
      +0x001 Timer2Disabled       : 0y0
      +0x001 Timer2ReservedFlags  : 0y00 (0n0)
      +0x002 Timer2ComponentId    : 0x6 ''
      +0x003 Timer2RelativeId     : 0 ''
      +0x000 QueueType            : 0x1 ''
      +0x001 QueueControlFlags    : 0 ''
      +0x001 Abandoned            : 0y0
      +0x001 DisableIncrement     : 0y0
      +0x001 QueueReservedControlFlags : 0y000000 (0)
      +0x002 QueueSize            : 0x6 ''
      +0x003 QueueReserved        : 0 ''
      +0x000 ThreadType           : 0x1 ''
      +0x001 ThreadReserved       : 0 ''
      +0x002 ThreadControlFlags   : 0x6 ''
      +0x002 CycleProfiling       : 0y0
      +0x002 CounterProfiling     : 0y1
      +0x002 GroupScheduling      : 0y1
      +0x002 AffinitySet          : 0y0
      +0x002 Tagged               : 0y0
      +0x002 EnergyProfiling      : 0y0
      +0x002 SchedulerAssist      : 0y0
      +0x002 ThreadReservedControlFlags : 0y0
      +0x003 DebugActive          : 0 ''
      +0x003 ActiveDR7            : 0y0
      +0x003 Instrumented         : 0y0
      +0x003 Minimal              : 0y0
      +0x003 Reserved4            : 0y00 (0n0)
      +0x003 AltSyscall           : 0y0
      +0x003 Emulation            : 0y0
      +0x003 Reserved5            : 0y0
      +0x000 MutantType           : 0x1 ''
      +0x001 MutantSize           : 0 ''
      +0x002 DpcActive            : 0x6 ''
      +0x003 MutantReserved       : 0 ''
      +0x004 SignalState          : 0n0 --> Non Signaled State
      +0x008 WaitListHead         : _LIST_ENTRY [ 0xffff8004`d2e7d180 - 0xffff8004`d5773180 ]


Number of waiters: 2
```

The following two threads are waiting on a synchronization event object:

```
0: kd> !mex.obj -waiters ffff8004d569df40
Process Thread             Id CSwitches User Kernel State      Time Reason    Wait Function
======= ================ ==== ========= ==== ====== ======= ======= ========= ===================================
System  ffff8004d2e7d040 1410         1    0      0 Waiting 22s.046 Executive TestDriver!FileMoveThread+0x47
System  ffff8004d5773040 24dc      3723    0  453ms Waiting 20s.296 Executive TestDriver!FileCreationThread+0x14b
```
