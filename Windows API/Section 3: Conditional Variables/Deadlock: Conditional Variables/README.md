# What are Deadlocks?

Deadlocks are situations where two or more threads are unable to proceed with their execution because they are each waiting for the other to release a resource. Deadlocks typically involve a circular wait condition, where Thread A is waiting for a resource held by Thread B, and Thread B is waiting for a resource held by Thread A.

Let's demonstrate this with wrongly implementing Conditional Variables.

In order to demonstrate this demo. Please follow the following steps:

- Create the following 2 folders:
  1. **C:\Temp**
  2. **C:\Temp2**

Compile and run the following code:

```c
#include <windows.h>
#include <iostream>
#include <string>
#include <random>
#include <TlHelp32.h>
#include <queue>

#define NUM_FILES 1000
#define NUM_REPEATS_PROCESSES 10

CONDITION_VARIABLE ConditionVar;
CRITICAL_SECTION CritSection;
std::queue<std::wstring> fileQueue;

DWORD WINAPI CreateAndWriteFiles(LPVOID lpParam) {
    std::random_device rd;
    std::mt19937 gen(rd());

    for (int i = 0; i < NUM_FILES; i++) {
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
            std::wcerr << L"CreateFile failed for " << fileName << L". Error: " << GetLastError() << std::endl;
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
                std::wcerr << L"WriteFile failed for " << fileName << L". Error: " << GetLastError() << std::endl;
                break;
            }
        }

        if (!CloseHandle(hFile)) {
            std::wcerr << L"CloseHandle failed for " << fileName << L". Error: " << GetLastError() << std::endl;
        }

        std::wcout << L"Created and wrote to file: " << fileName << std::endl;

        EnterCriticalSection(&CritSection);
        fileQueue.push(fileName);
        // Signal the condition variable only if more than NUM_FILES files are in the queue.
        // This condition can never be met, leading to a deadlock.
        if (fileQueue.size() > NUM_FILES) {
            WakeConditionVariable(&ConditionVar);
        }
        LeaveCriticalSection(&CritSection);
    }

    return 0;
}

DWORD WINAPI MoveFilesToNewDirectory(LPVOID lpParam) {
    for (int i = 0; i < NUM_FILES; i++) {
        EnterCriticalSection(&CritSection);
        // Wait indefinitely for the condition variable to be signaled.
        // Due to the logic in CreateAndWriteFiles, it will never be signaled, leading to a deadlock.
        while (fileQueue.empty()) {
            SleepConditionVariableCS(&ConditionVar, &CritSection, INFINITE);
        }

        std::wstring source = fileQueue.front();
        fileQueue.pop();
        LeaveCriticalSection(&CritSection);

        std::wstring destination = L"C:\\Temp2\\" + source.substr(source.find_last_of(L"\\") + 1);
        if (!MoveFile(source.c_str(), destination.c_str())) {
            std::wcerr << L"MoveFile failed for " << source << L" to " << destination << L". Error: " << GetLastError() << std::endl;
        }
        else {
            std::wcout << L"Moved file from " << source << L" to " << destination << std::endl;
        }
    }

    return 0;
}

DWORD WINAPI EnumerateProcesses(LPVOID lpParam) {
    for (int j = 0; j < NUM_REPEATS_PROCESSES; j++) {
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
    InitializeConditionVariable(&ConditionVar);
    InitializeCriticalSection(&CritSection);

    LARGE_INTEGER frequency;
    QueryPerformanceFrequency(&frequency);

    LARGE_INTEGER start;
    QueryPerformanceCounter(&start);

    DWORD threadID;
    HANDLE hThreads[3];

    hThreads[0] = CreateThread(NULL, 0, CreateAndWriteFiles, NULL, 0, &threadID);
    if (hThreads[0] == NULL) {
        std::cerr << "Failed to create CreateAndWriteFiles thread. Error: " << GetLastError() << std::endl;
        return 1;
    }

    hThreads[1] = CreateThread(NULL, 0, MoveFilesToNewDirectory, NULL, 0, &threadID);
    if (hThreads[1] == NULL) {
        std::cerr << "Failed to create MoveFilesToNewDirectory thread. Error: " << GetLastError() << std::endl;
        return 1;
    }

    hThreads[2] = CreateThread(NULL, 0, EnumerateProcesses, NULL, 0, &threadID);
    if (hThreads[2] == NULL) {
        std::cerr << "Failed to create EnumerateProcesses thread. Error: " << GetLastError() << std::endl;
        return 1;
    }

    if (WaitForMultipleObjects(3, hThreads, TRUE, INFINITE) == WAIT_FAILED) {
        std::cerr << "WaitForMultipleObjects failed. Error: " << GetLastError() << std::endl;
    }

    for (int i = 0; i < 3; i++) {
        if (!CloseHandle(hThreads[i])) {
            std::cerr << "CloseHandle failed for thread " << i << ". Error: " << GetLastError() << std::endl;
        }
    }

    LARGE_INTEGER end;
    QueryPerformanceCounter(&end);
    double time = static_cast<double>(end.QuadPart - start.QuadPart) / frequency.QuadPart;
    std::wcout << L"Total execution time: " << time << L" seconds" << std::endl;

    return 0;
}
```

