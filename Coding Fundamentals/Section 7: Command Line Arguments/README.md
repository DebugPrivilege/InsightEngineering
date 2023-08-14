# What are Command Line Arguments?

Command line arguments are parameters or inputs that are passed to a program when it is run from the command line interface. These arguments are used to specify certain parameters to control the behavior of a program.

| Parameter  | Description |
| ------------- | ------------- |
| `int argc`  | This is the argument count. It represents the number of command-line arguments that were passed to the program, including the name of the program itself. |
| `char *argv[]`  | This is the argument vector. It is an array of character pointers (strings) representing the actual command-line arguments. The first argument (`argv[0]`) is the name of the program, `argv[1]` is the first actual argument, `argv[2]` the second, and so on. |


# Simple Program with a Command Line Argument

Let's demonstrate this with a simple code snippet to showcase the usage of command line arguments.

```c
#include <stdio.h>
#include <string.h>

void show_help() {
    printf("How can I help?\n");
}

int main(int argc, char *argv[]) {
    for(int i = 0; i < argc; i++) {
        if(strcmp(argv[i], "--help") == 0) {
            show_help();
            return 0;
        }
    }
    printf("Program continues...\n");
    return 0;
}
```

The code goes through each command-line argument and compares it to the string **`"--help"`** using the **strcmp** function. If it finds a match, it calls the **`show_help`** function and then ends the program. If it doesn't find a match after checking all arguments, it continues with whatever other code you want to put after the loop.

**[IMPORTANT]**

Let's break down the code:

This is the initialization. It declares a new integer variable **i** and sets it to **0**. This variable is often called the loop counter or loop variable.

```c
int i = 0;
```

This is the condition that is checked before each iteration of the loop. If **'i'** is less than **`argc`** (the number of command-line arguments), then the loop continues. If **'i'** is not less than **`argc`**, then the loop ends.

```c
i < argc;
```

Inside the loop, this line uses the **strcmp** function to compare the current command-line argument to the string **`"--help"`**. If the current command-line argument is **`"--help"`**, then **strcmp** returns 0 and the code inside the **'if'** statement is executed.

```c
if (strcmp(argv[i], "--help") == 0)
```

The **i++** part in the for loop is called the increment step. It is what allows the loop to progress and not run indefinitely. Each time the loop completes an iteration, i++ increments the value of **'i'** by 1.

The loop begins with **'i'** equal to 0. After the code inside the loop has executed once, **i++** increments **'i'** to 1. The next time through the loop, i is 1. After the code inside the loop executes again, **i++** increments **'i'** to 2, and so on.

```c
for(int i = 0; i < argc; i++)
```

This is important because **'i'** is being used as an index to access the elements of the **`argv`** array, which holds the command-line arguments. **`argv[0]`** is the first element (usually the program name), **`argv[1]`** is the second element (the first command-line argument), etc. By incrementing **'i'** with each iteration, the loop is able to examine each command-line argument.

Let's run this program first without any command-line argument. It will print **Program continues...**

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/dc159cf5-9d5f-4872-805f-ecbe3182aecd)

If we now run the program with the **--help** option. It will print **How can I help?** this time.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/2bbab8b1-b202-4ff9-b17b-291fa4e628ed)

If we run **HelloWorld.exe --help**, **`argc`** (the count of command-line arguments) would be **2**, and the **`argv`** (argument vector) array would be as follows:

- **`argv[0]`** would be "Program.exe". This is the name of the program.
- **`argv[1]`** would be "--help". This is the first command-line argument we provided.

# Unrecognized Argument

What if we now run **Program.exe --help --invalid** which contains an invalid argument. However, it will still print **"How can I help?"**

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/42efab27-0288-48e2-9fd1-df3a244eef8a)

The **for** loop iterates over all command-line arguments. When the program sees **--help** as an argument (regardless of its position in the argument list), it calls the **show_help()** function, which prints out **"How can I help?"**.

The point is that the **return 0;** statement causes the program to stop immediately after the **--help** argument is processed, even if there are more arguments after it.

Let's now make a modification to our code and handle invalid arguments, so every time that we insert a command-line that doesn't exists. It throws an error message.

