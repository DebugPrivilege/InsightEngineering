# What are Interlocked Functions?

The Interlocked Functions provide a simple mechanism for synchronizing access to a **variable** that is shared by multiple threads. They also perform operations on variables in an atomic manner. Operations performed by interlocked functions are atomic, which means they are completed in a single, uninterruptible step. 

There are a lot of Windows APIs that can be used to work with Interlocked Functions. Here are the **common** one's:

| Function Name           | Description                                                                                                  |
|-------------------------|--------------------------------------------------------------------------------------------------------------|
| `InterlockedIncrement()`    | Atomically increments (increases by one) the value of the specified integer variable.                         |
| `InterlockedDecrement()`    | Atomically decrements (decreases by one) the value of the specified integer variable.                         |
| `InterlockedExchange()`     | Atomically sets a variable to the specified value.                                                           |
| `InterlockedCompareExchange()` | Atomically compares the value of a specified variable to a specified value and, if they are equal, changes the variable to a new value. |
| `InterlockedExchangeAdd()`  | Atomically adds two integers together and replaces the first integer with the result.                        |
| `InterlockedAdd()`          | Atomically adds two integers and stores the result in the destination.                                       |
| `InterlockedAnd()`          | Atomically performs a bitwise AND operation on the specified integer values.                                 |
| `InterlockedOr()`           | Atomically performs a bitwise OR operation on the specified integer values.                                  |
| `InterlockedXor()`          | Atomically performs a bitwise XOR operation on the specified integer values.                                 |
| `InterlockedExchangePointer()` | Atomically exchanges the values of two pointers.                                                            |

# Code Sample (1) - Threads accessing a global variable integer without interlocked access functions

This code could have race conditions because multiple threads are concurrently incrementing a shared variable, **`counter`**, without any synchronization. This could result in lost increments, as the operation **`counter++`** is not atomic and thus two threads could interfere with each other, **leading to incorrect final counter value**.

```c
#include <windows.h>
#include <iostream>
#include <vector>

// Global counter
int counter = 0;

// Function to increment counter, to be run in a separate thread
DWORD WINAPI incrementCounter(LPVOID lpParam) {
    for (int i = 0; i < 100000; i++) {
        counter++;
    }
    return 0;
}

int main() {
    // Create a number of threads
    const int numThreads = 10;
    std::vector<HANDLE> threads(numThreads);

    for (int i = 0; i < numThreads; i++) {
        threads[i] = CreateThread(
            NULL,       // default security attributes
            0,          // default stack size
            incrementCounter, // thread function name
            NULL,       // argument to thread function
            0,          // use default creation flags
            NULL);      // returns the thread identifier
    }

    // Wait until all threads have terminated.
    WaitForMultipleObjects(numThreads, &threads[0], TRUE, INFINITE);

    // Close all thread handles upon completion.
    for (int i = 0; i < numThreads; i++) {
        CloseHandle(threads[i]);
    }

    // Print final counter value
    std::cout << "Final counter value: " << counter << std::endl;

    return 0;
}
```

The expected counter value would be **1,000,000**. However, because of the race condition, the actual counter value could likely be less than this. When we ran this code, we may get some inconsistence results here and there.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/3e8b06ad-e8a7-414a-ad5c-be7bbd2af669)

# Code Sample (2) - Threads accessing a global variable integer with interlocked access functions

This code uses a function called **InterlockedIncrement**. This function is special because it makes sure that the entire process of increasing the counter by one (the increment operation) happens all at once and can't be interrupted by anything else. This is what we mean by 'atomicity'.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/34d52fde-0193-4603-8df2-9ed39eb522e1)

