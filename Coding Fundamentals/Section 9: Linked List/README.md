# What is a Linked List?

A linked list is a type of data structure that consists of a group of nodes in a chain, where each node holds its own data and a pointer or reference to the next node in the chain. This structure allows for efficient insertions and deletions.

There are several types of linked lists, but the two most common are singly linked lists and doubly linked lists.

- **Singly Linked List**

Each node in the list stores the data of the node and a pointer or reference to the next node in the list. It does not store any pointer or reference to the previous node. It is called a singly linked list because each node only has a single link to another node.

```c
typedef struct Node {
  int data;
  struct Node* next;
} Node;
```

- **Doubly Linked List**

A doubly linked list is a linked list in which each node of the list has two links, one link to the next node and another link to the previous node.

```c
typedef struct Node {
  int data;
  struct Node* next;
  struct Node* prev;
} Node;
```

Linked lists are a fundamental concept in computer science. Here are the core concepts associated with a linked list:

| Concept | Description | Example |
|---------|-------------|---------|
| Node | The basic unit of a linked list, containing data and one or more pointers | `struct Node { int data; struct Node* next; };` |
| Data | The information stored in each node | `int data;` in a node structure |
| Pointer | A variable storing the address of another variable. Used in a linked list to connect nodes | `struct Node* next;` in a node structure |
| Head | The first node in the list. It provides access to the entire list | `struct Node* head = &firstNode;` |
| Tail | The last node in the list. In a singly linked list, its next pointer is NULL | `struct Node* tail = &lastNode; tail->next = NULL;` |
| Singly Linked List | A type of linked list where each node points to the next node | `struct Node { int data; struct Node* next; };` |
| Doubly Linked List | A type of linked list where each node has pointers to the next and the previous node | `struct Node { int data; struct Node* next; struct Node* prev; };` |
| Operations | Basic actions that can be performed on a linked list like insertion, deletion, traversal, and searching | `insertNode(head, newNode); deleteNode(head, targetNode);` |
| Dynamic Data Structure | Linked lists are dynamic, they can grow and shrink at runtime, allocating and deallocating memory as necessary | When you add a new node to the list, memory is allocated for it dynamically |
| Null | Used to denote the end of the list. It is the value of the next pointer in the last node | `lastNode->next = NULL;` |

# Singly Linked List

This is an example of a basic implementation of a singly linked list and its operations such as inserting a node at the end of the list, traversing the list to print the nodes, and freeing the list to avoid memory leaks. Each node in the list represents a Mixed Martial Arts (MMA) fighter and contains a string with the fighter's name and a pointer to the next node in the list.

```c
#include <stdio.h> 
#include <stdlib.h> 
#include <string.h>

// Define structure for a node
typedef struct Node {
    char fighter[50];
    struct Node* next;
} Node;

// Function to add a new node at the end of the list
void insertEnd(Node** head_ref, const char* name) {
    // Allocate memory for new node and check if allocation was successful
    Node* newNode = (Node*)malloc(sizeof(Node));
    if (newNode == NULL) {
        fprintf(stderr, "Error! Memory not allocated.\n");
        exit(EXIT_FAILURE);
    }

    // Assign data to the new node using strcpy_s
    strcpy_s(newNode->fighter, sizeof(newNode->fighter), name);
    newNode->next = NULL; // Initialize next pointer to NULL

    // If the list is empty, the new node is the first node
    if (*head_ref == NULL) {
        *head_ref = newNode;
        return;
    }

    // Else, traverse till the last node
    Node* last = *head_ref;
    while (last->next != NULL) {
        last = last->next;
    }

    last->next = newNode; // Link the last node to the new node
}

// Function to print the linked list
void printList(Node* node) {
    while (node != NULL) {
        printf("%s\n", node->fighter);
        node = node->next;
    }
}

// Function to free the linked list
void freeList(Node* head) {
    Node* tmp;

    while (head != NULL) {
        tmp = head;
        head = head->next;
        free(tmp);
    }
}

// Driver program to test the functions
int main() {
    Node* head = NULL;  // Start with an empty list

    insertEnd(&head, "Jon Jones");  // Insert Jon Jones at the end. 
    insertEnd(&head, "Khabib Nurmagomedov");  // Insert Khabib Nurmagomedov at the end. 
    insertEnd(&head, "Conor McGregor");  // Insert Conor McGregor at the end. 
    insertEnd(&head, "Alex Pereira");  // Insert Alex Pereira at the end.

    printf("MMA Fighters:\n");
    printList(head);  // Print the linked list

    freeList(head);  // Free the linked list
    return 0;
}
```

At this code and within the **`main`** function, a linked list is created and four MMA fighters are added. The list is then printed and the memory is freed.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/6dfd3be9-a86f-4038-b9a2-06868452d213)

# Reversing the Singly Linked List

Let's break down this code in different pieces:

- **Node**