```c
#include <stdio.h>
#include <string.h>

int main(int argc, char* argv[]) {
    
    // Declare and initialize a flag to check if --help argument is passed.
    int helpFlag = 0;

    // Loop through all command line arguments.
    // Start from 1 because argv[0] is the name of the program itself.
    for (int i = 1; i < argc; i++) {
        // Check if the current argument is --help.
        // Use _stricmp for a case-insensitive comparison.
        if (_stricmp(argv[i], "--help") == 0) {
            // If --help is found, set the helpFlag to 1.
            helpFlag = 1;
        }
        else {
            // If the argument is not --help, print an error message and return 1 to indicate an error.
            printf("Unrecognized argument: %s\n", argv[i]);
            return 1;
        }
    }

    // After going through all arguments, check if --help was passed.
    if (helpFlag == 1) {
        // If --help was passed (i.e., helpFlag is 1), print a help message.
        printf("How can I help?\n");
    }

    return 0;
}
```

Here, a variable named **helpFlag** of type integer is declared and initialized with value 0. This flag is used to keep track whether **--help** argument is passed or not.

```c
int helpFlag = 0;
```

This is a **for** loop that starts from 1 and goes up to the number of arguments (argc). The **'i'** variable is used to index the **`argv`** array. This loop is necessary because it allows your program to check each command-line argument that was passed to it.

```c
for(int i = 1; i < argc; i++)
```

First, let's run our program without specifying any command-line arguments. The program doesn't enter the loop where it would check for **--help** or print an error message for **unrecognized arguments**. Because neither of these conditions are met and the program doesn't print anything.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/f42e1fd6-8544-4f35-b0d8-63f8260913e5)

Let's now run the program with an invalid argument. When we run the command **HelloWorld.exe --help --zzz**, the program checks each argument one by one. First, it checks **argv[1]**, which is **--help**. This matches the condition in the loop **(if(_stricmp(argv[i], "--help") == 0))**, and **helpFlag** is set to **1**.

Next, the program checks **argv[2]**, which is **--zzz**. This does not match the **--help** condition, so it goes into the **else** block of the conditional statement. There, it prints **"Unrecognized argument: --zzz"**

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/7397aa59-0d8c-4b0d-a1e6-980770197731)

# Another Example

Let's cover another example and end this section. When we run this code with either the **"--A"** or **"--B"** argument, it creates a new process to execute the **"whoami /all"** or **"net user"** commands, and any other argument results in an error message.

```c
#include <windows.h> 
#include <stdio.h>  

void executeCommand(const wchar_t* command) {
    STARTUPINFO si;                 // STARTUPINFO structure specifies the window station, desktop, standard handles, and appearance of the main window for a new process.
    PROCESS_INFORMATION pi;         // PROCESS_INFORMATION structure receives identification information about the new process.

    ZeroMemory(&si, sizeof(si));    // Initialize si to zero
    si.cb = sizeof(si);             // Size of structure in bytes
    ZeroMemory(&pi, sizeof(pi));    // Initialize pi to zero

    wchar_t commandCopy[MAX_PATH];
    wcscpy_s(commandCopy, MAX_PATH, command);  // Copy the command into commandCopy

    // Create a new process and execute the command
    if (!CreateProcess(NULL, commandCopy, NULL, NULL, FALSE, 0, NULL, NULL, &si, &pi)) {
        printf("CreateProcess failed (%d).\n", GetLastError()); // Print an error message if CreateProcess fails
        return;
    }

    WaitForSingleObject(pi.hProcess, INFINITE); // Wait until the child process exits.

    CloseHandle(pi.hProcess); // Close the process handle
    CloseHandle(pi.hThread);  // Close the thread handle
}

int wmain(int argc, wchar_t* argv[]) {
    if (argc > 1) { // If there are command line arguments
        for (int i = 1; i < argc; i++) {
            // If the command line argument is --A
            if (_wcsicmp(argv[i], L"--A") == 0) {
                executeCommand(L"whoami /all"); // Execute the whoami /all command
            }
            // If the command line argument is --B
            else if (_wcsicmp(argv[i], L"--B") == 0) {
                executeCommand(L"net user");    // Execute the net user command
            }
            else {
                wprintf(L"Unrecognized argument: %s\n", argv[i]); // Print an error message for unrecognized arguments
                return 1; 
            }
        }
    }
    return 0; 
}
```