The deadlock arises due to the misuse of conditional variables: the thread creating filenames doesn't signal the condition because of a condition that's never met, while the thread moving files waits indefinitely for this signal to process the filenames.

The program starts hanging as we can see here:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/7236ea46-e966-463b-9c9f-fefeaa2e39fa)


Here we can see that a thread is waiting on something like a conditional variable, critical section or a SRW lock. This can be recognized due to the wait reason **WrAlertByThreadId**.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/f3c6e1a4-b1a5-40ed-a026-3ee0445dc46f)


Let's take a full dump of this program that is currently hanging:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/08a7e516-e611-4dbc-b241-3cdfca2039bb)


# WinDbg Walk Through - Analyzing Memory Dump

Start loading the memory dump in WinDbg:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/590a7698-eff2-4efc-b42b-d76298ff26a6)


Load the **MEX** extension:

```
0:000> .load mex
Mex External 3.0.0.7172 Loaded!
```

Run the **!mex.p** command to get the current process:

```
0:000> !mex.p
Name     Ses PID            PEB              Mods Handle Thrd
======== === ============== ================ ==== ====== ====
test.exe   1 2dc0 (0n11712) 000000421024c000   10     54    2

CommandLine: Test.exe
Last event: 2dc0.2dc4: Break instruction exception - code 80000003 (first/second chance not available)

Show Threads: Unique Stacks    !listthreads (!lt)    ~*kv
```

We know that the wait reason is **WrAlertByThreadId** and the **MEX** extension allows us to filter on specific wait reasons. For this demo, it doesn't really makes sense to use this command, anyways. Since we only have two threads, but it's good to be aware of this command when we are dealing with kernel memory dumps for instance.

```
0:000> !mex.lt -wr WrAlertByThreadId
 # DbgID ThdID Wait Function                       User Kernel Info     TEB              Create Time
== ===== ===== =================================== ==== ====== ======== ================ ==========================
->     0  2dc4 Test!main+0x152                        0      0 Event... 000000421024d000 08/09/2023 06:10:12.776 AM
       1  2dd0 ntdll!NtWaitForAlertByThreadId+0x14    0      0          0000004210251000 08/09/2023 06:10:12.793 AM
```

Let's examine the call stack of the thread ID **2dd0**.

```
0:000> !mex.t -t 0x2dd0
DbgID ThreadID       User Kernel Create Time (UTC)
1     2dd0 (0n11728)    0      0 08/09/2023 06:10:12.793 AM

# Child-SP         Return           Call Site                                Source
0 00000042106ff4b8 00007ff91c358d44 ntdll!NtWaitForAlertByThreadId+0x14      
1 00000042106ff4c0 00007ff919d62239 ntdll!RtlSleepConditionVariableCS+0x104  <--- Function is used to put a thread to sleep until a condition variable is signaled
2 00000042106ff540 00007ff63a172eba KERNELBASE!SleepConditionVariableCS+0x29 
3 00000042106ff570 00007ff91b7826ad Test!MoveFilesToNewDirectory+0x8a        C:\Users\User\source\repos\Test\Test\Test.cpp @ 75
4 00000042106ff740 00007ff91c34aa68 kernel32!BaseThreadInitThunk+0x1d        
5 00000042106ff770 0000000000000000 ntdll!RtlUserThreadStart+0x28
```  

Let's pause for a second and start interpreting the call stack that we're seeing within this thread. The thread (with ID **11728**) is currently blocked, waiting on a condition variable inside the **`MoveFilesToNewDirectory`** function.

