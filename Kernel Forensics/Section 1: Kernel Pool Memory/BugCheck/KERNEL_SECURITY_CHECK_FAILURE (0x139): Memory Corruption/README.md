# Summary

The **`KERNEL_SECURITY_CHECK_FAILURE`** bug check has a value of **`0x139`**. This indicates that the kernel has detected the corruption of a critical data structure. This is too generic and doesn't tell us much unfortunately.

Link to this CrashDump: https://mega.nz/file/W9lHjRRR#OuyTWA4ar4nLJNY-XywbxOIJUB5CcXfeDvKSIYbnpCE

# Debug Notes

Load the CrashDump in WinDbg:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/e1ca9b3f-0a43-4323-8537-bd6327c74b6d)


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
Kernel base = 0xfffff807`1ac00000 PsLoadedModuleList = 0xfffff807`1b8130e0
Debug session time: Wed Sep 27 13:03:39.545 2023 (UTC - 7:00)
System Uptime: 0 days 0:14:35.699
SystemManufacturer = Microsoft Corporation
SystemProductName = Virtual Machine
Processor: 11th Gen Intel(R) Core(TM) i7-1185G7 @ 3.00GHz
Bugcheck: 139 (1D, FFFFB88F1487EC30, FFFFB88F1487EB88, 0)
KernelMode Dump Path: C:\Windows\MEMORY_0x139.DMP
Share Path: \\WINDEV2305EVAL\ADMIN$\MEMORY_0x139.DMP
```

Let's run the **`!analyze -v`** command to quickly triage the issue as it provides a quick and detailed overview of the state of the system at the time of the crash. It automates many aspects of crash dump analysis, providing a quick way to get valuable information.

```
0: kd> !analyze -v
*******************************************************************************
*                                                                             *
*                        Bugcheck Analysis                                    *
*                                                                             *
*******************************************************************************

KERNEL_SECURITY_CHECK_FAILURE (139)
A kernel component has corrupted a critical data structure.  The corruption
could potentially allow a malicious user to gain control of this machine.
Arguments:
Arg1: 000000000000001d, An RTL_BALANCED_NODE RBTree entry has been corrupted.
Arg2: ffffb88f1487ec30, Address of the trap frame for the exception that caused the BugCheck
Arg3: ffffb88f1487eb88, Address of the exception record for the exception that caused the BugCheck
Arg4: 0000000000000000, Reserved

Debugging Details:
------------------


KEY_VALUES_STRING: 1

    Key  : Analysis.CPU.mSec
    Value: 3218

    Key  : Analysis.Elapsed.mSec
    Value: 3935

    Key  : Analysis.IO.Other.Mb
    Value: 0

    Key  : Analysis.IO.Read.Mb
    Value: 1

    Key  : Analysis.IO.Write.Mb
    Value: 6

    Key  : Analysis.Init.CPU.mSec
    Value: 4452

    Key  : Analysis.Init.Elapsed.mSec
    Value: 727511

    Key  : Analysis.Memory.CommitPeak.Mb
    Value: 152

    Key  : Bugcheck.Code.KiBugCheckData
    Value: 0x139

    Key  : Bugcheck.Code.LegacyAPI
    Value: 0x139

    Key  : Dump.Attributes.AsUlong
    Value: 800

    Key  : FailFast.Name
    Value: INVALID_BALANCED_TREE

    Key  : FailFast.Type
    Value: 29

    Key  : Failure.Bucket
    Value: 0x139_1d_INVALID_BALANCED_TREE_Driver!unknown_function

    Key  : Failure.Hash
    Value: {8ebd2998-ea64-b96c-aae8-2108d8c92247}

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


BUGCHECK_CODE:  139

BUGCHECK_P1: 1d

BUGCHECK_P2: ffffb88f1487ec30

BUGCHECK_P3: ffffb88f1487eb88

BUGCHECK_P4: 0

FILE_IN_CAB:  MEMORY_0x139.DMP

VIRTUAL_MACHINE:  HyperV

DUMP_FILE_ATTRIBUTES: 0x800

