# Summary

The **`SYSTEM_THREAD_EXCEPTION_NOT_HANDLED`** error, commonly represented by the bug check code **`0x7E`**, occurs when a system thread generates an exception that the error handler does not catch. This is typically a sign of a critical system problem, often related to corrupt or incompatible drivers or faulty hardware.

Link to this CrashDump: https://mega.nz/file/egl0USIS#uoO4-w_40Xm9Wi-BdPRfstVyp6JFNNwcNIoasWDyJRA

# Code Sample - Buffer Overflow

This code has a buffer overflow. It allocates a buffer of size **`dataSize`** but then fills it with twice as much data **`(dataSize * 2)`** using **RtlFillMemory**. This can lead to memory corruption and system instability. The **RtlFillMemory** function is used to fill a block of memory with a specific value. It's commonly used in drivers or other kernel-mode code to initialize or set a chunk of memory.

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
    writeBuffer = (PCHAR)ExAllocatePool2(POOL_FLAG_PAGED, dataSize, 'WrB1');
    if (writeBuffer == NULL) {
        KdPrint(("Failed to allocate memory for write buffer.\n"));
        ZwClose(fileHandle);
        return STATUS_INSUFFICIENT_RESOURCES;
    }

    // Fill the write buffer with some data (e.g., all 'A's)
    // Corrupt memory by filling more than allocated size
    RtlFillMemory(writeBuffer, dataSize * 2, 'A');

    // Write to the file
    status = ZwWriteFile(fileHandle, NULL, NULL, NULL, &ioStatusBlock, writeBuffer, dataSize, NULL, NULL);

    // Clean up
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

When we load this kernel driver, it will start the bugchecking the system. Compiling this code and loading this kernel driver will not always guarantee that we'll get the same bugcheck code. It could be different as well.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/f082066d-f332-4157-b172-757399625b25)


# WinDbg Walk Through - Analyzing CrashDump

Load the CrashDump in WinDbg:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/fc87a9f6-0395-4e6c-a749-10bf0d2a7d9b)


Start with loading the **MEX** extension:

```
3: kd> .load mex
Mex External 3.0.0.7172 Loaded!
```

Let's start with the **`!di`** command which stands for display information. This command retrieves information about the machine from which the memory dump originated. It's especially handy when working with multiple memory dumps, as it helps distinguish the analysis related to each specific dump. This is an improved version of the **`vertarget`** command.

```
3: kd> !di
Computer Name:  Not Found
Windows 10 Kernel Version 22621 MP (4 procs) Free x64
Product: WinNt, suite: TerminalServer SingleUserTS
Edition build lab: 22621.1928.amd64fre.ni_release_svc_prod3.230622-0951
Kernel base = 0xfffff807`66000000 PsLoadedModuleList = 0xfffff807`66c130e0
Debug session time: Fri Sep 29 11:46:06.702 2023 (UTC - 7:00)
System Uptime: 0 days 0:10:16.931
SystemManufacturer = Microsoft Corporation
SystemProductName = Virtual Machine
Processor: 11th Gen Intel(R) Core(TM) i7-1185G7 @ 3.00GHz
Bugcheck: 7E (FFFFFFFFC0000005, FFFFF8076624F3D4, FFFFA1088A4AEBF8, FFFFA1088A4AE410)
KernelMode Dump Path: C:\Windows\MEMORY_0x7E (2).DMP
Share Path: \\WINDEV2305EVAL\ADMIN$\MEMORY_0x7E (2).DMP
```

Let's run the **`!analyze -v`** command to quickly triage the issue as it provides a quick and detailed overview of the state of the system at the time of the crash. It automates many aspects of crash dump analysis, providing a quick way to get valuable information.

