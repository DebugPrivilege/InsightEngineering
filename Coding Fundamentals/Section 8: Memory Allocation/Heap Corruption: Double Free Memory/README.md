# What is Double Free Memory?

Double freeing memory in C and C++ leads to undefined behavior, potentially corrupting the memory management structures of the program. This can result in program crashes, data corruption, and more.

Here is an example of code that is freeing memory twice:

```c
#include <windows.h>
#include <stdio.h>
#include <wchar.h>

int main() {
    // Define the filenames as an array of wide character pointers
    wchar_t* filenames[3];

    wprintf(L"Enter the names of three files (including extension):\n");

    for (int i = 0; i < 3; i++) {
        // Allocate initial buffer size of 1 character
        size_t bufferSize = 1;
        filenames[i] = malloc(bufferSize * sizeof(wchar_t));
        if (filenames[i] == NULL) {
            wprintf(L"Memory allocation failed\n");
            return 1;
        }

        // Read the filename
        wprintf(L"Filename %d: ", i + 1);
        wint_t ch;
        size_t length = 0;
        while ((ch = fgetwc(stdin)) != L'\n' && ch != WEOF) {
            filenames[i][length++] = (wchar_t)ch;
            if (length >= bufferSize) {
                bufferSize *= 2;
                wchar_t* temp = realloc(filenames[i], bufferSize * sizeof(wchar_t));
                if (temp == NULL) {
                    wprintf(L"Memory allocation failed\n");
                    free(filenames[i]);
                    return 1;
                }
                filenames[i] = temp;
            }
        }

        // Null-terminate the filename
        filenames[i][length] = L'\0';

        // Use the CreateFile function to create a new file
        HANDLE hFile = CreateFile(
            filenames[i],             // name of the file
            GENERIC_WRITE,            // open for writing
            0,                        // do not share
            NULL,                     // default security
            CREATE_NEW,               // create new file only
            FILE_ATTRIBUTE_NORMAL,    // normal file
            NULL                      // no attr. template
        );

        // Check if the file was created successfully
        if (hFile == INVALID_HANDLE_VALUE) {
            wprintf(L"Unable to create file %ls\n", filenames[i]);
        }
        else {
            wprintf(L"File %ls created successfully\n", filenames[i]);

            // Write BOM for UTF-16 Little Endian
            const wchar_t bom = 0xFEFF;
            DWORD bytesWritten;
            WriteFile(hFile, &bom, sizeof(bom), &bytesWritten, NULL);

            // Write "Hello World!" 100 times to the file
            const wchar_t* message = L"Hello World!\n";
            for (int j = 0; j < 100; j++) {
                WriteFile(hFile, message, wcslen(message) * sizeof(wchar_t), &bytesWritten, NULL);
            }

            // Close the file handle
            CloseHandle(hFile);
        }


        // Free the memory allocated for the filename
        free(filenames[i]);

        // Freeing the memory again (intentional mistake for demonstration)
        free(filenames[i]);
    }

    return 0;
}
```

Start with compiling the code and run it:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/07f9fd1f-ca53-4d07-a37b-ca69a471e792)


After we created a file on disk, the program will start crashing. It doesn't allow us to create additional files, so when we open **Event Viewer** and go to the **Application** logs. We can indeed see that this program has crashed:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/e90c22e8-1e7a-4a97-96e3-0f805c18002d)


The error code **0xc0000374** is a status code which corresponds to **STATUS_HEAP_CORRUPTION**. In simpler terms, this error indicates that there's been a corruption in the heap, which is the region of a computer's memory space used for dynamic memory allocation.

Heap corruption can arise from various programming mistakes, such as:

- Freeing a block of memory more than once (double free).
- Freeing invalid memory.
- Using a block of memory after it has been freed (dangling pointer).
- Not initializing memory before using it.

When a program encounters heap corruption, its behavior becomes unpredictable, and it may even crash.

# WinDbg Walk Through - Analyzing Memory Dump

Start with loading the memory dump of the crashed program in WinDbg:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/dd0af44a-be73-4636-ac5c-91e908112c31)


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
    Value: 389

    Key  : Analysis.Elapsed.mSec
    Value: 565

    Key  : Analysis.IO.Other.Mb
    Value: 0

    Key  : Analysis.IO.Read.Mb
    Value: 0

    Key  : Analysis.IO.Write.Mb
    Value: 0

    Key  : Analysis.Init.CPU.mSec
    Value: 171

    Key  : Analysis.Init.Elapsed.mSec
    Value: 6350

    Key  : Analysis.Memory.CommitPeak.Mb
    Value: 62

    Key  : Failure.Bucket
    Value: HEAP_CORRUPTION_ACTIONABLE_BlockNotBusy_DOUBLE_FREE_c0000374_ntdll.dll!RtlpFreeHeapInternal

    Key  : Failure.Hash
    Value: {f9e860eb-b03f-7415-804c-7e671e26c730}

    Key  : Timeline.OS.Boot.DeltaSec
    Value: 16476

    Key  : Timeline.Process.Start.DeltaSec
    Value: 4

    Key  : WER.OS.Branch
    Value: ni_release

    Key  : WER.OS.Version
    Value: 10.0.22621.1


