# Summary

Cobalt Strike is a commercial penetration testing tool primarily used by security professionals to assess network security, but it has also been co-opted by attackers for malicious purposes. One of the key features within Cobalt Strike is a so-called "Beacon", which is a payload used for establishing persistence and command-and-control (C2) communications with the compromised system. Beacons consistently establish contact with a command and control (C2) server to acquire commands and relay information. They have the capability to collect and transmit sensitive data from the compromised system back to the attacker.

Link to this Memory Dump: https://mega.nz/file/3l9FzbDQ#HuWmR5I46F-9RdrtcMUEjZbqI0rIO5nn1GgQI66lA00

Here is a simple and high-level ASCII diagram:

```
+----------------+         +-------------------+         +-----------------+
|                |         |                   |         |                 |
|   Attacker's   |         |   Compromised     |         |   Command and   |
|   Machine      |         |   System with     |         |   Control (C2)  |
|   (Optional)   |         |   Cobalt Strike   |         |   Server        |
|                |         |   Beacon          |         |                 |
+-------+--------+         +---------+---------+         +--------+--------+
        |                              |                          |
        |                              |                          |
        |   1. Deploy Beacon           |                          |
        +----------------------------->                           |
        |                              |                          |
        |                              |                          |
        |   2. Beacon establishes      |                          |
        |      communication with C2   |                          |
        |                              +------------------------->+
        |                              |                          |
        |                              |    3. Beacon checks in   |
        |                              |       with C2 server     |
        |                              |       and awaits         |
        |                              |       commands           |
        |                              |                          |
        |                              | <------------------------+
        |                              |                          |
        |                              |    4. C2 server sends    |
        |                              |       commands           |
        |                              |                          |
        |                              +------------------------->+
        |                              |                          |
        |                              |    5. Beacon executes    |
        |                              |       commands,          |
        |                              |       gathers data       |
        |                              |                          |
        |                              |                          |
        |                              |    6. Beacon sends       |
        |                              |       results/data back  |
        |                              |       to C2 server       |
        |                              |                          |
        |                              +------------------------->+
        |                              |                          |
        +                              +                          +
```

# Initial Triage - WER

This event log we're looking at is from the Windows Error Reporting (WER) system and it indicates that an application named **beacon.exe** has crashed. The event log includes paths to files attached to the error report. These files include memory dumps, which are particularly valuable for forensic analysis as they contain the state of the application at the time of the crash.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/0b1c5db2-0f19-4824-978c-87e568dc9e85)


The **`Report.wer`** file is a Windows Error Report file that contains information about an error that occurred on a Windows system. From this file, we can observe that the application that crashed is **beacon.exe**. The **`TargetAppId`** field in the **`Report.wer`** file includes a string that is the SHA1 hash of the crashed application.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/6acb6ca1-03af-427e-8988-59035a64a1e2)


The **SHA1** hash can be used on VirusTotal to check if the file has been previously analyzed and identified as malicious by any of the antivirus engines that VirusTotal aggregates.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/9d89d03e-4b48-4b2d-907d-3de519fba6fb)


The YARA scan indicates that the memory dump from a crashed **beacon.exe** process matches patterns from rules designed to detect Cobalt Strike Beacon.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/582c5009-4710-4553-a470-d65e731791bd)


# WinDbg Walk Through - Minidump

Let's start first with loading the **minidump** in WinDbg:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/241e333d-1e37-4ac6-a2f4-e7fbacc2a957)


Let's start with the **`!mex.di`** command which stands for dump information. This command retrieves information about the machine from which the memory dump originated. It's especially handy when working with multiple memory dumps, as it helps distinguish the analysis related to each specific dump. This is an improved version of the **`vertarget`** command.

