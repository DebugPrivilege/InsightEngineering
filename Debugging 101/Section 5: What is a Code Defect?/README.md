# What is a Code Defect?

A code defect, often called a bug, is a mistake or oversight in a software program that leads to incorrect or unexpected behavior. These defects can appear at any point during the software development process. They may result in issues like software malfunctions, system crashes, inaccurate calculations, or other unexpected outcomes.

There are different types of code defects:

- **Logical Errors:** Errors in the logic of the program, which may lead to incorrect results but won't necessarily cause a crash.
- **Runtime Errors:** Errors that occur while the program is running, such as dividing by zero or trying to access an invalid memory location.
- **Semantic Errors:** Errors where the code is syntactically correct but doesn't do what the programmer intended.

# Code Sample (1) - Unexpected result

This is a simple C++ example that attempts to read from an array, but it contains a defect that leads to unexpected results.

```c
#include <iostream>

int main() {
    int arr[] = {1, 2, 3, 4, 5};
    int size = sizeof(arr) / sizeof(arr[0]);

    // Attempt to read from the array
    for (int i = 1; i <= size; ++i) {  // Defect: Loop should start from 0 and go up to size-1
        std::cout << arr[i] << " ";
    }

    std::cout << std::endl;
    return 0;
}
```

**Defects in the code:**

The loop starts at index **1** and goes up to **`size`**, which means the first element of the array is skipped, and an attempt is made to access an element beyond the **array** boundary. This will lead to undefined behavior.

**Expected Output:**

The expected output should be the elements of the array: **`1 2 3 4 5`**

**Actual Output:**

The actual output will be: **`2 3 4 5 0`**

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/2b3f5d57-7351-44ed-8644-2637d8a7fdbe)

# Code Sample (2) - How to fix the Code Defect?

To fix this defect, the loop should start at index **0** and go up to **`size - 1`**.

```c
#include <iostream>

int main() {
    int arr[] = {1, 2, 3, 4, 5};
    int size = sizeof(arr) / sizeof(arr[0]);

    // Corrected loop to read from the array
    for (int i = 0; i < size; ++i) {
        std::cout << arr[i] << " ";
    }

    std::cout << std::endl;
    return 0;
}
```

This corrected version of the code will produce the expected output: **`1 2 3 4 5`**

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/ae750b25-fadc-46c0-b2da-28a5b4aa6e8f)

