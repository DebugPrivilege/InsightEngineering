# Pages

In Windows and other operating systems, memory is managed in chunks called "pages." These pages are like blocks of memory that the computer uses to organize and keep track of data. There are two types of pages:

- **Small Pages:** These are the standard size, usually 4 KB (kilobytes). They are used for most regular memory tasks.
- **Large Pages:** These are much bigger, like 2 MB (megabytes) or more. They're special because they help make things faster for programs that need a lot of memory at once.

**Large Pages** have some restricted rules:

- They can't be moved to disk storage (they are "non-pageable"), which means they stay in the fast-access memory (RAM).
- Only certain programs with special permissions can use large pages. The ability to allocate large pages requires the process to have a specific privilege set: the **`SeLockMemoryPrivilege`**. This privilege is not given to processes by default because locking large amounts of memory can affect system performance. 

| Architecture | Small Page Size | Large Page Size | Small Pages per Large Page |
|--------------|-----------------|-----------------|----------------------------|
| x86 (PAE)    | 4 KB            | 2 MB            | 512                        |
| x64          | 4 KB            | 2 MB            | 512                        |
| ARM          | 4 KB            | 4 MB            | 1024                       |

# Heap

When an application needs to allocate memory, especially smaller amounts, it uses the heap manager. This is because the heap manager is optimized for small allocations, which is more memory-efficient than using functions like VirtualAlloc, which allocate memory in larger chunks equal to the size of a page (typically 4 KB). The heap manager can make much finer allocations, down to 8 bytes on 32-bit systems and 16 bytes on 64-bit systems.

# Pool Memory

Pool memory in Windows is used for allocating system memory, especially in kernel mode, for objects that require dynamic memory allocation and have varied lifetimes. To allocate memory from the system memory pool, we would typically use the following APIs:

- **ExAllocatePool2:** used for allocating memory from the system's nonpaged or paged pool, providing advanced options for type, tagging, and allocation priority.

To troubleshoot pool memory issues from a kernel memory dump, we would typically focus on pool-related structures and commands like **`!pool`** and **`!poolused`** to identify potential memory leaks or corrupt allocations.

Here is a snippet from the **`!poolused`** command:

```
                            NonPaged                                         Paged
 Tag       Allocs       Frees      Diff         Used       Allocs       Frees      Diff         Used

 -UMD           2           1         1           48            0           0         0            0	UNKNOWN pooltag '-UMD', please update pooltag.txt
 .UMD           2           1         1          144            0           0         0            0	UNKNOWN pooltag '.UMD', please update pooltag.txt
 ALPC        4528        2111      2417      1440624            0           0         0            0	ALPC port objects , Binary: nt!alpc
 APpt         968           0       968       124112       109457      107115      2342       255952	UNKNOWN pooltag 'APpt', please update pooltag.txt
```

The columns from this output mean the following:

| Column Name         | Explanation                                                                                                                                                                      |
|---------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Tag                 | Represents the pool tag associated with the memory allocation, aiding in tracking and debugging memory usage.                                                                     |
| NonPaged Allocs     | Shows the number of allocations made from the nonpaged pool for this tag, where the nonpaged pool stores memory that cannot be swapped out and is essential for system stability. |
| NonPaged Frees      | Indicates the number of times memory was freed in the nonpaged pool for this tag.                                                                                                |
| NonPaged Diff       | Reflects the difference between allocations and frees in the nonpaged pool for this tag, which can highlight memory leak concerns.                                              |
| NonPaged Used       | Represents the total amount of memory currently in use in the nonpaged pool for this tag.                                                                                        |
| Paged Allocs        | Displays the number of allocations made from the paged pool for this tag, where the paged pool is used for objects that can be paged out to disk.                              |
| Paged Frees         | Depicts the number of times memory was freed in the paged pool for this tag.                                                                                                      |
| Paged Diff          | Shows the difference between allocations and frees in the paged pool for this tag, which can also be indicative of memory usage patterns.                                       |
| Paged Used          | Represents the total amount of memory currently in use in the paged pool for this tag.                                                                                           |
| Description         | Provides additional context or information about the pool tag, aiding in understanding its purpose or status, such as being unknown or requiring an update.                    |

