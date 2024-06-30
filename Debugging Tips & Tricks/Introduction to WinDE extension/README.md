# Introduction

The WinDE extension provides a set of commands for debugging in kernel mode. It includes analyzing system activity, handling locks, searching call stacks, examining memory usage, and detecting common hang and leak symptoms. Commands like **`!WinDE.activity`**, **`!WinDE.blocks`**, **`!WinDE.calls`**, and **`!WinDE.leak`** help diagnose performance issues, memory leaks, and system hangs. The extension also offers detailed information about processes, threads, and handles. 

The extension also offers information about processes, threads, and handles. It appears to be an older extension, dating back to around 2011-ish? It was developed by some Escalation Engineers in the past, as I've been informed. Not every command will be displayed, just the most relevant one's. Also, I've picked commands that also don't take forever to run.

This extension can be downloaded here: https://github.com/DebugPrivilege/InsightEngineering/tree/main/Debugging%20Case%20Studies/Extensions

**DISCLAIMER:** This extension is available at https://www.zhaodll.com/dll/softdown.asp?softid=111931&iz2=dc975e09c06714c45ca02c7244678aa3, which is where I downloaded it from. Some functionality may require private symbols, so not every command might work fully.

# WalkThrough

The **`!WinDE.help`** command provides a detailed list of all available commands within the **`WinDE.dll`** extension. It even includes brief descriptions and usage information for each command.

```
0: kd> !winde.help

==================
Help for WinDE.dll				Support alias: WinDE
==================

!WinDE.<command> -?  - Detailed help for the command

Alphabetical list of Kernel-Mode WinDE commands:

!WinDE.activity      =  Check the System activity for numerous common hang symptoms
!WinDE.blocks        =  Dump ERESOURCE locks with waiters, and waiter/owner threads
!WinDE.calls         =  Search call stacks for string(s)
!WinDE.cx            =  Dump CRITICAL_SECTION lock, and its waiter/owner threads
!WinDE.debug         =  display current WinDE debug settings, and optionally set debug flags
!WinDE.diskq         =  list DISK devices with queued IO, or one device's IO queue
!WinDE.dtpool        -  Reads pool allocation and attempts to determine type
!WinDE.eflags        -  Displays flags from the eflags register
!WinDE.er            =  Dump ERESOURCE lock, and its waiter/owner threads
!WinDE.evtlog        =  Locate event log file data in memory
!WinDE.exe           =  Find processes with EXE names containing the string
!WinDE.exq           =  Scan Executive Work Queues for active threads and queued work items
!WinDE.filedevice    -  more details about handles
!WinDE.findfile      -  Searches Nonpaged pool for File objects matching search string
!WinDE.findhandle    -  Searches for open handles to a given object
!WinDE.findsvc       =  Find a running service by name
!WinDE.fm            =  Dump FAST_MUTEX lock, and its waiter/owner threads
!WinDE.gdihandle     -  Dum!ups GDIHandleCount and UserHandleCount for each process
!WinDE.gettickcount  -  Get elapsed time since last input (keyboard/mouse)
!WinDE.gm            =  Dump KGUARDED_MUTEX lock, and its waiter/owner threads
!WinDE.handles       =  Statistical analysis of handles in a process
!WinDE.handletype    -  more details about handles
!WinDE.hang          -  Hang analysis for the system
!WinDE.idle          =  Dump the Idle process (/D=more detail, /V=verbose)
!WinDE.info          =  display general information about the current debug session
!WinDE.io            =  Dump an IRP, and its associated event and file object
!WinDE.ioe           =  Shortcut for "!irpfind 0 0 userevent args"
!WinDE.ioq           =  List disk-related devices with stuck IO
!WinDE.ipconfig      -  Display Information for TCPIP Interfaces on the system
!WinDE.irps          =  Find all active IRPs, and display IRP count statistics
!WinDE.k             =  Dump and analyze a thread's stack
!WinDE.kp            =  Dump and analyze a thread's stack, with Parameters in DML
!WinDE.kpu           =  Dump and analyze a thread's stack, with Parameters in DML, and user symbols reload
!WinDE.ku            =  Dump and analyze a thread's stack, with user symbols reload
!WinDE.leak          =  Check the System for numerous common leak symptoms
!WinDE.lm            =  Dump LPC message, waiter thread, LPC process/thread, etc
!WinDE.lp            =  List process/thread times and handle counts, etc
!WinDE.lpcwait       =  List LPC server threads and connections waiting
!WinDE.lswn          -  Show information about the SRV Work Queues and allocated work items
!WinDE.mem           =  Display the top memory consumers
!WinDE.mod           =  Show module information
!WinDE.modref        =  Scan Modules for references to an address
!WinDE.mpioq         =  (beta) list MPIO devices with queued IO, or one device's IO queue
!WinDE.mu            =  Dump KMUTANT lock, and its waiter/owner threads
!WinDE.netstat       -  Display the valid endpoints on the system
!WinDE.objects       =  Show system Objects object and handle counts
!WinDE.objdir        =  Browse the Object Manager's Object Directory
!WinDE.pcrs          =  Dump all PCRs, and all Running and Standby threads
!WinDE.pl            =  Dump EX_PUSH_LOCK lock
!WinDE.poolfailures  =  Displays pool allocation failure statistics and reasons
!WinDE.poolref       =  Scan NonPaged pool for references to an address
!WinDE.pooltag       =  Show the pooltag for the pool block containing an address
!WinDE.pup           =  Shortcut for .Process adr;.reload /User;!Process adr 7
!WinDE.rdq           =  Scan Remote Desktop Work Queues for active threads and queued work items
!WinDE.rdy           =  List Ready threads
!WinDE.re            =  Dump RTL_RESOURCE lock, and its waiter/owner threads
!WinDE.regref        =  Show Registry key Reference counts
!WinDE.rootkit       =  Check for RootKit-style hooked system services
!WinDE.rpc           =  Show client information for an RPC message
!WinDE.rw            =  Dump RTL_SRWLOCK lock, and its waiter/owner threads
!WinDE.sessionview   -  Dumps Session View Space for the current session
!WinDE.shutdown      =  List shutdown-related data in WINLOGON
!WinDE.sl            =  Dump Kernel SpinLock
!WinDE.spl           =  List SPOOLSV Spooler/Printer/Driver/Port/Monitor/Job status
!WinDE.spltype       =  Dump a Spooler structure based on its embedded signature
!WinDE.srvendp       =  List Server service Endpoints
!WinDE.srvfiles      =  List Server service Files
!WinDE.srvsess       =  List Server service Sessions
!WinDE.srvtrees      =  List Server service Tree connections
!WinDE.stax          =  Scan processes for thread stacks relevant to hung systems
!WinDE.svc           =  List SERVICES Service records
!WinDE.svq           =  Scan Server Work Queues for active threads and queued work items
!WinDE.t             =  Dump and analyze a thread
!WinDE.tu            =  Dump and analyze a thread, with user symbols reload
!WinDE.tag           =  Find pool tag string in loaded modules
!WinDE.ticks         =  Convert a thread Elapsed Ticks value of n ticks to HH:MM:SS.mmm
!WinDE.update        -  Check for (and update to) the latest version
!WinDE.waittime      -  Display wait times for all threads in all processes
!WinDE.watson        =  Find AeDebug Debugger process, and debug its parent
!WinDE.wsq           =  Scan Workstation Work Queues for active threads and queued work items
!WinDE.zp            =  List Zombie Processes - Process objects not in the active process list
!WinDE.zt            =  List Zombie Threads - Thread objects no longer in the active process thread lists


View help on WinDE commands relevant to:

     Strategy  Settings  Hangs  Leaks  Processes  Threads  IRPs  Services : EventLog  Spooler  Server  WinLogon
```

The **`!WinDE.info`** command provides general information about the current debug session. This includes details such as the Windows version, build, and edition, the kernel base address, the system uptime, and the bugcheck code with arguments. It also displays CPU information, including the manufacturer, model, and features for each processor core.

```
0: kd> !WinDe.info

\\\
 >>>  WinDE Version 2.0.2.5 Debug Target Information:
///

vertarget
Windows 10 Kernel Version 19041 MP (8 procs) Free x64
Product: WinNt, suite: TerminalServer SingleUserTS
Edition build lab: 19041.1.amd64fre.vb_release.191206-1406
Kernel base = 0xfffff801`22000000 PsLoadedModuleList = 0xfffff801`22c2a2b0
Debug session time: Thu Nov 24 01:43:05.332 2022 (UTC + 1:00)
System Uptime: 0 days 3:17:48.015

.bugcheck
Bugcheck code 00000133
Arguments 00000000`00000001 00000000`00001e00 fffff801`22cfb320 00000000`00000000

!cpuinfo
CP  F/M/S Manufacturer  MHz PRCB Signature    MSR 8B Signature Features ArchitectureClass
 0  6,158,13 GenuineIntel 3600 000000b800000000                   3d1b3fff 0
 1  6,158,13 GenuineIntel 3600 000000b800000000                   3d1b3fff 0
 2  6,158,13 GenuineIntel 3600 000000b800000000                   3d1b3fff 0
 3  6,158,13 GenuineIntel 3600 000000b800000000                   3d1b3fff 0
 4  6,158,13 GenuineIntel 3600 000000b800000000                   3d1b3fff 0
 5  6,158,13 GenuineIntel 3600 000000b800000000                   3d1b3fff 0
 6  6,158,13 GenuineIntel 3600 000000b800000000                   3d1b3fff 0
 7  6,158,13 GenuineIntel 3600 000000b800000000                   3d1b3fff 0
                      Cached Update Signature 000000b800000000
                     Initial Update Signature 000000b800000000

!WinDE.debug

Debug session time : 2022-11-24 00:43:05.000

Kernel-Only Dump, AMD x64, 64-bit Addresses, PAE=n/a, 3GB=n/a.

Windows Windows 7 (NO Service Pack).
```

The **`!WinDE.activity`** performs an automated analysis of system activity, specifically looking for common hang symptoms and other issues. The output contains, but not limited to the following:

- Information about the idle process and its threads
- Overview of the state of all threads
- Lists threads that are ready to run or on standby
- Details of currently running threads
- Information on locks that might be causing thread blocks

```
0: kd> !winde.activity

\\\   
 >>>  Activity Analysis 0 - Idle Process
///   

   Thread                       Ses Own  From  Bas   zContext      Kernel Time        User Time     Elapsed Time   Q-TSA  1- 2- 3- 4- 5- 6- 7- Handle  Virt  Commit    Work   Paged   Non-P                
NO. Count  Process               Id  Id    Id  Pri   Switches   d:hh:mm:ss.ttt   d:hh:mm:ss.ttt   d:hh:mm:ss.ttt  RrBLsrl Rn Ry Bk Lc ws wr Ls  Count Siz-M  Priv-K   Set-K    Pool    Pool  Image Name    

  0     8  !lp fffff801`22d24a00  0 0000  0000   0  229237237     20:32:55.656                0      3:17:44.080  R------  8  .  .  .  .  .  .      0     0      60       0       0       0  Idle 

                                          Cur  Bas   zContext      Kernel Time        User Time    Elapsed Ticks   Q-TSA   Thread    Wait                                      Waiting  | Overview | Start     
      NO.   Thread                   Id   Pri  Pri   Switches   d:hh:mm:ss.ttt   d:hh:mm:ss.ttt   d:hh:mm:ss.ttt  RrBLsrl   State  Reason  Notes        Waiting On             Function | Function | Function  

        1   !t fffff801`22d27a00    0000    0    0   30380688      2:12:35.890                0         2:00.109  R------ RUNNING  Exec                  (CPU)                 nt!KiIdleLoop+0x176
        2   !t ffffbe00`46be5240    0000    0    0   43351962      2:24:28.859                0         2:00.015  R------ RUNNING  Exec                  (CPU)                 intelppm!MWaitIdle+0x1f
        3   !t ffffbe00`46d60240    0000    0    0   27093924      2:32:18.625                0         2:00.046  R------ RUNNING  Exec                  (CPU)                 intelppm!MWaitIdle+0x1f
        4   !t ffffbe00`470f1240    0000    0    0   24335148      2:36:28.750                0         2:00.265  R------ RUNNING  Exec                  (CPU)                 intelppm!MWaitIdle+0x1f
        5   !t ffffbe00`4716b240    0000    0    0   30247335      2:41:13.890                0         2:00.328  R------ RUNNING  Exec                  (CPU)                 intelppm!MWaitIdle+0x1f
        6   !t ffffbe00`46db7240    0000    0    0   27164116      2:42:50.921                0         2:00.203  R------ RUNNING  Exec                  (CPU)                 intelppm!MWaitIdle+0x1f
        7   !t ffffbe00`472f0240    0000    0    0   23982670      2:50:08.812                0             .015  R------ RUNNING  Exec                  (CPU)                 intelppm!MWaitIdle+0x1f
        8   !t ffffbe00`473aa240    0000    0    0   22681394      2:32:49.906                0         2:00.656  R------ RUNNING  Exec                  (CPU)                 intelppm!MWaitIdle+0x1f

                                          Cur  Bas   zContext      Kernel Time        User Time    Elapsed Ticks   Q-TSA   Thread    Wait                                      Waiting  | Overview | Start     
      NO.   Thread                   Id   Pri  Pri   Switches   d:hh:mm:ss.ttt   d:hh:mm:ss.ttt   d:hh:mm:ss.ttt  RrBLsrl   State  Reason  Notes        Waiting On             Function | Function | Function  



\\\   
 >>>  Activity Analysis 1 - Quick Process Thread State Analysis
