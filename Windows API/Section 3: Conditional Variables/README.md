# What are Conditional Variables?

Condition variables are synchronization primitives that enable threads to wait until a particular condition occurs. Condition variables are user-mode objects that cannot be shared across processes. Condition variables enable threads to atomically release a lock and enter the sleeping state. They can be used with critical sections or slim reader/writer (SRW) locks.

There are Windows APIs that provides a set of functions for managing conditional variables in multithreaded applications. These include initializing condition variables, making threads wait while releasing locks, and waking up one or all threads waiting on a condition variable.

| Function | Description |
|-----------------------------|-------------|
| `InitializeConditionVariable()` | Sets up a new condition variable. |
| `SleepConditionVariableCS()` | Makes a thread wait and lets go of a critical section lock at the same time. |
| `SleepConditionVariableSRW()` | Makes a thread wait and lets go of a Slim Reader/Writer (SRW) lock at the same time. |
| `WakeAllConditionVariable()` | Wakes up all threads that are waiting on a condition variable. |
| `WakeConditionVariable()` | Wakes up one thread that is waiting on a condition variable. |

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

# Code Sample (2) - Conditional Variables

This code uses a global condition variable and a critical section to synchronize the creation and moving of files. It also uses a queue to hold the filenames of the files as they are created. 

```c
#include <windows.h>
#include <iostream>
#include <string>
#include <random>
#include <TlHelp32.h>
#include <queue>

#define NUM_FILES 1000
#define NUM_REPEATS_PROCESSES 10

// Create a global condition variable, critical section, and a queue for filenames
CONDITION_VARIABLE ConditionVar;
CRITICAL_SECTION CritSection;
// This is a queue to hold the filenames of the files created by the CreateAndWriteFiles function.
// The filenames are added to the queue as they are created, and they are removed from the queue as they are moved.
std::queue<std::wstring> fileQueue;


// This function creates files and writes to them
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

        // Add the filename to the queue and signal the condition variable
        EnterCriticalSection(&CritSection);
        fileQueue.push(fileName);
        WakeConditionVariable(&ConditionVar);
        LeaveCriticalSection(&CritSection);
    }

    return 0;
}

// This function moves the created files to a new directory
DWORD WINAPI MoveFilesToNewDirectory(LPVOID lpParam) {
    for (int i = 0; i < NUM_FILES; i++) {
        // Wait for the condition variable to be signaled
        EnterCriticalSection(&CritSection);
        while (fileQueue.empty()) {
            SleepConditionVariableCS(&ConditionVar, &CritSection, INFINITE);
        }

        // Get the filename from the queue
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


// This function enumerates processes
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
    // Initialize the global condition variable and critical section
    InitializeConditionVariable(&ConditionVar);
    InitializeCriticalSection(&CritSection);

    // Get the high-resolution performance counter frequency
    LARGE_INTEGER frequency;
    QueryPerformanceFrequency(&frequency);

    // Get the current counter value at the start of the program
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

    // Get the current counter value at the end of the program
    LARGE_INTEGER end;
    QueryPerformanceCounter(&end);

    // Calculate the total execution time in seconds
    double time = static_cast<double>(end.QuadPart - start.QuadPart) / frequency.QuadPart;

    std::wcout << L"Total execution time: " << time << L" seconds" << std::endl;

    return 0;
}
```

The **`CONDITION_VARIABLE`** is a synchronization primitive provided by the Windows API that enables threads to wait until a particular condition is met. It is used here to synchronize the activities of the **`CreateAndWriteFiles`** and **`MoveFilesToNewDirectory`** threads.

The **`CreateAndWriteFiles`** thread is responsible for creating files and writing content to them. After creating a file, it adds the file's name to a shared queue and signals the **`ConditionVar`** using the **WakeConditionVariable** function. This signal serves as a notification that a new file has been created and is ready to be processed.

On the other hand, the **`MoveFilesToNewDirectory`** thread is initially in a waiting state, blocked on the **`ConditionVar`** using the **SleepConditionVariableCS** function. The thread is waiting for the condition (the creation of a new file) to be met. If the shared queue is empty, indicating that there are no new files to process, the thread remains blocked.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/1209cd83-cdcb-4066-99fc-b0c6d6393688)

# What is the difference between Critical Sections and Conditional Variables?

A **Critical Section** only allows one thread to execute code within the critical section at a time, while a **Condition Variable** is used for synchronization between threads based on certain conditions (i.e., one or more threads can wait for a signal to continue execution). Condition variables are useful when you have a condition that depends on some data that is shared between threads.

# Conditional Variables - Visualization

This ASCII visualization represents the flow of data and control between different threads.

```
|-------------------------------------------------------------------------------|
|                        CreateAndWriteFiles Thread                             |
|-------------------------------------------------------------------------------|
|  Creates a file in "C:\\Temp", puts its name into the queue, then signals     |
|  the condition variable                                                       |
|-------------------------------------------------------------------------------|
                                  |
                                  V
|-------------------------------------------------------------------------------|
|                              fileQueue (Queue)                                |
|-------------------------------------------------------------------------------|
|  Holds the names of the files created by the CreateAndWriteFiles thread       |
|-------------------------------------------------------------------------------|
                                  |
                                  V
|-------------------------------------------------------------------------------|
|                     MoveFilesToNewDirectory Thread                            |
|-------------------------------------------------------------------------------|
|  Waits until there's a file in the queue, then moves it from "C:\\Temp"       |
|  to "C:\\Temp2"                                                               |
|-------------------------------------------------------------------------------|

|-------------------------------------------------------------------------------|
|                           EnumerateProcesses Thread                           |
|-------------------------------------------------------------------------------|
|  Enumerates running processes, runs independently                             |
|-------------------------------------------------------------------------------|
```
