# What is a Semaphore?

A semaphore is a synchronization mechanism that can be used to control access to shared resources in a concurrent or multi-threaded program. 

The following Windows APIs can be used to interact with Semaphores:

| Function                | Description                                                                 |
|-------------------------|-----------------------------------------------------------------------------|
| `CreateSemaphore()` / `CreateSemaphoreEx()` | These functions are used to create a semaphore. They return a handle to the semaphore that is used in semaphore functions. |
| `OpenSemaphore()`           | This function opens an existing named semaphore.                            |
| `ReleaseSemaphore()`        | This function increases the count of the semaphore object by a specified amount.    |

# Code Sample (1) - Code without a Semaphore

This code sample has issues because we don't have a synchronization mechanism in place. We're sharing a resource, specifically the file at **C:\Temp\log.txt**, with multiple threads without any safeguards. This lack of protection can lead to race conditions, where multiple threads try to read or write to the file at the same time. 

The purpose of this goal is to attempt to write to a file, read from a file, and move a file to a different folder. However, this code will have some unpredictable behavior and data inconsistencies due to the race condition.

```c
#include <windows.h>
#include <iostream>
#include <sstream>
#include <thread>
#include <vector>

// Define the number of threads for file operations and total threads
#define NUM_FILE_OP_THREADS 10
#define TOTAL_THREADS (NUM_FILE_OP_THREADS + 1)  // +1 for the file mover thread
#define NUM_OPERATIONS 100

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


int main() {
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

    // Create the thread to move the file
    hThreads[NUM_FILE_OP_THREADS] = CreateThread(NULL, 0, MoveFileToFolder, NULL, 0, NULL);
    // Check if the thread was created successfully
    if (hThreads[NUM_FILE_OP_THREADS] == NULL) {
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

Running this code will lead to inconsistent output, failed operations, and unpredictable behavior due to the lack of synchronization mechanism.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/7361fd26-a2b9-4be6-a0cf-c5b8f840d4b1)

As we can see here, we have 10 threads in total. However, we can see that only 1 thread has written to the log file. Also, pay a close attention to the size of the file. It is only 5 KB in this example.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/82e8ecd5-29fa-4125-b8cc-aa44754ece53)

We also can see that the log file hasn't moved to the different folder either.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/6797bc7c-3bd4-4b8a-9f7a-83bd8516d3e6)


# Code Sample (2) - Code with Semaphore

This version of the code is better than the previous one due to the addition of a semaphore, which provides synchronization between the threads.

```c
#include <windows.h>
#include <iostream>
#include <sstream>
#include <thread>
#include <vector>

// Define the number of threads for file operations and total threads
#define NUM_FILE_OP_THREADS 10
#define TOTAL_THREADS (NUM_FILE_OP_THREADS + 1)  // +1 for the file mover thread
#define NUM_OPERATIONS 100

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


