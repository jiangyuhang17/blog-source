---
title: C++多线程并发系列（二）
subtitle: C++ Concurrency in Action 2nd Edition 读书笔记
date: 2020-03-22 14:35:05
categories: C++
tags: [C++, 并发控制]
---

# 同步对象的类型

## Locks

Lock（锁）是一个抽象概念，锁保护了对某种共享资源的访问，只有拥有锁，才能访问受保护的共享资源。要拥有锁，首先需要某个 lockable 的对象，然后从该对象中获得锁。

锁的概念还有着某种 exclusion（排斥），有时可能无法获得锁的所有权，这个操作会失败（返回一些错误代码或异常，以表明获取锁的尝试失败）或阻塞（直到拥有锁才能返回）。

## Mutexes

Mutex（互斥量，Mutual Exclusion 的缩写），如果不加术语限定，它就是一种 lockable 对象类型，一次只能由一个线程拥有它的锁，只有获得锁的线程才能释放 mutex 上的锁。当 mutex 被锁时，任何获取锁的尝试都将失败或阻塞，即使该尝试是由同一线程发出的。

### Recursive Mutexes

Recursive mutex（嵌套互斥量）类似于 mutex，但是一个线程可以同时拥有多个它的锁。线程A可以在不释放已持有的锁的情况下，获取 recursive mutex 上更多的锁；但是线程B不能获得 recursive mutex 上的任何锁，直到线程A持有的所有锁都已释放。

在大多数情况下，recursive mutex 是不可取的，因为它使得代码难以理解。

### Reader/Writer Mutexes

Reader/Writer Mutex（读写互斥量），它们提供两种不同的所有权类型：

* exclusive ownership（独占所有权 /写所有权），工作方式类似于普通 mutex 的所有权

* shared ownerhip（共享所有权 / 读所有权），共享所有权更宽松，任意数量的线程可以同时获得 mutex 的共享所有权。当任何线程持有共享锁时，任何线程都不能在 mutex 上获得排他锁。

读写互斥量通常用于保护很少更新的共享数据。读取线程在读取数据时将获得共享所有权；当需要修改数据时，修改线程首先获取 mutex 的独占所有权，从而确保没有其他线程正在读取数据，然后在修改完成后释放互斥量的锁。

### Spinlocks

Spinlock（自旋锁）是一种特殊的 mutex，它在必须等待锁时，并不调用操作系统的同步函数，而是会不断循环尝试去获得 mutex 的锁。

如果锁不是经常被持有，或者仅持有很短的时间，那么自旋锁的做法比调用重量级线程同步函数更加有效。

## Semaphores

Semaphore（信号量）是一个非常宽松的 lockable 对象。给定的信号量具有预定义的最大计数和当前计数，通过P操作（减少信号量）来获得信号量的所有权；通过V操作（增加信号量）来释放信号量的所有权。

P操作会减少当前计数，如果计数已经为零，无法减少时，将等线程阻塞，直到另一个线程通过V操作来增加计数。

V操作会增加当前计数，如果在调用之前计数为零，并且有一个线程在等待中被阻塞，则该线程将被唤醒；如果有多个线程在等待，则只会唤醒一个；如果该计数已经达到最大值，则通常会忽略该信号，尽管某些信号量可能会报告错误。

mutex 的所有权和一个线程紧密相关，并且只有获得了 mutex 锁的线程才能释放它；但 semaphore 所有权却更为宽松和短暂，任何线程都可以随时向该semaphore发出信号，无论该线程先前是否在等待 semaphore。

**C++ 标准将在 C++ 20 引入 semaphore 类型**

## Critial Section

Critical Section（临界区）是拥有锁的那段代码，它从获取锁的点开始，到释放锁的点结束。

# 使用 mutexes 保护共享数据

## 条件竞争

`invariant`（不变量）是对于特定数据结构而言始终正确的陈述。修改线程间共享的数据最简单的潜在问题是不变量被破坏，这些不变量通常在更新期间被破坏。

`race conditon`（竞争条件）是指并发的结果取决于在两个或多个线程上执行操作的相对顺序，错误的执行顺序可能会导致不变量的破坏等问题。`data race`（数据竞争）是竞争条件的一个具体类型，即由于同时修改单个对象而产生的竞争条件。数据竞争会导致可怕的未定义行为。

## C++ mutexes && lock 对象