This **RtlSleepConditionVariableCS** function is used to put a thread to sleep until a condition variable is signaled. It's specifically designed for threads that are using critical sections (hence the CS at the end of the function name) for synchronization.

When a thread wants to wait on a condition (typically because some resource it needs is not available), it will call this function, passing in a reference to a condition variable and a critical section. The thread will then be put to sleep. When another thread signals the condition variable (indicating that the resource is now available or the condition the waiting thread was waiting for has been met), the waiting thread will wake up and attempt to continue its execution.

Since we have the source code, we can navigate to line of code and gather more information:

```  
3 00000042106ff570 00007ff91b7826ad Test!MoveFilesToNewDirectory+0x8a        C:\Users\User\source\repos\Test\Test\Test.cpp @ 75
```

This line (@ **75**) is where the **`MoveFilesToNewDirectory`** thread is waiting for the **`ConditionVar`** condition variable to be signaled. If the condition variable is not signaled (for instance, if there's a logic error and the **WakeConditionVariable** function isn't being called correctly), the thread will wait indefinitely due to the **INFINITE** timeout, which could lead to a deadlock scenario.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/304cc661-1b33-4e70-b9cd-1e46302f8b59)




Let's review the call stack of the second thread as well:

```  
0:001> !mex.t -t 0x2dc4
DbgID ThreadID       User Kernel Create Time (UTC)
0     2dc4 (0n11716)    0      0 08/09/2023 06:10:12.776 AM

# Child-SP         Return           Call Site                                Source
0 00000042104ff9e8 00007ff919d4f499 ntdll!NtWaitForMultipleObjects+0x14      
1 00000042104ff9f0 00007ff919d4f39e KERNELBASE!WaitForMultipleObjectsEx+0xe9 
2 00000042104ffcd0 00007ff63a1735b2 KERNELBASE!WaitForMultipleObjects+0xe    
3 00000042104ffd10 00007ff63a17fc30 Test!main+0x152                          C:\Users\User\source\repos\Test\Test\Test.cpp @ 153
4 (Inline)         ---------------- Test!invoke_main+0x22                    D:\a\_work\1\s\src\vctools\crt\vcstartup\src\startup\exe_common.inl @ 78
5 00000042104ffda0 00007ff91b7826ad Test!__scrt_common_main_seh+0x10c        D:\a\_work\1\s\src\vctools\crt\vcstartup\src\startup\exe_common.inl @ 288
6 00000042104ffde0 00007ff91c34aa68 kernel32!BaseThreadInitThunk+0x1d        
7 00000042104ffe10 0000000000000000 ntdll!RtlUserThreadStart+0x28
```

The function **NtWaitForMultipleObjects** is a system call that waits for one or more synchronization objects to become signaled. The **`main`** function within this call stack is waiting for multiple threads to complete their execution using a function like **WaitForMultipleObjects**.

Since we have the source code of our program, let's navigate to line **153** and gather more information.

```
3 00000042104ffd10 00007ff63a17fc30 Test!main+0x152                          C:\Users\User\source\repos\Test\Test\Test.cpp @ 153
```

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/347dab28-64a2-44d3-a894-12f74b08bff4)



The call to **WaitForMultipleObjects** at line **153** is waiting for all three threads to complete their execution. If any of these threads becomes deadlocked or is delayed indefinitely, the **`main`** thread will remain blocked at this line.

# Code Review - How did the issue occurred?

The macro **`NUM_FILES`** is set to **1000**. The loop in the **`CreateAndWriteFiles`** function ensures that exactly **1000** file names are added to the **`fileQueue`**. Therefore, the size of **`fileQueue`** will never exceed 1000 during the program's execution.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/828514dd-0004-4a8e-b880-292f28595c55)


The condition **`if (fileQueue.size() > NUM_FILES)`** checks if the size of **`fileQueue`** is greater than 1000. But, given the program's logic, this condition will never be true. As a result, the code inside this conditional block, specifically **`WakeConditionVariable(&ConditionVar)`**, will never be executed.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/595bde91-d51a-4098-a7f2-8ef6ce17d7e9)


Since the **WakeConditionVariable** function is what signals the condition variable to wake up other waiting threads, the failure to execute this function means the waiting threads will never be woken up, leading to a deadlock.

Fix:

