# What is a Class?

A class in C++ is a template used to create objects. This template defines what kind of data the object can hold and what actions it can take. The class is a core concept in C++ and is the basic unit of encapsulation, which is one of the foundational principles of object-oriented programming (OOP). Encapsulation refers to bundling of data, and the methods that operate on this data, into a single unit, which is the class.

The most **common** core concepts of a class in C++ are:

- **Data Members/Attributes:** These are the variables where the data associated with the objects of a class are stored.
- **Member Functions/Methods:** These are the functions that provide the behavior of the objects.
- **Access Specifiers:** These are used to define the visibility and accessibility of the class members. The most frequently used are public and private.
- **Constructors and Destructors:** Constructors initialize an object when it is created, and destructors clean up as necessary when the object is no longer needed. These are important for managing resources.

# Code Sample

Let's demonstrate the concept of a **class** with a code snippet:

```c
#include <iostream>

// Define the class 'MMAFighter'
class MMAFighter {
public:
    std::string name;  // Declare a string to hold the fighter's name
    int wins;  // Declare an integer to hold the fighter's win count
    int losses;  // Declare an integer to hold the fighter's loss count

    // Declare a member function to display the fighter's record
    void displayRecord() {
        std::cout << "Name: " << name << "\n";
        std::cout << "Wins: " << wins << "\n";
        std::cout << "Losses: " << losses << "\n";
    }
};

int main() {
    MMAFighter fighter1;  // Create an object of MMAFighter class

    fighter1.name = "Conor McGregor";  // Assign a name to fighter1
    fighter1.wins = 22;  // Assign win count to fighter1
    fighter1.losses = 6;  // Assign loss count to fighter1

    fighter1.displayRecord();  // Call the displayRecord function for fighter1

    return 0;
}
```

Let's break down this code:

# Data Members

A data member in C++ is a variable that's part of a class (or struct). It is used to store data associated with the class and the objects of that class. Data members can be of any data type, and each instance (or object) of the class has its own copy of each data member.

- **`name:`** is a data member of type **std::string** that represents the name of an MMA fighter.

- **`wins:`** is a data member of type **int** that represents the number of wins for an MMA fighter.

- **`losses:`** is a data member of type **int** that represents the number of losses for an MMA fighter.

```c
class MMAFighter {
public:
    std::string name;  // Declare a string to hold the fighter's name
    int wins;  // Declare an integer to hold the fighter's win count
    int losses;  // Declare an integer to hold the fighter's loss count
```


# Object

**`fighter1`** is an object (or instance) of the **`MMAFighter`** class.

```c
MMAFighter fighter1;
```

This **`fighter1`** object has its own copies of all the data members (**`name`**, **`wins`**, and **`losses`**) of the **`MMAFighter`** class. After the **`fighter1`** object is created, we can access and manipulate its data members using the dot operator **(.)**.

```c
    fighter1.name = "Conor McGregor";  // Assign a name to fighter1
    fighter1.wins = 22;  // Assign win count to fighter1
    fighter1.losses = 6;  // Assign loss count to fighter1
```

When you create an object from a class, you're basically creating a variable of the type defined by the class. This variable (or object) will have its own copies of the class's data members, and it can perform any actions defined by the class's member functions.

For example, in a **`MMAFighter`** class, each fighter like **"Conor McGregor"** or **"Khabib Nurmagomedov"** would be an object of the class. Each of these objects would have its own **`name`**, number of **`wins`**, and number of **`losses`** (the class's data members), and could perform actions like displaying their records (a class's member function).

Let's demonstrate this with a code snippet now:

```c
#include <iostream>

// Define the class 'MMAFighter'
class MMAFighter {
public:
    std::string name;  // Declare a string to hold the fighter's name
    int wins;  // Declare an integer to hold the fighter's win count
    int losses;  // Declare an integer to hold the fighter's loss count

    // Declare a member function to display the fighter's record
    void displayRecord() {
        std::cout << "Name: " << name << "\n";
        std::cout << "Wins: " << wins << "\n";
        std::cout << "Losses: " << losses << "\n";
    }
};

int main() {
    MMAFighter fighter1;  // Create an object of MMAFighter class

    fighter1.name = "Conor McGregor";  // Assign a name to fighter1
    fighter1.wins = 22;  // Assign win count to fighter1
    fighter1.losses = 6;  // Assign loss count to fighter1

    fighter1.displayRecord();  // Call the displayRecord function for fighter1
    std::cout << "\n";  // Print a blank line

    MMAFighter fighter2;  // Create another object of MMAFighter class

    fighter2.name = "Khabib Nurmagomedov";  // Assign a name to fighter2
    fighter2.wins = 29;  // Assign win count to fighter2
    fighter2.losses = 0;  // Assign loss count to fighter2

    fighter2.displayRecord();  // Call the displayRecord function for fighter2
    std::cout << "\n";  // Print a blank line

    return 0;
}
```

