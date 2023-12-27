+++
title = 'Cpp Concurrency'
date = 2023-12-08T14:52:36+08:00
draft = false
+++

```cpp
#include <iostream>
#include <thread>
void hello() { std::cout << "Hello Concurrent World\n"; }
int main() {
  std::cout << "first main" << std::endl;
  std::thread t(hello);
  std::cout << "from main thread!" << std::endl;
  // t.join();
  t.detach();
}
```

等待：join

不等：detach 如后台监控线程

精细化控制： condition variables and futures

join之前发生exception怎么办呢：try catch. 更好的方式：RAII

```cpp
class thread_guard {
  std::thread &t;

public:
  explicit thread_guard(std::thread &t_) : t(std::move(t_)) {
    if (!t.joinable()) {
      throw std::logic_error(“No thread”);
    }
  }
  ~thread_guard() {
    t.join(); 
  }
  thread_guard(thread_guard const &) = delete;
  thread_guard &operator=(thread_guard const &) = delete;
};

// usage
std::thread t(my_func);
thread_guard g(t);
```
向thread对象传参的时候要注意隐式类型转换：
```cpp
char buffer[1024];
// dangling pointer alert!
std::thread t(f,3,buffer);
// Ok
std::thread t(f,3,std::string(buffer));
```

thread对象的function的参数要引用的情况：
```cpp
// widget_data 需要引用
void update_data_for_widget(widget_id w, widget_data &data);
void oops_again(widget_id w) {
  widget_data data;
  // 这里先拷贝w 和 data，然后传rvalues 给需要non-const reference的function
  // compile error!
  std::thread t(update_data_for_widget, w, data);
  // 解决办法：
  std::thread t(update_data_for_widget, w, std::ref(data));
  display_status();
  t.join();
  process_widget_data(data);
}
```
std::thread像std::bind一样
```cpp
class X {
public:
  void do_lengthy_work();
};
X my_x;
// 第三个参数开始就是成员函数的参数
std::thread t(&X::do_lengthy_work, &my_x);
```
可以把unique_ptr move进参数。thread对象本身也是可以move不可以copy的

```cpp
std::thread t1(some_function);
std::thread t2 = std::thread(some_other_function);
// terminate! you must explicitly wait for a thread to complete or detach it before destruction,
t1 = std::move(t2);
```

用std::vector管理std::thread 可以在runtime创建动态数量的thread

thread id : std::thread::get_id() 或者 std::this_thread::get_id()


## Sharing Data with mutex
If all shared data is read-only, there’s no problem。

std::mutex -> std::lock_guard -> std::scoped_lock

Don’t pass pointers and references to protected data outside the scope of the lock, 

### 避免死锁的方法：
* lock the two mutexes in a fixed order
* std::lock 同时锁多个mutex 而不会死锁
```cpp
void swap(X &lhs, X &rhs) {
  if (&lhs == &rhs)
    return;
  std::lock(lhs.m, rhs.m);
  // 注意 adopt_lock
  std::lock_guard<std::mutex> lock_a(lhs.m, std::adopt_lock);
  std::lock_guard<std::mutex> lock_b(rhs.m, std::adopt_lock);
  swap(lhs.some_detail, rhs.some_detail);
}
```
用 scoped_lock 简化：
```cpp
void swap(X &lhs, X &rhs) {
  if (&lhs == &rhs)
    return;
  std::scoped_lock guard(lhs.m, rhs.m);
  swap(lhs.some_detail, rhs.some_detail);
}
```

* don’t wait for another thread if there’s a chance it’s waiting for you
* AVOID NESTED LOCKS 已经有一个锁的时候就不要再acquire lock
* AVOID CALLING USER-SUPPLIED CODE WHILE HOLDING A LOCK
* USE A LOCK HIERARCHY: 从上层锁到下层

