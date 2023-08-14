# Freeing Invalid Memory

This code allocates memory on the heap and stores its address in the **`corruptPtr`** pointer. However, instead of freeing the original memory block, it attempts to free an offset from the original address **`(corruptPtr + 5)`**. This misuse of the **free** function can lead to heap corruption or crashes, as it's trying to release an address that wasn't directly allocated by **malloc**.

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

    // Attempt to free a random memory address (that wasn't allocated via malloc)
    free(corruptPtr + 5);

    // Properly close the file handle
    CloseHandle(fileHandle);

    return 0;
}
```

Running the code with the "Freeing Invalid Memory" issue can lead to unpredictable behavior, ranging from program crashes to silent heap corruption:

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/f01ad777-4dcb-47f2-806c-66f03bd5aba6)

After we created a file on disk, the program will start crashing. When we open **Event Viewer** and go to the **Application** logs. We can indeed see that this program has crashed:

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/e34f8ab2-282a-4939-b4ba-e25a6d000dfb)

The error code **0xc0000374** is a status code which corresponds to **STATUS_HEAP_CORRUPTION**. In simpler terms, this error indicates that there's been a corruption in the heap, which is the region of a computer's memory space used for dynamic memory allocation.

Heap corruption can arise from various programming mistakes, such as:

- Freeing a block of memory more than once (double free).
- Freeing invalid memory.
- Using a block of memory after it has been freed (dangling pointer).
- Not initializing memory before using it.

When a program encounters heap corruption, its behavior becomes unpredictable, and it may even crash.

# WinDbg Walk Through - Analyzing Memory Dump

Load the memory dump of the crashed program in WinDbg:

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/ee1165a0-3f3c-40df-aa73-750b851ee7c6)

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
    Value: 483

    Key  : Analysis.Elapsed.mSec
    Value: 646

    Key  : Analysis.IO.Other.Mb
    Value: 0

    Key  : Analysis.IO.Read.Mb
    Value: 0

    Key  : Analysis.IO.Write.Mb
    Value: 0

    Key  : Analysis.Init.CPU.mSec
    Value: 108

    Key  : Analysis.Init.Elapsed.mSec
    Value: 2213

    Key  : Analysis.Memory.CommitPeak.Mb
    Value: 66

    Key  : Failure.Bucket
    Value: HEAP_CORRUPTION_ACTIONABLE_InvalidArgument_c0000374_heap_corruption!Mem.exe

    Key  : Failure.Hash
    Value: {72ed97fe-38ec-48eb-7070-f52fe2155b8f}

    Key  : Timeline.OS.Boot.DeltaSec
    Value: 5827

    Key  : Timeline.Process.Start.DeltaSec
    Value: 8

    Key  : WER.OS.Branch
    Value: ni_release

    Key  : WER.OS.Version
    Value: 10.0.22621.1


FILE_IN_CAB:  Mem.exe_230814_084154.dmp

COMMENT:  
*** "c:\Users\User\Desktop\procdump64.exe" -accepteula -ma -j "C:\Users\User\Desktop\Minidumps" 5824 604 00000294257B0000
*** Just-In-Time debugger. PID: 5824 Event Handle: 604 JIT Context: .jdinfo 0x294257b0000

NTGLOBALFLAG:  0

APPLICATION_VERIFIER_FLAGS:  0

CONTEXT:  (.ecxr)
rax=0000029425817a60 rbx=00000000c0000374 rcx=00007ff966bf4060
rdx=000000cc64000165 rsi=0000000000000001 rdi=00007ff966bf18a0
rip=00007ff966b7c239 rsp=000000cc15def1d0 rbp=0000000000000000
 r8=000000cc15dee7a0  r9=00007ff9564e1588 r10=000000cc15dee6b0
r11=000000cc15dee7a0 r12=0000000000000000 r13=0000029425810000
r14=0000000000000000 r15=0000000000000000
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

ADDITIONAL_DEBUG_TEXT:  Followup set based on attribute [Is_ChosenCrashFollowupThread] from Frame:[0] on thread:[PSEUDO_THREAD]

FAULTING_THREAD:  ffffffff

STACK_TEXT:  
00000000`00000000 00000000`00000000 heap_corruption!Mem.exe+0x0


STACK_COMMAND:  !heap ; ** Pseudo Context ** ManagedPseudo ** Value: ffffffff ** ; kb

SYMBOL_NAME:  heap_corruption!Mem.exe

MODULE_NAME: heap_corruption

IMAGE_NAME:  heap_corruption

FAILURE_BUCKET_ID:  HEAP_CORRUPTION_ACTIONABLE_InvalidArgument_c0000374_heap_corruption!Mem.exe

OS_VERSION:  10.0.22621.1

BUILDLAB_STR:  ni_release

OSPLATFORM_TYPE:  x64

OSNAME:  Windows 10

FAILURE_ID_HASH:  {72ed97fe-38ec-48eb-7070-f52fe2155b8f}

Followup:     MachineOwner
---------
```

