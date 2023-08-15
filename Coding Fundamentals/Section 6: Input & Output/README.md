# What are Input/Output (I/O) functions?

Input/Output (I/O) functions are used for communication between a program and the external world, such as reading input from the user or writing output to the screen or files. These functions enable interaction between the program and the user. 

Here are **common** I/O functions:

| Function  | Description  | Example |
|---|---|---|
| printf()  | Writes formatted output to the standard output device (usually screen) | `printf("Hello, %s!", name);` |
| scanf()   | Reads formatted input from the standard input device (usually keyboard) | `scanf("%d", &num);` |
| getchar() | Reads a single character from the standard input device (keyboard) | `char ch = getchar();` |
| putchar() | Writes a single character to the standard output device (screen) | `putchar('A');` |
| fgets()   | Reads a string from the standard input device (until newline or EOF or the specified size) | `char str[50]; fgets(str, sizeof(str), stdin);` |
| puts()    | Writes a string to the standard output device (with newline) | `puts("Hello, World!");` |
| fscanf_s()| Reads formatted input from a file | `fscanf(file, "%s %d", name, &age);` |
| fprintf() | Writes formatted output to a file | `fprintf(file, "Hello, %s!", name);` |
| sprintf() | Writes formatted output to a string | `sprintf(buffer, "The value is: %d", value);` |
| getc()    | Reads a character from a specified input stream | `char ch = getc(file);` |
| fgetc()   | Reads a character from a specified file stream | `char ch = fgetc(file);` |



# printf

**printf** is a used function in the C programming language for displaying formatted output. It stands for "print formatted" and it is part of the standard input/output library **(stdio.h)**.

```c
#include <stdio.h>

int main() {
    int age = 25;
    float height = 1.75;

    printf("My age is %d and my height is %.2f meters.", age, height);

    return 0;
}
```

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/4cf888bb-070c-4ae3-8127-3b9dfb91e0b7)


# scanf

**scanf** is a function in the C programming language used for reading formatted input from the standard input (typically the keyboard) or other input streams. It is part of the standard input/output library **(stdio.h)**. The **scanf** function allows us to read input and store the values into variables based on the format specified in the format string. 

```c
#include <stdio.h>

int main() {
    int age;
    float height;

    printf("Enter your age: ");
    scanf_s("%d", &age);

    printf("Enter your height in meters: ");
    scanf_s("%f", &height);

    printf("Your age is %d and your height is %.2f meters.", age, height);

    return 0;
}
```
**scanf** is used to read user input for the **age** and **height** variables. The format specifiers **%d** and **%f** in the **scanf** calls specify that an **integer** and a **floating-point** number should be read.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/45c5501a-bcb7-4894-8d05-44006c638ce8)


# getchar

**getchar()** is a function in the C programming language that is used to read a **single** character from the standard input device (which is typically the keyboard). The **getchar()** function waits for user input, and it's often used in loops to read multiple characters from input. **getchar()** function reads a character from input, and the **putchar()** function writes that character to the output.

```c
#include <stdio.h>

int main() {
    int c;

    printf("Please enter a character: ");

    // read a character
    c = getchar();

    // print the character
    printf("\nYou entered: ");
    putchar(c);

    return 0;
}
```

**getchar()** function only reads a single character at a time from the input. If we enter "Hello", the first call to **getchar()** would only read the **'H'**.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/5e53304d-8bc9-4ff2-abf5-ba9c39535d56)


 If we want to read the entire word, we need to use a loop that calls **getchar()** multiple times.

```c
 #include <stdio.h>

int main() {
    int ch;
    printf("Please enter a word: ");

    // read characters until '\n' or EOF is encountered
    while ((ch = getchar()) != '\n' && ch != EOF) {
        putchar(ch);
    }

    return 0;
}
```

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/b61e21d8-f704-432f-bd94-124e0d99118f)


# fgets

The fgets() function in C is a standard library function that reads a string from the specified stream into the array pointed to by **str**. Streams are entities that a program can use to either input data into the program or output data from the program.

```c
#include <stdio.h>

int main() {
    char str[100];

    printf("Enter a line of text: \n");
    fgets(str, sizeof(str), stdin);

    // Print the entered text
    printf("You entered: \n%s", str);

    return 0;
}
```

**fgets()** function reads a line of text entered by the user, up to 99 character. It then prints out the line of text that was entered. If the user enters more than 99 characters, only the first 99 will be stored in the **str** array.

The **stdin** stream is used as the input source for the **fgets()** function. This means **fgets()** will read from standard input, which is usually the keyboard.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/0d59a2b1-387d-4180-8f56-718ea54ee8c7)


# puts

The **puts()** function is a standard library function in C programming that writes a string to the standard output (stdout). 

```c
#include <stdio.h>

int main() {
    char str[100] = "Hello, World!";

    // Use puts() to write the string to stdout
    puts(str);

    return 0;
}
```
We first define a string "Hello, World!" and store it in the array **str**. Then we call **puts(str)**, which writes the string to the standard output (console).

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/ac18f710-9f8a-4a0a-9667-6284dc3ec779)


