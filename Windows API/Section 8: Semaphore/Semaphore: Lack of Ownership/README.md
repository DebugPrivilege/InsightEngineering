# What is Lack of Ownership?

"Lack of ownership" refers to the concept that certain synchronization primitives, like semaphores, don't "own" the specific thread that acquires them. Any thread can release the semaphore, regardless of which thread originally acquired it. Lack of ownership can lead to potential synchronization issues, such as deadlocks, if not managed carefully.

**Description:**

The danger in this code comes from the lack of strict ownership over the semaphore. The **`MaliciousFunction`** captures the semaphore and never releases it. This behavior is possible because semaphores don't tie ownership to a specific thread. Any thread can release a semaphore, regardless of which thread acquired it. This allows potential misuse, as seen here, where other threads are left waiting indefinitely due to one misbehaving thread:

```c
#include <windows.h>
#include <iostream>
#include <sstream>
#include <thread>
#include <vector>

// Define the number of threads for file operations and total threads
#define NUM_FILE_OP_THREADS 10
#define TOTAL_THREADS (NUM_FILE_OP_THREADS + 2)  // +1 for the file mover thread, +1 for the malicious thread
#define NUM_OPERATIONS 100

// Define the semaphore name
#define SEMAPHORE_NAME L"FileOperationSyncSemaphore"

// Define the semaphore
HANDLE hSemaphore;

// Function to write to a file
DWORD WINAPI WriteToFile(LPVOID lpParam) {
    // Create a string that contains the thread ID
    std::stringstream ss;
    ss << GetCurrentThreadId();
    std::string data = "Writer Thread " + ss.str() + " is writing to the log.\r\n";

    // Repeat the write operation a certain number of times
    for (int i = 0; i < NUM_OPERATIONS; i++) {
        // Wait for the semaphore to be available
        WaitForSingleObject(hSemaphore, INFINITE);

        // Open the file for writing
        HANDLE hFile = CreateFile(L"C:\\Temp\\log.txt", GENERIC_WRITE, 0, NULL, OPEN_ALWAYS, FILE_ATTRIBUTE_NORMAL, NULL);
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

        // Release the semaphore
        ReleaseSemaphore(hSemaphore, 1, NULL);

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
        // Wait for the semaphore to be available
        WaitForSingleObject(hSemaphore, INFINITE);

        // Open the file for reading
        HANDLE hFile = CreateFile(L"C:\\Temp\\log.txt", GENERIC_READ, 0, NULL, OPEN_ALWAYS, FILE_ATTRIBUTE_NORMAL, NULL);
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

        // Release the semaphore
        ReleaseSemaphore(hSemaphore, 1, NULL);

        // Pause for 100 milliseconds
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
    }

    return 0;
}

// Function to move a file
DWORD WINAPI MoveFileToFolder(LPVOID lpParam) {
    // Create a string that contains the thread ID
    std::stringstream ss;
    ss << GetCurrentThreadId();
    std::string data = "Mover Thread " + ss.str() + " is moving the log.\r\n";

    // Wait for the semaphore to be available
    WaitForSingleObject(hSemaphore, INFINITE);

    // Open the file for reading
    HANDLE hFile = CreateFile(L"C:\\Temp\\log.txt", GENERIC_READ, 0, NULL, OPEN_EXISTING, FILE_ATTRIBUTE_NORMAL, NULL);
    if (hFile == INVALID_HANDLE_VALUE) {
        std::cerr << "CreateFile (move) failed. Error: " << GetLastError() << std::endl;
        return 1;
    }

    // Close the file
    CloseHandle(hFile);

    // Move the file and overwrite if the file already exists
    if (!MoveFileEx(L"C:\\Temp\\log.txt", L"C:\\Temp2\\log.txt", MOVEFILE_REPLACE_EXISTING)) {
        std::cerr << "MoveFileEx failed. Error: " << GetLastError() << std::endl;
        return 1;
    }

    // Output a message to the console
    std::cout << "Mover Thread " << ss.str() << " moved the log.\n";

    // Release the semaphore
    ReleaseSemaphore(hSemaphore, 1, NULL);

    return 0;
}


// Function to maliciously acquire the semaphore without releasing
DWORD WINAPI MaliciousFunction(LPVOID lpParam) {
    WaitForSingleObject(hSemaphore, INFINITE);
    std::cout << "Malicious thread has acquired the semaphore and will not release it.\n";
    // Intentionally not releasing the semaphore.
    // This mimics an external process/thread acquiring the semaphore and not behaving correctly.
    while (true) { std::this_thread::sleep_for(std::chrono::hours(1)); } // Keep the thread running
    return 0;
}

int main() {
    // Create the named semaphore
    hSemaphore = CreateSemaphore(NULL, 1, 1, SEMAPHORE_NAME);
    if (hSemaphore == NULL) {
        std::cerr << "CreateSemaphore error: " << GetLastError() << '\n';
        return 1;
    }

    // Create an array to hold the thread handles
    HANDLE hThreads[TOTAL_THREADS];

    // Create the threads for file operations
    for (int i = 0; i < NUM_FILE_OP_THREADS; i++) {
        // Half of the threads will write to the file and the other half will read from it
        if (i < NUM_FILE_OP_THREADS / 2) {
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

    // Wait for all threads for file operations to finish
    WaitForMultipleObjects(NUM_FILE_OP_THREADS, hThreads, TRUE, INFINITE);

    // Start the malicious thread after all the read and write threads
    HANDLE hMaliciousThread = CreateThread(NULL, 0, MaliciousFunction, NULL, 0, NULL);
    if (hMaliciousThread == NULL) {
        std::cerr << "CreateThread error: " << GetLastError() << '\n';
        return 1;
    }

    // Let the malicious thread acquire the semaphore
    std::this_thread::sleep_for(std::chrono::seconds(2));

    // Create the thread to move the file
    HANDLE hMoveThread = CreateThread(NULL, 0, MoveFileToFolder, NULL, 0, NULL);
    if (hMoveThread == NULL) {
        std::cerr << "CreateThread error: " << GetLastError() << '\n';
        return 1;
    }

    // Wait for move thread to finish (this will hang indefinitely due to the malicious thread)
    WaitForSingleObject(hMoveThread, INFINITE);

    // Close all thread handles for file operations and malicious thread
    for (int i = 0; i < TOTAL_THREADS; i++) {
        CloseHandle(hThreads[i]);
    }
    CloseHandle(hMaliciousThread);
    CloseHandle(hMoveThread);

    // Close the semaphore
    CloseHandle(hSemaphore);

    return 0;
}
```