### 比lock_guard更灵活的 std::unique_lock
using std::unique_lock and std::defer_lock B, rather than std::lock_guard and std::adopt_lock. 更灵活但更耗空间，更慢。最好还是用scoped_lock。
```cpp
void swap(X &lhs, X &rhs) {
  if (&lhs == &rhs)
    return;
  std::unique_lock<std::mutex> lock_a(lhs.m, std::defer_lock);
  std::unique_lock<std::mutex> lock_b(rhs.m, std::defer_lock);
  std::lock(lock_a, lock_b);
  swap(lhs.some_detail, rhs.some_detail);
}
```

### Transferring mutex ownership between scopes
因为std::unique_lock不拥有mutex，mutex的ownership可以在intance之间move

### Locking at an appropriate granularity
```cpp
void get_and_process_data() {
  std::unique_lock<std::mutex> my_lock(the_mutex);
  some_class data_to_process = get_next_data_chunk();
  my_lock.unlock();
  result_type result = process(data_to_process);
  my_lock.lock();
  write_result(data_to_process, result);
}
```
appropriate granularity不仅指data的数量，还指锁的时长
```cpp
class Y {
private:
  int some_detail;
  mutable std::mutex m;
  int get_detail() const {
    std::lock_guard<std::mutex> lock_a(m);
    return some_detail;
  }

public:
  Y(int sd) : some_detail(sd) {}
  friend bool operator==(Y const &lhs, Y const &rhs) {
    if (&lhs == &rhs)
      return true;
    int const lhs_value = lhs.get_detail();
    int const rhs_value = rhs.get_detail();
    return lhs_value == rhs_value;
  }
};
```
if you don’t hold the required locks for the entire duration of an operation, you’re exposing yourself to race conditions.上面相等只能说明lhs在某一时间跟rhs在另外一个时间是相等。

## 除mutex 别的保护shared data的方法
One common case is where the shared data needs protection only from concurrent access while it’s being initialized, but after that no explicit synchronization is required(比如data创建之后就是read only的了)

### Protecting shared data during initialization
* 单线程的时候可以Lazy initialization：看一下没创建的话再去创建。

* naive加锁的多线程方法unnecessary serialization，性能太差

* double checked locking 会race：
```cpp
void undefined_behaviour_with_double_checked_locking() {
  if (!resource_ptr) { // read
    std::lock_guard<std::mutex> lk(resource_mutex);
    if (!resource_ptr) {
      resource_ptr.reset(new some_resource); // write
    }
  }
  resource_ptr->do_something();
}
```
read 和 write 没有同步，会race。
even if a thread sees the pointer written by another thread, it might not see the newly created instance of some_resource, resulting in the call to do_something() e operating on incorrect values.

* std::once_flag 和 std::call_once
```cpp
std::shared_ptr<some_resource> resource_ptr;
std::once_flag resource_flag;
void init_resource() { resource_ptr.reset(new some_resource); }
void foo() {
  // the pointer will have been initialized by some thread (synchronizly) by the time std::call_once returns
  // synchronization data 存在 once_flag 里
  std::call_once(resource_flag, init_resource);
  resource_ptr->do_something();
}
```

除了通过function的方式，还可以被用在lazy initialization of class
members
```cpp
class X {
private:
  connection_info connection_details;
  connection_handle connection;
  std::once_flag connection_init_flag;
  void open_connection() {
    connection = connection_manager.open(connection_details);
  }

public:
  X(connection_info const &connection_details_)
      : connection_details(connection_details_) {}
  void send_data(data_packet const &data) {
    std::call_once(connection_init_flag, &X::open_connection, this);
    connection.send_data(data);
  }
  data_packet receive_data() {
    std::call_once(connection_init_flag, &X::open_connection, this);
    return connection.receive_data();
  }
};
```
上面的例子中 send_data() 和 receive_data() 两者中更先被call的完成init.

像mutex一样，once_flag也不能被copy、move

* local static variables的初始化occur the first time control passes through its declaration。多线程环境下这就会race,c++11之后的compiler就没问题了。所以可以通过这种方式替代call_once() where a single global instance is required

### Protecting rarely updated data structures
比如DNS信息：读写锁：std::shared_mutex 和 std::shared_timed_mutex.

