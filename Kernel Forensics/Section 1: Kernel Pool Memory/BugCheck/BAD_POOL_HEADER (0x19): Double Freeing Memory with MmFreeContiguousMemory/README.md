# Summary

The **`BAD_POOL_HEADER`** bugcheck, with an error code of **`0x19`**, indicates that a problem has been detected with the Windows memory allocation. This bugcheck is usually caused by issues with the pool header's structure. The pool header is a small block of data that contains information about how the memory pool is to be used, and corruption of this header can lead to a system's bugcheck.

Link to the CrashDump: https://mega.nz/file/LwcnTSSC#XAt_uSc7QTXyxCPu-WseWiQ_RRZJBGIS1W8X93g_PsY

# Code Sample - Double Freeing Memory

The root cause of the bugcheck in this code is a "double free". The function **`MmFreeContiguousMemory`** is called twice on the same **`contiguousBuffer`**. When memory is freed the first time, the operating system marks it as available for future allocations. If the same memory is freed again, it could already have been reallocated for a different purpose. When the second free operation occurs, it disrupts the metadata in the pool header.

```c
#include <ntddk.h>

NTSTATUS CreateRandomFile(UINT32 fileNumber, SIZE_T dataSize) {
    HANDLE fileHandle;
    OBJECT_ATTRIBUTES objAttr;
    IO_STATUS_BLOCK ioStatusBlock;
    NTSTATUS status;
    WCHAR fileNameBuffer[] = L"\\??\\C:\\Temp\\fileX.txt";
    PCHAR writeBuffer;
    PCHAR contiguousBuffer;
    PHYSICAL_ADDRESS HighestAcceptableAddress;
    HighestAcceptableAddress.QuadPart = -1;
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

    contiguousBuffer = (PCHAR)MmAllocateContiguousMemory(dataSize, HighestAcceptableAddress);
    if (contiguousBuffer == NULL) {
        KdPrint(("Failed to allocate contiguous memory.\n"));
        ExFreePoolWithTag(writeBuffer, 'WrB1');
        ZwClose(fileHandle);
        return STATUS_INSUFFICIENT_RESOURCES;
    }

    // Write to the file
    status = ZwWriteFile(fileHandle, NULL, NULL, NULL, &ioStatusBlock, writeBuffer, dataSize, NULL, NULL);

    ExFreePoolWithTag(writeBuffer, 'WrB1');
    MmFreeContiguousMemory(contiguousBuffer); // First free

    MmFreeContiguousMemory(contiguousBuffer); // Double free (deliberate bug)

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

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/d40a921c-128d-457c-b156-81ade874564f)


# WinDbg Walk Through - Analyzing CrashDump

Load the CrashDump in WinDbg:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/ec0175f2-9889-4987-8a00-61aadbc47497)


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
Kernel base = 0xfffff802`11000000 PsLoadedModuleList = 0xfffff802`11c130e0
Debug session time: Thu Oct  5 10:31:56.588 2023 (UTC - 7:00)
System Uptime: 0 days 0:10:25.633
SystemManufacturer = Microsoft Corporation
SystemProductName = Virtual Machine
Processor: 11th Gen Intel(R) Core(TM) i7-1185G7 @ 3.00GHz
Bugcheck: 19 (22, FFFFC48068D7D000, 4, 0)
KernelMode Dump Path: C:\Windows\MEMORY_DoubleFree.DMP
Share Path: \\WINDEV2305EVAL\ADMIN$\MEMORY_DoubleFree.DMP
```

Let's run the **`!analyze -v`** command to quickly triage the issue as it provides a quick and detailed overview of the state of the system at the time of the crash. It automates many aspects of crash dump analysis, providing a quick way to get valuable information.