When we run the program, it starts to hang as we can see here:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/c10fc73e-b399-433b-ab18-b6001e036f17)


Open **Process Explorer** and we can see that the call stack indicates that a thread originating from **`MoveFileToFolder`** is waiting for a synchronization object:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/071a0305-eeb4-4d89-af32-ec28fa3ef7f5)


Create a memory dump of the program while it is hanging:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/8b832c15-b933-430a-bd24-a7e51cee8e21)


# Flow of the deadlock

1. **`MoveFileToFolder`** attempts to acquire the semaphore but can't, as **`MaliciousFunction`** has already taken it and never intends to release.
2. **`main`** function waits indefinitely for **`MoveFileToFolder`** to finish its operation. However, **`MoveFileToFolder`** is stuck because of the semaphore situation.

This creates a situation where the program cannot proceed, hence the deadlock.

```
Thread Creation Flow:

[main()]
   |
   |---[WriteToFile() x 5]----------------------\
   |                                             |
   |---[ReadFromFile() x 5]----------------------|---> WaitForMultipleObjects() 
                                                  |     (Waits for all these threads to complete)
   |                                             |
   |---[MaliciousFunction()] --------------------/
   |
   +---> std::this_thread::sleep_for() (Ensures MaliciousFunction() gets the semaphore first)
   |
   |---[MoveFileToFolder()]
   |
   +---> WaitForSingleObject(hMoveThread) 
         (Here's the DEADLOCK: Waits indefinitely due to MaliciousFunction() holding the semaphore)

Semaphore Usage Flow:

[Semaphore] (Initially available, named "FileOperationSyncSemaphore")
   |
   +---> [WriteToFile()/ReadFromFile()] 
         (Multiple threads use it, acquiring and releasing in proper manner multiple times)
   |
   +---> [MaliciousFunction()] (This is where the problem starts: Acquires and NEVER releases)
   |
   X---> [MoveFileToFolder()] 
         (Attempts to acquire, but DEADLOCKED as it waits indefinitely due to MaliciousFunction())
```

# WinDbg Walk Through - Analyzing Memory Dump

Load the **MEX** extension in WinDbg:

```
0:000> .load mex
Mex External 3.0.0.7172 Loaded!
```

Run the **!mex.p** command to get the current context of the process:

```
0:000> !mex.p
Name     Ses PID           PEB              Mods Handle Thrd
======== === ============= ================ ==== ====== ====
test.exe   1 1c9c (0n7324) 000000978c3d5000    4     54    4

CommandLine: Test.exe
Last event: 1c9c.3240: Break instruction exception - code 80000003 (first/second chance not available)

Show Threads: Unique Stacks    !listthreads (!lt)    ~*kv
```

There are **4** threads within this process. Since we already know that they are all in a waiting state. Let's verify that with the **!mex.us -w** command that will return threads that are currently in a waiting state:

```
0:000> !us -w
1 thread [stats]: 0
    00007ff92bb6edd4 ntdll!NtWaitForSingleObject+0x14      
    00007ff929163f8e KERNELBASE!WaitForSingleObjectEx+0x8e 
    00007ff6dd6535e1 Test!main+0x1b1                       (C:\Users\User\source\repos\Test\Test\Test.cpp @ 219)
    (Inline)         Test!invoke_main+0x22                 (D:\a\_work\1\s\src\vctools\crt\vcstartup\src\startup\exe_common.inl @ 78)
    00007ff6dd65b0c0 Test!__scrt_common_main_seh+0x10c     (D:\a\_work\1\s\src\vctools\crt\vcstartup\src\startup\exe_common.inl @ 288)
    00007ff92ad326ad kernel32!BaseThreadInitThunk+0x1d     
    00007ff92bb2aa68 ntdll!RtlUserThreadStart+0x28         

1 thread [stats]: 1
    00007ff92bb6f3d4 ntdll!NtDelayExecution+0x14                                                                                                  
    00007ff92bb25193 ntdll!RtlDelayExecution+0x43                                                                                                 
    00007ff92917444d KERNELBASE!SleepEx+0x7d                                                                                                      
    00007ff6dd65a8d4 Test!_Thrd_sleep+0x3c                                                                                                        (D:\a\_work\1\s\src\vctools\crt\github\stl\src\cthread.cpp @ 80)
    00007ff6dd6585fe Test!std::this_thread::sleep_until<std::chrono::steady_clock,std::chrono::duration<__int64,std::ratio<1,1000000000> > >+0xfe (C:\Program Files\Microsoft Visual Studio\2022\Community\VC\Tools\MSVC\14.36.32532\include\thread @ 199)
    (Inline)         Test!std::this_thread::sleep_for+0x52                                                                                        (C:\Program Files\Microsoft Visual Studio\2022\Community\VC\Tools\MSVC\14.36.32532\include\thread @ 204)
    00007ff6dd653419 Test!MaliciousFunction+0xa9                                                                                                  (C:\Users\User\source\repos\Test\Test\Test.cpp @ 163)
    00007ff92ad326ad kernel32!BaseThreadInitThunk+0x1d                                                                                            
    00007ff92bb2aa68 ntdll!RtlUserThreadStart+0x28                                                                                                

1 thread [stats]: 2
    00007ff92bb6edd4 ntdll!NtWaitForSingleObject+0x14      
    00007ff929163f8e KERNELBASE!WaitForSingleObjectEx+0x8e 
    00007ff6dd653106 Test!MoveFileToFolder+0x2c6           (C:\Users\User\source\repos\Test\Test\Test.cpp @ 132)
    00007ff92ad326ad kernel32!BaseThreadInitThunk+0x1d     
    00007ff92bb2aa68 ntdll!RtlUserThreadStart+0x28         

1 thread [stats]: 3
    00007ff92bb729a4 ntdll!NtWaitForWorkViaWorkerFactory+0x14
    00007ff92bb0536e ntdll!TppWorkerThread+0x2ee
    00007ff92ad326ad kernel32!BaseThreadInitThunk+0x1d
    00007ff92bb2aa68 ntdll!RtlUserThreadStart+0x28

Threads matching filter: 4 out of 4
```

Let's dump out the semaphore objects that resides within this memory dump:

```
0:000> !handle 0 f semaphore
Handle 0000000000000078
  Type         	Semaphore
  Attributes   	0
  GrantedAccess	0x1f0003:
         Delete,ReadControl,WriteDac,WriteOwner,Synch
         QueryState,ModifyState
  HandleCount  	2
  PointerCount 	65538
  Name         	\Sessions\1\BaseNamedObjects\SM0:7324:304:WilStaging_02_p0
  Object specific information
    Semaphore Count 1736680388
    Semaphore Limit 1736680388
Handle 000000000000007c
  Type         	Semaphore
  Attributes   	0
  GrantedAccess	0x1f0003:
         Delete,ReadControl,WriteDac,WriteOwner,Synch
         QueryState,ModifyState
  HandleCount  	2
  PointerCount 	65538
  Name         	\Sessions\1\BaseNamedObjects\SM0:7324:304:WilStaging_02_p0h
  Object specific information
    Semaphore Count 274
    Semaphore Limit 274
Handle 00000000000000a8
  Type         	Semaphore
  Attributes   	0
  GrantedAccess	0x1f0003:
         Delete,ReadControl,WriteDac,WriteOwner,Synch
         QueryState,ModifyState
  HandleCount  	2
  PointerCount 	63537
  Name         	\Sessions\1\BaseNamedObjects\FileOperationSyncSemaphore
  Object specific information
    Semaphore Count 0
    Semaphore Limit 1
3 handles of type Semaphore
```

We can use the **!mex.grep** command to only display the named semaphore object we're interested in:

```
0:000> !grep -A 5 "FileOperationSyncSemaphore" !handle 0 f semaphore
  Name         	\Sessions\1\BaseNamedObjects\FileOperationSyncSemaphore
  Object specific information
    Semaphore Count 0
    Semaphore Limit 1
3 handles of type Semaphore
```

To interpret the output that we're seeing:

If a semaphore's count is 0, this means that the semaphore has been acquired by some thread or process, and no other threads or processes can acquire it until it's released. If we see a semaphore count that's not zero, it means we still have available slots to acquire the semaphore before reaching its limit.
