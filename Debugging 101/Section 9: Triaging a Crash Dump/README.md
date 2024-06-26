# Description

Triaging a crash dump requires a blend of technical knowledge and analytical skills.  Each crash dump can present a unique challenge, which provides an new opportunity to learn more about the inner workings of the OS.

- **Windows Internals**: Proficiency in Windows internals, including how the operating system handles processes, threads, memory and drivers.
- **Code Structure and Logic:** Crash dump analysis often involves examining function calls and understanding how different parts of the code interact. This requires a good grasp of how functions are structured and called in programming.
- **Attention to detail:** Diligence in examining the crash dump, noting important details like error codes, memory addresses, call stacks, etc.

Having (some) programming knowledge helps you make better guesses about what caused the problem. This can lead you to focus on the right areas when you're trying to figure out why the crash happened.

# How to triage a Crash Dump?

Before diving into this, it is recommended for newcomers to Debugging or Windows Internals to explore fundamental concepts first:

- **Section 1:** Basic Concepts & System Architecture
- **Section 2:** Process & Threads
- **Section 3:** Memory
- **Section 4:** System Mechanism
- **Section 5:** Triaging a thread from a Memory Dump


Load the Memory Dump in WinDbg:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/6d17903a-7c21-487f-b89d-3bfd6bfbc22e)


Let's start with the **`!mex.di`** command which stands for dump information. This command retrieves information about the machine from which the memory dump originated. It's especially handy when working with multiple memory dumps, as it helps distinguish the analysis related to each specific dump. This is an improved version of the **`vertarget`** command.

```
2: kd> !mex.di
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
```

The **`!smt`** command provides information about the processors in a system.  It shows that the system has 4 logical processors, grouped into pairs (0 and 1, 2 and 3) utilizing SMT, with each physical processor containing 2 cores, each capable of handling 2 logical processors.

```
0: kd> !smt
SMT Summary:
------------

KeActiveProcessors:
****------------------------------------------------------------ (000000000000000f)
IdleSummary:
---------------------------------------------------------------- (0000000000000000)
 No PRCB             SMT Set                                                                             APIC Id
  0 fffff8052661b180 **-------------------------------------------------------------- (0000000000000003) 0x00000000
  1 ffff9581fbaa9180 **-------------------------------------------------------------- (0000000000000003) 0x00000001
  2 ffff9581fbb94180 --**------------------------------------------------------------ (000000000000000c) 0x00000002
  3 ffff9581fbbdb180 --**------------------------------------------------------------ (000000000000000c) 0x00000003

Maximum cores per physical processor:   2
Maximum logical processors per core:    2
```

Most people will start with running the **`!analyze -v`** command to quickly triage the issue as it provides a quick and detailed overview of the state of the system at the time of the crash. It automates many aspects of crash dump analysis, providing a quick way to get valuable information. 

```
0: kd> !analyze -v
*******************************************************************************
*                                                                             *
*                        Bugcheck Analysis                                    *
*                                                                             *
*******************************************************************************

DRIVER_IRQL_NOT_LESS_OR_EQUAL (d1)
An attempt was made to access a pageable (or completely invalid) address at an
interrupt request level (IRQL) that is too high.  This is usually
caused by drivers using improper addresses.
If kernel debugger is available get stack backtrace.
Arguments:
Arg1: ffff8001acdc6010, memory referenced
Arg2: 0000000000000002, IRQL
Arg3: 0000000000000000, value 0 = read operation, 1 = write operation
Arg4: fffff805736312d0, address which referenced memory

Debugging Details:
------------------


KEY_VALUES_STRING: 1

    Key  : Analysis.CPU.mSec
    Value: 1937

    Key  : Analysis.Elapsed.mSec
    Value: 2196

    Key  : Analysis.IO.Other.Mb
    Value: 0

    Key  : Analysis.IO.Read.Mb
    Value: 0

    Key  : Analysis.IO.Write.Mb
    Value: 6

    Key  : Analysis.Init.CPU.mSec
    Value: 983

    Key  : Analysis.Init.Elapsed.mSec
    Value: 81154

    Key  : Analysis.Memory.CommitPeak.Mb
    Value: 151

    Key  : Bugcheck.Code.KiBugCheckData
    Value: 0xd1

    Key  : Bugcheck.Code.LegacyAPI
    Value: 0xd1

    Key  : Dump.Attributes.AsUlong
    Value: 800

    Key  : Failure.Bucket
    Value: AV_myfault!unknown_function

    Key  : Failure.Hash
    Value: {9745090a-9bce-ccba-c096-ca6e9ca04c64}

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
    Value: 10.0.22621.2506


BUGCHECK_CODE:  d1

BUGCHECK_P1: ffff8001acdc6010

BUGCHECK_P2: 2

BUGCHECK_P3: 0

BUGCHECK_P4: fffff805736312d0

FILE_IN_CAB:  MEMORY.DMP

VIRTUAL_MACHINE:  HyperV

DUMP_FILE_ATTRIBUTES: 0x800

READ_ADDRESS:  ffff8001acdc6010 Paged pool

BLACKBOXBSD: 1 (!blackboxbsd)


BLACKBOXNTFS: 1 (!blackboxntfs)


BLACKBOXPNP: 1 (!blackboxpnp)


BLACKBOXWINLOGON: 1

PROCESS_NAME:  notmyfault64.exe

DEVICE_OBJECT: ffffc983f0533770

DRIVER_OBJECT: 0000000000000000

TRAP_FRAME:  ffffc60b87e8ed40 -- (.trap 0xffffc60b87e8ed40)
NOTE: The trap frame does not contain all registers.
Some register values may be zeroed or incorrect.
rax=0000000001580702 rbx=0000000000000000 rcx=0000000000000000
rdx=ffff8001adc67ff0 rsi=0000000000000000 rdi=0000000000000000
rip=fffff805736312d0 rsp=ffffc60b87e8eed0 rbp=ffffc60b87e8f141
 r8=0000000000000002  r9=ffff8001acd00000 r10=ffff80019be00300
r11=ffff8001acdbbff0 r12=0000000000000000 r13=0000000000000000
r14=0000000000000000 r15=0000000000000000
iopl=0         nv up ei pl zr na po nc
myfault+0x12d0:
fffff805`736312d0 8b03            mov     eax,dword ptr [rbx] ds:00000000`00000000=????????
Resetting default scope

STACK_TEXT:  
ffffc60b`87e8ebf8 fffff805`4d62bfa9     : 00000000`0000000a ffff8001`acdc6010 00000000`00000002 00000000`00000000 : nt!KeBugCheckEx
ffffc60b`87e8ec00 fffff805`4d627634     : 00000000`00000000 ffffc983`00000001 00000000`00000000 00000000`00000000 : nt!KiBugCheckDispatch+0x69
ffffc60b`87e8ed40 fffff805`736312d0     : ffffc983`656e6f4e ffff8001`acdbc010 ffffc983`00000003 00000000`00000880 : nt!KiPageFault+0x474
ffffc60b`87e8eed0 fffff805`7363168e     : ffffc983`01580702 00000000`00000aa0 ffffc983`f4584030 ffffc60b`87e8f050 : myfault+0x12d0
ffffc60b`87e8ef00 fffff805`736317f1     : ffffc983`f0533770 00000000`00000006 00000000`00000000 00000000`00000001 : myfault+0x168e
ffffc60b`87e8f040 fffff805`4d453075     : ffffc983`f0533770 00000000`00000002 00000000`00000000 00000000`000009d2 : myfault+0x17f1
ffffc60b`87e8f0a0 fffff805`4d8e26d0     : ffffc983`f0533770 ffffc60b`87e8f141 ffffc983`f4d13e40 ffffc983`f4d13e40 : nt!IofCallDriver+0x55
ffffc60b`87e8f0e0 fffff805`4d8e4100     : ffffc983`f0533770 00000000`00000000 ffffc60b`87e8f405 ffffc983`f4d13e40 : nt!IopSynchronousServiceTail+0x1d0
ffffc60b`87e8f190 fffff805`4d8e39e6     : 00000000`00000000 00000000`00003a98 00000000`00000000 00000000`00000000 : nt!IopXxxControlFile+0x700
ffffc60b`87e8f380 fffff805`4d62b6e5     : ffffffff`ee1e5d00 00000000`00000000 ffffc983`eded00b0 00007ffc`00000000 : nt!NtDeviceIoControlFile+0x56
ffffc60b`87e8f3f0 00007ffc`aa52f454     : 00000000`00000000 00000000`00000000 00000000`00000000 00000000`00000000 : nt!KiSystemServiceCopyEnd+0x25
00000008`f82ff3f8 00000000`00000000     : 00000000`00000000 00000000`00000000 00000000`00000000 00000000`00000000 : 0x00007ffc`aa52f454


SYMBOL_NAME:  myfault+12d0

MODULE_NAME: myfault

IMAGE_NAME:  myfault.sys

STACK_COMMAND:  .cxr; .ecxr ; kb

BUCKET_ID_FUNC_OFFSET:  12d0

FAILURE_BUCKET_ID:  AV_myfault!unknown_function

OS_VERSION:  10.0.22621.2506

BUILDLAB_STR:  ni_release_svc_prod3

OSPLATFORM_TYPE:  x64

OSNAME:  Windows 10

FAILURE_ID_HASH:  {9745090a-9bce-ccba-c096-ca6e9ca04c64}

