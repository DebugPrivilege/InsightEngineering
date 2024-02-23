# What are Control Flow Statements?

Control flow in programming refers to the order in which the program's code executes. This control flow can be manipulated using control flow statements, which include conditional statements, loop statements, and jump statements. These statements help in decision making, looping, and so on.

| **Type of Control Flow Statement** | **Keywords in C** | **Description** |
|:----------------------------------:|:-----------------:|:---------------:|
| Conditional Statements             | `if`              | It checks if an expression is true. If it is, then it executes the block of code inside the if statement. |
|                                    | `else if`         | It is used after an `if` or another `else if` to check multiple expressions. If the condition in `if` is false, then it checks the condition inside the `else if`. |
|                                    | `else`            | It is used after an `if` or `else if`. If all conditions above are false, then the code inside the else block is executed. |
|                                    | `switch`          | It is used to select one of many code blocks to be executed. |
| Loop Statements                    | `while`           | It executes a block of code as long as the condition is true. |
|                                    | `do while`        | It is similar to `while`, but it checks the condition after the loop has executed. This guarantees that the loop will execute at least once. |
|                                    | `for`             | It is often used when you know beforehand how many times you want to execute a block of code. |
| Jump Statements                    | `goto`            | It transfers control to a labeled statement within the same function.  |




# Conditional Statements

Conditional statements in programming are used to perform different actions based on whether a certain condition or set of conditions evaluates to true or false. 

Let's start with the **`if-else`** statements and demonstrate it with code. 

**`if`** is a conditional statement, which evaluates an expression. If the expression inside the **`if`** evaluates to **true**, the block of code inside the **`if`** statement is executed. If the expression is **false**, the **`if`** block is skipped.

**`else`** is used in conjunction with **`if`**. The **'else'** block is executed if the condition in the **`if`** statement is false.

An **`else if`** statement in the otherhand is a way to check multiple, distinct conditions where each condition is checked only if all previous conditions in the code were false. If any condition is **true**, the code associated with that condition is executed and no further conditions are checked.

```c
#include <stdio.h>

int main() {
    // Define an integer to store the user's input
    int num;

    // Prompt the user to enter a number
    printf("Enter a number: ");

    // Use scanf_s to read the user's input into the num variable
    // scanf_s is a safer version of scanf provided by Microsoft's C library.
    // Since we're reading an int, we don't need to specify a buffer size.
    scanf_s("%d", &num);

    // Check the number entered by the user
    if (num > 0) {
        // If the number is greater than 0, print that it's positive
        printf("The number is positive.\n");
    }
    else if (num < 0) {
        // If the number is less than 0, print that it's negative
        printf("The number is negative.\n");
    }
    else {
        // If the number is neither greater than 0 nor less than 0, it must be 0
        printf("The number is zero.\n");
    }

    return 0;
}
```
If we insert the integer **12**. It will return that it is a positive number.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/c771d7dc-2c78-4c81-9eff-647d9bb9f294)


The second example will be now using **-12** which returns the **`else-if`** statement, indicating that the number is negative.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/94ecc814-8963-48c0-9420-0b204315cf2d)


The last example will show us that it's neither a positive or negative number, so here is where the **`else`** statement will kick in.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/797afbe2-55bd-46c1-a2c1-cf20992fcbf6)


The **switch** statement in C is a type of control flow mechanism that allows you to perform different actions based on the value of an integer or character expression.

Let's demonstrate the **switch** statement with some code snippet:

```c
#include <stdio.h>

int main() {
    int choice;

    printf("Enter a number between 1 and 3: ");
    scanf_s("%d", &choice);

    switch (choice) {
    case 1:
        printf("You entered one.\n");
        break;
    case 2:
        printf("You entered two.\n");
        break;
    case 3:
        printf("You entered three.\n");
        break;
    default:
        printf("Invalid choice!\n");
    }

    return 0;
}
```
Inside the **switch** block, the **case** keyword is used to specify different blocks of code that should be executed if the **switch** expression **(choice)** equals a certain value. 

At the end of each **case** block, there is usually a **break** keyword. This is used to exit the **switch** statement. If **break** is not used, execution will continue on to the next **case** block, even if the **switch** expression **(choice)** doesn't match the next **case**.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/469fbe7d-1bbd-4fc5-8708-db447c7f6b30)


# Loop Statements

Loop statements are used in programming to execute a block of code repeatedly until a specified condition is met. There are three types of loop statements:

| Loop Type | Description |
| --------- | ----------- |
| `while`   | Repeatedly executes the loop body as long as the condition is true. |
| `do-while` | Similar to the while loop but the condition is evaluated after executing the loop body. Therefore, the loop body is executed at least once. |
| `for`     | Often used when you know beforehand how many times you want to execute a block of code. |


**While loop:**

A **`while`** loop is often used when you want the loop to run as long as a certain condition is true, but don't necessarily know how many times that will be.

