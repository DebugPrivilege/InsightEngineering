# What is Memory Allocation?

**READ ME:** The goal of this to have a 'general' understanding of the concept. 

Memory allocation in programming refers to the process of reserving memory space during the execution of a program, enabling efficient utilization and management of system resources. 

Memory allocation can be categorized into two types:

- **Dynamic Memory Allocation**

This method involves allocating memory during runtime, i.e., while the program is running. The exact amount of memory required isn't determined until the program is executed. Languages like C provide functions such as **malloc()**, **calloc()**, **realloc()**, and **free()** for dynamic memory allocation and deallocation. This type of allocation is critical when the memory requirement is not predetermined. It's frequently used in scenarios such as string manipulation, handling file data, data processing, buffer implementation, and managing command line arguments.

- **Static Memory Allocation**

This type of memory allocation occurs at compile time, meaning the memory size required by the program is predetermined before its execution. The allocated memory cannot be resized during runtime. Static memory allocation is performed when you declare variables, arrays, and functions.

During static memory allocation, the allocated memory persists for the entire duration of the program. Therefore, there's no need for the programmer to manage memory deallocation, as this is handled automatically by the system.

# Program Memory Layout

```plaintext
High Address

+------------------+
|                  |
|       Stack      |  ← SP (Stack grows down towards lower addresses)
|       ...        |
+------------------+ 
|                  |
|       Heap       |  ← Heap grows up towards higher addresses
|       ...        |
+------------------+
|                  |
| Uninitialized    |
| Data Segment (BSS)|
+------------------+
|                  |
| Initialized      |
| Data Segment     |
+------------------+
|                  |
|   Constant Data  |
|   Segment (.rodata) |
+------------------+
|                  |
|   Text Segment   |
| (Machine code)   |
|                  |
+------------------+

Low Address
```

**Stack Segment:** This is where local variables and function call information are stored. The stack grows downwards from high memory addresses to lower ones. The Stack Pointer (SP) points to the top of the stack.

**Heap Segment:** This is a region of memory used for dynamic memory allocation (through malloc, calloc, etc.). Unlike the stack, the heap grows upwards from low memory addresses to higher ones.

**Uninitialized Data Segment (BSS)**: This contains global and static variables that are not explicitly initialized by the programmer. These variables are automatically initialized to zero.

**Initialized Data Segment**: Contains global and static variables that are explicitly initialized by the programmer.

**Constant Data Segment (.rodata)**: This is a read-only segment that stores constant data, such as string literals.

**Text Segment**: This is a read-only segment that stores the machine code of the compiled program.

# Static Memory Allocation

This type of memory allocation occurs at compile time, meaning the memory size required by the program is **predetermined** before its execution.

Let's cover this with a code snippet:

```c
#include <windows.h>
#include <stdio.h>

int main() {
    // Define the filenames as an array of wide characters
    wchar_t filenames[3][50];

    wprintf(L"Enter the names of three files (including extension):\n");

    for (int i = 0; i < 3; i++) {
        // Prompt the user to enter the filename
        wprintf(L"Filename %d: ", i + 1);
        wscanf_s(L"%s", filenames[i], (unsigned int)(sizeof(filenames[i]) / sizeof(filenames[i][0])));

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

Memory allocation is a fundamental aspect that is required to hold data. When we are defining a variable or an array in a program, we're asking the system to allocate a certain amount of memory that can hold the values for these entities.

At this example, we asking the system to allocate memory to hold the filenames entered by the user. If we didn't allocate this memory, there would be no place to store the filenames, and the program wouldn't function correctly. Let's break down the following line of code:

```c
wchar_t filenames[3][50];
```

- **`wchar_t`**: This is a data type that represents a wide character. It's often used for unicode characters, and is used here because filenames can contain a wide range of characters.
- **`filenames:`** This is the name of the array we are defining. 
- **`[3]:`** This tells the system to allocate memory for 3 wchar_t arrays. This is because you want to get 3 filenames from the user.
- **`[50]:`** This tells the system that each of these 3 wchar_t arrays should have space for 50 wide characters.

Here we can see that we have created 3 files successfully on disk.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/aa91ff18-13d7-4d0c-ac08-29bc9e985b74)


![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/a4c01687-e41a-4e25-82f8-d8f033befebf)


# Improper Memory Allocation

An improper memory allocation could occur in your code if you allocate insufficient space. In this example, if a user tries to enter a filename that's longer than the allocated size, you'll get a buffer overflow.

Let's demonstrate this with a simple example:

```c
#include <windows.h>
#include <stdio.h>

