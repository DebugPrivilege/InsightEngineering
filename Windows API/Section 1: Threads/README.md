# What are Threads?

A thread is like a worker. In a program, each worker (thread) can do a different job at the same time as the others. The function of a thread is to enable a program to execute several tasks simultaneously or in a concurrent manner. Each thread has its own set of registers, its own stack, and its own program counter, but threads within the same process share the same address space, which means they have access to the same data and can communicate with each other more easily than separate processes can.

# Process Explorer - Examining Single Threaded Program

A single-threaded program is like a one-lane road where tasks, like cars, must go one after the other. It can only do one thing at a time, finishing one task before starting the next. Many command line tools and scripts are single-threaded because they typically perform a series of tasks in a specific order and don't require concurrent execution.

Here is an example:

```c
#include <windows.h>
#include <tlhelp32.h>
#include <iostream>
#include <string>

#define NUM_FILES 1000
#define NUM_REPEATS_FILES 3
#define NUM_REPEATS_PROCESSES 30

void CreateAndDeleteFiles() {
    for (int j = 0; j < NUM_REPEATS_FILES; j++) {
        for (int i = 0; i < NUM_FILES; i++) {
            std::wstring fileName = L"C:\\Temp\\file_" + std::to_wstring(i) + L".txt";
            HANDLE hFile = CreateFile(
                fileName.c_str(),
                GENERIC_WRITE,
                0,
                NULL,
                CREATE_ALWAYS,
                FILE_ATTRIBUTE_NORMAL,
                NULL
            );

            if (hFile == INVALID_HANDLE_VALUE) {
                std::wcerr << L"CreateFile failed for " << fileName << std::endl;
                continue;
            }
            else {
                std::wcout << L"Created file: " << fileName << std::endl;
            }

            DWORD bytesWritten;
            WriteFile(
                hFile,
                "Hello World",
                11, // number of bytes in "Hello World"
                &bytesWritten,
                NULL
            );

            CloseHandle(hFile);

            if (!DeleteFile(fileName.c_str())) {
                std::wcerr << L"DeleteFile failed for " << fileName << std::endl;
            }
            else {
                std::wcout << L"Deleted file: " << fileName << std::endl;
            }
        }
    }
}

void EnumerateProcesses() {
    for (int j = 0; j < NUM_REPEATS_PROCESSES; j++) {
        HANDLE hSnapshot = CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, 0);
        if (hSnapshot == INVALID_HANDLE_VALUE) {
            std::wcerr << L"CreateToolhelp32Snapshot failed" << std::endl;
            return;
        }

        PROCESSENTRY32W pe32;
        pe32.dwSize = sizeof(pe32);

        if (!Process32FirstW(hSnapshot, &pe32)) {
            std::wcerr << L"Process32First failed" << std::endl;
            CloseHandle(hSnapshot);
            return;
        }

        do {
            std::wcout << L"Process ID: " << pe32.th32ProcessID << ", Process Name: " << pe32.szExeFile << std::endl;
        } while (Process32NextW(hSnapshot, &pe32));

        CloseHandle(hSnapshot);
    }
}

int main() {
    // Get the high-resolution performance counter frequency
    LARGE_INTEGER frequency;
    QueryPerformanceFrequency(&frequency);

    // Get the current counter value at the start of the program
    LARGE_INTEGER start;
    QueryPerformanceCounter(&start);

    CreateAndDeleteFiles();
    EnumerateProcesses();

    // Get the current counter value at the end of the program
    LARGE_INTEGER end;
    QueryPerformanceCounter(&end);

    // Calculate the total execution time in seconds
    double time = static_cast<double>(end.QuadPart - start.QuadPart) / frequency.QuadPart;

    std::wcout << L"Total execution time: " << time << L" seconds" << std::endl;

    return 0;
}
```

