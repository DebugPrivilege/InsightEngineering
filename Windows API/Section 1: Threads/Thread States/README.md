# What are Thread States?

A thread state refers to the current condition of a thread in a process, indicating its activity or readiness level. It can be one of several values, such as **Ready**, **Running**, **Waiting**, **Standby**, **Transition**, or **Terminated**, each reflecting a different stage in the thread's lifecycle.

| Thread State | Description |
|--------------|-------------|
| Ready | The thread is prepared to run and is waiting for an available processor. The system's scheduler determines which ready threads get to use the processor based on their priority levels. |
| Standby | The thread is selected to run on a processor the next time the processor becomes available. This is the thread that the system will run next on that processor. |
| Running | The thread is currently executing instructions on a processor. |
| Waiting | The thread is not running, and is waiting for some resource to become available or for some event to occur. For example, a thread might be waiting for disk I/O to complete, for data from a network connection, or for a mutex to be released by another thread. This is the most common one that you will see. |
| Transition | The thread is waiting for resources to become available so that it can transition to the "Ready" state and then to the "Running" state. For example, it might be waiting for a page to be read from disk into memory. |
| Terminated | The thread has finished executing and has exited. A thread in this state is effectively dead and cannot be scheduled to run again. |
| Initialized | A state that indicates the thread has been initialized, but has not yet started. |

# How to view Thread States of a Process?

Open **Process Explorer** and search for a process of interest. Further, we can then click on 'Threads' and view all the threads and the call stack of a process. Here are a few examples that we can see within **Process Explorer**.

- **Running**

The thread is currently executing instructions on a processor. 

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/98524945-bd31-4c0c-bb57-54e5ab749f84)

- **Ready**

The thread is prepared to run and is waiting for an available processor. The system's scheduler determines which ready threads get to use the processor based on their priority levels.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/17f537fb-1e01-4ec4-ba6c-41cf1e4c993d)

- **Waiting**

The thread is not running, and is waiting for some resource to become available or for some event to occur. For example, a thread might be waiting for disk I/O to complete, for data from a network connection, or for a mutex to be released by another thread. This is the most common one that you will see.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/4ba34eff-22f2-40df-bd58-cbc7dd6cd2ec)

- **Terminated**

The thread has finished executing and has exited.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/c2f30a67-f1f8-4ffb-98cd-b53362546aa9)

