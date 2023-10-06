# Summary

The **`KERNEL_MODE_HEAP_CORRUPTION`** bug check has a value of **`0x0000013A`**. This bug check indicates that the kernel mode heap manager has detected corruption in a heap.

# Code Sample - Buffer Overflow

The root cause of the crash is the misuse of the **RtlZeroMemory** function. The function is instructed to zero out 1000 bytes, while **`anotherBuffer`** is only allocated 500 bytes. This mismatch leads to a buffer overflow, causing memory corruption and system instability.

The **RtlZeroMemory** function is a commonly used kernel-mode function in Windows drivers to set a block of memory to zeros. It's commonly used to initialize a newly allocated memory buffer to ensure that it doesn't contain any leftover or 'garbage' data.

```c
#include <ntddk.h>

NTSTATUS CreateRandomFile(UINT32 fileNumber, SIZE_T dataSize) {
    HANDLE fileHandle;
    OBJECT_ATTRIBUTES objAttr;
    IO_STATUS_BLOCK ioStatusBlock;
    NTSTATUS status;
    WCHAR fileNameBuffer[] = L"\\??\\C:\\Temp\\fileX.txt";
    PCHAR writeBuffer;
    PCHAR anotherBuffer;
    UNICODE_STRING filePath;

    fileNameBuffer[wcslen(fileNameBuffer) - 5] = L'0' + (WCHAR)fileNumber;
    RtlInitUnicodeString(&filePath, fileNameBuffer);
    InitializeObjectAttributes(&objAttr, &filePath, OBJ_CASE_INSENSITIVE | OBJ_KERNEL_HANDLE, NULL, NULL);

    status = ZwCreateFile(&fileHandle, GENERIC_WRITE, &objAttr, &ioStatusBlock, NULL, FILE_ATTRIBUTE_NORMAL, 0, FILE_OVERWRITE_IF, FILE_SYNCHRONOUS_IO_NONALERT, NULL, 0);
    if (!NT_SUCCESS(status)) {
        KdPrint(("Failed to create the file, status code: %x\n", status));
        return status;
    }

    writeBuffer = (PCHAR)ExAllocatePool2(POOL_FLAG_PAGED, dataSize, 'WrB1');
    if (writeBuffer == NULL) {
        KdPrint(("Failed to allocate memory for write buffer.\n"));
        ZwClose(fileHandle);
        return STATUS_INSUFFICIENT_RESOURCES;
    }

    anotherBuffer = (PCHAR)ExAllocatePool2(POOL_FLAG_PAGED, dataSize / 2, 'AnB1');  // Allocate a smaller buffer
    if (anotherBuffer == NULL) {
        KdPrint(("Failed to allocate memory for another buffer.\n"));
        ExFreePoolWithTag(writeBuffer, 'WrB1');
        ZwClose(fileHandle);
        return STATUS_INSUFFICIENT_RESOURCES;
    }

    RtlZeroMemory(anotherBuffer, dataSize);  // Misuse: Zeroing out more memory than anotherBuffer holds

    // Continue with the write operation as usual
    status = ZwWriteFile(fileHandle, NULL, NULL, NULL, &ioStatusBlock, writeBuffer, dataSize, NULL, NULL);

    ExFreePoolWithTag(writeBuffer, 'WrB1');
    ExFreePoolWithTag(anotherBuffer, 'AnB1');
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

    KeQueryTickCount(&tickCount);
    fileCount = (tickCount.LowPart % 10) + 1;

    for (UINT32 i = 0; i < fileCount; ++i) {
        KeQueryTickCount(&tickCount);
        dataSize = (tickCount.LowPart % 10000) + 1;
        CreateRandomFile(i, dataSize);
    }

    return STATUS_SUCCESS;
}
```

Here we can see that our driver has caused a bugcheck of the system:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/558954b5-a269-42ce-9860-1e4aa1994af0)


# WinDbg Walk Through - Analyzing CrashDump

Load the CrashDump in WinDbg:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/7df719f3-dc09-4e3c-a579-1b219e48c15a)


