# What are Variables?

Variables can be seen as 'containers' that are used to hold values (like a number or text) throughout a program. The value of a variable can change, and it's this capacity for change that makes variables useful. Variables are associated with a **data type**, and that **data type** determines what kind of data the variable can hold, and what operations can be performed on it.

Let's cover a simple example to demonstrate this:

```c
#include <stdio.h>

int main() {
    
    // Declare a local integer variable and assign it a value
    int number = 10;

    // Print the value of the local variable
    printf("The value of the local variable 'number' is: %d\n", number);

    return 0;
}
```
At this example, **number** is the **local** variable. A **local** variable is a variable that is declared within a function and can only be used within that function or block. In this example, our variable of the type **integer** is being declared with the value **10** inside the **main()** function.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/03d7cdbe-ebb6-41d5-a3ee-7204520a7d92)


Here is another example where **number** is a variable of the type **integer**. Initially we are assigning the variable with the value **10** and then print it out. However, at this example. We are assigning an additional value to the same variable and set the value to **20**. As you can see, the value stored in the variable **number** can be changed -- that's what makes it a "variable".

```c
#include <stdio.h>

int main() {
    // Declare an integer variable
    int number;

    // Assign a value to the variable
    number = 10;

    // Print the value of the variable
    printf("The value of number is: %d\n", number);

    // Change the value of the variable
    number = 20;

    // Print the new value of the variable
    printf("The value of number is now: %d\n", number);

    return 0;
}
```

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/33eac02c-8ddc-4170-8950-9344c1fa39f2)



# Declaration and Initialization

**Declaration** is the process where a variable is introduced to the compiler. The **declaration** specifies the variable's name and its data type. For instance, the declaration **int number;** tells the compiler that there's a variable named **number** of type **int**. At this stage, no value has been assigned to the variable. It is been declared, so the compiler knows it exists and what kind of data it is meant to hold.

```c
int number; // Declaration
```

**Initialization** on the other hand is the process of of assigning an value to a variable at the time of its **declaration**. In other words, when you initialize a variable, you not only declare it (tell the compiler about its name and type) but you also set its initial value.

```c
int number = 10; // Declare an integer variable 'number' and initialize it with the value 10
```

# Primitive Data Types

These data types are built-in or predefined data types and can be used directly by the user to **declare variables**. Here are some examples of common Primitive Data Types, but it is not the full list.


| Type                  | Bytes | Description                                             | Range of Values                                                     | Format Specifier (Used with printf) |
| --------------------- | ------|-------------------------------------------------------- | ------------------------------------------------------------------- | ---------------- |
| bool                  | 1     | Used to store true or false      | 0 (false), 1 (true)                                                 | `%d`             |
| char                  | 1     | Used to store single characters                         | -128 to 127                                                         | `%c`             |
| unsigned char         | 1     | Unsigned char                                           | 0 to 255                                                            | `%c`             |
| signed char           | 1     | Signed char                                             | -128 to 127                                                         | `%c`             |
| short                 | 2     | Smaller-sized integer                                   | -32,768 to 32,767                                                   | `%hd`            |
| unsigned short        | 2     | Unsigned smaller-sized integer                           | 0 to 65,535                                                         | `%hu`            |
| int                   | 4     | Used to store whole numbers                             | -2,147,483,648 to 2,147,483,647                                     | `%d` or `%i`     |
| unsigned              | 4     | Integer that can only hold non-negative numbers         | 0 to 4,294,967,295                                                  | `%u`             |
| long                  | 4     | Larger-sized integer                                    | -2,147,483,648 to 2,147,483,647                                     | `%ld` or `%li`   |
| unsigned long         | 4     | Unsigned larger-sized integer                            | 0 to 4,294,967,295                                                  | `%lu`            |
| long long             | 8     | Even larger-sized integer                               | -9,223,372,036,854,775,808 to 9,223,372,036,854,775,807             | `%lld` or `%lli` |
| unsigned long long    | 8     | Unsigned even larger-sized integer                       | 0 to 18,446,744,073,709,551,615                                     | `%llu`           |
| float                 | 4     | Used to store decimal numbers with single precision     | Approximately 1.175494e-38 to 3.402823e+38                          | `%f`, `%.2f`, `%e`, `%g` |
| double                | 8     | Used to store decimal numbers with double precision     | Approximately 2.225074e-308 to 1.797693e+308                        | `%lf` |
| long double           | 8     | Used to store decimal numbers with even more precision  | Approximately 2.225074e-308 to 1.797693e+308                        | `%Lf` |
| void                  |       | Special type representing the absence of value          | N/A                                                                 | N/A              |
| wchar_t               | 2     | Used to store wide characters                | Varies based on implementation                                      | `%lc` |

