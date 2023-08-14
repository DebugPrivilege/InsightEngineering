# What are Thread Wait Reasons?

A thread wait reason is a status that indicates why a thread is not currently executing. Here are the common one's that I often see within **Process Explorer**.

| Wait Reason | Description |
| ----------- | ----------- |
| DelayExecution | The thread is in a "sleep" mode, possibly because of a call to `Sleep`, `SleepEx`, or `KeDelayExecutionThread` in the kernel. |
| Executive | This is a generic placeholder. It means some kernel component (like a driver) asked the thread to wait. |
| WrQueue | The thread is waiting for some incoming work, usually from an I/O operation. |
| Suspended | The thread is on hold. It might be because it decided to hit the suspend itself, or another thread told it to. |
| UserRequest | This is the wait reason you'll see the most. It's often because the program asked the thread to wait, but it could also be because a kernel component (like a driver) made the request. |
| WrAlertByThreadId | This happens when a thread is waiting for things like Critical Sections, Conditional Variables, or Slim Reader/Writer locks. |
| WrLpcReceive | The thread is waiting for an incoming Local Procedure Call (LPC) message. LPC is a communication method used by different processes on the same machine to share data. |
| WrLpcReply | The thread is waiting for reply to a local procedure call to arrive. |
| WrPreempted | This refers to a thread that was paused on a CPU because a more important thread needed to use that CPU. |
| WrFreePage | The thread is waiting for a free virtual memory page. |
| WrVirtualMemory | The thread is waiting for the system to allocate virtual memory. |

# How to view Thread Wait Reasons in Process Explorer?

Here is an example in **Process Explorer:**

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/f264e36f-7554-4764-a99f-9ec1bd9fb606)

