# What is an Orphaned Lock?

An Orphan Lock refers to the situation where a thread acquires a lock and then **terminates** or **exits** without releasing the lock. Another thread that tries to acquire the same lock will then be blocked indefinitely, leading to a deadlock if it holds other resources.

The purpose of this code is the following:

- The function **`CreateAndWriteFiles`** creates 10 random files in the directory **C:\Temp**. Each file is named in the format **file_<random_number>.txt**.
- Each file is filled with the string **"Hello, World!\n"** repeated 100 times.
- The function **`MoveFilesToNewDirectory`** moves the files created by **`CreateAndWriteFiles`** from the directory **C:\Temp** to **C:\Temp2**.
- The function **`EnumerateProcesses`** lists running processes on the system 10 times. For each process, it outputs its ID and name.

Our goal is to answer why this code is not working as expected by debugging it.

In order to demonstrate this demo. Please follow the following steps: 

- Create the following 2 folders:
  1. **C:\Temp**
  2. **C:\Temp2**

Compile the following code and run it:

```c
#include <windows.h>
#include <iostream>
#include <string>
#include <random>
#include <tlhelp32.h>
#include <vector>

CRITICAL_SECTION critSection;  // Critical section for file creation and moving
CRITICAL_SECTION consoleCritSection;  // Critical section for console output
std::vector<std::wstring> createdFiles;  // Stores the names of the created files

DWORD WINAPI CreateAndWriteFiles(LPVOID lpParam) {
    std::random_device rd;
    std::mt19937 gen(rd());

    EnterCriticalSection(&critSection);

    for (int i = 0; i < 10; i++) {
        std::wstring fileName = L"C:\\Temp\\file_" + std::to_wstring(gen()) + L".txt";
        HANDLE hFile = CreateFile(
            fileName.c_str(),
            GENERIC_WRITE,
            0,
            NULL,
            CREATE_ALWAYS,
            FILE_ATTRIBUTE_NORMAL,
            NULL
        );

        if (hFile == INVALID_HANDLE_VALUE) {
            EnterCriticalSection(&consoleCritSection);
            std::wcerr << L"CreateFile failed for " << fileName << L". Error: " << GetLastError() << std::endl;
            LeaveCriticalSection(&consoleCritSection);
            continue;
        }

        for (int j = 0; j < 100; j++) {
            DWORD bytesWritten;
            std::string data = "Hello, World!\n";
            if (!WriteFile(
                hFile,
                data.c_str(),
                data.size(),
                &bytesWritten,
                NULL
            )) {
                EnterCriticalSection(&consoleCritSection);
                std::wcerr << L"WriteFile failed for " << fileName << L". Error: " << GetLastError() << std::endl;
                LeaveCriticalSection(&consoleCritSection);
                break;
            }
        }

        CloseHandle(hFile);
        createdFiles.push_back(fileName);

        EnterCriticalSection(&consoleCritSection);
        std::wcout << L"Created and wrote to file: " << fileName << std::endl;
        LeaveCriticalSection(&consoleCritSection);
    }

    // Exit this thread without releasing the lock
    ExitThread(0);
    return 0;  // This won't be reached
}


DWORD WINAPI MoveFilesToNewDirectory(LPVOID lpParam) {
    EnterCriticalSection(&critSection);
    EnterCriticalSection(&consoleCritSection);

    for (const auto& source : createdFiles) {
        std::wstring destination = L"C:\\Temp2\\" + source.substr(source.find_last_of(L"\\") + 1);
        if (!MoveFile(source.c_str(), destination.c_str())) {
            std::wcerr << L"MoveFile failed for " << source << L" to " << destination << L". Error: " << GetLastError() << std::endl;
        }
        else {
            std::wcout << L"Moved file from " << source << L" to " << destination << std::endl;
        }
    }

    LeaveCriticalSection(&consoleCritSection);
    LeaveCriticalSection(&critSection);
    return 0;
}

DWORD WINAPI EnumerateProcesses(LPVOID lpParam) {
    for (int j = 0; j < 10; j++) {
        HANDLE hSnapshot = CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, 0);
        if (hSnapshot == INVALID_HANDLE_VALUE) {
            std::wcerr << L"CreateToolhelp32Snapshot failed" << std::endl;
            return 1;
        }

        PROCESSENTRY32W pe32;
        pe32.dwSize = sizeof(pe32);

        if (!Process32FirstW(hSnapshot, &pe32)) {
            std::wcerr << L"Process32First failed" << std::endl;
            CloseHandle(hSnapshot);
            return 1;
        }

        do {
            std::wcout << L"Process ID: " << pe32.th32ProcessID << ", Process Name: " << pe32.szExeFile << std::endl;
        } while (Process32NextW(hSnapshot, &pe32));

        CloseHandle(hSnapshot);
    }

    return 0;
}

int main() {
    DWORD threadID;
    HANDLE hThreads[3];

    LARGE_INTEGER frequency;
    QueryPerformanceFrequency(&frequency);

    LARGE_INTEGER start;
    QueryPerformanceCounter(&start);

    InitializeCriticalSection(&critSection);
    InitializeCriticalSection(&consoleCritSection);

    hThreads[0] = CreateThread(NULL, 0, CreateAndWriteFiles, NULL, 0, &threadID);
    WaitForSingleObject(hThreads[0], INFINITE); // Wait for the file creation thread to finish

    hThreads[1] = CreateThread(NULL, 0, MoveFilesToNewDirectory, NULL, 0, &threadID);
    hThreads[2] = CreateThread(NULL, 0, EnumerateProcesses, NULL, 0, &threadID);

    WaitForMultipleObjects(3, hThreads, TRUE, INFINITE);

    for (int i = 0; i < 3; i++) {
        CloseHandle(hThreads[i]);
    }

    DeleteCriticalSection(&critSection);
    DeleteCriticalSection(&consoleCritSection);

    LARGE_INTEGER end;
    QueryPerformanceCounter(&end);

    double time = static_cast<double>(end.QuadPart - start.QuadPart) / frequency.QuadPart;

    std::wcout << L"Total execution time: " << time << L" seconds" << std::endl;

    return 0;
}
```

