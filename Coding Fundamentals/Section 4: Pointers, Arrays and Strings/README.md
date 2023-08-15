# What is a Pointer?

A **pointer** is a variable that stores the memory address of another variable as its value. A pointer variable points to a data type **(like int)** of the same type, and is created with the * operator.

Let's demonstrate this with a simple code snippet:

```c
#include <stdio.h>

int main() {
    int num = 5;    // Declare an integer
    int* p;         // Declare a pointer to an integer

    p = &num;       // Assign the address of num to the pointer p

    printf("Before change, value of num: %d\n", num);    // Print the initial value of num

    *p = 10;        // Change the value at the memory address p is pointing to

    printf("After change, value of num: %d\n", num);     // Print the value of num after change

    return 0;
}
```
This shows that you can use a pointer to change the value of a variable indirectly, by changing the value at the memory address the pointer is pointing to.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/ba3a09fb-4499-4f2c-9236-f164de817af7)


Let's break down different parts of this code:

**Declaration:**

Declaration in programming specifies the type and name of a variable, function, or any identifier

- We are declaring a variable called **num** and assigning it the integer value **5**

```c
int num = 5;
```
- At this line, we are declaring a pointer to an integer. The * in front of **p** indicates that **p** is a pointer.

```c
int* p; 
```

**Address Operator:**

The address operator **&** is used to get the memory address of a variable. Here, **&num** gets the memory address of the variable **num**, and this address is assigned to the pointer **p**.

```c
p = &num;
```

**Dereference Operator:**

The Dereference operator * gives us access to the value at the memory location pointed to by the pointer. Here, ***p** is used to set a new value **10** at the memory location that **p** points to. This changes the value of **num** indirectly through the pointer **p**.

```c
*p = 10;
```

Let's cover another example regarding pointers:

```c
#include <stdio.h>

int main() {
    int number = 30;    // Declare an integer variable
    int *p;             // Declare a pointer to an integer

    p = &number;        // Assign the address of number to the pointer p

    printf("Address of number variable: %p\n", &number);  // Print the address of number
    printf("Address stored in p variable: %p\n", p);      // Print the address stored in pointer p
    printf("Value at address stored in p variable: %d\n", *p);  // Dereference pointer p to get the value at that address

    return 0;
}
```

When we run this code, it prints the memory address of the **number** variable, the **address** stored in the **p** variable (which should be the same as the address of number), and the value at the address that **p** points to (which should be the value of number).

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/476551b3-45cf-4c48-847a-5ad47f399e1c)


At this last example, we initially establish a pointer named **message** to a string *"Hello, World!"*.  We are introducing a new pointer **'p'** and make it reference the same data that the **message** pointer is connected to. 

```c
#include <windows.h>

int main()
{
    const wchar_t* message = L"Hello, World!";
    const wchar_t** p = &message; // Declare a pointer to a pointer to a wide string

    // MessageBox function is called using the pointer
    MessageBox(NULL, *p, L"Message Box", MB_OK | MB_ICONINFORMATION);

    return 0;
}
```
Let's break down the code:

This line declares a pointer to a wide string (wchar_t*) named **message** and initializes it to point to the wide string *"Hello, World!"*. **const** keyword indicates that the string **message** cannot be changed.

```
const wchar_t* message = L"Hello, World!";
```

This line declares a pointer **'p'** that points to a pointer to a wide string, and assigns it the address of **message**.

```
const wchar_t** p = &message;
```

The pointer in this code is **'p'**, and it's used to pass the string *"Hello, World!"* to the **MessageBox** function.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/6962dbce-c898-4ce3-8171-66abd3462e0b)


# Arrays

An array is a fundamental data structure that is a collection of similar data elements stored at contiguous memory locations. Each data element in the array can be accessed directly using its index number.

You may now have additional questions, so let's try cover different examples to gather a better understanding.

```c
#include <windows.h>
#include <stdio.h>

int main() {
    // Define the filenames as an array of wide characters
    wchar_t filenames[3][10] = { L"file1.txt", L"file2.txt", L"file3.txt" };

    for (int i = 0; i < 3; i++) {
        // Use the CreateFile function to create a new file
        HANDLE hFile = CreateFile(
            filenames[i],             // name of the file
            GENERIC_WRITE,            // open for writing
            0,                        // do not share
            NULL,                     // default security
            CREATE_NEW,               // create new file only
            FILE_ATTRIBUTE_NORMAL,    // normal file
            NULL                      // no attr. template
        );

        // Check if the file was created successfully
        if (hFile == INVALID_HANDLE_VALUE) {
            wprintf(L"Unable to create file %ls\n", filenames[i]);
        }
        else {
            wprintf(L"File %ls created successfully\n", filenames[i]);

            // Close the file handle
            CloseHandle(hFile);
        }
    }

    return 0;
}

```

