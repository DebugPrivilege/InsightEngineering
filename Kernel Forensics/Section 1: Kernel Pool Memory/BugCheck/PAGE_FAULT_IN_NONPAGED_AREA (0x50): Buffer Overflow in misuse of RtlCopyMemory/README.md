# Summary

The **`PAGE_FAULT_IN_NONPAGED_AREA`** bug check has a value of **`0x00000050`**. This indicates that invalid system memory has been referenced.

- **`Arg1:`** The memory address that was improperly accessed.
- **`Arg2:`** Provides additional details about the memory access, such as whether it was a read or a write operation, and if the fault was due to a not-present page table entry (PTE).
- **`Arg3:`** If non-zero, the instruction address which referenced the bad memory address.
- **`Arg4:`** Reserved.

Link to this CrashDump: https://mega.nz/file/KpUgjK4Y#7eH7G1v3OK-NS1mfgiwWqBncy3uZO5L54Fr5SMhpLbA

# Code Sample - Buffer Overflow due to Size Mismatch

The root cause of the crash is a buffer overflow in the **RtlCopyMemory** function. It tries to copy more data into **`anotherBuffer`** than its allocated size, leading to memory corruption and system instability.

The **RtlCopyMemory** function is a Windows kernel-mode routine that copies a block of memory from one location to another. It's similar to the standard C library function **memcpy** but is designed to work within the Windows kernel environment.

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

    RtlFillMemory(writeBuffer, dataSize, 'A');  // Fill the write buffer with 'A's

    anotherBuffer = (PCHAR)ExAllocatePool2(POOL_FLAG_PAGED, dataSize / 2, 'AnB1');  // Allocate a smaller buffer
    if (anotherBuffer == NULL) {
        KdPrint(("Failed to allocate memory for another buffer.\n"));
        ExFreePoolWithTag(writeBuffer, 'WrB1');
        ZwClose(fileHandle);
        return STATUS_INSUFFICIENT_RESOURCES;
    }

    RtlCopyMemory(anotherBuffer, writeBuffer, dataSize);  // Misuse: Copying more data than anotherBuffer can hold

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

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/305258b0-771d-40e4-b043-98c120a1038c)


# WinDbg Walk Through - Analyzing CrashDump

Load the CrashDump in WinDbg:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/991d7efa-25fe-40cc-b07d-d2cdcb471e59)


Start with loading the **MEX** extension:

```
0: kd> .load mex
Mex External 3.0.0.7172 Loaded!
```

Let's start with the **`!di`** command which stands for display information. This command retrieves information about the machine from which the memory dump originated. It's especially handy when working with multiple memory dumps, as it helps distinguish the analysis related to each specific dump. This is an improved version of the **`vertarget`** command.

```
0: kd> !di
Computer Name:  Not Found
Windows 10 Kernel Version 22621 MP (4 procs) Free x64
Product: WinNt, suite: TerminalServer SingleUserTS
Edition build lab: 22621.1928.amd64fre.ni_release_svc_prod3.230622-0951
Kernel base = 0xfffff801`16000000 PsLoadedModuleList = 0xfffff801`16c130e0
Debug session time: Sat Sep 23 05:14:13.751 2023 (UTC - 7:00)
System Uptime: 0 days 0:31:32.798
SystemManufacturer = Microsoft Corporation
SystemProductName = Virtual Machine
Processor: 11th Gen Intel(R) Core(TM) i7-1185G7 @ 3.00GHz
Bugcheck: 50 (FFFFE5068D8A4000, 2, FFFFF801197F1D1E, 0)
KernelMode Dump Path: C:\Windows\MEMORY_0x50.DMP
Share Path: \\WINDEV2305EVAL\ADMIN$\MEMORY_0x50.DMP
```

Let's run the **`!analyze -v`** command to quickly triage the issue as it provides a quick and detailed overview of the state of the system at the time of the crash. It automates many aspects of crash dump analysis, providing a quick way to get valuable information.