```
3: kd> !analyze -v
*******************************************************************************
*                                                                             *
*                        Bugcheck Analysis                                    *
*                                                                             *
*******************************************************************************

SYSTEM_THREAD_EXCEPTION_NOT_HANDLED (7e)
This is a very common BugCheck.  Usually the exception address pinpoints
the driver/function that caused the problem.  Always note this address
as well as the link date of the driver/image that contains this address.
Arguments:
Arg1: ffffffffc0000005, The exception code that was not handled
Arg2: fffff8076624f3d4, The address that the exception occurred at
Arg3: ffffa1088a4aebf8, Exception Record Address
Arg4: ffffa1088a4ae410, Context Record Address

Debugging Details:
------------------


KEY_VALUES_STRING: 1

    Key  : AV.Fault
    Value: Read

    Key  : Analysis.CPU.mSec
    Value: 2577

    Key  : Analysis.Elapsed.mSec
    Value: 3181

    Key  : Analysis.IO.Other.Mb
    Value: 0

    Key  : Analysis.IO.Read.Mb
    Value: 1

    Key  : Analysis.IO.Write.Mb
    Value: 6

    Key  : Analysis.Init.CPU.mSec
    Value: 2311

    Key  : Analysis.Init.Elapsed.mSec
    Value: 157099

    Key  : Analysis.Memory.CommitPeak.Mb
    Value: 151

    Key  : Bugcheck.Code.KiBugCheckData
    Value: 0x7e

    Key  : Bugcheck.Code.LegacyAPI
    Value: 0x7e

    Key  : Dump.Attributes.AsUlong
    Value: 800

    Key  : Failure.Bucket
    Value: AV_EvilDriver!unknown_function

    Key  : Failure.Hash
    Value: {0a02cf11-80e3-568b-892e-59d7b590ecbf}

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


BUGCHECK_CODE:  7e

BUGCHECK_P1: ffffffffc0000005

BUGCHECK_P2: fffff8076624f3d4

BUGCHECK_P3: ffffa1088a4aebf8

BUGCHECK_P4: ffffa1088a4ae410

FILE_IN_CAB:  MEMORY_0x7E (2).DMP

VIRTUAL_MACHINE:  HyperV

DUMP_FILE_ATTRIBUTES: 0x800

EXCEPTION_RECORD:  ffffa1088a4aebf8 -- (.exr 0xffffa1088a4aebf8)
ExceptionAddress: fffff8076624f3d4 (nt!RtlpHpVsFreeChunkInsert+0x00000000000001b4)
   ExceptionCode: c0000005 (Access violation)
  ExceptionFlags: 00000000
NumberParameters: 2
   Parameter[0]: 0000000000000000
   Parameter[1]: ffffffffffffffff
Attempt to read from address ffffffffffffffff

CONTEXT:  ffffa1088a4ae410 -- (.cxr 0xffffa1088a4ae410)
rax=4141414141414139 rbx=0000000000006000 rcx=0000000002530001
rdx=4141414141414141 rsi=ffffe009a68dc000 rdi=ffffe009a68dffe0
rip=fffff8076624f3d4 rsp=ffffa1088a4aee30 rbp=0000000000000000
 r8=3be6192c0436d050  r9=0000000000000000 r10=ffffe0099a6002d0
r11=ffffe009a68dc000 r12=ffffe009a68dfb00 r13=ffffe0099a6002c0
r14=0000000000000001 r15=ffffe0099a6002c0
iopl=0         nv up ei pl nz na po nc
cs=0010  ss=0018  ds=002b  es=002b  fs=0053  gs=002b             efl=00050206
nt!RtlpHpVsFreeChunkInsert+0x1b4:
fffff807`6624f3d4 3300            xor     eax,dword ptr [rax] ds:002b:41414141`41414139=????????
Resetting default scope

BLACKBOXBSD: 1 (!blackboxbsd)


BLACKBOXNTFS: 1 (!blackboxntfs)


BLACKBOXPNP: 1 (!blackboxpnp)


BLACKBOXWINLOGON: 1

PROCESS_NAME:  System

READ_ADDRESS:  ffffffffffffffff 

ERROR_CODE: (NTSTATUS) 0xc0000005 - The instruction at 0x%p referenced memory at 0x%p. The memory could not be %s.

EXCEPTION_CODE_STR:  c0000005

EXCEPTION_PARAMETER1:  0000000000000000

EXCEPTION_PARAMETER2:  ffffffffffffffff

EXCEPTION_STR:  0xc0000005

STACK_TEXT:  
ffffa108`8a4aee30 fffff807`662ad6b1     : 00000000`00000000 00000000`00000000 ffffe009`a68dc000 00000000`00000000 : nt!RtlpHpVsFreeChunkInsert+0x1b4
ffffa108`8a4aee60 fffff807`66aad2b0     : ffffe009`00000000 00000000`00000401 00000000`000002a1 00000000`00000000 : nt!RtlpHpFreeHeap+0x421
ffffa108`8a4aef00 fffff807`84da115b     : 00000000`57724231 ffffe009`a68e0000 00000000`00000401 00000000`00002510 : nt!ExFreePoolWithTag+0x1a0
ffffa108`8a4aef90 fffff807`84da11ed     : 00000000`00000003 fffff807`84da32d8 fffff780`00000320 00000000`00000000 : EvilDriver+0x115b
ffffa108`8a4af090 fffff807`84da150b     : 00000000`00000000 ffff8008`e14b6000 ffff8008`e14b6000 ffff8008`e40022f0 : EvilDriver+0x11ed
ffffa108`8a4af0c0 fffff807`84da1440     : ffff8008`e14b6000 ffffa108`8a4af280 ffff8008`e4be9e3d ffff8008`e1b91e50 : EvilDriver+0x150b
ffffa108`8a4af100 fffff807`667654d4     : ffff8008`e14b6000 00000000`00000000 ffff8008`e1b91e50 fffff807`66306b80 : EvilDriver+0x1440
ffffa108`8a4af130 fffff807`66765b23     : ffff8008`e14b6000 00000000`00000000 00000000`00000000 ffffe009`a81e5650 : nt!PnpCallDriverEntry+0x54
ffffa108`8a4af180 fffff807`66764727     : 00000000`00000000 00000000`00000000 ffffe009`a452ce00 fffff807`66d49ac0 : nt!IopLoadDriver+0x523
ffffa108`8a4af340 fffff807`662b8bd5     : ffff8008`00000000 ffffffff`800017c4 ffff8008`e4604040 fffff807`00000000 : nt!IopLoadUnloadDriver+0x57
ffffa108`8a4af380 fffff807`66212667     : ffff8008`e4604040 00000000`00000b08 ffff8008`e4604040 fffff807`662b8a80 : nt!ExpWorkerThread+0x155
ffffa108`8a4af570 fffff807`664370a4     : ffffcb80`a5ddb180 ffff8008`e4604040 fffff807`66212610 ff0374fe`ff0374fe : nt!PspSystemThreadStartup+0x57
ffffa108`8a4af5c0 00000000`00000000     : ffffa108`8a4b0000 ffffa108`8a4a9000 00000000`00000000 00000000`00000000 : nt!KiStartSystemThread+0x34


SYMBOL_NAME:  EvilDriver+115b

MODULE_NAME: EvilDriver

IMAGE_NAME:  EvilDriver.sys

STACK_COMMAND:  .cxr 0xffffa1088a4ae410 ; kb

BUCKET_ID_FUNC_OFFSET:  115b

FAILURE_BUCKET_ID:  AV_EvilDriver!unknown_function

OS_VERSION:  10.0.22621.1928

BUILDLAB_STR:  ni_release_svc_prod3

OSPLATFORM_TYPE:  x64

OSNAME:  Windows 10

FAILURE_ID_HASH:  {0a02cf11-80e3-568b-892e-59d7b590ecbf}

Followup:     MachineOwner
---------
```