int main() {
    // Define the filenames as an array of wide characters
    // with insufficient space for typical filenames
    wchar_t filenames[3][2];

    wprintf(L"Enter the names of three files (including extension):\n");

    for (int i = 0; i < 3; i++) {
        // Prompt the user to enter the filename
        wprintf(L"Filename %d: ", i + 1);
        wscanf_s(L"%s", filenames[i], (unsigned int)(sizeof(filenames[i]) / sizeof(filenames[i][0])));

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

When we run our code again, there will be an error. We declared the array to hold three filenames, each of which can contain only **2** wide characters (including the null terminator). This is insufficient to hold a typical filename. **`"file1.txt"`** is much longer than 1 character (the maximum length that can be safely stored in filenames[i]). As a result, **wscanf_s** overflows the buffer, and writes past the end of **`filenames[i]`**.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/fca277bf-e164-42fa-9ca0-9c8a993aed6a)


# Dynamic Memory Allocation

This method involves allocating memory during runtime, i.e., while the program is running. The exact amount of memory required isn't determined until the program is executed. Languages like C provide functions such as **malloc()**, **calloc()**, **realloc()**, and **free()** for dynamic memory allocation and deallocation.

Let's demonstrate this with a code snippet:

```c
#include <windows.h>
#include <stdio.h>
#include <wchar.h>

int main() {
    // Define the filenames as an array of wide character pointers
    wchar_t* filenames[3];

    wprintf(L"Enter the names of three files (including extension):\n");

    for (int i = 0; i < 3; i++) {
        // Allocate initial buffer size of 1 character
        size_t bufferSize = 1;
        filenames[i] = malloc(bufferSize * sizeof(wchar_t));
        if (filenames[i] == NULL) {
            wprintf(L"Memory allocation failed\n");
            return 1;
        }

        // Read the filename
        wprintf(L"Filename %d: ", i + 1);
        wint_t ch;
        size_t length = 0;
        while ((ch = fgetwc(stdin)) != L'\n' && ch != WEOF) {
            filenames[i][length++] = (wchar_t)ch;
            if (length >= bufferSize) {
                bufferSize *= 2;
                wchar_t* temp = realloc(filenames[i], bufferSize * sizeof(wchar_t));
                if (temp == NULL) {
                    wprintf(L"Memory allocation failed\n");
                    free(filenames[i]);
                    return 1;
                }
                filenames[i] = temp;
            }
        }

        // Null-terminate the filename
        filenames[i][length] = L'\0';

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

        // Free the memory allocated for the filename
        free(filenames[i]);
    }

    return 0;
}
```

This code is a bit more complicated comparing the previous code snippet that was used as example for static memory allocation. Let's break down this code snippet in pieces:

A for loop is used to iterate three times, corresponding to the three files to be entered.

```c
    for (int i = 0; i < 3; i++) {
```

Inside the loop, an initial buffer of size 1 character is allocated using **malloc**. If the memory allocation fails, an error message is displayed, and the program exits.

```c
        // Allocate initial buffer size of 1 character
        size_t bufferSize = 1;
        filenames[i] = malloc(bufferSize * sizeof(wchar_t));
        if (filenames[i] == NULL) {
            wprintf(L"Memory allocation failed\n");
            return 1;
        }
```

The program prompts the user to enter the filename. It then reads the characters of each filename one by one using **`fgetwc(stdin)`** in a loop. The characters are stored in the **`filenames[i]`** buffer, which dynamically grows as needed using **realloc**. If the memory reallocation fails, an error message is displayed, and the program exits.

```c
        // Read the filename
        wprintf(L"Filename %d: ", i + 1);
        wint_t ch;
        size_t length = 0;
        while ((ch = fgetwc(stdin)) != L'\n' && ch != WEOF) {
            filenames[i][length++] = (wchar_t)ch;
            if (length >= bufferSize) {
                bufferSize *= 2;
                wchar_t* temp = realloc(filenames[i], bufferSize * sizeof(wchar_t));
                if (temp == NULL) {
                    wprintf(L"Memory allocation failed\n");
                    free(filenames[i]);
                    return 1;
                }
                filenames[i] = temp;
            }
        }
```

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/10ff1fa8-af0e-4ed4-ae4b-f7a3e122b8d3)


In the code with dynamic memory allocation, the filenames are stored as an array of wide character pointers (wchar_t* filenames[3]). During runtime, memory is dynamically allocated for each filename using **malloc** and **realloc** as needed. The buffer size starts with **1** character and grows dynamically when necessary to accommodate the user input. This allows for flexibility in handling filenames of varying lengths.

- Why is the **bufferSize** set to **1**?

The **bufferSize** refers to the size of the buffer allocated for storing the characters of the filename entered by the user. It represents the capacity of the buffer to hold characters.