TRAP_FRAME:  ffffb88f1487ec30 -- (.trap 0xffffb88f1487ec30)
NOTE: The trap frame does not contain all registers.
Some register values may be zeroed or incorrect.
rax=4141414141414140 rbx=0000000000000000 rcx=000000000000001d
rdx=ffff8005fed09f78 rsi=0000000000000000 rdi=0000000000000000
rip=fffff8071b09aae3 rsp=ffffb88f1487edc0 rbp=0000000000000001
 r8=ffff8005fe38b7a8  r9=0000000000000000 r10=ffff8005ef6002d0
r11=ffff8005fdfb7b28 r12=0000000000000000 r13=0000000000000000
r14=0000000000000000 r15=0000000000000000
iopl=0         nv up ei pl nz ac pe cy
nt!RtlRbRemoveNode+0x249f73:
fffff807`1b09aae3 cd29            int     29h
Resetting default scope

EXCEPTION_RECORD:  ffffb88f1487eb88 -- (.exr 0xffffb88f1487eb88)
ExceptionAddress: fffff8071b09aae3 (nt!RtlRbRemoveNode+0x0000000000249f73)
   ExceptionCode: c0000409 (Security check failure or stack buffer overrun)
  ExceptionFlags: 00000001
NumberParameters: 1
   Parameter[0]: 000000000000001d
Subcode: 0x1d FAST_FAIL_INVALID_BALANCED_TREE 

BLACKBOXBSD: 1 (!blackboxbsd)


BLACKBOXNTFS: 1 (!blackboxntfs)


BLACKBOXPNP: 1 (!blackboxpnp)


BLACKBOXWINLOGON: 1

PROCESS_NAME:  System

ERROR_CODE: (NTSTATUS) 0xc0000409 - The system detected an overrun of a stack-based buffer in this application. This overrun could potentially allow a malicious user to gain control of this application.

EXCEPTION_CODE_STR:  c0000409

EXCEPTION_PARAMETER1:  000000000000001d

EXCEPTION_STR:  0xc0000409

STACK_TEXT:  
ffffb88f`1487e908 fffff807`1b0477a9     : 00000000`00000139 00000000`0000001d ffffb88f`1487ec30 ffffb88f`1487eb88 : nt!KeBugCheckEx
ffffb88f`1487e910 fffff807`1b047d32     : ffffbd06`edf85b18 ffffbd06`edf85ab0 00000000`00000000 fffff807`1aeac0e5 : nt!KiBugCheckDispatch+0x69
ffffb88f`1487ea50 fffff807`1b045b06     : 00000000`00000000 00000000`00000000 00000000`00000000 fffff807`1ae4fdda : nt!KiFastFailDispatch+0xb2
ffffb88f`1487ec30 fffff807`1b09aae3     : ffff8005`fe38b7a0 00000000`0000017e ffff8005`fe387000 ffff8005`fe389fc0 : nt!KiRaiseSecurityCheckFailure+0x346
ffffb88f`1487edc0 fffff807`1ae50692     : 00000000`00000000 00000000`00000000 0000017c`00070000 00000000`0000017c : nt!RtlRbRemoveNode+0x249f73
ffffb88f`1487edf0 fffff807`1aead615     : ffff8005`ef6002c0 00000000`00000401 00000000`00000000 ffffb88f`1487ef00 : nt!RtlpHpVsChunkCoalesce+0x272
ffffb88f`1487ee50 fffff807`1b6ad2b0     : ffff8005`00000000 00000000`00000401 00000000`00000000 00000000`00000000 : nt!RtlpHpFreeHeap+0x385
ffffb88f`1487eef0 fffff807`3d0511b1     : 00000000`57724231 00000000`00000000 ffffbd06`00000401 00000000`000017a0 : nt!ExFreePoolWithTag+0x1a0
ffffb88f`1487ef80 fffff807`3d051251     : 00000000`00000002 fffff807`3d0532d8 fffff780`00000320 ffffffff`80001cec : Driver+0x11b1
ffffb88f`1487f090 fffff807`3d05156b     : 00000000`00000000 ffffbd06`eb912000 ffffbd06`eb912000 ffffbd06`ed143dd0 : Driver+0x1251
ffffb88f`1487f0c0 fffff807`3d0514a0     : ffffbd06`eb912000 ffffb88f`1487f280 ffffbd06`ea3e994d ffffbd06`ede649d0 : Driver+0x156b
ffffb88f`1487f100 fffff807`1b3654d4     : ffffbd06`eb912000 00000000`00000000 ffffbd06`ede649d0 fffff807`1af06b80 : Driver+0x14a0
ffffb88f`1487f130 fffff807`1b365b23     : ffffbd06`eb912000 00000000`00000000 00000000`00000000 ffff8005`fd6ce610 : nt!PnpCallDriverEntry+0x54
ffffb88f`1487f180 fffff807`1b364727     : 00000000`00000000 00000000`00000000 ffffbd06`e70ce400 fffff807`1b949ac0 : nt!IopLoadDriver+0x523
ffffb88f`1487f340 fffff807`1aeb8bd5     : ffffbd06`00000000 ffffffff`80001cec ffffbd06`e7e8f340 fffff807`00000000 : nt!IopLoadUnloadDriver+0x57
ffffb88f`1487f380 fffff807`1ae12667     : ffffbd06`e7e8f340 00000000`000003c6 ffffbd06`e7e8f340 fffff807`1aeb8a80 : nt!ExpWorkerThread+0x155
ffffb88f`1487f570 fffff807`1b0370a4     : ffffe500`178a8180 ffffbd06`e7e8f340 fffff807`1ae12610 ffffffff`ffffffff : nt!PspSystemThreadStartup+0x57
ffffb88f`1487f5c0 00000000`00000000     : ffffb88f`14880000 ffffb88f`14879000 00000000`00000000 00000000`00000000 : nt!KiStartSystemThread+0x34


SYMBOL_NAME:  Driver+11b1

MODULE_NAME: Driver

IMAGE_NAME:  Driver.sys

STACK_COMMAND:  .cxr; .ecxr ; kb

BUCKET_ID_FUNC_OFFSET:  11b1

FAILURE_BUCKET_ID:  0x139_1d_INVALID_BALANCED_TREE_Driver!unknown_function

OS_VERSION:  10.0.22621.1928

BUILDLAB_STR:  ni_release_svc_prod3

OSPLATFORM_TYPE:  x64

OSNAME:  Windows 10

FAILURE_ID_HASH:  {8ebd2998-ea64-b96c-aae8-2108d8c92247}

Followup:     MachineOwner
---------
```

