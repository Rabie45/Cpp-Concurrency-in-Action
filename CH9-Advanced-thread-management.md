# Ch 9 Advanced thread management
- employees who would normally spend their time in the office are
occasionally required to visit clients or suppliers or attend a trade show or conference.
Although these trips might be necessary, and on any given day there might be several
people making such a trip, it may well be months or even years between such trips for
any particular employee. Since it would therefore be rather expensive and impractical
for each employee to have a company car, companies often offer a car pool instead;
## 9.1 Thread pools
- A thread pool is a similar idea, except that threads are being shared rather than
cars.
- it’s impractical to have a separate thread for every task that can potentially be done in parallel with other tasks
- There are several key design issues when building a thread pool, such as how many threads to use, the most efficient way to allocate tasks to threads, and whether or not you can wait for a task to complete.
### 9.1.1 The simplest possible thread pool
- a thread pool is a fixed number of worker threads (typically the same
number as the value returned by std::thread::hardware_concurrency() )
- you call a function to put it on the queue of pending work.
-  The following listing shows a sample implementation of such a thread pool.
```
#include <iostream>
#include <atomic>
#include <thread>
#include <vector>
#include <functional>
#include <mutex>
#include <condition_variable>
#include <queue>
class join_threads
{
std::vector<std::thread>& threads;
public:
explicit join_threads(std::vector<std::thread>& threads_):
threads(threads_)
{}
~join_threads()
{
for(unsigned long i=0;i<threads.size();++i)
{
if(threads[i].joinable())
threads[i].join();
}
}
};

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
class thread_pool {
    std::atomic_bool done;
    threadsafe_queue<std::function<void()>> work_queue;
    std::vector<std::thread> threads;
    join_threads joiner;

    void worker_thread() {
        while (!done) {
            std::function<void()> task;
            if (work_queue.try_pop(task)) {
                task();
            } else {
                std::this_thread::yield();
            }
        }
    }

public:
    thread_pool() : done(false), joiner(threads) {
        unsigned const thread_count = std::thread::hardware_concurrency();
        try {
            for (unsigned i = 0; i < thread_count; ++i) {
                threads.push_back(std::thread(&thread_pool::worker_thread, this));
            }
        } catch (...) {
            done = true;
            throw;
        }
    }

    ~thread_pool() {
        done = true;
    }

    template<typename FunctionType>
    void submit(FunctionType f) {
        work_queue.push(std::function<void()>(f));
    }
};

int main() {
    // Create a thread pool
    thread_pool pool;

    // Submit a simple task to the thread pool
    pool.submit([] {
        std::cout << "Hello from the thread pool!" << std::endl;
    });

    // Submit multiple tasks
    for (int i = 0; i < 5; ++i) {
        pool.submit([i] {
            std::cout << "Task " << i << " is being processed by the thread pool." << std::endl;
        });
    }

    // Let the tasks complete before the program exits
    std::this_thread::sleep_for(std::chrono::seconds(1));

    return 0;
}
```
- This is very complicated :) we used  class join_threads from ch 8
and threadsafe_queue from ch 6
- This explination from chatGPT
- Breakdown of thread_pool Class

    - std::atomic_bool done;
        This is an atomic boolean flag used to signal when the thread pool should stop. It ensures thread-safe operations when multiple threads access this flag.

    - thread_safe_queue<std::function<void()> > work_queue;
        This is a thread-safe queue that holds tasks to be executed by the thread pool. The tasks are stored as std::function<void()>, which allows the queue to hold any callable object (e.g., functions, lambdas).

    - std::vector<std::thread> threads;
        This vector holds all the worker threads that will be created in the thread pool.

    - join_threads joiner;
        This is an instance of a helper class (join_threads), which ensures that all threads are joined (i.e., finish execution) when the thread pool is destroyed.

    - void worker_thread()
        This is a member function that each thread in the pool runs. It repeatedly checks for tasks in the work_queue. If a task is available, it executes it. If not, it yields control to allow other threads to run.

    - public: thread_pool(): done(false), joiner(threads)
        Constructor: Initializes the done flag to false and associates the joiner object with the threads vector. It then determines the number of hardware threads available and creates that many worker threads.

    - ~thread_pool()
        Destructor: Sets the done flag to true, signaling all worker threads to stop.

    - template<typename FunctionType> void submit(FunctionType f)
        A template function that accepts any callable object (f) and pushes it onto the work_queue to be executed by the threads.

## OUT PIC

### 9.1.2 Waiting for tasks submitted to a thread pool
- in the examples in chapter 8 that explicitly spawned threads, after dividing the work between threads, the master thread always waited for the newly spawned threads to finish, to ensure that the overall task was complete before returning to the caller.
- With thread pools, you’d need to wait for the tasks submitted to the thread pool to complete
- his is similar to the way that the std::async

### 9.1.5 Work stealing
- In order to allow a thread with no work to do to take work from another thread with a full queue, the queue must be accessible to the thread doing the stealing from run_pending_tasks().
- It’s possible to write a lock-free queue that allows the owner thread to push and pop at one end while other threads can steal entries from the other, but the implementation of such a queue is beyond the scope of this book.

## 9.2 Interrupting threads
- In many situations it’s desirable to signal to a long-running thread that it’s time to stop. This might be because it’s a worker thread for a thread pool and the pool is now being destroyed, or it might be because the work being done by the thread has been explicitly canceled by the user, or a myriad of other reasons.
### 9.2.1 Launching and interrupting another thread
To start with, let’s look at the external interface. What do you need from an interrupt-
ible thread? At the basic level, all you need is the same interface as you have for
std::thread , with an additional interrupt() function:
```
class interruptible_thread
{
public:
template<typename FunctionType>
interruptible_thread(FunctionType f);
void join();
void detach();
bool joinable() const;
void interrupt();
};
```
- You can check this part ur self