```c
#include <windows.h>
#include <iostream>
#include <vector>

// Global counter
LONG counter = 0;

// Function to increment counter, to be run in a separate thread
DWORD WINAPI incrementCounter(LPVOID lpParam) {
    for (int i = 0; i < 100000; i++) {
        InterlockedIncrement(&counter);
    }
    return 0;
}

int main() {
    // Create a number of threads
    const int numThreads = 10;
    std::vector<HANDLE> threads(numThreads);

    for (int i = 0; i < numThreads; i++) {
        threads[i] = CreateThread(
            NULL,       // default security attributes
            0,          // default stack size
            incrementCounter, // thread function name
            NULL,       // argument to thread function
            0,          // use default creation flags
            NULL);      // returns the thread identifier
    }

    // Wait until all threads have terminated.
    WaitForMultipleObjects(numThreads, &threads[0], TRUE, INFINITE);

    // Close all thread handles upon completion.
    for (int i = 0; i < numThreads; i++) {
        CloseHandle(threads[i]);
    }

    // Print final counter value
    std::cout << "Final counter value: " << counter << std::endl;

    return 0;
}
```

By using **InterlockedIncrement**, we ensure each increment is completed fully before anything else can happen to the counter. This ensures our counter is accurate, no matter how many threads are trying to increment it at the same time.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/d24699f8-28c7-419e-9094-ba23b476b38d)

# Code Sample (3) - Threads accessing a global variable integer without interlocked access functions

Here is another example of code that contains a global variable without interlocked access functions. The purpose of this code is to demonstrate multithreading, file operations, and the use of a shared global counter in a multithreaded context. 

However, this code has a race condition because the **increment** and **decrement** operations on the global counter are not **atomic**. This means that if two threads attempt to modify the counter at the same time, the final result might not be what we expect.

```c
#include <windows.h>
#include <iostream>
#include <string>
#include <vector>

// Set the number of files to create and delete
#define NUM_FILES 1000
// Set the number of repetitions for file creation and deletion
#define NUM_REPEATS_FILES 1
// Set the number of increments and decrements to perform on the global counter
#define NUM_REPEATS_COUNTER 1000000

// Global counter
LONG filesCreatedAndDeleted = 0;

DWORD WINAPI CreateAndDeleteFiles(LPVOID lpParam) {
    for (int j = 0; j < NUM_REPEATS_FILES; j++) {
        for (int i = 0; i < NUM_FILES; i++) {
            std::wstring fileName = L"C:\\Temp\\file_" + std::to_wstring(i) + L".txt";
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
                std::wcerr << L"CreateFile failed for " << fileName << std::endl;
                continue;
            }
            else {
                std::wcout << L"Created file: " << fileName << std::endl;
            }

            DWORD bytesWritten;
            WriteFile(
                hFile,
                "Hello World",
                11, // number of bytes in "Hello World"
                &bytesWritten,
                NULL
            );

            CloseHandle(hFile);

            if (!DeleteFile(fileName.c_str())) {
                std::wcerr << L"DeleteFile failed for " << fileName << std::endl;
            }
            else {
                std::wcout << L"Deleted file: " << fileName << std::endl;
                // Increment the global counter
                filesCreatedAndDeleted++;
            }
        }
    }

    return 0;
}

DWORD WINAPI IncrementCounterThread(LPVOID lpParam) {
    for (int i = 0; i < NUM_REPEATS_COUNTER; i++) {
        // Increment the global counter
        filesCreatedAndDeleted++;
    }
    return 0;
}

DWORD WINAPI DecrementCounterThread(LPVOID lpParam) {
    for (int i = 0; i < NUM_REPEATS_COUNTER; i++) {
        // Decrement the global counter
        filesCreatedAndDeleted--;
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

    hThreads[0] = CreateThread(NULL, 0, CreateAndDeleteFiles, NULL, 0, &threadID);
    hThreads[1] = CreateThread(NULL, 0, IncrementCounterThread, NULL, 0, &threadID);
    hThreads[2] = CreateThread(NULL, 0, DecrementCounterThread, NULL, 0, &threadID);

    WaitForMultipleObjects(3, hThreads, TRUE, INFINITE);

    // Get the current counter value at the end of the program
    LARGE_INTEGER end;
    QueryPerformanceCounter(&end);

    for (int i = 0; i < 3; i++) {
        CloseHandle(hThreads[i]);
    }

    // Calculate the total execution time in seconds
    double time = static_cast<double>(end.QuadPart - start.QuadPart) / frequency.QuadPart;

    std::wcout << L"Total execution time: " << time << L" seconds" << std::endl;
    // At the end, print the global counter
    std::wcout << L"Final value of the global counter: " << filesCreatedAndDeleted << std::endl;

    return 0;
}
```
The expected outcome of the counter should be **1000**, but due to the race condition. We might get different results, so I would suggest to run this code at least 3 times to see it by yourself. Here is an example after the 3rd attempt. I was getting a way different value than **100**.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/2a760015-3229-43b6-97c6-cae658f8f2dc)