The **for** loop iterates over each command-line argument passed to the program, starting from the second argument (since the first one is the name of the program itself). If an argument matches the string **"--A"** in a case-insensitive manner, it calls the **executeCommand** function with the command **"whoami /all"**. If an argument matches **"--B"**, it executes the command **"net user"**. If the argument does not match either **"--A"** or **"--B"**, the program prints an error message indicating that the argument is unrecognized.


```c
    if (argc > 1) { // If there are command line arguments
        for (int i = 1; i < argc; i++) {
            // If the command line argument is --A
            if (_wcsicmp(argv[i], L"--A") == 0) {
                executeCommand(L"whoami /all"); // Execute the whoami /all command
            }
            // If the command line argument is --B
            else if (_wcsicmp(argv[i], L"--B") == 0) {
                executeCommand(L"net user");    // Execute the net user command
            }
            else {
                wprintf(L"Unrecognized argument: %s\n", argv[i]); // Print an error message for unrecognized arguments
                return 1; 
            }
```

The reason we need wide strings **`(_wciscmp)`** has to do with the **CreateProcess** function and the **wmain** function, both of which operate with wide strings **(wchar_t type)**. The version of **CreateProcess** being used here expects its command line argument to be a **wide string**, which is why **`L"whoami /all"`** and **`L"net user"`** have the **L** prefix to make it a wide string literal. 

First example, we start with the **--A** parameter which will execute **"whoami /all"**

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/96f96d68-a902-4eea-8121-4dbe7f7efb02)

Second example, we will type the **--B** parameter which will execute **"net user"**

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/3330a04e-e5f2-4f97-b524-1ec34cd3cac4)


# Argument Validation

Let's re-write the code and validate the command-line arguments, so every time that we specify an argument that doesn't exists. It will throw an **"Unrecognized Argument"** message.

This code adds a validation step before executing any commands. The first loop goes through each argument provided to the program. If it encounters an argument that doesn't match **"--A"** or **"--B"** (case-insensitive), it prints an error message and exits the program with an error code.

```c
#include <windows.h> 
#include <stdio.h>  

void executeCommand(const wchar_t* command) {
    STARTUPINFO si;
    PROCESS_INFORMATION pi;

    ZeroMemory(&si, sizeof(si));
    si.cb = sizeof(si);
    ZeroMemory(&pi, sizeof(pi));

    wchar_t commandCopy[MAX_PATH];
    wcscpy_s(commandCopy, MAX_PATH, command);

    if (!CreateProcess(NULL, commandCopy, NULL, NULL, FALSE, 0, NULL, NULL, &si, &pi)) {
        printf("CreateProcess failed (%d).\n", GetLastError());
        return;
    }

    WaitForSingleObject(pi.hProcess, INFINITE);

    CloseHandle(pi.hProcess);
    CloseHandle(pi.hThread);
}

int wmain(int argc, wchar_t* argv[]) {
    if (argc > 1) {
        // First, validate all arguments
        for (int i = 1; i < argc; i++) {
            if (_wcsicmp(argv[i], L"--A") != 0 && _wcsicmp(argv[i], L"--B") != 0) {
                wprintf(L"Unrecognized argument: %s\n", argv[i]);
                return 1;  // Return an error code
            }
        }

        // If all arguments are valid, execute the commands
        for (int i = 1; i < argc; i++) {
            if (_wcsicmp(argv[i], L"--A") == 0) {
                executeCommand(L"whoami /all");
            }
            else if (_wcsicmp(argv[i], L"--B") == 0) {
                executeCommand(L"net user");
            }
        }
    }
    return 0;
}
```
Let's break down the code to gather a better understanding.

The command line argument validation is done by this part of the code in the **wmain** function:

```c
if (argc > 1) {
    // First, validate all arguments
    for (int i = 1; i < argc; i++) {
        if (_wcsicmp(argv[i], L"--A") != 0 && _wcsicmp(argv[i], L"--B") != 0) {
            wprintf(L"Unrecognized argument: %s\n", argv[i]);
            return 1;  // Return an error code
        }
    }
```

