# What is a Deadlock?

Deadlocks are situations where two or more threads are unable to proceed with their execution because they are each waiting for the other to release a resource. Deadlocks typically involve a circular wait condition, where Thread A is waiting for a resource held by Thread B, and Thread B is waiting for a resource held by Thread A.

Let's demonstrate a deadlock with wrongly implementing SRW locks.



In order to demonstrate this demo. Please follow the following steps:

- Create the following 3 folders:
  1. **C:\Temp**
  2. **C:\Temp2**
  3. **C:\Temp3**

The purpose of this code is the following:

- The function **`WriteToFile`** operates under **multiple** threads (defined by **`NUM_WRITER_THREADS`**). Each of these threads writes a distinct message to a single file located at **`C:\Temp\log.txt`**.
- The function **`MoveFileToFolder`** moves the file from **`C:\Temp\log.txt`** to **`C:\Temp2\log.txt`**.
- The function **`MoveFileToAnotherFolder`** relocates the file from **`C:\Temp2\log.txt`** to **`C:\Temp3\log.txt`**.

However, the code won't work since there is a deadlock due to releasing the locks.

```c
#include <windows.h>
#include <iostream>
#include <sstream>
#include <thread>

// Define constants for the number of writer threads, total threads, and operations
#define NUM_WRITER_THREADS 10
#define TOTAL_THREADS (NUM_WRITER_THREADS + 2)
#define NUM_OPERATIONS 100

// Declare a Slim Reader/Writer (SRW) Lock and counters for the completed threads and move operations
SRWLOCK srwLock;
int completedThreads = 0;

// This function writes to a file
DWORD WINAPI WriteToFile(LPVOID lpParam) {
    // Create a string that contains the thread ID
    std::stringstream ss;
    ss << GetCurrentThreadId();
    std::string data = "Hello, World from Writer Thread " + ss.str() + "\r\n";

    // Repeat the write operation a certain number of times
    for (int i = 0; i < NUM_OPERATIONS; i++) {
        // Acquire the SRW lock in exclusive mode
        AcquireSRWLockExclusive(&srwLock);

        // Open the file for writing
        HANDLE hFile = CreateFile(L"C:\\Temp\\log.txt", GENERIC_WRITE, 0, NULL, OPEN_ALWAYS, FILE_ATTRIBUTE_NORMAL, NULL);
        if (hFile == INVALID_HANDLE_VALUE) {
            // Handle the error if the file cannot be opened for writing
            std::cerr << "CreateFile (write) failed. Error: " << GetLastError() << std::endl;
            return 1;
        }

        // Move the file pointer to the end of the file
        SetFilePointer(hFile, 0, NULL, FILE_END);

        // Declare a variable to hold the number of bytes written
        DWORD bytesWritten;

        // Write to the file
        if (!WriteFile(hFile, data.c_str(), data.size(), &bytesWritten, NULL)) {
            // Handle the error if the file cannot be written to
            std::cerr << "WriteFile failed. Error: " << GetLastError() << std::endl;
            CloseHandle(hFile);
            return 1;
        }

        // Output a message to the console
        std::cout << "Writer Thread " << ss.str() << " wrote to the file: " << data;

        // Close the file
        CloseHandle(hFile);

        // DEADLOCK: Not releasing SRW lock
        //ReleaseSRWLockExclusive(&srwLock);

        // Pause for 100 milliseconds
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
    }

    // Increment completedThreads under the protection of the SRW lock
    AcquireSRWLockExclusive(&srwLock);
    completedThreads++;
    ReleaseSRWLockExclusive(&srwLock);

    return 0;
}

// This function moves a file from one folder to another
DWORD WINAPI MoveFileToFolder(LPVOID lpParam) {
    // Create a string that contains the thread ID
    std::stringstream ss;
    ss << GetCurrentThreadId();

    // Wait until all other threads have finished
    while (true) {
        // Check the completedThreads counter under the protection of the SRW lock
        AcquireSRWLockShared(&srwLock);
        bool allThreadsCompleted = (completedThreads == NUM_WRITER_THREADS);
        ReleaseSRWLockShared(&srwLock);

        if (allThreadsCompleted) {
            break;
        }

        // Avoid busy waiting
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
    }

    // Acquire the SRW lock in exclusive mode
    AcquireSRWLockExclusive(&srwLock);

    // Move the file and overwrite if the file already exists
    if (!MoveFileEx(L"C:\\Temp\\log.txt", L"C:\\Temp2\\log.txt", MOVEFILE_REPLACE_EXISTING)) {
        // Handle the error if the file cannot be moved
        std::cerr << "MoveFileEx failed. Error: " << GetLastError() << std::endl;
        ReleaseSRWLockExclusive(&srwLock);
        return 1;
    }

    // Output a message to the console
    std::cout << "Mover Thread " << ss.str() << " moved the file.\n";

    // DEADLOCK: Not releasing the SRW lock
    //ReleaseSRWLockExclusive(&srwLock); 

    // Increment completedThreads under the protection of the SRW lock
    AcquireSRWLockExclusive(&srwLock);
    completedThreads++;
    ReleaseSRWLockExclusive(&srwLock);

    return 0;
}

// This function moves a file from one folder to another
DWORD WINAPI MoveFileToAnotherFolder(LPVOID lpParam) {
    // Create a string that contains the thread ID
    std::stringstream ss;
    ss << GetCurrentThreadId();

    // Wait until all other threads have finished
    while (true) {
        // Check the completedThreads counter under the protection of the SRW lock
        AcquireSRWLockShared(&srwLock);
        bool allThreadsCompleted = (completedThreads == NUM_WRITER_THREADS + 1); // Increment by 1 to wait for the first move operation
        ReleaseSRWLockShared(&srwLock);

        if (allThreadsCompleted) {
            break;
        }

        // Avoid busy waiting
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
    }

    // Acquire the SRW lock in exclusive mode
    AcquireSRWLockExclusive(&srwLock);

    // Move the file and overwrite if the file already exists
    if (!MoveFileEx(L"C:\\Temp2\\log.txt", L"C:\\Temp3\\log.txt", MOVEFILE_REPLACE_EXISTING)) {
        // Handle the error if the file cannot be moved
        std::cerr << "MoveFileEx failed. Error: " << GetLastError() << std::endl;
        ReleaseSRWLockExclusive(&srwLock);
        return 1;
    }

    // Output a message to the console
    std::cout << "Mover Thread " << ss.str() << " moved the file to another folder.\n";

    // Release the SRW lock
    ReleaseSRWLockExclusive(&srwLock);

    return 0;
}

// The main function of the program
int main() {
    // Initialize the SRW lock
    InitializeSRWLock(&srwLock);

    // Create an array to hold the thread handles
    HANDLE hThreads[TOTAL_THREADS];

    // Create the writer threads
    for (int i = 0; i < NUM_WRITER_THREADS; i++) {
        // Create a new thread to write to the file
        hThreads[i] = CreateThread(NULL, 0, WriteToFile, NULL, 0, NULL);
        if (hThreads[i] == NULL) {
            // Handle the error if the thread cannot be created
            std::cerr << "CreateThread error: " << GetLastError() << '\n';
            return 1;
        }
    }

    // Create a new thread to move the file
    hThreads[NUM_WRITER_THREADS] = CreateThread(NULL, 0, MoveFileToFolder, NULL, 0, NULL);
    if (hThreads[NUM_WRITER_THREADS] == NULL) {
        // Handle the error if the thread cannot be created
        std::cerr << "CreateThread error: " << GetLastError() << '\n';
        return 1;
    }

    // Create a new thread to move the file to another folder
    hThreads[NUM_WRITER_THREADS + 1] = CreateThread(NULL, 0, MoveFileToAnotherFolder, NULL, 0, NULL);
    if (hThreads[NUM_WRITER_THREADS + 1] == NULL) {
        // Handle the error if the thread cannot be created
        std::cerr << "CreateThread error: " << GetLastError() << '\n';
        return 1;
    }

    // Wait for all threads to finish
    WaitForMultipleObjects(TOTAL_THREADS, hThreads, TRUE, INFINITE);

    // Close all thread handles
    for (int i = 0; i < TOTAL_THREADS; i++) {
        CloseHandle(hThreads[i]);
    }

    return 0;
}
```