The **`.trap`** output shows the trap frame, which is a data structure saved by the operating system when an exception occurs. The trap frame contains the state of the CPU registers at the time of the exception.

```
0: kd> .trap 0xffffb88f1487ec30
NOTE: The trap frame does not contain all registers.
Some register values may be zeroed or incorrect.
rax=4141414141414140 rbx=0000000000000000 rcx=000000000000001d
rdx=ffff8005fed09f78 rsi=0000000000000000 rdi=0000000000000000
rip=fffff8071b09aae3 rsp=ffffb88f1487edc0 rbp=0000000000000001
 r8=ffff8005fe38b7a8  r9=0000000000000000 r10=ffff8005ef6002d0
r11=ffff8005fdfb7b28 r12=0000000000000000 r13=0000000000000000
r14=0000000000000000 r15=0000000000000000
iopl=0         nv up ei pl nz ac pe cy
nt!RtlRbRemoveNode+0x249f73:
fffff807`1b09aae3 cd29            int     29h
```

The **`rax`** value seems suspicious **`(rax=4141414141414140)`** and might indicate that some memory corruption (potentially a buffer overflow) has occurred. The **`int 0x29`** instruction is used to trigger a "fast fail" mechanism in Windows. The "fast fail" mechanism is designed to immediately terminate a process when severe corruption or a security issue is detected.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/7407a868-c998-4b69-8b35-278ebe2148c6)


The output from the **`.exr`** command is showing the details of an exception record. It indicates that there's been a critical failure in the system involving a corrupted balanced tree data structure within the **`nt!RtlRbRemoveNode`** function. 

```
0: kd> .exr 0xffffb88f1487eb88
ExceptionAddress: fffff8071b09aae3 (nt!RtlRbRemoveNode+0x0000000000249f73)
   ExceptionCode: c0000409 (Security check failure or stack buffer overrun)
  ExceptionFlags: 00000001
