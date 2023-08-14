# What are Constructors?

A constructor is a member function of a class that is used to initialize objects of that class type. There are various types of constructors that C++ supports, two of them are the **copy** constructor and the **move** constructor.

- **Copy Constructor**

This is a process in C++ where a new object is created as a copy of an existing object. The new object gets the same values as the existing object.

- **Move Constructor**

This is a process in C++ where a new object takes over the data from an existing object. Instead of copying the data, it just moves it, so the existing object is left empty and the new object has all the data.

Both are ways to create a new object from an existing object, but the move constructor can be more efficient because it just moves data around, rather than making a copy of it. This can make a big difference when the data is large or complex.

# Code Sample - Copy Constructors (1)

This code demonstrates a simple example of the use of a **copy** constructor to create a new object **(box2)** as a copy of an existing object **(box1)**.

```c
#include <iostream>

class NumberBox {
public:
    int num;

    // Constructor
    NumberBox(int n) : num(n) {}

    // Copy constructor
    NumberBox(const NumberBox& src) : num(src.num) {}
};

int main() {
    NumberBox box1(42);  // box1 is initialized with 42
    NumberBox box2 = box1;  // box2 is a copy of box1

    std::cout << "box1's number: " << box1.num << std::endl;
    std::cout << "box2's number: " << box2.num << std::endl;

    return 0;
}
```

At this example, a class named **NumberBox** is defined. This class has a single public member variable **`num`** of type **int**.

Two constructors are provided for the **NumberBox** class. The first one is a **parameterized** constructor that takes an **integer** as an **argument** and initializes the **`num`** member variable with this value. This constructor allows us to create a **NumberBox** object with a specific value. For example, **`NumberBox box1(42)`** will create a **NumberBox** object where **`num`** is **42**.

```c
    // Constructor
    NumberBox(int n) : num(n) {}
```

The second constructor is a **copy** constructor. It takes a constant reference to another **NumberBox** object **(src)** and initializes the **`num`** member of the new **NumberBox** object to be the same as that of **src**. This constructor allows us to create a new **NumberBox** object that is a copy of an existing **NumberBox** object.

```c
    // Copy constructor
    NumberBox(const NumberBox& src) : num(src.num) {}
```

When we are looking at a class definition, we can recognize the copy constructor because it's a constructor **with the same name as the class** and it takes one parameter which is a **const reference** to an object of the same class. The **const** keyword indicates that the copy constructor won't modify the object being copied.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/a73ec4ee-ab83-40cd-8cfa-0af009460470)

In the **main** function, a **NumberBox** object **box1** is created and initialized with the value **42** using the parameterized constructor. Then, a second **NumberBox** object **box2** is created and initialized as a copy of **box1** using the copy constructor.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/2da8b20d-daaf-4a3e-98d5-d8c0a93c61c0)

Finally, the values of **`num`** for both **box1** and **box2** are printed to the console. As **box2** is a copy of **box1**, they both hold the same value, **42**.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/7d1e2719-c656-425e-b90b-39ae0b182569)

# Code Sample - Copy Constructor (2)

This code defines a **`Fighter`** class, which represents a fighter with a **name** and a **record** of fights, and provides a way to add fights to the record and display the fighter's information. It demonstrates the use of a **copy** constructor to create a new fighter with the same name and fight record as an existing one.

```c
#include <iostream>
#include <string>
#include <vector>

class Fighter {
private:
    std::string name;
    std::vector<std::string> fightRecord;

public:
    // Constructor
    Fighter(const std::string& name) : name(name) {}

    // Copy constructor
    Fighter(const Fighter& src) : name(src.name), fightRecord(src.fightRecord) {}

    // Method to add a fight to the fighter's record
    bool addFight(const std::string& fight) {
        try {
            fightRecord.push_back(fight);
        }
        catch (const std::exception& e) {
            std::cerr << "Failed to add fight: " << e.what() << '\n';
            return false;
        }
        return true;
    }

    // Method to display the fighter's info
    void display() const {
        std::cout << "Fighter: " << name << '\n'
            << "Fight Record:\n";
        for (const auto& fight : fightRecord) {
            std::cout << fight << '\n';
        }
        std::cout << '\n';  // extra newline for spacing
    }
};

int main() {
    // Create a Fighter object
    Fighter fighter1("Conor McGregor");
    fighter1.addFight("Win against Jose Aldo");
    fighter1.addFight("Loss against Nate Diaz");

    // Use the copy constructor to create a new Fighter that's a copy of fighter1
    Fighter fighter2 = fighter1;

    // Add a fight to fighter2's record
    fighter2.addFight("Win against Eddie Alvarez");

    // Display the fighters' info
    fighter1.display();
    fighter2.display();

    return 0;
}
```

