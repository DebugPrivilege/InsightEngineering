# Description

A semaphore is a synchronization primitive and is used where multiple threads or processes need to access a shared resource, like in an operating system or a multi-threaded application. It manages how many threads can access a certain resource at the same time. The semaphore does this by using a counter. 

The key points about semaphores are:

- **Counter:** The counter represents how many threads can access a shared resource simultaneously. When a thread wants to access the resource, it asks the semaphore first.

There are two types of Semaphores:

- **Binary Semaphore:** The counter can only be 0 or 1. If the counter is 1, it means the resource is available. If it's 0, the resource is being used. Only one thread can use the resource at a time. Ideal for ensuring mutual exclusion, like when accessing a critical section of code that must not be executed by more than one thread at a time.
- **Counting Semaphore:** This means a specified number of threads (as per the counter value) can access the resource simultaneously. It's like allowing a limited number of people to enter a room at the same time. Useful in scenarios where a certain number of identical resources are available, and multiple threads need to access these resources concurrently (e.g., database connections, thread pool management).

# Code Sample - Race Condition

In order to demonstrate this demo. Please follow the following steps:

- Create the following 2 folders:
  1. **C:\Temp**
  2. **C:\Temp2**

A race condition is a situation in programming where two or more threads access and manipulate shared data simultaneously, leading to unpredictable and incorrect results.

The root cause of the race condition in this code is the lack of synchronization primitive between the **`FileCreationThread`** and the **`FileMoveThread`**. Both threads operate on files located in the **`C:\Temp`** directory, but they do so concurrently. This unsynchronized access means that **`FileMoveThread`** might attempt to move a file that **`FileCreationThread`** is still writing to, or hasn't even created yet. While, **`FileCreationThread`** could potentially overwrite a file just as **`FileMoveThread`** is moving it. The result could be incomplete, missing, or corrupted files.

```c
#include <ntifs.h>
#include <ntstrsafe.h>

NTSTATUS FileCreationThread(PVOID Context) {
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

    return STATUS_SUCCESS;
}

NTSTATUS FileMoveThread(PVOID Context) {
    IO_STATUS_BLOCK ioStatusBlock;
    WCHAR oldFilePathBuffer[150], newFilePathBuffer[150];

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

    return STATUS_SUCCESS;
}

VOID UnloadDriver(IN PDRIVER_OBJECT DriverObject) {
    IoDeleteDevice(DriverObject->DeviceObject);
}

NTSTATUS DriverEntry(PDRIVER_OBJECT DriverObject, PUNICODE_STRING RegistryPath) {
    DriverObject->DriverUnload = UnloadDriver;

    PDEVICE_OBJECT DeviceObject = NULL;
    NTSTATUS status = IoCreateDevice(DriverObject, 0, NULL, FILE_DEVICE_UNKNOWN, FILE_DEVICE_SECURE_OPEN, FALSE, &DeviceObject);

    if (!NT_SUCCESS(status)) {
        return status;
    }

    HANDLE threadHandle1, threadHandle2;

    status = PsCreateSystemThread(&threadHandle1, (ACCESS_MASK)0, NULL, (PHANDLE)0, NULL, FileCreationThread, NULL);
    if (NT_SUCCESS(status)) {
        ZwClose(threadHandle1);
    }

    status = PsCreateSystemThread(&threadHandle2, (ACCESS_MASK)0, NULL, (PHANDLE)0, NULL, FileMoveThread, NULL);
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

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/a5f0c4ae-599b-4aa9-a1b2-246291193116)


Start the kernel driver:

```
sc.exe start TestDrv
```

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/55444380-3f6e-432a-9ef0-7d5b1c2646f9)


We can see that 1000 files were created in the **C:\Temp** folder.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/a6862167-6d6c-41e4-b3da-0a9a97c317cf)


However, the files have not been renamed and moved to the **C:\Temp2** folder.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/7240efb7-485e-4739-9295-4b9a2ba07e5e)


Now delete all the files from the **C:\Temp** folder.

# Code Sample - Binary Semaphore

In order to demonstrate this demo. Please follow the following steps:

- Create the following 2 folders:
  1. **C:\Temp**
  2. **C:\Temp2**

This code uses a Binary Semaphore for better control over file operations, helping avoid problems when two tasks happen at the same time. In the **`FileCreationThread`** function, this semaphore is grabbed first before starting to create and write files. This stops other tasks, like **`FileMoveThread`**, from using the files at the same time.

Once **`FileCreationThread`** is done with the files, it lets go of the semaphore. This tells **`FileMoveThread`** that it can now start moving the files. **`FileMoveThread`** waits for its turn to grab the semaphore. When it gets it, this means **`FileCreationThread`** is done, and it can safely move the files. After moving the files, **`FileMoveThread`** releases the semaphore too. 

```c
#include <ntifs.h>
#include <ntstrsafe.h>