When we run this code, it will start to hang as we can see here:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/5d13c210-e2a0-4702-81b6-7f8b7793b9f8)


Open **Process Explorer** and view all the threads of this program. We will notice that almost all of the thread wait reasons are **WrAlertByThreadId** and in this case, this means that a thread is waiting on a Slim Reader & Writer Lock.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/4f4ddb18-842c-4d87-85ac-76cf3d81c39c)


# WinDbg Walk Through - Analyzing Memory Dump

Let's take a memory dump of the program while it is still hanging and load it in WinDbg.

Start with loadig the **MEX** extension:

```
0:000> .load mex
Mex External 3.0.0.7172 Loaded!
```

Run the **!mex.p** command to get the current process context:

```
0:000> !mex.p
Name     Ses PID           PEB              Mods Handle Thrd
======== === ============= ================ ==== ====== ====
test.exe   1 1434 (0n5172) 000000cc67b6f000    4     53   13

CommandLine: Test.exe
Last event: 1434.1040: Break instruction exception - code 80000003 (first/second chance not available)

Show Threads: Unique Stacks    !listthreads (!lt)    ~*kv
```

Let's run the **!mex.lt** command to list all the threads of the process:

```
0:000> !mex.lt
 # DbgID ThdID Wait Function                       User Kernel Info     TEB              Create Time
== ===== ===== =================================== ==== ====== ======== ================ ==========================
->     0  1040 Test!main+0xe3                         0      0 Event... 000000cc67b70000 08/11/2023 07:37:29.175 AM
       1   2e8 ntdll!NtWaitForAlertByThreadId+0x14    0      0          000000cc67b72000 08/11/2023 07:37:29.181 AM
       2   3ac ntdll!NtWaitForAlertByThreadId+0x14    0      0          000000cc67b74000 08/11/2023 07:37:29.181 AM
       3  2994 ntdll!NtWaitForAlertByThreadId+0x14    0      0          000000cc67b76000 08/11/2023 07:37:29.181 AM
       4   7e8 ntdll!NtWaitForAlertByThreadId+0x14    0      0          000000cc67b78000 08/11/2023 07:37:29.181 AM
       5  2204 ntdll!NtWaitForAlertByThreadId+0x14 15ms      0          000000cc67b7a000 08/11/2023 07:37:29.181 AM
       6  1d00 ntdll!NtWaitForAlertByThreadId+0x14    0      0          000000cc67b7c000 08/11/2023 07:37:29.181 AM
       7   37c ntdll!NtWaitForAlertByThreadId+0x14    0      0          000000cc67b7e000 08/11/2023 07:37:29.181 AM
       8  1598 ntdll!NtWaitForAlertByThreadId+0x14    0      0          000000cc67b80000 08/11/2023 07:37:29.181 AM
       9  210c ntdll!NtWaitForAlertByThreadId+0x14    0      0          000000cc67b82000 08/11/2023 07:37:29.181 AM
      10   ebc ntdll!NtWaitForAlertByThreadId+0x14    0      0          000000cc67b84000 08/11/2023 07:37:29.181 AM
      11  1198 ntdll!NtWaitForAlertByThreadId+0x14    0      0          000000cc67b86000 08/11/2023 07:37:29.181 AM
      12  1054 ntdll!NtWaitForAlertByThreadId+0x14    0      0          000000cc67b88000 08/11/2023 07:37:29.181 AM
```