```
0: kd> !analyze -v
*******************************************************************************
*                                                                             *
*                        Bugcheck Analysis                                    *
*                                                                             *
*******************************************************************************

PAGE_FAULT_IN_NONPAGED_AREA (50)
Invalid system memory was referenced.  This cannot be protected by try-except.
Typically the address is just plain bad or it is pointing at freed memory.
Arguments:
Arg1: ffffe5068d8a4000, memory referenced.
Arg2: 0000000000000002, X64: bit 0 set if the fault was due to a not-present PTE.
	bit 1 is set if the fault was due to a write, clear if a read.
	bit 3 is set if the processor decided the fault was due to a corrupted PTE.
	bit 4 is set if the fault was due to attempted execute of a no-execute PTE.
	- ARM64: bit 1 is set if the fault was due to a write, clear if a read.
	bit 3 is set if the fault was due to attempted execute of a no-execute PTE.
Arg3: fffff801197f1d1e, If non-zero, the instruction address which referenced the bad memory
	address.
Arg4: 0000000000000000, (reserved)

Debugging Details:
------------------


KEY_VALUES_STRING: 1

    Key  : AV.Type
    Value: Write

    Key  : Analysis.CPU.mSec
    Value: 2500

    Key  : Analysis.Elapsed.mSec
    Value: 3170

    Key  : Analysis.IO.Other.Mb
    Value: 0

    Key  : Analysis.IO.Read.Mb
    Value: 1

    Key  : Analysis.IO.Write.Mb
    Value: 7

    Key  : Analysis.Init.CPU.mSec
    Value: 10905

    Key  : Analysis.Init.Elapsed.mSec
    Value: 562725

    Key  : Analysis.Memory.CommitPeak.Mb
    Value: 152

    Key  : Bugcheck.Code.KiBugCheckData
    Value: 0x50

    Key  : Bugcheck.Code.LegacyAPI
    Value: 0x50

    Key  : Dump.Attributes.AsUlong
    Value: 800

    Key  : Failure.Bucket
    Value: AV_W_(null)_Driver!unknown_function

    Key  : Failure.Hash
    Value: {2f4cb3f6-118f-01c9-d978-7b6c7a9a1257}

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


BUGCHECK_CODE:  50

BUGCHECK_P1: ffffe5068d8a4000

BUGCHECK_P2: 2

BUGCHECK_P3: fffff801197f1d1e

BUGCHECK_P4: 0

FILE_IN_CAB:  MEMORY_0x50.DMP

VIRTUAL_MACHINE:  HyperV

DUMP_FILE_ATTRIBUTES: 0x800

READ_ADDRESS:  ffffe5068d8a4000 Paged pool

MM_INTERNAL_CODE:  0

IMAGE_NAME:  Driver.sys

MODULE_NAME: Driver

FAULTING_MODULE: fffff801197f0000 Driver

BLACKBOXBSD: 1 (!blackboxbsd)


BLACKBOXNTFS: 1 (!blackboxntfs)


BLACKBOXPNP: 1 (!blackboxpnp)


BLACKBOXWINLOGON: 1

PROCESS_NAME:  System

TRAP_FRAME:  fffff90fb6d8ede0 -- (.trap 0xfffff90fb6d8ede0)
NOTE: The trap frame does not contain all registers.
Some register values may be zeroed or incorrect.
rax=ffffe5068d8a3db0 rbx=0000000000000000 rcx=ffffe5068d8a4040
rdx=000000000033f4d0 rsi=0000000000000000 rdi=0000000000000000
rip=fffff801197f1d1e rsp=fffff90fb6d8ef78 rbp=fffff90fb6d8f029
 r8=0000000000000024  r9=0000000000000007 r10=0000000000000655
r11=ffffe5068dbe36f4 r12=0000000000000000 r13=0000000000000000
r14=0000000000000000 r15=0000000000000000
iopl=0         nv up ei pl nz na pe nc
Driver+0x1d1e:
fffff801`197f1d1e 0f2949c0        movaps  xmmword ptr [rcx-40h],xmm1 ds:ffffe506`8d8a4000=????????????????????????????????
Resetting default scope

STACK_TEXT:  
fffff90f`b6d8ebb8 fffff801`164a1e1d     : 00000000`00000050 ffffe506`8d8a4000 00000000`00000002 fffff90f`b6d8ede0 : nt!KeBugCheckEx
fffff90f`b6d8ebc0 fffff801`162401dc     : fffff90f`b6d8ee20 00000000`00000002 00000000`00000000 ffffe506`8d8a4000 : nt!MiSystemFault+0x237b4d
fffff90f`b6d8ecc0 fffff801`16442d29     : fffff90f`b6d8ee20 00000000`00000000 00000000`00000002 ffffa508`47c54f78 : nt!MmAccessFault+0x29c
fffff90f`b6d8ede0 fffff801`197f1d1e     : fffff801`197f1176 00000000`00000474 fffff801`197f3001 ffffa508`4147a690 : nt!KiPageFault+0x369
fffff90f`b6d8ef78 fffff801`197f1176     : 00000000`00000474 fffff801`197f3001 ffffa508`4147a690 00000000`0000003b : Driver+0x1d1e
fffff90f`b6d8ef80 fffff801`197f1255     : 00000000`0000000a fffff801`197f32d8 fffff780`00000320 ffffffff`80000b04 : Driver+0x1176
fffff90f`b6d8f090 fffff801`197f156b     : 00000000`00000000 ffffa508`46322000 ffffa508`46322000 ffffa508`47c54dd0 : Driver+0x1255
fffff90f`b6d8f0c0 fffff801`197f14a0     : ffffa508`46322000 fffff90f`b6d8f280 ffffa508`424e95dd ffffa508`42844e50 : Driver+0x156b
fffff90f`b6d8f100 fffff801`167654d4     : ffffa508`46322000 00000000`00000000 ffffa508`42844e50 fffff801`16306b80 : Driver+0x14a0
fffff90f`b6d8f130 fffff801`16765b23     : ffffa508`46322000 00000000`00000000 00000000`00000000 ffffe506`8db14ed0 : nt!PnpCallDriverEntry+0x54
fffff90f`b6d8f180 fffff801`16764727     : 00000000`00000000 00000000`00000000 00000000`00000000 fffff801`16d49ac0 : nt!IopLoadDriver+0x523
fffff90f`b6d8f340 fffff801`162b8bd5     : ffffa508`00000000 ffffffff`80000b04 ffffa508`467e7040 ffffa508`00000000 : nt!IopLoadUnloadDriver+0x57
fffff90f`b6d8f380 fffff801`16212667     : ffffa508`467e7040 00000000`0000042a ffffa508`467e7040 fffff801`162b8a80 : nt!ExpWorkerThread+0x155
fffff90f`b6d8f570 fffff801`164370a4     : fffff801`14801180 ffffa508`467e7040 fffff801`16212610 0000025b`0a1955e8 : nt!PspSystemThreadStartup+0x57
fffff90f`b6d8f5c0 00000000`00000000     : fffff90f`b6d90000 fffff90f`b6d89000 00000000`00000000 00000000`00000000 : nt!KiStartSystemThread+0x34


SYMBOL_NAME:  Driver+1d1e

STACK_COMMAND:  .cxr; .ecxr ; kb

BUCKET_ID_FUNC_OFFSET:  1d1e

FAILURE_BUCKET_ID:  AV_W_(null)_Driver!unknown_function

OS_VERSION:  10.0.22621.1928

BUILDLAB_STR:  ni_release_svc_prod3

OSPLATFORM_TYPE:  x64

OSNAME:  Windows 10

FAILURE_ID_HASH:  {2f4cb3f6-118f-01c9-d978-7b6c7a9a1257}

Followup:     MachineOwner
---------
```

