# What is an Access Violation?

An Access Violation is an error that occurs when a program attempts an operation on a memory location in a way that conflicts with the memory's access rights. This could happen if the program tries to access data through a pointer that is either null or uninitialized, uses an uninitialized structure or variable, or attempts to execute code from a region of memory that is designated as non-executable.

To uncover the cause of an Access Violation (AV), we must first distinguish whether it's a read or write violation. We do this by examining the problematic address, and then we analyze how the address ended up in a state that caused the AV, whether in a read or write scenario.

When an Access Violation occurs, an exception is raised, known to its symbolic name, **STATUS_ACCESS_VIOLATION**. This translates to the hexadecimal value **0xc0000005** or the decimal value **-1073741819**, and it's the code we look for to confirm an Access Violation.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/a40bac22-d375-4ddf-85a8-5cac9efacfaf)

# Code Sample (1) - Access Violation

The root cause of the Access Violation in this code is the uninitialized **`STARTUPINFO`** structure **`si`**, which is declared but not assigned any values, leading to undefined contents. When passed to the **CreateProcess** function, these undefined contents may be interpreted as invalid memory addresses or other unexpected data, leading to unpredictable behavior, including an Access Violation.

```c
#include <windows.h>
#include <iostream>
#include <ntstatus.h>

int main()
{
    // Declare variables for the process information
    STARTUPINFO si; // <-- This structure is uninitialized
    PROCESS_INFORMATION pi;

    // Set the size of the structures
    si.cb = sizeof(si);
    ZeroMemory(&pi, sizeof(pi));

    // Command line arguments for the process
    wchar_t commandLine[] = L"cmd.exe /k whoami /all";

    // Set the CREATE_NEW_CONSOLE
    DWORD dwCreationFlags = CREATE_NEW_CONSOLE;

    // Create the process
    if (!CreateProcess(
        L"C:\\Windows\\System32\\cmd.exe",  // Path to the executable
        commandLine,                        // Command-line arguments
        NULL,                               // Process handle not inheritable
        NULL,                               // Thread handle not inheritable
        FALSE,                              // Set handle inheritance to FALSE
        dwCreationFlags,                    // Creation flags
        NULL,                               // Use parent's environment block
        NULL,                               // Use parent's starting directory
        &si,                                // Pointer to STARTUPINFO structure (uninitialized)
        &pi)                                // Pointer to PROCESS_INFORMATION structure
        ) {
        std::cout << "CreateProcess failed. Error: " << GetLastError() << std::endl;
        return 1;
    }

    // Wait for the process to complete
    WaitForSingleObject(pi.hProcess, INFINITE);

    // Close the handles
    CloseHandle(pi.hProcess);
    CloseHandle(pi.hThread);

    return 0;
}
```

Begin by compiling and running this code. We should then notice that the program crashes and does not work as expected. To confirm this, open **Event Viewer**, navigate to the **Application** logs, and search for Event ID **1000**. Here, we can also verify that our program has indeed crashed:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/c2cdf8bb-2e11-42af-894e-9986f59f3362)

# WinDbg Walk Through (1) - Analyzing Memory Dump

Start with loading the user mode dump of the process in WinDbg:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/9d11654b-4cbb-4960-9e7a-7d9b579a9f7b)


Let's start with a simple **`!analyze -v`**