The **`Arg1: ffffffffc0000005`** is the exception code that was not handled. The code **`0xC0000005`** corresponds to an "Access Violation" error. Access violations are a type of exception that usually occurs when code tries to read from or write to an address that is either not allocated or is protected.

```
3: kd> !error ffffffffc0000005
Error code: (NTSTATUS) 0xc0000005 (3221225477) - The instruction at 0x%p referenced memory at 0x%p. The memory could not be %s.
```

An exception record is a data structure that contains information about an exception (or fault) that has occurred in a program. An exception record is represented by the **`EXCEPTION_RECORD`** structure. This structure holds various pieces of information that describe the exception. 

The **`.exr`** WinDbg command displays the contents of an exception record at a specified address. The exception occurred within a function **`(nt!RtlpHpVsFreeChunkInsert)`** related to heap management. We also can see that the driver attempted to read from an invalid memory address **`(ffffffffffffffff)`**, which is a strong sign of memory corruption or an uninitialized pointer.

```
3: kd> .exr 0xffffa1088a4aebf8
ExceptionAddress: fffff8076624f3d4 (nt!RtlpHpVsFreeChunkInsert+0x00000000000001b4)
   ExceptionCode: c0000005 (Access violation)
  ExceptionFlags: 00000000
NumberParameters: 2
   Parameter[0]: 0000000000000000
   Parameter[1]: ffffffffffffffff
Attempt to read from address ffffffffffffffff
```