///   

   Thread                       Ses Own  From  Bas   zContext      Kernel Time        User Time     Elapsed Time   Q-TSA  1- 2- 3- 4- 5- 6- 7- Handle  Virt  Commit    Work   Paged   Non-P                
NO. Count  Process               Id  Id    Id  Pri   Switches   d:hh:mm:ss.ttt   d:hh:mm:ss.ttt   d:hh:mm:ss.ttt  RrBLsrl Rn Ry Bk Lc ws wr Ls  Count Siz-M  Priv-K   Set-K    Pool    Pool  Image Name    

  1   512  !lp ffff988d`77adb080  0 0004  0000   8   65348484        13:57.171                0      3:17:44.080  -rR--ri  .  9  4  .  . 25  .      0    24     236       0       0       0  System 
  5    12  !lp ffff988d`8b937140  0 03a0  0390  13      46288            1.859             .171      3:17:38.645  ---L--i  .  .  .  1  .  .  .      0 2101344    2108       0       0       0  csrss.exe 
  7    16  !lp ffff988d`85dbd0c0  1 03fc  03f0  13    3398603           47.953             .593      3:17:37.185  --RL--i  .  .  2  1  .  .  .      0 2101379    3680       0       0       0  csrss.exe 
 10    16  !lp ffff988d`863ce0c0  0 0364  0310   8     279776           14.312           46.218      3:17:36.955  --P----  .  .  1  .  .  .  .      0 2101380   12964       0       0       0  svchost.exe 
 18    23  !lp ffff988d`8728a0c0  1 05e4  0550  13    9566352         6:32.656        11:11.765      3:17:36.360  --R---i  .  .  1  .  .  .  .      0 2101715   91364       0       0       0  dwm.exe 
 20   111  !lp ffff988d`873ac0c0  0 061c  0310   8      49768            4.375            1.843      3:17:36.340  -r-----  .  2  .  .  .  .  .      0 2101474   18220       0       0       0  svchost.exe 
 38     5  !lp ffff988d`837680c0  0 09b8  0310   8       2413             .609             .140      3:17:36.232  ---L---  .  .  .  1  .  .  .      0 2101348    5424       0       0       0  svchost.exe 
 43     5  !lp ffff988d`86d350c0  0 0ab0  0310   8      45240           29.796            1.203      3:17:36.202  --P----  .  .  1  .  .  .  .      0    48    3712       0       0       0  AsusCertService.exe 
 50     6  !lp ffff988d`85755080  0 0c18  0310   8       5234            1.406             .593      3:17:36.129  ---L---  .  .  .  1  .  .  .      0 2105451    3012       0       0       0  svchost.exe 
 60    17  !lp ffff988d`86e33080  0 0df4  0310   8      31228            2.015            8.218      3:17:36.032  --PL---  .  .  1  4  .  .  .      0 2101369   12752       0       0       0  svchost.exe 
 61    41  !lp ffff988d`87133080  1 0e54  0968   8      90341            1.078            2.609      3:17:36.018  --R--r-  .  .  1  .  .  8  .      0 2101656   43780       0       0       0  NVDisplay.Container.exe 
 65     6  !lp ffff988d`83666080  0 0fe4  0f8c   8        244             .703             .359      3:17:35.785  ---L---  .  .  .  1  .  .  .      0 2101354    7280       0       0       0  dasHost.exe 
 68     7  !lp ffff988d`8e625080  0 112c  0364   8     255821           57.750           17.125      3:17:35.377  -rP--r-  .  1  1  .  .  1  .      0 2101348   14764       0       0       0  WmiPrvSE.exe 
102    18  !lp ffff988d`916cd080  0 1840  0310   8      18208            1.734            4.234      3:17:33.535  --P----  .  .  1  .  .  .  .      0 2101631   49448       0       0       0  svchost.exe 
141    38  !lp ffff988d`91cae080  1 1cf4  1a10  13    3505691             .515             .812      3:17:33.388  --R--r-  .  .  2  .  .  3  .      0  4457   76316       0       0       0  parsecd.exe 
152     2  !lp ffff988d`9289a0c0  1 25b0  1a38   8      19273             .015                0      3:17:32.794  --R----  .  .  1  .  .  .  .      0 2101349    1584       0       0       0  rundll32.exe 
155    81  !lp ffff988d`92558300  1 28c0  1648   8    1355543         2:15.312           47.203      3:17:32.254  -rR----  .  2  2  .  .  .  .      0  5054   43832       0       0       0  Corsair.Service.CpuIdRemote64.exe 
163    54  !lp ffff988d`9356b080  0 2dc0  19e4   8     147482            2.796           10.234      3:17:30.079  ---L-r-  .  .  .  1  . 25  .      0 2134597   51476       0       0       0  ModuleCoreService.exe 
166    70  !lp ffff988d`935cd080  0 2ee0  1960   8      58612            6.281           11.828      3:17:30.013  ---L-r-  .  .  .  1  . 27  .      0  4385   53244       0       0       0  MfeAVSvc.exe 
212    12  !lp ffff988d`96c93300  1 28b0  0774   8       7367            1.359             .984      3:15:21.507  --R--r-  .  .  1  .  .  1  .      0 2101438   11044       0       0       0  sihost.exe 
216    29  !lp ffff988d`96e640c0  1 1c4c  1a38   8      33090            5.906           21.484      3:15:21.442  --R--r-  .  .  1  .  .  4  .      0 2101578   55208       0       0       0  nvcontainer.exe 
221     7  !lp ffff988d`96eda080  1 2b68  0684   8       3760             .171             .203      3:15:21.410  --R----  .  .  1  .  .  .  .      0 2101632    9952       0       0       0  taskhostw.exe 
227   121  !lp ffff988d`970a8080  1 0fbc  0c64   8     848413           41.875           13.812      3:15:21.171  --RL-r-  .  .  5  1  .  3  .      0 2102055  153700       0       0       0  explorer.exe 
232   790  !lp ffff988d`9459d080  1 39fc  0684   8      51214            9.515            1.312      3:15:20.637  --R----  .  . 769  .  .  .  .      0  1132   90580       0       0       0  NahimicSvc32.exe 
236     5  !lp ffff988d`944d9080  1 3bfc  0310   8      29638             .046             .046      3:15:20.125  --R----  .  .  1  .  .  .  .      0 2101420    4184       0       0       0  svchost.exe 
240    19  !lp ffff988d`94ab4080  1 1990  0364   8      10934             .625            1.078      3:15:19.252  --R----  .  .  2  .  .  .  .      0 2101757   56680       0       0       0  StartMenuExperienceHost.exe 
242    69  !lp ffff988d`97c95080  1 1e00  0364   8      53671            1.921            7.390      3:15:18.872  ---L-r-  .  .  .  1  . 19  .      0 2135198  197756       0       0       0  SearchApp.exe 
243    15  !lp ffff988d`94af6080  1 3cdc  0364   8      14330            1.078             .796      3:15:18.617  --R----  .  .  1  .  .  .  .      0 2101471   13200       0       0       0  RuntimeBroker.exe 
256    28  !lp ffff988d`99221080  1 3d28  0fbc  13    5619077         1:00.109         1:10.078      3:15:11.655  --R--r-  .  .  3  .  .  3  .      0 2200519  327124       0       0       0  brave.exe 
258     5  !lp ffff988d`995950c0  1 3c7c  1a38   8    1001941             .187             .062      3:15:11.472  --R----  .  .  1  .  .  .  .      0  4225    3552       0       0       0  nvsphelper64.exe 
259    31  !lp ffff988d`9959c0c0  1 3cb8  22fc   8    5961521            2.843            1.906      3:15:11.440  --R--r-  .  .  2  .  .  3  .      0  4529   28676       0       0       0  NVIDIA Share.exe 
278    18  !lp ffff988d`99b0e080  1 4d70  0364   8      24315             .187             .171      3:15:09.420  --R----  .  .  2  .  .  .  .      0 2101672   34628       0       0       0  TextInputHost.exe 
284     3  !lp ffff988d`98e54080  1 527c  0fbc   8     377717            4.343             .890      3:15:06.636  --R----  .  .  1  .  .  .  .      0  4210    2144       0       0       0  tvnserver.exe 
286    74  !lp ffff988d`9a3020c0  1 53cc  53b8   8   17008288         1:00.765           50.328      3:15:05.908  --R--r-  .  .  3  .  . 14  .      0 54979  253960       0       0       0  iCUE.exe 
287    28  !lp ffff988d`99476080  1 53e4  0fbc   8      69806             .609             .312      3:15:05.870  --R--r-  .  .  1  .  .  8  .      0 2122806   36148       0       0       0  lghub.exe 
288   100  !lp ffff988d`9a213080  1 511c  53e4   8      70132            1.062            1.625      3:15:05.554  --R--r-  .  .  2  .  . 70  .      0  4478   52568       0       0       0  lghub_agent.exe 
291     4  !lp ffff988d`9aa24080  1 510c  0364   8       1701             .109             .031      3:15:04.793  --R----  .  .  2  .  .  .  .      0 2101352    2876       0       0       0  unsecapp.exe 
295    16  !lp ffff988d`9ae26340  1 55e0  0364   8     824731             .203             .109      3:15:03.583  --R----  .  .  1  .  .  .  .      0  4925   38660       0       0       0  IGCC.exe 
296     2  !lp ffff988d`9b0f5080  1 5554  0fbc   8    1373301            4.218            8.968      3:15:02.270  --R----  .  .  1  .  .  .  .      0   213   68796       0       0       0  PowerPanel Personal.exe 
328     4  !lp ffff988d`9cc42080  1 5afc  0364   8     610576        19:31.359           35.140      3:14:47.864  --R----  .  .  1  .  .  .  .      0  4205    2764       0       0       0  Aac3572MbHal_x64.exe 
344    28  !lp ffff988d`9b259080  1 62a4  19e4   8       3417             .640             .656      3:14:21.555  --R--r-  .  .  1  .  . 14  .      0 2101450   13624       0       0       0  ModuleCoreService.exe 
361    15  !lp ffff988d`92e31340  0 4608  0310   8        668             .031             .062      3:01:56.328  ---L---  .  .  .  1  .  .  .      0 2101655   10788       0       0       0  svchost.exe 
363   186  !lp ffff988d`866680c0  1 09a4  0fbc   8   69985203         9:16.078      1:14:19.890      3:01:52.542  R-R--r-  1  .  1  .  .  4  .      0  5261  363456       0       0       0  LorexCloud.exe 
371    45  !lp ffff988d`9e1e3080  1 1038  13ac   8   15051554         3:12.296      1:56:11.031      2:53:57.658  -sR----  .  1  2  .  .  .  .      0 2135000  213020       0       0       0  obs64.exe 
378    21  !lp ffff988d`9d8dd080  1 3200  0364   8       1289             .171             .093      2:45:21.268  ---L-r-  .  .  .  1  .  4  .      0 2101722   39360       0       0       0  GameBar.exe 
393     4  !lp ffff988d`8e2c9080  0 145c  0684   6         18                0                0           29.701  --P----  .  .  1  .  .  .  .      0    30    1096       0       0       0  CCleanerBrowserUpdate.exe 
394     4  !lp ffff988d`a107b080  0 64d8  2dc0   8         22                0                0           13.195  --P----  .  .  1  .  .  .  .      0 2101327    1348       0       0       0  McCBEntAndInstru.exe 
396     3  !lp ffff988d`8e2c8080  0 6618  6610   6          5                0                0            3.311  --P----  .  .  1  .  .  .  .      0 2101307     992       0       0       0  conhost.exe 

     1 Threads flagged as Running.
    15 Threads flagged as Ready.
   826 Threads flagged as Blocked.
    15 Threads flagged as waiting LPCs.

   857 Threads flagged in 48 Processes overall.

  5272 Threads in 396 Processes in total.


\\\   
 >>>  Activity Analysis 2 - Remote Desktop Work Queues
///   
fffff801`5844bc20 rdpdr!_TSQUEUE = ffff988d`8bb9fc80        : 0 Work Items Queued
Unable to load image \SystemRoot\System32\drivers\FnetHyramAS.sys, Win32 error 0n2
Unable to load image \SystemRoot\system32\DRIVERS\vsock.sys, Win32 error 0n2
Unable to load image \SystemRoot\System32\drivers\vmci.sys, Win32 error 0n2
Unable to load image \SystemRoot\System32\drivers\iaStorAC.sys, Win32 error 0n2
Unable to load image \SystemRoot\system32\drivers\mfehidk.sys, Win32 error 0n2
Unable to load image \SystemRoot\system32\DRIVERS\VBoxDrv.sys, Win32 error 0n2
Page 7f03f not present in the dump file. Type ".hh dbgerr004" for details
Unable to load image \SystemRoot\System32\DriverStore\FileRepository\nvmoduletracker.inf_amd64_0c1cc60a4b422185\NvModuleTracker.sys, Win32 error 0n2
Unable to load image \SystemRoot\system32\drivers\nvvad64v.sys, Win32 error 0n2
Unable to load image \SystemRoot\System32\drivers\CorsairVBusDriver.sys, Win32 error 0n2
Unable to load image \SystemRoot\system32\drivers\nvhda64v.sys, Win32 error 0n2
Unable to load image \SystemRoot\system32\drivers\RTKVHD64.sys, Win32 error 0n2
Unable to load image \SystemRoot\system32\drivers\mfefirek.sys, Win32 error 0n2
Unable to load image \SystemRoot\system32\DRIVERS\mfencbdc.sys, Win32 error 0n2
Unable to load image \??\C:\Program Files\BlueStacks_nxt\BstkDrv_nxt.sys, Win32 error 0n2
Page 1a9a7b not present in the dump file. Type ".hh dbgerr004" for details
Unable to load image \SystemRoot\system32\DRIVERS\hcmon.sys, Win32 error 0n2
Unable to load image \SystemRoot\System32\DriverStore\FileRepository\xvdd.inf_amd64_eae73d4477526335\xvdd.sys, Win32 error 0n2
Page 1c2fad not present in the dump file. Type ".hh dbgerr004" for details

    No Active Remote Desktop threads or Queued Work Items found


