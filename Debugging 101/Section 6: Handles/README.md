# What are Handles?

A handle is a reference or identifier that represents a resource. When a program wants to use a particular resource, like a file or a thread, the operating system provides a handle to the program. This handle is then used for further operations on that resource, such as reading from a file or making changes to what's displayed in a window.

Handles in Windows can refer to a variety of resource types. Here are **some** common examples:

- **File Handles:** Represent open files and directories.
- **Event Handles:** Used for event synchronization between threads or processes.
- **Mutex Handles:** Used for mutual exclusion in multi-threaded programming.
- **Semaphore Handles:** Used for limiting access to a resource.
- **Job Handles:** Represent a collection of processes as a single job object for resource allocation.
- **Token Handles:** Represent security tokens for user authentication and authorization.

As a basic guideline, if the official documentation of Microsoft for an API indicates that a handle is returned and advises you to invoke **`CloseHandle()`** to release it, then you can be confident that it's a real handle.

# What is Process Handle Table?

A Process Handle Table is an internal data structure maintained by the Windows operating system for each process. This table keeps track of all the "real" handles that the process has opened. These handles could be to various types of resources like files, registry keys, semaphores, events, and other Kernel Objects.

When a program calls a Windows API function that returns a handle (like CreateFile or OpenProcess), an entry is created in the Process Handle Table. This entry maps the handle value to the actual resource it represents.

The Process Handle Table is essential for resource management. It allows the operating system to keep track of which resources are being used by each process, ensuring proper cleanup when the process terminates. 

# Real Handles

Kernel Object Handles are considered "real handles." These handles are actual entries in a process's handle table and are managed by the Windows Kernel. They are used to interact with system-level resources like files, threads, processes, events, and more.

When you open or create a file using the **CreateFile** function, it returns a handle to a file object in the kernel space. This handle can be used to read from or write to the file.

```
HANDLE hFile = CreateFile(
    "test.txt",
    GENERIC_READ | GENERIC_WRITE,
    0,
    NULL,
    OPEN_EXISTING,
    FILE_ATTRIBUTE_NORMAL,
    NULL
);
```

Here, **`hFile`** is a Kernel Object Handle that you can use with functions like **ReadFile** and **WriteFile** to interact with the file. When we're done with a so-called "real handle", we typically need to close it explicitly using **`CloseHandle`** to free up system resources.

This can also be found in the official Microsoft documentation:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/86856a76-f3df-4150-8f4c-8ca0055e6034)


**Code Sample:**

```c
#include <windows.h>
#include <stdio.h>
#include <string.h>

int main() {
    HANDLE hFile;
    DWORD dwBytesWritten = 0;
    char data[] = "Hello, World!";

    // Create a real handle to a file
    hFile = CreateFile(L"test.txt",
        GENERIC_WRITE,
        0,
        NULL,
        CREATE_ALWAYS,
        FILE_ATTRIBUTE_NORMAL,
        NULL);

    if (hFile == INVALID_HANDLE_VALUE) {
        printf("Could not create file (error %d)\n", GetLastError());
        return 1;
    }

    // Print the real handle value to the console
    printf("Real handle for the file: %p\n", hFile);

    // Write data to the file
    if (!WriteFile(hFile, data, strlen(data), &dwBytesWritten, NULL)) {
        printf("Could not write to file (error %d)\n", GetLastError());
        CloseHandle(hFile);
        return 2;
    }

    // Close the real handle
    CloseHandle(hFile);

    return 0;
}
```

**Example:** The value 00000000000000A8 that we see is the real handle to the file we've created with **CreateFile**. This is an actual, unique identifier for the file resource, and it's stored in the process's handle table. We can use this handle to perform various operations on the file, like reading or writing for example.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/2a615630-593a-4698-8795-e61d63f46f6d)

# Pseudo Handles

A **pseudo handle** is a special identifier that refers to the calling process or thread. Unlike real handles, pseudo handles aren't stored in the process's handle table and can't be passed to child processes. There's no need to close a pseudo handle using **`CloseHandle()`**. However, if we convert it into a real handle, which can be used by other processes, then that new real handle must be explicitly closed to free system resources.

One common example of a pseudo handle is the handle returned by the **`GetCurrentProcess()`** function in the Windows API. This function returns a pseudo handle that refers to the calling process itself.

```c
#include <windows.h>
#include <stdio.h>

int main() {
    HANDLE hProcess = GetCurrentProcess();

    // Print the pseudo handle value to the console
    printf("Pseudo handle for the current process: %p\n", hProcess);

    // hProcess now contains a pseudo handle for the current process.
    // You can use hProcess with APIs that expect a process handle,
    // but only within the current process.

    // No need to close the pseudo handle
    // CloseHandle(hProcess);  // This line is not necessary

    return 0;
}
```

The value **`FFFFFFFFFFFFFFFF`** is a typical representation of a pseudo handle for the current process, which we get from calling **`GetCurrentProcess()`** in a 64-bit program. This handle is not a real handle. It's a constant value that the Windows API understands to refer to the current process

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/b3d6be22-d3f5-4ee5-8891-f3bf62927ad4)


Other functions that are using pseudo handles:

- **GetCurrentThread**
- **GetCurrentProcessToken**
- **GetCurrentThreadToken**
- **GetCurrentThreadEffectiveToken**

# Handles that are not real Handles

There are "API-specific handles" that are designed to be used with a particular set of API functions. Unlike Kernel Object Handles, which can be used across different types of system resources and functions, API-specific handles are limited in scope. **They are handles but are specialized for certain tasks.**

Here is an example:

```
HANDLE OpenEventLogA( <---- This is not a "real" Handle
  [in] LPCSTR lpUNCServerName,
  [in] LPCSTR lpSourceName
);
```

The handle returned by **`OpenEventLogA`** is considered an API-specific handle because it is customized for use with the Event Logging API in Windows. This means that the handle is not a Kernel Object Handle. It is a unique identifier that is only valid within the context of Event Logging functions.

Here are some characteristics that make it API-specific:

- **Limited Scope:** This handle is specifically designed for reading, writing, or querying event logs. We can't use it with other types of Windows API functions that expect different kinds of handles, like file or process handles.
- **Specialized Closure:** To close this handle, we would use **`CloseEventLog`** instead of the **`CloseHandle`** function.

**Code Sample:**

```c
#include <windows.h>
#include <iostream>

int main() {
    // Open the event log
    HANDLE hEventLog = OpenEventLogW(NULL, L"System");

    // Check for errors
    if (hEventLog == NULL) {
        std::cerr << "Could not open event log. Error code: " << GetLastError() << std::endl;
        return 1;
    }

    // Display the handle address
    std::cout << "Handle for the event log: " << hEventLog << std::endl;

    // Using CloseEventLog to close the event log handle
    CloseEventLog(hEventLog);

    return 0;
}
```

**Example:** The value **`0000028C12871D20`** is the memory address representation of the handle for the event log. This handle is a unique identifier provided by the **`OpenEventLogA`** or **`OpenEventLogW`** function and is specific to the Event Logging API. 

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/cd842e7d-8151-4075-a8bc-e0ef4ca534c1)


![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/3ebc244a-553d-4230-b44b-a24e1ffadfb8)

