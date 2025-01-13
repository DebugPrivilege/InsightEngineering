# Summary

This write-up provides hands-on examples to help you get started with analyzing a TTD trace. We'll work with two trace files: one that captures the actual issue and another that functions correctly for comparison. Both trace files are included in the ZIP file `net01.zip`. You can download, extract them, load them into WinDbg, and follow along.

## Root cause

The TTD trace `(net01.run)` captures an issue where the `net use` command fails to map an administrative share from a remote machine to a local drive. Although we have administrative access, the connection could not be established because inbound SMB traffic was blocked on the remote server. Technically, we don’t need a TTD trace since the event log already revealed the issue. But can we still identify the problem using a TTD trace?

![image](https://github.com/user-attachments/assets/5b59687c-99ff-404a-899c-efded99db5a2)


## Analysis

After capturing a TTD trace, you’ll get a `.run` file, which can be loaded into WinDbg for further analysis. In this example, we’re attempting to map an administrative share from a domain controller to a local drive letter. 

![image](https://github.com/user-attachments/assets/400c0c46-d353-4807-9452-27dc33a39215)


Let's load the trace file into WinDbg and get started. This command is something I'd use in Time Travel Debugging to dig into how and when errors were happening in the code being debugged. It looks at all the times the `GetLastError` function was called during the session.

- It filters out the calls where `GetLastError` didn’t return an error (i.e., where the return value was 0).
- For every call that did return an error, it pulls some key info: when the call happened (both in debug time and system time) and the actual error code.

This way, I can quickly pinpoint all the moments when `GetLastError` returned something meaningful, which is super handy for tracing unexpected issues or debugging edge cases. 

```
0:000> dx -g @$s = @$cursession.TTD.Calls("kernelbase!GetLastError").Where(c => c.ReturnValue != 0).Select(c => new { Time = c.TimeStart, Error = c.ReturnValue, SystemTimeStart = c.SystemTimeStart })
====================================================================================
=           = (+) Time    = (+) Error = (+) SystemTimeStart                        =
====================================================================================
= [0x0]     - 17:D42      - 0x57      - Wednesday, January 8, 2025 13:15:50.299    =
= [0x70]    - 1E9:41      - 0x8ca     - Wednesday, January 8, 2025 13:15:50.376    =
= [0x71]    - 1E9:95      - 0x8ca     - Wednesday, January 8, 2025 13:15:50.376    =
= [0x72]    - 1E9:7F2     - 0x8ca     - Wednesday, January 8, 2025 13:15:50.376    =
= [0x73]    - 2C6:4E0     - 0x7f      - Wednesday, January 8, 2025 13:15:50.408    =
= [0x74]    - 2C6:AD3     - 0x7f      - Wednesday, January 8, 2025 13:15:50.408    =
= [0x75]    - 394:117     - 0x7f      - Wednesday, January 8, 2025 13:15:50.439    =
= [0x76]    - 3A8:BA      - 0x2       - Wednesday, January 8, 2025 13:15:50.439    =
= [0x7c]    - 3E3:B5      - 0x3f0     - Wednesday, January 8, 2025 13:15:50.439    =
= [0x7d]    - 448:E       - 0xea      - Wednesday, January 8, 2025 13:15:50.454    =
= [0x7e]    - 465:999     - 0x7a      - Wednesday, January 8, 2025 13:15:50.470    =
= [0x7f]    - 479:16EB    - 0x7a      - Wednesday, January 8, 2025 13:15:50.470    =
= [0x80]    - 48B:E01     - 0x57      - Wednesday, January 8, 2025 13:16:20.470    =
= [0x81]    - 4B8:E46     - 0x57      - Wednesday, January 8, 2025 13:16:20.470    =
= [0x82]    - 4CE:B7      - 0x3f0     - Wednesday, January 8, 2025 13:16:42.439    =
= [0x83]    - 4E6:52      - 0x3e5     - Wednesday, January 8, 2025 13:16:42.439    =
= [0x84]    - 4FC:B7      - 0x3f0     - Wednesday, January 8, 2025 13:16:42.454    =
= [0x85]    - 510:52      - 0x3e5     - Wednesday, January 8, 2025 13:16:42.454    =
= [0x86]    - 512:1878    - 0x3e5     - Wednesday, January 8, 2025 13:16:42.454    =
= [0x87]    - 518:1E6     - 0x3e5     - Wednesday, January 8, 2025 13:16:42.454    =
= [0x88]    - 518:32A     - 0x3e5     - Wednesday, January 8, 2025 13:16:42.454    =
= [0x89]    - 51F:B7      - 0x3f0     - Wednesday, January 8, 2025 13:16:42.454    =
= [0x8a]    - 533:52      - 0x3e5     - Wednesday, January 8, 2025 13:16:42.470    =
= [0x8b]    - 581:4E2     - 0x3e5     - Wednesday, January 8, 2025 13:16:42.533    =
= [0x8c]    - 60D:1DB     - 0x3e5     - Wednesday, January 8, 2025 13:16:42.533    =
= [0x8d]    - 60D:31F     - 0x3e5     - Wednesday, January 8, 2025 13:16:42.533    =
= [0x8e]    - 666:B5      - 0x3f0     - Wednesday, January 8, 2025 13:16:42.658    =
= [0x8f]    - 7F8:18DC    - 0x3f0     - Wednesday, January 8, 2025 13:16:43.1      =
= [0x90]    - 7F8:1E4A    - 0x3f0     - Wednesday, January 8, 2025 13:16:43.1      =
= [0x91]    - 7F8:2031    - 0x3f0     - Wednesday, January 8, 2025 13:16:43.1      =
= [0x92]    - 7F8:231E    - 0x3f0     - Wednesday, January 8, 2025 13:16:43.1      =
= [0x93]    - 7F8:2515    - 0x3f0     - Wednesday, January 8, 2025 13:16:43.1      =
= [0x94]    - 7F9:2A      - 0x3f0     - Wednesday, January 8, 2025 13:16:43.17     =
= [0x95]    - 7F9:218     - 0x3f0     - Wednesday, January 8, 2025 13:16:43.17     =
= [0x96]    - 7F9:40D     - 0x3f0     - Wednesday, January 8, 2025 13:16:43.17     =
= [0x97]    - 7F9:607     - 0x3f0     - Wednesday, January 8, 2025 13:16:43.17     =
= [0x98]    - 7F9:801     - 0x3f0     - Wednesday, January 8, 2025 13:16:43.17     =
= [0x99]    - 7F9:9FB     - 0x3f0     - Wednesday, January 8, 2025 13:16:43.17     =
= [0x9a]    - 7F9:BF5     - 0x3f0     - Wednesday, January 8, 2025 13:16:43.17     =
= [0x9b]    - 7F9:DEF     - 0x3f0     - Wednesday, January 8, 2025 13:16:43.17     =
= [0x9c]    - 7F9:FE9     - 0x3f0     - Wednesday, January 8, 2025 13:16:43.17     =
= [0x9d]    - 7F9:11E3    - 0x3f0     - Wednesday, January 8, 2025 13:16:43.17     =
= [0x9e]    - 7F9:13DD    - 0x3f0     - Wednesday, January 8, 2025 13:16:43.17     =
= [0x9f]    - 7F9:15D7    - 0x3f0     - Wednesday, January 8, 2025 13:16:43.17     =
= [0xa0]    - 7F9:17D1    - 0x3f0     - Wednesday, January 8, 2025 13:16:43.17     =
= [0xa1]    - 7F9:19CB    - 0x3f0     - Wednesday, January 8, 2025 13:16:43.17     =
= [0xa2]    - 7F9:1BC5    - 0x3f0     - Wednesday, January 8, 2025 13:16:43.17     =
= [0xa3]    - 7F9:1DBF    - 0x3f0     - Wednesday, January 8, 2025 13:16:43.17     =
= [0xa4]    - 7F9:1FB9    - 0x3f0     - Wednesday, January 8, 2025 13:16:43.17     =
= [0xa5]    - 7F9:21B3    - 0x3f0     - Wednesday, January 8, 2025 13:16:43.17     =
= [0xa6]    - 7F9:23AD    - 0x3f0     - Wednesday, January 8, 2025 13:16:43.17     =
= [0xa7]    - 7F9:259D    - 0x3f0     - Wednesday, January 8, 2025 13:16:43.17     =
= [0xa8]    - 7FA:74      - 0x3f0     - Wednesday, January 8, 2025 13:16:43.17     =
= [0xa9]    - 7FA:25B     - 0x3f0     - Wednesday, January 8, 2025 13:16:43.17     =
= [0xaa]    - 7FA:442     - 0x3f0     - Wednesday, January 8, 2025 13:16:43.17     =
= [0xab]    - 7FA:6C6     - 0x3f0     - Wednesday, January 8, 2025 13:16:43.17     =
= [0xac]    - 7FA:8AD     - 0x3f0     - Wednesday, January 8, 2025 13:16:43.17     =
= [0xad]    - 7FA:A94     - 0x3f0     - Wednesday, January 8, 2025 13:16:43.17     =
= [0xae]    - 7FA:CC9     - 0x3f0     - Wednesday, January 8, 2025 13:16:43.17     =
= [0xaf]    - 7FA:ED3     - 0x3f0     - Wednesday, January 8, 2025 13:16:43.17     =
= [0xb0]    - 7FA:17D5    - 0x3f0     - Wednesday, January 8, 2025 13:16:43.17     =
= [0xb1]    - 7FA:19D2    - 0x3f0     - Wednesday, January 8, 2025 13:16:43.17     =
= [0xb2]    - 7FA:1BB9    - 0x3f0     - Wednesday, January 8, 2025 13:16:43.17     =
= [0xb3]    - 7FA:1DA0    - 0x3f0     - Wednesday, January 8, 2025 13:16:43.17     =
= [0xb4]    - 7FA:1F87    - 0x3f0     - Wednesday, January 8, 2025 13:16:43.17     =
= [0xb5]    - 7FA:216E    - 0x3f0     - Wednesday, January 8, 2025 13:16:43.17     =
= [0xb6]    - 7FA:2355    - 0x3f0     - Wednesday, January 8, 2025 13:16:43.17     =
= [0xb7]    - 7FA:253C    - 0x3f0     - Wednesday, January 8, 2025 13:16:43.17     =
= [0xb8]    - 7FB:13      - 0x3f0     - Wednesday, January 8, 2025 13:16:43.17     =
= [0xb9]    - 7FB:1FA     - 0x3f0     - Wednesday, January 8, 2025 13:16:43.17     =
= [0xba]    - 7FB:3E1     - 0x3f0     - Wednesday, January 8, 2025 13:16:43.17     =
= [0xbb]    - 7FB:5C8     - 0x3f0     - Wednesday, January 8, 2025 13:16:43.17     =
= [0xbc]    - 7FB:7AF     - 0x3f0     - Wednesday, January 8, 2025 13:16:43.17     =
= [0xbd]    - 7FB:99D     - 0x3f0     - Wednesday, January 8, 2025 13:16:43.17     =
= [0xbe]    - 7FB:B84     - 0x3f0     - Wednesday, January 8, 2025 13:16:43.17     =
= [0xbf]    - 7FB:D6B     - 0x3f0     - Wednesday, January 8, 2025 13:16:43.17     =
= [0xc0]    - 7FB:F59     - 0x3f0     - Wednesday, January 8, 2025 13:16:43.17     =
= [0xc1]    - 7FB:1147    - 0x3f0     - Wednesday, January 8, 2025 13:16:43.17     =
= [0xc2]    - 7FB:1335    - 0x3f0     - Wednesday, January 8, 2025 13:16:43.17     =
= [0xc3]    - 7FB:1523    - 0x3f0     - Wednesday, January 8, 2025 13:16:43.17     =
= [0xc4]    - 7FB:1711    - 0x3f0     - Wednesday, January 8, 2025 13:16:43.17     =
= [0xc5]    - 7FB:18FF    - 0x3f0     - Wednesday, January 8, 2025 13:16:43.17     =
= [0xc6]    - 7FB:1B12    - 0x3f0     - Wednesday, January 8, 2025 13:16:43.17     =
= [0xc7]    - 7FB:1D02    - 0x3f0     - Wednesday, January 8, 2025 13:16:43.17     =
= [0xc8]    - 7FB:1EF0    - 0x3f0     - Wednesday, January 8, 2025 13:16:43.17     =
= [0xc9]    - 7FB:20ED    - 0x3f0     - Wednesday, January 8, 2025 13:16:43.17     =
= [0xca]    - 7FB:22D4    - 0x3f0     - Wednesday, January 8, 2025 13:16:43.17     =
= [0xcb]    - 7FB:24BB    - 0x3f0     - Wednesday, January 8, 2025 13:16:43.17     =
= [0xcc]    - 7FB:26A2    - 0x3f0     - Wednesday, January 8, 2025 13:16:43.17     =
= [0xcd]    - 7FC:180     - 0x3f0     - Wednesday, January 8, 2025 13:16:43.17     =
= [0xce]    - 7FC:36E     - 0x3f0     - Wednesday, January 8, 2025 13:16:43.17     =
= [0xcf]    - 7FC:555     - 0x3f0     - Wednesday, January 8, 2025 13:16:43.17     =
= [0xd0]    - 7FC:D53     - 0x3f0     - Wednesday, January 8, 2025 13:16:43.17     =
= [0xd1]    - 7FC:F41     - 0x3f0     - Wednesday, January 8, 2025 13:16:43.17     =
= [0xd2]    - 7FC:1128    - 0x3f0     - Wednesday, January 8, 2025 13:16:43.17     =
= [0xd3]    - 7FC:130F    - 0x3f0     - Wednesday, January 8, 2025 13:16:43.17     =
= [0xd4]    - 7FC:14F6    - 0x3f0     - Wednesday, January 8, 2025 13:16:43.17     =
= [0xd5]    - 7FC:16DD    - 0x3f0     - Wednesday, January 8, 2025 13:16:43.17     =
= [0xd6]    - 7FC:18C4    - 0x3f0     - Wednesday, January 8, 2025 13:16:43.17     =
= [0xd7]    - 7FC:1AAB    - 0x3f0     - Wednesday, January 8, 2025 13:16:43.17     =
= [...]     -             -           -                                            =
====================================================================================
```

The next thing I’m doing is breaking down the error codes to figure out which one might be relevant to the issue I’m dealing with. You can use the `!error` command to decode the error values and get more details about what they mean.

| Error Code | Decimal Value | Description                                       |
|------------------------|---------------|---------------------------------------------------|
| (Win32) 0x57          | 87            | The parameter is incorrect.                      |
| (NetAPI) 0x8ca        | 2250          | The network connection could not be found.       |
| (Win32) 0x7f          | 127           | The specified procedure could not be found.      |
| (Win32) 0x3f0         | 1008          | An attempt was made to reference a token that does not exist. |
| (Win32) 0x3e5         | 997           | Overlapped I/O operation is in progress.         |


In this scenario, since we’re troubleshooting an SMB issue while trying to map an administrative share from a remote server to a local drive, the error code that seems most relevant is `0x8CA`, which translates to `The network connection could not be found`. This fits well with the problem we’re trying to address.

To check how often `0x8CA` was returned by `GetLastError`, you can use the following command:

```
0:000> dx @$cursession.TTD.Calls("kernelbase!GetLastError").Where(x => x.ReturnValue == 0x8CA).Count()
@$cursession.TTD.Calls("kernelbase!GetLastError").Where(x => x.ReturnValue == 0x8CA).Count() : 0x3
```

This command filters all calls to `GetLastError` and shows only the ones where the return value is `0x8CA`. For each match, it lists the debug time `(Time)` and the system time `(SystemTimeStart)` when the call occurred.

```
0:000> dx -g @$s = @$cursession.TTD.Calls("kernelbase!GetLastError").Where(c => c.ReturnValue == 0x8CA).Select(c => new { Time = c.TimeStart, SystemTimeStart = c.SystemTimeStart })
=======================================================================
=           = (+) Time   = (+) SystemTimeStart                        =
=======================================================================
= [0x70]    - 1E9:41     - Wednesday, January 8, 2025 13:15:50.376    =
= [0x71]    - 1E9:95     - Wednesday, January 8, 2025 13:15:50.376    =
= [0x72]    - 1E9:7F2    - Wednesday, January 8, 2025 13:15:50.376    =
=======================================================================
```

Let’s jump to the first TTD position where the `0x8CA` error code appears.

```
0:000> !tt 1E9:41
Setting position: 1E9:41
(348.430): Break instruction exception - code 80000003 (first/second chance not available)
Time Travel Position: 1E9:41
KERNELBASE!GetLastError:
00007ffb`c3124f10 65488b042530000000 mov   rax,qword ptr gs:[30h] gs:00000000`00000030=0000000000000000
```

The root cause is likely related to a failure in establishing a network connection using `WNetAddConnection2W`, which is part of the Windows networking API.

```
0:000> k
 # Child-SP          RetAddr               Call Site
00 00000011`a994e1f8 00007ffb`c2998fa3     KERNELBASE!GetLastError
01 00000011`a994e200 00007ffb`c299d02f     WINSTA!_DbgPrintMessage+0x53
02 00000011`a994e4c0 00007ffb`c299b8af     WINSTA!DllMain+0x3b
03 00000011`a994e500 00007ffb`c5549a1d     WINSTA!__DllMainCRTStartup+0xfb
04 00000011`a994e670 00007ffb`c559c1e7     ntdll!LdrpCallInitRoutine+0x61
05 00000011`a994e6e0 00007ffb`c559bf7a     ntdll!LdrpInitializeNode+0x1d3
06 00000011`a994e830 00007ffb`c559c000     ntdll!LdrpInitializeGraphRecurse+0x42
07 00000011`a994e870 00007ffb`c556d937     ntdll!LdrpInitializeGraphRecurse+0xc8
08 00000011`a994e8b0 00007ffb`c554fbae     ntdll!LdrpPrepareModuleForExecution+0xbf
09 00000011`a994e8f0 00007ffb`c55473e4     ntdll!LdrpLoadDllInternal+0x19a
0a 00000011`a994e970 00007ffb`c5546af4     ntdll!LdrpLoadDll+0xa8
0b 00000011`a994eb20 00007ffb`c311ae12     ntdll!LdrLoadDll+0xe4
0c 00000011`a994ec10 00007ffb`b45deaa7     KERNELBASE!LoadLibraryExW+0x162
0d 00000011`a994ec80 00007ffb`b45de3e7     MPR!MprLevel2Init+0x17f
0e 00000011`a994ecc0 00007ffb`b45d6ff8     MPR!CRoutedOperation::GetResult+0x47
0f 00000011`a994ed40 00007ffb`b45de70d     MPR!CUseConnection::GetResult+0x1f8
10 00000011`a994f280 00007ffb`b45de787     MPR!CMprOperation::Perform+0x4d
11 00000011`a994f2c0 00007ffb`b45d9b87     MPR!CRoutedOperation::Perform+0x2f
12 00000011`a994f2f0 00007ffb`b45d9039     MPR!WNetUseConnectionW+0xc7
13 00000011`a994f460 00007ff6`8fd361c5     MPR!WNetAddConnection2W+0x29 --> WNetUseConnection establishes a connection to a network resource 
14 00000011`a994f4b0 00007ff6`8fd3385a     net!use_add+0x5bd
15 00000011`a994f610 00007ff6`8fd31678     net!xx_parser+0x227a
16 00000011`a994f730 00007ff6`8fd313cb     net!xx_parser+0x98
17 00000011`a994f850 00007ff6`8fd3284d     net!main+0x3bb
18 00000011`a994fb10 00007ffb`c4ba7034     net!__mainCRTStartup+0x14d
19 00000011`a994fb50 00007ffb`c5582651     KERNEL32!BaseThreadInitThunk+0x14
1a 00000011`a994fb80 00000000`00000000     ntdll!RtlUserThreadStart+0x21
```

The `MPR!WNetAddConnection2W` function is particularly relevant to us. We can include it in our timeline view to identify the positions where this function was called.

![image](https://github.com/user-attachments/assets/b04f4a98-d4b9-4540-a606-fdf882e60ca7)


This output provides details about a specific call to the `MPR!WNetAddConnection2W` function during a Time Travel Debugging (TTD) session. 

```
0:000> dx @$cursession.TTD.Calls("MPR!WNetAddConnection2W")[0x0]
@$cursession.TTD.Calls("MPR!WNetAddConnection2W")[0x0]                
    EventType        : 0x0
    ThreadId         : 0x430
    UniqueThreadId   : 0x2
    TimeStart        : EE:8F [Time Travel]
    TimeEnd          : 2385:FDC [Time Travel]
    Function         : MPR!WNetAddConnection2W
    FunctionAddress  : 0x7ffbb45d9010
    ReturnAddress    : 0x7ff68fd361c5
    ReturnValue      : 0x35
    Parameters      
    SystemTimeStart  : Wednesday, January 8, 2025 13:15:50.330
    SystemTimeEnd    : Wednesday, January 8, 2025 13:16:54.830
```

The `ReturnValue` holds an error code that we can interpret.

```
0:000> !error 0x35
Error code: (Win32) 0x35 (53) - The network path was not found.
```

This call stack provides insight into the sequence of function calls leading up to the current execution point.

```
0:000> k
 # Child-SP          RetAddr               Call Site
00 00000011`a994f4a8 00007ff6`8fd361c5     MPR!WNetAddConnection2W
01 00000011`a994f4b0 00007ff6`8fd3385a     net!use_add+0x5bd
02 00000011`a994f610 00007ff6`8fd31678     net!xx_parser+0x227a
03 00000011`a994f730 00007ff6`8fd313cb     net!xx_parser+0x98
04 00000011`a994f850 00007ff6`8fd3284d     net!main+0x3bb
05 00000011`a994fb10 00007ffb`c4ba7034     net!__mainCRTStartup+0x14d
06 00000011`a994fb50 00007ffb`c5582651     KERNEL32!BaseThreadInitThunk+0x14
07 00000011`a994fb80 00000000`00000000     ntdll!RtlUserThreadStart+0x21
```

This output shows the state of the CPU at the exact point where the `MPR!WNetAddConnection2W` function is executing. The register values represent parameters or data the function is using, and the instruction being executed.

```
0:000> r
rax=0000000000000000 rbx=0000000000000001 rcx=00000011a994f540
rdx=0000016926d6460e rsi=0000016926d645d0 rdi=0000016926d645d0
rip=00007ffbb45d9010 rsp=00000011a994f4a8 rbp=00000011a994f5b0
 r8=0000016926d645f2  r9=0000000000000001 r10=0000000000000000
r11=00000011a994ee70 r12=0000016926d645f2 r13=0000016926d645d6
r14=0000016926d6460e r15=0000016926d6460e
iopl=0         nv up ei pl zr na po nc
cs=0033  ss=002b  ds=002b  es=002b  fs=0053  gs=002b             efl=00000246
MPR!WNetAddConnection2W:
00007ffb`b45d9010 4c8bdc          mov     r11,rsp
```

The command is displaying key values from the stack or memory that are being used by the function `MPR!WNetAddConnection2W`. 

```
DWORD WNetAddConnection2W(
  [in] LPNETRESOURCEW lpNetResource,
  [in] LPCWSTR        lpPassword,
  [in] LPCWSTR        lpUserName,
  [in] DWORD          dwFlags
);
```

- Username `("CONTOSO\James")` and password `("Passw0rd!")` are being passed as credentials.
- Network path `("\\DC\C$")` is the target resource for the network connection.

```
0:000> !pde.dpx -du 5
Start memory scan  : 0x00000011a994f4a8 ($csp)
End memory scan    : 0x00000011a9950000 (User Stack Base)
Values to scan     : 5 (0x5)

               rdx : 0x0000016926d6460e :  !du "Passw0rd!"
                r8 : 0x0000016926d645f2 :  !du "CONTOSO\James"
               r12 : 0x0000016926d645f2 :  !du "CONTOSO\James"
               r13 : 0x0000016926d645d6 :  !du "\\DC\C$"
               r14 : 0x0000016926d6460e :  !du "Passw0rd!"
```

## Analysis (2)

This trace file `(net02.run)` doesn’t contain any issues. It’s sa straightforward net use operation to map an administrative share from a remote server to a local drive. Let's load the trace file into WinDbg and get started and run the exact same commands that we did previously.

```
0:000> dx -g @$s = @$cursession.TTD.Calls("kernelbase!GetLastError").Where(c => c.ReturnValue != 0).Select(c => new { Time = c.TimeStart, Error = c.ReturnValue, SystemTimeStart = c.SystemTimeStart })
====================================================================================
=           = (+) Time    = (+) Error = (+) SystemTimeStart                        =
====================================================================================
= [0x0]     - 17:D4A      - 0x57      - Wednesday, January 8, 2025 13:21:35.674    =
= [0x70]    - 1FD:41      - 0x8ca     - Wednesday, January 8, 2025 13:21:35.720    =
= [0x71]    - 1FD:95      - 0x8ca     - Wednesday, January 8, 2025 13:21:35.720    =
= [0x72]    - 1FD:7F2     - 0x8ca     - Wednesday, January 8, 2025 13:21:35.720    =
= [0x73]    - 2D8:387     - 0x7f      - Wednesday, January 8, 2025 13:21:35.736    =
= [0x74]    - 2D8:97A     - 0x7f      - Wednesday, January 8, 2025 13:21:35.736    =
= [0x75]    - 3A9:117     - 0x7f      - Wednesday, January 8, 2025 13:21:35.767    =
= [0x76]    - 3BD:BA      - 0x2       - Wednesday, January 8, 2025 13:21:35.767    =
= [0x7c]    - 3F7:B5      - 0x3f0     - Wednesday, January 8, 2025 13:21:35.783    =
= [0x7d]    - 45D:984     - 0xea      - Wednesday, January 8, 2025 13:21:35.783    =
= [0x7e]    - 47A:827     - 0x7a      - Wednesday, January 8, 2025 13:21:35.814    =
= [0x7f]    - 490:169C    - 0x7a      - Wednesday, January 8, 2025 13:21:35.814    =
= [0x80]    - 4AA:B7      - 0x3f0     - Wednesday, January 8, 2025 13:22:00.332    =
= [0x81]    - 4C0:52      - 0x3e5     - Wednesday, January 8, 2025 13:22:00.332    =
= [0x82]    - 4D2:B7      - 0x3f0     - Wednesday, January 8, 2025 13:22:00.348    =
= [0x83]    - 4E8:52      - 0x3e5     - Wednesday, January 8, 2025 13:22:00.348    =
= [0x84]    - 4F3:B7      - 0x3f0     - Wednesday, January 8, 2025 13:22:00.348    =
= [0x85]    - 4F5:52      - 0x3e5     - Wednesday, January 8, 2025 13:22:00.348    =
= [0x86]    - 4FB:B7      - 0x3f0     - Wednesday, January 8, 2025 13:22:00.364    =
= [0x87]    - 4FD:52      - 0x3e5     - Wednesday, January 8, 2025 13:22:00.364    =
= [0x88]    - 4FF:1BB2    - 0x3e5     - Wednesday, January 8, 2025 13:22:00.364    =
= [0x89]    - 50C:B5      - 0x3f0     - Wednesday, January 8, 2025 13:22:00.364    =
= [0x8a]    - 514:B2      - 0x7a      - Wednesday, January 8, 2025 13:22:00.364    =
= [0x8b]    - 53A:B5      - 0x3f0     - Wednesday, January 8, 2025 13:22:00.379    =
= [0x8c]    - 542:B2      - 0x7a      - Wednesday, January 8, 2025 13:22:00.379    =
= [0x8d]    - 576:83      - 0x7e      - Wednesday, January 8, 2025 13:22:00.387    =
= [0x8e]    - 576:222     - 0x7e      - Wednesday, January 8, 2025 13:22:00.387    =
= [0x8f]    - 598:117C    - 0x7e      - Wednesday, January 8, 2025 13:22:00.387    =
= [0x90]    - 598:11D0    - 0x7e      - Wednesday, January 8, 2025 13:22:00.387    =
= [0x91]    - 598:15D9    - 0x7e      - Wednesday, January 8, 2025 13:22:00.387    =
= [0x92]    - 598:1DE1    - 0x7e      - Wednesday, January 8, 2025 13:22:00.387    =
= [0x93]    - 5A3:30C     - 0x7e      - Wednesday, January 8, 2025 13:22:00.387    =
= [0x94]    - 5A3:707     - 0x7e      - Wednesday, January 8, 2025 13:22:00.387    =
= [0x95]    - 5A3:B02     - 0x7e      - Wednesday, January 8, 2025 13:22:00.387    =
= [0x96]    - 5A3:F01     - 0x7e      - Wednesday, January 8, 2025 13:22:00.387    =
====================================================================================
```

This command filters all calls to `GetLastError` and shows only the ones where the return value is `0x8CA`. For each match, it lists the debug time `(Time)` and the system time `(SystemTimeStart)` when the call occurred.

```
0:000> dx -g @$s = @$cursession.TTD.Calls("kernelbase!GetLastError").Where(c => c.ReturnValue == 0x8CA).Select(c => new { Time = c.TimeStart, SystemTimeStart = c.SystemTimeStart })
=======================================================================
=           = (+) Time   = (+) SystemTimeStart                        =
=======================================================================
= [0x70]    - 1FD:41     - Wednesday, January 8, 2025 13:21:35.720    =
= [0x71]    - 1FD:95     - Wednesday, January 8, 2025 13:21:35.720    =
= [0x72]    - 1FD:7F2    - Wednesday, January 8, 2025 13:21:35.720    =
=======================================================================
```

Let’s jump to the first TTD position where the `0x8CA` error code appears.

```
0:000> !tt 1FD:41
Setting position: 1FD:41
(3ec.1724): Break instruction exception - code 80000003 (first/second chance not available)
Time Travel Position: 1FD:41
KERNELBASE!GetLastError:
00007ffb`c3124f10 65488b042530000000 mov   rax,qword ptr gs:[30h] gs:00000000`00000030=0000000000000000
```

Running the `k` command to display the call stack reveals that it’s identical to the call stack in the previous trace file.

```
0:000> k
 # Child-SP          RetAddr               Call Site
00 00000066`e5a7de68 00007ffb`c2998fa3     KERNELBASE!GetLastError
01 00000066`e5a7de70 00007ffb`c299d02f     WINSTA!_DbgPrintMessage+0x53
02 00000066`e5a7e130 00007ffb`c299b8af     WINSTA!DllMain+0x3b
03 00000066`e5a7e170 00007ffb`c5549a1d     WINSTA!__DllMainCRTStartup+0xfb
04 00000066`e5a7e2e0 00007ffb`c559c1e7     ntdll!LdrpCallInitRoutine+0x61
05 00000066`e5a7e350 00007ffb`c559bf7a     ntdll!LdrpInitializeNode+0x1d3
06 00000066`e5a7e4a0 00007ffb`c559c000     ntdll!LdrpInitializeGraphRecurse+0x42
07 00000066`e5a7e4e0 00007ffb`c556d937     ntdll!LdrpInitializeGraphRecurse+0xc8
08 00000066`e5a7e520 00007ffb`c554fbae     ntdll!LdrpPrepareModuleForExecution+0xbf
09 00000066`e5a7e560 00007ffb`c55473e4     ntdll!LdrpLoadDllInternal+0x19a
0a 00000066`e5a7e5e0 00007ffb`c5546af4     ntdll!LdrpLoadDll+0xa8
0b 00000066`e5a7e790 00007ffb`c311ae12     ntdll!LdrLoadDll+0xe4
0c 00000066`e5a7e880 00007ffb`b45deaa7     KERNELBASE!LoadLibraryExW+0x162
0d 00000066`e5a7e8f0 00007ffb`b45de3e7     MPR!MprLevel2Init+0x17f
0e 00000066`e5a7e930 00007ffb`b45d6ff8     MPR!CRoutedOperation::GetResult+0x47
0f 00000066`e5a7e9b0 00007ffb`b45de70d     MPR!CUseConnection::GetResult+0x1f8
10 00000066`e5a7eef0 00007ffb`b45de787     MPR!CMprOperation::Perform+0x4d
11 00000066`e5a7ef30 00007ffb`b45d9b87     MPR!CRoutedOperation::Perform+0x2f
12 00000066`e5a7ef60 00007ffb`b45d9039     MPR!WNetUseConnectionW+0xc7
13 00000066`e5a7f0d0 00007ff6`8fd361c5     MPR!WNetAddConnection2W+0x29
14 00000066`e5a7f120 00007ff6`8fd3385a     net!use_add+0x5bd
15 00000066`e5a7f280 00007ff6`8fd31678     net!xx_parser+0x227a
16 00000066`e5a7f3a0 00007ff6`8fd313cb     net!xx_parser+0x98
17 00000066`e5a7f4c0 00007ff6`8fd3284d     net!main+0x3bb
18 00000066`e5a7f780 00007ffb`c4ba7034     net!__mainCRTStartup+0x14d
19 00000066`e5a7f7c0 00007ffb`c5582651     KERNEL32!BaseThreadInitThunk+0x14
1a 00000066`e5a7f7f0 00000000`00000000     ntdll!RtlUserThreadStart+0x21
```

Let’s include the `MPR!WNetAddConnection2W` function in the timeline view and check what insights it provides. The first thing we notice is that the `ReturnValue` is no longer `0x35`. Instead, it’s `0x0`, meaning `The operation completed successfully.`

![image](https://github.com/user-attachments/assets/250c98a0-7f2d-473c-8751-8f2424b99d5f)