Let's break down this code.

The **`Fighter`** class has two **private** member variables: **`name`** of type **`std::string`** and **`fightRecord`** of type **`std::vector<std::string>`**.

- **`std::string name;`** is a variable of type **`std::string`** (a string of characters) that is used to store the name of the fighter. **`std::string`** is a class in the C++ Standard Library that represents a sequence of characters.
- **`std::vector<std::string> fightRecord;`** is a variable of type **`std::vector<std::string>`** which is used to store the fight record of the fighter. **`std::vector`** is a template class in the C++ Standard Library that represents a dynamic array, meaning its size can change during runtime. 

Here each element in the vector is a **`std::string`**. This means that the **`fightRecord`** can store a list of strings, where each string represents a fight.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/bf60f84e-d36e-4b47-b359-370086c56c5f)

Let's visualize this in ASCII. Each box in the row represents an element in the **`fightRecord`** vector. The text inside each box is a **`std::string`** that represents the outcome of a fight.

```
Fighter: Conor McGregor
Fight Record:
+------------------------+-----------------------+
| "Win against Jose Aldo" | "Loss against Nate Diaz" |
+------------------------+-----------------------+
```

Now, let's consider the **`Fighter`** object **`fighter2`** which is a copy of **`fighter1`** but has an additional fight record:

```
"Win against Jose Aldo"
"Loss against Nate Diaz"
"Win against Eddie Alvarez"
```

Let's visualize this as well. As you can see, the **`std::vector`** for **`fighter2`** has grown to accommodate the new fight record. 

```
Fighter: Conor McGregor 2
Fight Record:
+------------------------+-----------------------+---------------------------+
| "Win against Jose Aldo" | "Loss against Nate Diaz" | "Win against Eddie Alvarez" |
+------------------------+-----------------------+---------------------------+
```

The purpose of a constructor is to initialize the object's attributes (also known as member variables). At this example, the constructor is taking one argument, which is a reference to a constant string **`(const std::string&)`**. This string is expected to be the name of the fighter.

The **`: name(name)`** part is called an initializer list. It's a C++ feature used to directly initialize member variables when an object is created. Here, it's used to initialize the **`name`** member variable of the Fighter class with the **`name`** argument passed to the constructor.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/a32e5c34-a07a-4e12-9f55-f9a4f6d0cda5)

A copy constructor in C++ is a constructor that initializes a new object as a copy of an existing object. This copy constructor takes one argument: a reference to a constant **`Fighter`** object **`(const Fighter&)`**. This **`Fighter`** object is the one that will be copied.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/2859c7db-7867-4c92-b575-7ac62d50d004)

The **`addFight`** method's purpose is to add a new fight to the **`fightRecord`** of a **`Fighter`** object. The **`fightRecord`** is a **`std::vector`** that stores strings, where each string represents the outcome of a fight.

When we call the **`addFight`** method and pass a string (representing a fight), the method attempts to add this string to the **`fightRecord`**. It does this using a function of the **`std::vector`** class called **`push_back`**. 

The **`push_back`** function takes a value and puts it at the end of the vector, sort of like adding a new item to the end of a line. In this case, it takes the string that represents the new fight and places it at the end of the **`fightRecord`**.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/892f2b2c-00d8-44f9-ac65-16035f8cf09e)

Before calling **`addFight:`**

