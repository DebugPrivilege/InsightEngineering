# What is Error Handling?

Error handling in programming refers to the techniques used to manage and resolve unexpected problems, or "errors," that arise during the execution of code. It ensures that the program can respond gracefully to potential anomalies, allowing for either controlled failure with meaningful error messages.

# Why is just calling GetLastError() not enough?

**GetLastError()** is a useful function in Windows API programming, as it allows you to retrieve the last error code set by a function failing in the Windows API. However, there are several reasons why relying solely on **GetLastError()** can be insufficient for proper error handling

- **Insufficient Context:** While **GetLastError()** can provide an **error code**, it doesn't tell you anything about the context in which the error occurred. Good error handling should provide as much context as possible to help diagnose and fix the issue.

# Code Sample (1) - Insufficient Context

```c
#include <windows.h>
#include <iostream>
#include <string>
#include <memory>
#include <time.h>

int main() {
    // Seed the random number generator with the current time.
    srand(static_cast<unsigned int>(time(NULL)));

    // Generate a random number.
    int randNum = rand();

    // Convert the random number to a string.
    std::wstring randNumStr = std::to_wstring(randNum);

    // Construct the filename using the random number.
    std::wstring filename = L"C:\\Temp\\testfile_" + randNumStr + L".txt";

    // Attempt to create the file.
    HANDLE tempHandle = CreateFileW(
        filename.c_str(),
        GENERIC_WRITE,
        0,
        NULL,
        CREATE_NEW,
        FILE_ATTRIBUTE_NORMAL,
        NULL
    );

    // If creating the file failed, print an error message and exit.
    if (tempHandle == INVALID_HANDLE_VALUE) {
        std::wcout << L"Unable to create file due to error: " << GetLastError() << "\n";
        return 1;
    }

    // Create a unique_ptr to manage the file handle. The file will automatically be closed
    // when the unique_ptr is destroyed.
    std::unique_ptr<void, decltype(&CloseHandle)> fileHandle(tempHandle, CloseHandle);

    // The data to write to the file.
    const wchar_t data[] = L"Hello, World!";
    DWORD bytesWritten;

    // Attempt to write to the file.
    if (!WriteFile(
        fileHandle.get(),
        data,
        wcslen(data) * sizeof(wchar_t),
        &bytesWritten,
        NULL
    )) {
        // If writing to the file failed, print an error message and exit.
        std::wcout << L"Unable to write to file due to error: " << GetLastError() << "\n";
        return 1;
    }
    else {
        // If writing to the file succeeded, print a success message.
        std::wcout << L"File created and written successfully.\n";
    }

    // The unique_ptr automatically closes the file handle when the program ends,
    // so there's no need to manually close it.

    return 0;
}
```
The primary problem with the error handling in this code is that it only provides the error code without any meaningful description of the error that occurred. When we encounter an error, we're only told an error number like **"3"** with no additional information about what that error number means. This can make it difficult to debug the program if an error occurs.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/400ce6a1-9100-46bd-8044-fc7b8a4dc53e)

# Code Sample (2) - Proper Error Handling

```c
#include <windows.h>
#include <iostream>
#include <string>
#include <memory>
#include <time.h>

// Function to convert error code into a descriptive message string.
std::wstring GetLastErrorAsString(DWORD error)
{
    if (error == 0)
        return L"No error";

    LPWSTR msgBuffer = nullptr;
    size_t size = FormatMessageW(FORMAT_MESSAGE_ALLOCATE_BUFFER | FORMAT_MESSAGE_FROM_SYSTEM | FORMAT_MESSAGE_IGNORE_INSERTS,
        NULL, error, MAKELANGID(LANG_NEUTRAL, SUBLANG_DEFAULT), (LPWSTR)&msgBuffer, 0, NULL);

    std::wstring message(msgBuffer, size);
    LocalFree(msgBuffer);
    return message;
}

int main() {

    // Seed the random number generator with the current time.
    srand(static_cast<unsigned int>(time(NULL)));

    // Generate a random number.
    int randNum = rand();

    // Convert the random number to a string.
    std::wstring randNumStr = std::to_wstring(randNum);

    // Construct the filename using the random number.
    std::wstring filename = L"C:\\Temp\\testfile_" + randNumStr + L".txt";

    // Attempt to create the file.
    HANDLE tempHandle = CreateFileW(
        filename.c_str(),
        GENERIC_WRITE,
        0,
        NULL,
        CREATE_NEW,
        FILE_ATTRIBUTE_NORMAL,
        NULL
    );

    // If creating the file failed, print an error message and exit.
    if (tempHandle == INVALID_HANDLE_VALUE) {
        DWORD error = GetLastError();
        std::wcout << L"Unable to create file due to error: (" << error << L") " << GetLastErrorAsString(error) << "\n";
        return 1;
    }

    // Create a unique_ptr to manage the file handle. The file will automatically be closed
    // when the unique_ptr is destroyed.
    std::unique_ptr<void, decltype(&CloseHandle)> fileHandle(tempHandle, CloseHandle);

    // The data to write to the file.
    const wchar_t data[] = L"Hello, World!";
    DWORD bytesWritten;

    // Attempt to write to the file.
    if (!WriteFile(
        fileHandle.get(),
        data,
        wcslen(data) * sizeof(wchar_t),
        &bytesWritten,
        NULL
    )) {
        // If writing to the file failed, print an error message and exit.
        DWORD error = GetLastError();
        std::wcout << L"Unable to write to file due to error code: (" << error << L") " << GetLastErrorAsString(error) << "\n";
        return 1;
    }
    else {
        // If writing to the file succeeded, print a success message.
        std::wcout << L"File created and written successfully.\n";
    }

    // The unique_ptr automatically closes the file handle when the program ends,
    // so there's no need to manually close it.

    return 0;
}
```

