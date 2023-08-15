# What are Preprocessor Directives?

Preprocessor directives are lines included in the code of programs written in the C and C++ programming languages that are not actual program code. They are directions for the compiler's preprocessor, which processes them before actual compilation of code begins.

These directives tell the preprocessor to perform specific actions, such as including a header file, defining a constant, or performing conditional compilation.

| Directive        | Description |
| ---------------- | ----------- |
| `#include`       | This directive tells a C/C++ preprocessor to include a specific file in the program. For example, `#include <stdio.h>` tells the preprocessor to include the header file stdio.h, which contains the prototypes of standard input and output functions such as printf and scanf. |
| `#define`        | This directive is used to define a constant or a macro that will be replaced by the preprocessor with its value before the program is compiled. 
| `#ifdef`, `#ifndef`, `#if`, `#else`, `#elif`, `#endif` | These directives are used for conditional compilation. The preprocessor will test these conditions and depending on the result, the subsequent code will be included or ignored in the compilation process. |
| `#pragma`       | This directive is a method of providing additional information to the compiler, beyond what is conveyed in the language itself. The set of valid `#pragma`s is compiler-specific. |
| `#undef`        | This directive is used to undefine a macro. |
| `#error`        | This directive prints a user-specified error message on stderr at compile time if a certain condition is not met. |


![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/0af753a4-dc7a-427a-8af2-b3d6a1e67fd9)




# #include directive

This is a simple example of telling the preprocessor to include a Header file using the **#include** directive. This **<stdio.h>** file allows us to use the **printf** function to print out *Hello, World!* to the console.

```c
#include <stdio.h>

int main() {
    printf("Hello, World!\n");
    return 0;
}
```
![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/c78c69e3-f0a4-49e9-b458-b91b62b97171)


# #define directive

The **#define** directive in C lets us create **macros**, which can act like unchanging values or replacements for text. These are processed before the program is compiled. For example, we use **#define** to make a macro that represents the number of days in a week. This kind of **'constant'** is a symbol that, once defined, stays the same in the code and cannot be changed while the program is running

```c
#include <stdio.h>

// Define a constant for the number of days in a week
#define DAYS_IN_WEEK 7

int main() {
    // Declare an integer variable and assign it the value of 3
    int weeks = 3;

    // Calculate the number of days in the given number of weeks
    int days = weeks * DAYS_IN_WEEK;

    // Print the calculated number of days
    printf("There are %d days in %d weeks.\n", days, weeks);

    // Return 0 to indicate that the program finished successfully
    return 0;
}
```

**#define DAYS_IN_WEEK 7** defines a macro that represents the number of days in a week. Then we calculate the number of days in 3 weeks with the * operator and then print it out. In this case, every instance of **DAYS_IN_WEEK** in the code is replaced with **7** before the program is compiled.
 

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/3a7853b0-f8da-47b1-890d-88c9effd4c4c)


# #if directive

The **#if** directive in C and C++ is a preprocessor directive used for conditional compilation. Code within an **#if** and corresponding **#endif** block is only compiled if the condition in the **#if** directive evaluates to true (1).

Let's demonstrate this with a code snippet:

```c
#include <stdio.h>

// Define a preprocessor macro for conditional compilation
#define USE_HELLO 1


int main() {
    // Preprocessor will check if USE_HELLO is true (non-zero)
#if USE_HELLO  
    // If USE_HELLO is true, this line of code will be included in the compilation
    printf("Hello, World!\n");
#else
    // If USE_HELLO is false (zero), this line of code will be included in the compilation instead
    printf("Goodbye, World!\n");
#endif

    return 0;
}
```

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/01c10d18-cae1-4574-925e-ad7ec263bec6)


Another simple example:

```c
#include <stdio.h>

#define PLATFORM 1

int main() {
    #if PLATFORM == 1
        printf("Platform is Windows.\n");
    #elif PLATFORM == 2
        printf("Platform is Linux.\n");
    #elif PLATFORM == 3
        printf("Platform is MacOS.\n");
    #else
        printf("Platform is unknown.\n");
    #endif

    return 0;
}
```

The preprocessor will check the value of **PLATFORM**. If **PLATFORM** is **1**, it will compile the code inside the **#if** block. If **PLATFORM** is **2**, it will compile the code inside the first **#elif** block. If **PLATFORM** is **3**, it will compile the code inside the second **#elif** block. If **PLATFORM** is none of these, it will compile the code inside the **#else** block.

The **#elif** directive stands for "else if". It allows you to test for multiple different conditions.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/aeb9591f-676c-4057-84d9-fdb412358cb0)


# #pragma directive

The **#pragma** directive is a special feature of the C and C++ programming languages. It's used to provide additional instructions to the compiler that are not part of the standard language syntax.

Let's demonstrate an example. **#pragma comment(lib, "libraryname")** is a directive used in C/C++ programming, specifically with Microsoft's Visual C++ Compiler (MSVC). This directive tells the linker to add a specific library to the list of libraries to be used during the linking process.

```c
#include <windows.h>

#pragma comment(lib, "user32.lib")

int main() {
    MessageBox(NULL, L"Hello, World!", L"My Program", MB_OK);
    return 0;
}
```
We are telling the linker to include the **user32.lib** library, which provides the **MessageBox** function. Usually this is done through dynamic library loading, but as example we use the **#pragma** directive.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/9de19a85-b2e8-477f-8133-44b3e9909d56)


Another example, which you should never do. However, we can use the **#pragma** directive to disable a specific warning for example. **#pragma warning(disable : 4996)** disables warning C4996, which is generated for certain functions that Visual Studio considers to be unsafe, like **scanf()**.

```c
#include <stdio.h>

// Disables warning C4996, for 'scanf': This function or variable may be unsafe.
#pragma warning(disable : 4996)

int main() {
    int number;

    printf("Enter a number: ");
    scanf("%d", &number);

    printf("You entered: %d\n", number);

    return 0;
}
```

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/1545ae40-aa56-41c3-a2ce-367be3b04e16)


