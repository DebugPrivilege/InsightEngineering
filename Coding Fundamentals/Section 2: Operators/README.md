# What are Operators?

Operators in the (C) programming language are symbols that tell the compiler to perform specific **mathematical** or **logical** manipulations. They are used within the C program to perform operations on data and variables. Here is a list of **common** operators:

| Operator Type | Description | Operators |
| --- | --- | --- |
| Arithmetic | Used to perform mathematical operations like addition, subtraction, multiplication, division, and modulus. | `+`, `-`, `*`, `/`, `%` |
| Relational | Used to compare two values. | `==`, `!=`, `>`, `<`, `>=`, `<=` |
| Logical | Used to combine two or more conditions/constraints or to reverse the logical state. | `&&`, `\|\|`, `\|` |
| Bitwise | Used to perform bit-level operations. | `&`, `\|`, `^`, `~`, `<<`, `>>` |
| Assignment | Used to assign values to variables. | `=`, `+=`, `-=` , `*=` , `/=`, `%=`, `&=`, `\|=` , `^=`, `<<=`, `>>=` |
| Increment/Decrement | Unary operators that increase or decrease the value of the variable by one. | `++`, `--` |
| Conditional/Ternary | This operator consists of three operands and is used to evaluate Boolean expressions. The goal of the operator is to decide which value should be assigned to the variable. | `Variable = Expression1 ? Expression2 : Expression3` |
| Special | Special operators like sizeof (which gives the size of a variable), the comma operator (which separates two or more expressions), and the pointer operators * and &. | `sizeof`, `','`, `*`, `&` |

# Arithmetic Operators

Arithmetic operators are used in the C programming language (and most other programming languages as well) to perform mathematical operations on numerical values. They operate on numbers.

Here's a simple example of using arithmetic operators in C:

```c
#include <stdio.h>

int main() {

    int a = 10, b = 4;  // Declare two integer variables, a and b, and assign them the values 10 and 4.

    printf("a + b = %d\n", a + b);  // Add a and b (10 + 4), then print the result. %d is a placeholder for an integer value.

    printf("a - b = %d\n", a - b);  // Subtract b from a (10 - 4), then print the result. 

    printf("a * b = %d\n", a * b);  // Multiply a and b (10 * 4), then print the result. 

    printf("a / b = %f\n", (float)a / b);  // Divide a by b (10 / 4), with a cast to float to prevent integer division. This prints the result as a floating-point number. %f is a placeholder for a float value.

    printf("a %% b = %d\n", a % b);  // Find the remainder when a is divided by b (10 % 4), then print the result. %% is used to print a single % character.

    return 0; 
}
```

This code performs some basic arithmetic operations using two integer variables, **a** and **b**, with values **10** and **4**, respectively. It prints the results of adding, subtracting, multiplying, and dividing these two numbers, as well as the remainder of their division.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/73633770-87f1-441f-b5f8-11396c098416)


# Relational Operators

Relational operators in C are used to compare two values. They evaluate to **1** if the comparison is **true**, and to **0** if the comparison is **false** for instance.

Let's demonstrate that with code:

```c
#include <stdio.h>

int main() {
    int a = 10, b = 5;

    printf("a == b: %d\n", a == b);  // Checks if a is equal to b
    printf("a != b: %d\n", a != b);  // Checks if a is not equal to b
    printf("a > b: %d\n", a > b);  // Checks if a is greater than b
    printf("a < b: %d\n", a < b);  // Checks if a is less than b
    printf("a >= b: %d\n", a >= b);  // Checks if a is greater than or equal to b
    printf("a <= b: %d\n", a <= b);  // Checks if a is less than or equal to b

    return 0;
}
```
When we run this code, it compares the values of **a** and **b** using each of the relational operators. The result of each comparison is printed to the console. Because relational operators return **1** for **true** and **0** for **false**, the program will print **1** for any comparison that is **true** and **0** for any comparison that is **false**.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/b5277aa2-1d74-430f-bc10-8e794e442725)


# Logical Operators

Logical operators are used to perform logical operations, such as conjunction **(AND)**, disjunction **(OR)**, and negation **(NOT)**. They are often used in conditional statements.

