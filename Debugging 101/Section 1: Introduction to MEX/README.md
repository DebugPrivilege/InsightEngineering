# What is the MEX extension?

**MEX** is a debugging extension that is written in C#. It simplify common debugger tasks, and provides powerful text filtering capabilities to the debugger. We will be covering how to use this extension to navigate kernel memory dumps. I'm not a big fan of cheat sheets that lacks context, so the purpose is to demonstrate some useful commands with an explanation of the output that we're seeing.

This extension can be downloaded here: https://www.microsoft.com/en-us/download/details.aspx?id=53304

# How to take a memory dump?

In order to start navigating with **MEX**, we need a memory dump first. Let's configure our virtual machine to support **complete memory dump**. A **complete memory dump** is a comprehensive capture of the system's physical memory (RAM) contents at the time of a crash. This dump contains both the kernel-mode and user-mode memory spaces.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/5a360e72-47f2-429c-ba72-718da8796b5d)


The reason we configure a **complete memory dump** is that it facilitates demonstrations when compared to a **kernel memory dump**. The latter captures only the memory used by the kernel and excludes memory allocated to user-mode applications. However, I highly recommend taking memory dumps of both types to better understand the differences between them.

1. Download **NotMyFault** from Sysinternals: https://learn.microsoft.com/en-us/sysinternals/downloads/notmyfault
2. Open CMD as an administrator and type the following command: **NotMyFault64.exe /crash**
3. This will generate a BSOD and by default the **MEMORY.DMP** will be saved to **C:\Windows** folder.
4. Load the **MEMORY.DMP** file in **WinDbg**.

# System Information

Let's first load the **MEX** extension:

```
0: kd> .load mex
Mex External 3.0.0.7172 Loaded!
```

**MEX** contains **255** commands that are available to you. For every command, there is a help option to gather better understanding on what it is doing.

```
0: kd> !mex.help
Mex currently has 255 extensions available.  Please specify a keyword to search.
Or browse by category:

All PowerShell[6] SystemCenter[3] Networking[12] Process[5] Mex[2] Kernel[27] DotNet[32] Decompile[15] Utility[40] Thread[27] Binaries[6] General[22]
```

We typically begin with the **`!mex.di`** command. This command retrieves information about the machine from which the memory dump originated. It's especially handy when working with multiple memory dumps, as it helps distinguish the analysis related to each specific dump. This is an improved version of the **vertarget** command.

```
0: kd> !mex.di
Mex External 3.0.0.7172 Loaded!
Computer Name: WINDEV2305EVAL
Windows 10 Kernel Version 22621 MP (4 procs) Free x64
Product: WinNt, suite: TerminalServer SingleUserTS
Edition build lab: 22621.1928.amd64fre.ni_release_svc_prod3.230622-0951
Kernel base = 0xfffff801`51600000 PsLoadedModuleList = 0xfffff801`522130e0
Debug session time: Sun Aug 13 04:53:59.099 2023 (UTC - 7:00)
System Uptime: 0 days 0:28:16.641
SystemManufacturer = Microsoft Corporation
SystemProductName = Virtual Machine
Processor: 11th Gen Intel(R) Core(TM) i7-1185G7 @ 3.00GHz
Bugcheck: 5A (0, 0, 0, 0)
Bugcheck: This is a myfault.sys dump.
KernelMode Full Memory Dump Path: C:\Windows\MEMORY.DMP
```

