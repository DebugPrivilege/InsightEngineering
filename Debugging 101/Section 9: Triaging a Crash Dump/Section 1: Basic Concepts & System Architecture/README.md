# Process and Threads

A process acts as a container, including all the necessary resources utilized during the execution of a program instance. This container includes components such as the program's executable code, allocated memory, and various system handles.

Within this process container, there exist threads. A thread is a distinct execution path or a sequence of commands within the process. It's the actual line of execution that the Windows operating system schedules and manages. Each thread represents a separate stream of instructions that can be executed by the CPU.

The key thing to understand is that the process, which acts as the container, doesn't do any tasks on its own. It's the threads inside the process that actually get things done. Threads are vital because they take the program's code, which is just a set of instructions, and make it work. They are what turn the code into actions. Without threads, the process just sits there without doing anything, because there's no way to put the program's code into action.

```
2: kd> !mex.lt ffffa604f24ec040
Process PID Thread             Id State      Time Reason
======= === ================ ==== ======= ======= ===============
System    4 ffffa604f248d040    c Waiting 54s.062 Executive
System    4 ffffa604f248e040   10 Waiting 54s.062 Executive
System    4 ffffa604f2499040   14 Waiting 29s.375 Executive
System    4 ffffa604f2498200   18 Waiting 54s.062 Executive
System    4 ffffa604f2546080   1c Waiting 54s.062 Executive
System    4 ffffa604f2513080   30 Waiting 28s.593 Executive
System    4 ffffa604f258a080   34 Waiting 25s.500 WrQueue
System    4 ffffa604f2524080   38 Waiting    31ms WrQueue
System    4 ffffa604f259b080   3c Waiting 54s.062 Suspended
System    4 ffffa604f25ac080   40 Waiting 54s.062 Suspended
System    4 ffffa604f25bd080   44 Waiting 54s.062 Suspended
System    4 ffffa604f25ce080   48 Waiting 54s.062 Suspended
System    4 ffffa604f24df080   50 Waiting   656ms Executive
System    4 ffffa604f24e1080   54 Waiting   656ms Executive
System    4 ffffa604f24e5080   5c Waiting  6s.703 Executive
System    4 ffffa604f24e3080   58 Waiting 16s.640 WrQueue
System    4 ffffa604f2504080   60 Waiting 54s.062 WrFreePage
System    4 ffffa604f2506080   64 Waiting   656ms WrFreePage
System    4 ffffa604f2508080   68 Waiting   390ms WrFreePage
System    4 ffffa604f250a080   6c Waiting   390ms WrFreePage
System    4 ffffa604f2515080   78 Waiting 54s.062 WrQueue
System    4 ffffa604f2517080   7c Waiting 54s.062 WrQueue
System    4 ffffa604f2519080   80 Waiting 13s.421 WrQueue
System    4 ffffa604f268c080   84 Waiting 14s.468 WrQueue
System    4 ffffa604f251b080   88 Waiting 25s.500 WrQueue

<<< SNIPPET >>>
```

# Physical and Virtual Memory

**Physical Memory**, commonly referred to as RAM (Random Access Memory), is the main memory in a computer where data is stored for quick access by the computer's processor. For example, when we're working and editing a document, the content is stored in RAM. This allows our computers to quickly access and update the information as we type or make changes. 

RAM is a type of **volatile** memory, meaning it loses its content when the power is turned off. It is used for storing data that the computer needs to access quickly and temporarily. The amount of physical memory in a computer significantly impacts its performance. More RAM allows a computer to work with more information at the same time, which can speed up processes and allow for smoother multitasking.

**Virtual Memory** is a memory management capability of an operating system that uses both the computer's physical memory (RAM) and a portion of the secondary storage (usually a hard drive or SSD) to compensate for shortages of physical memory. 

When the **physical memory** (RAM) is not sufficient to handle all the data and programs currently in use, the OS uses virtual memory to increase the available memory. It does this by temporarily transferring data that is not actively used from the RAM to a space on the hard drive called the paging file or swap file.

**Example:**

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/8a73f77d-ca83-4e95-a9a2-de101a3f1295)


# Idle Process

The number of threads in the **Idle** Process typically corresponds to the number of logical processors in the system. The **Idle** Process, which has a Process ID (PID) of 0, is a special system process whose threads run whenever there are no runnable threads in the system. Each CPU core has its own idle thread within the **Idle** Process.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/caacc53e-6d3c-4f5c-a836-16e54400460a)


The output from the PowerShell command indicates that our system's processor has 2 physical cores and **4** logical processors:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/332bfffc-e0b8-48d7-a5c7-46bc596ab912)


You can examine the information Windows maintains for SMT processors using the **`!smt`** command in the kernel debugger

```
0: kd> !smt
SMT Summary:
------------

KeActiveProcessors:
****------------------------------------------------------------ (000000000000000f)
IdleSummary:
--**------------------------------------------------------------ (000000000000000c)
 No PRCB             SMT Set                                                                             APIC Id
  0 fffff8025414b180 **-------------------------------------------------------------- (0000000000000003) 0x00000000
  1 ffffd7018a9dc180 **-------------------------------------------------------------- (0000000000000003) 0x00000001
  2 ffffd7018ad48180 --**------------------------------------------------------------ (000000000000000c) 0x00000002
  3 ffffd7018ae80180 --**------------------------------------------------------------ (000000000000000c) 0x00000003

Maximum cores per physical processor:   2
Maximum logical processors per core:    2
```

