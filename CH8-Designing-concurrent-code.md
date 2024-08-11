# Designing concurrent code
## 8.1 Techniques for dividing work between threads
### 8.1.1 Dividing data between threads before processing begins
- The easiest algorithms to parallelize are simple algorithms such as std::for_each that perform an operation on each element in a data set.
- The simplest means of dividing the data is to allocate the first N elements to one thread, the next N elements to another thread, and so on.
- This structure will be familiar to anyone who has programmed using the Message Passing Interface ( MPI ) 1 or Open MP 2 frameworks: a task is split into a set of parallel tasks, the worker threads run these tasks independently
- Although this technique is powerful, it can’t be applied to everything. Sometimes the data can’t be divided neatly up front because the necessary divisions become apparent only as the data is processed.
### PIC
### 8.1.2 Dividing data recursively
- The Quicksort algorithm has two basic steps: partition the data into items that come before or after one of the elements (the pivot) in the final sort order and recursively sort those two “halves.” You can’t parallelize this by simply dividing the data up front
- If you’re going to parallelize such an algorithm, you need to make use of the recursive nature.
- In chapter 4, you saw such an implementation. Rather than just performing two
recursive calls for the higher and lower chunks, you used std::async() to spawn asynchronous tasks for the lower chunk at each stage. By using std::async() , you ask the C++ Thread Library to decide when to actually run the task on a new thread and when to run it synchronously.
### PIC
- This is important: if you’re sorting a large set of data, spawning a new thread for each recursion would quickly result in a lot of threads. As you’ll see when we look at perfor-
mance
- if you have too many threads, you might actually slow down the application.
- The idea of dividing the overall task in a recursive fashion like this is a good one; you just need to keep a tighter rein on the number of threads. std::async() can handle this in simple cases, but it’s not the only choice.
- simple cases, but it’s not the only choice. One alternative is to use the std::thread::hardware_concurrency() function to choose the number of threads, as you did with the parallel version of accumulate()
```
#include <list>
#include <thread>
#include <future>
#include <atomic>
#include <algorithm>
#include <vector>
#include <memory>
#include <stack>
// Define thread_safe_stack
#include <iostream>
#include <list>
template<typename T>
class thread_safe_stack {
public:
    void push(T value) {
        std::lock_guard<std::mutex> lock(mutex);
        stack.push(std::move(value));
    }

    std::shared_ptr<T> pop() {
        std::lock_guard<std::mutex> lock(mutex);
        if (stack.empty()) {
            return std::shared_ptr<T>();
        }
        std::shared_ptr<T> const res(std::make_shared<T>(std::move(stack.top())));
        stack.pop();
        return res;
    }

    // Add other necessary methods if needed

private:
    std::stack<T> stack;
    std::mutex mutex;
};

template<typename T>
struct sorter {
    struct chunk_to_sort {
        std::list<T> data;
        std::promise<std::list<T>> promise;
    };
    thread_safe_stack<chunk_to_sort> chunks;
    std::vector<std::thread> threads;
    unsigned const max_thread_count;
    std::atomic<bool> end_of_data;

    sorter() :
        max_thread_count(std::thread::hardware_concurrency() - 1),
        end_of_data(false) {}

    ~sorter() {
        end_of_data = true;
        for (auto& thread : threads) {
            thread.join();
        }
    }

    void try_sort_chunk() {
        std::shared_ptr<chunk_to_sort> chunk = chunks.pop();
        if (chunk) {
            sort_chunk(chunk);
        }
    }

    std::list<T> do_sort(std::list<T>& chunk_data) {
        if (chunk_data.empty()) {
            return chunk_data;
        }
        std::list<T> result;
        result.splice(result.begin(), chunk_data, chunk_data.begin());
        T const& partition_val = *result.begin();

        typename std::list<T>::iterator divide_point =
            std::partition(chunk_data.begin(), chunk_data.end(),
                [&](T const& val) { return val < partition_val; });
        chunk_to_sort new_lower_chunk;
        new_lower_chunk.data.splice(new_lower_chunk.data.end(),
            chunk_data, chunk_data.begin(),
            divide_point);
        std::future<std::list<T>> new_lower =
            new_lower_chunk.promise.get_future();
        chunks.push(std::move(new_lower_chunk));
        if (threads.size() < max_thread_count) {
            threads.push_back(std::thread(&sorter<T>::sort_thread, this));
        }

        std::list<T> new_higher(do_sort(chunk_data));
        result.splice(result.end(), new_higher);
        while (new_lower.wait_for(std::chrono::seconds(0)) !=
            std::future_status::ready) {
            try_sort_chunk();
        }
        result.splice(result.begin(), new_lower.get());
        return result;ذ
    }

    void sort_chunk(std::shared_ptr<chunk_to_sort> const& chunk) {
        chunk->promise.set_value(do_sort(chunk->data));
    }

    void sort_thread() {
        while (!end_of_data) {
            try_sort_chunk();
            std::this_thread::yield();
        }
    }
};

template<typename T>
std::list<T> parallel_quick_sort(std::list<T> input) {
    if (input.empty()) {
        return input;
    }
    sorter<T> s;
    return s.do_sort(input);
}

int main() {
    std::list<int> data = { 34, 7, 23, 32, 5, 62 };
    std::list<int> sorted_data = parallel_quick_sort(data);

    std::cout << "Sorted data: ";
    for (const int& value : sorted_data) {
        std::cout << value << " ";
    }
    std::cout << std::endl;

    return 0;
}

```
## PIC OUT
- Explanation:

    chunk_to_sort Struct: Contains a list of data to be sorted and a promise to return the sorted list.
    sorter Constructor: Initializes the sorter, setting the maximum thread count and end_of_data flag.
    sorter Destructor: Sets end_of_data to true and joins all threads to ensure proper cleanup.
    try_sort_chunk Method: Tries to pop a chunk from the stack and sort it.
    do_sort Method: Implements the sorting logic using a parallel quicksort algorithm:
        If the list is empty, it returns immediately.
        Otherwise, it partitions the list and creates a new chunk for the lower part.
        Pushes the new chunk onto the stack and possibly spawns a new thread.
        Recursively sorts the higher part of the list.
        Merges the sorted lower part back into the result once it's ready.
    sort_chunk Method: Calls do_sort on a chunk and sets the promise value.
    sort_thread Method: Runs in a loop, trying to sort chunks until end_of_data is true.

