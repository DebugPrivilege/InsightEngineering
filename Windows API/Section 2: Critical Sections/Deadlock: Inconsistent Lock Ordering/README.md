# What is a Deadlock?

Deadlocks are situations where two or more threads are unable to proceed with their execution because they are each waiting for the other to release a resource. Deadlocks typically involve a circular wait condition, where Thread A is waiting for a resource held by Thread B, and Thread B is waiting for a resource held by Thread A.

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

#define NUM_FILES 1000

CRITICAL_SECTION critSection;  // Critical section for file creation and moving
CRITICAL_SECTION consoleCritSection;  // Critical section for console output
std::vector<std::wstring> createdFiles;  // Stores the names of the created files

// This function creates files and writes to them
DWORD WINAPI CreateAndWriteFiles(LPVOID lpParam) {
    std::random_device rd;
    std::mt19937 gen(rd());

    EnterCriticalSection(&critSection);  // First acquire critSection
    Sleep(50);  // Allow some time for the other thread to run
    EnterCriticalSection(&consoleCritSection);  // Then acquire consoleCritSection

    // Create and write to NUM_FILES number of files
    for (int i = 0; i < NUM_FILES; i++) {
        // Generate a unique file name using the random number generator
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

        // If the file handle is invalid, print an error message and continue
        if (hFile == INVALID_HANDLE_VALUE) {
            std::wcerr << L"CreateFile failed for " << fileName << L". Error: " << GetLastError() << std::endl;
            continue;
        }

        // Write "Hello, World!\n" to the file 100 times
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
                std::wcerr << L"WriteFile failed for " << fileName << L". Error: " << GetLastError() << std::endl;
                break;
            }
        }

        // Close the file handle
        CloseHandle(hFile);
        createdFiles.push_back(fileName);

        std::wcout << L"Created and wrote to file: " << fileName << std::endl;
    }

    LeaveCriticalSection(&consoleCritSection);
    LeaveCriticalSection(&critSection);

    return 0;
}

// This function moves the created files to a new directory
DWORD WINAPI MoveFilesToNewDirectory(LPVOID lpParam) {
    EnterCriticalSection(&consoleCritSection);  // First acquire consoleCritSection
    Sleep(50);  // Allow some time for the other thread to run
    EnterCriticalSection(&critSection);  // Then acquire critSection

    // For each file that was created, move it to the new directory
    for (const auto& source : createdFiles) {
        std::wstring destination = L"C:\\Temp2\\" + source.substr(source.find_last_of(L"\\") + 1);
        if (!MoveFile(source.c_str(), destination.c_str())) {
            std::wcerr << L"MoveFile failed for " << source << L" to " << destination << L". Error: " << GetLastError() << std::endl;
        }
        else {
            std::wcout << L"Moved file from " << source << L" to " << destination << std::endl;
        }
    }

    LeaveCriticalSection(&critSection);
    LeaveCriticalSection(&consoleCritSection);

    return 0;
}

