# Summary

Cobalt Strike is a commercial penetration testing tool primarily used by security professionals to assess network security, but it has also been co-opted by attackers for malicious purposes. One of the key features within Cobalt Strike is a so-called "Beacon", which is a payload used for establishing persistence and command-and-control (C2) communications with the compromised system. Beacons consistently establish contact with a command and control (C2) server to acquire commands and relay information. They have the capability to collect and transmit sensitive data from the compromised system back to the attacker.

**NOTE:** The primary purpose of this write-up is mainly to demonstrate various debugging techniques, rather than identifying a C2. Be aware that extracting the ZIP file might trigger an antivirus alert, as it originates from a Cobalt Strike Beacon. It's highly recommended to do all of this stuff in a Virtual Machine.

Link to this Memory Dump: https://mega.nz/file/XhdXwa6A#NIMP-swi3rn0kzMXaNelbRaUXVmBhOGIsP2-qrJRZqI

Within this scenario, a specific Cobalt Strike Beacon was causing a crash, prompting Windows Error Reporting to generate a report. 

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/831a851d-317e-46c5-b375-d7eb54a03edb)


After executing our YARA rule, we observed a match that closely corresponds with a Cobalt Strike pattern:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/0b1b4c5f-dfbb-4b3c-8e6e-07f5c7ec2dd8)


Nevertheless, when attempting to extract the Cobalt Strike configuration using the **1768.py** script, the process fails and nothing has been extracted.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/e920eff9-441d-4891-b15d-f62ec3da2244)


From here, let's start examing the Memory Dump and see what information we are able to gather to come to a conclusion that is a Cobalt Strike Beacon.

The following extensions are used:

- **MEX:** https://www.microsoft.com/en-us/download/details.aspx?id=53304
- **PDE:** https://onedrive.live.com/?authkey=%21AJeSzeiu8SQ7T4w&cid=DAE128BD454CF957&id=DAE128BD454CF957%2112585&parId=DAE128BD454CF957%217152&o=OneUp

# WinDbg Walk Through - Analysis

First, load the **memory.hdmp** file in WinDbg:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/1a05be0e-de01-4a77-bc2f-3206400b8d1e)


Let's start with the **`!mex.di`** command which stands for dump information. This command retrieves information about the machine from which the memory dump originated. It's especially handy when working with multiple memory dumps, as it helps distinguish the analysis related to each specific dump. This is an improved version of the **`vertarget`** command.

```
0:000> !mex.di
Computer Name: DESKTOP-UJOHUVV
User Name: Admin
PID: 0xE7C = 0n3708
Windows 10 Version 19043 MP (5 procs) Free x64
Product: WinNt, suite: SingleUserTS
Edition build lab: 19041.1.amd64fre.vb_release.191206-1406
Debug session time: Sat Jan 13 22:03:58.000 2024 (UTC + 0:00)
System Uptime: 0 days 5:31:38.261
Process Uptime: 0 days 0:01:39.000
  Kernel time: 0 days 0:00:00.000
  User time: 0 days 0:00:00.000

UserMode Dump Path: C:\ProgramData\Microsoft\Windows\WER\ReportQueue\AppCrash_Beacon_x64.exe_2a37fdf27c1d99c6bc2e168f1f1ef43cc4e768_00c9c00e_cab_3642c184-965e-4b0b-8409-75d2f34b4093\memory.hdmp
```

Based upon the **`!analyze -v`** command, the crash was caused by an attempt to execute code from an invalid memory address. This could be due to a bug in the code, a failed exploit attempt, or some form of corruption.