C++中通过实例化`std::mutex`创建互斥量实例，通过成员函数`lock()`对互斥量上锁，`unlock()`进行解锁；但是**不推荐**直接去调用成员函数，调用成员函数就意味着，必须在每个函数出口都要去调用unlock()，也包括异常的情况。

C++标准库为 mutex 提供了一个RAII语法的类模板`std::lock_guard`，在构造时就能锁住互斥量，并在析构的时候进行解锁，从而保证了一个已锁的互斥量能被正确解锁。

``` C++
// 使用 mutex 保护 list
#include <list>
#include <mutex>
#include <algorithm>

std::list<int> some_list;
std::mutex some_mutex;

void add_to_list(int new_value)
{
  std::lock_guard<std::mutex> guard(some_mutex);
  some_list.push_back(new_value);
}

bool list_contains(int value_to_find) {
  std::lock_guard<std::mutex> guard(some_mutex);
  return std::find(some_list.begin(),some_list.end(),value_to_find) != some_list.end(); 
}
```

C++ 17中添加了一个新特性，模板类型参数推导，类似`std::lock_guard`这样简单的模板类型的模板参数列表可以省略。

``` C++
std::lock_guard guard(some_mutex1);
```

C++ 17共有六种 mutex 类型：

* `std::mutex`
* `std::timed_mutex`
* `std::recursive_mutex`
* `std::recursive_timed_mutex`
* `std::shared_mutex`
* `std::shared_timed_mutex`

名称中带有"timed"的变量与没有带的变量基本相同，除了lock操作可以指定超时以限制最大等待时间，且带有"timed"的变量会损失一些性能。其他的 mutex 会在之后介绍。

C++ 17共有三种 lock 的RAII类模版：

* `std::scoped_lock`（替代了`std::lock_guard`）
* `std::unique_lock`
* `std::shared_lock`

为了与各种互斥量类型配合使用，C++标准定义了三个RAII类模板来提供可以持有互斥量锁的对象。它们都可以在构造函数中获取锁，然后在析构函数中释放该锁，但是如果有需要可以通过更复杂的方式使用它们。

## 避免死锁

`std::lock`可以一次性锁住多个互斥量，并且没有副作用(使用避免死锁算法来避免死锁)。（`std::scoped_lock`提供此函数的 RAII 包装，下面会介绍）

`std::lock`对象通过一系列隐式调用来进行`lock()`、`try_lock()`和`unlock()`操作；如果调用`lock()`或`unlock()`导致异常，则在重新抛出之前，对所有被锁的对象调用`unlock()`。

``` C++
#include <mutex>

class some_big_object;
void swap(some_big_object& lhs,some_big_object& rhs);
class X
{
private:
  some_big_object some_detail;
  std::mutex m;
public:
  X(some_big_object const& sd):some_detail(sd){}

  // 类X的友元函数，定义在类的外部，但有权访问类的所有成员
  friend void swap(X& lhs, X& rhs)
  {
    if(&lhs==&rhs)
      return;
    std::lock(lhs.m,rhs.m);
    std::lock_guard<std::mutex> lock_a(lhs.m,std::adopt_lock);
    std::lock_guard<std::mutex> lock_b(rhs.m,std::adopt_lock);
    swap(lhs.some_detail,rhs.some_detail);
  }
};
```

`std::lock_guard`的第二个参数`std::adopt_lock`表示mutex已经被锁住，以及应该使用 mutex 现有锁的 ownership，而不需要在构造器里锁住 mutex。

C++ 17对这种情况提供了支持，`std::scoped_lock`是一种新的RAII类型模板，与`std::lock_guard`的功能等价，这个新类型能接受不定数量的互斥量类型作为模板参数，以及相应的互斥量(数量和类型)作为构造参数。互斥量支持构造即上锁，与`std::lock`的用法相同，其解锁阶段是在析构中进行。`std::scoped_lock`的好处在于，可以将所有`std::lock`替换掉，从而减少潜在错误的发生。

``` C++
// std::scoped_lock 替代 std::lock + std::lock_guard
friend void swap(X& lhs, X& rhs)
{
  if(&lhs==&rhs)
    return;
  std::scoped_lock(lhs.m,rhs.m);
  swap(lhs.some_detail,rhs.some_detail);
}
```

## 更灵活的锁 std::unique_lock

`std::unique_lock`比`std::lock_guard`更灵活，它不必总是拥有与其相关联的互斥量。

