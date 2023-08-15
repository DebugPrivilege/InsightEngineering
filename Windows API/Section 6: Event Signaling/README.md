# What is an Event?

An event in synchronization in the context of multithreaded programming allows a thread to notify one or more other threads about a particular condition or state change. It is used to coordinate the operations of multiple threads, ensuring certain tasks are executed in a specific order or certain conditions are met before proceeding. 

There are Windows APIs provided specifically for creating and managing event objects: 

| Function     | Description |
|--------------|-------------|
| `CreateEvent()`  | This function is used to create a new event object. It allows you to specify whether the event should automatically reset and what its initial state should be. |
| `SetEvent()`     | This function is used to set an event object to the signaled state. If the event is auto-reset, it will automatically return to the nonsignaled state after releasing a single waiting thread. |
| `ResetEvent()`   | This function is used to manually set an event object to the nonsignaled state. |
| `OpenEvent()`    | This function is used to open an existing named event object. It requires the name of the event and the access required (e.g., `EVENT_ALL_ACCESS`, `EVENT_MODIFY_STATE`). |


There are two types of event objects: **manual-reset** events and **auto-reset** events:

| Type | Description |
| --- | --- |
| Manual-Reset Event | This type of event must be manually reset back to the non-signaled state after it's been signaled. This means that if multiple threads are waiting on a manual-reset event, and the event becomes signaled, all waiting threads are released. However, the event stays in the signaled state until it's manually reset by a call to the `ResetEvent` function. |
| Auto-Reset Event| When an auto-reset event is signaled, it automatically returns to the non-signaled state after releasing a single waiting thread or after a single `WaitForSingleObject` call. In other words, it automatically resets to the non-signaled state as soon as a single waiting thread is released. |

# Event States

An event object can be in one of two states:

| State | Description |
|-------|-------------|
| Signaled | When an event object is in the signaled state, it means that the condition it represents has been met. In other words, something of interest to other threads or processes has happened. For example, if an event object represents the completion of a file download, it would be set to the signaled state when the file download is complete. |
| Non-signaled | When an event object is in the non-signaled state, it means that the condition it represents has not been met. Using the same example, if the file download has not yet completed, the event object representing the download completion would be in the non-signaled state. |

# Code Sample (1) - Auto-Reset Event using Non-Signaled State

Let's try to explain the theory from a code-level perspective now. We'll first start with an simple example that covers an **auto-reset event** using a **non-signaled** State. In this code, the **CreateEvent** function is called with the second parameter (**bManualReset**) set to **FALSE**, which means it creates an **auto-reset event**. 

Also, the third parameter (**bInitialState**) is set to **FALSE**. This means the initial state of the event object is **non-signaled**. A **non-signaled** state indicates that the event object, when first created, will not release any threads that are waiting on it. They will only be released when the event state becomes signaled.

An Event object becomes "signaled" when the **SetEvent** function is called on that Event object. This is how one thread communicates to another that a certain condition has been met or an event has occurred.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/1fe67351-3c66-4b27-9eb5-5f9131bac6a1)


This code demonstrates the use of an **automatic-reset** event, where one thread **`SignalEvent`** signals the event after sleeping, allowing another waiting thread **`WaitForEvent`** to proceed, print a message, and then wait again, demonstrating how the event automatically resets to **non-signaled** after releasing a single waiting thread. 