// Define a structure to hold our synchronization resources.
typedef struct _SYNC_RESOURCES {
    KSEMAPHORE Semaphore;  // KSEMAPHORE object for synchronized access to the files.
} SYNC_RESOURCES, * PSYNC_RESOURCES;

NTSTATUS FileCreationThread(PVOID Context) {
    PSYNC_RESOURCES SyncRes = (PSYNC_RESOURCES)Context;
    UNICODE_STRING uniFilePath;
    WCHAR filePathBuffer[150];
    char dataToWrite[] = "Hello, World!\r\n";
    IO_STATUS_BLOCK ioStatusBlock;

    // Acquire the Semaphore
    KeWaitForSingleObject(&SyncRes->Semaphore, Executive, KernelMode, FALSE, NULL);

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

    // Release the Semaphore
    KeReleaseSemaphore(&SyncRes->Semaphore, 0, 1, FALSE);

    return STATUS_SUCCESS;
}

NTSTATUS FileMoveThread(PVOID Context) {
    PSYNC_RESOURCES SyncRes = (PSYNC_RESOURCES)Context;
    IO_STATUS_BLOCK ioStatusBlock;
    WCHAR oldFilePathBuffer[150], newFilePathBuffer[150];

    // Acquire the Semaphore
    KeWaitForSingleObject(&SyncRes->Semaphore, Executive, KernelMode, FALSE, NULL);

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

    // Release the Semaphore
    KeReleaseSemaphore(&SyncRes->Semaphore, 0, 1, FALSE);

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

    // Initialize the Semaphore
    KeInitializeSemaphore(&SyncRes->Semaphore, 1, 1); // Initial and Maximum count set to 1.

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

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/f9cb3251-d83e-4d9a-a100-9c22d44c93f8)


Start the kernel driver:

```
sc.exe start TestDrv
```

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/03ee1e4a-7832-438b-98cd-e261eed3434e)


We can see that 1000 files were created in the **C:\Temp** folder.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/83dcdc6c-c7f7-44ec-849f-5f4511b12e48)


However, we can also see that those 1000 files have now been renamed and moved to **C:\Temp2** folder.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/3a6bb1f3-e6cc-4004-9d4d-5914ba1d0003)


# Binary Semaphore - Visualization

This flow demonstrates how the semaphore provides exclusive access to the shared resources between the threads, ensuring orderly and non-overlapping operations.

```
Semaphore State
+----------------+
| Available (1)  |
+----------------+
       |
       |  FileCreationThread starts
       |
       V
+---------------------+                  +---------------------+
| FileCreationThread  |                  |   FileMoveThread    |
+---------------------+                  +---------------------+
       |                                             |
       |---> Tries to acquire semaphore ------------>|
       |                                             |
       |<----- Semaphore now locked (0) -------------|
       |                                             |
       |     Performs file creation and writing      |
       |                                             |
       |---> Finished, releases semaphore ---------->|
       |                                             |
       |<----- Semaphore now available (1) ----------|
       |                                             |
       |                                             |---> Waits, then tries to acquire semaphore
       |                                             |
       |                                             |<----- Semaphore now locked (0) -------
       |                                             |
       |                                             |       Performs file moving
       |                                             |
       |                                             |---> Finished, releases semaphore
       |                                             |
       |                                             |<----- Semaphore now available (1) ------
       V                                             V
```

