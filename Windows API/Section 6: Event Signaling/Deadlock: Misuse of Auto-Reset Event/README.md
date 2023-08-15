# What is the root cause of this deadlock within this code?

In order to demonstrate this demo. Please follow the following steps:

- Create the following 2 folders:
  1. **C:\Temp**
  2. **C:\Temp2**

Once we've done this step, we can continue.


The purpose of this code is the following:

- Generate and write content to **100** files located in the **C:\Temp** directory.
- Move the created files from **C:\Temp** to a new directory, **C:\Temp2**.
- Delete the files that were moved to the **C:\Temp2** directory.

The code uses a single event, **`hFilesCreatedEvent`**, to manage the synchronization between three functions: **`CreateAndWriteFiles`**, **`MoveFilesToNewDirectory`**, and **`DeleteMovedFiles`**. After **`CreateAndWriteFiles`** completes its operation, it signals the event, allowing **`MoveFilesToNewDirectory`** to proceed. However, once **`MoveFilesToNewDirectory`** finishes, it doesn't re-signal the event, causing **`DeleteMovedFiles`** to wait indefinitely and creating a deadlock in the process.

```c
#include <windows.h>
#include <iostream>
#include <string>
#include <random>
#include <vector>

#define NUM_FILES 100

HANDLE hFilesCreatedEvent;  // Event for signaling file creation completion
std::vector<std::wstring> createdFiles;  // Stores the names of the created files

// Function to create files and write data to them.
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

    // Signal that all files are created.
    SetEvent(hFilesCreatedEvent);

    return 0;
}

// Function to move all created files to a new directory.
DWORD WINAPI MoveFilesToNewDirectory(LPVOID lpParam) {
    // Potential Deadlock Situation:
    // This thread waits for the 'hFilesCreatedEvent' to be signaled, which means
    // it waits for all files to be created before it starts moving them.
    WaitForSingleObject(hFilesCreatedEvent, INFINITE);

    for (const auto& source : createdFiles) {
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

// Function to delete the files that have been moved.
DWORD WINAPI DeleteMovedFiles(LPVOID lpParam) {
    // Potential Deadlock Situation:
    // This thread, too, waits for the 'hFilesCreatedEvent' to be signaled.
    // However, since the event is auto-reset, it may never get signaled again
    // if 'MoveFilesToNewDirectory' has already reset it, causing this thread to wait indefinitely.
    WaitForSingleObject(hFilesCreatedEvent, INFINITE);

    for (const auto& source : createdFiles) {
        if (!DeleteFile(source.c_str())) {
            std::wcerr << L"DeleteFile failed for " << source << L". Error: " << GetLastError() << std::endl;
        }
        else {
            std::wcout << L"Deleted file: " << source << std::endl;
        }
    }

    return 0;
}

int main() {
    DWORD threadID;  // Variable to store the thread identifier
    HANDLE hThreads[3];  // Array to store the handles of the threads

    // Create an auto-reset event with the name "Global\\FileProcessingEvent".
    hFilesCreatedEvent = CreateEvent(NULL, FALSE, FALSE, L"Global\\FileProcessingEvent");
    if (hFilesCreatedEvent == NULL) {
        std::cerr << "Failed to create event. Error: " << GetLastError() << std::endl;
        return 1;
    }

    // Create the 'CreateAndWriteFiles' thread.
    hThreads[0] = CreateThread(NULL, 0, CreateAndWriteFiles, NULL, 0, &threadID);
    if (hThreads[0] == NULL) {
        std::cerr << "Failed to create CreateAndWriteFiles thread. Error: " << GetLastError() << std::endl;
        return 1;
    }

    // Create the 'MoveFilesToNewDirectory' thread.
    hThreads[1] = CreateThread(NULL, 0, MoveFilesToNewDirectory, NULL, 0, &threadID);
    if (hThreads[1] == NULL) {
        std::cerr << "Failed to create MoveFilesToNewDirectory thread. Error: " << GetLastError() << std::endl;
        return 1;
    }

    // Create the 'DeleteMovedFiles' thread.
    hThreads[2] = CreateThread(NULL, 0, DeleteMovedFiles, NULL, 0, &threadID);
    if (hThreads[2] == NULL) {
        std::cerr << "Failed to create DeleteMovedFiles thread. Error: " << GetLastError() << std::endl;
        return 1;
    }

    // Wait for all threads to finish execution.
    WaitForMultipleObjects(3, hThreads, TRUE, INFINITE);

    // Close all thread handles.
    for (int i = 0; i < 3; i++) {
        if (!CloseHandle(hThreads[i])) {
            std::cerr << "CloseHandle failed for thread " << i << ". Error: " << GetLastError() << std::endl;
        }
    }

    // Close the event handle.
    if (!CloseHandle(hFilesCreatedEvent)) {
        std::cerr << "Failed to close event handle. Error: " << GetLastError() << std::endl;
    }

    return 0;
}
```

