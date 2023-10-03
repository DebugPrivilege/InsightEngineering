# What is Pool Memory?

Pool memory serves as a dedicated section of the computer's RAM, managed by the operating system, where the core system components and device drivers operate. This area is different from the memory used by regular applications. The main job of pool memory is to offer a space where important system functions can safely allocate and free up memory. It's like a workspace for the operating system and device drivers to carry out their tasks.

Here's are a few examples for what it can be used for:

- **Buffering File Data:** When reading from or writing to a file, a buffer is often required to temporarily hold the data. This buffer is frequently allocated from pool memory for quick access and performance.
- **Resource Tracking:** Keeping a record of system resources, like file handles or hardware resources, often requires dynamically allocated structures. Pool memory is a common choice for this.
- **I/O Request Packets (IRPs):** File operations often involve the use of IRPs, which are data structures used to describe a single I/O operation. These IRPs are commonly allocated from pool memory.

There are different categories of pool memory, such as **Paged** and **Non-Paged**.

**Paged Pool:**
- Can be swapped out to disk to free up RAM.
- There's usually more paged pool memory available, but it can be slower to use.
- Used for non-time-critical operations.
  
**Non-Paged Pool:**
- Always remains in physical RAM and never paged out.
- Faster to access but a more limited resource.
- Used for time-sensitive, critical operations.

Here are the common Kernel Mode APIs for pool memory allocation:

| Function                   | Description                                                                                                  |
|----------------------------|--------------------------------------------------------------------------------------------------------------|
| `ExAllocatePool()`         | Allocates a block of pool memory.                                                                             |
| `ExAllocatePoolWithTag()`  | Allocates a block of pool memory and associates a tag with it for debugging.                                  |
| `ExAllocatePoolZero()`     | Allocates a block of pool memory and initializes it to zero.                                                  |
| `ExAllocatePool2()`        | Allocates a block of pool memory with additional flags for more granular control over the allocation.          |
| `ExFreePool()`             | Frees a previously allocated block of pool memory.                                                            |
| `ExFreePoolWithTag()`      | Frees a previously allocated block of pool memory, specifying the tag it was associated with during allocation.|

# Pool Structure

The **`_POOL_HEADER`** is a small piece of information that sits in front of each chunk of memory managed by the Windows pool manager. This header helps keep track of important details about the memory chunk, like how big it is or what type it is (paged or non-paged), etc.

```
0: kd> dt nt!_POOL_HEADER
   +0x000 PreviousSize     : Pos 0, 8 Bits
   +0x000 PoolIndex        : Pos 8, 8 Bits
   +0x002 BlockSize        : Pos 0, 8 Bits
   +0x002 PoolType         : Pos 8, 8 Bits
   +0x000 Ulong1           : Uint4B
   +0x004 PoolTag          : Uint4B
   +0x008 ProcessBilled    : Ptr64 _EPROCESS
   +0x008 AllocatorBackTraceIndex : Uint2B
   +0x00a PoolTagHash      : Uint2B
```

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/d7c91fe6-2ffe-4327-ae60-c309d7d39d49)



# Code Sample (1) - Buffer Overflow

The issue with this code is the buffer overflow, which could lead to a system crash.  We've got a buffer named **`writeBuffer`** that is designed to hold only 10 bytes of data. Later on, we generate a "random" size for the data that we want to store in this buffer. The tricky part is that this random size can be anywhere between 1 and 10,000 bytes.