# Virtual Address Translation

A unique private address space is allocated for each process. To map the unique private address space of each process, individual Page Directory Pointer Tables (PDPT), Page Directories, and Page Tables are created. The physical address of this PDPT is securely stored within the **`KPROCESS`** structure assigned to each process. 

First, we'll begin by obtaining the memory address of the process we are focusing on:

```
2: kd> !process 0 1 winlogon.exe
PROCESS ffffa604f6820080
    SessionId: 1  Cid: 0344    Peb: c904737000  ParentCid: 02d8
    DirBase: f2451000  ObjectTable: ffff8000ada77640  HandleCount: 282.
    Image: winlogon.exe
    VadRoot ffffa604f669bf30 Vads 101 Clone 0 Private 349. Modified 11623. Locked 0.
    DeviceMap ffff8000ab218150
    Token                             ffff8000adb46990
    ElapsedTime                       00:00:53.132
    UserTime                          00:00:00.000
    KernelTime                        00:00:00.031
    QuotaPoolUsage[PagedPool]         160120
    QuotaPoolUsage[NonPagedPool]      14592
    Working Set Sizes (now,min,max)  (3887, 50, 345) (15548KB, 200KB, 1380KB)
    PeakWorkingSetSize                12810
    VirtualSize                       2101375 Mb
    PeakVirtualSize                   2101410 Mb
    PageFaultCount                    17047
    MemoryPriority                    BACKGROUND
    BasePriority                      13
    CommitCharge                      723
```

This output shows the contents of a memory address **`(0xf2451000)`** and nearby addresses, which seem to hold information about a process's memory mappings. The PDDT is saved in the **`KPROCESS`** structure.

```
2: kd> !dq f2451000
#f2451000 0a000000`86784867 0a000000`8606e867
#f2451010 00000000`00000000 00000000`00000000
#f2451020 0a000000`f2974867 00000000`00000000
#f2451030 00000000`00000000 00000000`00000000
#f2451040 00000000`00000000 00000000`00000000
#f2451050 00000000`00000000 00000000`00000000
#f2451060 00000000`00000000 00000000`00000000
#f2451070 00000000`00000000 00000000`00000000
```

This output displays information about a **`_KPROCESS`** structure located at memory address **`0xffffa604f6820080`**. It highlights the **`DirectoryTableBase`** field, which contains the value **`0xf2451000`**, representing the physical address of the Page Directory Pointer Table (PDPT) for the process associated with this structure.

```
2: kd> !mex.ddt nt!_KPROCESS ffffa604f6820080 -y Directory

dt nt!_KPROCESS ffffa604f6820080 -y Directory () Recursive: [ -r1 -r2 -r ] Verbose Normal dt
==================================================================================
   +0x028 DirectoryTableBase   : 0xf2451000 (0n4064612352)
```

# Virtual or Physical Memory

A page file is a reserved space that uses as an extension of physical memory (RAM) to temporarily store data that doesn't fit in RAM, allowing the operating system to manage memory more effectively.

If there is no page file in a Windows operating system, processes will still use virtual memory. Virtual memory is a fundamental concept in modern operating systems. It provides processes with the illusion of having more memory than is physically available by using a combination of RAM and disk space (in the absence of a page file) to store data.

Without a page file, Windows will rely solely on physical memory (RAM) for storing process data. When physical memory is exhausted, the operating system will have no additional space to store data, potentially leading to resource allocation issues

# Section Object and Control Area

A section object is a data structure that represents a region of virtual memory, typically associated with a file. It acts as a bridge between the virtual memory space and the storage medium, allowing data to be mapped into memory for reading or writing.