When we run this program, it will start to hang:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/12a792ad-cc90-469a-b8b8-279112cac22f)




# What is the flow that causes this deadlock?

1. The process begins with the **`CreateAndWriteFiles`** function, which writes files to the disk. Once completed, it signals the event.
2. After receiving the signal, the **`MoveFilesToNewDirectory`** function is activated. It moves the files to the new directory. After this action, the event is auto-reset to the **non-signaled** state.
3. **`DeleteMovedFiles`** function waits for the event to be signaled again. However, since no other function signals it, this function waits indefinitely, leading to a deadlock.

**`hFilesCreatedEvent`** is an auto-reset event. When an auto-reset event is signaled (set to the signaled state) and a waiting thread is released, the system automatically resets the event to the non-signaled state.

Here is an ASCII flow that explains this deadlock:

```
+-----------------------------------------------+
|                                               |
|   [CreateAndWriteFiles Thread Starts]         |
|                |                              |
|                v                              |
|   [Files are Written to Disk]                 |
|                |                              |
|                v                              |
|   [Signal Event: SetEvent called]             |
|                |                              |
|                +------------------------------+--------> [MoveFilesToNewDirectory Thread Released]
|                |                              |             |
|                |                              |             v
|                |                              |      [Files are Moved to New Directory]
|                |                              |             |
|                |                              |             v
|                |                              |      [End of MoveFilesToNewDirectory's Work]
|                |                              |             |
|                |                              |             v
|                |                              |      [Event is Auto-Reset to Non-signaled State]
|                |                              |             |
+----------------|------------------------------|-------------+-----------------+
                 |                              |                             |
                 |                              |                             |
                 v                              v                             v
    [CreateAndWriteFiles Thread Ends]  [MoveFilesToNewDirectory Thread Ends] [DeleteMovedFiles Thread]
                                                                                   |
                                                                                   v
                                                                             [Waits Indefinitely for Event]
                                                                                   |
                                                                                   v
                                                                             [Deadlock: Never Proceeds]
```

# WinDbg Walk Through - Analyzing Memory Dump

Load the **MEX** extension:

```
0:000> .load mex
Mex External 3.0.0.7172 Loaded!
```

List the threads of the current process:

```
0:000> !mex.lt
 # DbgID ThdID Wait Function              User Kernel Info     TEB              Create Time
== ===== ===== ========================== ==== ====== ======== ================ ==========================
->     0  25dc Test!main+0x17a               0      0 Event... 000000859f17f000 08/11/2023 08:04:20.141 PM
       1  3324 Test!DeleteMovedFiles+0x1c    0      0          000000859f185000 08/11/2023 08:04:20.146 PM
```

View the call stack of the thread and we can see that it is currently waiting on a event object:

```
0:000> !mex.t -t 0x3324
DbgID ThreadID       User Kernel Create Time (UTC)
1     3324 (0n13092)    0      0 08/11/2023 08:04:20.146 PM

# Child-SP         Return           Call Site                             Source
0 000000859f5ffb38 00007ff8beaf3f8e ntdll!NtWaitForSingleObject+0x14      
1 000000859f5ffb40 00007ff6dd0930cc KERNELBASE!WaitForSingleObjectEx+0x8e 
2 000000859f5ffbe0 00007ff8bf9626ad Test!DeleteMovedFiles+0x1c            C:\Users\User\source\repos\Test\Test\Test.cpp @ 88
3 000000859f5ffc10 00007ff8c13caa68 kernel32!BaseThreadInitThunk+0x1d     
4 000000859f5ffc40 0000000000000000 ntdll!RtlUserThreadStart+0x28   
```

Let's use the **!handle** command to display the event object:

```
0:001> !grep -A 5 "FileProcessingEvent" !handle 0 f event
  Name         	\BaseNamedObjects\FileProcessingEvent
  Object specific information
    Event Type Auto Reset
    Event is Waiting
6 handles of type Event
```

The output describes an auto-reset event named **`FileProcessingEvent`** that is currently in a non-signaled (waiting) state. We can also see this in **Process Explorer**.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/95f98eca-6421-42d9-be73-70782b5e0d7c)


