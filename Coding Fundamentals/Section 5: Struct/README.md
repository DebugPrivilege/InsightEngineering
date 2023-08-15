# What is a struct?

A struct (short for structure) is a user-defined data type that allows you to combine data items of different kinds. Structures are used when you want to group together related data items of different types. Each data item in a structure is called a **member**, and can have a different **type**. 

Structs are useful because they allow you to group related data together, simplifying code organization, improving readability, and enabling efficient manipulation of data as a single unit.

This is how a **struct** may look like:

```c
struct MMaFighter {
    char name[50];
    int age;
    float weight; // in kilograms
    int wins;
    int losses;
    char weightClass[20]; // example: "Lightweight", "Welterweight", etc.
    char fightingStyle[30]; // example: "Boxing", "Brazilian Jiu-Jitsu", etc.
};
```
We can now declare a variable of this **struct** type and initialize it. At this example, **`MMaFighter`** is the struct as it is defined with the **struct** keyword. The members of the struct are **`name`**, **`age`**, **`weight`**, **`wins`**, **`losses`**, **`weightClass`**, and **`fightingStyle`**. 

As we can see, the **members** of the struct can have a different type, such as **char** that represents the name of the MMA Fighter, and **int** represents the age of the MMA Fighter.

```c
#include <stdio.h>
#include <string.h>

// Definition of the MMaFighter struct
struct MMaFighter {
    char name[50];
    int age;
    float weight;
    int wins;
    int losses;
    char weightClass[20];
    char fightingStyle[30];
};

int main() {
    // Declare and initialize an instance of MMaFighter
    struct MMaFighter fighter1;

    strncpy_s(fighter1.name, sizeof(fighter1.name), "Conor McGregor", _TRUNCATE);
    fighter1.age = 33;
    fighter1.weight = 77.1;
    fighter1.wins = 22;
    fighter1.losses = 6;
    strncpy_s(fighter1.weightClass, sizeof(fighter1.weightClass), "Lightweight", _TRUNCATE);
    strncpy_s(fighter1.fightingStyle, sizeof(fighter1.fightingStyle), "Boxing", _TRUNCATE);

    // Print the values of the fighter1 struct to the console
    printf("Name: %s\n", fighter1.name);
    printf("Age: %d\n", fighter1.age);
    printf("Weight: %.1f kg\n", fighter1.weight);
    printf("Wins: %d\n", fighter1.wins);
    printf("Losses: %d\n", fighter1.losses);
    printf("Weight Class: %s\n", fighter1.weightClass);
    printf("Fighting Style: %s\n", fighter1.fightingStyle);

    return 0;
}
```

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/e620381d-b104-411f-aa58-eb645dcab354)

Let's cover another example and go a bit more in depth. At this example, we have a struct called **`FileInfo`**. This struct has two members, which happens to be **`filename`** and **`openAttempts`**.

**`filename:`** This field is an array of characters, also known as a string, which holds the name of the file that the program will attempt to open.

**`openAttempts:`** This field is an integer that keeps track of the number of attempts the program has made to open the file. Every time an attempt to open the file is made, this value is incremented by 1.

```c
#include <stdio.h>
#include <windows.h>

struct FileInfo {
    char filename[50];  // To store the name of the file (max 49 characters + null terminator)
    int openAttempts;   // To keep track of the number of attempts made to open the file
};

int main()
{
    FILE* file;  // Declare a file pointer

    // Declare and initialize a FileInfo struct
    struct FileInfo fileInfo;
    strcpy_s(fileInfo.filename, "file.txt");  // Assign the filename
    fileInfo.openAttempts = 0;              // Initialize openAttempts to 0

    // Loop to attempt opening the file for a maximum of 10 times
    for (fileInfo.openAttempts = 0; fileInfo.openAttempts < 10; fileInfo.openAttempts++) {
        // Try to open the file in read mode
        if (fopen_s(&file, fileInfo.filename, "r") == 0) {
            printf("File %s successfully opened.\n", fileInfo.filename);

            fclose(file);         
            return 0;             
        }
        else {
            // If the file couldn't be opened, print a failure message and the number of attempts so far
            printf("Attempt %d: File %s could not be opened. Trying again in 2 seconds...\n",
                fileInfo.openAttempts + 1, fileInfo.filename);
            Sleep(2000); // Pause the program for 2 seconds before the next attempt
        }
    }

    // After all attempts, if the file is still not opened, print a failure message
    printf("File %s could not be opened after %d attempts.\n", fileInfo.filename, fileInfo.openAttempts);

    return 1;  // Return 1 to indicate that the program completed with a failure
}
```
The purpose of this struct is to aggregate related information about a file — its name and the number of attempts to open it — into a single entity. This makes it easier to pass information around as a single unit.