The **`CreateAndWriteFiles`** function acquires the **`critSection`** lock with **`EnterCriticalSection(&critSection);`**. After creating and writing to some files, the function calls **`ExitThread`** to exit the thread. The problem is that the function exits without releasing the **`critSection`** lock, which it had acquired earlier. As a result, this lock becomes orphaned.

The **`MoveFilesToNewDirectory`** function tries to acquire the **`critSection`** lock as well. Since the lock has been orphaned by the **`CreateAndWriteFiles`** function, the **`MoveFilesToNewDirectory`** function will be blocked indefinitely when trying to acquire the **`critSection`** lock, and it will never execute its code.

Run the program until you see that is hanging:

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/f7f926ad-b09a-49c3-961c-1f3f4658508c)

Open **Process Explorer** and go to our program. We are able to see that the wait reason is **WrAlertByThreadId** which is generated when a thread is waiting for things like Critical Sections, Conditional Variables, or Slim Reader/Writer locks.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/02c11eca-775d-4921-9d92-7bc9649bc15b)

Let's now take a full memory dump of our program that is currently hanging.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/55fd0fc3-d7df-4c9b-b501-2c8131ddc5cb)

# WinDbg Walk Through (1) - Analyzing Potential Orphaned Lock

In the previous section, we took a memory dump of our program. Load the .dmp file in WinDbg, so we can analyze it further.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/8df60569-602d-42b7-a712-d666a8c31b43)

The **!ntsdexts.locks** extension displays a list of critical sections associated with the current process. If the **-v** option is used, all critical sections are displayed.

```
0:000> !locks

CritSec Wait2!critSection+0 at 00007ff605dd5af8
WaiterWoken        No
LockCount          1
RecursionCount     1
OwningThread       10b8
EntryCount         0
ContentionCount    1
*** Locked

Scanned 6 critical sections
```

Let's load the **MEX** extension:

```
0:000> .load mex
Mex External 3.0.0.7172 Loaded!
```

Let's now dump the (locked) critical sections within the current process by using the **!mex.cs** command. The output that we will be seeing contains information about a critical section in a process.

As we can see in the results, **MEX** has a built-in rule that can detect whether a thread may possibly be orphaned the critical section has been corrupted. It says that the thread ID **10b8** is not valid for the current process. *"10b8 - Warning!!! ThreadID is not valid for the current process. Possible orphaned or corrupt critical section"*

```
0:000> !mex.cs
Name              Critical Section Debug Info       State  Lock Count Waiter Woken                                                                                                  Owning Thread Recursion Count Event Handle Spin Count
================= ================ ================ ====== ========== ============ ============================================================================================================== =============== ============ ==========
Wait2!critSection 00007ff605dd5af8 00007ffa0f9d3220 Locked          1 No           10b8 - Warning!!! ThreadID is not valid for the current process. Possible orphaned or corrupt critical section               1 7ff605dd5b10   33556432

Count: 1
```

Let's analyze this further by running the **!mex.lt** command to display all the threads of the current process. We will notice that there is no thread ID **10b8** in the displayed output.

```
0:000> !mex.lt
 # DbgID ThdID Wait Function                       User Kernel Info     TEB              Create Time
== ===== ===== =================================== ==== ====== ======== ================ ==========================
->     0   700 Wait2!main+0xdc                        0      0 Event... 0000003ba316b000 08/07/2023 08:21:05.177 AM
       1  27b4 ntdll!NtWaitForAlertByThreadId+0x14    0      0          0000003ba316f000 08/07/2023 08:21:05.183 AM
```

Thread ID **27b4** contains the function **WrAlertByThreadId** which is generated when a thread is waiting for things like Critical Sections, Conditional Variables, or Slim Reader/Writer locks. 

Let's examine the call stack of this thread:

```
0:001> !mex.t -t 0x27b4
DbgID ThreadID       User Kernel Create Time (UTC)
1     27b4 (0n10164)    0      0 08/07/2023 08:21:05.183 AM

# Child-SP         Return           Call Site                                     Source
0 0000003ba34ff9c8 00007ffa0f883bf3 ntdll!NtWaitForAlertByThreadId+0x14           
1 0000003ba34ff9d0 00007ffa0f8718d4 ntdll!RtlpWaitOnCriticalSection+0x1e3         
2 0000003ba34ffae0 00007ffa0f8716c2 ntdll!RtlpEnterCriticalSectionContended+0x204 
3 0000003ba34ffb60 00007ff605d9273f ntdll!RtlEnterCriticalSection+0x42            
4 0000003ba34ffb90 00007ffa0e7426ad Wait2!MoveFilesToNewDirectory+0x4f            C:\Users\User\source\repos\Wait2\Wait2\Wait2.cpp @ 68
5 0000003ba34ffd40 00007ffa0f8aaa68 kernel32!BaseThreadInitThunk+0x1d             
6 0000003ba34ffd70 0000000000000000 ntdll!RtlUserThreadStart+0x28
```  

It seems like the function **`MoveFilesToNewDirectory`** in the **Wait2** program is trying to enter a critical section but is blocked because another thread is currently holding that critical section. If the critical section is already owned, the requesting thread will be blocked and placed in a wait state. **RtlpWaitForCriticalSection** handles this waiting process. 

The memory dump is useful, but it's not telling us the root cause of why the requested thread is getting blocked. Since we cannot examine the call stack of the thread that became invalid.

# WinDbg Walk Through (1) - Time Travel Debugging

Time Travel Debugging is a tool that allows you to capture a trace of your process as it executes and then replay it later both forwards and backwards. Time Travel Debugging (TTD) can help you debug issues easier by letting you "rewind" your debugger session, instead of having to reproduce the issue until you find the bug.

