# What are Smart Pointers?

Smart pointers are a feature of the C++ Standard Library that provide the functionality of pointers but with **automatic memory management**. They ensure that the object to which they point gets destroyed automatically when there is no more reference to the object, helping to prevent memory leaks. 

Here are two **common** examples:

| Smart Pointer | Description | Example |
| --- | --- | --- |
| `unique_ptr` | A `std::unique_ptr` is a smart pointer that maintains exclusive ownership of an object. When the `std::unique_ptr` is destroyed, it automatically deletes the object it owns. This behavior helps in managing dynamically allocated objects and reducing the risk of memory leaks. `std::unique_ptr` is particularly beneficial when you want to ensure a dynamically allocated object is deleted once it's no longer in use, even if that object is transferred between scopes or functions. | `std::unique_ptr<int> ptr(new int(10)); // ptr now owns the integer object` |
| `shared_ptr` | A `std::shared_ptr` is a smart pointer that allows multiple pointers to share ownership of an object. The object it points to is automatically destroyed when the last `shared_ptr` owning it is destroyed or reset. It's used when you want an object to remain alive for the duration of usage by multiple owners. | `std::shared_ptr<int> ptr1(new int(10)); std::shared_ptr<int> ptr2 = ptr1; // both ptr1 and ptr2 now share ownership of the integer object` |


# std::unique_ptr

In this code example, **`std::unique_ptr`** is used to ensure that a file handle is properly closed when it's no longer needed. 

**`std::unique_ptr`** is a smart pointer in C++ that maintains exclusive ownership of an object it points to, ensuring that the object is properly destroyed when the **`std::unique_ptr`** goes out of scope or is replaced. It is particularly useful in situations where we want automatic memory management to reduce memory leaks, but also want to ensure that only one pointer can access the object at any time, hence the 'unique' in its name. 

```c
#include <windows.h>
#include <iostream>
#include <string>
#include <memory>
#include <time.h>

int main() {
    
    // Seed the random number generator with the current time.
    srand(static_cast<unsigned int>(time(NULL)));

    // Generate a random number.
    int randNum = rand();

    // Convert the random number to a string.
    std::wstring randNumStr = std::to_wstring(randNum);

    // Construct the filename using the random number.
    std::wstring filename = L"C:\\Temp\\testfile_" + randNumStr + L".txt";

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

This line of code creates a **`std::unique_ptr:`**

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/fa8277fe-c40b-409f-b0a8-5673d68e2a70)

- **`std::unique_ptr<void, decltype(&CloseHandle)>`** is declaring a unique pointer that points to **void** (in this case, a **HANDLE**), and uses **`CloseHandle`** as a custom deleter. The type of the deleter is determined by **`decltype(&CloseHandle)`**, which is the type of the **`CloseHandle`** function pointer.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/b6ba5e03-08c3-45a0-9725-7e3e44217949)


- **`fileHandle(tempHandle, CloseHandle)`** is constructing the **`std::unique_ptr`**. It's initialized with **`tempHandle`** (the **HANDLE** returned by **CreateFileW**), and **`CloseHandle`** as the custom deleter.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/66e50f92-00f9-47b0-b4db-65bf7f6d6682)

This **`std::unique_ptr`**, **`fileHandle`**, now owns the file handle. When **`fileHandle`** is destroyed (which will happen automatically when it goes out of scope at the end of **`main`**), it will automatically call **`CloseHandle`** on the owned handle. 

Here's a simple ASCII diagram that demonstrates the relationship between the **`std::unique_ptr`** and the file handle it manages:

```
+-------------------+    manages     +------------+
| std::unique_ptr   |  ----------->  | File Handle|
|  (fileHandle)     |                | (tempHandle)|
+-------------------+                +------------+
     |                                     ^
     |                                     |
     | uses                               used by
     v                                     |
