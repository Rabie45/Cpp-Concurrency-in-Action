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
### CODE

### OUTPUT
- The first difference is the extra #include <thread>
- the code for writing the message has been moved to a separate function This is because every thread has to have an initial function
- for every other thread it’s specified in the constructor of a std::thread object
- join() is there e —as described in chapter 2, this causes
the calling thread (in main() ) to wait for the thread associated with the std::thread
object, in this case, t .