**`nt!MiSystemFault`** and **`nt!MmAccessFault`** in the call stack is a strong indication that the issue is related to memory management.

The output of **`!analyze -v`** confirms that the issue is indeed a buffer overflow. Let's break down the relevant parts of the crash dump to better understand what happened:

```
BUGCHECK_CODE: 50 (PAGE_FAULT_IN_NONPAGED_AREA)
BUGCHECK_P1: ffffe5068d8a4000 (memory referenced)
BUGCHECK_P2: 2 (write access)
```

This indicates that an invalid system memory was referenced. Specifically, a **write** access was attempted to a memory location that is not valid. 

We know that **Arg2** provides additional details about memory access. Let's use the **`!pte`** command in WinDbg is used to display the Page Table Entry (PTE) for a given virtual address. This is a key command when diagnosing issues related to memory management. It shows that the Page Table Entry (PTE) for the virtual address **`fffe5068d8a4000`** is not valid and the page has been freed.

```
0: kd> !pte ffffe5068d8a4000
                                           VA ffffe5068d8a4000
PXE at FFFFBEDF6FB7DE50    PPE at FFFFBEDF6FBCA0D0    PDE at FFFFBEDF7941A360    PTE at FFFFBEF28346C520
contains 0A00000107400863  contains 0A00000005DC8863  contains 1A0000025754A863  contains 00003E9800000000
pfn 107400    ---DA--KWEV  pfn 5dc8      ---DA--KWEV  pfn 25754a    ---DA--KWEV  not valid
                                                                                  Page has been freed
```