The **`.cxr`** command sets the context to the state saved in a specific context record. This allows us to examine the state of the CPU registers at the time the exception occurred. **`nt!RtlpHpVsFreeChunkInsert`** is the function where the exception occurred. The exception occurs in a system function related to heap management, which could indicate that the problem may be with how memory is being managed or accessed.

```
3: kd> .cxr 0xffffa1088a4ae410
rax=4141414141414139 rbx=0000000000006000 rcx=0000000002530001
rdx=4141414141414141 rsi=ffffe009a68dc000 rdi=ffffe009a68dffe0
rip=fffff8076624f3d4 rsp=ffffa1088a4aee30 rbp=0000000000000000
 r8=3be6192c0436d050  r9=0000000000000000 r10=ffffe0099a6002d0
r11=ffffe009a68dc000 r12=ffffe009a68dfb00 r13=ffffe0099a6002c0
r14=0000000000000001 r15=ffffe0099a6002c0
iopl=0         nv up ei pl nz na po nc
cs=0010  ss=0018  ds=002b  es=002b  fs=0053  gs=002b             efl=00050206
nt!RtlpHpVsFreeChunkInsert+0x1b4:
fffff807`6624f3d4 3300            xor     eax,dword ptr [rax] ds:002b:41414141`41414139=????????
```

The command **`.frame /r 8`** sets the context to the **8th** frame in the stack trace and displays the register values at that moment. By examining the register values and the current instruction, we can get insights into what the function is doing at that point.

This is a snapshot of the CPU state at the 8th frame of the stack, which is within the **`nt!RtlpHpVsFreeChunkInsert`** function.

```
3: kd> .frame /r 0n8
08 ffffa108`8a4aee30 fffff807`662ad6b1     nt!RtlpHpVsFreeChunkInsert+0x1b4
rax=4141414141414139 rbx=0000000000006000 rcx=0000000002530001
rdx=4141414141414141 rsi=ffffe009a68dc000 rdi=ffffe009a68dffe0
rip=fffff8076624f3d4 rsp=ffffa1088a4aee30 rbp=0000000000000000
 r8=3be6192c0436d050  r9=0000000000000000 r10=ffffe0099a6002d0
r11=ffffe009a68dc000 r12=ffffe009a68dfb00 r13=ffffe0099a6002c0
r14=0000000000000001 r15=ffffe0099a6002c0
iopl=0         nv up ei ng nz na po nc
cs=0010  ss=0018  ds=002b  es=002b  fs=0053  gs=002b             efl=00040286
nt!RtlpHpVsFreeChunkInsert+0x1b4:
fffff807`6624f3d4 3300            xor     eax,dword ptr [rax] ds:002b:41414141`41414139=????????
```

The **R15** register contains the memory address **`ffffe0099a6002c0`** which is pointing to a **`_HEAP_VS_CONTEXT`** structure. The **`_HEAP_VS_CONTEXT`** data structure is used for managing the Variable-Size (VS) blocks within a heap in the Windows kernel. This structure contains various fields that hold information about the heap's internal state.

```
3: kd> dt ntkrnlmp!_HEAP_VS_CONTEXT ffffe0099a6002c0 -r1
   +0x000 Lock             : 1
   +0x008 LockType         : 0 ( HeapLockPaged )
   +0x010 FreeChunkTree    : _RTL_RB_TREE
      +0x000 Root             : 0xffffe009`a1432f28 _RTL_BALANCED_NODE
      +0x008 Encoded          : 0y0
      +0x008 Min              : 0xffffe009`9ae09fc8 _RTL_BALANCED_NODE
   +0x020 SubsegmentList   : _LIST_ENTRY [ 0x00000000`008022e0 - 0x00000000`3cedc2e0 ]
      +0x000 Flink            : 0x00000000`008022e0 _LIST_ENTRY --> Not in the ffff prefixed range
      +0x008 Blink            : 0x00000000`3cedc2e0 _LIST_ENTRY --> Not in the ffff prefixed range
   +0x030 TotalCommittedUnits : 0x46a0
   +0x038 FreeCommittedUnits : 0xb
   +0x040 DelayFreeContext : _HEAP_VS_DELAY_FREE_CONTEXT
      +0x000 ListHead         : _SLIST_HEADER
   +0x080 BackendCtx       : 0x00000000`00000380 Void
   +0x088 Callbacks        : _HEAP_SUBALLOCATOR_CALLBACKS
      +0x000 Allocate         : 0x3be60122`f8626a50
      +0x008 Free             : 0x3be60122`f879ced0
      +0x010 Commit           : 0x3be60122`f87a88e0
      +0x018 Decommit         : 0x3be60122`f879fe70
      +0x020 ExtendContext    : 0
   +0x0b0 Config           : _RTL_HP_VS_CONFIG
      +0x000 Flags            : <unnamed-tag>
   +0x0b4 Flags            : 0