In this section, the code checks if the number of arguments **(argc)** is **greater** than **1**. If it is, it means that some command-line arguments were passed to the program. It uses a **for** loop to iterate over each of the command-line arguments, starting from **argv[1]** (since argv[0] is the name of the program itself).

This line is a condition that checks whether the current command-line argument argv[i] is not --A and also not --B.

```c
if (_wcsicmp(argv[i], L"--A") != 0 && _wcsicmp(argv[i], L"--B") != 0)
```

- **`argv[i]`** is the current argument being checked.
- **`_wcsicmp(argv[i], L"--A") != 0`** checks if argv[i] is not --A.
- **`_wcsicmp(argv[i], L"--B") != 0`** checks if argv[i] is not --B.

When both conditions are true (i.e., the argument is **neither** --A nor --B), the condition in the **if** statement becomes **true**. In this case:

```c
wprintf(L"Unrecognized argument: %s\n", argv[i]);
return 1;  // Exit the program with an error code
```

The program prints **"Unrecognized argument"** followed by the argument, and then exits with an error code of 1. This means it found an invalid argument.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/06c871f0-e5aa-49e8-a46b-f7b56b1315e5)

This code checks all the arguments provided. It doesn't stop checking after it finds a valid argument. The loop in the validation part of the code goes through every argument that is passed:

```c
for (int i = 1; i < argc; i++) {
    if (_wcsicmp(argv[i], L"--A") != 0 && _wcsicmp(argv[i], L"--B") != 0) {
        wprintf(L"Unrecognized argument: %s\n", argv[i]);
        return 1;  // Return an error code
    }
}
```

In this loop, **`argc`** is the count of arguments passed, and **`argv[i]`** accesses each command-line argument one by one. Even though that **--A** is a valid argument and gets recognized, the program continues to the next argument **--C** which is not recognized. This leads to the program exist with an error code of 1.

# **Final Example**

We should now have a basic understanding on how command-line arguments are implemented, so let's cover one final example.

```c
#include <windows.h>
#include <stdio.h>

void showMessageHelloWorld() {
    MessageBox(0, L"Hello, World!", L"Message", MB_OK);
}

void showMessageHelloEveryone() {
    MessageBox(0, L"Hello Everyone!", L"Message", MB_OK);
}

void showMessageGoodbyeWorld() {
    MessageBox(0, L"Goodbye, World!", L"Message", MB_OK);
}

int wmain(int argc, wchar_t* argv[]) {
    if (argc > 1) {
        // First, validate all arguments
        for (int i = 1; i < argc; i++) {
            if (_wcsicmp(argv[i], L"--C") != 0 && _wcsicmp(argv[i], L"--D") != 0 && _wcsicmp(argv[i], L"--E") != 0) {
                wprintf(L"Unrecognized argument: %s\n", argv[i]);
                return 1;  // Return an error code
            }
        }

        // If all arguments are valid, execute the commands
        for (int i = 1; i < argc; i++) {
            if (_wcsicmp(argv[i], L"--C") == 0) {
                showMessageHelloWorld();
            }
            else if (_wcsicmp(argv[i], L"--D") == 0) {
                showMessageHelloEveryone();
            }
            else if (_wcsicmp(argv[i], L"--E") == 0) {
                showMessageGoodbyeWorld();
            }
        }
    }
    return 0;
}
```

The first **for** loop is for argument validation. It iterates over each command line argument and checks if it is equal to **"--C"**, **"--D"**, or **"--E"**. If the argument doesn't match any of these valid inputs, it prints an **"Unrecognized argument"** error message and returns 1 to exit the program.

The second **for** loop is for executing the commands that is related to each valid command line argument. If an argument matches **"--C"**, it calls **`showMessageHelloWorld()`**. If it matches **"--D"**, it calls **`showMessageHelloEveryone()`**. And if it matches **"--E"**, it calls **`showMessageGoodbyeWorld()`**.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/1ffb9746-f367-46cd-ab55-128088eeab5b)