```
0:000> !analyze -v
*******************************************************************************
*                                                                             *
*                        Exception Analysis                                   *
*                                                                             *
*******************************************************************************

*** WARNING: Check Image - Checksum mismatch - Dump: 0x21bad9, File: 0x22051c - C:\Symbols\ntdll.dll\EEE69EC7214000\ntdll.dll

KEY_VALUES_STRING: 1

    Key  : AV.Fault
    Value: Read

    Key  : Analysis.CPU.mSec
    Value: 1343

    Key  : Analysis.Elapsed.mSec
    Value: 1579

    Key  : Analysis.IO.Other.Mb
    Value: 0

    Key  : Analysis.IO.Read.Mb
    Value: 1

    Key  : Analysis.IO.Write.Mb
    Value: 0

    Key  : Analysis.Init.CPU.mSec
    Value: 6484

    Key  : Analysis.Init.Elapsed.mSec
    Value: 683429

    Key  : Analysis.Memory.CommitPeak.Mb
    Value: 176

    Key  : Failure.Bucket
    Value: INVALID_POINTER_READ_c0000005_AccessV.exe!main

    Key  : Failure.Hash
    Value: {06a63211-3ee0-9330-318a-a8218924c64e}

    Key  : Timeline.OS.Boot.DeltaSec
    Value: 14320

    Key  : Timeline.Process.Start.DeltaSec
    Value: 1

    Key  : WER.OS.Branch
    Value: ni_release

    Key  : WER.OS.Version
    Value: 10.0.22621.1


FILE_IN_CAB:  AccessV.exe_230824_053605.dmp

COMMENT:  
*** "c:\Users\User\Desktop\procdump64.exe" -accepteula -ma -j "C:\Users\User\Desktop\Minidumps" 8956 640 000001802BCF0000
*** Just-In-Time debugger. PID: 8956 Event Handle: 640 JIT Context: .jdinfo 0x1802bcf0000

NTGLOBALFLAG:  0

APPLICATION_VERIFIER_FLAGS:  0

CONTEXT:  (.ecxr)
rax=0000000000000000 rbx=0000000000000000 rcx=0000003cda8fdf60
rdx=00005db157c7b535 rsi=0000003cda70c000 rdi=0000003cda8fe8b0
rip=00007ff8329aa826 rsp=0000003cda8fde68 rbp=0000003cda8fdf70
 r8=0000000000000000  r9=0000003cda8ffe90 r10=0000000000000000
r11=0000003cda8fe270 r12=0000000000000002 r13=00007ff6b8983310
r14=0000000000000000 r15=0000000000000000
iopl=0         nv up ei pl zr ac po nc
cs=0033  ss=002b  ds=002b  es=002b  fs=0053  gs=002b             efl=00010256
ntdll!RtlInitUnicodeStringEx+0x16:
00007ff8`329aa826 6644390442      cmp     word ptr [rdx+rax*2],r8w ds:00005db1`57c7b535=????
Resetting default scope

EXCEPTION_RECORD:  (.exr -1)
ExceptionAddress: 00007ff8329aa826 (ntdll!RtlInitUnicodeStringEx+0x0000000000000016)
   ExceptionCode: c0000005 (Access violation)
  ExceptionFlags: 00000000
NumberParameters: 2
   Parameter[0]: 0000000000000000
   Parameter[1]: 00005db157c7b535
Attempt to read from address 00005db157c7b535

PROCESS_NAME:  AccessV.exe

READ_ADDRESS:  00005db157c7b535 

ERROR_CODE: (NTSTATUS) 0xc0000005 - The instruction at 0x%p referenced memory at 0x%p. The memory could not be %s.

EXCEPTION_CODE_STR:  c0000005

EXCEPTION_PARAMETER1:  0000000000000000

EXCEPTION_PARAMETER2:  00005db157c7b535

STACK_TEXT:  
0000003c`da8fde68 00007ff8`2ffc31f6     : 00000000`00000000 00007ff8`329aca34 00000000`00000003 00000000`00000040 : ntdll!RtlInitUnicodeStringEx+0x16
0000003c`da8fde70 00007ff8`2ff909aa     : 0000003c`da8fe9c8 0000003c`da8febc0 0000003c`da8fe6c8 0000003c`da8fe704 : KERNELBASE!BasepCreateProcessParameters+0x142
0000003c`da8fe350 00007ff8`2ffcfc36     : 00650063`00630041 002e0056`00730073 00000065`00780065 00007ff8`304f1aa5 : KERNELBASE!CreateProcessInternalW+0xfda
0000003c`da8ffce0 00007ff8`307664f4     : 00000000`00000020 00000000`00000000 0000003c`da8ffe00 00000000`00000000 : KERNELBASE!CreateProcessW+0x66
0000003c`da8ffd50 00007ff6`b89810b2     : 0000003c`da8ffe50 00007ff8`304f354e 00000000`00000000 00007ff8`329e3310 : kernel32!CreateProcessWStub+0x54
0000003c`da8ffdb0 00007ff6`b89815e0     : 00000000`00000000 00007ff6`b8981659 00000000`00000000 00000000`00000000 : AccessV!main+0xb2
0000003c`da8ffee0 00007ff8`307626ad     : 00000000`00000000 00000000`00000000 00000000`00000000 00000000`00000000 : AccessV!__scrt_common_main_seh+0x10c
0000003c`da8fff20 00007ff8`329eaa68     : 00000000`00000000 00000000`00000000 00000000`00000000 00000000`00000000 : kernel32!BaseThreadInitThunk+0x1d
0000003c`da8fff50 00000000`00000000     : 00000000`00000000 00000000`00000000 00000000`00000000 00000000`00000000 : ntdll!RtlUserThreadStart+0x28


FAULTING_SOURCE_LINE:  C:\Users\User\source\repos\AccessV\AccessV\AccessV.cpp

FAULTING_SOURCE_FILE:  C:\Users\User\source\repos\AccessV\AccessV\AccessV.cpp

FAULTING_SOURCE_LINE_NUMBER:  21

FAULTING_SOURCE_CODE:  
    17: 
    18:     // Set the CREATE_NEW_CONSOLE
    19:     DWORD dwCreationFlags = CREATE_NEW_CONSOLE;
    20: 
>   21:     // Create the process
    22:     if (!CreateProcess(
    23:         L"C:\\Windows\\System32\\cmd.exe",  // Path to the executable
    24:         commandLine,                        // Command-line arguments
    25:         NULL,                               // Process handle not inheritable
    26:         NULL,                               // Thread handle not inheritable


SYMBOL_NAME:  accessv!main+b2

MODULE_NAME: AccessV

IMAGE_NAME:  AccessV.exe

STACK_COMMAND:  dt ntdll!LdrpLastDllInitializer BaseDllName ; dt ntdll!LdrpFailureData ; ~0s; .ecxr ; kb

FAILURE_BUCKET_ID:  INVALID_POINTER_READ_c0000005_AccessV.exe!main

OS_VERSION:  10.0.22621.1

BUILDLAB_STR:  ni_release

OSPLATFORM_TYPE:  x64

OSNAME:  Windows 10

FAILURE_ID_HASH:  {06a63211-3ee0-9330-318a-a8218924c64e}

Followup:     MachineOwner
---------
```