```

The **`Flink`** and **`Blink`** pointers have non-zero values, but they don't seem to point to kernel memory (not in the **ffff** prefixed range). This is unusual and could be a sign of heap corruption.

```
3: kd> dx @$heap_context = (ntkrnlmp!_HEAP_VS_CONTEXT*) 0xffffe0099a6002c0
@$heap_context = (ntkrnlmp!_HEAP_VS_CONTEXT*) 0xffffe0099a6002c0                 : 0xffffe0099a6002c0 [Type: _HEAP_VS_CONTEXT *]
    [+0x000] Lock             : 0x1 [Type: unsigned __int64]
    [+0x008] LockType         : HeapLockPaged (0) [Type: _RTLP_HP_LOCK_TYPE]
    [+0x010] FreeChunkTree    [Type: _RTL_RB_TREE]
    [+0x020] SubsegmentList   [Type: _LIST_ENTRY]
    [+0x030] TotalCommittedUnits : 0x46a0 [Type: unsigned __int64]
    [+0x038] FreeCommittedUnits : 0xb [Type: unsigned __int64]
    [+0x040] DelayFreeContext [Type: _HEAP_VS_DELAY_FREE_CONTEXT]
    [+0x080] BackendCtx       : 0x380 [Type: void *]
    [+0x088] Callbacks        [Type: _HEAP_SUBALLOCATOR_CALLBACKS]
    [+0x0b0] Config           [Type: _RTL_HP_VS_CONFIG]
    [+0x0b4] Flags            : 0x0 [Type: unsigned long]
```

The value for **`TotalCommittedUnits`** is **`0x46a0`**

```
3: kd> dx @$heap_context->TotalCommittedUnits
@$heap_context->TotalCommittedUnits : 0x46a0 [Type: unsigned __int64]
```

While the value for **`FreeCommittedUnits`** is only **`0xb`**. This could indicate that almost all of the heap's memory is being used.

```
3: kd> dx @$heap_context->FreeCommittedUnits
@$heap_context->FreeCommittedUnits : 0xb [Type: unsigned __int64]
```

Let's get to the current context of our process. In this case, we are in the context of the **System** process.

```
0: kd> !p
Name   Address                  Ses PID     Parent  PEB              Create Time                Mods Handle Thrd User Name
====== ======================== === ======= ======= ================ ========================== ==== ====== ==== =========================
System ffff8008dd8e9040 (E|K|O)   0 4 (0n4) 0 (0n0) 0000000000000000 09/29/2023 11:35:55.475 AM    0      0  223 WORKGROUP\WINDEV2305EVAL$

Memory Details:

    VM      Peak     Commit Size NPP Quota
    ======= ======== =========== =========
    3.98 MB 15.53 MB       40 KB     272 B

File Server Threads (Active threads running srv or srv2 driver)
Exective Work Queues

Show LPC Port information for process