int main() {
    // Create the semaphore
    // NULL: pointer to a SECURITY_ATTRIBUTES structure that determines whether the returned handle can be inherited by child processes. 
    // If NULL, the handle cannot be inherited.
    // 1: the initial count for the semaphore object. This value must be greater than or equal to 0 and less than or equal to lMaximumCount.
    // 1: the maximum count for the semaphore object. This value must be greater than zero.
    // NULL: pointer to a null-terminated string specifying the name of the semaphore object. If this parameter is NULL, the semaphore is unnamed.
    hSemaphore = CreateSemaphore(NULL, 1, 1, NULL);
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

    // Close all thread handles for file operations
    for (int i = 0; i < NUM_FILE_OP_THREADS; i++) {
        CloseHandle(hThreads[i]);
    }

    // Create the thread to move the file
    HANDLE hMoveThread = CreateThread(NULL, 0, MoveFileToFolder, NULL, 0, NULL);
    // Check if the thread was created successfully
    if (hMoveThread == NULL) {
        std::cerr << "CreateThread error: " << GetLastError() << '\n';
        return 1;
    }

    // Wait for move thread to finish
    WaitForSingleObject(hMoveThread, INFINITE);

    // Close move thread handle
    CloseHandle(hMoveThread);

    // Close the semaphore
    CloseHandle(hSemaphore);

    return 0;
}
```

This is the same code as the previous one that create multiple threads to write to, read from, and move a file. However, this time there is a semaphore that is used to ensure that these operations do not occur simultaneously, avoiding race conditions. 

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/9b31e122-e226-42ec-9549-d39da4048fd2)

The file has now been moved successfully and we are able to see that more threads have written to the file.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/efd8eb08-b680-45fb-bbac-0e547285a418)

# How does this Semaphore works within our code?

Here are the parameters of the **CreateSemaphore** function:

```
HANDLE CreateSemaphoreA(
  [in, optional] LPSECURITY_ATTRIBUTES lpSemaphoreAttributes,
  [in]           LONG                  lInitialCount,
  [in]           LONG                  lMaximumCount,
  [in, optional] LPCSTR                lpName
);
```
Let's now break down the code and later on... We will start playing around with the initial counts.

The semaphore is created in the **`main`** function with an initial count of **1** and a maximum count of **1**:

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/a76b997b-cec2-43cd-b78d-b405e539fc4d)

This line creates a semaphore with an initial count of 1 and a maximum count of 1. This means that only one thread can access the semaphore at a time.

Throughout the code, there are several places where a thread waits for the semaphore to become available:

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/662c4560-1925-4b2f-8e51-9c3d0091ef6a)

This line attempts to reduce the semaphore's count. If the semaphore's count is greater than **0**, the function immediately returns and the count is reduced. If the semaphore's count is **0**, the function blocks until the semaphore is released by another thread.

When a thread calls **`WaitForSingleObject(hSemaphore, INFINITE);`**, it's like it's asking the semaphore, *"Can I enter the critical section?"*

If the semaphore's count is already **0** when **WaitForSingleObject** is called, it means all permissions have been given out and none are available. In this case, the thread has to wait. The thread is put in a waiting state and will not continue its execution until it gets permission. 

Once the thread is done with the resource, it releases the semaphore, which increments the count:

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/294d4cd4-3aac-475d-8f79-00e85225a0f2)

This signals that the resource is now free for other threads to use.

By reducing the count when a thread acquires the semaphore and increasing the count when it's released, the semaphore keeps track of how many threads are using the resource at any given time.

# Why are we decrementing and incrementing the Semaphore Count?

We are reducing (or decrementing) the semaphore count to indicate that a thread has started using the shared resource (in this case, the file). When the count is 0, it means the resource is currently in use. If another thread attempts to decrement the count while it's 0, that thread will be blocked until the count is greater than 0 again. 

The count will be incremented back to 1 when the current thread using the resource calls **ReleaseSemaphore**, indicating that it has finished using the resource and it is available for other threads.

# Hands-on Excercise (1) - Violating the requirements of CreateSemaphore

Let's play around with the **lInitialCount** parameter of the **CreateSemaphore** function. First, let's compile the following code:

```c
#include <windows.h>
#include <iostream>
#include <sstream>
#include <thread>
#include <vector>

// Define the number of threads for file operations and total threads
#define NUM_FILE_OP_THREADS 10
#define TOTAL_THREADS (NUM_FILE_OP_THREADS + 1)  // +1 for the file mover thread
#define NUM_OPERATIONS 100

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


