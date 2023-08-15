# What are the C++ Standard Libraries?

The C++ Standard Library is a collection of classes, functions, constants, and templates defined as part of the C++ Standard. These components provide common programming functionalities, such as input/output processing, string and numeric data manipulation, container data structures, and much more. 

Here are some **common** examples:

| Library/Header | Description | Example |
|---|---|---|
| `<iostream>` | Defines objects like `std::cin` for input and `std::cout` for output. | `std::string input; std::cin >> input; std::cout << "You entered: " << input << std::endl;` |
| `<fstream>` | Provides classes for reading from and writing to files. | `std::ofstream myfile("example.txt"); myfile << "Writing to file"; myfile.close();` |
| `<string>` | Provides `std::string` for handling ASCII strings and `std::wstring` for handling wide character strings. | `std::string str = "Hello, world!";` |
| `<vector>` | Provides `std::vector`, a dynamic array that can grow and shrink at runtime. | `std::vector<int> vec = {1, 2, 3}; vec.push_back(4);` |
| `<map>` | Provides `std::map` and `std::multimap`, sorted associative containers that map keys to values. | `std::map<std::string, int> mymap = {{"apple", 1}, {"banana", 2}}; mymap["cherry"] = 3;` |
| `<exception>` | Provides classes and functions to handle exceptions. | `try { throw std::runtime_error("Error"); } catch (const std::exception& e) { std::cerr << e.what(); }` |
| `<memory>` | Provides utilities for managing memory, including smart pointers like `std::unique_ptr` and `std::shared_ptr` which automatically deallocate memory when they're no longer in use. | `std::unique_ptr<int> ptr(new int(10)); //ptr will be automatically deleted when it goes out of scope` |


# iostream

The **`<iostream>`** is a header file in the Standard Library of C++ that provides functionalities for standard input and output operations. It is one of the most commonly used libraries in C++ as it allows interaction with users and reading from or writing to files.

Here are a few common examples:

- **`std::cin:`** This is an instance of std::istream (input stream) tied with the standard input device, usually the keyboard. It is used to read input.
- **`std::cout:`** This is an instance of std::ostream (output stream) tied with the standard output device, usually the console. It is used to produce output.
- **`std::cerr:`** This is an instance of std::ostream tied with the standard error device, which is also the console but does not use a buffer. This means that output sent to cerr will be displayed immediately, and it is often used for error messages.

```c
#include <iostream>

int main() {
    int number;

    std::cout << "Please enter a number: ";

    if (!(std::cin >> number)) {
        std::cerr << "You did not enter a valid number!" << std::endl;
        return 1; // return a non-zero exit code to indicate an error
    }

    std::cout << "You entered " << number << std::endl;

    return 0;
}
```

**`std::cout`** is used to **print** messages to the console, prompting the user to enter a number and then confirming the number that was entered. **`std::cin`** is used to read the number **inputted** by the user from the console. If the user's input isn't a valid number, **`std::cerr`** is used to **print an error message** to the console. 

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/d3368d51-0b04-46f7-ad1a-6b30136ee7e8)


If we type invalid characters when the program is expecting an **integer** input, the condition **`if (!(std::cin >> number))`** will be true and the program will execute the code inside the **`if`** statement. This code includes the line **`std::cerr << "You did not enter a valid number!" <<`** 

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/aecd89a7-ba19-4cca-afed-1dac896101ea)


# fstream

The **`<fstream>`** header file in the C++ Standard Library provides functionality for file input and output through stream operations. Using these file streams, C++ programs are able to create, read, and write to files.

Here are a few common examples:

- **std::ifstream:** This is an input file stream that is used to read information from files.
- **std::ofstream:** This is an output file stream that is used to create files and write information to files. 
- **std::fstream:** This is a general file stream that supports both input and output operations.

```c
#include <fstream>
#include <iostream>
#include <string>

int main() {
    std::ofstream outFile("C:\\Temp\\HelloWorld.txt");
    if (outFile) { // check if the file was opened successfully
        outFile << "Hello World!\n";
        outFile.close();
    }
    else {
        std::cerr << "Could not open file for writing\n";
        return 1; // return a non-zero exit code to indicate an error
    }

    std::string line;
    std::ifstream inFile("C:\\Temp\\HelloWorld.txt");
    if (inFile) { // check if the file was opened successfully
        while (std::getline(inFile, line)) {
            std::cout << line << '\n';
        }
        inFile.close();
    }
    else {
        std::cerr << "Could not open file for reading\n";
        return 1; // return a non-zero exit code to indicate an error
    }

    return 0;
}
```

