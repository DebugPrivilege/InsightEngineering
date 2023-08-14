# What is Overloading Function Templates?

Function overloading is a feature that allows multiple functions of the same name but with different parameters. Function overloading can be viewed as a way to implement polymorphism, which is a core concept in object-oriented programming. Polymorphism is a concept in object-oriented programming where a single interface or method can represent different types of behaviors or objects. It allows objects of different types to be treated as objects, making code more flexible and extensible.

Function templates on the other hand are a way to achieve generic programming, which is a method of programming that allows functions to handle data of various types without having to rewrite the entire code for each type. When you combine these two, we get overloading function templates.

Overloading function templates in C++ involves several core concepts:

| Core Concept | Description | Example |
| --- | --- | --- |
| Function Templates | A blueprint for creating functions that can handle data of various types. | `template <typename T> T add(T a, T b) {...}` |
| Template Parameters | Placeholders for types or values within a function template. | `T` in `template <typename T>` |
| Function Overloading | A feature where two or more functions can have the same name but different parameters. | `void print(int i)` and `void print(double f)` |
| Template Specialization | A specific version of a function template defined for a particular type. | `template <> std::string add<std::string>(std::string a, std::string b) {...}` |
| Type Deduction | The compiler's automatic determination of the type of the template parameters based on the function arguments. | If `add(3, 4)` is called, the compiler deduces `T` is `int` |
| Compile-Time Polymorphism | The correct function to call is determined at compile time based on the function arguments. | If `add(3, 4)` is called, the `add` function for `int` is used |

# Code Sample

This example includes a function template for adding two numbers of any type, and an overloaded function template for concatenating two strings.

```c
#include <iostream>
#include <string>

// Function template for adding two numbers
template <typename T>
T add(T a, T b) {
    // Add the two numbers together and return the result
    return a + b;
}

// Overloaded function template for concatenating two strings
template <>
std::string add<std::string>(std::string a, std::string b) {
    // Concatenate the two strings and return the result
    return a + b;
}

int main() {
    // Test the function template with integers
    std::cout << "Addition of integers: " << add<int>(3, 4) << std::endl;

    // Test the function template with doubles
    std::cout << "Addition of doubles: " << add<double>(2.5, 3.5) << std::endl;

    // Test the overloaded function template with strings
    std::cout << "Concatenation of strings: " << add<std::string>("Hello, ", "World!") << std::endl;

    return 0;
}
```

Here, **`add`** is a **function template** that can add two numbers of any type. **`template <typename T>`** indicates that this is a template function, and **`T`** is a placeholder for any type. In the function definition, **`T add(T a, T b)`**, **`T`** is used as the type for the function's return value and both of its parameters.

```c
// Function template for adding two numbers
template <typename T>
T add(T a, T b) {
    // Add the two numbers together and return the result
    return a + b;
}
```

This is an overloaded version of **`add`** for **`std::string`**. It uses the template **`<>`** syntax to indicate that it is an explicit specialization of the function template for the type **`std::string`**. This function takes two **`std::string`** parameters, concatenates them using the **`+`** operator, and returns the result.

```c
// Overloaded function template for concatenating two strings
template <>
std::string add<std::string>(std::string a, std::string b) {
    // Concatenate the two strings and return the result
    return a + b;
}
```

**`main`** is the function where the program starts execution. In this function, we test the **`add`** function template with different types of parameters.

When **`add<int>(3, 4)`** is called, the compiler generates an **`add`** function that takes two **int** parameters and returns an **int**. Similarly, when **`add<double>(2.5, 3.5)`** is called, the compiler generates an **`add`** function that takes two **double** parameters and returns a double.

Finally, when **`add<std::string>("Hello, ", "World!")`** is called, the compiler uses the overloaded version of **`add`** that takes two **`std::string`** parameters and returns a **`std::string`**.

```c
int main() {
    // Test the function template with integers
    std::cout << "Addition of integers: " << add<int>(3, 4) << std::endl;

    // Test the function template with doubles
    std::cout << "Addition of doubles: " << add<double>(2.5, 3.5) << std::endl;

    // Test the overloaded function template with strings
    std::cout << "Concatenation of strings: " << add<std::string>("Hello, ", "World!") << std::endl;

    return 0;
}
```

The **std::cout** statements print the results of the add function calls to the console. 

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/07b98b2a-8d46-4b6c-a307-468f7993eeba)

# Function Templates - Template Parameters