NumberParameters: 1
   Parameter[0]: 000000000000001d
Subcode: 0x1d FAST_FAIL_INVALID_BALANCED_TREE
```

This thread that has been running on **CPU 0** is the responsible thread that was crashing:

```
0: kd> !t
Process                   Thread                       CID       UserTime KernelTime ContextSwitches Wait Reason Time State
System (ffffbd06e70eb040) ffffbd06e7e8f340 (E|K|W|R|V) 4.1c4            0      219ms            7875 Executive      0 Running on processor 0

Priority:
    Current Base Decrement ForegroundBoost IO Page
    12      12   0         0               0  5

 # Child-SP         Return           Call Site
 0 ffffb88f1487e908 fffff8071b0477a9 nt!KeBugCheckEx
 1 ffffb88f1487e910 fffff8071b047d32 nt!KiBugCheckDispatch+0x69
 2 ffffb88f1487ea50 fffff8071b045b06 nt!KiFastFailDispatch+0xb2
 3 ffffb88f1487ec30 fffff8071b09aae3 nt!KiRaiseSecurityCheckFailure+0x346
 4 ffffb88f1487edc0 fffff8071ae50692 nt!RtlRbRemoveNode+0x249f73
 5 ffffb88f1487edf0 fffff8071aead615 nt!RtlpHpVsChunkCoalesce+0x272
 6 ffffb88f1487ee50 fffff8071b6ad2b0 nt!RtlpHpFreeHeap+0x385
 7 ffffb88f1487eef0 fffff8073d0511b1 nt!ExFreePoolWithTag+0x1a0
 8 ffffb88f1487ef80 fffff8073d051251 Driver+0x11b1
 9 ffffb88f1487f090 fffff8073d05156b Driver+0x1251
 a ffffb88f1487f0c0 fffff8073d0514a0 Driver+0x156b
 b ffffb88f1487f100 fffff8071b3654d4 Driver+0x14a0
 c ffffb88f1487f130 fffff8071b365b23 nt!PnpCallDriverEntry+0x54
 d ffffb88f1487f180 fffff8071b364727 nt!IopLoadDriver+0x523
 e ffffb88f1487f340 fffff8071aeb8bd5 nt!IopLoadUnloadDriver+0x57
 f ffffb88f1487f380 fffff8071ae12667 nt!ExpWorkerThread+0x155