The **`.trap`** output shows the trap frame, which is a data structure saved by the operating system when an exception occurs. The trap frame contains the state of the CPU registers at the time of the exception.

```
0: kd> .trap 0xfffff90fb6d8ede0
NOTE: The trap frame does not contain all registers.
Some register values may be zeroed or incorrect.
rax=ffffe5068d8a3db0 rbx=0000000000000000 rcx=ffffe5068d8a4040
rdx=000000000033f4d0 rsi=0000000000000000 rdi=0000000000000000
rip=fffff801197f1d1e rsp=fffff90fb6d8ef78 rbp=fffff90fb6d8f029
 r8=0000000000000024  r9=0000000000000007 r10=0000000000000655
r11=ffffe5068dbe36f4 r12=0000000000000000 r13=0000000000000000
r14=0000000000000000 r15=0000000000000000
iopl=0         nv up ei pl nz na pe nc
Driver+0x1d1e:
fffff801`197f1d1e 0f2949c0        movaps  xmmword ptr [rcx-40h],xmm1 ds:ffffe506`8d8a4000=????????????????????????????????
```

When an exception or interrupt is triggered, the system saves the current state of the CPU registers into a **`KTRAP_FRAME`** structure. This allows the system to restore the CPU to its previous state after the exception or interrupt has been handled.

The trap frame contains various fields like:

- **General-purpose registers** (like RAX, RBX, RCX, etc.)
- **Instruction Pointer (RIP)**, which tells where the exception happened
- **Stack Pointer (RSP)**, which points to the current stack frame

This is how the data structure looks like:

```
0: kd> !ddt nt!_KTRAP_FRAME fffff90fb6d8ede0

dt nt!_KTRAP_FRAME fffff90fb6d8ede0 () Recursive: [ -r1 -r2 -r ] Verbose Normal dt
==================================================================================
   +0x000 P1Home               : 0xfffff90f`b6d8ee20 (0n-7629089214944)
   +0x008 P2Home               : 0
   +0x010 P3Home               : 2
   +0x018 P4Home               : 0xffffa508`47c54f78 (0n-100019994275976)
   +0x020 P5                   : 0
   +0x028 PreviousMode         : 0 ''
   +0x028 InterruptRetpolineState : 0 ''
   +0x029 PreviousIrql         : 0 ''
   +0x02a FaultIndicator       : 0x1 ''
   +0x02a NmiMsrIbrs           : 0x1 ''
   +0x02b ExceptionActive      : 0x1 ''
   +0x02c MxCsr                : 0x1f80 (0n8064)
   +0x030 Rax                  : 0xffffe506`8d8a3db0 (0n-29658669498960)
   +0x038 Rcx                  : 0xffffe506`8d8a4040 (0n-29658669498304)
   +0x040 Rdx                  : 0x33f4d0 (0n3405008)
   +0x048 R8                   : 0x24 (0n36)
   +0x050 R9                   : 7
   +0x058 R10                  : 0x655 (0n1621)
   +0x060 R11                  : 0xffffe506`8dbe36f4 (0n-29658666092812)
   +0x068 GsBase               : 1
   +0x068 GsSwap               : 1
   +0x070 Xmm0                 : _M128A
   +0x080 Xmm1                 : _M128A
   +0x090 Xmm2                 : _M128A
   +0x0a0 Xmm3                 : _M128A
   +0x0b0 Xmm4                 : _M128A
   +0x0c0 Xmm5                 : _M128A
   +0x0d0 FaultAddress         : 0xffffe506`8d8a4000 (0n-29658669498368)
   +0x0d0 ContextRecord        : 0xffffe506`8d8a4000 (0n-29658669498368)
   +0x0d8 Dr0                  : 0
   +0x0e0 Dr1                  : 0
   +0x0e8 Dr2                  : 0xfffff801`16aad78d (0n-8791417759859)
   +0x0f0 Dr3                  : 0x100 (0n256)
   +0x0f8 Dr6                  : 1
   +0x100 Dr7                  : 0x401 (0n1025)
   +0x0d8 ShadowStackFrame     : 0
   +0x0e0 Spare                : [5] 0
   +0x108 DebugControl         : 0xfffff90f`00000000 (0n-7632156884992)
   +0x110 LastBranchToRip      : 0xf1bf27ab`00000000 (0n-1027058574624292864)
   +0x118 LastBranchFromRip    : 0xbc100110`18405fc6 (0n-4895411626313752634)
   +0x120 LastExceptionToRip   : 0x474 (0n1140)
   +0x128 LastExceptionFromRip : 0xfffff801`197f1bd5 (0n-8791370294315)
   +0x130 SegDs                : 0xd458 (0n54360)
   +0x132 SegEs                : 0x414b (0n16715)
   +0x134 SegFs                : 0xa508 (0n42248)
   +0x136 SegGs                : 0xffff (0n65535)
   +0x138 TrapFrame            : 0
   +0x140 Rbx                  : 0x474 (0n1140)
   +0x148 Rdi                  : 0x30 (0n48)
   +0x150 Rsi                  : 0x41414141`41414141 (0n4702111234474983745)
   +0x158 Rbp                  : 0xfffff90f`b6d8f029 (0n-7629089214423)
   +0x160 ErrorCode            : 2
   +0x160 ExceptionFrame       : 2
   +0x168 Rip                  : 0xfffff801`197f1d1e (0n-8791370293986)
   +0x170 SegCs                : 0x10 (0n16)
   +0x172 Fill0                : 0 ''
   +0x173 Logging              : 0 ''
   +0x174 Fill1                : [2] 0
   +0x178 EFlags               : 0x50202 (0n328194)
   +0x17c Fill2                : 0
   +0x180 Rsp                  : 0xfffff90f`b6d8ef78 (0n-7629089214600)
   +0x188 SegSs                : 0x18 (0n24)
   +0x18a Fill3                : 0
   +0x18c Fill4                : 0
```

