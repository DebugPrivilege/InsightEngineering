# What are Thread Wait Reasons?

A thread wait reason is a status that indicates why a thread is not currently executing. Here are the common one's that I often see within **Process Explorer**.

| Wait Reason | Description |
| ----------- | ----------- |
| DelayExecution | The thread is in a "sleep" mode, possibly because of a call to `Sleep`, `SleepEx`, or `KeDelayExecutionThread` in the kernel. |
| Executive | This is a generic placeholder. It means some kernel component (like a driver) asked the thread to wait. |
| WrQueue | The thread is waiting for some incoming work, usually from an I/O operation. |
| Suspended | The thread is on hold. It might be because it decided to hit the suspend itself, or another thread told it to. |
| UserRequest | This is the wait reason you'll see the most. It's often because the program asked the thread to wait, but it could also be because a kernel component (like a driver) made the request. |
| WrAlertByThreadId | This happens when a thread is waiting for things like Critical Sections, Conditional Variables, or Slim Reader/Writer locks. |
| WrLpcReceive | The thread is waiting for an incoming Local Procedure Call (LPC) message. LPC is a communication method used by different processes on the same machine to share data. |
| WrLpcReply | The thread is waiting for reply to a local procedure call to arrive. |
| WrPreempted | This refers to a thread that was paused on a CPU because a more important thread needed to use that CPU. |
| WrFreePage | The thread is waiting for a free virtual memory page. |
| WrVirtualMemory | The thread is waiting for the system to allocate virtual memory. |

# How to view Thread Wait Reasons in Process Explorer?

Here is an example in **Process Explorer:**

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/d82642ec-7cf2-4671-8d94-8afbc78e0606)

# Why do Thread Wait Reasons even matter?

When analyzing crash dumps, one of the primary objectives is to determine the state of the system at the time of the crash. Within that system state, the status of each thread can provide useful information.

Here is an example of a crash that reveals the call stack of the thread that crashed. From the **PROCESS_NAME** field, we can determine that the thread was running under the **SYSTEM** process. As a next step, we could list all the threads residing in that process to delve deeper into the problem