TTD allows you to go back in time to better understand the conditions that lead up to the bug and replay it multiple times to learn how best to fix the problem. TTD can have advantages over crash dump files, which often miss the state and execution path that led to the ultimate failure.

Since we can reproduce this issue. Let's use TTD to examine it.

1. Open CMD as an administrator and type the following command:

```
tttracer -out "C:\Traces" C:\Users\User\source\repos\Wait2\x64\Release\Wait2.exe
```

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/d3a2806f-fdf0-4dc1-a18a-f0e2d140ac29)

2. Here we are able to see that our full trace has completed and dumped in the **C:\Traces** folder.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/3f4f02c7-d233-4939-97e7-dbb326910433)

3. The **.RUN** file is the recording file of the process which we can load in WinDbg to diagnose the issue.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/6878a095-4fbe-4a98-a21d-ab0639bc2ed5)

If the program is still hanging while recording. Open CMD again and run the following command:

```
tttracer -stop <PID of program>
```

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/23ef677c-05c7-4ab9-84ef-07b40455c7cc)



# WinDbg Walk Through (1) - Analyzing Trace File

Load the **.RUN** file in WinDbg:

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/d093956b-9f25-4bfd-b227-4129ae537076)

TTD allows us to use LINQ queries to diagnose the code that has been captured in a time travel trace. Since we know from the memory dump that the issue has something to do with an invalid thread. Let's take a step back and understand the theory again of an orphaned lock.

*"An Orphan Lock refers to the situation where a thread acquires a lock (or enters a critical section) and then **terminates** or **exits** without releasing the lock."*

With this information, let's take a look whether the **RtlExitUserThread** function has been called within our trace file. The function **RtlExitUserThread** is used to terminate a user-mode thread. It's essentially the lower-level mechanism behind functions like **ExitThread** in the Windows API.

``` 
0:000> dx -g @$cursession.TTD.Calls("ntdll!RtlExitUserThread").OrderByDescending(o => o.TimeStart)
================================================================================================================================================================================================================================================================================================
=          = (+) EventType = (+) ThreadId = (+) UniqueThreadId = (+) TimeStart = (+) TimeEnd     = (+) Function               = (+) FunctionAddress = (+) ReturnAddress = (+) ReturnValue = (+) Parameters = (+) SystemTimeStart                     = (+) SystemTimeEnd                       =
================================================================================================================================================================================================================================================================================================
= [0x1]    - 0x0           - 0x3760       - 0x7                - BDACF:1C      - Max Position    - ntdll!RtlExitUserThread    - 0x7ff8fb0eaaa0      - 0x7ff8fad926b6    - 0x0             - {...}          - Tuesday, August 8, 2023 12:12:44.498    - Tuesday, August 8, 2023 12:12:44.498    =
= [0x0]    - 0x0           - 0x3754       - 0x3                - 2C36:52       - Max Position    - ntdll!RtlExitUserThread    - 0x7ff8fb0eaaa0      - 0x7ff697862727    - 0x0             - {...}          - Tuesday, August 8, 2023 12:12:22.201    - Tuesday, August 8, 2023 12:12:44.498    =
================================================================================================================================================================================================================================================================================================
``` 

In the memory dump, we were able to identify that a critical section was locked. However, we couldn't exame call stack of the thread because it was invalid. What if we can do this with TTD though?

Let's try to dump the (locked) critical section within this trace file by using the **!mex.cs** command.

``` 
0:000> !mex.cs
Mex External 3.0.0.7172 Loaded!
No locked critical sections found
``` 