On the other hand, a control area is a data structure that is associated with a section object. It contains information about the section object, such as its size, protection attributes, and the location of the data on disk. 

This output details various file objects associated with .etl files, each identified by a unique **Object** Memory Address (e.g., **`ffffa604f53aa910`**, **`ffffa604f56ac090`**). These addresses represent the location of the file objects in the system's memory.

```
2: kd> !mex.grep -A 6 .etl !handle 0 3 0 File
        Directory Object: 00000000  Name: \Windows\System32\LogFiles\WMI\RtBackup\EtwRTDiagtrack-Listener.etl {HarddiskVolume4}

006c: Object: ffffa604f53aa910  GrantedAccess: 0012019f (Inherit) (Audit) Entry: ffff8000ab2be1b0
Object: ffffa604f53aa910  Type: (ffffa604f24fe820) File
    ObjectHeader: ffffa604f53aa8e0 (new version)
        HandleCount: 1  PointerCount: 32768
        Directory Object: 00000000  Name: \Device\HarddiskVolume4\$Extend\$RmMetadata\$TxfLog\$TxfLog {clfs}
        Directory Object: 00000000  Name: \ProgramData\Microsoft\Windows\wfp\wfpdiag.etl {HarddiskVolume4}

0108: Object: ffffa604f56ac090  GrantedAccess: 00100001 (Inherit) (Audit) Entry: ffff8000ab2be420
Object: ffffa604f56ac090  Type: (ffffa604f24fe820) File
    ObjectHeader: ffffa604f56ac060 (new version)
        HandleCount: 1  PointerCount: 1

        Directory Object: 00000000  Name: \Windows\System32\LogFiles\WMI\NtfsLog.etl {HarddiskVolume4}

018c: Object: ffffa604f53a7350  GrantedAccess: 00120089 (Inherit) Entry: ffff8000ab2be630
Object: ffffa604f53a7350  Type: (ffffa604f24fe820) File
    ObjectHeader: ffffa604f53a7320 (new version)
        HandleCount: 1  PointerCount: 32781
        Directory Object: 00000000  Name: \Windows\System32\drivers\en-US\ntfs.sys.mui {HarddiskVolume4}
        Directory Object: 00000000  Name: \Windows\System32\LogFiles\WMI\Wifi.etl {HarddiskVolume4}

0274: Object: ffffa604f56ada70  GrantedAccess: 0012019f (Audit) Entry: ffff8000ab2be9d0
Object: ffffa604f56ada70  Type: (ffffa604f24fe820) File
    ObjectHeader: ffffa604f56ada40 (new version)
        HandleCount: 1  PointerCount: 32769

        Directory Object: 00000000  Name: \Windows\System32\LogFiles\WMI\NetCore.etl {HarddiskVolume4}

02b8: Object: ffffa604f64c76a0  GrantedAccess: 0012008b (Protected) (Inherit) (Audit) Entry: ffff8000ab2beae0
Object: ffffa604f64c76a0  Type: (ffffa604f24fe820) File
    ObjectHeader: ffffa604f64c7670 (new version)
        HandleCount: 1  PointerCount: 32765
        Directory Object: 00000000  Name: \Windows\System32\LogFiles\WMI\RadioMgr.etl {HarddiskVolume4}

02bc: Object: ffffa604f64c5b50  GrantedAccess: 0012008b (Inherit) Entry: ffff8000ab2beaf0
Object: ffffa604f64c5b50  Type: (ffffa604f24fe820) File
    ObjectHeader: ffffa604f64c5b20 (new version)
        HandleCount: 1  PointerCount: 32766
        Directory Object: 00000000  Name: \Windows\System32\LogFiles\WMI\LwtNetLog.etl {HarddiskVolume4}

02c0: Object: ffffa604f61d59c0  GrantedAccess: 00020003 (Protected) Entry: ffff8000ab2beb00
Object: ffffa604f61d59c0  Type: (ffffa604f24fe820) File
    ObjectHeader: ffffa604f61d5990 (new version)
        HandleCount: 1  PointerCount: 32658
        Directory Object: 00000000  Name: \Windows\System32\config\SOFTWARE {HarddiskVolume4}
        Directory Object: 00000000  Name: \Windows\System32\LogFiles\WMI\RtBackup\EtwRTUBPM.etl {HarddiskVolume4}

02d0: Object: ffffa604f61d4250  GrantedAccess: 0012019f (Inherit) Entry: ffff8000ab2beb40
Object: ffffa604f61d4250  Type: (ffffa604f24fe820) File
    ObjectHeader: ffffa604f61d4220 (new version)
        HandleCount: 1  PointerCount: 32682
        Directory Object: 00000000  Name: \Windows\bootstat.dat {HarddiskVolume4}
        Directory Object: 00000000  Name: \Windows\Prefetch\ReadyBoot\ReadyBoot.etl {HarddiskVolume4}

0324: Object: ffffa604f56ab230  GrantedAccess: 0012019f Entry: ffff8000ab2bec90
Object: ffffa604f56ab230  Type: (ffffa604f24fe820) File
    ObjectHeader: ffffa604f56ab200 (new version)
        HandleCount: 1  PointerCount: 32769

        Directory Object: 00000000  Name: \Windows\System32\LogFiles\WMI\Microsoft-Windows-Rdp-Graphics-RdpIdd-Trace.etl {HarddiskVolume4}

0358: Object: ffffa604f64c5700  GrantedAccess: 0013008b (Protected) (Audit) Entry: ffff8000ab2bed60
Object: ffffa604f64c5700  Type: (ffffa604f24fe820) File
    ObjectHeader: ffffa604f64c56d0 (new version)
        HandleCount: 1  PointerCount: 32760
        Directory Object: 00000000  Name: \Windows\System32\LogFiles\WMI\RtBackup\EtwRTEventlog-Security.etl {HarddiskVolume4}

035c: Object: ffffa604f64c6110  GrantedAccess: 0012008b (Inherit) (Audit) Entry: ffff8000ab2bed70
Object: ffffa604f64c6110  Type: (ffffa604f24fe820) File
    ObjectHeader: ffffa604f64c60e0 (new version)
        HandleCount: 1  PointerCount: 32748
        Directory Object: 00000000  Name: \Windows\System32\WDI\LogFiles\WdiContextLog.etl.002 {HarddiskVolume4}

0360: Object: ffffa604f64e6930  GrantedAccess: 0012008b Entry: ffff8000ab2bed80
Object: ffffa604f64e6930  Type: (ffffa604f24fe820) File
    ObjectHeader: ffffa604f64e6900 (new version)
        HandleCount: 1  PointerCount: 32787
        Directory Object: 00000000  Name: \Windows\System32\WDI\LogFiles\BootPerfDiagLogger.etl {HarddiskVolume4}

0364: Object: ffffa604f64c3bb0  GrantedAccess: 0013008b Entry: ffff8000ab2bed90
Object: ffffa604f64c3bb0  Type: (ffffa604f24fe820) File
    ObjectHeader: ffffa604f64c3b80 (new version)
        HandleCount: 1  PointerCount: 32778
        Directory Object: 00000000  Name: \Windows\System32\LogFiles\WMI\RtBackup\EtwRTDiagLog.etl {HarddiskVolume4}

0368: Object: ffffa604f64c7c60  GrantedAccess: 0012008b (Protected) (Inherit) Entry: ffff8000ab2beda0
Object: ffffa604f64c7c60  Type: (ffffa604f24fe820) File
    ObjectHeader: ffffa604f64c7c30 (new version)
        HandleCount: 1  PointerCount: 32766
        Directory Object: 00000000  Name: \Windows\System32\LogFiles\WMI\ReFSLog.etl {HarddiskVolume4}

036c: Object: ffffa604f64c38d0  GrantedAccess: 0013008b (Inherit) Entry: ffff8000ab2bedb0
Object: ffffa604f64c38d0  Type: (ffffa604f24fe820) File
    ObjectHeader: ffffa604f64c38a0 (new version)
        HandleCount: 1  PointerCount: 32742
        Directory Object: 00000000  Name: \Windows\System32\LogFiles\WMI\RtBackup\EtwRTDefenderAuditLogger.etl {HarddiskVolume4}

0370: Object: ffffa604f64c3e90  GrantedAccess: 0013008b (Inherit) (Audit) Entry: ffff8000ab2bedc0
Object: ffffa604f64c3e90  Type: (ffffa604f24fe820) File
    ObjectHeader: ffffa604f64c3e60 (new version)
        HandleCount: 1  PointerCount: 32784
        Directory Object: 00000000  Name: \Windows\System32\LogFiles\WMI\RtBackup\EtwRTEventLog-System.etl {HarddiskVolume4}

0374: Object: ffffa604f64c3760  GrantedAccess: 0013008b (Protected) (Inherit) Entry: ffff8000ab2bedd0
Object: ffffa604f64c3760  Type: (ffffa604f24fe820) File
    ObjectHeader: ffffa604f64c3730 (new version)
        HandleCount: 1  PointerCount: 32742
        Directory Object: 00000000  Name: \Windows\System32\LogFiles\WMI\RtBackup\EtwRTDefenderApiLogger.etl {HarddiskVolume4}
```