```c
#include <windows.h>
#include <iostream>

// This function waits for an event to be signaled
DWORD WINAPI WaitForEvent(LPVOID lpParam) {
    HANDLE hEvent = *(HANDLE*)lpParam;

    WaitForSingleObject(hEvent, INFINITE);
    std::cout << "WaitForEvent: The event was signaled\n";

    WaitForSingleObject(hEvent, INFINITE);
    std::cout << "WaitForEvent: The event was signaled again\n";

    return 0;
}

// This function signals an event
DWORD WINAPI SignalEvent(LPVOID lpParam) {
    HANDLE hEvent = *(HANDLE*)lpParam;

    Sleep(1000);  // Sleep for 1 second
    std::cout << "SignalEvent: Signaling the event\n";
    SetEvent(hEvent);

    Sleep(5000);  // Sleep for 5 seconds
    std::cout << "SignalEvent: Signaling the event again\n";
    SetEvent(hEvent);

    return 0;
}

int main() {
    DWORD threadID;
    HANDLE hThreads[2];

    // Create an automatic reset event
    // Parameters for CreateEvent function:
    // 1. NULL - Security attributes, if NULL the handle cannot be inherited by child processes
    // 2. FALSE - Manual-reset flag; if FALSE, the function creates an auto-reset event object
    // 3. FALSE - Initial state; if FALSE, the initial state is non-signaled
    // 4. NULL - Name of the event object; if NULL, the event is unnamed
    HANDLE hEvent = CreateEvent(NULL, FALSE, FALSE, NULL);
    if (hEvent == NULL) {
        std::cerr << "Failed to create event. Error: " << GetLastError() << std::endl;
        return 1;
    }

    hThreads[0] = CreateThread(NULL, 0, WaitForEvent, &hEvent, 0, &threadID);
    if (hThreads[0] == NULL) {
        std::cerr << "Failed to create WaitForEvent thread. Error: " << GetLastError() << std::endl;
        return 1;
    }

    hThreads[1] = CreateThread(NULL, 0, SignalEvent, &hEvent, 0, &threadID);
    if (hThreads[1] == NULL) {
        std::cerr << "Failed to create SignalEvent thread. Error: " << GetLastError() << std::endl;
        return 1;
    }

    WaitForMultipleObjects(2, hThreads, TRUE, INFINITE);

    for (int i = 0; i < 2; i++) {
        if (!CloseHandle(hThreads[i])) {
            std::cerr << "CloseHandle failed for thread " << i << ". Error: " << GetLastError() << std::endl;
        }
    }

    if (!CloseHandle(hEvent)) {
        std::cerr << "Failed to close event handle. Error: " << GetLastError() << std::endl;
    }

    return 0;
}
```
Start with compiling this code and run it. Ask yourself the question. Why are these messages printed in this order and what is the programming logic behind it?

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/f0fe2f22-b940-41e8-a3b9-1114f4dabd65)


This code has two functions declared as **`WaitForEvent`** and **`SignalEvent`** that will be run as separate threads. Let's start with understanding why the following line is printed first to the console:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/3bb29056-1f6e-4540-88da-303e34890f48)


Let's review the code snippet of the **`WaitForEvent`** function. The thread running this code will **block** or **pause** at the **`WaitForSingleObject(hEvent, INFINITE)`** line, and will not proceed until the event object **hEvent** is signaled from somewhere else in the program (in this case, from the **SignalEvent** function).

The **WaitForSingleObject** function in this context is being used to wait for an event to become signaled. The parameters it takes are a handle to the event object (**hEvent**), and the amount of time the function is to wait for the event to become signaled

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/24e793dd-8244-408e-a55e-a37773e25d82)


In other words, the function is requesting that the thread should wait until the event object **hEvent** is signaled. While the **`WaitForEvent`** thread is waiting, the **`SignalEvent`** thread runs and calls **`SetEvent(hEvent);`**. 

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/822dc409-d936-4741-9639-f3117855552a)


This signals the event object **hEvent** to "signaled". This means that the condition for the **`WaitForEvent`** function that was waiting has been met, so the **`WaitForEvent`** function can now proceed past the **WaitForSingleObject** call.

So why is this line printed first again?

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/62bc254d-6802-4cb5-8df0-cad7e4da0893)


By calling **`SetEvent(hEvent)`** it releases the **`WaitForSingleObject(hEvent, INFINITE)`** call in **`WaitForEvent`** that was previously blocked, waiting for the event object to become signaled. 

Once **hEvent** is signaled, the **WaitForSingleObject** function stops blocking, and the next line is executed: 

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/fa5006fe-fab3-4982-a39a-64743f56e6ce)


Therefore, **`"SignalEvent: Signaling the event\n"`** is printed first because **`SignalEvent`** signals the event, which then releases the blocking **WaitForSingleObject** call in **`WaitForEvent`**, allowing it to print.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/7522b2c9-95f0-410d-8ac4-598b2b8b847c)


# Code Sample (2) - Auto-Reset Event using Signaled State

A signaled state indicates that the event object's condition has been met, unblocking any threads waiting on this event to proceed with their execution. In this example, we changed the initial state of the event object during its creation. We made the event object initially **signaled** by passing **TRUE** to the third parameter of **CreateEvent()**.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/741a9649-1876-4b3b-8bb7-76c5052d49f4)


