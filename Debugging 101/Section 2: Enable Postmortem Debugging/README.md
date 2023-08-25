# What is Postmortem Debugging?

Postmortem debugging is a process where we analyze a program's state after it has crashed, typically using a dump file. With **ProcDump**, we can enable postmortem debugging to automatically generate dump files when an application crashes. This allows us to analyze the crash later using a tool like **WinDbg**.

# How to enable Postmortem Debugging?

1. Start with downloading ProcDump: https://learn.microsoft.com/en-us/sysinternals/downloads/procdump
2. Open CMD as an administrator and run the following command:

```
procdump64.exe -ma -i <Path to user mode dumps of processes>
```

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/4091b744-d59f-4c46-9fe0-9b02a6922635)

3. Compile and run the following code:

```c
#include <windows.h>
#include <iostream>

int main() {
    // Create a file
    HANDLE hFile = CreateFile(L"HelloWorld.txt",
        GENERIC_WRITE,
        0,
        NULL,
        CREATE_ALWAYS,
        FILE_ATTRIBUTE_NORMAL,
        NULL);

    if (hFile == INVALID_HANDLE_VALUE) {
        std::cerr << "Failed to create file. Error: " << GetLastError() << std::endl;
        return 1;
    }

    // Data to write to the file
    const char* data = "Hello, World!";
    DWORD bytesWritten;

    // Write data to the file
    if (!WriteFile(hFile, data, strlen(data), &bytesWritten, NULL)) {
        std::cerr << "Failed to write to file. Error: " << GetLastError() << std::endl;
        CloseHandle(hFile);
        return 1;
    }

    // Close the file handle
    CloseHandle(hFile);

    // Introduce Access Violation (WRITE) by attempting to write to an uninitialized pointer
    int* uninitializedPointer{}; // Uninitialized pointer
    *uninitializedPointer = 42; // This line cause an Access Violation (WRITE)

    return 0;
}
```

4. The program has crashed and we will get a user mode dump of the process, which we can load in WinDbg.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/7a9ecd81-9154-4a01-bce7-d3001bd48455)