```c
#include <windows.h>
#include <iostream>
#include <string>
#include <random>
#include <TlHelp32.h>
#include <queue>

#define NUM_FILES 1000
#define NUM_REPEATS_PROCESSES 10

CONDITION_VARIABLE ConditionVar;
CRITICAL_SECTION CritSection;
std::queue<std::wstring> fileQueue;

DWORD WINAPI CreateAndWriteFiles(LPVOID lpParam) {
    std::random_device rd;
    std::mt19937 gen(rd());

    for (int i = 0; i < NUM_FILES; i++) {
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
            std::wcerr << L"CreateFile failed for " << fileName << L". Error: " << GetLastError() << std::endl;
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
                std::wcerr << L"WriteFile failed for " << fileName << L". Error: " << GetLastError() << std::endl;
                break;
            }
        }

        if (!CloseHandle(hFile)) {
            std::wcerr << L"CloseHandle failed for " << fileName << L". Error: " << GetLastError() << std::endl;
        }

        std::wcout << L"Created and wrote to file: " << fileName << std::endl;

        EnterCriticalSection(&CritSection);
        fileQueue.push(fileName);
        // FIX: Signal the condition variable every time a file is added to the queue.
        WakeConditionVariable(&ConditionVar);
        LeaveCriticalSection(&CritSection);
    }

    return 0;
}

DWORD WINAPI MoveFilesToNewDirectory(LPVOID lpParam) {
    for (int i = 0; i < NUM_FILES; i++) {
        EnterCriticalSection(&CritSection);
        while (fileQueue.empty()) {
            SleepConditionVariableCS(&ConditionVar, &CritSection, INFINITE);
        }

        std::wstring source = fileQueue.front();
        fileQueue.pop();
        LeaveCriticalSection(&CritSection);

        std::wstring destination = L"C:\\Temp2\\" + source.substr(source.find_last_of(L"\\") + 1);
        if (!MoveFile(source.c_str(), destination.c_str())) {
            std::wcerr << L"MoveFile failed for " << source << L" to " << destination << L". Error: " << GetLastError() << std::endl;
        }
        else {
            std::wcout << L"Moved file from " << source << L" to " << destination << std::endl;
        }
    }

    return 0;
}

DWORD WINAPI EnumerateProcesses(LPVOID lpParam) {
    for (int j = 0; j < NUM_REPEATS_PROCESSES; j++) {
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
    InitializeConditionVariable(&ConditionVar);
    InitializeCriticalSection(&CritSection);

    LARGE_INTEGER frequency;
    QueryPerformanceFrequency(&frequency);

    LARGE_INTEGER start;
    QueryPerformanceCounter(&start);

    DWORD threadID;
    HANDLE hThreads[3];

    hThreads[0] = CreateThread(NULL, 0, CreateAndWriteFiles, NULL, 0, &threadID);
    if (hThreads[0] == NULL) {
        std::cerr << "Failed to create CreateAndWriteFiles thread. Error: " << GetLastError() << std::endl;
        return 1;
    }

    hThreads[1] = CreateThread(NULL, 0, MoveFilesToNewDirectory, NULL, 0, &threadID);
    if (hThreads[1] == NULL) {
        std::cerr << "Failed to create MoveFilesToNewDirectory thread. Error: " << GetLastError() << std::endl;
        return 1;
    }

    hThreads[2] = CreateThread(NULL, 0, EnumerateProcesses, NULL, 0, &threadID);
    if (hThreads[2] == NULL) {
        std::cerr << "Failed to create EnumerateProcesses thread. Error: " << GetLastError() << std::endl;
        return 1;
    }

    if (WaitForMultipleObjects(3, hThreads, TRUE, INFINITE) == WAIT_FAILED) {
        std::cerr << "WaitForMultipleObjects failed. Error: " << GetLastError() << std::endl;
    }

    for (int i = 0; i < 3; i++) {
        if (!CloseHandle(hThreads[i])) {
            std::cerr << "CloseHandle failed for thread " << i << ". Error: " << GetLastError() << std::endl;
        }
    }

    LARGE_INTEGER end;
    QueryPerformanceCounter(&end);
    double time = static_cast<double>(end.QuadPart - start.QuadPart) / frequency.QuadPart;
    std::wcout << L"Total execution time: " << time << L" seconds" << std::endl;

    return 0;
}
```