The **`!analyze`** output provides detailed information about the crash, and it specifically identifies the issue as a read access violation (AV). This conclusion is supported by the following lines in the output:

**`Key : AV.Fault, Value: Read:`** This key-value pair explicitly states that the fault was a read access violation.

**`ExceptionCode: c0000005 (Access violation):`** This line represents the exception code for an access violation.

This information describes the nature of the access violation, confirming it as an attempt to read from a specific memory address **`(00005db157c7b535)`**.

```
EXCEPTION_RECORD:  (.exr -1)
ExceptionAddress: 00007ff8329aa826 (ntdll!RtlInitUnicodeStringEx+0x0000000000000016)
   ExceptionCode: c0000005 (Access violation)
  ExceptionFlags: 00000000
NumberParameters: 2
   Parameter[0]: 0000000000000000
   Parameter[1]: 00005db157c7b535
Attempt to read from address 00005db157c7b535
```

The output of the **`!address`** command provides information about the memory address **`00005db157c7b535`**, which was involved in the read access violation.

```
0:000> !address 00005db157c7b535

                                     
Mapping file section regions...
Mapping module regions...
Mapping PEB regions...
Mapping TEB and stack regions...
Mapping heap regions...
Mapping page heap regions...
Mapping other regions...
Mapping stack trace database regions...
Mapping activation context regions...

Usage:                  Free
Base Address:           00000180`2d661000
End Address:            00007ff4`cd0f0000
Region Size:            00007e74`9fa8f000 ( 126.456 TB)
State:                  00010000          MEM_FREE
Protect:                00000001          PAGE_NOACCESS <---- This protection flag indicates that no access is allowed to this memory region
Type:                   <info not present at the target>


