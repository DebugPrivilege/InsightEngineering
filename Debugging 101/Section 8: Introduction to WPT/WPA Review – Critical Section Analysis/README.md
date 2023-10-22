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

# Code Sample - Deadlock

This sample code demonstrates a classic example of a deadlock in a multithreaded program. The deadlock occurs because the two threads are trying to acquire two locks (Critical Sections) but in opposite order.

Start compiling the following code:

```c
#include <cstdio>
#include <Windows.h>

// Initialize critical sections
CRITICAL_SECTION zoneA, zoneB;

// Function for the additional thread
DWORD WINAPI ExtraThreadFunction(LPVOID lpParam);

int main()
{
    HANDLE extraThread;
    DWORD extraThreadId;

    wprintf(L"Current Process ID: %u\n", GetCurrentProcessId());
    wprintf(L"Current Thread ID: %u\n\n", GetCurrentThreadId());

    // Initialize critical sections
    InitializeCriticalSection(&zoneA);
    InitializeCriticalSection(&zoneB);

    // Create the secondary thread
    extraThread = CreateThread(NULL, 0, ExtraThreadFunction, NULL, 0, &extraThreadId);

    if (extraThread)
    {
        wprintf(L"New Thread Created, ID: %u\n", extraThreadId);

        if (CloseHandle(extraThread))
        {
            wprintf(L"Successfully closed thread handle %u\n", extraThreadId);
        }
        else
        {
            wprintf(L"Failed to close thread handle, Error: %u\n", GetLastError());
        }
        wprintf(L"\n");
    }
    else
    {
        wprintf(L"Failed to create new thread, Error: %u\n", GetLastError());
        return 0;
    }

    wprintf(L"========== Initiating Deadlock Test ==========\n");

    // Main loop (This is where deadlock occurs)
    while (true)
    {
        EnterCriticalSection(&zoneA);
        wprintf(L"Main Thread (%u) locked zoneA.\n", GetCurrentThreadId());

        EnterCriticalSection(&zoneB);
        wprintf(L"Main Thread (%u) locked both zones!\n", GetCurrentThreadId());

        LeaveCriticalSection(&zoneB);
        wprintf(L"Main Thread (%u) unlocked zoneB.\n", GetCurrentThreadId());

        LeaveCriticalSection(&zoneA);
        wprintf(L"Main Thread (%u) unlocked both zones.\n", GetCurrentThreadId());

        Sleep(50);
    }

    return 0;
}

DWORD WINAPI ExtraThreadFunction(LPVOID lpParam)
{
    while (true)
    {
        Sleep(50);
        EnterCriticalSection(&zoneB);
        wprintf(L"Extra Thread (%u) locked zoneB.\n", GetCurrentThreadId());

        EnterCriticalSection(&zoneA);
        wprintf(L"Extra Thread (%u) locked both zones!\n", GetCurrentThreadId());

        LeaveCriticalSection(&zoneA);
        wprintf(L"Extra Thread (%u) unlocked zoneA.\n", GetCurrentThreadId());

        LeaveCriticalSection(&zoneB);
        wprintf(L"Extra Thread (%u) unlocked both zones.\n", GetCurrentThreadId());

        Sleep(50);
    }

    return 0;
}
```

The **`main`** thread first locks **`zoneA`** and then attempts to lock **`zoneB`**. On the other side, the **`ExtraThread`** first locks **`zoneB`** and then attempts to lock **`zoneA`**. This creates a situation where each thread holds one lock and waits indefinitely for the other lock to be released, which never happens because each thread is waiting for the other. This leads to a deadlock.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/ac8edce3-191d-4464-b10c-d4666ca88087)

# Start Capturing WPA Trace

1. Open **Windows Performance Recorder** (WPRUI.exe) as an Administrator
2. Ensure that **First level triage** is checked, and then click on **Start**

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/bc1bec8c-a6eb-4635-80cc-0b7f64fbfaca)