```
Fight Record:
+---------------------+------------------------+
| "Win against Jose Aldo" | "Loss against Nate Diaz" |
+---------------------+------------------------+
```

After calling **`addFight("Win against Eddie Alvarez"):`**

```
Fight Record:
+---------------------+------------------------+-------------------------+
| "Win against Jose Aldo" | "Loss against Nate Diaz" | "Win against Eddie Alvarez" |
+---------------------+------------------------+-------------------------+
```

This code defines a method called **`display()`** inside the Fighter class. The **`display()`** method's purpose is to print out the fighter's information to the console.

**`void display() const`** This is the declaration of the **`display`** method. It doesn't return any value (void), and it's a **const** method, meaning it promises not to modify any of the object's member variables. 

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/4e5bf70f-b961-492e-bc2a-b2e0bf7d3bc5)

**`for (const auto& fight : fightRecord)`** This is the start of a range-based for loop. It's a convenient way to iterate over all elements in a container like a **`std::vector`**. In this case, it's going through each fight in the **`fightRecord`**. 

The **`const`** keyword means fight won't be modified in the loop. The **`auto`** keyword is telling the compiler to automatically determine the type of **`fight`** from the **`fightRecord`** (which is **`std::string`** in this case).

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/084c310a-8f3b-4053-9801-557b97f92f96)

Let's consider **`fightRecord`** as a **`std::vector`** containing the following strings:

```
Fight Record:
+---------------------+------------------------+
| "Win against Jose Aldo" | "Loss against Nate Diaz" |
+---------------------+------------------------+
```

When we use a range-based for loop like for **`(const auto& fight : fightRecord)`**, it's similar to having a pointer that goes through each box (element) in the row from left to right, one box at a time.

```
Fight Record:
+---------------------+------------------------+
| "Win against Jose Aldo" | "Loss against Nate Diaz" |
+---------------------+------------------------+
  ^
  |
Pointer (fight)
```

After the first iteration, the pointer moves to the next box:

```
Fight Record:
+---------------------+------------------------+
| "Win against Jose Aldo" | "Loss against Nate Diaz" |
+---------------------+------------------------+
                          ^
                          |
                      Pointer (fight)
```

In each iteration of the loop, **`fight`** is a reference to the current box (element) that the pointer is at. The loop continues until the pointer has visited all boxes (elements) in the **`fightRecord`**.

# Move Constructors

This C++ code sets up a **Fighter** class with a name and techniques, uses a special process to move fighters around efficiently, and stops us from making duplicates. In the main part of the program, a **Fighter** is made and sent into the fight function, which shows the fighter's name and techniques.

```c
#include <iostream>
#include <vector>
#include <string>

class Fighter {
private:  // Make data members private
    std::string name;
    std::vector<std::string> techniques;  // techniques of the fighter

public:
    // Constructor
    Fighter(std::string n, std::vector<std::string> t) : name(n), techniques(std::move(t)) {}

    // Getter functions
    std::string getName() const { return name; }
    std::vector<std::string> getTechniques() const { return techniques; }

    // Setter functions
    void setName(const std::string& n) { name = n; }
    void setTechniques(const std::vector<std::string>& t) { techniques = t; }

    // Move constructor
    Fighter(Fighter&& src) noexcept : name(std::move(src.name)), techniques(std::move(src.techniques)) {
        src.name = "";
        src.techniques.clear();
    }

    // Deleted copy constructor and copy assignment operator
    Fighter(const Fighter& src) = delete;
    Fighter& operator=(const Fighter& src) = delete;
};

void fight(Fighter newFighter) {
    std::cout << "In the octagon: " << newFighter.getName() << "\nUsing techniques:\n";
    for (const auto& technique : newFighter.getTechniques()) {
        std::cout << "- " << technique << '\n';
    }
}

int main() {
    Fighter jonJones("Jon Jones", { "Elbows", "Spinning Back Kick", "Take Downs" });  // Create a fighter
    fight(std::move(jonJones));  // Jon Jones enters the octagon

    return 0;
}
```

