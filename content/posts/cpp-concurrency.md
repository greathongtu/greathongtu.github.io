+++
title = 'Cpp Concurrency'
date = 2023-12-08T14:52:36+08:00
draft = true
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


## Sharing Data
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