We have 13 threads in total and multiple of them are currently waiting on a Slim Reader & Writer Lock. As an example, we will be running the **!mex.us -w** command which reveals that within the **`main`** function, we are calling **WaitForMultipleObjects** and it's visible at the top of the call stack. This means it's waiting for several threads to complete before resuming its tasks.

```
0:000> !mex.us -w
1 thread [stats]: 0
    00007ff99efcf8a4 ntdll!NtWaitForMultipleObjects+0x14      
    00007ff99c67f5a9 KERNELBASE!WaitForMultipleObjectsEx+0xe9 
    00007ff99c67f4ae KERNELBASE!WaitForMultipleObjects+0xe    
    00007ff6f3392d03 Test!main+0xe3                           (C:\Users\User\source\repos\Test\Test\Test.cpp @ 197)
    (Inline)         Test!invoke_main+0x22                    (D:\a\_work\1\s\src\vctools\crt\vcstartup\src\startup\exe_common.inl @ 78)
    00007ff6f339a7c0 Test!__scrt_common_main_seh+0x10c        (D:\a\_work\1\s\src\vctools\crt\vcstartup\src\startup\exe_common.inl @ 288)
    00007ff99de926ad kernel32!BaseThreadInitThunk+0x1d        
    00007ff99ef8aa68 ntdll!RtlUserThreadStart+0x28            

Threads matching filter: 1 out of 13
```

Since we have the source code of our program, let's navigate to line **197** and gather more information:

The **CloseHandle** function is used to close an open object handle. In this context, it's being used to close the thread handles after waiting for all of them to complete their tasks. The program waits for all the threads in the **`hThreads`** array to complete. The **`INFINITE`** parameter in **WaitForMultipleObject** ensures it will wait indefinitely until all threads have completed.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/28f0d139-7951-483b-a4e6-5b653b885dec)


There are 13 threads, so instead of viewing thread by thread. Let's use the **!mex.us** command, which will display all the call stack of each thread within the process:

```
0:000> !mex.us
1 thread [stats]: 0
    00007ff99efcf8a4 ntdll!NtWaitForMultipleObjects+0x14      
    00007ff99c67f5a9 KERNELBASE!WaitForMultipleObjectsEx+0xe9 
    00007ff99c67f4ae KERNELBASE!WaitForMultipleObjects+0xe    
    00007ff6f3392d03 Test!main+0xe3                           (C:\Users\User\source\repos\Test\Test\Test.cpp @ 197)
    (Inline)         Test!invoke_main+0x22                    (D:\a\_work\1\s\src\vctools\crt\vcstartup\src\startup\exe_common.inl @ 78)
    00007ff6f339a7c0 Test!__scrt_common_main_seh+0x10c        (D:\a\_work\1\s\src\vctools\crt\vcstartup\src\startup\exe_common.inl @ 288)
    00007ff99de926ad kernel32!BaseThreadInitThunk+0x1d        
    00007ff99ef8aa68 ntdll!RtlUserThreadStart+0x28            

1 thread [stats]: 11
    00007ff99efd2944 ntdll!NtWaitForAlertByThreadId+0x14 
    00007ff99ef56978 ntdll!RtlAcquireSRWLockShared+0x148 
    00007ff6f3392769 Test!MoveFileToFolder+0x69          (C:\Users\User\source\repos\Test\Test\Test.cpp @ 80)
    00007ff99de926ad kernel32!BaseThreadInitThunk+0x1d   
    00007ff99ef8aa68 ntdll!RtlUserThreadStart+0x28       

1 thread [stats]: 12
    00007ff99efd2944 ntdll!NtWaitForAlertByThreadId+0x14 
    00007ff99ef56978 ntdll!RtlAcquireSRWLockShared+0x148 
    00007ff6f3392a09 Test!MoveFileToAnotherFolder+0x69   (C:\Users\User\source\repos\Test\Test\Test.cpp @ 126)
    00007ff99de926ad kernel32!BaseThreadInitThunk+0x1d   
    00007ff99ef8aa68 ntdll!RtlUserThreadStart+0x28       

10 threads [stats]: 1 2 3 4 5 6 7 8 9 10
    00007ff99efd2944 ntdll!NtWaitForAlertByThreadId+0x14    
    00007ff99ef67965 ntdll!RtlAcquireSRWLockExclusive+0x165 
    00007ff6f3392310 Test!WriteToFile+0x2d0                 (C:\Users\User\source\repos\Test\Test\Test.cpp @ 28)
    00007ff99de926ad kernel32!BaseThreadInitThunk+0x1d      
    00007ff99ef8aa68 ntdll!RtlUserThreadStart+0x28          

4 stack(s) with 13 threads displayed (13 Total threads)
```

There are two functions **`ntdll!RtlAcquireSRWLockExclusive`** & **`ntdll!RtlAcquireSRWLockShared`** of interest in the call stacks. Let's first examine the call stacks that contains the **`ntdll!RtlAcquireSRWLockExclusive`** function. The **!mex.us** command allows us to filter and only display the call stacks that contain a specific string of interest.

```
0:000> !mex.us ntdll!RtlAcquireSRWLockExclusive
10 threads [stats]: 1 2 3 4 5 6 7 8 9 10
    00007ff99efd2944 ntdll!NtWaitForAlertByThreadId+0x14    
    00007ff99ef67965 ntdll!RtlAcquireSRWLockExclusive+0x165 
    00007ff6f3392310 Test!WriteToFile+0x2d0                 (C:\Users\User\source\repos\Test\Test\Test.cpp @ 28)
    00007ff99de926ad kernel32!BaseThreadInitThunk+0x1d      
    00007ff99ef8aa68 ntdll!RtlUserThreadStart+0x28          

Threads matching filter: 10 out of 13
```

The call stack(s) shows that the **`WriteToFile`** function contains the **`ntdll!RtlAcquireSRWLockExclusive`** function, which means that it tried to acquire an SRW lock in exclusive mode. The fact that this function call is below the waiting function suggests that the threads tried to acquire the SRW lock, but couldn't get immediate access, and went into a waiting state.

However, let's take a step back and try to gather a better understanding of the data that we're seeing after typing **!mex.us** command. What do these **'stats'** actually mean?

This shows that there are **10** threads with a **identical** call stack at line **28** in the source code that are trying to acquire a SRW lock:

```
10 threads [stats]: 1 2 3 4 5 6 7 8 9 10
    00007ff99efd2944 ntdll!NtWaitForAlertByThreadId+0x14    
    00007ff99ef67965 ntdll!RtlAcquireSRWLockExclusive+0x165 
    00007ff6f3392310 Test!WriteToFile+0x2d0                 (C:\Users\User\source\repos\Test\Test\Test.cpp @ 28)
    00007ff99de926ad kernel32!BaseThreadInitThunk+0x1d      
    00007ff99ef8aa68 ntdll!RtlUserThreadStart+0x28
```

Here we can double check this as well by clicking on **'stats'**, which redirects to the **!mex.fems** command. This command is looking for identical call stacks within threads. In this case, the 10 threads are currently blocked.

```
0:000> !mex.fems 1 -stat

Thread Statistics:

    Thread ID User Time Kernel Time Total Time WaitReason State
    ========= ========= =========== ========== ========== ===========
            5      15ms           0       15ms Executive  Initialized
            1         0           0          0 Executive  Initialized
            2         0           0          0 Executive  Initialized
            3         0           0          0 Executive  Initialized
            4         0           0          0 Executive  Initialized
            6         0           0          0 Executive  Initialized
            7         0           0          0 Executive  Initialized
            8         0           0          0 Executive  Initialized
            9         0           0          0 Executive  Initialized
           10         0           0          0 Executive  Initialized
```

# WinDbg Walk Through - Code Review

At line **25**, we are acquiring an SRW lock in exclusive mode. This is a standard usage, ensuring that the thread has exclusive access to the shared resource, which in this case is the file. The **10** threads are blocked at line **28** which is attempting to open the file for writing with the **CreateFile** function.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/532927aa-d7fb-4495-927c-1c5ff4b617fe)


Releasing the lock right after **`CloseHandle()`** ensures that other threads waiting to acquire the lock can proceed as soon as the file operations are completed. This minimizes the time the lock is held, which in turn reduces contention and the chances of deadlocks. 