The output is a detailed structure dump of a **`nt!_FILE_OBJECT`** at memory address **`ffffa604f64c3760`**, showing its properties including type, size, associated device and volume pointers, file system contexts, section object pointer, access rights, file flags, and the file name. It also provides information about the file's current byte offset, locking mechanisms, and associated IRP (I/O Request Packet) list.

The field **`SectionObjectPointer`** in the **`nt!_FILE_OBJECT`** structure output represents a pointer to a **`nt!_SECTION_OBJECT_POINTERS`** structure. This field is critical in the context of file objects because it links the file object to its associated section objects in memory.

```
2: kd> !mex.ddt nt!_FILE_OBJECT ffffa604f64c3760

dt nt!_FILE_OBJECT ffffa604f64c3760 () Recursive: [ -r1 -r2 -r ] Verbose Normal dt
==================================================================================
   +0x000 Type                 : 0n5
   +0x002 Size                 : 0n216 (0xd8)
   +0x008 DeviceObject         : 0xffffa604`f50e08f0 _DEVICE_OBJECT   !deviceobject   !devstack
   +0x010 Vpb                  : 0xffffa604`f5141c70 _VPB
   +0x018 FsContext            : 0xffff8000`abbed390 _SCB (PoolTag: Ntff)
   +0x020 FsContext2           : 0xffff8000`abbed608 Void  PoolTag: Ntff
   +0x028 SectionObjectPointer : 0xffffa604`f64e6828 _SECTION_OBJECT_POINTERS
   +0x030 PrivateCacheMap      : 0xffffa604`f5c6ee70 Void  PoolTag: CcSc
   +0x038 FinalStatus          : 0n0
   +0x040 RelatedFileObject    : (null)
   +0x048 LockOperation        : 0 ''
   +0x049 DeletePending        : 0 ''
   +0x04a ReadAccess           : 0x1 ''
   +0x04b WriteAccess          : 0x1 ''
   +0x04c DeleteAccess         : 0x1 ''
   +0x04d SharedRead           : 0x1 ''
   +0x04e SharedWrite          : 0 ''
   +0x04f SharedDelete         : 0x1 ''
   +0x050 Flags                : 0x40042 (0n262210)
   +0x058 FileName             : _UNICODE_STRING  "\Windows\System32\LogFiles\WMI\RtBackup\EtwRTDefenderApiLogger.etl"
   +0x068 CurrentByteOffset    : _LARGE_INTEGER 0x2f8c8 (0n194760)
   +0x070 Waiters              : 0
   +0x074 Busy                 : 0
   +0x078 LastLock             : (null)
   +0x080 Lock                 : _KEVENT
   +0x098 Event                : _KEVENT
   +0x0b0 CompletionContext    : (null)
   +0x0b8 IrpListLock          : 0
   +0x0c0 IrpList              : _LIST_ENTRY [ 0xffffa604`f64c3820 - 0xffffa604`f64c3820 ] [EMPTY OR 1 ELEMENT]
   +0x0d0 FileObjectExtension  : (null)
```