3. Run the compiled code and let it run for 30 seconds.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/c58d482f-63db-4d87-bb99-eaa922559e9e)

4. Save the WPA trace file

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/b33ad879-a084-48cb-91a8-07ad3ec28e16)

# WPA - Walk Through

1. Start with ensuring that the symbol path has been configured correctly

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/aea85f69-ac08-46b6-bfd9-3a2278115683)

2. At the left side, expand **Computation** and drag **CPU Usage (Precise)** to the **Analysis** view.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/7f9412c4-ac8a-4f93-b594-8232e45b608d)

3. Right-click on the application named **CriticalSectionApp.exe** and select the "Filter To Selection" option. We can now only see one process, which happens to have 5 threads.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/6943fcd0-a80f-44e3-86ad-fb77fd468722)

What is the identifier of the thread executing the **`main()`** function? To find this, select a cell under the "New Thread Stack" column and hit "CTRL+F" to open the search dialog.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/93723d65-e1d5-424a-b35e-5e7c7f19af20)

Thread ID **8012** happens to be the thread that executes the **`main()`** function.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/4803bc82-c9d9-4650-8e68-03d0e3a5ce5a)

If we start exploring other threads and view it's call stack. We can see that the thread with ID **1520** is in a deadlock situation. The call stack reveals that the thread is stuck at the function **`RtlpWaitOnCriticalSection`**, which means it is waiting to acquire a Critical Section lock.

The thread is unable to proceed because it's waiting for a resource that is held by another thread, which in turn is likely waiting for a resource held by this thread or is in a state that prevents it from releasing the lock.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/efce417f-1456-425d-a58d-48640bc58b51)

Thread ID **8120** was responsible for unblocking thread **1520** by invoking the **`LeaveCriticalSection`** function.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/66d9b186-16a8-45b7-8f22-1fdbd51d8346)

# How to interpret the columns in WPA?

Here are the interesting columns in the CPU Usage (Precise) table.

| Column             | Details                                                                                                      |
|--------------------|--------------------------------------------------------------------------------------------------------------|
| % CPU Usage        | The CPU usage of the new thread after it is switched. Expressed as a percentage of total CPU time.            |
| Count              | The number of context switches represented by the row. Always 1 for individual rows.                          |
| CPU Usage (ms)     | The CPU usage of the new thread after the context switch.                                                     |
| NewProcess         | The process of the new thread.                                                                                |
| NewThreadId        | The thread ID of the new thread.                                                                              |
| NewThreadStack     | The stack of the new thread when switched in. Usually indicates what the thread was blocked or waiting on.    |
| Ready(s)           | Time spent in the Ready queue due to pre-emption or CPU starvation.                                           |
| ReadyingThreadId   | The thread ID of the readying thread.                                                                         |
| ReadyingProcess    | The process that owns the readying thread.                                                                    |
| ReadyThreadStack   | The stack of the readying thread.                                                                             |
| ReadyTime (s)      | The time when the new thread was readied.                                                                     |
| SwitchInTime(s)    | The time when the new thread was switched in.                                                                 |
| Waits (s)          | Amount of time a thread waited on a logical or physical resource. Ends when NewThreadId is signaled by ReadyingThreadId. |

In WPA, the **New Thread Stack** column indicates threads that are currently in a waiting state. For instance, in this example, thread ID **6768** is in a waiting state to obtain a Critical Section lock.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/9d04018e-0eb2-4130-8e03-3ebaf01fbbf6)

The **Ready Thread Stack** column in WPA represents the thread responsible for freeing up the waiting thread. For example, in this case, thread ID **6640** clears the wait condition for thread ID **6768** by calling the **`LeaveCriticalSection`** function.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/74435224-1e28-44df-ac51-c11d224f31f4)

From the console output, it's evident that thread ID **6640** removes the wait condition for thread ID **6768**, allowing it to execute. However, as soon as thread ID **6768** starts running, the program gets stuck specifically because this thread enters into a deadlock.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/ded0a567-b9cc-43aa-98d9-b0b1874be568)










