# Summary

This section will concentrate on interpreting the data of a thread while it is in the process of acquiring a Mutex. The memory dump provided here did not result from a crash. The goal is to understand what happens when a thread is waiting on acquiring a Mutex, as well as viewing a data structure that is associated with KMUTEX.

Link to this CrashDump: https://mega.nz/file/S5NjDbpa#Ucp9lRtOAut6HybKZtERag91UXZB8LGTinqnFKgD6O4

# Code Sample - Delay Execution

The **`FileCreationThread`** function performs a sleep or delay in execution. This is accomplished by calling the **KeDelayExecutionThread** function, which suspends the execution of the current thread for a specified interval. It delay the thread's execution for 30 minutes. During this time, the thread is not using any CPU resources.

```c
#include <ntifs.h>
#include <ntstrsafe.h>

// Define a structure to hold our synchronization resources.
typedef struct _SYNC_RESOURCES {
    KMUTEX FileMutex;  // KMUTEX object for synchronized access to the files.
} SYNC_RESOURCES, * PSYNC_RESOURCES;

NTSTATUS FileCreationThread(PVOID Context) {
    PSYNC_RESOURCES SyncRes = (PSYNC_RESOURCES)Context;
    UNICODE_STRING uniFilePath;
    WCHAR filePathBuffer[150];
    char dataToWrite[] = "Hello, World!\r\n";
    IO_STATUS_BLOCK ioStatusBlock;

    // Acquire the KMUTEX
    KeWaitForSingleObject(&SyncRes->FileMutex, Executive, KernelMode, FALSE, NULL);

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

    // Hold the mutex for 30 minutes
    LARGE_INTEGER interval;
    // Convert 30 minutes to 100-nanosecond intervals
    interval.QuadPart = -30 * 60 * 10000000LL;

    // Delay execution for 30 minutes
    KeDelayExecutionThread(KernelMode, FALSE, &interval);

    // Release the KMUTEX
    KeReleaseMutex(&SyncRes->FileMutex, FALSE);

    return STATUS_SUCCESS;
}

NTSTATUS FileMoveThread(PVOID Context) {
    PSYNC_RESOURCES SyncRes = (PSYNC_RESOURCES)Context;
    IO_STATUS_BLOCK ioStatusBlock;
    WCHAR oldFilePathBuffer[150], newFilePathBuffer[150];

    // Acquire the KMUTEX
    KeWaitForSingleObject(&SyncRes->FileMutex, Executive, KernelMode, FALSE, NULL);

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

    // Release the KMUTEX
    KeReleaseMutex(&SyncRes->FileMutex, FALSE);

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

    // Initialize the KMUTEX
    KeInitializeMutex(&SyncRes->FileMutex, 0);

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

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/b0070084-13d9-46c6-a343-b974f587d08b)

Start the kernel driver:

```
sc.exe start TestDrv
```

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/d4e75981-c658-4a51-a2cc-4b8d24b267c1)


Let the driver run for a minute or two and then use **NotMyFault** to bugcheck the system:

```
notmyfault64.exe /crash
```

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/0c2dc3fe-6f9f-46dd-bc99-5784509e328b)

# WinDbg Walk Through - Analyzing Crash Dump

Load the CrashDump in WinDbg:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/b7aeb572-fa7b-408b-8a88-611179bf9cda)


Start with loading the **MEX** extension:

```
2: kd> .load mex
Mex External 3.0.0.7172 Loaded!
```

Let's start with the **`!di`** command which stands for dump information. This command retrieves information about the machine from which the memory dump originated. It's especially handy when working with multiple memory dumps, as it helps distinguish the analysis related to each specific dump. This is an improved version of the vertarget command.

```
2: kd> !di
Computer Name:  Not Found
Windows 10 Kernel Version 22621 MP (6 procs) Free x64
Product: WinNt, suite: TerminalServer SingleUserTS
Edition build lab: 22621.2506.amd64fre.ni_release_svc_prod3.231018-1809
Kernel base = 0xfffff805`4d200000 PsLoadedModuleList = 0xfffff805`4de134a0
Debug session time: Fri Nov  3 05:35:00.352 2023 (UTC - 8:00)
System Uptime: 0 days 0:17:02.325
SystemManufacturer = Microsoft Corporation
SystemProductName = Virtual Machine
Processor: 11th Gen Intel(R) Core(TM) i7-1185G7 @ 3.00GHz
Bugcheck: D1 (FFFF8001ACDC6010, 2, 0, FFFFF805736312D0)
Bugcheck: This is a myfault.sys dump.
KernelMode Dump Path: C:\Windows\MEMORY.DMP
Share Path: \\WINDEV2305EVAL\ADMIN$\MEMORY.DMP
```

Let's move our process over to **System.exe**. We can do this by using the **`!mex.tl`** command.

```
2: kd> !tl system
PID           Address          Name
============= ================ ========================
0x4    0n4    ffffc983ebceb040 System
0x2268 0n8808 ffffc983f3fec0c0 SystemSettingsBroker.exe
============= ================ ========================
PID           Address          Name

