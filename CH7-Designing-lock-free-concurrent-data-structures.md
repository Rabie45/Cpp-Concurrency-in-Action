# CH7 Designing lock-free concurrent data structures
## 7.1 Definitions and consequences
- Algorithms and data structures that use mutexes, condition variables, and futures to synchronize the data are called blocking data structures and algorithms.
### 7.1.1 Types of nonblocking data structures
```
class spinlock_mutex
{
std::atomic_flag flag;
public:
spinlock_mutex():
flag(ATOMIC_FLAG_INIT)
{}
void lock()
{
while(flag.test_and_set(std::memory_order_acquire));
}
void unlock()
{
flag.clear(std::memory_order_release);
}
};
```
- This code doesn’t call any blocking functions; lock() just keeps looping until the call to test_and_set() returns false .
### 7.1.2 Lock-free data structures
- a lock-free queue might allow one thread to push and one to pop but
break if two threads try to push new items at the same time.
- The reason for using a compare/exchange operation is that
another thread might have modified the data in the meantime, in which case the code will need to redo part of its operation before trying the compare/exchange again.
- Lock-free algorithms with such loops can result in one thread being subject to starvation. If another thread performs operations with the “wrong” timing, the other thread might make progress while the first thread continually has to retry its operation
### 7.1.3 Wait-free data structures
- A wait-free data structure is a lock-free data structure with the additional property that every thread accessing the data structure can complete its operation within a bounded number of steps, regardless of the behavior of other threads.
- Writing wait-free data structures correctly is extremely hard. In order to ensure that every thread can complete its operations within a bounded number of steps, you have to ensure that each operation can be performed in a single pass and that the steps performed by one thread don’t cause an operation on another thread to fail.

### 7.1.4 The pros and cons of lock-free data structures
- the primary reason for using lock-free data structures is to
enable maximum concurrency. With lock-based containers, there’s always the potential for one thread to have to block and wait for another to complete its operation before the first thread can proceed
- A second reason to use lock-free data structures is robustness. If a thread dies while holding a lock, that data structure is broken forever.
- To avoid the undefined behavior associated with a data race, you must use atomic operations for the modifications. But that alone
isn’t enough;
## Examples of lock-free data structures
## 7.2.1 Writing a thread-safe stack without locks
- The basic premise of a stack is relatively simple: nodes are retrieved in the reverse order to which they were added—last in, first out ( LIFO )
- t’s therefore important to ensure that once a value is added to the stack, it can safely be retrieved immediately by another thread
 - Under such a scheme, adding a node is relatively simple:
   - Create a new node.
   - Set its next pointer to the current head node.
   - Set the head node to point to it.
- but if other threads are also modifying
the stack, it’s not enough. Crucially, if two threads are adding nodes, there’s a race
condition between steps 2 and 3: a second thread could modify the value of head
between when your thread reads it in step 2 and you update it in step 3.
 - now that you have a means of adding data to the stack, you need a way
of getting it off again. On the face of it, this is quite simple:
   - Read the current value of head .
   - Read head->next .
   - Set head to head->next .
   - Return the data from the retrieved node .
   - Delete the retrieved node.
- If you return a smart pointer, you can just return nullptr to indicate that there’s no value to return, but this requires that the data be allocated on the heap.

### 7.2.2 Stopping those pesky leaks: managing memory in lock-free data structures
- if you need to handle multiple threads calling pop() on the same
stack instance, you need some way to track when it’s safe to delete a node. This essentially means you need to write a special-purpose garbage collector just for nodes.
   
### 7.2.3 Detecting nodes that can’t be reclaimed using hazard pointers
- hazard pointers is a reference to a technique discovered by Maged Michael. They are so called because deleting a node that might still be referenced by other threads is hazardous.
- If other threads do indeed hold references to that node and pro-
ceed to access the node through that reference, you have undefined behavior.
- When a thread wishes to delete an object, it must first check the hazard pointers belonging to the other threads in the system. If none of the hazard pointers reference
the object, it can safely be deleted.
- get_hazard_pointer_for_current_thread() that returns
a reference to your hazard pointer. You then need to set it when you read a pointer that you intend to dereference—in this case the head value from the list


### 7.2.5 Applying the memory model to the lock-free stack
- Before you go about changing the memory orderings, you need to examine the operations and identify the required relationships between them. You can then go back and find the minimum memory orderings that provide these required relationships.


### 7.2.6 Writing a thread-safe queue without locks
- A queue offers a slightly different challenge to a stack, because the push() and pop() operations access different parts of the data structure in a queue, whereas they both access the same head node for a stack.
```
#include <iostream>
#include <thread>
#include <vector>
#include <atomic>
#include <memory>

// Define the lock_free_queue class template
template<typename T>
class lock_free_queue {
private:
    struct node {
        std::shared_ptr<T> data;
        node* next;
        node() : next(nullptr) {}
    };

    std::atomic<node*> head;
    std::atomic<node*> tail;

    node* pop_head() {
        node* const old_head = head.load();
        if (old_head == tail.load()) {
            return nullptr;
        }
        head.store(old_head->next);
        return old_head;
    }

public:
    lock_free_queue() : head(new node), tail(head.load()) {}

    lock_free_queue(const lock_free_queue& other) = delete;
    lock_free_queue& operator=(const lock_free_queue& other) = delete;

    ~lock_free_queue() {
        while (node* const old_head = head.load()) {
            head.store(old_head->next);
            delete old_head;
        }
    }

    std::shared_ptr<T> pop() {
        node* old_head = pop_head();
        if (!old_head) {
            return std::shared_ptr<T>();
        }
        std::shared_ptr<T> const res(old_head->data);
        delete old_head;
        return res;
    }

    void push(T new_value) {
        std::shared_ptr<T> new_data(std::make_shared<T>(new_value));
        node* p = new node;
        node* const old_tail = tail.load();
        old_tail->data.swap(new_data);
        old_tail->next = p;
        tail.store(p);
    }
};

int main() {
    lock_free_queue<int> q;

    // Push elements into the queue
    q.push(1);
    q.push(2);
    q.push(3);

    // Pop and print elements from the queue
    std::shared_ptr<int> value;
    while ((value = q.pop())) {
        std::cout << *value << std::endl;
    }

    return 0;
}

```
![Screenshot from 2024-08-06 10-45-33](https://github.com/user-attachments/assets/e805d67b-19e0-4fb2-bdd9-a50b5b72befb)

- Each node in the queue holds a std::shared_ptr<T> to the data and a pointer to the next node. The constructor initializes the next pointer to nullptr.
- These atomic pointers represent the head and tail of the queue. Using std::atomic ensures that operations on these pointers are thread-safe.
- The constructor initializes the queue with a dummy node, setting both head and tail to point to this node. The destructor iterates through the queue, deleting each node to free memory.
- Deleted Copy Constructor and Assignment Operator  Deleted Copy Constructor and Assignment Operator .These are deleted to prevent copying the queue, which could lead to issues in a lock-free context.
- pop_head Function: This private function atomically loads the current head, checks if the queue is empty, updates the head to the next node, and returns the old head.
- pop Method:
This method calls pop_head to get the old head node. If the queue was empty, it returns an empty shared_ptr. Otherwise, it extracts the data, deletes the old node, and returns the data.
- push Method:
This method creates a new node and atomically updates the tail to point to the new node. It swaps the new data into the old tail's data pointer and updates the next pointer of the old tail to the new node.
