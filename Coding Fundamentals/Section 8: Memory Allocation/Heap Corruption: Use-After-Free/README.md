# What is Use-After-Free?

Use-After-Free is a type of software bug where a program continues to use a memory location after it has been **freed** or **released**. This can lead to various issues, including data corruption, crashes, and more.

This code has a "use-after-free" problem, where we try to use memory after it's been given back or "freed". This means after letting go of the memory for **`corruptPtr`**, the code attempts to write to that now-freed memory, which may lead to unpredictable behavior and potential crashes.

```c
#include <windows.h>
#include <stdio.h>

int main() {
    // Buffer to store the user-entered filename
    wchar_t filename[100];

    // Prompt the user to enter a filename
    wprintf(L"Enter filename: ");
    fgetws(filename, 100, stdin);

    // Remove any trailing newline character from the user input
    size_t len = wcslen(filename);
    if (filename[len - 1] == L'\n') {
        filename[len - 1] = L'\0';
    }

    // Construct the full path for the file, placing it in the C:\Temp directory
    wchar_t fullPath[150];
    swprintf(fullPath, 150, L"C:\\Temp\\%s", filename);

    // Try to create or open the specified file with write access
    HANDLE fileHandle = CreateFileW(
        fullPath,
        GENERIC_WRITE,
        0,
        NULL,
        CREATE_NEW,
        FILE_ATTRIBUTE_NORMAL,
        NULL
    );

    // Check if file creation/opening was successful
    if (fileHandle == INVALID_HANDLE_VALUE) {
        wprintf(L"Unable to create file due to error: %lu\n", GetLastError());
        return 1;
    }

    // Define the content to be written to the file
    wchar_t data[] = L"Hello, World!";
    DWORD bytesWritten;

    // Write the defined content to the file
    if (!WriteFile(
        fileHandle,
        data,
        wcslen(data) * sizeof(wchar_t), // Calculate the number of bytes to write
        &bytesWritten,
        NULL
    )) {
        wprintf(L"Unable to write to file due to error: %lu\n", GetLastError());
        return 1;
    }

    // Allocate 2000 bytes (1000 wide characters, each 2 bytes) on the heap
    wchar_t* corruptPtr = (wchar_t*)malloc(1000 * sizeof(wchar_t));

    // Immediately free the allocated memory
    free(corruptPtr);

    // Attempt to write to the memory block that was just freed, leading to heap corruption
    corruptPtr[4] = L'A';

    // Properly close the file handle
    CloseHandle(fileHandle);

    return 0;
}
```

Start compiling this code and run it:

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/055b1539-4194-4d47-aaa0-e59320598f78)

The program attempts to write to the memory that was just freed by changing the value at **`corruptPtr[4]`** to **`'A'`**. This is the "use-after-free" part, which leads to heap corruption.

After we created a file on disk, the program will start crashing. When we open **Event Viewer** and go to the **Application** logs. We can indeed see that this program has crashed:

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/be6fe9cb-392c-43b5-b2d7-291c7122444c)

The error code **0xc0000374** is a status code which corresponds to **STATUS_HEAP_CORRUPTION**. In simpler terms, this error indicates that there's been a corruption in the heap, which is the region of a computer's memory space used for dynamic memory allocation.

Heap corruption can arise from various programming mistakes, such as:

- Freeing a block of memory more than once (double free).
- Freeing invalid memory.
- Using a block of memory after it has been freed (dangling pointer).
- Not initializing memory before using it.

When a program encounters heap corruption, its behavior becomes unpredictable, and it may even crash.

# WinDbg Walk Through - Analyzing Memory Dump

Load the memory dump of the crashed program in WinDbg:

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/4a8b5fe7-53e0-4af5-9625-874f779d6958)

Start with the **`!analyze -v`** command:

```
0:000> !analyze -v
*******************************************************************************
*                                                                             *
*                        Exception Analysis                                   *
*                                                                             *
*******************************************************************************

*** WARNING: Check Image - Checksum mismatch - Dump: 0x21bad9, File: 0x22051c - C:\Symbols\ntdll.dll\EEE69EC7214000\ntdll.dll

KEY_VALUES_STRING: 1

    Key  : Analysis.CPU.mSec
    Value: 515

    Key  : Analysis.Elapsed.mSec
    Value: 629

    Key  : Analysis.IO.Other.Mb
    Value: 0

    Key  : Analysis.IO.Read.Mb
    Value: 0

    Key  : Analysis.IO.Write.Mb
    Value: 0

    Key  : Analysis.Init.CPU.mSec
    Value: 140

    Key  : Analysis.Init.Elapsed.mSec
    Value: 1927

    Key  : Analysis.Memory.CommitPeak.Mb
    Value: 81

    Key  : Failure.Bucket
    Value: HEAP_CORRUPTION_c0000374_heap_corruption!Mem.exe

    Key  : Failure.Hash
    Value: {4569e799-2e9f-5336-d420-e475db03ba4f}

    Key  : Timeline.OS.Boot.DeltaSec
    Value: 4261

    Key  : Timeline.Process.Start.DeltaSec
    Value: 4

    Key  : WER.OS.Branch
    Value: ni_release

    Key  : WER.OS.Version
    Value: 10.0.22621.1


FILE_IN_CAB:  Mem.exe_230814_081548.dmp

COMMENT:  
*** "c:\Users\User\Desktop\procdump64.exe" -accepteula -ma -j "C:\Users\User\Desktop\Minidumps" 8168 616 000001FFCA6E0000
*** Just-In-Time debugger. PID: 8168 Event Handle: 616 JIT Context: .jdinfo 0x1ffca6e0000

NTGLOBALFLAG:  0

APPLICATION_VERIFIER_FLAGS:  0

CONTEXT:  (.ecxr)
rax=3f3e3d3c3b3a3938 rbx=00000000c0000374 rcx=4746454443424140
rdx=4f4e4d4c4b4a4948 rsi=0000000000000001 rdi=00007ff966bf18a0
rip=00007ff966b7c239 rsp=0000007b4f8fed30 rbp=0000007b4f8ff1f9
 r8=7f7e7d7c7b7a7978  r9=8786858483828180 r10=8f8e8d8c8b8a8988
r11=9796959493929190 r12=000001ffca7177f0 r13=000001ffca721410
r14=000001ffca71e3d0 r15=0000000000000014
iopl=0         nv up ei pl nz na pe nc
cs=0033  ss=002b  ds=002b  es=002b  fs=0053  gs=002b             efl=00000202
ntdll!RtlReportFatalFailure+0x9:
00007ff9`66b7c239 eb00            jmp     ntdll!RtlReportFatalFailure+0xb (00007ff9`66b7c23b)
Resetting default scope

EXCEPTION_RECORD:  (.exr -1)
ExceptionAddress: 00007ff966b7c239 (ntdll!RtlReportFatalFailure+0x0000000000000009)
   ExceptionCode: c0000374
  ExceptionFlags: 00000081
NumberParameters: 1
   Parameter[0]: 00007ff966bf18a0

PROCESS_NAME:  Mem.exe

ERROR_CODE: (NTSTATUS) 0xc0000374 - A heap has been corrupted.

EXCEPTION_CODE_STR:  c0000374

EXCEPTION_PARAMETER1:  00007ff966bf18a0

ADDITIONAL_DEBUG_TEXT:  Enable Pageheap/AutoVerifer ; Followup set based on attribute [Is_ChosenCrashFollowupThread] from Frame:[0] on thread:[PSEUDO_THREAD]

FAULTING_THREAD:  0000150c

STACK_TEXT:  
00000000`00000000 00000000`00000000 heap_corruption!Mem.exe+0x0


STACK_COMMAND:  ** Pseudo Context ** ManagedPseudo ** Value: ffffffff ** ; kb

SYMBOL_NAME:  heap_corruption!Mem.exe

MODULE_NAME: heap_corruption

IMAGE_NAME:  heap_corruption

FAILURE_BUCKET_ID:  HEAP_CORRUPTION_c0000374_heap_corruption!Mem.exe

OS_VERSION:  10.0.22621.1

BUILDLAB_STR:  ni_release

OSPLATFORM_TYPE:  x64

OSNAME:  Windows 10

FAILURE_ID_HASH:  {4569e799-2e9f-5336-d420-e475db03ba4f}

Followup:     MachineOwner
---------
```

This is how the crashing call stack looks like:

```
0:001> !mex.crash
Last exception
============================================
ExceptionAddress: 00007ff966b7c239 (ntdll!RtlReportFatalFailure+0x0000000000000009)
   ExceptionCode: c0000374
  ExceptionFlags: 00000081
NumberParameters: 1
   Parameter[0]: 00007ff966bf18a0

Setting context to the last exception
============================================
rax=3f3e3d3c3b3a3938 rbx=00000000c0000374 rcx=4746454443424140
rdx=4f4e4d4c4b4a4948 rsi=0000000000000001 rdi=00007ff966bf18a0
rip=00007ff966b7c239 rsp=0000007b4f8fed30 rbp=0000007b4f8ff1f9
 r8=7f7e7d7c7b7a7978  r9=8786858483828180 r10=8f8e8d8c8b8a8988
r11=9796959493929190 r12=000001ffca7177f0 r13=000001ffca721410
r14=000001ffca71e3d0 r15=0000000000000014
iopl=0         nv up ei pl nz na pe nc
cs=0033  ss=002b  ds=002b  es=002b  fs=0053  gs=002b             efl=00000202
ntdll!RtlReportFatalFailure+0x9:
00007ff9`66b7c239 eb00            jmp     ntdll!RtlReportFatalFailure+0xb (00007ff9`66b7c23b)

Crashing Stack
============================================
  *** Stack trace for last set context - .thread/.cxr resets it
 # Child-SP          RetAddr               Call Site
00 0000007b`4f8fed30 00007ff9`66b7c203     ntdll!RtlReportFatalFailure+0x9
01 0000007b`4f8fed80 00007ff9`66b8529a     ntdll!RtlReportCriticalFailure+0x97
02 0000007b`4f8fee70 00007ff9`66b8557a     ntdll!RtlpHeapHandleError+0x12
03 0000007b`4f8feea0 00007ff9`66b91575     ntdll!RtlpHpHeapHandleError+0x7a
04 0000007b`4f8feed0 00007ff9`66ab02c6     ntdll!RtlpLogHeapFailure+0x45
05 0000007b`4f8fef00 00007ff9`66aacd49     ntdll!RtlpAllocateHeap+0x1686
06 0000007b`4f8ff160 00007ff9`66a9df48     ntdll!RtlpAllocateHeapInternal+0x6c9
07 0000007b`4f8ff260 00007ff9`66a9deae     ntdll!LdrpAllocateModuleEntry+0x38
08 0000007b`4f8ff290 00007ff9`66a9dd84     ntdll!LdrpAllocatePlaceHolder+0xce
09 0000007b`4f8ff2d0 00007ff9`66a988a6     ntdll!LdrpFindOrPrepareLoadingModule+0x98
0a 0000007b`4f8ff340 00007ff9`66a88c9c     ntdll!LdrpLoadDllInternal+0x182
0b 0000007b`4f8ff3e0 00007ff9`66a9a24a     ntdll!LdrpLoadDll+0xb0
0c 0000007b`4f8ff5a0 00007ff9`63fb5ef2     ntdll!LdrLoadDll+0xfa
0d 0000007b`4f8ff690 00007ff9`64731843     KERNELBASE!LoadLibraryExW+0x172
0e 0000007b`4f8ff700 00007ff9`64738f91     ucrtbase!try_get_module+0x4b
0f 0000007b`4f8ff730 00007ff9`6472ba90     ucrtbase!__acrt_AppPolicyGetProcessTerminationMethodInternal+0x79
10 0000007b`4f8ff760 00007ff9`6472bcd9     ucrtbase!exit_or_terminate_process+0x20
11 0000007b`4f8ff790 00007ff6`945717d3     ucrtbase!common_exit+0x79
12 0000007b`4f8ff7f0 00007ff9`656826ad     Mem!__scrt_common_main_seh+0x173 [D:\a\_work\1\s\src\vctools\crt\vcstartup\src\startup\exe_common.inl @ 295] 
13 0000007b`4f8ff830 00007ff9`66acaa68     kernel32!BaseThreadInitThunk+0x1d
14 0000007b`4f8ff860 00000000`00000000     ntdll!RtlUserThreadStart+0x28
```

This call stack appears to show that during the process of allocating memory or while loading a module, the program detected a heap corruption. The specific mention of **`ntdll!RtlpHeapHandleError`** and the error code **`c0000374`** confirms the heap corruption.
