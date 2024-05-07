# Summary

One of Cobalt Strike its features is a DNS Beacon, which is a type of payload that uses DNS for C2 communications. Once installed, the DNS Beacon uses the DNS protocol to establish communications with the C2 server. Instead of traditional HTTP/S based C2 communication, it uses DNS requests.

The following extensions are used:

- **MEX:** https://www.microsoft.com/en-us/download/details.aspx?id=53304
- **PDE:** https://onedrive.live.com/?authkey=%21AJeSzeiu8SQ7T4w&cid=DAE128BD454CF957&id=DAE128BD454CF957%2112585&parId=DAE128BD454CF957%217152&o=OneUp

Link to the **Memory Dump** and **ETL** file can be found here:

- https://mega.nz/file/XhFySDpC#BEE8nghfmo4wR6Mc8r12BfNgRy0ExgpNxqAFPWoDEWY

# How to triage a suspected CS Beacon?

First, take a memory dump of the suspected DNS Beacon and load it into WinDbg:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/a5589a89-5643-4b50-9ebf-73f9b338c59b)


Let's start with the **`!mex.di`** command which stands for dump information. This command retrieves information about the machine from which the memory dump originated. It's especially handy when working with multiple memory dumps, as it helps distinguish the analysis related to each specific dump. This is an improved version of the **`vertarget`** command.

```
0:000> !mex.di
Computer Name: etwtesting
User Name: RTCAdmin
PID: 0x1BBC = 0n7100
Windows 10 Version 19045 MP (2 procs) Free x64
Product: WinNt, suite: SingleUserTS
Edition build lab: 19041.1.amd64fre.vb_release.191206-1406
Debug session time: Wed Jan 24 17:21:11.000 2024 (UTC + 1:00)
System Uptime: 0 days 5:07:43.558
Process Uptime: 0 days 0:03:01.000
  Kernel time: 0 days 0:00:00.000
  User time: 0 days 0:00:00.000
```

We can use the **`!mex.us`** command to display unique stacks (call stacks) across threads in a process, helping to identify patterns or anomalies in thread execution. The presence of a call stack involving **`dnsapi`** functions like **`DnsQuery_A`** will suggest already that it could be a DNS Beacon.

```
0:000> !mex.us
1 thread [stats]: 0
    00007fffeba4d664 ntdll!NtDelayExecution+0x14
    00007fffe941b62e KERNELBASE!SleepEx+0x9e
    000000000040305f dnsbeacon+0x305f
    00000000004013b4 dnsbeacon+0x13b4
    00000000004014db dnsbeacon+0x14db
    00007fffea2f7344 kernel32!BaseThreadInitThunk+0x14
    00007fffeba026b1 ntdll!RtlUserThreadStart+0x21

1 thread [stats]: 1
    00007fffeba4e154 ntdll!NtAlpcSendWaitReceivePort+0x14
    00007fffeb883f8f rpcrt4!LRPC_BASE_CCALL::SendReceive+0x12f
    00007fffeb87073f rpcrt4!I_RpcSendReceive+0x6f
    00007fffeb8b9606 rpcrt4!NdrSendReceive+0x36
    00007fffeb91d562 rpcrt4!NdrpClientCall3+0x5d2
    00007fffeb9205d0 rpcrt4!NdrClientCall3+0xf0
    00007fffe84d8cc1 dnsapi!SyncResolverQueryRpc+0x171
    00007fffe84d89c3 dnsapi!Rpc_ResolverQuery+0xc3
    00007fffe84dc327 dnsapi!Query_PrivateExW+0x7a7
    00007fffe84d85a2 dnsapi!Query_Shim+0x10a
    00007fffe8523bf6 dnsapi!DnsQuery_A+0x46
    0000000000c9de27 0xc9de27
    000000000000002c 0x2c
    00000000007ad000 0x7ad000
    0000100000000001 0x100000000001

1 thread [stats]: 2
    00007fffeba50a74 ntdll!NtWaitForWorkViaWorkerFactory+0x14
    00007fffeba02e27 ntdll!TppWorkerThread+0x2f7
    00007fffea2f7344 kernel32!BaseThreadInitThunk+0x14
    00007fffeba026b1 ntdll!RtlUserThreadStart+0x21

3 stack(s) with 3 threads displayed (3 Total threads)
```

We can examine the unique stack to identify the thread that may be of interest. For instance, thread 1 appears relevant in this scenario. It includes calls to various **`dnsapi`** functions, and given our suspicion of a DNS Beacon, this thread is the most logical one to investigate.

```
0:000> !mex.t 1
DbgID ThreadID      User Kernel Create Time (UTC)
1     23b4 (0n9140) 46ms   15ms 01/24/2024 04:18:12.031 PM

# Child-SP         Return           Call Site
0 0000000000c8e868 00007fffeb883f8f ntdll!NtAlpcSendWaitReceivePort+0x14
1 0000000000c8e870 00007fffeb87073f rpcrt4!LRPC_BASE_CCALL::SendReceive+0x12f
2 0000000000c8e940 00007fffeb8b9606 rpcrt4!I_RpcSendReceive+0x6f
3 0000000000c8e970 00007fffeb91d562 rpcrt4!NdrSendReceive+0x36
4 0000000000c8e9a0 00007fffeb9205d0 rpcrt4!NdrpClientCall3+0x5d2
5 0000000000c8ed10 00007fffe84d8cc1 rpcrt4!NdrClientCall3+0xf0
6 0000000000c8f0a0 00007fffe84d89c3 dnsapi!SyncResolverQueryRpc+0x171
7 0000000000c8f2a0 00007fffe84dc327 dnsapi!Rpc_ResolverQuery+0xc3
8 0000000000c8f370 00007fffe84d85a2 dnsapi!Query_PrivateExW+0x7a7
9 0000000000c8fba0 00007fffe8523bf6 dnsapi!Query_Shim+0x10a
a 0000000000c8fcd0 0000000000c9de27 dnsapi!DnsQuery_A+0x46
b 0000000000c8fd20 000000000000002c 0xc9de27
c 0000000000c8fd28 00000000007ad000 0x2c
d 0000000000c8fd30 0000100000000001 0x7ad000
e 0000000000c8fd38 0000000000000000 0x100000000001
```

