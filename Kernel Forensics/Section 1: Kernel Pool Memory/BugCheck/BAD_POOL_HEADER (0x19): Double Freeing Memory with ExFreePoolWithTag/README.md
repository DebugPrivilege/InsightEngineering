# Summary

The **`BAD_POOL_HEADER`** bugcheck, with an error code of **`0x19`**, indicates that a problem has been detected with the Windows memory allocation. This bugcheck is usually caused by issues with the pool header's structure. The pool header is a small block of data that contains information about how the memory pool is to be used, and corruption of this header can lead to a system's bugcheck.

Link to the CrashDump: https://mega.nz/file/zg9mRLBJ#WBwHc-NdG3XhkZAuzIWn9xYdYkksqj2APjPHFbNlTK4

# Code Sample - Double Freeing Memory

Our sample code is causing this system bugcheck due to the double-free operation on the **`writeBuffer`**. When we free the same memory block twice, we end up corrupting the internal structures of the memory pool. 

```c
#include <ntddk.h>

NTSTATUS CreateRandomFile(UINT32 fileNumber, SIZE_T dataSize) {
    HANDLE fileHandle;
    OBJECT_ATTRIBUTES objAttr;
    IO_STATUS_BLOCK ioStatusBlock;
    NTSTATUS status;
    WCHAR fileNameBuffer[] = L"\\??\\C:\\Temp\\fileX.txt"; // X will be replaced
    PCHAR writeBuffer;
    UNICODE_STRING filePath;

    // Replace 'X' in fileNameBuffer with the actual file number
    fileNameBuffer[wcslen(fileNameBuffer) - 5] = L'0' + (WCHAR)fileNumber;

    // Initialize the file path
    RtlInitUnicodeString(&filePath, fileNameBuffer);

    // Initialize object attributes
    InitializeObjectAttributes(&objAttr, &filePath, OBJ_CASE_INSENSITIVE | OBJ_KERNEL_HANDLE, NULL, NULL);

    // Create the file
    status = ZwCreateFile(&fileHandle, GENERIC_WRITE, &objAttr, &ioStatusBlock, NULL, FILE_ATTRIBUTE_NORMAL, 0, FILE_OVERWRITE_IF, FILE_SYNCHRONOUS_IO_NONALERT, NULL, 0);

    if (!NT_SUCCESS(status)) {
        KdPrint(("Failed to create the file, status code: %x\n", status));
        return status;
    }

    // Allocate write buffer from paged pool
    writeBuffer = (PCHAR)ExAllocatePoolWithTag(PagedPool, dataSize, 'WrB1');
    if (writeBuffer == NULL) {
        KdPrint(("Failed to allocate memory for write buffer.\n"));
        ZwClose(fileHandle);
        return STATUS_INSUFFICIENT_RESOURCES;
    }

    // Fill the write buffer with some data (e.g., all 'A's)
    RtlFillMemory(writeBuffer, dataSize, 'A');

    // Write to the file
    status = ZwWriteFile(fileHandle, NULL, NULL, NULL, &ioStatusBlock, writeBuffer, dataSize, NULL, NULL);

    // Intentionally corrupt the pool freelist by freeing the same block twice.
    // This is very dangerous and should never be done in real-world code.
    ExFreePoolWithTag(writeBuffer, 'WrB1');
    ExFreePoolWithTag(writeBuffer, 'WrB1');

    ZwClose(fileHandle);

    return status;
}

VOID UnloadDriver(PDRIVER_OBJECT DriverObject) {
    KdPrint(("Driver Unloaded\n"));
}

NTSTATUS DriverEntry(PDRIVER_OBJECT DriverObject, PUNICODE_STRING RegistryPath) {
    DriverObject->DriverUnload = UnloadDriver;

    LARGE_INTEGER tickCount;
    UINT32 fileCount;
    SIZE_T dataSize;

    // Get system tick count for "random" seed
    KeQueryTickCount(&tickCount);

    // Create between 1-10 files
    fileCount = (tickCount.LowPart % 10) + 1;

    for (UINT32 i = 0; i < fileCount; ++i) {
        // Refresh tick count for another "random" seed
        KeQueryTickCount(&tickCount);

        // Generate a "random" data size between 1 and 10000 bytes
        dataSize = (tickCount.LowPart % 10000) + 1;

        CreateRandomFile(i, dataSize);
    }

    return STATUS_SUCCESS;
}
```