**`rip=fffff801197f1d1e`** is the instruction pointer, pointing to the address of the code that caused the exception. It's the same as **`Arg3`** in the bugcheck parameters, confirming that the instruction at this address tried to perform an invalid operation.

Let's get back to the crashing thread and display the call stack. We are interested in **`Driver+0x1d1e`**, which indicates an offset **(0x1d1e)** from the base address of a loaded driver. It pinpoints the location of an instruction within this driver where a noteworthy event occurred, such as a crash in this case.

```
0: kd> !crash
Dump Info
============================================
Computer Name:  Not Found
Windows 10 Kernel Version 22621 MP (4 procs) Free x64
Product: WinNt, suite: TerminalServer SingleUserTS
Edition build lab: 22621.1928.amd64fre.ni_release_svc_prod3.230622-0951
Kernel base = 0xfffff801`16000000 PsLoadedModuleList = 0xfffff801`16c130e0
Debug session time: Sat Sep 23 05:14:13.751 2023 (UTC - 7:00)
System Uptime: 0 days 0:31:32.798
SystemManufacturer = Microsoft Corporation
SystemProductName = Virtual Machine
Processor: 11th Gen Intel(R) Core(TM) i7-1185G7 @ 3.00GHz
Bugcheck: 50 (FFFFE5068D8A4000, 2, FFFFF801197F1D1E, 0)
KernelMode Dump Path: C:\Windows\MEMORY_0x50.DMP
Share Path: \\WINDEV2305EVAL\ADMIN$\MEMORY_0x50.DMP


Bugcheck details
============================================
Bugcheck code 00000050
Arguments ffffe506`8d8a4000 00000000`00000002 fffff801`197f1d1e 00000000`00000000

Crashing Stack
============================================
 # Child-SP          RetAddr               Call Site
00 fffff90f`b6d8ebb8 fffff801`164a1e1d     nt!KeBugCheckEx
01 fffff90f`b6d8ebc0 fffff801`162401dc     nt!MiSystemFault+0x237b4d
02 fffff90f`b6d8ecc0 fffff801`16442d29     nt!MmAccessFault+0x29c
03 fffff90f`b6d8ede0 fffff801`197f1d1e     nt!KiPageFault+0x369
04 fffff90f`b6d8ef78 fffff801`197f1176     Driver+0x1d1e
05 fffff90f`b6d8ef80 fffff801`197f1255     Driver+0x1176
06 fffff90f`b6d8f090 fffff801`197f156b     Driver+0x1255
07 fffff90f`b6d8f0c0 fffff801`197f14a0     Driver+0x156b
08 fffff90f`b6d8f100 fffff801`167654d4     Driver+0x14a0
09 fffff90f`b6d8f130 fffff801`16765b23     nt!PnpCallDriverEntry+0x54
0a fffff90f`b6d8f180 fffff801`16764727     nt!IopLoadDriver+0x523
0b fffff90f`b6d8f340 fffff801`162b8bd5     nt!IopLoadUnloadDriver+0x57
0c fffff90f`b6d8f380 fffff801`16212667     nt!ExpWorkerThread+0x155
0d fffff90f`b6d8f570 fffff801`164370a4     nt!PspSystemThreadStartup+0x57
0e fffff90f`b6d8f5c0 00000000`00000000     nt!KiStartSystemThread+0x34
```

