# What is Inheritance?

Inheritance in C++ is a concept from Object-Oriented Programming. It allows you to create a new class, called a 'child' or 'derived' class, that automatically gets all the variables and functions of an existing class, known as the 'parent' or 'base' class.

The main idea here is that if you have types of objects that are similar to each other, you don't have to repeat the same code for each one. You can create a general class that has all the common parts, and then create specific classes that inherit from the general class and add or change parts as needed.

We will not cover every core concept of inheritance, but these concepts will be covered:

- **Base Class (Parent Class):**
This is the class that is being inherited from. It provides a basic version of the functionality that can be shared among multiple derived classes.

- **Derived Class (Child Class):**
This is a class that inherits from another class. It takes all the functionality from its base class and can also add new functionality or modify the inherited functionality to suit its needs.

- **Access Specifiers:**
These define the visibility of the base class's members in the derived class. The three access specifiers are **public**, **protected**, and **private**.

# Code Sample (1)

This code demonstrates the principle of inheritance: the **Striker** and **Grappler** classes inherit members and methods from the **Fighter** base class and add their own unique methods. It also shows how to use the **protected** keyword to allow access to base class members in derived classes, and how to use constructors in inheritance.

```c
#include <iostream>
#include <string>

// Base class
class Fighter {
protected:
    std::string name;

public:
    // Constructor
    Fighter(std::string n) : name(n) {}

    // Public method
    void train() {
        std::cout << name << " is training.\n\n";
    }
};

// Derived class
class Striker : public Fighter {
public:
    // Constructor
    Striker(std::string n) : Fighter(n) {}

    // Additional method
    void boxing() {
        std::cout << name << " is practicing boxing.\n\n";
    }
};

// Another derived class
class Grappler : public Fighter {
public:
    // Constructor
    Grappler(std::string n) : Fighter(n) {}

    // Additional method
    void wrestling() {
        std::cout << name << " is practicing wrestling.\n\n";
    }
};

int main() {
    // Create a Striker object
    Striker striker("Conor McGregor");
    striker.train();  // method inherited from base class
    striker.boxing();  // method from Striker class

    // Create a Grappler object
    Grappler grappler("Khabib Nurmagomedov");
    grappler.train();  // method inherited from base class
    grappler.wrestling(); // method from Grappler class

    return 0;
}
```

**Base Class (Fighter)**

This is the base class that other classes will inherit from. It has one data member **`name`**, which is protected (meaning it can be accessed directly within this class and its derived classes), and one function **`train`**.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/40d10332-5753-4b24-a936-26142125b331)

**Derived Class (Striker)**

This class inherits from **`Fighter`** and adds a new function **`boxing`**. The constructor takes a string as a parameter and passes it to the **`Fighter`** constructor to set the **`name`**.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/79e7e4f4-88a8-4d2f-9ec7-f753510f48c5)

**Another Derived Class (Grappler)** 

This class also inherits from **`Fighter`** and adds a new function **`wrestling`**. Like the **`Striker`** class, its constructor takes a string as a parameter and passes it to the **`Fighter`** constructor to set the **`name`**.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/5f99fc5d-c5c1-4265-b2d0-80af10a0cac8)

**main Function:** 

In the **`main`** function, we create objects of the **`Striker`** and **`Grappler`** classes and call their methods. This demonstrates how objects of the derived classes can use the methods of the base class **`(train)`** as well as their own unique methods **`(boxing and wrestling)`**.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/0853ffdc-a8eb-4fd4-9d92-90ba6803e330)

Let's go a bit into detail. We will take this line as an example:

```c
grappler.wrestling();
```

This is an example of calling a method that's specific to the **`Grappler`** class.

- **`grappler`** is an instance of the **`Grappler`** class.
- **`wrestling()`** is a method defined in the **`Grappler`** class.

So, when we call **`grappler.wrestling();`**, we're calling the **`wrestling`** method on the **`grappler`** instance. This line of code is not directly related to inheritance. 

However, the fact that the **`grappler`** object can access the **`wrestling`** method is a result of the **`grappler`** object being an instance of the **`Grappler`** class, which is a derived class of the **`Fighter`** base class.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/6349b8cc-a17f-4d6e-92d2-d8ae31b1834b)

This is how it looks like when we run this code:

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/c7af42f2-3513-465c-8ad8-e182cc042fb7)


# Code Sample (2)

This C++ code uses inheritance to define two types of file writers: **`TextFileWriter`** for writing new text files and **`LogFileWriter`** for adding to existing log files. Both classes inherit from a base **`FileWriter`** class and provide their own implementation of a **`createAndWrite()`** method.