The variable **'i'** inside the **'for'** loop is used to access each element of the **filenames** array. On each iteration of the loop, **'i'** increments and the next filename is accessed using **filenames[i]**. **filenames[0]** refers to **"file1.txt"**, **filenames[1]** refers to **"file2.txt"**, and **filenames[2]** refers to **"file3.txt"**. These are the index numbers in this code.

```
wchar_t filenames[3][10] = { L"file1.txt", L"file2.txt", L"file3.txt" };
```
Let's break it down:

**filenames** is the name of the array and **[3][10]** is the dimension of the array. **3** is the number of the strings that the array can hold. **10** represents maximum number of wide characters that each string can contain.

This the initialization of the array. It sets the three strings that the array will initially contain.

```
{ L"file1.txt", L"file2.txt", L"file3.txt" }
```

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/5fb2a9ee-ab12-4414-aa50-8dd4f77eadf4)


Let's cover another example. This code demonstrates using an array to store the names of the programs (notepad.exe, cmd.exe, and calc.exe) and then uses **CreateProcess** to start each program at the same time.

```c
#include <windows.h>
#include <stdio.h>

int main() {
    wchar_t apps[3][MAX_PATH] = { L"C:\\Windows\\System32\\notepad.exe", L"C:\\Windows\\System32\\cmd.exe", L"C:\\Windows\\System32\\calc.exe" };
    STARTUPINFO si;
    PROCESS_INFORMATION pi;

    // Set size of the structures
    ZeroMemory(&si, sizeof(si));
    si.cb = sizeof(si);
    ZeroMemory(&pi, sizeof(pi));

    // Start each application specified in the array
    for (int i = 0; i < 3; i++) {
        if (!CreateProcessW(NULL,   // No module name (use command line)
            apps[i],                 // Command line
            NULL,           // Process handle not inheritable
            NULL,           // Thread handle not inheritable
            FALSE,          // Set handle inheritance to FALSE
            0,              // No creation flags
            NULL,           // Use parent's environment block
            NULL,           // Use parent's starting directory 
            &si,            // Pointer to STARTUPINFO structure
            &pi)           // Pointer to PROCESS_INFORMATION structure
            )
        {
            printf("CreateProcess failed (%d).\n", GetLastError());
            return 1;
        }
        // Close process and thread handles. 
        CloseHandle(pi.hProcess);
        CloseHandle(pi.hThread);
    }
    return 0;
}
```
![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/798677de-1cd9-404b-aa5a-c352f0230a58)


# Strings

A string is a type of data that represents a sequence of characters. Each programming language has a specific way to represent and manipulate strings. Typically, a string is created by enclosing characters in either single or double quotes.

Let's break down the previous code snippet:

```c
#include <windows.h>
#include <stdio.h>

int main() {
    wchar_t apps[3][MAX_PATH] = { L"C:\\Windows\\System32\\notepad.exe", L"C:\\Windows\\System32\\cmd.exe", L"C:\\Windows\\System32\\calc.exe" };
    STARTUPINFO si;
    PROCESS_INFORMATION pi;

    // Set size of the structures
    ZeroMemory(&si, sizeof(si));
    si.cb = sizeof(si);
    ZeroMemory(&pi, sizeof(pi));

    // Start each application specified in the array
    for (int i = 0; i < 3; i++) {
        if (!CreateProcessW(NULL,   // No module name (use command line)
            apps[i],                 // Command line
            NULL,           // Process handle not inheritable
            NULL,           // Thread handle not inheritable
            FALSE,          // Set handle inheritance to FALSE
            0,              // No creation flags
            NULL,           // Use parent's environment block
            NULL,           // Use parent's starting directory 
            &si,            // Pointer to STARTUPINFO structure
            &pi)           // Pointer to PROCESS_INFORMATION structure
            )
        {
            printf("CreateProcess failed (%d).\n", GetLastError());
            return 1;
        }
        // Close process and thread handles. 
        CloseHandle(pi.hProcess);
        CloseHandle(pi.hThread);
    }
    return 0;
}
```
The strings in this code are the following:

- **`"C:\\Windows\\System32\\notepad.exe"`** - This is a wide string literal representing the path to the notepad.exe program The L prefix indicates it is a wide string.

- **`"C:\\Windows\\System32\\cmd.exe"`** - This is a wide string literal representing the path to the cmd.exe program.

- **`"C:\\Windows\\System32\\calc.exe"`** - This is a wide string literal representing the path to the calc.exe program.

- **`"CreateProcess failed (%d).\n"`** - This is a string literal used in the printf function to output an error message if the CreateProcessW function fails.
