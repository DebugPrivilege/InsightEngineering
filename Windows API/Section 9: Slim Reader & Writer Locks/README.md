![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/cba3b5a1-e626-4e3e-9259-b1eef9c28154)# What are Slim Reader/Writer Locks?

Slim Reader/Writer (SRW) locks are a type of synchronization mechanism. These locks are designed to allow multiple threads to read data simultaneously, while ensuring that only one thread can write data at a time. Slim Reader/Writer (SRW) locks cannot be shared across processes. 

The following Windows APIs can be used to interact with SRW locks:

| Function | Description |
| --- | --- |
| `InitializeSRWLock` | Initializes a new SRW lock. |
| `AcquireSRWLockExclusive` | Acquires an SRW lock in exclusive mode. If the lock is already held, the calling thread will be blocked until the lock is released. |
| `AcquireSRWLockShared` | Acquires an SRW lock in shared mode. If an exclusive lock is already held, the calling thread will be blocked until the lock is released. |
| `ReleaseSRWLockExclusive` | Releases an SRW lock that was held in exclusive mode. |
| `ReleaseSRWLockShared` | Releases an SRW lock that was held in shared mode. |
| `TryAcquireSRWLockExclusive` | Attempts to acquire an SRW lock in exclusive mode. If the lock is already held, this function will return immediately instead of blocking the calling thread. |
| `TryAcquireSRWLockShared` | Attempts to acquire an SRW lock in shared mode. If the lock is already held in exclusive mode, this function will return immediately instead of blocking the calling thread. |

# Shared Mode vs Exclusive Mode

- **Shared Mode:**
Allows **multiple threads** to read the shared resource simultaneously. Used when the shared resource is being read but not modified.

```
SRWLOCK srwLock;
InitializeSRWLock(&srwLock);

AcquireSRWLockShared(&srwLock);
// Multiple threads can read the shared data simultaneously here
ReleaseSRWLockShared(&srwLock);
```

- **Exclusive Mode:**
Allows **only one thread** to modify the shared resource. Used when the shared resource is being written or modified.

```
SRWLOCK srwLock;
InitializeSRWLock(&srwLock);

AcquireSRWLockExclusive(&srwLock);
// Only one thread can modify the shared data here
ReleaseSRWLockExclusive(&srwLock);
```

In both cases, the **AcquireSRWLock**... functions are used to acquire the lock before accessing the shared resource, and the **ReleaseSRWLock**... functions are used to release the lock after accessing the resource. The lock must always be released in the same mode in which it was acquired.

# Code Sample (1) - Code without SRW locks

In order to demonstrate this demo. Please follow the following steps:

- Create the following 3 folders:
  1. **C:\Temp**
  2. **C:\Temp2**
  3. **C:\Temp3**

This example code is using multiple threads to write data to a file and then move the file to different directories. It creates several writer threads that write data to a file, then creates two other threads that move the file from one directory to another.

It doesn't use any locks to protect shared resources (the **file** and the **completedThreads** counter), which leads to a race conditions. For example, if two threads try to write to the file at the same time, the file data could be corrupted.