+-------------------+                      |
| WriteFile function|  <-------------------+
|      call         |
+-------------------+
```

1. **`std::unique_ptr`** (**fileHandle**) is created and takes ownership of the File Handle (**tempHandle**).

```c
std::unique_ptr<void, decltype(&CloseHandle)> fileHandle(tempHandle, CloseHandle);
```

2. When we call **`fileHandle.get()`**, it returns the raw HANDLE (**tempHandle**) that the **`std::unique_ptr`** is managing.

```c
fileHandle.get()
```
  
3. This raw handle is then used as an argument to the **`WriteFile`** function.

```c
if (!WriteFile(
    fileHandle.get(),
    data,
    wcslen(data) * sizeof(wchar_t),
    &bytesWritten,
    NULL
)) {
    // handle error
}
```

At the end of the **`main`** function, **`fileHandle`** goes out of scope. This triggers the **`std::unique_ptr`** destructor. In its destructor, **`std::unique_ptr`** calls **`CloseHandle`** on the File Handle it manages, effectively releasing the resource. This is the main advantage of **`std::unique_ptr`** and other smart pointers: they automatically manage the lifetime of resources, which helps prevent resource leaks in your programs.

# std::shared_ptr

**`std::shared_ptr`** is a smart pointer in C++ that retains shared ownership of an object. Multiple **`std::shared_ptr`** instances may own the same object, and the object is automatically destroyed when the last **`std::shared_ptr`** is destroyed or reset. 

Let's demonstrate this with an example code:

```c
#include <iostream>
#include <memory>
#include <vector>
#include <string>

// Fighter class that has a name, wins, and losses
class Fighter {
public:
    std::string name;
    int wins;
    int losses;

    // Constructor for Fighter
    Fighter(const std::string& name, int wins, int losses)
        : name(name), wins(wins), losses(losses) {}
};

// Company (UFC) class that has a list of fighters and a current champion
class Company {
public:
    std::vector<std::shared_ptr<Fighter>> fighters;
    std::shared_ptr<Fighter> currentChampion;

    // Function to add a new fighter to the UFC
    void addFighter(const std::string& name, int wins, int losses) {
        fighters.push_back(std::make_shared<Fighter>(name, wins, losses));
    }

    // Function to set the current champion
    void setChampion(const std::string& name) {
        for (const auto& fighter : fighters) {
            if (fighter->name == name) {
                currentChampion = fighter;
                break;
            }
        }
    }

    // Function to print all fighters in the UFC
    void listFighters() {
        for (const auto& fighter : fighters) {
            std::cout << "Name: " << fighter->name << ", Wins: " << fighter->wins << ", Losses: " << fighter->losses << "\n";
        }
    }

    // Function to print the current champion
    void printChampion() {
        if (currentChampion) {
            std::cout << "Current champion is: " << currentChampion->name << "\n";
        }
    }
};