```
0:000> !mex.di
PID: 0x29D8 = 0n10712
Windows 10 Version 19043 MP (5 procs) Free x64
Product: WinNt, suite: SingleUserTS
Edition build lab: 19041.1.amd64fre.vb_release.191206-1406
Debug session time: Sat Jan 13 19:28:35.000 2024 (UTC + 0:00)
System Uptime: 0 days 3:09:55.214
Process Uptime: 0 days 0:00:19.000
  Kernel time: 0 days 0:00:00.000
  User time: 0 days 0:00:00.000

UserMode Dump Path: C:\ProgramData\Microsoft\Windows\WER\ReportQueue\AppCrash_beacon.exe_4771544f8b1de5b41a2cf9a7a08473af3c7d29a5_2e38f81a_cab_af591e90-8a5e-4a3c-a9f1-3ddaa5e0b04b\WERE035.tmp.mdmp
```

Based upon the **`!analyze -v`** command, the crash was caused by an attempt to execute code from an invalid memory address. This could be due to a bug in the code, a failed exploit attempt, or some form of corruption.

```
0:000> !analyze -v
*******************************************************************************
*                                                                             *
*                        Exception Analysis                                   *
*                                                                             *
*******************************************************************************

WARNING: Teb 10 pointer is NULL - defaulting to 00000000`7ffde000
WARNING: 00000000`7ffde000 does not appear to be a TEB
WARNING: Teb 10 pointer is NULL - defaulting to 00000000`7ffde000
WARNING: 00000000`7ffde000 does not appear to be a TEB
WARNING: Teb 10 pointer is NULL - defaulting to 00000000`7ffde000
WARNING: 00000000`7ffde000 does not appear to be a TEB
WARNING: Teb 10 pointer is NULL - defaulting to 00000000`7ffde000
WARNING: 00000000`7ffde000 does not appear to be a TEB
WARNING: Teb 10 pointer is NULL - defaulting to 00000000`7ffde000
WARNING: 00000000`7ffde000 does not appear to be a TEB
WARNING: Teb 10 pointer is NULL - defaulting to 00000000`7ffde000
WARNING: 00000000`7ffde000 does not appear to be a TEB
WARNING: Teb 10 pointer is NULL - defaulting to 00000000`7ffde000
WARNING: 00000000`7ffde000 does not appear to be a TEB
WARNING: Teb 10 pointer is NULL - defaulting to 00000000`7ffde000
WARNING: 00000000`7ffde000 does not appear to be a TEB
WARNING: Teb 10 pointer is NULL - defaulting to 00000000`7ffde000
WARNING: 00000000`7ffde000 does not appear to be a TEB
WARNING: Teb 10 pointer is NULL - defaulting to 00000000`7ffde000
WARNING: 00000000`7ffde000 does not appear to be a TEB

KEY_VALUES_STRING: 1

    Key  : AV.Fault
    Value: Execute

    Key  : Analysis.CPU.mSec
    Value: 1406

    Key  : Analysis.Elapsed.mSec
    Value: 1415

    Key  : Analysis.IO.Other.Mb
    Value: 0

    Key  : Analysis.IO.Read.Mb
    Value: 1

    Key  : Analysis.IO.Write.Mb
    Value: 6

    Key  : Analysis.Init.CPU.mSec
    Value: 3514

    Key  : Analysis.Init.Elapsed.mSec
    Value: 478982

    Key  : Analysis.Memory.CommitPeak.Mb
    Value: 163

    Key  : Failure.Bucket
    Value: INVALID_POINTER_EXECUTE_c0000005_beacon.exe!unknown_error_in_process

    Key  : Failure.Hash
    Value: {ff9bfa3b-b4c8-ca2e-8eb0-65221bb24934}

    Key  : Timeline.OS.Boot.DeltaSec
    Value: 11395

    Key  : Timeline.Process.Start.DeltaSec
    Value: 19

    Key  : WER.OS.Branch
    Value: vb_release

    Key  : WER.OS.Version
    Value: 10.0.19041.1


FILE_IN_CAB:  WERE035.tmp.mdmp

NTGLOBALFLAG:  0

APPLICATION_VERIFIER_FLAGS:  0

CONTEXT:  (.ecxr)
rax=0000000000000000 rbx=0000000000000000 rcx=0000000000000000
rdx=0000000000000000 rsi=0000000000000000 rdi=0000000000000000
rip=0000000000000000 rsp=000000000394ff28 rbp=0000000000000000
 r8=0000000000000000  r9=0000000000000000 r10=0000000000000000
r11=0000000000000000 r12=0000000000000000 r13=0000000000000000
r14=0000000000000000 r15=0000000000000000
iopl=0         nv up ei pl zr na po nc
cs=0033  ss=002b  ds=002b  es=002b  fs=0053  gs=002b             efl=00010246
00000000`00000000 ??              ???
Resetting default scope

