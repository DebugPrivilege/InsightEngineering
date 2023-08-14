# What is a Critical Section?

A "critical section" in programming is like a special zone where only one task can go at a time. It's used to **protect** parts of a program that deal with shared stuff (like data or files), so that multiple tasks don't mess things up by trying to use or change them all at once.

Suppose you have a program where multiple threads are trying to write data to the same file. This is a critical section because if **two threads** try to write to the file **at the same time**, the data could get mixed up or overwritten, leading to a corrupted file.

Windows API provides several functions to implement Critical Sections in our code for ensuring synchronization between threads. Without synchronization, we can run into several issues in a multi-threaded program, such as a race condition or deadlock.

Here are a few examples:

| Function | Description |
| --- | --- |
| `InitializeCriticalSection()` | Initializes a critical section object, which can then be used by the threads of a single process. |
| `EnterCriticalSection()` | A thread uses this function to request ownership of a critical section. If the critical section is already owned by another thread, the function will block the calling thread until it can take ownership. |
| `LeaveCriticalSection()` | A thread uses this function to release its ownership of a critical section. If other threads are waiting to take ownership of the critical section, one of them will be granted ownership. |
| `DeleteCriticalSection()` | Releases all resources used by an uninitialized critical section object. After this function is called, the critical section object can no longer be used for synchronization. |

When a thread enters a critical section, it is said to "take ownership" of the critical section. This means that it is the only thread that is allowed to execute the code within that critical section. No other thread can enter this section of code until the owning thread leaves the critical section, thereby releasing its ownership.

If another thread tries to enter the critical section while it's owned by a different thread, it will be blocked (i.e., it will have to wait) until the owning thread leaves the critical section. Once the owning thread has left the critical section, another thread can then enter and take ownership.

# Code Sample (1) - Race Condition

A race condition is a situation in programming where two or more threads access and manipulate shared data simultaneously, leading to unpredictable and incorrect results. 

This example code creates three separate threads: one to generate and write to 1000 files in a directory, another to move these files to a new directory, and a third one to enumerate all the running processes on the system 10 times. 

However, a race condition occurs as the file-moving thread may attempt to move files that are still being written to or not yet created, leading to unpredictable behavior and potential errors.

```c
#include <windows.h>
#include <iostream>
#include <string>
#include <random>
#include <TlHelp32.h>

#define NUM_FILES 1000
#define NUM_REPEATS_PROCESSES 10

// This function creates files and writes to them
DWORD WINAPI CreateAndWriteFiles(LPVOID lpParam) {
    // Create a random number generator
    std::random_device rd;
    std::mt19937 gen(rd());

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

        // Add console output after file creation
        std::wcout << L"Created and wrote to file: " << fileName << std::endl;
    }

    return 0;
}

// This function moves the created files to a new directory
DWORD WINAPI MoveFilesToNewDirectory(LPVOID lpParam) {
    // Move NUM_FILES number of files
    for (int i = 0; i < NUM_FILES; i++) {
        std::wstring source = L"C:\\Temp\\file_" + std::to_wstring(i) + L".txt";
        std::wstring destination = L"C:\\Temp2\\file_" + std::to_wstring(i) + L".txt";

        // RACE CONDITION occurs here. This thread may attempt to move a file
        // that is currently being written to by the CreateAndWriteFiles thread,
        // or that hasn't been created yet.
        if (!MoveFile(source.c_str(), destination.c_str())) {
            std::wcerr << L"MoveFile failed for " << source << L" to " << destination << L". Error: " << GetLastError() << std::endl;
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

    DWORD threadID;
    HANDLE hThreads[3];

    // Create the file creation and writing thread
    hThreads[0] = CreateThread(NULL, 0, CreateAndWriteFiles, NULL, 0, &threadID);
    if (hThreads[0] == NULL) {
        std::cerr << "Failed to create CreateAndWriteFiles thread. Error: " << GetLastError() << std::endl;
        return 1;
    }

    // Create the file moving thread
    hThreads[1] = CreateThread(NULL, 0, MoveFilesToNewDirectory, NULL, 0, &threadID);
    if (hThreads[1] == NULL) {
        std::cerr << "Failed to create MoveFilesToNewDirectory thread. Error: " << GetLastError() << std::endl;
        return 1;
    }

    // Create the process enumeration thread
    hThreads[2] = CreateThread(NULL, 0, EnumerateProcesses, NULL, 0, &threadID);
    if (hThreads[2] == NULL) {
        std::cerr << "Failed to create EnumerateProcesses thread. Error: " << GetLastError() << std::endl;
        return 1;
    }

    // Wait for all three threads to finish
    if (WaitForMultipleObjects(3, hThreads, TRUE, INFINITE) == WAIT_FAILED) {
        std::cerr << "WaitForMultipleObjects failed. Error: " << GetLastError() << std::endl;
    }

    // Close the thread handles
    for (int i = 0; i < 3; i++) {
        if (!CloseHandle(hThreads[i])) {
            std::cerr << "CloseHandle failed for thread " << i << ". Error: " << GetLastError() << std::endl;
        }
    }

    return 0;
}
```

The race condition in this code exists because there is no synchronization between the **`CreateAndWriteFiles`** and **`MoveFilesToNewDirectory`** threads. The threads need to be coordinated so that the **`MoveFilesToNewDirectory`** thread doesn't start moving files until the **`CreateAndWriteFiles`** thread has finished creating and writing to all the files. 

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/5f85e737-ff95-40ff-b880-87ef87eb8ffa)