Content source: 0 (invalid), length: 224375474acb
```

The output tells us that the program attempted to read from a memory address that falls within a region that is not allocated (free) and marked with no access permissions.

# Code Sample (2) - Access Violation

The root cause of the Access Violation in this code is the use of an uninitialized pointer **`uninitializedPointer`**. Since the pointer is not initialized to a valid memory location or set to nullptr, it contains an undefined value that may point to an arbitrary location in memory. Attempting to write to this undefined memory location with the statement **`*uninitializedPointer = 42;`** can lead to a WRITE Access Violation, as the program may be trying to access a memory region that it does not have permission to modify.

```c
#include <windows.h>
#include <iostream>

int main() {
    // Create a file
    HANDLE hFile = CreateFile(L"HelloWorld.txt",
        GENERIC_WRITE,
        0,
        NULL,
        CREATE_ALWAYS,
        FILE_ATTRIBUTE_NORMAL,
        NULL);

    if (hFile == INVALID_HANDLE_VALUE) {
        std::cerr << "Failed to create file. Error: " << GetLastError() << std::endl;
        return 1;
    }

    // Data to write to the file
    const char* data = "Hello, World!";
    DWORD bytesWritten;

    // Write data to the file
    if (!WriteFile(hFile, data, strlen(data), &bytesWritten, NULL)) {
        std::cerr << "Failed to write to file. Error: " << GetLastError() << std::endl;
        CloseHandle(hFile);
        return 1;
    }

    // Close the file handle
    CloseHandle(hFile);

    // Introduce Access Violation (WRITE) by attempting to write to an uninitialized pointer
    int* uninitializedPointer{}; // Uninitialized pointer
    *uninitializedPointer = 42; // This line cause an Access Violation (WRITE)

    return 0;
}
```

Begin by compiling and running this code. We should then notice that the program crashes and does not work as expected. To confirm this, open **Event Viewer**, navigate to the **Application** logs, and search for Event ID **1000**. Here, we can also verify that our program has indeed crashed:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/7c668c05-b762-4123-b209-c57d3bf13f3f)

# WinDbg Walk Through (2) - Analyzing Memory Dump

Start with loading the user mode dump of the process in WinDbg:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/9dfdecf3-dcaa-47de-a5a2-9f0de692b8e6)


Let's start with a simple **`!analyze -v`**

```
0:000> !analyze -v
*******************************************************************************
*                                                                             *
*                        Exception Analysis                                   *
*                                                                             *
*******************************************************************************


KEY_VALUES_STRING: 1

    Key  : AV.Dereference
    Value: NullPtr

    Key  : AV.Fault
    Value: Write

    Key  : Analysis.CPU.mSec
    Value: 2265

    Key  : Analysis.Elapsed.mSec
    Value: 2713

    Key  : Analysis.IO.Other.Mb
    Value: 0

    Key  : Analysis.IO.Read.Mb
    Value: 0

    Key  : Analysis.IO.Write.Mb
    Value: 0

    Key  : Analysis.Init.CPU.mSec
    Value: 343

    Key  : Analysis.Init.Elapsed.mSec
    Value: 1889

    Key  : Analysis.Memory.CommitPeak.Mb
    Value: 94

    Key  : Failure.Bucket
    Value: NULL_POINTER_WRITE_c0000005_AccessV.exe!main

    Key  : Failure.Hash
    Value: {1c59fc7c-49ca-7215-0582-702a810098c4}

    Key  : Timeline.OS.Boot.DeltaSec
    Value: 15512

    Key  : Timeline.Process.Start.DeltaSec
    Value: 2

    Key  : WER.OS.Branch
    Value: ni_release

    Key  : WER.OS.Version
    Value: 10.0.22621.1


FILE_IN_CAB:  AccessV.exe_230824_131953.dmp

COMMENT:  
*** "c:\Users\User\Desktop\procdump64.exe" -accepteula -ma -j "C:\Users\User\Desktop\Minidumps" 7568 672 000001EF30700000
*** Just-In-Time debugger. PID: 7568 Event Handle: 672 JIT Context: .jdinfo 0x1ef30700000

NTGLOBALFLAG:  0

APPLICATION_VERIFIER_FLAGS:  0