Start with loading the driver:
```
sc.exe create TestDriver type= kernel binPath=C:\Users\User\source\repos\Driver\x64\Release\Driver.sys
```

Start the driver:
```
sc start TestDriver
```

Once the driver has been loaded, it will start bugchecking the system:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/dfb74382-aaf8-4f8c-aee2-eaaa1ddfcc03)


# WinDbg Walk Through - Analyzing CrashDump

Load the Crashdump in WinDbg:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/33c8c090-9ab4-4f54-b324-2832d0464388)


Start with loading the **MEX** extension:

```
0: kd> .load mex
Mex External 3.0.0.7172 Loaded!
```

Let's start with the **`!di`** command which stands for **dump information**. This command retrieves information about the machine from which the memory dump originated. It's especially handy when working with multiple memory dumps, as it helps distinguish the analysis related to each specific dump. This is an improved version of the vertarget command.

```
0: kd> !di
Computer Name:  Not Found
Windows 10 Kernel Version 22621 MP (4 procs) Free x64
Product: WinNt, suite: TerminalServer SingleUserTS
Edition build lab: 22621.1928.amd64fre.ni_release_svc_prod3.230622-0951
Kernel base = 0xfffff804`6f000000 PsLoadedModuleList = 0xfffff804`6fc130e0
Debug session time: Thu Sep 21 08:52:49.284 2023 (UTC - 7:00)
System Uptime: 0 days 0:02:18.209
SystemManufacturer = Microsoft Corporation
SystemProductName = Virtual Machine
Processor: 11th Gen Intel(R) Core(TM) i7-1185G7 @ 3.00GHz
Bugcheck: 19 (22, FFFFBD06D46E0000, 1, 0)
KernelMode Dump Path: C:\Users\User\Desktop\Dumps\GitHub\MEMORY.DMP
Share Path: \\WINDEV2305EVAL\C$\Users\User\Desktop\Dumps\GitHub\MEMORY.DMP
```

The **`!analyze`** command in WinDbg performs an automated analysis of the dump or the current state of the system to identify the root cause of the crash or issue. It provides detailed information, including the bug check code, the involved processes, and so on. 

```
0: kd> !analyze
*******************************************************************************
*                                                                             *
*                        Bugcheck Analysis                                    *
*                                                                             *
*******************************************************************************

BAD_POOL_HEADER (19)
The pool is already corrupt at the time of the current request.
This may or may not be due to the caller.
The internal pool links must be walked to figure out a possible cause of
the problem, and then special pool applied to the suspect tags or the driver
verifier to a suspect driver.
Arguments:
Arg1: 0000000000000022, 
Arg2: ffffbd06d46e0000
Arg3: 0000000000000001
Arg4: 0000000000000000

Debugging Details:
------------------


BUGCHECK_CODE:  19

BUGCHECK_P1: 22

BUGCHECK_P2: ffffbd06d46e0000

BUGCHECK_P3: 1

BUGCHECK_P4: 0

PROCESS_NAME:  System

SYMBOL_NAME:  Driver+1170

MODULE_NAME: Driver

IMAGE_NAME:  Driver.sys

FAILURE_BUCKET_ID:  0x19_22_Driver!unknown_function

FAILURE_ID_HASH:  {13274a29-c6c5-4e24-cb5a-0988e703af11}

Followup:     MachineOwner
---------
```

Before we are going further, let's search online and find more information about this bugcheck. See: https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/bug-check-0x19--bad-pool-header

It hints that when the first argument is **`0x22`** that we are dealing with the following issue:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/69afb420-1b92-4ae1-a8d0-d6779bcbd510)



Let's gather more information about this specific thread. We can use the **`!t`** command to analyze threads. This command is short for **threads** and it includes the state and the CPU on which it's operating.