```c
#include <windows.h>
#include <iostream>
#include <string>
#include <time.h>

// Base class
class FileWriter {
protected:
    std::wstring filename;  // Filename
    HANDLE fileHandle;  // File handle

public:
    // Constructor that generates a random filename
    FileWriter(std::wstring prefix) : fileHandle(INVALID_HANDLE_VALUE) {
        srand(static_cast<unsigned int>(time(NULL)));
        int randNum = rand();
        std::wstring randNumStr = std::to_wstring(randNum);
        filename = L"C:\\Temp\\" + prefix + randNumStr + L".txt";
    }

    // Pure virtual function for creating and writing to a file
    virtual void createAndWrite(std::wstring data) = 0;

    // Method to close the file
    void closeFile() {
        if (fileHandle != INVALID_HANDLE_VALUE) {
            CloseHandle(fileHandle);
        }
    }

    // Destructor that ensures the file handle is closed if it was opened
    virtual ~FileWriter() {
        closeFile();
    }
};

// Derived class for text file writing
class TextFileWriter : public FileWriter {
public:
    // Constructor
    TextFileWriter() : FileWriter(L"testfile_") {}

    // Method for creating and writing to a text file
    void createAndWrite(std::wstring data) override {
        // Create the file
        fileHandle = CreateFileW(
            filename.c_str(),
            GENERIC_WRITE,
            0,
            NULL,
            CREATE_ALWAYS,
            FILE_ATTRIBUTE_NORMAL,
            NULL
        );

        // Check if the file was created successfully
        if (fileHandle == INVALID_HANDLE_VALUE) {
            std::wcout << L"Unable to create file due to error: " << GetLastError() << "\n";
            return;
        }

        DWORD bytesWritten;

        // Write to the file
        if (!WriteFile(
            fileHandle,
            data.c_str(),
            data.size() * sizeof(wchar_t),
            &bytesWritten,
            NULL
        )) {
            std::wcout << L"Unable to write to file due to error: " << GetLastError() << "\n";
        }
        else {
            std::wcout << L"File created and written successfully.\n";
        }
    }
};

// Derived class for log file writing
class LogFileWriter : public FileWriter {
public:
    // Constructor
    LogFileWriter() : FileWriter(L"logfile_") {}

    // Method for adding data to the end of a log file
    void createAndWrite(std::wstring data) override {
        // Create the file
        fileHandle = CreateFileW(
            filename.c_str(),
            FILE_APPEND_DATA,
            0,
            NULL,
            CREATE_ALWAYS,
            FILE_ATTRIBUTE_NORMAL,
            NULL
        );

        // Check if the file was opened successfully
        if (fileHandle == INVALID_HANDLE_VALUE) {
            std::wcout << L"Unable to open file due to error: " << GetLastError() << "\n";
            return;
        }

        DWORD bytesWritten;

        // Append to the file
        if (!WriteFile(
            fileHandle,
            data.c_str(),
            data.size() * sizeof(wchar_t),
            &bytesWritten,
            NULL
        )) {
            std::wcout << L"Unable to write to file due to error: " << GetLastError() << "\n";
        }
        else {
            std::wcout << L"Log entry written successfully.\n";
        }
    }
};

int main() {
    // Create a TextFileWriter object and write some text to a file
    TextFileWriter textWriter;
    textWriter.createAndWrite(L"Hello, World!");

    // Create a LogFileWriter object and a log entry to a file
    LogFileWriter logWriter;
    logWriter.createAndWrite(L"Log entry");

    return 0;
}
```

**Base Class (FileWriter)**

**`FileWriter`** is the base class that provides a structure for file writing operations. It includes a constructor that generates a random filename and a pure virtual function **`createAndWrite()`**, which derived classes must override.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/4473ef88-12ce-4d4c-8820-7b3a7377edb6)

**Derived Class (LogFileWriter)**

**`LogFileWriter`** is another derived class from **`FileWriter`**. It also overrides the **`createAndWrite()`** function, but in this case, it's designed to add data to an existing file or create a new one if it doesn't exist:

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/d98d1554-a29e-45cf-a7b5-dd56254ff1e8)


**main Function**

In the **`main()`** function, objects of both **`TextFileWriter`** and **`LogFileWriter`** are created and used to write data to files:

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/412ac5be-62e6-4c34-be2e-aa34cbb7be8a)

This demonstrates the principle of **inheritance** and **polymorphism** in C++, where different classes can implement the same function in different ways to achieve different behaviors.

# What is a Virtual keyword?

The virtual keyword in C++ is used to allow for a function or a method in a base class to be overridden in a derived class. By declaring **`createAndWrite()`** as a **pure virtual function**, the **`FileWriter`** class is essentially stating that *Any class that inherits from me must provide an implementation for the **`createAndWrite()`** function.*

Let's demonstrate an ASCII visualization that demonstrates the concept of a pure virtual function:

```
      FileWriter
      /         \
     /           \
    /             \
TextFileWriter   LogFileWriter
(createAndWrite) (createAndWrite)
```

**`FileWriter`** is the base class and **`TextFileWriter`** and **`LogFileWriter`** are the derived classes. **`FileWriter`** declares a pure virtual function called **`createAndWrite()`**.

This means that **`createAndWrite()`** is a function that any class that inherits from **`FileWriter`** is required to implement. It is represented in the diagram as **`(createAndWrite)`** under each derived class.

Let's now look at our code again:

- **`TextFileWriter`**

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/53f747cb-3c32-452f-a5d5-ba8ecae3c1e1)

- **`LogFileWriter`**

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/b8353edc-1a5f-4477-b660-f6a028064d2d)