Initially, the **bufferSize** is set to a small value, such as **1**. This means that the buffer can initially hold only one character. As the program reads characters from the input and stores them in the buffer, it checks if the buffer is full (i.e., the number of characters entered exceeds the bufferSize).

When the length becomes equal to the bufferSize, it means that the buffer is full and cannot accommodate additional characters. At this point, the program doubles the **bufferSize** by using the **realloc** function. The **realloc** function allows for resizing the allocated memory block. By doubling the **bufferSize**, the program ensures that the buffer has enough space to hold additional characters. 

# What happens when I enter a filename?

What happens if we type in **'security.txt'**?

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/6bd36d33-0e16-4384-bd65-10f39bc23b1c)


This is a step-by-step explanation on how we are reallocating memory. Since explaining what a 'buffer' can be difficult, so we have to put it into different pieces.

1. You are prompted to enter the first filename. You start typing **security.txt**.

2. You press **s**. fgetwc(stdin) reads the **s** from the standard input and places it into the first position of the buffer pointed to by **`filenames[i]`**. The length is now **1**.

3. Since the length is now equal to **bufferSize** (which was initially set to 1), the buffer is full. The program then reallocates the buffer, doubling its size to **2** using **realloc()**.

4. You press **e**. fgetwc(stdin) reads the **e** from the standard input and places it into the second position of the buffer. The length is now **2**.

5. Again, since the length is now equal to **bufferSize**, the buffer is full. The program reallocates the buffer, doubling its size to **4**.

6. You press **c**. fgetwc(stdin) reads the **c** from the standard input and places it into the third position of the buffer. The length is now **3**. Since length is less than bufferSize (which is now **4**), no reallocation occurs this time.

7. You press **u**. fgetwc(stdin) reads the **u** and places it into the fourth position of the buffer. The length is now **4**, which equals **bufferSize**, so the buffer is reallocated again, doubling its size to **8**.

8. You press **r**. fgetwc(stdin) reads the **r** from the standard input and places it into the fifth position of the buffer. The length is now **5**. Since length is less than **bufferSize** (which is now **8**), no reallocation occurs at this time.

9. You press **i**. fgetwc(stdin) reads the **i** and places it into the sixth position of the buffer. The length is now **6**, which is still less than **bufferSize** **(8)**, so the buffer isn't reallocated yet.

10. You press **t**. fgetwc(stdin) reads the **t** and places it into the seventh position of the buffer. The length is now **7**. Still, the length is less than **bufferSize** **(8)**, so no reallocation occurs.

11. You press **y**. fgetwc(stdin) reads the **y** and places it into the eighth position of the buffer. The length is now **8**, which equals bufferSize, so the buffer is full. The program reallocates the buffer, doubling its size to **16**.

12. You press **.**. fgetwc(stdin) reads the **.** and places it into the ninth position of the buffer. The length is now **9**, which is less than bufferSize **(16)**, so the buffer isn't reallocated.

13. You press **t**. fgetwc(stdin) reads the **t** and places it into the tenth position of the buffer. The length is now 10, which is still less than bufferSize **(16)**, so no reallocation occurs.

14. You press **x**. fgetwc(stdin) reads the **x** and places it into the eleventh position of the buffer. The length is now **11**, which is still less than **bufferSize** **(16)**, so no reallocation occurs.

15. Finally, you press **t**. fgetwc(stdin) reads the **t** and places it into the twelfth position of the buffer. The length is now **12**. Still, the length is less than bufferSize **(16)**, so no reallocation occurs.

16. You then press Enter. fgetwc(stdin) reads the newline character \n, causing the loop to exit.

17. The program null-terminates the string in the buffer by appending a \0 character.

18. The **CreateFile** function is called with the filename you entered **(security.txt)**. It attempts to create a new file with that name. If successful, it outputs a message saying the file was created successfully; if not, it outputs a message saying the file could not be created.

19. The program then frees the memory allocated for the filename, and the process would repeat for the next filename if there were any.

# Some differences between Static & Dynamic Memory Allocation

Here is a simple comparison between static and dynamic memory allocation:

|                    | Static Memory Allocation | Dynamic Memory Allocation |
|--------------------|-------------------------|---------------------------|
| When is it allocated? | At compile time        | At runtime                |
| Size               | Fixed                   | Variable                  |
| Flexibility        | Less                    | More                      |
| Efficiency         | More                    | Less                      |
| Memory location    | Stack or Data Segment   | Heap                      |
| Modification       | Not possible            | Possible                  |
| Life               | Exists until end of program | Exists until deallocated  |


# What is a Memory Leak?

A memory leak in programming is like leaving the water running in your sink. If you keep the water on (keep using memory) but never turn it off (never free up that memory), eventually, you'll run out of water (run out of memory).