# Code Sample (4) - Threads accessing a global variable integer with interlocked access functions

This is an improved version of the previous code because it uses the **InterlockedIncrement** and **InterlockedDecrement** functions to manipulate the global counter **`filesCreatedAndDeleted`**. These functions ensure that increment and decrement operations are atomic, which means they're completed in one undisturbed step.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/d6dd4b7c-e7e1-417d-a5ed-fb72a2022318)


Code sample:

```c
#include <windows.h>
#include <iostream>
#include <string>
#include <vector>

// Set the number of files to create and delete
#define NUM_FILES 1000
// Set the number of repetitions for file creation and deletion
#define NUM_REPEATS_FILES 1
// Set the number of increments and decrements to perform on the global counter
#define NUM_REPEATS_COUNTER 1000000

// Global counter
LONG filesCreatedAndDeleted = 0;

DWORD WINAPI CreateAndDeleteFiles(LPVOID lpParam) {
    for (int j = 0; j < NUM_REPEATS_FILES; j++) {
        for (int i = 0; i < NUM_FILES; i++) {
            std::wstring fileName = L"C:\\Temp\\file_" + std::to_wstring(i) + L".txt";
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
                std::wcerr << L"CreateFile failed for " << fileName << std::endl;
                continue;
            }
            else {
                std::wcout << L"Created file: " << fileName << std::endl;
            }

            DWORD bytesWritten;
            WriteFile(
                hFile,
                "Hello World",
                11, // number of bytes in "Hello World"
                &bytesWritten,
                NULL
            );

            CloseHandle(hFile);

            if (!DeleteFile(fileName.c_str())) {
                std::wcerr << L"DeleteFile failed for " << fileName << std::endl;
            }
            else {
                std::wcout << L"Deleted file: " << fileName << std::endl;
                // Increment the global counter
                InterlockedIncrement(&filesCreatedAndDeleted);
            }
        }
    }

    return 0;
}

DWORD WINAPI IncrementCounterThread(LPVOID lpParam) {
    for (int i = 0; i < NUM_REPEATS_COUNTER; i++) {
        // Increment the global counter
        InterlockedIncrement(&filesCreatedAndDeleted);
    }
    return 0;
}

DWORD WINAPI DecrementCounterThread(LPVOID lpParam) {
    for (int i = 0; i < NUM_REPEATS_COUNTER; i++) {
        // Decrement the global counter
        InterlockedDecrement(&filesCreatedAndDeleted);
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

    hThreads[0] = CreateThread(NULL, 0, CreateAndDeleteFiles, NULL, 0, &threadID);
    hThreads[1] = CreateThread(NULL, 0, IncrementCounterThread, NULL, 0, &threadID);
    hThreads[2] = CreateThread(NULL, 0, DecrementCounterThread, NULL, 0, &threadID);

    WaitForMultipleObjects(3, hThreads, TRUE, INFINITE);

    // Get the current counter value at the end of the program
    LARGE_INTEGER end;
    QueryPerformanceCounter(&end);

    for (int i = 0; i < 3; i++) {
        CloseHandle(hThreads[i]);
    }

    // Calculate the total execution time in seconds
    double time = static_cast<double>(end.QuadPart - start.QuadPart) / frequency.QuadPart;

    std::wcout << L"Total execution time: " << time << L" seconds" << std::endl;
    // At the end, print the global counter
    std::wcout << L"Final value of the global counter: " << filesCreatedAndDeleted << std::endl;

    return 0;
}
```

With these interlocked functions, even if multiple threads try to increment or decrement the counter simultaneously, each operation will be completed fully before the next one begins. Run this code 3 times and see if you will get a different result than **1000**.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/01982c8b-c6ee-497a-9953-b4bac77327a7)