This code performs two primary tasks: creating and deleting files, and enumerating processes. It creates and deletes a specified number of files in a loop, lists all running processes on the computer in a loop, and calculates the total execution time for these operations. As the tasks are performed one after the other and there are no additional threads being created for parallel execution, it can be considered as a single-threaded program.

Let's run the code and use **Process Explorer** to examine the threads and the call stack. The expectation of this section is just to view a call stack of a thread, and find the exact same function names that we have in our code. A call stack is a data structure that tracks the sequence of function calls in a program, helping to know where the execution should return after each call completes.

1. The first step is to download **Process Explorer** and run it as an Administrator: https://learn.microsoft.com/en-us/sysinternals/downloads/process-explorer

![image](https://private-user-images.githubusercontent.com/63166600/256294761-5d018166-2644-4586-8dff-c2f849a7f9f1.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTEiLCJleHAiOjE2OTIwOTE0NTMsIm5iZiI6MTY5MjA5MTE1MywicGF0aCI6Ii82MzE2NjYwMC8yNTYyOTQ3NjEtNWQwMTgxNjYtMjY0NC00NTg2LThkZmYtYzJmODQ5YTdmOWYxLnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFJV05KWUFYNENTVkVINTNBJTJGMjAyMzA4MTUlMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjMwODE1VDA5MTkxM1omWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPTQ2ZTVkODcyM2JiNWFhYTFkMmM1NjcwZGE0YTM3YWRiNGU3NzA2YjVmY2JhM2E2NmFjOTUwNDg0NzA3NDcwZTAmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.M8-K6tUIzsOYz7RRW8IKTEtOUy-9bSOvAFCgA_EQZdY)

2. Start compiling the code and run it, while using **Process Explorer** to monitor the threads and call stack

![image](https://private-user-images.githubusercontent.com/63166600/256296918-7a0094c6-d717-4c24-b8c6-f8b8bf2eee37.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTEiLCJleHAiOjE2OTIwOTE0NTMsIm5iZiI6MTY5MjA5MTE1MywicGF0aCI6Ii82MzE2NjYwMC8yNTYyOTY5MTgtN2EwMDk0YzYtZDcxNy00YzI0LWI4YzYtZjhiOGJmMmVlZTM3LnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFJV05KWUFYNENTVkVINTNBJTJGMjAyMzA4MTUlMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjMwODE1VDA5MTkxM1omWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPWJkMWU0Mjg1ZWZjNGM3YzljZDk4NmU1YjkzZmNjZDA4MTIxNWE2OTgxNTc1YWYxMjAxZDQ4MmZlZTQ0YTIwYjcmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.PtLMOAOSSup-q_doaJgtWiR0DcPPNUmZh64fRhwYsvs)

3. Look for our program that we compiled and use **Process Explorer** to view the threads.

![image](https://private-user-images.githubusercontent.com/63166600/256297940-374282a2-bf4c-4415-b10a-d8e0916d0e21.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTEiLCJleHAiOjE2OTIwOTE0NTMsIm5iZiI6MTY5MjA5MTE1MywicGF0aCI6Ii82MzE2NjYwMC8yNTYyOTc5NDAtMzc0MjgyYTItYmY0Yy00NDE1LWIxMGEtZDhlMDkxNmQwZTIxLnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFJV05KWUFYNENTVkVINTNBJTJGMjAyMzA4MTUlMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjMwODE1VDA5MTkxM1omWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPTlhZTMwZWRhNTI3ZDRiYjkwOTU0YmY4ZTFjNzFhZGM2YzAxYWNjMGUwZWJhMTVjOWM0N2MyMzQ2NGE0NjhjMDUmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.i_ivCtIJlhCU55BnCDe80cGOF163XmUdYvIjyr4eeY8)

4. Click on the thread and view the call stack of it. Keep refreshing, since the call stack will change during the execution of the process.

Here is an example where we can see our function **`CreateAndDeleteFiles`** in a call stack of a thread:

![image](https://private-user-images.githubusercontent.com/63166600/256300031-921563c7-0116-4fd1-94e3-bf4c77642985.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTEiLCJleHAiOjE2OTIwOTE0NTMsIm5iZiI6MTY5MjA5MTE1MywicGF0aCI6Ii82MzE2NjYwMC8yNTYzMDAwMzEtOTIxNTYzYzctMDExNi00ZmQxLTk0ZTMtYmY0Yzc3NjQyOTg1LnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFJV05KWUFYNENTVkVINTNBJTJGMjAyMzA4MTUlMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjMwODE1VDA5MTkxM1omWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPWY1MWM1YWU4M2U3Nzc1NWQ0ZWMxMDY1OTMwYTZkOGFkMjVkNTAwY2FiZmZkMzY2YTZmMmIwZTVmYjkyZjUxNjAmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.oJ_quMe8vhVKsY5O9HyUg-9dlMyVsolQRFXg36dWxSc)

At this example, we can see **`EnumerateProcesses`** in the call stack of a thread:

![image](https://private-user-images.githubusercontent.com/63166600/256300744-3d9aeb5c-4903-4205-95df-20819b301a02.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTEiLCJleHAiOjE2OTIwOTE0NTMsIm5iZiI6MTY5MjA5MTE1MywicGF0aCI6Ii82MzE2NjYwMC8yNTYzMDA3NDQtM2Q5YWViNWMtNDkwMy00MjA1LTk1ZGYtMjA4MTliMzAxYTAyLnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFJV05KWUFYNENTVkVINTNBJTJGMjAyMzA4MTUlMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjMwODE1VDA5MTkxM1omWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPWJhZDNiMTcyZTkwNTllMGVkNWRlNTNiMjQzMTUxNWU2NWJhOTgzNTJlNTgwZDI4MjhhOTcyNDk4YzdmMDZiYjImWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.7g4R-E6_VmWIO4GLs0VlUb3y7FwJenes4vgDyJDxhas)

If we now review our code again, we can see these two functions as well within our code:

![image](https://private-user-images.githubusercontent.com/63166600/256303024-b6689a88-8116-4af9-9482-13e9a60cc62f.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTEiLCJleHAiOjE2OTIwOTE0NTMsIm5iZiI6MTY5MjA5MTE1MywicGF0aCI6Ii82MzE2NjYwMC8yNTYzMDMwMjQtYjY2ODlhODgtODExNi00YWY5LTk0ODItMTNlOWE2MGNjNjJmLnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFJV05KWUFYNENTVkVINTNBJTJGMjAyMzA4MTUlMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjMwODE1VDA5MTkxM1omWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPTZjY2Y4NmU4N2I1ZWJhMjVhMmY0ODIxZjAzZTc3NTIxYmIxOTZjNTJhMjQ5ZWRmZTcyMjIzNTQ3NTU0Y2QyZmEmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.2Y2fBiWwZyllNrUGf8OKj18Y6a5sSIIoCAG4Z3PDYL8)

![image](https://private-user-images.githubusercontent.com/63166600/256303132-ff78e5b1-f54c-41f4-9fe4-47afd0a4b93e.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTEiLCJleHAiOjE2OTIwOTE0NTMsIm5iZiI6MTY5MjA5MTE1MywicGF0aCI6Ii82MzE2NjYwMC8yNTYzMDMxMzItZmY3OGU1YjEtZjU0Yy00MWY0LTlmZTQtNDdhZmQwYTRiOTNlLnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFJV05KWUFYNENTVkVINTNBJTJGMjAyMzA4MTUlMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjMwODE1VDA5MTkxM1omWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPWYyN2NlOWRlZDdhYzI1MjQ1OWMwNmM2ZGJkMzNkODBjZjkzNmQwMjBhY2QyOWIzNzczMjliZjdmMTcyYzU1NzgmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.E7HxOt_69YvW7a7dHw8omWD_FSZEIxXzGkCKdCfaOpo)

Call stacks are very useful for debugging as they provide a roadmap of function execution in a program. For example, when an error occurs, the call stack offers a snapshot of the sequence of function calls that led to the error, enabling developers and IT Pro's to locate the source of the problem and the context around it more effectively.

# What have we learned now?

The call stack changes when a function has been completed because the call stack is a data structure that tracks the execution of functions in a program. Each time a function is called, it is added (or "pushed") onto the top of the call stack. When a function finishes executing, it is removed (or "popped") from the top of the stack.

# Process Explorer - Examining Multi Threaded Program

A multi-threaded program is a type of software that can do many things at once by using multiple "threads", which are like separate paths of instructions within the program. This way, different parts of the program can work independently and concurrently, making the program **faster** and more efficient.

Here is an example of a multi-threaded code:

```c
#include <windows.h>
#include <tlhelp32.h>
#include <iostream>
#include <string>
#include <vector>

#define NUM_FILES 1000
#define NUM_REPEATS_FILES 3
#define NUM_REPEATS_PROCESSES 30

DWORD WINAPI CreateAndDeleteFiles(LPVOID lpParam) {
    for (int j = 0; j < NUM_REPEATS_FILES; j++) {
        for (int i = 0; i < NUM_FILES; i++) {
            std::wstring fileName = L"C:\\Temp\\file_" + std::to_wstring(i) + L".txt";
            HANDLE hFile = CreateFile(
                fileName.c_str(),
                GENERIC_WRITE,
                0,
                NULL,
                CREATE_ALWAYS,
                FILE_ATTRIBUTE_NORMAL,
                NULL
            );

            if (hFile == INVALID_HANDLE_VALUE) {
                std::wcerr << L"CreateFile failed for " << fileName << std::endl;
                continue;
            }
            else {
                std::wcout << L"Created file: " << fileName << std::endl;
            }

            DWORD bytesWritten;
            WriteFile(
                hFile,
                "Hello World",
                11, // number of bytes in "Hello World"
                &bytesWritten,
                NULL
            );

            CloseHandle(hFile);

            if (!DeleteFile(fileName.c_str())) {
                std::wcerr << L"DeleteFile failed for " << fileName << std::endl;
            }
            else {
                std::wcout << L"Deleted file: " << fileName << std::endl;
            }
        }
    }

    return 0;
}


DWORD WINAPI EnumerateProcesses(LPVOID lpParam) {
    for (int j = 0; j < NUM_REPEATS_PROCESSES; j++) {
        HANDLE hSnapshot = CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, 0);
        if (hSnapshot == INVALID_HANDLE_VALUE) {
            std::wcerr << L"CreateToolhelp32Snapshot failed" << std::endl;
            return 1;
        }

        PROCESSENTRY32W pe32;
        pe32.dwSize = sizeof(pe32);

        if (!Process32FirstW(hSnapshot, &pe32)) {
            std::wcerr << L"Process32First failed" << std::endl;
            CloseHandle(hSnapshot);
            return 1;
        }

        do {
            std::wcout << L"Process ID: " << pe32.th32ProcessID << ", Process Name: " << pe32.szExeFile << std::endl;
        } while (Process32NextW(hSnapshot, &pe32));

        CloseHandle(hSnapshot);
    }

    return 0;
}

int main() {
    DWORD threadID;
    HANDLE hThreads[2];

    // Get the high-resolution performance counter frequency
    LARGE_INTEGER frequency;
    QueryPerformanceFrequency(&frequency);

    // Get the current counter value at the start of the program
    LARGE_INTEGER start;
    QueryPerformanceCounter(&start);

    hThreads[0] = CreateThread(NULL, 0, CreateAndDeleteFiles, NULL, 0, &threadID);
    hThreads[1] = CreateThread(NULL, 0, EnumerateProcesses, NULL, 0, &threadID);

    WaitForMultipleObjects(2, hThreads, TRUE, INFINITE);

    // Get the current counter value at the end of the program
    LARGE_INTEGER end;
    QueryPerformanceCounter(&end);

    for (int i = 0; i < 2; i++) {
        CloseHandle(hThreads[i]);
    }

    // Calculate the total execution time in seconds
    double time = static_cast<double>(end.QuadPart - start.QuadPart) / frequency.QuadPart;

    std::wcout << L"Total execution time: " << time << L" seconds" << std::endl;

    return 0;
}
```

This C++ program runs two concurrent threads, one creating and deleting files within a temporary directory and the other enumerating running processes, using the Windows API. The **CreateThread** function within this code is used to create two threads, one for the **`CreateAndDeleteFiles`** function and one for the **`EnumerateProcesses`** function. 

The purpose of creating these separate threads is to allow these functions to run concurrently. This means that the operations of creating and deleting files and enumerating processes can happen at the same time, rather than one after the other.

Let's examine the threads again with **Process Explorer**

1. Run **Process Explorer** as an administrator

![image](https://private-user-images.githubusercontent.com/63166600/256333733-c3fc0841-7f3f-4f21-a4f8-a653b4a9cf87.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTEiLCJleHAiOjE2OTIwOTE0NTMsIm5iZiI6MTY5MjA5MTE1MywicGF0aCI6Ii82MzE2NjYwMC8yNTYzMzM3MzMtYzNmYzA4NDEtN2YzZi00ZjIxLWE0ZjgtYTY1M2I0YTljZjg3LnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFJV05KWUFYNENTVkVINTNBJTJGMjAyMzA4MTUlMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjMwODE1VDA5MTkxM1omWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPTI3ZWM3MDRkM2IyM2QxNTlkZDAxNjAyZmVmZWYyNGFkYTQzMjZhOWNiZDJmMmVmZGZjOWRkOWIxNTJiM2ZlMTYmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.2730YwjuoY3hVmkx1wmu0DYyzLR99B2BclozafBhemY)

2. Compile the above code and run it, while having **Process Explorer** in the background to view all the threads.

![image](https://private-user-images.githubusercontent.com/63166600/256335465-72038707-5cb9-4943-af5e-a07e19542de3.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTEiLCJleHAiOjE2OTIwOTE0NTMsIm5iZiI6MTY5MjA5MTE1MywicGF0aCI6Ii82MzE2NjYwMC8yNTYzMzU0NjUtNzIwMzg3MDctNWNiOS00OTQzLWFmNWUtYTA3ZTE5NTQyZGUzLnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFJV05KWUFYNENTVkVINTNBJTJGMjAyMzA4MTUlMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjMwODE1VDA5MTkxM1omWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPTUzOGIzMmEwY2U2NjM0MWM5NWQyMTUwYTFhYmZhNTRmMmNjNjExZWZlZTMxMjAxNjNmYzlkODk5NTg4MWEyMjgmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.6RnvoPBQHmIq1llt5mizlxRwf84IoQhmHJHafBf-OWo)

3. When we run the code, it will create two additional threads apart from the main thread. **Process Explorer** should show us two thread ID's, and the two functions **`CreateAndDeleteFiles`** and **`EnumerateProcesses`** won't be seen in the same call stack of a thread, but in a seprate thread.

Here we are able to see the **`EnumerateProcesses`** function in a call stack of a thread.

![image](https://private-user-images.githubusercontent.com/63166600/256336837-666d1f17-6d53-428f-8239-c71db9f1e35a.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTEiLCJleHAiOjE2OTIwOTE0NTMsIm5iZiI6MTY5MjA5MTE1MywicGF0aCI6Ii82MzE2NjYwMC8yNTYzMzY4MzctNjY2ZDFmMTctNmQ1My00MjhmLTgyMzktYzcxZGI5ZjFlMzVhLnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFJV05KWUFYNENTVkVINTNBJTJGMjAyMzA4MTUlMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjMwODE1VDA5MTkxM1omWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPTljZTAwNzZlOGIwNzFhOGY5NTA0N2Q1NmUyZDk5NDhkY2MxNWVlZWZjNjE2MTA4ZjIxYmRiOTkxNDI0NTJlZjImWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.3_p-FPNpqSLH3KiiZhrN6xweg1sZD6svEdBKN0Snxk4)

In this example, we are able to see the **`CreateAndDeleteFiles`** function, but this function call is in a separate call stack of a different thread.

![image](https://private-user-images.githubusercontent.com/63166600/256337372-caf76f31-d1e4-4ffd-9834-f91b5e017e38.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTEiLCJleHAiOjE2OTIwOTE0NTMsIm5iZiI6MTY5MjA5MTE1MywicGF0aCI6Ii82MzE2NjYwMC8yNTYzMzczNzItY2FmNzZmMzEtZDFlNC00ZmZkLTk4MzQtZjkxYjVlMDE3ZTM4LnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFJV05KWUFYNENTVkVINTNBJTJGMjAyMzA4MTUlMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjMwODE1VDA5MTkxM1omWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPTY4OWFkMThmOGE1MDg5MDFiZjZmMzk2YjU5ZDQyMmY4ZWIwNGNiZjQ2OTZlODQ4YjNhZGIxMmIyZDM0Y2I2MGImWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.ceJ6ZljcLY0GBFdQssXvceReB4gHQ8qSuJONpZpbjdY)

In our code, we've designated these two functions to run on separate threads. Therefore, each function will have its own independent call stack. The call stack of a thread is a record of the functions that the thread is executing or has executed. Since these two functions are running on separate threads, each one will have its own call stack. 

Well that's been said, we will never see **`CreateAndDeleteFiles`** and **`EnumerateProcesses`** in the same call stack because they are not being called by the same thread. Each one is the starting point of its own separate thread of execution.

# Theory vs Practice

Let's quote again what we've mentioned previously:

*"A multi-threaded program is a type of software that can do many things at once by using multiple "threads", which are like separate paths of instructions within the program. This way, different parts of the program can work independently and concurrently, making the program **faster** and more efficient."*

One of a solid way to determine the time of execution is calling the **QueryPerformanceCounter** API function. The **QueryPerformanceCounter** function is used to measure the time taken for the execution of the **`CreateAndDeleteFiles`** and **`EnumerateProcesses`** functions.

- **Single-threaded Program**

Let's run the following code that is a single-threaded program and measure how many seconds it took before completing the execution:

```c
#include <windows.h>
#include <tlhelp32.h>
#include <iostream>
#include <string>

#define NUM_FILES 1000
#define NUM_REPEATS_FILES 3
#define NUM_REPEATS_PROCESSES 30

void CreateAndDeleteFiles() {
    for (int j = 0; j < NUM_REPEATS_FILES; j++) {
        for (int i = 0; i < NUM_FILES; i++) {
            std::wstring fileName = L"C:\\Temp\\file_" + std::to_wstring(i) + L".txt";
            HANDLE hFile = CreateFile(
                fileName.c_str(),
                GENERIC_WRITE,
                0,
                NULL,
                CREATE_ALWAYS,
                FILE_ATTRIBUTE_NORMAL,
                NULL
            );

            if (hFile == INVALID_HANDLE_VALUE) {
                std::wcerr << L"CreateFile failed for " << fileName << std::endl;
                continue;
            }
            else {
                std::wcout << L"Created file: " << fileName << std::endl;
            }

            DWORD bytesWritten;
            WriteFile(
                hFile,
                "Hello World",
                11, // number of bytes in "Hello World"
                &bytesWritten,
                NULL
            );

            CloseHandle(hFile);

            if (!DeleteFile(fileName.c_str())) {
                std::wcerr << L"DeleteFile failed for " << fileName << std::endl;
            }
            else {
                std::wcout << L"Deleted file: " << fileName << std::endl;
            }
        }
    }
}

void EnumerateProcesses() {
    for (int j = 0; j < NUM_REPEATS_PROCESSES; j++) {
        HANDLE hSnapshot = CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, 0);
        if (hSnapshot == INVALID_HANDLE_VALUE) {
            std::wcerr << L"CreateToolhelp32Snapshot failed" << std::endl;
            return;
        }

        PROCESSENTRY32W pe32;
        pe32.dwSize = sizeof(pe32);

        if (!Process32FirstW(hSnapshot, &pe32)) {
            std::wcerr << L"Process32First failed" << std::endl;
            CloseHandle(hSnapshot);
            return;
        }

        do {
            std::wcout << L"Process ID: " << pe32.th32ProcessID << ", Process Name: " << pe32.szExeFile << std::endl;
        } while (Process32NextW(hSnapshot, &pe32));

        CloseHandle(hSnapshot);
    }
}

int main() {
    // Get the high-resolution performance counter frequency
    LARGE_INTEGER frequency;
    QueryPerformanceFrequency(&frequency);

    // Get the current counter value at the start of the program
    LARGE_INTEGER start;
    QueryPerformanceCounter(&start);

    CreateAndDeleteFiles();
    EnumerateProcesses();

    // Get the current counter value at the end of the program
    LARGE_INTEGER end;
    QueryPerformanceCounter(&end);

    // Calculate the total execution time in seconds
    double time = static_cast<double>(end.QuadPart - start.QuadPart) / frequency.QuadPart;

    std::wcout << L"Total execution time: " << time << L" seconds" << std::endl;

    return 0;
}
```
In this example, it took around 50 seconds to complete the execution. This will always be different every time we run our code.

![image](https://private-user-images.githubusercontent.com/63166600/256340264-b5682ea8-86d2-40b2-a96a-05dd9415a6d5.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTEiLCJleHAiOjE2OTIwOTE0NTMsIm5iZiI6MTY5MjA5MTE1MywicGF0aCI6Ii82MzE2NjYwMC8yNTYzNDAyNjQtYjU2ODJlYTgtODZkMi00MGIyLWE5NmEtMDVkZDk0MTVhNmQ1LnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFJV05KWUFYNENTVkVINTNBJTJGMjAyMzA4MTUlMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjMwODE1VDA5MTkxM1omWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPTFhNjNmN2QyNmJhZTRmNTM2YTliZGE1OGZkODg1MzgwM2Y4ODkwOWE5NmJlNmUwNjZkNzZkNDk1ZDYyY2QwYmYmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.Kq9HjMXKLDj8hhG_UDfmx96PaO8mFde_e3sd6c1LsYo)

- **Multi-threaded Program**

Let's run the following code that is a multi-threaded program and measure how many seconds it took before completing the execution:

```c
#include <windows.h>
#include <tlhelp32.h>
#include <iostream>
#include <string>
#include <vector>

#define NUM_FILES 1000
#define NUM_REPEATS_FILES 3
#define NUM_REPEATS_PROCESSES 30

DWORD WINAPI CreateAndDeleteFiles(LPVOID lpParam) {
    for (int j = 0; j < NUM_REPEATS_FILES; j++) {
        for (int i = 0; i < NUM_FILES; i++) {
            std::wstring fileName = L"C:\\Temp\\file_" + std::to_wstring(i) + L".txt";
            HANDLE hFile = CreateFile(
                fileName.c_str(),
                GENERIC_WRITE,
                0,
                NULL,
                CREATE_ALWAYS,
                FILE_ATTRIBUTE_NORMAL,
                NULL
            );

            if (hFile == INVALID_HANDLE_VALUE) {
                std::wcerr << L"CreateFile failed for " << fileName << std::endl;
                continue;
            }
            else {
                std::wcout << L"Created file: " << fileName << std::endl;
            }

            DWORD bytesWritten;
            WriteFile(
                hFile,
                "Hello World",
                11, // number of bytes in "Hello World"
                &bytesWritten,
                NULL
            );

            CloseHandle(hFile);

            if (!DeleteFile(fileName.c_str())) {
                std::wcerr << L"DeleteFile failed for " << fileName << std::endl;
            }
            else {
                std::wcout << L"Deleted file: " << fileName << std::endl;
            }
        }
    }

    return 0;
}


DWORD WINAPI EnumerateProcesses(LPVOID lpParam) {
    for (int j = 0; j < NUM_REPEATS_PROCESSES; j++) {
        HANDLE hSnapshot = CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, 0);
        if (hSnapshot == INVALID_HANDLE_VALUE) {
            std::wcerr << L"CreateToolhelp32Snapshot failed" << std::endl;
            return 1;
        }

        PROCESSENTRY32W pe32;
        pe32.dwSize = sizeof(pe32);

        if (!Process32FirstW(hSnapshot, &pe32)) {
            std::wcerr << L"Process32First failed" << std::endl;
            CloseHandle(hSnapshot);
            return 1;
        }

        do {
            std::wcout << L"Process ID: " << pe32.th32ProcessID << ", Process Name: " << pe32.szExeFile << std::endl;
        } while (Process32NextW(hSnapshot, &pe32));

        CloseHandle(hSnapshot);
    }

    return 0;
}

int main() {
    DWORD threadID;
    HANDLE hThreads[2];

    // Get the high-resolution performance counter frequency
    LARGE_INTEGER frequency;
    QueryPerformanceFrequency(&frequency);

    // Get the current counter value at the start of the program
    LARGE_INTEGER start;
    QueryPerformanceCounter(&start);

    hThreads[0] = CreateThread(NULL, 0, CreateAndDeleteFiles, NULL, 0, &threadID);
    hThreads[1] = CreateThread(NULL, 0, EnumerateProcesses, NULL, 0, &threadID);

    WaitForMultipleObjects(2, hThreads, TRUE, INFINITE);

    // Get the current counter value at the end of the program
    LARGE_INTEGER end;
    QueryPerformanceCounter(&end);

    for (int i = 0; i < 2; i++) {
        CloseHandle(hThreads[i]);
    }

    // Calculate the total execution time in seconds
    double time = static_cast<double>(end.QuadPart - start.QuadPart) / frequency.QuadPart;

    std::wcout << L"Total execution time: " << time << L" seconds" << std::endl;

    return 0;
}
```
In this example, it took around 32 seconds to complete the execution. This will always be different every time we run our code.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/daa8226b-a774-43d2-940b-7df89f7048e0)

# Why is the Multi-threaded program faster?

Our multi-threaded program is faster because it uses concurrent execution. This means that it can do multiple tasks at the same time. In our specific program, we have two tasks: creating and deleting files, and listing processes. 

These two tasks don't depend on each other, which means they can run at the same time without causing any problems. This is why we can use multi-threading to make our program run faster: it allows us to do both tasks at the same time, instead of doing one after the other.

# Challenges of Multi-threading

Here are some common challenges when applying multi-threading:

| Challenge | Description |
|-----------|-------------|
| Complexity | Writing multithreaded code is more complex than writing single-threaded code. We need to carefully design our program to avoid problems like race conditions, where two threads try to access or modify the same data at the same time. |
| Synchronization | If threads need to share data, we need to use synchronization constructs like locks or semaphores to ensure that only one thread accesses the shared data at a time. This adds more complexity to our code, and if not done correctly, can lead to issues like deadlocks or starvations. |
| Overhead | Creating and managing threads has a certain amount of overhead. If the tasks we're performing are very small, this overhead can outweigh the benefits of parallel execution. In addition, if we have more threads than our processor has cores, the operating system has to spend time switching between threads, which can also add overhead. |
| Resource contention | If threads need to access the same hardware resource (like writing to the same file), they can end up waiting for each other, reducing the benefits of multithreading. |