This output is showing us that this thread **`(ffffa98fb6fa1040)`** on **CPU 0** that was crashing:

```
0: kd> !mex.t
Process                   Thread                       CID       UserTime KernelTime ContextSwitches Wait Reason Time State
System (ffffa98fb0aec040) ffffa98fb6fa1040 (E|K|W|R|V) 4.1bf0           0      141ms            1269 Executive      0 Running on processor 0

Priority:
    Current Base Decrement ForegroundBoost IO Page
    13      12   0         0               0  5

# Child-SP         Return           Call Site
0 fffffa86b7eaee88 fffff8046f4d2b58 nt!KeBugCheckEx
1 fffffa86b7eaee90 fffff8046faad3ac nt!ExpRemoveTagForBigPages+0x1b8878
2 fffffa86b7eaef00 fffff804789a1170 nt!ExFreePoolWithTag+0x29c
3 fffffa86b7eaef90 fffff804789a1205 Driver+0x1170
4 fffffa86b7eaf090 fffff804789a151b Driver+0x1205
5 fffffa86b7eaf0c0 fffff804789a1450 Driver+0x151b
6 fffffa86b7eaf100 fffff8046f7654d4 Driver+0x1450
7 fffffa86b7eaf130 fffff8046f765b23 nt!PnpCallDriverEntry+0x54
8 fffffa86b7eaf180 fffff8046f764727 nt!IopLoadDriver+0x523
9 fffffa86b7eaf340 fffff8046f2b8bd5 nt!IopLoadUnloadDriver+0x57
a fffffa86b7eaf380 fffff8046f212667 nt!ExpWorkerThread+0x155
b fffffa86b7eaf570 fffff8046f4370a4 nt!PspSystemThreadStartup+0x57
c fffffa86b7eaf5c0 0000000000000000 nt!KiStartSystemThread+0x34
```

When running the **`!pool`** command on the address **`ffffbd06d46e0000`**, we find that this memory belongs to the Paged pool and is marked as "Free." However, the garbage pool tags and the "Unknown" owning component are signs that something isn't right. This information hints that the memory could be corrupted.

```
0: kd> !pool ffffbd06d46e0000
Pool page ffffbd06d46e0000 region is Paged pool
*ffffbd06d46dffe0 size: 8020 previous size:    0  (Free)      *>.g.
		Owning component : Unknown (update pooltag.txt)
```

Looking at the **`nt!_POOL_HEADER`** structure, we see that all the fields in the pool header at address **`ffffbd06d46e0000`** contain the value **0x41414141** or variations of it. These values are the ASCII representation of 'AAAA'.

```
0: kd> dt nt!_POOL_HEADER ffffbd06d46e0000
   +0x000 PreviousSize     : 0y01000001 (0x41)
   +0x000 PoolIndex        : 0y01000001 (0x41)
   +0x002 BlockSize        : 0y01000001 (0x41)
   +0x002 PoolType         : 0y01000001 (0x41)
   +0x000 Ulong1           : 0x41414141
   +0x004 PoolTag          : 0x41414141
   +0x008 ProcessBilled    : 0x41414141`41414141 _EPROCESS
   +0x008 AllocatorBackTraceIndex : 0x4141
   +0x00a PoolTagHash      : 0x4141
```

The size of the pool allocation is listed as **0x8020** in hexadecimal format in the **`!pool`** output, but the **`BlockSize`** field in the **`nt!_POOL_HEADER`** structure doesn't align with each other. This mismatch further confirms that the pool header is corrupted. 

The **`!poolval`** command is used to validate the integrity of a pool block at a given address. It's a command to check for inconsistencies or corruption in the pool header. This output confirms that the pool block is corrupted. 

```
0: kd> !poolval ffffbd06d46e0000
Pool page ffffbd06d46e0000 region is Paged pool

Validating Pool headers for pool page: ffffbd06d46e0000

Pool page [ ffffbd06d46e0000 ] is INVALID.

Analyzing linked list...
[ ffffbd06d46e0000 ]: invalid previous size [ 0x41 ] should be [ 0x0 ]