**`fighter1`** and **`fighter2`** are two different objects of the **`MMAFighter`** class, each representing a different MMA fighter. They each have their own set of **`name`**, **`wins`**, and **`losses`**. This demonstrates how each instance (or object) of a class has its own set of the class's data members.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/2c96ee91-d961-45a4-b15e-80ea440a24ee)


# Access Specifiers

Access specifiers in C++ are used to implement the encapsulation feature of object-oriented programming. They set the access levels for the members (both data members and member functions) of a class. They define how the members of the class can be accessed. There are three types of access specifiers:

- **Public:** Members (data members and member functions) declared under the public specifier are accessible from anywhere. This means they can be accessed from within the class, from derived classes, and from outside the class.

- **Private:** Members declared as private can be accessed only from within the class itself. They are not accessible from other classes (including derived classes) or from outside the class.

- **Protected:** Members declared as protected can be accessed from within the class and from derived classes, but not from outside these classes.

Let's demonstrate a demo on how **private** access specifiers for example works:

```c
#include <iostream>

class MMAFighter {
private:
    std::string name;
    int wins;
    int losses;

public:
    void setName(std::string n) {
        name = n;
    }

    std::string getName() {
        return name;
    }

    void setWins(int w) {
        wins = w;
    }

    int getWins() {
        return wins;
    }

    void setLosses(int l) {
        losses = l;
    }

    int getLosses() {
        return losses;
    }

    void displayRecord() {
        std::cout << "Name: " << getName() << "\n";
        std::cout << "Wins: " << getWins() << "\n";
        std::cout << "Losses: " << getLosses() << "\n";
    }
};

int main() {
    MMAFighter fighter1;

    fighter1.setName("Conor McGregor");
    fighter1.setWins(22);
    fighter1.setLosses(6);

    fighter1.displayRecord();
    std::cout << "\n";

    MMAFighter fighter2;

    fighter2.setName("Khabib Nurmagomedov");
    fighter2.setWins(29);
    fighter2.setLosses(0);

    fighter2.displayRecord();
    std::cout << "\n";

    return 0;
}
```

The **private** keyword in the **`MMAFighter`** class definition means that the **`name`**, **`wins`**, and **`losses`** member variables are private. These private member variables can only be accessed directly within member functions of the **`MMAFighter`** class.

Let's say that we have a **`MMAFighter`** class with private variables **`name`**, **`wins`**, and **`losses`**. Now, if we try to directly access these private variables from outside the class, it will result in a compile error:

```c
#include <iostream>

class MMAFighter {
private:
    std::string name;
    int wins;
    int losses;

public:
    void setName(std::string n) {
        name = n;
    }

    std::string getName() {
        return name;
    }

    void setWins(int w) {
        wins = w;
    }

    int getWins() {
        return wins;
    }

    void setLosses(int l) {
        losses = l;
    }

    int getLosses() {
        return losses;
    }

    void displayRecord() {
        std::cout << "Name: " << getName() << "\n";
        std::cout << "Wins: " << getWins() << "\n";
        std::cout << "Losses: " << getLosses() << "\n";
    }
};

int main() {
    MMAFighter fighter1;

    // These lines of code will cause a compile error because 'name', 'wins', and 'losses' are private
    fighter1.name = "Conor McGregor";  
    fighter1.wins = 22; 
    fighter1.losses = 6; 

    fighter1.displayRecord();
    std::cout << "\n";

    return 0;
}
```

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/9005a6c1-6153-4ea7-a676-38f7518e28fc)


The correct way would be the following:

We have several **public** member functions, often called methods. These include **`setName`**, **`getName`**, **`setWins`**, **`getWins`**, **`setLosses`**, **`getLosses`**, and **`displayRecord`**. 

```c
public:
    void setName(std::string n) {
        name = n;
    }

    std::string getName() {
        return name;
    }

    void setWins(int w) {
        wins = w;
    }

    int getWins() {
        return wins;
    }

    void setLosses(int l) {
        losses = l;
    }

    int getLosses() {
        return losses;
    }

    void displayRecord() {
        std::cout << "Name: " << getName() << "\n";
        std::cout << "Wins: " << getWins() << "\n";
        std::cout << "Losses: " << getLosses() << "\n";
    }
```