10 ffffb88f1487f570 fffff8071b0370a4 nt!PspSystemThreadStartup+0x57
11 ffffb88f1487f5c0 0000000000000000 nt!KiStartSystemThread+0x34
```

The **`uf`** command stands for "Unassemble Function". It is used to display the disassembled code of a function, starting from the specified address. The output shows the assembly instructions that make up the function.

```
0: kd> uf Driver+0x11b1
Driver+0x1000:
fffff807`3d051000 48895c2408      mov     qword ptr [rsp+8],rbx
fffff807`3d051005 4889742418      mov     qword ptr [rsp+18h],rsi
fffff807`3d05100a 55              push    rbp
fffff807`3d05100b 57              push    rdi
fffff807`3d05100c 4156            push    r14
fffff807`3d05100e 488d6c24b9      lea     rbp,[rsp-47h]
fffff807`3d051013 4881ecf0000000  sub     rsp,0F0h
fffff807`3d05101a 488b05df1f0000  mov     rax,qword ptr [Driver+0x3000 (fffff807`3d053000)]
fffff807`3d051021 4833c4          xor     rax,rsp
fffff807`3d051024 4889453f        mov     qword ptr [rbp+3Fh],rax
fffff807`3d051028 0f1005910e0000  movups  xmm0,xmmword ptr [Driver+0x1ec0 (fffff807`3d051ec0)]
fffff807`3d05102f 8b05b30e0000    mov     eax,dword ptr [Driver+0x1ee8 (fffff807`3d051ee8)]
fffff807`3d051035 bf30000000      mov     edi,30h
fffff807`3d05103a 0f100d8f0e0000  movups  xmm1,xmmword ptr [Driver+0x1ed0 (fffff807`3d051ed0)]
fffff807`3d051041 894537          mov     dword ptr [rbp+37h],eax
fffff807`3d051044 6603cf          add     cx,di
fffff807`3d051047 0f11450f        movups  xmmword ptr [rbp+0Fh],xmm0
fffff807`3d05104b 4883c8ff        or      rax,0FFFFFFFFFFFFFFFFh
fffff807`3d05104f 488bda          mov     rbx,rdx
fffff807`3d051052 f20f1005860e0000 movsd   xmm0,mmword ptr [Driver+0x1ee0 (fffff807`3d051ee0)]
fffff807`3d05105a 488d550f        lea     rdx,[rbp+0Fh]
fffff807`3d05105e f20f11452f      movsd   mmword ptr [rbp+2Fh],xmm0
fffff807`3d051063 4533f6          xor     r14d,r14d
fffff807`3d051066 0f114d1f        movups  xmmword ptr [rbp+1Fh],xmm1

Driver+0x106a:
fffff807`3d05106a 48ffc0          inc     rax
fffff807`3d05106d 6644393442      cmp     word ptr [rdx+rax*2],r14w
fffff807`3d051072 75f6            jne     Driver+0x106a (fffff807`3d05106a)  Branch

Driver+0x1074:
fffff807`3d051074 66894c4505      mov     word ptr [rbp+rax*2+5],cx
fffff807`3d051079 488d550f        lea     rdx,[rbp+0Fh]
fffff807`3d05107d 488d4dbf        lea     rcx,[rbp-41h]
fffff807`3d051081 ff15c90f0000    call    qword ptr [Driver+0x2050 (fffff807`3d052050)]
fffff807`3d051087 4489742450      mov     dword ptr [rsp+50h],r14d
fffff807`3d05108c 488d45bf        lea     rax,[rbp-41h]
fffff807`3d051090 4c89742448      mov     qword ptr [rsp+48h],r14
fffff807`3d051095 4c8d4dcf        lea     r9,[rbp-31h]
fffff807`3d051099 c744244020000000 mov     dword ptr [rsp+40h],20h
fffff807`3d0510a1 4c8d45df        lea     r8,[rbp-21h]
fffff807`3d0510a5 c744243805000000 mov     dword ptr [rsp+38h],5
fffff807`3d0510ad 488d4db7        lea     rcx,[rbp-49h]
fffff807`3d0510b1 4489742430      mov     dword ptr [rsp+30h],r14d
fffff807`3d0510b6 0f57c0          xorps   xmm0,xmm0
fffff807`3d0510b9 c744242880000000 mov     dword ptr [rsp+28h],80h
fffff807`3d0510c1 ba00000040      mov     edx,40000000h
fffff807`3d0510c6 4c89742420      mov     qword ptr [rsp+20h],r14
fffff807`3d0510cb 897ddf          mov     dword ptr [rbp-21h],edi
fffff807`3d0510ce 4c8975e7        mov     qword ptr [rbp-19h],r14
fffff807`3d0510d2 c745f740020000  mov     dword ptr [rbp-9],240h
fffff807`3d0510d9 488945ef        mov     qword ptr [rbp-11h],rax
fffff807`3d0510dd f30f7f45ff      movdqu  xmmword ptr [rbp-1],xmm0
fffff807`3d0510e2 ff15700f0000    call    qword ptr [Driver+0x2058 (fffff807`3d052058)]
fffff807`3d0510e8 85c0            test    eax,eax
fffff807`3d0510ea 7908            jns     Driver+0x10f4 (fffff807`3d0510f4)  Branch

Driver+0x10ec:
fffff807`3d0510ec 0faee8          lfence
fffff807`3d0510ef e9d7000000      jmp     Driver+0x11cb (fffff807`3d0511cb)  Branch

Driver+0x10f4:
fffff807`3d0510f4 be00010000      mov     esi,100h
fffff807`3d0510f9 41b831427257    mov     r8d,57724231h
fffff807`3d0510ff 8bce            mov     ecx,esi
fffff807`3d051101 488bd3          mov     rdx,rbx
fffff807`3d051104 ff155e0f0000    call    qword ptr [Driver+0x2068 (fffff807`3d052068)]
fffff807`3d05110a 488bf8          mov     rdi,rax
fffff807`3d05110d 4885c0          test    rax,rax
fffff807`3d051110 7514            jne     Driver+0x1126 (fffff807`3d051126)  Branch