CONTEXT:  (.ecxr)
rax=0000000000000000 rbx=0000000000000000 rcx=00007ff95f96ef34
rdx=0000000000000000 rsi=0000000000000000 rdi=000000000000014c
rip=00007ff66d771101 rsp=0000009909b0fe70 rbp=0000000000000000
 r8=0000009909b0fe38  r9=0000000000000000 r10=0000000000000000
r11=0000000000000246 r12=0000000000000000 r13=0000000000000000
r14=0000000000000000 r15=0000000000000000
iopl=0         nv up ei pl zr na po nc
cs=0033  ss=002b  ds=002b  es=002b  fs=0053  gs=002b             efl=00010246
AccessV!main+0x101:
00007ff6`6d771101 c7032a000000    mov     dword ptr [rbx],2Ah ds:00000000`00000000=????????
Resetting default scope

EXCEPTION_RECORD:  (.exr -1)
ExceptionAddress: 00007ff66d771101 (AccessV!main+0x0000000000000101)
   ExceptionCode: c0000005 (Access violation)
  ExceptionFlags: 00000000
NumberParameters: 2
   Parameter[0]: 0000000000000001
   Parameter[1]: 0000000000000000
Attempt to write to address 0000000000000000

PROCESS_NAME:  AccessV.exe

WRITE_ADDRESS:  0000000000000000 

ERROR_CODE: (NTSTATUS) 0xc0000005 - The instruction at 0x%p referenced memory at 0x%p. The memory could not be %s.

EXCEPTION_CODE_STR:  c0000005

EXCEPTION_PARAMETER1:  0000000000000001

EXCEPTION_PARAMETER2:  0000000000000000

STACK_TEXT:  
00000099`09b0fe70 00007ff6`6d7715c0     : 000001ef`3075e940 00007ff6`6d771639 00000000`00000000 00000000`00000000 : AccessV!main+0x101
00000099`09b0fed0 00007ff9`5d9826ad     : 00000000`00000000 00000000`00000000 00000000`00000000 00000000`00000000 : AccessV!__scrt_common_main_seh+0x10c
00000099`09b0ff10 00007ff9`5f92aa68     : 00000000`00000000 00000000`00000000 00000000`00000000 00000000`00000000 : kernel32!BaseThreadInitThunk+0x1d
00000099`09b0ff40 00000000`00000000     : 00000000`00000000 00000000`00000000 00000000`00000000 00000000`00000000 : ntdll!RtlUserThreadStart+0x28


FAULTING_SOURCE_LINE:  C:\Users\User\source\repos\AccessV\AccessV\AccessV.cpp

FAULTING_SOURCE_FILE:  C:\Users\User\source\repos\AccessV\AccessV\AccessV.cpp

FAULTING_SOURCE_LINE_NUMBER:  37

FAULTING_SOURCE_CODE:  
    33:     // Introduce Access Violation (WRITE) by attempting to write to an uninitialized pointer
    34:     int* uninitializedPointer{}; // Uninitialized pointer
    35:     *uninitializedPointer = 42; // This line cause an Access Violation (WRITE)
    36: 
>   37:     return 0;
    38: }


SYMBOL_NAME:  AccessV!main+101

MODULE_NAME: AccessV

IMAGE_NAME:  AccessV.exe

STACK_COMMAND:  dt ntdll!LdrpLastDllInitializer BaseDllName ; dt ntdll!LdrpFailureData ; ~0s; .ecxr ; kb

FAILURE_BUCKET_ID:  NULL_POINTER_WRITE_c0000005_AccessV.exe!main

OS_VERSION:  10.0.22621.1

BUILDLAB_STR:  ni_release

OSPLATFORM_TYPE:  x64

OSNAME:  Windows 10

FAILURE_ID_HASH:  {1c59fc7c-49ca-7215-0582-702a810098c4}

Followup:     MachineOwner
```

The **`!analyze`** output provides detailed information about the crash, and it specifically identifies the issue as an Access Violation caused by attempting to write to a null pointer. The keys **`AV.Fault`** and **`AV.Dereference`** in the output confirm that the Access Violation occurred during a write operation and that a null pointer was dereferenced. 

This output indicates that an Access Violation (exception code **`c0000005`**) occurred within the **main** function of the AccessV module at the offset 0x0101, resulting from an attempt to write to a null memory address (0000000000000000), which is an illegal operation in the system and typically leads to a program crash.

```
EXCEPTION_RECORD:  (.exr -1)
ExceptionAddress: 00007ff66d771101 (AccessV!main+0x0000000000000101)
   ExceptionCode: c0000005 (Access violation)
  ExceptionFlags: 00000000
