# What is the root cause of this deadlock within this sample code?

The root cause of the deadlock in the code is the recursive attempt to acquire the SRW lock in exclusive mode using the **AcquireSRWLockExclusive** API function within the **`WriteToFile`** function. After the thread has successfully acquired the lock and performed its file writing operations, the same thread tries to invoke **AcquireSRWLockExclusive** again before releasing the lock with **ReleaseSRWLockExclusive**. 

**SRW locks do not support recursive acquisition in exclusive mode**. As a consequence, the thread becomes indefinitely blocked, waiting on itself to release a lock it already holds, leading to a deadlock.

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

        // Introduce deadlock: try to acquire the lock recursively in exclusive mode
        // SRWLocks do not support recursive acquisition in exclusive mode
        AcquireSRWLockExclusive(&srwLock);  // This will deadlock the thread

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

The **`WriteToFile`** function attempts to acquire the SRW lock in exclusive mode twice, without releasing it after the first acquisition. Since the SRW lock doesn't support **recursive acquisition** in exclusive mode, the thread gets stuck, waiting indefinitely for a lock it already holds. When we say **recursive acquisition**, we mean, it is trying to acquire a lock that is already held by the current thread. Some locks support recursive acquisition, but the SRW lock in exclusive mode doesn't.

```c
// This function writes to a file
DWORD WINAPI WriteToFile(LPVOID lpParam) {
    ...
    for (int i = 0; i < NUM_OPERATIONS; i++) {
        // Acquire the SRW lock in exclusive mode
        AcquireSRWLockExclusive(&srwLock);
        ...
        
        // Intentionally trying to recursively acquire the lock to demonstrate deadlock
        AcquireSRWLockExclusive(&srwLock);
        ...
        
        ReleaseSRWLockExclusive(&srwLock);  // Release the lock
        ...
    }
    ...
}
```

In the above code:

1. The first call to **`AcquireSRWLockExclusive(&srwLock);`** acquires the SRW lock in exclusive mode.
2. The function then tries to acquire the SRW lock again with another call to **`AcquireSRWLockExclusive(&srwLock);`** before it has been released. This is the point of contention, as the SRW lock does not support recursive acquisition in exclusive mode.

By ensuring that the SRW lock is acquired and released only once within the loop, we prevent the deadlock situation:

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