Followup:     MachineOwner
---------
```

If the **`!analyze -v`** command doesn't reveal the root cause of the crash, what alternative methods can we use to dive deeper into the crash dump? 

The first command will be **`!mex.tl -t`** command examines the state and wait reasons for every thread across all processes, presenting the results as statistics. This allows for a quick overview, such as determining how many processes have running, ready, blocking threads, and so on. It's like having a satellite view of all the thread state that are associated with the processes.

```
0: kd> !mex.tl -t
PID            Address          Name                                     !! Rn Ry Bk Lc IO Er
============== ================ ======================================== == == == == == == ==
0x0    0n0     fffff8054df49f40 Idle                                      .  5  7  .  .  .  .
0x4    0n4     ffffc983ebceb040 System                                    6  .  .  .  .  1  .
0x80   0n128   ffffc983ebce5080 Registry                                  .  .  .  .  .  .  .
0x234  0n564   ffffc983ef15f040 smss.exe                                  .  .  .  1  .  .  .
0x2c0  0n704   ffffc983ecfa6140 csrss.exe                                 .  .  .  .  1  .  .
0x308  0n776   ffffc983f05df080 wininit.exe                               .  .  .  .  .  .  .
0x31c  0n796   ffffc983f05ee140 csrss.exe                                 .  .  .  .  1  .  .
0x370  0n880   ffffc983f0619080 winlogon.exe                              .  .  .  .  .  .  .
0x398  0n920   ffffc983f0625080 services.exe                              .  .  .  .  .  .  .
0x3b4  0n948   ffffc983f04da080 LsaIso.exe                                .  .  .  .  .  .  .
0x3c0  0n960   ffffc983ecce9080 lsass.exe                                 .  .  .  .  .  .  .
0x300  0n768   ffffc983f068a080 svchost.exe                               .  .  .  .  .  .  .
0x3cc  0n972   ffffc983f099a080 fontdrvhost.exe                           .  .  .  .  .  .  .
0x3d4  0n980   ffffc983f06d30c0 fontdrvhost.exe                           .  .  .  .  .  .  .
0x45c  0n1116  ffffc983f0a8d0c0 svchost.exe                               .  .  .  .  .  .  .
0x488  0n1160  ffffc983f06be080 svchost.exe                               .  .  .  .  .  .  .
0x4e0  0n1248  ffffc983f0a4e0c0 dwm.exe                                   .  1  .  .  .  .  .
0x528  0n1320  ffffc983f0a6f080 svchost.exe                               .  .  .  .  .  .  .
0x554  0n1364  ffffc983f0955080 svchost.exe                               .  .  .  .  .  .  .
0x580  0n1408  ffffc983f0aa3080 svchost.exe                               .  .  .  .  .  .  .
0x594  0n1428  ffffc983f0aa7080 svchost.exe                               .  .  .  .  .  .  .
0x600  0n1536  ffffc983f0836080 svchost.exe                               .  .  .  .  .  .  .
0x620  0n1568  ffffc983f0acf080 svchost.exe                               .  .  .  .  .  .  .
0x654  0n1620  ffffc983f0d19080 svchost.exe                               .  .  .  .  .  .  .
0x660  0n1632  ffffc983f0d1f0c0 svchost.exe                               .  .  .  .  .  .  .
0x678  0n1656  ffffc983f0d24080 svchost.exe                               .  .  .  .  .  .  .
0x68c  0n1676  ffffc983f0769080 svchost.exe                               .  .  .  .  .  .  .
0x6c4  0n1732  ffffc983f07bd080 svchost.exe                               .  .  .  .  .  .  .
0x6cc  0n1740  ffffc983f07bf080 svchost.exe                               .  .  .  .  .  .  .
0x6dc  0n1756  ffffc983f0da00c0 svchost.exe                               .  .  .  .  .  .  .
0x6ec  0n1772  ffffc983f0da6080 svchost.exe                               .  .  .  .  .  .  .
0x78c  0n1932  ffffc983f085b0c0 svchost.exe                               .  .  .  .  .  .  .
0x7bc  0n1980  ffffc983f0879080 svchost.exe                               .  .  .  .  1  .  .
0x81c  0n2076  ffffc983f08f6080 svchost.exe                               .  .  .  .  .  .  .
0x840  0n2112  ffffc983f08bf080 VSSVC.exe                                 .  .  .  .  .  .  .
0x868  0n2152  ffffc983f08c8080 svchost.exe                               .  .  .  .  .  .  .
0x89c  0n2204  ffffc983f0c020c0 svchost.exe                               .  .  .  .  .  .  .
0x964  0n2404  ffffc983f08c7080 svchost.exe                               .  .  .  .  .  .  .
0x970  0n2416  ffffc983f0cb5080 svchost.exe                               .  .  .  .  .  .  .
0x998  0n2456  ffffc983f0cbc080 svchost.exe                               .  .  .  .  .  .  .
0x9a0  0n2464  ffffc983f0cbf080 svchost.exe                               .  .  .  .  .  .  .
0x9b0  0n2480  ffffc983f0cca080 svchost.exe                               .  .  .  .  .  .  .
0xa24  0n2596  ffffc983f0b40080 svchost.exe                               .  .  .  .  .  .  .
0xa34  0n2612  ffffc983f0b4c080 MemCompression                            .  1  .  .  .  .  .
0xa94  0n2708  ffffc983f0e3c080 svchost.exe                               .  .  .  .  .  .  .
0xaa4  0n2724  ffffc983f0e40080 svchost.exe                               .  .  .  .  .  .  .
0xab0  0n2736  ffffc983f0e4e080 svchost.exe                               .  .  .  .  .  .  .
0xb08  0n2824  ffffc983f0bed080 svchost.exe                               .  .  .  .  .  .  .
0xb78  0n2936  ffffc983ebd2e080 svchost.exe                               .  .  .  .  .  .  .
0xb80  0n2944  ffffc983ebd2a080 svchost.exe                               .  .  .  .  .  .  .
0xbb0  0n2992  ffffc983ebcb3080 svchost.exe                               .  .  .  .  .  .  .
0xbd4  0n3028  ffffc983ebdf0080 svchost.exe                               .  .  .  .  .  .  .
0xbf0  0n3056  ffffc983ebdc7080 svchost.exe                               .  .  .  .  .  .  .
0xc3c  0n3132  ffffc983f20760c0 svchost.exe                               .  .  .  .  .  .  .
0xc44  0n3140  ffffc983f0fd90c0 svchost.exe                               .  .  .  .  .  .  .
0xc90  0n3216  ffffc983f060c080 svchost.exe                               .  .  .  .  .  .  .
0xc98  0n3224  ffffc983f060a080 svchost.exe                               .  .  .  .  .  .  .
0xca0  0n3232  ffffc983f2006080 svchost.exe                               .  .  .  .  .  .  .
0xd18  0n3352  ffffc983f20f7080 svchost.exe                               .  .  .  .  .  .  .
0xd88  0n3464  ffffc983f21020c0 svchost.exe                               .  .  .  .  .  .  .
0xdfc  0n3580  ffffc983f21bc0c0 spoolsv.exe                               .  .  .  .  .  .  .
0xed8  0n3800  ffffc983f22240c0 svchost.exe                               .  .  .  .  .  .  .
0xeec  0n3820  ffffc983f2411080 svchost.exe                               .  .  .  .  .  .  .
0xef4  0n3828  ffffc983f2455080 svchost.exe                               .  .  .  .  .  .  .
0xefc  0n3836  ffffc983f2410080 AnyDesk.exe*32                            .  .  .  .  .  .  .
0xf04  0n3844  ffffc983f208e080 svchost.exe                               .  .  .  .  .  .  .
0xf10  0n3856  ffffc983f2454080 IpOverUsbSvc.exe*32                       .  .  .  .  .  .  .
0xf1c  0n3868  ffffc983f21e9080 sqlwriter.exe                             .  .  .  .  .  .  .
0xf44  0n3908  ffffc983f2222080 RustDesk.exe                              .  .  .  .  .  .  .
0xfac  0n4012  ffffc983f25ab0c0 MsMpEng.exe                               .  1  .  .  .  .  .
0xfb8  0n4024  ffffc983f2344080 svchost.exe                               .  .  .  .  .  .  .
0xfc0  0n4032  ffffc983f2355080 wlms.exe                                  .  .  .  .  .  .  .
0xfd0  0n4048  ffffc983f2364080 svchost.exe                               .  .  .  .  .  .  .
0xfe8  0n4072  ffffc983f2376080 svchost.exe                               .  .  .  .  .  .  .
0xff0  0n4080  ffffc983f2377080 svchost.exe                               .  .  .  .  .  .  .
0x101c 0n4124  ffffc983f2555080 svchost.exe                               .  .  .  .  .  .  .
0x1170 0n4464  ffffc983f276b080 sppsvc.exe                                .  .  .  .  .  .  .
0x1264 0n4708  ffffc983f28020c0 svchost.exe                               .  .  .  .  .  .  .
0x132c 0n4908  ffffc983f2820080 unsecapp.exe                              .  .  .  .  .  .  .
0x14ac 0n5292  ffffc983f2a30080 SppExtComObj.Exe                          .  .  .  .  .  .  .
0x14e8 0n5352  ffffc983f2a64080 svchost.exe                               .  .  .  .  .  .  .
0x15ac 0n5548  ffffc983f2a6a080 AggregatorHost.exe                        .  .  .  .  .  .  .
0x15ec 0n5612  ffffc983f2aea0c0 svchost.exe                               .  .  .  .  .  .  .
0x16fc 0n5884  ffffc983f2c320c0 svchost.exe                               .  .  .  .  .  .  .
0x17f0 0n6128  ffffc983f2d79080 sihost.exe                                .  .  .  .  .  .  .
0x2c4  0n708   ffffc983f2d78080 svchost.exe                               .  .  .  .  .  .  .
0x704  0n1796  ffffc983f2d770c0 svchost.exe                               .  .  .  .  .  .  .
0xf30  0n3888  ffffc983f2d64080 svchost.exe                               .  .  .  .  .  .  .
0xcb8  0n3256  ffffc983f2d61080 svchost.exe                               .  .  .  .  .  .  .
0x12f8 0n4856  ffffc983f2d5e080 taskhostw.exe                             .  .  .  .  .  .  .
0x18f4 0n6388  ffffc983f2d59080 svchost.exe                               .  .  .  .  .  .  .
0x196c 0n6508  ffffc983f2f18080 explorer.exe                              2  .  .  .  1  .  .
0x1b50 0n6992  ffffc983f2d5b080 svchost.exe                               .  .  .  .  .  .  .
0x1bf4 0n7156  ffffc983f2d60080 svchost.exe                               .  .  .  .  .  .  .
0x18e8 0n6376  ffffc983f2a69080 svchost.exe                               .  .  .  .  .  .  .
0x19e0 0n6624  ffffc983f2f15080 SearchHost.exe                            7  .  .  .  2  .  .
0x19d8 0n6616  ffffc983f2f14080 StartMenuExperienceHost.exe               .  .  .  .  .  .  .
0x155c 0n5468  ffffc983f2f09080 Widgets.exe                               .  .  .  .  .  .  .
0x1c3c 0n7228  ffffc983f3af5080 RuntimeBroker.exe                         .  .  .  .  .  .  .
0x1cac 0n7340  ffffc983f351a080 RuntimeBroker.exe                         .  .  .  .  1  .  .
0x1d34 0n7476  ffffc983f36c0080 svchost.exe                               .  .  .  .  .  .  .
0x1d70 0n7536  ffffc983f39f1080 svchost.exe                               .  .  .  .  .  .  .
0x1ef0 0n7920  ffffc983f385e080 dllhost.exe                               .  .  .  .  .  .  .
0x1e70 0n7792  ffffc983f36680c0 ctfmon.exe                                .  .  .  .  .  .  .
0x2020 0n8224  ffffc983f3e450c0 SecurityHealthSystray.exe                 .  .  .  .  .  .  .
0x2034 0n8244  ffffc983f3e560c0 SecurityHealthService.exe                 .  .  .  .  .  .  .
0x2128 0n8488  ffffc983f34ce080 NisSrv.exe                                .  .  .  .  .  .  .
0x2280 0n8832  ffffc983f3a84080 TextInputHost.exe                         .  .  .  .  .  .  .
0xa00  0n2560  ffffc983f47220c0 svchost.exe                               .  .  .  .  .  .  .
0x1eb4 0n7860  ffffc983f47270c0 SearchIndexer.exe                         .  .  .  .  .  .  .
0x1d64 0n7524  ffffc983f38e40c0 DbgSvc.exe                                .  .  .  .  .  .  .
0x1d40 0n7488  ffffc983ecd3f0c0 dllhost.exe                               .  .  .  .  .  .  .
0x1e5c 0n7772  ffffc983ef7300c0 msdtc.exe                                 .  .  .  1  .  .  .
0x258  0n600   ffffc983f3a9c0c0 svchost.exe                               .  .  .  .  .  .  .
0x358  0n856   ffffc983f3aa10c0 svchost.exe                               .  .  .  .  .  .  .
0x21b4 0n8628  ffffc983efd4d0c0 uhssvc.exe                                .  .  .  .  .  .  .
0x2288 0n8840  ffffc983f3a690c0 svchost.exe                               .  .  .  .  .  .  .
0x1e40 0n7744  ffffc983efc660c0 MoUsoCoreWorker.exe                       .  .  .  .  .  .  .
0x1274 0n4724  ffffc983f3518080 svchost.exe                               .  .  .  .  .  .  .
0x21e8 0n8680  ffffc983efcd7080 csrss.exe                                 .  .  .  .  1  .  .
0xbac  0n2988  ffffc983efcd6080 winlogon.exe                              .  .  .  .  1  .  .
0xf7c  0n3964  ffffc983ef86c080 WmiPrvSE.exe                              .  .  .  .  .  .  .
0x1af4 0n6900  ffffc983f22020c0 LogonUI.exe                               .  .  .  .  .  .  .
0x1e24 0n7716  ffffc983ef86b080 dwm.exe                                   .  .  .  .  .  .  .
0x1cb4 0n7348  ffffc983f3aa0080 fontdrvhost.exe                           .  .  .  .  .  .  .
0x1b40 0n6976  ffffc983f26e6080 RustDesk.exe                              .  .  .  .  .  .  .
0xb10  0n2832  ffffc983f3a9f080 rdpclip.exe                               .  .  .  .  .  .  .
0xe28  0n3624  ffffc983ecd4a080 WUDFHost.exe                              .  .  .  .  .  .  .
0xecc  0n3788  ffffc983ecd49080 svchost.exe                               .  .  .  .  .  .  .
0x15e4 0n5604  ffffc983f36d30c0 svchost.exe                               .  .  .  .  .  .  .
0x350  0n848   ffffc983ef9a80c0 rdpinput.exe                              .  .  .  .  .  .  .
0xeb4  0n3764  ffffc983f3844080 TabTip.exe                                .  .  .  .  .  .  .
0x954  0n2388  ffffc983f365c0c0 svchost.exe                               .  .  .  .  .  .  .
0x1cf8 0n7416  ffffc983efcd50c0 ShellExperienceHost.exe                   3  .  .  .  .  .  .
0x11b4 0n4532  ffffc983f0b740c0 svchost.exe                               .  .  .  .  .  .  .
0x2138 0n8504  ffffc983ecdd90c0 svchost.exe                               .  .  .  .  .  .  .
0x858  0n2136  ffffc983ece4e0c0 WidgetService.exe                         .  .  .  .  .  .  .
0x1394 0n5012  ffffc983f052a0c0 svchost.exe                               .  .  .  .  .  .  .
0x23f8 0n9208  ffffc983f3fe70c0 RuntimeBroker.exe                         .  .  .  .  1  .  .
0x2268 0n8808  ffffc983f3fec0c0 SystemSettingsBroker.exe                  .  .  .  .  2  .  .
0x2084 0n8324  ffffc983f3441180 svchost.exe                               .  .  .  .  .  .  .
0x23d8 0n9176  ffffc983ecf540c0 svchost.exe                               .  .  .  .  .  .  .
0x2518 0n9496  ffffc983f2074140 smartscreen.exe                           .  .  .  .  .  .  .
0x24a8 0n9384  ffffc983ef3a50c0 svchost.exe                               .  .  .  .  .  .  .
0x1a0c 0n6668  ffffc983ebee30c0 devenv.exe                                .  .  .  2  .  .  .
0x122c 0n4652  ffffc983ed2180c0 PerfWatson2.exe                           .  .  .  2  .  .  .
0x1be4 0n7140  ffffc983edfab0c0 Microsoft.ServiceHub.Controller.exe       .  .  .  1  .  .  .
0x19d4 0n6612  ffffc983efadd0c0 ServiceHub.VSDetouredHost.exe             .  .  .  1  .  .  .
0x1420 0n5152  ffffc983ef4c4080 ServiceHub.Host.dotnet.x64.exe            .  .  .  2  .  .  .
0x19bc 0n6588  ffffc983efc9c080 ServiceHub.SettingsHost.exe               .  .  .  2  .  .  .
0x1b88 0n7048  ffffc983eda770c0 ServiceHub.Host.netfx.x86.exe*32          .  .  .  4  .  .  .
0x2614 0n9748  ffffc983f42610c0 ServiceHub.ThreadedWaitDialog.exe         .  .  .  1  .  .  .
0x1660 0n5728  ffffc983efc6d140 ServiceHub.IndexingService.exe            .  .  .  1  .  .  .
0xda8  0n3496  ffffc983eddc70c0 ServiceHub.IntellicodeModelService.exe    .  .  .  .  .  .  .
0x1428 0n5160  ffffc983f41720c0 vshost.exe                                1  .  .  .  .  .  .
0x1ce8 0n7400  ffffc983f40ae0c0 vcpkgsrv.exe*32                           .  .  .  .  .  .  .
0x8e0  0n2272  ffffc983f42350c0 ServiceHub.RoslynCodeAnalysisService.exe  .  .  .  2  .  .  .
0x2658 0n9816  ffffc983edd020c0 ServiceHub.Host.AnyCPU.exe                .  .  .  1  .  .  .
0x1210 0n4624  ffffc983ed5e60c0 ServiceHub.TestWindowStoreHost.exe        .  .  .  1  .  .  .
0xcb0  0n3248  ffffc983edfd90c0 vcpkgsrv.exe*32                           .  .  .  .  .  .  .
0x24c0 0n9408  ffffc983ec692080 MSBuild.exe                               .  .  .  .  .  .  .
0x24c4 0n9412  ffffc983ecff0080 conhost.exe                               .  .  .  .  .  .  .
0x1bc4 0n7108  ffffc983ece85080 Notepad.exe                               .  .  .  .  .  .  .
0x2984 0n10628 ffffc983eda99080 audiodg.exe                               .  .  .  .  .  .  .
0x2a1c 0n10780 ffffc983ede2a080 cmd.exe                                   .  .  .  .  .  .  .
0x2a24 0n10788 ffffc983ed19e080 conhost.exe                               .  .  .  .  .  .  .
0x1050 0n4176  ffffc983edbe90c0 svchost.exe                               .  .  .  .  .  .  .
0x1544 0n5444  ffffc983f36c5080 svchost.exe                               .  .  .  .  .  .  .
0x10a4 0n4260  ffffc983edfae080 notmyfault64.exe                          .  1  .  .  .  .  .
============== ================ ======================================== == == == == == == ==
PID            Address          Name                                     !! Rn Ry Bk Lc IO Er