# Code Sample (2) - Deadlock: Waiting Unnecessarily and Indefinitely

Here is another code snippet that contains a deadlock.

Compile the following code and run it:

```c
#include <windows.h>
#include <iostream>
#include <string>
#include <random>
#include <TlHelp32.h>
#include <queue>

#define NUM_FILES 1000
#define NUM_REPEATS_PROCESSES 10

CONDITION_VARIABLE ConditionVar;
CRITICAL_SECTION CritSection;
std::queue<std::wstring> fileQueue;

DWORD WINAPI CreateAndWriteFiles(LPVOID lpParam) {
    std::random_device rd;
    std::mt19937 gen(rd());

    for (int i = 0; i < NUM_FILES; i++) {
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
            std::wcerr << L"CreateFile failed for " << fileName << L". Error: " << GetLastError() << std::endl;
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
                std::wcerr << L"WriteFile failed for " << fileName << L". Error: " << GetLastError() << std::endl;
                break;
            }
        }

        if (!CloseHandle(hFile)) {
            std::wcerr << L"CloseHandle failed for " << fileName << L". Error: " << GetLastError() << std::endl;
        }

        std::wcout << L"Created and wrote to file: " << fileName << std::endl;

        EnterCriticalSection(&CritSection);
        fileQueue.push(fileName);

        // Mistake: Wait for the consumer to consume the file.
        while (!fileQueue.empty()) {
            SleepConditionVariableCS(&ConditionVar, &CritSection, INFINITE);
        }

        LeaveCriticalSection(&CritSection);
    }

    return 0;
}

DWORD WINAPI MoveFilesToNewDirectory(LPVOID lpParam) {
    for (int i = 0; i < NUM_FILES; i++) {
        EnterCriticalSection(&CritSection);

        // Mistake: Always wait, even if there's data.
        SleepConditionVariableCS(&ConditionVar, &CritSection, INFINITE);

        std::wstring source = fileQueue.front();
        fileQueue.pop();
        LeaveCriticalSection(&CritSection);

        std::wstring destination = L"C:\\Temp2\\" + source.substr(source.find_last_of(L"\\") + 1);

        if (!MoveFile(source.c_str(), destination.c_str())) {
            std::wcerr << L"MoveFile failed for " << source << L" to " << destination << L". Error: " << GetLastError() << std::endl;
        }
        else {
            std::wcout << L"Moved file from " << source << L" to " << destination << std::endl;
        }

        WakeConditionVariable(&ConditionVar);  // Signal the producer after consuming the file.
    }

    return 0;
}

DWORD WINAPI EnumerateProcesses(LPVOID lpParam) {
    for (int j = 0; j < NUM_REPEATS_PROCESSES; j++) {
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
    InitializeConditionVariable(&ConditionVar);
    InitializeCriticalSection(&CritSection);

    LARGE_INTEGER frequency;
    QueryPerformanceFrequency(&frequency);

    LARGE_INTEGER start;
    QueryPerformanceCounter(&start);

    DWORD threadID;
    HANDLE hThreads[3];

    hThreads[0] = CreateThread(NULL, 0, CreateAndWriteFiles, NULL, 0, &threadID);
    if (hThreads[0] == NULL) {
        std::cerr << "Failed to create CreateAndWriteFiles thread. Error: " << GetLastError() << std::endl;
        return 1;
    }

    hThreads[1] = CreateThread(NULL, 0, MoveFilesToNewDirectory, NULL, 0, &threadID);
    if (hThreads[1] == NULL) {
        std::cerr << "Failed to create MoveFilesToNewDirectory thread. Error: " << GetLastError() << std::endl;
        return 1;
    }

    hThreads[2] = CreateThread(NULL, 0, EnumerateProcesses, NULL, 0, &threadID);
    if (hThreads[2] == NULL) {
        std::cerr << "Failed to create EnumerateProcesses thread. Error: " << GetLastError() << std::endl;
        return 1;
    }

    if (WaitForMultipleObjects(3, hThreads, TRUE, INFINITE) == WAIT_FAILED) {
        std::cerr << "WaitForMultipleObjects failed. Error: " << GetLastError() << std::endl;
    }

    for (int i = 0; i < 3; i++) {
        if (!CloseHandle(hThreads[i])) {
            std::cerr << "CloseHandle failed for thread " << i << ". Error: " << GetLastError() << std::endl;
        }
    }

    LARGE_INTEGER end;
    QueryPerformanceCounter(&end);

    double time = static_cast<double>(end.QuadPart - start.QuadPart) / frequency.QuadPart;
    std::wcout << L"Total execution time: " << time << L" seconds" << std::endl;

    return 0;
}
```