This code has better error handling compared to the previous example. This is because the error messages now include not only the error code but also a string description of the error. This makes debugging much easier because you can now see a meaningful error message when something goes wrong.

The function **GetLastErrorAsString()** is used to convert an error code into a human-readable error message. This function uses the **FormatMessageW()** function to get a string that describes the error code.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/c71210aa-ebe0-49d1-81ab-c8e4ae070d98)

# Code Sample (3) - Error Handling when using Native APIs

When using the Native API, instead of using GetLastError() like you would for the Win32 API, you would use the **NTSTATUS** value returned by the functions to check for errors. **NTSTATUS** is a data type used in Windows operating system's Native API that indicates the result of a system call or kernel routine, providing granular information about success, warning, or error conditions.

Let's cover an example:

```c
#include <windows.h>
#include <winternl.h>
#include <ntstatus.h>
#include <iostream>
#include <string>
#include <memory>
#include <time.h>

// Define the NtCreateFile function.
typedef NTSTATUS(NTAPI* pNtCreateFile)(
    PHANDLE FileHandle,
    ACCESS_MASK DesiredAccess,
    POBJECT_ATTRIBUTES ObjectAttributes,
    PIO_STATUS_BLOCK IoStatusBlock,
    PLARGE_INTEGER AllocationSize,
    ULONG FileAttributes,
    ULONG ShareAccess,
    ULONG CreateDisposition,
    ULONG CreateOptions,
    PVOID EaBuffer,
    ULONG EaLength
    );

// Define the NtWriteFile function.
typedef NTSTATUS(NTAPI* pNtWriteFile)(
    HANDLE FileHandle,
    HANDLE Event,
    PIO_APC_ROUTINE ApcRoutine,
    PVOID ApcContext,
    PIO_STATUS_BLOCK IoStatusBlock,
    PVOID Buffer,
    ULONG Length,
    PLARGE_INTEGER ByteOffset,
    PULONG Key
    );

// Define the RtlInitUnicodeString function.
typedef void (NTAPI* pRtlInitUnicodeString)(PUNICODE_STRING, PCWSTR);

int main() {
    // Load ntdll.dll.
    HMODULE hNtDll = LoadLibraryW(L"ntdll.dll");
    if (!hNtDll) {
        std::wcout << L"Failed to load ntdll.dll.\n";
        return 1;
    }

    // Get the addresses of the NtCreateFile, NtWriteFile, and RtlInitUnicodeString functions.
    pNtCreateFile NtCreateFile = (pNtCreateFile)GetProcAddress(hNtDll, "NtCreateFile");
    pNtWriteFile NtWriteFile = (pNtWriteFile)GetProcAddress(hNtDll, "NtWriteFile");
    pRtlInitUnicodeString RtlInitUnicodeString = (pRtlInitUnicodeString)GetProcAddress(hNtDll, "RtlInitUnicodeString");
    if (!NtCreateFile || !NtWriteFile || !RtlInitUnicodeString) {
        std::wcout << L"Failed to get function addresses.\n";
        FreeLibrary(hNtDll);
        return 1;
    }

    // Seed the random number generator with the current time.
    srand(static_cast<unsigned int>(time(NULL)));

    // Generate a random number.
    int randNum = rand();

    // Convert the random number to a string.
    std::wstring randNumStr = std::to_wstring(randNum);

    // Construct the filename using the random number.
    std::wstring filename = L"C:\\Temp\\testfile_" + randNumStr + L".txt";

    UNICODE_STRING ustr;
    RtlInitUnicodeString(&ustr, filename.c_str());

    OBJECT_ATTRIBUTES objAttr;
    InitializeObjectAttributes(&objAttr, &ustr, OBJ_CASE_INSENSITIVE, NULL, NULL);

    IO_STATUS_BLOCK ioStatusBlock;
    HANDLE fileHandle;

    // Attempt to create the file.
    NTSTATUS status = NtCreateFile(
        &fileHandle,
        GENERIC_WRITE | SYNCHRONIZE,
        &objAttr,
        &ioStatusBlock,
        NULL,
        FILE_ATTRIBUTE_NORMAL,
        0,
        FILE_CREATE,
        FILE_SYNCHRONOUS_IO_NONALERT,
        NULL,
        0
    );

    // If creating the file failed, print an error message and exit.
    if (!NT_SUCCESS(status)) {
        std::wcout << L"Unable to create file due to error: " << status << "\n";
        FreeLibrary(hNtDll);
        return 1;
    }

    // The data to write to the file.
    const wchar_t data[] = L"Hello, World!";
    DWORD bytesWritten;

    // Attempt to write to the file.
    status = NtWriteFile(
        fileHandle,
        NULL,
        NULL,
        NULL,
        &ioStatusBlock,
        (PVOID)data,
        wcslen(data) * sizeof(wchar_t),
        NULL,
        NULL
    );

    // If writing to the file failed, print an error message and exit.
    if (!NT_SUCCESS(status)) {
        std::wcout << L"Unable to write to file due to error: " << status << "\n";
        CloseHandle(fileHandle);
        FreeLibrary(hNtDll);
        return 1;
    }
    else {
        // If writing to the file succeeded, print a success message.
        std::wcout << L"File created and written successfully.\n";
    }

    // Close the file handle and free the library.
    CloseHandle(fileHandle);
    FreeLibrary(hNtDll);

    return 0;
}
```
The primary problem with the error handling in this code is that it only provides the error code without any meaningful description of the error that occurred. When we encounter an error, we're only told an error number like **"-1073741765"** with no additional information about what that error number means. This can make it difficult to debug the program if an error occurs.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/7880b48a-1b40-43f1-b748-cf5fd042ed9d)

