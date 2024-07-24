# Ch 3 Sharing data between threads
- benefits of using threads for concurrency is the potential to easily
and directly share data between them
- It’s the same with threads. If you’re sharing data between threads, you need to
have rules for which thread can access which bit of data
## Problems with sharing data between threads
- the problems with sharing data between threads are all due to the consequences of modifying data.
- if all shared data is read-only, there’s no problem,because the data read by one thread is unaffected
- if one thread is reading the doubly linked list while another is removing a node, it’s quite possible for the reading thread to see the list with a node only partially removed
- On the other hand, if the second thread is trying to delete the rightmost node in the diagram, it might end up permanently corrupting the data structure and eventually crashing the program this is called a race condition
### 3.1.1 Race conditions
- In concurrency, a race condition is anything where the outcome depends on the relative ordering of execution of operations on two or more threads
- data races cause the dreaded undefined behavior.
- Problematic race conditions typically occur where completing an operation requires modification of two or more distinct pieces of data

### Avoiding problematic race conditions
- The simplest option is
to wrap your data structure with a protection mechanism, to ensure that only the thread
actually performing a modification
- Another option is to modify the design of your data structure and its invariants so
that modifications are done as a series of indivisible changes
- This is generally referred to as lock-free programming and is difficult to get
right
- Another way of dealing with race conditions is to handle the updates to the data
structure as a transaction, just as updates to a database are done within a transaction.
- If the commit can’t proceed because the data struc-
ture has been modified by another thread, the transaction is restarted. This is termed software transactional memory ( STM )
- The most basic mechanism for protecting shared data provided by the C++ Stan-
dard is the mutex,

## 3.2 Protecting shared data with mutexes
- you want to protect shared data structure from race conditions and the potential broken invariants
that can ensue.
- Wouldn’t it be nice if you could mark all the pieces of code that access
the data structure as mutually exclusive. f any thread was running one of them,
any other thread that tried to access that data structure had to wait until the first
thread was finished
- Before accessing a shared data struc-
ture, you lock the mutex associated with that data, and when you’ve finished accessing
the data structure, you unlock the mutex.
- Mutexes are the most general of the data-protection mechanisms available in C++