In this example, we commented out line **56**, but at this line we should have to release the SRW lock by calling **ReleaseSRWLockExclusive**.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/21721208-ece8-41ed-898b-6eeb146d067c)


Let's now uncomment this and compile the code again and run it:

```c
#include <windows.h>
#include <iostream>
#include <sstream>
#include <thread>

// Define constants for the number of writer threads, total threads, and operations
#define NUM_WRITER_THREADS 10
#define TOTAL_THREADS (NUM_WRITER_THREADS + 2)
#define NUM_OPERATIONS 100

// Declare a Slim Reader/Writer (SRW) Lock and counters for the completed threads and move operations
SRWLOCK srwLock;
int completedThreads = 0;

// This function writes to a file
DWORD WINAPI WriteToFile(LPVOID lpParam) {
    // Create a string that contains the thread ID
    std::stringstream ss;
    ss << GetCurrentThreadId();
    std::string data = "Hello, World from Writer Thread " + ss.str() + "\r\n";

    // Repeat the write operation a certain number of times
    for (int i = 0; i < NUM_OPERATIONS; i++) {
        // Acquire the SRW lock in exclusive mode
        AcquireSRWLockExclusive(&srwLock);

        // Open the file for writing
        HANDLE hFile = CreateFile(L"C:\\Temp\\log.txt", GENERIC_WRITE, 0, NULL, OPEN_ALWAYS, FILE_ATTRIBUTE_NORMAL, NULL);
        if (hFile == INVALID_HANDLE_VALUE) {
            // Handle the error if the file cannot be opened for writing
            std::cerr << "CreateFile (write) failed. Error: " << GetLastError() << std::endl;
            return 1;
        }

        // Move the file pointer to the end of the file
        SetFilePointer(hFile, 0, NULL, FILE_END);

        // Declare a variable to hold the number of bytes written
        DWORD bytesWritten;

        // Write to the file
        if (!WriteFile(hFile, data.c_str(), data.size(), &bytesWritten, NULL)) {
            // Handle the error if the file cannot be written to
            std::cerr << "WriteFile failed. Error: " << GetLastError() << std::endl;
            CloseHandle(hFile);
            return 1;
        }

        // Output a message to the console
        std::cout << "Writer Thread " << ss.str() << " wrote to the file: " << data;

        // Close the file
        CloseHandle(hFile);

        // Releasing SRW lock
        ReleaseSRWLockExclusive(&srwLock);

        // Pause for 100 milliseconds
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
    }

    // Increment completedThreads under the protection of the SRW lock
    AcquireSRWLockExclusive(&srwLock);
    completedThreads++;
    ReleaseSRWLockExclusive(&srwLock);

    return 0;
}

// This function moves a file from one folder to another
DWORD WINAPI MoveFileToFolder(LPVOID lpParam) {
    // Create a string that contains the thread ID
    std::stringstream ss;
    ss << GetCurrentThreadId();

    // Wait until all other threads have finished
    while (true) {
        // Check the completedThreads counter under the protection of the SRW lock
        AcquireSRWLockShared(&srwLock);
        bool allThreadsCompleted = (completedThreads == NUM_WRITER_THREADS);
        ReleaseSRWLockShared(&srwLock);

        if (allThreadsCompleted) {
            break;
        }

        // Avoid busy waiting
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
    }

    // Acquire the SRW lock in exclusive mode
    AcquireSRWLockExclusive(&srwLock);

    // Move the file and overwrite if the file already exists
    if (!MoveFileEx(L"C:\\Temp\\log.txt", L"C:\\Temp2\\log.txt", MOVEFILE_REPLACE_EXISTING)) {
        // Handle the error if the file cannot be moved
        std::cerr << "MoveFileEx failed. Error: " << GetLastError() << std::endl;
        ReleaseSRWLockExclusive(&srwLock);
        return 1;
    }

    // Output a message to the console
    std::cout << "Mover Thread " << ss.str() << " moved the file.\n";

    // DEADLOCK: Not releasing the SRW lock
    //ReleaseSRWLockExclusive(&srwLock); 

    // Increment completedThreads under the protection of the SRW lock
    AcquireSRWLockExclusive(&srwLock);
    completedThreads++;
    ReleaseSRWLockExclusive(&srwLock);

    return 0;
}

// This function moves a file from one folder to another
DWORD WINAPI MoveFileToAnotherFolder(LPVOID lpParam) {
    // Create a string that contains the thread ID
    std::stringstream ss;
    ss << GetCurrentThreadId();

    // Wait until all other threads have finished
    while (true) {
        // Check the completedThreads counter under the protection of the SRW lock
        AcquireSRWLockShared(&srwLock);
        bool allThreadsCompleted = (completedThreads == NUM_WRITER_THREADS + 1); // Increment by 1 to wait for the first move operation
        ReleaseSRWLockShared(&srwLock);

        if (allThreadsCompleted) {
            break;
        }

        // Avoid busy waiting
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
    }

    // Acquire the SRW lock in exclusive mode
    AcquireSRWLockExclusive(&srwLock);

    // Move the file and overwrite if the file already exists
    if (!MoveFileEx(L"C:\\Temp2\\log.txt", L"C:\\Temp3\\log.txt", MOVEFILE_REPLACE_EXISTING)) {
        // Handle the error if the file cannot be moved
        std::cerr << "MoveFileEx failed. Error: " << GetLastError() << std::endl;
        ReleaseSRWLockExclusive(&srwLock);
        return 1;
    }

    // Output a message to the console
    std::cout << "Mover Thread " << ss.str() << " moved the file to another folder.\n";

    // Release the SRW lock
    ReleaseSRWLockExclusive(&srwLock);

    return 0;
}

// The main function of the program
int main() {
    // Initialize the SRW lock
    InitializeSRWLock(&srwLock);

    // Create an array to hold the thread handles
    HANDLE hThreads[TOTAL_THREADS];

    // Create the writer threads
    for (int i = 0; i < NUM_WRITER_THREADS; i++) {
        // Create a new thread to write to the file
        hThreads[i] = CreateThread(NULL, 0, WriteToFile, NULL, 0, NULL);
        if (hThreads[i] == NULL) {
            // Handle the error if the thread cannot be created
            std::cerr << "CreateThread error: " << GetLastError() << '\n';
            return 1;
        }
    }

    // Create a new thread to move the file
    hThreads[NUM_WRITER_THREADS] = CreateThread(NULL, 0, MoveFileToFolder, NULL, 0, NULL);
    if (hThreads[NUM_WRITER_THREADS] == NULL) {
        // Handle the error if the thread cannot be created
        std::cerr << "CreateThread error: " << GetLastError() << '\n';
        return 1;
    }

    // Create a new thread to move the file to another folder
    hThreads[NUM_WRITER_THREADS + 1] = CreateThread(NULL, 0, MoveFileToAnotherFolder, NULL, 0, NULL);
    if (hThreads[NUM_WRITER_THREADS + 1] == NULL) {
        // Handle the error if the thread cannot be created
        std::cerr << "CreateThread error: " << GetLastError() << '\n';
        return 1;
    }

    // Wait for all threads to finish
    WaitForMultipleObjects(TOTAL_THREADS, hThreads, TRUE, INFINITE);

    // Close all thread handles
    for (int i = 0; i < TOTAL_THREADS; i++) {
        CloseHandle(hThreads[i]);
    }

    return 0;
}
```

