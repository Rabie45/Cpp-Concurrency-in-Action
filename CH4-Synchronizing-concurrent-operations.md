# Ch4 Synchronizing concurrent operations
## Waiting for an event or other condition
- if one thread is waiting for a second thread to complete a task, it has several options. First, it could just keep checking a flag in
shared data (protected by a mutex) and have the second thread set the flag when it completes the task.
- the thread consumes valuable processing time repeatedly checking the flag, and when the mutex is locked by the waiting thread, it can’t be locked by any other thread.
- A second option is to have the waiting thread sleep for small periods between the checks using the std::this_thread::sleep_for()
```
#include <iostream>
#include <mutex>
#include <thread>

bool flag = false;
std::mutex m;
void wait_for_flag() {
  std::unique_lock<std::mutex> lk(m);
  while (!flag) {
    lk.unlock();
    std::this_thread::sleep_for(std::chrono::milliseconds(100));
    std::cout << "wait_for_flag\n";
    lk.lock();
  }
}
void set_flag() {
    // Simulate some work with sleep
    std::this_thread::sleep_for(std::chrono::seconds(1));
    // Lock the mutex
    std::lock_guard<std::mutex> lk(m);
    // Set the flag
    flag = true;
    std::cout << "Flag set to true\n";
}

int main() {
    std::thread waiter(wait_for_flag);
    std::thread setter(set_flag);

    waiter.join();
    setter.join();

    std::cout << "wait_for_flag function completed\n";
    return 0;
}

```
![Screenshot from 2024-07-28 09-48-10](https://github.com/user-attachments/assets/c3cb91f8-abf4-4e63-8703-bd8ca82ab923)
- In the loop, the function unlocks the mutex B before the sleep c and locks it again
afterward d , so another thread gets a chance to acquire it and set the flag .
- Preferred, option is to use the facilities from the C++ Standard
Library to wait for the event itself.
- The most basic mechanism for waiting for an event to be triggered by another thread is the condition variable.

### Waiting for a condition with condition variables
- two implementations of a condition variable: std::condition_variable and std::condition_variable_any .
- they need to work with a mutex in order to provide appropriate synchronization; the former is limited to working with std::mutex
- std::condition_variable_any is more general, there’s the potential for additional costs in terms of size, performance.
``` 
#include <condition_variable>
#include <iostream>
#include <mutex>
#include <queue>
#include <thread>

struct data_chunk {
  int id;
  // Add more fields as necessary
};

std::mutex mut;
std::queue<data_chunk> data_queue;
std::condition_variable data_cond;

bool more_data_to_prepare() {
  static int count = 0;
  return count++ < 10; // Simulate 10 chunks of data
}

data_chunk prepare_data() {
  static int id = 0;
  return data_chunk{id++}; // Simulate data preparation
}

void process(data_chunk data) {
  std::cout << "Processing data chunk with id: " << data.id << std::endl;
}

bool is_last_chunk(data_chunk data) {
  return data.id == 9; // The last chunk is id 9 in this example
}

void data_preparation_thread() {
  while (more_data_to_prepare()) {
    data_chunk const data = prepare_data();
    std::lock_guard<std::mutex> lk(mut);
    data_queue.push(data);
    data_cond.notify_one();
  }
}

void data_processing_thread() {
  while (true) {
    std::unique_lock<std::mutex> lk(mut);
    std::cout << "data_processing_thread " << std::endl;
    data_cond.wait(lk, [] { return !data_queue.empty(); });
    data_chunk data = data_queue.front();
    data_queue.pop();
    lk.unlock();
    process(data);
    if (is_last_chunk(data))
      break;
  }
}

int main() {
  std::thread producer(data_preparation_thread);
  std::thread consumer(data_processing_thread);

  producer.join();
  consumer.join();

  return 0;
}
```
![Screenshot from 2024-07-28 11-04-34](https://github.com/user-attachments/assets/f91ef521-72a5-43d2-a333-09185d0faf30)

- you have a queue B that’s used to pass the data between the two threads. When the data is ready, the thread preparing the data locks the mutex protecting the queue using a std::lock_guard and pushes the data onto the queue c . It then calls the notify_one() member function on the std::condition_variable instance to notify the waiting thread (if there is one)..
- lambda function []{return !data_queue.empty();} checks
to see if the data_queue is not empty() —that is, there’s some data in the queue ready
for processing.
- If the condition isn’t satisfied (the lambda function returned false ), wait() unlocks the mutex and puts the thread in a blocked or waiting state. When the condition variable is notified by a call to notify_one() from the data-preparation thread, the thread
wakes from its slumber

### Returning values from background tasks Suppose you have a long-running calculation
- You use std::async to start an asynchronous task for which you don’t need the result right away.
- When you need the value, you just call get() on the future, and the
thread blocks until the future is ready and then returns the value.
- std::async allows you to pass additional arguments to the function by adding extra arguments to the call, in the same way that std::thread does.

``` 
#include<deque>
#include<mutex>
#include<future>
#include<thread>
#include<utility>

std::mutex m;
std::deque<std::packaged_task<void()> > tasks;
bool gui_shutdown_message_received();
void get_and_process_gui_message();
void gui_thread()
{
while(!gui_shutdown_message_received())
{
get_and_process_gui_message();
std::packaged_task<void()> task;
{
std::lock_guard<std::mutex> lk(m);
if(tasks.empty())
continue;
task=std::move(tasks.front());
tasks.pop_front();
}
task();
}
}

std::thread gui_bg_thread(gui_thread);
template<typename Func>
std::future<void> post_task_for_gui_thread(Func f)
{
std::packaged_task<void()> task(f);
std::future<void> res=task.get_future();
std::lock_guard<std::mutex> lk(m);
tasks.push_back(std::move(task));
return res;
}
```
- the GUI thread loops until a message has been received
telling the GUI to shutdown , repeatedly polling for GUI messages to handle  , such as user clicks, and for tasks on the task queue.
- If there are no tasks on the queue  , it loops again; otherwise, it extracts the task from the queue, releases the lock on the queue, and then runs the task  . The future associated with the task will then be made ready when the task completes.
### Waiting from multiple threads
- std::future handles all the synchronization necessary to transfer data from one thread to another
- If you access a single std::future object from multiple threads without additional synchronization, you have a data race and undefined behavior.
### Waiting with a time limit
- suspending the thread until the event being waited for occurs.
- There are two sorts of timeouts you may wish to specify: a duration-based timeout where you wait for a specific amount of time
- for example, std::condition_variable has two overloads of the wait_for() member function and two overloads of the wait_until() member function that correspond to the two overloads of wait() —one overload that just waits until signaled
### Clocks
- a clock is a source of time informa-
tion. In particular, a clock is a class that provides four distinct pieces of information:The time now - The type of the value used to represent the times obtained from the clock - The tick period of the clock - Whether or not the clock ticks at a uniform rate and is thus considered to be a steady clock
- The current time of a clock can be obtained by calling the static member function now() for that clock class; for example, std::chrono::system_clock::now()
```
int main() {
  auto lol = std::chrono::system_clock::now();
  std::time_t end_time = std::chrono::system_clock::to_time_t(lol);
  std::cout << std::ctime(&end_time) << std::endl;
  return 0;
}
```
![Screenshot from 2024-07-29 10-21-04](https://github.com/user-attachments/assets/d7f89713-3d0a-4595-8288-b54281abaa5f)

### Durations
- Durations handled by the std:: chrono::duration<> class template


### Synchronizing operations with message passing
- if there’s no shared data, each thread can be reasoned about entirely independently, purely on the basis of how it behaves in response to the messages that it received.
  ![Screenshot from 2024-07-29 10-27-33](https://github.com/user-attachments/assets/141a7bea-f103-48a4-a714-47f8e33350b1)