### Using mutexes in C++
- In C++, you create a mutex by constructing an instance of std::mutex , lock it with a
call to the member function lock() , and unlock it with a call to the member func-
tion unlock() .
- e Standard
C++ Library provides the std::lock_guard class template, which implements that
RAII idiom for a mutex; it locks the supplied mutex on construction and unlocks it
on destruction, thus ensuring a locked mutex is always correctly unlocked.
- C++ Library provides the std::lock_guard class template, it locks the supplied mutex on construction and unlocks it on destruction,
```
#include <iostream>
#include <list>
#include <mutex>
#include <algorithm>
#include <thread>

// Global list and mutex
std::list<int> some_list; //single global variable
std::mutex some_mutex; // protected with a corresponding global instance of std::mutex

// Function to add a value to the list
void add_to_list(int new_value)
{
    std::lock_guard<std::mutex> guard(some_mutex); // means that the accesses in these functions are mutually exclusive
    some_list.push_back(new_value);
}

// Function to check if a value exists in the list
bool list_contains(int value_to_find)
{
    std::lock_guard<std::mutex> guard(some_mutex); // means that the accesses in these functions are mutually exclusive
    return std::find(some_list.begin(), some_list.end(), value_to_find) != some_list.end();
}

// Function to run in a thread, adding values to the list
void add_values_to_list()
{
    for (int i = 0; i < 10; ++i)
    {
        add_to_list(i);
    }
}

// Function to run in a thread, checking if values exist in the list
void check_values_in_list()
{
    for (int i = 0; i < 10; ++i)
    {
        if (list_contains(i))
        {
            std::cout << "List contains " << i << std::endl;
        }
        else
        {
            std::cout << "List does not contain " << i << std::endl;
        }
    }
}

int main()
{
    // Create threads to add values and check values in the list
    std::thread add_thread(add_values_to_list);
    std::thread check_thread(check_values_in_list);

    // Wait for threads to complete
    add_thread.detach();
    check_thread.join();

    return 0;
}
```
![Screenshot from 2024-07-23 10-26-02](https://github.com/user-attachments/assets/908353bf-ed61-41c7-acd5-2034f268cc08)

- be carfull if u make the detach with check thread function it will check element without add thread add any thing then it causes segmentation fault error 
- If all the member functions of the class lock the mutex before accessing any other data members and unlock it when done, the data is nicely protected from all comers.
  
![Screenshot from 2024-07-23 10-27-20](https://github.com/user-attachments/assets/38a5d8e8-3546-4cec-ae36-139f4d9b0a8f)

### Structuring code for protecting shared data
- it’s also important to check that they don’t pass such pointers or references in to func-
tions they call that aren’t under your control.
```

#include <iostream>
#include <string>
#include <mutex>

// Define some_data class
class some_data {
public:
    int a;
    std::string b;

    void do_something() {
        std::cout << "Doing something with data: " << a << ", " << b << std::endl;
    }
};

// Define data_wrapper class
class data_wrapper {
private:
    some_data data;
    std::mutex m;

public:
    template<typename Function>
    void process_data(Function func) {
        std::lock_guard<std::mutex> l(m);
        func(data);
    }
};

// Global pointer to unprotected some_data
some_data* unprotected;

// Define a malicious function
void malicious_function(some_data& protected_data) {
    unprotected = &protected_data;
}

// Main function to demonstrate the concept
int main() {
    // Create a data_wrapper object
    data_wrapper x;

    // Process the data with the malicious function
    x.process_data(malicious_function);

    // Now unprotected points to the protected_data inside data_wrapper
    if (unprotected) {
        unprotected->a = 42;
        unprotected->b = "Hacked!";
        unprotected->do_something();
    } else {
        std::cerr << "unprotected is nullptr" << std::endl;
    }

    return 0;
}

```
![Screenshot from 2024-07-23 10-40-41](https://github.com/user-attachments/assets/45124da4-5244-4969-b410-df632d35aaf5)

- the code in process_data looks harmless enough, nicely protected with std::lock_guard , but the call to the user-supplied function func B means that foo can pass in malicious_function to bypass the protection c and then call do_something() without the mutex being locked .
- Don’t pass pointers and references to protected data outside the scope of the lock, whether by
returning them from a function, storing them in externally visible memory, or passing them as
arguments to user-supplied functions.

### Spotting race conditions inherent in interfaces
- because you’re using a mutex or other mechanism to protect shared data, you’re not necessarily protected from race conditions
### OPTION 1: P ASS IN A REFERENCE
- The first option is to pass a reference to a variable in which you wish to receive the popped value as an argument in the call to pop() :
```
std::vector<int> result;
some_stack.pop(result);
```
### OPTION 2: REQUIRE A NO - THROW COPY CONSTRUCTOR OR MOVE CONSTRUCTOR
- There’s only an exception safety problem with a value-returning pop() if the return by value can throw an exception.

### OPTION 3: R ETURN A POINTER TO THE POPPED ITEM
- The third option is to return a pointer to the popped item rather than return the item
by value. The advantage here is that pointers can be freely copied without throwing an
exception, so you’ve avoided Cargill’s exception problem.
- The disadvantage is that
returning a pointer requires a means of managing the memory allocated to the
object, and for simple types such as integers

### OPTION 4: P ROVIDE BOTH OPTION 1 AND EITHER OPTION 2 OR 3
- Flexibility should never be ruled out, especially in generic code. If you’ve chosen
option 2 or 3, it’s relatively easy to provide option 1 as well, and this provides users of
your code the ability to choose whichever option is most appropriate for them for very little additional cost.
- EXAMPLE DEFINITION OF A THREAD - SAFE STACK
```
#include <exception>
#include <memory>
#include <stack>
#include <mutex>
#include <iostream>

// Define the custom exception for an empty stack
struct empty_stack : std::exception {
    const char* what() const noexcept override {
        return "Stack is empty";
    }
};

// Define the thread-safe stack template class
template<typename T>
class threadsafe_stack {
private:
    std::stack<T> data;
    mutable std::mutex m;

public:
    // Constructor
    threadsafe_stack() = default;

    // Copy constructor
    threadsafe_stack(const threadsafe_stack& other) {
        std::lock_guard<std::mutex> lock(other.m);
        data = other.data;
    }

    // Delete copy assignment operator
    threadsafe_stack& operator=(const threadsafe_stack&) = delete;

    // Push method
    void push(T new_value) {
        std::lock_guard<std::mutex> lock(m);
        data.push(std::move(new_value));
    }

    // Pop method returning std::shared_ptr<T>
    std::shared_ptr<T> pop() {
        std::lock_guard<std::mutex> lock(m);
        if (data.empty()) {
            throw empty_stack();
        }
        std::shared_ptr<T> const res(std::make_shared<T>(std::move(data.top())));
        data.pop();
        return res;
    }

    // Pop method with reference parameter
    void pop(T& value) {
        std::lock_guard<std::mutex> lock(m);
        if (data.empty()) {
            throw empty_stack();
        }
        value = std::move(data.top());
        data.pop();
    }

    // Empty method
    bool empty() const {
        std::lock_guard<std::mutex> lock(m);
        return data.empty();
    }
};

int main() {
    // Create a thread-safe stack
    threadsafe_stack<int> stack;

    // Push elements onto the stack
    stack.push(1);
    stack.push(2);
    stack.push(3);

    try {
        // Pop elements and print them
        std::shared_ptr<int> p1 = stack.pop();
        std::cout << "Popped: " << *p1 << std::endl;

        int value;
        stack.pop(value);
        std::cout << "Popped: " << value << std::endl;

        std::shared_ptr<int> p2 = stack.pop();
        std::cout << "Popped: " << *p2 << std::endl;

        // Attempt to pop from an empty stack to trigger the exception
        stack.pop();
    } catch (const empty_stack& e) {
        std::cerr << e.what() << std::endl;
    }

    return 0;
}

```
![Screenshot from 2024-07-23 12-55-34](https://github.com/user-attachments/assets/3b4c06f5-d1fe-49d3-8a53-835b5ff95710)

- This simplification of the interface allows for better control over the data; you can ensure that the mutex is locked for the entirety of an operation. The following listing shows a simple implementation that’s a wrapper around std::stack<> .
- The first versions of the Linux kernel that were designed to handle multi-
processor systems used a single global kernel lock. Although this worked, it meant that a two-processor system typically had much worse performance than two single-
processor systems, and performance on a four-processor system was nowhere near
that of four single-processor systems.
- One issue with fine-grained locking schemes is that sometimes you need more
than one mutex locked in order to protect all the data in an operation
- f you end up having to lock two or more mutexes for a given operation, there’s
another potential problem lurking in the wings: deadlock. This is almost the opposite
of a race condition: rather than two threads racing to be first, each one is waiting for
the other, so neither makes any progress.

### Deadlock: the problem and a solution
- threads arguing
over locks on mutexes: each of a pair of threads needs to lock both of a pair of
mutexes to perform some operation, and each thread has one mutex and is waiting
for the other. 
- Neither thread can proceed, because each is waiting for the other to
release its mutex. This scenario is called deadlock, and it’s the biggest problem with
having to lock two or more mutexes in order to perform an operation.
- always lock the two mutexes in the
same order: if you always lock mutex A before mutex B, then you’ll never deadlock.

### Further guidelines for avoiding deadlock
- you can create deadlock with two threads and no locks just by having each thread call join() on the std::thread object for the other.
```
#include <functional>
#include <iostream>
#include <thread>

void thread_fun(std::thread &thread) {
  std::cout << "Thread started and waiting for the other thread to join.\n";
  thread.join();
  std::cout << "This line will never be reached due to deadlock.\n";
}
void hello() {
  std::cout << "Hello Concurrent World\n";
}
int main() {
  std::thread t2(hello);
  std::thread t1(thread_fun, std::ref(t2));
  t2 = std::thread(thread_fun, std::ref(t1));

  t1.join();
  t2.join();
}
```
![Screenshot from 2024-07-23 13-18-07](https://github.com/user-attachments/assets/68767c43-a0a1-417f-b890-5379a2b944b9)


## ACQUIRE LOCKS IN A FIXED ORDER
- This hand-over-hand locking style allows multiple threads to access the list, pro-
vided each is accessing a different node. However, in order to prevent deadlock, the
nodes must always be locked in the same order: if two threads tried to traverse the list
in reverse order using hand-over-hand locking.
- One way to prevent deadlock here is to define an order of traversal, so a thread
must always lock A before B and B before C. This would eliminate the possibility of
deadlock at the expense of disallowing reverse traversal.
## USE A LOCK HIERARCHY
Although this is really a particular case of defining lock ordering, a lock hierarchy can
provide a means of checking that the convention is adhered to at runtime.

### Flexible locking with std::unique_lock
std::unique_lock provides a bit more flexibility than std::lock_guard by relaxing
the invariants; a std::unique_lock instance doesn’t always own the mutex that it’s
associated with.

## 3.2.7 Transferring mutex ownership between scopes
- Because std::unique_lock instances don’t have to own their associated mutexes, the
ownership of a mutex can be transferred between instances by moving the instances
around.
- you have to do it explicitly by calling std::move() .
### 3.3.3 Recursive locking
- With std::mutex , it’s an error for a thread to try to lock a mutex it already owns, and
attempting to do so will result in undefined behavior.
- t works just like std::mutex , except that you can acquire
multiple locks on a single instance from the same thread. You must release all your
locks before the mutex can be locked by another thread, so if you call lock() three
times, you must also call unlock() three times. Correct use of std::lock_guard
<std::recursive_mutex> and std::unique_lock<std::recursive_mutex> will han-
dle this for you.