When we run our program, we can see that it will start to hang and both threads are in a waiting state:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/44dfbaf4-982b-4b91-bfed-7dd20780a887)


Let's take a memory dump of this process, while it is hanging:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/7e7d7372-e71d-423a-acb6-130ff9b38803)


# WinDbg Walk Through (2) - Analyzing Memory Dump

Start loading the memory dump in WinDbg:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/3805bf92-db11-4cb1-a140-e6b2d28bc451)


Load the **MEX** extension:

```
0:000> .load mex
Mex External 3.0.0.7172 Loaded!
```

Start listening the threads by using the **!mex.lt** command:

```
0:000> !mex.lt
 # DbgID ThdID Wait Function                       User Kernel Info     TEB              Create Time
== ===== ===== =================================== ==== ====== ======== ================ ==========================
->     0  152c Test!main+0x152                        0      0 Event... 0000003b2162d000 08/09/2023 01:55:18.747 PM
       1  30f0 ntdll!NtWaitForAlertByThreadId+0x14    0      0          0000003b2162f000 08/09/2023 01:55:18.763 PM
       2  1c50 ntdll!NtWaitForAlertByThreadId+0x14    0      0          0000003b21631000 08/09/2023 01:55:18.763 PM
```

Let's start reviewing the call stack of each thread. We will first start with thread ID **152c**. From this call stack, we can see that the program named **"Test"** started a new thread, which eventually made a call to **WaitForMultipleObjects** to wait for some synchronization objects to be signaled. This thread has not completed its execution. Instead, it's currently blocked and waiting on the **WaitForMultipleObjects** function.

```
0:002> !mex.t -t 0x152c
DbgID ThreadID      User Kernel Create Time (UTC)
0     152c (0n5420)    0      0 08/09/2023 01:55:18.747 PM

# Child-SP         Return           Call Site                                Source
0 0000003b218ff648 00007ff919d4f499 ntdll!NtWaitForMultipleObjects+0x14      
1 0000003b218ff650 00007ff919d4f39e KERNELBASE!WaitForMultipleObjectsEx+0xe9 
2 0000003b218ff930 00007ff718a535b2 KERNELBASE!WaitForMultipleObjects+0xe    
3 0000003b218ff970 00007ff718a5fc30 Test!main+0x152                          C:\Users\User\source\repos\Test\Test\Test.cpp @ 155
4 (Inline)         ---------------- Test!invoke_main+0x22                    D:\a\_work\1\s\src\vctools\crt\vcstartup\src\startup\exe_common.inl @ 78
5 0000003b218ffa00 00007ff91b7826ad Test!__scrt_common_main_seh+0x10c        D:\a\_work\1\s\src\vctools\crt\vcstartup\src\startup\exe_common.inl @ 288
6 0000003b218ffa40 00007ff91c34aa68 kernel32!BaseThreadInitThunk+0x1d        
7 0000003b218ffa70 0000000000000000 ntdll!RtlUserThreadStart+0x28
```

Since we have the source code, let's navigate to line **155**:

```
3 0000003b218ff970 00007ff718a5fc30 Test!main+0x152                          C:\Users\User\source\repos\Test\Test\Test.cpp @ 155
```

At line **155** of the **`main`** function, the thread is paused due to a call to **WaitForMultipleObjects**. This means it's waiting for several threads to complete before resuming its tasks.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/353a0b67-9445-422c-ad2c-4d67d91b4781)


Here we are reviewing the call stack of the second thread ID which is **30f0**. Based on this call stack, we can see that a thread was running inside the **`CreateAndWriteFiles`** function. At some point, this function made a call to **RtlSleepConditionVariableCS** to wait on a condition variable. This means the thread has paused its execution and is currently waiting for a particular condition to be signaled before it can resume its work.