FILE_IN_CAB:  MemoryIssue.exe_230814_044449.dmp

COMMENT:  
*** "c:\Users\User\Desktop\procdump64.exe" -accepteula -ma -j "C:\Users\User\Desktop\Minidumps" 14904 636 000002911B590000
*** Just-In-Time debugger. PID: 14904 Event Handle: 636 JIT Context: .jdinfo 0x2911b590000

NTGLOBALFLAG:  0

APPLICATION_VERIFIER_FLAGS:  0

CONTEXT:  (.ecxr)
rax=000002911b438c50 rbx=00000000c0000374 rcx=0000000000000100
rdx=0000000000000000 rsi=0000000000000001 rdi=00007ff874e118a0
rip=00007ff874d9c239 rsp=000000b0106ffa10 rbp=000002911b43c770
 r8=0000000000000000  r9=0000000000000200 r10=0000000000000000
r11=000000b0106feda0 r12=0000000000000000 r13=000002911b430000
r14=0000000000000000 r15=0000000000000000
iopl=0         nv up ei pl nz na po nc
cs=0033  ss=002b  ds=002b  es=002b  fs=0053  gs=002b             efl=00000206
ntdll!RtlReportFatalFailure+0x9:
00007ff8`74d9c239 eb00            jmp     ntdll!RtlReportFatalFailure+0xb (00007ff8`74d9c23b)
Resetting default scope

EXCEPTION_RECORD:  (.exr -1)
ExceptionAddress: 00007ff874d9c239 (ntdll!RtlReportFatalFailure+0x0000000000000009)
   ExceptionCode: c0000374
  ExceptionFlags: 00000081
NumberParameters: 1
   Parameter[0]: 00007ff874e118a0

PROCESS_NAME:  MemoryIssue.exe

ERROR_CODE: (NTSTATUS) 0xc0000374 - A heap has been corrupted.

EXCEPTION_CODE_STR:  c0000374

EXCEPTION_PARAMETER1:  00007ff874e118a0

ADDITIONAL_DEBUG_TEXT:  Followup set based on attribute [Heap_Error_Type] from Frame:[0] on thread:[PSEUDO_THREAD] ; Followup set based on attribute [Is_ChosenCrashFollowupThread] from Frame:[0] on thread:[PSEUDO_THREAD]

FAULTING_THREAD:  ffffffff

STACK_TEXT:  
00000000`00000000 00000000`00000000 ntdll!RtlpFreeHeapInternal+0x0


STACK_COMMAND:  !heap ; ** Pseudo Context ** ManagedPseudo ** Value: ffffffff ** ; kb

SYMBOL_NAME:  ntdll!RtlpFreeHeapInternal+0

MODULE_NAME: ntdll

IMAGE_NAME:  ntdll.dll

FAILURE_BUCKET_ID:  HEAP_CORRUPTION_ACTIONABLE_BlockNotBusy_DOUBLE_FREE_c0000374_ntdll.dll!RtlpFreeHeapInternal

OS_VERSION:  10.0.22621.1

BUILDLAB_STR:  ni_release

OSPLATFORM_TYPE:  x64

OSNAME:  Windows 10

IMAGE_VERSION:  6.2.22621.2134

FAILURE_ID_HASH:  {f9e860eb-b03f-7415-804c-7e671e26c730}

Followup:     MachineOwner
---------
```

The **FAILURE_BUCKET_ID** suggests this heap corruption was caused by a "double free" (DOUBLE_FREE), which means memory was attempted to be freed twice.


This is how the crashing call stack looks like:

```
0:000> !mex.crash
Last exception
============================================
ExceptionAddress: 00007ff874d9c239 (ntdll!RtlReportFatalFailure+0x0000000000000009)
   ExceptionCode: c0000374
  ExceptionFlags: 00000081
NumberParameters: 1
   Parameter[0]: 00007ff874e118a0

Setting context to the last exception
============================================
rax=000002911b438c50 rbx=00000000c0000374 rcx=0000000000000100
rdx=0000000000000000 rsi=0000000000000001 rdi=00007ff874e118a0
rip=00007ff874d9c239 rsp=000000b0106ffa10 rbp=000002911b43c770
 r8=0000000000000000  r9=0000000000000200 r10=0000000000000000