```
0:000> !analyze -v
*******************************************************************************
*                                                                             *
*                        Exception Analysis                                   *
*                                                                             *
*******************************************************************************

WARNING: Thread 1afc access failure during dump writing, NTSTATUS 0xC0000022
WARNING: Teb 5 pointer is NULL - defaulting to 00000000`7ffde000
WARNING: 00000000`7ffde000 does not appear to be a TEB
WARNING: Thread 1afc access failure during dump writing, NTSTATUS 0xC0000022
WARNING: Teb 5 pointer is NULL - defaulting to 00000000`7ffde000
WARNING: 00000000`7ffde000 does not appear to be a TEB
WARNING: Teb 5 pointer is NULL - defaulting to 00000000`7ffde000
WARNING: 00000000`7ffde000 does not appear to be a TEB
WARNING: Teb 5 pointer is NULL - defaulting to 00000000`7ffde000
WARNING: 00000000`7ffde000 does not appear to be a TEB
WARNING: Teb 5 pointer is NULL - defaulting to 00000000`7ffde000
WARNING: 00000000`7ffde000 does not appear to be a TEB
WARNING: Teb 5 pointer is NULL - defaulting to 00000000`7ffde000
WARNING: 00000000`7ffde000 does not appear to be a TEB
WARNING: Teb 5 pointer is NULL - defaulting to 00000000`7ffde000
WARNING: 00000000`7ffde000 does not appear to be a TEB
WARNING: Teb 5 pointer is NULL - defaulting to 00000000`7ffde000
WARNING: 00000000`7ffde000 does not appear to be a TEB
WARNING: Teb 5 pointer is NULL - defaulting to 00000000`7ffde000
WARNING: 00000000`7ffde000 does not appear to be a TEB
WARNING: Teb 5 pointer is NULL - defaulting to 00000000`7ffde000
WARNING: 00000000`7ffde000 does not appear to be a TEB

KEY_VALUES_STRING: 1

    Key  : AV.Fault
    Value: Execute

    Key  : Analysis.CPU.mSec
    Value: 1171

    Key  : Analysis.Elapsed.mSec
    Value: 1176

    Key  : Analysis.IO.Other.Mb
    Value: 0

    Key  : Analysis.IO.Read.Mb
    Value: 0

    Key  : Analysis.IO.Write.Mb
    Value: 6

    Key  : Analysis.Init.CPU.mSec
    Value: 2858

    Key  : Analysis.Init.Elapsed.mSec
    Value: 79642

    Key  : Analysis.Memory.CommitPeak.Mb
    Value: 161

    Key  : Failure.Bucket
    Value: INVALID_POINTER_EXECUTE_c0000005_Beacon_x64.exe!unknown_error_in_process

    Key  : Failure.Hash
    Value: {075457bd-8f3f-c8af-ecc0-0d0ee9c7b280}

    Key  : Timeline.OS.Boot.DeltaSec
    Value: 19898

    Key  : Timeline.Process.Start.DeltaSec
    Value: 99

    Key  : WER.OS.Branch
    Value: vb_release

    Key  : WER.OS.Version
    Value: 10.0.19041.1


FILE_IN_CAB:  memory.hdmp

NTGLOBALFLAG:  0

PROCESS_BAM_CURRENT_THROTTLED: 0

PROCESS_BAM_PREVIOUS_THROTTLED: 0

APPLICATION_VERIFIER_FLAGS:  0

CONTEXT:  (.ecxr)
rax=0000000000000000 rbx=0000000000000000 rcx=0000000000000000
rdx=0000000000000000 rsi=0000000000000000 rdi=0000000000000000
rip=0000000000000000 rsp=000000b4ddcffba8 rbp=0000000000000000
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

PROCESS_NAME:  Beacon_x64.exe

EXECUTE_ADDRESS: 0

FAILED_INSTRUCTION_ADDRESS: 
+0
ERROR_CODE: (NTSTATUS) 0xc0000005 - The instruction at 0x%p referenced memory at 0x%p. The memory could not be %s.

EXCEPTION_CODE_STR:  c0000005

EXCEPTION_PARAMETER1:  0000000000000008

EXCEPTION_PARAMETER2:  0000000000000000

ADDITIONAL_DEBUG_TEXT:  Followup set based on attribute [Is_ChosenCrashFollowupThread] from Frame:[0] on thread:[PSEUDO_THREAD]

STACK_TEXT:  


SYMBOL_NAME:  Beacon_x64.exe!unknown_error_in_process

MODULE_NAME: Beacon_x64

IMAGE_NAME:  Beacon_x64.exe

STACK_COMMAND:  ~-1s; .ecxr ; kb

FAILURE_BUCKET_ID:  INVALID_POINTER_EXECUTE_c0000005_Beacon_x64.exe!unknown_error_in_process

OS_VERSION:  10.0.19041.1

BUILDLAB_STR:  vb_release

OSPLATFORM_TYPE:  x64

OSNAME:  Windows 10

FAILURE_ID_HASH:  {075457bd-8f3f-c8af-ecc0-0d0ee9c7b280}

Followup:     MachineOwner
---------
```

We can use the **`!mex.us`** command to display unique stacks (call stacks) across threads in a process, helping to identify patterns or anomalies in thread execution. The presence of a call stack involving **`wininet`** functions like **`HttpSendRequestA`** and **`InternalHttpSendRequestA`** in conjunction with a thread that has an empty call stack is a red flag.

```
0:000> !mex.us
1 thread [stats]: 0
    00007ff893facdf4 ntdll!NtWaitForSingleObject+0x14
    00007ff891971a8e KERNELBASE!WaitForSingleObjectEx+0x8e
    00007ff87eeaf4b8 wininet!CPendingSyncCall::HandlePendingSync_AppHangIsAppBugForCallingWinInetSyncOnUIThread+0xe0
    00007ff87eeaa10f wininet!INTERNET_HANDLE_OBJECT::HandlePendingSync+0x33
    00007ff87ee50a7e wininet!HttpWrapSendRequest+0xa476e
    00007ff87ee20ff5 wininet!InternalHttpSendRequestA+0x5d
    00007ff87ee20f68 wininet!HttpSendRequestA+0x58
    0000024fa0d01271 0x24fa0d01271
    0000024fa0a4d7d0 0x24fa0a4d7d0
    000000b4dd9afa20 0xb4dd9afa20
    0000000056a2b5f0 0x56a2b5f0
    0000024fa0c0dbd0 0x24fa0c0dbd0

1 thread [stats]: 5
    0000000000000000 0x0

4 threads [stats]: 1 2 3 4
    00007ff893fb07c4 ntdll!NtWaitForWorkViaWorkerFactory+0x14
    00007ff893f62dc7 ntdll!TppWorkerThread+0x2f7
    00007ff893be7034 kernel32!BaseThreadInitThunk+0x14
    00007ff893f62651 ntdll!RtlUserThreadStart+0x21

3 stack(s) with 6 threads displayed (6 Total threads)
```

The **`!mex.running`** command is used to display all threads in the process that are currently running or not in a wait state. This thread looks like to have an empty call stack, which is odd.

```
0:000> !mex.running
Display all threads that are not currently waiting
==================================================
WARNING: Thread 1afc access failure during dump writing, NTSTATUS 0xC0000022
WARNING: Teb 5 pointer is NULL - defaulting to 00000000`7ffde000
WARNING: 00000000`7ffde000 does not appear to be a TEB
1 thread [stats]: 5
    0000000000000000 0x0

Threads matching filter: 1 out of 6
```

The **`!mex.mods -3`** is used to display information about modules loaded in the process with a specific focus on limiting the output to third-party modules.