```
0:000> !mex.t -t 0x30f0
DbgID ThreadID       User Kernel Create Time (UTC)
1     30f0 (0n12528)    0      0 08/09/2023 01:55:18.763 PM

# Child-SP         Return           Call Site                                Source
0 0000003b219fe9b8 00007ff91c358d44 ntdll!NtWaitForAlertByThreadId+0x14      
1 0000003b219fe9c0 00007ff919d62239 ntdll!RtlSleepConditionVariableCS+0x104  
2 0000003b219fea40 00007ff718a52d8a KERNELBASE!SleepConditionVariableCS+0x29 
3 0000003b219fea70 00007ff91b7826ad Test!CreateAndWriteFiles+0x58a           C:\Users\User\source\repos\Test\Test\Test.cpp @ 61
4 0000003b219fff20 00007ff91c34aa68 kernel32!BaseThreadInitThunk+0x1d        
5 0000003b219fff50 0000000000000000 ntdll!RtlUserThreadStart+0x28
```

Since we have the source code, let's navigate to line **61**:

```
3 0000003b219fea70 00007ff91b7826ad Test!CreateAndWriteFiles+0x58a           C:\Users\User\source\repos\Test\Test\Test.cpp @ 61
```

The **`CreateAndWriteFiles`** function, which makes and fills files, adds the new file's name to a queue once it's done. Then, it waits and won't move on until the other thread has taken and used (or "consumed") that file name from the queue.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/d95e838b-0f2f-4c24-bfa4-781e569925a6)


Let's now review the last thread and look at the call stack of thread ID **1c50**. Based on this call stack, the thread was executing the **`MoveFilesToNewDirectory`** while executing this function, it reached a point where it needed to wait for a condition to be signaled. As a result, the thread went to sleep using the **RtlSleepConditionVariableCS** function, waiting for the specified condition to be met.

```
0:001> !mex.t -t 0x1c50
DbgID ThreadID      User Kernel Create Time (UTC)
2     1c50 (0n7248)    0      0 08/09/2023 01:55:18.763 PM

# Child-SP         Return           Call Site                                Source
0 0000003b21affb58 00007ff91c358d44 ntdll!NtWaitForAlertByThreadId+0x14      
1 0000003b21affb60 00007ff919d62239 ntdll!RtlSleepConditionVariableCS+0x104  
2 0000003b21affbe0 00007ff718a52eb7 KERNELBASE!SleepConditionVariableCS+0x29 
3 0000003b21affc10 00007ff91b7826ad Test!MoveFilesToNewDirectory+0x77        C:\Users\User\source\repos\Test\Test\Test.cpp @ 78
4 0000003b21affde0 00007ff91c34aa68 kernel32!BaseThreadInitThunk+0x1d        
5 0000003b21affe10 0000000000000000 ntdll!RtlUserThreadStart+0x28
```   

Since we have the source code, let's navigate to line **78**:

```
3 0000003b21affc10 00007ff91b7826ad Test!MoveFilesToNewDirectory+0x77        C:\Users\User\source\repos\Test\Test\Test.cpp @ 78
```

The **`MoveFilesToNewDirectory`** function, which handles moving files, starts by waiting for a sign that there's a new filename to move. But it doesn't check if there are already filenames waiting in the **`fileQueue`**. This means it might just sit and wait for the other thread to add a new filename, even if there are filenames already in the queue ready to be moved.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/dbb96e02-d4ec-4f10-86ba-3f436d96529d)


These behaviors together create a deadlock:

The first thread, **`CreateAndWriteFiles`**, after adding a filename to the **`fileQueue`**, waits for that queue to become empty. It is expecting the second thread to immediately take and process the filename it just added.

Meanwhile, the second thread, **`MoveFilesToNewDirectory`**, starts by waiting for a signal to begin processing filenames from the **`fileQueue`**. It doesn't check if there are already filenames in the queue before waiting.

The deadlock occurred because both threads were waiting unnecessarily and indefinitely.

# Code Fix

Instead of waiting for the **`fileQueue`** to be empty after pushing every file, **`CreateAndWriteFiles`** function now immediately signals the **`MoveFilesToNewDirectory`** function after adding a file to the **`fileQueue`**.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/9e519ab2-3100-4a5a-b1be-0fe08607bb87)


**`MoveFilesToNewDirectory`** function should only wait if the **`fileQueue`** is empty. This ensures that if there are files in the queue, it processes them immediately. If the queue is empty, it will then wait for the producer to add files and signal.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/94f9a582-3a47-4384-a1a1-e1fefb202124)


Here is the full code that should work as expected:

```c
#include <windows.h>
#include <iostream>
#include <string>
#include <random>
#include <TlHelp32.h>
#include <queue>

#define NUM_FILES 1000
#define NUM_REPEATS_PROCESSES 10

CONDITION_VARIABLE ConditionVar;
CRITICAL_SECTION CritSection;
std::queue<std::wstring> fileQueue;

DWORD WINAPI CreateAndWriteFiles(LPVOID lpParam) {
    std::random_device rd;
    std::mt19937 gen(rd());

    for (int i = 0; i < NUM_FILES; i++) {
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
            std::wcerr << L"CreateFile failed for " << fileName << L". Error: " << GetLastError() << std::endl;
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
                std::wcerr << L"WriteFile failed for " << fileName << L". Error: " << GetLastError() << std::endl;
                break;
            }
        }

        if (!CloseHandle(hFile)) {
            std::wcerr << L"CloseHandle failed for " << fileName << L". Error: " << GetLastError() << std::endl;
        }

        std::wcout << L"Created and wrote to file: " << fileName << std::endl;

        EnterCriticalSection(&CritSection);
        fileQueue.push(fileName);
        WakeConditionVariable(&ConditionVar);  // FIX: Signal immediately once a file is added.
        LeaveCriticalSection(&CritSection);
    }

    return 0;
}

DWORD WINAPI MoveFilesToNewDirectory(LPVOID lpParam) {
    for (int i = 0; i < NUM_FILES; i++) {
        EnterCriticalSection(&CritSection);

        // FIX: Only wait if the queue is empty.
        while (fileQueue.empty()) {
            SleepConditionVariableCS(&ConditionVar, &CritSection, INFINITE);
        }

        std::wstring source = fileQueue.front();
        fileQueue.pop();
        LeaveCriticalSection(&CritSection);

        std::wstring destination = L"C:\\Temp2\\" + source.substr(source.find_last_of(L"\\") + 1);
        if (!MoveFile(source.c_str(), destination.c_str())) {
            std::wcerr << L"MoveFile failed for " << source << L" to " << destination << L". Error: " << GetLastError() << std::endl;
        }
        else {
            std::wcout << L"Moved file from " << source << L" to " << destination << std::endl;
        }
    }

    return 0;
}

DWORD WINAPI EnumerateProcesses(LPVOID lpParam) {
    for (int j = 0; j < NUM_REPEATS_PROCESSES; j++) {
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
    InitializeConditionVariable(&ConditionVar);
    InitializeCriticalSection(&CritSection);

    LARGE_INTEGER frequency;
    QueryPerformanceFrequency(&frequency);

    LARGE_INTEGER start;
    QueryPerformanceCounter(&start);

    DWORD threadID;
    HANDLE hThreads[3];

    hThreads[0] = CreateThread(NULL, 0, CreateAndWriteFiles, NULL, 0, &threadID);
    if (hThreads[0] == NULL) {
        std::cerr << "Failed to create CreateAndWriteFiles thread. Error: " << GetLastError() << std::endl;
        return 1;
    }

    hThreads[1] = CreateThread(NULL, 0, MoveFilesToNewDirectory, NULL, 0, &threadID);
    if (hThreads[1] == NULL) {
        std::cerr << "Failed to create MoveFilesToNewDirectory thread. Error: " << GetLastError() << std::endl;
        return 1;
    }

    hThreads[2] = CreateThread(NULL, 0, EnumerateProcesses, NULL, 0, &threadID);
    if (hThreads[2] == NULL) {
        std::cerr << "Failed to create EnumerateProcesses thread. Error: " << GetLastError() << std::endl;
        return 1;
    }

    if (WaitForMultipleObjects(3, hThreads, TRUE, INFINITE) == WAIT_FAILED) {
        std::cerr << "WaitForMultipleObjects failed. Error: " << GetLastError() << std::endl;
    }

    for (int i = 0; i < 3; i++) {
        if (!CloseHandle(hThreads[i])) {
            std::cerr << "CloseHandle failed for thread " << i << ". Error: " << GetLastError() << std::endl;
        }
    }

    LARGE_INTEGER end;
    QueryPerformanceCounter(&end);

    double time = static_cast<double>(end.QuadPart - start.QuadPart) / frequency.QuadPart;
    std::wcout << L"Total execution time: " << time << L" seconds" << std::endl;

    return 0;
}
```