# Threads within System Process

The difference between threads running in the **SYSTEM** process and those in other processes in Windows is primarily about their role, privileges, and impact on the system's stability.

Threads in the **SYSTEM** process, which is a part of the Windows kernel and operate at the kernel level, meaning they have high privileges and direct access to the system's hardware and core functions. These threads are responsible for critical tasks like managing system resources, handling device drivers, and ensuring the operating system's stability and security. Due to their high privilege level, these threads are heavily protected and managed by the operating system. Errors or failures in these threads can lead to system issues, such as a blue screen of death.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/9bd70e3c-942f-4573-abb8-ff2014ee1939)



On the other hand, threads running in other processes, typically user-level applications or services, operate with more limited privileges. They are designed to perform specific tasks based on the application's functionality, such as processing user inputs, executing program logic, or managing file and network operations. They are isolated from each other and the system's core functions, which means that while they can crash due to bugs or errors, such crashes usually affect only the application they belong to and not the entire system.

# Image Header

The DOS Header, found at the beginning of every Portable Executable (PE) binary, starts with the signature **`MZ`**. This signature is a tribute to Mark Zbikowski, one of the early developers of MS-DOS, who played a key role in designing the MS-DOS image file format.

```
0: kd> lmvm SysmonDrv
Browse full module list
start             end                 module name
fffff802`5c000000 fffff802`5c02e000   SysmonDrv   (deferred)             
    Image path: SysmonDrv.sys
    Image name: SysmonDrv.sys
    Browse all global symbols  functions  data
    Timestamp:        Mon Nov 13 07:47:09 2023 (655244FD)
    CheckSum:         0002C5E2
    ImageSize:        0002E000
    Translations:     0000.04b0 0000.04e4 0409.04b0 0409.04e4
    Information from resource tables:
```

In this step, we are examining the Image Header, and we can confirm that it's indeed a DOS Header:

```
0: kd> dc fffff802`5c000000
fffff802`5c000000  00905a4d 00000003 00000004 0000ffff  MZ..............
fffff802`5c000010  000000b8 00000000 00000040 00000000  ........@.......
fffff802`5c000020  00000000 00000000 00000000 00000000  ................
fffff802`5c000030  00000000 00000000 00000000 00000108  ................
fffff802`5c000040  0eba1f0e cd09b400 4c01b821 685421cd  ........!..L.!Th
fffff802`5c000050  70207369 72676f72 63206d61 6f6e6e61  is program canno
fffff802`5c000060  65622074 6e757220 206e6920 20534f44  t be run in DOS 
fffff802`5c000070  65646f6d 0a0d0d2e 00000024 00000000  mode....$.......
```

# User Mode and Kernel Mode

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/e41efc55-e934-450f-9050-4ebf4b34fb73)

**User Mode** is where applications and subsystems run. It's a restricted mode of operation in which the code being executed has limited access to system memory and cannot directly interact with hardware. This is done for security and stability reasons; if an application crashes, it won't crash the entire system. 

**User Mode** is represented by the top half and includes:
- **System Processes:** These are processes that the system runs on behalf of the user, like services.
- **Services:** Background processes that provide features like networking, printing, etc.
- **Applications:** These are the programs that users interact with directly, such as web browsers, word processors, and games.
- **Subsystem dlls:** Dynamic link libraries used by various subsystems.

An example of **User Mode** operation is when you open a text editor and write a document. The text editor is an application running in **User Mode**, and it asks the system for resources (like memory to store your text).

**Kernel Mode** is the mode where the core of the operating system runs and has unrestricted access to the system memory and hardware. It's designed for trusted functions only and provides various services that are not available in User Mode for security reasons. 

**Kernel Mode** is represented by the bottom half and includes:

- **Executive:** Manages system resources and security, acts as an intermediary between the hardware and the user-level code.
- **Device and File System Drivers:** Software that interacts directly with hardware devices and the system's file storage mechanisms.
- **Kernel:** The core of the operating system, which has complete control over everything in the system.
- **Hardware Abstraction Layer (HAL):** A layer that hides hardware differences from the rest of the kernel.
- **Windowing and Graphics:** The system components that handle the graphical user interface and window management.

An example of **Kernel Mode** operation is when you save the document you're working on. The text editor (in User Mode) sends a request to the operating system to write to the disk. This request is handed down to the Kernel Mode, where the file system driver actually writes the data to the storage device.

Between these two modes is **NTDLL.dll**, which is a User Mode library that applications and subsystems use to request services from the Windows kernel. It contains the "system call" interface, which is the means by which **User Mode** code transitions into **Kernel Mode** to ask the kernel to perform tasks that only it has the privilege to execute.

The separation between **User Mode** and **Kernel Mode** is fundamental for system security and stability. **User Mode** provides a sandboxed environment for applications, protecting the system from potential crashes or malicious behavior in user applications. **Kernel Mode** has all the privileges and access.