Each Node in the linked list is like a box that contains some data and a reference to the next node. The node is the fundamental building block of the linked list. Without nodes, there wouldn't be any data to track or any structure to maintain.

This Node contains two elements: a **`char`** array (representing fighter's name) and a **pointer** to the next node. The node not only contains the data (fighter's name), but also contains information about where the next node is located in memory.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/4000b1df-5d8a-4485-b562-9e0378c9bc3d)

- **Data**

**`char fighter[50]`** is the data each node carries. The **`fighter`** field is a character array capable of storing up to 49 characters plus a null character **`(\0)`** at the end, which indicates the end of the string in C. This string represents the name of an MMA fighter, and it's the data that we are interested in within each node of our linked list.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/a5736eab-4f83-48e7-83de-f20fbd9f70eb)

The **strcpy_s** function copies the name (input data) into the fighter member of **`newNode`**.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/9d6ca045-75ac-475a-ab11-3ae62f3c32c2)

- **Pointers**

A pointer is a variable that stores a memory address. In our code, we use pointers to keep track of where nodes are located in memory. The next member of each node is a pointer that 'points to' the next node in the list.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/7fb44532-9155-41fc-b7ae-0ac82ff39577)

The **horizontal** arrows between nodes represent next pointers. These are part of each node's structure, and they store the address of the next node in the list.

```
head
  |
  v
  [Jon Jones| ] --> [Khabib Nurmagomedov| ] --> [Conor McGregor| ] --> [Alex Pereira|NULL]
```

While the visualization isn't a literal representation of what's happening in the computer's memory, it's a useful way to understand the logic of a linked list and the role of pointers in connecting nodes.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/135d95ae-c3a3-4403-91bb-a5dd6008bb04)

- **Head**

The head is a pointer that points to the **first** node of the linked list. If head is NULL, it means the list is empty.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/ec34b9a3-a1c2-47b5-bf47-33210b02fbf9)

However, as soon as we start adding nodes using the **`insertEnd`** function, the list is no longer empty.

In this line, we're adding a node containing the string **"Jon Jones"** to the end of the list. Since this is the first node we're adding, it becomes the head of the list, so head now points to this node instead of NULL. Therefore, at this point, the list is no longer empty.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/dbefe93f-523d-4a3e-8eb3-5557c70b7375)

We continue to add more nodes after this, making our list:

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/e3563e97-f7af-411c-a088-67c67378e27e)

After adding all these nodes, our list now has four nodes in total (**"Jon Jones"**, **"Khabib Nurmagomedov"**, **"Conor McGregor"**, **"Alex Pereira"**), and the head pointer points to the first node (**"Jon Jones"**). Try to see Head as the entry point. It is a pointer that points to the first node in the list.

- **Operations**

**Insertion:** Insertion operation is represented by the **`insertEnd`** function, which inserts a new node at the end of the linked list.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/a00e8ca5-1dc3-4602-8f91-0488fc52f1e6)

The function takes two arguments, a double pointer to the head of the linked list, and a character pointer that holds the name of the fighter. Using the double pointer in this way ensures that the changes made to the head pointer within the **`insertEnd()`** function are reflected in the **main()** function, resulting in a correctly updated linked list.

```c
void insertEnd(Node** head_ref, const char* name)
```

This code snippet is responsible for allocating memory for a new node in the linked list. It checks if the memory allocation was successful and handles the error if it fails.

```c
    // Allocate memory for new node and check if allocation was successful
    Node* newNode = (Node*)malloc(sizeof(Node));
    if (newNode == NULL) {
        fprintf(stderr, "Error! Memory not allocated.\n");
        exit(EXIT_FAILURE);
    }
```

This code snippet is responsible for assigning data to the created node. It uses the **strcpy_s** function to copy the fighter's name (passed as (**name**) to the **`fighter`** member of the new node. After assigning the name, it sets the next pointer of the new node to NULL, initializing it as the last node in the list.

```c
// Assign data to the new node using strcpy_s
strcpy_s(newNode->fighter, sizeof(newNode->fighter), name);
newNode->next = NULL; // Initialize next pointer to NULL
```

The visualization represents the new node after assigning data to it. The **`fighter`** member is now filled with the fighter's name, and the next pointer remains NULL as it's the last node in the list.

```
Before assignment:
newNode
+---------+-----+
| fighter |next |
+---------+-----+
|         |NULL |
+---------+-----+

After assignment:
newNode
+-----------------+-----+
|     fighter     |next |
+-----------------+-----+
|   Jon Jones     |NULL |
+-----------------+-----+
```

Before the operation, the list is empty, which is indicated by the head of the list being NULL. However, after the operation ***`head_ref = newNode;`**, the list is no longer empty. The new created node (newNode) is now the head of the list. The list currently contains one node, which holds the data ("Jon Jones" in this case) in the **`fighter`** field.

```c
    // If the list is empty, the new node is the first node
    if (*head_ref == NULL) {
        *head_ref = newNode;
        return;
    }
```

**Before:**