| Operator | Description | Example |
|---|---|---|
| `&&` (Logical AND) | Checks if both conditions are true. | `(5 > 3) && (10 > 5)` returns true because both 5 is greater than 3 and 10 is greater than 5. |
| `\|\|` (Logical OR) | Checks if at least one condition is true. | `(5 < 3) \|\| (10 > 5)` returns true because, even though 5 is not less than 3, 10 is greater than 5. |
| `!` (Logical NOT) | Makes a true condition false, and a false condition true. | `!(5 > 3)` returns false because 5 is greater than 3, but the `!` operator inverts it to false. |

Let's demonstrate logical operators with a code snippet:

```c
#include <stdio.h>

int main() {
    int a = 10; // Declaring integer variable a and assigning it the value 10
    int b = 20; // Declaring integer variable b and assigning it the value 20
    int c = 30; // Declaring integer variable c and assigning it the value 30

    // Using the AND operator (&&)
    // This operator returns true if both conditions are true.
    // Here, it checks whether a is less than b AND b is less than c.
    if (a < b && b < c) {
        printf("Both conditions are true.\n"); // If both conditions are true, this line is printed.
    } else {
        printf("At least one condition is false.\n"); // If either or both conditions are false, this line is printed.
    }

    // Using the OR operator (||)
    // This operator returns true if at least one of the conditions is true.
    // Here, it checks whether a is greater than b OR b is less than c.
    if (a > b || b < c) {
        printf("At least one condition is true.\n"); // If at least one condition is true, this line is printed.
    } else {
        printf("Both conditions are false.\n"); // If both conditions are false, this line is printed.
    }

    // Using the NOT operator (!)
    // This operator inverts the truth value of its operand.
    // Here, it checks whether a is NOT greater than b.
    if (!(a > b)) {
        printf("The condition (a > b) is false.\n"); // If a is not greater than b (i.e., a <= b), this line is printed.
    } else {
        printf("The condition (a > b) is true.\n"); // If a is greater than b, this line is printed.
    }

    return 0; // Indicates that the program has executed successfully.
}
```
![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/7a8ad54b-7643-48f5-b00a-f7d0d9fc1b47)


# Assignment Operators

Assignment operators in programming are used to assign values to variables. They allow you to store a value in a variable or update the value of a variable. In C, the most common assignment operator is the equals sign **(=)**. It assigns the value on the right side of the operator to the variable on the left side.

```c
int x = 5;  // Assigning the value 5 to variable x
```

List of common Assignment Operators:

| Operator | Description | Example |
| --- | --- | --- |
| `+=` | Add and assign: Adds the value on the right side to the variable on the left side and assigns the result to the variable. | `x += 5;` (Equivalent to `x = x + 5;`) |
| `-=` | Subtract and assign: Subtracts the value on the right side from the variable on the left side and assigns the result to the variable. | `x -= 3;` (Equivalent to `x = x - 3;`) |
| `*=` | Multiply and assign: Multiplies the value on the right side with the variable on the left side and assigns the result to the variable. | `x *= 2;` (Equivalent to `x = x * 2;`) |
| `/=` | Divide and assign: Divides the variable on the left side by the value on the right side and assigns the result to the variable. | `x /= 4;` (Equivalent to `x = x / 4;`) |
| `%=` | Modulo and assign: Performs the modulo operation on the variable on the left side with the value on the right side and assigns the result to the variable. | `x %= 3;` (Equivalent to `x = x % 3;`) |
| `&=` | Bitwise AND and assign: Performs a bitwise AND operation between the variable on the left side and the value on the right side, and assigns the result to the variable. | `x &= 6;` (Equivalent to `x = x & 6;`) |
| `\|=` | Bitwise OR and assign: Performs a bitwise OR operation between the variable on the left side and the value on the right side, and assigns the result to the variable. | `x \|= 9;` (Equivalent to `x = x \| 9;`) |
| `^=` | Bitwise XOR and assign: Performs a bitwise XOR operation between the variable on the left side and the value on the right side, and assigns the result to the variable. | `x ^= 12;` (Equivalent to `x = x ^ 12;`) |
| `<<=` | Left shift and assign: Shifts the bits of the variable on the left side to the left by the number of positions specified by the value on the right side, and assigns the result to the variable. | `x <<= 2;` (Equivalent to `x = x << 2;`) |
| `>>=` | Right shift and assign: Shifts the bits of the variable on the left side to the right by the number of positions specified by the value on the right side, and assigns the result to the variable. | `x >>= 3;` (Equivalent to `x = x >> 3;`) |

