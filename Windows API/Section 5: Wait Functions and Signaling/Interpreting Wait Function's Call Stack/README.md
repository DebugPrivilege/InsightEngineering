# How to Interpret a Wait Function's Call Stack?

When we see **WaitForSingleObject** in a call stack, it tells us that our thread is currently blocked, waiting for a specific synchronization object to become signaled. This object can be an event, mutex, semaphore, process, thread, or another type of kernel object. 

Every time we encounter **WaitForSingleObject**, it's a clear indicator that we're waiting on a synchronization object to be signaled before proceeding further in our code. Here is an example:

The call stack indicates that a thread in the **Test.exe** program was waiting for an object to become signaled. The waiting was initiated in the **`CreateAndWriteFiles`** function.

```
0:000> !mex.t 1
DbgID ThreadID        User Kernel Create Time (UTC)
1     2b70 (0n11120) 125ms 2s.031 08/14/2023 07:25:44.551 AM

# Child-SP         Return           Call Site                             Source
0 000000d1f73fe988 00007ff872213f8e ntdll!NtWaitForSingleObject+0x14      
1 000000d1f73fe990 00007ff600812cc7 KERNELBASE!WaitForSingleObjectEx+0x8e 
2 000000d1f73fea30 00007ff8744a26ad Test!CreateAndWriteFiles+0x4f7        C:\Users\User\source\repos\Test\Test\Test.cpp @ 62
3 000000d1f73ffee0 00007ff874ceaa68 kernel32!BaseThreadInitThunk+0x1d     
4 000000d1f73fff10 0000000000000000 ntdll!RtlUserThreadStart+0x28
```

From this call stack, we can see that the **`MoveFilesToNewDirectory`** function in the **`Test.exe`** program is currently waiting as well for a synchronization object to be signaled.

```
0:000> !mex.t 2
DbgID ThreadID      User Kernel Create Time (UTC)
2     10f8 (0n4344)    0      0 08/14/2023 07:25:44.551 AM

# Child-SP         Return           Call Site                             Source
0 000000d1f74ffc78 00007ff872213f8e ntdll!NtWaitForSingleObject+0x14      
1 000000d1f74ffc80 00007ff600812d67 KERNELBASE!WaitForSingleObjectEx+0x8e 
2 000000d1f74ffd20 00007ff8744a26ad Test!MoveFilesToNewDirectory+0x47     C:\Users\User\source\repos\Test\Test\Test.cpp @ 75
3 000000d1f74ffed0 00007ff874ceaa68 kernel32!BaseThreadInitThunk+0x1d     
4 000000d1f74fff00 0000000000000000 ntdll!RtlUserThreadStart+0x28
```

From the call stack, we can see that the **`main`** function of the **Test.exe** program is currently in a waiting state, specifically using the **WaitForMultipleObjects** function. This indicates that the **`main`** thread is waiting for multiple events or threads to signal their completion.

```
0:001> !mex.t 0
DbgID ThreadID       User Kernel Create Time (UTC)
0     2abc (0n10940)    0      0 08/14/2023 07:25:44.545 AM

# Child-SP         Return           Call Site                                Source
0 000000d1f72ffb38 00007ff87223f5a9 ntdll!NtWaitForMultipleObjects+0x14      
1 000000d1f72ffb40 00007ff87223f4ae KERNELBASE!WaitForMultipleObjectsEx+0xe9 
2 000000d1f72ffe20 00007ff60081322e KERNELBASE!WaitForMultipleObjects+0xe    
3 000000d1f72ffe60 00007ff60081f3b0 Test!main+0x13e                          C:\Users\User\source\repos\Test\Test\Test.cpp @ 118
4 (Inline)         ---------------- Test!invoke_main+0x22                    D:\a\_work\1\s\src\vctools\crt\vcstartup\src\startup\exe_common.inl @ 78
5 000000d1f72ffec0 00007ff8744a26ad Test!__scrt_common_main_seh+0x10c        D:\a\_work\1\s\src\vctools\crt\vcstartup\src\startup\exe_common.inl @ 288
6 000000d1f72fff00 00007ff874ceaa68 kernel32!BaseThreadInitThunk+0x1d        
7 000000d1f72fff30 0000000000000000 ntdll!RtlUserThreadStart+0x28
```

Given our prior understanding of the situation with **`MoveFilesToNewDirectory`** and **`CreateAndWriteFiles:`**

1. The **`MoveFilesToNewDirectory`** function is awaiting an event to indicate that files have been created before it can proceed.
2. Concurrently, the **`CreateAndWriteFiles`** function might also be waiting for some event to be signaled, or perhaps it hasn't sent the signal indicating that it has completed its task.

Now, if the **`main`** thread (via the **`main`** function) is using **WaitForMultipleObjects** to wait for both the **`MoveFilesToNewDirectory`** and **`CreateAndWriteFiles`** threads to finish, but neither of them ever signal their completion (because they are each waiting on something else), then we're faced with a deadlock.

There are a few things to keep in mind:

1. We can set a **timeout** value when calling **WaitForSingleObject**. If the object isn't signaled within the specified timeout, the function will return, indicating a timeout. This means we aren't always waiting indefinitely; it depends on how we've called the function.
   
2. Just because we see **WaitForSingleObject** in our call stack doesn't inherently indicate a problem. It's quite common for our threads to wait on synchronization objects as part of their regular operation. However, if we find our thread unexpectedly stuck or suspect a deadlock, seeing **WaitForSingleObject** can give us valuable insights into potential issues.