Scanning for single bit errors...

None found
```

Let's dump content of pool right before the corrupted region:

```
0: kd> dc ffffbd06d46e0000-100 l0x240/4
ffffbd06`d46dff00  00000000 00000000 00000000 00000000  ................
ffffbd06`d46dff10  00000000 00000000 00000000 00000000  ................
ffffbd06`d46dff20  00000000 00000000 00000000 00000000  ................
ffffbd06`d46dff30  00000000 00000000 00000000 00000000  ................
ffffbd06`d46dff40  00000000 00000000 00000000 00000000  ................
ffffbd06`d46dff50  00000000 00000000 00000000 00000000  ................
ffffbd06`d46dff60  00000000 00000000 00000000 00000000  ................
ffffbd06`d46dff70  00000000 00000000 00000000 00000000  ................
ffffbd06`d46dff80  00000000 00000000 00000000 00000000  ................
ffffbd06`d46dff90  00000000 00000000 00000000 00000000  ................
ffffbd06`d46dffa0  00000000 00000000 00000000 00000000  ................
ffffbd06`d46dffb0  00000000 00000000 00000000 00000000  ................
ffffbd06`d46dffc0  00000000 00000000 00000000 00000000  ................
ffffbd06`d46dffd0  00000000 00000000 00000000 00100000  ................
ffffbd06`d46dffe0  86e5f5d4 8d67903e 00000000 00000000  ....>.g.........
ffffbd06`d46dfff0  00000000 00000000 d3e3d809 ffffbd06  ................
ffffbd06`d46e0000  41414141 41414141 41414141 41414141  AAAAAAAAAAAAAAAA
ffffbd06`d46e0010  41414141 41414141 41414141 41414141  AAAAAAAAAAAAAAAA
ffffbd06`d46e0020  41414141 41414141 41414141 41414141  AAAAAAAAAAAAAAAA
ffffbd06`d46e0030  41414141 41414141 41414141 41414141  AAAAAAAAAAAAAAAA
ffffbd06`d46e0040  41414141 41414141 41414141 41414141  AAAAAAAAAAAAAAAA
ffffbd06`d46e0050  41414141 41414141 41414141 41414141  AAAAAAAAAAAAAAAA
ffffbd06`d46e0060  41414141 41414141 41414141 41414141  AAAAAAAAAAAAAAAA
ffffbd06`d46e0070  41414141 41414141 41414141 41414141  AAAAAAAAAAAAAAAA
ffffbd06`d46e0080  41414141 41414141 41414141 41414141  AAAAAAAAAAAAAAAA
ffffbd06`d46e0090  41414141 41414141 41414141 41414141  AAAAAAAAAAAAAAAA
ffffbd06`d46e00a0  41414141 41414141 41414141 41414141  AAAAAAAAAAAAAAAA
ffffbd06`d46e00b0  41414141 41414141 41414141 41414141  AAAAAAAAAAAAAAAA
ffffbd06`d46e00c0  41414141 41414141 41414141 41414141  AAAAAAAAAAAAAAAA
ffffbd06`d46e00d0  41414141 41414141 41414141 41414141  AAAAAAAAAAAAAAAA
ffffbd06`d46e00e0  41414141 41414141 41414141 41414141  AAAAAAAAAAAAAAAA
ffffbd06`d46e00f0  41414141 41414141 41414141 41414141  AAAAAAAAAAAAAAAA
ffffbd06`d46e0100  41414141 41414141 41414141 41414141  AAAAAAAAAAAAAAAA
ffffbd06`d46e0110  41414141 41414141 41414141 41414141  AAAAAAAAAAAAAAAA
ffffbd06`d46e0120  41414141 41414141 41414141 41414141  AAAAAAAAAAAAAAAA
ffffbd06`d46e0130  41414141 41414141 41414141 41414141  AAAAAAAAAAAAAAAA
```

The pool header is the first 8-bytes which are filled with **41** and represents 'A'. The first 4-bytes is **41414141** which we saw as well in the previous output, indicating that there is a pool corruption.

Let's navigate again to the crashing thread, which we can use with the **`!crash`** command. Based on this call stack, we can conclude that the crash is likely due to the pool corruption caused by the custom driver at or near the **`Driver+0x1170`** address.

```
0: kd> !crash
Dump Info
============================================
Computer Name:  Not Found
Windows 10 Kernel Version 22621 MP (4 procs) Free x64
Product: WinNt, suite: TerminalServer SingleUserTS
Edition build lab: 22621.1928.amd64fre.ni_release_svc_prod3.230622-0951
Kernel base = 0xfffff804`6f000000 PsLoadedModuleList = 0xfffff804`6fc130e0
Debug session time: Thu Sep 21 08:52:49.284 2023 (UTC - 7:00)
System Uptime: 0 days 0:02:18.209
SystemManufacturer = Microsoft Corporation
SystemProductName = Virtual Machine
Processor: 11th Gen Intel(R) Core(TM) i7-1185G7 @ 3.00GHz
Bugcheck: 19 (22, FFFFBD06D46E0000, 1, 0)
KernelMode Dump Path: C:\Users\User\Desktop\Dumps\GitHub\MEMORY.DMP
Share Path: \\WINDEV2305EVAL\C$\Users\User\Desktop\Dumps\GitHub\MEMORY.DMP