NumberParameters: 2
   Parameter[0]: 0000000000000001
   Parameter[1]: 0000000000000000
Attempt to write to address 0000000000000000
```

# Code Sample (3) - Access Violation

The root cause of the Access Violation in this code is the attempt to execute code within a memory region that lacks the necessary execute access rights.

```c
#include <windows.h>
#include <iostream>

int main() {
    // Create a buffer for code
    unsigned char code[] = { 0xC3 }; // RET instruction in x86

    // Allocate a read-write page of memory
    LPVOID lpAddress = VirtualAlloc(NULL, sizeof(code), MEM_COMMIT, PAGE_READWRITE);
    if (lpAddress == NULL) {
        std::cerr << "Memory allocation failed. Error: " << GetLastError() << std::endl;
        return 1;
    }

    // Copy the code to the read-write memory
    memcpy(lpAddress, code, sizeof(code));

    // Define a function pointer type that takes no arguments and returns void
    typedef void (*SimpleFunction)();

    // Cast the memory address to the function pointer type
    SimpleFunction function = (SimpleFunction)lpAddress;

    // Attempt to execute the code at the memory address
    // Since the memory was allocated as read-write (not executable), this will cause an access violation
    function(); // Access Violation (EXECUTE)

    // Free the allocated memory
    VirtualFree(lpAddress, 0, MEM_RELEASE);

    return 0;
}
```

Begin by compiling and running this code. We should then notice that the program crashes and does not work as expected. To confirm this, open **Event Viewer**, navigate to the **Application** logs, and search for Event ID **1000**. Here, we can also verify that our program has indeed crashed:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/2f6d6ef3-e532-4c4c-a6a3-5bdb2125ec13)

# WinDbg Walk Through (3) - Analyzing Memory Dump

Start with loading the user mode dump of the process in WinDbg:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/c48297d9-3735-42a1-ac47-6863a34dc794)


Let's start with a simple **`!analyze -v`**

```
0:000> !analyze -v
*******************************************************************************
*                                                                             *
*                        Exception Analysis                                   *
*                                                                             *
*******************************************************************************


KEY_VALUES_STRING: 1

    Key  : AV.Fault
    Value: Execute

    Key  : Analysis.CPU.mSec
    Value: 1467

    Key  : Analysis.Elapsed.mSec
    Value: 1782

    Key  : Analysis.IO.Other.Mb
    Value: 0

    Key  : Analysis.IO.Read.Mb
    Value: 0

    Key  : Analysis.IO.Write.Mb
    Value: 0

    Key  : Analysis.Init.CPU.mSec
    Value: 280

    Key  : Analysis.Init.Elapsed.mSec
    Value: 2093

    Key  : Analysis.Memory.CommitPeak.Mb
    Value: 94

    Key  : Failure.Bucket
    Value: SOFTWARE_NX_FAULT_c0000005_AccessV.exe!main

    Key  : Failure.Hash
    Value: {c74cf91c-fb17-d481-50a2-c86af1f8ac9a}

    Key  : Timeline.OS.Boot.DeltaSec
    Value: 16695

    Key  : Timeline.Process.Start.DeltaSec
    Value: 2

    Key  : WER.OS.Branch
    Value: ni_release

    Key  : WER.OS.Version
    Value: 10.0.22621.1


FILE_IN_CAB:  AccessV.exe_230824_133937.dmp

COMMENT:  
*** "c:\Users\User\Desktop\procdump64.exe" -accepteula -ma -j "C:\Users\User\Desktop\Minidumps" 5544 720 000001F3EF350000
*** Just-In-Time debugger. PID: 5544 Event Handle: 720 JIT Context: .jdinfo 0x1f3ef350000

NTGLOBALFLAG:  0

APPLICATION_VERIFIER_FLAGS:  0