This is a class definition for **Fighter** in C++. It has two private data members:

- **`name:`** This is a **`std::string`** that will be used to hold the **`name`** of the fighter.
- **`techniques:`** This is a **`std::vector<std::string>`** that will be used to hold a list of the fighter's **`techniques`**.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/56effeb6-bbb8-4cf0-a794-942cf3cfb4a5)

This is the constructor for the **`Fighter`** class. A constructor is a special function in a class that is automatically called when an object of the class is created. This constructor takes two parameters:

- A **`std::string`** called **`n`**, which is used to initialize the **`name`** member of the class.
- A **`std::vector<std::string>`** called **`t`**, which is used to initialize the **`techniques`** member of the class.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/73016017-3897-46f6-9219-049feaecef1d)

  
These are **getter functions** for the **`Fighter`** class, providing **read-only** access to the **`name`** and **`techniques`** data members. They don't modify any class members, as indicated by the **`const`** keyword.

- **`getName():`** This function returns the **`name`** of the fighter. It's a **`const`** method, which means it doesn't modify any class members.
- **`getTechniques():`** This function returns the **`techniques`** of the fighter. It's also a **`const`** method, meaning it doesn't modify any class members.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/6081cc7d-73e0-4050-bc57-6942f2cbb251)

**Setter functions** are methods used in a class to modify the values of its private data members. They provide a controlled way to change the data of an object from outside the class.

- **`setName():`** This function sets the **`name`** of the fighter to the string **`n`** passed as an argument.
- **`setTechniques():`** This function sets the **`techniques`** of the fighter to the vector **`t`** passed as an argument.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/c6f5d6fa-f668-4239-beec-eab55c7fc0c2)

This is the **move** constructor for the **`Fighter`** class. It's used to create a new **`Fighter`** object from an existing one by transferring ownership of resources, rather than copying them.

- **`Fighter(Fighter&& src) noexcept`** declares a move constructor that takes an rvalue reference to another **`Fighter`** object.
- **`: name(std::move(src.name)), techniques(std::move(src.techniques))`** is an initializer list that moves the **`name`** and **`techniques`** from the source object to the newly constructed object.

- **`src.name = ""; src.techniques.clear();`** sets the source object's **`name`** to an **empty** string and **clears** the **`techniques`** vector.

When a move operation is performed, we're transferring ownership of resources from the source object to a new object. Once the resources have been moved, they are no longer valid in the source object. By doing this, we ensure that the source object remains in a valid state after its resources have been moved. 

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/b95e55b4-d46c-4a48-8c56-6c2dba26a25c)

Let's visualize this with a simple diagram. We'll use two **`Fighter`** objects: **`src`** and **`newFighter`**.

Before the move operation:

```
src: 
  name: "Jon Jones"
  techniques: ["Elbows", "Spinning Back Kick", "Take Downs"]

newFighter:
  name: ""
  techniques: []
```

During the move operation, the **`name`** and **`techniques`** of **`src`** are moved to **`newFighter`**

```
src: 
  name: "Jon Jones" --> newFighter
  techniques: ["Elbows", "Spinning Back Kick", "Take Downs"] --> newFighter

newFighter:
  name: "Jon Jones"
  techniques: ["Elbows", "Spinning Back Kick", "Take Downs"]
```

After the move operation, **`src.name`** is set to an empty string and **`src.techniques`** is cleared. Now, **`newFighter`** has the **`name`** and **`techniques`** of the original **`src`** object, and src is left in a valid but unspecified state.

```
src: 
  name: ""
  techniques: []

newFighter:
  name: "Jon Jones"
  techniques: ["Elbows", "Spinning Back Kick", "Take Downs"]
```

These two lines of code prohibit creating a copy of a **`Fighter`** object, either by using the copy constructor or the copy assignment operator. If we try to copy a **`Fighter`** object, the compiler will throw an error.

To avoid any confusion. However, in the context of our code, **`delete`** is used to **delete** special member functions, such as the copy constructor and copy assignment operator. We are not talking about releasing dynamically allocated memory for example.