通过std::shared_lock<std::shared_mutex> 获得shared access；
通过std::lock_guard<std::shared_mutex> 或者std::unique_lock<std::shared_mutex> 获得 exclusive access

(mutable 关键词：在const 函数中也可以修改mutable变量，所以适合用在mutex上，因为mutex要lock、unlock)

### Recursive locking
重复锁同一个mutex是UB.

std::recursive_mutex：可重复锁，（但要记得锁了几次最后还得解锁几次，通过RAII就行）

**Most of the time, if you think you want a recursive mutex, you probably need to change your design instead.**

common use: 每一个public成员函数都lock了，一个public成员函数会call 另一个.
**But such usage is not recommended** because it can lead to sloppy thinking and bad design. The class invariants are typically broken.

Better to extract a new private member function which does not lock the mutex


# Synchronizing concurrent operations

上面讲的是保护data, 这里讲同步action：condition variables 和 futures。还有latches and barriers
* Waiting for an event
* Waiting for one-off events with futures
* Waiting with a time limit
* Using the synchronization of operations to simplify code

## condition_variables
为什么要条件变量：等event发生，要么不停地检查flag浪费资源；要么std::this_thread::sleep_for() 但是不好确定到底睡多久。
```cpp
std::mutex mut;
std::queue<data_chunk> data_queue;
std::condition_variable data_cond;
void data_preparation_thread() {
  while (more_data_to_prepare()) {
    data_chunk const data = prepare_data();
    {
      std::lock_guard<std::mutex> lk(mut);
      data_queue.push(data);
    }
    data_cond.notify_one();
  }
}
void data_processing_thread() {
  while (true) {
    std::unique_lock<std::mutex> lk(mut);
    data_cond.wait(lk, [] { return !data_queue.empty(); });
    data_chunk data = data_queue.front();
    data_queue.pop();
    lk.unlock();
    process(data);
    if (is_last_chunk(data))
      break;
  }
}
```
上面用unique_lock 搭配conditon_variable 是因为the waiting thread must unlock the mutex while it’s waiting and lock it again afterward, and std::lock_guard doesn’t provide that flexibility

## futures
std::future<> 和 std::shared_future<>

### Returning values from background tasks
use **std::async** to start an asynchronous task，returns a std::future object。 call get() on the future to block until returning the value.
```cpp
std::future<int> the_answer = std::async(find_the_answer_to_ltuae);
do_other_stuff();
std::cout << "The answer is " << the_answer.get() << std::endl;
```

```cpp
struct X {
  void foo(int, std::string const &);
  std::string bar(std::string const &);
};
X x;
// Calls p->foo(42,"hello") where p is &x
auto f1 = std::async(&X::foo, &x, 42, "hello");
// Calls tmpx.bar("goodbye") where tmpx is a copy of x
auto f2 = std::async(&X::bar, x, "goodbye");
struct Y {
  double operator()(double);
};
```

an additional parameter to std::async before the function to call: std::launch, 要么是std::launch::deferred 等到wait() 或者get()被call的时候才调用；要么是std::launch::async 开一个新线程。或者std::launch::deferred | std::launch::async 代表由实现自己选择(默认)。

### Associating a task with a future
wrapping the task in an instance of the **std::packaged_task<>** class template or by writing code to explicitly set the values using the **std::promise<>** class template.

当std::packaged_task<> 对象被invoke的时候，它call对应的function or callable object，让future ready. Can be used as a building block for thread pools

```cpp
std::mutex m;
std::deque<std::packaged_task<void()>> tasks;
bool gui_shutdown_message_received();
void get_and_process_gui_message();
void gui_thread() {
  while (!gui_shutdown_message_received()) {
    get_and_process_gui_message();
    std::packaged_task<void()> task;
    {
      std::lock_guard<std::mutex> lk(m);
      if (tasks.empty())
        continue;
      task = std::move(tasks.front());
      tasks.pop_front();
    }
    task();
  }
}
std::thread gui_bg_thread(gui_thread);
template <typename Func> std::future<void> post_task_for_gui_thread(Func f) {
  std::packaged_task<void()> task(f);
  std::future<void> res = task.get_future();
  std::lock_guard<std::mutex> lk(m);
  tasks.push_back(std::move(task));
  return res;
}
```
std::packaged_task wraps a function or other callable object

