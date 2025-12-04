# Summary

Kernel forensics, when applied to debugging and troubleshooting, involves a deep dive into the operating system's kernel to identify, analyze, and resolve issues at the system level. We often use debugging tools like WinDbg to help us. Because the kernel is the core of an operating system that handles critical tasks like hardware interactions, memory management, and process scheduling, any issues at this level can significantly impact the entire system. 

Our primary focus will be on investigating kernel crash dumps to pinpoint the causes of system crashes. We will also provide practical examples to demonstrate how a faulty driver can lead to a system-wide bugcheck.

**Chapters:**

```
- Section 1: Kernel Pool Memory
- Section 2: ERESOURCE
- Section 3: KMUTEX
- Section 4: Fast Mutex
- Section 5: Interrupt Request Level (IRQL)
- Section 6: Guarded Mutex
- Section 7: Event Signaling
- Section 8: Semaphore
- Section 9: Pushlock
- Section 10: Spinlock
- Section 11: Kernel Callbacks
```

# DISCLAIMER

I’ve always believed in reproducible analysis, so I used to include memory dumps and other files with each write-up so people could follow along. At the time I uploaded everything to MEGA, but I didn’t realize they would delete the data if I became inactive. Because of that, the links to those memory dumps no longer work.

<img width="421" height="635" alt="image" src="https://github.com/user-attachments/assets/7f714a88-2495-4ec9-b765-fdf0cf6ff936" />
