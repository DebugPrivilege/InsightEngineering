# What is a Handle Leak?

A handle leak occurs when a program obtains handles to system resources but fails to release them properly. Over time, this can lead to resource exhaustion, which can make the system or the program unstable. Handle leaks are a concern in long-running programs where they can accumulate and cause issues such as high memory usage, slow system performance, or even system crashes. The most direct cause is simply not closing a handle when it's no longer needed. This can happen if the code path that releases the handle is not executed, either due to logic errors or exceptions.

# How to view Handles of a Process?

The first example would be to use **Handle.exe** from SysInternals.

```
C:\Users\User\Desktop\Handle>handle.exe -h

Nthandle v5.0 - Handle viewer
Copyright (C) 1997-2022 Mark Russinovich
Sysinternals - www.sysinternals.com

usage: handle [[-a [-l]] [-v|-vt] [-u] | [-c <handle> [-y]] | [-s]] [-p <process>|<pid>] [name] [-nobanner]
  -a         Dump all handle information.
  -l         Just show pagefile-backed section handles.
  -c         Closes the specified handle (interpreted as a hexadecimal number).
             You must specify the process by its PID. Requires administrator
             rights.
             WARNING: Closing handles can cause application or system instability.
  -g         Print granted access.
  -y         Don't prompt for close handle confirmation.
  -s         Print count of each type of handle open.
  -u         Show the owning user name when searching for handles.
  -v         CSV output with comma delimiter.
  -vt        CSV output with tab delimiter.
  -p         Dump handles belonging to process (partial name accepted).
  name       Search for handles to objects with <name> (fragment accepted).
  -nobanner  Do not display the startup banner and copyright message.

No arguments will dump all file references.
```

We can filter on the process name that we want to retrieve the handles from:

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/3caf3c27-f600-4f58-bd02-f77087c550b3)

The second tool we can use is **Process Explorer** from SysInternals. This will also count the Handles of a process:

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/b7452166-790d-478a-badb-c071c229d3de)

# Using the Windows Performance Toolkit to analyze Handle Leaks

The **Windows Performance Toolkit (WPT)** is a collection of tools provided by Microsoft to analyze and troubleshoot performance issues in Windows operating systems. 

We can use the **Windows Performance Recorder (WPR)** to capture system events and other performance-related data into trace files (.etl). It allows us to configure what kind of data we want to collect, such as CPU usage, disk activity, Handles, and more.

The **Windows Performance Analyzer (WPA)** can then be used to open and analyze the trace files (.etl) that are collected by the Windows Performance Recorder. It provides a graphical interface where you can dig deep into the data to understand what's happening in your system.

1. Start with compiling the following code:

```c
#include <windows.h>
#include <iostream>
#include <string>

int main()
{
    // Declare variables for file handle and other file operations
    HANDLE hFile;
    DWORD dwBytesWritten = 0;
    char writeData[] = "Hello, World!\n"; 

    // Loop to create 2000 files
    for (int i = 1; i <= 2000; ++i)
    {
        // Construct filename
        std::wstring fileName = L"C:\\Temp\\file" + std::to_wstring(i) + L".txt";

        // Create or open a file
        hFile = CreateFile(fileName.c_str(),             // Filename
            GENERIC_WRITE,                // Write access
            0,                            // No sharing
            NULL,                         // Default security
            CREATE_ALWAYS,                // Overwrite existing
            FILE_ATTRIBUTE_NORMAL,        // Normal file attributes
            NULL);                        // No template

        if (hFile == INVALID_HANDLE_VALUE)
        {
            std::cerr << "Could not create or open file " << i << " (error " << GetLastError() << ")\n";
            // Normally, you'd return or break here, but we'll continue to try the other files
            continue;
        }

        // Write "Hello, World!" 20000 times to the file
        for (int j = 0; j < 20000; ++j)
        {
            if (!WriteFile(hFile, writeData, strlen(writeData), &dwBytesWritten, NULL))
            {
                std::cerr << "Could not write to file " << i << " (error " << GetLastError() << ")\n";
                // not closing the handle to create a handle leak
                break;
            }
        }

        // not closing the handle to create a handle leak
    }

    std::cout << "Created 2000 files with 'Hello, World!' written 20000 times in each, and with handle leaks.\n";
    return 0;
}
```