The **`kv`** command is used to display a stack trace with local variables and function parameters for each frame. From the call stack, we can identify the **`DnsQuery_A`** function which is used to retrieve DNS records for a specified domain name.

```
0:001> kv
  *** Stack trace for last set context - .thread/.cxr resets it
 # Child-SP          RetAddr               : Args to Child                                                           : Call Site
00 00000000`00c8e868 00007fff`eb883f8f     : 00000000`007be1e0 00000000`00000004 00000000`0000004c 00000000`007c214b : ntdll!NtAlpcSendWaitReceivePort+0x14
01 00000000`00c8e870 00007fff`eb87073f     : 00000000`00c8eb20 00000000`00c8f0b8 00000000`007b3ba0 00000000`00000004 : rpcrt4!LRPC_BASE_CCALL::SendReceive+0x12f
02 00000000`00c8e940 00007fff`eb8b9606     : 00000000`00c8eb20 00007fff`e8563090 00000000`00000000 00000000`00000004 : rpcrt4!I_RpcSendReceive+0x6f
03 00000000`00c8e970 00007fff`eb91d562     : 00000000`00c8ed40 00000000`00000004 00000000`00000000 00000000`00000004 : rpcrt4!NdrSendReceive+0x36
04 00000000`00c8e9a0 00007fff`eb9205d0     : 00007fff`e85637e0 00000000`00000004 00000000`00000000 00007fff`e8563090 : rpcrt4!NdrpClientCall3+0x5d2
05 00000000`00c8ed10 00007fff`e84d8cc1     : 00000000`00000000 00000000`00000000 00000000`00000000 00000000`007b3470 : rpcrt4!NdrClientCall3+0xf0
06 00000000`00c8f0a0 00007fff`e84d89c3     : 00000000`00000000 00000000`00000050 00000000`00000000 00000000`00000000 : dnsapi!SyncResolverQueryRpc+0x171
07 00000000`00c8f2a0 00007fff`e84dc327     : 00000000`00000001 00000000`00000000 00000000`00000001 00007fff`00000000 : dnsapi!Rpc_ResolverQuery+0xc3
08 00000000`00c8f370 00007fff`e84d85a2     : 00000000`00000000 00000000`00000029 00000000`00c8fc81 00000000`00020000 : dnsapi!Query_PrivateExW+0x7a7
09 00000000`00c8fba0 00007fff`e8523bf6     : 00000000`00cc1b9c 00000000`00cb363f 00000000`00000000 00000000`00000000 : dnsapi!Query_Shim+0x10a
0a 00000000`00c8fcd0 00000000`00c9de27     : 00000000`0000002c 00000000`007ad000 00001000`00000001 00000000`00000000 : dnsapi!DnsQuery_A+0x46
0b 00000000`00c8fd20 00000000`0000002c     : 00000000`007ad000 00001000`00000001 00000000`00000000 00000000`00c8fdb0 : 0xc9de27
0c 00000000`00c8fd28 00000000`007ad000     : 00001000`00000001 00000000`00000000 00000000`00c8fdb0 00000000`00000000 : 0x2c
0d 00000000`00c8fd30 00001000`00000001     : 00000000`00000000 00000000`00c8fdb0 00000000`00000000 0000fed9`7f11721f : 0x7ad000
0e 00000000`00c8fd38 00000000`00000000     : 00000000`00c8fdb0 00000000`00000000 0000fed9`7f11721f 00000000`00000042 : 0x00001000`00000001
```

The output from the **`db`** command shows a hexadecimal and ASCII representation of the memory contents. The one that we're interested at the arguments at the **`DnsQuery_A`** function, since that would reveal the C2 domain:

```
0:001> db 00000000`007ad000
00000000`007ad000  62 65 61 63 6f 6e 2e 61-7a 75 72 65 6c 61 62 73  beacon.azurelabs
00000000`007ad010  69 6d 75 6c 61 74 69 6f-6e 73 2e 73 74 6f 72 65  imulations.store
00000000`007ad020  2c 2f 5f 5f 75 74 6d 2e-67 69 66 00 00 00 00 00  ,/__utm.gif.....
00000000`007ad030  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
00000000`007ad040  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
00000000`007ad050  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
00000000`007ad060  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
00000000`007ad070  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
```

The **`!mex.mods -3`** is used to display information about modules loaded in the process with a specific focus on limiting the output to third-party modules. From the results, we don't have any third-party modules that were loaded into the current process.

```
0:001> !mex.mods -3
Number of modules: loaded: 20 unloaded: 0 
Base     End        Flags   Time stamp           CLR   Module name   Version   Path   Filters:[ Interest|3rd|K|U|A ] More...
========================================================================================================================
00400000 00454000 |       |  Invalid Timestamp |  No | dnsbeacon   | 0.0.0.0 | C:\Users\RTCAdmin\Downloads\dnsbeacon.exe
```

We can look for the loaded imports to give us insight into its capabilities and potential actions it can perform. We can see some memory management functions for instance with the likes of **`VirtualAlloc`**, **`VirtualProtect`**, and **`VirtualFree`**. 

As well as other functions, such as **`CreateNamedPipeA`** and **`ConnectNamedPipe`**. **`CreateNamedPipeA`** is used to create an instance of a named pipe, while **`ConnectNamedPipe`** used to wait for a client process to connect to an instance of a named pipe.