At this stage, you might be wondering. How do we know the exact **size** in bytes of each data type and it's **range value** for instance? Well in terms of getting the **size** of each data type. We can use the **sizeof()** operator which is used to determine the size in bytes of a type or an expression.

Let's demonstrate that with code:

```c
#include <stdio.h>
#include <inttypes.h>

int main() {
    printf("Size of bool: %zu bytes\n", sizeof(bool));
    printf("Size of char: %zu bytes\n", sizeof(char));
    printf("Size of unsigned char: %zu bytes\n", sizeof(unsigned char));
    printf("Size of signed char: %zu bytes\n", sizeof(signed char));
    printf("Size of short: %zu bytes\n", sizeof(short));
    printf("Size of unsigned short: %zu bytes\n", sizeof(unsigned short));
    printf("Size of int: %zu bytes\n", sizeof(int));
    printf("Size of unsigned int: %zu bytes\n", sizeof(unsigned int));
    printf("Size of long: %zu bytes\n", sizeof(long));
    printf("Size of unsigned long: %zu bytes\n", sizeof(unsigned long));
    printf("Size of long long: %zu bytes\n", sizeof(long long));
    printf("Size of unsigned long long: %zu bytes\n", sizeof(unsigned long long));
    printf("Size of float: %zu bytes\n", sizeof(float));
    printf("Size of double: %zu bytes\n", sizeof(double));
    printf("Size of long double: %zu bytes\n", sizeof(long double));
    printf("Size of void: N/A\n");
    printf("Size of wchar_t: %zu bytes\n", sizeof(wchar_t));

    return 0;
}
```
This snippet of code prints the size in bytes for each specified data type. The **%zu** format specifier is used in this code to correctly print the value of type **size_t**, which is returned by the **sizeof** operator. 

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/d4c34ac8-4e7a-4171-84a7-0b6fc568d00c)

The second example demonstrates on how we are able to gather the range value of each specified data type. At this piece of code, we will see **constants** such as, **INT_MIN**, **INT_MAX**, etc. All these **constants** are representing the range and characteristics of various primitive data types. These constants are defined in the **limits.h** and **cfloat** header files, and they provide information about the minimum and maximum values that can be represented by the data types. 

For example **INT_MIN** is a constant that represents the minimum value for the **signed char** data type, while **INT_MAX** represents the maximum value of **signed char**.

```c
#include <stdio.h>
#include <limits.h>
#include <cfloat>

int main() {
    printf("Range of int: %d to %d\n", INT_MIN, INT_MAX);
    printf("Range of unsigned int: %u to %u\n", 0, UINT_MAX);
    printf("Range of short: %d to %d\n", SHRT_MIN, SHRT_MAX);
    printf("Range of unsigned short: %u to %u\n", 0, USHRT_MAX);
    printf("Range of long: %ld to %ld\n", LONG_MIN, LONG_MAX);
    printf("Range of unsigned long: %lu to %lu\n", 0L, ULONG_MAX);
    printf("Range of long long: %lld to %lld\n", LLONG_MIN, LLONG_MAX);
    printf("Range of unsigned long long: %llu to %llu\n", 0LL, ULLONG_MAX);
    printf("Range of char: %d to %d\n", CHAR_MIN, CHAR_MAX);
    printf("Range of unsigned char: %u to %u\n", 0, UCHAR_MAX);
    printf("Range of signed char: %d to %d\n", SCHAR_MIN, SCHAR_MAX);
    printf("Range of float: %e to %e\n", FLT_MIN, FLT_MAX);
    printf("Range of double: %e to %e\n", DBL_MIN, DBL_MAX);
    printf("Range of long double: %Le to %Le\n", LDBL_MIN, LDBL_MAX);

    return 0;
}
```

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/5d71c943-d8bb-47e9-8985-02da6abccfa7)

# Format Specifiers

Format specifiers are placeholders used in formatted input and output functions in programming languages like C and C++. They indicate the type and format of data to be printed or read. 

```c
int number = 10;
printf("The number is: %d\n", number); // Output: The number is: 10
```