\\\   
 >>>  Activity Analysis 3 - Workstation Work Queues
///   
fffff801`3b8ebac0 rdbss!RxActiveContexts       : 0 Work Items Queued


    ERROR: Cannot find rdbss!RxDispatcherWorkQueues
    ERROR: Cannot find Workstation Work Queues - Cannot check for Queued Work Items

        1   !t ffff988d`80be4040    01fc    8    8          1                0                0      3:17:44.187  -------    Wait  Exec    ACTIVE>                             rdbss!RxpIdleWorkerThread+0x25

     1 Active Workstation Work Queue Thread found
     0 Queued Workstation Work Items found


\\\   
 >>>  Activity Analysis 4 - Server Work Queues
///   

    ERROR: Unable to determine srv!_WORK_QUEUE size

    ERROR: Cannot find srv!SrvMultiProcessorDriver
    ERROR: Cannot find srv!SrvWorkQueues
    ERROR: Cannot find srv!SrvBlockingWorkQueues
    ERROR: Cannot find SRV Work Queues - Cannot check for Queued Work Items


     0 Active Server Work Queue Threads found


\\\   
 >>>  Activity Analysis 5 - Executive Work Queues
///   
    No Active Threads or Work Items found


\\\   
 >>>  Activity Analysis 6 - Ready Threads
///   

                                                          Cur  Bas   zContext      Kernel Time        User Time    Elapsed Ticks   Q-TSA   Thread    Wait                                      Waiting  | Overview | Start     
PROC Image Name       No.   Thread                   Id   Pri  Pri   Switches   d:hh:mm:ss.ttt   d:hh:mm:ss.ttt   d:hh:mm:ss.ttt  RrBLsrl   State  Reason  Notes        Waiting On             Function | Function | Function  

              CPU 00: 
0004 System             1   !t ffff988d`a1ed8040    4cf4   30   12          1                0                0         1:03.843  -r---r-  Ready   Preempt                                     
061C svchost.exe        2   !t ffff988d`8bd0b0c0    0af8   15   15       4474                0             .015         1:59.953  -r-----  Ready   UserRq                                      
28C0 Corsair.Servic     3   !t ffff988d`93206080    2988   15   15      12398             .031                0         1:59.515  -r-----  Ready   Delay                                       
061C svchost.exe        4   !t ffff988d`91492080    1f10   15   15        478                0                0         1:59.390  -r-----  Ready   UserRq                                      
03FC csrss.exe          5   !t ffff988d`99c52040    3290   13   12       9445             .046                0         1:13.703  -r---r-  Ready   Preempt                                     
112C WmiPrvSE.exe       6   !t ffff988d`9e83f080    13c0   12    8      89264            4.906            1.640           13.203  -r---r-  Ready   Preempt                                     
0004 System             7   !t ffff988d`7d5ce040    018c    8    8   12365520           43.484                0         2:00.000  -r-----  Ready   Exec                                        
28C0 Corsair.Servic     8   !t ffff988d`9321a080    2970    8    8      16754             .171             .015         1:59.375  -r-----  Ready   Delay                                       
0004 System             9   !t ffff988d`9acb3040    2698    2    2      22510             .156                0           19.781  -r---r-  Ready   Preempt                                     
0004 System            10   !t ffff988d`96ae0040    4b48    2    2      14306             .015                0           19.781  -r---r-  Ready   Preempt                                     
0004 System            11   !t ffff988d`92fb1040    21fc    2    2      14181             .109                0           19.781  -r---r-  Ready   Preempt                                     
0004 System            12   !t ffff988d`77be4040    005c    0    0      29946             .890                0         1:59.000  -r-----  Ready   Exec                                        

              CPU 01: No Ready Threads
              CPU 02: No Ready Threads
              CPU 03: No Ready Threads
              CPU 04: No Ready Threads
              CPU 05: No Ready Threads
              CPU 06: No Ready Threads
              CPU 07: No Ready Threads

    12 Ready Threads found


\\\   
 >>>  Activity Analysis 7 - Standby Threads
///   

                                                          Cur  Bas   zContext      Kernel Time        User Time    Elapsed Ticks   Q-TSA   Thread    Wait                                      Waiting  | Overview | Start     
PROC Image Name       No.   Thread                   Id   Pri  Pri   Switches   d:hh:mm:ss.ttt   d:hh:mm:ss.ttt   d:hh:mm:ss.ttt  RrBLsrl   State  Reason  Notes        Waiting On             Function | Function | Function  

0004 System             0   !t ffff988d`8ea16380    035c   30   16   21711844         7:12.218                0         2:00.000  -s---r- Standby  Preempt               (CPU)                 nt!KiSwapContext+0x76
1038 obs64.exe          2   !t ffff988d`9f2eb5c0    1df0    8    8    4467120         1:24.578        14:22.906             .015  -s----- Standby* UserRq                (CPU)                 nt!KiSystemServiceCopyEnd+0x25
0004 System             4   !t ffff988d`77b3d040    0048   17    8      11414             .187                0            1.000  -s----- Standby* Exec                  (CPU)                 nt!KeBalanceSetManager+0xb3

     3 Standby Threads found


\\\   
 >>>  Activity Analysis 8 - Running Threads
///   

                                                          Cur  Bas   zContext      Kernel Time        User Time    Elapsed Ticks   Q-TSA   Thread    Wait                                      Waiting  | Overview | Start     
PROC Image Name       No.   Thread                   Id   Pri  Pri   Switches   d:hh:mm:ss.ttt   d:hh:mm:ss.ttt   d:hh:mm:ss.ttt  RrBLsrl   State  Reason  Notes        Waiting On             Function | Function | Function  

09A4 LorexCloud.exe     0   !t ffff988d`97024580    3a24   15   15    1182042             .062             .046           17.718  R----r- RUNNING  Callout               (CPU)                 nt!KeBugCheckEx+0x0

     1 Running Thread found


\\\   
 >>>  Activity Analysis 9 - Queued DPCs
///   

Processor 0:  0 DPCs          DpcRoutineActive = 00000000`00000001   DpcStack = fffff801`27e69fb0   

    No Queued DPCs found


\\\   
 >>>  Activity Analysis A - Processor Summary
///   

    CPU#  Status  BugChk  Running Thread           Idle Thread              Standby Thread           Ready Threads  DPCs Queued

     00   ACTIVE  BUGCHK  !t ffff988d`97024580     !t fffff801`22d27a00     !t ffff988d`8ea16380        12 Ready       0     
     01    idle                                    !t ffffbe00`46be5240                                  0             0     
     02    idle                                    !t ffffbe00`46d60240     !t ffff988d`9f2eb5c0         0             0     
     03    idle                                    !t ffffbe00`470f1240                                  0             0     
     04    idle                                    !t ffffbe00`4716b240     !t ffff988d`77b3d040         0             0     
     05    idle                                    !t ffffbe00`46db7240                                  0             0     
     06    idle                                    !t ffffbe00`472f0240                                  0             0     
     07    idle                                    !t ffffbe00`473aa240                                  0             0     


    Active Threads on each processor:
                                                          Cur  Bas   zContext      Kernel Time        User Time    Elapsed Ticks   Q-TSA   Thread    Wait                                      Waiting  | Overview | Start     
PROC Image Name       No.   Thread                   Id   Pri  Pri   Switches   d:hh:mm:ss.ttt   d:hh:mm:ss.ttt   d:hh:mm:ss.ttt  RrBLsrl   State  Reason  Notes        Waiting On             Function | Function | Function  

09A4 LorexCloud.exe     0   !t ffff988d`97024580    3a24   15   15    1182042             .062             .046           17.718  R----r- RUNNING  Callout               (CPU)                 nt!KeBugCheckEx+0x0
0004 System             1   !t ffffbe00`46be5240    0000    0    0   43351962      2:24:28.859                0         2:00.015  R------ RUNNING  Exec                  (CPU)                 intelppm!MWaitIdle+0x1f
0004 System             2   !t ffffbe00`46d60240    0000    0    0   27093924      2:32:18.625                0         2:00.046  R------ RUNNING  Exec                  (CPU)                 intelppm!MWaitIdle+0x1f
0004 System             3   !t ffffbe00`470f1240    0000    0    0   24335148      2:36:28.750                0         2:00.265  R------ RUNNING  Exec                  (CPU)                 intelppm!MWaitIdle+0x1f
0004 System             4   !t ffffbe00`4716b240    0000    0    0   30247335      2:41:13.890                0         2:00.328  R------ RUNNING  Exec                  (CPU)                 intelppm!MWaitIdle+0x1f
0004 System             5   !t ffffbe00`46db7240    0000    0    0   27164116      2:42:50.921                0         2:00.203  R------ RUNNING  Exec                  (CPU)                 intelppm!MWaitIdle+0x1f
0004 System             6   !t ffffbe00`472f0240    0000    0    0   23982670      2:50:08.812                0             .015  R------ RUNNING  Exec                  (CPU)                 intelppm!MWaitIdle+0x1f
0004 System             7   !t ffffbe00`473aa240    0000    0    0   22681394      2:32:49.906                0         2:00.656  R------ RUNNING  Exec                  (CPU)                 intelppm!MWaitIdle+0x1f

\\\   
 >>>  Activity Analysis B - Blocking Locks Summary
///   
Invalid Thread(2) 00060001 Type = 0, should be 6.
Invalid Thread(2) 00060001 Type = 0, should be 6.
Invalid Thread(2) 00060001 Type = 0, should be 6.
Invalid Thread(2) 00060001 Type = 0, should be 6.
Invalid Thread(2) 00060001 Type = 0, should be 6.
Invalid Thread(2) 00060001 Type = 0, should be 6.
Invalid Thread(2) 00060001 Type = 0, should be 6.
Invalid Thread(2) 00060001 Type = 0, should be 6.
      No.  Tag  Type Lock Addr           Waiters  Owners  Owner Thread                  Owner Wait Time   Tag  Owner Waiting On       Owner Function
        1  NVRM  !er ffff988d`80d05a90       3       1    !t ffff988d`9a2f0040               2m:00.000s  NVRM  !se ffff988d`8b83b950  nvlddmkm+0xab26c 
        2  ????  !er ffff988d`8cfa3430       1       1    !t ffff988d`8073e040                   3.000s        !locks                 nvlddmkm+0xb8776 
        3  User  !er ffff988d`8eb65e10     809       1    !t ffff988d`92acb080                   2.468s        !locks                 win32kbase!EnterSharedCrit+0x7f 
        4  Gsem  !er ffff988d`86ef4cf0       1       2 *  !t ffff988d`8fe0a080                   2.734s        !locks                 dxgkrnl!DXGDEVICEACCESSLOCKSHARED::Acquire+0xce 
        5  DxgK  !er ffff988d`8eb6d290       2       1    !t ffff988d`80e170c0               1m:59.937s        ???                    dxgmms2!VidSchWaitForEvents+0x9c 
        6  Job   !er ffff988d`86b440b8       1       1    !t ffff988d`9e83f080                  13.203s        CPU - READY thread      

    6 blocking locks with 817 waiters found (108531 locks scanned) - for details run !blocks, !blocks <No.>, or one of the commands listed in the above summary


\\\   
 >>>  Activity Analysis C - Check for AeDebug Debugger
///   
    No processes running DRWTSN32.EXE found in the active process list


\\\   
 >>>  Activity Analysis D - Check for Stuck Disk IO
///   

Object: ffff988d7d5d0380  Type: (ffff988d77b42820) Driver
    ObjectHeader: ffff988d7d5d0350 (new version)
    HandleCount: 0  PointerCount: 6
    Directory Object: ffffce0142f745e0  Name: disk

  Disk Device             Name                                DevObj             DevExt             RemoveLock  FdoData            IRPs     Packets  Errors   


    No Stuck Disk IRPs found


\\\   
 >>>  Activity Analysis Summary - Points of Interest for Followup :
///   
  ***  1 Active DPC found - Look into that right away!
   **  6 Blocking Locks found. Run !WinDE.blocks for details.
    *  1 Running Thread found.
    *  1 Active Workstation Worker Thread found. Run !WinDE.wsq or !WinDE.wsq /D for details.
    +  12 Ready Threads found. Run !WinDE.r or !WinDE.r /D for details.

    >  15 Threads flagged as Ready.
    >  826 Threads flagged as Blocked.
    >  15 Threads flagged as waiting LPCs.

    >  857 Threads flagged in 48 Processes overall.

  If further analysis of the above does not reveal anything obvious, try running !WinDE.leak / !WinDE.stax / !WinDE.irps / !WinDE.calls, etc., and look for clues there.
```

The **`!WinDE.blocks`** command analyzes and displays detailed information about various types of locks that are currently being held.

```
0: kd> !WinDE.blocks

\\\   
 >>>  Blocking Lock 1 - nt!_ERESOURCE 0xffff988d80d05a90
///   

!er ffff988d`80d05a90 (in "NVRM" Pool)


