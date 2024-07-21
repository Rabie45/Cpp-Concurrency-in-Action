# What is concurrency?
- concurrency is about two or more separate activities happening at the same time.

## Concurrency in computer systems
we mean a single system performing multiple independent activities in parallel
- multitasking operating systems that allow a single computer to run multiple applications at the same time through task switching have been commonplace for many years
- Historically, most computers have had one processor, with a single processing
unit or core, and this remains true for many desktop machines today.
- task switching: machine can really only perform one task at a time, but it can switch between tasks many times per second. it appears that the tasks are happening concurrently.
- hardware concurrency: Computers containing multiple processors have been used for servers and high- performance computing tasks for a number of year
- Though the availability of concurrency in the hardware is most obvious with multi-
processor or multicore systems, some processors can execute multiple threads on a
single core.
- hardware threads:the measure of how many independent tasks the hardware can genuinely run concurrently.
![Screenshot from 2024-07-21 11-26-06](https://github.com/user-attachments/assets/38bbb6c2-450b-4f4f-b3f9-68ad6e367cc1)
![Screenshot from 2024-07-21 12-15-14](https://github.com/user-attachments/assets/af9c9459-9e55-4688-9c24-bc7a48ee71cb)

## Approaches to concurrency
- Each developer represents a thread, and each office represents a pro-
cess. The first approach is to have multiple single-threaded processes, which is similar
to having each developer in their own office, and the second approach is to have mul-
tiple threads in a single process, which is like having two developers in the same office.

## CONCURRENCY WITH MULTIPLE PROCESSES
![Screenshot from 2024-07-21 12-21-48](https://github.com/user-attachments/assets/06095a9b-d1c0-43df-b026-45b65b271fda)

- divide the application into multiple, separate, single-threaded processes that are run at the same time. These separate processes can then pass messages to each other through all the normal interprocess communication channels
## CONCURRENCY WITH MULTIPLE THREADS

![Screenshot from 2024-07-21 12-21-55](https://github.com/user-attachments/assets/0677a005-d0c3-4d77-89a6-b0a409c185fc)

- The alternative approach to concurrency is to run multiple threads in a single pro-
cess. Threads are much like lightweight processes
### Why use concurrency?
- There are two main reasons to use concurrency in an application: separation of concerns and performance.

### Using concurrency for separation of concerns
- In a single thread, the application has to check for user input at regular intervals during the playback, thus conflating the DVD playback code with the user interface code.
- By using multithreading to separate these concerns, the user interface code and DVD playback
code no longer have to be so closely intertwined


- separate threads are often used to run tasks that must run contin-
uously in the background, such as monitoring the filesystem for changes in a desktop
search application.

## Using concurrency for performance
-  chip manufacturers have increasingly been favoring multicore designs with 2, 4, 16, or more pro-
cessors on a single chip over better performance with a single core.
- The increased computing power of multicore embedded devices machines comes not
from running a single task faster but from running multiple tasks in parallel.
- There are two ways to use concurrency for performance. 
  -  task parallelism is to divide a single task into parts and run each in parallel, thus reducing the total runtime. 
  -  data parallelism each thread performs the same operation on different parts of the data.
  - embarrassingly parallel Algorithms that are readily susceptible to such parallelism.
  - The second way to use concurrency for performance is to use the available paral-
lelism to solve bigger problems;
  - an application of data parallelism, by performing the same operation on multiple sets of data concurrently
  
## When not to use concurrency
- the only reason not to use concurrency is when the benefit is not worth the cost
- Code using concurrency is harder to understand in many cases, so there’s a direct intellectual cost to writing and maintaining multithreaded code, and the additional complexity can also lead to more bugs.
- If you have too many threads running at once, this consumes OS resources and may make the system as a whole run slower.
- using too many threads can exhaust the available memory oraddress space for a process, because each thread requires a separate stack space.

- for 32-bit processes with a flat architecture where there’s a 4 GB limit in the available address space: if each thread has a a 1 MB stack (as is typical on many systems), then the address space would be all used up with 4096 threads, without allowing for any space for code or static data or heap data.
- Although 64-bit (or larger) systems don’t have this direct address-space limit, they still have finite resources: if you run too many threads, this will eventually cause problems. Though thread pools (see chapter 9) can be used to limit the number of threads, these are not a silver bul-
let, and they do have their own issues.
- If the server side of a client/server application launches a separate thread for each
connection, this works fine for a small number of connections
- the more threads you have running, the more context switching the operating system has to do. Each context switch takes time that could be spent doing useful work

## Concurrency and multithreading in C++
- the types in the C++ Thread Library may offer a native_handle() member function that allows
the underlying implementation to be directly manipulated using a platform-specific
API

# Getting started
## Hello, Concurrent World
```
#include <iostream>
#include <thread>

void hello() { std::cout << "Hello Concurrent World\n"; }
int main() {
  std::thread t(hello);
  t.join();
}
```
![Screenshot from 2024-07-21 14-56-54](https://github.com/user-attachments/assets/070f188b-2149-4041-ba7e-d15114338db1)
- The first difference is the extra #include <thread>
- the code for writing the message has been moved to a separate function This is because every thread has to have an initial function
- for every other thread it’s specified in the constructor of a std::thread object
- join() is there e —as described in chapter 2, this causes
the calling thread (in main() ) to wait for the thread associated with the std::thread
object, in this case, t .
