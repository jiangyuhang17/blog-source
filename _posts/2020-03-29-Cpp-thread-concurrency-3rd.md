---
title: C++多线程并发系列（三）
subtitle: C++ Concurrency in Action 2nd Edition 读书笔记
date: 2020-03-29 10:48:34
categories: C++
tags: [C++, 并发控制]
---

# 概述

上一篇中，我们看到各种在线程间保护共享数据的方法。有时候我们不仅想要保护数据，还想在独立的线程间进行同步操作。例如，在第一个线程完成前，可能需要等待另一个线程执行完成。通常情况下，线程会等待一个特定事件发生，或者等待某一条件达成。

尽管可以通过定期检查共享数据中存储的“任务完成”标志或类似内容来执行此操作，但这不是很理想。像这样的线程间同步操作的需求非常普遍，C++标准库以`condition variables`和`future`的形式提供了处理它的工具。

# 等待一个事件或者其他条件

当一个线程等待另一个线程完成任务时，它可以用很多种选择。第一，它可以持续地检查共享数据标志，直到另一线程完成工作时对这个标志进行重设；这样线程会消耗宝贵的执行时间来持续地检查对应标志，并且当互斥量被等待线程上锁后，其他线程就没有办法获取锁，这样线程就会持续等待。

第二个选择线程使用`std::this_thread::sleep_for()`在检查之间进行周期性的休眠。这个实现就进步很多，因为当线程休眠时，线程没有浪费执行时间，但是很难确定正确的休眠时间。太短的休眠和没有休眠一样，都会浪费执行时间；太长的休眠时间，可能会让任务等待线程醒来。

``` C++
bool flag;
std::mutex m;

void wait_for_flag()
{
  std::unique_lock<std::mutex> lk(m);
  while(!flag)
  {
    lk.unlock();
    std::this_thread::sleep_for(std::chrono::milliseconds(100));
    lk.lock();
  }
}
```

第三个选择，也是优先的选择，使用C++标准库提供的工具去等待事件的发生。等待由另一个线程触发的事件的最基础机制就是`condition variables`（条件变量）。

## condition variables

C++标准库对条件变量有两套实现：`std::condition_variable`和`std::condition_variable_any`。这两个实现都包含在`<condition_variable>`头文件的声明中。

两者都需要与一个互斥量一起才能工作(互斥量是为了同步)；前者仅限于与`std::mutex`一起工作，而后者可以和任何满足最低标准的互斥量一起工作，从而加上了_any的后缀。因为`std::condition_variable_any`更加通用，这就可能从体积、性能，以及系统资源的使用方面产生额外的开销，所以`std::condition_variable`一般作为首选的类型，当对灵活性有硬性要求时，我们才会去考虑`std::condition_variable_any`。

``` C++
// 使用条件变量处理数据等待
std::mutex mut;
std::queue<data_chunk> data_queue;
std::condition_variable data_cond;

void data_preparation_thread()
{
  while(more_data_to_prepare())
  {
    data_chunk const data=prepare_data();
    std::lock_guard<std::mutex> lk(mut);
    data_queue.push(data);
    data_cond.notify_one();
  }
}

void data_processing_thread()
{
  while(true)
  {
    std::unique_lock<std::mutex> lk(mut);
    data_cond.wait(
         lk,[]{return !data_queue.empty();});
    data_chunk data=data_queue.front();
    data_queue.pop();
    lk.unlock();
    process(data);
    if(is_last_chunk(data))
      break;
  }
}
```

对于正在处理数据的线程，这个线程首先对互斥量上锁，这里`std::unique_lock`要比`std::lock_guard`更加合适。线程之后会调用`std::condition_variable`的成员函数`wait()`，传递一个锁和一个lambda函数表达式(作为等待的条件)。Lambda函数是C++ 11添加的新特性，它可以让一个匿名函数作为其他表达式的一部分，本质是一个函数对象，并且非常合适作为标准函数的谓词。本质上来说，`std::condition_variable::wait`是对“busy-wait”的一种优化。

很多线程可能等待同一事件，对于通知，他们都需要做出回应。在这种情况下，线程准备好数据时，就会通过条件变量调用`notify_all()`成员函数，而非直接调用`notify_one()`函数。

# 使用future等待一次性事件

C++标准库模型将一次性事件称为`future`(期望值)。当线程需要等待特定的一次性事件时，某种程度上来说就需要知道这个事件在未来的期望结果。之后，这个线程会周期性的等待或检查，事件是否触发；检查期间也会执行其他任务。另外，等待任务期间它可以先执行另外一些任务，直到对应的任务触发，而后等待期望值的状态会变为就绪。一个期望值可能是数据相关的，也可能不是。当事件发生时(并且期望状态为就绪)，并且这个期望值就不能被重置。

