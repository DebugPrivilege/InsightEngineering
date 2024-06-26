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

From the output, we can see that there are a couple of **running** threads. Let's examine the **running** threads on the system. The output of the **`!mex.running`** command in provides a snapshot of the currently running threads on the system. 

Examining threads in a **running** state is the most important thing to do first, because they (may) contain the immediate context and operations that led to the crash, allowing for a direct insight into the cause of the issue.

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

When a thread is **stalled for more than 30 seconds**, it means that the thread has been in the ready state (or a similar state like standby) but has not been scheduled to run for over 30 seconds. **It's worth to investigate these threads first and prioritize them over others**, because if a thread is ready to run but is not getting CPU time, the tasks it is supposed to perform are delayed. This can lead to overall system sluggishness.

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

Another aspect worth exploring is the number of threads within each process. For instance, **are there processes containing over or close to 500 threads?** While it is not by definition a problem, such a high thread count is worth further investigation.

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

High CPU time means a process is using a lot of CPU resources, which can affect system performance. If a process has more CPU time than others, it usually means it has been using the CPU heavily for a long period. High CPU usage can sometimes be caused by resource leaks such as memory leaks or handle leaks. We can use the  we can use the **`!mex.tl -cpu`** command to get the CPU time of every process.

```
0: kd> !mex.tl -cpu
PID            Address          Name                                          User        Kernel         Total
============== ================ ===================================== ============ ============= =============
0x0    0n0     fffff80122d24a00 Idle                                             0 20h:32:55.657 20h:32:55.657
0x4    0n4     ffff988d77adb080 System                                           0    13m:57.177    13m:57.177
0x8c   0n140   ffff988d77bf4040 Registry                                         0         750ms         750ms
0x214  0n532   ffff988d80ca20c0 smss.exe                                         0         172ms         172ms
0x3a0  0n928   ffff988d8b937140 csrss.exe                                    172ms        1s.860        2s.032
0x3f4  0n1012  ffff988d85d020c0 wininit.exe                                      0          16ms          16ms
0x3fc  0n1020  ffff988d85dbd0c0 csrss.exe                                    594ms       47s.955       48s.549
0x310  0n784   ffff988d8eca8280 services.exe                                7s.453        6s.767       14s.220
0x30c  0n780   ffff988d8609b0c0 lsass.exe                                   3s.171        2s.593        5s.764
0x364  0n868   ffff988d863ce0c0 svchost.exe                                46s.218       14s.314     1m:00.532
0x41c  0n1052  ffff988d864570c0 fontdrvhost.exe                                  0             0             0
0x43c  0n1084  ffff988d864ce0c0 WUDFHost.exe                                  62ms          16ms          78ms
0x480  0n1152  ffff988d866240c0 svchost.exe                                 6s.907        4s.141       11s.048
0x4b0  0n1200  ffff988d868bd0c0 svchost.exe                                  438ms         500ms         938ms
0x550  0n1360  ffff988d86c350c0 winlogon.exe                                     0          78ms          78ms
0x598  0n1432  ffff988d77ab6080 fontdrvhost.exe                              500ms         703ms        1s.203
0x5e4  0n1508  ffff988d8728a0c0 dwm.exe                                 11m:11.767     6m:32.657    17m:44.424
0x61c  0n1564  ffff988d873ac0c0 svchost.exe                                 1s.844        4s.377        6s.221
0x630  0n1584  ffff988d874130c0 svchost.exe                                      0          16ms          16ms
0x684  0n1668  ffff988d876020c0 svchost.exe                                  390ms         891ms        1s.281
0x6b0  0n1712  ffff988d876ce0c0 svchost.exe                                   15ms         219ms         234ms
0x6b8  0n1720  ffff988d876df0c0 svchost.exe                                   15ms          16ms          31ms
0x6c4  0n1732  ffff988d877240c0 svchost.exe                                   15ms          31ms          46ms
0x764  0n1892  ffff988d87aac0c0 svchost.exe                                      0          16ms          16ms
0x774  0n1908  ffff988d87c9b0c0 svchost.exe                                  265ms         266ms         531ms
0x77c  0n1916  ffff988d87cbd0c0 svchost.exe                                   15ms          16ms          31ms
0x784  0n1924  ffff988d87d020c0 svchost.exe                                   16ms             0          16ms
0x7e4  0n2020  ffff988d87edf0c0 svchost.exe                                 4s.516         531ms        5s.047
0x814  0n2068  ffff988d882240c0 svchost.exe                                      0          16ms          16ms
0x874  0n2164  ffff988d884ce0c0 svchost.exe                                   31ms          63ms          94ms
0x898  0n2200  ffff988d8ba410c0 svchost.exe                                 1s.469        1s.313        2s.782
0x940  0n2368  ffff988d7e2740c0 svchost.exe                                  265ms         281ms         546ms
0x968  0n2408  ffff988d88d430c0 NVDisplay.Container.exe                      313ms         235ms         548ms
0x9b8  0n2488  ffff988d837680c0 svchost.exe                                  141ms         610ms         751ms
0x9dc  0n2524  ffff988d837130c0 svchost.exe                                      0             0             0
0xa48  0n2632  ffff988d8cc340c0 svchost.exe                                      0          16ms          16ms
0xa80  0n2688  ffff988d8ea250c0 PresentationFontCache.exe                        0          63ms          63ms
0xaa8  0n2728  ffff988d86d680c0 svchost.exe                                  688ms         672ms        1s.360
0xab0  0n2736  ffff988d86d350c0 AsusCertService.exe*32                      1s.203       29s.797       31s.000
0xb20  0n2848  ffff988d887240c0 svchost.exe                                 5s.375        1s.750        7s.125
0xb5c  0n2908  ffff988d889790c0 svchost.exe                                      0          16ms          16ms
0xbbc  0n3004  ffff988d83aac0c0 svchost.exe                                      0          47ms          47ms
0xbd8  0n3032  ffff988d86dac0c0 svchost.exe                                  281ms         625ms         906ms
0x9ec  0n2540  ffff988d8be8f140 svchost.exe                                      0             0             0
0xc10  0n3088  ffff988d85744080 svchost.exe                                   47ms          47ms          94ms
0xc18  0n3096  ffff988d85755080 svchost.exe                                  594ms        1s.406        2s.000
0xc40  0n3136  ffff988d85833080 svchost.exe                                 1s.922        6s.204        8s.126
0xc74  0n3188  ffff988d859aa080 svchost.exe                                  156ms         844ms        1s.000
0xc80  0n3200  ffff988d85b22080 svchost.exe                                  234ms          48ms         282ms
0xcb4  0n3252  ffff988d85d11040 MemCompression                                   0         126ms         126ms
0xd54  0n3412  ffff988d86922080 svchost.exe                                      0          16ms          16ms
0xda0  0n3488  ffff988d86955080 svchost.exe                                      0          16ms          16ms
0xde0  0n3552  ffff988d86e11080 svchost.exe                                   31ms         235ms         266ms
0xdf4  0n3572  ffff988d86e33080 svchost.exe                                 8s.219        2s.016       10s.235
0xe54  0n3668  ffff988d87133080 NVDisplay.Container.exe                     2s.611        1s.080        3s.691
0xf8c  0n3980  ffff988d83511080 svchost.exe                                   15ms         203ms         218ms
0xfa8  0n4008  ffff988d85b11300 svchost.exe                                      0          48ms          48ms
0xfe4  0n4068  ffff988d83666080 dasHost.exe                                  359ms         703ms        1s.062
0x1010 0n4112  ffff988d83555300 svchost.exe                                      0          16ms          16ms
0x104c 0n4172  ffff988d8e5bf080 svchost.exe                                   78ms          16ms          94ms
0x112c 0n4396  ffff988d8e625080 WmiPrvSE.exe                               17s.126       57s.750     1m:14.876
0x11e4 0n4580  ffff988d8e6d4080 svchost.exe                                   63ms          94ms         157ms
0x1250 0n4688  ffff988d8e6c5080 svchost.exe                                   78ms          32ms         110ms
0x12ac 0n4780  ffff988d8e7970c0 svchost.exe                                  719ms        4s.313        5s.032
0x12b4 0n4788  ffff988d8e79a0c0 svchost.exe                                16s.640       30s.766       47s.406
0x1360 0n4960  ffff988d8e82b080 svchost.exe                                17s.859        8s.673       26s.532
0x13fc 0n5116  ffff988d8e86d080 svchost.exe                                  187ms         219ms         406ms
0xfd0  0n4048  ffff988d8e896080 svchost.exe                                 1s.031         454ms        1s.485
0x12dc 0n4828  ffff988d8e8e9080 svchost.exe                                  125ms         454ms         579ms
0x1420 0n5152  ffff988d8e9db080 atkexComSvc.exe*32                           828ms         125ms         953ms
0x14a4 0n5284  ffff988d8fc3c080 WmiPrvSE.exe                                  63ms         345ms         408ms
0x1574 0n5492  ffff988d8feda240 dasHost.exe                                      0             0             0
0x16b0 0n5808  ffff988d912de080 svchost.exe                                      0          31ms          31ms
0x16b8 0n5816  ffff988d912c7080 svchost.exe                                  249ms         328ms         577ms
0x174c 0n5964  ffff988d914ab0c0 svchost.exe                                  657ms         689ms        1s.346
0x17a0 0n6048  ffff988d914ae080 svchost.exe                                   16ms          32ms          48ms
0x11b0 0n4528  ffff988d915a60c0 spoolsv.exe                                  736ms        1s.111        1s.847
0x15a4 0n5540  ffff988d915bd080 AsusUpdateCheck.exe                              0             0             0
0x15f0 0n5616  ffff988d915c1080 AppleMobileDeviceService.exe                 359ms         454ms         813ms
0x15c8 0n5576  ffff988d915ea0c0 mDNSResponder.exe                            156ms         297ms         453ms
0x15ac 0n5548  ffff988d915eb080 CorsairGamingAudioCfgService64.exe            16ms          16ms          32ms
0x15dc 0n5596  ffff988d915c5080 ArmouryCrate.Service.exe                     312ms         532ms         844ms
0x15e8 0n5608  ffff988d915ee080 AsusFanControlService.exe*32                 813ms          31ms         844ms
0x15ec 0n5612  ffff988d915f4080 CueLLAccessService.exe                        16ms         109ms         125ms
0x15bc 0n5564  ffff988d915f1080 armsvc.exe*32                                    0             0             0
0x15cc 0n5580  ffff988d915ec080 service.exe                                   31ms          16ms          47ms
0x970  0n2416  ffff988d915ef080 AdobeUpdateService.exe*32                        0             0             0
0x1648 0n5704  ffff988d915c42c0 Corsair.Service.exe*32                   3m:45.468     2m:16.314     6m:01.782
0x1734 0n5940  ffff988d9169e080 svchost.exe                                   93ms         359ms         452ms
0x1738 0n5944  ffff988d9169b080 OfficeClickToRun.exe                         720ms         406ms        1s.126
0x1824 0n6180  ffff988d916bf080 svchost.exe                                 1s.860        1s.016        2s.876
0x1840 0n6208  ffff988d916cd080 svchost.exe                                 4s.235        1s.736        5s.971
0x18a8 0n6312  ffff988d91717080 HPPrintScanDoctorService.exe                     0          16ms          16ms
0x18b8 0n6328  ffff988d91719080 OneApp.IGCC.WinService.exe                   188ms         141ms         329ms
0x1908 0n6408  ffff988d9177d080 jhi_service.exe                                  0             0             0
0x1918 0n6424  ffff988d9177f0c0 lghub_updater.exe                            157ms         127ms         284ms
0x1930 0n6448  ffff988d91789080 svchost.exe                                      0          63ms          63ms
0x1938 0n6456  ffff988d9178c080 LightingService.exe*32                       843ms         391ms        1s.234
0x1960 0n6496  ffff988d917cd080 mfemms.exe                                 34s.922       19s.000       53s.922
0x19e4 0n6628  ffff988d917a7080 ModuleCoreService.exe                       1s.171         237ms        1s.408
0x19f4 0n6644  ffff988d917a8080 NahimicService.exe                          1s.016        2s.750        3s.766
0x1a10 0n6672  ffff988d917aa080 pservice.exe                                     0          32ms          32ms
0x1a20 0n6688  ffff988d9189b080 ppped.exe*32                               24s.406        7s.219       31s.625
0x1a38 0n6712  ffff988d918a2080 nvcontainer.exe                              609ms         627ms        1s.236
0x1a40 0n6720  ffff988d918a60c0 PEFService.exe                               156ms         251ms         407ms
0x1a60 0n6752  ffff988d918b3080 pppServiceMonitor.exe*32                      32ms             0          32ms
0x1a78 0n6776  ffff988d918c80c0 ROGLiveService.exe                          2s.407        4s.531        6s.938
0x1a84 0n6788  ffff988d918b4080 svchost.exe                                      0          16ms          16ms
0x1a98 0n6808  ffff988d918c92c0 RstMwService.exe                                 0             0             0
0x1abc 0n6844  ffff988d918cd080 OriginWebHelperService.exe*32                470ms         500ms         970ms
0x1ad0 0n6864  ffff988d918f0080 spdsvc.exe*32                                    0          16ms          16ms
0x1adc 0n6876  ffff988d918ee080 RtkAudUService64.exe                          31ms          78ms         109ms
0x1af0 0n6896  ffff988d919a50c0 sqlwriter.exe                                 16ms          16ms          32ms
0x1b00 0n6912  ffff988d918f9080 NetFaxServer64.exe                            16ms          16ms          32ms
0x1b1c 0n6940  ffff988d919d70c0 ss_conn_service2.exe*32                          0             0             0
0x1b24 0n6948  ffff988d919eb080 ss_conn_service.exe*32                           0             0             0
0x1b2c 0n6956  ffff988d919ec080 svchost.exe                                      0          16ms          16ms
0x1b5c 0n7004  ffff988d919ef240 vmware-authd.exe*32                              0          32ms          32ms
0x1b70 0n7024  ffff988d91a76080 tvnserver.exe                               1s.079        2s.330        3s.409
0x1b78 0n7032  ffff988d91a73080 TeamViewer_Service.exe                       282ms         203ms         485ms
0x1b90 0n7056  ffff988d91a5a240 vmnetdhcp.exe*32                                 0             0             0
0x1c20 0n7200  ffff988d91bb3240 vmware-usbarbitrator64.exe                    32ms          32ms          64ms
0x1cd4 0n7380  ffff988d91ca2080 svchost.exe                                   15ms             0          15ms
0x1cec 0n7404  ffff988d91be4080 sqlceip.exe                                  406ms         640ms        1s.046
0x1cf4 0n7412  ffff988d91cae080 parsecd.exe                                  813ms         519ms        1s.332
0x1e4c 0n7756  ffff988d920b90c0 svchost.exe                                      0             0             0
0x2368 0n9064  ffff988d92441080 svchost.exe                                   16ms          47ms          63ms
0x2478 0n9336  ffff988d9267e0c0 MMSSHOST.exe                               53s.503       35s.471     1m:28.974
0x250c 0n9484  ffff988d927a60c0 mfevtps.exe                                 3s.187        5s.735        8s.922
0x2518 0n9496  ffff988d927b4080 audiodg.exe                              3m:05.922       13s.734     3m:19.656
0x258c 0n9612  ffff988d92897080 ProtectedModuleHost.exe                       47ms          79ms         126ms
0x25b0 0n9648  ffff988d9289a0c0 rundll32.exe                                     0          16ms          16ms
0x28c0 0n10432 ffff988d92558300 Corsair.Service.CpuIdRemote64.exe          47s.204     2m:15.313     3m:02.517
0x28c8 0n10440 ffff988d925bc2c0 conhost.exe                                   16ms             0          16ms
0x2908 0n10504 ffff988d930ac080 svchost.exe                                  531ms        1s.094        1s.625
0x2af8 0n11000 ffff988d93310080 Corsair.Service.DisplayAdapter.exe*32       1s.063         469ms        1s.532
0x2b14 0n11028 ffff988d91c93080 conhost.exe                                  141ms         125ms         266ms
0x2aa0 0n10912 ffff988d933eb080 unsecapp.exe                                     0          16ms          16ms
0x2d84 0n11652 ffff988d933c9080 McCSPServiceHost.exe                         672ms         846ms        1s.518
0x2dc0 0n11712 ffff988d9356b080 ModuleCoreService.exe                      10s.235        2s.799       13s.034
0x2dd0 0n11728 ffff988d9356e080 conhost.exe                                   16ms             0          16ms
0x2de8 0n11752 ffff988d935b60c0 mcapexe.exe                                  141ms         251ms         392ms
0x2ee0 0n12000 ffff988d935cd080 MfeAVSvc.exe                               11s.829        6s.283       18s.112
0x2eb4 0n11956 ffff988d92fc40c0 mcshield.exe                                8s.922        3s.111       12s.033
0x3264 0n12900 ffff988d93e240c0 svchost.exe                                      0             0             0
0x3320 0n13088 ffff988d93e2f080 svchost.exe                                  109ms          16ms         125ms
0x3328 0n13096 ffff988d93e32080 vmnat.exe*32                                 110ms         110ms         220ms
0x3330 0n13104 ffff988d93e17080 WMIRegistrationService.exe*32                 15ms          16ms          31ms
0x1310 0n4880  ffff988d940f4080 AacKingstonDramHal_x64.exe                       0          47ms          47ms
0x8c0  0n2240  ffff988d93aab080 svchost.exe                                   31ms          79ms         110ms
0x3450 0n13392 ffff988d933dc300 svchost.exe                                   16ms          63ms          79ms
0x1f48 0n8008  ffff988d9423f080 extensionCardHal_x86.exe*32                      0          31ms          31ms
0x37b4 0n14260 ffff988d93ee5080 AacKingstonDramHal_x86.exe*32                 31ms          94ms         125ms
0x3544 0n13636 ffff988d943c3280 Aac3572MbHal_x86.exe*32                      687ms       11s.969       12s.656
0x367c 0n13948 ffff988d80dd9080 dllhost.exe                                      0             0             0
0x1950 0n6480  ffff988d8f8cd0c0 servicehost.exe                              718ms        2s.751        3s.469
0xc4c  0n3148  ffff988d8f5020c0 SearchIndexer.exe                           4s.235        2s.720        6s.955
0x1a2c 0n6700  ffff988d94004080 BraveCrashHandler.exe*32                         0          16ms          16ms
0x37cc 0n14284 ffff988d80e20080 BraveCrashHandler64.exe                       16ms             0          16ms
0x1c30 0n7216  ffff988d94267080 CCleanerBrowserCrashHandler.exe*32               0             0             0
0x1c70 0n7280  ffff988d94d7f080 CCleanerBrowserCrashHandler64.exe                0          31ms          31ms
0x2314 0n8980  ffff988d94d82080 svchost.exe                                      0          63ms          63ms
0x3354 0n13140 ffff988d94d8d080 GoogleCrashHandler.exe*32                        0          32ms          32ms
0x1170 0n4464  ffff988d96497080 GoogleCrashHandler64.exe                         0          31ms          31ms
0x1678 0n5752  ffff988d96495080 SgrmBroker.exe                               109ms          79ms         188ms
0x15c0 0n5568  ffff988d8f7ef080 TTHOMEService.exe                                0          16ms          16ms
0x76c  0n1900  ffff988d964cf080 svchost.exe                                   78ms         157ms         235ms
0x2af0 0n10992 ffff988d965cc080 svchost.exe                                   47ms          47ms          94ms
0x1e4  0n484   ffff988d96b230c0 uihost.exe                                  2s.156        4s.595        6s.751
0x1118 0n4376  ffff988d96ae3080 ArmouryCrate.UserSessionHelper.exe          2s.657        3s.378        6s.035
0x22fc 0n8956  ffff988d96b41080 nvcontainer.exe                              516ms        1s.141        1s.657
0x28b0 0n10416 ffff988d96c93300 sihost.exe                                   984ms        1s.360        2s.344
0x1448 0n5192  ffff988d96cda140 svchost.exe                                  437ms         313ms         750ms
0x1c44 0n7236  ffff988d96bef080 svchost.exe                                 3s.125         797ms        3s.922
0x1c4c 0n7244  ffff988d96e640c0 nvcontainer.exe                            21s.485        5s.907       27s.392
0x2aec 0n10988 ffff988d96e7f080 svchost.exe                                  906ms         516ms        1s.422
0x2f08 0n12040 ffff988d96dfd080 ArmourySocketServer.exe                      187ms         171ms         358ms
0x13a0 0n5024  ffff988d96dc9080 asus_framework.exe*32                       3s.500         156ms        3s.656
0x26f0 0n9968  ffff988d96ed70c0 NoiseCancelingEngine.exe                         0          16ms          16ms
0x2b68 0n11112 ffff988d96eda080 taskhostw.exe                                203ms         172ms         375ms
0x2568 0n9576  ffff988d96eef080 ArmouryAIOFanServer.exe                       31ms          95ms         126ms
0xa40  0n2624  ffff988d96f830c0 AcPowerNotification.exe*32                   172ms         203ms         375ms
0x1768 0n5992  ffff988d96f76080 svchost.exe                                   16ms          32ms          48ms
0x24c4 0n9412  ffff988d970a5340 ctfmon.exe                                   343ms         782ms        1s.125
0xfbc  0n4028  ffff988d970a8080 explorer.exe                               13s.814       41s.878       55s.692
0x38f0 0n14576 ffff988d971d5080 NahimicSvc64.exe                             140ms         345ms         485ms
0x3908 0n14600 ffff988d970c5080 NahimicSvc32.exe*32                          125ms         484ms         609ms
0x395c 0n14684 ffff988d971c5300 svchost.exe                                  484ms          94ms         578ms
0x39e8 0n14824 ffff988d944d1080 NahimicSvc64.exe                             251ms         438ms         689ms
0x39fc 0n14844 ffff988d9459d080 NahimicSvc32.exe*32                         1s.311        9s.515       10s.826
0x3b64 0n15204 ffff988d97199080 NVIDIA Web Helper.exe*32                    1s.157         503ms        1s.660
0x3bfc 0n15356 ffff988d944d9080 svchost.exe                                   46ms          47ms          93ms
0x163c 0n5692  ffff988d971c3300 conhost.exe                                   16ms          32ms          48ms
0x1744 0n5956  ffff988d94aea0c0 asus_framework.exe*32                        204ms         314ms         518ms
0x1990 0n6544  ffff988d94ab4080 StartMenuExperienceHost.exe                 1s.078         625ms        1s.703
0x3964 0n14692 ffff988d97c85080 RuntimeBroker.exe                            437ms         547ms         984ms
0x1e00 0n7680  ffff988d97c95080 SearchApp.exe                               7s.394        1s.928        9s.322
0x3cdc 0n15580 ffff988d94af6080 RuntimeBroker.exe                            798ms        1s.079        1s.877
0x3dd0 0n15824 ffff988d98030080 svchost.exe                                      0          31ms          31ms
0x3fa8 0n16296 ffff988d97e8e0c0 conhost.exe                                   16ms             0          16ms
0x41a0 0n16800 ffff988d984980c0 RuntimeBroker.exe                            906ms        2s.125        3s.031
0x4734 0n18228 ffff988d988970c0 ArmourySwAgent.exe*32                        125ms         156ms         281ms
0x3d28 0n15656 ffff988d99221080 brave.exe                                1m:10.079     1m:00.111     2m:10.190
0x3f90 0n16272 ffff988d98c36080 brave.exe                                     16ms         110ms         126ms
0x3c7c 0n15484 ffff988d995950c0 nvsphelper64.exe                              63ms         188ms         251ms
0x3cb8 0n15544 ffff988d9959c0c0 NVIDIA Share.exe                            1s.908        2s.844        4s.752
0x4834 0n18484 ffff988d994e9080 NVIDIA Share.exe                             171ms         125ms         296ms
0x4980 0n18816 ffff988d99432080 NVIDIA Share.exe                            1s.593         203ms        1s.796
0x4aa0 0n19104 ffff988d990f3340 brave.exe                                7m:13.115     2m:26.364     9m:39.479
0x4aec 0n19180 ffff988d996e7200 brave.exe                                  19s.595       18s.593       38s.188
0x4b08 0n19208 ffff988d99695240 brave.exe                                     63ms          63ms         126ms
0x4b38 0n19256 ffff988d997b3200 brave.exe                                    453ms         187ms         640ms
0x282c 0n10284 ffff988d9997c080 brave.exe                                6m:15.127       36s.062     6m:51.189
0x48ec 0n18668 ffff988d99a95080 brave.exe                                   6s.704         781ms        7s.485
0x4a0c 0n18956 ffff988d99a7c080 brave.exe                                  12s.343        2s.204       14s.547
0x3db4 0n15796 ffff988d99ad90c0 brave.exe                                   1s.594         532ms        2s.126
0x978  0n2424  ffff988d99a9e080 brave.exe                                   5s.657         547ms        6s.204
0x4b24 0n19236 ffff988d99aa7080 brave.exe                                    157ms          47ms         204ms
0x4c5c 0n19548 ffff988d99a17080 brave.exe                                    328ms          94ms         422ms
0x4d70 0n19824 ffff988d99b0e080 TextInputHost.exe                            172ms         188ms         360ms
0x501c 0n20508 ffff988d97ca90c0 SecurityHealthSystray.exe                        0          94ms          94ms
0x504c 0n20556 ffff988d9a2c9080 SecurityHealthService.exe                    109ms         109ms         218ms
0x50dc 0n20700 ffff988d98dd2080 RtkAudUService64.exe                          16ms          78ms          94ms
0x5188 0n20872 ffff988d9a547080 CDASrv.exe                                    47ms          94ms         141ms
0x51a0 0n20896 ffff988d991f0080 brave.exe                                   2s.171        2s.234        4s.405
0x527c 0n21116 ffff988d98e54080 tvnserver.exe                                890ms        4s.344        5s.234
0x52d0 0n21200 ffff988d99eca080 svchost.exe                                      0          16ms          16ms
0x53cc 0n21452 ffff988d9a3020c0 iCUE.exe                                   50s.330     1m:00.769     1m:51.099
0x53e4 0n21476 ffff988d99476080 lghub.exe                                    312ms         611ms         923ms
0x511c 0n20764 ffff988d9a213080 lghub_agent.exe                             1s.626        1s.062        2s.688
0x4f00 0n20224 ffff988d9a219080 lghub.exe                                    140ms         109ms         249ms
0x5174 0n20852 ffff988d97f57180 lghub.exe                                        0         126ms         126ms
0x510c 0n20748 ffff988d9aa24080 unsecapp.exe                                  31ms         109ms         140ms
0x47b0 0n18352 ffff988d9ac14080 CCleaner64.exe                               312ms         985ms        1s.297
0x5420 0n21536 ffff988d9aaa9080 svchost.exe                                   46ms         344ms         390ms
0x5450 0n21584 ffff988d9921b080 IGCCTray.exe                                 813ms         376ms        1s.189
0x55e0 0n21984 ffff988d9ae26340 IGCC.exe                                     110ms         204ms         314ms
0x5554 0n21844 ffff988d9b0f5080 PowerPanel Personal.exe*32                  8s.969        4s.218       13s.187
0x5580 0n21888 ffff988d9b091080 vmware-tray.exe*32                            31ms         110ms         141ms
0x57dc 0n22492 ffff988d9b2aa080 iCUEDevicePluginHost.exe                      47ms          16ms          63ms
0x3c28 0n15400 ffff988d9b284080 iCUEDevicePluginHost.exe                   18s.734       11s.798       30s.532
0x56f4 0n22260 ffff988d9b274080 conhost.exe                                   15ms          16ms          31ms
0x563c 0n22076 ffff988d9b26a080 iCUEDevicePluginHost.exe                      31ms             0          31ms
0x55c0 0n21952 ffff988d9b385080 conhost.exe                                   16ms             0          16ms
0x557c 0n21884 ffff988d98eac080 iCUEDevicePluginHost.exe                      16ms             0          16ms
0x5778 0n22392 ffff988d9af822c0 conhost.exe                                   16ms             0          16ms
0x604  0n1540  ffff988d9b388080 iCUEDevicePluginHost.exe                      15ms          63ms          78ms
0x52b4 0n21172 ffff988d9b386080 conhost.exe                                   15ms             0          15ms
0x54b8 0n21688 ffff988d9b30a080 iCUEDevicePluginHost.exe                      16ms          16ms          32ms
0x5584 0n21892 ffff988d9b321080 iCUEDevicePluginHost.exe                      16ms          16ms          32ms
0x5134 0n20788 ffff988d9b3640c0 conhost.exe                                      0             0             0
0x582c 0n22572 ffff988d99af4080 iCUEDevicePluginHost.exe                     188ms        4s.906        5s.094
0x5834 0n22580 ffff988d9b38a080 conhost.exe                                      0             0             0
0x5840 0n22592 ffff988d9b4130c0 conhost.exe                                   16ms             0          16ms
0x58e0 0n22752 ffff988d99f0b2c0 ppuser.exe*32                                    0             0             0
0x58fc 0n22780 ffff988d97be9300 NetFaxTray64.exe                              16ms          63ms          79ms
0x591c 0n22812 ffff988d99c50080 brave.exe                                     63ms          63ms         126ms
0x5ab4 0n23220 ffff988d9956a080 brave.exe                                  11s.109        5s.031       16s.140
0x3dfc 0n15868 ffff988d9c0c8080 asusns.exe                                   297ms         642ms         939ms
0x3968 0n14696 ffff988d9cc14300 extensionCardHal_x64.exe                         0          94ms          94ms
0x4d84 0n19844 ffff988d9c14f140 AacKingstonDramHal_x64.exe                    16ms         141ms         157ms
0x5afc 0n23292 ffff988d9cc42080 Aac3572MbHal_x64.exe                       35s.141    19m:31.360    20m:06.501
0x26c0 0n9920  ffff988d9c3542c0 ShellExperienceHost.exe                      312ms         299ms         611ms
0x4394 0n17300 ffff988d9cd31080 RuntimeBroker.exe                             62ms          32ms          94ms
0x62a4 0n25252 ffff988d9b259080 ModuleCoreService.exe                        658ms         642ms        1s.300
0x6240 0n25152 ffff988d9cd85080 conhost.exe                                   31ms             0          31ms
0x5d64 0n23908 ffff988d8bbe2080 McUICnt.exe                                  828ms        1s.267        2s.095
0x5d70 0n23920 ffff988d96b2e300 svchost.exe                                   93ms         156ms         249ms
0x63bc 0n25532 ffff988d9cd87080 svchost.exe                                      0          16ms          16ms
0x5d08 0n23816 ffff988d9951d080 AdobeNotificationClient.exe*32                46ms         157ms         203ms
0x479c 0n18332 ffff988d9d5d32c0 RuntimeBroker.exe                             46ms          31ms          77ms
0x21c8 0n8648  ffff988d9d5d6080 svchost.exe                                   78ms          94ms         172ms
0x8b8  0n2232  ffff988d98891080 gamingservices.exe                           329ms        1s.250        1s.579
0x5c90 0n23696 ffff988d9e832080 gamingservicesnet.exe                            0             0             0
0xa58  0n2648  ffff988d981d0080 gameinputsvc.exe                                 0          16ms          16ms
0x5f14 0n24340 ffff988d97cb7080 gameinputsvc.exe                            1s.453        5s.703        7s.156
0xda8  0n3496  ffff988d96ee3080 svchost.exe                                   47ms          31ms          78ms
0x31cc 0n12748 ffff988d9cc2a080 UserOOBEBroker.exe                               0             0             0
0x4608 0n17928 ffff988d92e31340 svchost.exe                                   62ms          31ms          93ms
0x27a8 0n10152 ffff988d97e5e080 dllhost.exe                                   16ms          78ms          94ms
0x9a4  0n2468  ffff988d866680c0 LorexCloud.exe                        1h:14:19.897     9m:16.085  1h:23:35.982
0x60b8 0n24760 ffff988d975d9080 DSMessageNotify.exe                              0         125ms         125ms
0x1728 0n5928  ffff988d97904080 brave.exe                               12m:55.502     1m:01.736    13m:57.238
0x3480 0n13440 ffff988d9dcf2080 brave.exe                                    296ms          47ms         343ms
0xf34  0n3892  ffff988d9e8c0280 brave.exe                                    563ms          79ms         642ms
0x2af4 0n10996 ffff988d9cc2d080 brave.exe                                    609ms          47ms         656ms
0x1038 0n4152  ffff988d9e1e3080 obs64.exe                             1h:56:11.030     3m:12.298  1h:59:23.328
0x268c 0n9868  ffff988d9cce1080 ApplicationFrameHost.exe                     110ms          63ms         173ms
0x35d4 0n13780 ffff988d9c7ed340 HxOutlook.exe                                752ms         453ms        1s.205
0x3b88 0n15240 ffff988d9c7e6300 RuntimeBroker.exe                             15ms          16ms          31ms
0x5190 0n20880 ffff988d9f2db2c0 HxTsr.exe                                    813ms         188ms        1s.001
0x2acc 0n10956 ffff988d9baf2080 svchost.exe                                   62ms          31ms          93ms
0x3200 0n12800 ffff988d9d8dd080 GameBar.exe                                   94ms         173ms         267ms
0x520  0n1312  ffff988d9e8a7080 GameBarFTServer.exe                           16ms         141ms         157ms
0x3934 0n14644 ffff988d9c9e9080 RuntimeBroker.exe                             15ms          63ms          78ms
0x1670 0n5744  ffff988d945d1080 svchost.exe                                      0         141ms         141ms
0x4f1c 0n20252 ffff988d9fde70c0 Microsoft.Photos.exe                         454ms         375ms         829ms
0x63c8 0n25544 ffff988d9d4e0080 RuntimeBroker.exe                            375ms         516ms         891ms
0x542c 0n21548 ffff988d9c20f080 svchost.exe                                   31ms          32ms          63ms
0x5e48 0n24136 ffff988d97185080 QcShm.exe                                     47ms          16ms          63ms
0x5c1c 0n23580 ffff988d9cad7080 svchost.exe                                      0          16ms          16ms
0x39a4 0n14756 ffff988d9cd4f080 brave.exe                                        0          47ms          47ms
0x145c 0n5212  ffff988d8e2c9080 CCleanerBrowserUpdate.exe*32                     0             0             0
0x64d8 0n25816 ffff988da107b080 McCBEntAndInstru.exe                             0             0             0
0x6610 0n26128 ffff988da107a240 mcdatrep.exe*32                               16ms             0          16ms
0x6618 0n26136 ffff988d8e2c8080 conhost.exe                                      0             0             0
============== ================ ===================================== ============ ============= =============
PID            Address          Name                                          User        Kernel         Total

Warning! Zombie process(es) detected (not displayed). Count: 91 [zombie report]
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
