# What is the root cause of this deadlock within this code?

In order to demonstrate this demo. Please follow the following steps:

- Create the following 2 folders:
  1. **C:\Temp**
  2. **C:\Temp2**

Once we've done this step, we can continue.

The purpose of this code is the following:

- The **`CreateAndWriteFiles`** function generates and writes to 1000 unique files in the **`C:\Temp`** directory, each containing the message "Hello, World!\n" repeated 100 times.
- The **`MoveFilesToNewDirectory`** function waits for all files to be created, then moves each of the **1000** files from **`C:\Temp`** to **`C:\Temp2`**.
- The main program sets up synchronization between these functions using two events and executes them on separate threads.

**Description:**

The deadlock in this code is **from the order** in which the two threads, **`CreateAndWriteFiles`** and **`MoveFilesToNewDirectory`**, wait on two synchronization events (**`hFilesCreatedEvent`** and **`hFilesMovedEvent`**).

Here is the core problem:

1. The **`CreateAndWriteFiles`** thread is waiting for the **`MoveFilesToNewDirectory`** thread to move all files (by waiting for **`hFilesMovedEvent`** to be signaled).
2. The **`MoveFilesToNewDirectory`** thread is waiting for the **`CreateAndWriteFiles`** thread to finish creating all files (by waiting for **`hFilesCreatedEvent`** to be signaled).

Even after all files are created, the **`CreateAndWriteFiles`** thread is still waiting for files to be moved before it signals the **`hFilesCreatedEvent`**. While the **`MoveFilesToNewDirectory`** thread is waiting for the **`hFilesCreatedEvent`** to be signaled before it can start moving the files.

Compile and run the following code:

```c
#include <windows.h>
#include <iostream>
#include <string>
#include <random>
#include <tlhelp32.h>

#define NUM_FILES 1000

HANDLE hFilesCreatedEvent, hFilesMovedEvent;  // Events for signaling file creation and moving completion

std::vector<std::wstring> createdFiles;  // Stores the names of the created files

// This function creates files and writes to them
DWORD WINAPI CreateAndWriteFiles(LPVOID lpParam) {
    std::random_device rd;
    std::mt19937 gen(rd());

    // Create and write to NUM_FILES number of files
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
                static_cast<DWORD>(data.size()),
                &bytesWritten,
                NULL
            )) {
                std::wcerr << L"WriteFile failed for " << fileName << L". Error: " << GetLastError() << std::endl;
                break;
            }
        }

        CloseHandle(hFile);
        createdFiles.push_back(fileName);
        std::wcout << L"Created and wrote to file: " << fileName << std::endl;
    }

    // DEADLOCK: This part of the code is waiting for the files to be moved.
    // But the moving can't start until this part of the code says it's done.
    // So they are both waiting for each other, and nothing can happen. 
    WaitForSingleObject(hFilesMovedEvent, INFINITE);

    // Signal that all files are created
    SetEvent(hFilesCreatedEvent);

    return 0;
}

// This function moves the created files to a new directory
DWORD WINAPI MoveFilesToNewDirectory(LPVOID lpParam) {
    // DEADLOCK: This part of the code is waiting for the files to be created.
    // But the creating can't finish until this part of the code is done moving.
    // So they are both waiting for each other, and nothing can happen. 
    WaitForSingleObject(hFilesCreatedEvent, INFINITE);

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

    // Signal that all files are created
    SetEvent(hFilesCreatedEvent);

    return 0;
}

int main() {
    DWORD threadID;  // Variable to store the thread identifier
    HANDLE hThreads[2];  // Array to store the handles of the threads

    // Create two separate events
    hFilesCreatedEvent = CreateEvent(NULL, TRUE, FALSE, L"Global\\FilesCreatedEvent");
    hFilesMovedEvent = CreateEvent(NULL, TRUE, FALSE, L"Global\\FilesMovedEvent");
    if (hFilesCreatedEvent == NULL || hFilesMovedEvent == NULL) {
        std::cerr << "Failed to create event(s). Error: " << GetLastError() << std::endl;
        return 1;
    }

    // Create the 'CreateAndWriteFiles' thread
    hThreads[0] = CreateThread(NULL, 0, CreateAndWriteFiles, NULL, 0, &threadID);
    if (hThreads[0] == NULL) {
        std::cerr << "Failed to create CreateAndWriteFiles thread. Error: " << GetLastError() << std::endl;
        return 1;
    }

    // Create the 'MoveFilesToNewDirectory' thread
    hThreads[1] = CreateThread(NULL, 0, MoveFilesToNewDirectory, NULL, 0, &threadID);
    if (hThreads[1] == NULL) {
        std::cerr << "Failed to create MoveFilesToNewDirectory thread. Error: " << GetLastError() << std::endl;
        return 1;
    }

    // Wait for all threads to finish execution
    WaitForMultipleObjects(2, hThreads, TRUE, INFINITE);

    // Loop through 'hThreads' and close all the thread handles
    for (int i = 0; i < 2; i++) {
        if (!CloseHandle(hThreads[i])) {
            std::cerr << "CloseHandle failed for thread " << i << ". Error: " << GetLastError() << std::endl;
        }
    }

    // Close the event handles
    if (!CloseHandle(hFilesCreatedEvent) || !CloseHandle(hFilesMovedEvent)) {
        std::cerr << "Failed to close event handle(s). Error: " << GetLastError() << std::endl;
    }

    return 0;
}
```