```
head_ref: +------+
          | NULL | 
          +------+
```

**After:**

```
head_ref: +------+
          |  *   | ---> newNode:  +-----------+    +------+
          +------+                | fighter   |    | next |
                                  +-----------+    +------+
                                  | "Jon Jones" |  | NULL |
                                  +-----------+    +------+
```

This code snippet is responsible for adding a new node to the end of a non-empty linked list.

```c
    // Else, traverse till the last node
    Node* last = *head_ref;
    while (last->next != NULL) {
        last = last->next;
    }

    last->next = newNode; // Link the last node to the new node
}
```
This line creates a new pointer **last** that initially points to the **first node** in the list (the node that the head pointer points to).

```c
Node* last = *head_ref;
```

**head** and **last** are pointing to the first node in the list (the node containing "Jon Jones"). The **last** pointer will then be used to traverse the list and find the last node.

Here's an ASCII visualization of the state after executing **`Node* last = *head_ref;`**:

```
head
 | \
 |  \
 v   v
+---------------+     +----+
| "Jon Jones"   |     |    |
|     fighter   | --> |NULL|
+---------------+     +----+
      ^
      |
    last
```

This is a while loop. The purpose of this line is to traverse the linked list until it reaches the last node. It does this by continually moving to the next node **`(last = last->next;)`** as long as the next pointer of the current node is not NULL.

```c
    while (last->next != NULL) {
        last = last->next;
    }
```

Before the while loop starts, **`last`** is pointing to the head of the list:

```
head
  |
  V
[Jon Jones] --> [Khabib Nurmagomedov] --> [Conor McGregor] --> NULL
  ^
  |
last
```

The while loop starts. The next pointer of **"Jon Jones"** is not NULL, so last moves to the next node:

```
head
  |
  V
[Jon Jones] --> [Khabib Nurmagomedov] --> [Conor McGregor] --> NULL
                 ^
                 |
               last
```

Again, the next pointer of **"Khabib Nurmagomedov"** is not NULL, so last moves to the next node:

```
head
  |
  V
[Jon Jones] --> [Khabib Nurmagomedov] --> [Conor McGregor] --> NULL
                                       ^
                                       |
                                     last
```

Now, the next pointer of "Conor McGregor" is NULL, which indicates that **"Conor McGregor"** is the last node in the list. The condition **`last->next != NULL`** becomes false, and the loop ends. 

**`last = last->next;`** is used to traverse the list, moving the last pointer from one node to the next. In a loop, this line can be used to move through the entire list.

```
head
  |
  V
[Jon Jones] --> [Khabib Nurmagomedov] --> [Conor McGregor] --> NULL
                                       ^
                                       |
                                     last
```

In a singly linked list, each node has a next pointer that points to the next node in the list. The last node's next pointer is NULL, indicating the end of the list.

```
last->next = newNode; // Link the last node to the new node
```

Before the execution of **`last->next = newNode;`**, the last node is the last node in the list:

```
head
  |
  V
[Jon Jones] --> [Khabib Nurmagomedov] --> [Conor McGregor] --> NULL
                                       ^
                                       |
                                     last
```

After the execution of **`last->next = newNode;`**, the next pointer of the last node (which was NULL) is updated to point to the new node:

```
head
  |
  V
[Jon Jones] --> [Khabib Nurmagomedov] --> [Conor McGregor] --> [Alex Pereira] --> NULL
                                       ^
                                       |
                                     last
```

**last** is still pointing to **"Conor McGregor"**, but **"Conor McGregor"**'s next is now pointing to the new node **"Alex Pereira"**, effectively adding **"Alex Pereira"** to the end of the list.

**Traversal:** Goes through the list, typically starting from the head (the first node), and visiting each node in turn until the end of the list (where the next pointer is NULL) is reached. This operation is often used in conjunction with other operations, such as searching, printing, insertion, and deletion.

At this example, the **`printList`** function is the traversal operation. This function effectively traverses the list from the head to the tail, printing the name of each fighter as it goes.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/8e83e223-b9e5-4808-8683-4137bde4a749)

The function takes a pointer to a Node (typically the head of the list) as an argument. It enters a while loop that continues as long as the node it's looking at is not NULL. Inside the loop, it first prints the **`fighter`** field of the current node. It then updates the node pointer to point to the next node in the list using **`node = node->next;`**. Once the end of the list is reached (i.e., node->next is NULL), the loop exits.

Here's how the linked list looks initially:

```
head
  |
  v
[Jon Jones] --> [Khabib Nurmagomedov] --> [Conor McGregor] --> [Alex Pereira] --> NULL
```

When we call **`printList(head)`**, the function starts with the head of the list. The node variable initially points to the head of the list:

```
node
  |
  v
[Jon Jones] --> [Khabib Nurmagomedov] --> [Conor McGregor] --> [Alex Pereira] --> NULL
```

It enters the **while** loop and prints **"Jon Jones"**, the fighter field of the current node. It then moves to the next node using **`node = node->next;`**:

```
                node
                 |
                 v
[Jon Jones] --> [Khabib Nurmagomedov] --> [Conor McGregor] --> [Alex Pereira] --> NULL
```

It then prints **"Khabib Nurmagomedov"** and moves to the next node:

```
                                    node
                                     |
                                     v
[Jon Jones] --> [Khabib Nurmagomedov] --> [Conor McGregor] --> [Alex Pereira] --> NULL
```

Next, it prints **"Conor McGregor"** and moves to the next node:

```
                                                    node
                                                     |
                                                     v
[Jon Jones] --> [Khabib Nurmagomedov] --> [Conor McGregor] --> [Alex Pereira] --> NULL
```

It prints **"Alex Pereira"** and moves to the next node. However, **`node->next`** is NULL at this point, so node becomes NULL:

```
                                                                    node
                                                                     |
                                                                     v
[Jon Jones] --> [Khabib Nurmagomedov] --> [Conor McGregor] --> [Alex Pereira] --> NULL
```

# Doubly Linked List

A Doubly Linked List is a type of list where each item or node has a link to both the next item and the previous item in the list. This allows you to move forward and backward through the list, which can make certain operations easier or more efficient. It's like a chain where each link knows both the link ahead of it and the link behind it.

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

// Define structure for a node
typedef struct Node {
    char fighter[50];
    struct Node* next;
    struct Node* prev;
} Node;

// Function to add a new node at the end of the list
void insertEnd(Node** head_ref, const char* name) {
    // Allocate memory for new node and check if allocation was successful
    Node* newNode = (Node*)malloc(sizeof(Node));
    if (newNode == NULL) {
        fprintf(stderr, "Error! Memory not allocated.\n");
        exit(EXIT_FAILURE);
    }

    // Assign data to the new node using strcpy_s
    strcpy_s(newNode->fighter, sizeof(newNode->fighter), name);
    newNode->next = NULL; // Initialize next pointer to NULL
    newNode->prev = NULL; // Initialize prev pointer to NULL

    // If the list is empty, the new node is the first node
    if (*head_ref == NULL) {
        *head_ref = newNode;
        return;
    }

    // Else, traverse till the last node
    Node* last = *head_ref;
    while (last->next != NULL) {
        last = last->next;
    }

    last->next = newNode; // Link the last node to the new node
    newNode->prev = last; // Link the new node back to the last node
}

// Function to print the linked list
void printList(Node* node) {
    while (node != NULL) {
        printf("%s\n", node->fighter);
        node = node->next;
    }
}

// Function to free the linked list
void freeList(Node* head) {
    Node* tmp;

    while (head != NULL) {
        tmp = head;
        head = head->next;
        free(tmp);
    }
}

