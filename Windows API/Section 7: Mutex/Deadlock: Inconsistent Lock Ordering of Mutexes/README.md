# What is the root cause of this deadlock within this code?

The root cause of this deadlock is the inconsistent ordering in which the two mutexes, **`hMutex`** and **`hMutex2`**, are acquired and released by the **`WriteToFile`** and **`ReadFromFile`** functions. **`WriteToFile`** first acquires **`hMutex`** and then tries to acquire **`hMutex2`**, while **`ReadFromFile`** does the opposite by first acquiring **`hMutex2`** and then trying to acquire **`hMutex`**. 

Compile the following code and run it:

```c
#include <windows.h>
#include <iostream>
#include <sstream>
#include <thread>
#include <vector>

// Define the number of threads and operations
#define NUM_THREADS 10
#define NUM_OPERATIONS 100

// Declare mutex handles
HANDLE hMutex;
HANDLE hMutex2; // Second mutex for introducing deadlock

// Function to write to a file
DWORD WINAPI WriteToFile(LPVOID lpParam) {
    std::stringstream ss;
    ss << GetCurrentThreadId();
    std::string data = "Writer Thread " + ss.str() + " is writing to the log.\r\n";

    for (int i = 0; i < NUM_OPERATIONS; i++) {
        // Acquire the first mutex
        WaitForSingleObject(hMutex, INFINITE);

        // Introducing "Hold and Wait" deadlock
        WaitForSingleObject(hMutex2, INFINITE);

        // Open the file for writing
        HANDLE hFile = CreateFile(L"C:\\Temp\\log.txt", GENERIC_WRITE, 0, NULL, OPEN_ALWAYS, FILE_ATTRIBUTE_NORMAL, NULL);

        // Check if the file opened successfully
        if (hFile == INVALID_HANDLE_VALUE) {
            std::cerr << "CreateFile (write) failed. Error: " << GetLastError() << std::endl;
            return 1;
        }

        // Move the file pointer to the end of the file
        SetFilePointer(hFile, 0, NULL, FILE_END);

        // Declare a variable to hold the number of bytes written
        DWORD bytesWritten;

        // Write to the file
        if (!WriteFile(hFile, data.c_str(), data.size(), &bytesWritten, NULL)) {
            std::cerr << "WriteFile failed. Error: " << GetLastError() << std::endl;
            CloseHandle(hFile);
            return 1;
        }

        // Output a message to the console
        std::cout << "Writer Thread " << ss.str() << " is writing to the log.\n";

        // Close the file
        CloseHandle(hFile);

        // Release the mutexes
        ReleaseMutex(hMutex);
        ReleaseMutex(hMutex2);

        // Pause for 100 milliseconds
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
    }

    return 0;
}

// Function to read from a file
DWORD WINAPI ReadFromFile(LPVOID lpParam) {
    std::stringstream ss;
    ss << GetCurrentThreadId();

    for (int i = 0; i < NUM_OPERATIONS; i++) {
        // Acquire the second mutex
        WaitForSingleObject(hMutex2, INFINITE);

        // Introducing "Hold and Wait" deadlock
        WaitForSingleObject(hMutex, INFINITE);

        // Open the file for reading
        HANDLE hFile = CreateFile(L"C:\\Temp\\log.txt", GENERIC_READ, 0, NULL, OPEN_ALWAYS, FILE_ATTRIBUTE_NORMAL, NULL);

        // Check if the file opened successfully
        if (hFile == INVALID_HANDLE_VALUE) {
            std::cerr << "CreateFile (read) failed. Error: " << GetLastError() << std::endl;
            return 1;
        }

        // Get the size of the file
        DWORD fileSize = GetFileSize(hFile, NULL);

        // Create a vector to hold the file content
        std::vector<char> buffer(fileSize + 1);

        // Declare a variable to hold the number of bytes read
        DWORD bytesRead;

        // Read from the file
        if (!ReadFile(hFile, buffer.data(), fileSize, &bytesRead, NULL)) {
            std::cerr << "ReadFile failed. Error: " << GetLastError() << std::endl;
            CloseHandle(hFile);
            return 1;
        }

        // Null-terminate the buffer and create a string with the content
        buffer[fileSize] = '\0';
        std::string content(buffer.begin(), buffer.end());

        // Find the last line in the file
        size_t lastNewlinePos = content.find_last_of("\n");
        std::string lastLine = content.substr(lastNewlinePos + 1);

        // Output the last line to the console
        std::cout << "Reader Thread " << ss.str() << " read from the log: " << lastLine << "\n";

        // Close the file
        CloseHandle(hFile);

        // Release the mutexes
        ReleaseMutex(hMutex2);
        ReleaseMutex(hMutex);

        // Pause for 100 milliseconds
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
    }

    return 0;
}

int main() {
    hMutex = CreateMutex(NULL, FALSE, L"NewMutex");
    if (hMutex == NULL) {
        std::cerr << "CreateMutex error: " << GetLastError() << '\n';
        return 1;
    }

    hMutex2 = CreateMutex(NULL, FALSE, L"NewMutex2");
    if (hMutex2 == NULL) {
        std::cerr << "CreateMutex error for hMutex2: " << GetLastError() << '\n';
        return 1;
    }

    // Create an array to hold the thread handles
    HANDLE hThreads[NUM_THREADS];

    // Create the threads
    for (int i = 0; i < NUM_THREADS; i++) {
        // Half of the threads will write to the file and the other half will read from it
        if (i < NUM_THREADS / 2) {
            hThreads[i] = CreateThread(NULL, 0, WriteToFile, NULL, 0, NULL);
        }
        else {
            hThreads[i] = CreateThread(NULL, 0, ReadFromFile, NULL, 0, NULL);
        }

        // Check if the thread was created successfully
        if (hThreads[i] == NULL) {
            std::cerr << "CreateThread error: " << GetLastError() << '\n';
            return 1;
        }
    }

    // Wait for all threads to finish
    WaitForMultipleObjects(NUM_THREADS, hThreads, TRUE, INFINITE);

    // Close all thread handles
    for (int i = 0; i < NUM_THREADS; i++) {
        CloseHandle(hThreads[i]);
    }

    // Close both mutexes
    CloseHandle(hMutex);
    CloseHandle(hMutex2);

    return 0;
}
```