When we run this program, it starts to hang after it has created the files in **C:\Temp**. However, the files are not moved to a differrent folder.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/102e0c69-db13-4c12-a44f-7ef687f469ac)

Open **Process Explorer** and view the wait reason of this program. It means that the wait was requested by the program.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/58745d8b-311b-4147-b9ff-45e584629495)

Create a memory dump while the program is hanging and load it in WinDbg.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/01ab5e69-a3c2-4585-804b-b91c0ab04b94)

# WinDbg Walk Through - Analyzing Memory Dump

Load the **MEX** extension in WinDbg:

```
0:000> .load mex
Mex External 3.0.0.7172 Loaded!
```

Get the current context of the process we're in by running the **!mex.p** command:

```
0:000> !mex.p
Name     Ses PID          PEB              Mods Handle Thrd
======== === ============ ================ ==== ====== ====
test.exe   1 9f4 (0n2548) 00000031b630c000   10     55    3

CommandLine: Test.exe
Last event: 9f4.2cc0: Break instruction exception - code 80000003 (first/second chance not available)

Show Threads: Unique Stacks    !listthreads (!lt)    ~*kv
```

Start listening the threads within this process with the **!mex.lt** command:

```
0:000> !mex.lt
 # DbgID ThdID Wait Function                     User Kernel Info     TEB              Create Time
== ===== ===== ================================= ==== ====== ======== ================ ==========================
->     0  2cc0 Test!main+0x13e                      0      0 Event... 00000031b630d000 08/11/2023 12:51:33.630 PM
       1  13f8 Test!CreateAndWriteFiles+0x4f7    15ms 1s.781          00000031b630f000 08/11/2023 12:51:33.656 PM
       2  2c40 Test!MoveFilesToNewDirectory+0x47    0      0          00000031b6311000 08/11/2023 12:51:33.656 PM
```

Let's look at thread ID **13f8** first. The call stack suggests that the **`CreateAndWriteFiles`** function is waiting for a synchronization object to become signaled using the **WaitForSingleObject** API. While it's waiting, that specific thread is effectively blocked, meaning it is not executing any further code or making progress until the event it's waiting for becomes signaled.

```
0:000> !mex.t -t 0x13f8
DbgID ThreadID      User Kernel Create Time (UTC)
1     13f8 (0n5112) 15ms 1s.781 08/11/2023 12:51:33.656 PM

# Child-SP         Return           Call Site                             Source
0 00000031b64fe748 00007fff13de3f8e ntdll!NtWaitForSingleObject+0x14      
1 00000031b64fe750 00007ff682312cc7 KERNELBASE!WaitForSingleObjectEx+0x8e 
2 00000031b64fe7f0 00007fff153226ad Test!CreateAndWriteFiles+0x4f7        C:\Users\User\source\repos\Test\Test\Test.cpp @ 62
3 00000031b64ffca0 00007fff1690aa68 kernel32!BaseThreadInitThunk+0x1d     
4 00000031b64ffcd0 0000000000000000 ntdll!RtlUserThreadStart+0x28
```