Resource @ 0xffff988d80d05a90    Exclusively owned
    Contention Count = 91801
    NumberOfExclusiveWaiters = 3
     Threads: ffff988d9a2f0040-01<*> 

     Threads Waiting On Exclusive Access:
              ffff988d8073e040       ffff988d93199080       ffff988d87a680c0       
1 total locks


                                                          Cur  Bas   zContext      Kernel Time        User Time    Elapsed Ticks   Q-TSA   Thread    Wait                                      Waiting  | Overview | Start     
PROC Image Name       No.   Thread                   Id   Pri  Pri   Switches   d:hh:mm:ss.ttt   d:hh:mm:ss.ttt   d:hh:mm:ss.ttt  RrBLsrl   State  Reason  Notes        Waiting On             Function | Function | Function  

   3 Exclusive Waiters:
Invalid Thread(2) 00060001 Type = 0, should be 6.

   1 Exclusive Owner:
0004 System             1   !t ffff988d`9a2f0040    5908   16   12     140456            1.406                0         2:00.000  -------    Wait  Exec                 !se ffff988d`8b83b950  nvlddmkm+0xab26c


\\\   
 >>>  Blocking Lock 2 - nt!_ERESOURCE 0xffff988d8cfa3430
///   

!er ffff988d`8cfa3430 


Resource @ 0xffff988d8cfa3430    Exclusively owned
    Contention Count = 5876
    NumberOfExclusiveWaiters = 1
     Threads: ffff988d8073e040-01<*> 

     Threads Waiting On Exclusive Access:
              ffff988d93a0b040       
1 total locks


                                                          Cur  Bas   zContext      Kernel Time        User Time    Elapsed Ticks   Q-TSA   Thread    Wait                                      Waiting  | Overview | Start     
PROC Image Name       No.   Thread                   Id   Pri  Pri   Switches   d:hh:mm:ss.ttt   d:hh:mm:ss.ttt   d:hh:mm:ss.ttt  RrBLsrl   State  Reason  Notes        Waiting On             Function | Function | Function  

   1 Exclusive Waiter:
Invalid Thread(2) 00060001 Type = 0, should be 6.

   1 Exclusive Owner:
0004 System             1   !t ffff988d`8073e040    01dc   13   13     272745            2.484                0            3.000  --R----    Wait !er                   !ev ffff8087`a785ef30  nvlddmkm+0xb8776


\\\   
 >>>  Blocking Lock 3 - nt!_ERESOURCE 0xffff988d8eb65e10
///   

!er ffff988d`8eb65e10 (in "User" Pool)


