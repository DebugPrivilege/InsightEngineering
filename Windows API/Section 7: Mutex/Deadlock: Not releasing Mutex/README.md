# What is the root cause of this deadlock within this code sample?

The purpose of this code is the following:

- To spawn **10** threads, where **5** threads execute the **`WriteToFile`** function to write data to a log file, and the other **5** threads run the **`ReadFromFile`** function to read and display the last line of the same file.
- To display each thread's activity on the console, indicating which threads are writing to the file and which ones are reading from it.

**Description:**

The root cause of the deadlock is that the main thread immediately claims the mutex using **CreateMutex** set with **TRUE** at the **bInitialOwner** parameter, meaning it owns the mutex from the start. However, the main function **never releases** this ownership using **ReleaseMutex**. When worker threads try to access the mutex through **WaitForSingleObject**, they can't, as the main thread has it locked and never lets it go.

Compile and run the following code:

```c
#include <windows.h>
#include <iostream>
#include <sstream>
#include <thread>
#include <vector>

// Define the number of threads and operations
#define NUM_THREADS 10
#define NUM_OPERATIONS 100

// Declare a mutex handle
HANDLE hMutex;

// Function to write to a file
DWORD WINAPI WriteToFile(LPVOID lpParam) {
    // Create a string that contains the thread ID
    std::stringstream ss;
    ss << GetCurrentThreadId();
    std::string data = "Writer Thread " + ss.str() + " is writing to the log.\r\n";

    // Repeat the write operation a certain number of times
    for (int i = 0; i < NUM_OPERATIONS; i++) {
        // Wait for the mutex before proceeding
        WaitForSingleObject(hMutex, INFINITE);

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
        // Release the mutex
        ReleaseMutex(hMutex);

        // Pause for 100 milliseconds
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
    }

    return 0;
}

// Function to read from a file
DWORD WINAPI ReadFromFile(LPVOID lpParam) {
    // Create a string that contains the thread ID
    std::stringstream ss;
    ss << GetCurrentThreadId();

    // Repeat the read operation a certain number of times
    for (int i = 0; i < NUM_OPERATIONS; i++) {
        // Wait for the mutex before proceeding
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
        // Release the mutex
        ReleaseMutex(hMutex);

        // Pause for 100 milliseconds
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
    }

    return 0;
}

// The main function
int main() {
    // Create a mutex
    hMutex = CreateMutex(NULL, TRUE, L"NewMutex");
    // Check if the mutex was created successfully
    if (hMutex == NULL) {
        std::cerr << "CreateMutex error: " << GetLastError() << '\n';
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

    // Close the mutex
    CloseHandle(hMutex);

    return 0;
}
```

When we run this program, it will start to hang as we can see here:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/a22fe6d5-d2a0-4bc1-96df-4418d60ab0ad)


Open **Process Explorer** and we can see multiple threads are currently waiting:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/4a355718-8e02-4d12-a61f-be09377f9f30)


Take a memory dump of the process and load it in WinDbg:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/e5bb03da-298b-4366-bc23-2493649b772a)


# WinDbg Walk Through - Analyzing Memory Dump

Load the **MEX** extension:

```
0:000> .load mex
Mex External 3.0.0.7172 Loaded!
```

Start with listening all the threads:

```
0:000> !mex.lt
 # DbgID ThdID Wait Function          User Kernel Info     TEB              Create Time
== ===== ===== ====================== ==== ====== ======== ================ ==========================
->     0  23a0 Test!main+0xe7            0      0 Event... 00000061cc761000 08/12/2023 08:40:47.058 AM
       1   f20 Test!WriteToFile+0x2d2    0      0          00000061cc763000 08/12/2023 08:40:47.072 AM
       2   b9c Test!WriteToFile+0x2d2    0      0          00000061cc765000 08/12/2023 08:40:47.072 AM
       3   be0 Test!WriteToFile+0x2d2    0      0          00000061cc767000 08/12/2023 08:40:47.072 AM
       4  2ca8 Test!WriteToFile+0x2d2    0      0          00000061cc769000 08/12/2023 08:40:47.072 AM
       5  243c Test!WriteToFile+0x2d2    0      0          00000061cc76b000 08/12/2023 08:40:47.072 AM
       6  22fc Test!ReadFromFile+0x90    0      0          00000061cc76d000 08/12/2023 08:40:47.072 AM
       7  21dc Test!ReadFromFile+0x90    0      0          00000061cc76f000 08/12/2023 08:40:47.072 AM
       8   704 Test!ReadFromFile+0x90    0      0          00000061cc771000 08/12/2023 08:40:47.072 AM
       9  2b28 Test!ReadFromFile+0x90    0      0          00000061cc773000 08/12/2023 08:40:47.072 AM
      10  2fa4 Test!ReadFromFile+0x90    0      0          00000061cc775000 08/12/2023 08:40:47.072 AM
```

We have **11** threads in total, and we knew from **Process Explorer** that they are all waiting. A cool trick we can use within **MEX** is the **!mex.us -w** command to only show the waiting threads:

```
0:000> !mex.us -w
1 thread [stats]: 0
    00007ff92bb6f8a4 ntdll!NtWaitForMultipleObjects+0x14      
    00007ff92918f5a9 KERNELBASE!WaitForMultipleObjectsEx+0xe9 
    00007ff92918f4ae KERNELBASE!WaitForMultipleObjects+0xe    
    00007ff6bbf12e67 Test!main+0xe7                           (C:\Users\User\source\repos\Test\Test\Test.cpp @ 146)
    (Inline)         Test!invoke_main+0x22                    (D:\a\_work\1\s\src\vctools\crt\vcstartup\src\startup\exe_common.inl @ 78)
    00007ff6bbf1a990 Test!__scrt_common_main_seh+0x10c        (D:\a\_work\1\s\src\vctools\crt\vcstartup\src\startup\exe_common.inl @ 288)
    00007ff92ad326ad kernel32!BaseThreadInitThunk+0x1d        
    00007ff92bb2aa68 ntdll!RtlUserThreadStart+0x28            

5 threads [stats]: 6 7 8 9 10
    00007ff92bb6edd4 ntdll!NtWaitForSingleObject+0x14      
    00007ff929163f8e KERNELBASE!WaitForSingleObjectEx+0x8e 
    00007ff6bbf12760 Test!ReadFromFile+0x90                (C:\Users\User\source\repos\Test\Test\Test.cpp @ 73)
    00007ff92ad326ad kernel32!BaseThreadInitThunk+0x1d     
    00007ff92bb2aa68 ntdll!RtlUserThreadStart+0x28         

5 threads [stats]: 1 2 3 4 5
    00007ff92bb6edd4 ntdll!NtWaitForSingleObject+0x14      
    00007ff929163f8e KERNELBASE!WaitForSingleObjectEx+0x8e 
    00007ff6bbf12312 Test!WriteToFile+0x2d2                (C:\Users\User\source\repos\Test\Test\Test.cpp @ 27)
    00007ff92ad326ad kernel32!BaseThreadInitThunk+0x1d     
    00007ff92bb2aa68 ntdll!RtlUserThreadStart+0x28         

Threads matching filter: 11 out of 11
```

This output indicates that both **`WriteToFile`** and **`ReadFromFile`** are currently waiting on an object, which happens to be a Mutex. However, the **`main`** thread takes initial ownership of the mutex by setting the **bInitialOwner** parameter to **TRUE** in the **CreateMutex** function, but it never releases it. This prevents **`WriteToFile`** and **`ReadFromFile`** functions from obtaining the mutex to execute their tasks.

Let's dump out all the Mutex that were identified within the memory dump:

```
0:000> !handle 0 f mutant
Handle 0000000000000078
  Type         	Mutant
  Attributes   	0
  GrantedAccess	0x1f0001:
         Delete,ReadControl,WriteDac,WriteOwner,Synch
         QueryState
  HandleCount  	2
  PointerCount 	65536
  Name         	\Sessions\1\BaseNamedObjects\SM0:11736:304:WilStaging_02
  Object specific information
    Mutex is Free
    Mutant Owner 0.0
Handle 00000000000000b0
  Type         	Mutant
  Attributes   	0
  GrantedAccess	0x1f0001:
         Delete,ReadControl,WriteDac,WriteOwner,Synch
         QueryState
  HandleCount  	2
  PointerCount 	65538
  Name         	\Sessions\1\BaseNamedObjects\NewMutex
  Object specific information
    Mutex is Owned
    Mutant Owner 2dd8.23a0
2 handles of type Mutant
```

Let's filter on our Mutex name to be more precise in the results of the output:

```
0:000> !grep -A 5 "NewMutex" !handle 0 f mutant
  Name         	\Sessions\1\BaseNamedObjects\NewMutex
  Object specific information
    Mutex is Owned
    Mutant Owner 2dd8.23a0
2 handles of type Mutant
```

Let's take a step back and understand what we're seeing in this output. The first thing of interest that we can see is the **"Mutex is Owned"** description. This indicates that a thread currently holds (or "owns") the mutex. When a mutex is owned, no other thread can acquire it until the current owning thread releases it.

The second thing of interest is **"Mutant Owner 2dd8.23a0"**. This specifies the ID of the thread (and its process) that currently owns the mutex. The format is "processID.threadID". In this case, the process ID is **2dd8** and the thread ID is **23a0**.

Here we can indeed see that **2dd8** is the Process ID:

```
0:000> !mex.p -p 2dd8
Name     Ses PID            PEB              Mods Handle Thrd
======== === ============== ================ ==== ====== ====
test.exe   1 2dd8 (0n11736) 00000061cc760000    4     52   11

CommandLine: Test.exe
Last event: 2dd8.23a0: Break instruction exception - code 80000003 (first/second chance not available)

Show Threads: Unique Stacks    !listthreads (!lt)    ~*kv
```

Here we can also see that **23a0** is indeed the thread ID. It also indicates that the **`main`** function holding the mutex.

```
0:000> !mex.t -t 23a0
DbgID ThreadID      User Kernel Create Time (UTC)
0     23a0 (0n9120)    0      0 08/12/2023 08:40:47.058 AM

# Child-SP         Return           Call Site                                Source
0 00000061cc8ffa08 00007ff92918f5a9 ntdll!NtWaitForMultipleObjects+0x14      
1 00000061cc8ffa10 00007ff92918f4ae KERNELBASE!WaitForMultipleObjectsEx+0xe9 
2 00000061cc8ffcf0 00007ff6bbf12e67 KERNELBASE!WaitForMultipleObjects+0xe    
3 00000061cc8ffd30 00007ff6bbf1a990 Test!main+0xe7                           C:\Users\User\source\repos\Test\Test\Test.cpp @ 146
4 (Inline)         ---------------- Test!invoke_main+0x22                    D:\a\_work\1\s\src\vctools\crt\vcstartup\src\startup\exe_common.inl @ 78
5 00000061cc8ffdd0 00007ff92ad326ad Test!__scrt_common_main_seh+0x10c        D:\a\_work\1\s\src\vctools\crt\vcstartup\src\startup\exe_common.inl @ 288
6 00000061cc8ffe10 00007ff92bb2aa68 kernel32!BaseThreadInitThunk+0x1d        
7 00000061cc8ffe40 0000000000000000 ntdll!RtlUserThreadStart+0x28
```

# Fix

When we set the **bInitialOwner** parameter to **FALSE** during the creation of the mutex, no thread initially owns the mutex. Therefore, when the worker threads start and attempt to acquire the mutex, they can successfully obtain it without any issues.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/6d773ad1-64b7-40f4-a58c-27972bd25aef)


If **bInitialOwner** is set to **FALSE:** No thread owns the mutex immediately after creation. The first thread that tries to acquire the mutex using **WaitForSingleObject** (or a related function) will take ownership. That thread then becomes responsible for releasing the mutex. 