The **`ub`** command stands for "unassemble backward". This command disassembles the code in reverse, starting from a specific address. In our case, we're looking at the assembly instructions leading up to the address **`Driver+0x1d1e`**

```
0: kd> ub Driver+0x1d1e
Driver+0x1cf1:
fffff801`197f1cf1 666666666666660f1f840000000000 nop word ptr [rax+rax]
fffff801`197f1d00 0f100c11        movups  xmm1,xmmword ptr [rcx+rdx]
fffff801`197f1d04 0f10541110      movups  xmm2,xmmword ptr [rcx+rdx+10h]
fffff801`197f1d09 0f105c1120      movups  xmm3,xmmword ptr [rcx+rdx+20h]
fffff801`197f1d0e 0f10641130      movups  xmm4,xmmword ptr [rcx+rdx+30h]
fffff801`197f1d13 0f2941f0        movaps  xmmword ptr [rcx-10h],xmm0
fffff801`197f1d17 4883c140        add     rcx,40h -->  Adds 0x40 to rcx, effectively pointing it to a new memory location.
fffff801`197f1d1b 49ffc9          dec     r9
```

The use of **`rcx`** for memory operations is interesting. The fault was at **`Driver+0x1d1e`**, where **`rcx`** was used as the base address for a memory write **`(movaps xmmword ptr [rcx-40h],xmm1)`**. If **`rcx`** contains an invalid address, that would explain the page fault.

As discussed previously, the output of the **`.trap`** command provides the context of the thread at the time the exception occurred. 

```
0: kd> .trap 0xfffff90fb6d8ede0
NOTE: The trap frame does not contain all registers.
Some register values may be zeroed or incorrect.
rax=ffffe5068d8a3db0 rbx=0000000000000000 rcx=ffffe5068d8a4040 --> Interesting
rdx=000000000033f4d0 rsi=0000000000000000 rdi=0000000000000000
rip=fffff801197f1d1e rsp=fffff90fb6d8ef78 rbp=fffff90fb6d8f029
 r8=0000000000000024  r9=0000000000000007 r10=0000000000000655
r11=ffffe5068dbe36f4 r12=0000000000000000 r13=0000000000000000
r14=0000000000000000 r15=0000000000000000
iopl=0         nv up ei pl nz na pe nc
Driver+0x1d1e:
fffff801`197f1d1e 0f2949c0        movaps  xmmword ptr [rcx-40h],xmm1 ds:ffffe506`8d8a4000=????????????????????????????????
```

This instruction tries to move a value from the **`xmm1`** register to the calculated memory address **`(rcx-0x40)`**. The issue here is that the memory access is invalid, as indicated by the **`ds:ffffe5068d8a4000=????????????????????????????????`**

```
fffff801`197f1d1e 0f2949c0        movaps  xmmword ptr [rcx-40h],xmm1 ds:ffffe506`8d8a4000=????????????????????????????????
```

The PTE is not valid, which means that the virtual address **`ffffe5068d8a4040`** is not currently mapped to a physical address.

```
0: kd> !pte ffffe5068d8a4040
                                           VA ffffe5068d8a4040
PXE at FFFFBEDF6FB7DE50    PPE at FFFFBEDF6FBCA0D0    PDE at FFFFBEDF7941A360    PTE at FFFFBEF28346C520
contains 0A00000107400863  contains 0A00000005DC8863  contains 1A0000025754A863  contains 00003E9800000000
pfn 107400    ---DA--KWEV  pfn 5dc8      ---DA--KWEV  pfn 25754a    ---DA--KWEV  not valid
                                                                                  Page has been freed
```

The issue is that the instruction at **`Driver+0x1d1e`** is attempting to write a value to a memory location calculated as **`rcx-40h`** which is **`rcx=ffffe5068d8a4040`**. However, this virtual address is not mapped to a valid physical address, which leads to a bugcheck of this system.