Resource @ 0xffff988d8eb65e10    Shared 1 owning threads
    Contention Count = 30051809
    NumberOfSharedWaiters = 9
    NumberOfExclusiveWaiters = 800
     Threads: ffff988d8e5e0080-01<*> ffff988d92acb080-01    ffff988d91c81080-01    ffff988d9d3f3080-01    
              ffff988d9145a080-01    ffff988d9317a080-01    ffff988d9224b080-01    ffff988da0bb4080-01    
              ffff988da13ec080-01    ffff988da0c1d080-01    

     Threads Waiting On Exclusive Access:
              ffff988d99402640       ffff988d9ac91080       ffff988d99203540       ffff988d86cce0c0       
              ffff988d88eac080       ffff988d94d94040       ffff988d94d85040       ffff988d984dd040       
              ffff988d9e8d9040       ffff988d926dc080       ffff988d91fb0240       ffff988d9a2dc080       
              ffff988d933b2040       ffff988d93be2040       ffff988d985c1040       ffff988d984be080       
              ffff988d945a1040       ffff988d9e5d6040       ffff988da0bac040       ffff988d99ed1080       
              ffff988d939d7040       ffff988d9a9b1040       ffff988d9fce3040       ffff988d92d1e040       
              ffff988d9efe8040       ffff988d984df040       ffff988d9e308040       ffff988d9ac9b040       
              ffff988d9a9a8040       ffff988d97cbd040       ffff988da0b7e040       ffff988d9af4f040       
              ffff988da12d1040       ffff988da0be9040       ffff988d93edb040       ffff988d990cc040       
              ffff988d9a8d4040       ffff988d9a9cc080       ffff988d9c23f040       ffff988d999b0080       
              ffff988d93cc9040       ffff988da0c3e040       ffff988d98e67040       ffff988d93c20040       
              ffff988d97e47040       ffff988d981ca040       ffff988d97e5a040       ffff988d97f5d040       
              ffff988d93318040       ffff988d9e840040       ffff988d93e18040       ffff988d98676040       
              ffff988d96492080       ffff988d97408240       ffff988d93bce2c0       ffff988da0b47040       
              ffff988d93c73040       ffff988d93be7040       ffff988d9fbf5040       ffff988d9d4cb040       
              ffff988da11ef040       ffff988d97bb9040       ffff988d9c8f2040       ffff988d941ef040       
              ffff988d93ca4040       ffff988d920b4040       ffff988d94af4040       ffff988d8e83b040       
              ffff988d9e5ce040       ffff988d8fc41040       ffff988d9cfdd040       ffff988d8e47e080       
              ffff988da0b42240       ffff988da0b62240       ffff988d9cacf240       ffff988d9c5e8240       
              ffff988d97909040       ffff988d9fce5240       ffff988da1aa0240       ffff988d9ebe7240       
              ffff988da0a49240       ffff988d94837040       ffff988d9a2ec240       ffff988d992f3080       
              ffff988da1a8a240       ffff988d9f8f5540       ffff988d861bb040       ffff988d9b6ea0c0       
              ffff988d98675300       ffff988d83f66040       ffff988d8e09f040       ffff988d8e470040       
              ffff988d8e3a3040       ffff988d985a1580       ffff988da0ff5040       ffff988d92beb040       
              ffff988d981dd580       ffff988d9ffd9580       ffff988da1a9a240       ffff988d9e6ee040       
              ffff988d94b89040       ffff988d96e85080       ffff988d9f2df040       ffff988da18dd040       
              ffff988da11cc040       ffff988d9c7e9040       ffff988d9fcd6080       ffff988da12d6080       
              ffff988da0a5e040       ffff988d98dd9080       ffff988d97bb6080       ffff988d9b4ef040       
              ffff988d9fcc7040       ffff988da0b7b240       ffff988d9a1842c0       ffff988d94b85040       
              ffff988da1ec5080       ffff988da0c51040       ffff988da0df6380       ffff988d9e854040       
              ffff988d9f2d3040       ffff988d975be040       ffff988d9fff6040       ffff988d93bc9040       
              ffff988d975e9040       ffff988d93b45040       ffff988d93c7c040       ffff988da0b7c040       
              ffff988d9c5f5040       ffff988da1def040       ffff988d9f0ce040       ffff988d9c206040       
              ffff988d9c203040       ffff988d9a8d7040       ffff988da0c31040       ffff988d9e9312c0       
              ffff988d9e788040       ffff988da0b44040       ffff988d9e87a040       ffff988d9c237300       
              ffff988d97cee040       ffff988da0b85040       ffff988da0b83040       ffff988d9c8ee040       
              ffff988da0ef5040       ffff988d975bd040       ffff988da0b7f040       ffff988da0b7d040       
              ffff988d9f2d8040       ffff988d9fccd040       ffff988d924c5040       ffff988d9ecab040       
              ffff988d9eed7040       ffff988d9a2f1040       ffff988d9f6ee040       ffff988d91ce3040       
              ffff988d9d7ea040       ffff988d9c6e7080       ffff988d9f0c5080       ffff988d9e88a080       
              ffff988da1a98080       ffff988da1aed080       ffff988da0b13080       ffff988da1a87080       
              ffff988da1ee7080       ffff988d9caca100       ffff988d9d9cb240       ffff988d9e948240       
              ffff988da0a4b240       ffff988da0e68240       ffff988d9f0cf2c0       ffff988d8f8c9080       
              ffff988d96eee080       ffff988da1aa3040       ffff988da0c5b040       ffff988d9cae3040       
              ffff988d9e791040       ffff988d923a1040       ffff988d949cf040       ffff988d9a2f6040       
              ffff988da18ea040       ffff988d9fae4040       ffff988da11de040       ffff988d94408040       
              ffff988da11e4040       ffff988d9c8e7040       ffff988d9ecb2040       ffff988d92681040       
              ffff988da11ed040       ffff988d9c208040       ffff988d9fcd8040       ffff988d9c7ea040       
              ffff988da0bdd040       ffff988da18de040       ffff988d9f7e7040       ffff988d9c150040       
              ffff988d94b7f040       ffff988d9f6eb0c0       ffff988da0c57080       ffff988da0aa6080       
              ffff988da0b35080       ffff988d94b76080       ffff988d970b6040       ffff988d9c202080       
              ffff988da0fdd040       ffff988da0fdc040       ffff988da0c58040       ffff988da1a46040       
              ffff988d975ed040       ffff988d8fc25040       ffff988d9e5d0040       ffff988d9a2f2040       
              ffff988d9e6c6040       ffff988d97409040       ffff988d8bd03040       ffff988d9e6d3040       
              ffff988d94918040       ffff988d9c6dd040       ffff988da0b88040       ffff988da0df3040       
              ffff988da1afc040       ffff988d97411040       ffff988da0a63040       ffff988da0abd040       
              ffff988d91775040       ffff988d98c1b040       ffff988da0a55040       ffff988da0b04040       
              ffff988da0b03040       ffff988da1a77040       ffff988da0c06040       ffff988da0c05040       
              ffff988d9c9ec040       ffff988d910e6040       ffff988d910e5040       ffff988d97405040       
              ffff988d97404040       ffff988da1a47040       ffff988da12d4040       ffff988da12d3040       
              ffff988d9f0d4040       ffff988d9f0d3040       ffff988da0eda040       ffff988da0b22040       
              ffff988da0b21040       ffff988d9f8f0040       ffff988d9f8ef040       ffff988da18d7040       
              ffff988da0fe2040       ffff988da0fe1040       ffff988da0ab2040       ffff988da0ab1040       
              ffff988d9e94b040       ffff988d9e94a040       ffff988da11e2040       ffff988da11e1040       
              ffff988d93c6d040       ffff988d93c6c040       ffff988d93c6b040       ffff988da1a79040       
              ffff988d9f0ef040       ffff988d9eec5540       ffff988d9f0ee040       ffff988d9f0ed040       
              ffff988da1ded040       ffff988da1dec040       ffff988da0ad4040       ffff988da0ad3040       
              ffff988d939c9080       ffff988da0ad2040       ffff988da11cf040       ffff988da11cd040       
              ffff988da0fd9040       ffff988da0fd8040       ffff988d9d4d5040       ffff988da0a45040       
              ffff988da0b05080       ffff988d96e93080       ffff988da0e71040       ffff988da0ac1040       
              ffff988da0ac0040       ffff988da18d8040       ffff988da0abf040       ffff988da0abe040       
              ffff988da0ed8040       ffff988da0ed7040       ffff988da0ed6040       ffff988da11f4040       
              ffff988da11f3040       ffff988da11f2040       ffff988da0b5a040       ffff988da0b58040       
              ffff988d9f5d4040       ffff988d9f5d3040       ffff988d9f5d2040       ffff988d9c20d040       
              ffff988d9c20c040       ffff988da0b3c040       ffff988da0b3b040       ffff988da0b6e040       
              ffff988da0b6d040       ffff988da0b6c040       ffff988da0b6b040       ffff988da11ce040       
              ffff988da0b6a040       ffff988da0b69040       ffff988d91fb5040       ffff988d91fb4040       
              ffff988d91fb2040       ffff988d91fb1040       ffff988da0fd0040       ffff988da0fcf040       
              ffff988da0fce040       ffff988da0fcc040       ffff988d9f0f3040       ffff988d9f0f2040       
              ffff988d9f0f0040       ffff988d94105040       ffff988d9e1ee040       ffff988d9e1ed040       
              ffff988d9e1ec040       ffff988da0b59040       ffff988d9e1eb040       ffff988d9e1ea040       
              ffff988d93eca040       ffff988da0ad0040       ffff988da0acf040       ffff988d9f0ca040       
              ffff988d9f0c9040       ffff988d9f0c7040       ffff988d9f0c6040       ffff988da1cde040       
              ffff988da1cdd040       ffff988da1cdb040       ffff988da1cda040       ffff988da1cd9040       
              ffff988da1cd7040       ffff988da0b10040       ffff988da0b0f040       ffff988da0b0e040       
              ffff988da0b0d040       ffff988da0b0c040       ffff988da0b0b040       ffff988da0b0a040       
              ffff988da0b09040       ffff988da0fcd040       ffff988da0bf4040       ffff988da0bf3040       
              ffff988da0bf2040       ffff988da0bf0040       ffff988da0bef040       ffff988da0bee040       
              ffff988d9fcdf040       ffff988d9fcde040       ffff988d9fcdd040       ffff988d9fcdc040       
              ffff988da0b40040       ffff988d9f0cb040       ffff988d9c5ef040       ffff988d9c5ee040       
              ffff988d9c5ed040       ffff988d9c5ec040       ffff988d9c5eb040       ffff988da1cdc040       
              ffff988d91076040       ffff988d91074040       ffff988d91073040       ffff988d91072040       
              ffff988d91071040       ffff988d91070040       ffff988d9106f040       ffff988da1de5040       
              ffff988da1de4040       ffff988da1de2040       ffff988da1de1040       ffff988da1de0040       
              ffff988da1ddf040       ffff988da1a93040       ffff988da1a92040       ffff988da1a91040       
              ffff988da1a90040       ffff988da1a8e040       ffff988da1a8d040       ffff988da1a8c040       
              ffff988da0b3e040       ffff988d9c4e6080       ffff988da0b3d040       ffff988d9f0cd040       
              ffff988d9f0cc040       ffff988da0acd040       ffff988da1a4d040       ffff988da1a4c040       
              ffff988da1a4b040       ffff988da1a4a040       ffff988d9caf5040       ffff988d9caf4040       
              ffff988d9caf2040       ffff988d9caf1040       ffff988d9caef040       ffff988d9caee040       
              ffff988d9ebf4040       ffff988d9ebf3040       ffff988d9ebf2040       ffff988d9ebf1040       
              ffff988da1de3040       ffff988d9ebef040       ffff988d9ebed040       ffff988da0ab0040       
              ffff988da0aae040       ffff988d94104040       ffff988d9caf3040       ffff988da0aad040       
              ffff988da0aab040       ffff988da0aa9040       ffff988d9ebec040       ffff988d93cb6040       
              ffff988da0b20040       ffff988da0b1e040       ffff988da0b1c040       ffff988da0b1a040       
              ffff988da0aac040       ffff988da1a7e040       ffff988da1a7c040       ffff988d9af7b080       
              ffff988d9c6f3040       ffff988d9c6f2040       ffff988da0b1f040       ffff988d9c6f0040       
              ffff988d9c6ee040       ffff988d9f5f6040       ffff988d9f5f4040       ffff988da1a7d040       
              ffff988d9f5f2040       ffff988d9f5ef040       ffff988d9f5ed040       ffff988d92251080       
              ffff988d927c2080       ffff988da0b76040       ffff988d9c6ef040       ffff988d9e2d0040       
              ffff988da0ba2500       ffff988da0b72040       ffff988da0b71040       ffff988d9f5ee040       
              ffff988da0b6f040       ffff988d9432e040       ffff988d980c8040       ffff988da1a51040       
              ffff988d94a08040       ffff988da1a4f040       ffff988d9c5f2040       ffff988da0b18040       
              ffff988d8e173040       ffff988d9e1f0600       ffff988d984e4040       ffff988d984a5040       
              ffff988d9950b040       ffff988da0d6a040       ffff988da1a4e040       ffff988da0d69040       
              ffff988da0d67040       ffff988da0ea4040       ffff988da0ea3040       ffff988da0ea2040       
              ffff988d9e91a040       ffff988d92fb0040       ffff988da0e9f040       ffff988da0e9c040       
              ffff988da0d66040       ffff988d986e8040       ffff988da0e50040       ffff988da0e4e040       
              ffff988da0e4d040       ffff988d9d4d1040       ffff988da0e4a040       ffff988da0e9d040       
              ffff988d8e8c1040       ffff988da0cf7040       ffff988da0e4f040       ffff988da0cf6040       
              ffff988da0cf5040       ffff988da0e4b040       ffff988da1a82040       ffff988da1a81040       
              ffff988da0e49040       ffff988da1a80040       ffff988da1a85040       ffff988da1a84080       
              ffff988da0cf4040       ffff988da0cf2040       ffff988da0b48040       ffff988da1ed1040       
              ffff988da1ed0040       ffff988da1ece040       ffff988da1a7f040       ffff988da1ecc040       
              ffff988da1ecb040       ffff988da1ec6040       ffff988da1cf5040       ffff988da1ed3040       
              ffff988da1cf4040       ffff988da1cf3040       ffff988da1cf1040       ffff988da1cef040       
              ffff988da1eca040       ffff988d8e983080       ffff988da1cee040       ffff988da1ced040       
              ffff988da1ce9040       ffff988da1ce8040       ffff988d98084040       ffff988da1afb040       
              ffff988da1af9040       ffff988da1af6040       ffff988da0ee6080       ffff988da1cec040       
              ffff988da1af5040       ffff988da1af4040       ffff988da1af2040       ffff988da1aef040       
              ffff988da1afa040       ffff988da1aee040       ffff988d9f5cc040       ffff988da0d72040       
              ffff988da1dcc040       ffff988da1af3040       ffff988da1dcb040       ffff988da1dca040       
              ffff988da1dc9040       ffff988da1dc6040       ffff988da0d73040       ffff988da1dc5080       
              ffff988d9e5f5040       ffff988d9e5f3040       ffff988d9e5f0040       ffff988d9e5ef040       
              ffff988d857dd040       ffff988d9e5ee040       ffff988d9e5ec040       ffff988d9e5e9040       
              ffff988d9e5f4040       ffff988d9e5e8040       ffff988d9e5e7040       ffff988da1ef6040       
              ffff988da1ef3040       ffff988da1ef2040       ffff988d9e5ed040       ffff988da1ef1040       
              ffff988da1eef040       ffff988da1eec040       ffff988d9e5e6080       ffff988da1eeb040       
              ffff988da1eea040       ffff988da1ee9040       ffff988da0b52040       ffff988da0b51040       
              ffff988da1ef0040       ffff988da0b4f040       ffff988da0b4c040       ffff988da0d6f040       
              ffff988da0b87040       ffff988d8e31d040       ffff988d8e31c040       ffff988d8e31a040       
              ffff988d8e317040       ffff988da0b50040       ffff988d8e61b040       ffff988d8e316040       
              ffff988d8e315040       ffff988d8e063040       ffff988d8e062040       ffff988d8e622040       
              ffff988d8e31b040       ffff988d8e05d040       ffff988d8e05c040       ffff988d8e05b040       
              ffff988d8e314040       ffff988d9af15040       ffff988d8e057040       ffff988da1ee5040       
              ffff988da1ee3040       ffff988da1ee2040       ffff988da1ee1040       ffff988da1ede040       
              ffff988da1edd040       ffff988da1edc040       ffff988d8e056040       ffff988d93c1f040       
              ffff988da1ed7040       ffff988d938b4040       ffff988d91446040       ffff988da1df0040       
              ffff988d9b9e8040       ffff988d9c2cf040       ffff988d96e7a040       ffff988d9ac9e040       
              ffff988da1ed6080       ffff988d86cdf0c0       ffff988d9fae3500       ffff988d99af1040       
              ffff988d97e62040       ffff988da0fe5040       ffff988d98493040       ffff988d94b79040       
              ffff988d9cd41040       ffff988d93ee2040       ffff988d9adf1040       ffff988da0ad6040       
              ffff988d9704a040       ffff988d970ca040       ffff988d9e875040       ffff988d9eff4040       
              ffff988da0b74040       ffff988d9f7c7040       ffff988da1dd1040       ffff988da1dd0040       
              ffff988d9239b040       ffff988da1dcf040       ffff988d938b1040       ffff988da1dcd040       
              ffff988da0d6c040       ffff988d8e13b040       ffff988d8e137040       ffff988d8e136040       
              ffff988d8e3d6040       ffff988da1dd2040       ffff988d8e3d2040       ffff988d9483e080       
              ffff988d8e3d0040       ffff988da0d6e040       ffff988d8e3cf040       ffff988d8e3cd040       
              ffff988d8e3cb040       ffff988d8e3c8040       ffff988d8e3d3040       ffff988d8e37a040       
              ffff988d8e379040       ffff988d8e376040       ffff988d8e375040       ffff988d8e3cc040       
              ffff988d8e374040       ffff988d8e373040       ffff988d8e371040       ffff988d8e36e040       
              ffff988d8e36d080       ffff988d8e322040       ffff988d8e320040       ffff988d8e31e040       
              ffff988d8e4aa040       ffff988d8e372040       ffff988d8e4a8040       ffff988d8e4a5040       
              ffff988d8e4a4040       ffff988d8e4a1040       ffff988d8e4a0080       ffff988d8e442040       
              ffff988d8e441040       ffff988d8e440040       ffff988d8e43f040       ffff988d94de1040       
              ffff988d8e43d040       ffff988d8e43c040       ffff988d8e438040       ffff988d8e436040       
              ffff988d8e435080       ffff988d8e1e5040       ffff988d8e1e4040       ffff988d8e1e2040       
              ffff988d8e1e1040       ffff988d8e43a040       ffff988d88144080       ffff988d9228b040       
              ffff988d8e1db040       ffff988d8e1d8080       ffff988d9c6ec040       ffff988d8e19d040       
              ffff988d8e199040       ffff988d8e198040       ffff988d99c460c0       ffff988d8e197040       
              ffff988d8e196040       ffff988d9a1ca080       ffff988d8e195040       ffff988d8e192040       
              ffff988d8e193040       ffff988da0f0b040       ffff988da0f0a040       ffff988da0f08040       
              ffff988da0f05040       ffff988da0f04040       ffff988da0f03040       ffff988d994e5040       
              ffff988da11b4040       ffff988d9c8f0040       ffff988da11b0040       ffff988da11ae040       
              ffff988da11ac040       ffff988da0b02300       ffff988da11a9040       ffff988da11a7040       
              ffff988d8e13e040       ffff988da0fb4040       ffff988da11b1040       ffff988da0fb3040       
              ffff988da0fb1040       ffff988da11af040       ffff988da0fad040       ffff988da0faa040       
              ffff988da0fa9040       ffff988d938af040       ffff988d91bdc080       ffff988da08f5040       
              ffff988da08f2040       ffff988da08f0040       ffff988da0b4e040       ffff988da08ef040       
              ffff988da08ee040       ffff988da08eb040       ffff988da08e9040       ffff988da08e8040       
              ffff988d8e4ae040       ffff988da08f1040       ffff988d8e0c0040       ffff988d9bdf5040       
              ffff988da1a8f040       ffff988d8e0bd040       ffff988d8e0bc040       ffff988d8e0b9080       
              ffff988d8e032040       ffff988d8e031040       ffff988d8e02e040       ffff988d8e02d040       
              ffff988d8e02b040       ffff988d8e029040       ffff988d8e0bb040       ffff988d8e025040       
              ffff988da0be8040       ffff988d8e021040       ffff988d8e01f040       ffff988d8e01e040       
              ffff988d8e01b040       ffff988d8e01a040       ffff988d8e018040       ffff988da1edb040       
              ffff988d8e017040       ffff988d8e016040       ffff988d8e015040       ffff988d8e011040       
              ffff988d8e010040       ffff988d8e059040       ffff988d96dfa040       ffff988d8e00e040       
              ffff988d8e00c040       ffff988da1ecd040       ffff988d8e00b040       ffff988d9ccd6580       
              ffff988d8e00a040       ffff988d8e008040       ffff988d8e006040       ffff988d8e004040       
              ffff988da1eda040       ffff988da0f10040       ffff988d8e4ac040       ffff988d9a21b040       
              ffff988d8e2c3040       ffff988d8e009040       ffff988d8e2be080       ffff988d8e2bb040       
              ffff988d8e2b9040       ffff988d8e2b7040       ffff988d88999040       ffff988d8e2c4040       
              ffff988d8e2b1080       ffff988d8e2b0040       ffff988d8e2ae040       ffff988d8e0ea040       
              ffff988d8e0e8040       ffff988d8e0e5040       ffff988d941d9040       ffff988d8e0e1040       
              ffff988d8e0e0040       ffff988d8e0df040       ffff988d8e0dc040       ffff988d8e2ad080       
              ffff988d8e0d7040       ffff988d8e0d6040       ffff988d8e0e4040       ffff988d8e0d4040       
              ffff988d8e143040       ffff988d8e0cf040       ffff988d8e0cd040       ffff988d8e0cb040       
              ffff988d8e0ca080       ffff988d8e0c8040       ffff988d8e0c6040       ffff988d8e0c5040       
              ffff988d8e0d3040       ffff988d8e0c4040       ffff988d8e0c3040       ffff988d97fcf080       
1 total locks


                                                          Cur  Bas   zContext      Kernel Time        User Time    Elapsed Ticks   Q-TSA   Thread    Wait                                      Waiting  | Overview | Start     
PROC Image Name       No.   Thread                   Id   Pri  Pri   Switches   d:hh:mm:ss.ttt   d:hh:mm:ss.ttt   d:hh:mm:ss.ttt  RrBLsrl   State  Reason  Notes        Waiting On             Function | Function | Function  

 800 Exclusive Waiters:
Invalid Thread(2) 00060001 Type = 0, should be 6.

   9 Shared Waiters:
Invalid Thread(2) 00060001 Type = 0, should be 6.

   1 Shared Owner :
0E54 NVDisplay.Cont     1   !t ffff988d`8e5e0080    1088   15    8       2802             .156             .062            2.093  --R----    Wait !er                   !ev ffff8087`a92d56d8  win32kbase!EngAcquireSemaphore+0x23
1038 obs64.exe          2   !t ffff988d`92acb080    5d28   15    8     506627             .062             .046            2.468  --R----    Wait !er                   !ev ffff8087`a8a12828  win32kbase!EnterSharedCrit+0x7f
1CF4 parsecd.exe        3   !t ffff988d`91c81080    1cf8   15   13     127205             .031             .031            2.437  --R----    Wait !er                   !ev ffff8087`aaf8c868  win32kbase!EnterSharedCrit+0x7f
39FC NahimicSvc32.e     4   !t ffff988d`9d3f3080    2bdc   15    8       6037                0                0            2.046  --R----    Wait !er                   !ev ffff8087`aaa90868  win32kbase!EnterSharedCrit+0x7f
1CF4 parsecd.exe        5   !t ffff988d`9145a080    2468   15   13     146003             .171             .531            2.421  --R----    Wait !er                   !ev ffff8087`abdad7f8  win32kbase!EnterSharedCrit+0x7f
28C0 Corsair.Servic     6   !t ffff988d`9317a080    299c   15    8     344753           29.921           10.062            1.687  --R----    Wait !er                   !ev ffff8087`ac8bf7d8  win32kbase!NtUserGetKeyboardLayout+0x90
5AFC Aac3572MbHal_x     7   !t ffff988d`9224b080    2ff0   15    8       1825             .015             .046            2.687  --R----    Wait !er                   !ev ffff8087`b55637d8  win32kbase!NtUserGetKeyboardLayout+0x90
2B68 taskhostw.exe      8   !t ffff988d`a0bb4080    610c   15    8        124                0                0            1.390  --R----    Wait !er                   !ev ffff8087`b83e17d8  win32kbase!NtUserGetKeyboardLayout+0x90
0FBC explorer.exe       9   !t ffff988d`a13ec080    0128   15    8        489                0                0            1.125  --R----    Wait !er                   !ev ffff8087`a9be17d8  win32kbase!NtUserGetKeyboardLayout+0x90
510C unsecapp.exe      10   !t ffff988d`a0c1d080    42fc   15    8         27                0                0             .703  --R----    Wait !er                   !ev ffff8087`b76c37d8  win32kbase!NtUserGetKeyboardLayout+0x90