int main() {
    Node* head = NULL;  // Start with an empty list

    insertEnd(&head, "Jon Jones");  // Insert Jon Jones at the end. 
    insertEnd(&head, "Khabib Nurmagomedov");  // Insert Khabib Nurmagomedov at the end. 
    insertEnd(&head, "Conor McGregor");  // Insert Conor McGregor at the end. 
    insertEnd(&head, "Alex Pereira");  // Insert Alex Pereira at the end.

    printf("MMA Fighters:\n");
    printList(head);  // Print the linked list

    freeList(head);  // Free the linked list
    return 0;
}
```

The power of a doubly linked list comes into play when we want to traverse the list in a backward direction (from tail to head), or when we want to insert or delete nodes at both ends of the list more efficiently, or when we need to perform some operations that require knowledge of the previous node.

For example, we could add a function to traverse and print the list in reverse order like this:

```c
void printListReverse(Node* node) {
    Node* last = node;
    // Go to the last node
    while (last->next != NULL) {
        last = last->next;
    }
    // Print from the last node
    while (last != NULL) {
        printf("%s\n", last->fighter);
        last = last->prev;
    }
}
```
Let's visualize this step-by-step.

Suppose we have the following doubly linked list:

```
NULL <--> Jon Jones <--> Khabib Nurmagomedov <--> Conor McGregor <--> Alex Pereira <--> NULL
```

When we call **`printListReverse(head)`**, where head points to **"Jon Jones"**, we start by setting last = node (or last = head). Then we enter the first while loop, which moves last to the end of the list:

```
NULL <--> Jon Jones <--> Khabib Nurmagomedov <--> Conor McGregor <--> Alex Pereira <-- last
```

Now we enter the second while loop. On the first iteration, we print **"Alex Pereira"** and move last to its previous node:

```
NULL <--> Jon Jones <--> Khabib Nurmagomedov <--> Conor McGregor <-- last Alex Pereira
```

We continue this process, printing the fighter's name at last and moving last to its previous node, until last becomes **NULL**:

```
NULL <-- last Jon Jones <--> Khabib Nurmagomedov <--> Conor McGregor <--> Alex Pereira
```

At this point, we've printed all the fighters' names in reverse order:

```
Alex Pereira
Conor McGregor
Khabib Nurmagomedov
Jon Jones
```

We can now call **`printListReverse(head);`** to print the list in reverse order, which is something we couldn't do with a Singly Linked List. Here is the updated code:

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

// Define structure for a node
typedef struct Node {
    char fighter[50];
    struct Node* next;
    struct Node* prev;
} Node;

// Function to add a new node at the end of the list
void insertEnd(Node** head_ref, const char* name) {
    // Allocate memory for new node and check if allocation was successful
    Node* newNode = (Node*)malloc(sizeof(Node));
    if (newNode == NULL) {
        fprintf(stderr, "Error! Memory not allocated.\n");
        exit(EXIT_FAILURE);
    }

    // Assign data to the new node using strcpy_s
    strcpy_s(newNode->fighter, sizeof(newNode->fighter), name);
    newNode->next = NULL; // Initialize next pointer to NULL
    newNode->prev = NULL; // Initialize prev pointer to NULL

    // If the list is empty, the new node is the first node
    if (*head_ref == NULL) {
        *head_ref = newNode;
        return;
    }

    // Else, traverse till the last node
    Node* last = *head_ref;
    while (last->next != NULL) {
        last = last->next;
    }

    last->next = newNode; // Link the last node to the new node
    newNode->prev = last; // Link the new node back to the last node
}

// Function to print the linked list
void printList(Node* node) {
    while (node != NULL) {
        printf("%s\n", node->fighter);
        node = node->next;
    }
}

// Function to print the linked list in reverse order
void printListReverse(Node* node) {
    Node* last = node;
    // Go to the last node
    while (last->next != NULL) {
        last = last->next;
    }
    // Print from the last node
    while (last != NULL) {
        printf("%s\n", last->fighter);
        last = last->prev;
    }
}

// Function to free the linked list
void freeList(Node* head) {
    Node* tmp;

    while (head != NULL) {
        tmp = head;
        head = head->next;
        free(tmp);
    }
}

int main() {
    Node* head = NULL;  // Start with an empty list

    insertEnd(&head, "Jon Jones");  // Insert Jon Jones at the end. 
    insertEnd(&head, "Khabib Nurmagomedov");  // Insert Khabib Nurmagomedov at the end. 
    insertEnd(&head, "Conor McGregor");  // Insert Conor McGregor at the end. 
    insertEnd(&head, "Alex Pereira");  // Insert Alex Pereira at the end.

    printf("MMA Fighters in original order:\n");
    printList(head);  // Print the linked list in original order

    printf("\nMMA Fighters in reverse order:\n");
    printListReverse(head);  // Print the linked list in reverse order

    freeList(head);  // Free the linked list
    return 0;
}
```

And that's how **`printListReverse`** works. It takes advantage of the **`prev`** pointers in the doubly linked list to traverse the list in reverse order.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/03b2a358-e055-4d9a-abfd-6c9ed1c71d21)

# Final Example of Singly Linked List

Linked lists are often used in situations where you need to frequently **`add`** and **`remove`** elements, because these operations can be performed very efficiently. 

- **Singly Linked List**

This code creates a linked list of strings, and then writes those strings to a file.