```c
#include <windows.h>
#include <iostream>

// This function waits for an event to be signaled
DWORD WINAPI WaitForEvent(LPVOID lpParam) {
    HANDLE hEvent = *(HANDLE*)lpParam;

    WaitForSingleObject(hEvent, INFINITE);
    std::cout << "WaitForEvent: The event was signaled\n";

    WaitForSingleObject(hEvent, INFINITE);
    std::cout << "WaitForEvent: The event was signaled again\n";

    return 0;
}

// This function signals an event
DWORD WINAPI SignalEvent(LPVOID lpParam) {
    HANDLE hEvent = *(HANDLE*)lpParam;

    Sleep(1000);  // Sleep for 1 second
    std::cout << "SignalEvent: Signaling the event\n";
    SetEvent(hEvent);

    Sleep(5000);  // Sleep for 5 seconds
    std::cout << "SignalEvent: Signaling the event again\n";
    SetEvent(hEvent);

    return 0;
}

int main() {
    DWORD threadID;
    HANDLE hThreads[2];

    // Create an automatic reset event
    // Parameters for CreateEvent function:
    // 1. NULL - Security attributes, if NULL the handle cannot be inherited by child processes
    // 2. FALSE - Manual-reset flag; if FALSE, the function creates an auto-reset event object
    // 3. TRUE - Initial state; if TRUE, the initial state is signaled
    // 4. NULL - Name of the event object; if NULL, the event is unnamed
    HANDLE hEvent = CreateEvent(NULL, FALSE, TRUE, NULL);
    if (hEvent == NULL) {
        std::cerr << "Failed to create event. Error: " << GetLastError() << std::endl;
        return 1;
    }

    hThreads[0] = CreateThread(NULL, 0, WaitForEvent, &hEvent, 0, &threadID);
    if (hThreads[0] == NULL) {
        std::cerr << "Failed to create WaitForEvent thread. Error: " << GetLastError() << std::endl;
        return 1;
    }

    hThreads[1] = CreateThread(NULL, 0, SignalEvent, &hEvent, 0, &threadID);
    if (hThreads[1] == NULL) {
        std::cerr << "Failed to create SignalEvent thread. Error: " << GetLastError() << std::endl;
        return 1;
    }

    WaitForMultipleObjects(2, hThreads, TRUE, INFINITE);

    for (int i = 0; i < 2; i++) {
        if (!CloseHandle(hThreads[i])) {
            std::cerr << "CloseHandle failed for thread " << i << ". Error: " << GetLastError() << std::endl;
        }
    }

    if (!CloseHandle(hEvent)) {
        std::cerr << "Failed to close event handle. Error: " << GetLastError() << std::endl;
    }

    return 0;
}
```
The output will now look like this:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/ecc0f3ce-e065-4962-a3c3-118cb36069eb)


When **`WaitForEvent`** first calls **`WaitForSingleObject(hEvent, INFINITE);`**, the event is in the signaled state, and so **WaitForSingleObject** does not block and continues. After this, the system automatically resets the event to the **non-signaled** state because this is an **auto-reset** event.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/07818d74-b429-4ac4-a190-4142df28cbec)


This means that if **WaitForSingleObject** is called again with this event object (like it is later in the **WaitForEvent** function), it will block until the event object is set to the **signaled** state again.

Now, the **`WaitForEvent`** thread attempts to call **WaitForSingleObject** for the second time. However, because the event object is in a **non-signaled** state, this call will block, and the **`WaitForEvent`** thread will wait until the event is **signaled** again.

At this point, the **`SignalEvent`** thread resumes execution after its initial Sleep call. It prints out **`"SignalEvent: Signaling the event"`**, then calls **SetEvent**, which sets the event object to the **signaled** state.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/8d0c3ac9-fb41-4c5c-b0cd-02f0a09b3010)


This action allows the **WaitForSingleObject** in the **`WaitForEvent`** thread to complete, and the message **`"WaitForEvent: The event was signaled again"`** is printed.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/95a75354-88d0-4128-bde4-0610520b8810)


The process then repeats for the second **signaling** event. The **`SignalEvent`** thread pauses for 5 seconds due to the second Sleep call, then prints **`"SignalEvent: Signaling the event again"`** and calls **SetEvent** again, allowing the event object to return to the **signaled** state. 

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/6d227747-563f-44de-a29d-53c94ff3592a)


This is how the output once again looks like:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/8696f368-01b9-4610-bceb-cbe366afe9a2)


# Code Sample (3) - Manuel-Reset Event using Signaled State

A manual reset event is a synchronization mechanism that, once signaled, stays in the signaled state until it's manually reset to the non-signaled state.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/ae533071-fec6-4bba-845d-02ac0d999f53)