2. Open the **Windows Performance Recorder** and select **Handle usage**. Once done, click on **Start** to start the recording.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/c66062fc-2977-4e0b-9c29-425926d12982)


3. Now run the compiled the code until the execution has been completed.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/549f1d1b-9d98-443e-a191-be8abd1fd0dc)


4. Let's now save the recording once the execution has been completed

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/a96a7776-5012-4906-94f0-45030179fa39)


5. Open the .etl file in **Windows Performance Analyzer (WPA)**

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/eebf053b-2fba-44ee-9a47-58fc05f9e505)


# Analyzing trace file with WPA

Load the .etl file in **Windows Performance Analyzer (WPA)** and under **Memory**, select **Handles** and add it to the Analysis view:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/27c32266-521b-496f-833b-5b98753fdc26)


Select the process of interest and click on **Filter to selection**.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/b860499d-042e-4814-a427-9440d56fc9fc)


Once we have set the filter, we should now see only one process:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/e618adaa-5a13-4c72-a00a-1e427e72c35e)


Open **View Editor** and select **Create Stack**

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/ae42c373-6a08-47b2-9012-6d199defabd2)


The call stack shows that **Test.exe** is creating file handles through a series of function calls, both in user-mode and kernel-mode. The process starts at the **`main`** function in **Test.exe** and goes all the way to the kernel's object manager to actually create the handle. The high "Count" of **11,675** indicates a handle leak, which we can see as well in the graph that keeps increasing. It includes the function(s) that are leaking handles.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/74aa9f93-a4cd-4f8b-8a4b-6bc84c01e531)

# Code Sample - Code without Handle Leaks

The modified code creates 2000 text files in the "C:\Temp\" directory and writes the string "Hello, World!\n" 20,000 times into each file. Unlike the original version, it properly closes each file handle after use to prevent handle leaks.

```c
#include <windows.h>
#include <iostream>
#include <string>

int main()
{
    // Declare variables for file handle and other file operations
    HANDLE hFile;
    DWORD dwBytesWritten = 0;
    char writeData[] = "Hello, World!\n"; 

    // Loop to create 2000 files
    for (int i = 1; i <= 2000; ++i)
    {
        // Construct filename
        std::wstring fileName = L"C:\\Temp\\file" + std::to_wstring(i) + L".txt";

        // Create or open a file
        hFile = CreateFile(fileName.c_str(),             // Filename
            GENERIC_WRITE,                // Write access
            0,                            // No sharing
            NULL,                         // Default security
            CREATE_ALWAYS,                // Overwrite existing
            FILE_ATTRIBUTE_NORMAL,        // Normal file attributes
            NULL);                        // No template

        if (hFile == INVALID_HANDLE_VALUE)
        {
            std::cerr << "Could not create or open file " << i << " (error " << GetLastError() << ")\n";
            // Normally, you'd return or break here, but we'll continue to try the other files
            continue;
        }

        // Write "Hello, World!" 20000 times to the file
        for (int j = 0; j < 20000; ++j)
        {
            if (!WriteFile(hFile, writeData, strlen(writeData), &dwBytesWritten, NULL))
            {
                std::cerr << "Could not write to file " << i << " (error " << GetLastError() << ")\n";
                break;
            }
        }

        // Close the file handle
        CloseHandle(hFile);
    }

    std::cout << "Created 2000 files with 'Hello, World!' written 20000 times in each, without handle leaks.\n";
    return 0;
}
```

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/19f54541-a428-42ca-bf41-1ad3fbdd92ff)


When we analyze this process using Windows Performance Analyzer (WPA), we can see in the graph that the metrics remain consistent over time:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/3ae0b47c-e2d4-4318-9699-275def3a0d3e)
