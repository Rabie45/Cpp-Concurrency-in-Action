# CH10 Testing and debugging multithreaded applications
- Testing and debugging are like two sides of a coin
## 10.1 Types of concurrency-related bugs
- You can get just about any sort of bug in concurrent code; it’s not special in that regard. But some types of bugs are directly related to the use of concurrency and therefore of particular relevance to this book. Typically, these concurrency-related bugs fall into two primary categories:
 - Unwanted blocking
 - Race conditions


### 10.1.1 Unwanted blocking
- A thread is blocked when it’s unable to proceed because it’s waiting for something. This is typically something like a mutex, a condition variable, or a future, but it could be waiting for I/O .
- why is this blocking unwanted? Typically, this is because some other thread is also waiting for the blocked thread to perform some action, and so that thread in turn is blocked. There are several variations on this theme:
   - Deadlock: one thread is waiting for another, which is in turn waiting for the first. If your threads deadlock, the tasks they’re supposed to be doing won’t get done.
   - In the most visible cases, one of the threads involved is the thread responsible for the user interface, in which case the interface will cease to respond. In other cases, the interface will remain responsive, but some required task won’t complete, such as a search not returning or a document not printing
   - Livelock:  is similar to deadlock in that one thread is waiting for another, which is in turn waiting for the first. The key difference here is that the wait is not a blocking wait but an active checking loop, such as a spin lock
   - Blocking on I/O or other external input If your thread is blocked waiting for external input, it can’t proceed, even if the waited-for input is never going to come.
### 10.1.2 Race conditions
- many deadlocks and livelocks only actually manifest because of a race condition.
- Not all race conditions are problematic—a race condition occurs anytime the behavior depends on the relative scheduling of operations in separate threads.
- race conditions often cause the following types of problems:
  - A data race is the specific type of race condition that results in
undefined behavior because of unsynchronized concurrent access to a shared
memory location.
- Broken invariants—These can manifest as dangling pointers (because another
thread deleted the data being accessed), random memory corruption (due to a
thread reading inconsistent values resulting from partial updates), and double-
free (such as when two threads pop the same value from a queue, and so both
delete some associated data), among others.
- Lifetime issues—Although you could bundle these problems in with broken invariants, this really is a separate category.
- If you manu- ally call join() in order to wait for the thread to complete, you need to ensure that the call to join() can’t be skipped if an exception is thrown.

## 10.2 Techniques for locating concurrency-related bugs
### 10.2.1 Reviewing code to locate potential bugs
- Q UESTIONS TO THINK ABOUT WHEN REVIEWING MULTITHREADED CODE
- Which data needs to be protected from concurrent access?
- How do you ensure that the data is protected?
- Where in the code could other threads be at this time?
- Which mutexes does this thread hold?
- Which mutexes might other threads hold?
- Are there any ordering requirements between the operations done in this
- thread and those done in another? How are those requirements enforced?
- Is the data loaded by this thread still valid? Could it have been modified by
other threads?
- If you assume that another thread could be modifying the data, what would that
- mean and how could you ensure that this never happens?
### 10.2.2 Locating concurrency-related bugs by testing
- Testing multithreaded code is an order of magnitude harder, because the precise
scheduling of the threads is indeterminate and may vary from run to run.
There’s more to testing concurrent code than the structure of the code being
tested; the structure of the test is just as important, as is the test environment. If you
continue on with the example of testing a concurrent queue, you have to think about
various scenarios:

-  One thread calling push() or pop() on its own to verify that the queue does
- work at a basic level
- One thread calling push() on an empty queue while another thread calls pop()
- Multiple threads calling push() on an empty queue
- Multiple threads calling push() on a full queue
- Multiple threads calling pop() on an empty queue
- Multiple threads calling pop() on a full queue
- Multiple threads calling pop() on a partially full queue with insufficient items
- for all threads
- Multiple threads calling push() while one thread calls pop() on an empty queue
- Multiple threads calling push() while one thread calls pop() on a full queue
- Multiple threads calling push() while multiple threads call pop() on an empty
 queue
- Multiple threads calling push() while multiple threads call pop() on a full queue
### 10.2.3 Designing for testability
- code is easier to test if the following factors apply:
- The responsibilities of each function and class are clear.
- The functions are short and to the point.
- Your tests can take complete control of the environment surrounding the code
being tested.
- The code that performs the particular operation being tested is close together
- rather than spread throughout the system.
- You thought about how to test the code before you wrote it.
### 10.2.4 Multithreaded testing techniques
#### BRUTE - FORCE TESTING
- The idea behind brute-force testing is to stress the code to see if it breaks. This typically means running the code many times, possibly with many threads running at once.
- The downside to brute-force testing is that it might give you false 
- confidence. If the way you’ve written the test means that the problematic circumstances can’t occur, you can run the test as many times as you like and it won’t fail, even if it would fail every time in slightly different circumstances.
- C OMBINATION SIMULATION TESTING
- That’s a bit of a mouthful, so I’d best explain what I mean. The idea is that you run your code with a special piece of software that simulates the real runtime environment of the code.
### 10.2.5 Structuring multithreaded test code
- you need to identify the distinct parts of each test:

- The general setup code that must be executed before anything else
- The thread-specific setup code that must run on each thread
- The actual code for each thread that you desire to run concurrently
- The code to be run after the concurrent execution has finished, possibly
including assertions on the state of the code
An example test for concurrent push() and pop() calls on a queue
```
void test_concurrent_push_and_pop_on_empty_queue()
{
threadsafe_queue<int> q;
std::promise<void> go,push_ready,pop_ready;
std::shared_future<void> ready(go.get_future());
std::future<void> push_done;
std::future<int> pop_done;
{
push_done=std::async(std::launch::async,
[&q,ready,&push_ready]()
{
push_ready.set_value();
ready.wait();
q.push(42);
}
);
pop_done=std::async(std::launch::async,
[&q,ready,&pop_ready]()
{
pop_ready.set_value();
ready.wait();
return q.pop();
}
);
push_ready.get_future().wait();
pop_ready.get_future().wait();
go.set_value();

push_done.get();
assert(pop_done.get()==42);
assert(q.empty());
}
catch(...)
{
go.set_value();
throw;
}
}
```
- First, you create your empty queue as part of the general setup . Then, you create all your promises for the “ready” signals  and get a std::shared_future for the go signal . Then, you create the futures you’ll use to indicate that the threads have finished  .
- Inside the try block you can then start the threads you use std::
launch::async to guarantee that the tasks are each running on their own thread.
Note that the use of std::async makes your exception-safety task easier than it would be with plain std::thread because the destructor for the future will join with the thread.

