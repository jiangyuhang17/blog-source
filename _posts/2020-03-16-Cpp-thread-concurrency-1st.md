---
title: C++多线程并发系列（一）
subtitle: C++ Concurrency in Action 2nd Edition 读书笔记
date: 2020-03-16 23:22:08
categories: C++
tags: [C++, 并发控制]
---

# 一些概念

C++标准本身不支持IPC，所以应用需要依赖具体平台的API来实现IPC。本文不涉及IPC，想要了解IPC这方面内容可以参考特定操作系统的实现，比如[《UNIX》网络编程 卷二](https://book.douban.com/subject/4118577/)。

## 并行 & 并发：

一般解释都是，并发是指宏观上在一段时间内能同时运行多个程序，而并行则指同一时刻能运行多个指令。下面给出一个从目的角度出发的解释：

并行的主要目的是在硬件支持的基础上，来提升处理大量数据的性能，用来解决一个单一的任务。

并发的主要目的是 1. 将无关的代码分离，使程序更容易理解和测试 2. 提高任务响应速度。

## 任务并行 & 数据并行

任务并行：将单个任务划分成多个任务，同时运行，减少整体运行时间，任务之间可能存在许多依赖关系。

数据并行：将数据划分成多个不同的部分，每个线程同时对不同的数据部分执行相同的操作。

# 线程管理

线程使得程序能在数个处理器核心同时执行，定义于头文件`<thread>`

`<thread>`主要内容：定义了一个类`std::thread`；定义了命名空间`std::this_thread`及其内部的辅助函数。更多内容详见 [cppreference](https://zh.cppreference.com/w/cpp/header/thread)。

## 线程基础

### 启动线程

`std::thread` 构造函数

``` C++
std::thread my_thread1(func_name); // 将函数名作为参数
std::thread my_thread2(func_obj); // 将函数对象作为参数
```

注意：当将`函数对象`作为参数传递时，如果传递了一个临时变量，而不是一个命名的变量，那么C++编译器会将其解析为函数声明，而不是类型对象的定义。

``` C++
std::thread my_thread(class_name()); // 将类的临时变量作为参数，会被错误解析成函数声明

// 解决方法
std::thread my_thread1((class_name())); //多加一层括号
std::thread my_thread2 {class_name()}; //使用新统一的初始化语法
```

非常推荐使用上面的第二种解决方法，C++11 的`花括号初始化`语法（花括号初始化不会允许 narrowing conversions，即会丢失信息的转换，例如int转char），下面的例子都将用 {} 初始化。

使用`lambda表达式`也能避免这个问题。lambda表达式是 C++11 的一个新特性，它使用一个可以捕获局部变量的局部函数来生成一个函数对象。

**注意：`std::thread`的对象是线程句柄，可能处于不表示任何线程的状态（默认构造、被移动、join 或 detach 后），并且执行线程可能与任何 thread 对象无关（detach 后）**

``` C++
std::thread my_thread1; // 默认构造，不表示任何线程

std::thread my_thread2 {func_name};
std::thread my_thread3 {my_thread2}; // my_thread2 被移动给 my_thread3，不再表示任何线程
```

没有两个`std::thread`对象会表示同一执行线程,所以`std::thread`的对象是可移动的 (movable)，但不是可复制的 (copyable)。

### join & detach

在启动线程之后，需要明确是 `join`（等待线程结束）还是 `detach`（分离线程与`std::thread`对象）；如果在`std::thread`对象被销毁之前还没有做出决定，程序会被终止（`std::thread`的析构函数会调用`std::terminate()`）。因此，要保证即使有异常存在，也需要确保线程能够正确的 join 或 detech。

#### join

调用join()的行为，清理了线程相关的存储部分，这样std::thread对象将不再与已经完成的线程有任何关联。这意味着，只能对一个线程使用一次join()；一旦已经使用过join()，std::thread对象就不能再次join了，当对其使用joinable()时，将返回false。

考虑到对线程使用join()之前，调用的函数可能会产生异常而终止，有两种方式可以保证join()在调用函数退出前被执行：

* `try / catch`语句块捕获异常；

* 使用`RAII`(在构造函数内获取资源，在析构函数内释放资源)，提供一个类，在析构函数中使用join()。

#### detach

使用detach()会让线程在后台运行，这就意味着主线程不能与之产生直接交互。C++运行库保证，当线程退出时，相关资源的能够正确回收，后台线程的归属和控制由C++运行库来处理。通常称 detach 后的线程为`守护线程`(daemon threads)。

如果要 detach 线程，就必须保证线程结束之前，可访问的数据的有效性，避免产生`未定义行为`。这种情况很可能发生在线程还没结束，函数已经退出的时候，这时线程函数还持有函数局部变量的指针或引用。

## 向线程函数传参

``` C++
void func(int i, std::string const& s); 
std::thread t {func, 3, "hello"};
```

使用带有数据成员的函数对象类似于给函数传递参数。`std::string const&`和`const std::string&`没有区别。

但请务必记住，默认情况下，参数会复制到内部存储中，新创建的执行线程可以在其中访问它们，然后将它们作为右值传递给可调用对象或函数，就像它们是临时的一样。 即使函数中的相应参数需要引用，也会执行此操作。

#### 案例一

``` C++
void func(int i, std::string const& s);
void oops(int some_param)
{
  char buffer[1024];
  sprintf(buffer, "%i", some_param);
  std::thread t {func, 3, buffer}; // 可能导致未定义行为
  t.detach();
}
```

上述程序在传递参数时，将字符串的字面值（即buffer，char const * 类型）传给线程函数的第二个参数，之后，在线程的上下文中需要完成字面值向 std::string 对象的转化。如果函数 oops 在 buffer 转换成 std::string 对象之前结束，则会导致一些未定义行为。

解决方案是在传递到std::thread构造函数之前就将字面值转化为std::string对象。

``` C++
std::thread t {func, 3, std::string(buffer)};
```

#### 案例二

``` C++
void update_data(int id, std::string& data);

void oops(int id) {
    std::string data;
    std:thread t {update_data, id, data}; // 会导致编译失败
    t.join();
}
```

`std::thread`的构造函数无视线程函数期待的`非常量引用`参数类型，并盲目地拷贝已提供的变量，会导致编译失败。

解决方案是使用`std::ref`将参数转换成引用的形式。这和`std::bind`（预先把指定可调用对象的某些参数绑定到已有的变量，产生一个新的可调用对象，有点类似默认参数）一样，`std::thread`构造函数和`std::bind`的操作在标准库中以相同的机制进行定义.

```C++
std::thread t {update_data, id, std::ref(data)};
```

#### 案例三

可以传递一个成员函数指针作为线程函数，并提供一个合适的对象指针作为成员函数的第一个参数

``` C++
class X
{
public:
  void func();
};
X my_x;
std::thread t {&X::do_lengthy_work, &my_x};
```

#### 案例四

线程函数的参数只能支持移动，不支持拷贝的情况，使用`std::move`将对象的所有权转移到线程函数中。

``` C++
void process_data(std::unique_ptr<data>);

std::unique_ptr<data> p {new data};
std::thread t {process_data, std::move(p)};
```

## 运行时决定线程数量

`std::thread::hardware_concurrency()`这个函数返回能并发的线程数量，多核系统中，返回值可以是CPU核芯的数量。返回值也仅仅是一个提示，当系统信息无法获取时，函数也会返回0。

下面是一个并行版的求和算法

```  C++
template<typename Iterator,typename T>
struct accumulate_block
{
  void operator()(Iterator first,Iterator last,T& result)
  {
    result=std::accumulate(first,last,result);
  }
};

template<typename Iterator,typename T>
T parallel_accumulate(Iterator first,Iterator last,T init)
{
  unsigned long const length=std::distance(first,last);

  if(!length)
    return init;

  unsigned long const min_per_thread=25;
  unsigned long const max_threads=
      (length+min_per_thread-1)/min_per_thread;

  unsigned long const hardware_threads=
      std::thread::hardware_concurrency();

  unsigned long const num_threads=
      std::min(hardware_threads != 0 ? hardware_threads : 2, max_threads);

  unsigned long const block_size=length/num_threads;

  std::vector<T> results(num_threads);
  std::vector<std::thread> threads(num_threads-1);

  Iterator block_start=first;
  for(unsigned long i=0; i < (num_threads-1); ++i)
  {
    Iterator block_end=block_start;
    std::advance(block_end,block_size);
    threads[i]=std::thread(
        accumulate_block<Iterator,T>(),
        block_start,block_end,std::ref(results[i]));
    block_start=block_end;
  }
  accumulate_block<Iterator,T>()(
      block_start,last,results[num_threads-1]);

  for (auto& entry : threads)
    entry.join();

  return std::accumulate(results.begin(),results.end(),init);
}
```

其中决定线程数量的代码如下

``` C++
unsigned long const min_per_thread = 25;
unsigned long const max_threads = (length+min_per_thread-1) / min_per_thread;

unsigned long const hardware_threads = std::thread::hardware_concurrency();

unsigned long const num_threads = std::min(hardware_threads != 0 ? hardware_threads : 2, max_threads);
```

## 标识线程

线程标识类型为`std::thread::id`，并可以通过两种方式进行检索：

* 可以通过调用`std::thread`对象的成员函数`get_id()`来直接获取（如果`std::thread`对象没有与任何执行线程相关联，`get_id()`将返回`std::thread::id`默认构造值，这个值表示“无线程”）

* 当前线程中调用`std::this_thread::get_id()`也可以获得线程标识

`std::thread::id`类型对象提供相当丰富的对比操作，这允许程序员将其当做为容器的键值，做排序的比较。标准库也提供`std::hash<std::thread::id>`容器，所以`std::thread::id`也可以作为无序容器的键值。

`std::thread::id`实例常用作检测线程是否需要进行一些操作

``` C++
std::thread::id master_thread;
void some_core_part_of_algorithm()
{
  if(std::this_thread::get_id()==master_thread)
  {
    do_master_thread_work();
  }
  do_common_work();
}
```