Let's go back to the crashing thread and it's call stack and try to exhaust it:

```
Crashing Stack
============================================
 # Child-SP          RetAddr               Call Site
00 fffff90f`b6d8ebb8 fffff801`164a1e1d     nt!KeBugCheckEx
01 fffff90f`b6d8ebc0 fffff801`162401dc     nt!MiSystemFault+0x237b4d
02 fffff90f`b6d8ecc0 fffff801`16442d29     nt!MmAccessFault+0x29c
03 fffff90f`b6d8ede0 fffff801`197f1d1e     nt!KiPageFault+0x369
04 fffff90f`b6d8ef78 fffff801`197f1176     Driver+0x1d1e
05 fffff90f`b6d8ef80 fffff801`197f1255     Driver+0x1176
06 fffff90f`b6d8f090 fffff801`197f156b     Driver+0x1255
07 fffff90f`b6d8f0c0 fffff801`197f14a0     Driver+0x156b
08 fffff90f`b6d8f100 fffff801`167654d4     Driver+0x14a0
09 fffff90f`b6d8f130 fffff801`16765b23     nt!PnpCallDriverEntry+0x54
0a fffff90f`b6d8f180 fffff801`16764727     nt!IopLoadDriver+0x523
0b fffff90f`b6d8f340 fffff801`162b8bd5     nt!IopLoadUnloadDriver+0x57
0c fffff90f`b6d8f380 fffff801`16212667     nt!ExpWorkerThread+0x155
0d fffff90f`b6d8f570 fffff801`164370a4     nt!PspSystemThreadStartup+0x57
0e fffff90f`b6d8f5c0 00000000`00000000     nt!KiStartSystemThread+0x34
```

Unassembling backwards starting from the address **`Driver+0x1176`** to display the preceding assembly instructions.

```
0: kd> ub Driver+0x1176
Driver+0x1158:
fffff801`197f1158 488bcf          mov     rcx,rdi
fffff801`197f115b ff15ff0e0000    call    qword ptr [Driver+0x2060 (fffff801`197f2060)] --> Call to some function
fffff801`197f1161 bb9a0000c0      mov     ebx,0C000009Ah --> NTSTATUS error code
fffff801`197f1166 eb5a            jmp     Driver+0x11c2 (fffff801`197f11c2)
fffff801`197f1168 4c8bc3          mov     r8,rbx
fffff801`197f116b 488bd7          mov     rdx,rdi
fffff801`197f116e 488bce          mov     rcx,rsi
fffff801`197f1171 e88a0a0000      call    Driver+0x1c00 (fffff801`197f1c00)
```

The output indicates that this specific memory location contains a pointer to the function **`nt!ExFreePoolWithTag`**.

```
0: kd> dps fffff801`197f2060 l1
fffff801`197f2060  fffff801`16aad110 nt!ExFreePoolWithTag
```

The error code **`0xC000009A`** corresponds to **`STATUS_INSUFFICIENT_RESOURCES`**, which indicates that the system doesn't have enough resources to complete the requested operation. This is usually associated with low memory conditions or other resource limitations.

```
0: kd> !error 0C000009Ah
Error code: (NTSTATUS) 0xc000009a (3221225626) - Insufficient system resources exist to complete the API.
```

The **`!vm 1`** command in WinDbg is used to display detailed information about the virtual memory statistics of the system. The command is often used for troubleshooting issues related to memory usage, leaks, or other system resource problems. It provides a snapshot of how the system’s virtual memory is currently allocated and utilized. Since the **NTSTATUS** error code **0xc000009a** indicates that there is insufficient system resources to complete the requested operation. Let's see if we can find anything that looks odd.