# fscanf

The **fscanf()** function is a function in the C standard library that **reads formatted input from a file**. It is very similar to the scanf() function which reads formatted input from the standard input (stdin), but fscanf() can read from any **file** stream.

```c
#include <stdio.h>
#include <string.h>


int main() {
    
    // Declare a FILE pointer
    FILE* file;

    // Declare a character array to hold strings read from the file
    char str[100];

    // Declare a character array to hold the file path
    char filePath[200];

    // Declare an error variable for checking file opening
    errno_t err;

    // Prompt user to enter the file path
    printf("Please enter the full path to the file: ");

    // Use fgets to read the file path input from the user
    fgets(filePath, 200, stdin);

    // Remove newline character from the end of string read by fgets
    size_t len = strlen(filePath);
    if (len > 0 && filePath[len - 1] == '\n') {
        filePath[len - 1] = '\0';
    }

    // Use fopen_s to open the file and store the returned error code
    err = fopen_s(&file, filePath, "r");

    // If an error occurred when opening the file, print error and exit
    if (err != 0 || file == NULL) {
        printf("Cannot open file \n");
        return 0;
    }

    // Read lines from the file until EOF is reached
    // The %99[^\n] format specifier tells fscanf_s to read up to 99 characters or until a newline is encountered
    while (fscanf_s(file, "%99[^\n]\n", str, (unsigned)sizeof(str)) != EOF) {
        // Print each line read from the file
        printf("Read string: %s \n", str);
    }

    fclose(file);

    return 0;
}
```
At this example, we have a notepad file that contains two lines.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/082d06eb-fd42-42e8-acd9-52cee6ca03ed)


Let's break down the code in small pieces:

This line declares a **file** pointer which will be used to refer to the file that you'll open for reading.

```c
FILE* file;
```

This declares a character array **str** which is used to store strings that are read from the file.

```c
char str[100];
```

This declares a character array **filePath** which will store the path of the file that we are going to open.

```c
char filePath[200];
```

This line reads the file path from standard input (the keyboard) and stores it in the **filePath** variable.

```c
fgets(filePath, 200, stdin);
```

The **while** loop reads lines from the file until the end of the file is reached (EOF). **fscanf_s(file, "%99[^\n]\n", str, (unsigned)sizeof(str))** reads a line from the file and stores it in **str**. It reads up to 99 characters or until a newline is encountered.

```c
    while (fscanf_s(file, "%99[^\n]\n", str, (unsigned)sizeof(str)) != EOF) {
        // Print each line read from the file
        printf("Read string: %s \n", str);
```

This line prints each line read from the file.

```c
printf("Read string: %s \n", str);
```

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/f1231d29-1ea4-4f35-875d-1f16426bd398)



# fprintf

**fprintf** is a function in the C programming language found in the **stdio.h** library that allows for formatted data output to a specific file stream. The name stands for "file print formatted" and it's most often used for writing formatted text to files

```c
#include <stdio.h>  

int main() {  
    
    FILE* fp;  // Declare a pointer to a FILE structure
    const char* message = "Hello, World!";  // Declare a constant string
    errno_t err;  // Declare a variable to store the error code

    // Try to open the file 'file.txt' in write mode, storing any error code
    err = fopen_s(&fp, "file.txt", "w");

    // If an error occurred while opening the file (err != 0)
    if (err != 0) {
        printf("Error opening file!\n");  // Print an error message to the console
        return 1;  // Exit the program with status code 1 to signify an error
    }

    // Write a formatted string to the file, using 'message' as a substitute for the %s specifier
    fprintf(fp, "This is a message: %s\n", message);

    fclose(fp); 

    // Print a message to the console to inform the user that the message was successfully written
    printf("Message written to file.txt\n");

    return 0;  
}  

```

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/39f47b45-445f-4175-b58e-453362a19f22)



A file is created on disk when the **fopen_s** function is invoked with the "w" (write) mode. If the file named "file.txt" does not exist, it will be created. If it does exist, it will be truncated. Once the file is opened, the **fprintf** function is used to write a message into the file.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/be2bf672-6f5c-446a-a3d8-156691c60b26)


# sprintf

The **sprintf** function in C is similar to printf, but instead of sending output to stdout (the console), it formats and stores the output in a **character array** or **string**.

```c
#include <stdio.h>

int main() {
    char buffer[50];  // Declare a buffer to hold the formatted string

    // Use sprintf_s to store "Hello, World!" in the buffer
    // The size of the buffer is specified to prevent overflow
    sprintf_s(buffer, sizeof(buffer), "Hello, World!");

    // Print the contents of the buffer to the console
    printf("Formatted string: %s\n", buffer);

    return 0;
}
```

This code creates a character array named **buffer** to hold the formatted string. It then uses **sprintf_s** to store the string "Hello, World!" in buffer. The size of buffer is specified in the call to **sprintf_s** to prevent buffer overflows.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/b0bbccc4-8c34-4979-b7f4-6c94dc2dfd6d)