```c
#include <windows.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>

// Node struct for singly linked list
typedef struct Node {
    wchar_t* data;  // The data stored in this node
    struct Node* next;  // Pointer to the next node
} Node;

// Singly linked list struct
typedef struct LinkedList {
    Node* head;  // Pointer to the first node
} LinkedList;

// Function to create a new node with given data
Node* createNode(const wchar_t* data) {
    // Allocate memory for new node
    Node* node = (Node*)malloc(sizeof(Node));
    if (node == NULL) {
        fwprintf(stderr, L"Failed to allocate memory for new node\n");
        exit(EXIT_FAILURE);
    }

    // Duplicate the given data for this node
    node->data = _wcsdup(data);
    if (node->data == NULL) {
        fwprintf(stderr, L"Failed to allocate memory for node data\n");
        exit(EXIT_FAILURE);
    }

    // Set the next node to NULL (end of list)
    node->next = NULL;
    return node;
}

// Function to add a new node with given data to the end of the list
void add(LinkedList* list, const wchar_t* data) {
    Node* node = createNode(data);
    if (list->head == NULL) {
        // If the list is empty, the new node is the head
        list->head = node;
    }
    else {
        // Find the last node
        Node* current = list->head;
        while (current->next != NULL) {
            current = current->next;
        }
        // Attach the new node at the end
        current->next = node;
    }
}

// Function to insert a new node with given data at the beginning of the list
void insertAtBeginning(LinkedList* list, const wchar_t* data) {
    Node* node = createNode(data);
    // The new node points to what was previously the first node
    node->next = list->head;
    // The new node becomes the list head
    list->head = node;
}

// Function to free the memory of a node
void freeNode(Node* node) {
    free(node->data);
    free(node);
}

// Function to free the memory of a linked list
void freeList(LinkedList* list) {
    Node* current = list->head;
    while (current != NULL) {
        Node* nextNode = current->next;
        freeNode(current);
        current = nextNode;
    }
}

// Function to write the data from the linked list to a file
void writeToFile(LinkedList* list, const wchar_t* filename) {
    // Create or overwrite the file
    HANDLE fileHandle = CreateFileW(
        filename,
        GENERIC_WRITE,
        0,
        NULL,
        CREATE_NEW,
        FILE_ATTRIBUTE_NORMAL,
        NULL
    );

    if (fileHandle == INVALID_HANDLE_VALUE) {
        fwprintf(stderr, L"Unable to create file due to error: %lu\n", GetLastError());
        return;
    }

    // Iterate over the linked list
    Node* current = list->head;
    while (current != NULL) {
        // Add a newline to each string before writing it
        size_t len = wcslen(current->data);
        wchar_t* dataWithNewline = (wchar_t*)malloc((len + 2) * sizeof(wchar_t));  // +2 for newline and null terminator
        if (dataWithNewline == NULL) {
            fwprintf(stderr, L"Failed to allocate memory for dataWithNewline\n");
            exit(EXIT_FAILURE);
        }

        // Copy original string and append newline
        errno_t err = wcscpy_s(dataWithNewline, len + 2, current->data);
        err = wcscat_s(dataWithNewline, len + 2, L"\n");

        // Write the string to the file
        DWORD bytesWritten;
        if (!WriteFile(
            fileHandle,
            dataWithNewline,
            wcslen(dataWithNewline) * sizeof(wchar_t),
            &bytesWritten,
            NULL
        )) {
            fwprintf(stderr, L"Unable to write to file due to error: %lu\n", GetLastError());
        }

        free(dataWithNewline);
        current = current->next;
    }

    fwprintf(stdout, L"File created and written successfully.\n");

    CloseHandle(fileHandle);
}

// Function to generate a random file name
wchar_t* generateRandomFilename() {
    // Seed the random number generator
    srand(time(NULL));
    int randNum = rand();

    // Allocate memory for the filename
    wchar_t* filename = (wchar_t*)malloc(sizeof(wchar_t) * 256);
    if (filename == NULL) {
        fwprintf(stderr, L"Failed to allocate memory for filename\n");
        exit(EXIT_FAILURE);
    }

    // Generate the filename
    swprintf(filename, 256, L"C:\\Temp\\testfile_%d.txt", randNum);
    return filename;
}

// Main function
int main() {
    // Create a linked list
    LinkedList list;
    list.head = NULL;

    // Add some data to the list
    add(&list, L"Hello, World!");
    insertAtBeginning(&list, L"New Beginning!");

    // Generate a random file name
    wchar_t* filename = generateRandomFilename();

    // Write the list data to the file
    writeToFile(&list, filename);

    // Clean up
    free(filename);
    freeList(&list);

    return 0;
}
```
When we execute this code, it will create a file on disk in the **C:\Temp** folder.

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/f92f06a0-2bea-4459-a4dc-85071092ddda)

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/758d3e8c-1b27-4adc-9b00-7d2bd3c70000)

