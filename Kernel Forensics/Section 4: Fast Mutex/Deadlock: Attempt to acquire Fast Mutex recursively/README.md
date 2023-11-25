# Description

Fast mutexes cannot be acquired recursively. If a thread already holding a fast mutex attempts to acquire it again, the thread will enter a deadlock. This information is available at: https://learn.microsoft.com/en-us/windows-hardware/drivers/kernel/fast-mutexes-and-guarded-mutexes. 

However, it's important to verify this theory in practice as well. 

Link to this Memory Dump: https://mega.nz/file/zhFH0KiB#4P67pzgCySdsgYy2Qrn7jb6Nc-1qx72lPdWPGIcN3tQ

```
+-----------------------------------------------------+
|              System State Overview                  |
+-----------------------------------------------------+ 
|                                                     |
|  +-----------+      +-------------------------+     |
|  | Thread A  |      |       Fast Mutex        |     |
|  +-----------+      +-------------------------+     |
|       |                      |                      |
|       | 1. Acquire Mutex     |                      |
|       |-----> (Lock acquired)|                      |
|       |                      |                      |
|       | 2. Attempt to        |                      |
|       |    Acquire Again     |                      |
|       |-----> (Already held) |                      |
|       |                      |                      |
|       | 3. Deadlock          |                      |
|       |-----> (Waiting for release that never comes)|
|       |                      |                      |
+-----------------------------------------------------+
```

# Code Sample - Attempting to acquire a Fast Mutex recursively

In order to demonstrate this demo. Please follow the following steps:

- Create the following 2 folders:
  1. **C:\Temp**
  2. **C:\Temp2**

The **`FileCreationThread`** thread attempts to acquire a Fast Mutex recursively, which is not supported and will result in a deadlock.

```c
#include <ntifs.h>
#include <ntstrsafe.h>

// Define a structure to hold our synchronization resources.
typedef struct _SYNC_RESOURCES {
    FAST_MUTEX FileMutex;  // FAST_MUTEX object for synchronized access to the files.
} SYNC_RESOURCES, * PSYNC_RESOURCES;

NTSTATUS FileCreationThread(PVOID Context) {
    PSYNC_RESOURCES SyncRes = (PSYNC_RESOURCES)Context;
    UNICODE_STRING uniFilePath;
    WCHAR filePathBuffer[150];
    char dataToWrite[] = "Hello, World!\r\n";
    IO_STATUS_BLOCK ioStatusBlock;

    // Acquire the FAST_MUTEX
    ExAcquireFastMutex(&SyncRes->FileMutex);

    // Attempt to acquire the FAST_MUTEX recursively
    ExAcquireFastMutex(&SyncRes->FileMutex);

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

    // The code will not reach this point due to the deadlock
    ExReleaseFastMutex(&SyncRes->FileMutex);

    return STATUS_SUCCESS;
}

NTSTATUS FileMoveThread(PVOID Context) {
    PSYNC_RESOURCES SyncRes = (PSYNC_RESOURCES)Context;
    IO_STATUS_BLOCK ioStatusBlock;
    WCHAR oldFilePathBuffer[150], newFilePathBuffer[150];

    // Acquire the FAST_MUTEX
    ExAcquireFastMutex(&SyncRes->FileMutex);

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

    // Release the FAST_MUTEX
    ExReleaseFastMutex(&SyncRes->FileMutex);

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

    // Initialize the FAST_MUTEX
    ExInitializeFastMutex(&SyncRes->FileMutex);

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

Start loading the kernel driver:

```
sc.exe create TestDrv type= kernel binPath=C:\Users\Admin\source\repos\TestDriver\x64\Release\TestDriver.sys
```

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/aa6586dd-fa4b-44d0-ae08-fd941e9367cd)


Start the kernel driver:

```
sc.exe start TestDrv
```

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/92c56d2e-e5bf-4bbe-ab67-961280e823e9)


Since the thread is in a deadlock, no files will be created in the **C:\Temp** folder.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/c1241e74-6d20-4f9e-997c-b7a47973e4d9)


If you're using Windows 11, you can initiate the creation of a full kernel memory dump through Task Manager. This is done by targeting the System.exe process:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/96fafcb4-03be-4120-8b1f-709df94f173d)



# WinDbg Walk Through - Analysis

Load the Memory Dump in WinDbg:

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/173b2003-8b23-47f6-94c4-c2857f5da2a6)

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

The **`!WrFastMutex`** displays detailed information about threads that are waiting for a Fast Mutex. There are two threads that are currently in a waiting state for a Fast Mutex.

```
0: kd> !WrFastMutex
Process PID Thread            Id CSwitches User Kernel State      Time Reason      Waiting On                      Wait Function
======= === ================ === ========= ==== ====== ======= ======= =========== =============================== ===========================
System    4 ffff9709ac20e040 dd8         1    0      0 Waiting 43s.296 WrFastMutex Owning Thread: ffff9709ac20e040 nt!ExAcquireFastMutex+0x122
System    4 ffff9709ab278040 c80         1    0      0 Waiting 43s.296 WrFastMutex Owning Thread: ffff9709ac20e040 nt!ExAcquireFastMutex+0x122
Count: 2
```

This thread has been waiting 43s.296 on a fast mutex owned by System thread **`ffff9709ac20e040`**

```
0: kd> !mex.t ffff9709ac20e040
Process                   Thread                       CID       UserTime KernelTime ContextSwitches Wait Reason    Time State
System (ffff9709a54e7040) ffff9709ac20e040 (E|K|W|R|V) 4.dd8            0          0               1 WrFastMutex 43s.296 Waiting

