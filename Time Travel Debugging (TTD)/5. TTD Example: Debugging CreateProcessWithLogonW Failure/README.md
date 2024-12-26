# Summary

This is a simple example of using Time Travel Debugging to investigate an issue involving the `CreateProcessWithLogonW` API. `CreateProcessWithLogonW` is a Windows API used to create a new process under the context of a specified user account. It allows you to run a process with alternate credentials, typically for tasks requiring elevated or different privileges, without the user being logged in interactively.

## Sample Analysis

An exception is an event that disrupts the normal flow of a program when an error or unexpected condition occurs. It signals that something went wrong, such as invalid input, a missing resource, or a runtime issue. 

```
0:000> dx -g @$curprocess.TTD.Events.Where(t => t.Type == "Exception")
=============================================================================================================================================================
=                                                              = (+) Type     = (+) Position = (+) Exception                                                =
=============================================================================================================================================================
= [0x0] : Exception 0x000006BA of type Software at PC: 0X7F... - Exception    - 2CD:0        - Exception 0x000006BA of type Software at PC: 0X7FFAD2421020  =
=============================================================================================================================================================
```

The output indicates a single exception in our trace file. Let’s navigate to that position.

```
0:000> !tt 2CD:0
Setting position: 2CD:0
(234c.2154): Break instruction exception - code 80000003 (first/second chance not available)
Time Travel Position: 2CD:0
ntdll!RtlRaiseException:
00007ffa`d2421020 4055            push    rbp
```

Let's display the call stack at this position, which reveals the sequence of function calls leading to an exception in a process attempting to use the `CreateProcessWithLogonW.` Seeing `ntdll!RtlRaiseException` in a call stack indicates that an exception was raised during program execution, either due to an error (like invalid parameters or access violations) or intentionally by the program to signal an exceptional condition.

```
0:000> k
 # Child-SP          RetAddr               Call Site
00 000000cb`ddafe8d8 00007ffa`d0184f99     ntdll!RtlRaiseException
01 000000cb`ddafe8e0 00007ffa`d0787740     KERNELBASE!RaiseException+0x69
02 000000cb`ddafe9c0 00007ffa`d0787704     RPCRT4!RpcpRaiseException+0x34
03 000000cb`ddafe9f0 00007ffa`d0810229     RPCRT4!RpcRaiseException+0x14
04 000000cb`ddafea20 00007ffa`d0813840     RPCRT4!NdrpClientCall3+0x849
05 000000cb`ddafed90 00007ffa`d1155454     RPCRT4!NdrClientCall3+0xf0
06 000000cb`ddaff120 00007ffa`d1154a39     ADVAPI32!c_SeclCreateProcessWithLogonW+0xb4
07 000000cb`ddaff180 00007ffa`d1197e2e     ADVAPI32!CreateProcessWithLogonCommonW+0x54d
08 000000cb`ddaff820 00007ff6`e368128c     ADVAPI32!CreateProcessWithLogonW+0x6e
09 000000cb`ddaff890 00007ff6`e3681448     ConsoleApplication1!CreateProcessWithCredentials+0xcc [C:\Users\Admin\source\repos\ConsoleApplication1\ConsoleApplication1\ConsoleApplication1.cpp @ 14] 
0a 000000cb`ddaff9b0 00007ff6`e3681f50     ConsoleApplication1!main+0xb8 [C:\Users\Admin\source\repos\ConsoleApplication1\ConsoleApplication1\ConsoleApplication1.cpp @ 46] 
0b (Inline Function) --------`--------     ConsoleApplication1!invoke_main+0x22 [D:\a\_work\1\s\src\vctools\crt\vcstartup\src\startup\exe_common.inl @ 78] 
0c 000000cb`ddaffae0 00007ffa`d1217034     ConsoleApplication1!__scrt_common_main_seh+0x10c [D:\a\_work\1\s\src\vctools\crt\vcstartup\src\startup\exe_common.inl @ 288] 
0d 000000cb`ddaffb20 00007ffa`d2422651     KERNEL32!BaseThreadInitThunk+0x14
0e 000000cb`ddaffb50 00000000`00000000     ntdll!RtlUserThreadStart+0x21
```