```c
#include <stdio.h>

int main() {
    int i = 1; // This is the initialization. We start from number 1.

    while (i <= 10) { // This is the condition. As long as i is less than or equal to 10, the loop will keep running.
        printf("%d\n", i); // Print the current number.
        i++; // Increment the number. This is important! Without this, the loop would run forever because i would always be 1 and hence always less than 10.
    }

    return 0;
}
```
The condition of the **`while`** loop checks if **'i'** is **less than or equal** to **10**. Since **1** is **less than or equal** to **10**, the condition is **true**, and the loop's body is executed. 

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/de1753fd-76a0-46f6-9e66-39fbfdc6374b)




Let's cover another example of the **`while`** loop. At this example, we keep trying to open a specified file inside the **`while`** loop until the number of attempts reaches **10**. Each time we fail to open the file, we increment attempts, wait for 2 seconds, and then try again.

```c
#include <stdio.h>      
#include <windows.h>    

int main()
{
    FILE* file;          // Declare a file pointer
    char filename[] = "file.txt";  // Define the file name
    int attempts = 0;    // Initialize the attempts counter

    while (attempts < 10) {
        // Attempt to open the file in read mode
        if (fopen_s(&file, filename, "r") == 0) {
            printf("File %s successfully opened.\n", filename);
            fclose(file);
            return 0;
        }
        else {
            // If the file couldn't be opened, print a failure message
            printf("Attempt %d: File %s could not be opened. Trying again in 2 seconds...\n", attempts + 1, filename);
            Sleep(2000); // Wait for 2 seconds before the next attempt
        }
        // Increase the attempts counter
        attempts++;
    }

    printf("File %s could not be opened after 10 attempts.\n", filename);

    return 1;
}
```
The reason that it couldn't open the file was because this file didn't exists in the current directory.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/1892e9b4-17cd-414d-a588-f37694840726)


Once we are specifying the right file, we can open it successfully.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/118f5d86-9bf3-4b76-b8f3-f94fb544b987)


We have covered a **`while`** loop where the condition was set to **true**. What happens if the condition is set to **false?** Can we still execute our code or will it fail?

Let's demonstrate this with a code snippet that is an example of a **`while`** loop that has the condition set to **false**.

```c
#include <stdio.h>      
#include <windows.h>    

int main()
{
    FILE* file;          // Declare a file pointer
    char filename[] = "test.txt";  // Define the file name
    int attempts = 11;    // Condition is set to false, since 11 is higher than 10

    while (attempts < 10) {
        // Attempt to open the file in read mode
        if (fopen_s(&file, filename, "r") == 0) {
            printf("File %s successfully opened.\n", filename);
            fclose(file);
            return 0;
        }
        else {
            // If the file couldn't be opened, print a failure message
            printf("Attempt %d: File %s could not be opened. Trying again in 2 seconds...\n", attempts + 1, filename);
            Sleep(2000); // Wait for 2 seconds before the next attempt
        }
        // Increase the attempts counter
        attempts++;
    }

    printf("File %s could not be opened.\n", filename);

    return 1;
}
```

The **`while`** loop condition is **`attempts < 10`**. At this example, the **attempts** variable is initialized with the value of **11**. Because **11** is not less than **10**, the condition for the **`while`** loop is **false** from the start. This means that the loop body will not be executed.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/e634e010-2735-4e9b-b37a-12f4d1abc05e)


**Do-while loop:**

A **`do-while`** loop is a variant of the **`while`** loop. The key difference between a while loop and a **`do-while`** loop is when the condition is evaluated. In a while loop, the condition is checked before the loop body is executed, but in a do-while loop, the condition is checked after the loop body is executed.

In other words, this means that a **`do-while`** loop will **always execute its body at least once**, regardless of the condition. 

Let's demonstrate this with a code snippet:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/980f8f84-687f-40e7-80cb-57003fa8e57f)


```c
#include <stdio.h>      
#include <windows.h>    

int main()
{
    FILE* file;          // Declare a file pointer
    char filename[] = "test.txt";  // Define the file name
    int attempts = 11;    // Initialize the attempts counter

    do {
        // Attempt to open the file in read mode
        if (fopen_s(&file, filename, "r") == 0) {
            printf("File %s successfully opened.\n", filename);
            fclose(file);
            return 0;
        }
        else {
            // If the file couldn't be opened, print a failure message
            printf("Attempt %d: File %s could not be opened. Trying again in 2 seconds...\n", attempts + 1, filename);
            Sleep(2000); // Wait for 2 seconds before the next attempt
        }
        // Increase the attempts counter
        attempts++;
    } while (attempts < 10);

    printf("File %s could not be opened.\n", filename);

    return 1;
}
```
Even though our loop condition **`attempts < 10`** is initially **false**, the code inside the **`do-while`** loop will still runs once, which is why it can successfully open the file.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/0ffd4ab0-a367-4911-930e-790aa6433e94)


**For loop:**

A **`for`** loop is a control flow structure in programming that allows code to be executed repeatedly for a set number of times or until a certain condition is met. A **`for`** loop is often used when you know beforehand how many times you want the loop to run. In a for loop, you start with initializing a counter, define the condition for the loop to run, and specify how the counter should be updated.

Let's demonstrate this with a code snippet:

```c
#include <stdio.h>      
#include <windows.h>    

int main()
{
    FILE* file;          // Declare a file pointer
    char filename[] = "file.txt";  // Define the file name

    for (int attempts = 0; attempts < 10; attempts++) {
        // Attempt to open the file in read mode
        if (fopen_s(&file, filename, "r") == 0) {
            printf("File %s successfully opened.\n", filename);
            fclose(file);
            return 0;
        }
        else {
            // If the file couldn't be opened, print a failure message
            printf("Attempt %d: File %s could not be opened. Trying again in 2 seconds...\n", attempts + 1, filename);
            Sleep(2000); // Wait for 2 seconds before the next attempt
        }
    }

    printf("File %s could not be opened.\n", filename);

    return 1;
}
```
This code does exactly the same thing as the previous code. It attempts to open a file up to 10 times, waiting 2 seconds between each attempt, and breaks the loop if the file is successfully opened.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/da0e1d8b-2e09-45da-bc6d-a5ba014e046d)


Another example that will use the **`for`** loop to call **CreateProcess** and create multiple instances of **notepad.exe**

```c
#include <windows.h>
#include <stdio.h>

int main() {
    STARTUPINFO si;
    PROCESS_INFORMATION pi;
    int processCount = 5; // Change this value to create a different number of processes

    ZeroMemory(&si, sizeof(si));
    si.cb = sizeof(si);
    ZeroMemory(&pi, sizeof(pi));

    wchar_t cmd[] = L"C:\\Windows\\System32\\notepad.exe"; // Command line

    for (int i = 0; i < processCount; i++) {
        // Start the child process. 
        if (!CreateProcess(NULL, // No module name (use command line)
            cmd, // Command line
            NULL, // Process handle not inheritable
            NULL, // Thread handle not inheritable
            FALSE, // Set handle inheritance to FALSE
            0, // No creation flags
            NULL, // Use parent's environment block
            NULL, // Use parent's starting directory 
            &si, // Pointer to STARTUPINFO structure
            &pi) // Pointer to PROCESS_INFORMATION structure
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

# Jump Statements

Jump statements are a type of control flow mechanism in many programming languages that allow the execution of a program to be transferred to another part of the code. 

**goto:**

A **`goto`** statement is a feature that permits an unconditional jump from one point in the code to another. It provides a way to direct the flow of the code execution.

To understand how **`goto`** works, let's first understand two key parts associated with it: the goto keyword itself and a label.

1. **`goto`** keyword: This is the part of the statement that signals the compiler that there will be a jump in control from this point in the code.

2. **`label`** This is an identifier that you, the programmer, create. It serves as the "landing point" for the **`goto`** statement. The label is placed elsewhere in the code where you want the program control to jump to when goto is encountered.

Let's demonstrate a simple example:

This code creates a file with a random name in the **C:\Temp** folder and performs basic error handling.

```c
#include <windows.h>
#include <stdio.h>
#include <time.h>

int main() {
    srand(time(NULL)); // Seed the random number generator with the current time

    // Define the file name directly on the stack
    wchar_t filename[100];
    HANDLE fileHandle = INVALID_HANDLE_VALUE;

    // Generate a random number
    int randNum = rand();

    // Create the file name with the random number
    swprintf(filename, sizeof(filename) / sizeof(wchar_t), L"C:\\Temp\\testfile_%d.txt", randNum);

    // Create the file
    fileHandle = CreateFile(
        filename,                // name of the file
        GENERIC_WRITE,           // open for writing
        0,                       // do not share
        NULL,                    // default security
        CREATE_NEW,              // create new file only
        FILE_ATTRIBUTE_NORMAL,   // normal file
        NULL);                   // no attr. template

    if (fileHandle == INVALID_HANDLE_VALUE) {
        goto error_file_creation;  // Jump to specific error handling section
    }

    wprintf(L"File created successfully.\n");
    goto cleanup;  // If no errors, proceed to cleanup

error_file_creation:
    wprintf(L"Unable to create file due to error: %lu\n", GetLastError());

cleanup:
    // Regardless of success or failure, always close the file handle if it is valid
    if (fileHandle != INVALID_HANDLE_VALUE) {
        CloseHandle(fileHandle);
    }

    // Wait for a character input from the user before the program ends
    printf("Press any key to exit...\n");
    getchar();

    return 0;
}
```

The **error_file_creation** label is at a later part of the code and it will print out an error message that includes the specific error code. The **cleanup** label is another section of the code where the program ensures that if a file handle was opened, it will get closed with CloseHandle().

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/7a08e2e3-7b04-4853-ac8e-e5234fd9749d)


At the first example, we are able to successfully create a file on disk. Since there are no errors, we can go to the **cleanup** label, which will close the file handle.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/381f5b40-2afb-4b20-99b4-c84d4397033e)


At the second example, we are going to delete the **C:\Temp** folder, so it can't create a file in this folder. Since we are trying to create a file in **C:\Temp** that doesn't exist anymore. We are hitting **INVALID_HANDLE_VALUE**, which will jump our code to the **error_file_creation** label.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/85872816-88fa-4ba9-8fa2-cad8a5f1e696)


![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/581a2626-b4f4-4c60-8303-682a62c7a197)