Show Threads: Unique Stacks    !mex.listthreads (!lt) ffff8008dd8e9040    !process ffff8008dd8e9040 7
```

With the **`!us`** command we can filter on a specific function, so it will only display the the call stacks that contains the function we're looking for. Here is an example where we are going to look for call stacks that contain the **`nt!RtlpHpVsFreeChunkInsert`** function.

```
0: kd> !us nt!RtlpHpVsFreeChunkInsert
1 thread [stats]: ffff8008e4604040
    fffff80766432140 nt!KeBugCheckEx
    fffff80766450c9c nt!PspSystemThreadStartup$filt$0+0x44
    fffff807663f0e81 nt!_C_specific_handler+0xa1
    fffff8076643d3af nt!RtlpExecuteHandlerForException+0xf
    fffff807662660c3 nt!RtlDispatchException+0x2f3
    fffff80766206fee nt!KiDispatchException+0x1ae
    fffff807664478fc nt!KiExceptionDispatch+0x13c
    fffff80766442983 nt!KiGeneralProtectionFault+0x343
    fffff8076624f3d4 nt!RtlpHpVsFreeChunkInsert+0x1b4
    fffff807662ad6b1 nt!RtlpHpFreeHeap+0x421
    fffff80766aad2b0 nt!ExFreePoolWithTag+0x1a0
    fffff80784da115b EvilDriver+0x115b
    fffff80784da11ed EvilDriver+0x11ed
    fffff80784da150b EvilDriver+0x150b
    fffff80784da1440 EvilDriver+0x1440
    fffff807667654d4 nt!PnpCallDriverEntry+0x54
    fffff80766765b23 nt!IopLoadDriver+0x523
    fffff80766764727 nt!IopLoadUnloadDriver+0x57
    fffff807662b8bd5 nt!ExpWorkerThread+0x155
    fffff80766212667 nt!PspSystemThreadStartup+0x57
    fffff807664370a4 nt!KiStartSystemThread+0x34

Threads matching filter: 1 out of 223
```

The thread that is running on **CPU 3** is the crashing thread.

```
0: kd> !mex.t ffff8008e4604040
Process                   Thread                       CID       UserTime KernelTime ContextSwitches Wait Reason Time State
System (ffff8008dd8e9040) ffff8008e4604040 (E|K|W|R|V) 4.15dc           0      266ms            5730 Executive      0 Running on processor 3

Priority:
    Current Base Decrement ForegroundBoost IO Page
    13      12   0         0               0  5

 # Child-SP         Return           Call Site
 0 ffffa1088a4adb88 fffff80766450c9c nt!KeBugCheckEx
 1 ffffa1088a4adb90 fffff807663f0e81 nt!PspSystemThreadStartup$filt$0+0x44
 2 ffffa1088a4adbd0 fffff8076643d3af nt!_C_specific_handler+0xa1
 3 ffffa1088a4adc40 fffff807662660c3 nt!RtlpExecuteHandlerForException+0xf
 4 ffffa1088a4adc70 fffff80766206fee nt!RtlDispatchException+0x2f3
 5 ffffa1088a4ae3e0 fffff807664478fc nt!KiDispatchException+0x1ae
 6 ffffa1088a4aeac0 fffff80766442983 nt!KiExceptionDispatch+0x13c
 7 ffffa1088a4aeca0 fffff8076624f3d4 nt!KiGeneralProtectionFault+0x343
 8 ffffa1088a4aee30 fffff807662ad6b1 nt!RtlpHpVsFreeChunkInsert+0x1b4
 9 ffffa1088a4aee60 fffff80766aad2b0 nt!RtlpHpFreeHeap+0x421
 a ffffa1088a4aef00 fffff80784da115b nt!ExFreePoolWithTag+0x1a0
 b ffffa1088a4aef90 fffff80784da11ed EvilDriver+0x115b
 c ffffa1088a4af090 fffff80784da150b EvilDriver+0x11ed
 d ffffa1088a4af0c0 fffff80784da1440 EvilDriver+0x150b
 e ffffa1088a4af100 fffff807667654d4 EvilDriver+0x1440
 f ffffa1088a4af130 fffff80766765b23 nt!PnpCallDriverEntry+0x54