In the current position that we're in, it doesn't show us that a critical section was locked. What if we jump for example to **10%** (I'm just picking a random percentage) of our trace? We already know that there is a (potential) orphaned lock based out of the memory dump though, so we should see this as well in our trace file.

``` 
0:000> !tt 10
Setting position to 10% into the trace
Setting position: 12F89:0
ModLoad: 00007ff8`fa220000 00007ff8`fa2c7000   C:\Windows\System32\msvcrt.dll
ModLoad: 00007ff8`fae90000 00007ff8`faf34000   C:\Windows\System32\sechost.dll
ModLoad: 00007ff8`fa5d0000 00007ff8`fa6e7000   C:\Windows\System32\RPCRT4.dll
ModLoad: 00007ff8`f9030000 00007ff8`f90de000   C:\Windows\System32\advapi32.dll
ModLoad: 00007ff8`f7c00000 00007ff8`f7c0c000   C:\Windows\SYSTEM32\CRYPTBASE.DLL
ModLoad: 00007ff8`f8be0000 00007ff8`f8c5a000   C:\Windows\System32\bcryptPrimitives.dll
(2a4c.3760): Break instruction exception - code 80000003 (first/second chance not available)
Time Travel Position: 12F89:0
ntdll!RtlEnterCriticalSection+0xd:
00007ff8`fb0b168d f00fba710800    lock btr dword ptr [rcx+8],0 ds:000001f7`a0559d70=ffffffff
```

Let's now run **!mex.cs** command again to see if there are locked critical sections and we can see that thread ID **3754** was invalid.

```
0:004> !mex.cs
Name              Critical Section Debug Info       State  Lock Count Waiter Woken                                                                                                  Owning Thread Recursion Count Event Handle Spin Count
================= ================ ================ ====== ========== ============ ============================================================================================================== =============== ============ ==========
Wait2!critSection 00007ff6978a5af8 00007ff8fb213220 Locked          1 No           3754 - Warning!!! ThreadID is not valid for the current process. Possible orphaned or corrupt critical section               1 7ff6978a5b10   33556432

Count: 1
```

We can double check this by looking at the **RTL_CRITICAL_SECTION** structure:

```
0:004> !mex.ddt ntdll!_RTL_CRITICAL_SECTION 00007ff6978a5af8

dt ntdll!_RTL_CRITICAL_SECTION 00007ff6978a5af8 () Recursive: [ -r1 -r2 -r ] Verbose Normal dt
==================================================================================
   +0x000 DebugInfo            : 0x00007ff8`fb213220 _RTL_CRITICAL_SECTION_DEBUG
   +0x008 LockCount            : 0n-6
   +0x00c RecursionCount       : 0n1
   +0x010 OwningThread         : 0x00000000`00003754 Void  [thread]
   +0x018 LockSemaphore        : 0xffffffff`ffffffff Void  [handle]
   +0x020 SpinCount            : 0x20007d0 (0n33556432)
```

The **OwningThread** field in the **_RTL_CRITICAL_SECTION** structure represents the **thread ID** of the thread that currently owns or holds the critical section. In this example, it happens to be **3754**.

Wait a minute... Didn't we saw this thread ID previously? Let's run the first LINQ query again to query for the specific API call to **`ntdll!RtlExitUserThread`**

```
0:004> dx -g @$cursession.TTD.Calls("ntdll!RtlExitUserThread").OrderByDescending(o => o.TimeStart)
================================================================================================================================================================================================================================================================================================
=          = (+) EventType = (+) ThreadId = (+) UniqueThreadId = (+) TimeStart = (+) TimeEnd     = (+) Function               = (+) FunctionAddress = (+) ReturnAddress = (+) ReturnValue = (+) Parameters = (+) SystemTimeStart                     = (+) SystemTimeEnd                       =
================================================================================================================================================================================================================================================================================================
= [0x1]    - 0x0           - 0x3760       - 0x7                - BDACF:1C      - Max Position    - ntdll!RtlExitUserThread    - 0x7ff8fb0eaaa0      - 0x7ff8fad926b6    - 0x0             - {...}          - Tuesday, August 8, 2023 12:12:44.498    - Tuesday, August 8, 2023 12:12:44.498    =
= [0x0]    - 0x0           - 0x3754       - 0x3                - 2C36:52       - Max Position    - ntdll!RtlExitUserThread    - 0x7ff8fb0eaaa0      - 0x7ff697862727    - 0x0             - {...}          - Tuesday, August 8, 2023 12:12:22.201    - Tuesday, August 8, 2023 12:12:44.498    =
================================================================================================================================================================================================================================================================================================
```

We can see this exact same **thread ID** at position **2C36:52**, so let's travel to this position.

```
0:004> !tt 2C36:52
Setting position: 2C36:52
(2a4c.3754): Break instruction exception - code 80000003 (first/second chance not available)
Time Travel Position: 2C36:52
ntdll!RtlExitUserThread:
00007ff8`fb0eaaa0 4053            push    rbx
```

Let's use the **!mex.lt** command to list all the threads that were seen at this position within the trace. We are also able to see that it is the most recent event happened at the specific time of the trace.

```
0:003> !mex.lt
 # DbgID ThdID Wait Function                            User Kernel Info     TEB
== ===== ===== ======================================== ==== ====== ======== ================
       0  2e44 Wait2!main+0x84                             0      0          000000079d135000
       1  3740 ntdll!NtWaitForWorkViaWorkerFactory+0x12    0      0          000000079d13d000
       2  3758 ntdll!NtWaitForWorkViaWorkerFactory+0x12    0      0          000000079d13f000
->     3  3754 ntdll!RtlExitUserThread                     0      0 Event... 000000079d13b000
```

We can now dump the call stack of thread ID **3754** and we are able to see that our function **`CreateAndWriteFiles`** was invoking **RtlUserExitThread**.

```
0:003> !mex.t -t 0x3754
DbgID ThreadID       User Kernel
3     3754 (0n14164)    0      0

# Child-SP         Return           Call Site                         Source
0 000000079d3fe378 00007ff697862727 ntdll!RtlExitUserThread      <---- Calling ExitThread()     
1 000000079d3fe380 00007ff8fad926ad Wait2!CreateAndWriteFiles+0x2a7   C:\Users\User\source\repos\Wait2\Wait2\Wait2.cpp @ 63
2 000000079d3ff800 00007ff8fb0eaa68 KERNEL32!BaseThreadInitThunk+0x1d 
3 000000079d3ff830 0000000000000000 ntdll!RtlUserThreadStart+0x28
```

# Code Review - What was the root cause?

The root cause of the orphan lock in this code is the use of the **`ExitThread(0)`** function in the **`CreateAndWriteFiles`** thread, which exits the thread without releasing the **`critSection`** critical section that it previously acquired using **`EnterCriticalSection(&critSection)`**. This leaves the lock orphaned, as it is never released, which will cause issues with the program, such as hangs.

- **Issue within the code:**

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/8a3982c6-ded2-4fd8-bf63-d5cc70af6087)