The program starts hanging as we can see here:

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/9cb1fb9e-00fe-4fa5-81d6-ef3bba3529a6)

Open **Process Explorer** and view the wait reasons of this program. Further, take a memory dump while the program is hanging.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/a5b21920-d8f8-45ad-8b50-a8f01a81561e)

# WinDbg Walk Through - Analyzing Memory Dump

Run the **!mex.p** command to get the current context of the process:

```
0:000> !mex.p
Name     Ses PID            PEB              Mods Handle Thrd
======== === ============== ================ ==== ====== ====
test.exe   1 28b8 (0n10424) 00000044d545d000    4     53   11

CommandLine: Test.exe
Last event: 28b8.2ee0: Break instruction exception - code 80000003 (first/second chance not available)

Show Threads: Unique Stacks    !listthreads (!lt)    ~*kv
```

We can see that it has **11** threads in total. Let's now look for all threads that are in a waiting state:

```
0:000> !mex.us -w
1 thread [stats]: 0
    00007ff92bb6f8a4 ntdll!NtWaitForMultipleObjects+0x14      
    00007ff92918f5a9 KERNELBASE!WaitForMultipleObjectsEx+0xe9 
    00007ff92918f4ae KERNELBASE!WaitForMultipleObjects+0xe    
    00007ff74a342ed7 Test!main+0x117                          (C:\Users\User\source\repos\Test\Test\Test.cpp @ 163)
    (Inline)         Test!invoke_main+0x22                    (D:\a\_work\1\s\src\vctools\crt\vcstartup\src\startup\exe_common.inl @ 78)
    00007ff74a34aa10 Test!__scrt_common_main_seh+0x10c        (D:\a\_work\1\s\src\vctools\crt\vcstartup\src\startup\exe_common.inl @ 288)
    00007ff92ad326ad kernel32!BaseThreadInitThunk+0x1d        
    00007ff92bb2aa68 ntdll!RtlUserThreadStart+0x28            

1 thread [stats]: 4
    00007ff92bb6edd4 ntdll!NtWaitForSingleObject+0x14      
    00007ff929163f8e KERNELBASE!WaitForSingleObjectEx+0x8e 
    00007ff74a342324 Test!WriteToFile+0x2e4                (C:\Users\User\source\repos\Test\Test\Test.cpp @ 29)
    00007ff92ad326ad kernel32!BaseThreadInitThunk+0x1d     
    00007ff92bb2aa68 ntdll!RtlUserThreadStart+0x28         

1 thread [stats]: 6
    00007ff92bb6edd4 ntdll!NtWaitForSingleObject+0x14      
    00007ff929163f8e KERNELBASE!WaitForSingleObjectEx+0x8e 
    00007ff74a342792 Test!ReadFromFile+0xa2                (C:\Users\User\source\repos\Test\Test\Test.cpp @ 80)
    00007ff92ad326ad kernel32!BaseThreadInitThunk+0x1d     
    00007ff92bb2aa68 ntdll!RtlUserThreadStart+0x28         

4 threads [stats]: 7 8 9 10
    00007ff92bb6edd4 ntdll!NtWaitForSingleObject+0x14      
    00007ff929163f8e KERNELBASE!WaitForSingleObjectEx+0x8e 
    00007ff74a342780 Test!ReadFromFile+0x90                (C:\Users\User\source\repos\Test\Test\Test.cpp @ 77)
    00007ff92ad326ad kernel32!BaseThreadInitThunk+0x1d     
    00007ff92bb2aa68 ntdll!RtlUserThreadStart+0x28         

4 threads [stats]: 1 2 3 5
    00007ff92bb6edd4 ntdll!NtWaitForSingleObject+0x14      
    00007ff929163f8e KERNELBASE!WaitForSingleObjectEx+0x8e 
    00007ff74a342312 Test!WriteToFile+0x2d2                (C:\Users\User\source\repos\Test\Test\Test.cpp @ 26)
    00007ff92ad326ad kernel32!BaseThreadInitThunk+0x1d     
    00007ff92bb2aa68 ntdll!RtlUserThreadStart+0x28         

Threads matching filter: 11 out of 11
```

