# ch5 he C++ memory model and operations on atomic types
## Memory model basics
- There are two aspects to the memory model: the basic structural aspects, which relate to how things are laid out in memoryt
- The concurrency aspects. The structural aspects are important for concurrency
### 5.1.1 Objects and memory locations
- There are four important things to take away from this:
  - Every variable is an object, including those that are members of other objects.
  - Every object occupies at least one memory location.
  - Variables of fundamental type such as int or char are exactly on memory location, whatever their size, even if they’re adjacent or part of an array.
  - Adjacent bit fields are part of the same memory location
###  Objects, memory locations, and concurrency
- If two threads access separate memory locations, there’s no problem: everything works fine.
- On the other hand, if two threads access the same memory location, then you have to be careful
- If neither thread is updating the
memory location, you’re fine; read-only data doesn’t need protection or synchronization. If either thread is modifying the data, there’s a potential for a race condition.
- the synchronization properties of atomic operations on the same
or other memory locations to enforce an ordering between the accesses in the two threads.

### Modification orders
- Every object in a C++ program has a defined modification order composed of all the writes to that object from all threads in the program.
- If the object in question isn’t one of the atomic types described in section 5.2, you’re responsible for making certain that there’s sufficient synchronization to ensure that threads agree on the modification order of each variable.
- If different threads see distinct sequences of values
for a single variable, you have a data race and undefined behavior
- If you do use atomic operations, the compiler is responsible for ensuring that the necessary synchronization is in place.

## 5.2 Atomic operations and types in C++
-  An atomic operation is an indivisible (غير قابله للتجرئة) operation.
- If the load operation that reads the value of an object and all modifications are atomic, that load will retrieve either the initial value of the object or the value stored.