C++标准库中，有两种期望值，使用两种类型模板实现，声明在`<future>`头文件中：`unique futures`(唯一期望值)(`std::future<>`)和`shared futures`(共享期望值)(`std::shared_future<>`)。`std::future`的实例只能与一个指定事件相关联，而`std::shared_future`的实例就能关联多个事件。后者的实现中，所有实例会在同时变为就绪状态，并且他们可以访问与事件相关的任何数据。与数据无关处，可以使用`std::future<void>`与`std::shared_future<void>`的特化模板。**期望值对象本身并不提供同步访问**，当多个线程需要访问一个独立期望值对象时，必须使用互斥量或类似同步机制对访问进行保护

## 后台返回任务值

当不着急要任务结果时，可以使用`std::async`启动一个异步任务。与`std::thread`对象等待的方式（不提供直接接收返回值的机制）不同，`std::async`会返回一个`std::future`对象，这个对象持有最终计算出来的结果。当需要这个值时，只需要调用这个对象的get()成员函数；并且会阻塞线程直到期望值状态为就绪为止；之后，返回计算结果。

``` C++
// 使用std::future从异步任务中获取返回值
#include <future>
#include <iostream>

int find_the_answer_to_ltuae();
void do_other_stuff();
int main()
{
  std::future<int> the_answer=std::async(find_the_answer_to_ltuae);
  do_other_stuff();
  std::cout<<"The answer is "<<the_answer.get()<<std::endl;
}
```

与`std::thread`做的方式一样，`std::async`允许你通过添加额外的调用参数，向函数传递额外的参数。

``` C++
// 使用std::async向函数传递参数
#include <string>
#include <future>
struct X
{
  void foo(int,std::string const&);
  std::string bar(std::string const&);
};
X x;
auto f1=std::async(&X::foo,&x,42,"hello");  // 调用p->foo(42, "hello")，p是指向x的指针
auto f2=std::async(&X::bar,x,"goodbye");  // 调用tmpx.bar("goodbye")， tmpx是x的拷贝副本
struct Y
{
  double operator()(double);
};
Y y;
auto f3=std::async(Y(),3.141);  // 调用tmpy(3.141)，tmpy通过Y的移动构造函数得到
auto f4=std::async(std::ref(y),2.718);  // 调用y(2.718)
X baz(X&);
std::async(baz,std::ref(x));  // 调用baz(x)
class move_only
{
public:
  move_only();
  move_only(move_only&&)
  move_only(move_only const&) = delete;
  move_only& operator=(move_only&&);
  move_only& operator=(move_only const&) = delete;

  void operator()();
};
auto f5=std::async(move_only());  // 调用tmp()，tmp是通过std::move(move_only())构造得到
```

默认情况下，`std::async`启动新线程，还是当future等待时才同步运行任务，取决于实现。大多数情况下，就是你想要的结果，但是你也可以在函数调用之前向std::async传递一个额外参数。这个参数的类型是`std::launch`，既可以是`std::launch::defered`，表明函数调用被延迟到`wait()`或`get()`函数调用时才执行，也可以是`std::launch::async`，表明函数必须在其所在的独立线程上执行，还可以是 `std::launch::deferred | std::launch::async`，表明实现可以选择这两种方式的一种，最后一个选项是默认的。当函数调用被延迟，它可能不会再运行了。如下所示：

``` C++
auto f6=std::async(std::launch::async,Y(),1.2);  // 在新线程上执行
auto f7=std::async(std::launch::deferred,baz,std::ref(x));  // 在wait()或get()调用时执行
auto f8=std::async(
              std::launch::deferred | std::launch::async,
              baz,std::ref(x));  // 实现选择执行方式
auto f9=std::async(baz,std::ref(x));
f7.wait();  //  调用延迟函数
```

### 创造future的三种方式

`std::async`不是让`std::future`与任务实例相关联的唯一方式；你也可以将任务包装入`std::packaged_task<>`实例中，或通过编写代码的方式，使用`std::promise<>`类型模板显式设置值。与`std::promise<>`对比，`std::packaged_task<>`具有更高层的抽象，所以我们从“高抽象”的模板说起。

## 任务与future关联

`std::packaged_task<>`对一个函数或可调用对象，绑定一个期望值。当调用`std::packaged_task<>`对象时，它就会调用相关函数或可调用对象，将期望状态置为就绪，返回值也会被存储。

`std::packaged_task<>`的模板参数是一个函数签名。函数签名的返回类型可以用来标识从`get_future()`返回的`std::future<>`的类型，而函数签名的参数列表，可用来指定packaged_task的函数调用操作符。

由于`std::packaged_task`对象是一个可调用对象，所以可以封装在`std::function`对象中，从而作为线程函数传递到`std::thread`对象中，或作为可调用对象传递另一个函数中，或可以直接进行调用。当`std::packaged_task`作为一个函数被调用时，实参将由函数调用操作符传递到底层函数，并且返回值作为异步结果存储在`std::future`，可通过`get_future()`获取。