# Code Sample 2 - Critical Section

This code doesn't have a race condition anymore because it uses synchronization to ensure that file creation and moving are not happening simultaneously by different threads. This is achieved by using a **critical section**, which is a method for preventing multiple threads from accessing shared resources at the same time.

**`CreateAndWriteFiles`** function enters the critical section before it starts creating and writing files and only leaves it once it has finished. The **`MoveFilesToNewDirectory`** function also enters this critical section before moving the files and leaves it once it's done.

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

    EnterCriticalSection(&critSection);  // Enter the critical section

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
            EnterCriticalSection(&consoleCritSection);
            std::wcerr << L"CreateFile failed for " << fileName << L". Error: " << GetLastError() << std::endl;
            LeaveCriticalSection(&consoleCritSection);
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
                EnterCriticalSection(&consoleCritSection);
                std::wcerr << L"WriteFile failed for " << fileName << L". Error: " << GetLastError() << std::endl;
                LeaveCriticalSection(&consoleCritSection);
                break;
            }
        }

        // Close the file handle
        if (!CloseHandle(hFile)) {
            EnterCriticalSection(&consoleCritSection);
            std::wcerr << L"CloseHandle failed for " << fileName << L". Error: " << GetLastError() << std::endl;
            LeaveCriticalSection(&consoleCritSection);
        }

        // Add the file name to the createdFiles vector
        createdFiles.push_back(fileName);

        EnterCriticalSection(&consoleCritSection);
        std::wcout << L"Created and wrote to file: " << fileName << std::endl;
        LeaveCriticalSection(&consoleCritSection);
    }

    LeaveCriticalSection(&critSection);  // Leave the critical section

    return 0;
}

// This function moves the created files to a new directory
DWORD WINAPI MoveFilesToNewDirectory(LPVOID lpParam) {
    EnterCriticalSection(&critSection);  // Enter the critical section

    // For each file that was created, move it to the new directory
    for (const auto& source : createdFiles) {
        std::wstring destination = L"C:\\Temp2\\" + source.substr(source.find_last_of(L"\\") + 1);
        if (!MoveFile(source.c_str(), destination.c_str())) {
            EnterCriticalSection(&consoleCritSection);
            std::wcerr << L"MoveFile failed for " << source << L" to " << destination << L". Error: " << GetLastError() << std::endl;
            LeaveCriticalSection(&consoleCritSection);
        }
        else {
            EnterCriticalSection(&consoleCritSection);
            std::wcout << L"Moved file from " << source << L" to " << destination << std::endl;
            LeaveCriticalSection(&consoleCritSection);
        }
    }

    LeaveCriticalSection(&critSection);  // Leave the critical section

    return 0;
}

// This function enumerates all the running processes on the machine 10 times
DWORD WINAPI EnumerateProcesses(LPVOID lpParam) {
    for (int j = 0; j < 10; j++) {
        HANDLE hSnapshot = CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, 0);
        if (hSnapshot == INVALID_HANDLE_VALUE) {
            EnterCriticalSection(&consoleCritSection);
            std::wcerr << L"CreateToolhelp32Snapshot failed" << std::endl;
            LeaveCriticalSection(&consoleCritSection);
            return 1;
        }

        PROCESSENTRY32W pe32;
        pe32.dwSize = sizeof(pe32);

        if (!Process32FirstW(hSnapshot, &pe32)) {
            EnterCriticalSection(&consoleCritSection);
            std::wcerr << L"Process32First failed" << std::endl;
            LeaveCriticalSection(&consoleCritSection);
            CloseHandle(hSnapshot);
            return 1;
        }

        do {
            EnterCriticalSection(&consoleCritSection);
            std::wcout << L"Process ID: " << pe32.th32ProcessID << ", Process Name: " << pe32.szExeFile << std::endl;
            LeaveCriticalSection(&consoleCritSection);
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

    WaitForMultipleObjects(3, hThreads, TRUE, INFINITE);

    for (int i = 0; i < 3; i++) {
        if (!CloseHandle(hThreads[i])) {
            std::cerr << "CloseHandle failed for thread " << i << ". Error: " << GetLastError() << std::endl;
        }
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

**`CreateAndWriteFiles`** and **`MoveFilesToNewDirectory`** functions use the same critical section, defined as **`critSection`** in the code. When a thread enters a critical section, no other thread can enter the same critical section until the first thread leaves it.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/b9ef1590-d8c6-4f80-a0ef-ab158dbafffa)

# ASCII Visualization of Critical Section

Let's have an ASCII visualization of how the Critical Section works within our code:

```
Thread 1 (CreateAndWriteFiles)
|---------------------|                        |---------------------|
   Creating Files  ------------------------>  Waiting for Crit. Section
|---------------------|                        |---------------------|
                     Critical Section
                     |---------------------|
                         Writing to Files
                     |---------------------|

Thread 2 (MoveFilesToNewDirectory)
                     |---------------------|                       |---------------------|
                     Waiting for Crit. Section  -------------->          Moving Files
                     |---------------------|                       |---------------------|
                        Critical Section
                     |---------------------|
                        Moving Files
                     |---------------------|

Thread 3 (EnumerateProcesses)
                                           |---------------------|
                                           Enumerating Processes
                                           |---------------------|
```

- **Thread 1** starts by creating files. Once it's done, it waits for the critical section.
- **Thread 2** starts by waiting for the critical section. Once **Thread 1** is done and releases the critical section, **Thread 2** moves the files.
- **Thread 3** independently enumerates processes without interacting with the critical section.