Function templates are a feature in C++ that allow you to define a blueprint for a function without specifying the exact types of the inputs. This lets you create a function that can work with many different types of data without having to write separate versions of the function for each type.

A function template starts with the keyword **`template`** followed by a list of template parameters enclosed in angle brackets **`<...>`**. These template parameters act as placeholders that will be replaced by actual types when the function is used.

```c
template <typename T>
T add(T a, T b) {
    return a + b;
}
```

**`T`** is a **template parameter** that acts as a placeholder for any type. The **`add`** function can take two parameters of any type **`T`** and return a value of the same type. The type **`T`** is determined when the function is called. 

For instance, if we call **`add<int>(3, 4)`**, the compiler generates a version of **`add`** that works with **int** values. If you call **`add<double>(2.5, 3.5)`**, the compiler generates a version of **`add`** that works with **double** values. 

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/ce96c764-0d03-4426-b08b-d0281c1bc1ee)

The power of function templates in C++, as demonstrated by the **`add`** function, is that you can write a function once and then use it with multiple **different** types.

# Function Overloading

The function overloading occurs with the **`add`** function. There are two versions (or "overloads") of the **`add`** function:

- A generic version that can add two values of any type T, as long as the + operator is defined for that type. The function signature is **`T add(T a, T b)`**.
- A specialized version specifically for std::string type that concatenates two strings with a space between them. The function signature is std::string add<std::string>**(std::string a, std::string b)**.

These two functions are considered "overloads" of each other because they have the same name **(`add`)** but different parameters (one version takes any type **`T`**, and the other takes specifically **`std::string`**).

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/9636b3f1-6eae-4c31-80dc-89c378f7c02f)

# Template Specialization

This refers to the second **`add`** function, which is a specialized version of the **`add`** function template for the **`std::string`** type. Template specialization allows you to define a different implementation for a particular data type of the template. In this case, it's **`std::string`**. This function concatenates two strings with a space in between, which is specific to **`std::string`**.

The second **`add`** function has been highlighted here:

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/94f0b802-5ac9-4372-a317-8e87fb24e070)

# Type Deduction

Type deduction in C++ is a feature where the compiler automatically infers the type of an entity based on how it's used. To understand this better, let's have an example. The first code snippet is not using type deduction, because when you explicitly provide the template argument (e.g., **int**, **double**, **<std::string>**), the compiler does not need to deduce (figure out) the type because it is already provided.

```c
#include <iostream>
#include <string>

// Function template for adding two numbers
template <typename T>
T add(T a, T b) {
    // Add the two numbers together and return the result
    return a + b;
}

// Overloaded function template for concatenating two strings
template <>
std::string add<std::string>(std::string a, std::string b) {
    // Concatenate the two strings and return the result
    return a + b;
}

int main() {
    // Test the function template with integers
    std::cout << "Addition of integers: " << add<int>(3, 4) << std::endl;

    // Test the function template with doubles
    std::cout << "Addition of doubles: " << add<double>(2.5, 3.5) << std::endl;

    // Test the overloaded function template with strings
    std::cout << "Concatenation of strings: " << add<std::string>("Hello, ", "World!") << std::endl;

    return 0;
}
```

**IMPORTANT:** Pay a close attention on what has been highlighted within this code.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/0b0e6a09-d550-4115-bc43-e3cdfcea6465)

This modified version of the code, when we are calling the **`add`** function, we do not explicitly provide the template argument. Instead, we allow the compiler to deduce (figure out) the type **`T`** based on the types of the arguments passed to the function. 

For **`add(3, 4)`**, **`T`** is deduced to be **int**. For **`add(2.5, 3.5)`**, **`T`** is deduced to be double. And for **`add(std::string("Hello, "), std::string("World!"))`**, **`T`** is deduced to be **`std::string`**. 

```c
#include <iostream>
#include <string>

// Function template
template <typename T>
T add(T a, T b) {
    return a + b;
}

// Specialized version of the function template for strings
template <>
std::string add<std::string>(std::string a, std::string b) {
    return a + " " + b;
}

int main() {
    // Test the function template with integers
    std::cout << "Addition of integers: " << add(3, 4) << std::endl;  // Here, T is deduced to be int

    // Test the function template with doubles
    std::cout << "Addition of doubles: " << add(2.5, 3.5) << std::endl;  // Here, T is deduced to be double

    // Test the overloaded function template with strings
    std::string str1 = "Hello, ";
    std::string str2 = "World!";
    std::cout << "Concatenation of strings: " << add(str1, str2) << std::endl;  // Here, T is deduced to be std::string

    return 0;
}
```