// This function enumerates all the running processes on the machine 10 times
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

    // Get the high-resolution performance counter frequency
    LARGE_INTEGER frequency;
    QueryPerformanceFrequency(&frequency);

    // Get the current counter value at the start of the program
    LARGE_INTEGER start;
    QueryPerformanceCounter(&start);

    InitializeCriticalSection(&critSection);
    InitializeCriticalSection(&consoleCritSection);

    hThreads[0] = CreateThread(NULL, 0, CreateAndWriteFiles, NULL, 0, &threadID);
    hThreads[1] = CreateThread(NULL, 0, MoveFilesToNewDirectory, NULL, 0, &threadID);
    hThreads[2] = CreateThread(NULL, 0, EnumerateProcesses, NULL, 0, &threadID);

    WaitForMultipleObjects(3, hThreads, TRUE, INFINITE);

    for (int i = 0; i < 3; i++) {
        CloseHandle(hThreads[i]);
    }

    DeleteCriticalSection(&critSection);
    DeleteCriticalSection(&consoleCritSection);

    // Get the current counter value at the end of the program
    LARGE_INTEGER end;
    QueryPerformanceCounter(&end);

    // Calculate the total execution time in seconds
    double time = static_cast<double>(end.QuadPart - start.QuadPart) / frequency.QuadPart;

    std::wcout << L"Total execution time: " << time << L" seconds" << std::endl;

    return 0;
}
```
Within this code, a deadlock scenario arises between the **`CreateAndWriteFiles`** and **`MoveFilesToNewDirectory`** functions due to the order in which they acquire critical sections (**`critSection`** and **`consoleCritSection`**).

When we run this code, the program will start enumerating running processes via the **`EnumerateProcesses`** function. However, the program will get stuck right in the middle. It won't create any files in **C:\Temp** and it won't move files to **C:\Temp2**.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/47c32232-64b5-45ca-88a7-4a5566b5c03d)

Open **Process Explorer** and go to our program. We are able to see that the wait reason is **WrAlertByThreadId** which is generated when a thread is waiting for things like **Critical Sections**, Conditional Variables, or Slim Reader/Writer locks. 

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/d2f27814-ccf4-44c1-b2e1-2d6758d058c8)

Let's now take a full memory dump of our program that is currently hanging.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/55fd0fc3-d7df-4c9b-b501-2c8131ddc5cb)

# WinDbg Walk Through - Displaying Critical Sections

In the previous section, we took a memory dump of our program. Load the .dmp file in WinDbg, so we can analyze it further.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/8df60569-602d-42b7-a712-d666a8c31b43)


The **!ntsdexts.locks** extension displays a list of critical sections associated with the current process. If the **-v** option is used, all critical sections are displayed.

```
0:000> !locks

CritSec Wait2!consoleCritSection+0 at 00007ff6492e5ad0
WaiterWoken        No
LockCount          1
RecursionCount     1
OwningThread       33bc
EntryCount         0
ContentionCount    1
*** Locked

CritSec Wait2!critSection+0 at 00007ff6492e5af8
WaiterWoken        No
LockCount          1
RecursionCount     1
OwningThread       2aa0
EntryCount         0
ContentionCount    1
*** Locked

Scanned 6 critical sections
```

If we know the address of the critical section we wish to display, we can use the **!critsec** extension. This displays the same collection of information as **!ntsdexts.locks**.

```
0:000> !critsec 00007ff6492e5ad0

CritSec Wait2!consoleCritSection+0 at 00007ff6492e5ad0
WaiterWoken        No
LockCount          1
RecursionCount     1
OwningThread       33bc
EntryCount         0
ContentionCount    1
*** Locked
```

The **!cs** extension can display a critical section based on its address, search an address range for critical sections, and even display the stack trace associated with each critical section.

```
0:000> !cs 00007ff6492e5ad0
-----------------------------------------
Critical section   = 0x00007ff6492e5ad0 (Wait2!consoleCritSection+0x0)
DebugInfo          = 0x00007ff8e31d3220
LOCKED
LockCount          = 0x1
WaiterWoken        = No
OwningThread       = 0x00000000000033bc
RecursionCount     = 0x1
LockSemaphore      = 0xFFFFFFFF
SpinCount          = 0x00000000020007d0
```

The **dt** (Display Type) command can be used to display the literal contents of the **RTL_CRITICAL_SECTION** structure.

```
0:000> dt RTL_CRITICAL_SECTION 00007ff6492e5af8
Wait2!RTL_CRITICAL_SECTION
   +0x000 DebugInfo        : 0x00007ff8`e31d3250 _RTL_CRITICAL_SECTION_DEBUG
   +0x008 LockCount        : 0n-6
   +0x00c RecursionCount   : 0n1
   +0x010 OwningThread     : 0x00000000`00002aa0 Void
   +0x018 LockSemaphore    : 0xffffffff`ffffffff Void
   +0x020 SpinCount        : 0x20007d0
```

The most important fields of the critical section structure are as follows:

- **LockCount:** indicates the number of times that any thread has called the **EnterCriticalSection** routine for this critical section, minus one. This field starts at **-1** for an **unlocked critical section**. Each call of **EnterCriticalSection** decrements this value; each call of **LeaveCriticalSection** increments it. For example, if LockCount is 5, this critical section is locked, one thread has acquired it, and five additional threads are waiting for this lock.
- **RecursionCount:** indicates the number of times that the owning thread has called **EnterCriticalSection** for this critical section.
- **EntryCount:** indicates the number of times that a thread other than the owning thread has called **EnterCriticalSection** for this critical section.