```c
#include <ntddk.h>

NTSTATUS CreateRandomFile(UINT32 fileNumber, SIZE_T dataSize) {
    HANDLE fileHandle;
    OBJECT_ATTRIBUTES objAttr;
    IO_STATUS_BLOCK ioStatusBlock;
    NTSTATUS status;
    WCHAR fileNameBuffer[] = L"\\??\\C:\\Temp\\fileX.txt";  // X will be replaced
    CHAR writeBuffer[10];  // Fixed size for demonstration
    UNICODE_STRING filePath;

    // Replace 'X' in fileNameBuffer with the actual file number
    fileNameBuffer[wcslen(fileNameBuffer) - 5] = L'0' + (WCHAR)fileNumber;

    // Initialize the file path
    RtlInitUnicodeString(&filePath, fileNameBuffer);

    // Initialize object attributes
    InitializeObjectAttributes(&objAttr, &filePath, OBJ_CASE_INSENSITIVE | OBJ_KERNEL_HANDLE, NULL, NULL);

    // Create the file
    status = ZwCreateFile(&fileHandle, GENERIC_WRITE, &objAttr, &ioStatusBlock, NULL, FILE_ATTRIBUTE_NORMAL, 0, FILE_OVERWRITE_IF, FILE_SYNCHRONOUS_IO_NONALERT, NULL, 0);

    if (!NT_SUCCESS(status)) {
        KdPrint(("Failed to create the file, status code: %x\n", status));
        return status;
    }

    // Fill the write buffer with some data (e.g., all 'A's)
    RtlFillMemory(writeBuffer, dataSize, 'A');

    // Write to the file
    status = ZwWriteFile(fileHandle, NULL, NULL, NULL, &ioStatusBlock, writeBuffer, dataSize, NULL, NULL);

    // Clean up
    ZwClose(fileHandle);

    return status;
}

VOID UnloadDriver(PDRIVER_OBJECT DriverObject) {
    KdPrint(("Driver Unloaded\n"));
}

NTSTATUS DriverEntry(PDRIVER_OBJECT DriverObject, PUNICODE_STRING RegistryPath) {
    DriverObject->DriverUnload = UnloadDriver;

    LARGE_INTEGER tickCount;
    UINT32 fileCount;
    SIZE_T dataSize;

    // Get system tick count for "random" seed
    KeQueryTickCount(&tickCount);

    // Create between 1-10 files
    fileCount = (tickCount.LowPart % 10) + 1;

    for (UINT32 i = 0; i < fileCount; ++i) {
        // Refresh tick count for another "random" seed
        KeQueryTickCount(&tickCount);

        // Generate a "random" data size between 1 and 10000 bytes (size of writeBuffer)
        dataSize = (tickCount.LowPart % 10000) + 1;

        CreateRandomFile(i, dataSize);
    }

    return STATUS_SUCCESS;
}
```

The issue arises because we then attempt to fill **`writeBuffer`** with 'A's based on this randomly generated size. Since **`writeBuffer`** has room for only 10 bytes, trying to stuff it with more data—up to 10,000 bytes—creates a buffer overflow. This also leads to bugchecking the system:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/d5d5ccb3-21ac-4b02-b844-5f76976d86ec)


# Code Sample (2) - Dynamic Pool Memory Allocation

Here we are using the **`ExAllocatePool2`** API to dynamically allocate space from the Pool memory, matching it exactly with the size of data for each random file we're creating. This prevents buffer overflows and eliminates the risk of crashes. After we're done, we use **`ExFreePoolWithTag`** to safely release this Pool memory back, ensuring the driver runs without issues.

