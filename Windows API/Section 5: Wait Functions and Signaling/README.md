# What are Wait Functions?

Wait functions let a thread pause until certain conditions are met, often related to synchronization objects like mutexes or events. When used, these functions check if the conditions, based on the synchronization object's state, are fulfilled before continuing.

The following Windows APIs are used when implementing Wait functions:

| Function Name           | Description                                                                                                                                                                               |
|-------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `WaitForSingleObject()` / `WaitForSingleObjectEx()`     | This function is used to wait until the specified object is in the signaled state or the time-out interval elapses. The object can be a variety of things, such as a process, thread, mutex, semaphore, or event. |
| `WaitForMultipleObjects()` / `WaitForMultipleObjectsEx()` | This function is similar to WaitForSingleObject, but it can wait for one or more objects to become signaled. It can wait for all objects to become signaled (a conjunction or 'and' condition) or for any one object to become signaled (a disjunction or 'or' condition). |

Here are the parameters of the **WaitForSingleObject** function:

```c
DWORD WaitForSingleObject(
  HANDLE hHandle, --> A handle to the object
  DWORD  dwMilliseconds --> The time-out interval in milliseconds.
);
```
The **dwMilliseconds** parameter of the **WaitForSingleObject** and **WaitForMultipleObjects** function specifies the time-out interval, in milliseconds. The function will return when either the specified object is in the signaled state or the time-out interval elapses. The **dwMilliseconds** parameter can take several types of values:

| Value         | Description  | Example |
|---------------|--------------|---------|
| Positive Integer | Specifies a time-out interval in milliseconds. The function will return either when the specified object is in the signaled state or when the interval elapses. If the interval elapses and the object is still not signaled, the function will return `WAIT_TIMEOUT`. | `WaitForSingleObject(hHandle, 5000); // Waits for 5000 milliseconds or 5 seconds` |
| Zero (0)      | This tests the state of the specified object and returns immediately. If the object is signaled, the function will return `WAIT_OBJECT_0`. If the object is not signaled, the function will return `WAIT_TIMEOUT`. | `WaitForSingleObject(hHandle, 0); // Checks the state and returns immediately` |
| `INFINITE`    | Specifies an infinite time-out interval. The function will return only when the specified object is in the signaled state. This means that the calling thread will wait indefinitely until the object is signaled. | `WaitForSingleObject(hHandle, INFINITE); // Waits indefinitely until the object is signaled` |

Here are the parameters of **WaitForMultipleObjects**:

The **WaitForMultipleObjects** function is a part of the Windows API. It allows a thread to block its execution until one or all of the specified objects are in a signaled state, or until the time-out interval expires.

```c
DWORD WaitForMultipleObjects(
  DWORD  nCount, --> The number of object handles in the array pointed to by lpHandles
  const HANDLE* lpHandles, --> An array of object handles.
  BOOL   bWaitAll, --> If this parameter is TRUE, the function returns when the state of all objects in the lpHandles array is signaled. If FALSE, the function returns when the state of any one of the objects is set to signaled. 
  DWORD  dwMilliseconds -->  The time-out interval, in milliseconds.
);
```

The return values from the **WaitForSingleObject** and **WaitForMultipleObjects** function indicate why the function returned and can help to understand the status of the object we're waiting on. These return values provide useful information to a program, allowing it to decide how to proceed based on the state of the object it was waiting for.

| Return Value  | Hexadecimal Value | Description  |
|---------------|-------------------|--------------|
| `WAIT_ABANDONED` | `0x00000080L` | The specified object is a mutex object that was not released by the thread that owned the mutex object before the owning thread terminated. Ownership of the mutex object is granted to the calling thread and the mutex is set to nonsignaled. If the mutex was protecting persistent state information, you should check it for consistency. |
| `WAIT_OBJECT_0` | `0x00000000L` | The state of the specified object is signaled. This means that the object the function was waiting for became signaled within the timeout interval. |
| `WAIT_TIMEOUT` | `0x00000102L` | The time-out interval elapsed, and the object's state is nonsignaled. This means that the object did not become signaled within the timeout interval. |
| `WAIT_FAILED` | `(DWORD)0xFFFFFFFF` | The function has failed. To get extended error information, call `GetLastError()`. This is usually due to a problem with the parameters or the state of the system. |
| `WAIT_IO_COMPLETION` | `0x000000C0L` | The wait operation ended because a user-mode asynchronous procedure call (APC) was queued to the thread. |

# What is the difference again between Signaled State and Non-Signaled State?

- **Signaled State**

An object is in the signaled state when the condition it represents has been met. Here are a few examples:

An **event object** becomes **signaled** when the **SetEvent** function is called on it. For example, a thread might call **SetEvent** when it has completed a task, signaling that the task is done and that any threads waiting on the event can proceed.

A **mutex** object becomes **signaled** when it is released by the owning thread. This is done using the **ReleaseMutex** function. If a thread owns a mutex and it finishes the critical section of code it was protecting, it calls **ReleaseMutex**, which makes the mutex available for other threads and sets it to the signaled state.

A **semaphore** becomes signaled when its count is increased from zero, indicating that resources are available. This is done using the **ReleaseSemaphore** function. 

- **Non-Signaled State**

An object is in the non-signaled state when the condition it represents has not been met. For example, if a thread is still running and hasn't signaled an event object, that event object is in the non-signaled state. 

Wait functions in Windows use these states to manage thread execution. If the object a thread is waiting on is in the signaled state, the wait function will return immediately and the thread will continue execution. If the object is in the non-signaled state, the thread will enter a wait state until the object becomes signaled.

# Code Sample (1) - Wait Return values

In this example, we create an event object **`hEvent`** but we never signal it. We then call **WaitForSingleObject** to wait for this event to become signaled. We've set a timeout of 1000 milliseconds (1 second) for this wait. 

```c
#include <windows.h>
#include <iostream>

int main() {
    // Create an auto-reset event object
    HANDLE hEvent = CreateEvent(NULL, FALSE, FALSE, NULL);
    if (hEvent == NULL) {
        std::cerr << "CreateEvent failed with error: " << GetLastError() << std::endl;
        return 1;
    }

    // Call WaitForSingleObject with a 1000 millisecond timeout
    DWORD waitResult = WaitForSingleObject(hEvent, 1000);
    switch (waitResult) {
    case WAIT_ABANDONED:
        std::cout << "Wait abandoned." << std::endl;
        break;
    case WAIT_OBJECT_0:
        std::cout << "Object signaled." << std::endl;
        break;
    case WAIT_TIMEOUT:
        std::cout << "Wait timed out." << std::endl;
        break;
    case WAIT_FAILED:
        std::cerr << "Wait failed with error: " << GetLastError() << std::endl;
        break;
    default:
        std::cerr << "Unexpected result: " << waitResult << std::endl;
        break;
    }

    // Close the event object
    CloseHandle(hEvent);

    return 0;
}
```

Since we never signal **`hEvent`**, **WaitForSingleObject** waits until the timeout period expires. As a result, it returns **WAIT_TIMEOUT**, indicating that the wait ended due to a timeout. This is why our program outputs "Wait timed out."

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/52a97a03-4f1a-47a5-8578-9d634f35ccd7)


# Code Sample (2) - Wait Return values

In this example, we are creating an auto-reset event object (**`hEvent`**) and then signaling it using **`SetEvent(hEvent)`**.

```c
#include <windows.h>
#include <iostream>

int main() {
    // Create an auto-reset event object
    HANDLE hEvent = CreateEvent(NULL, FALSE, FALSE, NULL);
    if (hEvent == NULL) {
        std::cerr << "CreateEvent failed with error: " << GetLastError() << std::endl;
        return 1;
    }

    // Signal the event
    if (!SetEvent(hEvent)) {
        std::cerr << "SetEvent failed with error: " << GetLastError() << std::endl;
        return 1;
    }

    // Call WaitForSingleObject with a 1000 millisecond timeout
    DWORD waitResult = WaitForSingleObject(hEvent, 1000);
    switch (waitResult) {
    case WAIT_ABANDONED:
        std::cout << "Wait abandoned." << std::endl;
        break;
    case WAIT_OBJECT_0:
        std::cout << "Object signaled." << std::endl;
        break;
    case WAIT_TIMEOUT:
        std::cout << "Wait timed out." << std::endl;
        break;
    case WAIT_FAILED:
        std::cerr << "Wait failed with error: " << GetLastError() << std::endl;
        break;
    default:
        std::cerr << "Unexpected result: " << waitResult << std::endl;
        break;
    }

    // Close the event object
    CloseHandle(hEvent);

    return 0;
}
```

The event object is in the signaled state at the time of the function call. It sees that **`hEvent`** is already signaled and immediately returns **WAIT_OBJECT_0**. This is why our program outputs "Object signaled." 

The **WaitForSingleObject** function found that the event it was waiting for was already signaled. The timeout value of 1000 milliseconds isn't relevant in this specific scenario because the event is signaled before the wait function is even called, causing the function to return immediately.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/72dc0880-4c4e-4609-8ba8-20864511a6cd)