因此你可以把用`std::packaged_task`打包任务，并在它被传到别处之前的适当时机取回期望值。当需要异步任务的返回值时，你可以等待期望的状态变为“就绪”。

``` C++
#include <deque>
#include <mutex>
#include <future>
#include <thread>
#include <utility>

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

## 使用promises

`std::promise<T>`提供设定值的方式，这个类型会和后面看到的`std::future<T>`对象相关联。一对`std::promise`/`std::future`提供了一个可行的机制：`future`可以阻塞等待线程，同时，提供数据的线程可以使用组合中的`promise`来对相关值进行设置，并将期望值的状态置为“就绪”。

可以通过一个给定的`std::promise`的`get_future()`成员函数来获取与之相关的`std::future`对象，跟`std::packaged_task`的用法类似。当承诺值已经通过调用`set_value()`成员函数设置完毕，对应期望值的状态变为“就绪”，并且可用于检索已存储的值。当在设置值之前销毁`std::promise`，将会存储一个异常。

``` C++
// 使用promise解决单线程多连接问题
#include <future>

void process_connections(connection_set& connections)
{
  while(!done(connections))
  {
    for(connection_iterator
            connection=connections.begin(),end=connections.end();
          connection!=end;
          ++connection)
    {
      if(connection->has_incoming_data())
      {
        data_packet data=connection->incoming();
        std::promise<payload_type>& p=
            connection->get_promise(data.id);
        p.set_value(data.payload);
      }
      if(connection->has_outgoing_data())
      {
        outgoing_packet data=
            connection->top_of_outgoing_queue();
        connection->send(data.payload);
        data.promise.set_value(true);
      }
    }
  }
}
```

## 将异常存在future中

函数作为`std::async`的一部分时，当调用抛出一个异常时，这个异常就会存储到期望值中，之后期望值的状态被置为“就绪”，之后调用`get()`会抛出这个已存储异常。

将函数打包入`std::packaged_task`任务包中后，到任务被调用时，同样的事情也会发生；打包函数抛出一个异常，这个异常将被存储在期望值中，准备在`get()`调用时再次抛出。

当然，通过函数的显式调用，`std::promise`也能提供同样的功能。当存入的是一个异常而非一个数值时，就需要调用`set_exception()`成员函数，而非`set_value()`。这通常是用在一个catch块中，并作为算法的一部分，为了捕获异常，使用异常填充承诺值。

``` C++
extern std::promise<double> some_promise;
try
{
  some_promise.set_value(calculate_value());
}
catch(...)
{
  some_promise.set_exception(std::current_exception());
}
```

## 多个线程等待

尽管`std::future`处理了将数据从一个线程传输到另一个线程所需的所有同步，但是对特定`std::future`实例的成员函数的调用不会彼此同步。如果从多个线程访问单个`std::future`对象而不进行额外的同步，则将发生数据争用和未定义的行为。这是因为`std::future`模型独享同步结果的所有权，并且通过调用`get()`函数，一次性的获取数据，这就让并发访问变的毫无意义，只有一个线程可以获取结果值，因为在第一次调用`get()`后，就没有值可以再获取了。

`std::future`是只移动的，所以其所有权可以在不同的实例中互相传递，但是只有一个实例可以获得特定的同步结果，而`std::shared_future`实例是可拷贝的，所以多个对象可以引用同一关联期望值的结果。

每一个`std::shared_future`的独立对象上，成员函数调用返回的结果还是不同步的，所以为了在多个线程访问一个独立对象时避免数据竞争，必须使用锁来对访问进行保护。优先使用的办法：为了替代只有一个拷贝对象的情况，可以让每个线程都拥有自己对应的拷贝对象；这样，当每个线程都通过自己拥有的`std::shared_future`对象获取结果，那么多个线程访问共享同步结果就是安全的。

`std::shared_future`的实例同步`std::future`实例的状态。当`std::future`对象没有与其他对象共享同步状态所有权，那么所有权必须使用`std::move`将所有权传递到`std::shared_future`，其默认构造函数如下：

``` C++
std::promise<int> p;
std::future<int> f(p.get_future());
assert(f.valid());  // 期望值 f 是合法的
std::shared_future<int> sf(std::move(f));
assert(!f.valid());  // 期望值 f 现在是不合法的
assert(sf.valid());  // sf 现在是合法的

std::promise<std::string> p;
std::shared_future<std::string> sf(p.get_future());  // 隐式转移所有权

std::promise<std::map< SomeIndexType, SomeDataType, SomeComparator,
     SomeAllocator>::iterator> p;
auto sf=p.get_future().share(); // 自动类型推导，使得代码容易修改
```