WaitBlockList:
    Object           Type                 Other Waiters Info
    ffff9709ac3d4f38 SynchronizationEvent             1 Fast Mutex Owner: ffff9709ac3d4f28

# Child-SP         Return           Call Site
0 ffffbb0cb276ec00 fffff80327c6d645 nt!KiSwapContext+0x76
1 ffffbb0cb276ed40 fffff80327c6f827 nt!KiSwapThread+0xab5
2 ffffbb0cb276ee90 fffff80327c71746 nt!KiCommitThreadWait+0x137
3 ffffbb0cb276ef40 fffff80327c2acb6 nt!KeWaitForSingleObject+0x256
4 ffffbb0cb276f2e0 fffff80327ced832 nt!ExpAcquireFastMutexContended+0x7a
5 ffffbb0cb276f320 fffff8034291113d nt!ExAcquireFastMutex+0x122
6 ffffbb0cb276f350 fffff80327d07287 TestDriver+0x113d
7 ffffbb0cb276f570 fffff80327e1b8e4 nt!PspSystemThreadStartup+0x57
8 ffffbb0cb276f5c0 0000000000000000 nt!KiStartSystemThread+0x34
```

The command **`.frame /r 0n5`** sets the context to the 5th frame in the stack trace and displays the register values at that moment. By examining the register values and the current instruction, we can get insights into what the function is doing at that point.

This is a snapshot of the CPU state at the 5th frame of the stack, which is within the **`nt!ExAcquireFastMutex`** function. The **`ExAcquireFastMutex`** function in Windows kernel is used to acquire a fast mutex.

```
0: kd> .frame /r 0n5
05 ffffbb0c`b276f320 fffff803`4291113d     nt!ExAcquireFastMutex+0x122
rax=0000000000000000 rbx=ffff9709ac20e740 rcx=0000000000000000
rdx=0000000000000000 rsi=0000000000000001 rdi=ffff9709ac3d4f20
rip=fffff80327ced832 rsp=ffffbb0cb276f320 rbp=ffffbb0cb276f450
 r8=0000000000000000  r9=0000000000000000 r10=0000000000000000
r11=0000000000000000 r12=00000000000001de r13=ffff9709acc60040
r14=ffff9709a54e7040 r15=ffffbf805ed51000
iopl=0         nv up di pl nz na pe nc
cs=0000  ss=0000  ds=0000  es=0000  fs=0000  gs=0000             efl=00000000
nt!ExAcquireFastMutex+0x122:
fffff803`27ced832 eb9d            jmp     nt!ExAcquireFastMutex+0xc1 (fffff803`27ced7d1)
```

The **`Owner`** field is the thread that last acquired the mutex. The **`Contention`** value is 2, indicating that there have been 2 instances where threads attempted to acquire this mutex but had to wait because it was already owned by another thread.

```
0: kd> dt nt!_FAST_MUTEX ffff9709ac3d4f20
   +0x000 Count            : 0n8
   +0x008 Owner            : 0xffff9709`ac20e040 Void
   +0x010 Contention       : 2
   +0x018 Event            : _KEVENT
   +0x030 OldIrql          : 0
```