EXCEPTION_RECORD:  (.exr -1)
ExceptionAddress: 0000000000000000
   ExceptionCode: c0000005 (Access violation)
  ExceptionFlags: 00000000
NumberParameters: 2
   Parameter[0]: 0000000000000008
   Parameter[1]: 0000000000000000
Attempt to execute non-executable address 0000000000000000

PROCESS_NAME:  beacon.exe

EXECUTE_ADDRESS: 0

FAILED_INSTRUCTION_ADDRESS: 
+0
ERROR_CODE: (NTSTATUS) 0xc0000005 - The instruction at 0x%p referenced memory at 0x%p. The memory could not be %s.

EXCEPTION_CODE_STR:  c0000005

EXCEPTION_PARAMETER1:  0000000000000008

EXCEPTION_PARAMETER2:  0000000000000000

ADDITIONAL_DEBUG_TEXT:  Followup set based on attribute [Is_ChosenCrashFollowupThread] from Frame:[0] on thread:[PSEUDO_THREAD]

STACK_TEXT:  


SYMBOL_NAME:  beacon.exe!unknown_error_in_process

MODULE_NAME: beacon

IMAGE_NAME:  beacon.exe

STACK_COMMAND:  ~-1s; .ecxr ; kb

FAILURE_BUCKET_ID:  INVALID_POINTER_EXECUTE_c0000005_beacon.exe!unknown_error_in_process

OS_VERSION:  10.0.19041.1

BUILDLAB_STR:  vb_release

OSPLATFORM_TYPE:  x64

OSNAME:  Windows 10

FAILURE_ID_HASH:  {ff9bfa3b-b4c8-ca2e-8eb0-65221bb24934}

Followup:     MachineOwner
---------
```

We can use the **`!mex.us`** command to display unique stacks (call stacks) across threads in a process, helping to identify patterns or anomalies in thread execution. The presence of a call stack involving **`wininet`** functions like **`HttpSendRequestA`** and **`InternalHttpSendRequestA`** in conjunction with a thread that has an empty call stack is a red flag.

```
0:010> !mex.us
1 thread [stats]: 0
    00007ff893fad3f4 ntdll!NtDelayExecution+0x14 --> Delaying or sleeping the execution of the thread
    00007ff89199967e KERNELBASE!SleepEx+0x9e
    000000000040305f beacon+0x305f
    0000000000000001 0x1

1 thread [stats]: 2
    00007ff893facdf4 ntdll!NtWaitForSingleObject+0x14
    00007ff891971a8e KERNELBASE!WaitForSingleObjectEx+0x8e
    00007ff87eeaf4b8 wininet!CPendingSyncCall::HandlePendingSync_AppHangIsAppBugForCallingWinInetSyncOnUIThread+0xe0
    00007ff87eeaa10f wininet!INTERNET_HANDLE_OBJECT::HandlePendingSync+0x33
    00007ff87ee50a7e wininet!HttpWrapSendRequest+0xa476e
    00007ff87ee20ff5 wininet!InternalHttpSendRequestA+0x5d
    00007ff87ee20f68 wininet!HttpSendRequestA+0x58 --> Windows API call to send an HTTP request
    000000000065e526 0x65e526
    0000000000cc000c 0xcc000c --> Likely custom or obfuscated addresses in the implant
    0000000000c2f9a0 0xc2f9a0