![image](https://github.com/DebugPrivilege/InsightEngineering/assets/63166600/c076f4e0-806a-456c-9303-7145c91f163a)

Let's break down this code now. It's more important to understand the logic from my point of view, rather than knowing that this code will create a file on disk.

This code snippet is defining a new data type called **`Node`** which represents a node in a singly linked list. **typedef** is a keyword in the C language that is used to give a new name to an existing type. 

**`struct Node`** is a structure which is a user-defined data type in C that allows to combine data items of different kinds. Here we are defining a structure named **Node**, and then using **typedef** to allow us to refer to this type as **Node** instead of **struct Node**.

**`struct Node* next;`** is another member of the **Node** structure. This is a **pointer** to the next node in the list. This is what makes it a linked list: each node has a link to the next node in the list. In a singly linked list like this one, each node has a link to the next node, but not to the previous node.

```c
// Node struct for singly linked list
typedef struct Node {
    wchar_t* data;  // The data stored in this node
    struct Node* next;  // Pointer to the next node
} Node;
```

This code defines a new type **`LinkedList`** that represents a singly linked list. A **`LinkedList`** contains a head pointer which points to the first node in the list. If the list is empty, head would be NULL.

**`Node* head`;** is a member of the **`LinkedList`** structure. This member is a **pointer** to a Node structure. This head **pointer** is typically used to keep track of the first node in the linked list. When we want to traverse the list, we start from the head and follow the next pointers of each Node until you reach the end of the list 

```c
typedef struct LinkedList {
    Node* head;  // Pointer to the first node
} LinkedList;
```
Let's visualize this:

**`LinkedList`** where we've added two pieces of data: **`"Hello, World!"`** and **`"New Beginning!"`** using the **`'add'`** and **`insertAtBeginning`** functions. At this visualization, the **`LinkedList`** has a **`head`** pointer that points to the first Node in the list. Each Node contains a wide character string (wchar_t*) data and a next pointer to the next node in the list.

```
LinkedList
   |
   v
  head
   |
   v
  Node ("New Beginning!") --> Node ("Hello, World!") --> NULL
```

This function, **`createNode`**, is responsible for creating a new node in a linked list. It takes a wide character string **`(const wchar_t* data)`** as input, which will be the data stored in the node.

```c
// Function to create a new node with given data
Node* createNode(const wchar_t* data) {
    // Allocate memory for new node
    Node* node = (Node*)malloc(sizeof(Node));
    if (node == NULL) {
        fwprintf(stderr, L"Failed to allocate memory for new node\n");
        exit(EXIT_FAILURE);
    }

    // Duplicate the given data for this node
    node->data = _wcsdup(data);
    if (node->data == NULL) {
        fwprintf(stderr, L"Failed to allocate memory for node data\n");
        exit(EXIT_FAILURE);
    }

    // Set the next node to NULL (end of list)
    node->next = NULL;
    return node;
}
```
Let's break down this function in 3 different pieces, so we understand it better.

This piece of code is the initial part of the **`createNode`** function which is responsible for creating a new node in a linked list. It declares a function named **`createNode`** that takes a constant wide character string (pointer to wchar_t) as **input** and returns a **pointer** to a Node. 

It allocates memory for a new Node using the **malloc** function. The size of the memory to be allocated is the size of a Node structure, which is determined by **`sizeof(Node)`**.

```c
// Function to create a new node with given data
Node* createNode(const wchar_t* data) {
    // Allocate memory for new node
    Node* node = (Node*)malloc(sizeof(Node));
    if (node == NULL) {
        fwprintf(stderr, L"Failed to allocate memory for new node\n");
        exit(EXIT_FAILURE);
    }
```

Let's read this sentence once again:

*It declares a function named **`createNode`** that takes a constant **wide character string** (pointer to wchar_t) as **input** and returns a **pointer** to a Node.*

Let's say we're calling **`createNode(L"Hello, World!")`**.

Before memory allocation:

```
createNode(L"Hello, World!")
  |
  v
data: "Hello, World!"
```

After memory allocation:

```
createNode(L"Hello, World!")
  |
  v
data: "Hello, World!"
  |
  v
malloc(sizeof(Node))
  |
  v
node: empty Node
```

If the **malloc** function fails to allocate memory, **node** will be NULL, and the program will print an error message and exit. If it succeeds, we have a new, empty Node ready to be filled with data, which happens to be part of the **`createNode`** function.

Let's now move to the second piece:

This piece of code is duplicating the input string and assigning it to the **`data`** field of the created Node. The function **`_wcsdup(data)`** duplicates the **wide character string** data by allocating memory for a new string of the same size, copying the contents of data into the new memory, and returning a pointer to the new string. 

We duplicate the strings to ensure that each node in the linked list has its own independent copy of the data, preventing potential issues with data being changed or deleted elsewhere in the program.

```c
    // Duplicate the given data for this node
    node->data = _wcsdup(data);
    if (node->data == NULL) {
        fwprintf(stderr, L"Failed to allocate memory for node data\n");
        exit(EXIT_FAILURE);
    }
```
Before duplication:

```
Node* node = (Node*)malloc(sizeof(Node));
  |
  v
[ Uninitialized Node ]
  |
  v
data: "Hello, World!"
```

After duplication:

```
node->data = _wcsdup(data);
  |
  v
[ Node with data pointing to "Hello, World!" ]
```

When we call the **`createNode`** function with a wide string literal like **`createNode(L"Hello, World!")`**, **`data`** points to the string **`"Hello, World!"`**. Inside the **`createNode`** function, when the line **`node->data = _wcsdup(data)`**; is executed, a new string is created that duplicates **`"Hello, World!"`**, and **`node->data`** becomes a **pointer** to this created duplicate string.

This part of the code is setting up the next pointer of the created node to **NULL**, indicating that this node is currently the last one in the list since there's no next node to point to. The function then returns the pointer to the created node.

```c
    // Set the next node to NULL (end of list)
    node->next = NULL;
    return node;
```

Before the assignment:

```
[ Node with data pointing to "Hello, World!" and uninitialized next pointer ]
```

After the assignment:

```
[ Node with data pointing to "Hello, World!" and next pointing to NULL ]
```


This part of code is a function named **`add`**. Its job is to create a new node and add it to a linked list. Let's look at the step-by-step:

```c
// Function to add a new node with given data to the end of the list
void add(LinkedList* list, const wchar_t* data) {
    Node* node = createNode(data);
    if (list->head == NULL) {
        // If the list is empty, the new node is the head
        list->head = node;
    }
    else {
        // Find the last node
        Node* current = list->head;
        while (current->next != NULL) {
            current = current->next;
        }
        // Attach the new node at the end
        current->next = node;
    }
}
```

This part of the code is adding a new node to the start of the list if the list is currently empty.

**`Node* node = createNode(data);`** is calling the function **`createNode`**, which creates a new node with the data passed into it (in this case **data**). The new node is stored in the **`node`** variable.

**`if (list->head == NULL)`** is checking if the linked list is currently empty. It does this by looking at the head of the list. If **`list->head`** is **NULL**, it means the list has no nodes yet, so it is empty.

**`list->head = node;`** If the list is empty, this line is setting the new node as the head of the list. This is adding the new node to the list. In a linked list, the head is the first node.

```c
// Function to add a new node with given data to the end of the list
void add(LinkedList* list, const wchar_t* data) {
    Node* node = createNode(data);
    if (list->head == NULL) {
        // If the list is empty, the new node is the head
        list->head = node;
    }
```

Before adding a node, the list is empty:

```
head -> NULL
```

We call **`add(&list, L"Hello, World!")`**, and after that, the list has one node:

```
head -> [ "Hello, World!" | NULL ]
```

This part of the code is handling the case where the linked list is not empty. Its job is to add a new node to the end of an existing list.

**`Node* current = list->head;`** It starts with current pointing to the head of the list.

**`while (current->next != NULL)`** This loop keeps going as long as there's another node after the current one.

**`current = current->next;`** This line moves current one step forward in the list.

**`current->next = node;`** This line makes the new node the one after the current one. In other words, it adds the new node to the end of the list.

```c
    else {
        // Find the last node
        Node* current = list->head;
        while (current->next != NULL) {
            current = current->next;
        }
        // Attach the new node at the end
        current->next = node;
    }
}
```

Let's say we already have a list with **"`Hello, World!`"** and we're calling **`add(&list, L"New Beginning!")`**.

Before calling **`add`**, the list looks like this:

```
head
 |
 v
[ Node with data "Hello, World!" and next pointing to NULL ]
```

After calling **`add(&list, L"New Beginning!")`**, the list looks like this:

```
head
 |
 v
[ Node with data "Hello, World!" ] --> [ Node with data "New Beginning!" and next pointing to NULL ]
```

This function, **`insertAtBeginning`**, is responsible for creating a new node with the given data and adding it to the beginning of the linked list, making it the new head of the list.

**`Node* node = createNode(data);`** creates a new node with the provided data using the **`createNode`** function. The **`createNode`** function allocates memory for the new node, duplicates the input string, and assigns it to the **`data`** field of the node.

**`node->next = list->head;`** The **next** pointer of the new node is set to the current head of the list. This means the new node will point to the node that was previously the first node in the list.

**`list->head = node;`** The head of the list is updated to be the new node, making it the first node in the list

```c
// Function to insert a new node with given data at the beginning of the list
void insertAtBeginning(LinkedList* list, const wchar_t* data) {
    Node* node = createNode(data);
    // The new node points to what was previously the first node
    node->next = list->head;
    // The new node becomes the list head
    list->head = node;
}
```
Let's say we're calling **`insertAtBeginning(&list, L"New Beginning!")`**, and the current state of the list is:

```
head
 |
 v
[ Node with data "Hello, World!" and next pointing to NULL ]
```

The new node with the data **`"New Beginning!"`** has now been inserted at the beginning of the list, making it the new head of the list. The node with **`"Hello, World!"`** is now the second node in the list.

```
head
 |
 v
[ Node with data "New Beginning!" ] --> [ Node with data "Hello, World!" and next pointing to NULL ]
```

This function, **`freeNode`**, is responsible for freeing the memory that was previously allocated for a node in the linked list. When we dynamically allocate memory like calling **malloc** or **_wcsdup**, it's important to manually free that memory when we're done with it to prevent memory leaks. 

```c
// Function to free the memory of a linked list
void freeList(LinkedList* list) {
    Node* current = list->head;
    while (current != NULL) {
        Node* nextNode = current->next;
        freeNode(current);
        current = nextNode;
    }
}
```

Before calling **`freeNode(node)`**, we have a node:

```
[ Node with data "Hello, World!" and next pointing to NULL ]
```

After calling **`freeNode(node)`**, the memory previously occupied by the node and its data has been freed:

```
[ Freed memory ]
```

This function **`freeList`** is used to free up the memory that has been allocated for the linked list. It iterates through the entire list, and for each node, it frees the memory using the freeNode function. The **`freeNode`** function frees the memory of the node's data and the node itself.

```c
// Function to free the memory of a linked list
void freeList(LinkedList* list) {
    Node* current = list->head;
    while (current != NULL) {
        Node* nextNode = current->next;
        freeNode(current);
        current = nextNode;
    }
}
```

Let's say we're calling **`freeList(&list)`** and the current state of the list is:

```
head
 |
 v
[ Node with data "New Beginning!" ] --> [ Node with data "Hello, World!" and next pointing to NULL ]
```

After **`freeList(&list)`** is called, the state of the list would be:

```
head
 |
 v
NULL
```