Start with loading the MEX extension:

```
1: kd> .load mex
Mex External 3.0.0.7172 Loaded!
```

Let's start with the **`!di`** command which stands for display information. This command retrieves information about the machine from which the memory dump originated. It's especially handy when working with multiple memory dumps, as it helps distinguish the analysis related to each specific dump. This is an improved version of the **`vertarget`** command.

```
1: kd> !di
Computer Name:  Not Found
Windows 10 Kernel Version 22621 MP (4 procs) Free x64
Product: WinNt, suite: TerminalServer SingleUserTS
Edition build lab: 22621.1928.amd64fre.ni_release_svc_prod3.230622-0951
Kernel base = 0xfffff805`41c00000 PsLoadedModuleList = 0xfffff805`428130e0
Debug session time: Wed Oct  4 09:04:44.418 2023 (UTC - 7:00)
System Uptime: 0 days 0:07:18.672
SystemManufacturer = Microsoft Corporation
SystemProductName = Virtual Machine
Processor: 11th Gen Intel(R) Core(TM) i7-1185G7 @ 3.00GHz
Bugcheck: 13A (12, FFFFE2808C200140, FFFFE2809ABEE000, 0)
KernelMode Dump Path: C:\Windows\MEMORY.DMP
Share Path: \\WINDEV2305EVAL\ADMIN$\MEMORY.DMP
```

Let's run the **`!analyze -v`** command to quickly triage the issue as it provides a quick and detailed overview of the state of the system at the time of the crash. It automates many aspects of crash dump analysis, providing a quick way to get valuable information.