Warning! Zombie process(es) detected (not displayed). Count: 5 [zombie report]
```

The second command that is very useful is the **`!mex.trep`** command, which provides a quick overview of the number of threads in various states within the system.

```
0: kd> !mex.trep
Count State
===== =======
    1 Ready
    4 Running
1,831 Waiting
===== =======
1,836
```

The first thing we need to understand is the thread **state**. The **state** of a thread will determine how you analyze everything else. If the thread is in a **waiting** state, you need to know how long it has been waiting and why it is waiting. If it is **running**, you want to know how long it has been running. However, we don't really need to know why it is running; it's on a CPU and executing.

From the output, we can see that there are a couple of **running** threads. Let's examine the running threads on the system. The output of the **`!mex.running`** command in provides a snapshot of the currently running threads on the system. 

Examining threads in a running state is the most important thing to do first, because they (may) contain the immediate context and operations that led to the crash, allowing for a direct insight into the cause of the issue.

```
0: kd> !mex.running
Process           PID Thread             Id Pri Base Pri Next CPU CSwitches   User     Kernel State      Time Reason
================ ==== ================ ==== === ======== ======== ========= ====== ========== ======= ======= ===========
MsMpEng.exe       fac ffffc983ebd6e080 18cc   9        8        0      7857  922ms      250ms Running       0 UserRequest
Idle                0 ffffc983ebd89080    0   0        0        1    278333      0 15m:37.641 Running   125ms Executive
notmyfault64.exe 10a4 ffffc983f08a0080  504   9        8        2        37      0          0 Running       0 WrLpcReply
MemCompression    a34 ffffc983f0be7240  a6c   7        7        3     39550      0      797ms Running       0 Executive
dwm.exe           4e0 ffffc983f28a1080  a04  15       15        4      4598 1s.000      297ms Running       0 UserRequest
Idle                0 ffffc983ebcc7080    0  63        0        5     90506      0 16m:10.656 Running 16s.859 Executive