This example shows the state of the CPU registers and the instruction pointer (rip) at the moment an exception is being raised by the `ntdll!RtlRaiseException` function.

```
0:000> r
rax=00007ffad0184f30 rbx=00000000000006ba rcx=000000cbddafe900
rdx=0000000000000001 rsi=0000000000000000 rdi=00007ffad11b8228
rip=00007ffad2421020 rsp=000000cbddafe8d8 rbp=0000000000000000
 r8=0000000000000000  r9=0000000000000000 r10=000001cf8b970000
r11=000000cbddafe840 r12=000001cf8b991200 r13=00000000000006ba
r14=0000000000000000 r15=000000cbddafeba0
iopl=0         nv up ei pl zr na po nc
cs=0033  ss=002b  ds=002b  es=002b  fs=0053  gs=002b             efl=00000246
ntdll!RtlRaiseException:
00007ffa`d2421020 4055            push    rbp
```

The value `0x6ba` in both `rbx` and `r13` corresponds to the error code `ERROR_RPC_SERVER_UNAVAILABLE`. You can confirm this by running the following command:

```
0:000> !error 06ba
Error code: (Win32) 0x6ba (1722) - The RPC server is unavailable.
```

Let’s take a step back to understand what RPC is used for. It is a communication protocol that allows programs to execute code or procedures on a remote server or process as if they were local. `CreateProcessWithLogonW` uses RPC internally to communicate with the Secondary Logon Service (seclogon), which is responsible for securely managing credentials and creating the process.

We know that `CreateProcessWithLogonW` failed in this case, but the reason is still unclear. To investigate further, we can query the `GetLastError` API, which retrieves the last error code set by the system for the current thread and provides insight into why the previous API call failed.

```
0:000> dx -g @$s = @$cursession.TTD.Calls("kernelbase!GetLastError").Where(c => c.ReturnValue != 0).Select(c => new { Time = c.TimeStart, Error = c.ReturnValue, SystemTimeStart = c.SystemTimeStart })
=====================================================================================
=           = (+) Time    = (+) Error = (+) SystemTimeStart                         =
=====================================================================================
= [0x0]     - 15:D05      - 0x57      - Thursday, December 26, 2024 09:44:39.806    =
= [0x1]     - 15:13D1     - 0x57      - Thursday, December 26, 2024 09:44:39.806    =
= [0x2]     - 15:179C     - 0x57      - Thursday, December 26, 2024 09:44:39.806    =
= [0x3]     - 1B:10A      - 0x57      - Thursday, December 26, 2024 09:44:39.806    =
= [0x4]     - 1B:1C3      - 0x57      - Thursday, December 26, 2024 09:44:39.806    =
= [0x9b]    - 259:B5      - 0x3f0     - Thursday, December 26, 2024 09:44:39.931    =
= [0x9c]    - 322:10FD    - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0x9d]    - 374:C8E     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0x9e]    - 374:F40     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0x9f]    - 376:27A     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xa0]    - 378:27A     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xa1]    - 37A:27A     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xa2]    - 37C:27A     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xa3]    - 37E:27A     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xa4]    - 380:27A     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xa5]    - 382:27A     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xa6]    - 384:27A     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xa7]    - 386:27A     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xa8]    - 388:27A     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xa9]    - 38A:27A     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xaa]    - 38C:27A     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xab]    - 38E:27A     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xac]    - 390:27A     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xad]    - 392:27A     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xae]    - 394:27A     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xaf]    - 396:27A     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xb0]    - 398:27A     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xb1]    - 39A:27A     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xb2]    - 39C:27A     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xb3]    - 39E:27A     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xb4]    - 3A0:27A     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xb5]    - 3A2:27A     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xb6]    - 3A4:27A     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xb7]    - 3A6:27A     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xb8]    - 3A8:27A     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xb9]    - 3AA:27A     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xba]    - 3AC:27A     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xbb]    - 3AE:27A     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xbc]    - 3B0:27A     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xbd]    - 3B2:27A     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xbe]    - 3B4:27A     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xbf]    - 3B6:27A     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xc0]    - 3B8:27A     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xc1]    - 3BA:27A     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xc2]    - 3BC:27A     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xc3]    - 3BE:27A     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xc4]    - 3C0:27A     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xc5]    - 3C2:27A     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xc6]    - 3C4:F9      - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xc7]    - 3C4:330     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xc8]    - 3C4:5E2     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xc9]    - 3C6:27A     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xca]    - 3C8:27A     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xcb]    - 3CA:27A     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xcc]    - 3CC:27A     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xcd]    - 3CE:27A     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xce]    - 3D0:27A     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xcf]    - 3D2:27A     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xd0]    - 3D4:27A     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xd1]    - 3D6:27A     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xd2]    - 3D8:27A     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xd3]    - 3DA:F8      - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xd4]    - 3DA:362     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xd5]    - 3DA:614     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xd6]    - 3DC:27A     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xd7]    - 3DE:27A     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xd8]    - 3E0:27A     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xd9]    - 3E2:27A     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xda]    - 3E4:27A     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xdb]    - 3E6:27A     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xdc]    - 3E8:27A     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xdd]    - 3EA:27A     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xde]    - 3EC:27A     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xdf]    - 3EE:27A     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xe0]    - 3F0:27A     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xe1]    - 3F2:27A     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xe2]    - 3F4:27A     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xe3]    - 3F6:27A     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xe4]    - 3F8:27A     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xe5]    - 3FA:27A     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xe6]    - 3FC:F9      - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xe7]    - 3FC:32A     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xe8]    - 3FC:80B     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xe9]    - 3FC:92F     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xea]    - 3FC:A02     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xeb]    - 3FC:256A    - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xec]    - 3FC:2647    - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xed]    - 3FC:26A2    - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xee]    - 3FC:26FD    - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xef]    - 3FD:7D      - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xf0]    - 3FD:831     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xf1]    - 3FD:955     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xf2]    - 3FD:A28     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xf3]    - 3FE:CE      - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xf4]    - 3FE:4D2     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xf5]    - 3FE:52D     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xf6]    - 3FE:588     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xf7]    - 3FE:6DD     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xf8]    - 3FE:EAA     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [0xf9]    - 3FE:FCE     - 0x52e     - Thursday, December 26, 2024 09:44:40.72     =
= [...]     -             -           -                                             =
=====================================================================================
```

From the output, I usually focus on identifying the most common **error code** in HEX and begin my investigation there. In this example, it happens to be `0x52e`, which is `1326` in decimal numbers. In this example, the issue arises from using incorrect credentials when attempting to call `CreateProcessWithLogonW`.

```
0:000> !error 0x52e
Error code: (Win32) 0x52e (1326) - The user name or password is incorrect.
```

Finally, here’s a handy trick in TTD to query the counts of each error code observed in the trace file.

```
0:000> dx -g @$cursession.TTD.Calls("kernelbase!GetLastError").Where( x=> x.ReturnValue != 0).GroupBy(x => x.ReturnValue).Select(x => new { ErrorNumber = x.First().ReturnValue, ErrorCount = x.Count()}).OrderByDescending(p => p.ErrorCount)
=============================================
=            = (+) ErrorNumber = ErrorCount =
=============================================
= [0x52e]    - 0x52e           - 0x14e      =
= [0x7e]     - 0x7e            - 0x8        =
= [0x57]     - 0x57            - 0x5        =
= [0x3f0]    - 0x3f0           - 0x1        =
=============================================
```


If you prefer decimal numbers instead of HEX, you can use this command to display the output in decimal format.

```
0:000> dx -g @$cursession.TTD.Calls("kernelbase!GetLastError").Where( x=> x.ReturnValue != 0).GroupBy(x => x.ReturnValue).Select(x => new { ErrorNumber = x.First().ReturnValue, ErrorCount = x.Count()}).OrderByDescending(p => p.ErrorCount),d
============================================
=           = (+) ErrorNumber = ErrorCount =
============================================
= [1326]    - 1326            - 334        =
= [126]     - 126             - 8          =
= [87]      - 87              - 5          =
= [1008]    - 1008            - 1          =
============================================
```