```
1: kd> !analyze -v
*******************************************************************************
*                                                                             *
*                        Bugcheck Analysis                                    *
*                                                                             *
*******************************************************************************

KERNEL_MODE_HEAP_CORRUPTION (13a)
The kernel mode heap manager has detected corruption in a heap.
Arguments:
Arg1: 0000000000000012, Type of corruption detected
Arg2: ffffe2808c200140, Address of the heap that reported the corruption
Arg3: ffffe2809abee000, Address at which the corruption was detected
Arg4: 0000000000000000

Debugging Details:
------------------


KEY_VALUES_STRING: 1

    Key  : Analysis.CPU.mSec
    Value: 3155

    Key  : Analysis.Elapsed.mSec
    Value: 3649

    Key  : Analysis.IO.Other.Mb
    Value: 0

    Key  : Analysis.IO.Read.Mb
    Value: 1

    Key  : Analysis.IO.Write.Mb
    Value: 6

    Key  : Analysis.Init.CPU.mSec
    Value: 6921

    Key  : Analysis.Init.Elapsed.mSec
    Value: 177491

    Key  : Analysis.Memory.CommitPeak.Mb
    Value: 151

    Key  : Bugcheck.Code.KiBugCheckData
    Value: 0x13a

    Key  : Bugcheck.Code.LegacyAPI
    Value: 0x13a

    Key  : Dump.Attributes.AsUlong
    Value: 800

    Key  : Failure.Bucket
    Value: 0x13a_12_Driver!CreateRandomFile

    Key  : Failure.Hash
    Value: {ea91b8db-b8d2-b485-3fb8-7116e18cbc08}

    Key  : Hypervisor.Enlightenments.Value
    Value: 1113060

    Key  : Hypervisor.Enlightenments.ValueHex
    Value: 10fbe4

    Key  : Hypervisor.Flags.AnyHypervisorPresent
    Value: 1

    Key  : Hypervisor.Flags.ApicEnlightened
    Value: 0

    Key  : Hypervisor.Flags.ApicVirtualizationAvailable
    Value: 0

    Key  : Hypervisor.Flags.AsyncMemoryHint
    Value: 0

    Key  : Hypervisor.Flags.CoreSchedulerRequested
    Value: 0

    Key  : Hypervisor.Flags.CpuManager
    Value: 0

    Key  : Hypervisor.Flags.DeprecateAutoEoi
    Value: 1

    Key  : Hypervisor.Flags.DynamicCpuDisabled
    Value: 1

    Key  : Hypervisor.Flags.Epf
    Value: 0

    Key  : Hypervisor.Flags.ExtendedProcessorMasks
    Value: 1

    Key  : Hypervisor.Flags.HardwareMbecAvailable
    Value: 1

    Key  : Hypervisor.Flags.MaxBankNumber
    Value: 0

    Key  : Hypervisor.Flags.MemoryZeroingControl
    Value: 0

    Key  : Hypervisor.Flags.NoExtendedRangeFlush
    Value: 0

    Key  : Hypervisor.Flags.NoNonArchCoreSharing
    Value: 0

    Key  : Hypervisor.Flags.Phase0InitDone
    Value: 1

    Key  : Hypervisor.Flags.PowerSchedulerQos
    Value: 0

    Key  : Hypervisor.Flags.RootScheduler
    Value: 0

    Key  : Hypervisor.Flags.SynicAvailable
    Value: 1

    Key  : Hypervisor.Flags.UseQpcBias
    Value: 0

    Key  : Hypervisor.Flags.Value
    Value: 659708

    Key  : Hypervisor.Flags.ValueHex
    Value: a10fc

    Key  : Hypervisor.Flags.VpAssistPage
    Value: 1

    Key  : Hypervisor.Flags.VsmAvailable
    Value: 1

    Key  : Hypervisor.RootFlags.AccessStats
    Value: 0

    Key  : Hypervisor.RootFlags.CrashdumpEnlightened
    Value: 0

    Key  : Hypervisor.RootFlags.CreateVirtualProcessor
    Value: 0

    Key  : Hypervisor.RootFlags.DisableHyperthreading
    Value: 0

    Key  : Hypervisor.RootFlags.HostTimelineSync
    Value: 0

    Key  : Hypervisor.RootFlags.HypervisorDebuggingEnabled
    Value: 0

    Key  : Hypervisor.RootFlags.IsHyperV
    Value: 0

    Key  : Hypervisor.RootFlags.LivedumpEnlightened
    Value: 0

    Key  : Hypervisor.RootFlags.MapDeviceInterrupt
    Value: 0

    Key  : Hypervisor.RootFlags.MceEnlightened
    Value: 0

    Key  : Hypervisor.RootFlags.Nested
    Value: 0

    Key  : Hypervisor.RootFlags.StartLogicalProcessor
    Value: 0

    Key  : Hypervisor.RootFlags.Value
    Value: 0

    Key  : Hypervisor.RootFlags.ValueHex
    Value: 0

    Key  : SecureKernel.HalpHvciEnabled
    Value: 0

    Key  : WER.OS.Branch
    Value: ni_release_svc_prod3

    Key  : WER.OS.Version
    Value: 10.0.22621.1928


BUGCHECK_CODE:  13a

BUGCHECK_P1: 12

BUGCHECK_P2: ffffe2808c200140

BUGCHECK_P3: ffffe2809abee000

BUGCHECK_P4: 0

FILE_IN_CAB:  MEMORY.DMP

VIRTUAL_MACHINE:  HyperV

DUMP_FILE_ATTRIBUTES: 0x800

CORRUPTING_POOL_ADDRESS:  ffffe2809abee000 Paged pool

BLACKBOXBSD: 1 (!blackboxbsd)


BLACKBOXNTFS: 1 (!blackboxntfs)


BLACKBOXPNP: 1 (!blackboxpnp)


BLACKBOXWINLOGON: 1

PROCESS_NAME:  System

STACK_TEXT:  
fffffd0c`b5e9ec48 fffff805`421c88b8     : 00000000`0000013a 00000000`00000012 ffffe280`8c200140 ffffe280`9abee000 : nt!KeBugCheckEx
fffffd0c`b5e9ec50 fffff805`421c8918     : 00000000`00000012 fffffd0c`b5e9ee21 ffffe280`8c200140 ffffe280`8c200000 : nt!RtlpHeapHandleError+0x40
fffffd0c`b5e9ec90 fffff805`421c8535     : ffffe280`9ac04fe8 ffffe280`9ac04fe0 ffff9b0e`000001fb ffffe280`8c2002c0 : nt!RtlpHpHeapHandleError+0x58
fffffd0c`b5e9ecc0 fffff805`4209a8dc     : fffffd0c`b5e9f008 fffffd0c`b5e9eff8 00000000`00000002 00000000`00000000 : nt!RtlpLogHeapFailure+0x45
fffffd0c`b5e9ecf0 fffff805`41eae7be     : 00000000`000000ff fffffd0c`00000fe0 00000000`00000100 00000000`00001f8b : nt!RtlpHpVsContextAllocateInternal+0x24b4ac
fffffd0c`b5e9ed60 fffff805`41eae07f     : 00000000`00000002 ffffffff`8000404c 00000000`416e4231 00000000`00000000 : nt!ExAllocateHeapPool+0x70e
fffffd0c`b5e9ee80 fffff805`426ad78d     : 00000000`00000100 00000000`00000001 00000000`00000401 fffff805`00000000 : nt!ExpAllocatePoolWithTagFromNode+0x5f
fffffd0c`b5e9eed0 fffff805`62c8113b     : 00000000`00001f8b 00000000`00000000 ffff9b0e`00000003 00000000`00000fe0 : nt!ExAllocatePool2+0xdd
fffffd0c`b5e9ef80 fffff805`62c81245     : 00000000`00000005 fffff805`62c832d8 fffff780`00000320 ffffffff`8000404c : Driver!CreateRandomFile+0x13b [C:\Users\User\source\repos\Driver\Driver\Driver.c @ 30] 
fffffd0c`b5e9f090 fffff805`62c8155b     : 00000000`00000000 ffff9b0e`81603000 ffff9b0e`81603000 ffff9b0e`84c608b0 : Driver!DriverEntry+0x61 [C:\Users\User\source\repos\Driver\Driver\Driver.c @ 67] 
fffffd0c`b5e9f0c0 fffff805`62c81490     : ffff9b0e`81603000 fffffd0c`b5e9f280 ffff9b0e`851024fd ffff9b0e`85372840 : Driver!FxDriverEntryWorker+0xbf [minkernel\wdf\framework\kmdf\src\dynamic\stub\stub.cpp @ 360] 
fffffd0c`b5e9f100 fffff805`423654d4     : ffff9b0e`81603000 00000000`00000000 ffff9b0e`85372840 fffff805`41f06b80 : Driver!FxDriverEntry+0x20 [minkernel\wdf\framework\kmdf\src\dynamic\stub\stub.cpp @ 249] 
fffffd0c`b5e9f130 fffff805`42365b23     : ffff9b0e`81603000 00000000`00000000 00000000`00000000 ffffe280`9b6141d0 : nt!PnpCallDriverEntry+0x54
fffffd0c`b5e9f180 fffff805`42364727     : 00000000`00000000 00000000`00000000 ffffab81`6702b200 fffff805`42949ac0 : nt!IopLoadDriver+0x523
fffffd0c`b5e9f340 fffff805`41eb8bd5     : ffff9b0e`00000000 ffffffff`8000404c ffff9b0e`83720040 fffff805`00000000 : nt!IopLoadUnloadDriver+0x57
fffffd0c`b5e9f380 fffff805`41e12667     : ffff9b0e`83720040 00000000`0000096c ffff9b0e`83720040 fffff805`41eb8a80 : nt!ExpWorkerThread+0x155
fffffd0c`b5e9f570 fffff805`420370a4     : ffffab81`66392180 ffff9b0e`83720040 fffff805`41e12610 38d30008`0038d2f0 : nt!PspSystemThreadStartup+0x57
fffffd0c`b5e9f5c0 00000000`00000000     : fffffd0c`b5ea0000 fffffd0c`b5e99000 00000000`00000000 00000000`00000000 : nt!KiStartSystemThread+0x34


FAULTING_SOURCE_LINE:  C:\Users\User\source\repos\Driver\Driver\Driver.c

FAULTING_SOURCE_FILE:  C:\Users\User\source\repos\Driver\Driver\Driver.c

FAULTING_SOURCE_LINE_NUMBER:  30

FAULTING_SOURCE_CODE:  
    26:         ZwClose(fileHandle);
    27:         return STATUS_INSUFFICIENT_RESOURCES;
    28:     }
    29: 
>   30:     anotherBuffer = (PCHAR)ExAllocatePool2(POOL_FLAG_PAGED, dataSize / 2, 'AnB1');  // Allocate a smaller buffer
    31:     if (anotherBuffer == NULL) {
    32:         KdPrint(("Failed to allocate memory for another buffer.\n"));
    33:         ExFreePoolWithTag(writeBuffer, 'WrB1');
    34:         ZwClose(fileHandle);
    35:         return STATUS_INSUFFICIENT_RESOURCES;


SYMBOL_NAME:  Driver!CreateRandomFile+13b

MODULE_NAME: Driver

IMAGE_NAME:  Driver.sys

STACK_COMMAND:  .cxr; .ecxr ; kb

BUCKET_ID_FUNC_OFFSET:  13b

FAILURE_BUCKET_ID:  0x13a_12_Driver!CreateRandomFile

OS_VERSION:  10.0.22621.1928

BUILDLAB_STR:  ni_release_svc_prod3

OSPLATFORM_TYPE:  x64

OSNAME:  Windows 10

FAILURE_ID_HASH:  {ea91b8db-b8d2-b485-3fb8-7116e18cbc08}

Followup:     MachineOwner
---------
```