Count: 6 | Show Unique Stacks
```

Besides of examing threads in a running state, we should also take a look at the **readyish** threads. These are threads that are prepared to run and are queued for CPU time but are not currently executing. They are in a state where they are waiting for their turn to use the CPU, having all necessary resources to execute but are pending scheduling by the system's thread scheduler. 

Examine the **`Ry`** (Readyish) column in the **`!mex.tl -t`** output. Are there any threads in a **`Readyish`** state, excluding those from the **Idle** process?

```
0: kd> !mex.tl -t
PID            Address          Name                                     !! Rn Ry Bk Lc IO Er
============== ================ ======================================== == == == == == == ==
0x0    0n0     fffff8054df49f40 Idle                                      .  5  12  .  .  .  .
```

We can observe **several** threads in this example that are in a **`readyish`**. When we say that a thread is in a **ready** state, it means that it has completed its previous task or is waiting for a new task to be assigned. The thread is placed in the ready queue, waiting for the CPU scheduler to select it for execution. It is not currently running because another thread is using the CPU.

When a thread is **stalled for more than 30 seconds**, it means that the thread has been in the ready state (or a similar state like standby) but has not been scheduled to run for over 30 seconds. It's worth to investigate these threads and prioritize them, because if a thread is ready to run but is not getting CPU time, the tasks it is supposed to perform are delayed. This can lead to overall system sluggishness.

```
0: kd> !mex.ready
Process                            PID Thread             Id Pri Base Pri Affinity Next CPU Ideal CPU CSwitches   User  Kernel State      Time Reason
================================= ==== ================ ==== === ======== ======== ======== ========= ========= ====== ======= ===== ========= ==============
System                               4 ffff988da1ed8040 4cf4  30       12        1        0         0         1      0       0 Ready 1m:03.843 WrPreempted
svchost.exe                        61c ffff988d8bd0b0c0  af8  15       15        1        0         0      4474   16ms       0 Ready 1m:59.953 UserRequest
Corsair.Service.CpuIdRemote64.exe 28c0 ffff988d93206080 2988  15       15        1        0         0     12398      0    31ms Ready 1m:59.515 DelayExecution
svchost.exe                        61c ffff988d91492080 1f10  15       15        1        0         0       478      0       0 Ready 1m:59.390 UserRequest
System                               4 ffff988d99c52040 3290  13       12        1        0         0      9445      0    47ms Ready 1m:13.703 WrPreempted
WmiPrvSE.exe                      112c ffff988d9e83f080 13c0  12        8        1        0         0     89264 1s.641  4s.906 Ready   13s.203 WrPreempted
System                               4 ffff988d7d5ce040  18c   8        8        1        0         0  12365520      0 43s.484 Ready 2m:00.000 Executive
Corsair.Service.CpuIdRemote64.exe 28c0 ffff988d9321a080 2970   8        8        1        0         0     16754   16ms   172ms Ready 1m:59.375 DelayExecution
System                               4 ffff988d9acb3040 2698   2        2        1        0         0     22510      0   156ms Ready   19s.781 WrPreempted
System                               4 ffff988d96ae0040 4b48   2        2        1        0         0     14306      0    16ms Ready   19s.781 WrPreempted
System                               4 ffff988d92fb1040 21fc   2        2        1        0         0     14181      0   109ms Ready   19s.781 WrPreempted
System                               4 ffff988d77be4040   5c   0        0        1        0         0     29946      0   891ms Ready 1m:59.000 Executive