There are some important fields that we can use for our analysis:
- **Computer Name**
- **Debug session time** 
- **System Uptime:** The system was running for 28 minutes and 16.641 seconds before the dump.
- **SystemManufacturer:** Microsoft Corporation (indicating it's a virtual machine hosted on a Microsoft platform)
- **Bugcheck:** 0x5A translates to **CRITICAL_SERVICE_FAILED**
- **KernelMode:** Full Memory Dump indicates that it is a complete memory dump containing both kernel-mode and user-mode memory spaces.

Once the **MEX** extension is loaded, there's no need to prefix "mex" before every command, thanks to **MEX** supporting aliases. For instance, we can simply type the **`!di`** command, and it will display the same information.

```
0: kd> !di
Computer Name: WINDEV2305EVAL
Windows 10 Kernel Version 22621 MP (4 procs) Free x64
Product: WinNt, suite: TerminalServer SingleUserTS
Edition build lab: 22621.1928.amd64fre.ni_release_svc_prod3.230622-0951
Kernel base = 0xfffff801`51600000 PsLoadedModuleList = 0xfffff801`522130e0
Debug session time: Sun Aug 13 04:53:59.099 2023 (UTC - 7:00)
System Uptime: 0 days 0:28:16.641
SystemManufacturer = Microsoft Corporation
SystemProductName = Virtual Machine
Processor: 11th Gen Intel(R) Core(TM) i7-1185G7 @ 3.00GHz
Bugcheck: 5A (0, 0, 0, 0)
Bugcheck: This is a myfault.sys dump.
KernelMode Full Memory Dump Path: C:\Windows\MEMORY.DMP
```

As discussed previously, every command contains a help **`(-h)`** option. Let's take a look at that, and see how that looks like:

```
0: kd> !di -h
!dumpinfo (!di) - Display dump information

Usage:
    !dumpinfo [-v] 
                    NoArgs - Dumps out the details of the current dump file.
        -v|-verbose    : Verbose Mode

    !dumpinfo [-?|-h] 
        -?|-h|-help    : Display this help text

Keywords: dump, information, details
Current Owner: mexfeedback
```

Ah, there is a **-v** option that shows the verbose mode. If we type **`!di -v`** it will show us a different output:

```
0: kd> !di -v
Computer Name: WINDEV2305EVAL
Windows 10 Kernel Version 22621 MP (4 procs) Free x64
Product: WinNt, suite: TerminalServer SingleUserTS
Edition build lab: 22621.1928.amd64fre.ni_release_svc_prod3.230622-0951
Kernel base = 0xfffff801`51600000 PsLoadedModuleList = 0xfffff801`522130e0
Debug session time: Sun Aug 13 04:53:59.099 2023 (UTC - 7:00)
System Uptime: 0 days 0:28:16.641
SystemManufacturer = Microsoft Corporation
SystemProductName = Virtual Machine
Processor: 11th Gen Intel(R) Core(TM) i7-1185G7 @ 3.00GHz
Bugcheck: 5A (0, 0, 0, 0)
Bugcheck: This is a myfault.sys dump.
KernelMode Full Memory Dump Path: C:\Windows\MEMORY.DMP
Share Path: \\WINDEV2305EVAL\ADMIN$\MEMORY.DMP
Verbose mode

Event        Times (UTC - 7:00)
============ ==========================
System Start 08/13/2023 04:25:42.458 AM
Dump Start   08/13/2023 04:53:59.099 AM

Stat          Duration
============= ==========
System Uptime 28m:16.641


File size 4,386,490,421
File date 08/13/2023 04:54:47
ExceptionAddress: fffff80151a31250 (nt!KeBugCheckEx)
   ExceptionCode: 80000003 (Break instruction exception)
  ExceptionFlags: 00000001
NumberParameters: 0

Local Machine Name WINDEV2305EVAL (WINDEV2305EVAL\User)
```

The verbose details provide information like:

- **Event Times:** Lists the system start time and the time the memory dump began.
- **Statistics:** Provides the system uptime duration.
- **File Information:** Details about the memory dump file, including its size and creation date.

If we can't get the computer name with the **`!di`** command, we can try the **`!cn`** command which is short for 'computer name'

```
0: kd> !cn
Computer Name: WINDEV2305EVAL
```

Another useful command is **`!crash`**. It helps us quickly find the crashing call stack. While it's similar to **`!analyze -v`**, it provides shorter explanations about the bugcheck.

```
5: kd> !crash
Dump Info
============================================
Computer Name:  Not Found
Windows 10 Kernel Version 22621 MP (8 procs) Free x64
Product: WinNt, suite: TerminalServer SingleUserTS
Edition build lab: 22621.1.amd64fre.ni_release.220506-1250
Kernel base = 0xfffff802`1ae00000 PsLoadedModuleList = 0xfffff802`1ba134b0
Debug session time: Sun Jul  9 07:30:00.740 2023 (UTC - 7:00)
System Uptime: 8 days 0:39:50.817
SystemManufacturer = LENOVO
SystemProductName = 20Y0S0280W
Processor: 11th Gen Intel(R) Core(TM) i7-1185G7 @ 3.00GHz
Bugcheck: 19C (50, FFFFD50C0EFF9080, 0, 0)
KernelMode Dump Path: C:\Users\User\Desktop\MEMORY.DMP
Share Path: \\WINDEV2305EVAL\C$\Users\User\Desktop\MEMORY.DMP


Bugcheck details
============================================
Bugcheck code 0000019C
Arguments 00000000`00000050 ffffd50c`0eff9080 00000000`00000000 00000000`00000000

Crashing Stack
============================================
 # Child-SP          RetAddr               Call Site
00 ffffbe8d`3fe471c8 fffff802`1b30948a     nt!KeBugCheckEx
01 ffffbe8d`3fe471d0 fffff802`1b0b0c65     nt!PopWatchdogWorker+0x15b15a
02 ffffbe8d`3fe47300 fffff802`1b0d8dc7     nt!ExpWorkerThread+0x155
03 ffffbe8d`3fe474f0 fffff802`1b2311e4     nt!PspSystemThreadStartup+0x57
04 ffffbe8d`3fe47540 00000000`00000000     nt!KiStartSystemThread+0x34
```

# Threads

After we ran the **`!crash`** command and got our initial analysis of the crashing call stack. Let's find out to which thread this call stack belongs to. We can use the **`!w`** command, which is short for *'Where am I?'* - The output provides details about a specific process, which includes the associated thread and session ID in which the process was running.

```
0: kd> !w
Session: 1
Process: ffffa389afab70c0 notmyfault64.exe
Thread:  ffffa389b497f080 notmyfault64.exe ffffa389afab70c0
Pid: 20f0
Tid: 1c4c
Frame: 0
Teb: ca7d684000
Dbgid: 0 (Debugger thread ID in usermode / Processor ID in kernelmode)
```

Let's gather more information about this specific thread. We can use the **!mex.t** command to analyze threads. First, let's find out which options it supports.

```
0: kd> !t -h
!t - A new implementation of !thread for user & kernel mode

Usage:
    !t [-t <Thread>] [-d] [-c] [-l] [-i] [-f | -fv] [-v] [-raw] [-cxr <cxr>] [-w] [<ThreadId>] 
        -t|-tid <Thread>    : Specify a different thread by TID or thread number (Default Radix: Base 10)
        -d                  : Sets your context back to the current thread after running !t
        -c                  : Clean display (no stack pointers, no nothing). Like kc
        -l                  : Hide source lines (disregard .lines preference)
        -i|-info            : Only show the thread info, not the stack
        -f                  : Show frame size
        -fv                 : Show frame size with summary stats
        -v|-verbose         : Verbose
        -raw                : Dumps (dps) from the stack limit the base only showing items that include the ! followed by +0x
        -cxr <cxr>          : sets the context record for the current thread
        -w                  : Show windows associated with the thread
        ThreadId            : _ETHREAD address, or debugger ID (Default Radix: Hex in Kernel, Base 10 in Usermode)

    !t {-a | -av} [-t <Thread>] [<ThreadId>] 
        -a                  : Analyze a thead to try to determine why it is waiting
        -av                 : Analyze a thead (verbose mode) to try to determine why it is waiting
        -t|-tid <Thread>    : Specify a different thread by TID or thread number (Default Radix: Base 10)
        ThreadId            : _ETHREAD address, or debugger ID (Default Radix: Hex in Kernel, Base 10 in Usermode)

    !t [-?|-h] 
        -?|-h|-help    : Display this help text

                e.g.  !t 12 - display thread 12 where 12 is the usermode debugger id for a thread
                      !t -t 0x3a08 - display the thread with the thread ID of 0x3a08
                      !t 83e60b30 - display the thread with the ethread address of 83e60b30
                      !t   display the current thread (you can check your current thread context with !mex.context
Current Owner: mexfeedback
```

We can now view detailed information about the thread responsible for the crash with the **`!t`** command (short for thread), including its **state** and the **CPU** on which it's operating. In this particular case, **the thread has been running on CPU 0 for 15ms**.

```
0: kd> !t ffffa389b497f080
Process                             Thread                       CID       TEB              UserTime KernelTime ContextSwitches Wait Reason Time State
notmyfault64.exe (ffffa389afab70c0) ffffa389b497f080 (E|K|W|R|V) 20f0.1c4c 000000ca7d684000        0       16ms              56 WrLpcReply  15ms Running on processor 0

Irp List:
    IRP              File Driver
    ffffa389b17fe400      MYFAULT

Priority:
    Current Base Decrement ForegroundBoost IO Page
    9       8    0         0               0  5

# Child-SP         Return           Call Site
0 ffff85893e726ea8 fffff801781e101b nt!KeBugCheckEx
1 ffff85893e726eb0 fffff801781e169d myfault+0x101b
2 ffff85893e726ef0 fffff801781e17f1 myfault+0x169d
3 ffff85893e727030 fffff8015181f5f5 myfault+0x17f1
4 ffff85893e727090 fffff80151cc5200 nt!IofCallDriver+0x55
5 ffff85893e7270d0 fffff80151cc6c37 nt!IopSynchronousServiceTail+0x1d0
6 ffff85893e727180 fffff80151cc6516 nt!IopXxxControlFile+0x707
7 ffff85893e727380 fffff80151a45fe5 nt!NtDeviceIoControlFile+0x56
8 ffff85893e7273f0 00007ff8346eee34 nt!KiSystemServiceCopyEnd+0x25
9 000000ca7d8ff738 00007ff831f360eb ntdll!NtDeviceIoControlFile+0x14
a 000000ca7d8ff740 00007ff8325a27f1 KERNELBASE!DeviceIoControl+0x6b
b 000000ca7d8ff7b0 00007ff7ebea437e KERNEL32!DeviceIoControlImplementation+0x81
c 000000ca7d8ff800 0000000000000000 notmyfault64+0x437e
```

We remain in the process context of **notmyfault64.exe**. Another action we can take is to execute the **`!p`** command, which stands for "process". This command provides an overview of the modules, handles, and threads within the process. It also reveals the user under whom the process was running and offers details about memory usage. 

```
0: kd> !p
Name             Address                  Ses PID           Parent        PEB              Create Time                Mods Handle Thrd User Name
================ ======================== === ============= ============= ================ ========================== ==== ====== ==== ===================
notmyfault64.exe ffffa389afab70c0 (E|K|O)   1 20f0 (0n8432) 1890 (0n6288) 000000ca7d683000 08/13/2023 04:53:59.028 AM   21      0    4 WINDEV2305EVAL\User

Command Line: notmyfault64.exe  /bugcheck 0x5A

Memory Details:

    VM      Peak    Commit Size PP Quota  NPP Quota
    ======= ======= =========== ========= =========
    4.09 GB 4.09 GB     1.21 MB 121.32 KB   7.85 KB

Show LPC Port information for process

Show Threads: Unique Stacks    !mex.listthreads (!lt) ffffa389afab70c0    !process ffffa389afab70c0 7
```
There are **4** threads within process, which we can dump out as well. This can be achieved using the **!lt** command, which stands for "list threads".

```
0: kd> !lt
Process           PID Thread             Id State   Time Reason
================ ==== ================ ==== ======= ==== ==========
notmyfault64.exe 20f0 ffffa389b497f080 1c4c Running 15ms WrLpcReply
notmyfault64.exe 20f0 ffffa389b3ec8080 2100 Waiting 93ms WrQueue
notmyfault64.exe 20f0 ffffa389b35c1080 26ac Waiting 93ms WrQueue
notmyfault64.exe 20f0 ffffa389b284b080 2610 Waiting 93ms WrQueue

Thread Count: 4
```

If we want to inspect a thread, we can just click on it to find more information. Here is an example where **this thread has been waiting 93ms for incoming work**.

```
0: kd> !t ffffa389b3ec8080
Process                             Thread                       CID       TEB              UserTime KernelTime ContextSwitches Wait Reason Time State
notmyfault64.exe (ffffa389afab70c0) ffffa389b3ec8080 (E|K|W|R|V) 20f0.2100 000000ca7d686000        0          0              17 WrQueue     93ms Waiting

WaitBlockList:
    Object           Type  Other Waiters
    ffffa389b3f7c640 Queue             2

# Child-SP         Return           Call Site
0 ffff85893b286a10 fffff8015185e095 nt!KiSwapContext+0x76
1 ffff85893b286b50 fffff8015185c3c4 nt!KiSwapThread+0xae5
2 ffff85893b286ca0 fffff8015193102d nt!KiCommitThreadWait+0x134
3 ffff85893b286d50 fffff8015192fcc8 nt!KeRemoveQueueEx+0x111d
4 ffff85893b287110 fffff8015192f521 nt!IoRemoveIoCompletion+0x98
5 ffff85893b287230 fffff80151a45fe5 nt!NtWaitForWorkViaWorkerFactory+0x381
6 ffff85893b2873f0 00007ff8346f29a4 nt!KiSystemServiceCopyEnd+0x25
7 000000ca7d9ff738 00007ff83468536e ntdll!NtWaitForWorkViaWorkerFactory+0x14
8 000000ca7d9ff740 00007ff8325b26ad ntdll!TppWorkerThread+0x2ee
9 000000ca7d9ffa20 00007ff8346aaa68 KERNEL32!BaseThreadInitThunk+0x1d
a 000000ca7d9ffa50 0000000000000000 ntdll!RtlUserThreadStart+0x28
```
Let's take a look which options we have with the **!lt** command:

```
0: kd> !lt -h
!listthreads (!lt) - Displays a list of threads

Usage:
    !listthreads [-v] [-z] [-wr <waitReasonFilter>] [-s <stateFilter>] [-b] [-wf] [-a] [-r] [-pid <PID>] [<Process Address>] 
        -v|-verbose                : Verbose analysis of UserRequest threads (runs slower).
        -z|-zombie                 : Show zombie threads (kernel).
        -wr <waitReasonFilter>     : Wait reason you want to see
        -s|-state <stateFilter>    : Thread state you want to see
        -b|-bk                     : Show blocked threads
        -wf                        : Show wait function
        -a|-all                    : Show all threads from all processes (kernel)
        -r|-report                 : Show thread report (kernel)
        -pid <PID>                 : Shows a specificed process's threads given its ID (PID)
        Process Address            : Address of _EPROCESS Object

    !listthreads [-?|-h] 
        -?|-h|-help    : Display this help text

Current Owner: mexfeedback
```

Instead of listening all the threads, we can filter on the thread state. For example, let's find out which threads are currently running in the context of **notmyfault64.exe**.

```
0: kd> !lt -s running
Process           PID Thread             Id State   Time Reason
================ ==== ================ ==== ======= ==== ==========
notmyfault64.exe 20f0 ffffa389b497f080 1c4c Running 15ms WrLpcReply

Thread Count: 1
```

We can also filter on a thread wait reason, like for example looking at threads that are waiting for incoming work.

```
0: kd> !lt -wr WrQueue
Process           PID Thread             Id State   Time Reason
================ ==== ================ ==== ======= ==== =======
notmyfault64.exe 20f0 ffffa389b3ec8080 2100 Waiting 93ms WrQueue
notmyfault64.exe 20f0 ffffa389b35c1080 26ac Waiting 93ms WrQueue
notmyfault64.exe 20f0 ffffa389b284b080 2610 Waiting 93ms WrQueue
```

While we're aware that a thread within **notmyfault64.exe** was running, suppose we wanted to identify other processes with active threads. The **`!running`** command can help us with this. Its output provides a concise overview of active threads and the processes they are associated with.

```
0: kd> !running
Process           PID Thread             Id Pri Base Pri Next CPU CSwitches    User     Kernel State     Time Reason
================ ==== ================ ==== === ======== ======== ========= ======= ========== ======= ====== ===========
notmyfault64.exe 20f0 ffffa389b497f080 1c4c   9        8        0        56       0       16ms Running   15ms WrLpcReply
Idle                0 ffffa389ae6b0040    0   0        0        1    119807       0 12m:07.156 Running  453ms Executive
Idle                0 ffffa389ae767080    0   0        0        2     52869       0 12m:08.781 Running 6s.718 Executive
Sysmon.exe        f30 ffffa389b411a080 12c8   9        8        3      5017 11s.453     9s.313 Running   15ms UserRequest

Count: 4 | Show Unique Stacks
```

We can for example switch to a different process context like **Sysmon.exe** and then examine this process:

```
0: kd> !mex.p ffffa389b3d44080
Name       Address                  Ses PID          Parent      PEB              Create Time                Mods Handle Thrd User Name
========== ======================== === ============ =========== ================ ========================== ==== ====== ==== =========================
Sysmon.exe ffffa389b3d44080 (E|K|O)   0 f30 (0n3888) 368 (0n872) 0000005871e4a000 08/13/2023 04:19:13.797 AM   65      0   11 WORKGROUP\WINDEV2305EVAL$

Command Line: C:\Windows\Sysmon.exe

Memory Details:

    VM      Peak    Commit Size PP Quota  NPP Quota
    ======= ======= =========== ========= =========
    4.11 GB 4.13 GB     9.69 MB 144.42 KB 272.86 KB

Show LPC Port information for process

Show Threads: Unique Stacks    !mex.listthreads (!lt) ffffa389b3d44080    !process ffffa389b3d44080 7
```

When we run the **`!w`** command, we are able to see that we have switched the context of our process to **Sysmon.exe**.

```
0: kd> !w
Session: 0
Process: ffffa389b3d44080 Sysmon.exe
Thread:  ffffa389b3cdb080 Sysmon.exe ffffa389b3d44080
Pid: f30
Tid: f34
Frame: 0
Teb: 5871e4b000
Dbgid: 0 (Debugger thread ID in usermode / Processor ID in kernelmode)
```

Let's now list all the threads of this process with the **`!lt`** command. There are in total **11** threads as we can see in the results:

```
0: kd> !lt
Process    PID Thread             Id State         Time Reason
========== === ================ ==== ======= ========== ===========
Sysmon.exe f30 ffffa389b3cdb080  f34 Waiting 10m:52.531 UserRequest
Sysmon.exe f30 ffffa389b3eea080 1034 Waiting 27m:55.218 UserRequest
Sysmon.exe f30 ffffa389b4120080 12b0 Waiting      265ms UserRequest
Sysmon.exe f30 ffffa389b411f080 12b4 Waiting      140ms UserRequest
Sysmon.exe f30 ffffa389b411d080 12bc Waiting       15ms UserRequest
Sysmon.exe f30 ffffa389b411c080 12c0 Waiting 27m:55.218 Executive
Sysmon.exe f30 ffffa389b411a080 12c8 Running       15ms UserRequest
Sysmon.exe f30 ffffa389b4114080 12e4 Waiting     8s.468 UserRequest
Sysmon.exe f30 ffffa389b423d080 12fc Waiting 27m:25.156 UserRequest
Sysmon.exe f30 ffffa389b40f5080 1314 Waiting 25m:48.875 WrQueue
Sysmon.exe f30 ffffa389b40f9080  224 Waiting       15ms UserRequest

Thread Count: 11
```

Let's view a random thread in the results that we got. As a result, we can see here that a **thread has been waiting 15ms on a user mode request**.

```
0: kd> !mex.t ffffa389b40f9080
Process                       Thread                       CID       TEB              UserTime KernelTime ContextSwitches Wait Reason Time State
Sysmon.exe (ffffa389b3d44080) ffffa389b40f9080 (E|K|W|R|V) f30.224   0000005871f77000        0          0               3 UserRequest 15ms Waiting

WaitBlockList:
    Object           Type                 Other Waiters
    ffffa389af48fc60 SynchronizationEvent             0

# Child-SP         Return           Call Site
0 ffff85893b326330 fffff8015185e095 nt!KiSwapContext+0x76
1 ffff85893b326470 fffff8015185c3c4 nt!KiSwapThread+0xae5
2 ffff85893b3265c0 fffff8015185b746 nt!KiCommitThreadWait+0x134
3 ffff85893b326670 fffff8015194b830 nt!KeWaitForSingleObject+0x256
4 ffff85893b326a10 fffff80151cc4cf4 nt!KeWaitForMultipleObjects+0x640
5 ffff85893b326c60 fffff80151d286bc nt!ObWaitForMultipleObjects+0x2e4
6 ffff85893b327160 fffff80151a45fe5 nt!NtWaitForMultipleObjects+0x11c
7 ffff85893b3273f0 00007ff8346ef8a4 nt!KiSystemServiceCopyEnd+0x25
8 00000058720ff9f8 00007ff831f6f5a9 ntdll!NtWaitForMultipleObjects+0x14
9 00000058720ffa00 00007ff831c04aa7 KERNELBASE!WaitForMultipleObjectsEx+0xe9
a 00000058720ffce0 00007ff8325b26ad CRYPT32!ILS_WaitForThreadProc+0x37
b 00000058720ffd20 00007ff8346aaa68 KERNEL32!BaseThreadInitThunk+0x1d
c 00000058720ffd50 0000000000000000 ntdll!RtlUserThreadStart+0x28
```

Sometimes we may face challenges where we can't switch to a different thread context. In our context, the debugger couldn't switch to the desired thread because the system's paging file was not specified or configured.

```
0: kd> !mex.t ffffa389b411c080
Could not switch to thread. Error d0000147
Process                       Thread                       CID       TEB              UserTime KernelTime ContextSwitches Wait Reason       Time State
Sysmon.exe (ffffa389b3d44080) ffffa389b411c080 (E|K|W|R|V) f30.12c0  0000005871e61000        0          0               2 Executive   27m:55.218 Waiting

WaitBlockList:
    Object           Type                 Other Waiters
    ffffa389b40d3590 SynchronizationEvent             0

Kernel stack not resident
```

Another useful command is the **`!lt -wf`** which will display all the threads that are currently waiting. The "Wait Function" column points to the specific function or location where the thread is currently waiting. For instance, some threads are waiting on functions within the sechost module.

```
0: kd> !lt -wf
Process    PID Thread             Id State         Time Reason      Wait Function
========== === ================ ==== ======= ========== =========== ===========================================
Sysmon.exe f30 ffffa389b3cdb080  f34 Waiting 10m:52.531 UserRequest sechost!ScSendResponseReceiveControls+0x149
Sysmon.exe f30 ffffa389b3eea080 1034 Waiting 27m:55.218 UserRequest Kernel stack not resident
Sysmon.exe f30 ffffa389b4120080 12b0 Waiting      265ms UserRequest sechost!EtwpProcessRealTimeTraces+0x79
Sysmon.exe f30 ffffa389b411f080 12b4 Waiting      140ms UserRequest Sysmon+0x10d377
Sysmon.exe f30 ffffa389b411d080 12bc Waiting       15ms UserRequest Sysmon+0x107739
Sysmon.exe f30 ffffa389b411c080 12c0 Waiting 27m:55.218 Executive   Kernel stack not resident
Sysmon.exe f30 ffffa389b411a080 12c8 Running       15ms UserRequest 
Sysmon.exe f30 ffffa389b4114080 12e4 Waiting     8s.468 UserRequest sechost!EtwpProcessRealTimeTraces+0x79
Sysmon.exe f30 ffffa389b423d080 12fc Waiting 27m:25.156 UserRequest Kernel stack not resident
Sysmon.exe f30 ffffa389b40f5080 1314 Waiting 25m:48.875 WrQueue     Kernel stack not resident
Sysmon.exe f30 ffffa389b40f9080  224 Waiting       15ms UserRequest CRYPT32!ILS_WaitForThreadProc+0x37

Thread Count: 11
```

# Process

In the previous section, we focused on threads residing within a process. Now, let's examine "processes" from a broader perspective. The first command to consider is **`!tl`**, which stands for 'tasklist'. This command enumerates all the running processes present in the system at the time the memory dump was taken. Its intent is to provide a concise overview of all active processes, rather than detailed information.

```
0: kd> !tl
PID           Address          Name
============= ================ =======================================
0x0    0n0    fffff80152348f40 Idle
0x4    0n4    ffffa389ae6eb040 System
0x68   0n104  ffffa389ae6df080 Registry
0x1f8  0n504  ffffa389b1713080 smss.exe
0x28c  0n652  ffffa389b29960c0 csrss.exe
0x2d4  0n724  ffffa389b2def080 wininit.exe
0x2dc  0n732  ffffa389b2df1140 csrss.exe
0x338  0n824  ffffa389b2eaf080 winlogon.exe
0x368  0n872  ffffa389b2f08080 services.exe
0x370  0n880  ffffa389b29d30c0 LsaIso.exe
0x38c  0n908  ffffa389b2f09080 lsass.exe
0x22c  0n556  ffffa389b141e080 svchost.exe(-p)
0x25c  0n604  ffffa389b302f080 fontdrvhost.exe
0x218  0n536  ffffa389b3037080 fontdrvhost.exe
0x418  0n1048 ffffa389b2f5a080 svchost.exe(-p)
0x444  0n1092 ffffa389b2f9b080 svchost.exe(LSM)
0x494  0n1172 ffffa389b3168080 dwm.exe
0x4e4  0n1252 ffffa389b31e9080 svchost.exe(TermService)
0x508  0n1288 ffffa389b3210080 svchost.exe
0x544  0n1348 ffffa389b323b0c0 svchost.exe(NcbService)
0x550  0n1360 ffffa389b3240080 svchost.exe(-p)
0x560  0n1376 ffffa389b323e080 svchost.exe(TimeBrokerSvc)
0x5cc  0n1484 ffffa389b32c1080 svchost.exe(Schedule)
0x608  0n1544 ffffa389b32dc080 svchost.exe(ProfSvc)
0x628  0n1576 ffffa389b3351080 svchost.exe(nsi)
0x664  0n1636 ffffa389b336b080 svchost.exe(DispBrokerDesktopSvc)
0x66c  0n1644 ffffa389b336e080 svchost.exe(netprofm)
0x674  0n1652 ffffa389b335b080 svchost.exe(UmRdpService)
0x6b8  0n1720 ffffa389b33a0080 svchost.exe(UserManager)
0x720  0n1824 ffffa389b33c7080 svchost.exe(CertPropSvc)
0x738  0n1848 ffffa389b33d0080 svchost.exe(vmicheartbeat)
0x75c  0n1884 ffffa389b33e3080 svchost.exe(vmicrdv)
0x768  0n1896 ffffa389b33db080 svchost.exe
0x778  0n1912 ffffa389b33eb080 svchost.exe
0x790  0n1936 ffffa389b341c0c0 svchost.exe(vmictimesync)
0x7c8  0n1992 ffffa389b343b080 svchost.exe(vmicvss)
0x488  0n1160 ffffa389b34a1080 svchost.exe
0x86c  0n2156 ffffa389ae719080 svchost.exe(-p)
0x8e4  0n2276 ffffa389b349c080 VSSVC.exe
0x924  0n2340 ffffa389ae796080 svchost.exe(EventLog)
0x954  0n2388 ffffa389ae76e080 svchost.exe(SysMain)
0x960  0n2400 ffffa389ae770080 svchost.exe(EventSystem)
0x96c  0n2412 ffffa389ae75d080 svchost.exe
0x99c  0n2460 ffffa389ae78c080 svchost.exe(SessionEnv)
0x9ec  0n2540 ffffa389ae754080 svchost.exe(Dhcp)
0x9fc  0n2556 ffffa389ae757040 MemCompression
0xae8  0n2792 ffffa389b368b080 svchost.exe(SENS)
0xb24  0n2852 ffffa389b3708080 svchost.exe(AudioEndpointBuilder)
0xb40  0n2880 ffffa389b373e080 svchost.exe(FontCache)
0xb7c  0n2940 ffffa389b375f080 svchost.exe(WinHttpAutoProxySvc)
0xb9c  0n2972 ffffa389b378a080 svchost.exe(StateRepository)
0xbfc  0n3068 ffffa389b37ea080 svchost.exe(-p)
0xaf4  0n2804 ffffa389b37ed080 svchost.exe(TextInputManagementService)
0xc10  0n3088 ffffa389b385e080 svchost.exe(-p)
0xc20  0n3104 ffffa389b3860080 svchost.exe(wuauserv)
0xc60  0n3168 ffffa389b3905140 svchost.exe(-p)
0xc70  0n3184 ffffa389b39020c0 svchost.exe(-p)
0xcc4  0n3268 ffffa389b395c080 svchost.exe(ShellHWDetection)
0xd40  0n3392 ffffa389b3a380c0 spoolsv.exe
0xd9c  0n3484 ffffa389b3b9b0c0 svchost.exe
0xe78  0n3704 ffffa389b3b33080 svchost.exe(iphlpsvc)
0xe84  0n3716 ffffa389b3b20080 svchost.exe
0xe8c  0n3724 ffffa389b3b21080 svchost.exe(-p)
0xe94  0n3732 ffffa389b3b22080 AnyDesk.exe*32
0xea8  0n3752 ffffa389b3b88080 svchost.exe(-p)
0xeb8  0n3768 ffffa389b3c75080 svchost.exe(DPS)
0xed0  0n3792 ffffa389b3c86080 IpOverUsbSvc.exe*32
0xf1c  0n3868 ffffa389b3d32080 svchost.exe(Winmgmt)
0xf24  0n3876 ffffa389b3d22080 sqlwriter.exe
0xf30  0n3888 ffffa389b3d44080 Sysmon.exe
0xf3c  0n3900 ffffa389b3ca8080 RustDesk.exe
0xf44  0n3908 ffffa389b3aa9080 svchost.exe(LanmanServer)
0xf58  0n3928 ffffa389b3a98080 svchost.exe(TrkWks)
0xf7c  0n3964 ffffa389b3bee080 MsMpEng.exe
0xf84  0n3972 ffffa389b3b77080 wlms.exe
0xf9c  0n3996 ffffa389b3b66080 svchost.exe(WpnService)
0x1060 0n4192 ffffa389b3dcb080 sppsvc.exe
0x1174 0n4468 ffffa389b406e080 unsecapp.exe
0x12dc 0n4828 ffffa389b406c080 AggregatorHost.exe
0x13c8 0n5064 ffffa389b34a30c0 WmiPrvSE.exe
0x10b4 0n4276 ffffa389b4241100 svchost.exe(RmSvc)
0x1414 0n5140 ffffa389b42c90c0 svchost.exe(webthreatdefsvc)
0x1574 0n5492 ffffa389b13f2080 svchost.exe(StorSvc)
0x1620 0n5664 ffffa389b45e6080 sihost.exe
0x164c 0n5708 ffffa389b13ea080 svchost.exe(CDPUserSvc)
0x1694 0n5780 ffffa389b45b3080 svchost.exe(webthreatdefusersvc)
0x1700 0n5888 ffffa389b4705080 svchost.exe(WpnUserService)
0x175c 0n5980 ffffa389b47650c0 svchost.exe(TokenBroker)
0x17ac 0n6060 ffffa389b470a080 taskhostw.exe
0x1610 0n5648 ffffa389b476c080 svchost.exe(CDPSvc)
0x2fc  0n764  ffffa389b47a3080 explorer.exe
0x1300 0n4864 ffffa389b31b00c0 ctfmon.exe
0xcd8  0n3288 ffffa389b39ac080 svchost.exe(cbdhsvc)
0x1450 0n5200 ffffa389b3d33080 svchost.exe(camsvc)
0xbf4  0n3060 ffffa389b481e080 svchost.exe(Appinfo)
0x758  0n1880 ffffa389b48130c0 SearchIndexer.exe
0x230  0n560  ffffa389b49760c0 SearchHost.exe
0x14a0 0n5280 ffffa389b49740c0 StartMenuExperienceHost.exe
0x1608 0n5640 ffffa389b4cf70c0 RuntimeBroker.exe
0x1454 0n5204 ffffa389b48af080 RuntimeBroker.exe
0x8cc  0n2252 ffffa389b4cdf0c0 svchost.exe(UdkUserSvc)
0xaac  0n2732 ffffa389b4073080 svchost.exe(PcaSvc)
0xcd0  0n3280 ffffa389b4d27080 svchost.exe(UnistackSvcGroup)
0xd88  0n3464 ffffa389aee4c080 dllhost.exe
0xe2c  0n3628 ffffa389aee45080 SppExtComObj.Exe
0x13e8 0n5096 ffffa389aee44080 SgrmBroker.exe
0x4b0  0n1200 ffffa389b605b080 SecurityHealthSystray.exe
0xb68  0n2920 ffffa389b61da0c0 SecurityHealthService.exe
0x744  0n1860 ffffa389b61df0c0 svchost.exe
0xad8  0n2776 ffffa389b35b10c0 OneDrive.exe
0x1344 0n4932 ffffa389b4ced080 svchost.exe(hns)
0x1844 0n6212 ffffa389b1e1b080 svchost.exe(SSDPSRV)
0x1968 0n6504 ffffa389b35b60c0 msedge.exe
0x1a94 0n6804 ffffa389b4d180c0 svchost.exe(-p)
0x1b28 0n6952 ffffa389b4d86080 msedge.exe
0x1914 0n6420 ffffa389b2b0b080 AnyDesk.exe*32
0x4a0  0n1184 ffffa389b2d460c0 msedge.exe
0x4ec  0n1260 ffffa389b13070c0 msedge.exe
0x10ec 0n4332 ffffa389b13290c0 msedge.exe
0x64c  0n1612 ffffa389b276d0c0 uhssvc.exe
0xbe8  0n3048 ffffa389b2a850c0 msteamsupdate.exe
0x1670 0n5744 ffffa389b27680c0 msedge.exe
0x1018 0n4120 ffffa389b13020c0 svchost.exe(UsoSvc)
0xdf4  0n3572 ffffa389b6210080 ShellExperienceHost.exe
0xe28  0n3624 ffffa389b279f080 Widgets.exe
0x1bb8 0n7096 ffffa389b3004080 WidgetService.exe
0x1960 0n6496 ffffa389b2ccf100 NisSrv.exe
0xc2c  0n3116 ffffa389b24c7080 msedgewebview2.exe
0xbbc  0n3004 ffffa389b2cc80c0 svchost.exe(wscsvc)
0xb64  0n2916 ffffa389b24ea0c0 msedgewebview2.exe
0x1c98 0n7320 ffffa389b2a800c0 msedgewebview2.exe
0x1ca0 0n7328 ffffa389b25d40c0 msedgewebview2.exe
0x1d54 0n7508 ffffa389b3003080 msedgewebview2.exe
0x1e54 0n7764 ffffa389b24e30c0 msedgewebview2.exe
0x14f0 0n5360 ffffa389b29e40c0 svchost.exe(lfsvc)
0x1150 0n4432 ffffa389b1d110c0 MoUsoCoreWorker.exe
0x1854 0n6228 ffffa389b61a70c0 svchost.exe(NPSMSvc)
0x90c  0n2316 ffffa389b4fae080 RuntimeBroker.exe
0x1b34 0n6964 ffffa389b61c3080 SystemSettingsBroker.exe
0xfcc  0n4044 ffffa389b3d460c0 svchost.exe(WdiSystemHost)
0x9b4  0n2484 ffffa389af7950c0 svchost.exe(SstpSvc)
0x18f8 0n6392 ffffa389b4782080 svchost.exe(netsvcs)
0x17b8 0n6072 ffffa389b4c9b080 svchost.exe(DisplayEnhancementService)
0xa34  0n2612 ffffa389b2dee140 csrss.exe
0x14e0 0n5344 ffffa389afa540c0 winlogon.exe
0x7c4  0n1988 ffffa389af7b60c0 fontdrvhost.exe
0x2a0  0n672  ffffa389af9240c0 LogonUI.exe
0x7fc  0n2044 ffffa389b61af080 dwm.exe
0x1b64 0n7012 ffffa389b4fb1080 RustDesk.exe
0x21e8 0n8680 ffffa389b2beb080 WUDFHost.exe
0x2378 0n9080 ffffa389b1bdf080 rdpclip.exe
0x2178 0n8568 ffffa389b1963080 svchost.exe(ScDeviceEnum)
0x2424 0n9252 ffffa389af9570c0 rdpinput.exe
0x26c4 0n9924 ffffa389b6094080 TabTip.exe
0xd4c  0n3404 ffffa389b2ad9080 TextInputHost.exe
0x2654 0n9812 ffffa389af7c40c0 msedge.exe
0x1890 0n6288 ffffa389af9d10c0 cmd.exe
0x19bc 0n6588 ffffa389af9e80c0 conhost.exe
0x26b8 0n9912 ffffa389b2e4b140 svchost.exe(DsmSvc)
0x20f0 0n8432 ffffa389afab70c0 notmyfault64.exe
============= ================ =======================================
PID           Address          Name
```

We can filter on processes that contains a word, so instead of listening all the processes. We can return all the processes that contains **"svchost"** for example.

```
0: kd> !tl svchost
PID           Address          Name
============= ================ =======================================
0x22c  0n556  ffffa389b141e080 svchost.exe(-p)
0x418  0n1048 ffffa389b2f5a080 svchost.exe(-p)
0x444  0n1092 ffffa389b2f9b080 svchost.exe(LSM)
0x4e4  0n1252 ffffa389b31e9080 svchost.exe(TermService)
0x508  0n1288 ffffa389b3210080 svchost.exe
0x544  0n1348 ffffa389b323b0c0 svchost.exe(NcbService)
0x550  0n1360 ffffa389b3240080 svchost.exe(-p)
0x560  0n1376 ffffa389b323e080 svchost.exe(TimeBrokerSvc)
0x5cc  0n1484 ffffa389b32c1080 svchost.exe(Schedule)
0x608  0n1544 ffffa389b32dc080 svchost.exe(ProfSvc)
0x628  0n1576 ffffa389b3351080 svchost.exe(nsi)
0x664  0n1636 ffffa389b336b080 svchost.exe(DispBrokerDesktopSvc)
0x66c  0n1644 ffffa389b336e080 svchost.exe(netprofm)
0x674  0n1652 ffffa389b335b080 svchost.exe(UmRdpService)
0x6b8  0n1720 ffffa389b33a0080 svchost.exe(UserManager)
0x720  0n1824 ffffa389b33c7080 svchost.exe(CertPropSvc)
0x738  0n1848 ffffa389b33d0080 svchost.exe(vmicheartbeat)
0x75c  0n1884 ffffa389b33e3080 svchost.exe(vmicrdv)
0x768  0n1896 ffffa389b33db080 svchost.exe
0x778  0n1912 ffffa389b33eb080 svchost.exe
0x790  0n1936 ffffa389b341c0c0 svchost.exe(vmictimesync)
0x7c8  0n1992 ffffa389b343b080 svchost.exe(vmicvss)
0x488  0n1160 ffffa389b34a1080 svchost.exe
0x86c  0n2156 ffffa389ae719080 svchost.exe(-p)
0x924  0n2340 ffffa389ae796080 svchost.exe(EventLog)
0x954  0n2388 ffffa389ae76e080 svchost.exe(SysMain)
0x960  0n2400 ffffa389ae770080 svchost.exe(EventSystem)
0x96c  0n2412 ffffa389ae75d080 svchost.exe
0x99c  0n2460 ffffa389ae78c080 svchost.exe(SessionEnv)
0x9ec  0n2540 ffffa389ae754080 svchost.exe(Dhcp)
0xae8  0n2792 ffffa389b368b080 svchost.exe(SENS)
0xb24  0n2852 ffffa389b3708080 svchost.exe(AudioEndpointBuilder)
0xb40  0n2880 ffffa389b373e080 svchost.exe(FontCache)
0xb7c  0n2940 ffffa389b375f080 svchost.exe(WinHttpAutoProxySvc)
0xb9c  0n2972 ffffa389b378a080 svchost.exe(StateRepository)
0xbfc  0n3068 ffffa389b37ea080 svchost.exe(-p)
0xaf4  0n2804 ffffa389b37ed080 svchost.exe(TextInputManagementService)
0xc10  0n3088 ffffa389b385e080 svchost.exe(-p)
0xc20  0n3104 ffffa389b3860080 svchost.exe(wuauserv)
0xc60  0n3168 ffffa389b3905140 svchost.exe(-p)
0xc70  0n3184 ffffa389b39020c0 svchost.exe(-p)
0xcc4  0n3268 ffffa389b395c080 svchost.exe(ShellHWDetection)
0xd9c  0n3484 ffffa389b3b9b0c0 svchost.exe
0xe78  0n3704 ffffa389b3b33080 svchost.exe(iphlpsvc)
0xe84  0n3716 ffffa389b3b20080 svchost.exe
0xe8c  0n3724 ffffa389b3b21080 svchost.exe(-p)
0xea8  0n3752 ffffa389b3b88080 svchost.exe(-p)
0xeb8  0n3768 ffffa389b3c75080 svchost.exe(DPS)
0xf1c  0n3868 ffffa389b3d32080 svchost.exe(Winmgmt)
0xf44  0n3908 ffffa389b3aa9080 svchost.exe(LanmanServer)
0xf58  0n3928 ffffa389b3a98080 svchost.exe(TrkWks)
0xf9c  0n3996 ffffa389b3b66080 svchost.exe(WpnService)
0x10b4 0n4276 ffffa389b4241100 svchost.exe(RmSvc)
0x1414 0n5140 ffffa389b42c90c0 svchost.exe(webthreatdefsvc)
0x1574 0n5492 ffffa389b13f2080 svchost.exe(StorSvc)
0x164c 0n5708 ffffa389b13ea080 svchost.exe(CDPUserSvc)
0x1694 0n5780 ffffa389b45b3080 svchost.exe(webthreatdefusersvc)
0x1700 0n5888 ffffa389b4705080 svchost.exe(WpnUserService)
0x175c 0n5980 ffffa389b47650c0 svchost.exe(TokenBroker)
0x1610 0n5648 ffffa389b476c080 svchost.exe(CDPSvc)
0xcd8  0n3288 ffffa389b39ac080 svchost.exe(cbdhsvc)
0x1450 0n5200 ffffa389b3d33080 svchost.exe(camsvc)
0xbf4  0n3060 ffffa389b481e080 svchost.exe(Appinfo)
0x8cc  0n2252 ffffa389b4cdf0c0 svchost.exe(UdkUserSvc)
0xaac  0n2732 ffffa389b4073080 svchost.exe(PcaSvc)
0xcd0  0n3280 ffffa389b4d27080 svchost.exe(UnistackSvcGroup)
0x744  0n1860 ffffa389b61df0c0 svchost.exe
0x1344 0n4932 ffffa389b4ced080 svchost.exe(hns)
0x1844 0n6212 ffffa389b1e1b080 svchost.exe(SSDPSRV)
0x1a94 0n6804 ffffa389b4d180c0 svchost.exe(-p)
0x1018 0n4120 ffffa389b13020c0 svchost.exe(UsoSvc)
0xbbc  0n3004 ffffa389b2cc80c0 svchost.exe(wscsvc)
0x14f0 0n5360 ffffa389b29e40c0 svchost.exe(lfsvc)
0x1854 0n6228 ffffa389b61a70c0 svchost.exe(NPSMSvc)
0xfcc  0n4044 ffffa389b3d460c0 svchost.exe(WdiSystemHost)
0x9b4  0n2484 ffffa389af7950c0 svchost.exe(SstpSvc)
0x18f8 0n6392 ffffa389b4782080 svchost.exe(netsvcs)
0x17b8 0n6072 ffffa389b4c9b080 svchost.exe(DisplayEnhancementService)
0x2178 0n8568 ffffa389b1963080 svchost.exe(ScDeviceEnum)
0x26b8 0n9912 ffffa389b2e4b140 svchost.exe(DsmSvc)
============= ================ =======================================
PID           Address          Name
```

Here we are just looking for processes that contains the word **"event"**.

```
0: kd> !tl event
PID          Address          Name
============ ================ ========================
0x924 0n2340 ffffa389ae796080 svchost.exe(EventLog)
0x960 0n2400 ffffa389ae770080 svchost.exe(EventSystem)
============ ================ ========================
PID          Address          Name

Warning! Zombie process(es) detected (not displayed). Count: 10 [zombie report]
```

Let's now list all the running processes again, but also include the session that those processes are running in. Sessions are distinct environments in which user processes run, enabling system isolation for different users or tasks. Most system related processes are usually in session 0.

```
0: kd> !tl -s
PID           Address          Name                                    Ses
============= ================ ======================================= ===
0x0    0n0    fffff80152348f40 Idle                                      0
0x4    0n4    ffffa389ae6eb040 System                                    0
0x68   0n104  ffffa389ae6df080 Registry                                  0
0x1f8  0n504  ffffa389b1713080 smss.exe                                  0
0x28c  0n652  ffffa389b29960c0 csrss.exe                                 0
0x2d4  0n724  ffffa389b2def080 wininit.exe                               0
0x2dc  0n732  ffffa389b2df1140 csrss.exe                                 1
0x338  0n824  ffffa389b2eaf080 winlogon.exe                              1
0x368  0n872  ffffa389b2f08080 services.exe                              0
0x370  0n880  ffffa389b29d30c0 LsaIso.exe                                0
0x38c  0n908  ffffa389b2f09080 lsass.exe                                 0
0x22c  0n556  ffffa389b141e080 svchost.exe(-p)                           0
0x25c  0n604  ffffa389b302f080 fontdrvhost.exe                           1
0x218  0n536  ffffa389b3037080 fontdrvhost.exe                           0
0x418  0n1048 ffffa389b2f5a080 svchost.exe(-p)                           0
0x444  0n1092 ffffa389b2f9b080 svchost.exe(LSM)                          0
0x494  0n1172 ffffa389b3168080 dwm.exe                                   1
0x4e4  0n1252 ffffa389b31e9080 svchost.exe(TermService)                  0
0x508  0n1288 ffffa389b3210080 svchost.exe                               0
0x544  0n1348 ffffa389b323b0c0 svchost.exe(NcbService)                   0
0x550  0n1360 ffffa389b3240080 svchost.exe(-p)                           0
0x560  0n1376 ffffa389b323e080 svchost.exe(TimeBrokerSvc)                0
0x5cc  0n1484 ffffa389b32c1080 svchost.exe(Schedule)                     0
0x608  0n1544 ffffa389b32dc080 svchost.exe(ProfSvc)                      0
0x628  0n1576 ffffa389b3351080 svchost.exe(nsi)                          0
0x664  0n1636 ffffa389b336b080 svchost.exe(DispBrokerDesktopSvc)         0
0x66c  0n1644 ffffa389b336e080 svchost.exe(netprofm)                     0
0x674  0n1652 ffffa389b335b080 svchost.exe(UmRdpService)                 0
0x6b8  0n1720 ffffa389b33a0080 svchost.exe(UserManager)                  0
0x720  0n1824 ffffa389b33c7080 svchost.exe(CertPropSvc)                  0
0x738  0n1848 ffffa389b33d0080 svchost.exe(vmicheartbeat)                0
0x75c  0n1884 ffffa389b33e3080 svchost.exe(vmicrdv)                      0
0x768  0n1896 ffffa389b33db080 svchost.exe                               0
0x778  0n1912 ffffa389b33eb080 svchost.exe                               0
0x790  0n1936 ffffa389b341c0c0 svchost.exe(vmictimesync)                 0
0x7c8  0n1992 ffffa389b343b080 svchost.exe(vmicvss)                      0
0x488  0n1160 ffffa389b34a1080 svchost.exe                               0
0x86c  0n2156 ffffa389ae719080 svchost.exe(-p)                           0
0x8e4  0n2276 ffffa389b349c080 VSSVC.exe                                 0
0x924  0n2340 ffffa389ae796080 svchost.exe(EventLog)                     0
0x954  0n2388 ffffa389ae76e080 svchost.exe(SysMain)                      0
0x960  0n2400 ffffa389ae770080 svchost.exe(EventSystem)                  0
0x96c  0n2412 ffffa389ae75d080 svchost.exe                               0
0x99c  0n2460 ffffa389ae78c080 svchost.exe(SessionEnv)                   0
0x9ec  0n2540 ffffa389ae754080 svchost.exe(Dhcp)                         0
0x9fc  0n2556 ffffa389ae757040 MemCompression                            0
0xae8  0n2792 ffffa389b368b080 svchost.exe(SENS)                         0
0xb24  0n2852 ffffa389b3708080 svchost.exe(AudioEndpointBuilder)         0
0xb40  0n2880 ffffa389b373e080 svchost.exe(FontCache)                    0
0xb7c  0n2940 ffffa389b375f080 svchost.exe(WinHttpAutoProxySvc)          0
0xb9c  0n2972 ffffa389b378a080 svchost.exe(StateRepository)              0
0xbfc  0n3068 ffffa389b37ea080 svchost.exe(-p)                           0
0xaf4  0n2804 ffffa389b37ed080 svchost.exe(TextInputManagementService)   0
0xc10  0n3088 ffffa389b385e080 svchost.exe(-p)                           0
0xc20  0n3104 ffffa389b3860080 svchost.exe(wuauserv)                     0
0xc60  0n3168 ffffa389b3905140 svchost.exe(-p)                           0
0xc70  0n3184 ffffa389b39020c0 svchost.exe(-p)                           0
0xcc4  0n3268 ffffa389b395c080 svchost.exe(ShellHWDetection)             0
0xd40  0n3392 ffffa389b3a380c0 spoolsv.exe                               0
0xd9c  0n3484 ffffa389b3b9b0c0 svchost.exe                               0
0xe78  0n3704 ffffa389b3b33080 svchost.exe(iphlpsvc)                     0
0xe84  0n3716 ffffa389b3b20080 svchost.exe                               0
0xe8c  0n3724 ffffa389b3b21080 svchost.exe(-p)                           0
0xe94  0n3732 ffffa389b3b22080 AnyDesk.exe*32                            0
0xea8  0n3752 ffffa389b3b88080 svchost.exe(-p)                           0
0xeb8  0n3768 ffffa389b3c75080 svchost.exe(DPS)                          0
0xed0  0n3792 ffffa389b3c86080 IpOverUsbSvc.exe*32                       0
0xf1c  0n3868 ffffa389b3d32080 svchost.exe(Winmgmt)                      0
0xf24  0n3876 ffffa389b3d22080 sqlwriter.exe                             0
0xf30  0n3888 ffffa389b3d44080 Sysmon.exe                                0
0xf3c  0n3900 ffffa389b3ca8080 RustDesk.exe                              0
0xf44  0n3908 ffffa389b3aa9080 svchost.exe(LanmanServer)                 0
0xf58  0n3928 ffffa389b3a98080 svchost.exe(TrkWks)                       0
0xf7c  0n3964 ffffa389b3bee080 MsMpEng.exe                               0
0xf84  0n3972 ffffa389b3b77080 wlms.exe                                  0
0xf9c  0n3996 ffffa389b3b66080 svchost.exe(WpnService)                   0
0x1060 0n4192 ffffa389b3dcb080 sppsvc.exe                                0
0x1174 0n4468 ffffa389b406e080 unsecapp.exe                              0
0x12dc 0n4828 ffffa389b406c080 AggregatorHost.exe                        0
0x13c8 0n5064 ffffa389b34a30c0 WmiPrvSE.exe                              0
0x10b4 0n4276 ffffa389b4241100 svchost.exe(RmSvc)                        0
0x1414 0n5140 ffffa389b42c90c0 svchost.exe(webthreatdefsvc)              0
0x1574 0n5492 ffffa389b13f2080 svchost.exe(StorSvc)                      0
0x1620 0n5664 ffffa389b45e6080 sihost.exe                                1
0x164c 0n5708 ffffa389b13ea080 svchost.exe(CDPUserSvc)                   1
0x1694 0n5780 ffffa389b45b3080 svchost.exe(webthreatdefusersvc)          1
0x1700 0n5888 ffffa389b4705080 svchost.exe(WpnUserService)               1
0x175c 0n5980 ffffa389b47650c0 svchost.exe(TokenBroker)                  0
0x17ac 0n6060 ffffa389b470a080 taskhostw.exe                             1
0x1610 0n5648 ffffa389b476c080 svchost.exe(CDPSvc)                       0
0x2fc  0n764  ffffa389b47a3080 explorer.exe                              1
0x1300 0n4864 ffffa389b31b00c0 ctfmon.exe                                1
0xcd8  0n3288 ffffa389b39ac080 svchost.exe(cbdhsvc)                      1
0x1450 0n5200 ffffa389b3d33080 svchost.exe(camsvc)                       0
0xbf4  0n3060 ffffa389b481e080 svchost.exe(Appinfo)                      0
0x758  0n1880 ffffa389b48130c0 SearchIndexer.exe                         0
0x230  0n560  ffffa389b49760c0 SearchHost.exe                            1
0x14a0 0n5280 ffffa389b49740c0 StartMenuExperienceHost.exe               1
0x1608 0n5640 ffffa389b4cf70c0 RuntimeBroker.exe                         1
0x1454 0n5204 ffffa389b48af080 RuntimeBroker.exe                         1
0x8cc  0n2252 ffffa389b4cdf0c0 svchost.exe(UdkUserSvc)                   1
0xaac  0n2732 ffffa389b4073080 svchost.exe(PcaSvc)                       0
0xcd0  0n3280 ffffa389b4d27080 svchost.exe(UnistackSvcGroup)             1
0xd88  0n3464 ffffa389aee4c080 dllhost.exe                               1
0xe2c  0n3628 ffffa389aee45080 SppExtComObj.Exe                          0
0x13e8 0n5096 ffffa389aee44080 SgrmBroker.exe                            0
0x4b0  0n1200 ffffa389b605b080 SecurityHealthSystray.exe                 1
0xb68  0n2920 ffffa389b61da0c0 SecurityHealthService.exe                 0
0x744  0n1860 ffffa389b61df0c0 svchost.exe                               0
0xad8  0n2776 ffffa389b35b10c0 OneDrive.exe                              1
0x1344 0n4932 ffffa389b4ced080 svchost.exe(hns)                          0
0x1844 0n6212 ffffa389b1e1b080 svchost.exe(SSDPSRV)                      0
0x1968 0n6504 ffffa389b35b60c0 msedge.exe                                1
0x1a94 0n6804 ffffa389b4d180c0 svchost.exe(-p)                           0
0x1b28 0n6952 ffffa389b4d86080 msedge.exe                                1
0x1914 0n6420 ffffa389b2b0b080 AnyDesk.exe*32                            1
0x4a0  0n1184 ffffa389b2d460c0 msedge.exe                                1
0x4ec  0n1260 ffffa389b13070c0 msedge.exe                                1
0x10ec 0n4332 ffffa389b13290c0 msedge.exe                                1
0x64c  0n1612 ffffa389b276d0c0 uhssvc.exe                                0
0xbe8  0n3048 ffffa389b2a850c0 msteamsupdate.exe                         1
0x1670 0n5744 ffffa389b27680c0 msedge.exe                                1
0x1018 0n4120 ffffa389b13020c0 svchost.exe(UsoSvc)                       0
0xdf4  0n3572 ffffa389b6210080 ShellExperienceHost.exe                   1
0xe28  0n3624 ffffa389b279f080 Widgets.exe                               1
0x1bb8 0n7096 ffffa389b3004080 WidgetService.exe                         1
0x1960 0n6496 ffffa389b2ccf100 NisSrv.exe                                0
0xc2c  0n3116 ffffa389b24c7080 msedgewebview2.exe                        1
0xbbc  0n3004 ffffa389b2cc80c0 svchost.exe(wscsvc)                       0
0xb64  0n2916 ffffa389b24ea0c0 msedgewebview2.exe                        1
0x1c98 0n7320 ffffa389b2a800c0 msedgewebview2.exe                        1
0x1ca0 0n7328 ffffa389b25d40c0 msedgewebview2.exe                        1
0x1d54 0n7508 ffffa389b3003080 msedgewebview2.exe                        1
0x1e54 0n7764 ffffa389b24e30c0 msedgewebview2.exe                        1
0x14f0 0n5360 ffffa389b29e40c0 svchost.exe(lfsvc)                        0
0x1150 0n4432 ffffa389b1d110c0 MoUsoCoreWorker.exe                       0
0x1854 0n6228 ffffa389b61a70c0 svchost.exe(NPSMSvc)                      1
0x90c  0n2316 ffffa389b4fae080 RuntimeBroker.exe                         1
0x1b34 0n6964 ffffa389b61c3080 SystemSettingsBroker.exe                  1
0xfcc  0n4044 ffffa389b3d460c0 svchost.exe(WdiSystemHost)                0
0x9b4  0n2484 ffffa389af7950c0 svchost.exe(SstpSvc)                      0
0x18f8 0n6392 ffffa389b4782080 svchost.exe(netsvcs)                      0
0x17b8 0n6072 ffffa389b4c9b080 svchost.exe(DisplayEnhancementService)    0
0xa34  0n2612 ffffa389b2dee140 csrss.exe                                 3
0x14e0 0n5344 ffffa389afa540c0 winlogon.exe                              3
0x7c4  0n1988 ffffa389af7b60c0 fontdrvhost.exe                           3
0x2a0  0n672  ffffa389af9240c0 LogonUI.exe                               3
0x7fc  0n2044 ffffa389b61af080 dwm.exe                                   3
0x1b64 0n7012 ffffa389b4fb1080 RustDesk.exe                              3
0x21e8 0n8680 ffffa389b2beb080 WUDFHost.exe                              0
0x2378 0n9080 ffffa389b1bdf080 rdpclip.exe                               1
0x2178 0n8568 ffffa389b1963080 svchost.exe(ScDeviceEnum)                 0
0x2424 0n9252 ffffa389af9570c0 rdpinput.exe                              1
0x26c4 0n9924 ffffa389b6094080 TabTip.exe                                1
0xd4c  0n3404 ffffa389b2ad9080 TextInputHost.exe                         1
0x2654 0n9812 ffffa389af7c40c0 msedge.exe                                1
0x1890 0n6288 ffffa389af9d10c0 cmd.exe                                   1
0x19bc 0n6588 ffffa389af9e80c0 conhost.exe                               1
0x26b8 0n9912 ffffa389b2e4b140 svchost.exe(DsmSvc)                       0
0x20f0 0n8432 ffffa389afab70c0 notmyfault64.exe                          1
============= ================ ======================================= ===
PID           Address          Name                                    Ses
```

Let's filter only on processes that are running in session 3.

```
0: kd> !tl -s 3
PID           Address          Name
============= ================ ===================
0xe94  0n3732 ffffa389b3b22080 AnyDesk.exe*32
0xed0  0n3792 ffffa389b3c86080 IpOverUsbSvc.exe*32
0x1914 0n6420 ffffa389b2b0b080 AnyDesk.exe*32
============= ================ ===================
PID           Address          Name
```

We can also include all the running processes that is associated with a user. This helps us to identify under which user context, the process was running.

```
0: kd> !tl -u
PID           Address          Name                                    Ses User Name
============= ================ ======================================= === ==========================
0x0    0n0    fffff80152348f40 Idle                                      0 WORKGROUP\WINDEV2305EVAL$
0x4    0n4    ffffa389ae6eb040 System                                    0 WORKGROUP\WINDEV2305EVAL$
0x68   0n104  ffffa389ae6df080 Registry                                  0 WORKGROUP\WINDEV2305EVAL$
0x1f8  0n504  ffffa389b1713080 smss.exe                                  0 WORKGROUP\WINDEV2305EVAL$
0x28c  0n652  ffffa389b29960c0 csrss.exe                                 0 WORKGROUP\WINDEV2305EVAL$
0x2d4  0n724  ffffa389b2def080 wininit.exe                               0 WORKGROUP\WINDEV2305EVAL$
0x2dc  0n732  ffffa389b2df1140 csrss.exe                                 1 WORKGROUP\WINDEV2305EVAL$
0x338  0n824  ffffa389b2eaf080 winlogon.exe                              1 WORKGROUP\WINDEV2305EVAL$
0x368  0n872  ffffa389b2f08080 services.exe                              0 WORKGROUP\WINDEV2305EVAL$
0x370  0n880  ffffa389b29d30c0 LsaIso.exe                                0 WORKGROUP\WINDEV2305EVAL$
0x38c  0n908  ffffa389b2f09080 lsass.exe                                 0 WORKGROUP\WINDEV2305EVAL$
0x22c  0n556  ffffa389b141e080 svchost.exe(-p)                           0 WORKGROUP\WINDEV2305EVAL$
0x25c  0n604  ffffa389b302f080 fontdrvhost.exe                           1 Font Driver Host\UMFD-1
0x218  0n536  ffffa389b3037080 fontdrvhost.exe                           0 Font Driver Host\UMFD-0
0x418  0n1048 ffffa389b2f5a080 svchost.exe(-p)                           0 WORKGROUP\WINDEV2305EVAL$
0x444  0n1092 ffffa389b2f9b080 svchost.exe(LSM)                          0 WORKGROUP\WINDEV2305EVAL$
0x494  0n1172 ffffa389b3168080 dwm.exe                                   1 Window Manager\DWM-1
0x4e4  0n1252 ffffa389b31e9080 svchost.exe(TermService)                  0 WORKGROUP\WINDEV2305EVAL$
0x508  0n1288 ffffa389b3210080 svchost.exe                               0 NT AUTHORITY\LOCAL SERVICE
0x544  0n1348 ffffa389b323b0c0 svchost.exe(NcbService)                   0 WORKGROUP\WINDEV2305EVAL$
0x550  0n1360 ffffa389b3240080 svchost.exe(-p)                           0 NT AUTHORITY\LOCAL SERVICE
0x560  0n1376 ffffa389b323e080 svchost.exe(TimeBrokerSvc)                0 NT AUTHORITY\LOCAL SERVICE
0x5cc  0n1484 ffffa389b32c1080 svchost.exe(Schedule)                     0 WORKGROUP\WINDEV2305EVAL$
0x608  0n1544 ffffa389b32dc080 svchost.exe(ProfSvc)                      0 WORKGROUP\WINDEV2305EVAL$
0x628  0n1576 ffffa389b3351080 svchost.exe(nsi)                          0 NT AUTHORITY\LOCAL SERVICE
0x664  0n1636 ffffa389b336b080 svchost.exe(DispBrokerDesktopSvc)         0 NT AUTHORITY\LOCAL SERVICE
0x66c  0n1644 ffffa389b336e080 svchost.exe(netprofm)                     0 WORKGROUP\WINDEV2305EVAL$
0x674  0n1652 ffffa389b335b080 svchost.exe(UmRdpService)                 0 WORKGROUP\WINDEV2305EVAL$
0x6b8  0n1720 ffffa389b33a0080 svchost.exe(UserManager)                  0 WORKGROUP\WINDEV2305EVAL$
0x720  0n1824 ffffa389b33c7080 svchost.exe(CertPropSvc)                  0 WORKGROUP\WINDEV2305EVAL$
0x738  0n1848 ffffa389b33d0080 svchost.exe(vmicheartbeat)                0 WORKGROUP\WINDEV2305EVAL$
0x75c  0n1884 ffffa389b33e3080 svchost.exe(vmicrdv)                      0 WORKGROUP\WINDEV2305EVAL$
0x768  0n1896 ffffa389b33db080 svchost.exe                               0 WORKGROUP\WINDEV2305EVAL$
0x778  0n1912 ffffa389b33eb080 svchost.exe                               0 WORKGROUP\WINDEV2305EVAL$
0x790  0n1936 ffffa389b341c0c0 svchost.exe(vmictimesync)                 0 NT AUTHORITY\LOCAL SERVICE
0x7c8  0n1992 ffffa389b343b080 svchost.exe(vmicvss)                      0 WORKGROUP\WINDEV2305EVAL$
0x488  0n1160 ffffa389b34a1080 svchost.exe                               0 WORKGROUP\WINDEV2305EVAL$
0x86c  0n2156 ffffa389ae719080 svchost.exe(-p)                           0 WORKGROUP\WINDEV2305EVAL$
0x8e4  0n2276 ffffa389b349c080 VSSVC.exe                                 0 WORKGROUP\WINDEV2305EVAL$
0x924  0n2340 ffffa389ae796080 svchost.exe(EventLog)                     0 NT AUTHORITY\LOCAL SERVICE
0x954  0n2388 ffffa389ae76e080 svchost.exe(SysMain)                      0 WORKGROUP\WINDEV2305EVAL$
0x960  0n2400 ffffa389ae770080 svchost.exe(EventSystem)                  0 NT AUTHORITY\LOCAL SERVICE
0x96c  0n2412 ffffa389ae75d080 svchost.exe                               0 WORKGROUP\WINDEV2305EVAL$
0x99c  0n2460 ffffa389ae78c080 svchost.exe(SessionEnv)                   0 WORKGROUP\WINDEV2305EVAL$
0x9ec  0n2540 ffffa389ae754080 svchost.exe(Dhcp)                         0 NT AUTHORITY\LOCAL SERVICE
0x9fc  0n2556 ffffa389ae757040 MemCompression                            0 WORKGROUP\WINDEV2305EVAL$
0xae8  0n2792 ffffa389b368b080 svchost.exe(SENS)                         0 WORKGROUP\WINDEV2305EVAL$
0xb24  0n2852 ffffa389b3708080 svchost.exe(AudioEndpointBuilder)         0 WORKGROUP\WINDEV2305EVAL$
0xb40  0n2880 ffffa389b373e080 svchost.exe(FontCache)                    0 NT AUTHORITY\LOCAL SERVICE
0xb7c  0n2940 ffffa389b375f080 svchost.exe(WinHttpAutoProxySvc)          0 NT AUTHORITY\LOCAL SERVICE
0xb9c  0n2972 ffffa389b378a080 svchost.exe(StateRepository)              0 WORKGROUP\WINDEV2305EVAL$
0xbfc  0n3068 ffffa389b37ea080 svchost.exe(-p)                           0 NT AUTHORITY\LOCAL SERVICE
0xaf4  0n2804 ffffa389b37ed080 svchost.exe(TextInputManagementService)   0 WORKGROUP\WINDEV2305EVAL$
0xc10  0n3088 ffffa389b385e080 svchost.exe(-p)                           0 NT AUTHORITY\LOCAL SERVICE
0xc20  0n3104 ffffa389b3860080 svchost.exe(wuauserv)                     0 WORKGROUP\WINDEV2305EVAL$
0xc60  0n3168 ffffa389b3905140 svchost.exe(-p)                           0 NT AUTHORITY\LOCAL SERVICE
0xc70  0n3184 ffffa389b39020c0 svchost.exe(-p)                           0 NT AUTHORITY\LOCAL SERVICE
0xcc4  0n3268 ffffa389b395c080 svchost.exe(ShellHWDetection)             0 WORKGROUP\WINDEV2305EVAL$
0xd40  0n3392 ffffa389b3a380c0 spoolsv.exe                               0 WORKGROUP\WINDEV2305EVAL$
0xd9c  0n3484 ffffa389b3b9b0c0 svchost.exe                               0 WORKGROUP\WINDEV2305EVAL$
0xe78  0n3704 ffffa389b3b33080 svchost.exe(iphlpsvc)                     0 WORKGROUP\WINDEV2305EVAL$
0xe84  0n3716 ffffa389b3b20080 svchost.exe                               0 WORKGROUP\WINDEV2305EVAL$
0xe8c  0n3724 ffffa389b3b21080 svchost.exe(-p)                           0 WORKGROUP\WINDEV2305EVAL$
0xe94  0n3732 ffffa389b3b22080 AnyDesk.exe*32                            0 WORKGROUP\WINDEV2305EVAL$
0xea8  0n3752 ffffa389b3b88080 svchost.exe(-p)                           0 WORKGROUP\WINDEV2305EVAL$
0xeb8  0n3768 ffffa389b3c75080 svchost.exe(DPS)                          0 NT AUTHORITY\LOCAL SERVICE
0xed0  0n3792 ffffa389b3c86080 IpOverUsbSvc.exe*32                       0 WORKGROUP\WINDEV2305EVAL$
0xf1c  0n3868 ffffa389b3d32080 svchost.exe(Winmgmt)                      0 WORKGROUP\WINDEV2305EVAL$
0xf24  0n3876 ffffa389b3d22080 sqlwriter.exe                             0 WORKGROUP\WINDEV2305EVAL$
0xf30  0n3888 ffffa389b3d44080 Sysmon.exe                                0 WORKGROUP\WINDEV2305EVAL$
0xf3c  0n3900 ffffa389b3ca8080 RustDesk.exe                              0 WORKGROUP\WINDEV2305EVAL$
0xf44  0n3908 ffffa389b3aa9080 svchost.exe(LanmanServer)                 0 WORKGROUP\WINDEV2305EVAL$
0xf58  0n3928 ffffa389b3a98080 svchost.exe(TrkWks)                       0 WORKGROUP\WINDEV2305EVAL$
0xf7c  0n3964 ffffa389b3bee080 MsMpEng.exe                               0 WORKGROUP\WINDEV2305EVAL$
0xf84  0n3972 ffffa389b3b77080 wlms.exe                                  0 WORKGROUP\WINDEV2305EVAL$
0xf9c  0n3996 ffffa389b3b66080 svchost.exe(WpnService)                   0 WORKGROUP\WINDEV2305EVAL$
0x1060 0n4192 ffffa389b3dcb080 sppsvc.exe                                0 WORKGROUP\WINDEV2305EVAL$
0x1174 0n4468 ffffa389b406e080 unsecapp.exe                              0 WORKGROUP\WINDEV2305EVAL$
0x12dc 0n4828 ffffa389b406c080 AggregatorHost.exe                        0 WORKGROUP\WINDEV2305EVAL$
0x13c8 0n5064 ffffa389b34a30c0 WmiPrvSE.exe                              0 WORKGROUP\WINDEV2305EVAL$
0x10b4 0n4276 ffffa389b4241100 svchost.exe(RmSvc)                        0 NT AUTHORITY\LOCAL SERVICE
0x1414 0n5140 ffffa389b42c90c0 svchost.exe(webthreatdefsvc)              0 NT AUTHORITY\LOCAL SERVICE
0x1574 0n5492 ffffa389b13f2080 svchost.exe(StorSvc)                      0 WORKGROUP\WINDEV2305EVAL$
0x1620 0n5664 ffffa389b45e6080 sihost.exe                                1 WINDEV2305EVAL\User
0x164c 0n5708 ffffa389b13ea080 svchost.exe(CDPUserSvc)                   1 WINDEV2305EVAL\User
0x1694 0n5780 ffffa389b45b3080 svchost.exe(webthreatdefusersvc)          1 WINDEV2305EVAL\User
0x1700 0n5888 ffffa389b4705080 svchost.exe(WpnUserService)               1 WINDEV2305EVAL\User
0x175c 0n5980 ffffa389b47650c0 svchost.exe(TokenBroker)                  0 WORKGROUP\WINDEV2305EVAL$
0x17ac 0n6060 ffffa389b470a080 taskhostw.exe                             1 WINDEV2305EVAL\User
0x1610 0n5648 ffffa389b476c080 svchost.exe(CDPSvc)                       0 NT AUTHORITY\LOCAL SERVICE
0x2fc  0n764  ffffa389b47a3080 explorer.exe                              1 WINDEV2305EVAL\User
0x1300 0n4864 ffffa389b31b00c0 ctfmon.exe                                1 WINDEV2305EVAL\User
0xcd8  0n3288 ffffa389b39ac080 svchost.exe(cbdhsvc)                      1 WINDEV2305EVAL\User
0x1450 0n5200 ffffa389b3d33080 svchost.exe(camsvc)                       0 WORKGROUP\WINDEV2305EVAL$
0xbf4  0n3060 ffffa389b481e080 svchost.exe(Appinfo)                      0 WORKGROUP\WINDEV2305EVAL$
0x758  0n1880 ffffa389b48130c0 SearchIndexer.exe                         0 WORKGROUP\WINDEV2305EVAL$
0x230  0n560  ffffa389b49760c0 SearchHost.exe                            1 WINDEV2305EVAL\User
0x14a0 0n5280 ffffa389b49740c0 StartMenuExperienceHost.exe               1 WINDEV2305EVAL\User
0x1608 0n5640 ffffa389b4cf70c0 RuntimeBroker.exe                         1 WINDEV2305EVAL\User
0x1454 0n5204 ffffa389b48af080 RuntimeBroker.exe                         1 WINDEV2305EVAL\User
0x8cc  0n2252 ffffa389b4cdf0c0 svchost.exe(UdkUserSvc)                   1 WINDEV2305EVAL\User
0xaac  0n2732 ffffa389b4073080 svchost.exe(PcaSvc)                       0 WORKGROUP\WINDEV2305EVAL$
0xcd0  0n3280 ffffa389b4d27080 svchost.exe(UnistackSvcGroup)             1 WINDEV2305EVAL\User
0xd88  0n3464 ffffa389aee4c080 dllhost.exe                               1 WINDEV2305EVAL\User
0xe2c  0n3628 ffffa389aee45080 SppExtComObj.Exe                          0 WORKGROUP\WINDEV2305EVAL$
0x13e8 0n5096 ffffa389aee44080 SgrmBroker.exe                            0 WORKGROUP\WINDEV2305EVAL$
0x4b0  0n1200 ffffa389b605b080 SecurityHealthSystray.exe                 1 WINDEV2305EVAL\User
0xb68  0n2920 ffffa389b61da0c0 SecurityHealthService.exe                 0 WORKGROUP\WINDEV2305EVAL$
0x744  0n1860 ffffa389b61df0c0 svchost.exe                               0 WORKGROUP\WINDEV2305EVAL$
0xad8  0n2776 ffffa389b35b10c0 OneDrive.exe                              1 WINDEV2305EVAL\User
0x1344 0n4932 ffffa389b4ced080 svchost.exe(hns)                          0 WORKGROUP\WINDEV2305EVAL$
0x1844 0n6212 ffffa389b1e1b080 svchost.exe(SSDPSRV)                      0 NT AUTHORITY\LOCAL SERVICE
0x1968 0n6504 ffffa389b35b60c0 msedge.exe                                1 WINDEV2305EVAL\User
0x1a94 0n6804 ffffa389b4d180c0 svchost.exe(-p)                           0 WORKGROUP\WINDEV2305EVAL$
0x1b28 0n6952 ffffa389b4d86080 msedge.exe                                1 WINDEV2305EVAL\User
0x1914 0n6420 ffffa389b2b0b080 AnyDesk.exe*32                            1 WINDEV2305EVAL\User
0x4a0  0n1184 ffffa389b2d460c0 msedge.exe                                1 WINDEV2305EVAL\User
0x4ec  0n1260 ffffa389b13070c0 msedge.exe                                1 WINDEV2305EVAL\User
0x10ec 0n4332 ffffa389b13290c0 msedge.exe                                1 WINDEV2305EVAL\User
0x64c  0n1612 ffffa389b276d0c0 uhssvc.exe                                0 WORKGROUP\WINDEV2305EVAL$
0xbe8  0n3048 ffffa389b2a850c0 msteamsupdate.exe                         1 WINDEV2305EVAL\User
0x1670 0n5744 ffffa389b27680c0 msedge.exe                                1 WINDEV2305EVAL\User
0x1018 0n4120 ffffa389b13020c0 svchost.exe(UsoSvc)                       0 WORKGROUP\WINDEV2305EVAL$
0xdf4  0n3572 ffffa389b6210080 ShellExperienceHost.exe                   1 WINDEV2305EVAL\User
0xe28  0n3624 ffffa389b279f080 Widgets.exe                               1 WINDEV2305EVAL\User
0x1bb8 0n7096 ffffa389b3004080 WidgetService.exe                         1 WINDEV2305EVAL\User
0x1960 0n6496 ffffa389b2ccf100 NisSrv.exe                                0 NT AUTHORITY\LOCAL SERVICE
0xc2c  0n3116 ffffa389b24c7080 msedgewebview2.exe                        1 WINDEV2305EVAL\User
0xbbc  0n3004 ffffa389b2cc80c0 svchost.exe(wscsvc)                       0 NT AUTHORITY\LOCAL SERVICE
0xb64  0n2916 ffffa389b24ea0c0 msedgewebview2.exe                        1 WINDEV2305EVAL\User
0x1c98 0n7320 ffffa389b2a800c0 msedgewebview2.exe                        1 WINDEV2305EVAL\User
0x1ca0 0n7328 ffffa389b25d40c0 msedgewebview2.exe                        1 WINDEV2305EVAL\User
0x1d54 0n7508 ffffa389b3003080 msedgewebview2.exe                        1 WINDEV2305EVAL\User
0x1e54 0n7764 ffffa389b24e30c0 msedgewebview2.exe                        1 WINDEV2305EVAL\User
0x14f0 0n5360 ffffa389b29e40c0 svchost.exe(lfsvc)                        0 WORKGROUP\WINDEV2305EVAL$
0x1150 0n4432 ffffa389b1d110c0 MoUsoCoreWorker.exe                       0 WORKGROUP\WINDEV2305EVAL$
0x1854 0n6228 ffffa389b61a70c0 svchost.exe(NPSMSvc)                      1 WINDEV2305EVAL\User
0x90c  0n2316 ffffa389b4fae080 RuntimeBroker.exe                         1 WINDEV2305EVAL\User
0x1b34 0n6964 ffffa389b61c3080 SystemSettingsBroker.exe                  1 WINDEV2305EVAL\User
0xfcc  0n4044 ffffa389b3d460c0 svchost.exe(WdiSystemHost)                0 WORKGROUP\WINDEV2305EVAL$
0x9b4  0n2484 ffffa389af7950c0 svchost.exe(SstpSvc)                      0 NT AUTHORITY\LOCAL SERVICE
0x18f8 0n6392 ffffa389b4782080 svchost.exe(netsvcs)                      0 WORKGROUP\WINDEV2305EVAL$
0x17b8 0n6072 ffffa389b4c9b080 svchost.exe(DisplayEnhancementService)    0 WORKGROUP\WINDEV2305EVAL$
0xa34  0n2612 ffffa389b2dee140 csrss.exe                                 3 WORKGROUP\WINDEV2305EVAL$
0x14e0 0n5344 ffffa389afa540c0 winlogon.exe                              3 WORKGROUP\WINDEV2305EVAL$
0x7c4  0n1988 ffffa389af7b60c0 fontdrvhost.exe                           3 Font Driver Host\UMFD-3
0x2a0  0n672  ffffa389af9240c0 LogonUI.exe                               3 WORKGROUP\WINDEV2305EVAL$
0x7fc  0n2044 ffffa389b61af080 dwm.exe                                   3 Window Manager\DWM-3
0x1b64 0n7012 ffffa389b4fb1080 RustDesk.exe                              3 WORKGROUP\WINDEV2305EVAL$
0x21e8 0n8680 ffffa389b2beb080 WUDFHost.exe                              0 NT AUTHORITY\LOCAL SERVICE
0x2378 0n9080 ffffa389b1bdf080 rdpclip.exe                               1 WINDEV2305EVAL\User
0x2178 0n8568 ffffa389b1963080 svchost.exe(ScDeviceEnum)                 0 WORKGROUP\WINDEV2305EVAL$
0x2424 0n9252 ffffa389af9570c0 rdpinput.exe                              1 WINDEV2305EVAL\User
0x26c4 0n9924 ffffa389b6094080 TabTip.exe                                1 WINDEV2305EVAL\User
0xd4c  0n3404 ffffa389b2ad9080 TextInputHost.exe                         1 WINDEV2305EVAL\User
0x2654 0n9812 ffffa389af7c40c0 msedge.exe                                1 WINDEV2305EVAL\User
0x1890 0n6288 ffffa389af9d10c0 cmd.exe                                   1 WINDEV2305EVAL\User
0x19bc 0n6588 ffffa389af9e80c0 conhost.exe                               1 WINDEV2305EVAL\User
0x26b8 0n9912 ffffa389b2e4b140 svchost.exe(DsmSvc)                       0 WORKGROUP\WINDEV2305EVAL$
0x20f0 0n8432 ffffa389afab70c0 notmyfault64.exe                          1 WINDEV2305EVAL\User
============= ================ ======================================= === ==========================
PID           Address          Name                                    Ses User Name
```

If we're interested in a particular process and want to determine if a specific module has been loaded within it, we can use the **`!tl`** command with the **`-m`** option followed by the desired string. For example, we are searching if a module that contains "crypto" was loaded in lsass.exe

```
0: kd> !tl lsass -m bcrypt
PID         Address          Name      Modules
=========== ================ ========= =======================
0x38c 0n908 ffffa389b2f09080 lsass.exe bcrypt,bcryptprimitives
=========== ================ ========= =======================
PID         Address          Name      Modules
```

We can also search for all processes that have a module with "bcrypt" in its name among their loaded modules.

```
0: kd> !tl -m bcrypt
PID           Address          Name                                    Modules
============= ================ ======================================= =========================================
0x28c  0n652  ffffa389b29960c0 csrss.exe                               bcryptprimitives
0x2d4  0n724  ffffa389b2def080 wininit.exe                             bcrypt,bcryptprimitives
0x2dc  0n732  ffffa389b2df1140 csrss.exe                               bcrypt,bcryptprimitives
0x338  0n824  ffffa389b2eaf080 winlogon.exe                            bcrypt,bcryptprimitives
0x368  0n872  ffffa389b2f08080 services.exe                            bcryptprimitives
0x38c  0n908  ffffa389b2f09080 lsass.exe                               bcrypt,bcryptprimitives
0x22c  0n556  ffffa389b141e080 svchost.exe(-p)                         bcrypt,bcryptprimitives
0x418  0n1048 ffffa389b2f5a080 svchost.exe(-p)                         bcrypt,bcryptprimitives
0x444  0n1092 ffffa389b2f9b080 svchost.exe(LSM)                        bcryptprimitives
0x494  0n1172 ffffa389b3168080 dwm.exe                                 bcrypt,bcryptprimitives
0x4e4  0n1252 ffffa389b31e9080 svchost.exe(TermService)                bcrypt,bcryptprimitives
0x508  0n1288 ffffa389b3210080 svchost.exe                             bcryptprimitives
0x544  0n1348 ffffa389b323b0c0 svchost.exe(NcbService)                 bcrypt,bcryptprimitives
0x550  0n1360 ffffa389b3240080 svchost.exe(-p)                         bcryptprimitives
0x560  0n1376 ffffa389b323e080 svchost.exe(TimeBrokerSvc)              bcryptprimitives
0x5cc  0n1484 ffffa389b32c1080 svchost.exe(Schedule)                   bcrypt,bcryptprimitives
0x608  0n1544 ffffa389b32dc080 svchost.exe(ProfSvc)                    bcrypt,bcryptprimitives
0x628  0n1576 ffffa389b3351080 svchost.exe(nsi)                        bcryptprimitives
0x664  0n1636 ffffa389b336b080 svchost.exe(DispBrokerDesktopSvc)       bcryptprimitives
0x66c  0n1644 ffffa389b336e080 svchost.exe(netprofm)                   bcrypt,bcryptprimitives
0x674  0n1652 ffffa389b335b080 svchost.exe(UmRdpService)               bcryptprimitives
0x6b8  0n1720 ffffa389b33a0080 svchost.exe(UserManager)                bcryptprimitives
0x720  0n1824 ffffa389b33c7080 svchost.exe(CertPropSvc)                bcrypt,bcryptprimitives
0x738  0n1848 ffffa389b33d0080 svchost.exe(vmicheartbeat)              bcryptprimitives
0x75c  0n1884 ffffa389b33e3080 svchost.exe(vmicrdv)                    bcryptprimitives
0x768  0n1896 ffffa389b33db080 svchost.exe                             bcryptprimitives
0x778  0n1912 ffffa389b33eb080 svchost.exe                             bcryptprimitives
0x790  0n1936 ffffa389b341c0c0 svchost.exe(vmictimesync)               bcryptprimitives
0x7c8  0n1992 ffffa389b343b080 svchost.exe(vmicvss)                    bcryptprimitives
0x488  0n1160 ffffa389b34a1080 svchost.exe                             bcryptprimitives
0x86c  0n2156 ffffa389ae719080 svchost.exe(-p)                         bcryptprimitives
0x8e4  0n2276 ffffa389b349c080 VSSVC.exe                               bcryptprimitives
0x924  0n2340 ffffa389ae796080 svchost.exe(EventLog)                   bcrypt,bcryptprimitives
0x954  0n2388 ffffa389ae76e080 svchost.exe(SysMain)                    bcryptprimitives
0x960  0n2400 ffffa389ae770080 svchost.exe(EventSystem)                bcryptprimitives
0x96c  0n2412 ffffa389ae75d080 svchost.exe                             bcryptprimitives
0x99c  0n2460 ffffa389ae78c080 svchost.exe(SessionEnv)                 bcrypt,bcryptprimitives
0x9ec  0n2540 ffffa389ae754080 svchost.exe(Dhcp)                       bcryptprimitives
0xae8  0n2792 ffffa389b368b080 svchost.exe(SENS)                       bcryptprimitives
0xb24  0n2852 ffffa389b3708080 svchost.exe(AudioEndpointBuilder)       bcryptprimitives
0xb40  0n2880 ffffa389b373e080 svchost.exe(FontCache)                  bcryptprimitives
0xb7c  0n2940 ffffa389b375f080 svchost.exe(WinHttpAutoProxySvc)        bcryptprimitives
0xb9c  0n2972 ffffa389b378a080 svchost.exe(StateRepository)            bcrypt,bcryptprimitives
0xbfc  0n3068 ffffa389b37ea080 svchost.exe(-p)                         bcryptprimitives
0xaf4  0n2804 ffffa389b37ed080 svchost.exe(TextInputManagementService) bcryptprimitives
0xc10  0n3088 ffffa389b385e080 svchost.exe(-p)                         bcrypt,bcryptprimitives
0xc20  0n3104 ffffa389b3860080 svchost.exe(wuauserv)                   bcrypt,bcryptprimitives
0xc60  0n3168 ffffa389b3905140 svchost.exe(-p)                         bcryptprimitives
0xc70  0n3184 ffffa389b39020c0 svchost.exe(-p)                         bcrypt,bcryptprimitives
0xcc4  0n3268 ffffa389b395c080 svchost.exe(ShellHWDetection)           bcrypt,bcryptprimitives
0xd40  0n3392 ffffa389b3a380c0 spoolsv.exe                             bcrypt,bcryptprimitives
0xd9c  0n3484 ffffa389b3b9b0c0 svchost.exe                             bcryptprimitives
0xe78  0n3704 ffffa389b3b33080 svchost.exe(iphlpsvc)                   bcrypt,bcryptprimitives
0xe84  0n3716 ffffa389b3b20080 svchost.exe                             bcryptprimitives
0xe8c  0n3724 ffffa389b3b21080 svchost.exe(-p)                         bcrypt,bcryptprimitives
0xe94  0n3732 ffffa389b3b22080 AnyDesk.exe*32                          bcryptprimitives
0xea8  0n3752 ffffa389b3b88080 svchost.exe(-p)                         bcrypt,bcryptprimitives
0xeb8  0n3768 ffffa389b3c75080 svchost.exe(DPS)                        bcryptprimitives
0xed0  0n3792 ffffa389b3c86080 IpOverUsbSvc.exe*32                     bcrypt
0xf1c  0n3868 ffffa389b3d32080 svchost.exe(Winmgmt)                    bcryptprimitives
0xf24  0n3876 ffffa389b3d22080 sqlwriter.exe                           bcryptprimitives
0xf30  0n3888 ffffa389b3d44080 Sysmon.exe                              bcrypt,bcryptprimitives
0xf3c  0n3900 ffffa389b3ca8080 RustDesk.exe                            bcrypt,bcryptprimitives
0xf44  0n3908 ffffa389b3aa9080 svchost.exe(LanmanServer)               bcryptprimitives
0xf58  0n3928 ffffa389b3a98080 svchost.exe(TrkWks)                     bcryptprimitives
0xf7c  0n3964 ffffa389b3bee080 MsMpEng.exe                             bcrypt,bcryptprimitives
0xf9c  0n3996 ffffa389b3b66080 svchost.exe(WpnService)                 bcrypt,bcryptprimitives
0x1060 0n4192 ffffa389b3dcb080 sppsvc.exe                              bcrypt,bcryptprimitives
0x1174 0n4468 ffffa389b406e080 unsecapp.exe                            bcryptprimitives
0x12dc 0n4828 ffffa389b406c080 AggregatorHost.exe                      bcryptprimitives
0x13c8 0n5064 ffffa389b34a30c0 WmiPrvSE.exe                            bcrypt,bcryptprimitives
0x10b4 0n4276 ffffa389b4241100 svchost.exe(RmSvc)                      bcryptprimitives
0x1414 0n5140 ffffa389b42c90c0 svchost.exe(webthreatdefsvc)            bcrypt,bcryptprimitives
0x1574 0n5492 ffffa389b13f2080 svchost.exe(StorSvc)                    bcryptprimitives
0x1620 0n5664 ffffa389b45e6080 sihost.exe                              bcryptprimitives
0x164c 0n5708 ffffa389b13ea080 svchost.exe(CDPUserSvc)                 bcrypt,bcryptprimitives
0x1694 0n5780 ffffa389b45b3080 svchost.exe(webthreatdefusersvc)        bcryptprimitives
0x1700 0n5888 ffffa389b4705080 svchost.exe(WpnUserService)             bcryptprimitives
0x175c 0n5980 ffffa389b47650c0 svchost.exe(TokenBroker)                bcrypt,bcryptprimitives
0x17ac 0n6060 ffffa389b470a080 taskhostw.exe                           bcryptprimitives
0x1610 0n5648 ffffa389b476c080 svchost.exe(CDPSvc)                     bcrypt,bcryptprimitives
0x2fc  0n764  ffffa389b47a3080 explorer.exe                            bcrypt,bcryptprimitives
0x1300 0n4864 ffffa389b31b00c0 ctfmon.exe                              bcryptprimitives
0xcd8  0n3288 ffffa389b39ac080 svchost.exe(cbdhsvc)                    bcrypt,bcryptprimitives
0x1450 0n5200 ffffa389b3d33080 svchost.exe(camsvc)                     bcrypt,bcryptprimitives
0xbf4  0n3060 ffffa389b481e080 svchost.exe(Appinfo)                    bcrypt,bcryptprimitives
0x758  0n1880 ffffa389b48130c0 SearchIndexer.exe                       bcryptprimitives
0x230  0n560  ffffa389b49760c0 SearchHost.exe                          bcrypt,bcryptprimitives
0x14a0 0n5280 ffffa389b49740c0 StartMenuExperienceHost.exe             bcrypt,bcryptprimitives
0x1608 0n5640 ffffa389b4cf70c0 RuntimeBroker.exe                       bcrypt,bcryptprimitives
0x1454 0n5204 ffffa389b48af080 RuntimeBroker.exe                       bcrypt,bcryptprimitives
0x8cc  0n2252 ffffa389b4cdf0c0 svchost.exe(UdkUserSvc)                 bcrypt,bcryptprimitives
0xaac  0n2732 ffffa389b4073080 svchost.exe(PcaSvc)                     bcrypt,bcryptprimitives
0xcd0  0n3280 ffffa389b4d27080 svchost.exe(UnistackSvcGroup)           bcrypt,bcryptprimitives
0xd88  0n3464 ffffa389aee4c080 dllhost.exe                             bcryptprimitives
0xe2c  0n3628 ffffa389aee45080 SppExtComObj.Exe                        bcryptprimitives
0x13e8 0n5096 ffffa389aee44080 SgrmBroker.exe                          bcrypt,bcryptprimitives
0x4b0  0n1200 ffffa389b605b080 SecurityHealthSystray.exe               bcryptprimitives
0xb68  0n2920 ffffa389b61da0c0 SecurityHealthService.exe               bcrypt,bcryptprimitives
0x744  0n1860 ffffa389b61df0c0 svchost.exe                             bcryptprimitives
0xad8  0n2776 ffffa389b35b10c0 OneDrive.exe                            libcrypto_1_1_x64,bcrypt,bcryptprimitives
0x1344 0n4932 ffffa389b4ced080 svchost.exe(hns)                        bcryptprimitives
0x1844 0n6212 ffffa389b1e1b080 svchost.exe(SSDPSRV)                    bcrypt,bcryptprimitives
0x1968 0n6504 ffffa389b35b60c0 msedge.exe                              bcrypt,bcryptprimitives
0x1a94 0n6804 ffffa389b4d180c0 svchost.exe(-p)                         bcrypt,bcryptprimitives
0x1b28 0n6952 ffffa389b4d86080 msedge.exe                              bcryptprimitives
0x1914 0n6420 ffffa389b2b0b080 AnyDesk.exe*32                          bcryptprimitives
0x4a0  0n1184 ffffa389b2d460c0 msedge.exe                              bcryptprimitives
0x4ec  0n1260 ffffa389b13070c0 msedge.exe                              bcrypt,bcryptprimitives
0x10ec 0n4332 ffffa389b13290c0 msedge.exe                              bcryptprimitives
0x64c  0n1612 ffffa389b276d0c0 uhssvc.exe                              bcryptprimitives
0xbe8  0n3048 ffffa389b2a850c0 msteamsupdate.exe                       bcrypt,bcryptprimitives
0x1670 0n5744 ffffa389b27680c0 msedge.exe                              bcryptprimitives
0x1018 0n4120 ffffa389b13020c0 svchost.exe(UsoSvc)                     bcrypt,bcryptprimitives
0xe28  0n3624 ffffa389b279f080 Widgets.exe                             bcrypt,bcryptprimitives
0x1bb8 0n7096 ffffa389b3004080 WidgetService.exe                       bcrypt,bcryptprimitives
0x1960 0n6496 ffffa389b2ccf100 NisSrv.exe                              bcryptprimitives
0xc2c  0n3116 ffffa389b24c7080 msedgewebview2.exe                      bcrypt,bcryptprimitives
0xbbc  0n3004 ffffa389b2cc80c0 svchost.exe(wscsvc)                     bcryptprimitives
0xb64  0n2916 ffffa389b24ea0c0 msedgewebview2.exe                      bcryptprimitives
0x1c98 0n7320 ffffa389b2a800c0 msedgewebview2.exe                      bcryptprimitives
0x1ca0 0n7328 ffffa389b25d40c0 msedgewebview2.exe                      bcrypt,bcryptprimitives
0x14f0 0n5360 ffffa389b29e40c0 svchost.exe(lfsvc)                      bcrypt,bcryptprimitives
0x1150 0n4432 ffffa389b1d110c0 MoUsoCoreWorker.exe                     bcrypt,bcryptprimitives
0x1854 0n6228 ffffa389b61a70c0 svchost.exe(NPSMSvc)                    bcryptprimitives
0x90c  0n2316 ffffa389b4fae080 RuntimeBroker.exe                       bcrypt,bcryptprimitives
0x1b34 0n6964 ffffa389b61c3080 SystemSettingsBroker.exe                bcryptprimitives
0xfcc  0n4044 ffffa389b3d460c0 svchost.exe(WdiSystemHost)              bcryptprimitives
0x9b4  0n2484 ffffa389af7950c0 svchost.exe(SstpSvc)                    bcryptprimitives
0x18f8 0n6392 ffffa389b4782080 svchost.exe(netsvcs)                    bcrypt,bcryptprimitives
0x17b8 0n6072 ffffa389b4c9b080 svchost.exe(DisplayEnhancementService)  bcrypt,bcryptprimitives
0xa34  0n2612 ffffa389b2dee140 csrss.exe                               bcryptprimitives
0x14e0 0n5344 ffffa389afa540c0 winlogon.exe                            bcrypt,bcryptprimitives
0x2a0  0n672  ffffa389af9240c0 LogonUI.exe                             bcrypt,bcryptprimitives
0x7fc  0n2044 ffffa389b61af080 dwm.exe                                 bcrypt,bcryptprimitives
0x1b64 0n7012 ffffa389b4fb1080 RustDesk.exe                            bcrypt,bcryptprimitives
0x21e8 0n8680 ffffa389b2beb080 WUDFHost.exe                            bcryptprimitives
0x2378 0n9080 ffffa389b1bdf080 rdpclip.exe                             bcrypt,bcryptprimitives
0x2178 0n8568 ffffa389b1963080 svchost.exe(ScDeviceEnum)               bcryptprimitives
0x26c4 0n9924 ffffa389b6094080 TabTip.exe                              bcryptprimitives
0xd4c  0n3404 ffffa389b2ad9080 TextInputHost.exe                       bcrypt,bcryptprimitives
0x2654 0n9812 ffffa389af7c40c0 msedge.exe                              bcryptprimitives
0x19bc 0n6588 ffffa389af9e80c0 conhost.exe                             bcryptprimitives
0x26b8 0n9912 ffffa389b2e4b140 svchost.exe(DsmSvc)                     bcrypt,bcryptprimitives
============= ================ ======================================= =========================================
PID           Address          Name                                    Modules
```

If we're troubleshooting a memory leak and need to determine the memory usage of processes, we can use the **`!tl -mem`** command.

```
0: kd> !tl -mem
PID           Address          Name                                           VM      Peak    Shared Awe Size Commit Size  PP Quota NPP Quota
============= ================ ======================================= ========= ========= ========= ======== =========== ========= =========
0x0    0n0    fffff80152348f40 Idle                                         8 KB      8 KB                  0       60 KB               272 B
0x4    0n4    ffffa389ae6eb040 System                                    3.98 MB  20.48 MB    296 KB        0       40 KB               272 B
0x68   0n104  ffffa389ae6df080 Registry                                131.03 MB 131.56 MB                  0     7.46 MB 266.19 KB  12.48 KB
0x1f8  0n504  ffffa389b1713080 smss.exe                                     2 TB      2 TB    132 KB        0     1.12 MB  13.37 KB   3.95 KB
0x28c  0n652  ffffa389b29960c0 csrss.exe                                    2 TB      2 TB   8.52 MB        0     2.07 MB 241.06 KB  24.75 KB
0x2d4  0n724  ffffa389b2def080 wininit.exe                                  2 TB      2 TB   1.76 MB        0     1.57 MB  71.87 KB  12.05 KB
0x2dc  0n732  ffffa389b2df1140 csrss.exe                                    2 TB      2 TB  59.86 MB        0     2.24 MB 296.75 KB  19.69 KB
0x338  0n824  ffffa389b2eaf080 winlogon.exe                                 2 TB      2 TB  10.52 MB        0     2.66 MB 160.78 KB  13.87 KB
0x368  0n872  ffffa389b2f08080 services.exe                                 2 TB      2 TB    240 KB        0     5.29 MB 165.81 KB  14.25 KB
0x370  0n880  ffffa389b29d30c0 LsaIso.exe                                4.05 GB   4.05 GB      4 KB        0     1.24 MB  28.21 KB    6.9 KB
0x38c  0n908  ffffa389b2f09080 lsass.exe                                    2 TB      2 TB   2.07 MB        0     8.35 MB 156.84 KB  26.16 KB
0x22c  0n556  ffffa389b141e080 svchost.exe(-p)                              2 TB      2 TB    2.2 MB        0      9.3 MB 492.61 KB  26.59 KB
0x25c  0n604  ffffa389b302f080 fontdrvhost.exe                              2 TB      2 TB    236 KB        0     1.66 MB  40.66 KB   7.03 KB
0x218  0n536  ffffa389b3037080 fontdrvhost.exe                              2 TB      2 TB    236 KB        0     1.38 MB  38.41 KB   6.77 KB
0x418  0n1048 ffffa389b2f5a080 svchost.exe(-p)                              2 TB      2 TB    504 KB        0     6.13 MB 146.55 KB  21.53 KB
0x444  0n1092 ffffa389b2f9b080 svchost.exe(LSM)                             2 TB      2 TB   1.82 MB        0     3.12 MB 149.77 KB  16.06 KB
0x494  0n1172 ffffa389b3168080 dwm.exe                                      2 TB      2 TB 172.46 MB        0   113.95 MB 894.71 KB  68.84 KB
0x4e4  0n1252 ffffa389b31e9080 svchost.exe(TermService)                     2 TB      2 TB  118.9 MB        0   155.08 MB 409.21 KB  32.23 KB
0x508  0n1288 ffffa389b3210080 svchost.exe                                  2 TB      2 TB   1.78 MB        0     1.21 MB  58.53 KB   8.08 KB
0x544  0n1348 ffffa389b323b0c0 svchost.exe(NcbService)                      2 TB      2 TB   1.82 MB        0     2.34 MB  91.55 KB  12.19 KB
0x550  0n1360 ffffa389b3240080 svchost.exe(-p)                              2 TB      2 TB   1.78 MB        0     1.59 MB  59.48 KB   7.56 KB
0x560  0n1376 ffffa389b323e080 svchost.exe(TimeBrokerSvc)                   2 TB      2 TB   1.79 MB        0     1.96 MB  99.11 KB  10.09 KB
0x5cc  0n1484 ffffa389b32c1080 svchost.exe(Schedule)                        2 TB      2 TB   1.82 MB        0     5.98 MB 267.36 KB  20.22 KB
0x608  0n1544 ffffa389b32dc080 svchost.exe(ProfSvc)                         2 TB      2 TB   1.82 MB        0     2.21 MB 116.62 KB  11.02 KB
0x628  0n1576 ffffa389b3351080 svchost.exe(nsi)                             2 TB      2 TB   1.78 MB        0     4.62 MB  57.73 KB  23.48 KB
0x664  0n1636 ffffa389b336b080 svchost.exe(DispBrokerDesktopSvc)            2 TB      2 TB   1.78 MB        0     1.27 MB  60.56 KB    7.7 KB
0x66c  0n1644 ffffa389b336e080 svchost.exe(netprofm)                        2 TB      2 TB   2.08 MB        0     5.12 MB 144.21 KB  21.09 KB
0x674  0n1652 ffffa389b335b080 svchost.exe(UmRdpService)                    2 TB      2 TB   1.84 MB        0     2.54 MB 129.02 KB  35.53 KB
0x6b8  0n1720 ffffa389b33a0080 svchost.exe(UserManager)                     2 TB      2 TB   1.82 MB        0      2.2 MB  79.06 KB  10.88 KB
0x720  0n1824 ffffa389b33c7080 svchost.exe(CertPropSvc)                     2 TB      2 TB   1.82 MB        0     2.23 MB  86.83 KB 143.61 KB
0x738  0n1848 ffffa389b33d0080 svchost.exe(vmicheartbeat)                   2 TB      2 TB   1.82 MB        0     1.77 MB  61.92 KB  10.22 KB
0x75c  0n1884 ffffa389b33e3080 svchost.exe(vmicrdv)                         2 TB      2 TB   1.82 MB        0      2.1 MB  76.84 KB  11.41 KB
0x768  0n1896 ffffa389b33db080 svchost.exe                                  2 TB      2 TB   1.82 MB        0      1.3 MB  60.06 KB   8.61 KB
0x778  0n1912 ffffa389b33eb080 svchost.exe                                  2 TB      2 TB   1.82 MB        0     1.23 MB  58.76 KB   8.09 KB
0x790  0n1936 ffffa389b341c0c0 svchost.exe(vmictimesync)                    2 TB      2 TB   1.78 MB        0     1.25 MB  58.77 KB   8.23 KB
0x7c8  0n1992 ffffa389b343b080 svchost.exe(vmicvss)                         2 TB      2 TB   1.82 MB        0     1.43 MB  61.13 KB   8.89 KB
0x488  0n1160 ffffa389b34a1080 svchost.exe                                  2 TB      2 TB   1.78 MB        0     1.67 MB  82.89 KB   10.7 KB
0x86c  0n2156 ffffa389ae719080 svchost.exe(-p)                              2 TB      2 TB   1.78 MB        0     2.75 MB  89.53 KB  15.91 KB
0x8e4  0n2276 ffffa389b349c080 VSSVC.exe                                    2 TB      2 TB   1.82 MB        0     1.64 MB  78.58 KB  10.22 KB
0x924  0n2340 ffffa389ae796080 svchost.exe(EventLog)                        2 TB      2 TB   1.78 MB        0    19.07 MB  86.79 KB  15.87 KB
0x954  0n2388 ffffa389ae76e080 svchost.exe(SysMain)                      2.01 TB   2.01 TB   1.82 MB        0    50.87 MB 110.95 KB  16.06 KB
0x960  0n2400 ffffa389ae770080 svchost.exe(EventSystem)                     2 TB      2 TB   1.78 MB        0     1.89 MB  75.62 KB   9.88 KB
0x96c  0n2412 ffffa389ae75d080 svchost.exe                                  2 TB      2 TB  10.82 MB        0      1.3 MB   65.3 KB   8.09 KB
0x99c  0n2460 ffffa389ae78c080 svchost.exe(SessionEnv)                      2 TB      2 TB   1.82 MB        0     2.07 MB   85.8 KB  14.62 KB
0x9ec  0n2540 ffffa389ae754080 svchost.exe(Dhcp)                            2 TB      2 TB   1.78 MB        0     2.17 MB  80.34 KB  11.17 KB
0x9fc  0n2556 ffffa389ae757040 MemCompression                          176.75 MB    203 MB                  0      472 KB   4.12 KB          
0xae8  0n2792 ffffa389b368b080 svchost.exe(SENS)                            2 TB      2 TB   1.82 MB        0     1.77 MB  79.91 KB  10.23 KB
0xb24  0n2852 ffffa389b3708080 svchost.exe(AudioEndpointBuilder)            2 TB      2 TB   1.82 MB        0     1.53 MB  82.79 KB   10.2 KB
0xb40  0n2880 ffffa389b373e080 svchost.exe(FontCache)                       2 TB      2 TB   2.21 MB        0     1.74 MB 137.21 KB  11.12 KB
0xb7c  0n2940 ffffa389b375f080 svchost.exe(WinHttpAutoProxySvc)             2 TB      2 TB   1.78 MB        0     1.87 MB  75.63 KB  10.27 KB
0xb9c  0n2972 ffffa389b378a080 svchost.exe(StateRepository)                 2 TB      2 TB   1.82 MB        0     8.31 MB  91.31 KB  10.75 KB
0xbfc  0n3068 ffffa389b37ea080 svchost.exe(-p)                              2 TB      2 TB   1.79 MB        0     2.94 MB 116.28 KB  13.99 KB
0xaf4  0n2804 ffffa389b37ed080 svchost.exe(TextInputManagementService)      2 TB      2 TB   1.82 MB        0     1.41 MB  64.03 KB   9.15 KB
0xc10  0n3088 ffffa389b385e080 svchost.exe(-p)                              2 TB      2 TB   1.78 MB        0    12.12 MB 120.84 KB  34.35 KB
0xc20  0n3104 ffffa389b3860080 svchost.exe(wuauserv)                        2 TB      2 TB   3.13 MB        0    35.31 MB 358.84 KB   86.1 KB
0xc60  0n3168 ffffa389b3905140 svchost.exe(-p)                              2 TB      2 TB   1.78 MB        0     1.33 MB  70.35 KB   8.46 KB
0xc70  0n3184 ffffa389b39020c0 svchost.exe(-p)                              2 TB      2 TB   1.79 MB        0     2.09 MB  96.31 KB  12.37 KB
0xcc4  0n3268 ffffa389b395c080 svchost.exe(ShellHWDetection)                2 TB      2 TB   1.82 MB        0     2.44 MB  95.87 KB  15.02 KB
0xd40  0n3392 ffffa389b3a380c0 spoolsv.exe                                  2 TB      2 TB   1.82 MB        0     6.75 MB 176.66 KB   23.8 KB
0xd9c  0n3484 ffffa389b3b9b0c0 svchost.exe                                  2 TB      2 TB   1.82 MB        0     1.35 MB  65.27 KB   8.76 KB
0xe78  0n3704 ffffa389b3b33080 svchost.exe(iphlpsvc)                        2 TB      2 TB   1.82 MB        0     2.69 MB  103.2 KB  15.62 KB
0xe84  0n3716 ffffa389b3b20080 svchost.exe                                  2 TB      2 TB   1.82 MB        0     1.34 MB  68.56 KB   8.94 KB
0xe8c  0n3724 ffffa389b3b21080 svchost.exe(-p)                              2 TB      2 TB   1.91 MB        0      3.7 MB 123.56 KB  27.46 KB
0xe94  0n3732 ffffa389b3b22080 AnyDesk.exe*32                          102.93 MB 123.14 MB   3.88 MB        0    20.82 MB  182.6 KB  21.95 KB
0xea8  0n3752 ffffa389b3b88080 svchost.exe(-p)                              2 TB      2 TB   2.11 MB        0       16 MB 206.59 KB  30.14 KB
0xeb8  0n3768 ffffa389b3c75080 svchost.exe(DPS)                             2 TB      2 TB   1.79 MB        0    11.21 MB 130.16 KB  19.95 KB
0xed0  0n3792 ffffa389b3c86080 IpOverUsbSvc.exe*32                     122.68 MB 127.68 MB   1.88 MB        0     8.42 MB 180.98 KB  18.15 KB
0xf1c  0n3868 ffffa389b3d32080 svchost.exe(Winmgmt)                         2 TB      2 TB   1.82 MB        0     7.96 MB 108.47 KB  16.29 KB
0xf24  0n3876 ffffa389b3d22080 sqlwriter.exe                             4.07 GB   4.09 GB   1.82 MB        0     1.76 MB  84.49 KB  10.02 KB
0xf30  0n3888 ffffa389b3d44080 Sysmon.exe                                4.11 GB   4.13 GB   1.82 MB        0     9.69 MB 144.42 KB 272.86 KB
0xf3c  0n3900 ffffa389b3ca8080 RustDesk.exe                              4.12 GB   4.12 GB   1.84 MB        0     2.55 MB 178.71 KB  11.69 KB
0xf44  0n3908 ffffa389b3aa9080 svchost.exe(LanmanServer)                    2 TB      2 TB   1.82 MB        0     2.26 MB   89.2 KB  12.48 KB
0xf58  0n3928 ffffa389b3a98080 svchost.exe(TrkWks)                          2 TB      2 TB   1.82 MB        0     1.23 MB  60.11 KB      8 KB
0xf7c  0n3964 ffffa389b3bee080 MsMpEng.exe                                  2 TB   2.01 TB   1.91 MB        0   244.71 MB 561.64 KB 101.62 KB
0xf84  0n3972 ffffa389b3b77080 wlms.exe                                     2 TB      2 TB    240 KB        0      732 KB  29.74 KB   5.43 KB
0xf9c  0n3996 ffffa389b3b66080 svchost.exe(WpnService)                      2 TB      2 TB    2.1 MB        0     3.96 MB 148.23 KB   18.7 KB
0x1060 0n4192 ffffa389b3dcb080 sppsvc.exe                                   2 TB      2 TB   1.76 MB        0     5.31 MB 113.45 KB  12.88 KB
0x1174 0n4468 ffffa389b406e080 unsecapp.exe                                 2 TB      2 TB   1.82 MB        0     1.22 MB  65.96 KB   7.96 KB
0x12dc 0n4828 ffffa389b406c080 AggregatorHost.exe                           2 TB      2 TB    256 KB        0     1.84 MB  60.59 KB   8.36 KB
0x13c8 0n5064 ffffa389b34a30c0 WmiPrvSE.exe                                 2 TB      2 TB   1.82 MB        0     2.37 MB  78.08 KB  10.22 KB
0x10b4 0n4276 ffffa389b4241100 svchost.exe(RmSvc)                           2 TB      2 TB   2.06 MB        0     2.15 MB  74.93 KB  11.08 KB
0x1414 0n5140 ffffa389b42c90c0 svchost.exe(webthreatdefsvc)                 2 TB      2 TB   1.78 MB        0     3.29 MB  93.48 KB  11.88 KB
0x1574 0n5492 ffffa389b13f2080 svchost.exe(StorSvc)                         2 TB      2 TB   2.11 MB        0     2.18 MB 108.12 KB  10.22 KB
0x1620 0n5664 ffffa389b45e6080 sihost.exe                                   2 TB      2 TB   2.39 MB        0     5.44 MB 272.74 KB  17.98 KB
0x164c 0n5708 ffffa389b13ea080 svchost.exe(CDPUserSvc)                      2 TB      2 TB   2.39 MB        0     4.55 MB 183.05 KB  15.16 KB
0x1694 0n5780 ffffa389b45b3080 svchost.exe(webthreatdefusersvc)             2 TB      2 TB   2.11 MB        0     1.42 MB 109.44 KB   8.76 KB
0x1700 0n5888 ffffa389b4705080 svchost.exe(WpnUserService)                  2 TB      2 TB   2.39 MB        0     6.55 MB 270.25 KB  17.51 KB
0x175c 0n5980 ffffa389b47650c0 svchost.exe(TokenBroker)                     2 TB      2 TB    2.1 MB        0     3.29 MB 157.73 KB  13.94 KB
0x17ac 0n6060 ffffa389b470a080 taskhostw.exe                                2 TB      2 TB   3.77 MB        0     6.19 MB 190.37 KB  32.79 KB
0x1610 0n5648 ffffa389b476c080 svchost.exe(CDPSvc)                          2 TB      2 TB   2.07 MB        0     3.81 MB 130.38 KB  18.53 KB
0x2fc  0n764  ffffa389b47a3080 explorer.exe                                 2 TB      2 TB  47.21 MB        0   140.01 MB   2.17 MB  94.72 KB
0x1300 0n4864 ffffa389b31b00c0 ctfmon.exe                                   2 TB      2 TB   3.59 MB        0     4.68 MB  238.4 KB  18.23 KB
0xcd8  0n3288 ffffa389b39ac080 svchost.exe(cbdhsvc)                         2 TB      2 TB   2.52 MB        0     4.21 MB 251.91 KB   15.8 KB
0x1450 0n5200 ffffa389b3d33080 svchost.exe(camsvc)                          2 TB      2 TB    2.1 MB        0     3.44 MB 107.93 KB  13.14 KB
0xbf4  0n3060 ffffa389b481e080 svchost.exe(Appinfo)                         2 TB      2 TB   1.82 MB        0     1.64 MB  73.07 KB  10.22 KB
0x758  0n1880 ffffa389b48130c0 SearchIndexer.exe                            2 TB      2 TB   1.97 MB        0    41.37 MB 169.13 KB  20.15 KB
0x230  0n560  ffffa389b49760c0 SearchHost.exe                            2.04 TB   2.04 TB  17.81 MB        0   154.26 MB   1.69 MB 145.66 KB
0x14a0 0n5280 ffffa389b49740c0 StartMenuExperienceHost.exe                  2 TB      2 TB  15.88 MB        0    36.32 MB 717.12 KB  41.11 KB
0x1608 0n5640 ffffa389b4cf70c0 RuntimeBroker.exe                            2 TB      2 TB   3.89 MB        0    15.88 MB 420.63 KB  33.83 KB
0x1454 0n5204 ffffa389b48af080 RuntimeBroker.exe                            2 TB      2 TB   2.62 MB        0      5.9 MB  259.5 KB  20.84 KB
0x8cc  0n2252 ffffa389b4cdf0c0 svchost.exe(UdkUserSvc)                      2 TB      2 TB   2.39 MB        0     2.91 MB 192.34 KB   14.2 KB
0xaac  0n2732 ffffa389b4073080 svchost.exe(PcaSvc)                          2 TB      2 TB   1.82 MB        0     3.89 MB 116.29 KB  15.36 KB
0xcd0  0n3280 ffffa389b4d27080 svchost.exe(UnistackSvcGroup)                2 TB      2 TB   2.39 MB        0     2.79 MB 157.73 KB  14.73 KB
0xd88  0n3464 ffffa389aee4c080 dllhost.exe                                  2 TB      2 TB   2.23 MB        0     7.27 MB 170.92 KB  28.68 KB
0xe2c  0n3628 ffffa389aee45080 SppExtComObj.Exe                             2 TB      2 TB   1.78 MB        0     1.62 MB  96.37 KB    9.6 KB
0x13e8 0n5096 ffffa389aee44080 SgrmBroker.exe                               2 TB      2 TB    220 KB        0     7.91 MB  40.19 KB   9.16 KB
0x4b0  0n1200 ffffa389b605b080 SecurityHealthSystray.exe                    2 TB      2 TB   2.11 MB        0     1.72 MB 157.51 KB  10.22 KB
0xb68  0n2920 ffffa389b61da0c0 SecurityHealthService.exe                    2 TB      2 TB   1.83 MB        0     7.05 MB 195.86 KB   20.3 KB
0x744  0n1860 ffffa389b61df0c0 svchost.exe                                  2 TB      2 TB   1.82 MB        0     1.62 MB  66.44 KB   9.29 KB
0xad8  0n2776 ffffa389b35b10c0 OneDrive.exe                                 2 TB      2 TB   4.26 MB        0    21.38 MB 735.09 KB  48.15 KB
0x1344 0n4932 ffffa389b4ced080 svchost.exe(hns)                             2 TB      2 TB   1.82 MB        0     3.27 MB 101.22 KB  12.12 KB
0x1844 0n6212 ffffa389b1e1b080 svchost.exe(SSDPSRV)                         2 TB      2 TB   1.78 MB        0     1.84 MB  78.63 KB  15.46 KB
0x1968 0n6504 ffffa389b35b60c0 msedge.exe                                2.13 TB   2.13 TB   9.77 MB        0    45.17 MB   1.02 MB  49.33 KB
0x1a94 0n6804 ffffa389b4d180c0 svchost.exe(-p)                              2 TB      2 TB   2.13 MB        0     5.44 MB 182.31 KB  18.15 KB
0x1b28 0n6952 ffffa389b4d86080 msedge.exe                                2.07 TB   2.07 TB    3.3 MB        0     2.19 MB 129.64 KB  11.15 KB
0x1914 0n6420 ffffa389b2b0b080 AnyDesk.exe*32                           116.7 MB 121.38 MB   4.18 MB        0     21.7 MB 222.92 KB  19.73 KB
0x4a0  0n1184 ffffa389b2d460c0 msedge.exe                                 2.1 TB    2.1 TB   3.07 MB        0    11.21 MB 717.02 KB  19.45 KB
0x4ec  0n1260 ffffa389b13070c0 msedge.exe                                 2.1 TB    2.1 TB   2.62 MB        0     11.9 MB 720.28 KB  18.35 KB
0x10ec 0n4332 ffffa389b13290c0 msedge.exe                                 2.1 TB    2.1 TB    944 KB        0     5.98 MB 656.45 KB  13.14 KB
0x64c  0n1612 ffffa389b276d0c0 uhssvc.exe                                   2 TB      2 TB    236 KB        0      1.3 MB  61.95 KB   7.23 KB
0xbe8  0n3048 ffffa389b2a850c0 msteamsupdate.exe                            2 TB      2 TB   2.51 MB        0     4.87 MB 237.23 KB  20.99 KB
0x1670 0n5744 ffffa389b27680c0 msedge.exe                                3.17 TB   3.17 TB    2.8 MB        0    13.07 MB 673.45 KB  14.05 KB
0x1018 0n4120 ffffa389b13020c0 svchost.exe(UsoSvc)                          2 TB      2 TB   1.82 MB        0     2.79 MB  86.13 KB  14.58 KB
0xdf4  0n3572 ffffa389b6210080 ShellExperienceHost.exe                      2 TB      2 TB   3.03 MB        0    10.72 MB 540.39 KB  26.13 KB
0xe28  0n3624 ffffa389b279f080 Widgets.exe                               2.04 TB   2.04 TB   3.95 MB        0     9.48 MB  418.2 KB  34.02 KB
0x1bb8 0n7096 ffffa389b3004080 WidgetService.exe                            2 TB      2 TB   2.39 MB        0     4.34 MB 223.78 KB  16.76 KB
0x1960 0n6496 ffffa389b2ccf100 NisSrv.exe                                   2 TB      2 TB   1.78 MB        0     4.01 MB 126.21 KB  27.95 KB
0xc2c  0n3116 ffffa389b24c7080 msedgewebview2.exe                        2.13 TB   2.13 TB   11.3 MB        0    26.86 MB   1.03 MB  47.95 KB
0xbbc  0n3004 ffffa389b2cc80c0 svchost.exe(wscsvc)                          2 TB      2 TB   1.78 MB        0     2.95 MB  83.12 KB  13.45 KB
0xb64  0n2916 ffffa389b24ea0c0 msedgewebview2.exe                        2.07 TB   2.07 TB    3.3 MB        0      2.1 MB  119.7 KB  10.62 KB
0x1c98 0n7320 ffffa389b2a800c0 msedgewebview2.exe                         2.1 TB    2.1 TB  10.59 MB        0    14.44 MB  781.8 KB  24.16 KB
0x1ca0 0n7328 ffffa389b25d40c0 msedgewebview2.exe                         2.1 TB    2.1 TB   2.87 MB        0     7.99 MB 718.19 KB  16.49 KB
0x1d54 0n7508 ffffa389b3003080 msedgewebview2.exe                         2.1 TB    2.1 TB   1.17 MB        0     7.14 MB 656.02 KB  12.88 KB
0x1e54 0n7764 ffffa389b24e30c0 msedgewebview2.exe                        3.17 TB   3.17 TB   3.42 MB        0    29.65 MB 674.21 KB  14.58 KB
0x14f0 0n5360 ffffa389b29e40c0 svchost.exe(lfsvc)                           2 TB      2 TB   1.83 MB        0     2.91 MB 135.88 KB  13.73 KB
0x1150 0n4432 ffffa389b1d110c0 MoUsoCoreWorker.exe                          2 TB      2 TB   2.13 MB        0    11.71 MB 183.72 KB  21.09 KB
0x1854 0n6228 ffffa389b61a70c0 svchost.exe(NPSMSvc)                         2 TB      2 TB   2.11 MB        0     1.73 MB 132.42 KB   9.42 KB
0x90c  0n2316 ffffa389b4fae080 RuntimeBroker.exe                            2 TB      2 TB   3.33 MB        0     3.64 MB 199.05 KB  15.63 KB
0x1b34 0n6964 ffffa389b61c3080 SystemSettingsBroker.exe                     2 TB      2 TB    240 KB        0     1.02 MB  43.62 KB   5.97 KB
0xfcc  0n4044 ffffa389b3d460c0 svchost.exe(WdiSystemHost)                   2 TB      2 TB   1.82 MB        0      1.3 MB  59.89 KB   7.96 KB
0x9b4  0n2484 ffffa389af7950c0 svchost.exe(SstpSvc)                         2 TB      2 TB   1.78 MB        0      1.6 MB  72.14 KB  41.86 KB
0x18f8 0n6392 ffffa389b4782080 svchost.exe(netsvcs)                         2 TB      2 TB   1.82 MB        0     3.27 MB 124.55 KB  23.33 KB
0x17b8 0n6072 ffffa389b4c9b080 svchost.exe(DisplayEnhancementService)       2 TB      2 TB    2.1 MB        0     3.52 MB  94.27 KB  15.87 KB
0xa34  0n2612 ffffa389b2dee140 csrss.exe                                    2 TB      2 TB   2.13 MB        0     1.87 MB 112.77 KB     12 KB
0x14e0 0n5344 ffffa389afa540c0 winlogon.exe                                 2 TB      2 TB   3.02 MB        0      2.3 MB  131.2 KB  11.62 KB
0x7c4  0n1988 ffffa389af7b60c0 fontdrvhost.exe                              2 TB      2 TB    236 KB        0     1.31 MB  37.53 KB   6.63 KB
0x2a0  0n672  ffffa389af9240c0 LogonUI.exe                                  2 TB      2 TB   8.35 MB        0    14.25 MB  554.4 KB  37.72 KB
0x7fc  0n2044 ffffa389b61af080 dwm.exe                                      2 TB      2 TB   6.18 MB        0     15.5 MB 367.55 KB  25.48 KB
0x1b64 0n7012 ffffa389b4fb1080 RustDesk.exe                              4.16 GB   4.17 GB   1.79 MB        0     3.81 MB 221.95 KB  15.59 KB
0x21e8 0n8680 ffffa389b2beb080 WUDFHost.exe                                 2 TB      2 TB 758.57 MB        0     7.93 MB   1.65 MB  20.96 KB
0x2378 0n9080 ffffa389b1bdf080 rdpclip.exe                                  2 TB      2 TB    3.3 MB        0     3.48 MB 231.12 KB  17.82 KB
0x2178 0n8568 ffffa389b1963080 svchost.exe(ScDeviceEnum)                    2 TB      2 TB   1.82 MB        0     1.47 MB  72.05 KB  10.88 KB
0x2424 0n9252 ffffa389af9570c0 rdpinput.exe                                 2 TB      2 TB   3.29 MB        0      1.4 MB  105.1 KB   8.23 KB
0x26c4 0n9924 ffffa389b6094080 TabTip.exe                                   2 TB      2 TB   2.14 MB        0     3.52 MB 245.65 KB  16.05 KB
0xd4c  0n3404 ffffa389b2ad9080 TextInputHost.exe                            2 TB      2 TB   3.34 MB        0     14.7 MB 650.65 KB  31.52 KB
0x2654 0n9812 ffffa389af7c40c0 msedge.exe                                3.17 TB   3.17 TB   2.86 MB        0    48.33 MB 704.36 KB  15.77 KB
0x1890 0n6288 ffffa389af9d10c0 cmd.exe                                      2 TB      2 TB    240 KB        0     2.07 MB  48.85 KB   6.14 KB
0x19bc 0n6588 ffffa389af9e80c0 conhost.exe                                  2 TB      2 TB  20.09 MB        0     6.67 MB 266.73 KB  15.13 KB
0x26b8 0n9912 ffffa389b2e4b140 svchost.exe(DsmSvc)                          2 TB      2 TB   1.82 MB        0     3.86 MB 116.96 KB  17.98 KB
0x20f0 0n8432 ffffa389afab70c0 notmyfault64.exe                          4.09 GB   4.09 GB   2.11 MB        0     1.21 MB 121.32 KB   7.85 KB
============= ================ ======================================= ========= ========= ========= ======== =========== ========= =========
PID           Address          Name                                           VM      Peak    Shared Awe Size Commit Size  PP Quota NPP Quota
```

Instead of looking for all the processes, we can filter this down to only look for the memory consumption of processes running in session 3 for example.

```
0: kd> !tl -s 3 -mem
PID           Address          Name            Ses      VM    Peak  Shared Awe Size Commit Size  PP Quota NPP Quota
============= ================ =============== === ======= ======= ======= ======== =========== ========= =========
0xa34  0n2612 ffffa389b2dee140 csrss.exe         3    2 TB    2 TB 2.13 MB        0     1.87 MB 112.77 KB     12 KB
0x14e0 0n5344 ffffa389afa540c0 winlogon.exe      3    2 TB    2 TB 3.02 MB        0      2.3 MB  131.2 KB  11.62 KB
0x7c4  0n1988 ffffa389af7b60c0 fontdrvhost.exe   3    2 TB    2 TB  236 KB        0     1.31 MB  37.53 KB   6.63 KB
0x2a0  0n672  ffffa389af9240c0 LogonUI.exe       3    2 TB    2 TB 8.35 MB        0    14.25 MB  554.4 KB  37.72 KB
0x7fc  0n2044 ffffa389b61af080 dwm.exe           3    2 TB    2 TB 6.18 MB        0     15.5 MB 367.55 KB  25.48 KB
0x1b64 0n7012 ffffa389b4fb1080 RustDesk.exe      3 4.16 GB 4.17 GB 1.79 MB        0     3.81 MB 221.95 KB  15.59 KB
============= ================ =============== === ======= ======= ======= ======== =========== ========= =========
PID           Address          Name            Ses      VM    Peak  Shared Awe Size Commit Size  PP Quota NPP Quota
```

We can do the same thing as well for the "svchost" processes that are all running in session 0 and get their memory consumption as well.

```
0: kd> !tl svchost -s 0 -mem
PID           Address          Name                                    Ses      VM    Peak   Shared Awe Size Commit Size  PP Quota NPP Quota
============= ================ ======================================= === ======= ======= ======== ======== =========== ========= =========
0x22c  0n556  ffffa389b141e080 svchost.exe(-p)                           0    2 TB    2 TB   2.2 MB        0      9.3 MB 492.61 KB  26.59 KB
0x418  0n1048 ffffa389b2f5a080 svchost.exe(-p)                           0    2 TB    2 TB   504 KB        0     6.13 MB 146.55 KB  21.53 KB
0x444  0n1092 ffffa389b2f9b080 svchost.exe(LSM)                          0    2 TB    2 TB  1.82 MB        0     3.12 MB 149.77 KB  16.06 KB
0x4e4  0n1252 ffffa389b31e9080 svchost.exe(TermService)                  0    2 TB    2 TB 118.9 MB        0   155.08 MB 409.21 KB  32.23 KB
0x508  0n1288 ffffa389b3210080 svchost.exe                               0    2 TB    2 TB  1.78 MB        0     1.21 MB  58.53 KB   8.08 KB
0x544  0n1348 ffffa389b323b0c0 svchost.exe(NcbService)                   0    2 TB    2 TB  1.82 MB        0     2.34 MB  91.55 KB  12.19 KB
0x550  0n1360 ffffa389b3240080 svchost.exe(-p)                           0    2 TB    2 TB  1.78 MB        0     1.59 MB  59.48 KB   7.56 KB
0x560  0n1376 ffffa389b323e080 svchost.exe(TimeBrokerSvc)                0    2 TB    2 TB  1.79 MB        0     1.96 MB  99.11 KB  10.09 KB
0x5cc  0n1484 ffffa389b32c1080 svchost.exe(Schedule)                     0    2 TB    2 TB  1.82 MB        0     5.98 MB 267.36 KB  20.22 KB
0x608  0n1544 ffffa389b32dc080 svchost.exe(ProfSvc)                      0    2 TB    2 TB  1.82 MB        0     2.21 MB 116.62 KB  11.02 KB
0x628  0n1576 ffffa389b3351080 svchost.exe(nsi)                          0    2 TB    2 TB  1.78 MB        0     4.62 MB  57.73 KB  23.48 KB
0x664  0n1636 ffffa389b336b080 svchost.exe(DispBrokerDesktopSvc)         0    2 TB    2 TB  1.78 MB        0     1.27 MB  60.56 KB    7.7 KB
0x66c  0n1644 ffffa389b336e080 svchost.exe(netprofm)                     0    2 TB    2 TB  2.08 MB        0     5.12 MB 144.21 KB  21.09 KB
0x674  0n1652 ffffa389b335b080 svchost.exe(UmRdpService)                 0    2 TB    2 TB  1.84 MB        0     2.54 MB 129.02 KB  35.53 KB
0x6b8  0n1720 ffffa389b33a0080 svchost.exe(UserManager)                  0    2 TB    2 TB  1.82 MB        0      2.2 MB  79.06 KB  10.88 KB
0x720  0n1824 ffffa389b33c7080 svchost.exe(CertPropSvc)                  0    2 TB    2 TB  1.82 MB        0     2.23 MB  86.83 KB 143.61 KB
0x738  0n1848 ffffa389b33d0080 svchost.exe(vmicheartbeat)                0    2 TB    2 TB  1.82 MB        0     1.77 MB  61.92 KB  10.22 KB
0x75c  0n1884 ffffa389b33e3080 svchost.exe(vmicrdv)                      0    2 TB    2 TB  1.82 MB        0      2.1 MB  76.84 KB  11.41 KB
0x768  0n1896 ffffa389b33db080 svchost.exe                               0    2 TB    2 TB  1.82 MB        0      1.3 MB  60.06 KB   8.61 KB
0x778  0n1912 ffffa389b33eb080 svchost.exe                               0    2 TB    2 TB  1.82 MB        0     1.23 MB  58.76 KB   8.09 KB
0x790  0n1936 ffffa389b341c0c0 svchost.exe(vmictimesync)                 0    2 TB    2 TB  1.78 MB        0     1.25 MB  58.77 KB   8.23 KB
0x7c8  0n1992 ffffa389b343b080 svchost.exe(vmicvss)                      0    2 TB    2 TB  1.82 MB        0     1.43 MB  61.13 KB   8.89 KB
0x488  0n1160 ffffa389b34a1080 svchost.exe                               0    2 TB    2 TB  1.78 MB        0     1.67 MB  82.89 KB   10.7 KB
0x86c  0n2156 ffffa389ae719080 svchost.exe(-p)                           0    2 TB    2 TB  1.78 MB        0     2.75 MB  89.53 KB  15.91 KB
0x924  0n2340 ffffa389ae796080 svchost.exe(EventLog)                     0    2 TB    2 TB  1.78 MB        0    19.07 MB  86.79 KB  15.87 KB
0x954  0n2388 ffffa389ae76e080 svchost.exe(SysMain)                      0 2.01 TB 2.01 TB  1.82 MB        0    50.87 MB 110.95 KB  16.06 KB
0x960  0n2400 ffffa389ae770080 svchost.exe(EventSystem)                  0    2 TB    2 TB  1.78 MB        0     1.89 MB  75.62 KB   9.88 KB
0x96c  0n2412 ffffa389ae75d080 svchost.exe                               0    2 TB    2 TB 10.82 MB        0      1.3 MB   65.3 KB   8.09 KB
0x99c  0n2460 ffffa389ae78c080 svchost.exe(SessionEnv)                   0    2 TB    2 TB  1.82 MB        0     2.07 MB   85.8 KB  14.62 KB
0x9ec  0n2540 ffffa389ae754080 svchost.exe(Dhcp)                         0    2 TB    2 TB  1.78 MB        0     2.17 MB  80.34 KB  11.17 KB
0xae8  0n2792 ffffa389b368b080 svchost.exe(SENS)                         0    2 TB    2 TB  1.82 MB        0     1.77 MB  79.91 KB  10.23 KB
0xb24  0n2852 ffffa389b3708080 svchost.exe(AudioEndpointBuilder)         0    2 TB    2 TB  1.82 MB        0     1.53 MB  82.79 KB   10.2 KB
0xb40  0n2880 ffffa389b373e080 svchost.exe(FontCache)                    0    2 TB    2 TB  2.21 MB        0     1.74 MB 137.21 KB  11.12 KB
0xb7c  0n2940 ffffa389b375f080 svchost.exe(WinHttpAutoProxySvc)          0    2 TB    2 TB  1.78 MB        0     1.87 MB  75.63 KB  10.27 KB
0xb9c  0n2972 ffffa389b378a080 svchost.exe(StateRepository)              0    2 TB    2 TB  1.82 MB        0     8.31 MB  91.31 KB  10.75 KB
0xbfc  0n3068 ffffa389b37ea080 svchost.exe(-p)                           0    2 TB    2 TB  1.79 MB        0     2.94 MB 116.28 KB  13.99 KB
0xaf4  0n2804 ffffa389b37ed080 svchost.exe(TextInputManagementService)   0    2 TB    2 TB  1.82 MB        0     1.41 MB  64.03 KB   9.15 KB
0xc10  0n3088 ffffa389b385e080 svchost.exe(-p)                           0    2 TB    2 TB  1.78 MB        0    12.12 MB 120.84 KB  34.35 KB
0xc20  0n3104 ffffa389b3860080 svchost.exe(wuauserv)                     0    2 TB    2 TB  3.13 MB        0    35.31 MB 358.84 KB   86.1 KB
0xc60  0n3168 ffffa389b3905140 svchost.exe(-p)                           0    2 TB    2 TB  1.78 MB        0     1.33 MB  70.35 KB   8.46 KB
0xc70  0n3184 ffffa389b39020c0 svchost.exe(-p)                           0    2 TB    2 TB  1.79 MB        0     2.09 MB  96.31 KB  12.37 KB
0xcc4  0n3268 ffffa389b395c080 svchost.exe(ShellHWDetection)             0    2 TB    2 TB  1.82 MB        0     2.44 MB  95.87 KB  15.02 KB
0xd9c  0n3484 ffffa389b3b9b0c0 svchost.exe                               0    2 TB    2 TB  1.82 MB        0     1.35 MB  65.27 KB   8.76 KB
0xe78  0n3704 ffffa389b3b33080 svchost.exe(iphlpsvc)                     0    2 TB    2 TB  1.82 MB        0     2.69 MB  103.2 KB  15.62 KB
0xe84  0n3716 ffffa389b3b20080 svchost.exe                               0    2 TB    2 TB  1.82 MB        0     1.34 MB  68.56 KB   8.94 KB
0xe8c  0n3724 ffffa389b3b21080 svchost.exe(-p)                           0    2 TB    2 TB  1.91 MB        0      3.7 MB 123.56 KB  27.46 KB
0xea8  0n3752 ffffa389b3b88080 svchost.exe(-p)                           0    2 TB    2 TB  2.11 MB        0       16 MB 206.59 KB  30.14 KB
0xeb8  0n3768 ffffa389b3c75080 svchost.exe(DPS)                          0    2 TB    2 TB  1.79 MB        0    11.21 MB 130.16 KB  19.95 KB
0xf1c  0n3868 ffffa389b3d32080 svchost.exe(Winmgmt)                      0    2 TB    2 TB  1.82 MB        0     7.96 MB 108.47 KB  16.29 KB
0xf44  0n3908 ffffa389b3aa9080 svchost.exe(LanmanServer)                 0    2 TB    2 TB  1.82 MB        0     2.26 MB   89.2 KB  12.48 KB
0xf58  0n3928 ffffa389b3a98080 svchost.exe(TrkWks)                       0    2 TB    2 TB  1.82 MB        0     1.23 MB  60.11 KB      8 KB
0xf9c  0n3996 ffffa389b3b66080 svchost.exe(WpnService)                   0    2 TB    2 TB   2.1 MB        0     3.96 MB 148.23 KB   18.7 KB
0x10b4 0n4276 ffffa389b4241100 svchost.exe(RmSvc)                        0    2 TB    2 TB  2.06 MB        0     2.15 MB  74.93 KB  11.08 KB
0x1414 0n5140 ffffa389b42c90c0 svchost.exe(webthreatdefsvc)              0    2 TB    2 TB  1.78 MB        0     3.29 MB  93.48 KB  11.88 KB
0x1574 0n5492 ffffa389b13f2080 svchost.exe(StorSvc)                      0    2 TB    2 TB  2.11 MB        0     2.18 MB 108.12 KB  10.22 KB
0x175c 0n5980 ffffa389b47650c0 svchost.exe(TokenBroker)                  0    2 TB    2 TB   2.1 MB        0     3.29 MB 157.73 KB  13.94 KB
0x1610 0n5648 ffffa389b476c080 svchost.exe(CDPSvc)                       0    2 TB    2 TB  2.07 MB        0     3.81 MB 130.38 KB  18.53 KB
0x1450 0n5200 ffffa389b3d33080 svchost.exe(camsvc)                       0    2 TB    2 TB   2.1 MB        0     3.44 MB 107.93 KB  13.14 KB
0xbf4  0n3060 ffffa389b481e080 svchost.exe(Appinfo)                      0    2 TB    2 TB  1.82 MB        0     1.64 MB  73.07 KB  10.22 KB
0xaac  0n2732 ffffa389b4073080 svchost.exe(PcaSvc)                       0    2 TB    2 TB  1.82 MB        0     3.89 MB 116.29 KB  15.36 KB
0x744  0n1860 ffffa389b61df0c0 svchost.exe                               0    2 TB    2 TB  1.82 MB        0     1.62 MB  66.44 KB   9.29 KB
0x1344 0n4932 ffffa389b4ced080 svchost.exe(hns)                          0    2 TB    2 TB  1.82 MB        0     3.27 MB 101.22 KB  12.12 KB
0x1844 0n6212 ffffa389b1e1b080 svchost.exe(SSDPSRV)                      0    2 TB    2 TB  1.78 MB        0     1.84 MB  78.63 KB  15.46 KB
0x1a94 0n6804 ffffa389b4d180c0 svchost.exe(-p)                           0    2 TB    2 TB  2.13 MB        0     5.44 MB 182.31 KB  18.15 KB
0x1018 0n4120 ffffa389b13020c0 svchost.exe(UsoSvc)                       0    2 TB    2 TB  1.82 MB        0     2.79 MB  86.13 KB  14.58 KB
0xbbc  0n3004 ffffa389b2cc80c0 svchost.exe(wscsvc)                       0    2 TB    2 TB  1.78 MB        0     2.95 MB  83.12 KB  13.45 KB
0x14f0 0n5360 ffffa389b29e40c0 svchost.exe(lfsvc)                        0    2 TB    2 TB  1.83 MB        0     2.91 MB 135.88 KB  13.73 KB
0xfcc  0n4044 ffffa389b3d460c0 svchost.exe(WdiSystemHost)                0    2 TB    2 TB  1.82 MB        0      1.3 MB  59.89 KB   7.96 KB
0x9b4  0n2484 ffffa389af7950c0 svchost.exe(SstpSvc)                      0    2 TB    2 TB  1.78 MB        0      1.6 MB  72.14 KB  41.86 KB
0x18f8 0n6392 ffffa389b4782080 svchost.exe(netsvcs)                      0    2 TB    2 TB  1.82 MB        0     3.27 MB 124.55 KB  23.33 KB
0x17b8 0n6072 ffffa389b4c9b080 svchost.exe(DisplayEnhancementService)    0    2 TB    2 TB   2.1 MB        0     3.52 MB  94.27 KB  15.87 KB
0x2178 0n8568 ffffa389b1963080 svchost.exe(ScDeviceEnum)                 0    2 TB    2 TB  1.82 MB        0     1.47 MB  72.05 KB  10.88 KB
0x26b8 0n9912 ffffa389b2e4b140 svchost.exe(DsmSvc)                       0    2 TB    2 TB  1.82 MB        0     3.86 MB 116.96 KB  17.98 KB
============= ================ ======================================= === ======= ======= ======== ======== =========== ========= =========
PID           Address          Name                                    Ses      VM    Peak   Shared Awe Size Commit Size  PP Quota NPP Quota
```

What if we want to find out the **CPU time** was consumed in both user mode and kernel mode for a particular process like **"svchost"** that is running in session **0**?

```
0: kd> !tl svchost -s 0 -cpu
PID           Address          Name                                         User    Kernel     Total Ses
============= ================ ======================================= ========= ========= ========= ===
0x22c  0n556  ffffa389b141e080 svchost.exe(-p)                           10s.328   27s.062   37s.390   0
0x418  0n1048 ffffa389b2f5a080 svchost.exe(-p)                           23s.174   36s.595   59s.769   0
0x444  0n1092 ffffa389b2f9b080 svchost.exe(LSM)                           1s.704    1s.767    3s.471   0
0x4e4  0n1252 ffffa389b31e9080 svchost.exe(TermService)                  18s.392    3s.737   22s.129   0
0x508  0n1288 ffffa389b3210080 svchost.exe                                  31ms      16ms      47ms   0
0x544  0n1348 ffffa389b323b0c0 svchost.exe(NcbService)                     468ms     375ms     843ms   0
0x550  0n1360 ffffa389b3240080 svchost.exe(-p)                            1s.047     657ms    1s.704   0
0x560  0n1376 ffffa389b323e080 svchost.exe(TimeBrokerSvc)                  531ms     437ms     968ms   0
0x5cc  0n1484 ffffa389b32c1080 svchost.exe(Schedule)                      3s.876    9s.454   13s.330   0
0x608  0n1544 ffffa389b32dc080 svchost.exe(ProfSvc)                        344ms    1s.281    1s.625   0
0x628  0n1576 ffffa389b3351080 svchost.exe(nsi)                            671ms     735ms    1s.406   0
0x664  0n1636 ffffa389b336b080 svchost.exe(DispBrokerDesktopSvc)            15ms      63ms      78ms   0
0x66c  0n1644 ffffa389b336e080 svchost.exe(netprofm)                      2s.672    2s.328    5s.000   0
0x674  0n1652 ffffa389b335b080 svchost.exe(UmRdpService)                   125ms     141ms     266ms   0
0x6b8  0n1720 ffffa389b33a0080 svchost.exe(UserManager)                   2s.453    5s.141    7s.594   0
0x720  0n1824 ffffa389b33c7080 svchost.exe(CertPropSvc)                    141ms      78ms     219ms   0
0x738  0n1848 ffffa389b33d0080 svchost.exe(vmicheartbeat)                 2s.859    3s.219    6s.078   0
0x75c  0n1884 ffffa389b33e3080 svchost.exe(vmicrdv)                         32ms     110ms     142ms   0
0x768  0n1896 ffffa389b33db080 svchost.exe                                  31ms      79ms     110ms   0
0x778  0n1912 ffffa389b33eb080 svchost.exe                                  15ms      16ms      31ms   0
0x790  0n1936 ffffa389b341c0c0 svchost.exe(vmictimesync)                    62ms    3s.141    3s.203   0
0x7c8  0n1992 ffffa389b343b080 svchost.exe(vmicvss)                         15ms     125ms     140ms   0
0x488  0n1160 ffffa389b34a1080 svchost.exe                                 109ms     156ms     265ms   0
0x86c  0n2156 ffffa389ae719080 svchost.exe(-p)                             969ms    2s.048    3s.017   0
0x924  0n2340 ffffa389ae796080 svchost.exe(EventLog)                      3s.157    6s.142    9s.299   0
0x954  0n2388 ffffa389ae76e080 svchost.exe(SysMain)                    1m:39.656 1m:16.907 2m:56.563   0
0x960  0n2400 ffffa389ae770080 svchost.exe(EventSystem)                    297ms     359ms     656ms   0
0x96c  0n2412 ffffa389ae75d080 svchost.exe                                 125ms     219ms     344ms   0
0x99c  0n2460 ffffa389ae78c080 svchost.exe(SessionEnv)                      31ms     282ms     313ms   0
0x9ec  0n2540 ffffa389ae754080 svchost.exe(Dhcp)                           609ms     735ms    1s.344   0
0xae8  0n2792 ffffa389b368b080 svchost.exe(SENS)                           249ms     470ms     719ms   0
0xb24  0n2852 ffffa389b3708080 svchost.exe(AudioEndpointBuilder)            31ms      47ms      78ms   0
0xb40  0n2880 ffffa389b373e080 svchost.exe(FontCache)                     8s.437     548ms    8s.985   0
0xb7c  0n2940 ffffa389b375f080 svchost.exe(WinHttpAutoProxySvc)           1s.140    2s.547    3s.687   0
0xb9c  0n2972 ffffa389b378a080 svchost.exe(StateRepository)              26s.968    9s.110   36s.078   0
0xbfc  0n3068 ffffa389b37ea080 svchost.exe(-p)                             172ms    1s.390    1s.562   0
0xaf4  0n2804 ffffa389b37ed080 svchost.exe(TextInputManagementService)     156ms     359ms     515ms   0
0xc10  0n3088 ffffa389b385e080 svchost.exe(-p)                            6s.671    8s.890   15s.561   0
0xc20  0n3104 ffffa389b3860080 svchost.exe(wuauserv)                   1m:11.281   35s.298 1m:46.579   0
0xc60  0n3168 ffffa389b3905140 svchost.exe(-p)                              46ms      63ms     109ms   0
0xc70  0n3184 ffffa389b39020c0 svchost.exe(-p)                             234ms     344ms     578ms   0
0xcc4  0n3268 ffffa389b395c080 svchost.exe(ShellHWDetection)               126ms     188ms     314ms   0
0xd9c  0n3484 ffffa389b3b9b0c0 svchost.exe                                 109ms      94ms     203ms   0
0xe78  0n3704 ffffa389b3b33080 svchost.exe(iphlpsvc)                       640ms     734ms    1s.374   0
0xe84  0n3716 ffffa389b3b20080 svchost.exe                                  46ms      79ms     125ms   0
0xe8c  0n3724 ffffa389b3b21080 svchost.exe(-p)                             797ms    2s.250    3s.047   0
0xea8  0n3752 ffffa389b3b88080 svchost.exe(-p)                           25s.936   23s.298   49s.234   0
0xeb8  0n3768 ffffa389b3c75080 svchost.exe(DPS)                           8s.079    8s.895   16s.974   0
0xf1c  0n3868 ffffa389b3d32080 svchost.exe(Winmgmt)                      16s.672   10s.062   26s.734   0
0xf44  0n3908 ffffa389b3aa9080 svchost.exe(LanmanServer)                   219ms     406ms     625ms   0
0xf58  0n3928 ffffa389b3a98080 svchost.exe(TrkWks)                          16ms     125ms     141ms   0
0xf9c  0n3996 ffffa389b3b66080 svchost.exe(WpnService)                    4s.016    2s.265    6s.281   0
0x10b4 0n4276 ffffa389b4241100 svchost.exe(RmSvc)                           62ms     109ms     171ms   0
0x1414 0n5140 ffffa389b42c90c0 svchost.exe(webthreatdefsvc)               3s.875    2s.188    6s.063   0
0x1574 0n5492 ffffa389b13f2080 svchost.exe(StorSvc)                        405ms    1s.422    1s.827   0
0x175c 0n5980 ffffa389b47650c0 svchost.exe(TokenBroker)                   6s.937    5s.937   12s.874   0
0x1610 0n5648 ffffa389b476c080 svchost.exe(CDPSvc)                         687ms     703ms    1s.390   0
0x1450 0n5200 ffffa389b3d33080 svchost.exe(camsvc)                        6s.735   14s.688   21s.423   0
0xbf4  0n3060 ffffa389b481e080 svchost.exe(Appinfo)                        219ms     687ms     906ms   0
0xaac  0n2732 ffffa389b4073080 svchost.exe(PcaSvc)                         469ms    2s.906    3s.375   0
0x744  0n1860 ffffa389b61df0c0 svchost.exe                                 187ms     484ms     671ms   0
0x1344 0n4932 ffffa389b4ced080 svchost.exe(hns)                            749ms    1s.782    2s.531   0
0x1844 0n6212 ffffa389b1e1b080 svchost.exe(SSDPSRV)                        250ms     297ms     547ms   0
0x1a94 0n6804 ffffa389b4d180c0 svchost.exe(-p)                            9s.656    7s.016   16s.672   0
0x1018 0n4120 ffffa389b13020c0 svchost.exe(UsoSvc)                         968ms    1s.908    2s.876   0
0xbbc  0n3004 ffffa389b2cc80c0 svchost.exe(wscsvc)                         766ms    1s.720    2s.486   0
0x14f0 0n5360 ffffa389b29e40c0 svchost.exe(lfsvc)                          906ms    1s.484    2s.390   0
0xfcc  0n4044 ffffa389b3d460c0 svchost.exe(WdiSystemHost)                  218ms     172ms     390ms   0
0x9b4  0n2484 ffffa389af7950c0 svchost.exe(SstpSvc)                            0      78ms      78ms   0
0x18f8 0n6392 ffffa389b4782080 svchost.exe(netsvcs)                         48ms     204ms     252ms   0
0x17b8 0n6072 ffffa389b4c9b080 svchost.exe(DisplayEnhancementService)      140ms     422ms     562ms   0
0x2178 0n8568 ffffa389b1963080 svchost.exe(ScDeviceEnum)                    31ms         0      31ms   0
0x26b8 0n9912 ffffa389b2e4b140 svchost.exe(DsmSvc)                          63ms      93ms     156ms   0
============= ================ ======================================= ========= ========= ========= ===
PID           Address          Name                                         User    Kernel     Total Ses
```

This command lets us focus on retrieving details like thread count, session, process creation time, and the associated command-line. The **`-proc`** option provides these in-depth insights about the process.

```
0: kd> !tl -s 3 -proc
PID           Address          Name            Ses Thd Hnd Create Time         Command Line
============= ================ =============== === === === =================== ====================================================================================================================================================================================================================================================
0xa34  0n2612 ffffa389b2dee140 csrss.exe         3  12   0 08/13/2023 04:43 AM %SystemRoot%\system32\csrss.exe ObjectDirectory=\Windows SharedSection=1024,20480,768 Windows=On SubSystemType=Windows ServerDll=basesrv,1 ServerDll=winsrv:UserServerDllInitialization,3 ServerDll=sxssrv,4 ProfileControl=Off MaxRequestThreads=16
0x14e0 0n5344 ffffa389afa540c0 winlogon.exe      3   5   0 08/13/2023 04:43 AM winlogon.exe
0x7c4  0n1988 ffffa389af7b60c0 fontdrvhost.exe   3   5   0 08/13/2023 04:43 AM "fontdrvhost.exe"
0x2a0  0n672  ffffa389af9240c0 LogonUI.exe       3  18   0 08/13/2023 04:43 AM "LogonUI.exe" /flags:0x2 /state0:0xa3f62055 /state1:0x41c64e6d
0x7fc  0n2044 ffffa389b61af080 dwm.exe           3  17   0 08/13/2023 04:43 AM "dwm.exe"
0x1b64 0n7012 ffffa389b4fb1080 RustDesk.exe      3  14   0 08/13/2023 04:43 AM "C:\Program Files\RustDesk\RustDesk.exe" --server
============= ================ =============== === === === =================== ====================================================================================================================================================================================================================================================
PID           Address          Name            Ses Thd Hnd Create Time         Command Line
```

This command allows us to sort on the thread count of all the running processes. 

```
0: kd> !tl -sort thread
PID           Address          Name                                    Thd
============= ================ ======================================= ===
0xe2c  0n3628 ffffa389aee45080 SppExtComObj.Exe                          1
0xcd0  0n3280 ffffa389b4d27080 svchost.exe(UnistackSvcGroup)             1
0x4b0  0n1200 ffffa389b605b080 SecurityHealthSystray.exe                 1
0x664  0n1636 ffffa389b336b080 svchost.exe(DispBrokerDesktopSvc)         1
0x1574 0n5492 ffffa389b13f2080 svchost.exe(StorSvc)                      1
0x12dc 0n4828 ffffa389b406c080 AggregatorHost.exe                        1
0xc60  0n3168 ffffa389b3905140 svchost.exe(-p)                           1
0xe84  0n3716 ffffa389b3b20080 svchost.exe                               1
0xbf4  0n3060 ffffa389b481e080 svchost.exe(Appinfo)                      1
0xd9c  0n3484 ffffa389b3b9b0c0 svchost.exe                               1
0x544  0n1348 ffffa389b323b0c0 svchost.exe(NcbService)                   1
0x9b4  0n2484 ffffa389af7950c0 svchost.exe(SstpSvc)                      1
0x1b34 0n6964 ffffa389b61c3080 SystemSettingsBroker.exe                  1
0x1890 0n6288 ffffa389af9d10c0 cmd.exe                                   1
0xb24  0n2852 ffffa389b3708080 svchost.exe(AudioEndpointBuilder)         2
0x1694 0n5780 ffffa389b45b3080 svchost.exe(webthreatdefusersvc)          2
0x8e4  0n2276 ffffa389b349c080 VSSVC.exe                                 2
0xae8  0n2792 ffffa389b368b080 svchost.exe(SENS)                         2
0xf84  0n3972 ffffa389b3b77080 wlms.exe                                  2
0xc70  0n3184 ffffa389b39020c0 svchost.exe(-p)                           2
0xf24  0n3876 ffffa389b3d22080 sqlwriter.exe                             2
0x1174 0n4468 ffffa389b406e080 unsecapp.exe                              2
0x370  0n880  ffffa389b29d30c0 LsaIso.exe                                2
0x550  0n1360 ffffa389b3240080 svchost.exe(-p)                           2
0xfcc  0n4044 ffffa389b3d460c0 svchost.exe(WdiSystemHost)                2
0x508  0n1288 ffffa389b3210080 svchost.exe                               3
0x2d4  0n724  ffffa389b2def080 wininit.exe                               3
0x1f8  0n504  ffffa389b1713080 smss.exe                                  3
0xf3c  0n3900 ffffa389b3ca8080 RustDesk.exe                              3
0xaf4  0n2804 ffffa389b37ed080 svchost.exe(TextInputManagementService)   3
0x608  0n1544 ffffa389b32dc080 svchost.exe(ProfSvc)                      3
0x1a94 0n6804 ffffa389b4d180c0 svchost.exe(-p)                           3
0x64c  0n1612 ffffa389b276d0c0 uhssvc.exe                                3
0xf58  0n3928 ffffa389b3a98080 svchost.exe(TrkWks)                       3
0x790  0n1936 ffffa389b341c0c0 svchost.exe(vmictimesync)                 3
0x7c8  0n1992 ffffa389b343b080 svchost.exe(vmicvss)                      3
0x768  0n1896 ffffa389b33db080 svchost.exe                               3
0x778  0n1912 ffffa389b33eb080 svchost.exe                               3
0x488  0n1160 ffffa389b34a1080 svchost.exe                               3
0x560  0n1376 ffffa389b323e080 svchost.exe(TimeBrokerSvc)                3
0x96c  0n2412 ffffa389ae75d080 svchost.exe                               3
0x1454 0n5204 ffffa389b48af080 RuntimeBroker.exe                         4
0xe94  0n3732 ffffa389b3b22080 AnyDesk.exe*32                            4
0x8cc  0n2252 ffffa389b4cdf0c0 svchost.exe(UdkUserSvc)                   4
0x164c 0n5708 ffffa389b13ea080 svchost.exe(CDPUserSvc)                   4
0x2178 0n8568 ffffa389b1963080 svchost.exe(ScDeviceEnum)                 4
0x1700 0n5888 ffffa389b4705080 svchost.exe(WpnUserService)               4
0x20f0 0n8432 ffffa389afab70c0 notmyfault64.exe                          4
0x2424 0n9252 ffffa389af9570c0 rdpinput.exe                              4
0x1060 0n4192 ffffa389b3dcb080 sppsvc.exe                                4
0x13c8 0n5064 ffffa389b34a30c0 WmiPrvSE.exe                              4
0x1854 0n6228 ffffa389b61a70c0 svchost.exe(NPSMSvc)                      4
0x954  0n2388 ffffa389ae76e080 svchost.exe(SysMain)                      4
0x960  0n2400 ffffa389ae770080 svchost.exe(EventSystem)                  4
0x68   0n104  ffffa389ae6df080 Registry                                  4
0xb7c  0n2940 ffffa389b375f080 svchost.exe(WinHttpAutoProxySvc)          4
0x0    0n0    fffff80152348f40 Idle                                      4
0x1844 0n6212 ffffa389b1e1b080 svchost.exe(SSDPSRV)                      5
0x1344 0n4932 ffffa389b4ced080 svchost.exe(hns)                          5
0x628  0n1576 ffffa389b3351080 svchost.exe(nsi)                          5
0x19bc 0n6588 ffffa389af9e80c0 conhost.exe                               5
0x9ec  0n2540 ffffa389ae754080 svchost.exe(Dhcp)                         5
0x338  0n824  ffffa389b2eaf080 winlogon.exe                              5
0x14e0 0n5344 ffffa389afa540c0 winlogon.exe                              5
0x7c4  0n1988 ffffa389af7b60c0 fontdrvhost.exe                           5
0x25c  0n604  ffffa389b302f080 fontdrvhost.exe                           5
0x1960 0n6496 ffffa389b2ccf100 NisSrv.exe                                5
0x14f0 0n5360 ffffa389b29e40c0 svchost.exe(lfsvc)                        5
0x218  0n536  ffffa389b3037080 fontdrvhost.exe                           5
0xb40  0n2880 ffffa389b373e080 svchost.exe(FontCache)                    5
0xf44  0n3908 ffffa389b3aa9080 svchost.exe(LanmanServer)                 5
0xe78  0n3704 ffffa389b3b33080 svchost.exe(iphlpsvc)                     5
0xf9c  0n3996 ffffa389b3b66080 svchost.exe(WpnService)                   5
0x720  0n1824 ffffa389b33c7080 svchost.exe(CertPropSvc)                  6
0xcc4  0n3268 ffffa389b395c080 svchost.exe(ShellHWDetection)             6
0xbe8  0n3048 ffffa389b2a850c0 msteamsupdate.exe                         6
0x738  0n1848 ffffa389b33d0080 svchost.exe(vmicheartbeat)                6
0x744  0n1860 ffffa389b61df0c0 svchost.exe                               6
0x1bb8 0n7096 ffffa389b3004080 WidgetService.exe                         6
0x6b8  0n1720 ffffa389b33a0080 svchost.exe(UserManager)                  6
0x90c  0n2316 ffffa389b4fae080 RuntimeBroker.exe                         6
0x99c  0n2460 ffffa389ae78c080 svchost.exe(SessionEnv)                   6
0x175c 0n5980 ffffa389b47650c0 svchost.exe(TokenBroker)                  6
0x1414 0n5140 ffffa389b42c90c0 svchost.exe(webthreatdefsvc)              6
0x444  0n1092 ffffa389b2f9b080 svchost.exe(LSM)                          6
0x10b4 0n4276 ffffa389b4241100 svchost.exe(RmSvc)                        6
0x674  0n1652 ffffa389b335b080 svchost.exe(UmRdpService)                 7
0xb9c  0n2972 ffffa389b378a080 svchost.exe(StateRepository)              7
0xb64  0n2916 ffffa389b24ea0c0 msedgewebview2.exe                        7
0x26c4 0n9924 ffffa389b6094080 TabTip.exe                                7
0x1610 0n5648 ffffa389b476c080 svchost.exe(CDPSvc)                       7
0xe8c  0n3724 ffffa389b3b21080 svchost.exe(-p)                           7
0x1450 0n5200 ffffa389b3d33080 svchost.exe(camsvc)                       7
0x38c  0n908  ffffa389b2f09080 lsass.exe                                 8
0x1150 0n4432 ffffa389b1d110c0 MoUsoCoreWorker.exe                       8
0x17ac 0n6060 ffffa389b470a080 taskhostw.exe                             8
0x1b28 0n6952 ffffa389b4d86080 msedge.exe                                8
0x1914 0n6420 ffffa389b2b0b080 AnyDesk.exe*32                            8
0xbbc  0n3004 ffffa389b2cc80c0 svchost.exe(wscsvc)                       8
0x1d54 0n7508 ffffa389b3003080 msedgewebview2.exe                        8
0x368  0n872  ffffa389b2f08080 services.exe                              8
0xbfc  0n3068 ffffa389b37ea080 svchost.exe(-p)                           8
0xed0  0n3792 ffffa389b3c86080 IpOverUsbSvc.exe*32                       9
0xd40  0n3392 ffffa389b3a380c0 spoolsv.exe                               9
0x13e8 0n5096 ffffa389aee44080 SgrmBroker.exe                            9
0x924  0n2340 ffffa389ae796080 svchost.exe(EventLog)                     9
0x86c  0n2156 ffffa389ae719080 svchost.exe(-p)                           9
0xb68  0n2920 ffffa389b61da0c0 SecurityHealthService.exe                 9
0xd88  0n3464 ffffa389aee4c080 dllhost.exe                               9
0x1018 0n4120 ffffa389b13020c0 svchost.exe(UsoSvc)                      10
0xea8  0n3752 ffffa389b3b88080 svchost.exe(-p)                          10
0x10ec 0n4332 ffffa389b13290c0 msedge.exe                               10
0x1ca0 0n7328 ffffa389b25d40c0 msedgewebview2.exe                       11
0xf1c  0n3868 ffffa389b3d32080 svchost.exe(Winmgmt)                     11
0x18f8 0n6392 ffffa389b4782080 svchost.exe(netsvcs)                     11
0xf30  0n3888 ffffa389b3d44080 Sysmon.exe                               11
0x418  0n1048 ffffa389b2f5a080 svchost.exe(-p)                          11
0x26b8 0n9912 ffffa389b2e4b140 svchost.exe(DsmSvc)                      11
0x758  0n1880 ffffa389b48130c0 SearchIndexer.exe                        11
0x28c  0n652  ffffa389b29960c0 csrss.exe                                11
0x5cc  0n1484 ffffa389b32c1080 svchost.exe(Schedule)                    11
0x21e8 0n8680 ffffa389b2beb080 WUDFHost.exe                             11
0x1608 0n5640 ffffa389b4cf70c0 RuntimeBroker.exe                        12
0x1300 0n4864 ffffa389b31b00c0 ctfmon.exe                               12
0x75c  0n1884 ffffa389b33e3080 svchost.exe(vmicrdv)                     12
0x66c  0n1644 ffffa389b336e080 svchost.exe(netprofm)                    12
0xa34  0n2612 ffffa389b2dee140 csrss.exe                                12
0xc10  0n3088 ffffa389b385e080 svchost.exe(-p)                          13
0x2378 0n9080 ffffa389b1bdf080 rdpclip.exe                              13
0xaac  0n2732 ffffa389b4073080 svchost.exe(PcaSvc)                      13
0x2dc  0n732  ffffa389b2df1140 csrss.exe                                13
0x1b64 0n7012 ffffa389b4fb1080 RustDesk.exe                             14
0x1620 0n5664 ffffa389b45e6080 sihost.exe                               14
0x1e54 0n7764 ffffa389b24e30c0 msedgewebview2.exe                       14
0xdf4  0n3572 ffffa389b6210080 ShellExperienceHost.exe                  15
0x1670 0n5744 ffffa389b27680c0 msedge.exe                               15
0xcd8  0n3288 ffffa389b39ac080 svchost.exe(cbdhsvc)                     15
0x14a0 0n5280 ffffa389b49740c0 StartMenuExperienceHost.exe              15
0x4ec  0n1260 ffffa389b13070c0 msedge.exe                               16
0xc20  0n3104 ffffa389b3860080 svchost.exe(wuauserv)                    16
0xeb8  0n3768 ffffa389b3c75080 svchost.exe(DPS)                         16
0x7fc  0n2044 ffffa389b61af080 dwm.exe                                  17
0x2654 0n9812 ffffa389af7c40c0 msedge.exe                               17
0x4a0  0n1184 ffffa389b2d460c0 msedge.exe                               18
0x22c  0n556  ffffa389b141e080 svchost.exe(-p)                          18
0x1c98 0n7320 ffffa389b2a800c0 msedgewebview2.exe                       18
0xe28  0n3624 ffffa389b279f080 Widgets.exe                              18
0x2a0  0n672  ffffa389af9240c0 LogonUI.exe                              18
0x17b8 0n6072 ffffa389b4c9b080 svchost.exe(DisplayEnhancementService)   19
0xd4c  0n3404 ffffa389b2ad9080 TextInputHost.exe                        21
0xad8  0n2776 ffffa389b35b10c0 OneDrive.exe                             22
0x494  0n1172 ffffa389b3168080 dwm.exe                                  24
0xf7c  0n3964 ffffa389b3bee080 MsMpEng.exe                              29
0x4e4  0n1252 ffffa389b31e9080 svchost.exe(TermService)                 30
0x9fc  0n2556 ffffa389ae757040 MemCompression                           34
0xc2c  0n3116 ffffa389b24c7080 msedgewebview2.exe                       46
0x1968 0n6504 ffffa389b35b60c0 msedge.exe                               48
0x230  0n560  ffffa389b49760c0 SearchHost.exe                           60
0x2fc  0n764  ffffa389b47a3080 explorer.exe                             61
0x4    0n4    ffffa389ae6eb040 System                                  224
============= ================ ======================================= ===
PID           Address          Name                                    Thd
```

The **`!tl -t`** command examines the state and wait reasons for every thread across all processes, presenting the results as statistics. This allows for a quick overview, such as determining how many processes have 4 running threads, and so on. This is one of my favorite commands in **MEX**.

```
PID           Address          Name                                    !! Rn Ry Bk Lc IO Er
============= ================ ======================================= == == == == == == ==
0x4    0n4    ffffa389ae6eb040 System                                   4  .  .  .  .  1  .
0x68   0n104  ffffa389ae6df080 Registry                                 .  .  .  .  .  .  .
0x1f8  0n504  ffffa389b1713080 smss.exe                                 .  .  .  1  .  .  .
0x28c  0n652  ffffa389b29960c0 csrss.exe                                .  .  .  .  1  .  .
0x2d4  0n724  ffffa389b2def080 wininit.exe                              .  .  .  .  .  .  .
0x2dc  0n732  ffffa389b2df1140 csrss.exe                                .  .  .  .  1  .  .
0x338  0n824  ffffa389b2eaf080 winlogon.exe                             .  .  .  .  .  .  .
0x368  0n872  ffffa389b2f08080 services.exe                             .  .  .  .  .  .  .
0x370  0n880  ffffa389b29d30c0 LsaIso.exe                               .  .  .  .  .  .  .
0x38c  0n908  ffffa389b2f09080 lsass.exe                                .  .  .  .  .  .  .
0x22c  0n556  ffffa389b141e080 svchost.exe(-p)                          .  .  .  .  .  .  .
0x25c  0n604  ffffa389b302f080 fontdrvhost.exe                          .  .  .  .  .  .  .
0x218  0n536  ffffa389b3037080 fontdrvhost.exe                          .  .  .  .  .  .  .
0x418  0n1048 ffffa389b2f5a080 svchost.exe(-p)                          .  .  .  .  .  .  .
0x444  0n1092 ffffa389b2f9b080 svchost.exe(LSM)                         .  .  .  .  .  .  .
0x494  0n1172 ffffa389b3168080 dwm.exe                                  .  .  .  .  .  .  .
0x4e4  0n1252 ffffa389b31e9080 svchost.exe(TermService)                 .  .  .  .  .  .  .
0x508  0n1288 ffffa389b3210080 svchost.exe                              .  .  .  .  .  .  .
0x544  0n1348 ffffa389b323b0c0 svchost.exe(NcbService)                  .  .  .  .  .  .  .
0x550  0n1360 ffffa389b3240080 svchost.exe(-p)                          .  .  .  .  .  .  .
0x560  0n1376 ffffa389b323e080 svchost.exe(TimeBrokerSvc)               .  .  .  .  .  .  .
0x5cc  0n1484 ffffa389b32c1080 svchost.exe(Schedule)                    .  .  .  .  .  .  .
0x608  0n1544 ffffa389b32dc080 svchost.exe(ProfSvc)                     .  .  .  .  .  .  .
0x628  0n1576 ffffa389b3351080 svchost.exe(nsi)                         .  .  .  .  .  .  .
0x664  0n1636 ffffa389b336b080 svchost.exe(DispBrokerDesktopSvc)        .  .  .  .  .  .  .
0x66c  0n1644 ffffa389b336e080 svchost.exe(netprofm)                    .  .  .  .  1  .  .
0x674  0n1652 ffffa389b335b080 svchost.exe(UmRdpService)                .  .  .  .  .  .  .
0x6b8  0n1720 ffffa389b33a0080 svchost.exe(UserManager)                 .  .  .  .  .  .  .
0x720  0n1824 ffffa389b33c7080 svchost.exe(CertPropSvc)                 .  .  .  .  .  .  .
0x738  0n1848 ffffa389b33d0080 svchost.exe(vmicheartbeat)               .  .  .  .  .  .  .
0x75c  0n1884 ffffa389b33e3080 svchost.exe(vmicrdv)                     .  .  .  .  .  .  .
0x768  0n1896 ffffa389b33db080 svchost.exe                              .  .  .  .  .  .  .
0x778  0n1912 ffffa389b33eb080 svchost.exe                              .  .  .  .  .  .  .
0x790  0n1936 ffffa389b341c0c0 svchost.exe(vmictimesync)                .  .  .  .  .  .  .
0x7c8  0n1992 ffffa389b343b080 svchost.exe(vmicvss)                     .  .  .  .  .  .  .
0x488  0n1160 ffffa389b34a1080 svchost.exe                              .  .  .  .  .  .  .
0x86c  0n2156 ffffa389ae719080 svchost.exe(-p)                          .  .  .  .  .  .  .
0x8e4  0n2276 ffffa389b349c080 VSSVC.exe                                .  .  .  .  .  .  .
0x924  0n2340 ffffa389ae796080 svchost.exe(EventLog)                    .  .  .  .  .  .  .
0x954  0n2388 ffffa389ae76e080 svchost.exe(SysMain)                     .  .  .  .  .  .  .
0x960  0n2400 ffffa389ae770080 svchost.exe(EventSystem)                 .  .  .  .  .  .  .
0x96c  0n2412 ffffa389ae75d080 svchost.exe                              .  .  .  .  .  .  .
0x99c  0n2460 ffffa389ae78c080 svchost.exe(SessionEnv)                  .  .  .  .  .  .  .
0x9ec  0n2540 ffffa389ae754080 svchost.exe(Dhcp)                        .  .  .  .  .  .  .
0x9fc  0n2556 ffffa389ae757040 MemCompression                           .  .  .  .  .  .  .
0xae8  0n2792 ffffa389b368b080 svchost.exe(SENS)                        .  .  .  .  .  .  .
0xb24  0n2852 ffffa389b3708080 svchost.exe(AudioEndpointBuilder)        .  .  .  .  .  .  .
0xb40  0n2880 ffffa389b373e080 svchost.exe(FontCache)                   .  .  .  .  .  .  .
0xb7c  0n2940 ffffa389b375f080 svchost.exe(WinHttpAutoProxySvc)         .  .  .  .  .  .  .
0xb9c  0n2972 ffffa389b378a080 svchost.exe(StateRepository)             .  .  .  .  .  .  .
0xbfc  0n3068 ffffa389b37ea080 svchost.exe(-p)                          .  .  .  .  .  .  .
0xaf4  0n2804 ffffa389b37ed080 svchost.exe(TextInputManagementService)  .  .  .  .  .  .  .
0xc10  0n3088 ffffa389b385e080 svchost.exe(-p)                          .  .  .  .  .  .  .
0xc20  0n3104 ffffa389b3860080 svchost.exe(wuauserv)                    .  .  .  .  .  .  .
0xc60  0n3168 ffffa389b3905140 svchost.exe(-p)                          .  .  .  .  .  .  .
0xc70  0n3184 ffffa389b39020c0 svchost.exe(-p)                          .  .  .  .  .  .  .
0xcc4  0n3268 ffffa389b395c080 svchost.exe(ShellHWDetection)            .  .  .  .  .  .  .
0xd40  0n3392 ffffa389b3a380c0 spoolsv.exe                              .  .  .  .  .  .  .
0xd9c  0n3484 ffffa389b3b9b0c0 svchost.exe                              .  .  .  .  .  .  .
0xe78  0n3704 ffffa389b3b33080 svchost.exe(iphlpsvc)                    .  .  .  .  .  .  .
0xe84  0n3716 ffffa389b3b20080 svchost.exe                              .  .  .  .  .  .  .
0xe8c  0n3724 ffffa389b3b21080 svchost.exe(-p)                          .  .  .  .  .  .  .
0xe94  0n3732 ffffa389b3b22080 AnyDesk.exe*32                           .  .  .  .  .  .  .
0xea8  0n3752 ffffa389b3b88080 svchost.exe(-p)                          .  .  .  .  .  .  .
0xeb8  0n3768 ffffa389b3c75080 svchost.exe(DPS)                         .  .  .  .  .  .  .
0xed0  0n3792 ffffa389b3c86080 IpOverUsbSvc.exe*32                      .  .  .  .  .  .  .
0xf1c  0n3868 ffffa389b3d32080 svchost.exe(Winmgmt)                     .  .  .  .  .  .  .
0xf24  0n3876 ffffa389b3d22080 sqlwriter.exe                            .  .  .  .  .  .  .
0xf3c  0n3900 ffffa389b3ca8080 RustDesk.exe                             .  .  .  .  .  .  .
0xf44  0n3908 ffffa389b3aa9080 svchost.exe(LanmanServer)                .  .  .  .  .  .  .
0xf58  0n3928 ffffa389b3a98080 svchost.exe(TrkWks)                      .  .  .  .  .  .  .
0xf7c  0n3964 ffffa389b3bee080 MsMpEng.exe                              .  .  .  .  .  .  .
0xf84  0n3972 ffffa389b3b77080 wlms.exe                                 .  .  .  .  .  .  .
0xf9c  0n3996 ffffa389b3b66080 svchost.exe(WpnService)                  .  .  .  .  .  .  .
0x1060 0n4192 ffffa389b3dcb080 sppsvc.exe                               .  .  .  .  .  .  .
0x1174 0n4468 ffffa389b406e080 unsecapp.exe                             .  .  .  .  .  .  .
0x12dc 0n4828 ffffa389b406c080 AggregatorHost.exe                       .  .  .  .  .  .  .
0x13c8 0n5064 ffffa389b34a30c0 WmiPrvSE.exe                             .  .  .  .  .  .  .
0x10b4 0n4276 ffffa389b4241100 svchost.exe(RmSvc)                       .  .  .  .  .  .  .
0x1414 0n5140 ffffa389b42c90c0 svchost.exe(webthreatdefsvc)             .  .  .  .  .  .  .
0x1574 0n5492 ffffa389b13f2080 svchost.exe(StorSvc)                     .  .  .  .  .  .  .
0x1620 0n5664 ffffa389b45e6080 sihost.exe                               .  .  .  .  .  .  .
0x164c 0n5708 ffffa389b13ea080 svchost.exe(CDPUserSvc)                  .  .  .  .  .  .  .
0x1694 0n5780 ffffa389b45b3080 svchost.exe(webthreatdefusersvc)         .  .  .  .  .  .  .
0x1700 0n5888 ffffa389b4705080 svchost.exe(WpnUserService)              .  .  .  .  .  .  .
0x175c 0n5980 ffffa389b47650c0 svchost.exe(TokenBroker)                 .  .  .  .  .  .  .
0x17ac 0n6060 ffffa389b470a080 taskhostw.exe                            .  .  .  .  .  .  .
0x1610 0n5648 ffffa389b476c080 svchost.exe(CDPSvc)                      .  .  .  .  .  .  .
0x2fc  0n764  ffffa389b47a3080 explorer.exe                             1  .  .  .  1  .  .
0x1300 0n4864 ffffa389b31b00c0 ctfmon.exe                               .  .  .  .  .  .  .
0xcd8  0n3288 ffffa389b39ac080 svchost.exe(cbdhsvc)                     .  .  .  .  .  .  .
0x1450 0n5200 ffffa389b3d33080 svchost.exe(camsvc)                      .  .  .  .  .  .  .
0xbf4  0n3060 ffffa389b481e080 svchost.exe(Appinfo)                     .  .  .  .  .  .  .
0x758  0n1880 ffffa389b48130c0 SearchIndexer.exe                        .  .  .  .  .  .  .
0x230  0n560  ffffa389b49760c0 SearchHost.exe                           6  .  .  .  2  .  .
0x14a0 0n5280 ffffa389b49740c0 StartMenuExperienceHost.exe              .  .  .  .  .  .  .
0x1608 0n5640 ffffa389b4cf70c0 RuntimeBroker.exe                        .  .  .  .  1  .  .
0x1454 0n5204 ffffa389b48af080 RuntimeBroker.exe                        .  .  .  .  .  .  .
0x8cc  0n2252 ffffa389b4cdf0c0 svchost.exe(UdkUserSvc)                  .  .  .  .  .  .  .
0xaac  0n2732 ffffa389b4073080 svchost.exe(PcaSvc)                      .  .  .  .  .  .  .
0xcd0  0n3280 ffffa389b4d27080 svchost.exe(UnistackSvcGroup)            .  .  .  .  .  .  .
0xd88  0n3464 ffffa389aee4c080 dllhost.exe                              .  .  .  .  .  .  .
0xe2c  0n3628 ffffa389aee45080 SppExtComObj.Exe                         .  .  .  .  .  .  .
0x13e8 0n5096 ffffa389aee44080 SgrmBroker.exe                           .  .  .  .  .  .  .
0x4b0  0n1200 ffffa389b605b080 SecurityHealthSystray.exe                .  .  .  .  .  .  .
0xb68  0n2920 ffffa389b61da0c0 SecurityHealthService.exe                .  .  .  .  .  .  .
0x744  0n1860 ffffa389b61df0c0 svchost.exe                              .  .  .  .  .  .  .
0xad8  0n2776 ffffa389b35b10c0 OneDrive.exe                             .  .  .  .  .  .  .
0x1344 0n4932 ffffa389b4ced080 svchost.exe(hns)                         .  .  .  .  .  .  .
0x1844 0n6212 ffffa389b1e1b080 svchost.exe(SSDPSRV)                     .  .  .  .  .  .  .
0x1968 0n6504 ffffa389b35b60c0 msedge.exe                               .  .  .  .  .  .  .
0x1a94 0n6804 ffffa389b4d180c0 svchost.exe(-p)                          .  .  .  .  .  .  .
0x1b28 0n6952 ffffa389b4d86080 msedge.exe                               .  .  .  .  .  .  .
0x1914 0n6420 ffffa389b2b0b080 AnyDesk.exe*32                           .  .  .  .  .  .  .
0x4a0  0n1184 ffffa389b2d460c0 msedge.exe                               .  .  .  .  .  .  .
0x4ec  0n1260 ffffa389b13070c0 msedge.exe                               .  .  .  .  .  .  .
0x10ec 0n4332 ffffa389b13290c0 msedge.exe                               .  .  .  .  .  .  .
0x64c  0n1612 ffffa389b276d0c0 uhssvc.exe                               .  .  .  .  .  .  .
0xbe8  0n3048 ffffa389b2a850c0 msteamsupdate.exe                        .  .  .  .  .  .  .
0x1670 0n5744 ffffa389b27680c0 msedge.exe                               .  .  .  .  .  .  .
0x1018 0n4120 ffffa389b13020c0 svchost.exe(UsoSvc)                      .  .  .  .  .  .  .
0xdf4  0n3572 ffffa389b6210080 ShellExperienceHost.exe                  6  .  .  .  .  .  .
0xe28  0n3624 ffffa389b279f080 Widgets.exe                              .  .  .  .  .  .  .
0x1bb8 0n7096 ffffa389b3004080 WidgetService.exe                        .  .  .  .  .  .  .
0x1960 0n6496 ffffa389b2ccf100 NisSrv.exe                               .  .  .  .  .  .  .
0xc2c  0n3116 ffffa389b24c7080 msedgewebview2.exe                       .  .  .  .  .  .  .
0xbbc  0n3004 ffffa389b2cc80c0 svchost.exe(wscsvc)                      .  .  .  .  .  .  .
0xb64  0n2916 ffffa389b24ea0c0 msedgewebview2.exe                       .  .  .  .  .  .  .
0x1c98 0n7320 ffffa389b2a800c0 msedgewebview2.exe                       .  .  .  .  .  .  .
0x1ca0 0n7328 ffffa389b25d40c0 msedgewebview2.exe                       .  .  .  .  .  .  .
0x1d54 0n7508 ffffa389b3003080 msedgewebview2.exe                       .  .  .  .  .  .  .
0x1e54 0n7764 ffffa389b24e30c0 msedgewebview2.exe                       .  .  .  .  .  .  .
0x14f0 0n5360 ffffa389b29e40c0 svchost.exe(lfsvc)                       .  .  .  .  .  .  .
0x1150 0n4432 ffffa389b1d110c0 MoUsoCoreWorker.exe                      .  .  .  .  .  .  .
0x1854 0n6228 ffffa389b61a70c0 svchost.exe(NPSMSvc)                     .  .  .  .  .  .  .
0x90c  0n2316 ffffa389b4fae080 RuntimeBroker.exe                        .  .  .  .  2  .  .
0x1b34 0n6964 ffffa389b61c3080 SystemSettingsBroker.exe                 .  .  .  .  .  .  .
0xfcc  0n4044 ffffa389b3d460c0 svchost.exe(WdiSystemHost)               .  .  .  .  .  .  .
0x9b4  0n2484 ffffa389af7950c0 svchost.exe(SstpSvc)                     .  .  .  .  .  .  .
0x18f8 0n6392 ffffa389b4782080 svchost.exe(netsvcs)                     .  .  .  .  .  .  .
0x17b8 0n6072 ffffa389b4c9b080 svchost.exe(DisplayEnhancementService)   .  .  .  .  .  .  .
0xa34  0n2612 ffffa389b2dee140 csrss.exe                                .  .  .  .  1  .  .
0x14e0 0n5344 ffffa389afa540c0 winlogon.exe                             .  .  .  .  1  .  .
0x7c4  0n1988 ffffa389af7b60c0 fontdrvhost.exe                          .  .  .  .  .  .  .
0x2a0  0n672  ffffa389af9240c0 LogonUI.exe                              .  .  .  .  .  .  .
0x7fc  0n2044 ffffa389b61af080 dwm.exe                                  .  .  .  .  .  .  .
0x1b64 0n7012 ffffa389b4fb1080 RustDesk.exe                             .  .  .  .  .  .  .
0x21e8 0n8680 ffffa389b2beb080 WUDFHost.exe                             .  .  .  .  .  .  .
0x2378 0n9080 ffffa389b1bdf080 rdpclip.exe                              .  .  .  .  .  .  .
0x2178 0n8568 ffffa389b1963080 svchost.exe(ScDeviceEnum)                .  .  .  .  .  .  .
0x2424 0n9252 ffffa389af9570c0 rdpinput.exe                             .  .  .  .  .  .  .
0x26c4 0n9924 ffffa389b6094080 TabTip.exe                               .  .  .  .  .  .  .
0xd4c  0n3404 ffffa389b2ad9080 TextInputHost.exe                        .  .  .  .  .  .  .
0x2654 0n9812 ffffa389af7c40c0 msedge.exe                               .  .  .  .  .  .  .
0x1890 0n6288 ffffa389af9d10c0 cmd.exe                                  .  .  .  .  .  .  .
0x19bc 0n6588 ffffa389af9e80c0 conhost.exe                              .  .  .  .  .  .  .
0x26b8 0n9912 ffffa389b2e4b140 svchost.exe(DsmSvc)                      .  .  .  .  .  .  .
0xf30  0n3888 ffffa389b3d44080 Sysmon.exe                               .  1  .  .  .  .  .
0x20f0 0n8432 ffffa389afab70c0 notmyfault64.exe                         .  1  .  .  .  .  .
0x0    0n0    fffff80152348f40 Idle                                     .  3  5  .  .  .  .
============= ================ ======================================= == == == == == == ==
PID           Address          Name                                    !! Rn Ry Bk Lc IO Er
```

# Call stacks

Call stacks are sequences of function calls that show how a program arrived at a particular point. They are essential in debugging because they trace the path of execution, which can help us to identify where problems may occur.

The **`!mex.us`** command allows us to display unique stacks within threads. First, we need to navigate to a specific process of interest. In this case, we'll use lsass.exe as our example. Use the **`!tl`** command to list the memory address.

```
0: kd> !tl lsass
PID         Address          Name
=========== ================ =========
0x38c 0n908 ffffa389b2f09080 lsass.exe
=========== ================ =========
PID         Address          Name
```

Let's now switch to the process context of **lsass.exe** with the **`!p`** command.

```
0: kd> !mex.p ffffa389b2f09080
Name      Address                  Ses PID         Parent      PEB              Create Time                Mods Handle Thrd User Name
========= ======================== === =========== =========== ================ ========================== ==== ====== ==== =========================
lsass.exe ffffa389b2f09080 (E|K|O)   0 38c (0n908) 2d4 (0n724) 0000005b6c0e6000 08/13/2023 04:19:06.206 AM   88      0    8 WORKGROUP\WINDEV2305EVAL$

Command Line: C:\Windows\system32\lsass.exe

Memory Details:

    VM   Peak Commit Size PP Quota  NPP Quota
    ==== ==== =========== ========= =========
    2 TB 2 TB     8.35 MB 156.84 KB  26.16 KB

Show LPC Port information for process

Show Threads: Unique Stacks    !mex.listthreads (!lt) ffffa389b2f09080    !process ffffa389b2f09080 7
```

We can now run the **!mex.us** command to list the call stacks that resides within the threads. 

The data shows the activity of **8** threads in total. Of these, **4** unique call stacks are identified. The first **3** call stacks each belong to a **single thread**, while the **4rd** call stack is **shared among 4 threads**, suggesting they are performing similar tasks. An additional note indicates that there's a stack which couldn't be displayed due to context switching issues or an empty trace.

```
0: kd> !us
1 thread [stats]: ffffa389b2f06080
    fffff80151a3a756 nt!KiSwapContext+0x76
    fffff8015185e095 nt!KiSwapThread+0xae5
    fffff8015185c3c4 nt!KiCommitThreadWait+0x134
    fffff8015185b746 nt!KeWaitForSingleObject+0x256
    fffff801518f8f2a nt!AlpcpWaitForSingleObject+0x3e
    fffff80151d40b0f nt!AlpcpCompleteDeferSignalRequestAndWait+0x57
    fffff80151dc1fb3 nt!AlpcpReceiveMessagePort+0x323
    fffff80151c8af03 nt!AlpcpReceiveLegacyMessage+0x103
    fffff80151c8ad6c nt!NtReplyWaitReceivePortEx+0xcc
    fffff80151c8ac8f nt!NtReplyWaitReceivePort+0xf
    fffff80151a45fe5 nt!KiSystemServiceCopyEnd+0x25
    00007ff8346eeeb4 ntdll!NtReplyWaitReceivePort+0x14
    00007ff678db4786 lsass!LsapRmServerThread+0x66
    00007ff8325b26ad KERNEL32!BaseThreadInitThunk+0x1d
    00007ff8346aaa68 ntdll!RtlUserThreadStart+0x28

1 thread [stats]: ffffa389b2f0c080
    fffff80151a3a756 nt!KiSwapContext+0x76
    fffff8015185e095 nt!KiSwapThread+0xae5
    fffff8015185c3c4 nt!KiCommitThreadWait+0x134
    fffff8015185b746 nt!KeWaitForSingleObject+0x256
    fffff80151cc47cc nt!ObWaitForSingleObject+0xcc
    fffff80151cc46be nt!NtWaitForSingleObject+0x3e
    fffff80151a45fe5 nt!KiSystemServiceCopyEnd+0x25
    00007ff8346eedd4 ntdll!NtWaitForSingleObject+0x14
    00007ff831f43f8e KERNELBASE!WaitForSingleObjectEx+0x8e
    00007ff83456b4e9 sechost!ScSendResponseReceiveControls+0x149
    00007ff83456add9 sechost!ScDispatcherLoop+0x11d
    00007ff8345683bb sechost!StartServiceCtrlDispatcherW+0x6b
    00007ff8314ff6c3 lsasrv!ServiceDispatcherThread+0x1b3

1 thread [stats]: ffffa389b45be080
    fffff80151a3a756 nt!KiSwapContext+0x76
    fffff8015185e095 nt!KiSwapThread+0xae5
    fffff8015185c3c4 nt!KiCommitThreadWait+0x134
    fffff8015185b746 nt!KeWaitForSingleObject+0x256
    fffff8015194b830 nt!KeWaitForMultipleObjects+0x640
    fffff80151cc4cf4 nt!ObWaitForMultipleObjects+0x2e4
    fffff80151d286bc nt!NtWaitForMultipleObjects+0x11c
    fffff80151a45fe5 nt!KiSystemServiceCopyEnd+0x25
    00007ff8346ef8a4 ntdll!NtWaitForMultipleObjects+0x14
    00007ff831f6f5a9 KERNELBASE!WaitForMultipleObjectsEx+0xe9
    00007ff831f6f4ae KERNELBASE!WaitForMultipleObjects+0xe
    00007ff81c16323d SecureTimeAggregator!ComputeSecureTimeBackground+0xcd
    00007ff832371503 ucrtbase!thread_start<void (__cdecl*)(void *),0>+0x93
    00007ff8325b26ad KERNEL32!BaseThreadInitThunk+0x1d
    00007ff8346aaa68 ntdll!RtlUserThreadStart+0x28

4 threads [stats]: ffffa389b2f11080 ffffa389b497e080 ffffa389b61b3040 ffffa389b30eb040
    fffff80151a3a756 nt!KiSwapContext+0x76
    fffff8015185e095 nt!KiSwapThread+0xae5
    fffff8015185c3c4 nt!KiCommitThreadWait+0x134
    fffff8015193102d nt!KeRemoveQueueEx+0x111d
    fffff8015192fcc8 nt!IoRemoveIoCompletion+0x98
    fffff8015192f521 nt!NtWaitForWorkViaWorkerFactory+0x381
    fffff80151a45fe5 nt!KiSystemServiceCopyEnd+0x25
    00007ff8346f29a4 ntdll!NtWaitForWorkViaWorkerFactory+0x14
    00007ff83468536e ntdll!TppWorkerThread+0x2ee
    00007ff8325b26ad KERNEL32!BaseThreadInitThunk+0x1d
    00007ff8346aaa68 ntdll!RtlUserThreadStart+0x28

4 stack(s) with 7 threads displayed (8 Total threads)
1 stack(s) were not displayed because we could not switch to thread context, or stack trace was empty
```

With the **!us** command we can also filter on a specific string, so it will only display the the call stacks that contains a specific keyword. Here is an example where I'm looking for call stacks that contains "lsass".

```
0: kd> !us lsass
1 thread [stats]: ffffa389b2f06080
    fffff80151a3a756 nt!KiSwapContext+0x76
    fffff8015185e095 nt!KiSwapThread+0xae5
    fffff8015185c3c4 nt!KiCommitThreadWait+0x134
    fffff8015185b746 nt!KeWaitForSingleObject+0x256
    fffff801518f8f2a nt!AlpcpWaitForSingleObject+0x3e
    fffff80151d40b0f nt!AlpcpCompleteDeferSignalRequestAndWait+0x57
    fffff80151dc1fb3 nt!AlpcpReceiveMessagePort+0x323
    fffff80151c8af03 nt!AlpcpReceiveLegacyMessage+0x103
    fffff80151c8ad6c nt!NtReplyWaitReceivePortEx+0xcc
    fffff80151c8ac8f nt!NtReplyWaitReceivePort+0xf
    fffff80151a45fe5 nt!KiSystemServiceCopyEnd+0x25
    00007ff8346eeeb4 ntdll!NtReplyWaitReceivePort+0x14
    00007ff678db4786 lsass!LsapRmServerThread+0x66
    00007ff8325b26ad KERNEL32!BaseThreadInitThunk+0x1d
    00007ff8346aaa68 ntdll!RtlUserThreadStart+0x28

Threads matching filter: 1 out of 7
```

Occasionally, a call stack might be shared across **4** threads. These threads are arranged based on their runtime duration, with the longest-running thread being **ffffa389b2f11080**, followed by **ffffa389b497e080**, and so forth.

```
4 threads [stats]: ffffa389b2f11080 ffffa389b497e080 ffffa389b61b3040 ffffa389b30eb040
    fffff80151a3a756 nt!KiSwapContext+0x76
    fffff8015185e095 nt!KiSwapThread+0xae5
    fffff8015185c3c4 nt!KiCommitThreadWait+0x134
    fffff8015193102d nt!KeRemoveQueueEx+0x111d
    fffff8015192fcc8 nt!IoRemoveIoCompletion+0x98
    fffff8015192f521 nt!NtWaitForWorkViaWorkerFactory+0x381
    fffff80151a45fe5 nt!KiSystemServiceCopyEnd+0x25
    00007ff8346f29a4 ntdll!NtWaitForWorkViaWorkerFactory+0x14
    00007ff83468536e ntdll!TppWorkerThread+0x2ee
    00007ff8325b26ad KERNEL32!BaseThreadInitThunk+0x1d
    00007ff8346aaa68 ntdll!RtlUserThreadStart+0x28
```

We can use the **`-e`** option within the **`!us`** command to exclude a specific keyword, so every time when we run the command. It will exclude the call stack that contains the specified keyword.

```
0: kd> !us -e TppWorkerThread
1 thread [stats]: ffffa389b2f06080
    fffff80151a3a756 nt!KiSwapContext+0x76
    fffff8015185e095 nt!KiSwapThread+0xae5
    fffff8015185c3c4 nt!KiCommitThreadWait+0x134
    fffff8015185b746 nt!KeWaitForSingleObject+0x256
    fffff801518f8f2a nt!AlpcpWaitForSingleObject+0x3e
    fffff80151d40b0f nt!AlpcpCompleteDeferSignalRequestAndWait+0x57
    fffff80151dc1fb3 nt!AlpcpReceiveMessagePort+0x323
    fffff80151c8af03 nt!AlpcpReceiveLegacyMessage+0x103
    fffff80151c8ad6c nt!NtReplyWaitReceivePortEx+0xcc
    fffff80151c8ac8f nt!NtReplyWaitReceivePort+0xf
    fffff80151a45fe5 nt!KiSystemServiceCopyEnd+0x25
    00007ff8346eeeb4 ntdll!NtReplyWaitReceivePort+0x14
    00007ff678db4786 lsass!LsapRmServerThread+0x66
    00007ff8325b26ad KERNEL32!BaseThreadInitThunk+0x1d
    00007ff8346aaa68 ntdll!RtlUserThreadStart+0x28

1 thread [stats]: ffffa389b2f0c080
    fffff80151a3a756 nt!KiSwapContext+0x76
    fffff8015185e095 nt!KiSwapThread+0xae5
    fffff8015185c3c4 nt!KiCommitThreadWait+0x134
    fffff8015185b746 nt!KeWaitForSingleObject+0x256
    fffff80151cc47cc nt!ObWaitForSingleObject+0xcc
    fffff80151cc46be nt!NtWaitForSingleObject+0x3e
    fffff80151a45fe5 nt!KiSystemServiceCopyEnd+0x25
    00007ff8346eedd4 ntdll!NtWaitForSingleObject+0x14
    00007ff831f43f8e KERNELBASE!WaitForSingleObjectEx+0x8e
    00007ff83456b4e9 sechost!ScSendResponseReceiveControls+0x149
    00007ff83456add9 sechost!ScDispatcherLoop+0x11d
    00007ff8345683bb sechost!StartServiceCtrlDispatcherW+0x6b
    00007ff8314ff6c3 lsasrv!ServiceDispatcherThread+0x1b3

1 thread [stats]: ffffa389b45be080
    fffff80151a3a756 nt!KiSwapContext+0x76
    fffff8015185e095 nt!KiSwapThread+0xae5
    fffff8015185c3c4 nt!KiCommitThreadWait+0x134
    fffff8015185b746 nt!KeWaitForSingleObject+0x256
    fffff8015194b830 nt!KeWaitForMultipleObjects+0x640
    fffff80151cc4cf4 nt!ObWaitForMultipleObjects+0x2e4
    fffff80151d286bc nt!NtWaitForMultipleObjects+0x11c
    fffff80151a45fe5 nt!KiSystemServiceCopyEnd+0x25
    00007ff8346ef8a4 ntdll!NtWaitForMultipleObjects+0x14
    00007ff831f6f5a9 KERNELBASE!WaitForMultipleObjectsEx+0xe9
    00007ff831f6f4ae KERNELBASE!WaitForMultipleObjects+0xe
    00007ff81c16323d SecureTimeAggregator!ComputeSecureTimeBackground+0xcd
    00007ff832371503 ucrtbase!thread_start<void (__cdecl*)(void *),0>+0x93
    00007ff8325b26ad KERNEL32!BaseThreadInitThunk+0x1d
    00007ff8346aaa68 ntdll!RtlUserThreadStart+0x28

Threads matching filter: 3 out of 7
1 stack(s) were not displayed because we could not switch to thread context, or stack trace was empty
```

Let's say that we are in the context of a process that has a lot of call stacks, but we want to do some strict filtering. We can use the **`-r`** option within the **`!us`** command. This will treat the filter as a regular expression. As we can see here, there are **9** call stacks. Let's now narrow down the results.

```
0: kd> !us -p ffffa389b31e9080
1 thread [stats]: ffffa389b323f540
    fffff80151a3a756 nt!KiSwapContext+0x76                              
    fffff8015185e095 nt!KiSwapThread+0xae5                              
    fffff8015185c3c4 nt!KiCommitThreadWait+0x134                        
    fffff8015185b746 nt!KeWaitForSingleObject+0x256                     
    fffff8015194b830 nt!KeWaitForMultipleObjects+0x640                  
    fffff80151cc4cf4 nt!ObWaitForMultipleObjects+0x2e4                  
    fffff80151d286bc nt!NtWaitForMultipleObjects+0x11c                  
    fffff80151a45fe5 nt!KiSystemServiceCopyEnd+0x25                     
    00007ff8346ef8a4 ntdll!NtWaitForMultipleObjects+0x14                
    00007ff831f6f5a9 KERNELBASE!WaitForMultipleObjectsEx+0xe9           
    00007ff83299b35d combase!WaitCoalesced+0xa9                         
    00007ff83299b1ca combase!CROIDTable::WorkerThreadLoop+0x5a          
    00007ff83299afcb combase!CRpcThread::WorkerLoop+0x57                
    00007ff83299af49 combase!CRpcThreadCache::RpcWorkerThreadEntry+0x29 
    00007ff8325b26ad KERNEL32!BaseThreadInitThunk+0x1d                  
    00007ff8346aaa68 ntdll!RtlUserThreadStart+0x28                      

1 thread [stats]: ffffa389b32be0c0
    fffff80151a3a756 nt!KiSwapContext+0x76
    fffff8015185e095 nt!KiSwapThread+0xae5
    fffff8015185c3c4 nt!KiCommitThreadWait+0x134
    fffff8015194b4d2 nt!KeWaitForMultipleObjects+0x2e2
    fffff80151cc4cf4 nt!ObWaitForMultipleObjects+0x2e4
    fffff80151d286bc nt!NtWaitForMultipleObjects+0x11c
    fffff80151a45fe5 nt!KiSystemServiceCopyEnd+0x25
    00007ff8346ef8a4 ntdll!NtWaitForMultipleObjects+0x14
    00007ff831f6f5a9 KERNELBASE!WaitForMultipleObjectsEx+0xe9
    00007ff831f6f4ae KERNELBASE!WaitForMultipleObjects+0xe
    00007ff82ccdf42c termsrv!CProtocolExMgr::NotificationWorker+0x3c
    00007ff8325b26ad KERNEL32!BaseThreadInitThunk+0x1d
    00007ff8346aaa68 ntdll!RtlUserThreadStart+0x28

1 thread [stats]: ffffa389b331b080
    fffff80151a3a756 nt!KiSwapContext+0x76
    fffff8015185e095 nt!KiSwapThread+0xae5
    fffff8015185c3c4 nt!KiCommitThreadWait+0x134
    fffff8015185b746 nt!KeWaitForSingleObject+0x256
    fffff8015194b830 nt!KeWaitForMultipleObjects+0x640
    ffff8f189c171dc9 win32kfull!xxxRealSleepThread+0x2c9
    ffff8f189c171a53 win32kfull!xxxSleepThread2+0xb3
    ffff8f189c16fee8 win32kfull!xxxRealInternalGetMessage+0x1078
    ffff8f189c16ec50 win32kfull!NtUserGetMessage+0x90
    ffff8f189c6a65e2 win32k!NtUserGetMessage+0x16
    fffff80151a45fe5 nt!KiSystemServiceCopyEnd+0x25
    00007ff8322c1534 win32u!NtUserGetMessage+0x14
    00007ff832f8532a user32!GetMessageW+0x2a
    00007ff82c243e55 rdpcorets!CTiNotify::staticThreadProc+0x55
    00007ff8325b26ad KERNEL32!BaseThreadInitThunk+0x1d
    00007ff8346aaa68 ntdll!RtlUserThreadStart+0x28

1 thread [stats]: ffffa389b3361080
    fffff80151a3a756 nt!KiSwapContext+0x76
    fffff8015185e095 nt!KiSwapThread+0xae5
    fffff8015185c3c4 nt!KiCommitThreadWait+0x134
    fffff8015194b4d2 nt!KeWaitForMultipleObjects+0x2e2
    fffff80151cc4cf4 nt!ObWaitForMultipleObjects+0x2e4
    ffff8f189c223f4b win32kfull!xxxMsgWaitForMultipleObjectsEx+0xd3
    ffff8f189c1da887 win32kfull!NtUserMsgWaitForMultipleObjectsEx+0x3d7
    ffff8f189c6a7514 win32k!NtUserMsgWaitForMultipleObjectsEx+0x20
    fffff80151a45fe5 nt!KiSystemServiceCopyEnd+0x25
    00007ff8322cacf4 win32u!NtUserMsgWaitForMultipleObjectsEx+0x14
    00007ff82c0906fa RDPBASE!PAL_System_CondWait+0x1ca
    00007ff82c09049d RDPBASE!CTSThreadInternal::ThreadSignalWait+0x5d
    00007ff82c070bec RDPBASE!CTSThread::internalThreadMsgLoop+0xcc
    00007ff82c070731 RDPBASE!CTSThread::internalThreadWaitForMultipleObjects+0xe1
    00007ff82c0702fe RDPBASE!CTSThread::ThreadWaitForMultipleObjects+0x8e
    00007ff82c245ec0 rdpcorets!CPipeVCMgr::PipeVCMgrThreadProc+0xa0
    00007ff82c245dee rdpcorets!CPipeVCMgr::STATIC_PipeVCMgrThreadProc+0xe
    00007ff82c0fab31 RDPBASE!CTSThread::TSStaticThreadEntry+0x2b1
    00007ff82c101d9c RDPBASE!PAL_System_Win32_ThreadProcWrapper+0x3c
    00007ff8325b26ad KERNEL32!BaseThreadInitThunk+0x1d
    00007ff8346aaa68 ntdll!RtlUserThreadStart+0x28

1 thread [stats]: ffffa389b604f080
    fffff80151a3a756 nt!KiSwapContext+0x76
    fffff8015185e095 nt!KiSwapThread+0xae5
    fffff8015185c3c4 nt!KiCommitThreadWait+0x134
    fffff8015194b4d2 nt!KeWaitForMultipleObjects+0x2e2
    fffff80151cc4cf4 nt!ObWaitForMultipleObjects+0x2e4
    fffff80151d286bc nt!NtWaitForMultipleObjects+0x11c
    fffff80151a45fe5 nt!KiSystemServiceCopyEnd+0x25
    00007ff8346ef8a4 ntdll!NtWaitForMultipleObjects+0x14
    00007ff831f6f5a9 KERNELBASE!WaitForMultipleObjectsEx+0xe9
    00007ff831f6f4ae KERNELBASE!WaitForMultipleObjects+0xe
    00007ff82bec3015 RDPSERVERBASE!CRDPWDUMXStack::WDShareReconnectionWorker+0x81
    00007ff82bebbfb5 RDPSERVERBASE!CRDPWDUMXStack::STATIC_WDShareWorkerThreadProc+0xd5
    00007ff8325b26ad KERNEL32!BaseThreadInitThunk+0x1d
    00007ff8346aaa68 ntdll!RtlUserThreadStart+0x28

2 threads [stats]: ffffa389b2707080 ffffa389b131c080
    fffff80151a3a756 nt!KiSwapContext+0x76
    fffff8015185e095 nt!KiSwapThread+0xae5
    fffff8015185c3c4 nt!KiCommitThreadWait+0x134
    fffff8015194b4d2 nt!KeWaitForMultipleObjects+0x2e2
    fffff80151cc4cf4 nt!ObWaitForMultipleObjects+0x2e4
    fffff80151d286bc nt!NtWaitForMultipleObjects+0x11c
    fffff80151a45fe5 nt!KiSystemServiceCopyEnd+0x25
    00007ff8346ef8a4 ntdll!NtWaitForMultipleObjects+0x14
    00007ff831f6f5a9 KERNELBASE!WaitForMultipleObjectsEx+0xe9
    00007ff831f6f4ae KERNELBASE!WaitForMultipleObjects+0xe
    00007ff82c08286e RDPBASE!Rdp8Scheduler::RunThreadProc+0x38e
    00007ff8325b26ad KERNEL32!BaseThreadInitThunk+0x1d
    00007ff8346aaa68 ntdll!RtlUserThreadStart+0x28

3 threads [stats]: ffffa389b3947080 ffffa389b61a50c0 ffffa389b33b4080
    fffff80151a3a756 nt!KiSwapContext+0x76
    fffff8015185e095 nt!KiSwapThread+0xae5
    fffff8015185c3c4 nt!KiCommitThreadWait+0x134
    fffff8015194b4d2 nt!KeWaitForMultipleObjects+0x2e2
    fffff80151cc4cf4 nt!ObWaitForMultipleObjects+0x2e4
    ffff8f189c223f4b win32kfull!xxxMsgWaitForMultipleObjectsEx+0xd3
    ffff8f189c1da887 win32kfull!NtUserMsgWaitForMultipleObjectsEx+0x3d7
    ffff8f189c6a7514 win32k!NtUserMsgWaitForMultipleObjectsEx+0x20
    fffff80151a45fe5 nt!KiSystemServiceCopyEnd+0x25
    00007ff8322cacf4 win32u!NtUserMsgWaitForMultipleObjectsEx+0x14
    00007ff82c0906fa RDPBASE!PAL_System_CondWait+0x1ca
    00007ff82c09049d RDPBASE!CTSThreadInternal::ThreadSignalWait+0x5d
    00007ff82c0a938b RDPBASE!CTSThread::internalMsgPump+0x77
    00007ff82c070d0c RDPBASE!CTSThread::internalThreadMsgLoop+0x1ec
    00007ff82c0faf3c RDPBASE!CTSThread::ThreadMsgLoop+0x1c
    00007ff82c11d806 RDPBASE!CRDPENCPlatformContext::STAThreadProc+0x6a
    00007ff82c0fab31 RDPBASE!CTSThread::TSStaticThreadEntry+0x2b1
    00007ff82c101d9c RDPBASE!PAL_System_Win32_ThreadProcWrapper+0x3c
    00007ff8325b26ad KERNEL32!BaseThreadInitThunk+0x1d
    00007ff8346aaa68 ntdll!RtlUserThreadStart+0x28

6 threads [stats]: ffffa389b184b080 ffffa389b2d32080 ffffa389b4c880c0 ffffa389b131a080 ffffa389b4c9a080 ffffa389b28a60c0
    fffff80151a3a756 nt!KiSwapContext+0x76
    fffff8015185e095 nt!KiSwapThread+0xae5
    fffff8015185c3c4 nt!KiCommitThreadWait+0x134
    fffff8015194b4d2 nt!KeWaitForMultipleObjects+0x2e2
    fffff80151cc4cf4 nt!ObWaitForMultipleObjects+0x2e4
    fffff80151d286bc nt!NtWaitForMultipleObjects+0x11c
    fffff80151a45fe5 nt!KiSystemServiceCopyEnd+0x25
    00007ff8346ef8a4 ntdll!NtWaitForMultipleObjects+0x14
    00007ff831f6f5a9 KERNELBASE!WaitForMultipleObjectsEx+0xe9
    00007ff831f6f4ae KERNELBASE!WaitForMultipleObjects+0xe
    00007ff82c08274d RDPBASE!Rdp8Scheduler::RunThreadProc+0x26d
    00007ff8325b26ad KERNEL32!BaseThreadInitThunk+0x1d
    00007ff8346aaa68 ntdll!RtlUserThreadStart+0x28

8 threads [stats]: ffffa389b2cd7080 ffffa389b3ef6080 ffffa389af3c7080 ffffa389b13f4040 ffffa389af1fb040 ffffa389b1ea0080 ffffa389b45aa040 ffffa389af7a5080
    fffff80151a3a756 nt!KiSwapContext+0x76
    fffff8015185e095 nt!KiSwapThread+0xae5
    fffff8015185c3c4 nt!KiCommitThreadWait+0x134
    fffff8015193102d nt!KeRemoveQueueEx+0x111d
    fffff8015192fcc8 nt!IoRemoveIoCompletion+0x98
    fffff8015192f521 nt!NtWaitForWorkViaWorkerFactory+0x381
    fffff80151a45fe5 nt!KiSystemServiceCopyEnd+0x25
    00007ff8346f29a4 ntdll!NtWaitForWorkViaWorkerFactory+0x14
    00007ff83468536e ntdll!TppWorkerThread+0x2ee
    00007ff8325b26ad KERNEL32!BaseThreadInitThunk+0x1d
    00007ff8346aaa68 ntdll!RtlUserThreadStart+0x28

9 stack(s) with 24 threads displayed (30 Total threads)
6 stack(s) were not displayed because we could not switch to thread context, or stack trace was empty
```

Within this example, we are going to use the **`-r`** option to filter strictly. In this example, we are going to look for every call stack that starts with **"combase"**, followed by a string that contains **"crp"**. Here we were able to narrow down to only one result.

```
0: kd> !us -r combase.*crp
1 thread [stats]: ffffa389b323f540
    fffff80151a3a756 nt!KiSwapContext+0x76                              
    fffff8015185e095 nt!KiSwapThread+0xae5                              
    fffff8015185c3c4 nt!KiCommitThreadWait+0x134                        
    fffff8015185b746 nt!KeWaitForSingleObject+0x256                     
    fffff8015194b830 nt!KeWaitForMultipleObjects+0x640                  
    fffff80151cc4cf4 nt!ObWaitForMultipleObjects+0x2e4                  
    fffff80151d286bc nt!NtWaitForMultipleObjects+0x11c                  
    fffff80151a45fe5 nt!KiSystemServiceCopyEnd+0x25                     
    00007ff8346ef8a4 ntdll!NtWaitForMultipleObjects+0x14                
    00007ff831f6f5a9 KERNELBASE!WaitForMultipleObjectsEx+0xe9           
    00007ff83299b35d combase!WaitCoalesced+0xa9                         
    00007ff83299b1ca combase!CROIDTable::WorkerThreadLoop+0x5a          
    00007ff83299afcb combase!CRpcThread::WorkerLoop+0x57                
    00007ff83299af49 combase!CRpcThreadCache::RpcWorkerThreadEntry+0x29 
    00007ff8325b26ad KERNEL32!BaseThreadInitThunk+0x1d                  
    00007ff8346aaa68 ntdll!RtlUserThreadStart+0x28                      

Threads matching filter: 1 out of 24
6 stack(s) were not displayed because we could not switch to thread context, or stack trace was empty
```

This command will scan the entire memory dump to look for threads that are running in NDIS. We can do this with the **`-k`** option in the **`!us`** command. This command will only review the kernel mode stacks though.

```
0: kd> !us -k ndis!
Kernel Summary
============================================================
4 threads [stats]: ffffa389b1098040 ffffa389b1099040 ffffa389b1097080 ffffa389b109c040
    fffff80151a3a756 nt!KiSwapContext+0x76
    fffff8015185e095 nt!KiSwapThread+0xae5
    fffff8015185c3c4 nt!KiCommitThreadWait+0x134
    fffff8015185b746 nt!KeWaitForSingleObject+0x256
    fffff80153e9b151 NDIS!ndisWaitForKernelObject+0x21
    fffff80153dcbd2f NDIS!ndisReceiveWorkerThread+0x883f
    fffff80151924e47 nt!PspSystemThreadStartup+0x57
    fffff80151a361b4 nt!KiStartSystemThread+0x34

12 threads [stats]: ffffa389b26220c0 ffffa389b269b040 ffffa389b27e4040 ffffa389b13220c0 ffffa389b1dc4040 ffffa389b24c8040 ffffa389b4fcc040 ffffa389b61de040 ffffa389b44cb040 ffffa389b2b57040 ...
    fffff80151a3a756 nt!KiSwapContext+0x76
    fffff8015185e095 nt!KiSwapThread+0xae5
    fffff8015185c3c4 nt!KiCommitThreadWait+0x134
    fffff8015185b746 nt!KeWaitForSingleObject+0x256
    fffff80153db45f0 NDIS!NdisWaitEvent+0x50
    fffff80177f6c0b7 AgileVpn!AgileVpnProcessPackets+0xa7
    fffff80151924e47 nt!PspSystemThreadStartup+0x57
    fffff80151a361b4 nt!KiStartSystemThread+0x34

Threads matching filter: 16 out of 1228
410 stack(s) were not displayed because we could not switch to thread context, or stack trace was empty
```

The **`!us -k`** command will scan every thread that can be found in the memory dump and display the kernel mode stack of it. Running this command only will generate tons of results, but when filtering on specific things. Here we are scanning every thread and excluding specific keywords, which allowed us to narrow down to only **10** results.

```
0: kd> !us -k win32kbase -r -e WaitForSingleObject|NtWaitForWorkViaWorkerFactory|ExpWorkerThread
Kernel Summary
============================================================
2 threads [stats]: ffffa389b24d40c0 ffffa389b32cc080
    fffff80151a3a756 nt!KiSwapContext+0x76
    fffff8015185e095 nt!KiSwapThread+0xae5
    fffff8015185c3c4 nt!KiCommitThreadWait+0x134
    fffff8015193102d nt!KeRemoveQueueEx+0x111d
    fffff8015192fcc8 nt!IoRemoveIoCompletion+0x98
    fffff80151d329fe nt!NtRemoveIoCompletionEx+0xfe
    fffff80151a45fe5 nt!KiSystemServiceCopyEnd+0x25
    fffff80151a36ae0 nt!KiServiceLinkage
    ffff8f189be581ef win32kbase!IOCPDispatcher::Wait+0x6f
    ffff8f189be57fea win32kbase!UserKSTWait+0xca
    ffff8f189bec67b1 win32kbase!NtKSTWait+0x21
    ffff8f189c6b0a7e win32k!NtKSTWait+0x16
    fffff80151a45fe5 nt!KiSystemServiceCopyEnd+0x25

2 threads [stats]: ffffa389af835080 ffffa389b3166080
    fffff80151a3a756 nt!KiSwapContext+0x76
    fffff8015185e095 nt!KiSwapThread+0xae5
    fffff8015185c3c4 nt!KiCommitThreadWait+0x134
    fffff8015194b4d2 nt!KeWaitForMultipleObjects+0x2e2
    ffff8f189c050bec win32kbase!ImpWorkerRoutine+0x2ec
    fffff80151924e47 nt!PspSystemThreadStartup+0x57
    fffff80151a361b4 nt!KiStartSystemThread+0x34

3 threads [stats]: ffffa389b37e1080 ffffa389b2e93080 ffffa389b2eb5080
    fffff80151a3a756 nt!KiSwapContext+0x76
    fffff8015185e095 nt!KiSwapThread+0xae5
    fffff8015185c3c4 nt!KiCommitThreadWait+0x134
    fffff8015194b4d2 nt!KeWaitForMultipleObjects+0x2e2
    ffff8f189be51f63 win32kbase!LegacyInputDispatcher::WaitAndDispatch+0x73
    ffff8f189c1f0ed4 win32kfull!xxxDesktopThreadWaiter+0xe0
    ffff8f189c2c10e5 win32kfull!xxxDesktopThread+0x585
    ffff8f189bea1a7e win32kbase!xxxCreateSystemThreads+0x11e
    ffff8f189bea1945 win32kbase!NtUserCreateSystemThreads+0x75
    ffff8f189c6aa6a2 win32k!NtUserCreateSystemThreads+0x16
    fffff80151a45fe5 nt!KiSystemServiceCopyEnd+0x25

3 threads [stats]: ffffa389b3b44080 ffffa389b2e8e080 ffffa389b2eb6080
    fffff80151a3a756 nt!KiSwapContext+0x76
    fffff8015185e095 nt!KiSwapThread+0xae5
    fffff8015185c3c4 nt!KiCommitThreadWait+0x134
    fffff8015194b4d2 nt!KeWaitForMultipleObjects+0x2e2
    ffff8f189be51f63 win32kbase!LegacyInputDispatcher::WaitAndDispatch+0x73
    ffff8f189c23e9c5 win32kfull!RawInputThread+0x935
    ffff8f189bea1a7e win32kbase!xxxCreateSystemThreads+0x11e
    ffff8f189bea1945 win32kbase!NtUserCreateSystemThreads+0x75
    ffff8f189c6aa6a2 win32k!NtUserCreateSystemThreads+0x16
    fffff80151a45fe5 nt!KiSystemServiceCopyEnd+0x25

Threads matching filter: 10 out of 1228
410 stack(s) were not displayed because we could not switch to thread context, or stack trace was empty
```

Wrapping up, suppose we have a specific keyword in mind. We aim to identify every call stack matching our keyword and then display the corresponding threads. To do this, we can use the **`!fems -s <keyword> !t`** command. For instance, we can examine every thread that includes the keyword **"lsa"** in its call stack.

```
0: kd> !fems -s lsass* !t
Process                      Thread                       CID       TEB              UserTime KernelTime ContextSwitches Wait Reason        Time State
lsass.exe (ffffa389b2f09080) ffffa389b2f06080 (E|K|W|R|V) 38c.39c   0000005b6c0eb000        0          0               7 WrLpcReceive 10m:51.031 Waiting

WaitBlockList:
    Object           Type      Other Waiters Info
    ffffa389b2f06558 Semaphore             0 Limit: 1

Priority:
    Current Base Decrement ForegroundBoost IO Page
    10      9    0         0               0  5

# Child-SP         Return           Call Site
0 ffff85893abf1b00 fffff8015185e095 nt!KiSwapContext+0x76
1 ffff85893abf1c40 fffff8015185c3c4 nt!KiSwapThread+0xae5
2 ffff85893abf1d90 fffff8015185b746 nt!KiCommitThreadWait+0x134
3 ffff85893abf1e40 fffff801518f8f2a nt!KeWaitForSingleObject+0x256
4 ffff85893abf21e0 fffff80151d40b0f nt!AlpcpWaitForSingleObject+0x3e
5 ffff85893abf2220 fffff80151dc1fb3 nt!AlpcpCompleteDeferSignalRequestAndWait+0x57
6 ffff85893abf2260 fffff80151c8af03 nt!AlpcpReceiveMessagePort+0x323
7 ffff85893abf22e0 fffff80151c8ad6c nt!AlpcpReceiveLegacyMessage+0x103
8 ffff85893abf2380 fffff80151c8ac8f nt!NtReplyWaitReceivePortEx+0xcc
9 ffff85893abf2420 fffff80151a45fe5 nt!NtReplyWaitReceivePort+0xf
a ffff85893abf2460 00007ff8346eeeb4 nt!KiSystemServiceCopyEnd+0x25
b 0000005b6c3ffad8 00007ff678db4786 ntdll!NtReplyWaitReceivePort+0x14
c 0000005b6c3ffae0 00007ff8325b26ad lsass!LsapRmServerThread+0x66
d 0000005b6c3fff20 00007ff8346aaa68 KERNEL32!BaseThreadInitThunk+0x1d
e 0000005b6c3fff50 0000000000000000 ntdll!RtlUserThreadStart+0x28

Process                      Thread                       CID       TEB              UserTime KernelTime ContextSwitches Wait Reason       Time State
lsass.exe (ffffa389b2f09080) ffffa389b2f0c080 (E|K|W|R|V) 38c.3a8   0000005b6c0f1000        0       16ms              28 UserRequest 10m:52.546 Waiting

WaitBlockList:
    Object           Type                 Other Waiters
    ffffa389b364c960 SynchronizationEvent             0

Priority:
    Current Base Decrement ForegroundBoost IO Page
    9       9    0         0               0  5

# Child-SP         Return           Call Site
0 ffff85893a8f6ce0 fffff8015185e095 nt!KiSwapContext+0x76
1 ffff85893a8f6e20 fffff8015185c3c4 nt!KiSwapThread+0xae5
2 ffff85893a8f6f70 fffff8015185b746 nt!KiCommitThreadWait+0x134
3 ffff85893a8f7020 fffff80151cc47cc nt!KeWaitForSingleObject+0x256
4 ffff85893a8f73c0 fffff80151cc46be nt!ObWaitForSingleObject+0xcc
5 ffff85893a8f7420 fffff80151a45fe5 nt!NtWaitForSingleObject+0x3e
6 ffff85893a8f7460 00007ff8346eedd4 nt!KiSystemServiceCopyEnd+0x25
7 0000005b6c6ff548 00007ff831f43f8e ntdll!NtWaitForSingleObject+0x14
8 0000005b6c6ff550 00007ff83456b4e9 KERNELBASE!WaitForSingleObjectEx+0x8e
9 0000005b6c6ff5f0 00007ff83456add9 sechost!ScSendResponseReceiveControls+0x149
a 0000005b6c6ff730 00007ff8345683bb sechost!ScDispatcherLoop+0x11d
b 0000005b6c6ff870 00007ff8314ff6c3 sechost!StartServiceCtrlDispatcherW+0x6b
c 0000005b6c6ff8a0 0000000000000000 lsasrv!ServiceDispatcherThread+0x1b3
```


# Modules, Imports, and Strings

Third-party products frequently cause crash and hang issues. **MEX** provides several useful commands that allow us to inspect these third-party modules.

```
0:026> !mex.p
Name      Ses PID         PEB              Mods Handle Thrd
========= === =========== ================ ==== ====== ====
lsass.exe   0 1e4 (0n484) 00007ff773b08000  118   1793   42

CommandLine: C:\Windows\system32\lsass.exe
Last event: 1e4.444: Unknown exception - code e0010004 (first/second chance not available)

Show Threads: Unique Stacks    !listthreads (!lt)    ~*kv
```

This command **`!us -3`** will look for the call stacks within the threads that contain 3rd party modules.

```
0:026> !us -3
1 thread [stats]: 20
    00007fffa439abca ntdll!NtWaitForSingleObject+0xa
    00007fffa1b1e11e KERNELBASE!LocalOpenDynData+0x145c0
    00007fff9fdd242e password_sync_dll!PasswordChangeNotify+0xf2a
    00007fff9fdd1c6d password_sync_dll!PasswordChangeNotify+0x769
    00007fffa29116ad kernel32!BaseThreadInitThunk+0xd
    00007fffa4374629 ntdll!RtlUserThreadStart+0x1d

Threads matching filter: 1 out of 42
```

This is an interesting 3rd party product:

```
0:026> lmvm password_sync_dll
Browse full module list
start             end                 module name
00007fff`9fdd0000 00007fff`9fe75000   password_sync_dll   (export symbols)       password_sync_dll.DLL
    Loaded symbol image file: password_sync_dll.DLL
    Image path: C:\Windows\System32\password_sync_dll.DLL
    Image name: password_sync_dll.DLL
    Browse all global symbols  functions  data
    Timestamp:        Tue Jan 24 14:12:06 2017 (5887D136)
    CheckSum:         000A8B0C
    ImageSize:        000A5000
    File version:     1.6.17.0
    Product version:  1.6.17.0
    File flags:       0 (Mask 17)
    File OS:          4 Unknown Win32
    File type:        2.0 Dll
    File date:        00000000.00000000
    Translations:     0409.04b0
    Information from resource tables:
        CompanyName:      Google
        ProductName:      G Suite Password Sync
        InternalName:     Password Sync
        OriginalFilename: password_sync_dll.dll
        ProductVersion:   1.6.17.0
        FileVersion:      1.6.17.0
        FileDescription:  G Suite Password Sync
        LegalCopyright:   Copyright (C) 2011
```

We can use the **`!import`** command to display the import table of a module of interest:

```
0:026> !imports password_sync_dll.DLL
Dumping Import table at 00007fff9fdd0000 + 96858 
ModuleImageName: password_sync_dll
password_sync_dll imports ADVAPI32.dll!ConvertStringSecurityDescriptorToSecurityDescriptorW
password_sync_dll imports ADVAPI32.dll!SetSecurityInfo
password_sync_dll imports ADVAPI32.dll!GetSecurityDescriptorDacl
password_sync_dll imports ADVAPI32.dll!CryptDestroyHash
password_sync_dll imports ADVAPI32.dll!CryptCreateHash
password_sync_dll imports ADVAPI32.dll!CryptGenRandom
password_sync_dll imports ADVAPI32.dll!CryptReleaseContext
password_sync_dll imports ADVAPI32.dll!CryptAcquireContextW
password_sync_dll imports ADVAPI32.dll!CryptGetHashParam
password_sync_dll imports ADVAPI32.dll!ReportEventW
password_sync_dll imports ADVAPI32.dll!DeregisterEventSource
password_sync_dll imports ADVAPI32.dll!RegisterEventSourceW
password_sync_dll imports ADVAPI32.dll!RegCloseKey
password_sync_dll imports ADVAPI32.dll!RegOpenKeyExW
password_sync_dll imports ADVAPI32.dll!RegQueryValueExW
password_sync_dll imports ADVAPI32.dll!CryptHashData
password_sync_dll imports KERNEL32.dll!FormatMessageA
password_sync_dll imports KERNEL32.dll!CreateThread
password_sync_dll imports KERNEL32.dll!DisconnectNamedPipe
password_sync_dll imports KERNEL32.dll!WriteFile
password_sync_dll imports KERNEL32.dll!CreateNamedPipeW
password_sync_dll imports KERNEL32.dll!ConnectNamedPipe
password_sync_dll imports KERNEL32.dll!DeleteCriticalSection
password_sync_dll imports KERNEL32.dll!DecodePointer
password_sync_dll imports KERNEL32.dll!GetLastError
password_sync_dll imports KERNEL32.dll!RaiseException
password_sync_dll imports KERNEL32.dll!InitializeCriticalSectionAndSpinCount
password_sync_dll imports KERNEL32.dll!WideCharToMultiByte
password_sync_dll imports KERNEL32.dll!FindFirstFileW
password_sync_dll imports KERNEL32.dll!TlsGetValue
password_sync_dll imports KERNEL32.dll!HeapAlloc
password_sync_dll imports KERNEL32.dll!SystemTimeToFileTime
password_sync_dll imports KERNEL32.dll!HeapFree
password_sync_dll imports KERNEL32.dll!CreateDirectoryW
password_sync_dll imports KERNEL32.dll!GetProcessHeap
password_sync_dll imports KERNEL32.dll!FileTimeToSystemTime
password_sync_dll imports KERNEL32.dll!CreateFileW
password_sync_dll imports KERNEL32.dll!FindClose
password_sync_dll imports KERNEL32.dll!GetLocalTime
password_sync_dll imports KERNEL32.dll!IsDebuggerPresent
password_sync_dll imports KERNEL32.dll!FindNextFileW
password_sync_dll imports KERNEL32.dll!OutputDebugStringA
password_sync_dll imports KERNEL32.dll!GetCurrentThreadId
password_sync_dll imports KERNEL32.dll!TlsAlloc
password_sync_dll imports KERNEL32.dll!CloseHandle
password_sync_dll imports KERNEL32.dll!DeleteFileW
password_sync_dll imports KERNEL32.dll!GetCurrentProcessId
password_sync_dll imports KERNEL32.dll!DebugBreak
password_sync_dll imports KERNEL32.dll!TlsFree
password_sync_dll imports KERNEL32.dll!LocalFileTimeToFileTime
password_sync_dll imports KERNEL32.dll!InitializeCriticalSection
password_sync_dll imports KERNEL32.dll!Sleep
password_sync_dll imports KERNEL32.dll!LeaveCriticalSection
password_sync_dll imports KERNEL32.dll!EnterCriticalSection
password_sync_dll imports KERNEL32.dll!FreeLibrary
password_sync_dll imports KERNEL32.dll!GetModuleHandleExW
password_sync_dll imports KERNEL32.dll!GetModuleFileNameW
password_sync_dll imports KERNEL32.dll!GetFileInformationByHandle
password_sync_dll imports KERNEL32.dll!GetSystemDirectoryW
password_sync_dll imports KERNEL32.dll!LoadLibraryW
password_sync_dll imports KERNEL32.dll!FormatMessageW
password_sync_dll imports KERNEL32.dll!OutputDebugStringW
password_sync_dll imports KERNEL32.dll!EncodePointer
password_sync_dll imports KERNEL32.dll!GetCommandLineA
password_sync_dll imports KERNEL32.dll!IsProcessorFeaturePresent
password_sync_dll imports KERNEL32.dll!RtlPcToFileHeader
password_sync_dll imports KERNEL32.dll!RtlLookupFunctionEntry
password_sync_dll imports KERNEL32.dll!RtlUnwindEx
password_sync_dll imports KERNEL32.dll!ExitProcess
password_sync_dll imports KERNEL32.dll!GetProcAddress
password_sync_dll imports KERNEL32.dll!MultiByteToWideChar
password_sync_dll imports KERNEL32.dll!HeapSize
password_sync_dll imports KERNEL32.dll!SetLastError
password_sync_dll imports KERNEL32.dll!GetStdHandle
password_sync_dll imports KERNEL32.dll!GetFileType
password_sync_dll imports KERNEL32.dll!GetStartupInfoW
password_sync_dll imports KERNEL32.dll!GetModuleFileNameA
password_sync_dll imports KERNEL32.dll!QueryPerformanceCounter
password_sync_dll imports KERNEL32.dll!GetSystemTimeAsFileTime
password_sync_dll imports KERNEL32.dll!GetEnvironmentStringsW
password_sync_dll imports KERNEL32.dll!FreeEnvironmentStringsW
password_sync_dll imports KERNEL32.dll!RtlCaptureContext
password_sync_dll imports KERNEL32.dll!RtlVirtualUnwind
password_sync_dll imports KERNEL32.dll!UnhandledExceptionFilter
password_sync_dll imports KERNEL32.dll!SetUnhandledExceptionFilter
password_sync_dll imports KERNEL32.dll!GetCurrentProcess
password_sync_dll imports KERNEL32.dll!TerminateProcess
password_sync_dll imports KERNEL32.dll!TlsSetValue
password_sync_dll imports KERNEL32.dll!GetModuleHandleW
password_sync_dll imports KERNEL32.dll!IsValidCodePage
password_sync_dll imports KERNEL32.dll!GetACP
password_sync_dll imports KERNEL32.dll!GetOEMCP
password_sync_dll imports KERNEL32.dll!GetCPInfo
password_sync_dll imports KERNEL32.dll!GetStringTypeW
password_sync_dll imports KERNEL32.dll!FlushFileBuffers
password_sync_dll imports KERNEL32.dll!GetConsoleCP
password_sync_dll imports KERNEL32.dll!GetConsoleMode
password_sync_dll imports KERNEL32.dll!HeapReAlloc
password_sync_dll imports KERNEL32.dll!LoadLibraryExW
password_sync_dll imports KERNEL32.dll!LCMapStringW
password_sync_dll imports KERNEL32.dll!SetFilePointerEx
password_sync_dll imports KERNEL32.dll!SetStdHandle
password_sync_dll imports KERNEL32.dll!WriteConsoleW
password_sync_dll imports ole32.dll!CoTaskMemAlloc
password_sync_dll imports ole32.dll!CoTaskMemFree
password_sync_dll imports ole32.dll!CoGetMalloc
password_sync_dll imports SHELL32.dll!SHGetFolderPathW
password_sync_dll imports SHLWAPI.dll!PathRemoveExtensionW
password_sync_dll imports SHLWAPI.dll!SHRegGetValueW
password_sync_dll imports SHLWAPI.dll!PathAppendW
password_sync_dll imports SHLWAPI.dll!PathRemoveBackslashW
```

We can look for specific strings within this module as well by using the **`!strings`** command:

```
0:026> !grep Directory !strings -m password_sync_dll
00007fff9fe3efa8 directory not empty
00007fff9fe3eff0 no such file or directory
00007fff9fe3f578 is a directory
00007fff9fe3f688 not a directory
00007fff9fe45948 No such file or directory
00007fff9fe45ae8 Not a directory
00007fff9fe45af8 Is a directory
00007fff9fe45c88 Directory not empty
00007fff9fe47970 If true, CSIDL_COMMON_APPDATA directory is used for tracing. Else CSIDL_LOCAL_APPDATA will be used.
00007fff9fe47a40 use_common_appdata_directory_for_tracing
00007fff9fe66f3c CreateDirectoryW
00007fff9fe67110 GetSystemDirectoryW
00007fff9fe722d0 The password change for Active Directory user '%1' was synchronized to
on the Google Account, and will be out of sync with Active Directory.%n%n
00007fff9fe72e0c The account '%1' was not found on Active Directory.%n%n
00007fff9fe72ec8 An unexpected error occurred while querying Active Directory.%n%n
```

If we want to look for all the strings within a module, we can do something like this:

```
0:026> !strings -m password_sync_dll
00007fff9fdd004d !This program cannot be run in DOS mode.\r

$
00007fff9fdd0210 .text
00007fff9fdd0237 `.rdata
00007fff9fdd025f @.data
00007fff9fdd0288 .pdata

<<<<< SNIP >>>>>>>>>.>>
```

If we want to look for all the loaded modules within a process, we can run **`!mex.mods`** command:

```
0:000> !mex.mods
Number of modules: loaded: 20 unloaded: 0 
Base             End                Flags   Time stamp            CLR   Module name        Version                                  Path   Filters:[ Interest|3rd|K|U|A ] More...
=============================================================================================================================================================================
        00400000         00454000 |       |  Invalid Timestamp  |  No | dnsbeacon        | 0.0.0.0                                | C:\Users\RTCAdmin\Downloads\dnsbeacon.exe
00007fffe67d0000 00007fffe6860000 |       | 2084-07-20 05:49:26 |  No | apphelp          | 10.0.19041.3636 (WinBuild.160101.0800) | C:\Windows\System32\apphelp.dll
00007fffe8070000 00007fffe80a4000 |       | 2010-10-07 23:34:55 |  No | rsaenh           | 10.0.19041.3636 (WinBuild.160101.0800) | C:\Windows\System32\rsaenh.dll
00007fffe8430000 00007fffe846b000 |       | 2059-12-30 05:20:04 |  No | IPHLPAPI         | 10.0.19041.3636 (WinBuild.160101.0800) | C:\Windows\System32\IPHLPAPI.DLL
00007fffe84d0000 00007fffe859a000 |       | 2025-01-05 09:36:39 |  No | dnsapi           | 10.0.19041.3636 (WinBuild.160101.0800) | C:\Windows\System32\dnsapi.dll
00007fffe8740000 00007fffe87aa000 |       | 2091-09-11 10:05:13 |  No | mswsock          | 10.0.19041.3636 (WinBuild.160101.0800) | C:\Windows\System32\mswsock.dll
00007fffe8930000 00007fffe893c000 |       | 2023-03-12 12:15:38 |  No | CRYPTBASE        | 10.0.19041.3636 (WinBuild.160101.0800) | C:\Windows\System32\CRYPTBASE.dll
```

# Registry

This command **`!mreg`** will display all the registry values of a certain reg key of interest:

```
0: kd> !mreg -p "\REGISTRY\MACHINE\SYSTEM\ControlSet001\Control\SecurityProviders\WDigest"


Found KCB = ffff908dfb9d6d90 :: \REGISTRY\MACHINE\SYSTEM\CONTROLSET001\CONTROL\SECURITYPROVIDERS\WDIGEST

Hive         ffff908dfb876000
KeyNode      ffff908dfe0c9634

[ValueType]         [ValueName]                   [ValueData]
REG_DWORD           Debuglevel                    0
REG_DWORD           Negotiate                     0
REG_DWORD           UTF8HTTP                      1
REG_DWORD           UTF8SASL                      1
REG_SZ              DigestEncryptionAlgorithms    3des,rc4
```

Example of looking what the computer name is:

```
0: kd> !mreg -p "\REGISTRY\MACHINE\SYSTEM\CONTROLSET001\CONTROL\ComputerName\ComputerName"


Found KCB = ffff908e0218ed80 :: \REGISTRY\MACHINE\SYSTEM\CONTROLSET001\CONTROL\COMPUTERNAME\COMPUTERNAME

Hive         ffff908dfb876000
KeyNode      ffff908dfd530024

[ValueType]         [ValueName]                   [ValueData]
REG_SZ              <nameless value>              mnmsrvc
REG_SZ              ComputerName                  WINDEV2305EVAL
```

Let's look at the services on the machine. I've noticed that not every service is listed though, but we can still give it a try:

```
0: kd> !mreg -p "\REGISTRY\MACHINE\SYSTEM\CONTROLSET001\SERVICES"


Found KCB = ffff908dfb9ad570 :: \REGISTRY\MACHINE\SYSTEM\CONTROLSET001\SERVICES

Hive         ffff908dfb876000
KeyNode      ffff908dfd79753c

[SubKeyAddr]         [SubKeyName]
ffff908dfd797594     .NET CLR Data
ffff908dfd797c9c     .NET CLR Networking
ffff908dfd79993c     .NET CLR Networking 4.0.0.0
ffff908dfd79a254     .NET Data Provider for Oracle
ffff908dfd79ad24     .NET Data Provider for SqlServer
ffff908dfd7a7824     .NET Memory Cache 4.0
ffff908dfd7a7ee4     .NETFramework
ffff908dfd7ab24c     1394ohci
ffff908dfd7ab4a4     3ware
ffff908dfd7ab80c     AarSvc
ffff908dfd68d024     AarSvc_51db5
ffff908dfd7abdbc     ACPI
ffff908dfd7ac2dc     AcpiDev
ffff908dfd7ac57c     acpiex
ffff908dfd7ac884     acpipagr
ffff908dfd7acac4     AcpiPmi
ffff908dfd7acd04     acpitime
ffff908dfd7acfac     Acx01000
ffff908dfd7b01e4     ADOVMPPackage
ffff908dfd7b0394     ADP80XX
ffff908dfd7b06cc     adsi
ffff908dfd7b08ac     AFD
ffff908dfd7b0bec     afunix
ffff908dfd7b415c     ahcache
ffff908dfd7b43c4     AJRouter
ffff908dfd7b4a14     ALG
ffff908dfd7b4e04     amdgpio2
ffff908dfd7b80d4     amdi2c
ffff908dfd7b839c     AmdK8
ffff908dfd7b86b4     AmdPPM
ffff908dfd7b89cc     amdsata
ffff908dfd7b8d04     amdsbs
ffff908dfd7bc0c4     amdxata
ffff908dfd7bc3fc     AnyDesk
ffff908dfd7bc73c     AppID
ffff908dfd7bca64     AppIDSvc
ffff908dfd7c0104     Appinfo
ffff908dfd7c098c     AppleSSD
ffff908dfd7c0d6c     applockerfltr
ffff908dfd7c1364     AppMgmt
ffff908dfd7c1a24     AppReadiness
ffff908dfd7c1f6c     AppVClient
ffff908dfd7c53ec     AppvStrm
ffff908dfd7c5894     AppvVemgr
ffff908dfd7c5d6c     AppvVfs
ffff908dfd7c9284     AppXSvc
ffff908dfd7c9c9c     arcsas
ffff908dfd7cd0d4     AssignedAccessManagerSvc
ffff908dfd7cd64c     AsyncMac
ffff908dfd7cd8ec     atapi
ffff908dfd7cdc2c     AudioEndpointBuilder
ffff908dfd7d1284     Audiosrv
ffff908dfd7d1a1c     autotimesvc
ffff908dfd7d209c     AxInstSV
ffff908dfd7d281c     b06bdrv
ffff908dfd7d2be4     bam
ffff908dfd7d7ccc     BasicDisplay
ffff908dfd7db0d4     BasicRender
ffff908dfd7db3fc     BattC
ffff908dfd7db4c4     BcastDVRUserService
ffff908dfd68d364     BcastDVRUserService_51db5
ffff908dfd7dbb6c     bcmfn2
ffff908dfd7dbdfc     BDESVC
ffff908dfd7df62c     Beep
ffff908dfd7df794     BFE
ffff908dfd7dfec4     bfs
ffff908dfd7e045c     bindflt
ffff908dfd7e0a5c     BITS
ffff908dfd7e45dc     BluetoothUserService
ffff908dfe1e70ac     BluetoothUserService_51db5
ffff908dfd7e8024     bowser
ffff908dfd7e82cc     BrokerInfrastructure
ffff908dfd7e8cd4     BTAGService
ffff908dfd7ec344     BthA2dp
ffff908dfd7ec5f4     BthAvctpSvc
ffff908dfd7ecddc     BthEnum
ffff908dfd7ed054     BthHFEnum
ffff908dfd7ed3cc     BthLEEnum
ffff908dfd7ed66c     BthMini
ffff908dfd7ed99c     BTHMODEM
ffff908dfd7edc04     BTHPORT
ffff908dfd7f6ea4     bthserv
ffff908dfd7fa484     BTHUSB
ffff908dfd7fa7bc     bttflt
ffff908dfd7fab5c     buttonconverter
ffff908dfd7fadfc     CAD
ffff908dfd7fb1ec     camsvc
ffff908dfd7fb77c     CaptureService
ffff908dfe1ee764     CaptureService_51db5
ffff908dfd7fbd54     cbdhsvc
ffff908dfe1eeb24     cbdhsvc_51db5
ffff908dfd7ff3f4     cdfs
ffff908dfd7ff6ec     CDPSvc
ffff908dfd7ffe1c     CDPUserSvc
ffff908dfe1eef7c     CDPUserSvc_51db5
ffff908dfda024a4     cdrom
ffff908dfda02a5c     CertPropSvc
ffff908dfd808294     cht4iscsi
ffff908dfd808604     cht4vbd
ffff908dfd808964     CimFS
ffff908dfd808a7c     circlass
ffff908dfd808e1c     CldFlt
ffff908dfd80c2f4     CLFS
ffff908dfd80c884     ClipSVC
ffff908dfd8100fc     cloudidsvc
ffff908dfd8106cc     clr_optimization_v4.0.30319_32
ffff908dfd81095c     clr_optimization_v4.0.30319_64
ffff908dfd810bec     CmBatt
ffff908dfd810e64     CNG
ffff908dfd811024     cnghwassist
ffff908dfd8112cc     CompositeBus
ffff908dfd8116f4     COMSysApp
ffff908dfd811c04     condrv
ffff908dfd811dcc     ConsentUxUserSvc
ffff908dfe1f676c     ConsentUxUserSvc_51db5
ffff908dfd815404     CoreMessagingRegistrar
ffff908dfd815a2c     CoreUI
ffff908dfd815e34     CredentialEnrollmentManagerUserSvc
ffff908dfd560ea4     CredentialEnrollmentManagerUserSvc_51db5
ffff908dfd8192f4     crypt32
ffff908dfd81934c     CryptSvc
ffff908dfd819b2c     CSC
ffff908dfd819e0c     CscService
ffff908dfd81
```

If we don't want to scroll through the entire list, we can use the **!grep** command to filter:

```
0: kd> !grep core !mreg -p "\REGISTRY\MACHINE\SYSTEM\CONTROLSET001\SERVICES"
ffff908dfd815404     CoreMessagingRegistrar
ffff908dfd815a2c     CoreUI
```

The **`!svcreg`** command allows us to dump the service/driver registry key:

```
0: kd> !svcreg spooler


Sorry <\registry\machine\system\controlset001\services\spooler> is not cached 

===========================================================================================
Falling back to traversing the tree of nodes.

Hive         ffff908dfb876000
KeyNode      ffff908dfd955d34

[SubKeyAddr]         [SubKeyName]
ffff908dfd95623c     Performance
ffff908dfd95644c     Security

 Use '!reg keyinfo ffff908dfb876000 <SubKeyAddr>' to dump the subkey details

[ValueType]         [ValueName]                   [ValueData]
REG_MULTI_SZ        DependOnService               RPCSS\0http\0
REG_SZ              Description                   @%systemroot%\system32\spoolsv.exe,-2
REG_SZ              DisplayName                   @%systemroot%\system32\spoolsv.exe,-1
REG_DWORD           ErrorControl                  1
REG_BINARY          FailureActions                0xffff908dfd955f4c - 10 0e 00 00 00 00 00 00 00 00 00 00 03 00 00 00 14 00 00 00 01 00 00 00 88 13 00 00 01 00 00 00 88 13 00 00 00 00 00 00 00 00 00 00 
REG_SZ              Group                         SpoolerGroup
REG_EXPAND_SZ       ImagePath                     %SystemRoot%\System32\spoolsv.exe
REG_SZ              ObjectName                    LocalSystem
REG_MULTI_SZ        RequiredPrivileges            SeTcbPrivilege\0SeImpersonatePrivilege\0SeAuditPrivilege\0SeChangeNotifyPrivilege\0SeAssignPrimaryTokenPrivilege\0SeLoadDriverPrivilege\0
REG_DWORD           ServiceSidType                1
REG_DWORD           Start                         2
REG_DWORD           Type                          110
```