Because these methods are declared **public**, they can be accessed from any part of the program. For example, from within the main function.

```c
int main() {
    
    MMAFighter fighter1;

    fighter1.setName("Conor McGregor");
    fighter1.setWins(22);
    fighter1.setLosses(6);

    fighter1.displayRecord();

    std::cout << "\n";

    return 0;
}
```

# An example of Private Access Specifier

Private access specifiers in C++ are used to achieve a key principle of object-oriented programming, which is **encapsulation**. **Encapsulation** is the bundling of data with the methods that operate on that data. 

The use of private data and public methods to control access to an object's internal state is a fundamental aspect of object-oriented programming. It helps to maintain the integrity of the data and can make the code easier to understand and maintain.


Let's imagine you've just opened a new bank account with the account number **`"123456789"`**. This account number is like your identity, and the balance is the amount of money you have. Just like in real life, these pieces of information are kept **private** within the bank's systems, shown by the private keyword in the **`BankAccount`** class.

Now, you want to deposit $2000 into your new account. But you can't just go and change the balance directly - it's private. Instead, you use a method the bank has given you, called **`deposit`**. This method takes the amount you want to deposit, checks that it's a positive amount, then adds it to your balance.

Once the money is in your account, you decide to withdraw $1500. Again, you can't just take it - you have to use the bank's **`withdraw`** method. This method checks that you're trying to withdraw a positive amount, and that you have enough money in your account. If both conditions are met, it subtracts the amount from your balance.

After these transactions, you want to check your account balance. The balance is private, but the bank provides a **`getBalance`** method, which returns the current balance without letting you change it directly. This method is like checking your balance at an ATM or on a banking app.

The entire **`BankAccount`** class is like the bank's systems. It keeps your account number and balance private, but provides methods you can use to interact with your account. This is a great example of encapsulation - the data (your account number and balance) is bundled with the methods that operate on it (**`deposit`**, **`withdraw`**, **`getBalance`**), and everything is kept in one 'class'.

```c
#include <iostream>

class BankAccount {
private:
    double balance;

public:
    BankAccount() : balance(0) {}

    void deposit(double amount) {
        // Check if the amount to be deposited is greater than 0.
        if (amount > 0) {
            // If it is, add it to the balance.
            balance += amount;
        }
    }

    bool withdraw(double amount) {
        // Check if the amount to be withdrawn is greater than 0 and less than or equal to the balance.
        if (amount > 0 && balance >= amount) {
            // If it is, subtract it from the balance and return true to indicate the withdrawal was successful.
            balance -= amount;
            return true;
        }
        else {
            // If not, return false to indicate the withdrawal failed.
            return false;
        }
    }

    double getBalance() {
        return balance;
    }
};

int main() {
    BankAccount myAccount;

    myAccount.deposit(1000);
    std::cout << "Balance: $" << myAccount.getBalance() << "\n";  // Balance: $1000

    bool success = myAccount.withdraw(500);
    // Check if the withdrawal was successful.
    if (success) {
        std::cout << "Withdrawal successful!\n";
        std::cout << "Balance: $" << myAccount.getBalance() << "\n";  // Balance: $500
    }
    else {
        std::cout << "Withdrawal failed.\n";
    }

    return 0;
}
```

At this example, **deposit**, **withdraw**, and **getBalance** functions are all methods of the **BankAccount** class. They define the operations that can be performed on **BankAccount** objects, such as depositing money into the account, withdrawing money from the account, or getting the current balance of the account.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/ceff71d1-5d95-4986-822f-8833d9fa2c82)


# Constructors and Destructors

Constructors and Destructors are special member functions of a class that are used for initializing and cleaning up objects of the class.

**Constructors:**

The constructor is defined in the **`BankAccount`** class. This is a **default constructor** because it doesn't take any arguments. When we create a new **`BankAccount`** object, this constructor is automatically called. The purpose of this constructor is to set up the object in a valid state. Here, it initializes the balance field to 0.

```c
BankAccount() : balance(0) {}
```

When we write the following line in the main:

```c
BankAccount myAccount;
```

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/0c1c2241-b98e-49ea-891c-d9ceb11890a1)


We're creating a new **`BankAccount`** object, and automatically calling the **`BankAccount`** constructor to set that object's balance to **0**. The **0** here is the initial state of the **`balance`** attribute of the **`BankAccount`** object. 