Let's now understand how an unlocked critical section looks like. First, let's run **!lock -v** command.

```
0:000> !locks -v

CritSec ntdll!RtlpProcessHeapsListLock+0 at 00007ff8e31d3000
LockCount          NOT LOCKED
RecursionCount     0
OwningThread       0
EntryCount         0
ContentionCount    0

CritSec +fb1802c0 at 00000192fb1802c0
LockCount          NOT LOCKED
RecursionCount     0
OwningThread       0
EntryCount         0
ContentionCount    0

CritSec +faf902c0 at 00000192faf902c0
LockCount          NOT LOCKED
RecursionCount     0
OwningThread       0
EntryCount         0
ContentionCount    0

CritSec +fb4402c0 at 00000192fb4402c0
LockCount          NOT LOCKED
RecursionCount     0
OwningThread       0
EntryCount         0
ContentionCount    0

CritSec Wait2!consoleCritSection+0 at 00007ff6492e5ad0
WaiterWoken        No
LockCount          1 
RecursionCount     1
OwningThread       33bc
EntryCount         0
ContentionCount    1
*** Locked

CritSec Wait2!critSection+0 at 00007ff6492e5af8
WaiterWoken        No
LockCount          1
RecursionCount     1
OwningThread       2aa0
EntryCount         0
ContentionCount    1
*** Locked

Scanned 6 critical sections
```
For example if the **LockCount** is **1**, it means that the critical section is currently locked by a thread, and there is one other thread waiting to acquire the lock. If it says **NOT LOCKED**, the critical section is NOT locked or owned by any thread.

Copy a memory address of an **unlocked critical section** and look it up against the **RTL_CRITICAL_SECTION** structure:

```
0:000> dt RTL_CRITICAL_SECTION 00000192fb1802c0
Wait2!RTL_CRITICAL_SECTION
   +0x000 DebugInfo        : 0x00007ff8`e31d3190 _RTL_CRITICAL_SECTION_DEBUG
   +0x008 LockCount        : 0n-1 <----- NOT LOCKED
   +0x00c RecursionCount   : 0n0
   +0x010 OwningThread     : (null) 
   +0x018 LockSemaphore    : (null) 
   +0x020 SpinCount        : 0x20007d0
```

# WinDbg - Analyzing Critical Sections

Start with loading the **Mex** extension.

```
0:000> .load mex
Mex External 3.0.0.7172 Loaded!
```
The second command we will be running is **!mex.p** that show us the **current** process context we're in.

```
0:000> !mex.p
Name      Ses PID            PEB              Mods Handle Thrd
========= === ============== ================ ==== ====== ====
wait2.exe   2 2810 (0n10256) 0000005e70daf000   10     54    6

CommandLine: Wait2.exe
Last event: 2810.2814: Break instruction exception - code 80000003 (first/second chance not available)

Show Threads: Unique Stacks    !listthreads (!lt)    ~*kv
```

Let's now dump the critical sections within the current process by using the **!mex.cs** command. The output that we will be seeing contains information about two critical sections in a process. Both of these critical sections are locked by different threads (**281c** and **2818**).

```
0:000> !mex.cs
Name                     Critical Section Debug Info       State  Lock Count Waiter Woken Owning Thread Recursion Count Event Handle Spin Count
======================== ================ ================ ====== ========== ============ ============= =============== ============ ==========
Wait2!consoleCritSection 00007ff78ab95ad0 00007ff941493220 Locked          1 No                    281c               1 7ff78ab95ae8   33556432
Wait2!critSection        00007ff78ab95af8 00007ff941493250 Locked          1 No                    2818               1 7ff78ab95b10   33556432

Count: 2
```

Let's display all the threads within the current process by using the **!mex.lt** command. There are in total 6 threads. If we look closely to all the Thread IDs in the output, we are able to identify both of the threads that are locking the two critical sections. **281c** and **2818**.

```
0:001> !mex.lt
 # DbgID ThdID Wait Function                            User Kernel Info     TEB              Create Time