```
0:000> !mex.mods -3
Number of modules: loaded: 42 unloaded: 0 
Base             End                Flags   Time stamp            CLR   Module name   Version   Path   Filters:[ Interest|3rd|K|U|A ] More...
=====================================================================================================================================
00007ff7d9ad0000 00007ff7d9daf000 |       | 2023-08-25 11:22:53 |  No | Beacon_x64  | 0.0.0.0 | C:\Users\Admin\Desktop\Beacon_x64.exe
```

We can look for the loaded imports to give us insight into its capabilities and potential actions it can perform. We can see some memory management functions for instance with the likes of **`VirtualAlloc`**, **`VirtualProtect`**, and **`VirtualFree`**. 

```
0:000> !mex.imports -a 0x00007ff7d9ad0000
Dumping Import table at 00007ff7d9ad0000 + 2da36c 
ModuleImageName: Beacon_x64
Beacon_x64 imports KERNEL32.dll!VirtualAlloc
Beacon_x64 imports KERNEL32.dll!VirtualProtect
Beacon_x64 imports KERNEL32.dll!VirtualFree
Beacon_x64 imports KERNEL32.dll!CloseHandle
Beacon_x64 imports KERNEL32.dll!ReleaseSRWLockExclusive
Beacon_x64 imports KERNEL32.dll!ReleaseMutex
Beacon_x64 imports KERNEL32.dll!ReleaseSRWLockShared
Beacon_x64 imports KERNEL32.dll!GetLastError
Beacon_x64 imports KERNEL32.dll!AddVectoredExceptionHandler
Beacon_x64 imports KERNEL32.dll!SetThreadStackGuarantee
Beacon_x64 imports KERNEL32.dll!AcquireSRWLockExclusive
Beacon_x64 imports KERNEL32.dll!GetCurrentProcess
Beacon_x64 imports KERNEL32.dll!GetCurrentThread
Beacon_x64 imports KERNEL32.dll!RtlCaptureContext
Beacon_x64 imports KERNEL32.dll!GetProcAddress
Beacon_x64 imports KERNEL32.dll!RtlLookupFunctionEntry
Beacon_x64 imports KERNEL32.dll!SetLastError
Beacon_x64 imports KERNEL32.dll!GetCurrentDirectoryW
Beacon_x64 imports KERNEL32.dll!GetEnvironmentVariableW
Beacon_x64 imports KERNEL32.dll!GetStdHandle
Beacon_x64 imports KERNEL32.dll!GetCurrentProcessId
Beacon_x64 imports KERNEL32.dll!WaitForSingleObject
Beacon_x64 imports KERNEL32.dll!TryAcquireSRWLockExclusive
Beacon_x64 imports KERNEL32.dll!QueryPerformanceCounter
Beacon_x64 imports KERNEL32.dll!HeapAlloc
Beacon_x64 imports KERNEL32.dll!GetProcessHeap
Beacon_x64 imports KERNEL32.dll!HeapFree
Beacon_x64 imports KERNEL32.dll!HeapReAlloc
Beacon_x64 imports KERNEL32.dll!AcquireSRWLockShared
Beacon_x64 imports KERNEL32.dll!WaitForSingleObjectEx
Beacon_x64 imports KERNEL32.dll!LoadLibraryA
Beacon_x64 imports KERNEL32.dll!CreateMutexA
Beacon_x64 imports KERNEL32.dll!GetModuleHandleA
Beacon_x64 imports KERNEL32.dll!GetConsoleMode
Beacon_x64 imports KERNEL32.dll!GetModuleHandleW
Beacon_x64 imports KERNEL32.dll!FormatMessageW
Beacon_x64 imports KERNEL32.dll!MultiByteToWideChar
Beacon_x64 imports KERNEL32.dll!WriteConsoleW
Beacon_x64 imports KERNEL32.dll!TlsGetValue
Beacon_x64 imports KERNEL32.dll!TlsSetValue
Beacon_x64 imports KERNEL32.dll!GetSystemTimeAsFileTime
Beacon_x64 imports KERNEL32.dll!SetUnhandledExceptionFilter
Beacon_x64 imports KERNEL32.dll!UnhandledExceptionFilter
Beacon_x64 imports KERNEL32.dll!IsDebuggerPresent
Beacon_x64 imports KERNEL32.dll!RtlVirtualUnwind
Beacon_x64 imports KERNEL32.dll!InitializeSListHead
Beacon_x64 imports KERNEL32.dll!GetCurrentThreadId
Beacon_x64 imports KERNEL32.dll!IsProcessorFeaturePresent
Beacon_x64 imports ntdll.dll!RtlNtStatusToDosError
Beacon_x64 imports ntdll.dll!NtWriteFile
Beacon_x64 imports VCRUNTIME140.dll!_CxxThrowException
Beacon_x64 imports VCRUNTIME140.dll!memcmp
Beacon_x64 imports VCRUNTIME140.dll!memset
Beacon_x64 imports VCRUNTIME140.dll!memcpy
Beacon_x64 imports VCRUNTIME140.dll!__CxxFrameHandler3
Beacon_x64 imports VCRUNTIME140.dll!__current_exception_context
Beacon_x64 imports VCRUNTIME140.dll!__C_specific_handler
Beacon_x64 imports VCRUNTIME140.dll!__current_exception
Beacon_x64 imports VCRUNTIME140.dll!memmove
Beacon_x64 imports api-ms-win-crt-runtime-l1-1-0.dll!_initterm_e
Beacon_x64 imports api-ms-win-crt-runtime-l1-1-0.dll!exit
Beacon_x64 imports api-ms-win-crt-runtime-l1-1-0.dll!_exit
Beacon_x64 imports api-ms-win-crt-runtime-l1-1-0.dll!_initterm
Beacon_x64 imports api-ms-win-crt-runtime-l1-1-0.dll!__p___argc
Beacon_x64 imports api-ms-win-crt-runtime-l1-1-0.dll!__p___argv
Beacon_x64 imports api-ms-win-crt-runtime-l1-1-0.dll!_get_initial_narrow_environment
Beacon_x64 imports api-ms-win-crt-runtime-l1-1-0.dll!_c_exit
Beacon_x64 imports api-ms-win-crt-runtime-l1-1-0.dll!_register_thread_local_exe_atexit_callback
Beacon_x64 imports api-ms-win-crt-runtime-l1-1-0.dll!_initialize_narrow_environment
Beacon_x64 imports api-ms-win-crt-runtime-l1-1-0.dll!_configure_narrow_argv
Beacon_x64 imports api-ms-win-crt-runtime-l1-1-0.dll!_set_app_type
Beacon_x64 imports api-ms-win-crt-runtime-l1-1-0.dll!_seh_filter_exe
Beacon_x64 imports api-ms-win-crt-runtime-l1-1-0.dll!_register_onexit_function
Beacon_x64 imports api-ms-win-crt-runtime-l1-1-0.dll!_crt_atexit
Beacon_x64 imports api-ms-win-crt-runtime-l1-1-0.dll!terminate
Beacon_x64 imports api-ms-win-crt-runtime-l1-1-0.dll!_cexit
Beacon_x64 imports api-ms-win-crt-runtime-l1-1-0.dll!_initialize_onexit_table
Beacon_x64 imports api-ms-win-crt-math-l1-1-0.dll!__setusermatherr
Beacon_x64 imports api-ms-win-crt-stdio-l1-1-0.dll!__p__commode
Beacon_x64 imports api-ms-win-crt-stdio-l1-1-0.dll!_set_fmode
Beacon_x64 imports api-ms-win-crt-locale-l1-1-0.dll!_configthreadlocale
Beacon_x64 imports api-ms-win-crt-heap-l1-1-0.dll!_set_new_mode
Beacon_x64 imports api-ms-win-crt-heap-l1-1-0.dll!free
00007ff7`d9aee2d8  00007ff7`d9aebb6c Beacon_x64+0x1bb6c
```