**`std::ofstream:`** is a class that represents an output file stream and is used to create and write to files. An object of **`std::ofstream`** is created with the name **`outFile`**, which is then used to open the file **`C:\\Temp\\HelloWorld.txt`** for writing. If the file is opened successfully, the string **`"Hello World!\n"`** is written to it.

**`std::ifstream:`** is a class that represents an input file stream and is used to read data from files. An object of **`std::ifstream`** is created with the name **`inFile`**, which is then used to open the file **`C:\\Temp\\HelloWorld.txt`** for reading. If the file is opened successfully, the lines in the file are read one by one until the end of the file is reached.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/a7f7a7eb-2fab-4640-82a0-65df815a339f)

![image](https://private-user-images.githubusercontent.com/63166600/255331037-1d8bf47c-fd0e-4b88-af0a-c32aa206c87a.png?jwt=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJnaXRodWIuY29tIiwiYXVkIjoicmF3LmdpdGh1YnVzZXJjb250ZW50LmNvbSIsImtleSI6ImtleTEiLCJleHAiOjE2OTIwOTA3NTUsIm5iZiI6MTY5MjA5MDQ1NSwicGF0aCI6Ii82MzE2NjYwMC8yNTUzMzEwMzctMWQ4YmY0N2MtZmQwZS00Yjg4LWFmMGEtYzMyYWEyMDZjODdhLnBuZz9YLUFtei1BbGdvcml0aG09QVdTNC1ITUFDLVNIQTI1NiZYLUFtei1DcmVkZW50aWFsPUFLSUFJV05KWUFYNENTVkVINTNBJTJGMjAyMzA4MTUlMkZ1cy1lYXN0LTElMkZzMyUyRmF3czRfcmVxdWVzdCZYLUFtei1EYXRlPTIwMjMwODE1VDA5MDczNVomWC1BbXotRXhwaXJlcz0zMDAmWC1BbXotU2lnbmF0dXJlPTgxNmI1OTI5MDYxOWJmNjkwMzIxYzJhYWQ1YTg0OGNiZTQ4MjgzODQ4NWMzOGJiZmExYWE4NDgxYjA2MzkwNjMmWC1BbXotU2lnbmVkSGVhZGVycz1ob3N0JmFjdG9yX2lkPTAma2V5X2lkPTAmcmVwb19pZD0wIn0.hyHPZkpEQE09FOtgmloo-tF7RHvD7Wg3Ubbf1YNI8nw)

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/06bb37f3-b0b4-473d-bcd5-ebc1ed3c7182)




# string

The **`<string>`** header file in the C++ Standard Library contains the definitions of two important classes for string manipulation: **`std::string`** and **`std::wstring`**.

- **`std::string:`** This class represents a sequence of characters. It's used for manipulating ASCII strings. **`std::string`** provides a wide range of functionalities such as accessing and modifying individual characters, comparing strings, searching and replacing substrings, and more.
- **`std::wstring:`** This class is similar to **`std::string`**, but it represents a sequence of wide characters.

```c
#include <string>
#include <iostream>

int main() {
    std::string greeting = "Hello, world!";
    std::cout << greeting << std::endl;
    return 0;
}
```

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/57cd7c92-7dfb-43d3-8b3e-1ac6da96cbc7)

# vector

The **`<vector>`** library in C++ is part of the Standard Template Library. It provides the **`std::vector`** container class, which implements a **dynamic array**. This means that it can resize itself automatically when an element is inserted or deleted, with the storage being handled automatically by the container.

A **`std::vector`** is a container like array, list, and deque, but with the added feature that its size can change dynamically. Here are a few **common** operations that can be performed with **`std::vector`**:

- **push_back():** Adds an item at the end of the vector.
- **pop_back():** Removes the last item of the vector.
- **begin():** Gives you a way to look at the first item.
- **end():** Gives you a way to look at the space after the last item.
- **insert():** Adds an item at a certain spot in the vector.

This code creates a vector, adds some numbers to it, and then prints them out.

