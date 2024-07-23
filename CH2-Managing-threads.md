# Ch2 Managing threads
## Basic thread management
- Every C++ program has at least one thread
- if you have a std::thread object for a thread, you can wait for it to finish;
but first you have to start it, so let’s look at launching threads.

### Launching a thread
- Threads are started by constructing a std::thread object that specifies the task to run on that thread.
- That task is just a plain, ordinary void -returning function that takes no parameters.
- This function runs on its own thread until it returns, and then the thread stops.
- the task could be a function object that takes additional parameters and performs a series of independent operations that are specified through some kind of messaging system
while it’s running,
```
void do_some_work();
std::thread my_thread(do_some_work);
```
- std::thread works with any callable type, so you can pass an instance of a class with a function call operator to the std::thread constructor instead:
```
class background_task
{
public:
void operator()() const
{
do_something();
do_something_else();
}
};
background_task f;
std::thread my_thread(f);
```
- In this case, the supplied function object is copied into the storage belonging to the
newly created thread of execution and invoked from there.
- Avoid what is dubbed “C++’s most vexing parse.”
- the extra parentheses prevent interpretation as a function declaration, thus allowing my_thread to be declared as a variable of type std::thread.
- uses the new uniform initialization syntax with braces rather than parentheses, and thus would also declare a variable.
```
std::thread my_thread((background_task()));
std::thread my_thread{background_task()};
```
- One type of callable object that avoids this problem is a lambda expression. allows you to write a local function, possibly capturing some local variables and avoiding the need of passing additional arguments.
- The previous example can be written using a lambda expression as follows:
```
std::thread my_thread([](
do_something();
do_something_else();
});
```
- if you used the resource after the thread us finished it will be undefined behavior to access an object after it’s been destroyed.
- 
```
struct func
{
int& i;
func(int& i_):i(i_){}
void operator()()
{
for(unsigned j=0;j<1000000;++j)
{
do_something(i);
}
}
};
void oops()
{
int some_local_state=0;
func my_func(some_local_state);
std::thread my_thread(my_func);
my_thread.detach();
}
```
- the new thread associated with my_thread will probably still be running
when oops exits because you’ve explicitly decided not to wait for it by calling
detach()
- If the thread is still running, then the next call to do_something(i)
- but it’s easier to make the mistake with multithreaded code, because it isn’t necessarily immediately apparent that this has happened.
- to handle this scenario is to make the thread function selfcontained and copy the data into the thread rather than sharing the data.
### Waiting for a thread to complete
- If you need to wait for a thread to complete, you can do this by calling join() on the associated std::thread instance
- join() is simple and brute force—either you wait for a thread to finish or you
don’t.
### Waiting in exceptional circumstances
- you need to ensure that you’ve called either join() or detach() before a std::thread object is destroyed.
- use the standard Resource Acquisition Is Initialization
( RAII ) idiom and provide a class that does the join() in its destructor

### Running threads in the background
- Calling detach() on a std::thread object leaves the thread to run in the back-
ground

## Passing arguments to a thread function
- passing arguments to the callable object or function is fundamentally as simple as passing additional arguments to the std::thread constructor.

``` 
void f(int i,std::string const& s);
std::thread t(f,3,”hello”);
```

- Another interesting scenario for supplying arguments is where the arguments
can’t be copied but can only be moved
- An example of such a type is std::unique_ptr , which provides automatic memory management for dynamically allocated objects.
- The move constructor and move assignment operator allow the ownership of an object to be transferred around between std::unique_ptr instances


## Transferring ownership of a thread
- shows the creation of two threads of execution and the transfer of ownership of those
threads among three std::thread instances, t1 , t2 , and t3 :

```
void some_function();
void some_other_function();
std::thread t1(some_function);
std::thread t2=std::move(t1);
t1=std::thread(some_other_function);
std::thread t3;
t3=std::move(t2);
t1=std::move(t3);
```
## Choosing the number of threads at runtime
- std::thread::hardware_concurrency()  This function returns an indication of the number of threads that can.
truly run concurrently for a given execution of a program.

## Identifying threads

- Thread identifiers are of type std::thread::id and can be retrieved in two ways
- the identifier for a thread can be obtained from its associated std::thread
object by calling the get_id() member function.
- f the std::thread object doesn’t have an associated thread of execution, the call to get_id() returns a defaultconstructed std::thread::id object, which indicates “not any thread.” Alternatively, the identifier for the current thread can be obtained by calling std::this_thread:: get_id() , which is also defined in the <thread> header.