```
0:000> !mex.imports -a 0x0000000000400000
Dumping Import table at 0000000000400000 + 51000 
ModuleImageName: dnsbeacon
dnsbeacon imports KERNEL32.dll!CloseHandle
dnsbeacon imports KERNEL32.dll!ConnectNamedPipe
dnsbeacon imports KERNEL32.dll!CreateFileA
dnsbeacon imports KERNEL32.dll!CreateNamedPipeA
dnsbeacon imports KERNEL32.dll!CreateThread
dnsbeacon imports KERNEL32.dll!DeleteCriticalSection
dnsbeacon imports KERNEL32.dll!EnterCriticalSection
dnsbeacon imports KERNEL32.dll!GetCurrentProcess
dnsbeacon imports KERNEL32.dll!GetCurrentProcessId
dnsbeacon imports KERNEL32.dll!GetCurrentThreadId
dnsbeacon imports KERNEL32.dll!GetLastError
dnsbeacon imports KERNEL32.dll!GetModuleHandleA
dnsbeacon imports KERNEL32.dll!GetProcAddress
dnsbeacon imports KERNEL32.dll!GetStartupInfoA
dnsbeacon imports KERNEL32.dll!GetSystemTimeAsFileTime
dnsbeacon imports KERNEL32.dll!GetTickCount
dnsbeacon imports KERNEL32.dll!InitializeCriticalSection
dnsbeacon imports KERNEL32.dll!LeaveCriticalSection
dnsbeacon imports KERNEL32.dll!QueryPerformanceCounter
dnsbeacon imports KERNEL32.dll!ReadFile
dnsbeacon imports KERNEL32.dll!RtlAddFunctionTable
dnsbeacon imports KERNEL32.dll!RtlCaptureContext
dnsbeacon imports KERNEL32.dll!RtlLookupFunctionEntry
dnsbeacon imports KERNEL32.dll!RtlVirtualUnwind
dnsbeacon imports KERNEL32.dll!SetUnhandledExceptionFilter
dnsbeacon imports KERNEL32.dll!Sleep
dnsbeacon imports KERNEL32.dll!TerminateProcess
dnsbeacon imports KERNEL32.dll!TlsGetValue
dnsbeacon imports KERNEL32.dll!UnhandledExceptionFilter
dnsbeacon imports KERNEL32.dll!VirtualAlloc
dnsbeacon imports KERNEL32.dll!VirtualProtect
dnsbeacon imports KERNEL32.dll!VirtualQuery
dnsbeacon imports KERNEL32.dll!WriteFile
```

We can run strings against the binary itself to see if we can find some quick patterns that matches Cobalt Strike with the **`!mex.strings`** command. In this particular case, knowing that a named pipe is created, we can look for named pipes within the strings.

```
0:000> !mex.grep -r "(MSSE-[0-9]{4}-server)" !mex.strings -m 0x0000000000400000
0000000000450980 \\.\pipe\MSSE-1378-server
```

Finding a memory region with **`PAGE_EXECUTE_READWRITE`** protection, especially in the context of potential malware can be indicative of malicious activity. **`PAGE_EXECUTE_READWRITE`** is memory protection flag allows a region of memory to be executed, read, and written to. Malware often uses **`PAGE_EXECUTE_READWRITE`** permissions for its memory regions to dynamically write and execute malicious code.

```
0:000> !address -f:PAGE_EXECUTE_READWRITE

                                     
Mapping file section regions...
Mapping module regions...
Mapping PEB regions...
Mapping TEB and stack regions...
Mapping heap regions...
Mapping page heap regions...
Mapping other regions...
Mapping stack trace database regions...
Mapping activation context regions...

        BaseAddress      EndAddress+1        RegionSize     Type       State                 Protect             Usage
--------------------------------------------------------------------------------------------------------------------------
       0`00c90000        0`00ce6000        0`00056000 MEM_PRIVATE MEM_COMMIT  PAGE_EXECUTE_READWRITE             <unknown>  [..ARUH..H......H]
```

Cobalt Strike utilizes various prefixes for DNS queries, which we can read here: https://hstechdocs.helpsystems.com/manuals/cobaltstrike/current/userguide/content/topics/malleable-c2_dns-beacons.htm - We can use the ETW Provider **`Microsoft-Windows-DNS-Client`** for dynamic analysis and gather the different DNS queries of the DNS Beacon.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/010bc326-840c-4cc2-88ce-b04f2771ca54)



The results from the ETW trace aligns well with what is described here: https://blog.sekoia.io/hunting-and-detecting-cobalt-strike/

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/867b616c-d464-4f82-a29a-e0b79b9cdaac)



I was exploring call stacks to determine if I could pinpoint different DNS queries. After poking around here and there, I was able to identify at a particular thread that does contains DNS queries. The call stack begins with a call to **`NtDelayExecution`**, which shows that the thread is involved in a sleep or delay operation through **`SleepEx`**.

```
0:001> !mex.t 0
DbgID ThreadID      User Kernel Create Time (UTC)
0     1278 (0n4728)    0      0 01/24/2024 04:18:10.972 PM