```c
BankAccount() : balance(0) {}
```

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/9ad07cd6-15f7-43a8-bba3-e0c77d6cbfde)


Our current example is using a **default constructor** and doesn't have any arguments. A **constructor argument** is a value that we can pass in when we create an instance of a class. These arguments provide initial values for the object properties or other initialization tasks. 

Let's now modify our current code to include a **constructor argument:**

```c
#include <iostream>

class BankAccount {
private:
    double balance;

public:
    // Modify the constructor to accept an argument
    BankAccount(double initialBalance) : balance(initialBalance) {}

    void deposit(double amount) {
        // Check if the amount to be deposited is greater than 0.
        if (amount > 0) {
            // If it is, add it to the balance.
            balance += amount;
        }
    }

    bool withdraw(double amount) {
        // Check if the amount to be withdrawn is greater than 0 and less than or equal to the balance.
        if (amount > 0 && balance >= amount) {
            // If it is, subtract it from the balance and return true to indicate the withdrawal was successful.
            balance -= amount;
            return true;
        }
        else {
            // If not, return false to indicate the withdrawal failed.
            return false;
        }
    }

    double getBalance() {
        return balance;
    }
};

int main() {
    // When we create the object, we pass in an initial balance of 1000
    BankAccount myAccount(1000);
    std::cout << "Balance: $" << myAccount.getBalance() << "\n";  // Balance: $1000

    bool success = myAccount.withdraw(500);
    // Check if the withdrawal was successful.
    if (success) {
        std::cout << "Withdrawal successful!\n";
        std::cout << "Balance: $" << myAccount.getBalance() << "\n";  // Balance: $500
    }
    else {
        std::cout << "Withdrawal failed.\n";
    }

    return 0;
}
```

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/473a6a14-2711-46ce-aeab-ac0eafd6d5f0)



In the first version of the code, we had a default constructor **`BankAccount() : balance(0) {}`** that always set the initial balance to **0** when a **`BankAccount`** object was created. If we wanted to set a different initial balance, we would have to do after creating the object, using the **`deposit()`** method. 

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/4022bb10-6ee3-4eb4-b014-09bb909f01c5)


In the updated version of the code, we can specify the initial balance directly when we create the **`BankAccount`** object. This is done through the constructor. This constructor sets the **`balance`** of the account directly when the object is created. It does this by taking the **`initialBalance`** argument that we pass in when we are creating the **`BankAccount`** object, and it uses that value to set the **`balance`** member of the object.

```c
BankAccount(double initialBalance) : balance(initialBalance) {}
```
![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/9c713fb4-f507-407b-a6ea-4600875414be)


# Final Example

Let's now recap everything we have learned with two final examples. Here we have a simple code snippet using the **private** access specifier.

```c
#include <windows.h>
#include <iostream>
#include <string>
#include <time.h>

class FileWriter {
private:
    // These are private data members of the class
    std::wstring filename;
    HANDLE fileHandle;

public:
    FileWriter() : fileHandle(INVALID_HANDLE_VALUE) {
        srand(static_cast<unsigned int>(time(NULL)));
        int randNum = rand();
        std::wstring randNumStr = std::to_wstring(randNum);
        filename = L"C:\\Temp\\testfile_" + randNumStr + L".txt";
    }

    // This is a public method to create and write to a file
    void createAndWrite() {
        fileHandle = CreateFileW(
            filename.c_str(),
            GENERIC_WRITE,
            0,
            NULL,
            CREATE_NEW,
            FILE_ATTRIBUTE_NORMAL,
            NULL
        );

        if (fileHandle == INVALID_HANDLE_VALUE) {
            std::wcout << L"Unable to create file due to error: " << GetLastError() << "\n";
            return;
        }

        const wchar_t data[] = L"Hello, World!";
        DWORD bytesWritten;

        if (!WriteFile(
            fileHandle,
            data,
            wcslen(data) * sizeof(wchar_t),
            &bytesWritten,
            NULL
        )) {
            std::wcout << L"Unable to write to file due to error: " << GetLastError() << "\n";
        }
        else {
            std::wcout << L"File created and written successfully.\n";
        }
    }

    // This is a public method to close the file
    void closeFile() {
        if (fileHandle != INVALID_HANDLE_VALUE) {
            CloseHandle(fileHandle);
        }
    }

    ~FileWriter() {
        closeFile(); // ensure the file handle is closed if it was opened
    }
};

int main() {
    FileWriter writer;
    writer.createAndWrite();
    return 0;
}
```

