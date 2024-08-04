# CH6 Designing lock-based concurrent data structures

## 6.1 What does it mean to design for concurrency?
- At the basic level, designing a data structure for concurrency means that multiple threads can access the data structure concurrently
- a data structure will be safe only for particular types of concurrent access. It may be possible to have multiple threads performing one type of operation on the data structure concurrently
- designing for concurrency means providing the opportunity for concurrency to threads accessing the data structure. By its very nature, a mutex provides mutual exclusion: only one thread can acquire a lock on the
mutex at a time. A mutex protects a data structure by explicitly preventing true concurrent access to the data it protects.
- This is called serialization: threads take turns accessing the data protected by the mutex; they must access it serially rather than concurrently.
## 6.1.1 Guidelines for designing data structures for concurrency
- you have two aspects to consider when designing data structures
for concurrent access: ensuring that the accesses are safe and enabling genuine concurrent access.
- To make the data structure thread safe:
  - Ensure that no thread can see a state where the invariants of the data structure have been broken by the actions of another thread.
  - Take care to avoid race conditions inherent in the interface to the data structure by providing functions for complete operations rather than for operation steps.
  - Pay attention to how the data structure behaves in the presence of exceptions to ensure that the invariants are not broken.
  - Minimize the opportunities for deadlock when using the data structure by restricting the scope of locks and avoiding nested locks where possible. 

- how can you minimize the amount of serialization that must occur and enable the greatest amount of true concurrency?
- It’s not uncommon for data structures to allow concurrent access from multiple threads that merely read the data structure, whereas a thread that can modify the data structure must have exclusive access. This is supported by using constructs like boost:: shared_mutex .
- The simplest thread-safe data structures typically use mutexes and locks to protect the data. 
## 6.2 Lock-based concurrent data structures
### 6.2.1 A thread-safe stack using lock
- This defines a custom exception empty_stack that inherits from std::exception. The what method returns an error message when the exception is thrown.
- This starts the definition of a template class threadsafe_stack which can handle any type T.
- These are the private members of the class:

    data: The stack that holds the elements.
    m: A mutable mutex to protect access to the stack.