```
0: kd> !analyze -v
*******************************************************************************
*                                                                             *
*                        Bugcheck Analysis                                    *
*                                                                             *
*******************************************************************************

KERNEL_MODE_HEAP_CORRUPTION (13a)
The kernel mode heap manager has detected corruption in a heap.
Arguments:
Arg1: 0000000000000012, Type of corruption detected
Arg2: ffff95029d402140, Address of the heap that reported the corruption
Arg3: ffff9502a4b5c000, Address at which the corruption was detected
Arg4: 0000000000000000

Debugging Details:
------------------

PROCESS_NAME:  System

STACK_TEXT:  
ffff9007`658d9e78 fffff803`677c8298     : 00000000`0000013a 00000000`00000012 ffff9502`9d402140 ffff9502`a4b5c000 : nt!KeBugCheckEx
ffff9007`658d9e80 fffff803`677c82f8     : 00000000`00000012 00000000`00000000 ffff9502`9d402140 00000000`00000040 : nt!RtlpHeapHandleError+0x40
ffff9007`658d9ec0 fffff803`677c7f15     : 00000000`00000000 00000000`00000000 00000000`00000000 00000000`00000100 : nt!RtlpHpHeapHandleError+0x58
ffff9007`658d9ef0 fffff803`676e0110     : ffff9502`9d402140 fffff803`00000000 00000000`00000000 ffff9502`a5393870 : nt!RtlpLogHeapFailure+0x45
ffff9007`658d9f20 fffff803`67cae4c0     : ffff9502`00000000 00000000`00000004 fffff803`9abc003b 00000000`00000001 : nt!RtlpHpFreeHeap+0x1add80
ffff9007`658d9fc0 fffff803`9abc10ae     : 00006afd`00000000 00000000`00000000 ffff9502`00000004 00000000`00000000 : nt!ExFreePoolWithTag+0x1a0
ffff9007`658da050 fffff803`9abc128b     : 00000000`00000000 ffff9502`a17ca000 ffff9502`a17ca000 ffff9502`a5393870 : TestDriver!DriverEntry+0xae [C:\Users\User\source\repos\TestDriver\TestDriver\Driver.c @ 49] 
ffff9007`658da0c0 fffff803`9abc11c0     : ffff9502`a17ca000 ffff9007`658da280 ffff9502`a548a5ed ffff9502`a4745950 : TestDriver!FxDriverEntryWorker+0xbf [minkernel\wdf\framework\kmdf\src\dynamic\stub\stub.cpp @ 360] 
ffff9007`658da100 fffff803`679d3480     : ffff9502`a17ca000 00000000`00000000 ffff9502`a4745950 fffff803`675674a4 : TestDriver!FxDriverEntry+0x20 [minkernel\wdf\framework\kmdf\src\dynamic\stub\stub.cpp @ 249] 
ffff9007`658da130 fffff803`679d3d4f     : ffff9502`a17ca000 00000000`00000000 00000000`00000000 ffffad02`94ef8810 : nt!PnpCallDriverEntry+0x54
ffff9007`658da180 fffff803`679d2b27     : 00000000`00000000 00000000`00000000 00000000`00000000 fffff803`67f49ac0 : nt!IopLoadDriver+0x523
ffff9007`658da340 fffff803`674176a5     : ffff9502`00000000 ffffffff`8000203c ffff9502`9db18080 ffff9502`00000000 : nt!IopLoadUnloadDriver+0x57
ffff9007`658da380 fffff803`67524e47     : ffff9502`9db18080 00000000`00000178 ffff9502`9db18080 fffff803`67417550 : nt!ExpWorkerThread+0x155
ffff9007`658da570 fffff803`676361b4     : ffffbd81`45992180 ffff9502`9db18080 fffff803`67524df0 00000000`00000000 : nt!PspSystemThreadStartup+0x57
ffff9007`658da5c0 00000000`00000000     : ffff9007`658db000 ffff9007`658d4000 00000000`00000000 00000000`00000000 : nt!KiStartSystemThread+0x34
```

There are a total of **210** threads, as shown in the output. Each of them has a thread wait reason, so understanding these wait reasons is beneficial.