Count: 12
```

After completing the previous step, examine the states of other threads. Are there any threads currently blocked or waiting for an LPC from processes that are of interest, such as **System.exe**, **svchost.exe**, **lsass.exe**, **services.exe**, or **smss.exe** for example?

```
0: kd> !mex.tl -t
PID            Address          Name                                     !! Rn Ry Bk Lc IO Er
```

- **Rn:** Threads that are running
- **Ry:** Threads that are currently in a readyish state
- **Bk:** Threads that are blocked (e.g. waiting on a Mutex or Pushlock)
- **Lc:** Threads waiting on a response from a server thread
- **IO:** Threads that are waiting to page in and out
- **Er:** Threads waiting on an ERESOURCE

Another aspect worth exploring is the number of threads within each process. For instance, are there processes containing over 500 threads? While it is not by definition a problem, such a high thread count is worth further investigation.

```
0: kd> !mex.tl -sort thread
PID            Address          Name                                     Thd
============== ================ ======================================== ===
0xc3c  0n3132  ffffc983f20760c0 svchost.exe                                1
0x1428 0n5160  ffffc983f41720c0 vshost.exe                                 1
0x101c 0n4124  ffffc983f2555080 svchost.exe                                1
0x258  0n600   ffffc983f3a9c0c0 svchost.exe                                1
0x2084 0n8324  ffffc983f3441180 svchost.exe                                1
0x1264 0n4708  ffffc983f28020c0 svchost.exe                                1
0x2a1c 0n10780 ffffc983ede2a080 cmd.exe                                    1
0x868  0n2152  ffffc983f08c8080 svchost.exe                                1
0x15ac 0n5548  ffffc983f2a6a080 AggregatorHost.exe                         1
0x14ac 0n5292  ffffc983f2a30080 SppExtComObj.Exe                           1
0x594  0n1428  ffffc983f0aa7080 svchost.exe                                1
0x2020 0n8224  ffffc983f3e450c0 SecurityHealthSystray.exe                  1
0x580  0n1408  ffffc983f0aa3080 svchost.exe                                1
0x1d34 0n7476  ffffc983f36c0080 svchost.exe                                1
0x600  0n1536  ffffc983f0836080 svchost.exe                                2
0xa94  0n2708  ffffc983f0e3c080 svchost.exe                                2
0xc44  0n3140  ffffc983f0fd90c0 svchost.exe                                2
0x660  0n1632  ffffc983f0d1f0c0 svchost.exe                                2
0x24c4 0n9412  ffffc983ecff0080 conhost.exe                                2
0x3b4  0n948   ffffc983f04da080 LsaIso.exe                                 2
0xfc0  0n4032  ffffc983f2355080 wlms.exe                                   2
0xf1c  0n3868  ffffc983f21e9080 sqlwriter.exe                              2
0x840  0n2112  ffffc983f08bf080 VSSVC.exe                                  2
0xbb0  0n2992  ffffc983ebcb3080 svchost.exe                                2
0xa24  0n2596  ffffc983f0b40080 svchost.exe                                2
0x704  0n1796  ffffc983f2d770c0 svchost.exe                                2
0x68c  0n1676  ffffc983f0769080 svchost.exe                                3
0x6c4  0n1732  ffffc983f07bd080 svchost.exe                                3
0x554  0n1364  ffffc983f0955080 svchost.exe                                3
0xcb8  0n3256  ffffc983f2d61080 svchost.exe                                3
0x6ec  0n1772  ffffc983f0da6080 svchost.exe                                3
0x1c3c 0n7228  ffffc983f3af5080 RuntimeBroker.exe                          3
0xecc  0n3788  ffffc983ecd49080 svchost.exe                                3
0x998  0n2456  ffffc983f0cbc080 svchost.exe                                3
0x21b4 0n8628  ffffc983efd4d0c0 uhssvc.exe                                 3
0x6dc  0n1756  ffffc983f0da00c0 svchost.exe                                3
0x2138 0n8504  ffffc983ecdd90c0 svchost.exe                                3
0x234  0n564   ffffc983ef15f040 smss.exe                                   3
0x350  0n848   ffffc983ef9a80c0 rdpinput.exe                               3
0x24a8 0n9384  ffffc983ef3a50c0 svchost.exe                                3
0x132c 0n4908  ffffc983f2820080 unsecapp.exe                               3
0x1544 0n5444  ffffc983f36c5080 svchost.exe                                3
0x308  0n776   ffffc983f05df080 wininit.exe                                3
0xbd4  0n3028  ffffc983ebdf0080 svchost.exe                                3
0xfd0  0n4048  ffffc983f2364080 svchost.exe                                3
0xf44  0n3908  ffffc983f2222080 RustDesk.exe                               3
0x80   0n128   ffffc983ebce5080 Registry                                   4
0x10a4 0n4260  ffffc983edfae080 notmyfault64.exe                           4
0xf7c  0n3964  ffffc983ef86c080 WmiPrvSE.exe                               4
0x9a0  0n2464  ffffc983f0cbf080 svchost.exe                                4
0x2c4  0n708   ffffc983f2d78080 svchost.exe                                4
0x78c  0n1932  ffffc983f085b0c0 svchost.exe                                4
0x2128 0n8488  ffffc983f34ce080 NisSrv.exe                                 4
0x2a24 0n10788 ffffc983ed19e080 conhost.exe                                4
0x954  0n2388  ffffc983f365c0c0 svchost.exe                                4
0xf30  0n3888  ffffc983f2d64080 svchost.exe                                4
0x2268 0n8808  ffffc983f3fec0c0 SystemSettingsBroker.exe                   4
0x18e8 0n6376  ffffc983f2a69080 svchost.exe                                4
0x1394 0n5012  ffffc983f052a0c0 svchost.exe                                4
0xed8  0n3800  ffffc983f22240c0 svchost.exe                                5
0x2984 0n10628 ffffc983eda99080 audiodg.exe                                5
0xefc  0n3836  ffffc983f2410080 AnyDesk.exe*32                             5
0x1d70 0n7536  ffffc983f39f1080 svchost.exe                                5
0xd18  0n3352  ffffc983f20f7080 svchost.exe                                5
0x370  0n880   ffffc983f0619080 winlogon.exe                               5
0xa00  0n2560  ffffc983f47220c0 svchost.exe                                5
0x678  0n1656  ffffc983f0d24080 svchost.exe                                5
0x964  0n2404  ffffc983f08c7080 svchost.exe                                5
0x3cc  0n972   ffffc983f099a080 fontdrvhost.exe                            5
0xfe8  0n4072  ffffc983f2376080 svchost.exe                                5
0x1bf4 0n7156  ffffc983f2d60080 svchost.exe                                5
0x654  0n1620  ffffc983f0d19080 svchost.exe                                5
0x1170 0n4464  ffffc983f276b080 sppsvc.exe                                 5
0x858  0n2136  ffffc983ece4e0c0 WidgetService.exe                          5
0xb78  0n2936  ffffc983ebd2e080 svchost.exe                                5
0xab0  0n2736  ffffc983f0e4e080 svchost.exe                                5
0xb08  0n2824  ffffc983f0bed080 svchost.exe                                5
0x14e8 0n5352  ffffc983f2a64080 svchost.exe                                5
0xbac  0n2988  ffffc983efcd6080 winlogon.exe                               5
0x16fc 0n5884  ffffc983f2c320c0 svchost.exe                                5
0x488  0n1160  ffffc983f06be080 svchost.exe                                5
0x970  0n2416  ffffc983f0cb5080 svchost.exe                                5
0x18f4 0n6388  ffffc983f2d59080 svchost.exe                                5
0x1cb4 0n7348  ffffc983f3aa0080 fontdrvhost.exe                            5
0x3d4  0n980   ffffc983f06d30c0 fontdrvhost.exe                            5
0x0    0n0     fffff8054df49f40 Idle                                       6
0x23f8 0n9208  ffffc983f3fe70c0 RuntimeBroker.exe                          6
0xaa4  0n2724  ffffc983f0e40080 svchost.exe                                6
0xbf0  0n3056  ffffc983ebdc7080 svchost.exe                                6
0xff0  0n4080  ffffc983f2377080 svchost.exe                                6
0x2518 0n9496  ffffc983f2074140 smartscreen.exe                            6
0xeb4  0n3764  ffffc983f3844080 TabTip.exe                                 7
0x620  0n1568  ffffc983f0acf080 svchost.exe                                7
0x1ef0 0n7920  ffffc983f385e080 dllhost.exe                                7
0x1274 0n4724  ffffc983f3518080 svchost.exe                                7
0x155c 0n5468  ffffc983f2f09080 Widgets.exe                                8
0x12f8 0n4856  ffffc983f2d5e080 taskhostw.exe                              8
0x15ec 0n5612  ffffc983f2aea0c0 svchost.exe                                8
0xdfc  0n3580  ffffc983f21bc0c0 spoolsv.exe                                8
0xb80  0n2944  ffffc983ebd2a080 svchost.exe                                8
0x81c  0n2076  ffffc983f08f6080 svchost.exe                                8
0xd88  0n3464  ffffc983f21020c0 svchost.exe                                8
0x2034 0n8244  ffffc983f3e560c0 SecurityHealthService.exe                  8
0xcb0  0n3248  ffffc983edfd90c0 vcpkgsrv.exe*32                            8
0x1ce8 0n7400  ffffc983f40ae0c0 vcpkgsrv.exe*32                            8
0xda8  0n3496  ffffc983eddc70c0 ServiceHub.IntellicodeModelService.exe     8
0x1e40 0n7744  ffffc983efc660c0 MoUsoCoreWorker.exe                        8
0x2288 0n8840  ffffc983f3a690c0 svchost.exe                                8
0xef4  0n3828  ffffc983f2455080 svchost.exe                                9
0x1050 0n4176  ffffc983edbe90c0 svchost.exe                                9
0x1e5c 0n7772  ffffc983ef7300c0 msdtc.exe                                  9
0x3c0  0n960   ffffc983ecce9080 lsass.exe                                  9
0xf10  0n3856  ffffc983f2454080 IpOverUsbSvc.exe*32                        9
0x11b4 0n4532  ffffc983f0b740c0 svchost.exe                                9
0x9b0  0n2480  ffffc983f0cca080 svchost.exe                                9
0x89c  0n2204  ffffc983f0c020c0 svchost.exe                               10
0xfb8  0n4024  ffffc983f2344080 svchost.exe                               10
0x398  0n920   ffffc983f0625080 services.exe                              10
0xe28  0n3624  ffffc983ecd4a080 WUDFHost.exe                              10
0x24c0 0n9408  ffffc983ec692080 MSBuild.exe                               10
0xeec  0n3820  ffffc983f2411080 svchost.exe                               10
0x7bc  0n1980  ffffc983f0879080 svchost.exe                               11
0x300  0n768   ffffc983f068a080 svchost.exe                               11
0x1cac 0n7340  ffffc983f351a080 RuntimeBroker.exe                         11
0xc98  0n3224  ffffc983f060a080 svchost.exe                               11
0x2658 0n9816  ffffc983edd020c0 ServiceHub.Host.AnyCPU.exe                11
0x17f0 0n6128  ffffc983f2d79080 sihost.exe                                11
0x1210 0n4624  ffffc983ed5e60c0 ServiceHub.TestWindowStoreHost.exe        11
0x1eb4 0n7860  ffffc983f47270c0 SearchIndexer.exe                         12
0x1420 0n5152  ffffc983ef4c4080 ServiceHub.Host.dotnet.x64.exe            12
0x1d40 0n7488  ffffc983ecd3f0c0 dllhost.exe                               12
0x1d64 0n7524  ffffc983f38e40c0 DbgSvc.exe                                12
0x45c  0n1116  ffffc983f0a8d0c0 svchost.exe                               12
0x6cc  0n1740  ffffc983f07bf080 svchost.exe                               12
0x23d8 0n9176  ffffc983ecf540c0 svchost.exe                               12
0x1b50 0n6992  ffffc983f2d5b080 svchost.exe                               12
0x1660 0n5728  ffffc983efc6d140 ServiceHub.IndexingService.exe            13
0xb10  0n2832  ffffc983f3a9f080 rdpclip.exe                               13
0x21e8 0n8680  ffffc983efcd7080 csrss.exe                                 13
0x2c0  0n704   ffffc983ecfa6140 csrss.exe                                 13
0x358  0n856   ffffc983f3aa10c0 svchost.exe                               13
0xca0  0n3232  ffffc983f2006080 svchost.exe                               13
0x8e0  0n2272  ffffc983f42350c0 ServiceHub.RoslynCodeAnalysisService.exe  13
0x1e70 0n7792  ffffc983f36680c0 ctfmon.exe                                14
0x1be4 0n7140  ffffc983edfab0c0 Microsoft.ServiceHub.Controller.exe       14
0x19d4 0n6612  ffffc983efadd0c0 ServiceHub.VSDetouredHost.exe             14
0x31c  0n796   ffffc983f05ee140 csrss.exe                                 15
0x1b40 0n6976  ffffc983f26e6080 RustDesk.exe                              15
0x19d8 0n6616  ffffc983f2f14080 StartMenuExperienceHost.exe               15
0x1af4 0n6900  ffffc983f22020c0 LogonUI.exe                               15
0xf04  0n3844  ffffc983f208e080 svchost.exe                               17
0xc90  0n3216  ffffc983f060c080 svchost.exe                               17
0x1e24 0n7716  ffffc983ef86b080 dwm.exe                                   17
0x19bc 0n6588  ffffc983efc9c080 ServiceHub.SettingsHost.exe               18
0x1cf8 0n7416  ffffc983efcd50c0 ShellExperienceHost.exe                   22
0x15e4 0n5604  ffffc983f36d30c0 svchost.exe                               22
0x1bc4 0n7108  ffffc983ece85080 Notepad.exe                               22
0x1b88 0n7048  ffffc983eda770c0 ServiceHub.Host.netfx.x86.exe*32          23
0x4e0  0n1248  ffffc983f0a4e0c0 dwm.exe                                   26
0xa34  0n2612  ffffc983f0b4c080 MemCompression                            26
0x2614 0n9748  ffffc983f42610c0 ServiceHub.ThreadedWaitDialog.exe         27
0x122c 0n4652  ffffc983ed2180c0 PerfWatson2.exe                           28
0x528  0n1320  ffffc983f0a6f080 svchost.exe                               36
0xfac  0n4012  ffffc983f25ab0c0 MsMpEng.exe                               38
0x2280 0n8832  ffffc983f3a84080 TextInputHost.exe                         40
0x19e0 0n6624  ffffc983f2f15080 SearchHost.exe                            63
0x1a0c 0n6668  ffffc983ebee30c0 devenv.exe                                64
0x196c 0n6508  ffffc983f2f18080 explorer.exe                             137
0x4    0n4     ffffc983ebceb040 System                                   257
============== ================ ======================================== ===
PID            Address          Name                                     Thd
```

If we're troubleshooting a memory leak and need to determine the memory usage of processes, we can use the **`!mex.tl -mem`** command. For each process, the output lists various memory metrics such as Virtual Memory (VM), Peak Memory Usage, Shared Memory, AWE (Address Windowing Extensions) Size, Commit Size, and quotas for both paged (PP) and non-paged (NPP) pool resources.

```
0: kd> !mex.tl -mem
PID            Address          Name                                            VM      Peak    Shared Awe Size Commit Size   PP Quota NPP Quota
============== ================ ======================================== ========= ========= ========= ======== =========== ========== =========
0x0    0n0     fffff8054df49f40 Idle                                          8 KB      8 KB                  0       60 KB                272 B
0x4    0n4     ffffc983ebceb040 System                                        4 MB  86.52 MB    296 KB        0       44 KB                272 B
0x80   0n128   ffffc983ebce5080 Registry                                    153 MB  153.1 MB                  0    18.62 MB  310.13 KB  14.34 KB
0x234  0n564   ffffc983ef15f040 smss.exe                                      2 TB      2 TB    132 KB        0     1.14 MB   13.39 KB   3.95 KB
0x2c0  0n704   ffffc983ecfa6140 csrss.exe                                     2 TB      2 TB   8.86 MB        0     2.27 MB  250.85 KB  25.81 KB
0x308  0n776   ffffc983f05df080 wininit.exe                                   2 TB      2 TB   1.76 MB        0     1.57 MB   72.05 KB  12.05 KB
0x31c  0n796   ffffc983f05ee140 csrss.exe                                     2 TB      2 TB  60.79 MB        0     2.35 MB  299.78 KB  21.28 KB
0x370  0n880   ffffc983f0619080 winlogon.exe                                  2 TB      2 TB  10.68 MB        0     2.64 MB  160.84 KB   13.6 KB
0x398  0n920   ffffc983f0625080 services.exe                                  2 TB      2 TB    240 KB        0     5.62 MB   166.6 KB  13.86 KB
0x3b4  0n948   ffffc983f04da080 LsaIso.exe                                 4.05 GB   4.05 GB      4 KB        0      1.2 MB   28.26 KB   6.37 KB
0x3c0  0n960   ffffc983ecce9080 lsass.exe                                     2 TB      2 TB   2.11 MB        0     9.18 MB  181.66 KB  26.95 KB
0x300  0n768   ffffc983f068a080 svchost.exe                                   2 TB      2 TB    2.2 MB        0     9.03 MB  496.38 KB  25.05 KB
0x3cc  0n972   ffffc983f099a080 fontdrvhost.exe                               2 TB      2 TB    228 KB        0     1.38 MB   38.76 KB   6.37 KB
0x3d4  0n980   ffffc983f06d30c0 fontdrvhost.exe                               2 TB      2 TB    228 KB        0     1.83 MB   43.69 KB    6.9 KB
0x45c  0n1116  ffffc983f0a8d0c0 svchost.exe                                   2 TB      2 TB   2.29 MB        0     6.74 MB     173 KB  23.79 KB
0x488  0n1160  ffffc983f06be080 svchost.exe                                   2 TB      2 TB   1.82 MB        0     3.12 MB  149.73 KB   15.8 KB
0x4e0  0n1248  ffffc983f0a4e0c0 dwm.exe                                       2 TB      2 TB 178.16 MB        0   209.39 MB 1011.73 KB  91.08 KB
0x528  0n1320  ffffc983f0a6f080 svchost.exe                                   2 TB      2 TB 117.52 MB        0    162.2 MB  407.59 KB  34.05 KB
0x554  0n1364  ffffc983f0955080 svchost.exe                                   2 TB      2 TB   1.78 MB        0      1.2 MB   58.65 KB   8.08 KB
0x580  0n1408  ffffc983f0aa3080 svchost.exe                                   2 TB      2 TB   1.82 MB        0     2.38 MB   91.97 KB  12.19 KB
0x594  0n1428  ffffc983f0aa7080 svchost.exe                                   2 TB      2 TB   1.79 MB        0     1.82 MB  100.66 KB   9.55 KB
0x600  0n1536  ffffc983f0836080 svchost.exe                                   2 TB      2 TB   1.78 MB        0     1.63 MB   59.63 KB   7.56 KB
0x620  0n1568  ffffc983f0acf080 svchost.exe                                   2 TB      2 TB   1.82 MB        0     5.83 MB  274.14 KB  20.13 KB
0x654  0n1620  ffffc983f0d19080 svchost.exe                                   2 TB      2 TB   1.82 MB        0     2.44 MB   119.6 KB  11.55 KB
0x660  0n1632  ffffc983f0d1f0c0 svchost.exe                                   2 TB      2 TB   1.78 MB        0     4.51 MB   57.88 KB  24.56 KB
0x678  0n1656  ffffc983f0d24080 svchost.exe                                   2 TB      2 TB   1.82 MB        0     1.81 MB   62.05 KB   9.95 KB
0x68c  0n1676  ffffc983f0769080 svchost.exe                                   2 TB      2 TB   1.81 MB        0      1.3 MB    60.2 KB   8.61 KB
0x6c4  0n1732  ffffc983f07bd080 svchost.exe                                   2 TB      2 TB   1.81 MB        0     1.23 MB   58.89 KB   8.09 KB
0x6cc  0n1740  ffffc983f07bf080 svchost.exe                                   2 TB      2 TB   1.81 MB        0     2.09 MB   77.41 KB  11.41 KB
0x6dc  0n1756  ffffc983f0da00c0 svchost.exe                                   2 TB      2 TB   1.82 MB        0     1.43 MB   61.27 KB   8.89 KB
0x6ec  0n1772  ffffc983f0da6080 svchost.exe                                   2 TB      2 TB   1.78 MB        0     1.27 MB    58.9 KB   8.23 KB
0x78c  0n1932  ffffc983f085b0c0 svchost.exe                                   2 TB      2 TB   1.82 MB        0     2.12 MB   79.24 KB  10.35 KB
0x7bc  0n1980  ffffc983f0879080 svchost.exe                                   2 TB      2 TB   2.08 MB        0     5.17 MB  142.33 KB  20.51 KB
0x81c  0n2076  ffffc983f08f6080 svchost.exe                                   2 TB      2 TB   1.84 MB        0     2.59 MB  129.89 KB   35.8 KB
0x840  0n2112  ffffc983f08bf080 VSSVC.exe                                     2 TB      2 TB   1.82 MB        0     1.73 MB   78.75 KB  10.22 KB
0x868  0n2152  ffffc983f08c8080 svchost.exe                                   2 TB      2 TB   1.78 MB        0     1.23 MB    60.7 KB    7.7 KB
0x89c  0n2204  ffffc983f0c020c0 svchost.exe                                   2 TB      2 TB   1.78 MB        0     19.9 MB   86.95 KB  16.84 KB
0x964  0n2404  ffffc983f08c7080 svchost.exe                                   2 TB      2 TB   1.81 MB        0      2.2 MB   86.97 KB 143.48 KB
0x970  0n2416  ffffc983f0cb5080 svchost.exe                                   2 TB      2 TB   1.78 MB        0     2.83 MB  105.55 KB  10.27 KB
0x998  0n2456  ffffc983f0cbc080 svchost.exe                                   2 TB      2 TB  10.82 MB        0      1.3 MB   65.43 KB   8.09 KB
0x9a0  0n2464  ffffc983f0cbf080 svchost.exe                                2.01 TB   2.01 TB   1.82 MB        0    81.98 MB  111.03 KB  15.93 KB
0x9b0  0n2480  ffffc983f0cca080 svchost.exe                                   2 TB      2 TB   1.78 MB        0     2.65 MB    89.5 KB  15.79 KB
0xa24  0n2596  ffffc983f0b40080 svchost.exe                                   2 TB      2 TB   1.82 MB        0     1.77 MB   80.23 KB  10.23 KB
0xa34  0n2612  ffffc983f0b4c080 MemCompression                              186 MB    186 MB                  0      440 KB    4.12 KB          
0xa94  0n2708  ffffc983f0e3c080 svchost.exe                                   2 TB      2 TB   1.82 MB        0     1.59 MB   82.95 KB   10.2 KB
0xaa4  0n2724  ffffc983f0e40080 svchost.exe                                   2 TB      2 TB   2.21 MB        0     1.87 MB   138.8 KB  11.52 KB
0xab0  0n2736  ffffc983f0e4e080 svchost.exe                                   2 TB      2 TB   1.78 MB        0     1.82 MB   83.01 KB  11.23 KB
0xb08  0n2824  ffffc983f0bed080 svchost.exe                                   2 TB      2 TB   1.78 MB        0     2.26 MB   80.72 KB  11.17 KB
0xb78  0n2936  ffffc983ebd2e080 svchost.exe                                   2 TB      2 TB   1.82 MB        0     3.37 MB   101.5 KB  12.12 KB
0xb80  0n2944  ffffc983ebd2a080 svchost.exe                                   2 TB      2 TB   1.79 MB        0     2.97 MB  118.36 KB  13.93 KB
0xbb0  0n2992  ffffc983ebcb3080 svchost.exe                                   2 TB      2 TB   1.78 MB        0     1.75 MB   75.81 KB   9.73 KB
0xbd4  0n3028  ffffc983ebdf0080 svchost.exe                                   2 TB      2 TB   1.82 MB        0     1.44 MB   64.22 KB   9.15 KB
0xbf0  0n3056  ffffc983ebdc7080 svchost.exe                                   2 TB      2 TB   1.81 MB        0     2.05 MB   86.11 KB  14.62 KB
0xc3c  0n3132  ffffc983f20760c0 svchost.exe                                   2 TB      2 TB   1.78 MB        0     1.38 MB   70.52 KB   8.46 KB
0xc44  0n3140  ffffc983f0fd90c0 svchost.exe                                   2 TB      2 TB   1.79 MB        0     2.18 MB   96.52 KB  12.37 KB
0xc90  0n3216  ffffc983f060c080 svchost.exe                                   2 TB      2 TB   3.12 MB        0     35.1 MB  360.57 KB  86.37 KB
0xc98  0n3224  ffffc983f060a080 svchost.exe                                   2 TB      2 TB   1.78 MB        0    12.94 MB  120.77 KB  33.95 KB
0xca0  0n3232  ffffc983f2006080 svchost.exe                                   2 TB      2 TB   1.79 MB        0     5.95 MB   141.4 KB  21.34 KB
0xd18  0n3352  ffffc983f20f7080 svchost.exe                                   2 TB      2 TB   1.82 MB        0    10.04 MB   91.45 KB  10.35 KB
0xd88  0n3464  ffffc983f21020c0 svchost.exe                                   2 TB      2 TB   1.82 MB        0     2.62 MB  104.08 KB  18.16 KB
0xdfc  0n3580  ffffc983f21bc0c0 spoolsv.exe                                   2 TB      2 TB   1.82 MB        0     6.92 MB  177.12 KB  23.67 KB
0xed8  0n3800  ffffc983f22240c0 svchost.exe                                   2 TB      2 TB   1.82 MB        0     2.67 MB  105.03 KB  15.74 KB
0xeec  0n3820  ffffc983f2411080 svchost.exe                                   2 TB      2 TB   2.11 MB        0     17.1 MB  248.38 KB   32.4 KB
0xef4  0n3828  ffffc983f2455080 svchost.exe                                   2 TB      2 TB   1.91 MB        0     4.29 MB  132.34 KB  28.26 KB
0xefc  0n3836  ffffc983f2410080 AnyDesk.exe*32                           116.03 MB 137.57 MB   2.03 MB        0     28.7 MB  195.25 KB  26.32 KB
0xf04  0n3844  ffffc983f208e080 svchost.exe                                   2 TB      2 TB   1.79 MB        0     9.85 MB  132.02 KB  20.21 KB
0xf10  0n3856  ffffc983f2454080 IpOverUsbSvc.exe*32                      122.96 MB 130.36 MB   1.88 MB        0     8.36 MB  181.45 KB  18.28 KB
0xf1c  0n3868  ffffc983f21e9080 sqlwriter.exe                              4.08 GB   4.09 GB   1.82 MB        0     1.77 MB   84.73 KB  10.02 KB
0xf44  0n3908  ffffc983f2222080 RustDesk.exe                               4.12 GB   4.12 GB   1.83 MB        0      2.5 MB  179.59 KB  11.69 KB
0xfac  0n4012  ffffc983f25ab0c0 MsMpEng.exe                                   2 TB   2.01 TB    2.2 MB        0   277.18 MB  601.12 KB 111.91 KB
0xfb8  0n4024  ffffc983f2344080 svchost.exe                                   2 TB      2 TB   1.82 MB        0      6.7 MB   108.8 KB  16.02 KB
0xfc0  0n4032  ffffc983f2355080 wlms.exe                                      2 TB      2 TB    240 KB        0      744 KB   29.79 KB   5.43 KB
0xfd0  0n4048  ffffc983f2364080 svchost.exe                                   2 TB      2 TB   1.81 MB        0     1.22 MB   60.23 KB    8.5 KB
0xfe8  0n4072  ffffc983f2376080 svchost.exe                                   2 TB      2 TB   1.82 MB        0     2.26 MB   89.44 KB  12.48 KB
0xff0  0n4080  ffffc983f2377080 svchost.exe                                   2 TB      2 TB    2.1 MB        0     4.09 MB  152.71 KB  19.23 KB
0x101c 0n4124  ffffc983f2555080 svchost.exe                                   2 TB      2 TB   1.82 MB        0     1.38 MB   65.39 KB   8.76 KB
0x1170 0n4464  ffffc983f276b080 sppsvc.exe                                    2 TB      2 TB   1.76 MB        0    11.91 MB  116.95 KB  13.94 KB
0x1264 0n4708  ffffc983f28020c0 svchost.exe                                   2 TB      2 TB   1.81 MB        0     1.32 MB   68.79 KB   8.94 KB
0x132c 0n4908  ffffc983f2820080 unsecapp.exe                                  2 TB      2 TB   1.82 MB        0     1.29 MB   66.19 KB   8.09 KB
0x14ac 0n5292  ffffc983f2a30080 SppExtComObj.Exe                              2 TB      2 TB   1.78 MB        0     1.62 MB    97.1 KB    9.6 KB
0x14e8 0n5352  ffffc983f2a64080 svchost.exe                                   2 TB      2 TB   2.06 MB        0     2.14 MB   75.12 KB  11.01 KB
0x15ac 0n5548  ffffc983f2a6a080 AggregatorHost.exe                            2 TB      2 TB    252 KB        0     1.45 MB   60.46 KB   8.09 KB
0x15ec 0n5612  ffffc983f2aea0c0 svchost.exe                                   2 TB      2 TB   1.78 MB        0     4.67 MB  104.45 KB  13.47 KB
0x16fc 0n5884  ffffc983f2c320c0 svchost.exe                                   2 TB      2 TB   2.09 MB        0      3.9 MB  144.18 KB  17.62 KB
0x17f0 0n6128  ffffc983f2d79080 sihost.exe                                    2 TB      2 TB   2.54 MB        0     5.43 MB   272.7 KB  17.85 KB
0x2c4  0n708   ffffc983f2d78080 svchost.exe                                   2 TB      2 TB   2.55 MB        0     4.72 MB  184.49 KB  15.16 KB
0x704  0n1796  ffffc983f2d770c0 svchost.exe                                   2 TB      2 TB   2.54 MB        0     1.84 MB  112.58 KB   9.02 KB
0xf30  0n3888  ffffc983f2d64080 svchost.exe                                   2 TB      2 TB   2.55 MB        0     5.68 MB  241.62 KB  16.05 KB
0xcb8  0n3256  ffffc983f2d61080 svchost.exe                                   2 TB      2 TB    2.1 MB        0     3.05 MB  162.15 KB  13.27 KB
0x12f8 0n4856  ffffc983f2d5e080 taskhostw.exe                                 2 TB      2 TB   3.86 MB        0     6.24 MB  192.57 KB  31.96 KB
0x18f4 0n6388  ffffc983f2d59080 svchost.exe                                   2 TB      2 TB   2.07 MB        0     3.75 MB  132.01 KB  19.45 KB
0x196c 0n6508  ffffc983f2f18080 explorer.exe                               2.01 TB   2.01 TB 129.45 MB        0      288 MB     2.9 MB 167.61 KB
0x1b50 0n6992  ffffc983f2d5b080 svchost.exe                                   2 TB      2 TB   2.55 MB        0     5.79 MB  324.07 KB  17.72 KB
0x1bf4 0n7156  ffffc983f2d60080 svchost.exe                                   2 TB      2 TB    2.1 MB        0     3.17 MB   98.62 KB  12.34 KB
0x18e8 0n6376  ffffc983f2a69080 svchost.exe                                   2 TB      2 TB    2.1 MB        0     2.35 MB   84.41 KB  12.16 KB
0x19e0 0n6624  ffffc983f2f15080 SearchHost.exe                             2.04 TB   2.04 TB  14.59 MB        0   154.52 MB    1.71 MB 147.24 KB
0x19d8 0n6616  ffffc983f2f14080 StartMenuExperienceHost.exe                2.01 TB   2.01 TB  16.04 MB        0    39.91 MB  754.06 KB  40.71 KB
0x155c 0n5468  ffffc983f2f09080 Widgets.exe                                   2 TB      2 TB   2.64 MB        0     8.12 MB  361.18 KB  26.97 KB
0x1c3c 0n7228  ffffc983f3af5080 RuntimeBroker.exe                             2 TB      2 TB   2.56 MB        0     3.51 MB  227.46 KB   15.8 KB
0x1cac 0n7340  ffffc983f351a080 RuntimeBroker.exe                             2 TB      2 TB   3.93 MB        0     15.1 MB  421.48 KB  30.64 KB
0x1d34 0n7476  ffffc983f36c0080 svchost.exe                                   2 TB      2 TB    2.1 MB        0     2.25 MB  108.35 KB  10.22 KB
0x1d70 0n7536  ffffc983f39f1080 svchost.exe                                   2 TB      2 TB   2.55 MB        0     2.99 MB  194.97 KB  14.34 KB
0x1ef0 0n7920  ffffc983f385e080 dllhost.exe                                   2 TB      2 TB   2.39 MB        0     7.51 MB  170.88 KB  30.41 KB
0x1e70 0n7792  ffffc983f36680c0 ctfmon.exe                                    2 TB      2 TB   3.74 MB        0     6.98 MB  266.64 KB  20.09 KB
0x2020 0n8224  ffffc983f3e450c0 SecurityHealthSystray.exe                     2 TB      2 TB   2.27 MB        0     1.71 MB  158.59 KB  10.22 KB
0x2034 0n8244  ffffc983f3e560c0 SecurityHealthService.exe                     2 TB      2 TB   1.83 MB        0     7.12 MB  200.47 KB  20.57 KB
0x2128 0n8488  ffffc983f34ce080 NisSrv.exe                                    2 TB      2 TB   1.78 MB        0     3.72 MB  127.02 KB  39.29 KB
0x2280 0n8832  ffffc983f3a84080 TextInputHost.exe                          2.04 TB   2.04 TB   5.15 MB        0    71.79 MB    1.06 MB  79.53 KB
0xa00  0n2560  ffffc983f47220c0 svchost.exe                                   2 TB      2 TB   1.78 MB        0     1.84 MB   78.86 KB  15.46 KB
0x1eb4 0n7860  ffffc983f47270c0 SearchIndexer.exe                             2 TB      2 TB   1.97 MB        0    26.25 MB  165.16 KB  19.09 KB
0x1d64 0n7524  ffffc983f38e40c0 DbgSvc.exe                                 4.12 GB   4.13 GB   1.82 MB        0     5.74 MB   136.8 KB  16.51 KB
0x1d40 0n7488  ffffc983ecd3f0c0 dllhost.exe                                   2 TB      2 TB   1.81 MB        0     4.25 MB  108.12 KB   14.7 KB
0x1e5c 0n7772  ffffc983ef7300c0 msdtc.exe                                     2 TB      2 TB   1.79 MB        0     3.11 MB  104.05 KB  13.58 KB
0x258  0n600   ffffc983f3a9c0c0 svchost.exe                                   2 TB      2 TB   2.54 MB        0      2.8 MB  157.64 KB  14.59 KB
0x358  0n856   ffffc983f3aa10c0 svchost.exe                                   2 TB      2 TB   1.82 MB        0     3.88 MB  116.28 KB  15.36 KB
0x21b4 0n8628  ffffc983efd4d0c0 uhssvc.exe                                    2 TB      2 TB    236 KB        0     1.28 MB      62 KB   7.23 KB
0x2288 0n8840  ffffc983f3a690c0 svchost.exe                                   2 TB      2 TB   1.82 MB        0     2.57 MB   86.38 KB  14.05 KB
0x1e40 0n7744  ffffc983efc660c0 MoUsoCoreWorker.exe                           2 TB      2 TB   2.12 MB        0    11.25 MB  195.83 KB  23.21 KB
0x1274 0n4724  ffffc983f3518080 svchost.exe                                   2 TB      2 TB   1.78 MB        0     2.88 MB   83.34 KB  13.18 KB
0x21e8 0n8680  ffffc983efcd7080 csrss.exe                                     2 TB      2 TB   2.13 MB        0     2.01 MB  115.29 KB  12.27 KB
0xbac  0n2988  ffffc983efcd6080 winlogon.exe                                  2 TB      2 TB   3.02 MB        0     2.31 MB  131.56 KB  11.62 KB
0xf7c  0n3964  ffffc983ef86c080 WmiPrvSE.exe                                  2 TB      2 TB   1.82 MB        0     2.68 MB   80.16 KB  11.02 KB
0x1af4 0n6900  ffffc983f22020c0 LogonUI.exe                                   2 TB      2 TB  15.12 MB        0    14.23 MB  569.73 KB  38.12 KB
0x1e24 0n7716  ffffc983ef86b080 dwm.exe                                       2 TB      2 TB  11.09 MB        0    25.77 MB  380.57 KB  25.61 KB
0x1cb4 0n7348  ffffc983f3aa0080 fontdrvhost.exe                               2 TB      2 TB    228 KB        0     1.31 MB   37.88 KB   6.23 KB
0x1b40 0n6976  ffffc983f26e6080 RustDesk.exe                               4.16 GB   4.17 GB   1.79 MB        0      3.2 MB  217.33 KB  13.13 KB
0xb10  0n2832  ffffc983f3a9f080 rdpclip.exe                                   2 TB      2 TB   3.46 MB        0     3.65 MB  230.07 KB  17.69 KB
0xe28  0n3624  ffffc983ecd4a080 WUDFHost.exe                               2.01 TB   2.01 TB    1.3 GB        0    11.16 MB    2.79 MB  26.14 KB
0xecc  0n3788  ffffc983ecd49080 svchost.exe                                   2 TB      2 TB   1.81 MB        0     1.41 MB   72.16 KB  10.75 KB
0x15e4 0n5604  ffffc983f36d30c0 svchost.exe                                   2 TB      2 TB    2.1 MB        0     3.71 MB  100.46 KB  16.48 KB
0x350  0n848   ffffc983ef9a80c0 rdpinput.exe                                  2 TB      2 TB   3.44 MB        0     1.38 MB  105.61 KB   8.09 KB
0xeb4  0n3764  ffffc983f3844080 TabTip.exe                                    2 TB      2 TB    2.3 MB        0     3.57 MB  249.59 KB  16.19 KB
0x954  0n2388  ffffc983f365c0c0 svchost.exe                                   2 TB      2 TB   2.12 MB        0     6.41 MB  205.37 KB  20.14 KB
0x1cf8 0n7416  ffffc983efcd50c0 ShellExperienceHost.exe                       2 TB      2 TB   3.62 MB        0    16.77 MB    1.18 MB  31.45 KB
0x11b4 0n4532  ffffc983f0b740c0 svchost.exe                                   2 TB      2 TB    2.1 MB        0    10.27 MB  149.62 KB  18.29 KB
0x2138 0n8504  ffffc983ecdd90c0 svchost.exe                                   2 TB      2 TB   1.82 MB        0     2.74 MB   136.4 KB   13.2 KB
0x858  0n2136  ffffc983ece4e0c0 WidgetService.exe                             2 TB      2 TB   2.55 MB        0     4.44 MB  229.92 KB  17.55 KB
0x1394 0n5012  ffffc983f052a0c0 svchost.exe                                   2 TB      2 TB   2.26 MB        0     1.74 MB  132.89 KB   9.42 KB
0x23f8 0n9208  ffffc983f3fe70c0 RuntimeBroker.exe                             2 TB      2 TB   3.73 MB        0     3.78 MB  199.83 KB   15.5 KB
0x2268 0n8808  ffffc983f3fec0c0 SystemSettingsBroker.exe                      2 TB      2 TB   2.55 MB        0     4.98 MB   275.7 KB  19.96 KB
0x2084 0n8324  ffffc983f3441180 svchost.exe                                   2 TB      2 TB   1.78 MB        0     1.61 MB   72.31 KB  42.16 KB
0x23d8 0n9176  ffffc983ecf540c0 svchost.exe                                   2 TB      2 TB   1.82 MB        0     3.44 MB  124.77 KB  23.59 KB
0x2518 0n9496  ffffc983f2074140 smartscreen.exe                               2 TB      2 TB   2.26 MB        0     2.57 MB  129.07 KB  10.93 KB
0x24a8 0n9384  ffffc983ef3a50c0 svchost.exe                                   2 TB      2 TB   1.81 MB        0     1.57 MB   71.38 KB   9.09 KB
0x1a0c 0n6668  ffffc983ebee30c0 devenv.exe                                 2.01 TB   2.01 TB   5.73 MB        0   748.84 MB    3.14 MB 200.77 KB
0x122c 0n4652  ffffc983ed2180c0 PerfWatson2.exe                            4.78 GB   4.82 GB   2.88 MB        0    48.99 MB  456.25 KB  49.48 KB
0x1be4 0n7140  ffffc983edfab0c0 Microsoft.ServiceHub.Controller.exe        4.64 GB   4.65 GB   2.75 MB        0    42.98 MB  460.55 KB  33.06 KB
0x19d4 0n6612  ffffc983efadd0c0 ServiceHub.VSDetouredHost.exe               5.1 GB   5.12 GB   2.76 MB        0      169 MB    1.35 MB  70.66 KB
0x1420 0n5152  ffffc983ef4c4080 ServiceHub.Host.dotnet.x64.exe                2 TB      2 TB    2.4 MB        0    48.61 MB   412.8 KB  65.13 KB
0x19bc 0n6588  ffffc983efc9c080 ServiceHub.SettingsHost.exe                 5.1 GB   5.11 GB   2.76 MB        0   160.18 MB    1.32 MB  66.66 KB
0x1b88 0n7048  ffffc983eda770c0 ServiceHub.Host.netfx.x86.exe*32         423.58 MB 440.28 MB   2.75 MB        0    58.72 MB  733.23 KB  63.37 KB
0x2614 0n9748  ffffc983f42610c0 ServiceHub.ThreadedWaitDialog.exe          4.91 GB   4.93 GB   4.44 MB        0   100.65 MB  945.56 KB  57.32 KB
0x1660 0n5728  ffffc983efc6d140 ServiceHub.IndexingService.exe                2 TB      2 TB    2.4 MB        0    36.15 MB  378.89 KB  56.71 KB
0xda8  0n3496  ffffc983eddc70c0 ServiceHub.IntellicodeModelService.exe     4.83 GB   4.89 GB   2.68 MB        0   121.52 MB  806.38 KB  43.59 KB
0x1428 0n5160  ffffc983f41720c0 vshost.exe                                  5.4 MB    5.4 MB    144 KB        0      372 KB    8.65 KB   1.59 KB
0x1ce8 0n7400  ffffc983f40ae0c0 vcpkgsrv.exe*32                            1.07 GB   1.07 GB   2.25 MB        0    45.12 MB  187.62 KB  27.46 KB
0x8e0  0n2272  ffffc983f42350c0 ServiceHub.RoslynCodeAnalysisService.exe   5.26 GB   5.28 GB   2.75 MB        0    222.4 MB    1.65 MB  59.45 KB
0x2658 0n9816  ffffc983edd020c0 ServiceHub.Host.AnyCPU.exe                 5.45 GB   5.46 GB   2.76 MB        0   266.84 MB    2.04 MB  73.44 KB
0x1210 0n4624  ffffc983ed5e60c0 ServiceHub.TestWindowStoreHost.exe         4.75 GB   4.76 GB   2.75 MB        0    63.78 MB  660.84 KB  49.38 KB
0xcb0  0n3248  ffffc983edfd90c0 vcpkgsrv.exe*32                           91.41 MB  95.16 MB   2.26 MB        0    11.66 MB  160.65 KB  15.51 KB
0x24c0 0n9408  ffffc983ec692080 MSBuild.exe                                4.65 GB   4.73 GB   2.77 MB        0    51.46 MB   419.7 KB   29.3 KB
0x24c4 0n9412  ffffc983ecff0080 conhost.exe                                   2 TB      2 TB   2.24 MB        0     5.22 MB  122.82 KB   7.82 KB
0x1bc4 0n7108  ffffc983ece85080 Notepad.exe                                4.39 GB   4.42 GB  24.43 MB        0    23.23 MB   867.3 KB  34.62 KB
0x2984 0n10628 ffffc983eda99080 audiodg.exe                                   2 TB      2 TB   1.78 MB        0     6.88 MB  113.77 KB  14.06 KB
0x2a1c 0n10780 ffffc983ede2a080 cmd.exe                                       2 TB      2 TB    240 KB        0     2.09 MB   48.89 KB   6.14 KB
0x2a24 0n10788 ffffc983ed19e080 conhost.exe                                   2 TB      2 TB  20.25 MB        0     6.69 MB  268.27 KB  14.87 KB
0x1050 0n4176  ffffc983edbe90c0 svchost.exe                                   2 TB      2 TB   1.82 MB        0     3.94 MB  118.95 KB  17.45 KB
0x1544 0n5444  ffffc983f36c5080 svchost.exe                                   2 TB      2 TB   1.81 MB        0     1.37 MB   60.01 KB   8.23 KB
0x10a4 0n4260  ffffc983edfae080 notmyfault64.exe                            4.1 GB    4.1 GB   2.27 MB        0     1.25 MB   123.2 KB   7.98 KB
============== ================ ======================================== ========= ========= ========= ======== =========== ========== =========
PID            Address          Name                                            VM      Peak    Shared Awe Size Commit Size   PP Quota NPP Quota