\\\   
 >>>  Blocking Lock 4 - nt!_ERESOURCE 0xffff988d86ef4cf0
///   

!er ffff988d`86ef4cf0 (in "Gsem" Pool)


Resource @ 0xffff988d86ef4cf0    Shared 2 owning threads
    Contention Count = 349
    NumberOfExclusiveWaiters = 1
     Threads: ffff988d926d8080-01<*> ffff988d8fe0a080-01<*> 

     Threads Waiting On Exclusive Access:
              ffff988d8e5e0080       
1 total locks


                                                          Cur  Bas   zContext      Kernel Time        User Time    Elapsed Ticks   Q-TSA   Thread    Wait                                      Waiting  | Overview | Start     
PROC Image Name       No.   Thread                   Id   Pri  Pri   Switches   d:hh:mm:ss.ttt   d:hh:mm:ss.ttt   d:hh:mm:ss.ttt  RrBLsrl   State  Reason  Notes        Waiting On             Function | Function | Function  

   1 Exclusive Waiter:
Invalid Thread(2) 00060001 Type = 0, should be 6.

   2 Shared Owners:
1038 obs64.exe          1   !t ffff988d`926d8080    1164   15    8    2385363           12.953           11.640            3.406  --R----    Wait !er                   !ev ffff8087`b1f82f90  dxgkrnl!DxgkpCddSyncGPUAccess+0x50c
03FC csrss.exe          2   !t ffff988d`8fe0a080    4e8c   15    8    5920683            3.046            3.093            2.734  --R----    Wait !er                   !ev ffff8087`ac61abc8  dxgkrnl!DXGDEVICEACCESSLOCKSHARED::Acquire+0xce


\\\   
 >>>  Blocking Lock 5 - nt!_ERESOURCE 0xffff988d8eb6d290
///   

!er ffff988d`8eb6d290 (in "DxgK" Pool)


Resource @ 0xffff988d8eb6d290    Exclusively owned
    Contention Count = 390
    NumberOfSharedWaiters = 1
    NumberOfExclusiveWaiters = 1
     Threads: ffff988d80e170c0-01<*> ffff988d8fe0a080-01    

     Threads Waiting On Exclusive Access:
              ffff988d926d8080       
1 total locks


                                                          Cur  Bas   zContext      Kernel Time        User Time    Elapsed Ticks   Q-TSA   Thread    Wait                                      Waiting  | Overview | Start     
PROC Image Name       No.   Thread                   Id   Pri  Pri   Switches   d:hh:mm:ss.ttt   d:hh:mm:ss.ttt   d:hh:mm:ss.ttt  RrBLsrl   State  Reason  Notes        Waiting On             Function | Function | Function  

   1 Exclusive Waiter:
Invalid Thread(2) 00060001 Type = 0, should be 6.

   1 Shared Waiter:
Invalid Thread(2) 00060001 Type = 0, should be 6.

   1 Exclusive Owner:
03FC csrss.exe          1   !t ffff988d`80e170c0    057c   15   14    1588582           35.640                0         1:59.937  -------    Wait* Exec                 !ev ffff8087`a7919740  dxgmms2!VidSchWaitForEvents+0x9c


\\\   
 >>>  Blocking Lock 6 - nt!_ERESOURCE 0xffff988d86b440b8
///   

!er ffff988d`86b440b8 (in "Job " Pool)


Resource @ 0xffff988d86b440b8    Shared 1 owning threads
    Contention Count = 1
    NumberOfExclusiveWaiters = 1
     Threads: ffff988d9e83f080-01<*> 

     Threads Waiting On Exclusive Access:
              ffff988d8ff92040       
1 total locks


                                                          Cur  Bas   zContext      Kernel Time        User Time    Elapsed Ticks   Q-TSA   Thread    Wait                                      Waiting  | Overview | Start     
PROC Image Name       No.   Thread                   Id   Pri  Pri   Switches   d:hh:mm:ss.ttt   d:hh:mm:ss.ttt   d:hh:mm:ss.ttt  RrBLsrl   State  Reason  Notes        Waiting On             Function | Function | Function  

   1 Exclusive Waiter:
Invalid Thread(2) 00060001 Type = 0, should be 6.

   1 Shared Owner :
112C WmiPrvSE.exe       1   !t ffff988d`9e83f080    13c0   12    8      89264            4.906            1.640           13.203  -r---r-  Ready   Preempt               (CPU)                 nt!KiSwapContext+0x76


   || 
   || Blocking Locks Summary
   || 

      No.  Tag  Type Lock Addr           Waiters  Owners  Owner Thread                  Owner Wait Time   Tag  Owner Waiting On       Owner Function
        1  NVRM  !er ffff988d`80d05a90       3       1    !t ffff988d`9a2f0040               2m:00.000s  NVRM  !se ffff988d`8b83b950  nvlddmkm+0xab26c 
        2  ????  !er ffff988d`8cfa3430       1       1    !t ffff988d`8073e040                   3.000s        !locks                 nvlddmkm+0xb8776 
        3  User  !er ffff988d`8eb65e10     809       1    !t ffff988d`92acb080                   2.468s        !locks                 win32kbase!EnterSharedCrit+0x7f 
        4  Gsem  !er ffff988d`86ef4cf0       1       2 *  !t ffff988d`8fe0a080                   2.734s        !locks                 dxgkrnl!DXGDEVICEACCESSLOCKSHARED::Acquire+0xce 
        5  DxgK  !er ffff988d`8eb6d290       2       1    !t ffff988d`80e170c0               1m:59.937s        ???                    dxgmms2!VidSchWaitForEvents+0x9c 
        6  Job   !er ffff988d`86b440b8       1       1    !t ffff988d`9e83f080                  13.203s        CPU - READY thread      

    6 blocking locks with 817 waiters found (108531 locks scanned) - for details run !blocks, !blocks <No.>, or one of the commands listed in the above summary
```

The **`!WinDE.rdy`** command lists all the threads that are in the "Ready" state for execution across all CPUs.

```
0: kd> !winde.rdy

\\\   
 >>>  Ready Threads
///   


                                                          Cur  Bas   zContext      Kernel Time        User Time    Elapsed Ticks   Q-TSA   Thread    Wait                                      Waiting  | Overview | Start     
PROC Image Name       No.   Thread                   Id   Pri  Pri   Switches   d:hh:mm:ss.ttt   d:hh:mm:ss.ttt   d:hh:mm:ss.ttt  RrBLsrl   State  Reason  Notes        Waiting On             Function | Function | Function  

              CPU 00: 
0004 System             1   !t ffff988d`a1ed8040    4cf4   30   12          1                0                0         1:03.843  -r---r-  Ready   Preempt                                     
061C svchost.exe        2   !t ffff988d`8bd0b0c0    0af8   15   15       4474                0             .015         1:59.953  -r-----  Ready   UserRq                                      
28C0 Corsair.Servic     3   !t ffff988d`93206080    2988   15   15      12398             .031                0         1:59.515  -r-----  Ready   Delay                                       
061C svchost.exe        4   !t ffff988d`91492080    1f10   15   15        478                0                0         1:59.390  -r-----  Ready   UserRq                                      
03FC csrss.exe          5   !t ffff988d`99c52040    3290   13   12       9445             .046                0         1:13.703  -r---r-  Ready   Preempt                                     
112C WmiPrvSE.exe       6   !t ffff988d`9e83f080    13c0   12    8      89264            4.906            1.640           13.203  -r---r-  Ready   Preempt                                     
0004 System             7   !t ffff988d`7d5ce040    018c    8    8   12365520           43.484                0         2:00.000  -r-----  Ready   Exec                                        
28C0 Corsair.Servic     8   !t ffff988d`9321a080    2970    8    8      16754             .171             .015         1:59.375  -r-----  Ready   Delay                                       
0004 System             9   !t ffff988d`9acb3040    2698    2    2      22510             .156                0           19.781  -r---r-  Ready   Preempt                                     
0004 System            10   !t ffff988d`96ae0040    4b48    2    2      14306             .015                0           19.781  -r---r-  Ready   Preempt                                     
0004 System            11   !t ffff988d`92fb1040    21fc    2    2      14181             .109                0           19.781  -r---r-  Ready   Preempt                                     
0004 System            12   !t ffff988d`77be4040    005c    0    0      29946             .890                0         1:59.000  -r-----  Ready   Exec                                        

              CPU 01: No Ready Threads
              CPU 02: No Ready Threads
              CPU 03: No Ready Threads
              CPU 04: No Ready Threads
              CPU 05: No Ready Threads
              CPU 06: No Ready Threads
              CPU 07: No Ready Threads

    12 Ready Threads found
```

The **`!WinDE.stax`** command is used to scan processes for thread stacks that may be causing the system to hang. It helps identify threads that are stuck or contributing to system performance issues.

```
0: kd> !WinDE.stax

                                                          Cur  Bas   zContext      Kernel Time        User Time    Elapsed Ticks   Q-TSA   Thread    Wait                                      Waiting  | Overview | Start     
PROC Image Name       No.   Thread                   Id   Pri  Pri   Switches   d:hh:mm:ss.ttt   d:hh:mm:ss.ttt   d:hh:mm:ss.ttt  RrBLsrl   State  Reason  Notes        Waiting On             Function | Function | Function  

0004 System             1   !t fffff801`22d27a00    0000    0    0   30380688      2:12:35.890                0         2:00.109  R------ RUNNING  Exec                  (CPU)                 nt!KiIdleLoop+0x176
0004 System             2   !t ffffbe00`46be5240    0000    0    0   43351962      2:24:28.859                0         2:00.015  R------ RUNNING  Exec                  (CPU)                 intelppm!MWaitIdle+0x1f
0004 System             3   !t ffffbe00`46d60240    0000    0    0   27093924      2:32:18.625                0         2:00.046  R------ RUNNING  Exec                  (CPU)                 intelppm!MWaitIdle+0x1f
0004 System             4   !t ffffbe00`470f1240    0000    0    0   24335148      2:36:28.750                0         2:00.265  R------ RUNNING  Exec                  (CPU)                 intelppm!MWaitIdle+0x1f
0004 System             5   !t ffffbe00`4716b240    0000    0    0   30247335      2:41:13.890                0         2:00.328  R------ RUNNING  Exec                  (CPU)                 intelppm!MWaitIdle+0x1f
0004 System             6   !t ffffbe00`46db7240    0000    0    0   27164116      2:42:50.921                0         2:00.203  R------ RUNNING  Exec                  (CPU)                 intelppm!MWaitIdle+0x1f
0004 System             7   !t ffffbe00`472f0240    0000    0    0   23982670      2:50:08.812                0             .015  R------ RUNNING  Exec                  (CPU)                 intelppm!MWaitIdle+0x1f
0004 System             8   !t ffffbe00`473aa240    0000    0    0   22681394      2:32:49.906                0         2:00.656  R------ RUNNING  Exec                  (CPU)                 intelppm!MWaitIdle+0x1f
0004 System             1   !t ffff988d`77b3d040    0048   17    8      11414             .187                0            1.000  -s----- Standby* Exec                  (CPU)                 nt!KeBalanceSetManager+0xb3
0004 System             2   !t ffff988d`77be4040    005c    0    0      29946             .890                0         1:59.000  -r-----  Ready   Exec                  (CPU)                 nt!MiZeroLargePages+0x2ef
0004 System             3   !t ffff988d`7d5ce040    018c    8    8   12365520           43.484                0         2:00.000  -r-----  Ready   Exec                  (CPU)                 ndis!ndisWaitForKernelObject+0x21
0004 System             4   !t ffff988d`8ea16380    035c   30   16   21711844         7:12.218                0         2:00.000  -s---r- Standby  Preempt               (CPU)                 nt!KiSwapContext+0x76
0004 System             5   !t ffff988d`96ae0040    4b48    2    2      14306             .015                0           19.781  -r---r-  Ready   Preempt               (CPU)                 nt!KiSwapContext+0x76
0004 System             6   !t ffff988d`9acb3040    2698    2    2      22510             .156                0           19.781  -r---r-  Ready   Preempt               (CPU)                 nt!KiSwapContext+0x76
03FC csrss.exe          7   !t ffff988d`99c52040    3290   13   12       9445             .046                0         1:13.703  -r---r-  Ready   Preempt               (CPU)                 nt!KiSwapContext+0x76
0004 System             8   !t ffff988d`92fb1040    21fc    2    2      14181             .109                0           19.781  -r---r-  Ready   Preempt               (CPU)                 nt!KiSwapContext+0x76
0004 System             9   !t ffff988d`a1ed8040    4cf4   30   12          1                0                0         1:03.843  -r---r-  Ready   Preempt               (CPU)                 nt!KiSwapContext+0x76
0214 smss.exe           1   !t ffff988d`80bdd040    0218   12   11        805             .125                0      3:17:37.218  -------    Wait* UserRq               !lp ffff988d`8b937140  nt!KiSystemServiceCopyEnd+0x25
03A0 csrss.exe          1   !t ffff988d`85c8a0c0    03e0   15   15          7                0                0      3:17:29.000  ---L---    Wait  LpcRply              !lm ffffce01491e2ba0 (!lp ffff988d`868bd0c0 svchost.exe, !t ffff988d`86adf080 ) nt!AlpcpSignalAndWait+0x143

    1 Threads waiting for LPC message replies found in the above process

03FC csrss.exe          1   !t ffff988d`86bbd0c0    0538   15   15         48             .015                0      3:11:47.859  ---L---    Wait  LpcRply              !lm ffffce0148fd4b60 (!lp ffff988d`868bd0c0 svchost.exe, !t ffff988d`86adf080 ) nt!AlpcpSignalAndWait+0x143

    1 Threads waiting for LPC message replies found in the above process