可将`std::adopt_lock`作为第二个参数传入构造函数，对互斥量进行管理；也可以将`std::defer_lock`作为第二个参数传递进去，表明互斥量应保持**解锁状态**，这样互斥量的锁就可以被`std::unique_lock`对象的`lock()`函数所获取，或传递`std::unique_lock`对象到std::lock()中。

允许`std::unique_lock`实例不总是拥有互斥量的代价就是是否拥有互斥量的标志被存储，且会被更新。所以`std::unique_lock`会占用比较多的空间，并且比`std::lock_guard`稍慢一些。

``` C++
// std::unique_lock + std::lock 替代 std::lock + std::lock_guard
friend void swap(X& lhs, X& rhs)
{
  if(&lhs==&rhs)
    return;
  std::unique_lock<std::mutex> lock_a(lhs.m,std::defer_lock); 
  std::unique_lock<std::mutex> lock_b(rhs.m,std::defer_lock);
  std::lock(lock_a,lock_b);
  swap(lhs.some_detail,rhs.some_detail);
}
```

由于`std::unique_lock`支持`lock()`，`try_lock()`和`unlock()`成员函数，所以能将`std::unique_lock`对象传递到`std::lock()`。

除非你想将`std::unique_lock`的所有权进行转让，或是对其做一些其他的事情外，你最好使用C++ 17中提供的`std::scoped_lock`。

### 不同域中互斥量所有权的传递

因为`std::unique_lock`实例不必与自身相关的互斥量，一个互斥量的所有权可以通过移动操作，在不同的实例中进行传递。`std::unique_lock`是可移动，但不可赋值的类型。

某些情况下，这种转移是自动发生的，例如：当函数返回一个实例（右值）；另一些情况下，需要显式的调用`std::move()`来执行移动操作（左值）。从本质上来说，需要依赖于源值是否是左值（一个实际的值或是引用，有变量名）或一个右值（一个临时类型，无变量名）。

一种使用可能是允许一个函数去锁住一个互斥量，并且将所有权移到调用者上，所以调用者可以在这个锁保护的范围内执行额外的动作。

``` C++
std::unique_lock<std::mutex> get_lock()
{
  extern std::mutex some_mutex;
  std::unique_lock<std::mutex> lk(some_mutex);
  prepare_data();
  return lk;
}
void process_data()
{
  std::unique_lock<std::mutex> lk(get_lock());
  do_something();
}
```

### 锁的合理粒度

锁的粒度是用来描述单个锁保护的数据量大小；选择足够粗的加锁粒度以确保保护所需的数据很重要，而确保仅针对需要加锁的操作才持有锁也很重要。

特别要注意的是，请勿在持有锁时执行任何耗时的活动，例如文件IO。除非该锁旨在保护对文件的访问，否则在持有该锁的同时执行IO会不必要地延迟其他线程（因为它们将在等待获取锁时阻塞），从而可能消除使用多个线程带来的任何性能提升。

`std::unique_lock`在这种情况下效果很好，因为您可以在代码不再需要访问共享数据时调用`unlock()`，如果以后需要访问代码，则可以再次调用`lock()`

``` C++
void get_and_process_data() { 
  std::unique_lock<std::mutex> my_lock(the_mutex); 
  some_class data_to_process=get_next_data_chunk(); 
  my_lock.unlock(); 
  result_type result=process(data_to_process); 
  my_lock.lock(); 
  write_result(data_to_process,result); 
}
```

假设您正在尝试比较一个简单的数据成员，该成员是一个普通的int。由于int的复制成本很低，因此您可以轻松地复制每个要比较的对象的数据，而只需持有该对象的锁，然后比较复制的值。这意味着您将每个互斥量上的锁保持最短的时间，并且您没有在锁定另一个互斥量的同时持有该锁。

``` C++
// 在比较运算符中，一次只锁住一个互斥量
class Y { 
private:
  int some_detail; 
  mutable std::mutex m; 
  int get_detail() const {
    std::lock_guard<std::mutex> lock_a(m);
    return some_detail; 
  }
public:
  Y(int sd): some_detail(sd) {} 
  friend bool operator==(Y const& lhs, Y const& rhs) { 
    if(&lhs==&rhs) 
      return true; 
    int const lhs_value=lhs.get_detail(); 
    int const rhs_value=rhs.get_detail(); 
    return lhs_value==rhs_value; 
  }
};
```