```
0: kd> !mex.lt
Process PID Thread             Id State        Time Reason
======= === ================ ==== ======= ========= ===============
System    4 ffff95029da90200    c Waiting 1m:45.390 Executive
System    4 ffff95029db02080   10 Waiting 5m:57.359 Executive
System    4 ffff95029db78080   14 Waiting 5m:57.359 Executive
System    4 ffff95029db34080   18 Waiting 5m:57.359 Executive
System    4 ffff95029db9a080   1c Waiting 5m:57.359 Executive
System    4 ffff95029db67080   30 Waiting   57s.328 Executive
System    4 ffff95029db89080   34 Waiting   31s.765 WrQueue
System    4 ffff95029dbcd080   38 Waiting     125ms WrQueue
System    4 ffff95029dbde080   3c Waiting 5m:57.359 Suspended
System    4 ffff95029dbef080   40 Waiting 5m:57.359 Suspended
System    4 ffff95029da75080   44 Waiting 5m:57.359 Suspended
System    4 ffff95029da77080   48 Waiting 5m:57.359 Suspended
System    4 ffff95029da8c080   50 Waiting     390ms Executive
System    4 ffff95029db18080   54 Running         0 WrPageIn
System    4 ffff95029db1a080   58 Waiting   10s.375 Executive
System    4 ffff95029db1c080   5c Waiting    9s.203 Executive
System    4 ffff95029db1e080   60 Waiting 5m:57.359 WrFreePage
System    4 ffff95029db20080   64 Waiting     390ms WrFreePage
System    4 ffff95029dad6080   68 Waiting     390ms WrFreePage
System    4 ffff95029dad8080   6c Waiting     390ms WrFreePage
System    4 ffff95029dadc080   78 Waiting 5m:57.312 WrQueue
System    4 ffff95029dade080   7c Waiting 5m:57.312 WrQueue
System    4 ffff95029dae0080   80 Waiting   31s.765 WrQueue
System    4 ffff95029db49080   84 Waiting 1m:07.203 WrQueue
System    4 ffff95029db53080   88 Waiting    8s.125 Executive
System    4 ffff95029db5c080   8c Waiting   38s.750 Executive
System    4 ffff95029db5e080   90 Waiting    8s.187 Executive
System    4 ffff95029db62080   94 Waiting    3s.265 Executive
System    4 ffff95029db64080   98 Waiting     906ms Executive
System    4 ffff95029db6b080   9c Waiting    2s.875 Executive
System    4 ffff95029db6d080   a0 Waiting   38s.750 Executive
System    4 ffff95029db71080   a4 Waiting   19s.046 Executive
System    4 ffff95029db73080   a8 Waiting 5m:53.843 Executive
System    4 ffff95029db7a080   ac Waiting 5m:53.843 Executive
System    4 ffff95029db7c080   b0 Waiting 1m:21.218 Executive
System    4 ffff95029db80080   b4 Waiting 5m:53.843 Executive
System    4 ffff95029db82080   b8 Waiting 5m:53.843 Executive
System    4 ffff95029db8b080   c0 Waiting 5m:53.843 Executive
System    4 ffff95029db91080   c4 Waiting   42s.984 Executive
System    4 ffff95029db93080   c8 Waiting 5m:49.656 Executive
System    4 ffff95029db97080   cc Waiting 5m:53.843 Executive
System    4 ffff95029e23e080   d0 Waiting    9s.968 WrQueue
System    4 ffff95029dba0080   d8 Waiting 5m:57.218 WrQueue
System    4 ffff95029dba2080   dc Waiting 5m:57.218 WrQueue
System    4 ffff95029dba4080   e0 Waiting 5m:57.218 WrQueue
System    4 ffff95029dba6080   e4 Waiting 5m:57.218 WrQueue
System    4 ffff95029dc68400   e8 Waiting   16s.156 WrQueue
System    4 ffff95029dc66100   ec Waiting 1m:45.390 DelayExecution
System    4 ffff95029dbaf080   f0 Waiting   31s.765 WrQueue
System    4 ffff95029dbc4080   f4 Waiting   11s.312 Executive
System    4 ffff95029dbe0080   f8 Waiting 5m:50.187 WrQueue
System    4 ffff95029dbe8080   fc Waiting    8s.093 WrQueue
System    4 ffff95029da7b080  100 Waiting 5m:57.140 WrQueue
System    4 ffff95029dbfe040  104 Waiting 5m:57.140 Executive
System    4 ffff95029dc6d040  108 Waiting 5m:57.140 WrQueue
System    4 ffff95029da94040  10c Waiting 5m:57.140 WrQueue
System    4 ffff95029dc67040  110 Waiting 5m:57.140 WrQueue
System    4 ffff95029db0d040  114 Waiting 5m:57.140 WrQueue
System    4 ffff95029db0e040  118 Waiting 5m:57.140 WrQueue
System    4 ffff95029db0f040  11c Waiting 5m:57.140 WrQueue
System    4 ffff95029db10040  120 Waiting 5m:57.140 WrQueue
System    4 ffff95029db11040  124 Waiting 5m:57.140 WrQueue
System    4 ffff95029dac5040  128 Waiting 5m:57.140 WrQueue
System    4 ffff95029dac6040  12c Waiting 5m:57.140 WrQueue
System    4 ffff95029dac7040  130 Waiting 5m:57.140 WrQueue
System    4 ffff95029dac8040  134 Waiting 5m:57.140 WrQueue
System    4 ffff95029dac9040  138 Waiting 5m:57.140 Executive
System    4 ffff95029daca040  13c Waiting 5m:51.968 Executive
System    4 ffff95029e385040  140 Waiting 5m:57.125 Executive
System    4 ffff95029e386040  144 Waiting 5m:57.125 Executive
System    4 ffff95029e387040  148 Waiting 5m:57.125 Executive
System    4 ffff95029e388040  14c Waiting 5m:57.125 Executive
System    4 ffff95029e389040  150 Waiting 5m:57.125 Executive
System    4 ffff95029e38a040  154 Waiting 5m:57.125 Executive
System    4 ffff95029e392080  158 Waiting 5m:57.125 Executive
System    4 ffff95029e393040  15c Waiting 5m:57.125 Executive
System    4 ffff95029e3ab080  160 Waiting      15ms Executive
System    4 ffff95029e3a1040  164 Waiting   14s.703 Executive
System    4 ffff95029e3ae040  168 Waiting   10s.171 Executive
System    4 ffff95029e3af040  16c Waiting    3s.859 Executive
System    4 ffff95029e3b0040  170 Waiting   14s.687 Executive
System    4 ffff95029dbb5080  174 Waiting 5m:57.109 Executive
System    4 ffff95029dbe2080  178 Waiting 5m:57.109 Executive
System    4 ffff95029e3be040  17c Waiting 5m:57.109 Executive
System    4 ffff95029e47e040  180 Waiting 5m:57.109 Executive
System    4 ffff95029e489040  184 Waiting 5m:57.109 Executive
System    4 ffff95029e48a040  188 Waiting 5m:57.109 Executive
System    4 ffff95029e48b040  18c Waiting 5m:57.093 Executive
System    4 ffff95029e490040  190 Waiting 5m:57.109 Executive
System    4 ffff95029e6ee500  19c Waiting   16s.781 WrVirtualMemory
System    4 ffff95029e6ef040  1a0 Waiting 1m:48.140 WrFreePage
System    4 ffff95029e6f2040  1a4 Waiting     421ms WrFreePage
System    4 ffff95029e6f3040  1a8 Waiting   40s.265 WrFreePage
System    4 ffff95029e6f4040  1ac Waiting 5m:57.046 WrFreePage
System    4 ffff95029e6f5040  1b0 Waiting 5m:57.046 WrFreePage
System    4 ffff95029e999040  1b8 Waiting 5m:21.906 WrQueue
System    4 ffff95029e99a040  1bc Waiting    9s.968 WrQueue
System    4 ffff95029e874080  1c4 Waiting      62ms WrQueue
System    4 ffff95029e98c040  1c8 Waiting 5m:56.218 WrFreePage
System    4 ffff95029e8d0040  1cc Waiting 5m:56.218 WrFreePage
System    4 ffff95029e8d1040  1d0 Waiting 5m:56.218 WrFreePage
System    4 ffff95029e8d2040  1d4 Waiting 5m:56.218 WrFreePage
System    4 ffff95029e9f0040  1d8 Waiting 1m:05.343 Executive
System    4 ffff95029ea55540  1dc Waiting 5m:56.000 Executive
System    4 ffff95029ea5c040  1e0 Waiting 4m:48.437 WrFreePage
System    4 ffff95029eab0040  1e4 Waiting 5m:55.968 WrFreePage
System    4 ffff95029eab1040  1e8 Waiting 5m:55.968 WrFreePage
System    4 ffff95029eab2040  1ec Waiting 5m:55.968 WrFreePage
System    4 ffff95029eab3040  1f0 Waiting      15ms Executive
System    4 ffff95029eb6e040  1f4 Waiting 1m:05.375 Executive
System    4 ffff95029eb28040  1f8 Waiting   43s.093 WrLpcReceive
System    4 ffff95029eb3d440  204 Waiting 5m:55.890 Executive
System    4 ffff95029ec05040  208 Waiting     515ms Executive
System    4 ffff95029ec8c040  20c Waiting   16s.234 WrQueue
System    4 ffff9502a116a040  210 Waiting 5m:47.765 WrQueue
System    4 ffff9502a116b040  214 Waiting   31s.890 WrQueue
System    4 ffff9502a1d16040  238 Waiting 5m:35.828 WrFreePage
System    4 ffff9502a1d17040  23c Waiting 5m:51.109 WrFreePage
System    4 ffff9502a1d18040  240 Waiting 5m:53.875 WrFreePage
System    4 ffff9502a1d19040  244 Waiting 5m:53.875 WrFreePage
System    4 ffff95029ebaf080  284 Waiting    7s.171 WrPageOut
System    4 ffff9502a1da3080  2a4 Waiting    9s.625 Executive
System    4 ffff9502a1ed8080  2a8 Waiting 3m:47.406 Executive
System    4 ffff9502a1ed9080  2ac Waiting      78ms Executive
System    4 ffff9502a1ee5100  2b0 Waiting     140ms Executive
System    4 ffff9502a1eef0c0  2b4 Waiting    9s.046 Executive
System    4 ffff9502a1f79080  300 Waiting 3m:47.343 Executive
System    4 ffff9502a1f9e080  30c Waiting   10s.546 WrQueue
System    4 ffff9502a1f66080  434 Waiting 5m:52.859 Executive
System    4 ffff9502a22500c0  480 Waiting   52s.703 Executive
System    4 ffff9502a2251080  484 Waiting   47s.453 Executive
System    4 ffff9502a24f1080  720 Waiting 5m:52.609 Executive
System    4 ffff9502a2599540  7f8 Waiting     421ms WrQueue
System    4 ffff9502a270c040  880 Waiting    4s.515 Executive
System    4 ffff95029db32040  8dc Waiting 5m:52.343 WrQueue
System    4 ffff9502a2b8f480  864 Waiting 5m:52.093 Executive
System    4 ffff9502a2b98080  a88 Waiting 5m:52.093 Executive
System    4 ffff9502a2ba1040  adc Waiting 5m:52.093 Executive
System    4 ffff9502a2ba7040  b00 Waiting 5m:52.093 Executive
System    4 ffff9502a2a49280  af0 Waiting 5m:52.093 Executive
System    4 ffff9502a2a47140  af8 Waiting 5m:52.093 WrExecutive
System    4 ffff9502a2c27080  cc4 Waiting 1m:51.296 Executive
System    4 ffff9502a2c31040  ce4 Waiting 5m:51.984 Executive
System    4 ffff9502a2c32040  ce8 Waiting 5m:51.984 Executive
System    4 ffff9502a2c78080  cf4 Waiting 5m:51.984 Executive
System    4 ffff9502a2ccb080  d04 Waiting 5m:11.265 Executive
System    4 ffff9502a2f57080  d94 Waiting 4m:28.000 Executive
System    4 ffff9502a2635080  da0 Waiting 3m:52.718 WrQueue
System    4 ffff9502a2b20080  da4 Waiting      15ms WrVirtualMemory
System    4 ffff9502a2ddc040  f44 Waiting      46ms Executive
System    4 ffff9502a2f44040  f5c Waiting 5m:51.578 Executive
System    4 ffff9502a2da8040  f64 Waiting    2s.093 Executive
System    4 ffff9502a3193080  fc4 Waiting    7s.937 Executive
System    4 ffff9502a3381040 1120 Waiting 5m:51.343 Executive
System    4 ffff9502a338b040 1174 Waiting      15ms Executive
System    4 ffff9502a340b040 11ac Waiting 1m:05.312 Executive
System    4 ffff9502a3464040 121c Waiting 1m:05.312 Executive
System    4 ffff9502a29db080 1264 Waiting 5m:04.468 Executive
System    4 ffff9502a3474040 126c Waiting 5m:15.187 Executive
System    4 ffff9502a3473040 1270 Waiting 4m:45.140 Executive
System    4 ffff9502a3471040 1274 Waiting 5m:51.140 Executive
System    4 ffff9502a3470040 1278 Waiting 5m:51.140 Executive
System    4 ffff9502a37b1080 1298 Waiting 1m:05.312 Executive
System    4 ffff9502a340e040 12f8 Waiting 1m:05.312 Executive
System    4 ffff9502a37e0080 1310 Waiting   20s.828 Executive
System    4 ffff9502a37d8040 1334 Waiting 1m:05.312 Executive
System    4 ffff9502a34ac040 1364 Waiting 1m:05.312 Executive
System    4 ffff9502a362e040 1384 Waiting 1m:05.312 Executive
System    4 ffff9502a34e6040 1394 Waiting 1m:05.312 Executive
System    4 ffff9502a34de040 13c0 Waiting 1m:05.312 Executive
System    4 ffff9502a36cb040 13f0 Waiting 1m:05.312 Executive
System    4 ffff9502a36c5080 119c Waiting 1m:05.312 Executive
System    4 ffff9502a36e4040    8 Waiting 1m:05.312 Executive
System    4 ffff95029dac4080 142c Waiting 1m:05.312 Executive
System    4 ffff9502a373c040 1460 Waiting 1m:05.312 Executive
System    4 ffff9502a373a080 1468 Waiting 5m:38.546 Executive
System    4 ffff9502a3504080 149c Waiting 5m:38.546 Executive
System    4 ffff9502a3503080 14a0 Waiting 5m:38.546 Executive
System    4 ffff9502a35020c0 14a4 Waiting 5m:38.546 Executive
System    4 ffff9502a3785080 14ac Waiting      78ms Executive
System    4 ffff9502a3569040 14d0 Waiting 1m:05.312 Executive
System    4 ffff9502a3567080 14d8 Waiting 5m:38.546 Executive
System    4 ffff9502a3911080 1674 Waiting 5m:50.156 Executive
System    4 ffff9502a3a7f080 179c Waiting   14s.734 Executive
System    4 ffff9502a3b1a040 1404 Waiting   31s.781 WrQueue
System    4 ffff9502a3b19040 14b8 Waiting      31ms WrQueue
System    4 ffff9502a3b16040 11e0 Waiting      31ms WrQueue
System    4 ffff9502a41e3040 1eac Waiting   31s.765 WrQueue
System    4 ffff9502a41e2040 1eb0 Waiting   31s.765 WrQueue
System    4 ffff9502a41e1040 1eb4 Waiting   31s.781 WrQueue
System    4 ffff9502a41e0040 1eb8 Waiting   31s.765 WrQueue
System    4 ffff9502a41df040 1ebc Waiting         0 WrQueue
System    4 ffff9502a41de040 1ec0 Waiting   16s.156 WrQueue
System    4 ffff9502a41dd040 1ec4 Waiting   31s.765 WrQueue
System    4 ffff9502a41dc040 1ec8 Waiting   31s.765 WrQueue
System    4 ffff9502a44e5080 1d98 Waiting 5m:32.343 Executive
System    4 ffff9502a2b21040 27e0 Waiting 5m:21.906 WrQueue
System    4 ffff95029dbae040 27e4 Waiting   31s.765 WrQueue
System    4 ffff9502a4f4d080 2a1c Waiting      15ms Executive
System    4 ffff9502a5183040 2a40 Waiting 4m:34.437 Executive
System    4 ffff9502a50d9040 2a7c Waiting 1m:04.000 WrQueue
System    4 ffff9502a2a86040 2ac4 Waiting 3m:52.718 WrQueue
System    4 ffff9502a3ec2040 2a80 Waiting 1m:04.000 WrQueue
System    4 ffff9502a43f3040 2aac Waiting 1m:04.000 WrQueue
System    4 ffff9502a37d7040 2a78 Waiting 1m:04.000 WrQueue
System    4 ffff9502a4fbf040 2638 Waiting 3m:52.718 WrQueue
System    4 ffff9502a1b60080 135c Waiting    1s.859 Executive
System    4 ffff9502a3ee8080 187c Waiting 3m:49.671 Executive
System    4 ffff9502a2a92080 23e8 Waiting      15ms Executive
System    4 ffff9502a35a1080 12c8 Waiting    6s.078 Executive

Thread Count: 210
```