CONTEXT:  (.ecxr)
rax=000001f3ef330000 rbx=000001f3ef330000 rcx=00007ff95f96f054
rdx=0000000000000000 rsi=0000000000000000 rdi=000001f3ed961f00
rip=000001f3ef330000 rsp=00000088358ffd88 rbp=0000000000000000
 r8=00000088358ffd48  r9=0000000000000000 r10=0000000000000000
r11=0000000000000246 r12=0000000000000000 r13=0000000000000000
r14=0000000000000000 r15=0000000000000000
iopl=0         nv up ei pl nz na po nc
cs=0033  ss=002b  ds=002b  es=002b  fs=0053  gs=002b             efl=00010206
000001f3`ef330000 c3              ret
Resetting default scope

EXCEPTION_RECORD:  (.exr -1)
ExceptionAddress: 000001f3ef330000
   ExceptionCode: c0000005 (Access violation)
  ExceptionFlags: 00000000
NumberParameters: 2
   Parameter[0]: 0000000000000008
   Parameter[1]: 000001f3ef330000
Attempt to execute non-executable address 000001f3ef330000

PROCESS_NAME:  AccessV.exe

EXECUTE_ADDRESS: 1f3ef330000

FAILED_INSTRUCTION_ADDRESS: 
+0
000001f3`ef330000 c3              ret

ERROR_CODE: (NTSTATUS) 0xc0000005 - The instruction at 0x%p referenced memory at 0x%p. The memory could not be %s.

EXCEPTION_CODE_STR:  c0000005

EXCEPTION_PARAMETER1:  0000000000000008

EXCEPTION_PARAMETER2:  000001f3ef330000

IP_ON_HEAP:  000001f3ef330000
The fault address in not in any loaded module, please check your build's rebase
log at <releasedir>\bin\build_logs\timebuild\ntrebase.log for module which may
contain the address if it were loaded.

STACK_TEXT:  
00000088`358ffd88 00007ff6`67f11064     : 000001f3`ef330000 00000000`00001000 00000000`00000000 00000000`00000000 : 0x000001f3`ef330000
00000088`358ffd90 00007ff6`67f11500     : 00000000`00000000 00007ff6`67f11579 00000000`00000000 00000000`00000000 : AccessV!main+0x64
00000088`358ffdc0 00007ff9`5d9826ad     : 00000000`00000000 00000000`00000000 00000000`00000000 00000000`00000000 : AccessV!__scrt_common_main_seh+0x10c
00000088`358ffe00 00007ff9`5f92aa68     : 00000000`00000000 00000000`00000000 00000000`00000000 00000000`00000000 : kernel32!BaseThreadInitThunk+0x1d
00000088`358ffe30 00000000`00000000     : 00000000`00000000 00000000`00000000 00000000`00000000 00000000`00000000 : ntdll!RtlUserThreadStart+0x28


FAULTING_SOURCE_LINE:  C:\Users\User\source\repos\AccessV\AccessV\AccessV.cpp

FAULTING_SOURCE_FILE:  C:\Users\User\source\repos\AccessV\AccessV\AccessV.cpp

FAULTING_SOURCE_LINE_NUMBER:  29

FAULTING_SOURCE_CODE:  
    25:     // Since the memory was allocated as read-write (not executable), this will cause an access violation
    26:     function(); // Access Violation (EXECUTE)
    27: 
    28:     // Free the allocated memory
>   29:     VirtualFree(lpAddress, 0, MEM_RELEASE);
    30: 
    31:     return 0;
    32: }


SYMBOL_NAME:  accessv!main+64

MODULE_NAME: AccessV

IMAGE_NAME:  AccessV.exe

STACK_COMMAND:  dt ntdll!LdrpLastDllInitializer BaseDllName ; dt ntdll!LdrpFailureData ; ~0s; .ecxr ; kb

FAILURE_BUCKET_ID:  SOFTWARE_NX_FAULT_c0000005_AccessV.exe!main

OS_VERSION:  10.0.22621.1

BUILDLAB_STR:  ni_release

OSPLATFORM_TYPE:  x64

OSNAME:  Windows 10

FAILURE_ID_HASH:  {c74cf91c-fb17-d481-50a2-c86af1f8ac9a}

Followup:     MachineOwner
---------
```