Warning! Zombie process(es) detected (not displayed). Count: 5 [zombie report]
```

We can now switch to the **System.exe** context with **`!mex.p`** command.

```
2: kd> !mex.p ffffc983ebceb040
Name   Address                  Ses PID     Parent  PEB              Create Time                Mods Handle Thrd User Name
====== ======================== === ======= ======= ================ ========================== ==== ====== ==== =========================
System ffffc983ebceb040 (E|K|O)   0 4 (0n4) 0 (0n0) 0000000000000000 11/03/2023 06:17:56.694 AM    0      0  257 WORKGROUP\WINDEV2305EVAL$

Memory Details:

    VM   Peak     Commit Size NPP Quota
    ==== ======== =========== =========
    4 MB 86.52 MB       44 KB     272 B

File Server Threads (Active threads running srv or srv2 driver)
Exective Work Queues

Show LPC Port information for process

Show Threads: Unique Stacks    !mex.listthreads (!lt) ffffc983ebceb040    !process ffffc983ebceb040 7
```

The **`!mex.us`** command allows us to view the call stacks in the threads of the **System.exe** process. But, rather than showing all call stacks, we'll narrow it down to only include those related to **`TestDriver`**.

```
0: kd> !us TestDriver
1 thread [stats]: ffffc983f2d47040
    fffff8054d61fe76 nt!KiSwapContext+0x76               
    fffff8054d4cb3f5 nt!KiSwapThread+0xab5               
    fffff8054d4cd5d7 nt!KiCommitThreadWait+0x137         
    fffff8054d4d1e29 nt!KeDelayExecutionThread+0xf9      
    fffff805735d1244 TestDriver!FileCreationThread+0x164 (C:\Users\User\source\repos\TestDriver\TestDriver\Driver.c @ 47)
    fffff8054d5573d7 nt!PspSystemThreadStartup+0x57      
    fffff8054d61b8d4 nt!KiStartSystemThread+0x34         

1 thread [stats]: ffffc983f3fb4040
    fffff8054d61fe76 nt!KiSwapContext+0x76          
    fffff8054d4cb3f5 nt!KiSwapThread+0xab5          
    fffff8054d4cd5d7 nt!KiCommitThreadWait+0x137    
    fffff8054d4cf4f6 nt!KeWaitForSingleObject+0x256 
    fffff805735d12c7 TestDriver!FileMoveThread+0x47 (C:\Users\User\source\repos\TestDriver\TestDriver\Driver.c @ 58)
    fffff8054d5573d7 nt!PspSystemThreadStartup+0x57 
    fffff8054d61b8d4 nt!KiStartSystemThread+0x34    

Threads matching filter: 2 out of 257
```

The thread we are examining is currently scheduled to sleep for 1 minute and 53.953 seconds, which is indicated by the **Wait Reason** being set to **DelayExecution**

```
0: kd> !mex.t ffffc983f2d47040
Process                   Thread                       CID       UserTime KernelTime ContextSwitches Wait Reason         Time State
System (ffffc983ebceb040) ffffc983f2d47040 (E|K|W|R|V) 4.c1c            0      594ms            4118 DelayExecution 1m:53.953 Waiting

Priority:
    Current Base Decrement ForegroundBoost IO Page
    9       8    0         0               0  5