```
0: kd> !analyze -v
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
Arg2: ffffc48068d7d000
Arg3: 0000000000000004
Arg4: 0000000000000000

Debugging Details:
------------------


KEY_VALUES_STRING: 1

    Key  : Analysis.CPU.mSec
    Value: 1468

    Key  : Analysis.Elapsed.mSec
    Value: 1870

    Key  : Analysis.IO.Other.Mb
    Value: 0

    Key  : Analysis.IO.Read.Mb
    Value: 1

    Key  : Analysis.IO.Write.Mb
    Value: 6

    Key  : Analysis.Init.CPU.mSec
    Value: 968

    Key  : Analysis.Init.Elapsed.mSec
    Value: 2125144

    Key  : Analysis.Memory.CommitPeak.Mb
    Value: 141

    Key  : Bugcheck.Code.KiBugCheckData
    Value: 0x19

    Key  : Bugcheck.Code.LegacyAPI
    Value: 0x19

    Key  : Dump.Attributes.AsUlong
    Value: 800

    Key  : Failure.Bucket
    Value: 0x19_22_Driver!unknown_function

    Key  : Failure.Hash
    Value: {13274a29-c6c5-4e24-cb5a-0988e703af11}

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


BUGCHECK_CODE:  19

BUGCHECK_P1: 22

BUGCHECK_P2: ffffc48068d7d000

BUGCHECK_P3: 4

BUGCHECK_P4: 0

FILE_IN_CAB:  MEMORY_DoubleFree.DMP

VIRTUAL_MACHINE:  HyperV

DUMP_FILE_ATTRIBUTES: 0x800

POOL_ADDRESS:  ffffc48068d7d000 

BLACKBOXBSD: 1 (!blackboxbsd)


BLACKBOXNTFS: 1 (!blackboxntfs)


BLACKBOXPNP: 1 (!blackboxpnp)


BLACKBOXWINLOGON: 1

PROCESS_NAME:  System

STACK_TEXT:  
ffff9b0b`95f7ee78 fffff802`11503442     : 00000000`00000019 00000000`00000022 ffffc480`68d7d000 00000000`00000004 : nt!KeBugCheckEx
ffff9b0b`95f7ee80 fffff802`113af573     : ffffc480`00000000 ffff9b0b`95f7ef90 ffffc480`68d7d000 00000000`00000029 : nt!ExRemovePoolTag+0x1657ae
ffff9b0b`95f7eef0 fffff802`3fbc11b3     : 00000000`00000000 00000000`00000004 ffffe709`00000000 00000000`00000000 : nt!MmFreeContiguousMemory+0x93
ffff9b0b`95f7ef80 fffff802`3fbc1249     : 00000000`00000001 fffff802`3fbc32d8 fffff780`00000320 00000000`00000000 : Driver+0x11b3
ffff9b0b`95f7f090 fffff802`3fbc156b     : 00000000`00000000 ffffe709`67cc9000 ffffe709`67cc9000 ffffe709`6428ea80 : Driver+0x1249
ffff9b0b`95f7f0c0 fffff802`3fbc14a0     : ffffe709`67cc9000 ffff9b0b`95f7f280 ffffe709`6ac873dd ffffe709`68a36320 : Driver+0x156b
ffff9b0b`95f7f100 fffff802`117654d4     : ffffe709`67cc9000 00000000`00000000 ffffe709`68a36320 fffff802`11306b80 : Driver+0x14a0
ffff9b0b`95f7f130 fffff802`11765b23     : ffffe709`67cc9000 00000000`00000000 00000000`00000000 ffffb283`dec26f10 : nt!PnpCallDriverEntry+0x54
ffff9b0b`95f7f180 fffff802`11764727     : 00000000`00000000 00000000`00000000 00000000`00000200 fffff802`11d49ac0 : nt!IopLoadDriver+0x523
ffff9b0b`95f7f340 fffff802`112b8bd5     : ffffe709`00000000 ffffffff`8000146c ffffe709`685e1040 ffffe709`00000000 : nt!IopLoadUnloadDriver+0x57
ffff9b0b`95f7f380 fffff802`11212667     : ffffe709`685e1040 00000000`00000350 ffffe709`685e1040 fffff802`112b8a80 : nt!ExpWorkerThread+0x155
ffff9b0b`95f7f570 fffff802`114370a4     : ffffc480`542a8180 ffffe709`685e1040 fffff802`11212610 00000000`00000246 : nt!PspSystemThreadStartup+0x57
ffff9b0b`95f7f5c0 00000000`00000000     : ffff9b0b`95f80000 ffff9b0b`95f79000 00000000`00000000 00000000`00000000 : nt!KiStartSystemThread+0x34


SYMBOL_NAME:  Driver+11b3

MODULE_NAME: Driver

IMAGE_NAME:  Driver.sys

STACK_COMMAND:  .cxr; .ecxr ; kb

BUCKET_ID_FUNC_OFFSET:  11b3

FAILURE_BUCKET_ID:  0x19_22_Driver!unknown_function

OS_VERSION:  10.0.22621.1928

BUILDLAB_STR:  ni_release_svc_prod3

OSPLATFORM_TYPE:  x64

OSNAME:  Windows 10

FAILURE_ID_HASH:  {13274a29-c6c5-4e24-cb5a-0988e703af11}

Followup:     MachineOwner
---------
```