The program is now writing data to the file and it has been moved to the **C:\Temp2**, but the size of the file is very small comparing on what we should expect.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/7ecf1ee7-0c45-4401-8549-da30b2597a62)


Open **Process Explorer** and view the wait reasons of this program. We can see that there are two threads waiting on a Slim Reader & Writer lock:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/c7eb614b-37e0-4798-b702-d387eb3a9025)


Take a memory dump of the process that is hanging and load it in WinDbg.

First, let's load the **MEX** extension:

```
0:000> .load mex
Mex External 3.0.0.7172 Loaded!
```

Start with listening all the threads within the process:

```
0:000> !mex.lt
 # DbgID ThdID Wait Function                       User Kernel Info     TEB              Create Time
== ===== ===== =================================== ==== ====== ======== ================ ==========================
->     0   d34 Test!main+0xe3                         0      0 Event... 000000aaa4507000 08/11/2023 08:04:46.770 AM
       1  1378 ntdll!NtWaitForAlertByThreadId+0x14    0      0          000000aaa451d000 08/11/2023 08:04:46.779 AM
       2  139c ntdll!NtWaitForAlertByThreadId+0x14    0      0          000000aaa451f000 08/11/2023 08:04:46.779 AM
```

Start reviewing the call stack of both threads:

```
0:002> !mex.t -t 0x1378
DbgID ThreadID      User Kernel Create Time (UTC)
1     1378 (0n4984)    0      0 08/11/2023 08:04:46.779 AM

# Child-SP         Return           Call Site                              Source
0 000000aaa51ff948 00007ff99ef67965 ntdll!NtWaitForAlertByThreadId+0x14    
1 000000aaa51ff950 00007ff6908c294b ntdll!RtlAcquireSRWLockExclusive+0x165 
2 000000aaa51ff9c0 00007ff99de926ad Test!MoveFileToFolder+0x24b            C:\Users\User\source\repos\Test\Test\Test.cpp @ 110
3 000000aaa51ffb40 00007ff99ef8aa68 kernel32!BaseThreadInitThunk+0x1d      
4 000000aaa51ffb70 0000000000000000 ntdll!RtlUserThreadStart+0x28
```

This call stack suggests that a thread is attempting to acquire an SRW lock in the **`MoveFileToFolder`** function. The thread has entered a wait state due to the SRW lock being held by another thread. Let's review the code that is associated with the **`MoveFileToFolder`** function.