- The default constructor initializes an empty threadsafe_stack.
- The copy constructor creates a new stack by copying an existing one. It locks the mutex of the other stack to ensure thread safety during the copy.
- The copy assignment operator is deleted to prevent assignment of threadsafe_stack objects.
- The push method adds a new element to the stack. It locks the mutex to ensure thread safety and uses std::move to efficiently transfer the value.
- The first pop method removes the top element from the stack and returns it as a shared_ptr. It locks the mutex, checks if the stack is empty, throws an empty_stack exception if it is, otherwise returns the top element.
- The second pop method removes the top element and assigns it to a reference passed by the caller. It locks the mutex, checks if the stack is empty, throws an empty_stack exception if it is, otherwise assigns the top element to the reference.
- The empty method checks if the stack is empty. It locks the mutex to ensure thread safety and returns whether the stack is empty.
```
#include <iostream>
#include <stack>
#include <mutex>
#include <memory>
#include <exception>

// Define the empty_stack exception
struct empty_stack: std::exception {
    const char* what() const throw() {
        return "Stack is empty";
    }
};

// Define the threadsafe_stack template class
template<typename T>
class threadsafe_stack {
private:
    std::stack<T> data;
    mutable std::mutex m;

public:
    threadsafe_stack() {}

    // Copy constructor
    threadsafe_stack(const threadsafe_stack& other) {
        std::lock_guard<std::mutex> lock(other.m);
        data = other.data;
    }

    // Delete copy assignment operator
    threadsafe_stack& operator=(const threadsafe_stack&) = delete;

    // Push method to add an element to the stack
    void push(T new_value) {
        std::lock_guard<std::mutex> lock(m);
        data.push(std::move(new_value));
    }

    // Pop method that returns a shared pointer to the top element
    std::shared_ptr<T> pop() {
        std::lock_guard<std::mutex> lock(m);
        if(data.empty()) throw empty_stack();
        std::shared_ptr<T> const res(std::make_shared<T>(std::move(data.top())));
        data.pop();
        return res;
    }

    // Pop method that returns the top element by reference
    void pop(T& value) {
        std::lock_guard<std::mutex> lock(m);
        if(data.empty()) throw empty_stack();
        value = std::move(data.top());
        data.pop();
    }

    // Method to check if the stack is empty
    bool empty() const {
        std::lock_guard<std::mutex> lock(m);
        return data.empty();
    }
};

// Main function to demonstrate the usage of threadsafe_stack
int main() {
    threadsafe_stack<int> stack;

    // Push some elements onto the stack
    stack.push(10);
    stack.push(20);
    stack.push(30);

    try {
        // Pop elements from the stack
        std::shared_ptr<int> value = stack.pop();
        std::cout << "Popped value: " << *value << std::endl;

        int value2;
        stack.pop(value2);
        std::cout << "Popped value: " << value2 << std::endl;

        // Check if the stack is empty
        if(stack.empty()) {
            std::cout << "Stack is empty" << std::endl;
        } else {
            std::cout << "Stack is not empty" << std::endl;
        }

        // Pop another element
        value = stack.pop();
        std::cout << "Popped value: " << *value << std::endl;

        // Try popping from an empty stack
        value = stack.pop(); // This will throw an exception
    } catch(const empty_stack& e) {
        std::cerr << "Exception: " << e.what() << std::endl;
    }

    return 0;
}
```
![Screenshot from 2024-08-04 11-22-02](https://github.com/user-attachments/assets/948e9fda-422b-4225-a5d0-b9beb86c20c9)

### 6.2.2 A thread-safe queue using locks and condition variables
These are the private members of the class:

    mut: A mutable mutex to protect access to the queue.
    data_queue: The queue that holds the elements.
    data_cond: A condition variable to manage synchronization.
```
#include <iostream>
#include <queue>
#include <memory>
#include <mutex>
#include <condition_variable>
template<typename T>
class threadsafe_queue {
private:
    mutable std::mutex mut;
    std::queue<T> data_queue;
    std::condition_variable data_cond;
public:
    threadsafe_queue() {}
    void push(T new_value) {
        std::lock_guard<std::mutex> lk(mut);
        data_queue.push(std::move(new_value));
        data_cond.notify_one();
    }
    void wait_and_pop(T& value) {
        std::unique_lock<std::mutex> lk(mut);
        data_cond.wait(lk, [this]{return !data_queue.empty();});
        value = std::move(data_queue.front());
        data_queue.pop();
    }
    std::shared_ptr<T> wait_and_pop() {
        std::unique_lock<std::mutex> lk(mut);
        data_cond.wait(lk, [this]{return !data_queue.empty();});
        std::shared_ptr<T> res(std::make_shared<T>(std::move(data_queue.front())));
        data_queue.pop();
        return res;
    }
    bool try_pop(T& value) {
        std::lock_guard<std::mutex> lk(mut);
        if (data_queue.empty())
            return false;
        value = std::move(data_queue.front());
        data_queue.pop();
        return true;
    }
    std::shared_ptr<T> try_pop() {
        std::lock_guard<std::mutex> lk(mut);
        if (data_queue.empty())
            return std::shared_ptr<T>();
        std::shared_ptr<T> res(std::make_shared<T>(std::move(data_queue.front())));
        data_queue.pop();
        return res;
    }
    bool empty() const {
        std::lock_guard<std::mutex> lk(mut);
        return data_queue.empty();
    }
};

int main() {
    threadsafe_queue<int> queue;

    // Push some elements onto the queue
    queue.push(10);
    queue.push(20);
    queue.push(30);

    try {
        // Wait and pop elements from the queue
        int value1;
        queue.wait_and_pop(value1);
        std::cout << "Popped value: " << value1 << std::endl;

        auto value2 = queue.wait_and_pop();
        std::cout << "Popped value: " << *value2 << std::endl;

        // Try popping elements from the queue
        int value3;
        if (queue.try_pop(value3)) {
            std::cout << "Popped value: " << value3 << std::endl;
        } else {
            std::cout << "Queue is empty" << std::endl;
        }

        auto value4 = queue.try_pop();
        if (value4) {
            std::cout << "Popped value: " << *value4 << std::endl;
        } else {
            std::cout << "Queue is empty" << std::endl;
        }

        // Check if the queue is empty
        if (queue.empty()) {
            std::cout << "Queue is empty" << std::endl;
        } else {
            std::cout << "Queue is not empty" << std::endl;
        }
    } catch (const std::exception& e) {
        std::cerr << "Exception: " << e.what() << std::endl;
    }

    return 0;
}
```

- This serialization of threads can potentially limit the per-
formance of an application where there’s significant contention on the stack : while a thread is waiting for the lock.
- The default constructor initializes an empty threadsafe_queue.
- The push method adds a new element to the queue. It locks the mutex to ensure thread safety, uses std::move to efficiently transfer the value, and notifies one waiting thread that new data is available.
- The wait_and_pop method waits until an element is available, then pops it. It locks the mutex, waits on the condition variable until the queue is not empty, assigns the front element to the reference, and removes the element from the queue.
- The second wait_and_pop method does the same as the first, but returns a shared_ptr to the popped element instead of assigning it to a reference.
- The try_pop method tries to pop an element without waiting. It locks the mutex, checks if the queue is empty, and if not, assigns the front element to the reference and removes it from the queue. It returns true if successful, false otherwise.
- The second try_pop method does the same as the first, but returns a shared_ptr to the popped element instead of assigning it to a reference.- The empty method checks if the queue is empty. It locks the mutex to ensure thread safety and returns whether the queue is empty.
  ![Screenshot from 2024-08-04 11-22-09](https://github.com/user-attachments/assets/36d09875-51f4-42e0-af0a-76504ae2e596)
  
### 6.2.3 A thread-safe queue using fine-grained locks and condition variables
- The simplest data structure for a queue is a singly linked list, as shown in figure 6.1.
- The queue contains a head pointer, which points to the first item in the list, and each item then points to the next item. Data items are removed from the queue by replacing the head pointer with the pointer to the next item and then returning the data from the old head.
- Items are added to the queue at the other end. In order to do this, the queue also contains a tail pointer, which refers to the last item in the list. New nodes are added by changing the next pointer of the last item to point to the new node and then updating the tail pointer to refer to the new item. When the list is empty, both the head and tail
pointers are NULL .