r11=000000b0106feda0 r12=0000000000000000 r13=000002911b430000
r14=0000000000000000 r15=0000000000000000
iopl=0         nv up ei pl nz na po nc
cs=0033  ss=002b  ds=002b  es=002b  fs=0053  gs=002b             efl=00000206
ntdll!RtlReportFatalFailure+0x9:
00007ff8`74d9c239 eb00            jmp     ntdll!RtlReportFatalFailure+0xb (00007ff8`74d9c23b)

Crashing Stack
============================================
  *** Stack trace for last set context - .thread/.cxr resets it
 # Child-SP          RetAddr               Call Site
00 000000b0`106ffa10 00007ff8`74d9c203     ntdll!RtlReportFatalFailure+0x9
01 000000b0`106ffa60 00007ff8`74da529a     ntdll!RtlReportCriticalFailure+0x97
02 000000b0`106ffb50 00007ff8`74da557a     ntdll!RtlpHeapHandleError+0x12
03 000000b0`106ffb80 00007ff8`74db1575     ntdll!RtlpHpHeapHandleError+0x7a
04 000000b0`106ffbb0 00007ff8`74ccbebc     ntdll!RtlpLogHeapFailure+0x45
05 000000b0`106ffbe0 00007ff8`74ccab01     ntdll!RtlpFreeHeapInternal+0x84c
06 000000b0`106ffca0 00007ff6`4a0497d0     ntdll!RtlFreeHeap+0x51
07 000000b0`106ffce0 00007ff6`4a04120f     MemoryIssue!_free_base+0x1c [minkernel\crts\ucrt\src\appcrt\heap\free_base.cpp @ 105] 
08 000000b0`106ffd10 00007ff6`4a041490     MemoryIssue!main+0x19f [C:\Users\User\source\repos\MemoryIssue\MemoryIssue\MemoryIssue.c @ 79] 
09 (Inline Function) --------`--------     MemoryIssue!invoke_main+0x22 [D:\a\_work\1\s\src\vctools\crt\vcstartup\src\startup\exe_common.inl @ 78] 
0a 000000b0`106ffd90 00007ff8`744a26ad     MemoryIssue!__scrt_common_main_seh+0x10c [D:\a\_work\1\s\src\vctools\crt\vcstartup\src\startup\exe_common.inl @ 288] 
0b 000000b0`106ffdd0 00007ff8`74ceaa68     kernel32!BaseThreadInitThunk+0x1d
0c 000000b0`106ffe00 00000000`00000000     ntdll!RtlUserThreadStart+0x28
```

The program **MemoryIssue.exe** encountered a heap corruption error. There are functions in the call stack related to error reporting and exception handling (**`ntdll!RtlpLogHeapFailure`**, **`ntdll!RtlpHpHeapHandleError`**, **`ntdll!RtlpHeapHandleError`**, **`ntdll!RtlReportCriticalFailure`**, and others). 

This suggests that there was an error during the heap operation, likely related to memory management. The specific function **`ntdll!RtlReportFatalFailure`** suggests a fatal error in the heap, further confirming issues with memory management.

In **MEX**, there's another beneficial command, **`!mheap`**, designed specifically for user mode dumps. It can be used to diagnose heap issues and provides a summary of the heap.

```
0:000> !mheap


************************************************************************************************************************
                                              NT HEAP STATS BELOW
************************************************************************************************************************
**************************************************************
*                                                            *
*                  HEAP ERROR DETECTED                       *
*                                                            *
**************************************************************

Details:

Heap address:  000002decbae0000
Error address: 000002decbaec750
Error type: HEAP_FAILURE_BLOCK_NOT_BUSY
Details:    The caller performed an operation (such as a free
            or a size check) that is illegal on a free block.
Follow-up:  Check the error's stack trace to find the culprit.


Stack trace:
Stack trace at 0x00007ffbf6b318f8
    00007ffbf6ad1575: ntdll!RtlpLogHeapFailure+0x45
    00007ffbf69ebebc: ntdll!RtlpFreeHeapInternal+0x84c
    00007ffbf69eab01: ntdll!RtlFreeHeap+0x51
    00007ff767de97d0: MemoryIssue!_free_base+0x1c
    00007ff767de120f: MemoryIssue!main+0x19f
    00007ff767de1490: MemoryIssue!__scrt_common_main_seh+0x10c
    00007ffbf49f26ad: kernel32!BaseThreadInitThunk+0x1d
    00007ffbf6a0aa68: ntdll!RtlUserThreadStart+0x28

LFH Key                   : 0xf39d1efbdd3f221a
Termination on corruption : ENABLED
          Heap     Flags   Reserv  Commit  Virt   Free  List   UCR  Virt  Lock  Fast 
                            (k)     (k)    (k)     (k) length      blocks cont. heap 
-------------------------------------------------------------------------------------
000002decbae0000 00000002    1124     68   1020      1     2     1    0      0   LFH
000002decb8f0000 00008000      64      4     64      2     1     1    0      0      
-------------------------------------------------------------------------------------
```