# Semaphore - Debug Details

Link to this Memory Dump: https://mega.nz/file/3h0mhRZK#XDGpHNg-vr9xIJ1zAJ4MxVMob2UXP4MP40o2-3N3jjo

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/462fd68a-3e63-4bc8-8f0b-73e843a020d2)


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
Mex External 3.0.0.7172 Loaded!
PID           Address          Name
============= ================ ========================
0x4    0n4    ffffb381036eb040 System
0x20dc 0n8412 ffffb381045070c0 SystemSettingsBroker.exe
============= ================ ========================
PID           Address          Name

Warning! Zombie process(es) detected (not displayed). Count: 8 [zombie report]
```

We can for example switch to a different process context like **System.exe** and then examine this process by using the **`!mex.p`** command.

```
0: kd> !mex.p ffffb381036eb040
Name   Address                  Ses PID     Parent  PEB              Create Time                Mods Handle Thrd User Name
====== ======================== === ======= ======= ================ ========================== ==== ====== ==== =========================
System ffffb381036eb040 (E|K|O)   0 4 (0n4) 0 (0n0) 0000000000000000 12/23/2023 07:16:41.656 AM    0      0  252 WORKGROUP\WINDEV2305EVAL$

Memory Details:

    VM   Peak     Commit Size NPP Quota
    ==== ======== =========== =========
    4 MB 76.04 MB       40 KB     272 B

File Server Threads (Active threads running srv or srv2 driver)
Exective Work Queues

Show LPC Port information for process

Show Threads: Unique Stacks    !mex.listthreads (!lt) ffffb381036eb040    !process ffffb381036eb040 7
```

The **`!mex.us`** command allows us to display unique stacks within threads. First, we need to navigate to a specific process of interest. In this case, we are in the context of the **System.exe** process. With the **`!mex.us`** command we can also filter on a specific string, so it will only display the the call stacks that contains a specific keyword.

```
0: kd> !mex.us TestDriver
1 thread [stats]: ffffb38105506040
    fffff801362201c6 nt!KiSwapContext+0x76          
    fffff8013606c9d5 nt!KiSwapThread+0xab5          
    fffff8013606ebb7 nt!KiCommitThreadWait+0x137    
    fffff80136070ad6 nt!KeWaitForSingleObject+0x256 
    fffff80166971297 TestDriver!FileMoveThread+0x47 (C:\Users\User\source\repos\TestDriver\TestDriver\Driver.c @ 49)
    fffff80136107167 nt!PspSystemThreadStartup+0x57 
    fffff8013621bb94 nt!KiStartSystemThread+0x34    

Threads matching filter: 1 out of 252
```

Here is an example of a thread that is waiting on a Binary Semaphore. The semaphore being waited on has a limit of 1, meaning only one thread can access the associated resource at a time.

```
0: kd> !mex.t ffffb38105506040
Process                   Thread                       CID       UserTime KernelTime ContextSwitches Wait Reason    Time State
System (ffffb381036eb040) ffffb38105506040 (E|K|W|R|V) 4.16e4           0          0               1 Executive   47s.234 Waiting

WaitBlockList:
    Object           Type      Other Waiters Info
    ffffb3810a1c2f40 Semaphore             0 Limit: 1