== ===== ===== ======================================== ==== ====== ======== ================ ==========================
       0  2814 Wait2+0x2f2c                               0s     0s Event... 0000005e70db0000 08/06/2023 01:49:23.070 pm
->     1  2818 ntdll!ZwWaitForAlertByThreadId+0x14        0s     0s          0000005e70db2000 08/06/2023 01:49:23.081 pm
       2  281c ntdll!ZwWaitForAlertByThreadId+0x14        0s     0s          0000005e70db4000 08/06/2023 01:49:23.081 pm
       3  2824 ntdll!ZwWaitForWorkViaWorkerFactory+0x14   0s     0s          0000005e70db8000 08/06/2023 01:49:23.081 pm
       4  2828 ntdll!ZwWaitForWorkViaWorkerFactory+0x14   0s     0s          0000005e70dba000 08/06/2023 01:49:23.081 pm
       5  282c ntdll!ZwWaitForWorkViaWorkerFactory+0x14   0s     0s          0000005e70dbc000 08/06/2023 01:49:23.082 pm
```

We know that a critical section is locked by thread ID **2818**, so let's start with examing this thread first. By looking at the call stack of this thread. We may get some hints on what the problem could be. 

```
0:005> !mex.t -t 0x2818
DbgID ThreadID       User Kernel Create Time (UTC)
1     2818 (0n10264)    0      0 08/06/2023 01:49:23.081 PM

# Child-SP         Return           Call Site                                     Source
0 0000005e70ffe7f8 00007ff941343bf3 ntdll!NtWaitForAlertByThreadId+0x14           
1 0000005e70ffe800 00007ff9413318d4 ntdll!RtlpWaitOnCriticalSection+0x1e3         
2 0000005e70ffe910 00007ff9413316c2 ntdll!RtlpEnterCriticalSectionContended+0x204 
3 0000005e70ffe990 00007ff78ab5248b ntdll!RtlEnterCriticalSection+0x42            
4 0000005e70ffe9c0 00007ff93fba26ad Wait2!CreateAndWriteFiles+0xab                C:\Users\User\source\repos\Wait2\Wait2\Wait2.cpp @ 23
5 0000005e70fffe70 00007ff94136aa68 kernel32!BaseThreadInitThunk+0x1d             
6 0000005e70fffea0 0000000000000000 ntdll!RtlUserThreadStart+0x28  
```

There is a particular function of interest when we are debugging critical sections, which happens to be **ntdll!RtlpWaitOnCriticalSection**. According to the official documentation of Microsoft:

*"Some types of critical section time outs can be identified when the stack trace that shows the routine **RtlpWaitForCriticalSection** near the top of the stack. Another variety of critical section time outs is a possible deadlock application error."*

The purpose of **RtlpWaitForCriticalSection** is to manage the waiting mechanism for threads attempting to enter a critical section that is currently owned by another thread. When a thread wants to access a shared resource protected by a critical section, it must first acquire the critical section. If the critical section is already owned, the requesting thread will be blocked and placed in a wait state. **RtlpWaitForCriticalSection** handles this waiting process, ensuring that the thread remains blocked until the critical section is released by its current owner, at which point the waiting thread can proceed.

If we now have to tell a story based upon the call stack that we're seeing within thread ID **2818**. This thread is trying to enter a critical section within the **Wait2!CreateAndWriteFiles** function, but it's currently blocked because the critical section is owned by another thread. The thread will wait until it can acquire the critical section or until it's interrupted. 

```
0:001> kc
  *** Stack trace for last set context - .thread/.cxr resets it
 # Call Site
00 ntdll!NtWaitForAlertByThreadId
01 ntdll!RtlpWaitOnCriticalSection
02 ntdll!RtlpEnterCriticalSectionContended
03 ntdll!RtlEnterCriticalSection
04 Wait2!CreateAndWriteFiles
05 kernel32!BaseThreadInitThunk
06 ntdll!RtlUserThreadStart
```
Let's now examine the second thread, which happens to be **281c**. We will do the exact same thing and that is by reviewing the call stack of this thread.

```
0:001> !mex.t -t 0x281c
DbgID ThreadID       User Kernel Create Time (UTC)
2     281c (0n10268)    0      0 08/06/2023 01:49:23.081 PM