The system crashed due to a **`BAD_POOL_HEADER`** error, which is often a sign of kernel pool memory corruption. Let's try to understand the arguments before we go further with the analysis:

```
0: kd> !analyze -show
BAD_POOL_HEADER (19)
The pool is already corrupt at the time of the current request.
This may or may not be due to the caller.
The internal pool links must be walked to figure out a possible cause of
the problem, and then special pool applied to the suspect tags or the driver
verifier to a suspect driver.
Arguments:
Arg1: 0000000000000022, 
Arg2: ffffc48068d7d000
Arg3: 0000000000000004
Arg4: 0000000000000000
```

This is what the official documentation from Microsoft says:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/11d4b86e-fa6f-495e-ba96-a90e529787f3)


**`Arg2`** represents the pool entry that is being checked. The output from the **`!pool`** command is used to examine the state of a specific pool memory address. 

The output indicates that the pool address **`ffffc48068d7d000`** is neither a valid regular pool nor a large pool allocation. The debugger couldn't read the address, suggesting that it might be freed, corrupt, or paged out. 

```
0: kd> !pool ffffc48068d7d000
Pool page ffffc48068d7d000 region is Unknown
ffffc48068d7d000 is not a valid large pool allocation, checking large session pool...
Unable to read large session pool table (Session data is not present in mini and kernel-only dumps)
ffffc48068d7d000 is not valid pool. Checking for freed (or corrupt) pool
Address ffffc48068d7d000 could not be read. It may be a freed, invalid or paged out page
```

The **`!poolval`** output strongly suggests that the pool page at the address **`ffffc48068d7d000`** is corrupt, specifically mentioning that its headers are invalid.

```
0: kd> !poolval fffc48068d7d000
Pool page 0fffc48068d7d000 region is Unknown

Validating Pool headers for pool page: 0fffc48068d7d000

Pool page [ 0fffc48068d7d000 ] is INVALID.

Analyzing linked list...


Scanning for single bit errors...

None found
```

Let's use the **`!pte`** command to display the Page Table Entry (PTE) for a given virtual address. This is a key command when diagnosing issues related to memory management. It shows that the Page Table Entry (PTE) for the virtual address **`ffffc48068d7d000`** is not valid and the page has been freed.

```
0: kd> !pte ffffc48068d7d000
                                           VA ffffc48068d7d000
PXE at FFFFFC7E3F1F8C48    PPE at FFFFFC7E3F189008    PDE at FFFFFC7E31201A30    PTE at FFFFFC6240346BE8
contains 0A00000010338863  contains 0A00000010339863  contains 0A0000000E0F5863  contains 00068DC000001000
pfn 10338     ---DA--KWEV  pfn 10339     ---DA--KWEV  pfn e0f5      ---DA--KWEV  not valid
                                                                                  Page has been freed
```

This output indicates that the debugger was unable to read the **`_LIST_ENTRY`** structure at the memory address **`fffc48068d7d000`**. The **`Flink`** and **`Blink`** fields, which should hold valid addresses, are unreadable. This usually suggests that the memory might be corrupted, deallocated, or paged out. 

```
1: kd> dt nt!_LIST_ENTRY ffffc48068d7d000
   +0x000 Flink            : ???? 
   +0x008 Blink            : ???? 
Memory read error ffffc48068d7d008
```

Let's look back at our thread that is running on **CPU 0**. Here we have the call stack of our crashing thread. Let's examine this call stack further to provide as much as context as possible on the driver that is responsible for the crash.