Let's now look at the second thread ID **2C40**. this call stack reveals that the **`MoveFilesToNewDirectory`** function is waiting indefinitely on a synchronization object using the **WaitForSingleObject** API. In this wait state, the thread associated with this call stack is blocked as well, and not processing further.

```
0:001> !mex.t -t 0x2c40
DbgID ThreadID       User Kernel Create Time (UTC)
2     2c40 (0n11328)    0      0 08/11/2023 12:51:33.656 PM

# Child-SP         Return           Call Site                             Source
0 00000031b65ffc98 00007fff13de3f8e ntdll!NtWaitForSingleObject+0x14      
1 00000031b65ffca0 00007ff682312d67 KERNELBASE!WaitForSingleObjectEx+0x8e 
2 00000031b65ffd40 00007fff153226ad Test!MoveFilesToNewDirectory+0x47     C:\Users\User\source\repos\Test\Test\Test.cpp @ 75
3 00000031b65ffef0 00007fff1690aa68 kernel32!BaseThreadInitThunk+0x1d     
4 00000031b65fff20 0000000000000000 ntdll!RtlUserThreadStart+0x28
```  

Synchronization objects, such as events, mutexes, semaphores, etc., are implemented as "handles". A handle is a reference to a system resource, and these handles allow processes to interact with system resources or other system entities. When a synchronization object (e.g., an event) is created using Windows API functions like **CreateEvent**, the system returns a handle to that event. This handle can be used by the application to perform operations on the event, such as setting it, resetting it, or waiting for it.

This explains why we can see all those synchronization objects at the "Handles" tab in **Process Explorer:**

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/9bee9eed-d200-4330-bbb1-d8dc3a83fe63)

We can use the **!handle** command to display information about a handle or handles that one or all processes in the target system own. Here is an example to look for event objects within a memory dump:

``` 
0:000> !handle 0 f event
Handle 0000000000000008
  Type         	Event
  Attributes   	0
  GrantedAccess	0x1f0003:
         Delete,ReadControl,WriteDac,WriteOwner,Synch
         QueryState,ModifyState
  HandleCount  	2
  PointerCount 	65536
  Name         	<none>
  Object specific information
    Event Type Manual Reset
    Event is Set
Handle 000000000000000c
  Type         	Event
  Attributes   	0
  GrantedAccess	0x1f0003:
         Delete,ReadControl,WriteDac,WriteOwner,Synch
         QueryState,ModifyState
  HandleCount  	2
  PointerCount 	65536
  Name         	<none>
  Object specific information
    Event Type Manual Reset
    Event is Set
Handle 0000000000000010
  Type         	Event
  Attributes   	0
  GrantedAccess	0x1f0003:
         Delete,ReadControl,WriteDac,WriteOwner,Synch
         QueryState,ModifyState
  HandleCount  	2
  PointerCount 	65537
  Name         	<none>
  Object specific information
    Event Type Auto Reset
    Event is Waiting
Handle 0000000000000048
  Type         	Event
  Attributes   	0
  GrantedAccess	0x1f0003:
         Delete,ReadControl,WriteDac,WriteOwner,Synch
         QueryState,ModifyState
  HandleCount  	2
  PointerCount 	65525
  Name         	<none>
  Object specific information
    Event Type Auto Reset
    Event is Set
Handle 000000000000004c
  Type         	Event
  Attributes   	0
  GrantedAccess	0x1f0003:
         Delete,ReadControl,WriteDac,WriteOwner,Synch
         QueryState,ModifyState
  HandleCount  	2
  PointerCount 	65533
  Name         	<none>
  Object specific information
    Event Type Auto Reset
    Event is Set
Handle 00000000000000a8
  Type         	Event
  Attributes   	0
  GrantedAccess	0x1f0003:
         Delete,ReadControl,WriteDac,WriteOwner,Synch
         QueryState,ModifyState
  HandleCount  	2
  PointerCount 	65538
  Name         	\BaseNamedObjects\FilesCreatedEvent
  Object specific information
    Event Type Manual Reset
    Event is Waiting
Handle 00000000000000ac
  Type         	Event
  Attributes   	0
  GrantedAccess	0x1f0003:
         Delete,ReadControl,WriteDac,WriteOwner,Synch
         QueryState,ModifyState
  HandleCount  	2
  PointerCount 	65538
  Name         	\BaseNamedObjects\FilesMovedEvent
  Object specific information
    Event Type Manual Reset
    Event is Waiting
7 handles of type Event
```

