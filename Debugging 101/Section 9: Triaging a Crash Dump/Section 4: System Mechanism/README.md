# IRQL - PASSIVE_LEVEL

Kernel-mode threads typically run at the IRQL known as **`PASSIVE_LEVEL`** or IRQL 0. This is the lowest IRQL level for kernel-mode operations. At **`PASSIVE_LEVEL`**, threads can safely perform operations that might be time-consuming, such as file I/O or waiting for other system resources.

# IRQL - Thread Scheduler

The thread scheduler in Windows operates at the **`DISPATCH_LEVEL`** also known as IRQL 2. At this level, the scheduler makes decisions about which thread should run next, preempts lower-priority threads, and manages the ready queues.

When operating at **`DISPATCH_LEVEL`**, there are two important rules to follow:

- **No Paging I/O Operations:** At **`DISPATCH_LEVEL`**, the code cannot perform operations that might require paging. This is because paging operations might require accessing the disk, which could lead to a situation where the system deadlocks waiting for an I/O operation that itself requires a lower IRQL.
- **Limited Duration:** Code running at **`DISPATCH_LEVEL`** must execute for a limited duration and should not engage in long-term activities.

# IRQL Levels and Paging Requirements

Paging is a method where a computer's memory is divided into small chunks, allowing the system to use memory more efficiently by loading only the needed parts into physical memory.

**DISPATCH_LEVEL**
- Code executed at **`DISPATCH_LEVEL`**, such as that in response to a timer expiration, needs to be non-pageable. This is because page faults cannot be handled at **`DISPATCH_LEVEL`**.
- Access to pageable memory is also restricted at **`DISPATCH_LEVEL`**. Since handling a page fault is not possible, any memory accessed at this level must be guaranteed to be resident in physical memory, typically from the non-paged pool.

**APC_LEVEL**
- At **`APC_LEVEL`**, code can be pageable. This IRQL is lower than **`DISPATCH_LEVEL`** and allows for a wider range of operations, including handling page faults.

# Interrupt Service Routine

An Interrupt Service Routine (ISR) is a function designed to handle hardware interrupts. When a hardware device, like a network card or a keyboard, needs immediate attention from the CPU, it issues an interrupt. The ISR is the function that responds to this interrupt.

Imagine we press a key on our keyboard. This action generates an interrupt signal. Upon receiving this signal, the CPU momentarily pauses its current tasks to address the interrupt. It does this by executing an Interrupt Service Routine (ISR) that is specifically designed to handle keyboard interrupts. This ISR's job is to read the keystroke data directly from the keyboard hardware.

ISRs have a set of operating conditions and constraints:

- ISRs run at a high interrupt request level, ensuring they can respond to interrupts immediately without being preempted by less critical system activities.
- The ISR's code and any data it accesses must reside in non-pageable memory. This is because handling page faults at high IRQLs is not possible, and any attempt to access paged-out data would result in a system crash.

# Interrupt vs Exception

An interrupt is like a signal from hardware or software that tells the processor to stop and do something else, while an exception is like an error or special situation that happens while a program is running, causing the processor to change what it's doing.

The output from the **`!idt`** command is showing the Interrupt Descriptor Table (IDT) of the system. The IDT is a data structure that maps each interrupt or exception to its corresponding handler function. This output lists the address of each handler associated with various interrupts and exceptions.