A memory leak happens when a program uses memory but doesn't give it back when it's done. Over time, this can cause the program to slow down or even stop working because it runs out of memory. 

Let's demonstrate two common examples:

- **Not Freeing Memory**

This is the most common cause. When you allocate memory using functions like **malloc()**, **calloc()**, or **realloc()** and forget to free it when you're done will cause a memory leak

The root cause of the memory leak in this code is that the dynamically allocated memory for the **`filename`** variable is not being freed, even after it's no longer needed. This leaves memory that's no longer in use by your program but still allocated, causing a memory leak.

```c
#include <windows.h>
#include <stdio.h>
#include <time.h>

int main() {
    srand(time(NULL));  // Seed the random number generator with the current time

    // Allocate memory for the file name
    LPCWSTR filename = (wchar_t*)malloc(100 * sizeof(wchar_t));

    // Generate a random number
    int randNum = rand();

    // Create the file name with the random number
    swprintf(filename, 100, L"C:\\Temp\\testfile_%d.txt", randNum);

    // Create the file
    HANDLE fileHandle = CreateFile(
        filename,                // name of the file
        GENERIC_WRITE,           // open for writing
        0,                       // do not share
        NULL,                    // default security
        CREATE_NEW,              // create new file only
        FILE_ATTRIBUTE_NORMAL,   // normal file
        NULL);                   // no attr. template

    if (fileHandle == INVALID_HANDLE_VALUE) {
        wprintf(L"Unable to create file due to error %lu\n", GetLastError());
        // Even if the file creation fails, we should free the memory
        // free(filename);
        return 1;
    }
    else {
        wprintf(L"File created successfully.\n");
    }

    // Close the file handle
    CloseHandle(fileHandle);

    // Here we're not freeing the filename pointer, leading to a memory leak
    // Correct thing to do would be: free(filename);

    // Wait for a character input from the user before the program ends
    printf("Press any key to exit...\n");
    getchar();

    return 0;
}
```

- **Dangling Pointer**

This memory leak has occurred because we allocated a block of memory for the **`filename`** variable. However, before freeing that memory, we redirected the **`filename`** pointer to another string, creating a dangling pointer. We lost our reference to the initially allocated memory, making it impossible for us to release it back to the system.

```c
#include <windows.h>
#include <stdio.h>
#include <time.h>

int main() {
    srand(time(NULL));  // Seed the random number generator with the current time

    // Allocate memory for the file name
    LPCWSTR filename = (wchar_t*)malloc(100 * sizeof(wchar_t));

    // Generate a random number
    int randNum = rand();

    // Create the file name with the random number
    swprintf(filename, 100, L"C:\\Temp\\testfile_%d.txt", randNum);

    // Create the file
    HANDLE fileHandle = CreateFile(
        filename,                // name of the file
        GENERIC_WRITE,           // open for writing
        0,                       // do not share
        NULL,                    // default security
        CREATE_NEW,              // create new file only
        FILE_ATTRIBUTE_NORMAL,   // normal file
        NULL);                   // no attr. template

    if (fileHandle == INVALID_HANDLE_VALUE) {
        wprintf(L"Unable to create file due to error %lu\n", GetLastError());
        return 1;
    }
    else {
        wprintf(L"File created successfully.\n");

        // Prepare the text to be written to the file
        const char* text = "Hello, World!";
        DWORD bytesWritten;

        // Write the text to the file
        WriteFile(
            fileHandle,
            text,
            strlen(text),    // write only how many bytes are in the text
            &bytesWritten,
            NULL
        );

        if (bytesWritten < strlen(text)) {
            wprintf(L"Failed to write to the file.\n");
            // If writing to the file fails, close the handle and exit
            CloseHandle(fileHandle);
            return 1;
        }
        else {
            wprintf(L"Text written to the file successfully.\n");
        }
    }

    // Close the file handle
    CloseHandle(fileHandle);

    // Overwrite the filename pointer with a new value
    filename = L"New Value";  // Now the original memory block is inaccessible

    // This would lead to undefined behavior because we're trying to free a string literal,
    // not a block of memory allocated with malloc
    free(filename);

    // Wait for a character input from the user before the program ends
    printf("Press any key to exit...\n");
    getchar();

    return 0;
}
```

Here's the problematic part of the code:

```c
// Overwrite the filename pointer with a new value
filename = L"New Value";  // Now the original memory block is inaccessible
```

After this line, the memory that was allocated with **malloc** is still allocated, but we have no way to free it because the **`filename`** pointer now points to the string literal **`"New Value"`** instead of the allocated memory.

When we try to run this code, we will see this:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/eb0f9109-3e5e-4c39-b83f-71285d2efed0)