0364 svchost.exe        1   !t ffff988d`874020c0    062c    9    8       2426                0                0         1:12.187  --P----    Wait !pl                   !pl 00000000`00000000  nt!MiLockDynamicMemoryExclusive+0x19

    1 Locks with 1 Waiters found in the above process

      No.  Tag  Type Lock Addr           Waiters  Owners  Owner Thread                  Owner Wait Time   Tag  Owner Waiting On       Owner Function
        1  ????  !pl ffff988d`00000000       0     ???    ?? unknown                                ???        ???                    ???

    1 Locks with 1 Waiters found - for details run one of the commands listed in the above summary

041C fontdrvhost.ex     1   !t ffff988d`864680c0    0420    8    8         15                0                0      3:17:36.859  -------    Wait* UserRq                !t ffff988d`8648a0c0  nt!KiSystemServiceCopyEnd+0x25
0598 fontdrvhost.ex     1   !t ffff988d`871460c0    059c    8    8          6                0                0      3:17:36.359  -------    Wait* UserRq                !t ffff988d`8718a0c0  nt!KiSystemServiceCopyEnd+0x25
061C svchost.exe        1   !t ffff988d`8bd0b0c0    0af8   15   15       4474                0             .015         1:59.953  -r-----  Ready * UserRq                (CPU)                 nt!KiSystemServiceCopyEnd+0x25
061C svchost.exe        2   !t ffff988d`91492080    1f10   15   15        478                0                0         1:59.390  -r-----  Ready * UserRq                (CPU)                 nt!KiSystemServiceCopyEnd+0x25
0764 svchost.exe        1   !t ffff988d`87f8a0c0    07fc    9    8         11                0                0      3:17:36.312  -------    Wait  UserRq                !t ffff988d`880020c0  nt!KiSystemServiceCopyEnd+0x25
09B8 svchost.exe        1   !t ffff988d`8e826080    137c    8    8         16                0                0      3:17:31.656  ---L---    Wait  LpcRply              !lm ffffce014b6e5c70 (!lp ffff988d`8e6d4080 svchost.exe, !t ffff988d`8e82f080 ) nt!AlpcpSignalAndWait+0x143

    1 Threads waiting for LPC message replies found in the above process

0AB0 AsusCertServic     1   !t ffff988d`85677080    09b0    8    8      44669           29.687            1.046            5.265  --P----    Wait !pl                   !pl ffff988d`8e5bf080  nt!ExpGetProcessInformation+0xaeb

    1 Locks with 1 Waiters found in the above process

      No.  Tag  Type Lock Addr           Waiters  Owners  Owner Thread                  Owner Wait Time   Tag  Owner Waiting On       Owner Function
        1  ????  !pl ffff988d`00000000       0     ???    ?? unknown                                ???        ???                    ???

    1 Locks with 1 Waiters found - for details run one of the commands listed in the above summary

0C18 svchost.exe        1   !t ffff988d`96ad8080    0644    8    8         88                0                0         1:46.046  ---L---    Wait  LpcRply              !lm ffffce01670aca40 (!lp ffff988d`970a8080 explorer.exe, !t ffff988d`938b3080 ) nt!AlpcpSignalAndWait+0x143

    1 Threads waiting for LPC message replies found in the above process