We can use the **!mex.grep** command from **MEX** to filter on a specific string and only return results that are matching it. In this example, let's use it to search for our specific event objects.

```
0:000> !grep -A 11 "FilesCreatedEvent" !handle 0 f event
  Name         	\BaseNamedObjects\FilesCreatedEvent
  Object specific information
    Event Type Manual Reset
    Event is Waiting
Handle 00000000000000ac
  Type         	Event
  Attributes   	0
  GrantedAccess	0x1f0003:
         Delete,ReadControl,WriteDac,WriteOwner,Synch
         QueryState,ModifyState
  HandleCount  	2
  PointerCount 	65538
```

Based on this output, and especially the "Event is Waiting" means that there is at least one thread currently waiting on this event to become signaled.

Let's look at the second event object:

```
0:000> !grep -A 11 "FilesMovedEvent" !handle 0 f event
  Name         	\BaseNamedObjects\FilesMovedEvent
  Object specific information
    Event Type Manual Reset
    Event is Waiting
7 handles of type Event
```

The output is similar as the previous one, which means that at least one thread is currently waiting for this event to be signaled. 

**Summary:**

The program is in a deadlock state where the **`CreateAndWrite**Files`** thread is waiting for the **`FilesMovedEvent`** to be signaled before it can signal the **`FilesCreatedEvent`**. Meanwhile, the **`MoveFilesToNewDirectory`** thread is waiting for the **`FilesCreatedEvent`** to be signaled before it can proceed. 

# Code Review - How to fix this issue?

Within the **`CreateAndWriteFiles`** function, the thread is waiting for the **`FilesMovedEvent`** to be signaled before it signals the **`FilesCreatedEvent`**.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/cec0e4bd-468b-4ae4-87c4-c3f953d887b5)

Within the **`MoveFilesToNewDirectory`** function, the thread waits for the **`FilesCreatedEvent`** to be signaled before it moves the files. After moving the files, it attempts to signal the **`FilesCreatedEvent`** again, which is not necessary.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/2715eb6e-e94c-498f-861f-1d648ae6554c)


**FIX**

The **`CreateAndWriteFiles`** function should signal **`FilesCreatedEvent`** as soon as it finishes creating and writing to files:

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/e20e68aa-279d-4451-9d1d-b37e6e786f03)

The **`MoveFilesToNewDirectory`** function should wait for **`FilesCreatedEvent`** before starting the moving process. After moving the files, it should then signal **`FilesMovedEvent`**

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/580f033b-9f40-4508-a035-b33e54c1d6a0)

The **`CreateAndWriteFiles`** thread will immediately inform the **`MoveFilesToNewDirectory`** thread when it's done creating files, and then wait for to finish moving files. The **`MoveFilesToNewDirectory`** thread will wait to start moving files until after they've been created, and then signal when it's done.

Here is the complete code with the fixes:

```c
#include <windows.h>
#include <iostream>
#include <string>
#include <random>
#include <tlhelp32.h>
#include <vector>

#define NUM_FILES 1000

HANDLE hFilesCreatedEvent, hFilesMovedEvent;  // Events for signaling file creation and moving completion

std::vector<std::wstring> createdFiles;  // Stores the names of the created files

// This function creates files and writes to them
DWORD WINAPI CreateAndWriteFiles(LPVOID lpParam) {
    std::random_device rd;
    std::mt19937 gen(rd());

    // Create and write to NUM_FILES number of files
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
                static_cast<DWORD>(data.size()),
                &bytesWritten,
                NULL
            )) {
                std::wcerr << L"WriteFile failed for " << fileName << L". Error: " << GetLastError() << std::endl;
                break;
            }
        }

        CloseHandle(hFile);
        createdFiles.push_back(fileName);
        std::wcout << L"Created and wrote to file: " << fileName << std::endl;
    }

    // FIX: Signal that all files are created before waiting for them to be moved
    SetEvent(hFilesCreatedEvent);

    // Wait for the files to be moved
    WaitForSingleObject(hFilesMovedEvent, INFINITE);

    return 0;
}

// This function moves the created files to a new directory
DWORD WINAPI MoveFilesToNewDirectory(LPVOID lpParam) {
    // Wait for the files to be created
    WaitForSingleObject(hFilesCreatedEvent, INFINITE);

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

    // FIX: Signal that all files are moved
    SetEvent(hFilesMovedEvent);

    return 0;
}

int main() {
    DWORD threadID;  // Variable to store the thread identifier
    HANDLE hThreads[2];  // Array to store the handles of the threads

    // Create two separate events
    hFilesCreatedEvent = CreateEvent(NULL, TRUE, FALSE, L"Global\\FilesCreatedEvent");
    hFilesMovedEvent = CreateEvent(NULL, TRUE, FALSE, L"Global\\FilesMovedEvent");
    if (hFilesCreatedEvent == NULL || hFilesMovedEvent == NULL) {
        std::cerr << "Failed to create event(s). Error: " << GetLastError() << std::endl;
        return 1;
    }

    // Create the 'CreateAndWriteFiles' thread
    hThreads[0] = CreateThread(NULL, 0, CreateAndWriteFiles, NULL, 0, &threadID);
    if (hThreads[0] == NULL) {
        std::cerr << "Failed to create CreateAndWriteFiles thread. Error: " << GetLastError() << std::endl;
        return 1;
    }

    // Create the 'MoveFilesToNewDirectory' thread
    hThreads[1] = CreateThread(NULL, 0, MoveFilesToNewDirectory, NULL, 0, &threadID);
    if (hThreads[1] == NULL) {
        std::cerr << "Failed to create MoveFilesToNewDirectory thread. Error: " << GetLastError() << std::endl;
        return 1;
    }

    // Wait for all threads to finish execution
    WaitForMultipleObjects(2, hThreads, TRUE, INFINITE);

    // Loop through 'hThreads' and close all the thread handles
    for (int i = 0; i < 2; i++) {
        if (!CloseHandle(hThreads[i])) {
            std::cerr << "CloseHandle failed for thread " << i << ". Error: " << GetLastError() << std::endl;
        }
    }

    // Close the event handles
    if (!CloseHandle(hFilesCreatedEvent) || !CloseHandle(hFilesMovedEvent)) {
        std::cerr << "Failed to close event handle(s). Error: " << GetLastError() << std::endl;
    }

    return 0;
}
```

# WinDbg Walk Through - Theory vs Practice

Start with compiling the code above and start running it. After the program has created all the **1000** files and is in the progress of moving the file to the **C:\Temp2** folder. Open **Process Explorer** as an administrator to **suspend** the process.

Here we're in the progress of moving the files to the next folder after all thw **1000** files have been created:

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/af61ffb1-7328-4f25-a35d-f2dda87ab236)

Start now suspending the process:

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/5f940a95-722b-4b4b-a88c-5100a6f4db75)

Create a memory dump of the suspended process and load it in WinDbg.

Load the **MEX** extension:

```
0:000> .load mex
Mex External 3.0.0.7172 Loaded!
```

Since we have completed all the file creation, let's use the **!mex.grep** command to filter on this event object **`(FilesCreatedEvent)`** and confirm that the "Event is Set". This tells us that the event is currently in the signaled state, which means that any threads waiting on this event would now proceed.

```
0:000> !grep -A 11 "FilesCreatedEvent" !handle 0 f event
  Name         	\BaseNamedObjects\FilesCreatedEvent
  Object specific information
    Event Type Manual Reset
    Event is Set
Handle 00000000000000ac
  Type         	Event
  Attributes   	0
  GrantedAccess	0x1f0003:
         Delete,ReadControl,WriteDac,WriteOwner,Synch
         QueryState,ModifyState
  HandleCount  	2
  PointerCount 	65538
```

We also can use **Process Explorer** to find this out:

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/43d86c50-a915-4eeb-80a4-f1037720989f)