Driver+0x1112:
fffff807`3d051112 488b4db7        mov     rcx,qword ptr [rbp-49h]
fffff807`3d051116 ff152c0f0000    call    qword ptr [Driver+0x2048 (fffff807`3d052048)]
fffff807`3d05111c b89a0000c0      mov     eax,0C000009Ah
fffff807`3d051121 e9a5000000      jmp     Driver+0x11cb (fffff807`3d0511cb)  Branch

Driver+0x1126:
fffff807`3d051126 488bd3          mov     rdx,rbx
fffff807`3d051129 41b831427043    mov     r8d,43704231h
fffff807`3d05112f 48d1ea          shr     rdx,1
fffff807`3d051132 488bce          mov     rcx,rsi
fffff807`3d051135 ff152d0f0000    call    qword ptr [Driver+0x2068 (fffff807`3d052068)]
fffff807`3d05113b 488bf0          mov     rsi,rax
fffff807`3d05113e 488bcf          mov     rcx,rdi
fffff807`3d051141 4885c0          test    rax,rax
fffff807`3d051144 7512            jne     Driver+0x1158 (fffff807`3d051158)  Branch

Driver+0x1146:
fffff807`3d051146 ba31427257      mov     edx,57724231h
fffff807`3d05114b ff150f0f0000    call    qword ptr [Driver+0x2060 (fffff807`3d052060)]
fffff807`3d051151 bb9a0000c0      mov     ebx,0C000009Ah
fffff807`3d051156 eb67            jmp     Driver+0x11bf (fffff807`3d0511bf)  Branch

Driver+0x1158:
fffff807`3d051158 4c8bc3          mov     r8,rbx
fffff807`3d05115b ba41000000      mov     edx,41h
fffff807`3d051160 e89b080000      call    Driver+0x1a00 (fffff807`3d051a00)
fffff807`3d051165 4c8bc3          mov     r8,rbx
fffff807`3d051168 488bd7          mov     rdx,rdi
fffff807`3d05116b 488bce          mov     rcx,rsi
fffff807`3d05116e e88d0a0000      call    Driver+0x1c00 (fffff807`3d051c00)
fffff807`3d051173 488b4db7        mov     rcx,qword ptr [rbp-49h]
fffff807`3d051177 488d45cf        lea     rax,[rbp-31h]
fffff807`3d05117b 4c89742440      mov     qword ptr [rsp+40h],r14
fffff807`3d051180 4533c9          xor     r9d,r9d
fffff807`3d051183 4c89742438      mov     qword ptr [rsp+38h],r14
fffff807`3d051188 4533c0          xor     r8d,r8d
fffff807`3d05118b 895c2430        mov     dword ptr [rsp+30h],ebx
fffff807`3d05118f 33d2            xor     edx,edx
fffff807`3d051191 48897c2428      mov     qword ptr [rsp+28h],rdi
fffff807`3d051196 4889442420      mov     qword ptr [rsp+20h],rax
fffff807`3d05119b ff158f0e0000    call    qword ptr [Driver+0x2030 (fffff807`3d052030)]
fffff807`3d0511a1 ba31427257      mov     edx,57724231h
fffff807`3d0511a6 488bcf          mov     rcx,rdi
fffff807`3d0511a9 8bd8            mov     ebx,eax
fffff807`3d0511ab ff15af0e0000    call    qword ptr [Driver+0x2060 (fffff807`3d052060)]
fffff807`3d0511b1 ba31427043      mov     edx,43704231h
fffff807`3d0511b6 488bce          mov     rcx,rsi
fffff807`3d0511b9 ff15a10e0000    call    qword ptr [Driver+0x2060 (fffff807`3d052060)]

Driver+0x11bf:
fffff807`3d0511bf 488b4db7        mov     rcx,qword ptr [rbp-49h]
fffff807`3d0511c3 ff157f0e0000    call    qword ptr [Driver+0x2048 (fffff807`3d052048)]
fffff807`3d0511c9 8bc3            mov     eax,ebx

Driver+0x11cb:
fffff807`3d0511cb 488b4d3f        mov     rcx,qword ptr [rbp+3Fh]
fffff807`3d0511cf 4833cc          xor     rcx,rsp
fffff807`3d0511d2 e8b9000000      call    Driver+0x1290 (fffff807`3d051290)
fffff807`3d0511d7 4c8d9c24f0000000 lea     r11,[rsp+0F0h]
fffff807`3d0511df 498b5b20        mov     rbx,qword ptr [r11+20h]
fffff807`3d0511e3 498b7330        mov     rsi,qword ptr [r11+30h]
fffff807`3d0511e7 498be3          mov     rsp,r11
fffff807`3d0511ea 415e            pop     r14
fffff807`3d0511ec 5f              pop     rdi
fffff807`3d0511ed 5d              pop     rbp
fffff807`3d0511ee c3              ret
```

Some quick-wins when it comes down to hunting specific patterns. The **`!grep`** command in **MEX** is used to search for a specific string pattern within a given set of data. 

We are searching for the string **`"0C00000"`** within the output of the **`uf Driver+0x11b1`** command. The **`-r`** flag tells **`!grep`** to treat the pattern **`"0C00000`** as a regular expression. Since **`"0C00000"`** is not a regular expression special character, it just matches the string **`"0C00000"`** literally.

```
0: kd> !grep -r "0C00000" uf Driver+0x11b1
fffff807`3d05111c b89a0000c0      mov     eax,0C000009Ah
fffff807`3d051151 bb9a0000c0      mov     ebx,0C000009Ah
```

The value **`0xC000009A`** is a status code that indicates that an operation could not be completed due to insufficient system resources.

```
0: kd> !error 0C000009Ah
Error code: (NTSTATUS) 0xc000009a (3221225626) - Insufficient system resources exist to complete the API.
```

The **`!grep -r "call qword ptr" uf Driver+0x11b1`** command is used to search for instances of the string **`"call qword ptr"`** within the output of the **`uf Driver+0x11b1`** command. This is useful for identifying all the indirect function calls.

```
0: kd> !grep -r "call    qword ptr" uf Driver+0x11b1
fffff807`3d051081 ff15c90f0000    call    qword ptr [Driver+0x2050 (fffff807`3d052050)]
fffff807`3d0510e2 ff15700f0000    call    qword ptr [Driver+0x2058 (fffff807`3d052058)]
fffff807`3d051104 ff155e0f0000    call    qword ptr [Driver+0x2068 (fffff807`3d052068)]
fffff807`3d051116 ff152c0f0000    call    qword ptr [Driver+0x2048 (fffff807`3d052048)]
fffff807`3d051135 ff152d0f0000    call    qword ptr [Driver+0x2068 (fffff807`3d052068)]
fffff807`3d05114b ff150f0f0000    call    qword ptr [Driver+0x2060 (fffff807`3d052060)]
fffff807`3d05119b ff158f0e0000    call    qword ptr [Driver+0x2030 (fffff807`3d052030)]
fffff807`3d0511ab ff15af0e0000    call    qword ptr [Driver+0x2060 (fffff807`3d052060)]
fffff807`3d0511b9 ff15a10e0000    call    qword ptr [Driver+0x2060 (fffff807`3d052060)]
fffff807`3d0511c3 ff157f0e0000    call    qword ptr [Driver+0x2048 (fffff807`3d052048)]
```

Let's dump the call stack of the crashing thread again.

```
0: kd> knL
 # Child-SP          RetAddr               Call Site
00 ffffb88f`1487e908 fffff807`1b0477a9     nt!KeBugCheckEx
01 ffffb88f`1487e910 fffff807`1b047d32     nt!KiBugCheckDispatch+0x69
02 ffffb88f`1487ea50 fffff807`1b045b06     nt!KiFastFailDispatch+0xb2
03 ffffb88f`1487ec30 fffff807`1b09aae3     nt!KiRaiseSecurityCheckFailure+0x346
04 ffffb88f`1487edc0 fffff807`1ae50692     nt!RtlRbRemoveNode+0x249f73
05 ffffb88f`1487edf0 fffff807`1aead615     nt!RtlpHpVsChunkCoalesce+0x272
06 ffffb88f`1487ee50 fffff807`1b6ad2b0     nt!RtlpHpFreeHeap+0x385
07 ffffb88f`1487eef0 fffff807`3d0511b1     nt!ExFreePoolWithTag+0x1a0
08 ffffb88f`1487ef80 fffff807`3d051251     Driver+0x11b1
09 ffffb88f`1487f090 fffff807`3d05156b     Driver+0x1251
0a ffffb88f`1487f0c0 fffff807`3d0514a0     Driver+0x156b
0b ffffb88f`1487f100 fffff807`1b3654d4     Driver+0x14a0
0c ffffb88f`1487f130 fffff807`1b365b23     nt!PnpCallDriverEntry+0x54
0d ffffb88f`1487f180 fffff807`1b364727     nt!IopLoadDriver+0x523
0e ffffb88f`1487f340 fffff807`1aeb8bd5     nt!IopLoadUnloadDriver+0x57
0f ffffb88f`1487f380 fffff807`1ae12667     nt!ExpWorkerThread+0x155
10 ffffb88f`1487f570 fffff807`1b0370a4     nt!PspSystemThreadStartup+0x57
11 ffffb88f`1487f5c0 00000000`00000000     nt!KiStartSystemThread+0x34
```

The command **`.frame /r 0n8`** selects the 8th frame in the current thread's stack and displays its details, including the value of all registers at the point when the frame was created. This can be useful for debugging to understand the state of the system at a particular point in the call stack.

```
0: kd> .frame /r 0n8
08 ffffb88f`1487ef80 fffff807`3d051251     Driver+0x11b1
rax=4141414141414140 rbx=0000000000000000 rcx=000000000000001d
rdx=ffff8005fed09f78 rsi=ffff8005fed09380 rdi=ffff8005fe38a000
rip=fffff8073d0511b1 rsp=ffffb88f1487ef80 rbp=ffffb88f1487f029
 r8=ffff8005fe38b7a8  r9=0000000000000000 r10=ffff8005ef6002d0
r11=ffff8005fdfb7b28 r12=ffffffff80001cec r13=0000000000000002
r14=0000000000000000 r15=ffff8005fdef0f20
iopl=0         nv up ei ng nz na pe nc
cs=0010  ss=0018  ds=002b  es=002b  fs=0053  gs=002b             efl=00040282
Driver+0x11b1:
fffff807`3d0511b1 ba31427043      mov     edx,43704231h
```

The value **`(000000000000001d)`** in the **`rcx`** register is similar to the exception code in the bugcheck. Here we can confirm it as well when we run the **`.bugcheck`** command. The **`.bugcheck`** command displays the bug check code and its parameters, which were captured when the system crashed.

```
0: kd> .bugcheck
Bugcheck code 00000139
Arguments 00000000`0000001d ffffb88f`1487ec30 ffffb88f`1487eb88 00000000`00000000
```