Let's cover an example of using the **+=** operator.

```c
#include <stdio.h>

int main() {
    int x = 5; // Initialize variable x with the value 5

    printf("Initial value of x: %d\n", x);

    // Using the += operator to add 3 to x and assign the result back to x
    x += 3;

    printf("After using += operator: %d\n", x);

    return 0;
}
```

We initialize a variable **x** with the value **5** and then use the **+=** operator, we add **3** to **x** and assign the result back to **x**. Further, we print the updated value of **x** to the console.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/ac8cb0cb-5bbb-48ca-87c1-60e527c68461)


# Bitwise Operators

Bitwise operators are used in most programming languages for manipulating the individual bits of a number. Before we delve into more detail, let's explain some relevant concepts around binary numbers. 

A binary number is a number that uses only two symbols: **0** and **1**. It is expressed in the **base-2** numeral system. In this system, each digit in a binary number represents a specific 'power' of **2**. By combining these digits, binary numbers can represent numerical values and encode various types of information in computing and digital systems.

Here is a bit how the logic works:

| Power | Value |
|-------|-------|
| 2^7   | 128   |
| 2^6   | 64    |
| 2^5   | 32    |
| 2^4   | 16    |
| 2^3   | 8     |
| 2^2   | 4     |
| 2^1   | 2     |
| 2^0   | 1     |

Let's say we have the decimal **15** what would the binary representation of **15** look like? To represent the decimal number 15 in binary form, we start with the highest power in the table (2^7) and move towards the lowest power (2^0).

- **2^7 (128)**: Since 128 is larger than 15, we put a 0 in the corresponding bit position.
- **2^6 (64)**: Again, 64 is larger than 15, so we put a "0" in that position.
- **2^5 (32)**: 32 is also larger than 15, so we put a "0" in that position.
- **2^4 (16)**: Similar to the previous powers, 16 is larger than 15, so we put a "0" there.
- **2^3 (8)**: Now, 8 is smaller than 15, so we put a "1" in that position.
- **2^2 (4)**: 4 is also smaller than 15, so we put a "1" in that position.
- **2^1 (2)**: 2 is smaller than 15, so we put a "1" there.
- **2^0 (1)**: 1 is smaller than 15, so we put a "1" in that position.

The result will be the following:

| Power | Value |
|-------|-------|
| 2^7 (127   |   0   |
| 2^6 (64)   |   0   |
| 2^5 (32)   |   0   |
| 2^4 (16)   |   0   |
| 2^3 **(8)**   |   1   |
| 2^2 **(4)**   |   1   |
| 2^1 **(2)**   |   1   |
| 2^0 **(1)**   |   1   |

Adding up these powers of 2, we get: 8 + 4 + 2 + 1 = 15. Therefore, the binary representation **00001111** is equal to the decimal number **15**.

Here is a list of **Bitwise Operators**:

| Operator | Description | Example |
| -------- | ----------- | ------- |
| `&` | Bitwise AND: It takes two numbers as operands and does AND on every bit of two numbers. The result of AND is 1 only if both bits are 1. | `int a = 12; int b = 25; int result = a & b; // result is 8` |
| `\|` | Bitwise OR: It takes two numbers as operands and does OR on every bit of two numbers. The result of OR is 1 if any of the two bits is 1. | `int a = 12; int b = 25; int result = a \| b; // result is 29` |
| `^` | Bitwise XOR: It takes two numbers as operands and does XOR on every bit of two numbers. The result of XOR is 1 if the two bits are not the same. | `int a = 12; int b = 25; int result = a ^ b; // result is 21` |
| `~` | Bitwise NOT: It takes one number and inverts all bits of it. | `int a = 12; int result = ~a; // result is -13` |
| `<<` | Left shift: It takes two numbers, left shifts the bits of the first operand, the second operand decides the number of places to shift. | `int a = 5; int result = a << 2; // result is 20` |
| `>>` | Right shift: It takes two numbers, right shifts the bits of the first operand, the second operand decides the number of places to shift. | `int a = 20; int result = a >> 2; // result is 5` |
| `<<=` | Left shift and assignment: The left shift and assignment operator shifts the bits of the left operand to the left and assigns the result to the left operand. | `int a = 5; a <<= 2; // a is now 20` |
| `>>=` | Right shift and assignment: The right shift and assignment operator shifts the bits of the left operand to the right and assigns the result to the left operand. | `int a = 20; a >>= 2; // a is now 5` |

Let's demonstrate some code to understand this better and then further explain it. We will be using the **&** Operator as example. The **&** operator performs a bitwise **AND** operation, comparing each bit of the first operand to the corresponding bit of the second operand. If both bits are **1**, the corresponding result bit is set to **1**. Otherwise, the result bit is **0**.

```c
#include <stdio.h>

int main() {
    int a = 12; // Declare an integer 'a' and assign it the value 12
    int b = 25; // Declare an integer 'b' and assign it the value 25

    // Perform a bitwise AND operation on 'a' and 'b' and store the result in 'result'
    int result = a & b;

    // Print the result
    printf("Result of a & b is: %d\n", result);  // This will output: Result of a & b is: 8

    return 0; // Indicate that the program has executed successfully
}
```
You may start wondering. How did we came to the conclusion the value is **8**?

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/666114a7-7929-40e2-84f2-00bcdbe2fd61)


First, let's start with 12 and understand the binary representation of it.

| Power      | Value |
|------------|-------|
| 2^7 (128)  |   0   |
| 2^6 (64)   |   0   |
| 2^5 (32)   |   0   |
| 2^4 (16)   |   0   |
| 2^3 **(8)**   |   1   |
| 2^2 **(4)**   |   1   |
| 2^1 (2)  |   0   |
| 2^0 (1)   |   0   |

The 8-bit binary representation for 12 is **00001100**

For 25, it is the following:

| Power      | Value |
|------------|-------|
| 2^7 (128)  |   0   |
| 2^6 (64)   |   0   |
| 2^5 (32)   |   0   |
| 2^4 **(16)**   |   1   |
| 2^3 **(8)**   |   1   |
| 2^2 (4)  |   0   |
| 2^1 (2)  |   0   |
| 2^0 **(1)**   |   1   |

The 8-bit binary representation for 25 is **00011001**, so we can now start calculating.

```
00001100 <---- 12
00011001 <---- 25
----------------- (Applying AND operation)
00001000 (8 in Decimal)
```
Why is **00001000** equals **8**?

| Power      | Value |
|------------|-------|
| 2^7 (128)  |   0   |
| 2^6 (64)   |   0   |
| 2^5 (32)   |   0   |
| 2^4 (16) |   0   |
| 2^3 **(8)**   |   1   |
| 2^2 (4)  |   0   |
| 2^1 (2)  |   0   |
| 2^0 (1)  |   0   |

# Unary Operators - Increase/Decrement

Unary operators are single-operand operations in programming that perform specific tasks like incrementing a value **(++)**, negating a value **(-)**.

Let's start with incrementing a value using the **++** operator.

```c
#include <stdio.h>

int main() {
    int num = 0;

    printf("Initial value of num: %d\n", num);

    // Use the increment operator
    num++;

    printf("Value of num after increment: %d\n", num);

    return 0;
}
```
We start with a variable **num** set to **0**. We then use the **++** operator to increment the value of **num** by one. The printf function is used to print the value of **num**.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/e30642f6-4b58-4992-aac8-dc897c079dfe)



To finalize this. Here is an example of demonstrating the decrement operator **(--)**.

```c
#include <stdio.h>

int main() {
    int num = 10;

    printf("Initial value of num: %d\n", num);

    // Use the decrement operator
    num--;

    printf("Value of num after decrement: %d\n", num);

    return 0;
}
```

We start with a variable **num** set to **10**. We then use the **--** operator to decrement the value of **num** by one. 

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/6d3adf87-bfad-41d4-889d-d68b0e180de4)