### The standard atomic types
- atomic types can be found in the <atomic> header
- you can use mutexes to make other operations appear atomic.
- is_lock_free() member function, which allows the user to determine whether operations on a given type are done directly with atomic instructions
- std::atomic_flag are initialized to clear
- ![Screenshot from 2024-07-30 09-58-42](https://github.com/user-attachments/assets/65706578-6b4d-41b7-b8ed-f99a244e5ec0)


- As well as the basic atomic types, the C++ Standard Library also provides a set of typedef s for the atomic types corresponding to the various nonatomic Standard Library typedef s such as std::size_t .
![Screenshot from 2024-07-30 09-55-31](https://github.com/user-attachments/assets/499783e6-d282-4310-8b85-5d6861cebf01)


- they have no copy constructors or copy assignment operators.
- They do, however, support assignment from and implicit conversion to the corresponding built-in types as well as direct load() and store() member functions, exchange() , compare_exchange_weak() , and compare_exchange_strong() . They also support the compound assignment operators where appropriate: +=, -=, *=, |=, and so on, and the integral types and std::atomic<> specializations for pointers support ++ and -- .
- The std::atomic<> class template is a generic class template, the operations are limited to load() , store() (and assignment from and conversion to the user-defined type), exchange(), compare_exchange_weak() , and compare_exchange_strong() .
- the operations are divided into three categories:
  - Store operations, which can have memory_order_relaxed , memory_order_release , or memory_order_seq_cst ordering
  - Load operations, which can have memory_order_relaxed , memory_order_consume , memory_order_acquire, or memory_order_seq_cst ordering
  - Read-modify-write operations, which can have memory_order_relaxed , memory_order_consume , memory_order_acquire , memory_order_release , memory_order_acq_rel, or memory_order_seq_cst ordering
### 5.2.2 Operations on std::atomic_flag
- std::atomic_flag is the simplest standard atomic type, which represents a Boolean
flag have two states: set or clear.
- Objects of type std::atomic_flag must be initialized with ATOMIC_FLAG_INIT . This ini-
tializes the flag to a clear state. There’s no choice in the matter; the flag always starts clear:std::atomic_flag f=ATOMIC_FLAG_INIT;
- Once you have your flag object initialized there are only three things you can do
with it: destroy it, clear it, or set
- All operations on an atomic type are defined as atomic,
```
#include <iostream>
#include <atomic>
#include <thread>
#include <vector>

class spinlock_mutex
{
    std::atomic_flag flag = ATOMIC_FLAG_INIT;
public:
    spinlock_mutex() = default;
    
    void lock()
    {
        while(flag.test_and_set(std::memory_order_acquire));
    }
    
    void unlock()
    {
        flag.clear(std::memory_order_release);
    }
};

spinlock_mutex spinlock;
int counter = 0;

void increment_counter(int iterations)
{
    for(int i = 0; i < iterations; ++i)
    {
        spinlock.lock();
        ++counter;
        spinlock.unlock();
    }
}

int main()
{
    const int num_threads = 10;
    const int iterations = 1000;
    
    std::vector<std::thread> threads;
    
    // Create multiple threads
    for(int i = 0; i < num_threads; ++i)
    {
        threads.emplace_back(increment_counter, iterations);
    }
    
    // Join all threads
    for(auto& thread : threads)
    {
        thread.join();
    }
    
    std::cout << "Final counter value: " << counter << std::endl;
    
    return 0;
}

```
![Screenshot from 2024-07-31 10-22-31](https://github.com/user-attachments/assets/d954cab0-51d0-433b-bc56-50ae5d53aebc)
- while(flag.test_and_set(std::memory_order_acquire));

  - This line is part of the lock method in the spinlock_mutex class:

    -  flag.test_and_set(std::memory_order_acquire):
        test_and_set is an atomic operation that sets the atomic flag to true and returns its previous value.
        If the flag was already set (i.e., another thread holds the lock), test_and_set will return true.
        If the flag was not set (i.e., the lock is available), test_and_set will return false.
        The std::memory_order_acquire ensures that all subsequent memory operations in this thread are not reordered before this operation. This provides a synchronization point and establishes a "happens-before" relationship.

    - while Loop:
        The loop continues to execute test_and_set as long as the flag is already set (i.e., the lock is held by another thread).
        This is a "spin-wait" loop, where the thread continuously checks and tries to acquire the lock.

- flag.clear(std::memory_order_release);

   - This line is part of the unlock method in the spinlock_mutex class:
   - flag.clear(std::memory_order_release):
        clear is an atomic operation that sets the flag to false.
        The std::memory_order_release ensures that all previous memory operations in this thread are not reordered after this operation. This provides a synchronization point and establishes a "happens-before" relationship with any subsequent acquire operations.

-  threads.emplace_back(increment_counter, iterations);

   - This line is part of the main function, where multiple threads are created:

     - threads.emplace_back(increment_counter, iterations):
        emplace_back is a method of std::vector that constructs an object in place at the end of the vector. In this case, it constructs a std::thread object.
        increment_counter is the function that each thread will execute.
        iterations is the argument passed to the increment_counter function.
        This creates and starts a new thread that will execute the increment_counter function, incrementing the global counter variable iterations times.

### Operations on standard atomic integral types
- As well as the usual set of operations ( load() , store() , exchange() , compare_
exchange_weak() , and compare_exchange_strong() )
- fetch_add() , fetch_sub() , fetch_and() , fetch_or() , fetch_xor()
- ompound-assignment forms of these operations ( += , -= , &= , |= , and
^= ), and pre- and post-increment and decrement ( ++x , x++ , -- x , and x-- )
### Free functions for atomic operations
 ## PIC OUT
##  Synchronizing operations and enforcing ordering
- the first thread sets a flag to indicate that the data is read
- the second thread doesn’t read the data until the flag is set. The following listing shows such a scenario.

![Screenshot from 2024-07-30 10-57-22](https://github.com/user-attachments/assets/3a123e77-71a9-47b9-91d4-9f2df882965f)

### The synchronizes-with relationship
- a suitably tagged atomic write operation W on a variable x syn-
chronizes-with a suitably tagged atomic read operation on x that reads the value
stored by either that write ( W )
- subsequent atomic write operation on x by the
same thread that performed the initial write W , or a sequence of atomic read-modify-
write operations on x (such as fetch_add() or compare_exchange_weak() ) by any
thread
- if one operation is sequenced before another, then it also happens-before it. This means that if one operation (A) occurs in a statement prior to another (B) in the source code, then A happens-before B.
- The next part I think i cant summrize due to complexity