This code will work as expected:

 ```c
#include <windows.h>
#include <iostream>
#include <string>
#include <random>
#include <tlhelp32.h>
#include <vector>

CRITICAL_SECTION critSection;  // Critical section for file creation and moving
CRITICAL_SECTION consoleCritSection;  // Critical section for console output
std::vector<std::wstring> createdFiles;  // Stores the names of the created files

DWORD WINAPI CreateAndWriteFiles(LPVOID lpParam) {
    std::random_device rd;
    std::mt19937 gen(rd());

    EnterCriticalSection(&critSection);

    for (int i = 0; i < 10; i++) {
        std::wstring fileName = L"C:\\Temp\\file_" + std::to_wstring(gen()) + L".txt";
        HANDLE hFile = CreateFile(
            fileName.c_str(),
            GENERIC_WRITE,
            0,
            NULL,
            CREATE_ALWAYS,
            FILE_ATTRIBUTE_NORMAL,
            NULL
        );

        if (hFile == INVALID_HANDLE_VALUE) {
            EnterCriticalSection(&consoleCritSection);
            std::wcerr << L"CreateFile failed for " << fileName << L". Error: " << GetLastError() << std::endl;
            LeaveCriticalSection(&consoleCritSection);
            continue;
        }

        for (int j = 0; j < 100; j++) {
            DWORD bytesWritten;
            std::string data = "Hello, World!\n";
            if (!WriteFile(
                hFile,
                data.c_str(),
                data.size(),
                &bytesWritten,
                NULL
            )) {
                EnterCriticalSection(&consoleCritSection);
                std::wcerr << L"WriteFile failed for " << fileName << L". Error: " << GetLastError() << std::endl;
                LeaveCriticalSection(&consoleCritSection);
                break;
            }
        }

        CloseHandle(hFile);
        createdFiles.push_back(fileName);

        EnterCriticalSection(&consoleCritSection);
        std::wcout << L"Created and wrote to file: " << fileName << std::endl;
        LeaveCriticalSection(&consoleCritSection);
    }

    LeaveCriticalSection(&critSection);
    return 0; 
}


DWORD WINAPI MoveFilesToNewDirectory(LPVOID lpParam) {
    EnterCriticalSection(&critSection);
    EnterCriticalSection(&consoleCritSection);

    for (const auto& source : createdFiles) {
        std::wstring destination = L"C:\\Temp2\\" + source.substr(source.find_last_of(L"\\") + 1);
        if (!MoveFile(source.c_str(), destination.c_str())) {
            std::wcerr << L"MoveFile failed for " << source << L" to " << destination << L". Error: " << GetLastError() << std::endl;
        }
        else {
            std::wcout << L"Moved file from " << source << L" to " << destination << std::endl;
        }
    }

    LeaveCriticalSection(&consoleCritSection);
    LeaveCriticalSection(&critSection);
    return 0;
}

DWORD WINAPI EnumerateProcesses(LPVOID lpParam) {
    for (int j = 0; j < 10; j++) {
        HANDLE hSnapshot = CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, 0);
        if (hSnapshot == INVALID_HANDLE_VALUE) {
            std::wcerr << L"CreateToolhelp32Snapshot failed" << std::endl;
            return 1;
        }

        PROCESSENTRY32W pe32;
        pe32.dwSize = sizeof(pe32);

        if (!Process32FirstW(hSnapshot, &pe32)) {
            std::wcerr << L"Process32First failed" << std::endl;
            CloseHandle(hSnapshot);
            return 1;
        }

        do {
            std::wcout << L"Process ID: " << pe32.th32ProcessID << ", Process Name: " << pe32.szExeFile << std::endl;
        } while (Process32NextW(hSnapshot, &pe32));

        CloseHandle(hSnapshot);
    }

    return 0;
}

int main() {
    DWORD threadID;
    HANDLE hThreads[3];

    LARGE_INTEGER frequency;
    QueryPerformanceFrequency(&frequency);

    LARGE_INTEGER start;
    QueryPerformanceCounter(&start);

    InitializeCriticalSection(&critSection);
    InitializeCriticalSection(&consoleCritSection);

    hThreads[0] = CreateThread(NULL, 0, CreateAndWriteFiles, NULL, 0, &threadID);
    WaitForSingleObject(hThreads[0], INFINITE); // Wait for the file creation thread to finish

    hThreads[1] = CreateThread(NULL, 0, MoveFilesToNewDirectory, NULL, 0, &threadID);
    hThreads[2] = CreateThread(NULL, 0, EnumerateProcesses, NULL, 0, &threadID);

    WaitForMultipleObjects(3, hThreads, TRUE, INFINITE);

    for (int i = 0; i < 3; i++) {
        CloseHandle(hThreads[i]);
    }

    DeleteCriticalSection(&critSection);
    DeleteCriticalSection(&consoleCritSection);

    LARGE_INTEGER end;
    QueryPerformanceCounter(&end);

    double time = static_cast<double>(end.QuadPart - start.QuadPart) / frequency.QuadPart;

    std::wcout << L"Total execution time: " << time << L" seconds" << std::endl;

    return 0;
}
 ```

# Orphan Lock - Terminating Thread

In the previous example, we were talking about how **ExitThread** function was causing issues, since we're never releasing the acquired critical section. The memory dump revealed the warning:  *Warning!!! ThreadID is not valid for the current process. Possible orphaned or corrupt critical section"*.

I've simulated the same issue but with calling **TerminateThread** function. Terminating a thread this way is unsafe because it doesn't allow the thread to clean up its resources. If a thread is terminated while it holds a lock (critical section), other threads waiting for that lock will be left in a waiting state indefinitely.