```c
#include <windows.h>
#include <iostream>

// This function waits for an event to be signaled
DWORD WINAPI WaitForEvent(LPVOID lpParam) {
    HANDLE hEvent = *(HANDLE*)lpParam;

    WaitForSingleObject(hEvent, INFINITE);
    std::cout << "WaitForEvent: The event was signaled\n";

    WaitForSingleObject(hEvent, INFINITE);
    std::cout << "WaitForEvent: The event was signaled again\n";

    return 0;
}

// This function signals an event
DWORD WINAPI SignalEvent(LPVOID lpParam) {
    HANDLE hEvent = *(HANDLE*)lpParam;

    Sleep(1000);  // Sleep for 1 second
    std::cout << "SignalEvent: Signaling the event\n";
    SetEvent(hEvent);

    Sleep(5000);  // Sleep for 5 seconds
    std::cout << "SignalEvent: Signaling the event again\n";
    SetEvent(hEvent);

    return 0;
}

int main() {
    DWORD threadID;
    HANDLE hThreads[2];

    // Create a manual reset event
    // Parameters for CreateEvent function:
    // 1. NULL - Security attributes, if NULL the handle cannot be inherited by child processes
    // 2. TRUE - Manual-reset flag; if TRUE, the function creates a manual-reset event object
    // 3. TRUE - Initial state; if TRUE, the initial state is signaled
    // 4. NULL - Name of the event object; if NULL, the event is unnamed
    HANDLE hEvent = CreateEvent(NULL, TRUE, TRUE, NULL);
    if (hEvent == NULL) {
        std::cerr << "Failed to create event. Error: " << GetLastError() << std::endl;
        return 1;
    }

    hThreads[0] = CreateThread(NULL, 0, WaitForEvent, &hEvent, 0, &threadID);
    if (hThreads[0] == NULL) {
        std::cerr << "Failed to create WaitForEvent thread. Error: " << GetLastError() << std::endl;
        return 1;
    }

    hThreads[1] = CreateThread(NULL, 0, SignalEvent, &hEvent, 0, &threadID);
    if (hThreads[1] == NULL) {
        std::cerr << "Failed to create SignalEvent thread. Error: " << GetLastError() << std::endl;
        return 1;
    }

    WaitForMultipleObjects(2, hThreads, TRUE, INFINITE);

    for (int i = 0; i < 2; i++) {
        if (!CloseHandle(hThreads[i])) {
            std::cerr << "CloseHandle failed for thread " << i << ". Error: " << GetLastError() << std::endl;
        }
    }

    if (!CloseHandle(hEvent)) {
        std::cerr << "Failed to close event handle. Error: " << GetLastError() << std::endl;
    }

    return 0;
}
```

In the case of a **manual-reset** event, once the event is signaled (which we did in the **CreateEvent** call by setting the third parameter to **TRUE**), it remains in the **signaled** state until it is explicitly reset back to the **non-signaled** state. 

This means that any number of calls to **WaitForSingleObject** will return immediately because the event is already in the signaled state. The primary change is that in the manual-reset version of the code, the **`WaitForEvent`** thread doesn't block at all, and completes its execution before the **`SignalEvent`** thread even gets a chance to run.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/27605a5b-ca62-4ad7-8ac2-98712f7c0a51)


# Code Sample (4) - ResetEvent

Let's quote the following sentence:

*In the case of a **manual-reset** event, once the event is signaled (which we did in the **CreateEvent** call by setting the third parameter to **TRUE**), it remains in the **signaled** state until it is **explicitly reset back** to the **non-signaled** state.* 

What happens when we explicitly reset back by calling the **ResetEvent** API?

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/c24d256b-fdf6-4bd5-914c-bcaa3bcb956b)