The format specifier **%d** is used to indicate that the value should be interpreted as an **integer**. It is important to specify the correct specifier or otherwise data might not be parsed or printed correctly.

Here is a **BAD** example where data is not printed correctly because the wrong specifier is used:

```c
#include <stdio.h>

int main() {
    int number = 10;
    printf("The number is: %lc\n", number); // <--- Wrong Format Specifier leads to incorrect data that is printed

    return 0;
}
```

The format specifier **%lc** is used in the printf statement to print the value of **number** as a wide character. However, **number** is declared as an **int** type, not a wide character type. This is why we don't see the value **10** being printed.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/2be86308-5aec-4af3-b755-dda65c33ead0)

Let's now apply **format specifiers** in practice and use it with different data types to print the specified values. For example, **%d** is used to print **boolVar** as an **integer**, **%c** for **charVar** as a **character**, **%u** for **ucharVar** as an **unsigned integer**, and so on.

```c
#include <stdio.h>
#include <stdbool.h>

int main() {
    bool boolVar = true;
    char charVar = 'A';
    unsigned char ucharVar = 255;
    signed char scharVar = -127;
    short shortVar = -32768;
    unsigned short ushortVar = 65535;
    int intVar = -2147483647 - 1;
    unsigned int uintVar = 4294967295U;
    long longVar = -2147483647L - 1L;
    unsigned long ulongVar = 4294967295UL;
    long long longLongVar = -9223372036854775807LL - 1LL;
    unsigned long long ulongLongVar = 18446744073709551615ULL;
    float floatVar = 3.14f;
    double doubleVar = 3.14159;
    long double longDoubleVar = 3.141592653589793238;
    void* voidPtr = NULL;
    wchar_t wideCharVar = L'Z';

    printf("Boolean: %d\n", boolVar);
    printf("Character: %c\n", charVar);
    printf("Unsigned Character: %u\n", ucharVar);
    printf("Signed Character: %d\n", scharVar);
    printf("Short: %hd\n", shortVar);
    printf("Unsigned Short: %hu\n", ushortVar);
    printf("Integer: %d\n", intVar);
    printf("Unsigned Integer: %u\n", uintVar);
    printf("Long: %ld\n", longVar);
    printf("Unsigned Long: %lu\n", ulongVar);
    printf("Long Long: %lld\n", longLongVar);
    printf("Unsigned Long Long: %llu\n", ulongLongVar);
    printf("Float: %f\n", floatVar);
    printf("Double: %lf\n", doubleVar);
    printf("Long Double: %Lf\n", longDoubleVar);
    printf("Void Pointer: %p\n", voidPtr);
    printf("Wide Character: %lc\n", wideCharVar);

    return 0;
}
```
![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/a2991171-e48e-42e0-b181-494f84cf15f1)

# Global Variables

Global variables are variables that are **declared outside** of any function or block, at the **top of the program** or in a separate file. They are **accessible** and usable by **all** functions in the program.

Let's demonstrate that with some code snippet:

```c
#include <stdio.h>

// Global variables
int globalVar1 = 10; // Global variable 1
int globalVar2 = 20; // Global variable 2

// Function to print the values of global variables
void printGlobalVariables() {
    printf("Global Variable 1: %d\n", globalVar1);
    printf("Global Variable 2: %d\n", globalVar2);
}

// Function to modify the values of global variables
void modifyGlobalVariables() {
    globalVar1 += 5; // Increment globalVar1 by 5
    globalVar2 -= 3; // Decrement globalVar2 by 3
}

int main() {
    printf("Initial values of global variables:\n");
    printGlobalVariables();

    printf("\nModifying global variables...\n");
    modifyGlobalVariables();

    printf("\nModified values of global variables:\n");
    printGlobalVariables();

    return 0;
}
```
We have two global variables **globalVar1** and **globalVar2** declared at the top of the program, **outside** of any function. These variables are accessible by all functions in the program. 

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/b71d4607-c053-4cec-820d-a56f31539c7a)

Global variables are allocated and initialized before any functions are called, and their memory is retained until the program terminates. Global variables are stored on a seprate memory region called **data segment**. Global variables are stored in the data segment of the program's memory because they have a global scope and lifetime. Being in the **data segment** means they have a fixed memory location that remains constant throughout the program's execution. This allows the variables to be accessed and modified from any part of the program.

# Local Variables