Compile this code and run it:

 ```c
#include <windows.h>
#include <iostream>
#include <string>
#include <random>
#include <tlhelp32.h>
#include <vector>

CRITICAL_SECTION critSection;  // Critical section for file creation and moving
CRITICAL_SECTION consoleCritSection;  // Critical section for console output
std::vector<std::wstring> createdFiles;  // Stores the names of the created files

DWORD WINAPI CreateAndWriteFiles(LPVOID lpParam) {
    std::random_device rd;
    std::mt19937 gen(rd());

    EnterCriticalSection(&critSection);

    for (int i = 0; i < 10; i++) {
        std::wstring fileName = L"C:\\Temp\\file_" + std::to_wstring(gen()) + L".txt";
        HANDLE hFile = CreateFile(
            fileName.c_str(),
            GENERIC_WRITE,
            0,
            NULL,
            CREATE_ALWAYS,
            FILE_ATTRIBUTE_NORMAL,
            NULL
        );

        if (hFile == INVALID_HANDLE_VALUE) {
            EnterCriticalSection(&consoleCritSection);
            std::wcerr << L"CreateFile failed for " << fileName << L". Error: " << GetLastError() << std::endl;
            LeaveCriticalSection(&consoleCritSection);
            continue;
        }

        for (int j = 0; j < 100; j++) {
            DWORD bytesWritten;
            std::string data = "Hello, World!\n";
            if (!WriteFile(
                hFile,
                data.c_str(),
                data.size(),
                &bytesWritten,
                NULL
            )) {
                EnterCriticalSection(&consoleCritSection);
                std::wcerr << L"WriteFile failed for " << fileName << L". Error: " << GetLastError() << std::endl;
                LeaveCriticalSection(&consoleCritSection);
                break;
            }
        }

        CloseHandle(hFile);
        createdFiles.push_back(fileName);

        EnterCriticalSection(&consoleCritSection);
        std::wcout << L"Created and wrote to file: " << fileName << std::endl;
        LeaveCriticalSection(&consoleCritSection);
    }

    // Terminating the thread
    TerminateThread(GetCurrentThread(), 0);
    return 0;  // This won't be reached
}


DWORD WINAPI MoveFilesToNewDirectory(LPVOID lpParam) {
    EnterCriticalSection(&critSection);
    EnterCriticalSection(&consoleCritSection);

    for (const auto& source : createdFiles) {
        std::wstring destination = L"C:\\Temp2\\" + source.substr(source.find_last_of(L"\\") + 1);
        if (!MoveFile(source.c_str(), destination.c_str())) {
            std::wcerr << L"MoveFile failed for " << source << L" to " << destination << L". Error: " << GetLastError() << std::endl;
        }
        else {
            std::wcout << L"Moved file from " << source << L" to " << destination << std::endl;
        }
    }

    LeaveCriticalSection(&consoleCritSection);
    LeaveCriticalSection(&critSection);
    return 0;
}

DWORD WINAPI EnumerateProcesses(LPVOID lpParam) {
    for (int j = 0; j < 10; j++) {
        HANDLE hSnapshot = CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, 0);
        if (hSnapshot == INVALID_HANDLE_VALUE) {
            std::wcerr << L"CreateToolhelp32Snapshot failed" << std::endl;
            return 1;
        }

        PROCESSENTRY32W pe32;
        pe32.dwSize = sizeof(pe32);

        if (!Process32FirstW(hSnapshot, &pe32)) {
            std::wcerr << L"Process32First failed" << std::endl;
            CloseHandle(hSnapshot);
            return 1;
        }

        do {
            std::wcout << L"Process ID: " << pe32.th32ProcessID << ", Process Name: " << pe32.szExeFile << std::endl;
        } while (Process32NextW(hSnapshot, &pe32));

        CloseHandle(hSnapshot);
    }

    return 0;
}

int main() {
    DWORD threadID;
    HANDLE hThreads[3];

    LARGE_INTEGER frequency;
    QueryPerformanceFrequency(&frequency);

    LARGE_INTEGER start;
    QueryPerformanceCounter(&start);

    InitializeCriticalSection(&critSection);
    InitializeCriticalSection(&consoleCritSection);

    hThreads[0] = CreateThread(NULL, 0, CreateAndWriteFiles, NULL, 0, &threadID);
    WaitForSingleObject(hThreads[0], INFINITE); // Wait for the file creation thread to finish

    hThreads[1] = CreateThread(NULL, 0, MoveFilesToNewDirectory, NULL, 0, &threadID);
    hThreads[2] = CreateThread(NULL, 0, EnumerateProcesses, NULL, 0, &threadID);

    WaitForMultipleObjects(3, hThreads, TRUE, INFINITE);

    for (int i = 0; i < 3; i++) {
        CloseHandle(hThreads[i]);
    }

    DeleteCriticalSection(&critSection);
    DeleteCriticalSection(&consoleCritSection);

    LARGE_INTEGER end;
    QueryPerformanceCounter(&end);

    double time = static_cast<double>(end.QuadPart - start.QuadPart) / frequency.QuadPart;

    std::wcout << L"Total execution time: " << time << L" seconds" << std::endl;

    return 0;
}
 ```

The program is hanging as we can see here:

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/de21634a-418b-441b-970e-8c89d330fefa)

The main reason for the orphan lock in our code is the improper termination of the **`CreateAndWriteFiles`** thread using **TerminateThread**. Open **Process Explorer** and we can see the exact same wait reason which is **WrAlertByThreadId**. This happens when a thread is waiting for things like Critical Sections, Conditional Variables, or Slim Reader/Writer locks.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/f3f31c36-ccb6-4de9-8914-a541d07de70e)

Take a memory dump of the process and load it in WinDbg:

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/d1093a7a-e641-4d8d-9e20-5ad59b5731be)

# WinDbg Walk Through (2) - Analyzing Potential Orphaned Lock

Load the memory dump in WinDbg again.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/0a9ebbaf-34e4-46f8-b385-146f117ad1a8)

Run the **!mex.p** command to double check the context of the current process we're in.

 ```
0:000> !mex.p
Mex External 3.0.0.7172 Loaded!
Name      Ses PID            PEB              Mods Handle Thrd
========= === ============== ================ ==== ====== ====
wait3.exe   2 2e2c (0n11820) 000000e40bf45000   11     60    2

CommandLine: Wait3.exe
Last event: 2e2c.11f8: Break instruction exception - code 80000003 (first/second chance not available)

Show Threads: Unique Stacks    !listthreads (!lt)    ~*kv
 ```

Dump the locked critical sections via the **!mex.cs** command:

 ```
0:000> !mex.cs
Name          Critical Section Debug Info       State  Lock Count Waiter Woken                                                                                                  Owning Thread Recursion Count Event Handle Spin Count
============= ================ ================ ====== ========== ============ ============================================================================================================== =============== ============ ==========
Wait3+0x45af8 00007ff744ef5af8 00007ffc3b7b3220 Locked          1 No           3798 - Warning!!! ThreadID is not valid for the current process. Possible orphaned or corrupt critical section               1 7ff744ef5b10   33556432

Count: 1
 ```