We can run strings against the binary itself to see if we can find some quick patterns that matches Cobalt Strike with the **`!mex.strings`** command. From the result, it looks like the strings have been obfuscated though.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/e502dac9-3774-40cc-9924-b95f0ff0b85b)


This command does a grep on ".pdb" in the strings and here we can see an interesting path.

```
0:000> !mex.grep .pdb !mex.strings -m 0x00007ff7d9ad0000
00007ff7d9da7184 C:\tswsec2\tools\BypassAv_demo1_2\target\release\deps\BypassAv_demo1_2.pdb
```

Finding a memory region with **`PAGE_EXECUTE_READWRITE`** protection, especially in the context of potential malware can be indicative of malicious activity. **`PAGE_EXECUTE_READWRITE`** is memory protection flag allows a region of memory to be executed, read, and written to. Malware often uses **`PAGE_EXECUTE_READWRITE`** permissions for its memory regions to dynamically write and execute malicious code.

```
0:000> !address -f:PAGE_EXECUTE_READWRITE

        BaseAddress      EndAddress+1        RegionSize     Type       State                 Protect             Usage
--------------------------------------------------------------------------------------------------------------------------
     24f`a0a30000      24f`a0a87000        0`00057000 MEM_PRIVATE MEM_COMMIT  PAGE_EXECUTE_READWRITE             <unknown>  [MZARUH..H......H]
     24f`a0cf0000      24f`a0d54000        0`00064000 MEM_PRIVATE MEM_COMMIT  PAGE_EXECUTE_READWRITE             <unknown>  [MZARUH..H......H]
```

We can now search for strings within the specified memory range and look for Cobalt Strike artifacts. This string pattern is part of a default profile for a Cobalt Strike beacon.

```
0:000> s -a 24f`a0a30000 24f`a0a87000 "%s.4%08x%08x%08x%08x%08x.%08x%08x%08x%08x%08x%08x%08x.%08x%08x%08x%08x%08x%08x%08x.%08x%08x%08x%08x%08x%08x%08x.%x%x.%s"
0000024f`a0a6db53  25 73 2e 34 25 30 38 78-25 30 38 78 25 30 38 78  %s.4%08x%08x%08x
```

The **`!pde.dpx`** command can be used for displaying various types of information such as, the stack, inspect memory around specific addresses, or analyze trap frames. From the output, we are able to observe an IP address which is **45[.]143[.]235[.]32**

```
0:000> !pde.dpx
Start memory scan  : 0x000000b4dd9aef88 ($csp)
End memory scan    : 0x000000b4dd9b0000 (User Stack Base)

               rsp : 0x000000b4dd9aef88 : 0x00007ff891971a8e : KERNELBASE!WaitForSingleObjectEx+0x8e
               r11 : 0x000000b4dd9aee60 : 0x0000024fa0c43510 :  dt wininet!CFsm_HttpSendRequest