1 thread [stats]: 10
    0000000000000000 0x0 --> Thread with an empty call stack

8 threads [stats]: 1 3 4 5 6 7 8 9
    00007ff893fb07c4 ntdll!NtWaitForWorkViaWorkerFactory+0x14
    00007ff893f62dc7 ntdll!TppWorkerThread+0x2f7
    00007ff893be7034 kernel32!BaseThreadInitThunk+0x14
    00007ff893f62651 ntdll!RtlUserThreadStart+0x21

4 stack(s) with 11 threads displayed (11 Total threads)
```

The **`!mex.running`** command is used to display all threads in the process that are currently running or not in a wait state. This thread looks like to have an empty call stack, which is odd.

```
0:010> !mex.running
Display all threads that are not currently waiting
==================================================
1 thread [stats]: 10
    0000000000000000 0x0

Threads matching filter: 1 out of 11
```

The **`!mex.mods -3`** is used to display information about modules loaded in the process with a specific focus on limiting the output to third-party modules.

```
0:010> !mex.mods -3
Number of modules: loaded: 41 unloaded: 0 
Base     End        Flags   Time stamp           CLR   Module name   Version   Path   Filters:[ Interest|3rd|K|U|A ] More...
================================================================================================================
00400000 0044f000 |       |  Invalid Timestamp | ??? | beacon      | 0.0.0.0 | C:\Users\Admin\Desktop\beacon.exe
```

Since it's a minidump, it contains less information compared to full memory dumps. However, we can still run a strings analysis on it with the **`!mex.strings`** command. From the strings, we are able to identify a Cobalt Strike Named Pipe for instance.

```
0:010> !mex.strings -m 0x0000000000400000
0000000000404541 (TE$f
0000000000404562 {2ReZX
00000000004045ef fReTh%A
0000000000404613 fReTh$C
0000000000404627 fReRPQ
000000000040463b fReZk
0000000000404645 -Rl`:
00000000004046c8 Pe8ZPe
0000000000404779 aRHof
0000000000404793 {#hRJ_
00000000004047b0 m,RhT$
000000000044b980 \\.\pipe\MSSE-3854-server
000000000044c40e CloseHandle
000000000044c41c ConnectNamedPipe
000000000044c430 CreateFileA
000000000044c43e CreateNamedPipeA
000000000044c452 CreateThread
000000000044c462 DeleteCriticalSection
000000000044c47a EnterCriticalSection
000000000044c492 GetCurrentProcess
000000000044c4a6 GetCurrentProcessId
000000000044c4bc GetCurrentThreadId
000000000044c4d2 GetLastError
000000000044c4e2 GetModuleHandleA
000000000044c4f6 GetProcAddress
000000000044c508 GetStartupInfoA
000000000044c51a GetSystemTimeAsFileTime
000000000044c534 GetTickCount
000000000044c544 InitializeCriticalSection
000000000044c560 LeaveCriticalSection
000000000044c578 QueryPerformanceCounter
000000000044c592 ReadFile
000000000044c59e RtlAddFunctionTable
000000000044c5b4 RtlCaptureContext
000000000044c5c8 RtlLookupFunctionEntry
000000000044c5e2 RtlVirtualUnwind
000000000044c5f6 SetUnhandledExceptionFilter
000000000044c614 Sleep
000000000044c61c TerminateProcess
000000000044c630 TlsGetValue
000000000044c63e UnhandledExceptionFilter
000000000044c65a VirtualAlloc
000000000044c66a VirtualProtect
000000000044c67c VirtualQuery
000000000044c68c WriteFile
```

# WinDbg Walk Through - Heap Dump

After analyzing the Minidump, let's now analyze the **Heap Dump** file.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/f0df4a78-fab3-4501-8f90-1c010451444c)


Let's start with the **`!mex.di`** command which stands for dump information. This command retrieves information about the machine from which the memory dump originated. It's especially handy when working with multiple memory dumps, as it helps distinguish the analysis related to each specific dump. This is an improved version of the **`vertarget`**
command.

```
0:000> !mex.di
Computer Name: DESKTOP-UJOHUVV
User Name: Admin
PID: 0x29D8 = 0n10712
Windows 10 Version 19043 MP (5 procs) Free x64
Product: WinNt, suite: SingleUserTS
Edition build lab: 19041.1.amd64fre.vb_release.191206-1406
Debug session time: Sat Jan 13 19:28:35.000 2024 (UTC + 0:00)
System Uptime: 0 days 3:09:55.636
Process Uptime: 0 days 0:00:19.000
  Kernel time: 0 days 0:00:00.000
  User time: 0 days 0:00:00.000

UserMode Dump Path: C:\ProgramData\Microsoft\Windows\WER\ReportQueue\AppCrash_beacon.exe_4771544f8b1de5b41a2cf9a7a08473af3c7d29a5_2e38f81a_cab_af591e90-8a5e-4a3c-a9f1-3ddaa5e0b04b\memory.hdmp
```

Based upon the **`!analyze -v`** command, the crash was caused by an attempt to execute code from an invalid memory address. This could be due to a bug in the code, a failed exploit attempt, or some form of corruption.

```
0:000> !analyze -v
*******************************************************************************
*                                                                             *
*                        Exception Analysis                                   *
*                                                                             *
*******************************************************************************

WARNING: Thread 13b4 access failure during dump writing, NTSTATUS 0xC0000022
WARNING: Teb 10 pointer is NULL - defaulting to 00000000`7ffde000
WARNING: 00000000`7ffde000 does not appear to be a TEB
WARNING: Thread 13b4 access failure during dump writing, NTSTATUS 0xC0000022
WARNING: Teb 10 pointer is NULL - defaulting to 00000000`7ffde000
WARNING: 00000000`7ffde000 does not appear to be a TEB
WARNING: Teb 10 pointer is NULL - defaulting to 00000000`7ffde000
WARNING: 00000000`7ffde000 does not appear to be a TEB
WARNING: Teb 10 pointer is NULL - defaulting to 00000000`7ffde000
WARNING: 00000000`7ffde000 does not appear to be a TEB
WARNING: Teb 10 pointer is NULL - defaulting to 00000000`7ffde000
WARNING: 00000000`7ffde000 does not appear to be a TEB
WARNING: Teb 10 pointer is NULL - defaulting to 00000000`7ffde000
WARNING: 00000000`7ffde000 does not appear to be a TEB
WARNING: Teb 10 pointer is NULL - defaulting to 00000000`7ffde000
WARNING: 00000000`7ffde000 does not appear to be a TEB
WARNING: Teb 10 pointer is NULL - defaulting to 00000000`7ffde000
WARNING: 00000000`7ffde000 does not appear to be a TEB
WARNING: Teb 10 pointer is NULL - defaulting to 00000000`7ffde000
WARNING: 00000000`7ffde000 does not appear to be a TEB
WARNING: Teb 10 pointer is NULL - defaulting to 00000000`7ffde000
WARNING: 00000000`7ffde000 does not appear to be a TEB

KEY_VALUES_STRING: 1

    Key  : AV.Fault
    Value: Execute

    Key  : Analysis.CPU.mSec
    Value: 1468

    Key  : Analysis.Elapsed.mSec
    Value: 1461

    Key  : Analysis.IO.Other.Mb
    Value: 0

    Key  : Analysis.IO.Read.Mb
    Value: 0

    Key  : Analysis.IO.Write.Mb
    Value: 6

    Key  : Analysis.Init.CPU.mSec
    Value: 3389

    Key  : Analysis.Init.Elapsed.mSec
    Value: 273303

    Key  : Analysis.Memory.CommitPeak.Mb
    Value: 158

    Key  : Failure.Bucket
    Value: INVALID_POINTER_EXECUTE_c0000005_beacon.exe!unknown_error_in_process

    Key  : Failure.Hash
    Value: {ff9bfa3b-b4c8-ca2e-8eb0-65221bb24934}

    Key  : Timeline.OS.Boot.DeltaSec
    Value: 11395

    Key  : Timeline.Process.Start.DeltaSec
    Value: 19

    Key  : WER.OS.Branch
    Value: vb_release

    Key  : WER.OS.Version
    Value: 10.0.19041.1


FILE_IN_CAB:  memory.hdmp

NTGLOBALFLAG:  0

APPLICATION_VERIFIER_FLAGS:  0

CONTEXT:  (.ecxr)
rax=0000000000000000 rbx=0000000000000000 rcx=0000000000000000
rdx=0000000000000000 rsi=0000000000000000 rdi=0000000000000000
rip=0000000000000000 rsp=000000000394ff28 rbp=0000000000000000
 r8=0000000000000000  r9=0000000000000000 r10=0000000000000000
r11=0000000000000000 r12=0000000000000000 r13=0000000000000000
r14=0000000000000000 r15=0000000000000000
iopl=0         nv up ei pl zr na po nc
cs=0033  ss=002b  ds=002b  es=002b  fs=0053  gs=002b             efl=00010246
00000000`00000000 ??              ???
Resetting default scope

EXCEPTION_RECORD:  (.exr -1)
ExceptionAddress: 0000000000000000
   ExceptionCode: c0000005 (Access violation)
  ExceptionFlags: 00000000
NumberParameters: 2
   Parameter[0]: 0000000000000008
   Parameter[1]: 0000000000000000
Attempt to execute non-executable address 0000000000000000

PROCESS_NAME:  beacon.exe

EXECUTE_ADDRESS: 0

FAILED_INSTRUCTION_ADDRESS: 
+0
ERROR_CODE: (NTSTATUS) 0xc0000005 - The instruction at 0x%p referenced memory at 0x%p. The memory could not be %s.

EXCEPTION_CODE_STR:  c0000005

EXCEPTION_PARAMETER1:  0000000000000008

EXCEPTION_PARAMETER2:  0000000000000000

ADDITIONAL_DEBUG_TEXT:  Followup set based on attribute [Is_ChosenCrashFollowupThread] from Frame:[0] on thread:[PSEUDO_THREAD]

STACK_TEXT:  


SYMBOL_NAME:  beacon.exe!unknown_error_in_process

MODULE_NAME: beacon

IMAGE_NAME:  beacon.exe

STACK_COMMAND:  ~-1s; .ecxr ; kb

FAILURE_BUCKET_ID:  INVALID_POINTER_EXECUTE_c0000005_beacon.exe!unknown_error_in_process

OS_VERSION:  10.0.19041.1

BUILDLAB_STR:  vb_release

OSPLATFORM_TYPE:  x64

OSNAME:  Windows 10

FAILURE_ID_HASH:  {ff9bfa3b-b4c8-ca2e-8eb0-65221bb24934}

Followup:     MachineOwner
---------
```

We can use the **`!mex.us`** command to display unique stacks (call stacks) across threads in a process, helping to identify patterns or anomalies in thread execution. The presence of a call stack involving **`wininet`** functions like **`HttpSendRequestA`** and **`InternalHttpSendRequestA`** in conjunction with a thread that has an empty call stack is a red flag.

```
0:000> !mex.us
WARNING: Thread 13b4 access failure during dump writing, NTSTATUS 0xC0000022
WARNING: Teb 10 pointer is NULL - defaulting to 00000000`7ffde000
WARNING: 00000000`7ffde000 does not appear to be a TEB
1 thread [stats]: 0
    00007ff893fad3f4 ntdll!NtDelayExecution+0x14 --> Delaying or sleeping the execution of the thread
    00007ff89199967e KERNELBASE!SleepEx+0x9e
    000000000040305f beacon+0x305f
    0000000000000001 0x1

1 thread [stats]: 2
    00007ff893facdf4 ntdll!NtWaitForSingleObject+0x14
    00007ff891971a8e KERNELBASE!WaitForSingleObjectEx+0x8e
    00007ff87eeaf4b8 wininet!CPendingSyncCall::HandlePendingSync_AppHangIsAppBugForCallingWinInetSyncOnUIThread+0xe0
    00007ff87eeaa10f wininet!INTERNET_HANDLE_OBJECT::HandlePendingSync+0x33
    00007ff87ee50a7e wininet!HttpWrapSendRequest+0xa476e
    00007ff87ee20ff5 wininet!InternalHttpSendRequestA+0x5d
    00007ff87ee20f68 wininet!HttpSendRequestA+0x58 --> Windows API call to send an HTTP request
    000000000065e526 0x65e526
    0000000000cc000c 0xcc000c --> Likely custom or obfuscated addresses in the implant
    0000000000c2f9a0 0xc2f9a0

1 thread [stats]: 10
    0000000000000000 0x0 --> Thread with an empty call stack

8 threads [stats]: 1 3 4 5 6 7 8 9
    00007ff893fb07c4 ntdll!NtWaitForWorkViaWorkerFactory+0x14
    00007ff893f62dc7 ntdll!TppWorkerThread+0x2f7
    00007ff893be7034 kernel32!BaseThreadInitThunk+0x14
    00007ff893f62651 ntdll!RtlUserThreadStart+0x21

4 stack(s) with 11 threads displayed (11 Total threads)
```

The **`!mex.running`** command is used to display all threads in the process that are currently running or not in a wait state. This thread looks like to have an empty call stack, which is odd.

```
0:000> !mex.running
Display all threads that are not currently waiting
==================================================
1 thread [stats]: 10
    0000000000000000 0x0
```

The **`!mex.mods -3`** is used to display information about modules loaded in the process with a specific focus on limiting the output to third-party modules.

```
0:000> !mex.mods -3
Number of modules: loaded: 41 unloaded: 0 
Base     End        Flags   Time stamp           CLR   Module name   Version   Path   Filters:[ Interest|3rd|K|U|A ] More...
================================================================================================================
00400000 0044f000 |       |  Invalid Timestamp | ??? | beacon      | 0.0.0.0 | C:\Users\Admin\Desktop\beacon.exe
```

Let's try to check for the imports, but unfortunately we don't get any results back. It's very likely that the dump is incomplete, since in the end of the day. It's a Heap Dump which can provide detailed information about heap usage, allocation patterns, and content, etc. However, it's not a full memory process dump.

```
0:000> !mex.imports -a 0x0000000000400000
```

Finding a memory region with **`PAGE_EXECUTE_READWRITE`** protection, especially in the context of potential malware can be indicative of malicious activity. **`PAGE_EXECUTE_READWRITE`** is memory protection flag allows a region of memory to be executed, read, and written to. Malware often uses **`PAGE_EXECUTE_READWRITE`** permissions for its memory regions to dynamically write and execute malicious code.

```
0:000> !address -f:PAGE_EXECUTE_READWRITE

        BaseAddress      EndAddress+1        RegionSize     Type       State                 Protect             Usage
--------------------------------------------------------------------------------------------------------------------------
       0`00650000        0`006a2000        0`00052000 MEM_PRIVATE MEM_COMMIT  PAGE_EXECUTE_READWRITE             <unknown>  [MZARUH..H......H]
```

We can now search for strings within the specified memory range and look for Cobalt Strike artifacts. This string pattern is part of a default profile for a Cobalt Strike beacon.

```
0:000> s -a 0`00650000 0`006a2000 "%s.4%08x%08x%08x%08x%08x.%08x%08x%08x%08x%08x%08x%08x.%08x%08x%08x%08x%08x%08x%08x.%08x%08x%08x%08x%08x%08x%08x.%x%x.%s"
00000000`0067e9a3  25 73 2e 34 25 30 38 78-25 30 38 78 25 30 38 78  %s.4%08x%08x%08x
```
When we display the bytes, we can see the string pattern that matches default Cobalt Strike profile:

```
0:000> db 00000000`0067e9a3
00000000`0067e9a3  25 73 2e 34 25 30 38 78-25 30 38 78 25 30 38 78  %s.4%08x%08x%08x
00000000`0067e9b3  25 30 38 78 25 30 38 78-2e 25 30 38 78 25 30 38  %08x%08x.%08x%08
00000000`0067e9c3  78 25 30 38 78 25 30 38-78 25 30 38 78 25 30 38  x%08x%08x%08x%08
00000000`0067e9d3  78 25 30 38 78 2e 25 30-38 78 25 30 38 78 25 30  x%08x.%08x%08x%0
00000000`0067e9e3  38 78 25 30 38 78 25 30-38 78 25 30 38 78 25 30  8x%08x%08x%08x%0
00000000`0067e9f3  38 78 2e 25 30 38 78 25-30 38 78 25 30 38 78 25  8x.%08x%08x%08x%
00000000`0067ea03  30 38 78 25 30 38 78 25-30 38 78 25 30 38 78 2e  08x%08x%08x%08x.
00000000`0067ea13  25 78 25 78 2e 25 73 00-25 73 2e 33 25 30 38 78  %x%x.%s.%s.3%08x
```

To double check it, let's look for another specific string that matches a default Cobalt Strike profile:

```
0:000> s -a 0`00650000 0`006a2000 "IEX (New-Object Net.Webclient).DownloadString('http://127.0.0.1:%u/'); %s"
00000000`0067f4b7  49 45 58 20 28 4e 65 77-2d 4f 62 6a 65 63 74 20  IEX (New-Object
```

We can now display the bytes and we can see that this pattern does matches the default Cobalt Strike profile:

```
0:000> db 00000000`0067f4b7
00000000`0067f4b7  49 45 58 20 28 4e 65 77-2d 4f 62 6a 65 63 74 20  IEX (New-Object 
00000000`0067f4c7  4e 65 74 2e 57 65 62 63-6c 69 65 6e 74 29 2e 44  Net.Webclient).D
00000000`0067f4d7  6f 77 6e 6c 6f 61 64 53-74 72 69 6e 67 28 27 68  ownloadString('h
00000000`0067f4e7  74 74 70 3a 2f 2f 31 32-37 2e 30 2e 30 2e 31 3a  ttp://127.0.0.1:
00000000`0067f4f7  25 75 2f 27 29 3b 20 25-73 00 70 6f 77 65 72 73  %u/'); %s.powers
00000000`0067f507  68 65 6c 6c 20 2d 6e 6f-70 20 2d 65 78 65 63 20  hell -nop -exec 
00000000`0067f517  62 79 70 61 73 73 20 2d-45 6e 63 6f 64 65 64 43  bypass -EncodedC
00000000`0067f527  6f 6d 6d 61 6e 64 20 22-25 73 22 00 43 6f 75 6c  ommand "%s".Coul
```

To finalize it, we can now extract the Cobalt Strike Beacon configuration by running this Python script against the Heap Memory Dump file:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/7f040c51-c75d-4c6d-9008-b94096fad87f)


# References

- **Extracting Cobalt Strike from Windows Error Reporting:** https://bmcder.com/blog/extracting-cobalt-strike-from-windows-error-reporting
- **CobaltStrikeBeacon_Default.profile:** https://github.com/h3ll0clar1c3/CRTO/blob/main/CobaltStrikeBeacon_Default.profile
- **1768.py:** https://blog.didierstevens.com/programs/cobalt-strike-tools/