# Child-SP         Return           Call Site                      Source
0 ffffed8bfb6beb60 fffff8013606c9d5 nt!KiSwapContext+0x76          
1 ffffed8bfb6beca0 fffff8013606ebb7 nt!KiSwapThread+0xab5          
2 ffffed8bfb6bedf0 fffff80136070ad6 nt!KiCommitThreadWait+0x137    
3 ffffed8bfb6beea0 fffff80166971297 nt!KeWaitForSingleObject+0x256 
4 ffffed8bfb6bf240 fffff80136107167 TestDriver!FileMoveThread+0x47 C:\Users\User\source\repos\TestDriver\TestDriver\Driver.c @ 49
5 ffffed8bfb6bf570 fffff8013621bb94 nt!PspSystemThreadStartup+0x57 
6 ffffed8bfb6bf5c0 0000000000000000 nt!KiStartSystemThread+0x34
```

This is the associated data structure of a **`KSEMAPHORE`**, which also indicates that there is one thread waiting on it.

```
0: kd> !mex.sem ffffb3810a1c2f40

dt -r1 nt!_KSEMAPHORE ffffb3810a1c2f40 () Recursive: [ -r1 -r2 -r ] Verbose Normal dt
==================================================================================
   +0x000 Header               : _DISPATCHER_HEADER
      +0x000 Lock                 : 0n524293 (0x80005)
      +0x000 LockNV               : 0n524293 (0x80005)
      +0x000 Type                 : 0x5 ''
      +0x001 Signalling           : 0 ''
      +0x002 Size                 : 0x8 ''
      +0x003 Reserved1            : 0 ''
      +0x000 TimerType            : 0x5 ''
      +0x001 TimerControlFlags    : 0 ''
      +0x001 Absolute             : 0y0
      +0x001 Wake                 : 0y0
      +0x001 EncodedTolerableDelay : 0y000000 (0)
      +0x002 Hand                 : 0x8 ''
      +0x003 TimerMiscFlags       : 0 ''
      +0x003 Index                : 0y000000 (0)
      +0x003 Inserted             : 0y0
      +0x003 Expired              : 0y0
      +0x000 Timer2Type           : 0x5 ''
      +0x001 Timer2Flags          : 0 ''
      +0x001 Timer2Inserted       : 0y0
      +0x001 Timer2Expiring       : 0y0
      +0x001 Timer2CancelPending  : 0y0
      +0x001 Timer2SetPending     : 0y0
      +0x001 Timer2Running        : 0y0
      +0x001 Timer2Disabled       : 0y0
      +0x001 Timer2ReservedFlags  : 0y00 (0n0)
      +0x002 Timer2ComponentId    : 0x8 ''
      +0x003 Timer2RelativeId     : 0 ''
      +0x000 QueueType            : 0x5 ''
      +0x001 QueueControlFlags    : 0 ''
      +0x001 Abandoned            : 0y0
      +0x001 DisableIncrement     : 0y0
      +0x001 QueueReservedControlFlags : 0y000000 (0)
      +0x002 QueueSize            : 0x8 ''
      +0x003 QueueReserved        : 0 ''
      +0x000 ThreadType           : 0x5 ''
      +0x001 ThreadReserved       : 0 ''
      +0x002 ThreadControlFlags   : 0x8 ''
      +0x002 CycleProfiling       : 0y0
      +0x002 CounterProfiling     : 0y0
      +0x002 GroupScheduling      : 0y0
      +0x002 AffinitySet          : 0y1
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
      +0x000 MutantType           : 0x5 ''
      +0x001 MutantSize           : 0 ''
      +0x002 DpcActive            : 0x8 ''
      +0x003 MutantReserved       : 0 ''
      +0x004 SignalState          : 0n0
      +0x008 WaitListHead         : _LIST_ENTRY [ 0xffffb381`05506180 - 0xffffb381`05506180 ] [EMPTY OR 1 ELEMENT]
   +0x018 Limit                : 0n1


Number of waiters: 1
```

The following thread is waiting on a synchronization event object:

```
0: kd> !mex.obj -waiters ffffb3810a1c2f40
Process Thread             Id CSwitches User Kernel State      Time Reason    Wait Function
======= ================ ==== ========= ==== ====== ======= ======= ========= ==============================
System  ffffb38105506040 16e4         1    0      0 Waiting 47s.234 Executive TestDriver!FileMoveThread+0x47
```