# Child-SP         Return           Call Site                           Source
0 ffffc60b8849ef90 fffff8054d4cb3f5 nt!KiSwapContext+0x76               
1 ffffc60b8849f0d0 fffff8054d4cd5d7 nt!KiSwapThread+0xab5               
2 ffffc60b8849f220 fffff8054d4d1e29 nt!KiCommitThreadWait+0x137         
3 ffffc60b8849f2d0 fffff805735d1244 nt!KeDelayExecutionThread+0xf9      
4 ffffc60b8849f350 fffff8054d5573d7 TestDriver!FileCreationThread+0x164 C:\Users\User\source\repos\TestDriver\TestDriver\Driver.c @ 47
5 ffffc60b8849f570 fffff8054d61b8d4 nt!PspSystemThreadStartup+0x57      
6 ffffc60b8849f5c0 0000000000000000 nt!KiStartSystemThread+0x34
```

We will now examine our second thread. This thread has been in a wait state for 1 minute and 57.359 seconds, waiting on a **mutex** that is currently owned by a system thread with the identifier **`ffffc983f2d47040`**

```
0: kd> !mex.t ffffc983f3fb4040
Process                   Thread                       CID       UserTime KernelTime ContextSwitches Wait Reason      Time State
System (ffffc983ebceb040) ffffc983f3fb4040 (E|K|W|R|V) 4.878            0          0               1 Executive   1m:57.359 Waiting

WaitBlockList:
    Object           Type   Other Waiters Info
    ffffc983f2cf36a0 Mutant             0 Owning Thread: ffffc983f2d47040 (System)

# Child-SP         Return           Call Site                      Source
0 ffffc60b86656b60 fffff8054d4cb3f5 nt!KiSwapContext+0x76          
1 ffffc60b86656ca0 fffff8054d4cd5d7 nt!KiSwapThread+0xab5          
2 ffffc60b86656df0 fffff8054d4cf4f6 nt!KiCommitThreadWait+0x137    
3 ffffc60b86656ea0 fffff805735d12c7 nt!KeWaitForSingleObject+0x256 
4 ffffc60b86657240 fffff8054d5573d7 TestDriver!FileMoveThread+0x47 C:\Users\User\source\repos\TestDriver\TestDriver\Driver.c @ 58
5 ffffc60b86657570 fffff8054d61b8d4 nt!PspSystemThreadStartup+0x57 
6 ffffc60b866575c0 0000000000000000 nt!KiStartSystemThread+0x34
```

The **`_KMUTANT`** structure represents a kernel-mode **mutex** object. Here we can see that a mutex with the identifier **`ffffc983f2cf36a0`** is currently being owned by a thread with the identifier **`ffffc983f2d47040`** and there is one thread currently waiting on that mutex being released.

```
0: kd> !mex.mut ffffc983f2cf36a0

dt nt!_KMUTANT ffffc983f2cf36a0 () Recursive: [ -r1 -r2 -r ] Verbose Normal dt
==================================================================================
   +0x000 Header               : _DISPATCHER_HEADER
   +0x018 MutantListEntry      : _LIST_ENTRY [ 0xffffc983`f2d47348 - 0xffffc983`f2d47348 ] [EMPTY OR 1 ELEMENT]
   +0x028 OwnerThread          : 0xffffc983`f2d47040 _KTHREAD (System)
   +0x030 MutantFlags          : 0 ''
   +0x030 Abandoned            : 0y0
   +0x030 Spare1               : 0y0000000 (0)
   +0x030 Abandoned2           : 0y0
   +0x030 AbEnabled            : 0y0
   +0x030 Spare2               : 0y000000 (0)
   +0x031 ApcDisable           : 0x1 ''


Number of waiters: 1
Owning Thread: ffffc983f2d47040
```

Upon selecting the **1** under **Numbers of Waiters**, we are presented with the details of a thread that is currently in line waiting for the mutex.

```
0: kd> !mex.obj -waiters ffffc983f2cf36a0
Process Thread            Id CSwitches User Kernel State        Time Reason    Wait Function
======= ================ === ========= ==== ====== ======= ========= ========= ==============================
System  ffffc983f3fb4040 878         1    0      0 Waiting 1m:57.359 Executive TestDriver!FileMoveThread+0x47
```