```
2: kd> !idt

Dumping IDT: ffff9581fbba2000

/* Exception Trap */
00:	fffff8052b238700 nt!KiDivideErrorFault
01:	fffff8052b238a40 nt!KiDebugTrapOrFault	Stack = 0xFFFF9581FBBD2000
02:	fffff8052b239040 nt!KiNmiInterrupt	Stack = 0xFFFF9581FBBCB000
03:	fffff8052b2395c0 nt!KiBreakpointTrap
04:	fffff8052b239900 nt!KiOverflowTrap
05:	fffff8052b239c40 nt!KiBoundFault
06:	fffff8052b23a300 nt!KiInvalidOpcodeFault
07:	fffff8052b23a980 nt!KiNpxNotAvailableFault
08:	fffff8052b23ad40 nt!KiDoubleFaultAbort	Stack = 0xFFFF9581FBBBD000
09:	fffff8052b23b080 nt!KiNpxSegmentOverrunAbort
0a:	fffff8052b23b3c0 nt!KiInvalidTssFault
0b:	fffff8052b23b700 nt!KiSegmentNotPresentFault
0c:	fffff8052b23bac0 nt!KiStackFault
0d:	fffff8052b23be40 nt!KiGeneralProtectionFault
0e:	fffff8052b23c1c0 nt!KiPageFault
10:	fffff8052b23c980 nt!KiFloatingErrorFault
11:	fffff8052b23cd40 nt!KiAlignmentFault
12:	fffff8052b23d080 nt!KiMcheckAbort	Stack = 0xFFFF9581FBBC4000
13:	fffff8052b23de00 nt!KiXmmException
14:	fffff8052b23e200 nt!KiVirtualizationException
15:	fffff8052b23e8c0 nt!KiControlProtectionFault
1f:	fffff8052b231380 nt!KiApcInterrupt
20:	fffff8052b233680 nt!KiSwInterrupt
29:	fffff8052b23efc0 nt!KiRaiseSecurityCheckFailure
2c:	fffff8052b23f340 nt!KiRaiseAssertion
2d:	fffff8052b23f6c0 nt!KiDebugServiceTrap
2e:	fffff8052b23fa40 nt!KiSystemService
2f:	fffff8052b233e30 nt!KiDpcInterrupt
30:	fffff8052b231b00 nt!KiHvInterrupt
31:	fffff8052b231e50 nt!KiVmbusInterrupt0
32:	fffff8052b2321a0 nt!KiVmbusInterrupt1
33:	fffff8052b2324f0 nt!KiVmbusInterrupt2
34:	fffff8052b232840 nt!KiVmbusInterrupt3

/* ISR */
35:	fffff8052b22f088 nt!HalpInterruptCmciService (KINTERRUPT ffffa604f24d9900)

36:	fffff8052b22f090 nt!HalpInterruptCmciService (KINTERRUPT ffffa604f24d9a20)

b0:	fffff8052b22f460 ACPI!ACPIInterruptServiceRoutine (KINTERRUPT ffff9581fbdfedc0)

ce:	fffff8052b22f550 nt!HalpIommuInterruptRoutine (KINTERRUPT ffffa604f24da200)

d1:	fffff8052b22f568 nt!HalpTimerClockInterrupt (KINTERRUPT ffffa604f24da440)

d2:	fffff8052b22f570 nt!HalpTimerClockIpiRoutine (KINTERRUPT ffffa604f24da320)

d7:	fffff8052b22f598 nt!HalpInterruptRebootService (KINTERRUPT ffffa604f24d9d80)

d8:	fffff8052b22f5a0 nt!HalpInterruptStubService (KINTERRUPT ffffa604f24d9c60)

df:	fffff8052b22f5d8 nt!HalpInterruptSpuriousService (KINTERRUPT ffffa604f24d9b40)

e1:	fffff8052b234500 nt!KiIpiInterrupt
e2:	fffff8052b22f5f0 nt!HalpInterruptLocalErrorService (KINTERRUPT ffffa604f24d9ea0)

e3:	fffff8052b22f5f8 nt!HalpInterruptDeferredRecoveryService (KINTERRUPT ffffa604f24da0e0)

fd:	fffff8052b22f6c8 nt!HalpTimerProfileInterrupt (KINTERRUPT ffffa604f24da560)

fe:	fffff8052b22f6d0 nt!HalpPerfInterrupt (KINTERRUPT ffffa604f24d9fc0)
```

# Objects & Handles

An object refers to a data structure that represents a system resource, such as a thread, file, or a graphic image. The operating system utilizes a kernel component known as the object manager to efficiently manage these objects. 

- Assigning readable names to system resources, making them easier to identify.
- Facilitating the sharing of resources and data between different processes.
- Monitoring references to objects to determine when they are no longer in use, allowing for their automatic deallocation.

It's important to note that not everything qualifies as an object. The categorization as an object is reserved for data that requires sharing, protection, naming, or visibility to user-mode programs through system services.

For user-mode processes to interact with these objects, they must possess a handle to the object. This handle is obtained when a process creates or opens an object by name, representing the process's access rights to the object. Using a handle to reference an object is more efficient than using its name, as it allows the Object Manager to bypass the name lookup process and directly locate the object.

# Kernel Objects

Kernel objects are like containers that represents system resources, such as files, threads, or synchronization primitives (like mutexes and semaphores). They represent various system resources and provide a way to interact with and control these resources. 

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/4594980d-1b7b-43f1-9363-1d4db0b4f1c1)