```
0: kd> !mex.t
Mex External 3.0.0.7172 Loaded!
Process                   Thread                       CID       UserTime KernelTime ContextSwitches Wait Reason Time State
System (ffffe70963ae9040) ffffe709685e1040 (E|K|W|R|V) 4.21a0           0      406ms           10800 Executive      0 Running on processor 0

Priority:
    Current Base Decrement ForegroundBoost IO Page
    13      12   0         0               0  5

# Child-SP         Return           Call Site
0 ffff9b0b95f7ee78 fffff80211503442 nt!KeBugCheckEx
1 ffff9b0b95f7ee80 fffff802113af573 nt!ExRemovePoolTag+0x1657ae
2 ffff9b0b95f7eef0 fffff8023fbc11b3 nt!MmFreeContiguousMemory+0x93
3 ffff9b0b95f7ef80 fffff8023fbc1249 Driver+0x11b3
4 ffff9b0b95f7f090 fffff8023fbc156b Driver+0x1249
5 ffff9b0b95f7f0c0 fffff8023fbc14a0 Driver+0x156b
6 ffff9b0b95f7f100 fffff802117654d4 Driver+0x14a0
7 ffff9b0b95f7f130 fffff80211765b23 nt!PnpCallDriverEntry+0x54
8 ffff9b0b95f7f180 fffff80211764727 nt!IopLoadDriver+0x523
9 ffff9b0b95f7f340 fffff802112b8bd5 nt!IopLoadUnloadDriver+0x57
a ffff9b0b95f7f380 fffff80211212667 nt!ExpWorkerThread+0x155
b ffff9b0b95f7f570 fffff802114370a4 nt!PspSystemThreadStartup+0x57
c ffff9b0b95f7f5c0 0000000000000000 nt!KiStartSystemThread+0x34
```

The command **`.frame /r 8`** sets the context to the **8th** frame in the stack trace and displays the register values at that moment. **By examining the register values and the current instruction, we can get insights into what the function is doing at that point.**

This is a snapshot of the CPU state at the **8th** frame of the stack, which is within the **`nt!IopLoadDriver`** function. The **`IopLoadDriver`** function is responsible for loading a device driver into memory. 

```
0: kd> .frame /r 8
08 ffff9b0b`95f7f180 fffff802`11764727     nt!IopLoadDriver+0x523
rax=0000000000008000 rbx=ffffe70967cc9000 rcx=0000000000000019
rdx=0000000000000022 rsi=0000000000000000 rdi=0000000000000000
rip=fffff80211765b23 rsp=ffff9b0b95f7f180 rbp=ffff9b0b95f7f280
 r8=ffffc48068d7d000  r9=0000000000000004 r10=0000000000000000
r11=0000000000008000 r12=ffffffff8000146c r13=0000000000000002
r14=ffffe70968a36320 r15=ffffb283d6708070
iopl=0         nv up ei pl zr na po nc
cs=0010  ss=0018  ds=002b  es=002b  fs=0053  gs=002b             efl=00040246
nt!IopLoadDriver+0x523:
fffff802`11765b23 8bf0            mov     esi,eax
```

The structure **`nt!_OBJECT_NAME_INFORMATION`** is used to store name information for an object. This structure contains a single field **Name** of type **`_UNICODE_STRING`**, which holds the name of the object in Unicode format. This memory address **`ffffe70967cc9000`** is stored in the **RBX** register in the previous output that we observed.

```
0: kd> dt nt!_OBJECT_NAME_INFORMATION ffffe70967cc9000
   +0x000 Name             : _UNICODE_STRING "\REGISTRY\MACHINE\SYSTEM\ControlSet001\Services\BasicDrv"
```

**`!mex.svcreg`** command shows registry information related to a service named "BasicDrv". This is useful for understanding the configuration and state of this service as it's stored in the Windows Registry.

```
0: kd> !mex.svcreg BasicDrv


Found KCB = ffffb283d29529a0 :: \REGISTRY\MACHINE\SYSTEM\CONTROLSET001\SERVICES\BASICDRV

Hive         ffffb283cd277000
KeyNode      ffffb283cf1517bc