**Class**

**`FileWriter`** is the class in this program. A class is a blueprint for creating objects. It describes the kind of data and behavior that objects of this class will have.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/f152e02a-5908-42c0-9bbb-4c38205dd135)



**Object**

**`writer`** is an object of the **`FileWriter`** class. It is created in the **main** function with **`FileWriter writer;`**. This statement creates an instance of **FileWriter**.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/d43ab14f-95b0-4d52-b773-818d4969fd4d)


**Data Members**

The **`FileWriter`** class has two **private** data members: **`filename`** and **`fileHandle`**. **`filename`** is a string that stores the name of the file, and **`fileHandle`** is a HANDLE object (from the Windows API) that is used to interact with the file. 

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/4e0bf300-2e91-482e-8344-8be4c90aa919)


**Methods**

**`createAndWrite()`**, **`closeFile()`**, and the destructor **`~FileWriter()`** are methods (functions defined within the class). They define the behavior that the objects of the **`FileWriter`** class can perform.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/d410b90c-82c4-491b-a51c-932d75724717)


![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/c0c960d8-2902-4a5b-aadd-94dd83dd76e8)


**Constructor**

**`FileWriter()`** is the constructor of the class. A constructor is a method that is automatically called when an object of the class is created. In this case, it initializes the **`fileHandle`** to INVALID_HANDLE_VALUE and generates a random filename.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/88222425-8c06-4aea-ab72-0b6f0504bda5)


**Destructor**

**`~FileWriter()`** is the destructor of the class. A destructor is a special method that is called automatically when an object is about to be destroyed. Here, the destructor ensures the file handle is closed if it was opened. The tilde symbol **~** is used in C++ to denote a destructor for a class.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/2c161be6-de5a-4744-9975-1b8c7ea8656b)


Let's cover the second example, which may be a bit more complicated though. We are basically just walking through the code and explain what the class, object, data members, etc of this code.

```c
#include <windows.h>
#include <iostream>
#include <string>
#include <time.h>

class FileHandler {
private:
    HANDLE fileHandle;  // Declare a HANDLE to hold the file's handle

    // Function to generate a random filename
    std::wstring generateFilename() {
        // Seed the random number generator with the current time
        srand(static_cast<unsigned int>(time(NULL)));
        // Generate a random number
        int randNum = rand();
        // Convert the random number to a string
        std::wstring randNumStr = std::to_wstring(randNum);
        // Return the filename
        return L"C:\\Temp\\testfile_" + randNumStr + L".txt";
    }

    // Function to create a file with the given filename
    void createFile(std::wstring filename) {
        // Create the file
        fileHandle = CreateFileW(
            filename.c_str(),
            GENERIC_WRITE,
            0,
            NULL,
            CREATE_NEW,
            FILE_ATTRIBUTE_NORMAL,
            NULL
        );

        // Check if the file was created successfully
        if (fileHandle == INVALID_HANDLE_VALUE) {
            std::wcout << L"Unable to create file due to error: " << GetLastError() << "\n";
        }
    }

    // Function to write to the file
    void writeToFile() {
        // The data to write to the file
        const wchar_t data[] = L"Hello, World!";
        DWORD bytesWritten;

        // Write the data to the file
        if (!WriteFile(
            fileHandle,
            data,
            wcslen(data) * sizeof(wchar_t),
            &bytesWritten,
            NULL
        )) {
            std::wcout << L"Unable to write to file due to error: " << GetLastError() << "\n";
        }
        else {
            std::wcout << L"File created and written successfully.\n";
        }
    }

    // Function to close the file
    void closeFile() {
        // Check if the file handle is valid
        if (fileHandle != INVALID_HANDLE_VALUE) {
            // Close the file
            CloseHandle(fileHandle);
        }
    }

public:
    FileHandler() : fileHandle(INVALID_HANDLE_VALUE) {
        // Generate a random filename
        std::wstring filename = generateFilename();
        // Create a file with the generated filename
        createFile(filename);
        // Write to the file
        writeToFile();
    }

    ~FileHandler() {
        // Close the file
        closeFile();
    }
};

int main() {
    FileHandler fh;
    return 0;
}
```

**Class**

**`FileHandler`** is the class in this program. A class is a blueprint for creating objects. It describes the kind of data and behavior that objects of this class will have.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/f5f27176-24db-4fb3-b805-8b9e3c006e15)


**Object**