Well, what do we mean by *'single unit'*? Single unit means that the **`filename`** and **`openAttempts`** variables, which are both related to the file that we are trying to open, are bundled together under the **`FileInfo`** struct. Instead of passing around and managing two separate variables **`(filename and openAttempts)`**, we can now manage them as a single unit **`(fileInfo)`**.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/62d4737e-e374-4ccf-bc60-56f2287980c7)

# The process of declaring a struct

1. First, we need to declare the struct type. This starts with using the **struct** keyword followed by the struct name and a list of its members enclosed in braces **{}**. Each member has a type and a name. In our example, we have a struct type named **`FileInfo`** is declared with two members: **`filename`** (a string) and **`openAttempts`** (an integer).

```c
struct FileInfo {
    char filename[50];
    int openAttempts;
};
```

2. After declaring the struct type, we can create variables of that type. In our example, we have **`fileInfo`** that is a variable of type struct **`FileInfo`**.

```c
struct FileInfo fileInfo;
```

3. We can now access the individual members of a struct variable using the dot **(.)** operator. In our example, the **`filename`** and **`openAttempts`** members of the **`fileInfo`** variable are being accessed.

```c
strcpy(fileInfo.filename, "file.txt");
fileInfo.openAttempts = 0;
```

# What is the typedef keyword?

**typedef** is a keyword in C programming language that is used to assign an alias to an existing type. Using **typedef** keyword makes the code cleaner and more readable, especially if we have to declare multiple variables or function parameters of this struct type.

First, let's demonstrate a code snippet using the **typedef** keyword:

```c
#include <stdio.h>
#include <windows.h>

typedef struct {
    char filename[50];  // To store the name of the file (max 49 characters + null terminator)
    int openAttempts;   // To keep track of the number of attempts made to open the file
} FileInfo;

int main()
{
    FILE* file;  // Declare a file pointer

    // Declare and initialize a FileInfo struct
    FileInfo fileInfo;
    strcpy_s(fileInfo.filename, "file.txt");  // Assign the filename
    fileInfo.openAttempts = 0;              // Initialize openAttempts to 0

    // Loop to attempt opening the file for a maximum of 10 times
    for (fileInfo.openAttempts = 0; fileInfo.openAttempts < 10; fileInfo.openAttempts++) {
        // Try to open the file in read mode
        if (fopen_s(&file, fileInfo.filename, "r") == 0) {
            printf("File %s successfully opened.\n", fileInfo.filename);

            fclose(file);
            return 0;
        }
        else {
            // If the file couldn't be opened, print a failure message and the number of attempts so far
            printf("Attempt %d: File %s could not be opened. Trying again in 2 seconds...\n",
                fileInfo.openAttempts + 1, fileInfo.filename);
            Sleep(2000); // Pause the program for 2 seconds before the next attempt
        }
    }

    // After all attempts, if the file is still not opened, print a failure message
    printf("File %s could not be opened after %d attempts.\n", fileInfo.filename, fileInfo.openAttempts);

    return 1;  // Return 1 to indicate that the program completed with a failure
}
```

The difference between this struct and the previous version is how we've declared the **`FileInfo`** struct. This declares a struct type named **`FileInfo`**, and to use this type, we would have to always prefix it with the **`struct`** keyword, like this: **`struct FileInfo fileInfo;`**

```c
struct FileInfo {
    char filename[50];
    int openAttempts;
};
```
When we are specifying **typedef**, it still declares a new struct type that has the same members as before, but it also creates an alias **`FileInfo`** for this struct type. 

```c
typedef struct {
    char filename[50];
    int openAttempts;
} FileInfo;
```

Now, when we want to use this type, we can use the alias directly **without** the **struct** keyword: **`FileInfo fileInfo;`**.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/89cf3f19-64ab-4525-a2ad-f8156f7c14c4)