Local variables are variables that are declared and used **within** a specific block or scope, such as a function. Local variables have a limited scope and lifetime, which means they are only accessible within the block or function where they are defined. They exist only as long as that block is executing. Typically, local variables are stored on the stack, a region of memory dedicated to managing function calls and local variable storage.

In this example, we are trying to access a local variable outside of the function. This leads to an error when compiling the code.

```c
#include <stdio.h>

void foo() {
    int x = 10;
}

int main() {
    foo();
    printf("%d\n", x); // Trying to access the local variable 'x' outside its scope

    return 0;
}
```
The function **foo()** declares a local variable **x** within its scope. However, when we try to access the variable **x** in the **main()** function, which is outside the scope of **foo()**, it causes a compilation error. This is because local variables are only accessible within the block or function in which they are declared.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/3af3bff8-33e4-424c-905e-ef31cfa54ac5)

As previously discussed, local variables have a limited scope and lifetime. The lifetime of a variable refers to the period during which the variable exists and holds a valid value. For local variables, their lifetime is limited to the duration of the block or function where they are defined. Once the memory is released after the execution of a block or function, the local variables within that block will lose their memory address. 

Let's proof this as well with this snippet of code:

```c
#include <stdio.h>

int globalVariable;  // Global variable

void foo() {
    int localVariable;  // Local variable

    printf("Address of globalVariable: %p\n", &globalVariable);
    printf("Address of localVariable: %p\n", &localVariable);
}

int main() {
    foo();
    return 0;
}
```
Compile this code and execute it. Further, pay a close attention to the memory address that is set to the local variable. During the first execution, our memory address is **000000F18B9DF964**. 

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/70a0f7b6-ec1f-4f05-bac9-ad44b669083e)


When the block or function is executed again, new memory will be allocated on the stack for the local variables, and they will be assigned new memory addresses.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/f070bc7f-a5e5-4933-9dc1-7f90d6aa1435)

# Static Variables

Static variables in C retain their values across different function calls. They are initialized only once and keep their values throughout the program's execution. We declare static variables using the static keyword, and they are stored in the memory's data segment, just like global variables. Static variables can only be directly accessed within the block or function where they are defined, but they can be indirectly accessed outside if their reference or pointer is returned. 

Let's demonstrate first how defining a static variable looks like:

```c
#include <stdio.h>

// Function to keep track of the number of function calls
void countCalls() {
    // Declare a static variable 'callCount' with initial value of 0
    static int callCount = 0;
    callCount++; // Increment the value of 'callCount'
    printf("Number of function calls: %d\n", callCount); // Print the value of 'callCount'
}

int main() {
    // Call the countCalls function multiple times
    countCalls();  // Output: Number of function calls: 1
    countCalls();  // Output: Number of function calls: 2
    countCalls();  // Output: Number of function calls: 3

    return 0;
}
```

The static variable **callCount** keeps track of the number of times the **countCalls** function is called. Each time the function is executed, the value of **callCount** is incremented and printed. The static variable holds onto its value between function executions, allowing us to maintain the count across different function calls.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/83bd9107-2925-4689-8ac5-7126dafb411e)

Let's now cover another example on how we can indirectly access the static variable:

```c
#include <stdio.h>

// Function to keep track of the number of function calls
int* countCalls() {
    // Declare a static variable 'callCount' with initial value of 0
    static int callCount = 0;
    callCount++; // Increment the value of 'callCount'
    printf("Number of function calls: %d\n", callCount); // Print the value of 'callCount'
    return &callCount; // Return the address of 'callCount'
}

int main() {
    // Call the countCalls function multiple times
    int* callCountPtr;
    callCountPtr = countCalls();  // Output: Number of function calls: 1
    callCountPtr = countCalls();  // Output: Number of function calls: 2
    callCountPtr = countCalls();  // Output: Number of function calls: 3

    // Indirectly access 'callCount' using the pointer returned by 'countCalls()'
    printf("Indirectly accessed callCount: %d\n", *callCountPtr);  // Output: Indirectly accessed callCount: 3

    return 0;
}
```
![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/b855b885-4a51-47c7-aa9e-34e60a0bf5a7)

The **countCalls()** function now returns a pointer to the static **callCount** variable. In **main()**, we store this pointer in **callCountPtr** each time we call **countCalls()**. At the end of **main()**, we use **callCountPtr** to indirectly access the **callCount** variable and print its value. This is an example of indirect access to a static variable outside of its defining function.