```c
#include <windows.h>
#include <iostream>

// This function waits for an event to be signaled
DWORD WINAPI WaitForEvent(LPVOID lpParam) {
    HANDLE hEvent = *(HANDLE*)lpParam;

    WaitForSingleObject(hEvent, INFINITE);
    std::cout << "WaitForEvent: The event was signaled\n";
    ResetEvent(hEvent);

    WaitForSingleObject(hEvent, INFINITE);
    std::cout << "WaitForEvent: The event was signaled again\n";
    ResetEvent(hEvent);

    return 0;
}

// This function signals an event
DWORD WINAPI SignalEvent(LPVOID lpParam) {
    HANDLE hEvent = *(HANDLE*)lpParam;

    Sleep(1000);  // Sleep for 1 second
    std::cout << "SignalEvent: Signaling the event\n";
    SetEvent(hEvent);

    Sleep(5000);  // Sleep for 5 seconds
    std::cout << "SignalEvent: Signaling the event again\n";
    SetEvent(hEvent);

    return 0;
}

int main() {
    DWORD threadID;
    HANDLE hThreads[2];

    // Create a manual reset event in signaled state
    HANDLE hEvent = CreateEvent(NULL, TRUE, TRUE, NULL);
    if (hEvent == NULL) {
        std::cerr << "Failed to create event. Error: " << GetLastError() << std::endl;
        return 1;
    }

    hThreads[0] = CreateThread(NULL, 0, WaitForEvent, &hEvent, 0, &threadID);
    if (hThreads[0] == NULL) {
        std::cerr << "Failed to create WaitForEvent thread. Error: " << GetLastError() << std::endl;
        return 1;
    }

    hThreads[1] = CreateThread(NULL, 0, SignalEvent, &hEvent, 0, &threadID);
    if (hThreads[1] == NULL) {
        std::cerr << "Failed to create SignalEvent thread. Error: " << GetLastError() << std::endl;
        return 1;
    }

    WaitForMultipleObjects(2, hThreads, TRUE, INFINITE);

    for (int i = 0; i < 2; i++) {
        if (!CloseHandle(hThreads[i])) {
            std::cerr << "CloseHandle failed for thread " << i << ". Error: " << GetLastError() << std::endl;
        }
    }

    if (!CloseHandle(hEvent)) {
        std::cerr << "Failed to close event handle. Error: " << GetLastError() << std::endl;
    }

    return 0;
}
```
When we call **`ResetEvent(hEvent);`**, it changes the state of the event object to **non-signaled**. This means that after the event is signaled and the **`WaitForSingleObject(hEvent, INFINITE);`** line is passed, we reset the event to **non-signaled** state.

The output will now look like this:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/bc20444c-aee1-4d3b-bcfa-4a4c3fa0035f)


The first **WaitForSingleObject** call does not block because the event is initially in the **signaled** state. However, the second **WaitForSingleObject** call will block because **ResetEvent** had been called to reset the event to a **non-signaled** state. It will continue only after **SetEvent** is called in the **`SignalEvent`** function.

# Final Example - Auto-Reset Event with Non-Signaled State

This code creates 3 threads to perform different tasks. The tasks include creating and writing to files, moving the created files to a new directory, and enumerating all the running processes on the machine. To manage the synchronization between these threads, the code uses an event object, which is a kernel object that can be used to signal when a particular condition has occurred.

The event object **hEvent** is created as an auto-reset event in a non-signaled state:

```c
#include <windows.h>
#include <iostream>
#include <string>
#include <random>
#include <tlhelp32.h>

#define NUM_FILES 1000

HANDLE hEvent;  // Event for signaling file creation completion

std::vector<std::wstring> createdFiles;  // Stores the names of the created files

// This function creates files and writes to them
DWORD WINAPI CreateAndWriteFiles(LPVOID lpParam) {
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

        // If the file handle is invalid, print an error message and continue
        if (hFile == INVALID_HANDLE_VALUE) {
            std::wcerr << L"CreateFile failed for " << fileName << L". Error: " << GetLastError() << std::endl;
            continue;
        }

        // Write "Hello, World!\n" to the file 100 times
        for (int j = 0; j < 100; j++) {
            DWORD bytesWritten;
            std::string data = "Hello, World!\n";
            if (!WriteFile(
                hFile,
                data.c_str(),
                static_cast<DWORD>(data.size()),  // Cast size_t to DWORD
                &bytesWritten,
                NULL
            )) {
                std::wcerr << L"WriteFile failed for " << fileName << L". Error: " << GetLastError() << std::endl;
                break;
            }
        }

        // Close the file handle
        if (!CloseHandle(hFile)) {
            std::wcerr << L"CloseHandle failed for " << fileName << L". Error: " << GetLastError() << std::endl;
        }

        // Add the file name to the createdFiles vector
        createdFiles.push_back(fileName);

        std::wcout << L"Created and wrote to file: " << fileName << std::endl;
    }

    // Signal that all files are created
    SetEvent(hEvent);

    return 0;
}

// This function moves the created files to a new directory
DWORD WINAPI MoveFilesToNewDirectory(LPVOID lpParam) {
    // Wait for all files to be created
    WaitForSingleObject(hEvent, INFINITE);

    // For each file that was created, move it to the new directory
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

// This function enumerates all the running processes on the machine 10 times
DWORD WINAPI EnumerateProcesses(LPVOID lpParam) {
    for (int j = 0; j < 10; j++) {
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
    DWORD threadID;  // Variable to store the thread identifier
    HANDLE hThreads[3];  // Array to store the handles of the threads

    // Create an automatic reset event
    // Parameters for CreateEvent function:
    // 1. NULL - Security attributes, if NULL the handle cannot be inherited by child processes
    // 2. FALSE - Manual-reset flag; if FALSE, the function creates an auto-reset event object, which system automatically resets the event state to non-signaled after a single waiting thread has been released
    // 3. FALSE - Initial state; if FALSE, the initial state is non-signaled
    // 4. NULL - Name of the event object; if NULL, the event is unnamed
    hEvent = CreateEvent(NULL, FALSE, FALSE, NULL);
    // If the event creation fails, print an error message and exit the program
    if (hEvent == NULL) {
        std::cerr << "Failed to create event(s). Error: " << GetLastError() << std::endl;
        return 1;
    }

    // Create the 'CreateAndWriteFiles' thread
    hThreads[0] = CreateThread(NULL, 0, CreateAndWriteFiles, NULL, 0, &threadID);
    // If the thread creation fails, print an error message and exit the program
    if (hThreads[0] == NULL) {
        std::cerr << "Failed to create CreateAndWriteFiles thread. Error: " << GetLastError() << std::endl;
        return 1;
    }

    // Create the 'MoveFilesToNewDirectory' thread
    hThreads[1] = CreateThread(NULL, 0, MoveFilesToNewDirectory, NULL, 0, &threadID);
    // If the thread creation fails, print an error message and exit the program
    if (hThreads[1] == NULL) {
        std::cerr << "Failed to create MoveFilesToNewDirectory thread. Error: " << GetLastError() << std::endl;
        return 1;
    }

    // Create the 'EnumerateProcesses' thread
    hThreads[2] = CreateThread(NULL, 0, EnumerateProcesses, NULL, 0, &threadID);
    // If the thread creation fails, print an error message and exit the program
    if (hThreads[2] == NULL) {
        std::cerr << "Failed to create EnumerateProcesses thread. Error: " << GetLastError() << std::endl;
        return 1;
    }

    // Wait for all threads to finish execution
    WaitForMultipleObjects(3, hThreads, TRUE, INFINITE);

    // Loop through 'hThreads' and close all the thread handles
    for (int i = 0; i < 3; i++) {
        if (!CloseHandle(hThreads[i])) {
            std::cerr << "CloseHandle failed for thread " << i << ". Error: " << GetLastError() << std::endl;
        }
    }

    // Close the event handle
    if (!CloseHandle(hEvent)) {
        std::cerr << "Failed to close event handle(s). Error: " << GetLastError() << std::endl;
    }

    return 0;
}
```

**`CreateAndWriteFiles`** function creates and writes to files, then signals the event by calling **`SetEvent(hEvent)`**. This changes the event state to **signaled**.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/87a23a2c-2998-42f8-829b-26e588574e7d)


**`MoveFilesToNewDirectory`** waits for the event to be signaled by calling **`WaitForSingleObject(hEvent, INFINITE)`**. This ensures that the **`MoveFilesToNewDirectory`** thread does not start moving files until all files are created, and it won't start moving files again until **hEvent** is set to the **signaled** state again.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/49c46c8a-0738-4c99-b833-a3326492a118)


The primary purpose of the Event object in this program is to synchronize the execution of the file creation/writing thread and the file moving thread, ensuring that files are not attempted to be moved until they have been fully created and written to. 

# Bonus Section - Event Objects with Inter-Process Synchronization

Inter-process synchronization with events is a technique where different processes use event objects to coordinate their operations, typically by having one process signal an event when it reaches a certain state, and other processes wait for that event to be signaled before proceeding with their own operations.

1. Start compiling the following code as **EventObject.exe**:

```c
#include <windows.h>
#include <iostream>
#include <string>
#include <random>
#include <tlhelp32.h>

#define NUM_FILES 1000

HANDLE hFilesCreatedEvent, hFilesMovedEvent;  // Events for signaling file creation and moving completion

std::vector<std::wstring> createdFiles;  // Stores the names of the created files

// This function creates files and writes to them
DWORD WINAPI CreateAndWriteFiles(LPVOID lpParam) {
    std::random_device rd;
    std::mt19937 gen(rd());

    // Create and write to NUM_FILES number of files
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

    // Signal that all files are created
    SetEvent(hFilesCreatedEvent);

    return 0;
}

// This function moves the created files to a new directory
DWORD WINAPI MoveFilesToNewDirectory(LPVOID lpParam) {
    // Wait for all files to be created
    WaitForSingleObject(hFilesCreatedEvent, INFINITE);

    // For each file that was created, move it to the new directory
    for (const auto& source : createdFiles) {
        std::wstring destination = L"C:\\Temp2\\" + source.substr(source.find_last_of(L"\\") + 1);
        if (!MoveFile(source.c_str(), destination.c_str())) {
            std::wcerr << L"MoveFile failed for " << source << L" to " << destination << L". Error: " << GetLastError() << std::endl;
        }
        else {
            std::wcout << L"Moved file from " << source << L" to " << destination << std::endl;
        }
    }

    // Signal that all files are moved
    SetEvent(hFilesMovedEvent);

    return 0;
}

int main() {
    DWORD threadID;  // Variable to store the thread identifier
    HANDLE hThreads[2];  // Array to store the handles of the threads

    // Create two separate events
    hFilesCreatedEvent = CreateEvent(NULL, TRUE, FALSE, L"Global\\FilesCreatedEvent");
    hFilesMovedEvent = CreateEvent(NULL, TRUE, FALSE, L"Global\\FilesMovedEvent");
    if (hFilesCreatedEvent == NULL || hFilesMovedEvent == NULL) {
        std::cerr << "Failed to create event(s). Error: " << GetLastError() << std::endl;
        return 1;
    }

    // Create the 'CreateAndWriteFiles' thread
    hThreads[0] = CreateThread(NULL, 0, CreateAndWriteFiles, NULL, 0, &threadID);
    if (hThreads[0] == NULL) {
        std::cerr << "Failed to create CreateAndWriteFiles thread. Error: " << GetLastError() << std::endl;
        return 1;
    }

    // Create the 'MoveFilesToNewDirectory' thread
    hThreads[1] = CreateThread(NULL, 0, MoveFilesToNewDirectory, NULL, 0, &threadID);
    if (hThreads[1] == NULL) {
        std::cerr << "Failed to create MoveFilesToNewDirectory thread. Error: " << GetLastError() << std::endl;
        return 1;
    }

    // Wait for all threads to finish execution
    WaitForMultipleObjects(2, hThreads, TRUE, INFINITE);

    // Loop through 'hThreads' and close all the thread handles
    for (int i = 0; i < 2; i++) {
        if (!CloseHandle(hThreads[i])) {
            std::cerr << "CloseHandle failed for thread " << i << ". Error: " << GetLastError() << std::endl;
        }
    }

    // Close the event handles
    if (!CloseHandle(hFilesCreatedEvent) || !CloseHandle(hFilesMovedEvent)) {
        std::cerr << "Failed to close event handle(s). Error: " << GetLastError() << std::endl;
    }

    return 0;
}
```

This code creates and writes to a number of files in one thread and moves them to a new directory in another thread. The execution of these threads is synchronized using two event objects: **`hFilesCreatedEvent`** and **`hFilesMovedEvent`**.

- **`hFilesCreatedEvent`** is signaled once all files have been created and written to. This event is used to synchronize the **`CreateAndWriteFiles`** and **`MoveFilesToNewDirectory`** threads. The **`MoveFilesToNewDirectory`** thread waits for this event to be signaled before it starts moving the files.
- **`hFilesMovedEvent`** is signaled once all files have been moved to the new directory. This event can be used by other threads or processes to know when all the files have been moved.

The reason why **manual-reset** events are used here is to separate the completion of two distinct stages of the process: the creation of the files and the moving of the files. This allows other threads or processes to synchronize their operations based on these two stages.

- Start compiling the second code as **EventObject2.exe**