An object is an instance of a class. It is created using the blueprint provided by the class definition. In the **main()** function, **`fh`** is an object of the **`FileHandler`** class.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/6c65debe-cdcd-4464-ae29-2d2e9b89c524)


**Data Members**

Data members are variables declared within a class. They hold data that is associated with the class and its objects. In the **`FileHandler`** class, **`fileHandle`** is a private data member.

**Methods**

The **`FileHandler`** class has several other methods: **`generateFilename`**, **`createFile`**, **`writeToFile`**, and **`closeFile`**. These are functions that are associated with the class and define the behaviors of its objects. All these methods are private, meaning they can only be called from within the class.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/b0bbcc96-42ea-42cf-ac10-3a0163c66ce8)


![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/064d1679-41dd-43a1-8eaa-4f602e02107c)



**Constructor**

A constructor is a function in a class that is automatically called when an object of the class is created. It initializes data members of the class. In the **`FileHandler`** class, **`FileHandler()`** is the constructor, initializing **`fileHandle`** to **INVALID_HANDLE_VALUE** and then invoking other functions to generate a filename, create a file with that name, and write to the file.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/f3d31933-b41e-4ccd-9a31-54ab22d63128)


**Destructor**

A destructor is another function in a class that is automatically called when an object of the class is destroyed. It is often used to release resources that the object may have acquired during its life. In the **`FileHandler`** class, **`~FileHandler()`** is the destructor, which calls the **`closeFile`** function to close the file.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/1fabc85d-e9e1-48a1-8f94-e08c95ebaf6c)


# Attempt to call the private method outside of the class

Let's take the **`closeFile()`** method as an example. **`closeFile()`** is a private method in the **`FileHandler`** class. Private methods can only be called within the class they're declared in. When we say that **`closeFile()`** is private, we mean it can't be called from outside the **`FileHandler`** class.

**`closeFile()`** is defined in the private section of the **`FileHandler`** class, and it's used in the destructor, which is a part of the same class. This is a correct use of a private method.

```c
class FileHandler {
private:
    // ...
    
    void closeFile() {
        // ...
    }

public:
    // ...
    
    ~FileHandler() {
        closeFile();
    }
};
```
What happens if we now call **`closeFile()`** outside of the class, let's say in the **main()**. 

```c
#include <windows.h>
#include <iostream>
#include <string>
#include <time.h>

class FileHandler {
private:
    HANDLE fileHandle;

    std::wstring generateFilename() {
        srand(static_cast<unsigned int>(time(NULL)));
        int randNum = rand();
        std::wstring randNumStr = std::to_wstring(randNum);
        return L"C:\\Temp\\testfile_" + randNumStr + L".txt";
    }

    void createFile(std::wstring filename) {
        fileHandle = CreateFileW(
            filename.c_str(),
            GENERIC_WRITE,
            0,
            NULL,
            CREATE_NEW,
            FILE_ATTRIBUTE_NORMAL,
            NULL
        );

        if (fileHandle == INVALID_HANDLE_VALUE) {
            std::wcout << L"Unable to create file due to error: " << GetLastError() << "\n";
        }
    }

    void writeToFile() {
        const wchar_t data[] = L"Hello, World!";
        DWORD bytesWritten;

        if (!WriteFile(
            fileHandle,
            data,
            wcslen(data) * sizeof(wchar_t),
            &bytesWritten,
            NULL
        )) {
            std::wcout << L"Unable to write to file due to error: " << GetLastError() << "\n";
        }
        else {
            std::wcout << L"File created and written successfully.\n";
        }
    }

    void closeFile() {
        if (fileHandle != INVALID_HANDLE_VALUE) {
            CloseHandle(fileHandle);
        }
    }

public:
    FileHandler() : fileHandle(INVALID_HANDLE_VALUE) {
        std::wstring filename = generateFilename();
        createFile(filename);
        writeToFile();
    }

    ~FileHandler() {
        closeFile();
    }
};

int main() {
    FileHandler fh;
    fh.closeFile(); // This line will give a compile-time error
    return 0;
}
```

When we try to compile this code, we will get a compile-time error because **`closeFile()`** is a private method of **`FileHandler`** class. The line **`h.closeFile();`** in **main()** is trying to call this method directly on a **`FileHandler`** object, which is not allowed because **`closeFile()`** is private and can only be called from within the **`FileHandler`** class itself. This is an example of encapsulation in object-oriented programming.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/5134d73e-3df3-461c-b460-e86571f1338f)