The first **`Arg1`** indicates the parameter **`0x12`** which is the following:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/76957bf6-4637-4fef-b874-bcb4981e31d5)




This thread that is running for **15ms** on **CPU 1** is the crashing thread:

```
0: kd> !t
Process                   Thread                       CID       UserTime KernelTime ContextSwitches Wait Reason Time State
System (ffff9b0e7d8c3040) ffff9b0e83720040 (E|K|W|R|V) 4.520            0      141ms            1471 Executive   15ms Running on processor 1

Priority:
    Current Base Decrement ForegroundBoost IO Page
    13      12   0         0               0  5

 # Child-SP         Return           Call Site                                   Source
 0 fffffd0cb5e9ec48 fffff805421c88b8 nt!KeBugCheckEx                             
 1 fffffd0cb5e9ec50 fffff805421c8918 nt!RtlpHeapHandleError+0x40                 
 2 fffffd0cb5e9ec90 fffff805421c8535 nt!RtlpHpHeapHandleError+0x58               
 3 fffffd0cb5e9ecc0 fffff8054209a8dc nt!RtlpLogHeapFailure+0x45                  
 4 fffffd0cb5e9ecf0 fffff80541eae7be nt!RtlpHpVsContextAllocateInternal+0x24b4ac 
 5 fffffd0cb5e9ed60 fffff80541eae07f nt!ExAllocateHeapPool+0x70e                 
 6 fffffd0cb5e9ee80 fffff805426ad78d nt!ExpAllocatePoolWithTagFromNode+0x5f      
 7 fffffd0cb5e9eed0 fffff80562c8113b nt!ExAllocatePool2+0xdd                     
 8 fffffd0cb5e9ef80 fffff80562c81245 Driver!CreateRandomFile+0x13b               C:\Users\User\source\repos\Driver\Driver\Driver.c @ 30
 9 fffffd0cb5e9f090 fffff80562c8155b Driver!DriverEntry+0x61                     C:\Users\User\source\repos\Driver\Driver\Driver.c @ 67
 a fffffd0cb5e9f0c0 fffff80562c81490 Driver!FxDriverEntryWorker+0xbf             minkernel\wdf\framework\kmdf\src\dynamic\stub\stub.cpp @ 360
 b fffffd0cb5e9f100 fffff805423654d4 Driver!FxDriverEntry+0x20                   minkernel\wdf\framework\kmdf\src\dynamic\stub\stub.cpp @ 249
 c fffffd0cb5e9f130 fffff80542365b23 nt!PnpCallDriverEntry+0x54                  
 d fffffd0cb5e9f180 fffff80542364727 nt!IopLoadDriver+0x523                      
 e fffffd0cb5e9f340 fffff80541eb8bd5 nt!IopLoadUnloadDriver+0x57                 
 f fffffd0cb5e9f380 fffff80541e12667 nt!ExpWorkerThread+0x155                    
10 fffffd0cb5e9f570 fffff805420370a4 nt!PspSystemThreadStartup+0x57              
11 fffffd0cb5e9f5c0 0000000000000000 nt!KiStartSystemThread+0x34
```