```c
#include <windows.h>
#include <iostream>
#include <sstream>
#include <thread>

#define NUM_WRITER_THREADS 10
#define TOTAL_THREADS (NUM_WRITER_THREADS + 2)
#define NUM_OPERATIONS 100

int completedThreads = 0;

DWORD WINAPI WriteToFile(LPVOID lpParam) {
    std::stringstream ss;
    ss << GetCurrentThreadId();
    std::string data = "Hello, World from Writer Thread " + ss.str() + "\r\n";

    for (int i = 0; i < NUM_OPERATIONS; i++) {
        HANDLE hFile = CreateFile(L"C:\\Temp\\log.txt", GENERIC_WRITE, 0, NULL, OPEN_ALWAYS, FILE_ATTRIBUTE_NORMAL, NULL);
        if (hFile == INVALID_HANDLE_VALUE) {
            std::cerr << "CreateFile (write) failed. Error: " << GetLastError() << std::endl;
            return 1;
        }

        SetFilePointer(hFile, 0, NULL, FILE_END);

        DWORD bytesWritten;

        if (!WriteFile(hFile, data.c_str(), data.size(), &bytesWritten, NULL)) {
            std::cerr << "WriteFile failed. Error: " << GetLastError() << std::endl;
            CloseHandle(hFile);
            return 1;
        }

        std::cout << "Writer Thread " << ss.str() << " wrote to the file: " << data;

        CloseHandle(hFile);

        std::this_thread::sleep_for(std::chrono::milliseconds(100));
    }

    completedThreads++;

    return 0;
}

DWORD WINAPI MoveFileToFolder(LPVOID lpParam) {
    std::stringstream ss;
    ss << GetCurrentThreadId();

    while (completedThreads < NUM_WRITER_THREADS) {
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
    }

    if (!MoveFileEx(L"C:\\Temp\\log.txt", L"C:\\Temp2\\log.txt", MOVEFILE_REPLACE_EXISTING)) {
        std::cerr << "MoveFileEx failed. Error: " << GetLastError() << std::endl;
        return 1;
    }

    std::cout << "Mover Thread " << ss.str() << " moved the file.\n";

    completedThreads++;

    return 0;
}

DWORD WINAPI MoveFileToAnotherFolder(LPVOID lpParam) {
    std::stringstream ss;
    ss << GetCurrentThreadId();

    while (completedThreads < NUM_WRITER_THREADS + 1) {
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
    }

    if (!MoveFileEx(L"C:\\Temp2\\log.txt", L"C:\\Temp3\\log.txt", MOVEFILE_REPLACE_EXISTING)) {
        std::cerr << "MoveFileEx failed. Error: " << GetLastError() << std::endl;
        return 1;
    }

    std::cout << "Mover Thread " << ss.str() << " moved the file to another folder.\n";

    return 0;
}

int main() {
    HANDLE hThreads[TOTAL_THREADS];

    for (int i = 0; i < NUM_WRITER_THREADS; i++) {
        hThreads[i] = CreateThread(NULL, 0, WriteToFile, NULL, 0, NULL);
        if (hThreads[i] == NULL) {
            std::cerr << "CreateThread error: " << GetLastError() << '\n';
            return 1;
        }
    }

    hThreads[NUM_WRITER_THREADS] = CreateThread(NULL, 0, MoveFileToFolder, NULL, 0, NULL);
    if (hThreads[NUM_WRITER_THREADS] == NULL) {
        std::cerr << "CreateThread error: " << GetLastError() << '\n';
        return 1;
    }

    hThreads[NUM_WRITER_THREADS + 1] = CreateThread(NULL, 0, MoveFileToAnotherFolder, NULL, 0, NULL);
    if (hThreads[NUM_WRITER_THREADS + 1] == NULL) {
        std::cerr << "CreateThread error: " << GetLastError() << '\n';
        return 1;
    }

    WaitForMultipleObjects(TOTAL_THREADS, hThreads, TRUE, INFINITE);

    for (int i = 0; i < TOTAL_THREADS; i++) {
        CloseHandle(hThreads[i]);
    }

    return 0;
}
```
Because of the lack of proper synchronization mechanisms, this code will not work as expected and leads to race conditions and data corruption.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/7ca910a0-3670-4feb-9d18-d7e3af38f451)


If we review the **C:\Temp** location, we can see the created file. However, as you may have notice. There is only one thread that wrote data to the file. The file is also supposed to end in the **C:\Temp3** though, which didn't happen in this case.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/1a8bf0ca-204b-4339-8b7f-645cd2204d4b)


The program will hang at some point and it won't proceed further with other operations. Here we can see that the other two threads are sleeping:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/abae8111-9c7f-490e-ba23-818509c20b18)


# Code Sample (2) - Code with SRW locks

This version of the code introduces the use of Slim Reader/Writer (SRW) locks. The SRW lock provides efficient synchronization mechanisms to protect shared resources from concurrent access.

The Slim Reader/Writer (SRW) locks serve two main purposes. First, they ensure thread safety by controlling access to shared resources, specifically the file operations and the **`completedThreads`** counter, to avoid race conditions and data corruption. 

Second, they control the order of execution, ensuring that file move operations are only performed after all write operations have completed to reduce potential conflicts between writing and moving operations.

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

        // Release the SRW lock
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

    // Release the SRW lock
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

- **WriteToFile** function:

The SRW lock is acquired in exclusive mode before the file is opened for writing, written to, and closed. This ensures that no other thread can perform any file operation while this thread is writing to the file, avoiding potential race conditions and data corruption.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/b73f9915-04e7-4094-a24c-e13415aa5daf)


The SRW lock is also used to protect the increment operation on **`completedThreads`**, which is a shared variable. This prevents data races that could occur if another thread reads/writes **`completedThreads`** at the same time.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/26281eae-c75e-4825-b47b-e37169f819d6)


- **MoveFileToFolder** function

The SRW lock is first acquired in shared mode to check the **`completedThreads`** counter.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/f6b4724f-d75e-4af4-97eb-ca868b9e39a4)


When it's time to move the file, the lock is acquired in exclusive mode. This makes sure that no other thread can interfere with the file or the **`completedThreads`** counter during the move operation.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/45d0de3c-e8de-4cdf-9a03-e6a48127f3b2)



- **MoveFileToAnotherFolder** function

Similar to the **`MoveFileToFolder`** function, the SRW lock is first acquired in shared mode to check the **`completedThreads`** counter. 

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/59c9b353-131c-4051-b583-371d9ae87e42)


When it's time to move the file, the lock is acquired in exclusive mode. This ensures that the move operation and the read/write operations on **`completedThreads`** are thread-safe.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/0b40f4f9-707a-4751-b8d2-dfb054c24c7c)


This code is now working as expected.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/fa1622cc-fdc9-48e4-9f26-2f80225454b3)


The file has successfully been copied to the **C:\Temp3** as well.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/637b9f7a-bf1a-40e3-9573-019a03460b60)