10 ffffa1088a4af180 fffff80766764727 nt!IopLoadDriver+0x523
11 ffffa1088a4af340 fffff807662b8bd5 nt!IopLoadUnloadDriver+0x57
12 ffffa1088a4af380 fffff80766212667 nt!ExpWorkerThread+0x155
13 ffffa1088a4af570 fffff807664370a4 nt!PspSystemThreadStartup+0x57
14 ffffa1088a4af5c0 0000000000000000 nt!KiStartSystemThread+0x34
```

Unassembling backwards starting from the address **`EvilDriver+0x115b`** to display the preceding assembly instructions.

```
3: kd> ub EvilDriver+0x115b
EvilDriver+0x1139:
fffff807`84da1139 33d2            xor     edx,edx
fffff807`84da113b 48895c2428      mov     qword ptr [rsp+28h],rbx
fffff807`84da1140 4889442420      mov     qword ptr [rsp+20h],rax
fffff807`84da1145 ff15e50e0000    call    qword ptr [EvilDriver+0x2030 (fffff807`84da2030)] --> nt!ZwWriteFile
fffff807`84da114b ba31427257      mov     edx,57724231h --> This is the Pool Tag
fffff807`84da1150 488bcb          mov     rcx,rbx
fffff807`84da1153 8bf8            mov     edi,eax
fffff807`84da1155 ff15050f0000    call    qword ptr [EvilDriver+0x2060 (fffff807`84da2060)] --> nt!ExFreePoolWithTag
```

The output indicates that this specific memory location contains a pointer to the function **`nt!ZwWriteFile`**.

```
3: kd> dps fffff807`84da2030 l1
fffff807`84da2030  fffff807`6642e550 nt!ZwWriteFile
```

The instruction **`mov edx, 57724231h`** copies the hexadecimal value **`57724231`** into the **EDX** register.

```
fffff807`84da114b ba31427257      mov     edx,57724231h
```

The **`.formats`** command in WinDbg is used to display the value of an expression in multiple formats. It's a helpful command for understanding what a particular value represents in various number systems or formats.

In this case, **`WrB1`** is very likely the Pool Tag. **`57724231h`** is the hexadecimal representation of the characters **`WrB1`**, and a Pool Tag usually has a size of 4-bytes.

```
3: kd> .formats 57724231h
Evaluate expression:
  Hex:     00000000`57724231
  Decimal: 1467105841
  Decimal (unsigned) : 1467105841
  Octal:   0000000000012734441061
  Binary:  00000000 00000000 00000000 00000000 01010111 01110010 01000010 00110001
  Chars:   ....WrB1 --> Pool Tag
  Time:    Tue Jun 28 02:24:01 2016
  Float:   low 2.66366e+014 high 0
  Double:  7.24847e-315
```

Here we can confirm our theory, since a call is being made to **`nt!ExFreePoolWithTag`**

```
3: kd> dps fffff807`84da2060 l1
fffff807`84da2060  fffff807`66aad110 nt!ExFreePoolWithTag
```

Unassemable backwards again, but here we don't see interesting from my point of view.

```
3: kd> ub
EvilDriver+0x1118:
fffff807`84da1118 e8a3080000      call    EvilDriver+0x19c0 (fffff807`84da19c0)
fffff807`84da111d 488b4dc7        mov     rcx,qword ptr [rbp-39h]
fffff807`84da1121 488d45df        lea     rax,[rbp-21h]
fffff807`84da1125 4889742440      mov     qword ptr [rsp+40h],rsi
fffff807`84da112a 4533c9          xor     r9d,r9d
fffff807`84da112d 4889742438      mov     qword ptr [rsp+38h],rsi
fffff807`84da1132 4533c0          xor     r8d,r8d
fffff807`84da1135 897c2430        mov     dword ptr [rsp+30h],edi
```

Unassemable backwards again. Here we can see th value **`0xC000009A`** (in edi) is a status code, as it falls within the range of **NTSTATUS** codes. 

```
3: kd> ub
EvilDriver+0x10fd:
fffff807`84da10fd 488bd8          mov     rbx,rax
fffff807`84da1100 4885c0          test    rax,rax
fffff807`84da1103 7507            jne     EvilDriver+0x110c (fffff807`84da110c)
fffff807`84da1105 bf9a0000c0      mov     edi,0C000009Ah --> NTSTATUS code
fffff807`84da110a eb4f            jmp     EvilDriver+0x115b (fffff807`84da115b)
fffff807`84da110c 4c8d043f        lea     r8,[rdi+rdi]
fffff807`84da1110 ba41000000      mov     edx,41h
fffff807`84da1115 488bcb          mov     rcx,rbx
```

The status code **`0xC000009A`** corresponds to **`STATUS_INSUFFICIENT_RESOURCES`**, which is an **NTSTATUS** code used in the Windows operating system to indicate that an operation could not be completed due to a lack of system resources. For example, it could be related to insufficient memory.

```
3: kd> !error 0C000009Ah
Error code: (NTSTATUS) 0xc000009a (3221225626) - Insufficient system resources exist to complete the API.
```