0x000000b4dd9aef88 : 0x00007ff891971a8e : KERNELBASE!WaitForSingleObjectEx+0x8e
0x000000b4dd9aefa8 : 0x0000024fa0c1c320 :  dt wininet!HTTP_REQUEST_HANDLE_OBJECT
0x000000b4dd9aefb8 : 0x00007ff8919bb96b : KERNELBASE!TrySubmitThreadpoolCallback+0xb
0x000000b4dd9af010 : 0x0000024fa0c1c320 :  dt wininet!HTTP_REQUEST_HANDLE_OBJECT
0x000000b4dd9af028 : 0x00007ff87eeaf4b8 : wininet!CPendingSyncCall::HandlePendingSync_AppHangIsAppBugForCallingWinInetSyncOnUIThread+0xe0
0x000000b4dd9af038 : 0x00007ff893f607b0 : ntdll!RtlSetLastWin32Error+0x40
0x000000b4dd9af058 : 0x00007ff87eeaa10f : wininet!INTERNET_HANDLE_OBJECT::HandlePendingSync+0x33
0x000000b4dd9af060 : 0x0000024fa0b53b00 : 0x00007ff87f1dff80 : wininet!ThreadInfoList
0x000000b4dd9af080 : 0x0000024fa0c43510 :  dt wininet!CFsm_HttpSendRequest
0x000000b4dd9af090 : 0x0000024fa0b53b00 : 0x00007ff87f1dff80 : wininet!ThreadInfoList
0x000000b4dd9af0d8 : 0x0000024fa0c0cd30 :  dt wininet!INTERNET_CONNECT_HANDLE_OBJECT
0x000000b4dd9af0f0 : 0x0000024fa0c1c320 :  dt wininet!HTTP_REQUEST_HANDLE_OBJECT
0x000000b4dd9af120 : 0x0000024fa0c0d3d0 :  !da ""Cookie: hRUSkG0UwQZwGytnnIZmy54Xvobgwpro9O49Wa4rrXU/xO3UG9pX/acX57D3nkr76UcprsnF...""
0x000000b4dd9af158 : 0x00007ff87ef4c9a0 : wininet!CWxString::s_wszEmpty+0x20
0x000000b4dd9af218 : 0x0000024fa0c0d3d0 :  !da ""Cookie: hRUSkG0UwQZwGytnnIZmy54Xvobgwpro9O49Wa4rrXU/xO3UG9pX/acX57D3nkr76UcprsnF...""
0x000000b4dd9af228 : 0x00007ff87ee20ff5 : wininet!InternalHttpSendRequestA+0x5d
0x000000b4dd9af238 : 0x00007ff87eda42b9 : wininet!InternetRestoreThreadInfo+0x59
0x000000b4dd9af298 : 0x00007ff87ee20f68 : wininet!HttpSendRequestA+0x58
0x000000b4dd9af2b0 : 0x0000024fa0c0d3d0 :  !da ""Cookie: hRUSkG0UwQZwGytnnIZmy54Xvobgwpro9O49Wa4rrXU/xO3UG9pX/acX57D3nkr76UcprsnF...""
0x000000b4dd9af358 : 0x0000024fa0c0dbd0 :  !da "/visit.js"
0x000000b4dd9af3c0 : 0x0000024fa0c0d3d0 :  !da ""Cookie: hRUSkG0UwQZwGytnnIZmy54Xvobgwpro9O49Wa4rrXU/xO3UG9pX/acX57D3nkr76UcprsnF...""
0x000000b4dd9af3d0 : 0x0000024fa0c0d3d0 :  !da ""Cookie: hRUSkG0UwQZwGytnnIZmy54Xvobgwpro9O49Wa4rrXU/xO3UG9pX/acX57D3nkr76UcprsnF...""
0x000000b4dd9af3e0 : 0x0000024fa0c0dbd0 :  !da "/visit.js"
0x000000b4dd9af3f8 : 0x0000024fa0c0ffd0 :  !da "hRUSkG0UwQZwGytnnIZmy54Xvobgwpro9O49Wa4rrXU/xO3UG9pX/acX57D3nkr76UcprsnFYKzIEeGP..."
0x000000b4dd9af400 : 0x0000024fa0c11fd0 :  !da ""Cookie: hRUSkG0UwQZwGytnnIZmy54Xvobgwpro9O49Wa4rrXU/xO3UG9pX/acX57D3nkr76UcprsnF...""
0x000000b4dd9af408 : 0x0000024fa0c37eb0 : 0x0000024fa0c0d3d0 :  !da ""Cookie: hRUSkG0UwQZwGytnnIZmy54Xvobgwpro9O49Wa4rrXU/xO3UG9pX/acX57D3nkr76UcprsnF...""
0x000000b4dd9af420 : 0x6a2e74697369762f :  !da "/visit.js"
0x000000b4dd9af830 : 0x0000024fa0b71870 :  !da "/visit.js"
0x000000b4dd9af870 : 0x0000024fa0b71870 :  !da "/visit.js"
0x000000b4dd9af8c0 : 0x0000024fa0b57f80 : 0x0000024fa0b71870 :  !da "/visit.js"
0x000000b4dd9af8e8 : 0x0000024fa0b71970 :  !da "45.143.235.32"
0x000000b4dd9af8f0 : 0x0000024fa0b5a3e0 :  !da "45.143.235.32,/visit.js"
0x000000b4dd9af8f8 : 0x0000024fa0b71870 :  !da "/visit.js"
0x000000b4dd9af900 : 0x0000024fa0b59b60 :  !da ""1; Linux; es-VE; rv:52.9.0) Gecko/20100101 Firefox/52.9.0.""
0x000000b4dd9af918 : 0x0000024fa0b6e090 :  !da "/submit.php"
0x000000b4dd9af940 : 0x000000b4dd9af900 : 0x0000024fa0b59b60 :  !da ""1; Linux; es-VE; rv:52.9.0) Gecko/20100101 Firefox/52.9.0.""
0x000000b4dd9afaf0 : 0x00007ff7d9aee4c0 :  !da "###%%%4d###%%%5a###%%%41###%%%52###%%%55###%%%48###%%%89###%%%e5###%%%48###%%%81..."
0x000000b4dd9afb88 : 0x0000024fa0af4f50 : 0x0000024fa0af4f60 :  !da "C:\Users\Admin\desktop\Beacon_x64.exe"
0x000000b4dd9afc00 : 0x0000024fa0af4f50 : 0x0000024fa0af4f60 :  !da "C:\Users\Admin\desktop\Beacon_x64.exe"
0x000000b4dd9afc30 : 0x0000024fa0af4f50 : 0x0000024fa0af4f60 :  !da "C:\Users\Admin\desktop\Beacon_x64.exe"
0x000000b4dd9afc40 : 0x0000024fa0af97c0 : 0x0000024fa0af52c0 :  !da "ALLUSERSPROFILE=C:\ProgramData"
0x000000b4dd9afc98 : 0x00007ff893f347b1 : ntdll!RtlFreeHeap+0x51
0x000000b4dd9afcd8 : 0x00007ff891950000 : "KERNELBASE!UrlHashW <PERF> (KERNELBASE+0x0)"
0x000000b4dd9afcf8 : 0x00007ff8919553dc : KERNELBASE!GetProcAddressForCaller+0x6c
0x000000b4dd9afd00 : 0x00007ff891950000 : "KERNELBASE!UrlHashW <PERF> (KERNELBASE+0x0)"
0x000000b4dd9afd08 : 0x00007ff891982d25 : KERNELBASE!GetModuleHandleA+0x35
0x000000b4dd9afd10 : 0x00007ff891950000 : "KERNELBASE!UrlHashW <PERF> (KERNELBASE+0x0)"
0x000000b4dd9afd20 : 0x000000b4dd9afdb0 : 0x00007ff7d9ad1110 :  !da "UAWAVAUATVWSH"
0x000000b4dd9afd40 : 0x00007ff8919bce70 : KERNELBASE!WaitOnAddress
0x000000b4dd9afd58 : 0x00007ff893f72550 : ntdll!RtlWakeAddressSingle
0x000000b4dd9afd60 : 0x0000024fa0af97c0 : 0x0000024fa0af52c0 :  !da "ALLUSERSPROFILE=C:\ProgramData"
0x000000b4dd9afdb0 : 0x00007ff7d9ad1110 :  !da "UAWAVAUATVWSH"
0x000000b4dd9afdf8 : 0x00007ff893be7034 : kernel32!BaseThreadInitThunk+0x14
0x000000b4dd9afe28 : 0x00007ff893f62651 : ntdll!RtlUserThreadStart+0x21
```

This IP address is also listed as a Cobalt Strike C2 server on VirusTotal.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/da51e7dc-abe1-4081-9432-49ce2a46748e)


We can use regex to simplify the large output. The **`!mex.grep`** command allows regex, and we can use it to specifically look for IP addresses.

```
0:000> !mex.grep -r (\b25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.(\b25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.(\b25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.(\b25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?) !pde.dpx
0x000000b4dd9af8e8 : 0x0000024fa0b71970 :  !da "45.143.235.32"
0x000000b4dd9af8f0 : 0x0000024fa0b5a3e0 :  !da "45.143.235.32,/visit.js"
```


The output from this command is an indicative of patterns associated with Cobalt Strike. The **`!mex.grep`** command allows us to filter on a specific string "HTTP" within a memory range, while the **`!pde.dpx`** command displays memory contents, including pointers and data in a human-readable format. 

```
        BaseAddress      EndAddress+1        RegionSize     Type       State                 Protect             Usage
--------------------------------------------------------------------------------------------------------------------------
     24f`a0a30000      24f`a0a87000        0`00057000 MEM_PRIVATE MEM_COMMIT  PAGE_EXECUTE_READWRITE             <unknown>  [MZARUH..H......H]
```

All these strings are matching the patterns of a default Cobalt Strike Beacon profile, which has been aligned here: https://github.com/h3ll0clar1c3/CRTO/blob/main/CobaltStrikeBeacon_Default.profile

```
0:000> !mex.grep http !pde.dpx 24f`a0a30000 24f`a0a87000
               r11 : 0x000000b4dd9aee60 : 0x0000024fa0c43510 :  dt wininet!CFsm_HttpSendRequest
0x0000024fa0a6e648 : 0x2d77654e28205845 :  !da ""EX (New-Object Net.Webclient).DownloadString('http://127.0.0.1:%u/')""
0x0000024fa0a6e650 : 0x4e207463656a624f :  !da ""Object Net.Webclient).DownloadString('http://127.0.0.1:%u/')""
0x0000024fa0a6e658 : 0x6c636265572e7465 :  !da "et.Webclient).DownloadString('http://127.0.0.1:%u/')"
0x0000024fa0a6e660 : 0x6f442e29746e6569 :  !da "ient).DownloadString('http://127.0.0.1:%u/')"
0x0000024fa0a6e668 : 0x745364616f6c6e77 :  !da "wnloadString('http://127.0.0.1:%u/')"
0x0000024fa0a6e670 : 0x74682728676e6972 :  !da "ring('http://127.0.0.1:%u/')"
0x0000024fa0a6e6b8 : 0x624f2d77654e2820 :  !da "" (New-Object Net.Webclient).DownloadString('http://127.0.0.1:%u/'); %s""
0x0000024fa0a6e6c0 : 0x74654e207463656a :  !da ""ject Net.Webclient).DownloadString('http://127.0.0.1:%u/'); %s""
0x0000024fa0a6e6c8 : 0x65696c636265572e :  !da "".Webclient).DownloadString('http://127.0.0.1:%u/'); %s""
0x0000024fa0a6e6d0 : 0x6e776f442e29746e :  !da ""nt).DownloadString('http://127.0.0.1:%u/'); %s""
0x0000024fa0a6e6d8 : 0x6972745364616f6c :  !da ""loadString('http://127.0.0.1:%u/'); %s""
0x0000024fa0a6e6e0 : 0x707474682728676e :  !da ""ng('http://127.0.0.1:%u/'); %s""
0x0000024fa0a80560 : 0x646e655370747448 :  !da "HttpSendRequestA"
0x0000024fa0a82360 : 0x312e312f50545448 :  !da ""HTTP/1.1 200 OK..Content-Type: application/octet-stream..Content-Length: %d....""
```

Let's also examine this memory region that is marked with the **`PAGE_EXECUTE_READWRITE`** flag as well.

```
24f`a0cf0000      24f`a0d54000        0`00064000 MEM_PRIVATE MEM_COMMIT  PAGE_EXECUTE_READWRITE             <unknown>  [MZARUH..H......H]
```

The output from the **`!mex.grep`** command with a regular expression is used to find IP addresses in a specified memory range. Here we can see the IP address again of our C2, which happens to be **45[.]143[.]235[.]32**

```
0:000> !mex.grep -r (\b25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.(\b25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.(\b25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.(\b25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?) !pde.dpx 24f`a0cf0000 24f`a0d54000
0x0000024fa0d2f248 : 0x2d77654e28205845 :  !da ""EX (New-Object Net.Webclient).DownloadString('http://127.0.0.1:%u/')""
0x0000024fa0d2f250 : 0x4e207463656a624f :  !da ""Object Net.Webclient).DownloadString('http://127.0.0.1:%u/')""
0x0000024fa0d2f258 : 0x6c636265572e7465 :  !da "et.Webclient).DownloadString('http://127.0.0.1:%u/')"
0x0000024fa0d2f260 : 0x6f442e29746e6569 :  !da "ient).DownloadString('http://127.0.0.1:%u/')"
0x0000024fa0d2f268 : 0x745364616f6c6e77 :  !da "wnloadString('http://127.0.0.1:%u/')"
0x0000024fa0d2f270 : 0x74682728676e6972 :  !da "ring('http://127.0.0.1:%u/')"
0x0000024fa0d2f278 : 0x3732312f2f3a7074 :  !da "tp://127.0.0.1:%u/')"
0x0000024fa0d2f2b8 : 0x624f2d77654e2820 :  !da "" (New-Object Net.Webclient).DownloadString('http://127.0.0.1:%u/'); %s""
0x0000024fa0d2f2c0 : 0x74654e207463656a :  !da ""ject Net.Webclient).DownloadString('http://127.0.0.1:%u/'); %s""
0x0000024fa0d2f2c8 : 0x65696c636265572e :  !da "".Webclient).DownloadString('http://127.0.0.1:%u/'); %s""
0x0000024fa0d2f2d0 : 0x6e776f442e29746e :  !da ""nt).DownloadString('http://127.0.0.1:%u/'); %s""
0x0000024fa0d2f2d8 : 0x6972745364616f6c :  !da ""loadString('http://127.0.0.1:%u/'); %s""
0x0000024fa0d2f2e0 : 0x707474682728676e :  !da ""ng('http://127.0.0.1:%u/'); %s""
0x0000024fa0d2f2e8 : 0x302e3732312f2f3a :  !da ""://127.0.0.1:%u/'); %s""
0x0000024fa0d48b30 : 0x0000024fa0c37fb0 :  !da "45.143.235.32"
```

We have identified our C2, so we're technically done. However, let's exhaust this memory dump to see if we can proof more evidence. These commands **`!mex.grep`** and **`!address`** commands are used to filter and display information about memory regions with read-write protection that are allocated as heap memory. 

```
0:000> !mex.grep Heap !address -f:PAGE_READWRITE
     24f`a09f0000      24f`a09f3000        0`00003000 MEM_PRIVATE MEM_COMMIT  PAGE_READWRITE                     Heap       [ID: 0; Handle: 0000024fa0af0000; Type: Front End]
     24f`a0a90000      24f`a0a91000        0`00001000 MEM_PRIVATE MEM_COMMIT  PAGE_READWRITE                     Heap       [ID: 2; Handle: 0000024fa0ae0000; Type: Front End]
     24f`a0ae0000      24f`a0ae7000        0`00007000 MEM_PRIVATE MEM_COMMIT  PAGE_READWRITE                     Heap       [ID: 2; Handle: 0000024fa0ae0000; Type: Segment]
     24f`a0af0000      24f`a0b83000        0`00093000 MEM_PRIVATE MEM_COMMIT  PAGE_READWRITE                     Heap       [ID: 0; Handle: 0000024fa0af0000; Type: Segment]
     24f`a0bf0000      24f`a0c72000        0`00082000 MEM_PRIVATE MEM_COMMIT  PAGE_READWRITE                     Heap       [ID: 0; Handle: 0000024fa0af0000; Type: Segment]
     24f`a10a6000      24f`a11a7000        0`00101000 MEM_PRIVATE MEM_COMMIT  PAGE_READWRITE                     Heap       [ID: 0; Handle: 0000024fa0af0000; Type: Large Block]
```

The heap is a specific area within a process's virtual memory used for dynamic memory allocation. The heap grows and shrinks dynamically as the application allocates and deallocates memory during its execution. It is used for storing objects or variables that need to be accessed and modified during the runtime of a program and whose sizes cannot be determined at compile time.

A **Segment** in heap management refers to a portion of the heap. It is a general term for a contiguous block of memory within a heap, used for memory allocations. Each process has its own heap (or heaps), and the segments within these heaps are specific to that process's memory space.

```
     24f`a0ae0000      24f`a0ae7000        0`00007000 MEM_PRIVATE MEM_COMMIT  PAGE_READWRITE                     Heap       [ID: 2; Handle: 0000024fa0ae0000; Type: Segment]
     24f`a0af0000      24f`a0b83000        0`00093000 MEM_PRIVATE MEM_COMMIT  PAGE_READWRITE                     Heap       [ID: 0; Handle: 0000024fa0af0000; Type: Segment]
     24f`a0bf0000      24f`a0c72000        0`00082000 MEM_PRIVATE MEM_COMMIT  PAGE_READWRITE                     Heap       [ID: 0; Handle: 0000024fa0af0000; Type: Segment]
```

We know that Cobalt Strike uses **`rundll32.exe`** as the **default** process to spawn when it needs to inject a payload or perform a one-off operation. Let's try to hunt for this behavior:

```
0:000> !mex.grep -r "%windir%\\[^\\]+\\[^\\]+\.exe" !pde.dpx 24f`a0af0000 24f`a0b83000 -u
0x0000024fa0b6da00 : 0x257269646e697725 :  !da "%windir%\sysnative\rundll32.exe"
0x0000024fa0b6e130 : 0x257269646e697725 :  !da "%windir%\syswow64\rundll32.exe"
0x0000024fa0b6f100 : 0x0000024fa0b6e130 :  !da "%windir%\syswow64\rundll32.exe"
0x0000024fa0b6f120 : 0x0000024fa0b6da00 :  !da "%windir%\sysnative\rundll32.exe"
```

The output from the **`!mex.grep`** command with a regular expression is used to find IP addresses in a specified memory range of the Heap. Here we can see our IP address of interest again.

```
0:000> !mex.grep -r "(\b25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.(\b25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.(\b25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.(\b25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)" !pde.dpx 24f`a0af0000 24f`a0b83000 -u
0x0000024fa0b3b6d8 : 0x2d77654e28205845 :  !da ""EX (New-Object Net.Webclient).DownloadString('http://127.0.0.1:%u/')""
0x0000024fa0b3b6e0 : 0x4e207463656a624f :  !da ""Object Net.Webclient).DownloadString('http://127.0.0.1:%u/')""
0x0000024fa0b3b6e8 : 0x6c636265572e7465 :  !da "et.Webclient).DownloadString('http://127.0.0.1:%u/')"
0x0000024fa0b3b6f0 : 0x6f442e29746e6569 :  !da "ient).DownloadString('http://127.0.0.1:%u/')"
0x0000024fa0b3b6f8 : 0x745364616f6c6e77 :  !da "wnloadString('http://127.0.0.1:%u/')"
0x0000024fa0b3b700 : 0x74682728676e6972 :  !da "ring('http://127.0.0.1:%u/')"
0x0000024fa0b3b708 : 0x3732312f2f3a7074 :  !da "tp://127.0.0.1:%u/')"
0x0000024fa0b3b748 : 0x624f2d77654e2820 :  !da "" (New-Object Net.Webclient).DownloadString('http://127.0.0.1:%u/'); %s""
0x0000024fa0b3b750 : 0x74654e207463656a :  !da ""ject Net.Webclient).DownloadString('http://127.0.0.1:%u/'); %s""
0x0000024fa0b3b758 : 0x65696c636265572e :  !da "".Webclient).DownloadString('http://127.0.0.1:%u/'); %s""
0x0000024fa0b3b760 : 0x6e776f442e29746e :  !da ""nt).DownloadString('http://127.0.0.1:%u/'); %s""
0x0000024fa0b3b768 : 0x6972745364616f6c :  !da ""loadString('http://127.0.0.1:%u/'); %s""
0x0000024fa0b3b770 : 0x707474682728676e :  !da ""ng('http://127.0.0.1:%u/'); %s""
0x0000024fa0b3b778 : 0x302e3732312f2f3a :  !da ""://127.0.0.1:%u/'); %s""
0x0000024fa0b5a3e0 : 0x322e3334312e3534 :  !da "45.143.235.32,/visit.js"
0x0000024fa0b6f0c0 : 0x0000024fa0b5a3e0 :  !da "45.143.235.32,/visit.js"
0x0000024fa0b71970 : 0x322e3334312e3534 :  !da "45.143.235.32"
0x0000024fa0b755e8 : 0x0000024fa0bf5e70 :  !da "45.143.235.32"
0x0000024fa0b7d350 : 0x322e3334312e3534 :  !da "45.143.235.32"
0x0000024fa0b7df70 : 0x0031002e00350034 :  !du "45.143.235.32"
0x0000024fa0b7eb40 : 0x0031002e00350034 :  !du "45.143.235.32"
```

# Reference

- **Why is rundll32.exe connecting to the internet?** https://www.cobaltstrike.com/blog/why-is-rundll32-exe-connecting-to-the-internet
- **Extracting Cobalt Strike from Windows Error Reporting:** https://bmcder.com/blog/extracting-cobalt-strike-from-windows-error-reporting
- **CobaltStrikeBeacon_Default.profile:** https://github.com/h3ll0clar1c3/CRTO/blob/main/CobaltStrikeBeacon_Default.profile
- **1768.py:** https://blog.didierstevens.com/programs/cobalt-strike-tools/