# Child-SP         Return           Call Site                                     Source
0 0000005e710ff5f8 00007ff941343bf3 ntdll!NtWaitForAlertByThreadId+0x14           
1 0000005e710ff600 00007ff9413318d4 ntdll!RtlpWaitOnCriticalSection+0x1e3         
2 0000005e710ff710 00007ff9413316c2 ntdll!RtlpEnterCriticalSectionContended+0x204 
3 0000005e710ff790 00007ff78ab529a9 ntdll!RtlEnterCriticalSection+0x42            
4 0000005e710ff7c0 00007ff93fba26ad Wait2!MoveFilesToNewDirectory+0x59            C:\Users\User\source\repos\Wait2\Wait2\Wait2.cpp @ 78
5 0000005e710ff970 00007ff94136aa68 kernel32!BaseThreadInitThunk+0x1d             
6 0000005e710ff9a0 0000000000000000 ntdll!RtlUserThreadStart+0x28
```         
This call stack tells us that thread ID **281c**, while executing the **MoveFilesToNewDirectory** function, attempted to enter a critical section. However, because the critical section was already locked by another thread, this thread had to wait. This is essentially the same as the previous one.

Let's now summarize our analysis:

Both threads, with IDs **0x2818** and **0x281c**, are stuck waiting to enter critical sections, as indicated by the presence of **`ntdll!NtWaitForAlertByThreadId`** and **`ntdll!RtlpWaitOnCriticalSection`** in their call stacks. 

1. Thread **0x2818** is blocked inside **`Wait2!CreateAndWriteFiles`** trying to acquire a critical section.
2. Thread **0x281c** is blocked inside **`Wait2!MoveFilesToNewDirectory`** for a similar reason.

Given that both are waiting and neither can progress, it suggests a classic deadlock scenario.

# Code Review - How did the issue occurred?

The deadlock issue arises from the **inconsistent** ordering of acquiring the critical sections in the **`CreateAndWriteFiles`** and **`MoveFilesToNewDirectory`** functions. 

Let's take a step back:

This code has two critical sections which happens to be **`critSection`** and **`consoleCritSection`** in the code:

- **`critSection:`** This critical section is primarily used for synchronizing file operations (creation and moving) across threads. 
- **`consoleCritSection:`** This is used for console output.  By using this critical section, the program ensures that console output from different threads doesn't interleave or overlap, leading to more readable and consistent output. 

Both of these critical sections are initialized in the **`main`** function and are used in the **`CreateAndWriteFiles`** and **`MoveFilesToNewDirectory`** functions to synchronize access to shared resources to minimize race conditions and ensure thread safety.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/7789c3fc-9d22-4813-9f59-65713625c77e)

Let's now review the **`CreateAndWriteFiles`** and **`MoveFilesToNewDirectory`** functions:

**`CreateAndWriteFiles:`**

- First, **`critSection`** is acquired.
- After a sleep of 50ms, **`consoleCritSection`** is acquired.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/5f34a62e-ccfe-4858-8f92-8f7a872754e5)

**`MoveFilesToNewDirectory:`**

- First, **`consoleCritSection`** is acquired.
- Again, after a sleep of 50ms, **`critSection`** is acquired.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/025b215f-8024-4822-b218-efc394e45ecd)

This creates a classic deadlock scenario when both functions run concurrently:

1. Suppose **`CreateAndWriteFiles`** acquires the **`critSection`** first. While it's sleeping **before** it tries to acquire **`consoleCritSection`**, the **`MoveFilesToNewDirectory`** function runs and acquires **`consoleCritSection`**.
2. Now, **`CreateAndWriteFiles`** wakes up and tries to acquire **`consoleCritSection`**, but it's already locked by **`MoveFilesToNewDirectory`**.
3. Meanwhile, **`MoveFilesToNewDirectory`** tries to acquire **`critSection`**, but it's locked by **`CreateAndWriteFiles`**.

The result is a deadlock where each thread is waiting for the other to release a lock, and neither can progress.

Here is the code fix:

```c
#include <windows.h>
#include <iostream>
#include <string>
#include <random>
#include <tlhelp32.h>
#include <vector>

#define NUM_FILES 1000