The **`!analyze`** output provides detailed information about the crash, and it specifically identifies the issue as an Access Violation caused by attempting to execute code in a non-executable memory region. The key **`AV.Fault`** with the value **Execute** in the output confirms that the Access Violation occurred during an execute operation.

The **EXCEPTION_RECORD** output indicates that an Access Violation exception occurred at the memory address **`000001f3ef330000`**. Specifically, the program attempted to execute code at a memory location that was marked as **non-executable**, leading to the exception. 

```
EXCEPTION_RECORD:  (.exr -1)
ExceptionAddress: 000001f3ef330000
   ExceptionCode: c0000005 (Access violation)
  ExceptionFlags: 00000000
NumberParameters: 2
   Parameter[0]: 0000000000000008
   Parameter[1]: 000001f3ef330000
Attempt to execute non-executable address 000001f3ef330000
```

The output of the **`!address`** command provides information about the memory address **`000001f3ef330000`**, which was involved in the read access violation. The critical detail here is the **`PAGE_READWRITE`** protection flag, which confirms that the memory region is not marked as executable. This is consistent with the Access Violation exception that occurred when the code attempted to execute instructions from this non-executable region.

```
0:000> !address 000001f3ef330000

                                     
Mapping file section regions...
Mapping module regions...
Mapping PEB regions...
Mapping TEB and stack regions...
Mapping heap regions...
Mapping page heap regions...
Mapping other regions...
Mapping stack trace database regions...
Mapping activation context regions...

Usage:                  <unknown>
Base Address:           000001f3`ef330000
End Address:            000001f3`ef331000
Region Size:            00000000`00001000 (   4.000 kB)
State:                  00001000          MEM_COMMIT
Protect:                00000004          PAGE_READWRITE <--- The protection attributes of the region. PAGE_READWRITE means that the region is readable and writable but not executable.
Type:                   00020000          MEM_PRIVATE
Allocation Base:        000001f3`ef330000
Allocation Protect:     00000004          PAGE_READWRITE


Content source: 1 (target), length: 1000
```

The fix involves changing the memory protection flag from PAGE_READWRITE to **`PAGE_EXECUTE_READWRITE`** during the allocation of the memory block. This allows the code to be executed from that memory region.


![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/d2cf4444-0c5a-441b-8e34-6a496dd4ee82)

Another way to fix the issue is by using the **VirtualProtect** function to change the memory protection of the allocated region after writing the code to it. We can start by allocating the memory with read and write permissions, and then use **VirtualProtect** to **grant execute** permissions.

```c
#include <windows.h>
#include <iostream>

int main() {
    // Create a buffer for code
    unsigned char code[] = { 0xC3 }; // RET instruction in x86

    // Allocate a read-write page of memory
    LPVOID lpAddress = VirtualAlloc(NULL, sizeof(code), MEM_COMMIT, PAGE_READWRITE);
    if (lpAddress == NULL) {
        std::cerr << "Memory allocation failed. Error: " << GetLastError() << std::endl;
        return 1;
    }

    // Copy the code to the read-write memory
    memcpy(lpAddress, code, sizeof(code));

    // Change the protection to read-write-execute (Fix: Using VirtualProtect)
    DWORD oldProtect;
    if (!VirtualProtect(lpAddress, sizeof(code), PAGE_EXECUTE_READWRITE, &oldProtect)) { // Fix applied here
        std::cerr << "Failed to change memory protection. Error: " << GetLastError() << std::endl;
        return 1;
    }

    // Define a function pointer type that takes no arguments and returns void
    typedef void (*SimpleFunction)();

    // Cast the memory address to the function pointer type
    SimpleFunction function = (SimpleFunction)lpAddress;

    // Attempt to execute the code at the memory address
    // Since the memory protection has been changed to read-write-execute, this will not cause an access violation
    function(); // Fix ensures no Access Violation (EXECUTE)

    // Free the allocated memory
    VirtualFree(lpAddress, 0, MEM_RELEASE);

    return 0;
}
```

The key fix here is the addition of the **VirtualProtect** function, which changes the memory protection of the allocated region to **`PAGE_EXECUTE_READWRITE`** after the code has been written. This ensures that the region is executable, and the attempt to call the function will not cause an access violation.