### promises
tasks that can’t be expressed as a simple function call or those tasks where the result may come from more than one place, 第三种创建future的方式：std::promise

std::promise/std::future pair：通过std::promise的get_future()成员函数获得std::future object。std::promise<T> 通过set_value()设置value让 std::future<T> object ready,future object就可以读到。 

如果destroy the std::promise without setting a value： exception

```cpp
void process_connections(connection_set &connections) {
  while (!done(connections)) {
    for (connection_iterator connection = connections.begin(),
                             end = connections.end();
         connection != end; ++connection) {
      if (connection->has_incoming_data()) {
        data_packet data = connection->incoming();
        std::promise<payload_type> &p = connection->get_promise(data.id);
        p.set_value(data.payload);
      }
      if (connection->has_outgoing_data()) {
        outgoing_packet data = connection->top_of_outgoing_queue();
        connection->send(data.payload);
        data.promise.set_value(true);
      }
    }
  }
}
```

### Saving an exception for the future
future.get() 会返回异常而不是value 当发生异常的时候
```cpp
extern std::promise<double> some_promise;
try {
  some_promise.set_value(calculate_value());
} catch (...) {
  some_promise.set_exception(std::current_exception());
}
// 如果知道type of the exception，最好用这个而不是try catch
some_promise.set_exception(std::make_exception_ptr(std::logic_error("foo ")));
```

### Waiting from multiple threads
用std::shared_future, multiple threads can wait for the same event
```cpp
// implicit transfer of ownership
std::shared_future<std::string> sf(some_promise.get_future());
// share() 方法将future变为shared_future
auto sf = p.get_future().share();
```


## Waiting with a time limit



# memory order and atomic
memory order是线程之间进行顺序绑定用的。

release sequence 指一个release 可以和后面的很多个acquire 同步

如果acquire操作看见release fence后面的store的结果，那fence就和acquire同步；如果acquire fence后面的load看见release操作的结果，那release和fence同步。

Release fence可以防止fence前的内存操作重排到fence后的任意store之后，即阻止loadstore重排和storestore重排。

acquire fence可以防止fence后的内存操作重排到fence前的任意load之前，即阻止loadload重排和loadstore重排

std::atomic_thread_fence(memory_order_acq_rel)和std::atomic_thread_fence(memory_order_seq_cst)都是full fence。

基于atomic_thread_fence（外加一个任意序的原子变量操作）的同步和基于原子操作的同步很类似，比如最常用的，都可以形成release acquire语义，但是从上面的描述可以看出，fence的效果要比基于原子变量的效果更强，在weak memory order平台的开销也更大。

以release为例，对于基于原子变量的release opration，仅仅是阻止前面的内存操作重排到该release opration之后，而release fence则是阻止重排到fence之后的任意store operation之后。

因为fence的同步效果和原子操作上的同步效果比较相似，可以互相组合，自然的，使用fence的同步会有三种情况， fence - atomic同步，fence - fence同步和atomic-fence -同步。

If a non-atomic operation is sequenced before an atomic operation, and that atomic operation happens before an operation in another thread, the non-atomic operation also happens before that operation in the other thread. 参考代码：5.3.6的 listing5.13
所以mutex的实现的基础就是这个：lock（）是acquire，unlock是release，所以unlock之前的操作在unlock之前，也就在下一个thread的lock之前。

原子变量实现自旋锁：https://blog.51cto.com/quantfabric/2581173 相比mutex，是busy waiting而不是sleep waiting，不会使线程阻塞；少了用户态内核态的切换。

无锁编程性能并不一定更好，比如cache受影响。

原子变量的底层实现可能会使用mutex。