When we mark a function as **`= delete`**, we're telling the compiler that this function cannot be used in any context. If someone tries to use a deleted function, the code will not compile.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/04a0cb8a-8c0b-404a-90c8-c9391dcd18a5)

The line **`fight(std::move(jonJones));`** is responsible for calling the **`fight`** function with the **`jonJones`** object as its argument. Instead of passing **`jonJones`** directly, it uses **`std::move(jonJones)`**. This means that **`jonJones`** is being passed as an **rvalue** (temporary value), and it triggers the move constructor of the **`Fighter`** class.

This moves the **`jonJones`** object into the **`fight`** function, where it is accepted as **`newFighter`**. **`jonJones`** is moved into **`newFighter`** within the **`fight`** function, and **`jonJones`** is left in a valid but unspecified state.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/4b04d318-0222-4be7-9aaf-5a28671e882d)

Here's an ASCII visualization of how **`fight(std::move(jonJones));`** works:

Before the **`fight`** function call:

```
jonJones:
  name: | "Jon Jones" |
  techniques: | "Elbows", "Spinning Back Kick", "Take Downs" |
```

During the **`fight`** function call:

```
jonJones:
  name: | "Jon Jones" | --------> | "Jon Jones" |
  techniques: | "Elbows", "Spinning Back Kick", "Take Downs" | --------> | "Elbows", "Spinning Back Kick", "Take Downs" |

newFighter (within fight function):
  name: | "Jon Jones" |
  techniques: | "Elbows", "Spinning Back Kick", "Take Downs" |
```

After the **`fight`** function call:

```
jonJones:
  name: | |
  techniques: | |

newFighter (within fight function):
  name: | "Jon Jones" |
  techniques: | "Elbows", "Spinning Back Kick", "Take Downs" |
```

In this visualization, the lines with arrows represent the move operation, and **`jonJones`** and **`newFighter`** are **`Fighter`** objects. 

# Exploring Deleted Copy Mechanics in C++

This C++ code shows what happens when you try to copy an object but the copy feature is turned off. It tries to copy a Fighter to start a fight, but it can't because the copy feature is deleted. This makes the code unable to run.

```c
#include <iostream>
#include <vector>
#include <string>

class Fighter {
private:  // Make data members private
    std::string name;
    std::vector<std::string> techniques;  // techniques of the fighter

public:
    // Constructor
    Fighter(std::string n, std::vector<std::string> t) : name(n), techniques(std::move(t)) {}

    // Getter functions
    std::string getName() const { return name; }
    std::vector<std::string> getTechniques() const { return techniques; }

    // Setter functions
    void setName(const std::string& n) { name = n; }
    void setTechniques(const std::vector<std::string>& t) { techniques = t; }

    // Move constructor
    Fighter(Fighter&& src) noexcept : name(std::move(src.name)), techniques(std::move(src.techniques)) {
        src.name = "";
        src.techniques.clear();
    }

    // Deleted copy constructor and copy assignment operator
    Fighter(const Fighter& src) = delete;
    Fighter& operator=(const Fighter& src) = delete;
};

void fight(Fighter newfighter) {
    std::cout << "In the octagon: " << newfighter.getName() << "\nUsing techniques:\n";
    for (const auto& technique : newfighter.getTechniques()) {
        std::cout << "- " << technique << '\n';
    }
}

int main() {
    Fighter jonJones("Jon Jones", { "Elbows", "Spinning Back Kick", "Take Downs" });  // Create a fighter
    fight(jonJones);  // Jon Jones enters the octagon

    return 0;
}
```

When we call **`fight(jonJones)`**, the program tries to copy **`jonJones`** into **`newfighter`**. But, since the copy constructor is deleted, the compiler throws an error as it can't perform the copy operation.

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/54934fd3-b8da-42f7-935f-a3674d50f156)


This line tells the compiler that it should not allow copying of **`Fighter`** objects:

![image](https://github.com/DebugPrivilege/Debugging/assets/63166600/d028880e-e7b0-4f3e-8f45-498dd7636959)