0DF4 svchost.exe        1   !t ffff988d`91a7a080    078c    9    8      14161             .281             .718            5.953  ---L---    Wait  LpcRply              !lm ffffce016700f620 (!lp ffff988d`8e625080 WmiPrvSE.exe, !t ffff988d`a1dce080 ) nt!AlpcpSignalAndWait+0x143
0DF4 svchost.exe        2   !t ffff988d`93df0080    36d8    9    8       9262             .140             .468           13.203  ---L---    Wait  LpcRply              !lm ffffce016710a2b0 (!lp ffff988d`8e625080 WmiPrvSE.exe, !t ffff988d`9e83f080 ) nt!AlpcpSignalAndWait+0x143
0DF4 svchost.exe        3   !t ffff988d`96a4d080    4df8    8    8        373                0             .031         1:35.937  ---L---    Wait  LpcRply              !lm ffffce0167057040 (!lp ffff988d`9aa24080 unsecapp.exe, !t ffff988d`9af7b080 ) nt!AlpcpSignalAndWait+0x143
0DF4 svchost.exe        4   !t ffff988d`8e0d9080    1dcc    8    8          1                0                0           26.000  ---l---    Wait  LpcRply              !lm ffffce016729c040 (!lp ffff988d`9aa24080 unsecapp.exe) nt!AlpcpSignalAndWait+0x143
0DF4 svchost.exe        5   !t ffff988d`a10b1080    65b8    8    8          1                0                0            5.562  --P----    Wait !pl                   !lm ffffce016711f9d0  nt!PspChargeProcessWakeCounter+0x7c

    1 Locks with 1 Waiters found in the above process

      No.  Tag  Type Lock Addr           Waiters  Owners  Owner Thread                  Owner Wait Time   Tag  Owner Waiting On       Owner Function
        1  ????  !pl ffff988d`00000000       0     ???    ?? unknown                                ???        ???                    ???

    1 Locks with 1 Waiters found - for details run one of the commands listed in the above summary


    5 Threads waiting for LPC message replies found in the above process

0E54 NVDisplay.Cont     1   !t ffff988d`8e55b080    1070    8    8         74                0                0      3:15:18.171  -------    Wait* UserRq                !t ffff988d`8e5dc080  nt!KiSystemServiceCopyEnd+0x25
0FE4 dasHost.exe        1   !t ffff988d`8e824080    1384    8    8        103                0                0         2:19.437  ---L---    Wait  LpcRply              !lm ffffce01672c3850 (!lp ffff988d`8e6d4080 svchost.exe, !t ffff988d`8e831040 ) nt!AlpcpSignalAndWait+0x143

    1 Threads waiting for LPC message replies found in the above process

112C WmiPrvSE.exe       1   !t ffff988d`9e83f080    13c0   12    8      89264            4.906            1.640           13.203  -r---r-  Ready   Preempt               (CPU)                 nt!KiSwapContext+0x76
112C WmiPrvSE.exe       2   !t ffff988d`a1dce080    3664    9    8          2                0                0            5.953  --P----    Wait !pl                   !pl ffff988d`8e625080  nt!PspInsertThread+0xcf

    1 Locks with 1 Waiters found in the above process

      No.  Tag  Type Lock Addr           Waiters  Owners  Owner Thread                  Owner Wait Time   Tag  Owner Waiting On       Owner Function
        1  ????  !pl ffff988d`00000000       0     ???    ?? unknown                                ???        ???                    ???

    1 Locks with 1 Waiters found - for details run one of the commands listed in the above summary

1840 svchost.exe        1   !t ffff988d`9fad3580    6334    9    8         16                0                0            5.343  --P----    Wait !pl                   !pl ffff988d`8e5bf080  nt!ExpGetProcessInformation+0xaeb

    1 Locks with 1 Waiters found in the above process

      No.  Tag  Type Lock Addr           Waiters  Owners  Owner Thread                  Owner Wait Time   Tag  Owner Waiting On       Owner Function
        1  ????  !pl ffff988d`00000000       0     ???    ?? unknown                                ???        ???                    ???

    1 Locks with 1 Waiters found - for details run one of the commands listed in the above summary

1960 mfemms.exe         1   !t ffff988d`933af080    2d24    9    8          4                0                0      3:17:30.125  -------    Wait  UserRq                !t ffff988d`933aa080  nt!KiSystemServiceCopyEnd+0x25
19F4 NahimicService     1   !t ffff988d`920cc080    200c    9    8          5                0                0      3:17:33.203  -------    Wait  UserRq                !t ffff988d`920ee080  nt!KiSystemServiceCopyEnd+0x25
1A20 ppped.exe          1   !t ffff988d`933240c0    2304    8    8          6                0                0      3:17:24.875  -------    Wait  UserRq                !t ffff988d`932f6080  nt!KiSystemServiceCopyEnd+0x25
1A78 ROGLiveService     1   !t ffff988d`91cb1080    1d94    9    8         12                0                0      3:17:27.390  -------    Wait  UserRq                !t ffff988d`93ef9080  nt!KiSystemServiceCopyEnd+0x25
1ABC OriginWebHelpe     1   !t ffff988d`932e8080    22c0    5    4          4                0                0      3:17:31.875  -------    Wait  UserRq                !t ffff988d`93343080  nt!KiSystemServiceCopyEnd+0x25
2478 MMSSHOST.exe       1   !t ffff988d`922860c0    247c    9    8        460             .031             .031      3:17:30.062  -------    Wait  UserRq                !t ffff988d`935ba080  nt!KiSystemServiceCopyEnd+0x25
250C mfevtps.exe        1   !t ffff988d`9278d080    2510    9    8        138             .046                0      3:17:32.765  -------    Wait  UserRq                !t ffff988d`92950080  nt!KiSystemServiceCopyEnd+0x25
258C ProtectedModul     1   !t ffff988d`9288b080    2590   10    8        414             .031             .031      3:17:30.093  -------    Wait  UserRq                !t ffff988d`93606080  nt!KiSystemServiceCopyEnd+0x25
28C0 Corsair.Servic     1   !t ffff988d`93206080    2988   15   15      12398             .031                0         1:59.515  -r-----  Ready   Delay                 (CPU)                 nt!KiSwapContext+0x76
28C0 Corsair.Servic     2   !t ffff988d`9321a080    2970    8    8      16754             .171             .015         1:59.375  -r-----  Ready   Delay                 (CPU)                 nt!KiSwapContext+0x76
2DC0 ModuleCoreServ     1   !t ffff988d`9382f080    217c    9    8       5976             .234            2.703           13.203  ---L---    Wait  LpcRply              !lm ffffce016a6633b0 (!lp ffff988d`86e33080 svchost.exe, !t ffff988d`93c17040 ) nt!AlpcpSignalAndWait+0x143
2DC0 ModuleCoreServ     2   !t ffff988d`93fb2080    0160    9    8       8831             .359             .078           13.203  -------    Wait* UserRq               !lp ffff988d`a107b080  nt!KiSystemServiceCopyEnd+0x25

    1 Threads waiting for LPC message replies found in the above process

2EE0 MfeAVSvc.exe       1   !t ffff988d`935d80c0    2ee4    9    8        185             .078                0      3:17:29.875  -------    Wait  UserRq                !t ffff988d`935ef0c0  nt!KiSystemServiceCopyEnd+0x25
2EE0 MfeAVSvc.exe       2   !t ffff988d`93af4080    37f8    8    8          2                0                0      3:17:24.859  ---L---    Wait  LpcRply              !lm ffffce014a5be950 (!lp ffff988d`92fc40c0 mcshield.exe, !t ffff988d`93e2e040 ) nt!AlpcpSignalAndWait+0x143

    1 Threads waiting for LPC message replies found in the above process

2EB4 mcshield.exe       1   !t ffff988d`92fcc080    2eb0    9    8        117             .046                0      3:17:29.625  -------    Wait  UserRq                !t ffff988d`93a09080  nt!KiSystemServiceCopyEnd+0x25
1118 ArmouryCrate.U     1   !t ffff988d`96b36080    10e8   10    8        135                0                0      3:15:16.546  -------    Wait* UserRq               !lp ffff988d`915c5080  nt!KiSystemServiceCopyEnd+0x25
0FBC explorer.exe       1   !t ffff988d`9b8d7080    3344    8    8          1                0                0      3:14:49.781  -------    Wait  UserRq                !t ffff988d`98675300  nt!KiSystemServiceCopyEnd+0x25
0FBC explorer.exe       2   !t ffff988d`9c22d2c0    5708    8    8          3                0                0      3:14:49.781  -------    Wait  UserRq                !t ffff988d`9b6ea0c0  nt!KiSystemServiceCopyEnd+0x25
0FBC explorer.exe       3   !t ffff988d`938b3080    1260    8    8        338             .015                0         1:46.031  ---L---    Wait  LpcRply              !lm ffffce0166944c70 (!lp ffff988d`96c93300 sihost.exe, !t ffff988d`98536080 ) nt!AlpcpSignalAndWait+0x143

    1 Threads waiting for LPC message replies found in the above process

38F0 NahimicSvc64.e     1   !t ffff988d`970cd080    38f4    9    8        217             .015                0      3:15:19.593  -------    Wait  UserRq                !t ffff988d`914bc080  nt!KiSystemServiceCopyEnd+0x25
3908 NahimicSvc32.e     1   !t ffff988d`97197080    390c    9    8        320             .031             .015      3:15:19.609  -------    Wait  UserRq                !t ffff988d`94895080  nt!KiSystemServiceCopyEnd+0x25
39E8 NahimicSvc64.e     1   !t ffff988d`9458d080    39ec    9    8        213             .015                0      3:15:19.546  -------    Wait  UserRq                !t ffff988d`931a3080  nt!KiSystemServiceCopyEnd+0x25
39FC NahimicSvc32.e     1   !t ffff988d`9459e080    3a00    8    8        220             .031                0      3:15:19.609  -------    Wait  UserRq                !t ffff988d`970e1080  nt!KiSystemServiceCopyEnd+0x25
39FC NahimicSvc32.e     2   !t ffff988d`96cf3080    47f0    8    8      12365            8.718             .734            3.906  --P----    Wait !pl                   !pl ffff988d`8e5bf080  nt!ExpGetProcessInformation+0xaeb

    1 Locks with 769 Waiters found in the above process

      No.  Tag  Type Lock Addr           Waiters  Owners  Owner Thread                  Owner Wait Time   Tag  Owner Waiting On       Owner Function
        1  ????  !pl ffff988d`00000000       0     ???    ?? unknown                                ???        ???                    ???

    1 Locks with 769 Waiters found - for details run one of the commands listed in the above summary

1E00 SearchApp.exe      1   !t ffff988d`9a83c080    0e14    8    8         17                0                0      3:01:56.281  ---L---    Wait  LpcRply              !lm ffffce0157de4900 (!lp ffff988d`97e5e080 dllhost.exe, !t ffff988d`9a20f080 ) nt!AlpcpSignalAndWait+0x143

    1 Threads waiting for LPC message replies found in the above process

3F90 brave.exe          1   !t ffff988d`982b2500    3fe0   10    8          7                0                0      3:15:11.515  -------    Wait* UserRq               !lp ffff988d`99221080  nt!KiSystemServiceCopyEnd+0x25
0A58 gameinputsvc.e     1   !t ffff988d`9ae31080    61b0    8    8          1                0                0      3:11:19.921  -------    Wait  UserRq               !lp ffff988d`97cb7080  nt!KiSystemServiceCopyEnd+0x25
4608 svchost.exe        1   !t ffff988d`96491080    5c18    8    8          1                0                0      3:01:47.265  ---L---    Wait  LpcRply              !lm ffffce0163610c60 (!lp ffff988d`8e6d4080 svchost.exe, !t ffff988d`96ee6080 ) nt!AlpcpSignalAndWait+0x143

    1 Threads waiting for LPC message replies found in the above process

09A4 LorexCloud.exe     1   !t ffff988d`97024580    3a24   15   15    1182042             .062             .046           17.718  R----r- RUNNING  Callout               (CPU)                 nt!KeBugCheckEx+0x0
1038 obs64.exe          1   !t ffff988d`9f2eb5c0    1df0    8    8    4467120         1:24.578        14:22.906             .015  -s----- Standby* UserRq                (CPU)                 nt!KiSystemServiceCopyEnd+0x25
3200 GameBar.exe        1   !t ffff988d`97c55080    2218    9    8        118             .015                0      2:45:20.875  ---l---    Wait  LpcRply              !lm ffffce014b0d8040  nt!AlpcpSignalAndWait+0x143

    1 Threads waiting for LPC message replies found in the above process

145C CCleanerBrowse     1   !t ffff988d`8e2c2080    13dc    9    6          7                0                0           29.703  --P----    Wait !pl                   !pl 00000000`00000000  nt!MiLockDynamicMemoryExclusive+0x19

    1 Locks with 1 Waiters found in the above process

      No.  Tag  Type Lock Addr           Waiters  Owners  Owner Thread                  Owner Wait Time   Tag  Owner Waiting On       Owner Function
        1  ????  !pl ffff988d`00000000       0     ???    ?? unknown                                ???        ???                    ???

    1 Locks with 1 Waiters found - for details run one of the commands listed in the above summary

64D8 McCBEntAndInst     1   !t ffff988d`9e1d7080    64dc    9    8          6                0                0           13.203  --P----    Wait !pl                   !pl ffff988d`82012140  dxgkrnl!DXGAUTOMUTEX::Acquire+0xbe

    1 Locks with 1 Waiters found in the above process

      No.  Tag  Type Lock Addr           Waiters  Owners  Owner Thread                  Owner Wait Time   Tag  Owner Waiting On       Owner Function
        1  ????  !pl ffff988d`00000000       0     ???    ?? unknown                                ???        ???                    ???

    1 Locks with 1 Waiters found - for details run one of the commands listed in the above summary

6618 conhost.exe        1   !t ffff988d`a109b080    662c    7    6          3                0                0            3.312  --P----    Wait !pl                   !pl ffff988d`82012140  dxgkrnl!DXGAUTOMUTEX::Acquire+0xbe

    1 Locks with 1 Waiters found in the above process

      No.  Tag  Type Lock Addr           Waiters  Owners  Owner Thread                  Owner Wait Time   Tag  Owner Waiting On       Owner Function
        1  ????  !pl ffff988d`00000000       0     ???    ?? unknown                                ???        ???                    ???

    1 Locks with 1 Waiters found - for details run one of the commands listed in the above summary


                                                          Cur  Bas   zContext      Kernel Time        User Time    Elapsed Ticks   Q-TSA   Thread    Wait                                      Waiting  | Overview | Start     
PROC Image Name       No.   Thread                   Id   Pri  Pri   Switches   d:hh:mm:ss.ttt   d:hh:mm:ss.ttt   d:hh:mm:ss.ttt  RrBLsrl   State  Reason  Notes        Waiting On             Function | Function | Function  


    The following 12 processes have threads waiting on LPCs :

        Process                   Threads  Image Name

        !lp ffff988d`8b937140           1  csrss.exe
        !lp ffff988d`85dbd0c0           1  csrss.exe
        !lp ffff988d`837680c0           1  svchost.exe
        !lp ffff988d`85755080           1  svchost.exe
        !lp ffff988d`86e33080           5  svchost.exe
        !lp ffff988d`83666080           1  dasHost.exe
        !lp ffff988d`9356b080           1  ModuleCoreServ
        !lp ffff988d`935cd080           1  MfeAVSvc.exe
        !lp ffff988d`970a8080           1  explorer.exe
        !lp ffff988d`97c95080           1  SearchApp.exe
        !lp ffff988d`92e31340           1  svchost.exe
        !lp ffff988d`9d8dd080           1  GameBar.exe

    LPC Wait Chain Analysis

        No.  Process                Image Name       Threads    Waiting On  Process                Image Name      LPC Port Name

          1  !lp ffff988d`8b937140  csrss.exe            1            -->   !lp ffff988d`868bd0c0  svchost.exe     
          2  !lp ffff988d`85dbd0c0  csrss.exe            1            -->   !lp ffff988d`868bd0c0  svchost.exe     
          3  !lp ffff988d`837680c0  svchost.exe          1            -->   !lp ffff988d`8e6d4080  svchost.exe     
          4  !lp ffff988d`85755080  svchost.exe          1          No. 9   !lp ffff988d`970a8080  explorer.exe    
          5  !lp ffff988d`86e33080  svchost.exe          2            -->   !lp ffff988d`8e625080  WmiPrvSE.exe    
                                                         2            -->   !lp ffff988d`9aa24080  unsecapp.exe    
          6  !lp ffff988d`83666080  dasHost.exe          1            -->   !lp ffff988d`8e6d4080  svchost.exe     
          7  !lp ffff988d`9356b080  ModuleCoreServ       1          No. 5   !lp ffff988d`86e33080  svchost.exe     
          8  !lp ffff988d`935cd080  MfeAVSvc.exe         1            -->   !lp ffff988d`92fc40c0  mcshield.exe    
          9  !lp ffff988d`970a8080  explorer.exe         1            -->   !lp ffff988d`96c93300  sihost.exe      
         10  !lp ffff988d`97c95080  SearchApp.exe        1            -->   !lp ffff988d`97e5e080  dllhost.exe     
         11  !lp ffff988d`92e31340  svchost.exe          1            -->   !lp ffff988d`8e6d4080  svchost.exe     

    The following 9 processes have threads waiting on locks :

        Process                   Threads  Image Name

        !lp ffff988d`863ce0c0           1  svchost.exe
        !lp ffff988d`86d350c0           1  AsusCertServic
        !lp ffff988d`86e33080           1  svchost.exe
        !lp ffff988d`8e625080           1  WmiPrvSE.exe
        !lp ffff988d`916cd080           1  svchost.exe
        !lp ffff988d`9459d080         769  NahimicSvc32.e
        !lp ffff988d`8e2c9080           1  CCleanerBrowse
        !lp ffff988d`a107b080           1  McCBEntAndInst
        !lp ffff988d`8e2c8080           1  conhost.exe
```

The **`!WinDE.calls`** command searches for and lists call stacks that contain specific keywords related to synchronization and locking mechanisms, such as "SPINLOCK," "PUSHLOCK," "FASTMUTEX," and others.

```
0: kd> !winde.calls

\\\   
 >>>  Searching for call stacks containing : 
///   

        "SPINLOCK"
        "INTERLOCKED"
        "PUSHLOCK"
        "FASTMUTEX"
        "GUARDEDMUTEX"
        "EXCEPTION"
        "CDWAITSYNC"
        "FATWAITSYNC"
        "NTFSWAITSYNC"
        "NTFSWAITONIO"
        "NTFSCHECKPOINTALL"
        "NTFSCHECKPOINTFOR"
        "LOCKFILE"
        "LSAPADTADDTOQUEUE"
        "SMBCEASSOCIATEEXCHANGEWITHMID"
        "CRITICALSECTION"
        "RTLACQUIRERESOURCE"
        "ACQUIRESRWLOCK"

    NOTE: As this is not a Full dump, User-mode calls may be invisible.


                                                          Cur  Bas   zContext      Kernel Time        User Time    Elapsed Ticks   Q-TSA   Thread    Wait                                      Waiting  | Overview | Start     
PROC Image Name       No.   Thread                   Id   Pri  Pri   Switches   d:hh:mm:ss.ttt   d:hh:mm:ss.ttt   d:hh:mm:ss.ttt  RrBLsrl   State  Reason  Notes        Waiting On             Function | Function | Function  

0364 svchost.exe        1   !t ffff988d`874020c0    062c    9    8       2426                0                0         1:12.187  --P----    Wait !pl      str->        !pl 00000000`00000000  nt!ExfAcquirePushLockExclusiveEx+0x1a0
0AB0 AsusCertServic     1   !t ffff988d`85677080    09b0    8    8      44669           29.687            1.046            5.265  --P----    Wait !pl      str->        !pl ffff988d`8e5bf080  nt!ExfAcquirePushLockSharedEx+0x1b3
0DF4 svchost.exe        1   !t ffff988d`a10b1080    65b8    8    8          1                0                0            5.562  --P----    Wait !pl      str->        !lm ffffce016711f9d0  nt!ExfAcquirePushLockSharedEx+0x1b3
112C WmiPrvSE.exe       1   !t ffff988d`a1dce080    3664    9    8          2                0                0            5.953  --P----    Wait !pl      str->        !pl ffff988d`8e625080  nt!ExfAcquirePushLockExclusiveEx+0x1a0
1840 svchost.exe        1   !t ffff988d`9fad3580    6334    9    8         16                0                0            5.343  --P----    Wait !pl      str->        !pl ffff988d`8e5bf080  nt!ExfAcquirePushLockSharedEx+0x1b3
39FC NahimicSvc32.e     1   !t ffff988d`96cf3080    47f0    8    8      12365            8.718             .734            3.906  --P----    Wait !pl      str->        !pl ffff988d`8e5bf080  nt!ExfAcquirePushLockSharedEx+0x1b3
145C CCleanerBrowse     1   !t ffff988d`8e2c2080    13dc    9    6          7                0                0           29.703  --P----    Wait !pl      str->        !pl 00000000`00000000  nt!ExfAcquirePushLockExclusiveEx+0x1a0
64D8 McCBEntAndInst     1   !t ffff988d`9e1d7080    64dc    9    8          6                0                0           13.203  --P----    Wait !pl      str->        !pl ffff988d`82012140  nt!ExfAcquirePushLockExclusiveEx+0x1a0
6618 conhost.exe        1   !t ffff988d`a109b080    662c    7    6          3                0                0            3.312  --P----    Wait !pl      str->        !pl ffff988d`82012140  nt!ExfAcquirePushLockExclusiveEx+0x1a0

\\\   
 >>>  End of search for call stacks containing : 
///   

        "SPINLOCK"
        "INTERLOCKED"
      9 "PUSHLOCK"
        "FASTMUTEX"
        "GUARDEDMUTEX"
        "EXCEPTION"
        "CDWAITSYNC"
        "FATWAITSYNC"
        "NTFSWAITSYNC"
        "NTFSWAITONIO"
        "NTFSCHECKPOINTALL"
        "NTFSCHECKPOINTFOR"
        "LOCKFILE"
        "LSAPADTADDTOQUEUE"
        "SMBCEASSOCIATEEXCHANGEWITHMID"
        "CRITICALSECTION"
        "RTLACQUIRERESOURCE"
        "ACQUIRESRWLOCK"

    NOTE: As this is not a Full dump, User-mode calls may be invisible.


\\\   
 >>>  9 matching call stacks found
///
```