The **FAILURE_BUCKET_ID** indicates that the **Mem.exe** program crashed due to a heap corruption that might have been caused by passing an invalid argument to a function or method.

```
FAILURE_BUCKET_ID:  HEAP_CORRUPTION_ACTIONABLE_InvalidArgument_c0000374_heap_corruption!Mem.exe
```

This is how the crashing call stack looks like:

```
0:000> !mex.crash
Last exception
============================================
ExceptionAddress: 00007ff966b7c239 (ntdll!RtlReportFatalFailure+0x0000000000000009)
   ExceptionCode: c0000374
  ExceptionFlags: 00000081
NumberParameters: 1
   Parameter[0]: 00007ff966bf18a0

Setting context to the last exception
============================================
rax=0000029425817a60 rbx=00000000c0000374 rcx=00007ff966bf4060
rdx=000000cc64000165 rsi=0000000000000001 rdi=00007ff966bf18a0
rip=00007ff966b7c239 rsp=000000cc15def1d0 rbp=0000000000000000
 r8=000000cc15dee7a0  r9=00007ff9564e1588 r10=000000cc15dee6b0
r11=000000cc15dee7a0 r12=0000000000000000 r13=0000029425810000
r14=0000000000000000 r15=0000000000000000
iopl=0         nv up ei pl nz na pe nc
cs=0033  ss=002b  ds=002b  es=002b  fs=0053  gs=002b             efl=00000202
ntdll!RtlReportFatalFailure+0x9:
00007ff9`66b7c239 eb00            jmp     ntdll!RtlReportFatalFailure+0xb (00007ff9`66b7c23b)

Crashing Stack
============================================
  *** Stack trace for last set context - .thread/.cxr resets it
 # Child-SP          RetAddr               Call Site
00 000000cc`15def1d0 00007ff9`66b7c203     ntdll!RtlReportFatalFailure+0x9
01 000000cc`15def220 00007ff9`66b8529a     ntdll!RtlReportCriticalFailure+0x97
02 000000cc`15def310 00007ff9`66b8557a     ntdll!RtlpHeapHandleError+0x12
03 000000cc`15def340 00007ff9`66b91575     ntdll!RtlpHpHeapHandleError+0x7a
04 000000cc`15def370 00007ff9`66aabded     ntdll!RtlpLogHeapFailure+0x45
05 000000cc`15def3a0 00007ff9`66aaab01     ntdll!RtlpFreeHeapInternal+0x77d
06 000000cc`15def460 00007ff9`647237eb     ntdll!RtlFreeHeap+0x51
07 000000cc`15def4a0 00007ff6`e233125d     ucrtbase!_free_base+0x1b
08 000000cc`15def4d0 00007ff6`e233175c     Mem!main+0x18d [C:\Users\User\source\repos\Mem\Mem\Mem.c @ 62] 
09 (Inline Function) --------`--------     Mem!invoke_main+0x22 [D:\a\_work\1\s\src\vctools\crt\vcstartup\src\startup\exe_common.inl @ 78] 
0a 000000cc`15def760 00007ff9`656826ad     Mem!__scrt_common_main_seh+0x10c [D:\a\_work\1\s\src\vctools\crt\vcstartup\src\startup\exe_common.inl @ 288] 
0b 000000cc`15def7a0 00007ff9`66acaa68     kernel32!BaseThreadInitThunk+0x1d
0c 000000cc`15def7d0 00000000`00000000     ntdll!RtlUserThreadStart+0x28
```