```c
#include <iostream>
#include <vector>

int main() {
    // Create an empty vector
    std::vector<int> numbers;

    // Add numbers 1-5 to the vector
    for (int i = 1; i <= 5; ++i) {
        numbers.push_back(i);
    }

    // Print out the numbers
    std::cout << "The numbers in the vector are: ";
    for (int i = 0; i < numbers.size(); ++i) {
        std::cout << numbers[i] << ' ';
    }

    return 0;
}
```

This demonstrates the basic usage of **`std::vector`**, including creating a vector, adding elements to it using **`push_back()`**, and accessing its elements using the indexing operator **`[]`**.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/fc08c6ab-0dd8-4c09-9b5a-6b5114a641c0)

# map

The **`<map>`** library in C++ is part of the Standard Template Library. It provides the **`std::map`** container, which stores elements formed by a combination of a key value and a mapped value, following a specific order.

```c
#include <iostream>
#include <map>

int main() {
    // Create a map of strings to ints
    std::map<std::string, int> ages;

    // Add some elements to the map
    ages["Alice"] = 20;
    ages["Bob"] = 25;
    ages["Charlie"] = 30;

    // Print out the elements
    for (const auto& pair : ages) {
        std::cout << pair.first << " is " << pair.second << " years old.\n";
    }

    return 0;
}
```
What do we exactly mean by ***a combination of a key value and a mapped value***?

This code creates a **`std::map`** where the **keys** are names (strings) and the **values** are ages (ints). It adds some elements to the **`std::map`** and then prints them out. Each element is a **`pair`**, where **`pair.first`** is the **key** and **`pair.second`** is the value.

This **`for`** loop goes through each name-age pair in the ages map. For each pair, it prints the name and age in the format "Name is Age years old."

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/add0dada-0265-49c3-9f1f-cc1c288aff1b)

The **`auto`** keyword is being used to automatically figure out the type of elements in the **`ages`** map. The compiler sees that **`ages`** is a **`std::map<std::string, int>`**, so it understands that pair will be a pair of a **string** and an **integer**.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/54e740d9-8ac3-4d9f-b176-ddb4fb14ccfb)

# exception

The **`<exception>`** library in C++ is a header file in the C++ Standard Library that provides a base class specifically designed to declare objects to be thrown as exceptions.

In this example, we throw a **`std::runtime_error`**, which is a specific type of exception that derives from **`std::exception`**. We then catch it as a **`std::exception`**. The **`what()`** function is called on the caught exception to **print a description of the exception**, which in this case will be *"A runtime error occurred"*.

```c
#include <iostream>
#include <stdexcept>

int main() {
    try {
        throw std::runtime_error("A runtime error occurred");
    }
    catch (std::exception& e) {
        std::cout << "Caught an exception: " << e.what() << std::endl;
    }

    return 0;
}
```

This example demonstrates how we can catch any exception derived from **`std::exception`** using a catch block for **`std::exception`**, and how you can use the **`what()`** function to get a description of the exception.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/d3c367a7-4a7f-4fc2-8fa8-7cbbf2025e9b)

# memory

The **`<memory>`** library in C++ is part of the Standard Library that provides utilities for managing memory. This includes features for manipulating raw memory, as well as classes and utilities for working with dynamically and automatically managed objects.

The number one major component of this library are **Smart Pointers** which provides a more automated and robust way of handling memory management. 

This C++ code prompts the user for a filename, then creates and writes a **"Hello, World!"** message to a new file of that name in the **C:\\Temp\\** directory on a Windows system. It leverages smart pointers (**`std::unique_ptr`**) to automatically manage the file handle resource, ensuring that the file handle is properly closed when the operation completes or if an error occurs.

```c
#include <windows.h>
#include <iostream>
#include <string>
#include <memory>

int main() {
    // Get filename from user
    std::wcout << L"Enter filename: ";
    std::wstring filename;
    std::getline(std::wcin, filename);

    // Prepend the path to the filename
    filename = L"C:\\Temp\\" + filename;

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

The benefit of using **Smart Pointers** in this case is that they provide automatic life cycle management of the file handle. The **`std::unique_ptr`** will automatically close the file handle when it is no longer in use, so there's no need to manually close it. This is extremely useful in more complex programs where it can be easy to forget to close handles, or in cases where a function might have multiple return paths and manually closing the handle on each one can lead to mistakes and handle leaks. 

By leveraging the **`std::unique_ptr's`** automatic destructor, the code ensures that the file handle is always closed properly, which is a crucial aspect of resource management in programming.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/7b181570-e71f-4224-9b8a-a9ea8070c331)

