# What is a Mutex?

A Mutex, short for "mutual exclusion," is a concept used in multi-threading programs to prevent multiple **threads** or **processes** from accessing some piece of data or code at the same time. 

A Mutex is like a lock that a piece of code needs to acquire before it can proceed. If the Mutex is already "locked" (i.e., another thread or process has acquired it), then the piece of code must wait until the Mutex is "unlocked" before it can proceed.

The following Windows APIs can be used for working with Mutexes:

| Function      | Description |
| ------------- | ----------- |
| `CreateMutex()`   | This function creates a new mutex object. If a mutex with the specified name already exists, the function opens the existing mutex and returns a handle to it. |
| `OpenMutex()`     | This function opens an existing named mutex object. |
| `ReleaseMutex()`  | This function releases ownership of a mutex object. A thread can use this function to release a mutex that it owns. |

# Code Sample (1) - No Mutex

This example demonstrates an issue where there is a lack of synchronization between reading and writing threads, which can lead to race conditions and inconsistent data.

```c
#include <windows.h>
#include <iostream>
#include <sstream>
#include <thread>
#include <vector>

// Define the number of threads and operations
#define NUM_THREADS 10
#define NUM_OPERATIONS 100

// Function to write to a file
DWORD WINAPI WriteToFile(LPVOID lpParam) {
    // Create a string that contains the thread ID
    std::stringstream ss;
    ss << GetCurrentThreadId();
    std::string data = "Writer Thread " + ss.str() + " is writing to the log.\r\n";

    // Repeat the write operation a certain number of times
    for (int i = 0; i < NUM_OPERATIONS; i++) {
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

        // Pause for 100 milliseconds
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
    }

    return 0;
}

// The main function
int main() {
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

    return 0;
}
```

If we run this code, it will create ten threads; half of them will write a message including their thread ID to a file, and the other half will read from the file. However, since there is no synchronization mechanism, these operations may overlap in an unpredictable way, resulting in inconsistent data in the file or in the console output.

Here we already can see some inconsistent data in the console output:

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/168a7042-a587-4875-9c0d-2ee8338e980e)

The written file happens to be in this case also 0 KB:

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/1e943899-568c-4541-89a5-73f967d6364a)

When we re-run our program, it will now give another result. As we can see here, the output is very inconsistent due to the race condition.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/65205b6b-008b-4ec7-802d-5e41b6244fc3)

This time we have some data in the created text file, but it's only coming from one thread.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/873f5343-96a4-4d8e-a71b-9cf01f00eb1f)

# Code Sample (2) - Mutex

This code creates ten threads that either write to or read from a file. It improves the previous code by adding a **Mutex**, which can be used to reduce the risks of race conditions and ensure that only one thread can access the file at a time. The Mutex is locked with **WaitForSingleObject** before a file operation and released with **ReleaseMutex** afterwards. In our example, we assigned a name to our **Mutex**, which happens to be **NewMutex**. 

The threads created in this code all belong to the same process and can therefore access the Mutex via the **hMutex** handle, regardless of the Mutex's name. However, we will cover an example that involves inter-process synchronization later on.

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
    hMutex = CreateMutex(NULL, FALSE, L"NewMutex");
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

When a write thread wants to write to the file, it waits until it can acquire the **mutex**, writes to the file, and then releases the **mutex**. The read threads does the same, they wait until they can acquire the **mutex**, read from the file, and then release the mutex. The **mutex** acts as a kind of "lock" that ensures only one thread can access the file at any given time.

This time we are getting a much more consistent data because the mutex ensures that access to the file is properly synchronized.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/cda4bcd5-4121-406b-8e62-05bc68366623)

The same thing applies for the data that is written to the created text file, which looks much more consistent now comparing to the previous one.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/3ea61023-1e69-4ebf-81f5-99e0be6b48e9)

# Process Explorer - View Mutex of Process

As discussed previously, a mutex (short for "mutual exclusion") is a synchronization primitive that provides exclusive access to shared resources, ensuring that only one thread at a time can use a resource. This can be viewed in Process Explorer under the 'Handles' tab of a running process.

Notice the use of the type name "Mutant" instead of "Mutex".  It's a historic name.  

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/fa8f7f78-fc30-46fc-b5c4-621d1409c5d1)

# Code Sample (3) - Inter-Process Synchronization

Inter-process synchronization is a mechanism that prevents multiple processes from accessing the same resource simultaneously, avoiding inconsistencies or conflicts. For example, if we have **two processes** that **both need to write to the same file**, we can use a **mutex** to ensure that only one process writes to the file at any given time. The other process would wait until the mutex is released before it can proceed, thus preventing both processes from writing to the file simultaneously and possibly causing data corruption.

Let's put the theory to the test.

1. Start compiling the following code as **Program.exe**:

The Mutex in this code, named **NewMutex**, is responsible for coordinating access to a shared resource. It ensures that only one thread can write to or read from the file at any given time, preventing data corruption or inconsistent reads.

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
    std::string data = "Writer Thread " + ss.str() + " is writing to the log\r\n";

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
        std::cout << "Writer Thread " << ss.str() << " is writing to the log\n";

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
    hMutex = CreateMutex(NULL, FALSE, L"NewMutex");
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

    // Wait for user input before closing the program
    std::cout << "Press Enter to close the program...\n";
    std::cin.get();

    return 0;
}
```

2. Start now to compile the second code as **Program2.exe**:

This code is another multi-threaded program that is designed to work in conjunction with the previous code. It opens the same named mutex, **NewMutex**. This allows it to synchronize its operations with the other program.

```c
#include <windows.h>
#include <iostream>
#include <sstream>
#include <thread>

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
    std::string data = "Thread ID " + ss.str() + " says Hi to NewMutex\r\n";

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
        std::cout << "Thread ID " << ss.str() << " has opened the NewMutex\n";
        std::cout << data;

        // Close the file
        CloseHandle(hFile);
        // Release the mutex
        ReleaseMutex(hMutex);
    }

    return 0;
}

int main() {
    // Open the existing mutex
    hMutex = OpenMutex(SYNCHRONIZE, FALSE, L"NewMutex");
    // Check if the mutex was opened successfully
    if (hMutex == NULL) {
        std::cerr << "OpenMutex error: " << GetLastError() << '\n';
        return 1;
    }

    // Create an array to hold the thread handles
    HANDLE hThreads[NUM_THREADS];
    // Create the threads
    for (int i = 0; i < NUM_THREADS; i++) {
        hThreads[i] = CreateThread(NULL, 0, WriteToFile, NULL, 0, NULL);
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
This code, when run concurrently with the previous program, demonstrates inter-process synchronization. The two programs are able to write to the same file in a coordinated manner, without conflicting with each other.

A mutex name is a string that uniquely identifies a mutex object in an operating system. Mutexes are system-wide and can be accessed by any process in the system, provided they know the name of the mutex. This makes named mutexes especially useful for inter-process synchronization.

A named mutex can be created using the **CreateMutex** function and specifying a name. Other processes can then call **OpenMutex** with the same name to obtain a handle to the existing mutex. This allows multiple processes to synchronize access to shared resources, such as files or memory.

# Theory vs Practice - Mutex Inter-Process Synchronization

1. Let's run **Program.exe**

This code will write the following to the log file:

*`Writer Thread <ID> is writing to the log`*

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/63621805-3fc2-4c5c-88f0-d6202b7e57e8)


2. At the same time start **Program2.exe**

This code will write the following to the log file:

*`Thread <ID> says Hi to NewMutex`*

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/fdc28812-c751-452e-9b38-e19878bcc77d)

Let's now open the log file in the **C:\Temp** folder and see how it looks like. 

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/3f8a6bb8-1aa6-41cb-a324-85f882a3a0be)

**Program.exe** has in total 10 threads, but only **5** threads are writing data to the file. Each writer thread performs **100** write operations, so if we are doing the basic math. 5 x 100 = **500** lines. 

While **Program2.exe** has also 10 threads, but all these threads are doing the same thing which is writing. Since all threads are write threads, and each write thread performs 100 write operations. 10 x 100 = **1000** lines.

This is why we are seeing **1500** lines in total.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/a7f2c511-78f8-4f9a-8c4b-c7c3da08c4e6)

# Using a Singleton Mutex to Prevent Multiple Instances

We can use a Mutex to ensure that only one single instance of our program is running. The only thing we have to do is check for **ERROR_ALREADY_EXISTS** right after the creation of our Mutex.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/72788015-03b5-49c5-b55c-29c9e8ca53d6)

Code snippet:

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
    std::string data = "Writer Thread " + ss.str() + " is writing to the log\r\n";

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
        std::cout << "Writer Thread " << ss.str() << " is writing to the log\n";

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

int main() {
    // Create a mutex
    hMutex = CreateMutex(NULL, FALSE, L"NewMutex");
    // Check if the mutex was created successfully
    if (hMutex == NULL) {
        std::cerr << "CreateMutex error: " << GetLastError() << '\n';
        return 1;
    }

    // If the mutex already exists, then an instance is already running
    if (GetLastError() == ERROR_ALREADY_EXISTS) {
        std::cerr << "Another instance of the program is already running\n";
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

    // Wait for user input before closing the program
    std::cout << "Press Enter to close the program...\n";
    std::cin.get();

    return 0;
}
```

If we try to run a second instance of the program while the first instance is still running, the second instance will fail to create or open the singleton mutex because it already exists (as created by the first instance).

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/43d6fbc5-9786-4360-bf69-f50ac11dd533)