All threads are currently in a waiting state. Let's now dump the Mutex from the memory dumps, but use the **!mex.grep** to filter only on the Mutexes that we implemented within the code:

```
0:000> !grep -A 5 "NewMutex" !handle 0 f mutant
  Name         	\Sessions\1\BaseNamedObjects\NewMutex
  Object specific information
    Mutex is Owned
    Mutant Owner 28b8.33e4
Handle 00000000000000b0
  Type         	Mutant
  Name         	\Sessions\1\BaseNamedObjects\NewMutex2
  Object specific information
    Mutex is Owned
    Mutant Owner 28b8.b20
3 handles of type Mutant
```

The output shows that thread ID **33e4** is currently owning the mutex **"NewMutex"**, while thread ID **b20** is owning the **NewMutex2** one. When a mutex is said to be "owned," it means that a particular thread has currently acquired or locked that mutex and has exclusive rights to the protected resource or section of code.

Thread ID **33e4** indicates that a thread is currently waiting on an object (likely a mutex or event) within the **`WriteToFile`** function:

```
0:000> !mex.t -t 33e4
DbgID ThreadID       User Kernel Create Time (UTC)
4     33e4 (0n13284)    0      0 08/12/2023 11:04:45.862 AM

# Child-SP         Return           Call Site                             Source
0 00000044d59ffb38 00007ff929163f8e ntdll!NtWaitForSingleObject+0x14      
1 00000044d59ffb40 00007ff74a342324 KERNELBASE!WaitForSingleObjectEx+0x8e 
2 00000044d59ffbe0 00007ff92ad326ad Test!WriteToFile+0x2e4                C:\Users\User\source\repos\Test\Test\Test.cpp @ 29
3 00000044d59ffe00 00007ff92bb2aa68 kernel32!BaseThreadInitThunk+0x1d     
4 00000044d59ffe30 0000000000000000 ntdll!RtlUserThreadStart+0x28
```

This also applies for thread ID **b20** that is also waiting on an object, but within the **`ReadFromFile`** function:

```
0:004> !mex.t -t b20
DbgID ThreadID     User Kernel Create Time (UTC)
6     b20 (0n2848)    0      0 08/12/2023 11:04:45.862 AM

# Child-SP         Return           Call Site                             Source
0 00000044d5bff6a8 00007ff929163f8e ntdll!NtWaitForSingleObject+0x14      
1 00000044d5bff6b0 00007ff74a342792 KERNELBASE!WaitForSingleObjectEx+0x8e 
2 00000044d5bff750 00007ff92ad326ad Test!ReadFromFile+0xa2                C:\Users\User\source\repos\Test\Test\Test.cpp @ 80
3 00000044d5bffa80 00007ff92bb2aa68 kernel32!BaseThreadInitThunk+0x1d     
4 00000044d5bffab0 0000000000000000 ntdll!RtlUserThreadStart+0x28         
```

When we see the function **NtWaitForSingleObject** or **WaitForSingleObjectEx** at the top of our call stack, it provides a clear indication that our thread is currently waiting for a synchronization object, such as a mutex, event, or semaphore to be signaled or released. During this waiting period, no operations or tasks are being performed by that thread. The thread isn't executing any other parts of our code or making any progress.