Warning! Zombie process(es) detected (not displayed). Count: 5 [zombie report]
```

The **`!vm 1`** command is used for assessing the state of the system's virtual memory, identifying potential memory issues, and understanding how memory resources are being allocated and used. Can we identify based on this output if the machine has been running out of memory for example?

```
0: kd> !vm 1
Page File: \??\C:\pagefile.sys
  Current:  15728640 Kb  Free Space:  15632316 Kb
  Minimum:  15728640 Kb  Maximum:     20971520 Kb
Page File: \??\C:\swapfile.sys
  Current:    262144 Kb  Free Space:    258904 Kb
  Minimum:    262144 Kb  Maximum:     10129836 Kb
No Name for Paging File
  Current:  27724744 Kb  Free Space:  27170296 Kb
  Minimum:  27724744 Kb  Maximum:     27724744 Kb

Physical Memory:          2037490 (    8149960 Kb)
Available Pages:           693849 (    2775396 Kb)
ResAvail Pages:           1914220 (    7656880 Kb)
Locked IO Pages:                0 (          0 Kb)
Free System PTEs:      4295156169 (17180624676 Kb)

******* 351175 kernel stack PTE allocations have failed ******

Modified Pages:              9900 (      39600 Kb)
Modified PF Pages:           9751 (      39004 Kb)
Modified No Write Pages:       28 (        112 Kb)
NonPagedPool Usage:           268 (       1072 Kb)
NonPagedPoolNx Usage:       36623 (     146492 Kb)
NonPagedPool Max:      4294967296 (17179869184 Kb)
PagedPool Usage:            68600 (     274400 Kb)
PagedPool Maximum:     4294967296 (17179869184 Kb)
Processor Commit:            1052 (       4208 Kb)
Session Commit:               211 (        844 Kb)
Shared Commit:             435025 (    1740100 Kb)
Special Pool:                   0 (          0 Kb)
Kernel Stacks:              14686 (      58744 Kb)
Pages For MDLs:              3166 (      12664 Kb)
ContigMem Pages:                6 (         24 Kb)
Partition Pages:                0 (          0 Kb)
Pages For AWE:                  0 (          0 Kb)
NonPagedPool Commit:        38364 (     153456 Kb)
PagedPool Commit:  18446735300397918280 (18446708980463018272 Kb)
Driver Commit:              13482 (      53928 Kb)
Boot Commit:                 4890 (      19560 Kb)
PFN Array Commit:           24918 (      99672 Kb)
SmallNonPagedPtesCommit:      766 (       3064 Kb)
SlabAllocatorPages:          4608 (      18432 Kb)
SkPagesInUnchargedSlabs:     5641 (      22564 Kb)
System PageTables:           1389 (       5556 Kb)
ProcessLockedFilePages:        14 (         56 Kb)
Pagefile Hash Pages:           91 (        364 Kb)
Sum System Commit: 18446735300398461981 (18446708980465193076 Kb)
Total Private:            1073332 (    4293328 Kb)

********** Sum of individual system commit + Process commit exceeds overall commit by 18446708980462741736 Kb ? ********
Committed pages:          1686167 (    6744668 Kb)
Commit limit:             5969650 (   23878600 Kb)
```

# Reference

- Windows Internals 7th edition (Part 1): https://www.microsoftpressstore.com/store/windows-internals-part-1-system-architecture-processes-9780735684188