int main() {
    // Create the semaphore
    // NULL: pointer to a SECURITY_ATTRIBUTES structure that determines whether the returned handle can be inherited by child processes. 
    // If NULL, the handle cannot be inherited.
    // 1: the initial count for the semaphore object. This value must be greater than or equal to 0 and less than or equal to lMaximumCount.
    // 1: the maximum count for the semaphore object. This value must be greater than zero.
    // NULL: pointer to a null-terminated string specifying the name of the semaphore object. If this parameter is NULL, the semaphore is unnamed.
    hSemaphore = CreateSemaphore(NULL, 1, 1, NULL);
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

    // Close all thread handles for file operations
    for (int i = 0; i < NUM_FILE_OP_THREADS; i++) {
        CloseHandle(hThreads[i]);
    }

    // Create the thread to move the file
    HANDLE hMoveThread = CreateThread(NULL, 0, MoveFileToFolder, NULL, 0, NULL);
    // Check if the thread was created successfully
    if (hMoveThread == NULL) {
        std::cerr << "CreateThread error: " << GetLastError() << '\n';
        return 1;
    }

    // Wait for move thread to finish
    WaitForSingleObject(hMoveThread, INFINITE);

    // Close move thread handle
    CloseHandle(hMoveThread);

    // Close the semaphore
    CloseHandle(hSemaphore);

    return 0;
}
```

Once the code has been compiled. Let's go to the **main** function and change the following values to **0** of the **lInitialCount** and **lMaximumCount** parameters:

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/24393ffb-caf2-4622-850f-2f2f9916f890)

When we are trying to run this code, we encounter an error code:

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/d2e85db8-53b2-402f-b0a9-9aa126dc9061)

Why is this the case? Well, this is pretty straightforward. The error code **87** in Windows corresponds to **ERROR_INVALID_PARAMETER**. This means that one or more parameters we passed to the function **CreateSemaphore** are invalid.  We are setting both the **initial count** and the **maximum count** to 0. 

According to the function's documentation, the **maximum count** must be **greater** than **zero**. By setting it to zero, we are violating this requirement.

# Hands-on Excercise (2) - Non-Signaled State

If we initialize the semaphore with **`CreateSemaphore(NULL, 0, 1, NULL)`**, we are setting the semaphore's count to **0** initially, and we're stating that the maximum count for the semaphore can be **1**.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/37a655f7-13c2-4075-9ba4-d3a63c7a4592)

What this means for us is that our semaphore starts in the nonsignaled state (since its count is 0). So, **if we have any thread** that calls **`WaitForSingleObject(hSemaphore, INFINITE)`**, it will block because it's waiting for the semaphore to be available.

Let's run the program now and we will see that it gets stuck, since it's in a nonsignaled state.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/52b88f92-dbf5-4d9b-9e20-f3430abaf245)

No threads will be able to acquire the semaphore and start executing code, because the count is 0.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/9e04ce5d-5f6d-4ef8-b762-968e8be41da6)

# Hands-on Excercise (3) - Non-Signaled State + Releasing Semaphore at the Start

In the previous example, we were able to observe that the program was getting stuck, since it was still in a **nonsignaled state**. What happens if we keep it in a nonsignaled state, but release a Semaphore at the beginning of a function?

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/6cad229e-d9aa-4d70-9d72-28d220026b8a)

Here we are making a call to **ReleaseSemaphore** at the beginning of the **`WriteToFile`** function, incrementing the semaphore's count by **1**. This allows another thread to start executing because **WaitForSingleObject** will not block when the semaphore's count is greater than **0**.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/d92fc759-e507-443d-85ef-65da472e3a52)

The data that we are getting is very inconsistent and incomplete.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/88e3bc32-c4c7-44eb-974c-fe6a1eab19ba)

If we release the semaphore at the start of the **`WriteToFile`** function, we're essentially allowing another thread to start working with the file before the current thread has finished its task. The semaphore should be released only after a thread has finished working with the shared resource. This ensures that the shared resource (the file) is accessed by only one thread at a time.

# Bonus Section - Inter-Process Synchronization with Semaphore

A Semaphore can be used for inter-process synchronization by serving as a signal between processes. We won't go into much details at this section, but the purpose is to demonstrate that it is possible to use inter-process synchronization when using Semaphore's. In this example, we are specifiying a Semaphore Object name called **"NewSemaphoreObject"**.

1. Start compiling the main code as **SemaphoreObject.exe**:

```c
#include <windows.h>
#include <iostream>
#include <sstream>
#include <thread>
#include <vector>

// Define the number of threads for file operations and total threads
#define NUM_FILE_OP_THREADS 10
#define TOTAL_THREADS (NUM_FILE_OP_THREADS + 1)  // +1 for the file mover thread
#define NUM_OPERATIONS 100

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