The **`nt!_SECTION_OBJECT_POINTERS`** structure doesn't represent the section object itself. Instead, it is a data structure used to store pointers related to file objects and their associated section objects.

```
2: kd> !mex.ddt 0xffffa604`f64e6828 nt!_SECTION_OBJECT_POINTERS

dt 0xffffa604`f64e6828 nt!_SECTION_OBJECT_POINTERS () Recursive: [ -r1 -r2 -r ] Verbose Normal dt
==================================================================================
   +0x000 DataSectionObject    : 0xffffa604`f6536ed0 Void  [SectionObject]
   +0x008 SharedCacheMap       : 0xffffa604`f5c6ecf0 Void  [SectionObject]
   +0x010 ImageSectionObject   : (null)
```

The output from the **`!ca`** command displays the details of a control area at memory address **`0xffffa604f6536ed0`**, linked to the file **`EtwRTDefenderApiLogger.etl`**. It shows a single section reference, no page frame number references, and indicates that the file has been purged.

```
2: kd> !ca 0xffffa604`f6536ed0

ControlArea  @ ffffa604f6536ed0
  Segment      ffff8000abafe890  Flink      ffffa604f6536ed8  Blink        ffffa604f6536ed8
  Section Ref                 1  Pfn Ref                   0  Mapped Views                0
  User Ref                    0  WaitForDel                0  Flush Count                 0
  File Object  ffffa604f64c3760  ModWriteCount             0  System Views                0
  WritableRefs                0  PartitionId                0  
  Flags (8080) File WasPurged 

      \Windows\System32\LogFiles\WMI\RtBackup\EtwRTDefenderApiLogger.etl