CRITICAL_SECTION critSection;  // Critical section for file creation and moving
CRITICAL_SECTION consoleCritSection;  // Critical section for console output
std::vector<std::wstring> createdFiles;  // Stores the names of the created files

// This function creates files and writes to them
DWORD WINAPI CreateAndWriteFiles(LPVOID lpParam) {
    std::random_device rd;
    std::mt19937 gen(rd());

    EnterCriticalSection(&consoleCritSection);  // First acquire consoleCritSection
    EnterCriticalSection(&critSection);  // Then acquire critSection

    // Create and write to NUM_FILES number of files
    for (int i = 0; i < NUM_FILES; i++) {
        // Generate a unique file name using the random number generator
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

        // If the file handle is invalid, print an error message and continue
        if (hFile == INVALID_HANDLE_VALUE) {
            std::wcerr << L"CreateFile failed for " << fileName << L". Error: " << GetLastError() << std::endl;
            continue;
        }

        // Write "Hello, World!\n" to the file 100 times
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
                std::wcerr << L"WriteFile failed for " << fileName << L". Error: " << GetLastError() << std::endl;
                break;
            }
        }

        // Close the file handle
        CloseHandle(hFile);
        createdFiles.push_back(fileName);

        std::wcout << L"Created and wrote to file: " << fileName << std::endl;
    }

    LeaveCriticalSection(&critSection);
    LeaveCriticalSection(&consoleCritSection);

    return 0;
}

// This function moves the created files to a new directory
DWORD WINAPI MoveFilesToNewDirectory(LPVOID lpParam) {
    EnterCriticalSection(&consoleCritSection);  // First acquire consoleCritSection
    EnterCriticalSection(&critSection);  // Then acquire critSection

    // For each file that was created, move it to the new directory
    for (const auto& source : createdFiles) {
        std::wstring destination = L"C:\\Temp2\\" + source.substr(source.find_last_of(L"\\") + 1);
        if (!MoveFile(source.c_str(), destination.c_str())) {
            std::wcerr << L"MoveFile failed for " << source << L" to " << destination << L". Error: " << GetLastError() << std::endl;
        }
        else {
            std::wcout << L"Moved file from " << source << L" to " << destination << std::endl;
        }
    }

    LeaveCriticalSection(&critSection);
    LeaveCriticalSection(&consoleCritSection);

    return 0;
}

// This function enumerates all the running processes on the machine 10 times
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

    // Get the high-resolution performance counter frequency
    LARGE_INTEGER frequency;
    QueryPerformanceFrequency(&frequency);

    // Get the current counter value at the start of the program
    LARGE_INTEGER start;
    QueryPerformanceCounter(&start);

    InitializeCriticalSection(&critSection);
    InitializeCriticalSection(&consoleCritSection);

    hThreads[0] = CreateThread(NULL, 0, CreateAndWriteFiles, NULL, 0, &threadID);
    hThreads[1] = CreateThread(NULL, 0, MoveFilesToNewDirectory, NULL, 0, &threadID);
    hThreads[2] = CreateThread(NULL, 0, EnumerateProcesses, NULL, 0, &threadID);

    WaitForMultipleObjects(3, hThreads, TRUE, INFINITE);

    for (int i = 0; i < 3; i++) {
        CloseHandle(hThreads[i]);
    }

    DeleteCriticalSection(&critSection);
    DeleteCriticalSection(&consoleCritSection);

    // Get the current counter value at the end of the program
    LARGE_INTEGER end;
    QueryPerformanceCounter(&end);

    // Calculate the total execution time in seconds
    double time = static_cast<double>(end.QuadPart - start.QuadPart) / frequency.QuadPart;

    std::wcout << L"Total execution time: " << time << L" seconds" << std::endl;

    return 0;
}
```

By consistently acquiring **`consoleCritSection`** first and then **`critSection`** in both **`CreateAndWriteFiles`** and **`MoveFilesToNewDirectory`** functions, we've removed the potential for a deadlock scenario. This ensures that even if one thread manages to acquire the first critical section, it won't block the other thread from progressing, since they are both attempting to acquire the critical sections in the same order.

# Reference

- Critical Section Time Outs: https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/critical-section-time-outs
- Displaying a Critical Section: https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/displaying-a-critical-section