The **`!vm 1`** command in WinDbg is used to display detailed information about the virtual memory statistics of the system. The command is often used for troubleshooting issues related to memory usage, leaks, or other system resource problems. It provides a snapshot of how the system’s virtual memory is currently allocated and utilized. Since the **NTSTATUS** error code **`0xc000009a`** indicates that there is insufficient system resources to complete the requested operation. Let's see if we can find anything that looks odd.

```
0: kd> !vm 1
Page File: \??\C:\pagefile.sys
  Current:  12582912 Kb  Free Space:  12527392 Kb
  Minimum:  12582912 Kb  Maximum:     12582912 Kb
Page File: \??\C:\swapfile.sys
  Current:    262144 Kb  Free Space:    212904 Kb
  Minimum:    262144 Kb  Maximum:      6289836 Kb
No Name for Paging File
  Current:  16776136 Kb  Free Space:  16299184 Kb
  Minimum:  16776136 Kb  Maximum:     16776136 Kb

Physical Memory:          1601266 (    6405064 Kb)
Available Pages:           442347 (    1769388 Kb)
ResAvail Pages:           1488586 (    5954344 Kb)
Locked IO Pages:                0 (          0 Kb)
Free System PTEs:      4294965927 (17179863708 Kb)

******* 279962 kernel stack PTE allocations have failed ******

Modified Pages:              4436 (      17744 Kb)
Modified PF Pages:           4417 (      17668 Kb)
Modified No Write Pages:       27 (        108 Kb)
NonPagedPool Usage:           227 (        908 Kb)
NonPagedPoolNx Usage:       31727 (     126908 Kb)
NonPagedPool Max:      4294967296 (17179869184 Kb)
PagedPool Usage:            59028 (     236112 Kb)
PagedPool Maximum:     4294967296 (17179869184 Kb)
Processor Commit:             767 (       3068 Kb)
Session Commit:               214 (        856 Kb)
Shared Commit:             390110 (    1560440 Kb)
Special Pool:                   0 (          0 Kb)
Kernel Stacks:              15280 (      61120 Kb)
Pages For MDLs:              3008 (      12032 Kb)
ContigMem Pages:                4 (         16 Kb)
Partition Pages:                0 (          0 Kb)
Pages For AWE:                  0 (          0 Kb)
NonPagedPool Commit:        34206 (     136824 Kb)
PagedPool Commit:  18446735296008579352 (18446708962905662560 Kb)
Driver Commit:              13165 (      52660 Kb)
Boot Commit:                 4900 (      19600 Kb)
PFN Array Commit:           19809 (      79236 Kb)
SmallNonPagedPtesCommit:      462 (       1848 Kb)
SlabAllocatorPages:          4608 (      18432 Kb)
SkPagesInUnchargedSlabs:     4810 (      19240 Kb)
System PageTables:           1039 (       4156 Kb)
ProcessLockedFilePages:        14 (         56 Kb)
Pagefile Hash Pages:          102 (        408 Kb)
Sum System Commit: 18446735296009067242 (18446708962907614120 Kb)
Total Private:             798197 (    3192788 Kb)

********** Sum of individual system commit + Process commit exceeds overall commit by 18446708962905424028 Kb ? ********
Committed pages:          1345720 (    5382880 Kb)
Commit limit:             4746994 (   18987976 Kb)
```


A response to a customer could be:

*Given these observations, it's likely that **`EvilDriver`** is causing the system to crash due to improper memory management. The driver appears to be corrupting memory, leading to an access violation exception **`(c0000005)`** and ultimately bugchecking the system.*

```
0: kd> lmvm EvilDriver
Browse full module list
start             end                 module name
fffff807`84da0000 fffff807`84da7000   EvilDriver   (no symbols)           
    Loaded symbol image file: EvilDriver.sys
    Image path: \??\C:\Users\User\source\repos\EvilDriver\x64\Release\EvilDriver.sys
    Image name: EvilDriver.sys
    Browse all global symbols  functions  data
    Timestamp:        Fri Sep 29 11:45:20 2023 (65171B40)
    CheckSum:         0000B084
    ImageSize:        00007000
    Translations:     0000.04b0 0000.04e4 0409.04b0 0409.04e4
    Information from resource tables:
```