parallel_quick_sort Function

- This function is the entry point for the parallel quicksort algorithm.
- it does with std::async() . Instead, you limit the number of threads to the value of std::thread::hardware_concurrency() in order to avoid excessive task switching.
- This approach is a specialized version of a thread pool—there’s a set of threads that each take work to do from a list of pending work, do the work, and then go back to the list for more.
### 8.1.3 Dividing work by task type
#### D IVIDING A SEQUENCE OF TASKS BETWEEN THREADS
- If your task consists of applying the same sequence of operations to many independent data items, you can use a pipeline to exploit the available concurrency of your system. This is by analogy to a physical pipeline: data flows in at one end through a series of operations (pipes) and out at the other end.
- Pipelines are also good where each operation in the sequence is time consuming; by dividing the tasks between threads rather than the data
## 8.2 Factors affecting the performance of concurrent code
- Customers won’t thank you if your application runs more slowly on their shiny new 16-core machine than it did on their old single-core one

### 8.2.1 How many processors?
- The number (and structure) of processors is the first big factor that affects the performance of a multithreaded application, and it’s quite a crucial one.
- you might be developing on a dual- or quad-core system, but your customers’ systems may have one multicore processor (with any number of cores), or multiple single-core processors, or even multiple multicore processors. The behavior and performance characteristics of a concurrent program can vary considerably under such different circumstances, so you need to think carefully about what the impact may be and test things where possible.
- To a first approximation, a single 16-core processor is the same as 4 quad-core processors or 16 single-core processors: in each case the system can run 16 threads concurrently
- If it has fewer than 16, you’re leaving processor power on the table , your application will waste processor time switching between the threads the situation is called oversubscription.

### 8.2.2 Data contention and cache ping-pong
- If two threads are executing concurrently on different processors and they’re both reading the same data, this usually won’t cause a problem; the data will be copied into their respective caches, and both processors can proceed. However, if one of the threads modifies the data, this change then has to propagate to the cache on the other core, which takes time.
- high contention:  one processor is ready to update the value, but another processor is currently doing that, so it has to wait until the second processor has completed its update and the change has propagated
- low contention:If the processors rarely have to wait for each other
```
std::atomic<unsigned long> counter(0);
void processing_loop()
{
while(counter.fetch_add(1,std::memory_order_relaxed)<100000000)
{
do_something();
}
}
```
- the data for counter will be passed back and forth between
the caches many times. This is called cache ping-pong,
- Cache ping-pong effects can nullify the benefits of such a mutex if the workload is unfavorable, because all threads accessing the data (even reader threads) still have to modify the mutex itself.

### 8.2.3 False sharing
- Processor caches don’t generally deal in individual memory locations; instead, they deal in blocks of memory called cache lines. These blocks of memory are typically 32 or 64 bytes in size, but the exact details depend on the particular processor model being used.

### 8.2.5 Oversubscription and excessive task switching 
- In multithreaded systems, it’s typical to have more threads than processors, unless you’re running on massively parallel hardware.
- This isn’t always a good thing. If you have too many additional threads, there will be more threads ready to run than there are available processors, and the operating system will have to start task switching quite heavily in order to ensure they all get a fair time slice.


## 8.3 Designing data structures for multithreaded performance
- The key things to bear in mind when designing your data structures for multithreaded performance are contention, false sharing, and data proximity.

### 8.4.4 Improving responsiveness with concurrency
- Most modern graphical user interface frameworks are event driven; the user performs actions on the user interface by pressing keys or moving the mouse, which generate a series of events or messages that the application then handles.