# Kernel Handle Table

The Object Manager identifies these kernel handles by examining the high bit of the handle value. On a 32-bit system, kernel-handle-table handles have values greater than **`0x80000000`**, while on a 64-bit system, they have values greater than **`0xFFFFFFFF80000000`**. This high-bit check allows the Object Manager to distinguish between handles that are meant for kernel-mode operations and handles that are specific to user-mode processes

The handle **`0n4`** is associated with the process **`System`**, and it is a kernel-handle-table handle. 

```
2: kd> !handle 0n4

PROCESS ffffa604f24ec040
    SessionId: none  Cid: 0004    Peb: 00000000  ParentCid: 0000
    DirBase: 007d5000  ObjectTable: ffff8000ab28ae80  HandleCount: 3279.
    Image: System

Kernel handle table at ffff8000ab28ae80 with 3279 entries in use

0004: Object: ffffa604f24ec040  GrantedAccess: 001fffff (Protected) Entry: ffff8000ab2be010
Object: ffffa604f24ec040  Type: (ffffa604f24820c0) Process
    ObjectHeader: ffffa604f24ec010 (new version)
        HandleCount: 4  PointerCount: 131245
```

The handle **`0n912`** is associated with the process **`lsass.exe`**. In this case, the handle appears to be a user-mode handle.

```
2: kd> !handle 0n496

PROCESS ffffa604f683b180
    SessionId: 0  Cid: 0390    Peb: 810b68e000  ParentCid: 02e0
    DirBase: 86f7f000  ObjectTable: ffff8000ada76080  HandleCount: 1144.
    Image: lsass.exe

Handle table at ffff8000ada76080 with 1144 entries in use

01f0: Object: ffffa604f6825060  GrantedAccess: 001f0003 (Protected) (Inherit) Entry: ffff8000adccb7c0
Object: ffffa604f6825060  Type: (ffffa604f24ead20) Event
    ObjectHeader: ffffa604f6825030 (new version)
        HandleCount: 1  PointerCount: 1
```

# I/O Request Packet

An IRP, or I/O Request Packet, is a data structure used to represent and manage input/output (I/O) operations. IRPs are a core component of the I/O system in Windows and are used for communication between drivers, the kernel, and user-mode applications.

This sample output shows an IRP (I/O Request Packet) for a file cleanup operation, currently in process with the NTFS file system driver, which has not completed its handling, and the Filter Manager is waiting for the NTFS driver to finish, all within the context of **MsMpEng.exe**. Thus, the cleanup operation has not been completed yet. 

The NTFS file system driver, on the other hand, has a status of None, indicating that it is currently processing the IRP and has not yet completed its part of the operation.

```
0: kd> !mex.mirp ffffab88277799f0

Irp Details: ffffab88277799f0 [ verbose | !ddt | !irp ]

    Thread                        Frame Count User Event
    ============================= =========== ================
    ffffab8825e1f040(MsMpEng.exe)          12 ffff97011448f2c8

Irp Stack Frame(s)

       # Driver             Major   Minor Dispatch Routine    Flg Ctrl Status  Completion Invoker(s)  Device           File             Context                Completion Routine               Args                                               
    ==== ================== ======= ===== =================== === ==== ======= ====================== ================ ================ ====================== ================================ ===================================================================
    ->11 \FileSystem\Ntfs   CLEANUP     0 Ntfs!NtfsFsdCleanup   0   e0 None    Cancel, Success, Error ffffab8820ca9030 ffffab88216e4800 ffffab8826798aa0(FMic) FLTMGR!FltpPassThroughCompletion 0000000000000000 0000000000000000 0000000000000000 0000000000000000
      12 \FileSystem\FltMgr CLEANUP     0 FLTMGR!FltpDispatch   0    1 Pending                        ffffab8820c9d2e0 ffffab88216e4800                                                         0000000000000000 0000000000000000 0000000000000000 0000000000000000

File Details: ffffab88216e4800

    Name                                                   Device           Driver                      Vpb Flags Byte Offset              FsContext             FsContext2 Owning Process
    ====================================================== ================ ============== ================ ===== =========== ====================== ====================== ==============
    \Users\User\source\repos\Driver\x64\Release\Driver.sys ffffab8820c914b0 \Driver\volmgr ffffab8820b30680 40062           0 ffffe4013d5e2700(Ntff) ffffe4013d8d4240(Ntfc)
```