1. The thread acquires the SRW lock using **`AcquireSRWLockExclusive(&srwLock)`**.
2. It performs some operations **without releasing the lock**.
3. Then, it attempts to acquire the same SRW lock again with **`AcquireSRWLockExclusive(&srwLock)`** to increment the **`completedThreads`** variable.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/e49ee879-c6f5-4d34-ad4c-978663bac899)


This attempt to re-acquire the same SRW lock, without releasing it first, leads to the self-deadlock. A self-deadlock occurs when a thread, which has already acquired a non-reentrant lock, tries to acquire the same lock again without first releasing it. Since the thread is waiting for a resource (in this case, the SRW lock) that it itself holds, it will wait indefinitely, leading to a deadlock.

Let's review the second thread and the call stack of it:

```
0:001> !mex.t -t 0x139c
DbgID ThreadID      User Kernel Create Time (UTC)
2     139c (0n5020)    0      0 08/11/2023 08:04:46.779 AM

# Child-SP         Return           Call Site                           Source
0 000000aaa52ff628 00007ff99ef56978 ntdll!NtWaitForAlertByThreadId+0x14 
1 000000aaa52ff630 00007ff6908c2a47 ntdll!RtlAcquireSRWLockShared+0x148 
2 000000aaa52ff6a0 00007ff99de926ad Test!MoveFileToAnotherFolder+0xa7   C:\Users\User\source\repos\Test\Test\Test.cpp @ 126
3 000000aaa52ff820 00007ff99ef8aa68 kernel32!BaseThreadInitThunk+0x1d   
4 000000aaa52ff850 0000000000000000 ntdll!RtlUserThreadStart+0x28
```

The call stack indicates that the thread is blocked when trying to acquire the SRW lock in **shared mode**, which suggests that another thread might be holding this lock in exclusive mode, preventing the **`MoveFileToAnotherFolder`** thread from acquiring it. The deadlock scenario or the blocking situation could be a result of the **`MoveFileToFolder`** function holding the SRW lock in exclusive mode and not releasing it.

Let's ensure that the SRW lock is always released after it's acquired, especially in the **`MoveFileToFolder`** function.

Here is entire fix:

```c
#include <windows.h>
#include <iostream>
#include <sstream>
#include <thread>

// Define constants for the number of writer threads, total threads, and operations
#define NUM_WRITER_THREADS 10
#define TOTAL_THREADS (NUM_WRITER_THREADS + 2)
#define NUM_OPERATIONS 100

// Declare a Slim Reader/Writer (SRW) Lock and counters for the completed threads and move operations
SRWLOCK srwLock;
int completedThreads = 0;

// This function writes to a file
DWORD WINAPI WriteToFile(LPVOID lpParam) {
    // Create a string that contains the thread ID
    std::stringstream ss;
    ss << GetCurrentThreadId();
    std::string data = "Hello, World from Writer Thread " + ss.str() + "\r\n";

    // Repeat the write operation a certain number of times
    for (int i = 0; i < NUM_OPERATIONS; i++) {
        // Acquire the SRW lock in exclusive mode
        AcquireSRWLockExclusive(&srwLock);

        // Open the file for writing
        HANDLE hFile = CreateFile(L"C:\\Temp\\log.txt", GENERIC_WRITE, 0, NULL, OPEN_ALWAYS, FILE_ATTRIBUTE_NORMAL, NULL);
        if (hFile == INVALID_HANDLE_VALUE) {
            // Handle the error if the file cannot be opened for writing
            std::cerr << "CreateFile (write) failed. Error: " << GetLastError() << std::endl;
            return 1;
        }

        // Move the file pointer to the end of the file
        SetFilePointer(hFile, 0, NULL, FILE_END);

        // Declare a variable to hold the number of bytes written
        DWORD bytesWritten;

        // Write to the file
        if (!WriteFile(hFile, data.c_str(), data.size(), &bytesWritten, NULL)) {
            // Handle the error if the file cannot be written to
            std::cerr << "WriteFile failed. Error: " << GetLastError() << std::endl;
            CloseHandle(hFile);
            return 1;
        }

        // Output a message to the console
        std::cout << "Writer Thread " << ss.str() << " wrote to the file: " << data;

        // Close the file
        CloseHandle(hFile);

        // Releasing SRW lock
        ReleaseSRWLockExclusive(&srwLock);

        // Pause for 100 milliseconds
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
    }

    // Increment completedThreads under the protection of the SRW lock
    AcquireSRWLockExclusive(&srwLock);
    completedThreads++;
    ReleaseSRWLockExclusive(&srwLock);

    return 0;
}

// This function moves a file from one folder to another
DWORD WINAPI MoveFileToFolder(LPVOID lpParam) {
    // Create a string that contains the thread ID
    std::stringstream ss;
    ss << GetCurrentThreadId();

    // Wait until all other threads have finished
    while (true) {
        // Check the completedThreads counter under the protection of the SRW lock
        AcquireSRWLockShared(&srwLock);
        bool allThreadsCompleted = (completedThreads == NUM_WRITER_THREADS);
        ReleaseSRWLockShared(&srwLock);

        if (allThreadsCompleted) {
            break;
        }

        // Avoid busy waiting
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
    }

    // Acquire the SRW lock in exclusive mode
    AcquireSRWLockExclusive(&srwLock);

    // Move the file and overwrite if the file already exists
    if (!MoveFileEx(L"C:\\Temp\\log.txt", L"C:\\Temp2\\log.txt", MOVEFILE_REPLACE_EXISTING)) {
        // Handle the error if the file cannot be moved
        std::cerr << "MoveFileEx failed. Error: " << GetLastError() << std::endl;
        ReleaseSRWLockExclusive(&srwLock);
        return 1;
    }

    // Output a message to the console
    std::cout << "Mover Thread " << ss.str() << " moved the file.\n";

    // Releasing the SRW lock
    ReleaseSRWLockExclusive(&srwLock); 

    // Increment completedThreads under the protection of the SRW lock
    AcquireSRWLockExclusive(&srwLock);
    completedThreads++;
    ReleaseSRWLockExclusive(&srwLock);

    return 0;
}

// This function moves a file from one folder to another
DWORD WINAPI MoveFileToAnotherFolder(LPVOID lpParam) {
    // Create a string that contains the thread ID
    std::stringstream ss;
    ss << GetCurrentThreadId();

    // Wait until all other threads have finished
    while (true) {
        // Check the completedThreads counter under the protection of the SRW lock
        AcquireSRWLockShared(&srwLock);
        bool allThreadsCompleted = (completedThreads == NUM_WRITER_THREADS + 1); // Increment by 1 to wait for the first move operation
        ReleaseSRWLockShared(&srwLock);

        if (allThreadsCompleted) {
            break;
        }

        // Avoid busy waiting
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
    }

    // Acquire the SRW lock in exclusive mode
    AcquireSRWLockExclusive(&srwLock);

    // Move the file and overwrite if the file already exists
    if (!MoveFileEx(L"C:\\Temp2\\log.txt", L"C:\\Temp3\\log.txt", MOVEFILE_REPLACE_EXISTING)) {
        // Handle the error if the file cannot be moved
        std::cerr << "MoveFileEx failed. Error: " << GetLastError() << std::endl;
        ReleaseSRWLockExclusive(&srwLock);
        return 1;
    }

    // Output a message to the console
    std::cout << "Mover Thread " << ss.str() << " moved the file to another folder.\n";

    // Release the SRW lock
    ReleaseSRWLockExclusive(&srwLock);

    return 0;
}

// The main function of the program
int main() {
    // Initialize the SRW lock
    InitializeSRWLock(&srwLock);

    // Create an array to hold the thread handles
    HANDLE hThreads[TOTAL_THREADS];

    // Create the writer threads
    for (int i = 0; i < NUM_WRITER_THREADS; i++) {
        // Create a new thread to write to the file
        hThreads[i] = CreateThread(NULL, 0, WriteToFile, NULL, 0, NULL);
        if (hThreads[i] == NULL) {
            // Handle the error if the thread cannot be created
            std::cerr << "CreateThread error: " << GetLastError() << '\n';
            return 1;
        }
    }

    // Create a new thread to move the file
    hThreads[NUM_WRITER_THREADS] = CreateThread(NULL, 0, MoveFileToFolder, NULL, 0, NULL);
    if (hThreads[NUM_WRITER_THREADS] == NULL) {
        // Handle the error if the thread cannot be created
        std::cerr << "CreateThread error: " << GetLastError() << '\n';
        return 1;
    }

    // Create a new thread to move the file to another folder
    hThreads[NUM_WRITER_THREADS + 1] = CreateThread(NULL, 0, MoveFileToAnotherFolder, NULL, 0, NULL);
    if (hThreads[NUM_WRITER_THREADS + 1] == NULL) {
        // Handle the error if the thread cannot be created
        std::cerr << "CreateThread error: " << GetLastError() << '\n';
        return 1;
    }

    // Wait for all threads to finish
    WaitForMultipleObjects(TOTAL_THREADS, hThreads, TRUE, INFINITE);

    // Close all thread handles
    for (int i = 0; i < TOTAL_THREADS; i++) {
        CloseHandle(hThreads[i]);
    }

    return 0;
}
```

The code is now running fine and has completed it tasks:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/7138b990-b01e-4ea4-83db-cf24f269b74e)


# Conclusion

| **Lesson**              | **Description**  |
|-------------------------|------------------|
| Always Release Locks    | One of the most common pitfalls in multithreading is forgetting to release a lock after acquiring it. If you don't release a lock, other threads waiting for it can be blocked indefinitely, leading to a deadlock. |
| Avoid Long-Held Locks   | Locks should be held for the shortest time possible. Long-held locks can lead to contention, where many threads are waiting for a lock, reducing the overall performance. |
| Understand Lock Modes   | The Slim Reader/Writer (SRW) Lock offers two modes: shared and exclusive. Multiple threads can hold a shared lock simultaneously, but only one thread can hold an exclusive lock. If a thread has an exclusive lock, no other thread can acquire the lock in either mode. |