To verify once again that the thread has been invalid:

 ```
0:001> !mex.ddt ntdll!_RTL_CRITICAL_SECTION 00007ff744ef5af8

dt ntdll!_RTL_CRITICAL_SECTION 00007ff744ef5af8 () Recursive: [ -r1 -r2 -r ] Verbose Normal dt
==================================================================================
   +0x000 DebugInfo            : 0x00007ffc`3b7b3220 _RTL_CRITICAL_SECTION_DEBUG
   +0x008 LockCount            : 0n-6
   +0x00c RecursionCount       : 0n1
   +0x010 OwningThread         : 0x00000000`00003798 Void  <--- INVALID
   +0x018 LockSemaphore        : 0xffffffff`ffffffff Void  [handle]
   +0x020 SpinCount            : 0x20007d0 (0n33556432)
 ```

Thread is invalid as we can see here:

 ```
0:001> !mex.t -t 0x00000000`00003798
Invalid thread
 ```

Let's list all the threads within this current process:

 ```
0:000> !mex.lt
 # DbgID ThdID Wait Function                       User Kernel Info     TEB              Create Time
== ===== ===== =================================== ==== ====== ======== ================ ==========================
->     0  11f8 Wait3+0x2f3a                           0      0 Event... 000000e40bf46000 08/07/2023 08:42:46.969 PM
       1  1a9c ntdll!NtWaitForAlertByThreadId+0x14    0      0          000000e40bf4c000 08/07/2023 08:42:47.030 PM
 ```

Here we can see that a thread is trying to enter a critical section, but the request is blocked. Since another thread is already holding that critical section.

 ```
0:000> !mex.t -t 0x1a9c
DbgID ThreadID      User Kernel Create Time (UTC)
1     1a9c (0n6812)    0      0 08/07/2023 08:42:47.030 PM

# Child-SP         Return           Call Site
0 000000e40c0ffa28 00007ffc3b663bf3 ntdll!NtWaitForAlertByThreadId+0x14
1 000000e40c0ffa30 00007ffc3b6518d4 ntdll!RtlpWaitOnCriticalSection+0x1e3
2 000000e40c0ffb40 00007ffc3b6516c2 ntdll!RtlpEnterCriticalSectionContended+0x204
3 000000e40c0ffbc0 00007ff744eb29a2 ntdll!RtlEnterCriticalSection+0x42
4 000000e40c0ffbf0 00007ffc3aae26ad Wait3+0x29a2
5 000000e40c0ffda0 00007ffc3b68aa68 kernel32!BaseThreadInitThunk+0x1d
6 000000e40c0ffdd0 0000000000000000 ntdll!RtlUserThreadStart+0x28
 ```

Just like the previous example, the memory dump doesn't reveal the actual root cause on how this orphaned lock has occurred. Since we cannot examine the thread that became invalid.

# WinDbg Walk Through (2) - Time Travel Debugging

Since we can reproduce this issue, let's use TTD and see if we can answer why this orphan lock has occurred.

1. Start with recording the program by running the following command:

 ```
tttracer -out "C:\Traces" C:\Users\User\source\repos\Wait3\x64\Release\Wait3.exe
 ```

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/2fc5eb68-9308-4cad-bb09-023c4bd782c1)

2. When the program starts hanging, open CMD as an administrator and stop the recording:

 ```
tttracer -stop <PID of Program>
 ```

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/4fe330bb-1cff-4c28-a419-5de632375537)

3. Let's load the trace file in WinDbg now.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/5bb5be07-85f3-4fb3-94c7-606cabf081b4)


Let's query the API calls that can be used to **exit** or **terminate** a thread:

 ```
0:000> dx -g @$cursession.TTD.Calls("ntdll!NtTerminateThread","ntdll!RtlExitUserThread").OrderBy(o => o.TimeStart)
================================================================================================================================================================================================================================================================================================
=          = (+) EventType = (+) ThreadId = (+) UniqueThreadId = (+) TimeStart = (+) TimeEnd     = (+) Function               = (+) FunctionAddress = (+) ReturnAddress = (+) ReturnValue = (+) Parameters = (+) SystemTimeStart                     = (+) SystemTimeEnd                       =
================================================================================================================================================================================================================================================================================================
= [0x0]    - 0x0           - 0x1510       - 0x3                - 2BFF:15       - Max Position    - ntdll!NtTerminateThread    - 0x7ff8fb12f7a0      - 0x7ff8f870735c    - 0x0             - {...}          - Tuesday, August 8, 2023 13:59:42.608    - Tuesday, August 8, 2023 13:59:54.607    =
= [0x1]    - 0x0           - 0xdb4        - 0x7                - A5789:1C      - Max Position    - ntdll!RtlExitUserThread    - 0x7ff8fb0eaaa0      - 0x7ff8fad926b6    - 0x0             - {...}          - Tuesday, August 8, 2023 13:59:54.607    - Tuesday, August 8, 2023 13:59:54.607    =
= [0x2]    - 0x0           - 0xdb4        - 0x7                - A57BA:B8      - Max Position    - ntdll!NtTerminateThread    - 0x7ff8fb12f7a0      - 0x7ff8fb0eaaee    - 0x0             - {...}          - Tuesday, August 8, 2023 13:59:54.607    - Tuesday, August 8, 2023 13:59:54.607    =
================================================================================================================================================================================================================================================================================================
 ```

Let's run the **!mex.cs** command to see whether there are any locked critical sections in the **current** position:

 ```
0:000> !mex.cs
Mex External 3.0.0.7172 Loaded!
No locked critical sections found
 ```