## 保护共享数据的初始化过程

假设你有一个共享源，构建代价很昂贵（它可能会打开一个数据库连接或分配出很多的内存）。在多线程的情况下，我们要保证共享源只被初始化一次，C++ 标准库提供了`std::once_flag`和`std::call_once`来处理这种情况。

比起锁住互斥量并显式的检查共享源，每个线程只需要使用`std::call_once`就可以，在`std::call_once`的结束时，就能安全的知道指针已经被其他的线程初始化了。使用`std::call_once`比显式使用互斥量消耗的资源更少，特别是当初始化完成后。

``` C++
std::shared_ptr<some_resource> resource_ptr;
std::once_flag resource_flag;

void init_resource()
{
  resource_ptr.reset(new some_resource);
}

void foo()
{
  std::call_once(resource_flag, init_resource);  // 可以完整的进行一次初始化
  resource_ptr->do_something();
}
```

在类中使用`std::call_once`，注意`std::call_once(flag, &X::func, this);`这样的写法。

``` C++
class X { 
private:
  connection_info connection_details; 
  connection_handle connection; 
  std::once_flag connection_init_flag; 
  void open_connection() { 
    connection=connection_manager.open(connection_details); 
  } 
public:
  X(connection_info const& connection_details_): connection_details(connection_details_) {} 
  void send_data(data_packet const& data) { 
    std::call_once(connection_init_flag,&X::open_connection,this); 
    connection.send_data(data); 
  } 
  data_packet receive_data() { 
    std::call_once(connection_init_flag,&X::open_connection,this); 
    return connection.receive_data(); 
  }
};
```

在C++ 11中，`static变量的初始化是线程安全的`，所以当只有一个全局实例时（单例模式），可以不使用`std::call_once`而直接用static变量。（C++ 11规定static变量的初始化只发生在一个线程中，直到初始化完成前其他线程都不会做处理，从而避免了条件竞争。）

``` C++
class my_class; 

my_class& get_my_class_instance() {
  static my_class instance;
  return instance; 
}
```

## std::shared_mutex

对于不常更新的数据结构，使用`std::mutex`会影响并发地读取数据，这里需要另一种不同的互斥量，这种互斥量常被称为 reader-writer mutex，因为其允许两种不同的使用方式：一个"writer"线程独占访问，让多个"reader"线程共享并发访问。C++17标准库提供了两种非常好的互斥量，`std::shared_mutex`和`std::shared_timed_mutex`，后者比前者提供了更多操作，但前者性能更高。

对于更新操作，可以使用`std::lock_guard<std::shared_mutex>`和`std::unique_lock<std::shared_mutex>`上锁，与`std::mutex`所做的一样，这就能保证更新线程的独占访问。

对于其他不需要去修改数据结构的线程，其可以使用`std::shared_lock<std::shared_mutex>`获取共享访问。这种RAII类型模板`std::shared_lock`是在C++ 14中的新特性，这与使用`std::unique_lock`一样，除了多线程可以同时获取同一个`std::shared_mutex`上的共享锁之外。

``` C++
// 实现了简单的DNS缓存类
#include <map>
#include <string>
#include <mutex>
#include <shared_mutex>

class dns_entry;

class dns_cache
{
  std::map<std::string,dns_entry> entries;
  mutable std::shared_mutex entry_mutex;
public:
  dns_entry find_entry(std::string const& domain) const
  {
    std::shared_lock<std::shared_mutex> lk(entry_mutex);
    std::map<std::string,dns_entry>::const_iterator const it=
       entries.find(domain);
    return (it==entries.end())?dns_entry():it->second;
  }
  void update_or_add_entry(std::string const& domain,
                           dns_entry const& dns_details)
  {
    std::lock_guard<std::shared_mutex> lk(entry_mutex);
    entries[domain]=dns_details;
  }
};
```

在类中将成员函数修饰为const表明在该函数体内，不能修改对象的数据成员而且不能调用非const函数。`mutable`修饰词是为了突破const函数的限制而设置的。

## std::recursive_mutex

C++标准库提供了`std::recursive_mutex`类以支持嵌套互斥量。除了可以对同一线程的单个实例上获取多个锁，其他功能与`std::mutex`相同。互斥量锁住其他线程前，必须释放拥有的所有锁，所以当调用`lock()`三次后，也必须调用`unlock()`三次。