# Code Sample (4) - Proper Error Handling when using Native API

```c
#include <windows.h>
#include <winternl.h>
#include <ntstatus.h>
#include <iostream>
#include <string>
#include <memory>
#include <time.h>

// Define the NtCreateFile function.
typedef NTSTATUS(NTAPI* pNtCreateFile)(
    PHANDLE FileHandle,
    ACCESS_MASK DesiredAccess,
    POBJECT_ATTRIBUTES ObjectAttributes,
    PIO_STATUS_BLOCK IoStatusBlock,
    PLARGE_INTEGER AllocationSize,
    ULONG FileAttributes,
    ULONG ShareAccess,
    ULONG CreateDisposition,
    ULONG CreateOptions,
    PVOID EaBuffer,
    ULONG EaLength
    );

// Define the NtWriteFile function.
typedef NTSTATUS(NTAPI* pNtWriteFile)(
    HANDLE FileHandle,
    HANDLE Event,
    PIO_APC_ROUTINE ApcRoutine,
    PVOID ApcContext,
    PIO_STATUS_BLOCK IoStatusBlock,
    PVOID Buffer,
    ULONG Length,
    PLARGE_INTEGER ByteOffset,
    PULONG Key
    );

// Define the RtlNtStatusToDosError function.
typedef ULONG(NTAPI* pRtlNtStatusToDosError)(NTSTATUS Status);

// Function to convert NTSTATUS into a descriptive message string.
std::wstring NtStatusToString(NTSTATUS status, pRtlNtStatusToDosError RtlNtStatusToDosError) {
    DWORD win32Error = RtlNtStatusToDosError(status);

    LPWSTR msgBuffer = nullptr;
    size_t size = FormatMessageW(FORMAT_MESSAGE_ALLOCATE_BUFFER | FORMAT_MESSAGE_FROM_SYSTEM | FORMAT_MESSAGE_IGNORE_INSERTS,
        NULL, win32Error, MAKELANGID(LANG_NEUTRAL, SUBLANG_DEFAULT), (LPWSTR)&msgBuffer, 0, NULL);

    std::wstring message(msgBuffer, size);
    LocalFree(msgBuffer);
    return message;
}

int main() {
    // Load ntdll.dll.
    HMODULE hNtDll = LoadLibraryW(L"ntdll.dll");
    if (!hNtDll) {
        std::wcout << L"Failed to load ntdll.dll.\n";
        return 1;
    }

    // Get the addresses of the NtCreateFile, NtWriteFile and RtlNtStatusToDosError functions.
    pNtCreateFile NtCreateFile = (pNtCreateFile)GetProcAddress(hNtDll, "NtCreateFile");
    pNtWriteFile NtWriteFile = (pNtWriteFile)GetProcAddress(hNtDll, "NtWriteFile");
    pRtlNtStatusToDosError RtlNtStatusToDosError = (pRtlNtStatusToDosError)GetProcAddress(hNtDll, "RtlNtStatusToDosError");

    if (!NtCreateFile || !NtWriteFile || !RtlNtStatusToDosError) {
        std::wcout << L"Failed to get function addresses.\n";
        FreeLibrary(hNtDll);
        return 1;
    }

    // Seed the random number generator with the current time.
    srand(static_cast<unsigned int>(time(NULL)));

    // Generate a random number.
    int randNum = rand();

    // Convert the random number to a string.
    std::wstring randNumStr = std::to_wstring(randNum);

    // Construct the filename using the random number.
    std::wstring filename = L"C:\\Temp\\testfile_" + randNumStr + L".txt";

    UNICODE_STRING ustr;
    ustr.Buffer = &filename[0];
    ustr.Length = ustr.MaximumLength = filename.length() * sizeof(wchar_t);

    OBJECT_ATTRIBUTES objAttr;
    InitializeObjectAttributes(&objAttr, &ustr, OBJ_CASE_INSENSITIVE, NULL, NULL);

    IO_STATUS_BLOCK ioStatusBlock;
    HANDLE fileHandle;

    // Attempt to create the file.
    NTSTATUS status = NtCreateFile(
        &fileHandle,
        GENERIC_WRITE,
        &objAttr,
        &ioStatusBlock,
        NULL,
        FILE_ATTRIBUTE_NORMAL,
        0,
        FILE_CREATE,
        FILE_SYNCHRONOUS_IO_NONALERT,
        NULL,
        0
    );

    // If creating the file failed, print an error message and exit.
    if (!NT_SUCCESS(status)) {
        std::wcout << L"Unable to create file due to error: 0x" << std::hex << status << L" " << NtStatusToString(status, RtlNtStatusToDosError) << "\n";
        FreeLibrary(hNtDll);
        return 1;
    }

    // The data to write to the file.
    const wchar_t data[] = L"Hello, World!";

    // Attempt to write to the file.
    status = NtWriteFile(
        fileHandle,
        NULL,
        NULL,
        NULL,
        &ioStatusBlock,
        (PVOID)data,
        wcslen(data) * sizeof(wchar_t),
        NULL,
        NULL
    );

    // If writing to the file failed, print an error message and exit.
    if (!NT_SUCCESS(status)) {
        std::wcout << L"Unable to write to file due to error: 0x" << std::hex << status << L" " << NtStatusToString(status, RtlNtStatusToDosError) << "\n";
        CloseHandle(fileHandle);
        FreeLibrary(hNtDll);
        return 1;
    }
    else {
        // If writing to the file succeeded, print a success message.
        std::wcout << L"File created and written successfully.\n";
    }

    // Close the file handle and free the library.
    CloseHandle(fileHandle);
    FreeLibrary(hNtDll);

    return 0;
}
```

This code has better error handling than the previous one because it's not only detecting when errors occur, but also providing detailed information about those errors to help diagnose the issue. Specifically, it uses the **NtStatusToString** function to convert **NTSTATUS** error codes into a human-readable format using **RtlNtStatusToDosError** and **FormatMessageW**.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/f4cb5dce-e145-4653-b4e1-3cd228af8262)

# Use Hexadecimal for error codes when calling Native APIs

The **ntstatus.h** is a header file provided by the Windows Driver Kit (WDK) and the Windows SDK, which defines constants for **NTSTATUS** codes used in Windows. These **NTSTATUS** codes include information about the severity, facility, and specific condition of any errors, warnings, or other operational results. All of these error codes are documented as hexadecimal, so it makes sense to use hexadecimal numbers for error handling.

**Example:**

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/25b8142d-c649-48c5-a929-6c7957feec18)