int main() {
    // Create the semaphore
    // NULL: pointer to a SECURITY_ATTRIBUTES structure that determines whether the returned handle can be inherited by child processes. 
    // If NULL, the handle cannot be inherited.
    // 1: the initial count for the semaphore object. This value must be greater than or equal to 0 and less than or equal to lMaximumCount.
    // 1: the maximum count for the semaphore object. This value must be greater than zero.
    // "NewSemaphoreObject": pointer to a null-terminated string specifying the name of the semaphore object. If this parameter is NULL, the semaphore is unnamed.
    hSemaphore = CreateSemaphore(NULL, 1, 1, TEXT("NewSemaphoreObject"));
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

    // Close all thread handles for file operations
    for (int i = 0; i < NUM_FILE_OP_THREADS; i++) {
        CloseHandle(hThreads[i]);
    }

    // Create the thread to move the file
    HANDLE hMoveThread = CreateThread(NULL, 0, MoveFileToFolder, NULL, 0, NULL);
    // Check if the thread was created successfully
    if (hMoveThread == NULL) {
        std::cerr << "CreateThread error: " << GetLastError() << '\n';
        return 1;
    }

    // Wait for move thread to finish
    WaitForSingleObject(hMoveThread, INFINITE);

    // Close move thread handle
    CloseHandle(hMoveThread);

    // Close the semaphore
    CloseHandle(hSemaphore);

    return 0;
}
```

2. Start compiling the second code as **SemaphoreObject2.exe**

The first program creates the semaphore and uses it to synchronize the writing of data to a file from multiple threads. The second program opens the same semaphore and uses it to synchronize the writing of data to the console.

```c
#include <windows.h>
#include <iostream>
#include <sstream>
#include <thread>

#define NUM_OPERATIONS 100

// Function to write to console
DWORD WINAPI WriteToConsole(LPVOID lpParam) {
    // Create a string that contains the thread ID
    std::stringstream ss;
    ss << GetCurrentThreadId();
    std::string data = "Second Program's Thread " + ss.str() + " is writing to the console.\r\n";

    // Repeat the write operation a certain number of times
    for (int i = 0; i < NUM_OPERATIONS; i++) {
        // Wait for the semaphore to be available
        WaitForSingleObject(lpParam, INFINITE);

        // Write to the console
        std::cout << data;

        // Release the semaphore
        ReleaseSemaphore(lpParam, 1, NULL);

        // Pause for 100 milliseconds
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
    }

    return 0;
}

int main() {
    // Attempt to open the semaphore
    HANDLE hSemaphore = NULL;
    while (hSemaphore == NULL) {
        hSemaphore = OpenSemaphore(SYNCHRONIZE | SEMAPHORE_MODIFY_STATE, FALSE, TEXT("NewSemaphoreObject"));
        if (hSemaphore == NULL) {
            std::cout << "Waiting for semaphore...\n";
            std::this_thread::sleep_for(std::chrono::seconds(1));
        }
    }

    // Now that we have the semaphore, we can start the writing process
    WriteToConsole(hSemaphore);

    // Close the semaphore
    CloseHandle(hSemaphore);

    return 0;
}
```

# Theory vs Practice - Inter-Process Synchronization with Semaphore

1. Start running the **SemaphoreObject2.exe** first

We are waiting until the **"NewSemaphoreObject"** becomes available.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/f25aba68-59a3-4f61-a2ea-6cbcd56de9db)

2. Now run the second program; **SemaphoreObject.exe**

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/63beec7e-e82b-4fa5-bdf6-e3029c1545c5)

3. Go back to the **SemaphoreObject2.exe** and see if we have new messages being printed in the console. If the program can successfully open the semaphore, it will proceed with its operations and start writing to the console.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/d97e11c2-3d19-46c8-b49c-b2a7fcdfaa1d)

# Process Explorer - View Semaphore Objects

We can use **Process Explorer** to view (named) Semaphore Objects of a running process. Click on the process and go to the 'Handles' tab. From there, we should be able to see the **"NewSemaphoreObject"** for example.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/713679e9-d540-4519-a960-adc5c6e7f153)