There are no results at this position, so what if we just travel **10%** within the trace? (I'm just picking a random percentage)

 ```
0:000> !tt 10
Setting position to 10% into the trace
Setting position: 108CF:0
ModLoad: 00007ff8`fa220000 00007ff8`fa2c7000   C:\Windows\System32\msvcrt.dll
ModLoad: 00007ff8`fae90000 00007ff8`faf34000   C:\Windows\System32\sechost.dll
ModLoad: 00007ff8`fa5d0000 00007ff8`fa6e7000   C:\Windows\System32\RPCRT4.dll
ModLoad: 00007ff8`f9030000 00007ff8`f90de000   C:\Windows\System32\advapi32.dll
ModLoad: 00007ff8`f7c00000 00007ff8`f7c0c000   C:\Windows\SYSTEM32\CRYPTBASE.DLL
ModLoad: 00007ff8`f8be0000 00007ff8`f8c5a000   C:\Windows\System32\bcryptPrimitives.dll
(292c.db4): Break instruction exception - code 80000003 (first/second chance not available)
Time Travel Position: 108CF:0
ntdll!NtWriteFile+0x12:
00007ff8`fb12ee52 0f05            syscall
 ```

Let's now see if there are any locked critical section within this position via **!mex.cs** command:

 ```
0:004> !mex.cs
Name          Critical Section Debug Info       State  Lock Count Waiter Woken                                                                                                  Owning Thread Recursion Count Event Handle Spin Count
============= ================ ================ ====== ========== ============ ============================================================================================================== =============== ============ ==========
Wait3+0x45af8 00007ff735915af8 00007ff8fb213220 Locked          1 No           1510 - Warning!!! ThreadID is not valid for the current process. Possible orphaned or corrupt critical section               1 7ff735915b10   33556432

Count: 1
 ```

Ah, there you go. Thread ID **1510** is the one that is not valid.

 ```
0:004> !mex.ddt ntdll!_RTL_CRITICAL_SECTION 00007ff735915af8

dt ntdll!_RTL_CRITICAL_SECTION 00007ff735915af8 () Recursive: [ -r1 -r2 -r ] Verbose Normal dt
==================================================================================
   +0x000 DebugInfo            : 0x00007ff8`fb213220 _RTL_CRITICAL_SECTION_DEBUG
   +0x008 LockCount            : 0n-6
   +0x00c RecursionCount       : 0n1
   +0x010 OwningThread         : 0x00000000`00001510 Void  <---- INVALID
   +0x018 LockSemaphore        : 0xffffffff`ffffffff Void  [handle]
   +0x020 SpinCount            : 0x20007d0 (0n33556432)
 ```

Hmm, wait a minute. We saw this **thread ID** as well when we are querying the API calls:

 ```
0:004> dx -g @$cursession.TTD.Calls("ntdll!NtTerminateThread","ntdll!RtlExitUserThread").OrderBy(o => o.TimeStart)
================================================================================================================================================================================================================================================================================================
=          = (+) EventType = (+) ThreadId = (+) UniqueThreadId = (+) TimeStart = (+) TimeEnd     = (+) Function               = (+) FunctionAddress = (+) ReturnAddress = (+) ReturnValue = (+) Parameters = (+) SystemTimeStart                     = (+) SystemTimeEnd                       =
================================================================================================================================================================================================================================================================================================
= [0x0]    - 0x0           - 0x1510       - 0x3                - 2BFF:15       - Max Position    - ntdll!NtTerminateThread    - 0x7ff8fb12f7a0      - 0x7ff8f870735c    - 0x0             - {...}          - Tuesday, August 8, 2023 13:59:42.608    - Tuesday, August 8, 2023 13:59:54.607    =
= [0x1]    - 0x0           - 0xdb4        - 0x7                - A5789:1C      - Max Position    - ntdll!RtlExitUserThread    - 0x7ff8fb0eaaa0      - 0x7ff8fad926b6    - 0x0             - {...}          - Tuesday, August 8, 2023 13:59:54.607    - Tuesday, August 8, 2023 13:59:54.607    =
= [0x2]    - 0x0           - 0xdb4        - 0x7                - A57BA:B8      - Max Position    - ntdll!NtTerminateThread    - 0x7ff8fb12f7a0      - 0x7ff8fb0eaaee    - 0x0             - {...}          - Tuesday, August 8, 2023 13:59:54.607    - Tuesday, August 8, 2023 13:59:54.607    =
================================================================================================================================================================================================================================================================================================
 ```

Ah, we can see that thread ID **1510** is at the **2BFF:15**. Let's travel to this position:

 ```
0:004> !tt 2BFF:15
Setting position: 2BFF:15
(292c.1510): Break instruction exception - code 80000003 (first/second chance not available)
Time Travel Position: 2BFF:15
ntdll!NtTerminateThread:
00007ff8`fb12f7a0 4c8bd1          mov     r10,rcx
 ```

When we are dumping the call stack of this thread, we are able to see that a call is being made to **NtTerminateThread**. As explained earlier, terminating a thread this way is unsafe because it doesn't allow the thread to clean up its resources. If a thread is terminated while it holds a lock (critical section), other threads waiting for that lock will be left in a waiting state indefinitely.

 ```
0:003> ~kb
 # RetAddr               : Args to Child                                                           : Call Site
00 00007ff8`f870735c     : 00000135`c9161040 ffffffff`fffffffe 00000027`5abfeae0 00000000`00000000 : ntdll!NtTerminateThread
01 00007ff7`358d290c     : 00000000`00000064 00000027`5abfeae0 00000000`00000114 00000000`00000114 : KERNELBASE!TerminateThread+0xdc
02 00007ff8`fad926ad     : 00000000`00000000 00000000`00000000 00000000`00000000 00000000`00000000 : Wait3+0x290c
03 00007ff8`fb0eaa68     : 00000000`00000000 00000000`00000000 00000000`00000000 00000000`00000000 : KERNEL32!BaseThreadInitThunk+0x1d
04 00000000`00000000     : 00000000`00000000 00000000`00000000 00000000`00000000 00000000`00000000 : ntdll!RtlUserThreadStart+0x28
 ```

We would never be able to capture this evidence with a memory dump, since memory dumps provide a limited, static snapshot at a specific moment, making it harder to trace the root cause of problems.