The presence of an allocated block with a specific tag **`MmSt`**, followed by a free block with garbage pool tags, in the context of a **`KERNEL_MODE_HEAP_CORRUPTION`** bugcheck, suggests that the heap has likely been corrupted. A free block with garbage pool tags (such as .r.") is often a red flag. Normally, free blocks should have recognizable tags or should be zeroed out.

```
1: kd> !pool ffffe2809abee000
Pool page ffffe2809abee000 region is Paged pool
*ffffe2809abee000 size:  880 previous size:    0  (Allocated) *MmSt
		Pooltag MmSt : Mm section object prototype ptes, Binary : nt!mm
 ffffe2809abee880 size:  760 previous size:    0  (Free)       .r."
```

The **`!poolval`** output strongly suggests that the pool page at the address **`ffffe2809abee000`** is corrupt, specifically mentioning that its headers are invalid.

```
1: kd> !poolval ffffe2809abee000
Pool page ffffe2809abee000 region is Paged pool

Validating Pool headers for pool page: ffffe2809abee000

Pool page [ ffffe2809abee000 ] is INVALID.

Analyzing linked list...


Scanning for single bit errors...

None found
```

The **`!pte`** command output shows the Page Table Entry (PTE) details for the virtual address **`fffe2809abee000`** which looks valid though.

```
0: kd> !pte ffffe2809abee000
                                           VA ffffe2809abee000
PXE at FFFFCFE7F3F9FE28    PPE at FFFFCFE7F3FC5010    PDE at FFFFCFE7F8A026A8    PTE at FFFFCFF1404D5F70
contains 0A00000107400863  contains 0A00000107401863  contains 0A000001184EF863  contains 80000001660FF963
pfn 107400    ---DA--KWEV  pfn 107401    ---DA--KWEV  pfn 1184ef    ---DA--KWEV  pfn 1660ff    -G-DA--KW-V
```

This output shows the state of a **`_LIST_ENTRY`** structure located at the memory address **`ffffe2809abee000`**. Given that the **`Flink`** is pointing to a strange value and **`Blink`** is null, this **`_LIST_ENTRY`** is in an inconsistent or corrupted state. 

```
1: kd> dt nt!_LIST_ENTRY ffffe2809abee000
 [ 0x74536d4d`03880000 - 0x00000000`00000000 ]
   +0x000 Flink            : 0x74536d4d`03880000 _LIST_ENTRY
   +0x008 Blink            : (null)
```

The value **`0x74536d4d03880000`** we're looking at the "Chars" section from the **`.formats`** command looks like a Pool tag. The characters **`tSmM`** could be read in reverse as **`MmSt`**, which would be the tag for a memory pool allocated by the Memory Manager. However, finding this value in the Flink of a **`_LIST_ENTRY`** structure is unusual and is not expected behavior from my own experience.

```
1: kd> .formats 0x74536d4d`03880000
Evaluate expression:
  Hex:     74536d4d`03880000
  Decimal: 8382163509005778944
  Decimal (unsigned) : 8382163509005778944
  Octal:   0721233324640342000000
  Binary:  01110100 01010011 01101101 01001101 00000011 10001000 00000000 00000000
  Chars:   tSmM.... --> This is the Pool Tag
  Time:    Fri Jan  7 20:15:00.577 28163 (UTC - 7:00)
  Float:   low 7.99336e-037 high 6.70039e+031
  Double:  2.22547e+252
```