Bugcheck details
============================================
Bugcheck code 00000019
Arguments 00000000`00000022 ffffbd06`d46e0000 00000000`00000001 00000000`00000000

Crashing Stack
============================================
 # Child-SP          RetAddr               Call Site
00 fffffa86`b7eaee88 fffff804`6f4d2b58     nt!KeBugCheckEx
01 fffffa86`b7eaee90 fffff804`6faad3ac     nt!ExpRemoveTagForBigPages+0x1b8878
02 fffffa86`b7eaef00 fffff804`789a1170     nt!ExFreePoolWithTag+0x29c
03 fffffa86`b7eaef90 fffff804`789a1205     Driver+0x1170
04 fffffa86`b7eaf090 fffff804`789a151b     Driver+0x1205
05 fffffa86`b7eaf0c0 fffff804`789a1450     Driver+0x151b
06 fffffa86`b7eaf100 fffff804`6f7654d4     Driver+0x1450
07 fffffa86`b7eaf130 fffff804`6f765b23     nt!PnpCallDriverEntry+0x54
08 fffffa86`b7eaf180 fffff804`6f764727     nt!IopLoadDriver+0x523
09 fffffa86`b7eaf340 fffff804`6f2b8bd5     nt!IopLoadUnloadDriver+0x57
0a fffffa86`b7eaf380 fffff804`6f212667     nt!ExpWorkerThread+0x155
0b fffffa86`b7eaf570 fffff804`6f4370a4     nt!PspSystemThreadStartup+0x57
0c fffffa86`b7eaf5c0 00000000`00000000     nt!KiStartSystemThread+0x34
```



The output that is shown is a disassembly of the instructions leading up to the address **`Driver+0x1170`**, which is where the crash occurred according to the call stack. 

```
0: kd> ub Driver+0x1170
Driver+0x114c:
fffff804`789a114c ff15de0e0000    call    qword ptr [Driver+0x2030 (fffff804`789a2030)]
fffff804`789a1152 ba31427257      mov     edx,57724231h
fffff804`789a1157 488bcb          mov     rcx,rbx
fffff804`789a115a 8bf8            mov     edi,eax
fffff804`789a115c ff15fe0e0000    call    qword ptr [Driver+0x2060 (fffff804`789a2060)]
fffff804`789a1162 ba31427257      mov     edx,57724231h
fffff804`789a1167 488bcb          mov     rcx,rbx
fffff804`789a116a ff15f00e0000    call    qword ptr [Driver+0x2060 (fffff804`789a2060)]
```

The **`dps`** command displays memory as pointer-sized values with symbols. Here we are asking WinDbg to display the content at the memory address **`fffff804'789a2030`** and interpret it symbolically. This value is the address of the **`nt!ZwWriteFile`** function in the Windows kernel.

This means that when the code at **`Driver+0x114c`** executed the instruction **`call qword ptr [Driver+0x2030]`**, it was making a call to the **`nt!ZwWriteFile`** function.

```
0: kd> dps fffff804`789a2030 l1
fffff804`789a2030  fffff804`6f42e550 nt!ZwWriteFile
```

Here we can see that the driver was making a call to the **`nt!ExFreePoolWithTag`** function. The **`ExFreePoolWithTag`** function is used to free a block of pool memory in the Windows Kernel that was previously allocated.

```
0: kd> dps fffff804`789a2060 l1
fffff804`789a2060  fffff804`6faad110 nt!ExFreePoolWithTag
```

This function takes two parameters:

```
void ExFreePoolWithTag(
  PVOID P,
  ULONG Tag
);
```

The instruction **`mov edx, 57724231h`** copies the hexadecimal value **57724231** into the EDX register. 

```
fffff804`789a1152 ba31427257      mov     edx,57724231h
```

The **`.formats`** command in WinDbg is used to display the value of an expression in multiple formats. It's a helpful command for understanding what a particular value represents in various number systems or formats.

In this case, **`WrB1`** is very likely the Pool Tag. **`57724231h`** is the hexadecimal representation of the characters **`WrB1`**, and a Pool Tag usually has a size of **4-bytes**.

```
0: kd> .formats 57724231h
Evaluate expression:
  Hex:     00000000`57724231
  Decimal: 1467105841
  Decimal (unsigned) : 1467105841
  Octal:   0000000000012734441061
  Binary:  00000000 00000000 00000000 00000000 01010111 01110010 01000010 00110001
  Chars:   ....WrB1
  Time:    Tue Jun 28 02:24:01 2016
  Float:   low 2.66366e+014 high 0
  Double:  7.24847e-315
```


Here we can see that **`ExFreePoolWithTag`** has been called twice, which led to the double freeing memory issue.

```
0: kd> ub Driver+0x1170
Driver+0x114c:
fffff804`789a114c ff15de0e0000    call    qword ptr [Driver+0x2030 (fffff804`789a2030)]
fffff804`789a1152 ba31427257      mov     edx,57724231h
fffff804`789a1157 488bcb          mov     rcx,rbx
fffff804`789a115a 8bf8            mov     edi,eax
fffff804`789a115c ff15fe0e0000    call    qword ptr [Driver+0x2060 (fffff804`789a2060)] --> First call to ExFreePoolWithTag
fffff804`789a1162 ba31427257      mov     edx,57724231h --> This is the Pool Tag
fffff804`789a1167 488bcb          mov     rcx,rbx
fffff804`789a116a ff15f00e0000    call    qword ptr [Driver+0x2060 (fffff804`789a2060)] --> Second call to ExFreePoolWithTag
```

The **`lmvm`** command in WinDbg stands for "List Module Verbose Mode." It provides detailed information about a specific loaded module (usually a driver or a DLL) in the debugging session. It will display a variety of information, such as the module's base address, size, time stamp, and more.

Here is an example of the driver that was responsible for the system crash:

```
0: kd> lmvm Driver
Browse full module list
start             end                 module name
fffff804`789a0000 fffff804`789a7000   Driver     (no symbols)           
    Loaded symbol image file: Driver.sys
    Image path: \??\C:\Users\User\source\repos\Driver\x64\Release\Driver.sys
    Image name: Driver.sys
    Browse all global symbols  functions  data
    Timestamp:        Thu Sep 21 08:52:21 2023 (650C66B5)
    CheckSum:         0000E766
    ImageSize:        00007000
    Translations:     0000.04b0 0000.04e4 0409.04b0 0409.04e4
    Information from resource tables:
```

A response to a customer for example could look like:

*We have investigated the issue related to the system crash(es) caused by the installed driver. Our analysis points to a memory corruption issue within the driver's code, specifically related to improper pool memory management. Our investigation revealed that the driver is freeing the same pool memory block twice, which is a dangerous operation and a common cause of pool corruption.*