# Child-SP         Return           Call Site
0 000000000065fd58 00007fffe941b62e ntdll!NtDelayExecution+0x14
1 000000000065fd60 000000000040305f KERNELBASE!SleepEx+0x9e
2 000000000065fe00 00000000004013b4 dnsbeacon+0x305f
3 000000000065fe30 00000000004014db dnsbeacon+0x13b4
4 000000000065ff00 00007fffea2f7344 dnsbeacon+0x14db
5 000000000065ff30 00007fffeba026b1 kernel32!BaseThreadInitThunk+0x14
6 000000000065ff60 0000000000000000 ntdll!RtlUserThreadStart+0x21
```

The regex is designed to match specific prefixes followed by typical domain formats that are known being leveraged by DNS Beacons. The output reveals different DNS queries that we also could see in the ETL trace. 

```
0:000> !mex.grep -r "\b(api\.|cdn\.|www6\.|www\.|post\.)[\w.-]+\.[a-zA-Z]{2,}(\S*)" !pde.dpx 00000000`0065fd58 00007fff`e941b62e -u
0x00000000007a9370 : 0x356234322e697061 :  !da "api.24b5e57054d.609af4ba.beacon.azurelabsimulations.store"
0x00000000007a9410 : 0x353634322e697061 :  !da "api.2465e57054d.609af4ba.beacon.azurelabsimulations.store"
0x00000000007a94b0 : 0x353132322e697061 :  !da "api.2215e57054d.609af4ba.beacon.azurelabsimulations.store"
0x00000000007a9500 : 0x356535322e697061 :  !da "api.25e5e57054d.609af4ba.beacon.azurelabsimulations.store"
0x00000000007a95a0 : 0x353737312e697061 :  !da "api.1775e57054d.609af4ba.beacon.azurelabsimulations.store"
0x00000000007a9910 : 0x3065312e74736f70 :  !da "post.1e0.080c1b9a.609af4ba.beacon.azurelabsimulations.store"
0x00000000007a9960 : 0x353734322e697061 :  !da "api.2475e57054d.609af4ba.beacon.azurelabsimulations.store"
0x00000000007a9a50 : 0x353135322e697061 :  !da "api.2515e57054d.609af4ba.beacon.azurelabsimulations.store"
0x00000000007a9af0 : 0x353835322e697061 :  !da "api.2585e57054d.609af4ba.beacon.azurelabsimulations.store"
0x00000000007a9e60 : 0x353034322e697061 :  !da "api.2405e57054d.609af4ba.beacon.azurelabsimulations.store"
0x00000000007a9eb0 : 0x353664312e697061 :  !da "api.1d65e57054d.609af4ba.beacon.azurelabsimulations.store"
0x00000000007a9f50 : 0x356634322e697061 :  !da "api.24f5e57054d.609af4ba.beacon.azurelabsimulations.store"
0x00000000007aa220 : 0x353333322e697061 :  !da "api.2335e57054d.609af4ba.beacon.azurelabsimulations.store"
0x00000000007b1300 : 0x002e006900700061 :  !du "api.95e57054d.609af4ba.beacon.azurelabsimulations.store"
0x00000000007b1780 : 0x002e006900700061 :  !du "api.e5e57054d.609af4ba.beacon.azurelabsimulations.store"
0x00000000007b1810 : 0x002e006900700061 :  !du "api.85e57054d.609af4ba.beacon.azurelabsimulations.store"
0x00000000007b1b70 : 0x002e006900700061 :  !du "api.f5e57054d.609af4ba.beacon.azurelabsimulations.store"
0x00000000007b1db0 : 0x002e006900700061 :  !du "api.25e57054d.609af4ba.beacon.azurelabsimulations.store"
0x00000000007b1e40 : 0x002e006900700061 :  !du "api.118773d75.609af4ba.beacon.azurelabsimulations.store"
0x00000000007b1ed0 : 0x002e006900700061 :  !du "api.b5e57054d.609af4ba.beacon.azurelabsimulations.store"
0x00000000007b1ff0 : 0x002e006900700061 :  !du "api.75e57054d.609af4ba.beacon.azurelabsimulations.store"
```

Since we know the domain name, we could also do a regex based on the domain name. This will lead to more results:

```
0:000> !mex.grep -r "[a-zA-Z0-9-]+\.beacon\.azurelabsimulations\.store\b" !pde.dpx 00000000`0065fd58 00007fff`e941b62e -u
0x000000000079d2d0 : 0x00000000007abc00 :  !da "609af4ba.beacon.azurelabsimulations.store"
0x00000000007a8920 : 0x6162346661393036 :  !da "609af4ba.beacon.azurelabsimulations.store"
0x00000000007a89a0 : 0x6162346661393036 :  !da "609af4ba.beacon.azurelabsimulations.store"
0x00000000007a89e0 : 0x6162346661393036 :  !da "609af4ba.beacon.azurelabsimulations.store"
0x00000000007a8ca0 : 0x6162346661393036 :  !da "609af4ba.beacon.azurelabsimulations.store"
0x00000000007a8d60 : 0x6162346661393036 :  !da "609af4ba.beacon.azurelabsimulations.store"
0x00000000007a8de0 : 0x6162346661393036 :  !da "609af4ba.beacon.azurelabsimulations.store"
0x00000000007a8e20 : 0x6162346661393036 :  !da "609af4ba.beacon.azurelabsimulations.store"
0x00000000007a8fa0 : 0x6162346661393036 :  !da "609af4ba.beacon.azurelabsimulations.store"
0x00000000007a8fe0 : 0x6162346661393036 :  !da "609af4ba.beacon.azurelabsimulations.store"
0x00000000007a9370 : 0x356234322e697061 :  !da "api.24b5e57054d.609af4ba.beacon.azurelabsimulations.store"
0x00000000007a9378 : 0x2e64343530373565 :  !da "e57054d.609af4ba.beacon.azurelabsimulations.store"
0x00000000007a9380 : 0x6162346661393036 :  !da "609af4ba.beacon.azurelabsimulations.store"
0x00000000007a93d0 : 0x6162346661393036 :  !da "609af4ba.beacon.azurelabsimulations.store"
0x00000000007a9410 : 0x353634322e697061 :  !da "api.2465e57054d.609af4ba.beacon.azurelabsimulations.store"
0x00000000007a9418 : 0x2e64343530373565 :  !da "e57054d.609af4ba.beacon.azurelabsimulations.store"
0x00000000007a9420 : 0x6162346661393036 :  !da "609af4ba.beacon.azurelabsimulations.store"
0x00000000007a94b0 : 0x353132322e697061 :  !da "api.2215e57054d.609af4ba.beacon.azurelabsimulations.store"
0x00000000007a94b8 : 0x2e64343530373565 :  !da "e57054d.609af4ba.beacon.azurelabsimulations.store"
0x00000000007a94c0 : 0x6162346661393036 :  !da "609af4ba.beacon.azurelabsimulations.store"
0x00000000007a9500 : 0x356535322e697061 :  !da "api.25e5e57054d.609af4ba.beacon.azurelabsimulations.store"
0x00000000007a9508 : 0x2e64343530373565 :  !da "e57054d.609af4ba.beacon.azurelabsimulations.store"
0x00000000007a9510 : 0x6162346661393036 :  !da "609af4ba.beacon.azurelabsimulations.store"
0x00000000007a95a0 : 0x353737312e697061 :  !da "api.1775e57054d.609af4ba.beacon.azurelabsimulations.store"
0x00000000007a95a8 : 0x2e64343530373565 :  !da "e57054d.609af4ba.beacon.azurelabsimulations.store"
0x00000000007a95b0 : 0x6162346661393036 :  !da "609af4ba.beacon.azurelabsimulations.store"
0x00000000007a9910 : 0x3065312e74736f70 :  !da "post.1e0.080c1b9a.609af4ba.beacon.azurelabsimulations.store"
0x00000000007a9918 : 0x396231633038302e :  !da ".080c1b9a.609af4ba.beacon.azurelabsimulations.store"
0x00000000007a9920 : 0x3466613930362e61 :  !da "a.609af4ba.beacon.azurelabsimulations.store"
0x00000000007a9928 : 0x6f636165622e6162 :  !da "ba.beacon.azurelabsimulations.store"
0x00000000007a9960 : 0x353734322e697061 :  !da "api.2475e57054d.609af4ba.beacon.azurelabsimulations.store"
0x00000000007a9968 : 0x2e64343530373565 :  !da "e57054d.609af4ba.beacon.azurelabsimulations.store"
0x00000000007a9970 : 0x6162346661393036 :  !da "609af4ba.beacon.azurelabsimulations.store"
0x00000000007a9a50 : 0x353135322e697061 :  !da "api.2515e57054d.609af4ba.beacon.azurelabsimulations.store"
0x00000000007a9a58 : 0x2e64343530373565 :  !da "e57054d.609af4ba.beacon.azurelabsimulations.store"
0x00000000007a9a60 : 0x6162346661393036 :  !da "609af4ba.beacon.azurelabsimulations.store"
0x00000000007a9af0 : 0x353835322e697061 :  !da "api.2585e57054d.609af4ba.beacon.azurelabsimulations.store"
0x00000000007a9af8 : 0x2e64343530373565 :  !da "e57054d.609af4ba.beacon.azurelabsimulations.store"
0x00000000007a9b00 : 0x6162346661393036 :  !da "609af4ba.beacon.azurelabsimulations.store"
0x00000000007a9e60 : 0x353034322e697061 :  !da "api.2405e57054d.609af4ba.beacon.azurelabsimulations.store"
0x00000000007a9e68 : 0x2e64343530373565 :  !da "e57054d.609af4ba.beacon.azurelabsimulations.store"
0x00000000007a9e70 : 0x6162346661393036 :  !da "609af4ba.beacon.azurelabsimulations.store"
0x00000000007a9eb0 : 0x353664312e697061 :  !da "api.1d65e57054d.609af4ba.beacon.azurelabsimulations.store"
0x00000000007a9eb8 : 0x2e64343530373565 :  !da "e57054d.609af4ba.beacon.azurelabsimulations.store"
0x00000000007a9ec0 : 0x6162346661393036 :  !da "609af4ba.beacon.azurelabsimulations.store"
0x00000000007a9f50 : 0x356634322e697061 :  !da "api.24f5e57054d.609af4ba.beacon.azurelabsimulations.store"
0x00000000007a9f58 : 0x2e64343530373565 :  !da "e57054d.609af4ba.beacon.azurelabsimulations.store"
0x00000000007a9f60 : 0x6162346661393036 :  !da "609af4ba.beacon.azurelabsimulations.store"
0x00000000007aa220 : 0x353333322e697061 :  !da "api.2335e57054d.609af4ba.beacon.azurelabsimulations.store"
0x00000000007aa228 : 0x2e64343530373565 :  !da "e57054d.609af4ba.beacon.azurelabsimulations.store"
0x00000000007aa230 : 0x6162346661393036 :  !da "609af4ba.beacon.azurelabsimulations.store"
0x00000000007abc00 : 0x6162346661393036 :  !da "609af4ba.beacon.azurelabsimulations.store"
0x00000000007ac2f8 : 0x00000000007abc00 :  !da "609af4ba.beacon.azurelabsimulations.store"
0x00000000007b0960 : 0x000000000079d2d0 : 0x00000000007abc00 :  !da "609af4ba.beacon.azurelabsimulations.store"
0x00000000007b0980 : 0x00000000007abc00 :  !da "609af4ba.beacon.azurelabsimulations.store"
0x00000000007b1300 : 0x002e006900700061 :  !du "api.95e57054d.609af4ba.beacon.azurelabsimulations.store"
0x00000000007b1308 : 0x0035006500350039 :  !du "95e57054d.609af4ba.beacon.azurelabsimulations.store"
0x00000000007b1310 : 0x0034003500300037 :  !du "7054d.609af4ba.beacon.azurelabsimulations.store"
0x00000000007b1318 : 0x00300036002e0064 :  !du "d.609af4ba.beacon.azurelabsimulations.store"
0x00000000007b1320 : 0x0034006600610039 :  !du "9af4ba.beacon.azurelabsimulations.store"
0x00000000007b1328 : 0x0062002e00610062 :  !du "ba.beacon.azurelabsimulations.store"
0x00000000007b1780 : 0x002e006900700061 :  !du "api.e5e57054d.609af4ba.beacon.azurelabsimulations.store"
0x00000000007b1788 : 0x0035006500350065 :  !du "e5e57054d.609af4ba.beacon.azurelabsimulations.store"
0x00000000007b1790 : 0x0034003500300037 :  !du "7054d.609af4ba.beacon.azurelabsimulations.store"
0x00000000007b1798 : 0x00300036002e0064 :  !du "d.609af4ba.beacon.azurelabsimulations.store"
0x00000000007b17a0 : 0x0034006600610039 :  !du "9af4ba.beacon.azurelabsimulations.store"
0x00000000007b17a8 : 0x0062002e00610062 :  !du "ba.beacon.azurelabsimulations.store"
0x00000000007b1810 : 0x002e006900700061 :  !du "api.85e57054d.609af4ba.beacon.azurelabsimulations.store"
0x00000000007b1818 : 0x0035006500350038 :  !du "85e57054d.609af4ba.beacon.azurelabsimulations.store"
0x00000000007b1820 : 0x0034003500300037 :  !du "7054d.609af4ba.beacon.azurelabsimulations.store"
0x00000000007b1828 : 0x00300036002e0064 :  !du "d.609af4ba.beacon.azurelabsimulations.store"
0x00000000007b1830 : 0x0034006600610039 :  !du "9af4ba.beacon.azurelabsimulations.store"
0x00000000007b1838 : 0x0062002e00610062 :  !du "ba.beacon.azurelabsimulations.store"
0x00000000007b1b70 : 0x002e006900700061 :  !du "api.f5e57054d.609af4ba.beacon.azurelabsimulations.store"
0x00000000007b1b78 : 0x0035006500350066 :  !du "f5e57054d.609af4ba.beacon.azurelabsimulations.store"
0x00000000007b1b80 : 0x0034003500300037 :  !du "7054d.609af4ba.beacon.azurelabsimulations.store"
0x00000000007b1b88 : 0x00300036002e0064 :  !du "d.609af4ba.beacon.azurelabsimulations.store"
0x00000000007b1b90 : 0x0034006600610039 :  !du "9af4ba.beacon.azurelabsimulations.store"
0x00000000007b1b98 : 0x0062002e00610062 :  !du "ba.beacon.azurelabsimulations.store"
0x00000000007b1d38 : 0x00300036002e0064 :  !du "d.609af4ba.beacon.azurelabsimulations.store"
0x00000000007b1d40 : 0x0034006600610039 :  !du "9af4ba.beacon.azurelabsimulations.store"
0x00000000007b1d48 : 0x0062002e00610062 :  !du "ba.beacon.azurelabsimulations.store"
0x00000000007b1db0 : 0x002e006900700061 :  !du "api.25e57054d.609af4ba.beacon.azurelabsimulations.store"
0x00000000007b1db8 : 0x0035006500350032 :  !du "25e57054d.609af4ba.beacon.azurelabsimulations.store"
0x00000000007b1dc0 : 0x0034003500300037 :  !du "7054d.609af4ba.beacon.azurelabsimulations.store"
0x00000000007b1dc8 : 0x00300036002e0064 :  !du "d.609af4ba.beacon.azurelabsimulations.store"
0x00000000007b1dd0 : 0x0034006600610039 :  !du "9af4ba.beacon.azurelabsimulations.store"
0x00000000007b1dd8 : 0x0062002e00610062 :  !du "ba.beacon.azurelabsimulations.store"
0x00000000007b1e40 : 0x002e006900700061 :  !du "api.118773d75.609af4ba.beacon.azurelabsimulations.store"
0x00000000007b1e48 : 0x0037003800310031 :  !du "118773d75.609af4ba.beacon.azurelabsimulations.store"
0x00000000007b1e50 : 0x0037006400330037 :  !du "73d75.609af4ba.beacon.azurelabsimulations.store"
0x00000000007b1e58 : 0x00300036002e0035 :  !du "5.609af4ba.beacon.azurelabsimulations.store"
0x00000000007b1e60 : 0x0034006600610039 :  !du "9af4ba.beacon.azurelabsimulations.store"
0x00000000007b1e68 : 0x0062002e00610062 :  !du "ba.beacon.azurelabsimulations.store"
0x00000000007b1ed0 : 0x002e006900700061 :  !du "api.b5e57054d.609af4ba.beacon.azurelabsimulations.store"
0x00000000007b1ed8 : 0x0035006500350062 :  !du "b5e57054d.609af4ba.beacon.azurelabsimulations.store"
0x00000000007b1ee0 : 0x0034003500300037 :  !du "7054d.609af4ba.beacon.azurelabsimulations.store"
0x00000000007b1ee8 : 0x00300036002e0064 :  !du "d.609af4ba.beacon.azurelabsimulations.store"
0x00000000007b1ef0 : 0x0034006600610039 :  !du "9af4ba.beacon.azurelabsimulations.store"
0x00000000007b1ef8 : 0x0062002e00610062 :  !du "ba.beacon.azurelabsimulations.store"
0x00000000007b1ff0 : 0x002e006900700061 :  !du "api.75e57054d.609af4ba.beacon.azurelabsimulations.store"
0x00000000007b1ff8 : 0x0035006500350037 :  !du "75e57054d.609af4ba.beacon.azurelabsimulations.store"
0x00000000007b2000 : 0x0034003500300037 :  !du "7054d.609af4ba.beacon.azurelabsimulations.store"
0x00000000007b2008 : 0x00300036002e0064 :  !du "d.609af4ba.beacon.azurelabsimulations.store"
0x00000000007b2010 : 0x0034006600610039 :  !du "9af4ba.beacon.azurelabsimulations.store"
0x00000000007b2018 : 0x0062002e00610062 :  !du "ba.beacon.azurelabsimulations.store"
0x00000000007b3d28 : 0x3163303164306435 :  !da "5d0d10c160943b21e2b7e9c3.180c1b9a.609af4ba.beacon.azurelabsimulations.store"
0x00000000007b3d30 : 0x3132623334393036 :  !da "60943b21e2b7e9c3.180c1b9a.609af4ba.beacon.azurelabsimulations.store"
0x00000000007b3d38 : 0x3363396537623265 :  !da "e2b7e9c3.180c1b9a.609af4ba.beacon.azurelabsimulations.store"
0x00000000007b3d40 : 0x396231633038312e :  !da ".180c1b9a.609af4ba.beacon.azurelabsimulations.store"
0x00000000007b3d48 : 0x3466613930362e61 :  !da "a.609af4ba.beacon.azurelabsimulations.store"
0x00000000007b3d50 : 0x6f636165622e6162 :  !da "ba.beacon.azurelabsimulations.store"
0x00000000007b3ff8 : 0x6633663863653434 :  !da "44ec8f3fbb0e7c68c0974e3.144d45a74.609af4ba.beacon.azurelabsimulations.store"
0x00000000007b4000 : 0x3836633765306262 :  !da "bb0e7c68c0974e3.144d45a74.609af4ba.beacon.azurelabsimulations.store"
0x00000000007b4008 : 0x2e33653437393063 :  !da "c0974e3.144d45a74.609af4ba.beacon.azurelabsimulations.store"
0x00000000007b4010 : 0x3761353464343431 :  !da "144d45a74.609af4ba.beacon.azurelabsimulations.store"
0x00000000007b4018 : 0x3466613930362e34 :  !da "4.609af4ba.beacon.azurelabsimulations.store"
0x00000000007b4020 : 0x6f636165622e6162 :  !da "ba.beacon.azurelabsimulations.store"
0x00000000007b4778 : 0x6231373437386538 :  !da "8e87471bdf77bd394a4350cf.280c1b9a.609af4ba.beacon.azurelabsimulations.store"
0x00000000007b4780 : 0x3933646237376664 :  !da "df77bd394a4350cf.280c1b9a.609af4ba.beacon.azurelabsimulations.store"
0x00000000007b4788 : 0x6663303533346134 :  !da "4a4350cf.280c1b9a.609af4ba.beacon.azurelabsimulations.store"
0x00000000007b4790 : 0x396231633038322e :  !da ".280c1b9a.609af4ba.beacon.azurelabsimulations.store"
0x00000000007b4798 : 0x3466613930362e61 :  !da "a.609af4ba.beacon.azurelabsimulations.store"
0x00000000007b47a0 : 0x6f636165622e6162 :  !da "ba.beacon.azurelabsimulations.store"
0x00000000007b4bb0 : 0x0061003900300036 :  !du "609af4ba.beacon.azurelabsimulations.store"
0x00000000007b4bb8 : 0x0061006200340066 :  !du "f4ba.beacon.azurelabsimulations.store"
0x00000000007b4c70 : 0x0061003900300036 :  !du "609af4ba.beacon.azurelabsimulations.store"
0x00000000007b4c78 : 0x0061006200340066 :  !du "f4ba.beacon.azurelabsimulations.store"
0x00000000007b4d30 : 0x0061003900300036 :  !du "609af4ba.beacon.azurelabsimulations.store"
0x00000000007b4d38 : 0x0061006200340066 :  !du "f4ba.beacon.azurelabsimulations.store"
0x00000000007b4df0 : 0x0061003900300036 :  !du "609af4ba.beacon.azurelabsimulations.store"
0x00000000007b4df8 : 0x0061006200340066 :  !du "f4ba.beacon.azurelabsimulations.store"
0x00000000007b4e50 : 0x0061003900300036 :  !du "609af4ba.beacon.azurelabsimulations.store"
0x00000000007b4e58 : 0x0061006200340066 :  !du "f4ba.beacon.azurelabsimulations.store"
0x00000000007b4eb0 : 0x0061003900300036 :  !du "609af4ba.beacon.azurelabsimulations.store"
0x00000000007b4eb8 : 0x0061006200340066 :  !du "f4ba.beacon.azurelabsimulations.store"
0x00000000007b4f10 : 0x0061003900300036 :  !du "609af4ba.beacon.azurelabsimulations.store"
0x00000000007b4f18 : 0x0061006200340066 :  !du "f4ba.beacon.azurelabsimulations.store"
0x00000000007b4f70 : 0x0061003900300036 :  !du "609af4ba.beacon.azurelabsimulations.store"
0x00000000007b4f78 : 0x0061006200340066 :  !du "f4ba.beacon.azurelabsimulations.store"
0x00000000007b4fd0 : 0x0061003900300036 :  !du "609af4ba.beacon.azurelabsimulations.store"
0x00000000007b4fd8 : 0x0061006200340066 :  !du "f4ba.beacon.azurelabsimulations.store"
0x00000000007b5030 : 0x0061003900300036 :  !du "609af4ba.beacon.azurelabsimulations.store"
0x00000000007b5038 : 0x0061006200340066 :  !du "f4ba.beacon.azurelabsimulations.store"
0x00000000007b50f0 : 0x0061003900300036 :  !du "609af4ba.beacon.azurelabsimulations.store"
0x00000000007b50f8 : 0x0061006200340066 :  !du "f4ba.beacon.azurelabsimulations.store"
0x00000000007b5150 : 0x0061003900300036 :  !du "609af4ba.beacon.azurelabsimulations.store"
0x00000000007b5158 : 0x0061006200340066 :  !du "f4ba.beacon.azurelabsimulations.store"
0x00000000007b5270 : 0x0061003900300036 :  !du "609af4ba.beacon.azurelabsimulations.store"
0x00000000007b5278 : 0x0061006200340066 :  !du "f4ba.beacon.azurelabsimulations.store"
0x00000000007b5330 : 0x0061003900300036 :  !du "609af4ba.beacon.azurelabsimulations.store"
0x00000000007b5338 : 0x0061006200340066 :  !du "f4ba.beacon.azurelabsimulations.store"
0x00000000007b53f0 : 0x0061003900300036 :  !du "609af4ba.beacon.azurelabsimulations.store"
0x00000000007b53f8 : 0x0061006200340066 :  !du "f4ba.beacon.azurelabsimulations.store"
0x00000000007b5450 : 0x0061003900300036 :  !du "609af4ba.beacon.azurelabsimulations.store"
0x00000000007b5458 : 0x0061006200340066 :  !du "f4ba.beacon.azurelabsimulations.store"
0x00000000007b5510 : 0x0061003900300036 :  !du "609af4ba.beacon.azurelabsimulations.store"
0x00000000007b5518 : 0x0061006200340066 :  !du "f4ba.beacon.azurelabsimulations.store"
0x00000000007b5570 : 0x0061003900300036 :  !du "609af4ba.beacon.azurelabsimulations.store"
0x00000000007b5578 : 0x0061006200340066 :  !du "f4ba.beacon.azurelabsimulations.store"
0x00000000007b55d0 : 0x0061003900300036 :  !du "609af4ba.beacon.azurelabsimulations.store"
0x00000000007b55d8 : 0x0061006200340066 :  !du "f4ba.beacon.azurelabsimulations.store"
0x00000000007b5630 : 0x0061003900300036 :  !du "609af4ba.beacon.azurelabsimulations.store"
0x00000000007b5638 : 0x0061006200340066 :  !du "f4ba.beacon.azurelabsimulations.store"
0x00000000007b5690 : 0x0061003900300036 :  !du "609af4ba.beacon.azurelabsimulations.store"
0x00000000007b5698 : 0x0061006200340066 :  !du "f4ba.beacon.azurelabsimulations.store"
0x00000000007b56f0 : 0x0061003900300036 :  !du "609af4ba.beacon.azurelabsimulations.store"
0x00000000007b56f8 : 0x0061006200340066 :  !du "f4ba.beacon.azurelabsimulations.store"
0x00000000007b5870 : 0x0061003900300036 :  !du "609af4ba.beacon.azurelabsimulations.store"
0x00000000007b5878 : 0x0061006200340066 :  !du "f4ba.beacon.azurelabsimulations.store"
0x00000000007b5990 : 0x0061003900300036 :  !du "609af4ba.beacon.azurelabsimulations.store"
0x00000000007b5998 : 0x0061006200340066 :  !du "f4ba.beacon.azurelabsimulations.store"
0x00000000007b59f0 : 0x0061003900300036 :  !du "609af4ba.beacon.azurelabsimulations.store"
0x00000000007b59f8 : 0x0061006200340066 :  !du "f4ba.beacon.azurelabsimulations.store"
0x00000000007b5a50 : 0x0061003900300036 :  !du "609af4ba.beacon.azurelabsimulations.store"
0x00000000007b5a58 : 0x0061006200340066 :  !du "f4ba.beacon.azurelabsimulations.store"
0x00000000007b6808 : 0x0061003900300036 :  !du "609af4ba.beacon.azurelabsimulations.store"
0x00000000007b6810 : 0x0061006200340066 :  !du "f4ba.beacon.azurelabsimulations.store"
0x00000000007be828 : 0x0061003900300036 :  !du "609af4ba.beacon.azurelabsimulations.store"
0x00000000007be830 : 0x0061006200340066 :  !du "f4ba.beacon.azurelabsimulations.store"
0x00000000007be980 : 0x002e003300650034 :  !du "4e3.144d45a74.609af4ba.beacon.azurelabsimulations.store"
0x00000000007be988 : 0x0064003400340031 :  !du "144d45a74.609af4ba.beacon.azurelabsimulations.store"
0x00000000007be990 : 0x0037006100350034 :  !du "45a74.609af4ba.beacon.azurelabsimulations.store"
0x00000000007be998 : 0x00300036002e0034 :  !du "4.609af4ba.beacon.azurelabsimulations.store"
0x00000000007be9a0 : 0x0034006600610039 :  !du "9af4ba.beacon.azurelabsimulations.store"
0x00000000007be9a8 : 0x0062002e00610062 :  !du "ba.beacon.azurelabsimulations.store"
0x00000000007bf888 : 0x0061003900300036 :  !du "609af4ba.beacon.azurelabsimulations.store"
0x00000000007bf890 : 0x0061006200340066 :  !du "f4ba.beacon.azurelabsimulations.store"
0x00000000007bf940 : 0x0030006400300034 :  !du "40d0a88f392a1e1b4f43c337077.380c1b9a.609af4ba.beacon.azurelabsimulations.store"
0x00000000007bf948 : 0x0066003800380061 :  !du "a88f392a1e1b4f43c337077.380c1b9a.609af4ba.beacon.azurelabsimulations.store"
0x00000000007bf950 : 0x0061003200390033 :  !du "392a1e1b4f43c337077.380c1b9a.609af4ba.beacon.azurelabsimulations.store"
0x00000000007bf958 : 0x0062003100650031 :  !du "1e1b4f43c337077.380c1b9a.609af4ba.beacon.azurelabsimulations.store"
0x00000000007bf960 : 0x0033003400660034 :  !du "4f43c337077.380c1b9a.609af4ba.beacon.azurelabsimulations.store"
0x00000000007bf968 : 0x0037003300330063 :  !du "c337077.380c1b9a.609af4ba.beacon.azurelabsimulations.store"
0x00000000007bf970 : 0x002e003700370030 :  !du "077.380c1b9a.609af4ba.beacon.azurelabsimulations.store"
0x00000000007bf978 : 0x0063003000380033 :  !du "380c1b9a.609af4ba.beacon.azurelabsimulations.store"
0x00000000007bf980 : 0x0061003900620031 :  !du "1b9a.609af4ba.beacon.azurelabsimulations.store"
0x00000000007bf988 : 0x003900300036002e :  !du ".609af4ba.beacon.azurelabsimulations.store"
0x00000000007bf990 : 0x0062003400660061 :  !du "af4ba.beacon.azurelabsimulations.store"
0x00000000007bf998 : 0x00650062002e0061 :  !du "a.beacon.azurelabsimulations.store"
0x00000000007c08e8 : 0x0061003900300036 :  !du "609af4ba.beacon.azurelabsimulations.store"
0x00000000007c08f0 : 0x0061006200340066 :  !du "f4ba.beacon.azurelabsimulations.store"
0x00000000007c1948 : 0x0030006400300034 :  !du "40d0a88f392a1e1b4f43c337077.380c1b9a.609af4ba.beacon.azurelabsimulations.store"
0x00000000007c1950 : 0x0066003800380061 :  !du "a88f392a1e1b4f43c337077.380c1b9a.609af4ba.beacon.azurelabsimulations.store"
0x00000000007c1958 : 0x0061003200390033 :  !du "392a1e1b4f43c337077.380c1b9a.609af4ba.beacon.azurelabsimulations.store"
0x00000000007c1960 : 0x0062003100650031 :  !du "1e1b4f43c337077.380c1b9a.609af4ba.beacon.azurelabsimulations.store"
0x00000000007c1968 : 0x0033003400660034 :  !du "4f43c337077.380c1b9a.609af4ba.beacon.azurelabsimulations.store"
0x00000000007c1970 : 0x0037003300330063 :  !du "c337077.380c1b9a.609af4ba.beacon.azurelabsimulations.store"
0x00000000007c1978 : 0x002e003700370030 :  !du "077.380c1b9a.609af4ba.beacon.azurelabsimulations.store"
0x00000000007c1980 : 0x0063003000380033 :  !du "380c1b9a.609af4ba.beacon.azurelabsimulations.store"
0x00000000007c1988 : 0x0061003900620031 :  !du "1b9a.609af4ba.beacon.azurelabsimulations.store"
0x00000000007c1990 : 0x003900300036002e :  !du ".609af4ba.beacon.azurelabsimulations.store"
0x00000000007c1998 : 0x0062003400660061 :  !du "af4ba.beacon.azurelabsimulations.store"
0x00000000007c19a0 : 0x00650062002e0061 :  !du "a.beacon.azurelabsimulations.store"
0x00000000007c1a00 : 0x00300036002e0061 :  !du "a.609af4ba.beacon.azurelabsimulations.store"
0x00000000007c1a08 : 0x0034006600610039 :  !du "9af4ba.beacon.azurelabsimulations.store"
0x00000000007c1a10 : 0x0062002e00610062 :  !du "ba.beacon.azurelabsimulations.store"
0x00000000007c2268 : 0x0061003900300036 :  !du "609af4ba.beacon.azurelabsimulations.store"
0x00000000007c2270 : 0x0061006200340066 :  !du "f4ba.beacon.azurelabsimulations.store"
```

We can now use the Python script **1768.py** from Didier Stevens to extract the full configuration of this DNS Beacon:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/a541e8c6-934e-4a06-af40-e9a8176f571c)

Even though we managed to extract the Beacon configuration, the next step in this research should be to examine all the DNS queries and responses that are lingering in this memory dump. I haven't been able to figure that out yet, but perhaps an interesting topic to revisit later in the future.

# References

- https://hstechdocs.helpsystems.com/manuals/cobaltstrike/current/userguide/content/topics/malleable-c2_dns-beacons.htm
- https://blog.sekoia.io/hunting-and-detecting-cobalt-strike/
- https://github.com/DidierStevens/DidierStevensSuite/blob/master/1768.py