Look closely at the code, and focus on the **main()** function. The primitive data types aren't explicitly mentioned in the function calls to **`add`**. Instead, the compiler uses type deduction to infer the type of the template argument based on the type of the function arguments.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/edf827cf-dfbf-454c-9785-8eb06bfab16e)

# Code Sample (2)

At this example, we have a different code snippet. Let's break down this code:

```c
#include <windows.h>  
#include <iostream>   
#include <string>     
#include <sstream>    

// Function template for writing data to a file
template <typename T>
void writeToFile(HANDLE fileHandle, const T& data) {
    DWORD bytesWritten;  // Variable to store the number of bytes written

    std::wstringstream ss;  // Create a wide string stream object
    ss << data;  // Insert the data into the string stream
    std::wstring dataStr = ss.str() + L"\n";  // Convert the stream to a wide string and append a newline character

    // Write the data to the file
    if (!WriteFile(
        fileHandle,
        dataStr.c_str(),
        dataStr.size() * sizeof(wchar_t),
        &bytesWritten,
        NULL
    )) {
        // If the write operation failed, print an error message with the error code
        std::wcout << L"Unable to write to file due to error: " << GetLastError() << L"\n";
    }
    else {
        // If the write operation succeeded, print a success message with the data written
        std::wcout << L"Successfully wrote '" << ss.str() << L"' to the file.\n";
    }
}

// Function to generate a random filename
std::wstring generateFilename() {
    srand(static_cast<unsigned int>(time(NULL)));  // Seed the random number generator with the current time
    int randNum = rand();  // Generate a random number
    std::wstring randNumStr = std::to_wstring(randNum);  // Convert the random number to a wide string
    return L"C:\\Temp\\testfile_" + randNumStr + L".txt";  // Return a filename composed of a fixed part and the random number
}

// Function to create a file and return its handle
HANDLE createFile(const std::wstring& filename) {
    HANDLE fileHandle = CreateFileW(
        filename.c_str(),
        GENERIC_WRITE,
        0,
        NULL,
        CREATE_NEW,
        FILE_ATTRIBUTE_NORMAL,
        NULL
    );

    // If the file creation failed, print an error message with the error code
    if (fileHandle == INVALID_HANDLE_VALUE) {
        std::wcout << L"Unable to create file due to error: " << GetLastError() << L"\n";
    }

    return fileHandle;  // Return the handle to the created file (or INVALID_HANDLE_VALUE if the creation failed)
}

// Function to close a file given its handle
void closeFile(HANDLE fileHandle) {
    if (fileHandle != INVALID_HANDLE_VALUE) {  // If the file handle is valid...
        CloseHandle(fileHandle);  // ...close the file
    }
}

// Main function
int main() {
    std::wstring filename = generateFilename();  // Generate a random filename
    HANDLE fileHandle = createFile(filename);  // Create a file with the generated filename

    // Write "Hello, World!" to the file
    writeToFile(fileHandle, L"Hello, World!");

    // Write 42 to the file
    writeToFile(fileHandle, 42);

    closeFile(fileHandle); 

    return 0; 
}
```

**Function Template**

A function template is a blueprint for creating functions. It allows you to create a function that can operate on different data types. At this example, **`writeToFile`** is a function template. The template (`<typename T>`) line before the function definition indicates that it's a **function template**.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/5176f4c7-be5c-4d30-8a88-d3fdbc5a5a18)

**Template Parameters**

Template parameters are the placeholders in a template. **`T`** is a template parameter. It's a type parameter that can represent any type. In the **`writeToFile`** function, **`T`** can be any type that can be inserted into a **`std::wstringstream`**

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/b09d239c-0fa4-40ae-a214-24a27eb5383f)

**Type Deduction**

Type deduction is the process by which the C++ compiler determines the type of an object or expression. In this example, type deduction is used in the **`writeToFile`** calls in the main function. When you call **`writeToFile(fileHandle, L"Hello, World!")`**, the compiler deduces (figures out) that **`T`** is **`std::wstring`** (the type of L"Hello, World!")

The same things applies for **`writeToFile(fileHandle, 42)`** as well. The compiler deduces (figure out) that **`T`** is **int** (the type of 42).

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/f1a677e9-60d7-48b6-9b18-e3f57c8b38c2)