# Conditional/Ternary Operators

A ternary operator is a type of operator that takes three operands. The most common ternary operator is the conditional operator, which is represented by **?**:.

The conditional operator works like a compact form of an if-else statement.

```
condition ? expression_if_true : expression_if_false;
```

First, the **condition** is evaluated. If the **condition** is **true**, then **expression_if_true** is evaluated, and its result is the result of the whole ternary operation. If the condition is **false**, then **expression_if_false** is evaluated, and its result is the result of the whole ternary operation.

Let's demonstrate this with a simple code example:

```c
#include <stdio.h>

int main() {
    int a = 10;
    int b = 20;

    int max_value = (a > b) ? a : b; // Use ternary operator

    printf("The maximum value between a and b is: %d\n", max_value);

    return 0;
}
```
We have two integer variables, **a** and **b**. We then use the ternary operator to determine the maximum value between **a** and **b**, storing it in the **max_value** variable. The printf function is then used to display this maximum value. Since **b** is larger than **a**, the program will output 20.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/7cf8bcbb-8a29-4951-ab36-6e937fccaff5)


```
int max_value = (a > b) ? a : b;
```

The **condition** is a > b, which checks if the value of a is greater than the value of **b**. **expression_if_true** is **a**. This is the value that will be assigned to **max_value** if the condition a > b is true. If the condition a > b is true, then the ternary operator will return **a**, else it will return **b**.

# Special Operators

Special operators are used to perform specific operations that cannot be done with normal arithmetic or logical operators. These operators are special because they have their own unique syntax and functionality.

| Operator | Description |
| --- | --- |
| `&` | Used to get the memory address of a variable. In this case, it gets the address of `a`. |
| `*` | Used to dereference a pointer variable, i.e., get the value at the memory location pointed to by the pointer. In this case, it retrieves the value of `a` that `ptr` points to. |
| `,` | Used to link related expressions together. In this case, it increments `b` and then adds 10 to the result, storing the final result in `x`. |
| `sizeof` | Used to compute the size of a data type in bytes. In this case, it calculates the size of `int`. |

Let's demonstrate all the mentioned operators with a code snippet:

```c
#include <stdio.h>

int main() {
    int a = 10, b = 20;
    int* ptr;
    ptr = &a;  // '&' operator is used to get the address of the variable

    // Printing the memory address that 'ptr' is pointing to
    printf("Address of a: %p\n", (void*)ptr);

    // '*' operator is used for dereferencing pointer i.e. getting the value at pointer address
    printf("Value of a: %d\n", *ptr);

    // ',' operator is used to link related expressions together
    int x = (b++, b + 10);
    printf("Value of b: %d\n", b);  // Output will be 21
    printf("Value of x: %d\n", x);  // Output will be 31, as the increment happens before the addition

    // 'sizeof' operator is used to compute the size of its operand
    printf("Size of int: %zu bytes\n", sizeof(int));  // Output is typically 4 bytes

    return 0;
}
```

**Value of a: 10**

The variable **'a'** is initially assigned a value of **10** with the statement **int a = 10;**. Next, a pointer **ptr** is created and assigned the memory address of **'a'** with **ptr = &a;**. The **&** operator retrieves the memory address of **'a'**.

The * operator is used to dereference a pointer, i.e., to retrieve the value stored at the memory location pointed to by the pointer. ****ptr** is the value of **'a'** because **ptr** is holding the memory address of **'a'**. As a result of these operations, the value of **'a'** is printed as **10**. This is because **'a'** was initially assigned the value of **10**, and the * operator was used to retrieve this value through the pointer **ptr**.


**Value of b: 21**

The **b++** operation is performed first due to the comma operator which evaluates expressions from left to right. This operation increments **b** by **1**, so **b** changes from its initial value of **20** to **21**.

**Value of x: 31**

After **b** is incremented, the operation **b + 10** is performed, which is now **21 + 10**, resulting in **31**. The comma operator evaluates to the value of the rightmost expression, so **x** is assigned the result of **b + 10**, which is **31**.


![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/f2f0ecef-9a59-4cdb-99a0-6e6c44b201b3)