Segment @ ffff8000abafe890
  ControlArea       ffffa604f6536ed0  ExtendInfo    0000000000000000
  Total Ptes                     100
  Segment Size                100000  Committed                    0
  Flags (c0000) ProtectionMask 

Subsection 1 @ ffffa604f6536f50
  ControlArea  ffffa604f6536ed0  Starting Sector        0  Number Of Sectors  100
  Base Pte     0000000000000000  Ptes In Subsect      100  Unused Ptes          0
  Flags                       d  Sector Offset          0  Protection           6
  Accessed 
  Flink        ffffa604f6536fa0  Blink   ffffa604f6536fa0  MappedViews          0
```

# Working Set vs Private Bytes


The terms "process working set" and "private bytes" are used to describe different aspects of a process's memory usage:

- **Working Set:** The working set of a process refers to the set of memory pages currently visible to the process in physical RAM. The working set is a subset of the virtual address space of a process. It represents the total amount of memory that a process requires at a given time to continue working efficiently without causing too many page faults.
- **Private Bytes:** Private bytes refer to the amount of memory that a process has allocated that cannot be shared with other processes. These are used exclusively by that process. Private bytes are a part of the process's virtual memory, which includes both physical memory (RAM) and space in the page file on disk. It's a measure of the amount of memory a process is using which would be freed if the process was terminated. This includes memory allocations made by the process through the heap, stacks, and other memory allocations that are private to the process.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/e8404f17-51c4-4259-b03c-acb79a358596)