# Good Questions

Some good questions based on our code, which may clarify things better.

```c
#include <windows.h>  
#include <iostream>   
#include <string>     
#include <sstream>    

// Function template for writing data to a file
template <typename T>
void writeToFile(HANDLE fileHandle, const T& data) {
    DWORD bytesWritten;  // Variable to store the number of bytes written

    std::wstringstream ss;  // Create a wide string stream object
    ss << data;  // Insert the data into the string stream
    std::wstring dataStr = ss.str() + L"\n";  // Convert the stream to a wide string and append a newline character

    // Write the data to the file
    if (!WriteFile(
        fileHandle,
        dataStr.c_str(),
        dataStr.size() * sizeof(wchar_t),
        &bytesWritten,
        NULL
    )) {
        // If the write operation failed, print an error message with the error code
        std::wcout << L"Unable to write to file due to error: " << GetLastError() << L"\n";
    }
    else {
        // If the write operation succeeded, print a success message with the data written
        std::wcout << L"Successfully wrote '" << ss.str() << L"' to the file.\n";
    }
}

// Function to generate a random filename
std::wstring generateFilename() {
    srand(static_cast<unsigned int>(time(NULL)));  // Seed the random number generator with the current time
    int randNum = rand();  // Generate a random number
    std::wstring randNumStr = std::to_wstring(randNum);  // Convert the random number to a wide string
    return L"C:\\Temp\\testfile_" + randNumStr + L".txt";  // Return a filename composed of a fixed part and the random number
}

// Function to create a file and return its handle
HANDLE createFile(const std::wstring& filename) {
    HANDLE fileHandle = CreateFileW(
        filename.c_str(),
        GENERIC_WRITE,
        0,
        NULL,
        CREATE_NEW,
        FILE_ATTRIBUTE_NORMAL,
        NULL
    );

    // If the file creation failed, print an error message with the error code
    if (fileHandle == INVALID_HANDLE_VALUE) {
        std::wcout << L"Unable to create file due to error: " << GetLastError() << L"\n";
    }

    return fileHandle;  // Return the handle to the created file (or INVALID_HANDLE_VALUE if the creation failed)
}

// Function to close a file given its handle
void closeFile(HANDLE fileHandle) {
    if (fileHandle != INVALID_HANDLE_VALUE) {  // If the file handle is valid...
        CloseHandle(fileHandle);  // ...close the file
    }
}

// Main function
int main() {
    std::wstring filename = generateFilename();  // Generate a random filename
    HANDLE fileHandle = createFile(filename);  // Create a file with the generated filename

    // Write "Hello, World!" to the file
    writeToFile(fileHandle, L"Hello, World!");

    // Write 42 to the file
    writeToFile(fileHandle, 42);

    closeFile(fileHandle); 

    return 0; 
}
```

- **Why do we only use the template keyword for the writeFile function, and not for createFile?**

The **`template`** keyword is used when we want to define a function template, which is a way of making a function work for multiple types of data. In the case of the **`writeToFile`** function, we use template (<typename T>) because we want this function to be able to write different types of data to a file. 

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/5787d50f-ae0c-41b2-8e68-d53bd36e7420)

On the other hand, the **`createFile`** function does not need to be a template function because it is designed to work with a specific type of data: a wide string **(`std::wstring`)** representing a filename. This function does not need to work with different types of data, so there's no need to make it a template function.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/a82551da-fe56-4c77-b974-f4335d3ebabd)

- **What happens when we call writeToFile function?**

Not really related to the topic, but you may be wondering what these lines of code are doing:

The **`std::wstringstream`** object and the **`ss << data;`** line are used to convert the data of any type into a wide string **`(std::wstring)`**, which can then be written to a file.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/dd682ee2-7842-4fc8-8406-343c679cb7e1)



This line creates an instance of **`std::wstringstream`**, which is a stream class to operate on wide strings. You can think of a **`std::wstringstream`** as a high-level string builder, where you can insert various types of data (like int, float, std::wstring, etc.) and it will convert that data into a string format.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/91b9dc07-f75a-49f9-8256-c4120481725b)


This line is effectively converting the data to a wide string representation. The **`<<`** operator is overloaded for **`std::wstringstream`** to handle various types of data, and it will transform the data into a string format.

```c
ss << data;  // Insert the data into the string stream
```

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/5a485160-69d7-4216-aa7b-631146c4d65e)
