# What are Callbacks?

Callbacks in the Windows Kernel context are specific mechanisms used to notify a driver or a subsystem about certain events or conditions. A driver registers a callback function for a specific event or condition. This is usually done using a specific Windows Kernel API. When the event occurs (like process creation, registry access, etc.), the Windows Kernel calls the registered callback function. The callback function typically runs in the context of the thread that triggered the event.

```
+------------------------+
|     Application        |
|    or Kernel Driver    |
+------------------------+
        |
        | (1) Register Callback
        |
        v
+------------------------+
|      Windows           |
|        Kernel          |
+------------------------+
        |
        | (2) Event occurs (e.g.,
        |     process creation,
        |     file access)
        |
        v
+------------------------+
|      Callback          |
|      Function          |
+------------------------+
        |
        | (3) Execute callback logic
        |     (e.g., logging, monitoring,
        |     modifying behavior)
        |
        v
+------------------------+
|     Callback Action    |
|     (e.g., Logging,    |
|     Monitoring)        |
+------------------------+
        |
        | (4) Callback completes,
        |     returns to normal flow
        |
        v
+------------------------+
|     Back to App or     |
|     Kernel Driver      |
+------------------------+
```

# Types of Callbacks

Here are some of the common callbacks:

- **Process/Thread Notify Routine:** Notified when processes or threads are created or exit. Useful for monitoring or restricting process and thread activities.
- **Registry Filters:** Informed of registry operations. Useful for monitoring or altering registry accesses.
- **Image Load Routine:** Image Load Routines are called when an executable image (like a .exe or .dll file) is loaded into or unloaded from a process. 
- **Filter Manager Filter:** Receive notifications of file system activities, allowing for monitoring or modifying file operations.
- **Object Manager Callbacks:** Get notifications about operations on various Windows kernel objects.