```c
#include <windows.h>
#include <iostream>
#include <string>
#include <vector>

#define NUM_FILES 1000

// Declare hEvent as a global variable
HANDLE hEvent;

std::vector<std::wstring> createdFiles;  // Stores the names of the created files

// This function writes "Goodbye, World!" to the created files
DWORD WINAPI WriteToFiles(LPVOID lpParam) {
    DWORD threadId = GetCurrentThreadId();

    for (const auto& fileName : createdFiles) {
        HANDLE hFile = CreateFile(
            fileName.c_str(),
            FILE_APPEND_DATA,
            0,
            NULL,
            OPEN_EXISTING,
            FILE_ATTRIBUTE_NORMAL,
            NULL
        );

        if (hFile == INVALID_HANDLE_VALUE) {
            std::wcerr << L"CreateFile failed for " << fileName << L". Error: " << GetLastError() << std::endl;
            continue;
        }

        for (int j = 0; j < 100; j++) {
            DWORD bytesWritten;
            std::string data = "Goodbye, World!\n";
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

        std::wcout << L"Thread ID " << threadId << " has written to the file: " << fileName << std::endl;
    }

    return 0;
}

int main() {
    DWORD threadID;

    // Try to open the event object in a loop
    while (true) {
        hEvent = OpenEvent(EVENT_ALL_ACCESS, FALSE, L"Global\\FilesMovedEvent");
        if (hEvent != NULL) {
            break;  // If the event object was opened successfully, break the loop
        }
        Sleep(1000);  // Wait for 1 second before trying again
    }

    // Wait for all files to be moved
    WaitForSingleObject(hEvent, INFINITE);

    WIN32_FIND_DATA findFileData;
    HANDLE hFind = FindFirstFile(L"C:\\Temp2\\*", &findFileData);
    if (hFind == INVALID_HANDLE_VALUE) {
        std::cerr << "Failed to find files in the directory. Error: " << GetLastError() << std::endl;
        return 1;
    }
    do {
        if (!(findFileData.dwFileAttributes & FILE_ATTRIBUTE_DIRECTORY)) {
            createdFiles.push_back(L"C:\\Temp2\\" + std::wstring(findFileData.cFileName));
        }
    } while (FindNextFile(hFind, &findFileData) != 0);
    FindClose(hFind);

    HANDLE hThread = CreateThread(NULL, 0, WriteToFiles, NULL, 0, &threadID);
    if (hThread == NULL) {
        std::cerr << "Failed to create WriteToFiles thread. Error: " << GetLastError() << std::endl;
        return 1;
    }

    WaitForSingleObject(hThread, INFINITE);

    CloseHandle(hThread);
    CloseHandle(hEvent);

    return 0;
}
```

The event object is used for inter-process synchronization: the primary process creates and signals the event when it has finished moving the files, and this process waits for the event to be signaled before it starts processing the files.

This block of code is attempting to open an existing event object named **`Global\\FilesMovedEvent`** using the **OpenEvent** function. The **`WaitForSingleObject(hEvent, INFINITE)`** line then blocks execution until the **`FilesMovedEvent`** is signaled by the other process. This is the synchronization point where this process is waiting for the other process to finish moving all the files. Once the event is signaled, this process knows that all files have been moved and it can proceed with its operations, which in this case is writing to the files.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/811f05d3-9b3e-4ec2-a8bf-39a61e9e0bbf)


# Theory vs Practice - Event Objects with Inter-Process Synchronization

1. Start running **EventObject.exe**

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/d3469586-ba73-452c-85d5-f309b34dbc9b)


2. At the same time, start **EventObject2.exe**

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/9797e1af-c89a-4092-8c0a-48bbe141fcde)


3. Pay a close attention to **EventObject2.exe**, the program won't run right away. As discussed previously, the **`WaitForSingleObject(hEvent, INFINITE)`** line blocks execution until the **`FilesMovedEvent`** is signaled by the other process.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/ab319af8-7364-477f-b728-b2e698abe678)


4. **EventObject.exe** has completed all the Write/Move operations

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/d29bc9ee-e37d-428c-8c58-bb47ca25db60)


5. Now the event is signaled, **EventObject2.exe** knows that all files have been moved and it can proceed with its operations, which in this case is writing **"Goodbye, World!"** to the files in **C:\Temp2**.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/9ac1e4e3-af64-4048-85fb-6cb37c67fc80)


6. Here is a snippet of a created file with the text that has been written to it by both processes.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/d426f420-28eb-44bf-98c0-90315cf22dfa)


# Process Explorer - View Event Objects of running Processes

This section will be all about using **Process Explorer** to view the **Event** objects of all the running processes. This can be achieved by clicking on a process and then go to the 'Handles' tab.

Before we are doing this, let's quickly review the code of **EventObject.exe** again. As we can see, there are two event objects created. **`Global\FilesCreatedEvent`** is manual-reset event object that is signaled when all the files have been created and written to. **`Global\FilesMovedEvent`** is a manual-reset event object that is signaled when all the files have been moved to a new directory.

Threads or processes can wait for these events to know when they can safely start their operations that depend on the completion.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/227ffc23-3505-46b5-8b94-52aa447c8ee6)


Start running **EventObject.exe** and use **Process Explorer** to see if we can find the two **event** objects:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/92596eae-02a4-4a7a-a8e8-86b93549973d)