int main() {
    // create a new Company (UFC)
    Company ufc;

    // add two fighters to the UFC
    ufc.addFighter("Conor McGregor", 22, 6);
    ufc.addFighter("Michael Chandler", 23, 8);

    // set the current champion
    ufc.setChampion("Conor McGregor");

    // print all fighters in the UFC
    ufc.listFighters();

    // print the current champion
    ufc.printChampion();

    return 0;
}
```

This line declares a vector **`fighters`** that will hold shared pointers to **`Fighter`** objects. This means the **`Company`** class holds a list of shared pointers to **`Fighter`** objects.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/5b31cbeb-ab8d-439a-acfe-4da244788d13)

The **`addFighter`** method dynamically allocates a new **`Fighter`** and adds a **`std::shared_ptr`** to it in the fighters vector. **`std::make_shared`** is a utility function that creates a **`std::shared_ptr`**. **`std::make_shared`** is a utility function template in C++ that constructs an object of the given type and wraps it in a **`std::shared_ptr`**.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/26ba363c-d038-4672-b0bc-d67dd2b1de26)

The **`setChampion`** method sets **`currentChampion`**, which is a **`std::shared_ptr<Fighter>`**, to point to the **`Fighter`** object with the given name. This demonstrates shared ownership, as both **`currentChampion`** and an element of **`fighters`** are now **`std::shared_ptr`** instances pointing to the same **`Fighter`** object.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/1eda016c-b66e-48e1-8731-475e178f3fef)

When the **`Company`** object **`ufc`** goes out of scope at the end of **`main`**, the **`fighters`** vector and **`currentChampion`** are automatically destroyed. This triggers the destruction of the **`std::shared_ptrs`**, which automatically delete the objects they own if they are the last **`std::shared_ptr`** owning the objects. This means all **`Fighter`** objects are automatically deleted when **`main`** ends, reducing memory leaks.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/52828c9f-b0ad-42df-9233-20f4a6c605a7)

# What is Shared Ownership?

At the **`setChampion`** method, we discussed the following:

*The **`setChampion`** method sets **`currentChampion`**, which is a **`std::shared_ptr<Fighter>`**, to point to the **`Fighter`** object with the given name. This demonstrates **shared ownership**, as both **`currentChampion`** and an element of **`fighters`** are now **`std::shared_ptr`** instances pointing to the same **`Fighter`** object.*

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/37ebbee4-e4fa-4d5f-b77a-0b8122017dc3)


This may be confusing for the first time, so let's use an ASCII visualization to gather a better understanding.

Let's say we have two fighters: **"Conor McGregor"** and **"Michael Chandler"**. Each fighter is represented by a **`Fighter`** object. In our **`Company`** (UFC) class, we maintain a list of fighters and a current champion. Both the list of fighters and the current champion are represented using **`std::shared_ptr`**, which is a type of smart pointer that allows multiple pointers to refer to the same object.

Initially, when we create the **`fighters`**, it looks something like this:

```
+----------------------+                  +-----------------------+
| Fighter              |                  | shared_ptr<Fighter>   |
| "Conor McGregor"     |  <-------------- | (in fighters vector)  |
+----------------------+                  +-----------------------+

+----------------------+                  +-----------------------+
| Fighter              |                  | shared_ptr<Fighter>   |
| "Michael Chandler"   |  <-------------- | (in fighters vector)  |
+----------------------+                  +-----------------------+
```

Each **`shared_ptr<Fighter>`** in the **`fighters`** vector owns a **`Fighter`** object.

Next, when we call **`setChampion("Conor McGregor")`**, we find the Fighter object for **"Conor McGregor"** and set **`currentChampion`** to point to it:

```
+----------------------+                  +-----------------------+                  +-----------------------+
| Fighter              |                  | shared_ptr<Fighter>   |                  | currentChampion       |
| "Conor McGregor"     |  <-------------- | (in fighters vector)  |  <-------------- | (shared_ptr<Fighter>) |
+----------------------+                  +-----------------------+                  +-----------------------+

+----------------------+                  +-----------------------+
| Fighter              |                  | shared_ptr<Fighter>   |
| "Michael Chandler"   |  <-------------- | (in fighters vector)  |
+----------------------+                  +-----------------------+
```

Now, both **`currentChampion`** and the corresponding **`shared_ptr<Fighter>`** in the **`fighters`** vector point to the **`Fighter`** object for **"Conor McGregor"**. They **share ownership** of this **`Fighter`** object.

When we call **`setChampion("Michael Chandler")`**, **`currentChampion`** is updated to point to the **`Fighter`** object for **"Michael Chandler"**:

```
+----------------------+                  +-----------------------+
| Fighter              |                  | shared_ptr<Fighter>   |
| "Conor McGregor"     |  <-------------- | (in fighters vector)  |
+----------------------+                  +-----------------------+

+----------------------+                  +-----------------------+                  +-----------------------+
| Fighter              |                  | shared_ptr<Fighter>   |                  | currentChampion       |
| "Michael Chandler"   |  <-------------- | (in fighters vector)  |  <-------------- | (shared_ptr<Fighter>) |
+----------------------+                  +-----------------------+                  +-----------------------+
```

Again, **`currentChampion`** and the corresponding **`shared_ptr<Fighter>`** in the **`fighters`** vector share ownership of the **`Fighter`** object for **"Michael Chandler"**.