```c
#include <ntddk.h>

NTSTATUS CreateRandomFile(UINT32 fileNumber, SIZE_T dataSize) {
    HANDLE fileHandle;
    OBJECT_ATTRIBUTES objAttr;
    IO_STATUS_BLOCK ioStatusBlock;
    NTSTATUS status;
    WCHAR fileNameBuffer[] = L"\\??\\C:\\Temp\\fileX.txt"; // X will be replaced
    PCHAR writeBuffer;
    UNICODE_STRING filePath;

    // Replace 'X' in fileNameBuffer with the actual file number
    fileNameBuffer[wcslen(fileNameBuffer) - 5] = L'0' + (WCHAR)fileNumber;

    // Initialize the file path
    RtlInitUnicodeString(&filePath, fileNameBuffer);

    // Initialize object attributes
    InitializeObjectAttributes(&objAttr, &filePath, OBJ_CASE_INSENSITIVE | OBJ_KERNEL_HANDLE, NULL, NULL);

    // Create the file
    status = ZwCreateFile(&fileHandle, GENERIC_WRITE, &objAttr, &ioStatusBlock, NULL, FILE_ATTRIBUTE_NORMAL, 0, FILE_OVERWRITE_IF, FILE_SYNCHRONOUS_IO_NONALERT, NULL, 0);

    if (!NT_SUCCESS(status)) {
        KdPrint(("Failed to create the file, status code: %x\n", status));
        return status;
    }

    // Allocate write buffer from paged pool
    writeBuffer = (PCHAR)ExAllocatePool2(POOL_FLAG_PAGED, dataSize, 'WrB1');
    if (writeBuffer == NULL) {
        KdPrint(("Failed to allocate memory for write buffer.\n"));
        ZwClose(fileHandle);
        return STATUS_INSUFFICIENT_RESOURCES;
    }

    // Fill the write buffer with some data (e.g., all 'A's)
    RtlFillMemory(writeBuffer, dataSize, 'A');

    // Write to the file
    status = ZwWriteFile(fileHandle, NULL, NULL, NULL, &ioStatusBlock, writeBuffer, dataSize, NULL, NULL);

    // Clean up
    ExFreePoolWithTag(writeBuffer, 'WrB1');
    ZwClose(fileHandle);

    return status;
}

VOID UnloadDriver(PDRIVER_OBJECT DriverObject) {
    KdPrint(("Driver Unloaded\n"));
}

NTSTATUS DriverEntry(PDRIVER_OBJECT DriverObject, PUNICODE_STRING RegistryPath) {
    DriverObject->DriverUnload = UnloadDriver;

    LARGE_INTEGER tickCount;
    UINT32 fileCount;
    SIZE_T dataSize;

    // Get system tick count for "random" seed
    KeQueryTickCount(&tickCount);

    // Create between 1-10 files
    fileCount = (tickCount.LowPart % 10) + 1;

    for (UINT32 i = 0; i < fileCount; ++i) {
        // Refresh tick count for another "random" seed
        KeQueryTickCount(&tickCount);

        // Generate a "random" data size between 1 and 10000 bytes
        dataSize = (tickCount.LowPart % 10000) + 1;

        CreateRandomFile(i, dataSize);
    }

    return STATUS_SUCCESS;
}
```

Let's now load the kernel driver:

```
sc.exe create TestDriver type= kernel binPath=C:\Users\User\source\repos\Driver\x64\Release\Driver.sys
```

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/7eede9dd-db21-4a00-8b76-b49343a6473a)



Start the kernel driver:

```
sc start TestDriver
```

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/58e87b6b-d4a9-48b7-b20d-5288f86ed6a6)


This code is a Windows Kernel Driver that dynamically creates between 1 to 10 text files in the **`C:\Temp`** directory, with filenames containing a variable number generated based on the system's tick count. For each file, it allocates a write buffer from the paged pool.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/41503f13-4228-4d09-873d-9242fe9b7a6e)


# How does ExAllocatePool2 works?

The **`ExAllocatePool2`** function is a part of the Windows Kernel API and is used for dynamic memory allocation in kernel-mode drivers.

```
PVOID ExAllocatePool2(
  ULONG Flags,
  SIZE_T NumberOfBytes,
  ULONG Tag
);
```

- **Flags:** The type of pool memory to allocate. This can be **`POOL_FLAG_PAGED`** for paged pool or **`POOL_FLAG_NON_PAGED`** for non-paged pool.
- **NumberOfBytes:** The size, in bytes, of the memory block to allocate.
- **Tag:** A tag to assign to the allocated memory block for tracking or debugging purposes.

# How does ExFreePoolWithTag works?

The **`ExFreePoolWithTag`** function is used in Windows Kernel programming to free a block of memory that was previously allocated by the **`ExAllocatePoolWithTag`** or **`ExAllocatePool2`** functions. 

```
VOID ExFreePoolWithTag(
  _In_ PVOID P,            // A pointer to the block of memory to be freed.
  _In_ ULONG Tag           // A 4-byte tag that was used during memory allocation with ExAllocatePoolWithTag.
);
```