[ValueType]         [ValueName]                   [ValueData]
REG_DWORD           Type                          1
REG_DWORD           Start                         3
REG_DWORD           ErrorControl                  1
REG_EXPAND_SZ       ImagePath                     \??\C:\Users\User\source\repos\Driver\x64\Release\Driver.sys
```

The command **`.frame /r 7`** sets the context to the **7th** frame in the stack trace and displays the register values at that moment. This function is part of the Plug and Play (PnP) subsystem in the Windows kernel and is likely responsible for calling the **DriverEntry** function of a newly loaded driver.

```
0: kd> .frame /r 0n7
07 ffff9b0b`95f7f130 fffff802`11765b23     nt!PnpCallDriverEntry+0x54
rax=0000000000008000 rbx=ffffe70967cc9000 rcx=0000000000000019
rdx=0000000000000022 rsi=ffffe7096ac873dd rdi=ffffe70968a36320
rip=fffff802117654d4 rsp=ffff9b0b95f7f130 rbp=ffff9b0b95f7f280
 r8=ffffc48068d7d000  r9=0000000000000004 r10=0000000000000000
r11=0000000000008000 r12=ffffffff8000146c r13=0000000000000002
r14=ffffe70968a36320 r15=ffffb283d6708070
iopl=0         nv up ei pl zr na po nc
cs=0010  ss=0018  ds=002b  es=002b  fs=0053  gs=002b             efl=00040246
nt!PnpCallDriverEntry+0x54:
fffff802`117654d4 8bf8            mov     edi,eax
```

This output shows the contents of a **`_DRIVER_OBJECT`** structure located at the memory address **`fffe70968a36320`**, which is stored in the **RDI** register. This is a data structure in the Windows kernel for representing a loaded driver.

```
0: kd> dt nt!_DRIVER_OBJECT ffffe70968a36320
   +0x000 Type             : 0n4
   +0x002 Size             : 0n336
   +0x008 DeviceObject     : (null) 
   +0x010 Flags            : 2
   +0x018 DriverStart      : 0xfffff802`3fbc0000 Void
   +0x020 DriverSize       : 0x7000
   +0x028 DriverSection    : 0xffffe709`675684c0 Void
   +0x030 DriverExtension  : 0xffffe709`68a36470 _DRIVER_EXTENSION
   +0x038 DriverName       : _UNICODE_STRING "\Driver\BasicDrv"
   +0x048 HardwareDatabase : 0xfffff802`11d53710 _UNICODE_STRING "\REGISTRY\MACHINE\HARDWARE\DESCRIPTION\SYSTEM"
   +0x050 FastIoDispatch   : (null) 
   +0x058 DriverInit       : 0xfffff802`3fbc1480     long  +0
   +0x060 DriverStartIo    : (null) 
   +0x068 DriverUnload     : 0xfffff802`3fbc1270     void  +0
   +0x070 MajorFunction    : [28] 0xfffff802`1134bb10     long  nt!IopInvalidDeviceRequest+0
```

The **`_PNP_WATCHDOG`** structure provides details about a Plug and Play (PnP) watchdog timer associated with the "BasicDrv" driver. The memory address **`fffe7096ac873dd`** is the **RSI** register. A **`_UNICODE_STRING`** at the **`DriverToBlame`** that points to the name of the driver to blame for the watchdog timeout, which is "BasicDrv" in this case.

```
0: kd> dt nt!_PNP_WATCHDOG ffffe7096ac873dd -r1
   +0x000 WatchdogStart    : 0x00000001`74e7ad64
   +0x008 WatchdogTimer    : 0xffffe709`6ac87350 _WDT_HANDLE
      +0x000 Reserved         : 119 'w'
   +0x010 WatchdogContextType : 5 ( PNP_DRIVER_ENTRY_WATCHDOG )
   +0x018 WatchdogContext  : 0xffff9b0b`95f7f150 Void
   +0x020 FirstChanceTriggered : 0 ''
   +0x021 TelemetryGenerated : 0 ''
   +0x028 DriverToBlame    : _UNICODE_STRING "BasicDrv"
      +0x000 Length           : 0x10
      +0x002 MaximumLength    : 0x12
      +0x008 Buffer           : 0xffffe709`6ac87415  "BasicDrv"
   +0x038 DriverToBlameBuffer : [1]  "B"
```
