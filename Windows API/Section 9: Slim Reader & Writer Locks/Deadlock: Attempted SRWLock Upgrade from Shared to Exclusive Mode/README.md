# What is the root cause of this deadlock within this code sample?

The deadlock is caused by a thread holding the SRWLock in shared mode and then attempting to acquire it again in exclusive mode without first releasing the shared lock. The SRWLock does not support upgrading from a shared to an exclusive lock, which results in the thread becoming deadlocked.

Compile the following code and run it:

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

// This function writes data to a file
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
            // Release the lock in case of failure before returning
            ReleaseSRWLockExclusive(&srwLock);
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
            // Release the lock in case of failure before returning
            ReleaseSRWLockExclusive(&srwLock);
            return 1;
        }

        // Output a message to the console
        std::cout << "Writer Thread " << ss.str() << " wrote to the file: " << data;

        // Close the file
        CloseHandle(hFile);

        // FIX: Release the SRW lock after file operations to prevent deadlock
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
    std::stringstream ss;
    ss << GetCurrentThreadId();

    while (true) {
        AcquireSRWLockShared(&srwLock);
        bool allThreadsCompleted = (completedThreads == NUM_WRITER_THREADS);
        // Intentionally not releasing the shared lock here to introduce the deadlock scenario
        if (allThreadsCompleted) {
            break;
        }
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
    }

    // Issue: Trying to acquire the SRWLock in exclusive mode while still holding it in shared mode.
    // This will introduce a deadlock because SRWLock does not support upgrading from shared to exclusive mode.
    AcquireSRWLockExclusive(&srwLock);

    if (!MoveFileEx(L"C:\\Temp\\log.txt", L"C:\\Temp2\\log.txt", MOVEFILE_REPLACE_EXISTING)) {
        std::cerr << "MoveFileEx failed. Error: " << GetLastError() << std::endl;
        ReleaseSRWLockExclusive(&srwLock); // We won't reach here due to the deadlock
        return 1;
    }

    std::cout << "Mover Thread " << ss.str() << " moved the file.\n";
    ReleaseSRWLockExclusive(&srwLock); // We won't reach here due to the deadlock

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

The issue is in **`MoveFileToFolder`** function. The thread first acquires the SRWLock in shared mode using **`AcquireSRWLockShared(&srwLock)`**. However, before releasing this shared lock, the same thread tries to acquire the lock in exclusive mode using **`AcquireSRWLockExclusive(&srwLock)`**. 

```c
// Acquire the SRW lock in shared mode
AcquireSRWLockShared(&srwLock);

// ... some code ...

// Acquire the SRW lock in exclusive mode while still holding it in shared mode
AcquireSRWLockExclusive(&srwLock);
```


This is a problem because Slim Reader/Writer (SRW) Locks in Windows do not support upgrading a lock from shared to exclusive mode. Essentially, the thread is waiting to acquire a lock in exclusive mode that it already holds in shared mode, leading to a deadlock situation. 

Once we run the program, it will hang somewhere at the start:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/58cabf85-5ff0-44e5-b5d9-000217153cfd)


When we open **Process Explorer** and look at the wait reasons. We can see that the threads are waiting on a Slim Reader & Writer lock:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/b39ca6d5-6a29-41fa-96e8-b90385f7941f)


The fix was to release the shared lock **`(ReleaseSRWLockShared(&srwLock);)`** before attempting to acquire the exclusive lock, ensuring that the thread doesn't hold both locks at the same time, which would lead to a deadlock. The SRW lock does not support upgrading from a shared lock to an exclusive lock, which could lead to a deadlock if not managed correctly.

```c
// This function writes data to a file
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
            // Release the lock in case of failure before returning
            ReleaseSRWLockExclusive(&srwLock);
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
            // Release the lock in case of failure before returning
            ReleaseSRWLockExclusive(&srwLock);
            return 1;
        }

        // Output a message to the console
        std::cout << "Writer Thread " << ss.str() << " wrote to the file: " << data;

        // Close the file
        CloseHandle(hFile);

        // FIX: Release the SRW lock after file operations to prevent deadlock
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
```