```
0: kd> !vm 1
Page File: \??\C:\pagefile.sys
  Current:  12582912 Kb  Free Space:  12527296 Kb
  Minimum:  12582912 Kb  Maximum:     12582912 Kb
Page File: \??\C:\swapfile.sys
  Current:    262144 Kb  Free Space:    223824 Kb
  Minimum:    262144 Kb  Maximum:      6289836 Kb
No Name for Paging File
  Current:  16776136 Kb  Free Space:  16095604 Kb
  Minimum:  16776136 Kb  Maximum:     16776136 Kb

Physical Memory:          2844914 (   11379656 Kb)
Available Pages:           666710 (    2666840 Kb)
ResAvail Pages:           2714282 (   10857128 Kb)
Locked IO Pages:                0 (          0 Kb)
Free System PTEs:      4294977138 (17179908552 Kb)

******* 73746 kernel stack PTE allocations have failed ******

Modified Pages:             12504 (      50016 Kb)
Modified PF Pages:          12461 (      49844 Kb)
Modified No Write Pages:        0 (          0 Kb)
NonPagedPool Usage:           272 (       1088 Kb)
NonPagedPoolNx Usage:       33612 (     134448 Kb)
NonPagedPool Max:      4294967296 (17179869184 Kb)
PagedPool Usage:            66288 (     265152 Kb)
PagedPool Maximum:     4294967296 (17179869184 Kb)
Processor Commit:             741 (       2964 Kb)
Session Commit:               208 (        832 Kb)
Shared Commit:            1417735 (    5670940 Kb)
Special Pool:                   0 (          0 Kb)
Kernel Stacks:              14874 (      59496 Kb)
Pages For MDLs:              3008 (      12032 Kb)
ContigMem Pages:                4 (         16 Kb)
Partition Pages:                0 (          0 Kb)
Pages For AWE:                  0 (          0 Kb)
NonPagedPool Commit:        35482 (     141928 Kb)
PagedPool Commit:  18446735282293205272 (18446708908044166240 Kb)
Driver Commit:              13223 (      52892 Kb)
Boot Commit:                 4899 (      19596 Kb)
PFN Array Commit:           34410 (     137640 Kb)
SmallNonPagedPtesCommit:      462 (       1848 Kb)
SlabAllocatorPages:          4608 (      18432 Kb)
SkPagesInUnchargedSlabs:     7380 (      29520 Kb)
System PageTables:           1115 (       4460 Kb)
ProcessLockedFilePages:        14 (         56 Kb)
Pagefile Hash Pages:          106 (        424 Kb)
Sum System Commit: 18446735282294738933 (18446708908050300884 Kb)
Total Private:             754817 (    3019268 Kb)

********** Sum of individual system commit + Process commit exceeds overall commit by 18446708908043898408 Kb ? ********
Committed pages:          2355436 (    9421744 Kb)
Commit limit:             5990642 (   23962568 Kb)
```

There have been 73,746 failed kernel stack Page Table Entry (PTE) allocations. This is concerning and could indicate a resource exhaustion issue. The sum of the system and process commit values exceeds the overall commit, which doesn't make sense and points to potential issues in memory management.

This thread **`(ffffa508467e7040)`** that has been running on CPU 0 is the crashing thread and was responsible for bugchecking the system.

```
0: kd> !mex.t
Process                   Thread                       CID       UserTime KernelTime ContextSwitches Wait Reason Time State
System (ffffa508414e9040) ffffa508467e7040 (E|K|W|R|V) 4.13c0           0       31ms             400 Executive      0 Running on processor 0

Priority:
    Current Base Decrement ForegroundBoost IO Page
    12      12   0         0               0  5

# Child-SP         Return           Call Site
0 fffff90fb6d8ebb8 fffff801164a1e1d nt!KeBugCheckEx
1 fffff90fb6d8ebc0 fffff801162401dc nt!MiSystemFault+0x237b4d
2 fffff90fb6d8ecc0 fffff80116442d29 nt!MmAccessFault+0x29c
3 fffff90fb6d8ede0 fffff801197f1d1e nt!KiPageFault+0x369
4 fffff90fb6d8ef78 fffff801197f1176 Driver+0x1d1e
5 fffff90fb6d8ef80 fffff801197f1255 Driver+0x1176
6 fffff90fb6d8f090 fffff801197f156b Driver+0x1255
7 fffff90fb6d8f0c0 fffff801197f14a0 Driver+0x156b
8 fffff90fb6d8f100 fffff801167654d4 Driver+0x14a0
9 fffff90fb6d8f130 fffff80116765b23 nt!PnpCallDriverEntry+0x54
a fffff90fb6d8f180 fffff80116764727 nt!IopLoadDriver+0x523
b fffff90fb6d8f340 fffff801162b8bd5 nt!IopLoadUnloadDriver+0x57
c fffff90fb6d8f380 fffff80116212667 nt!ExpWorkerThread+0x155
d fffff90fb6d8f570 fffff801164370a4 nt!PspSystemThreadStartup+0x57
e fffff90fb6d8f5c0 0000000000000000 nt!KiStartSystemThread+0x34
```
