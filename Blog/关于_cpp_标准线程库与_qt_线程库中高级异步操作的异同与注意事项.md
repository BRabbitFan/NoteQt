<!--
author: BRabbitFan
date: 2022-03-02
title: 关于 C++ 标准线程库与 Qt 线程库中高级异步操作的异同与注意事项
tags: Qt,C++
category: Qt学习笔记
status: publish
summary: 本文简单介绍 C++ 标准线程库与 Qt 线程库高级异步操作的基本使用方式. 对比总结异同点, 与一些使用时特殊的注意事项.
-->

<div id="返回顶部"></div>

> 转载请注明原文链接: http://brabbit.xyz/blog/NoteQt/Blog/关于_cpp_标准线程库与_qt_线程库中高级异步操作的异同与注意事项.html

# 文章目录

- [前言](#前言)
- [基本使用方法介绍](#基本使用方法介绍)
  - [C++11 标准线程库中的高级异步操作 : `std::async`](#C++11标准线程库中的高级异步操作)
      - [`std::async`](#std::async)
      - [`std::future`](#std::future)
      - [`std::future::wait_for`](#std::future::wait_for)
      - [`std::future::get`](#std::future::get)
  - [Qt 线程库的高级异步操作 : `QtConcurrent::run`](#Qt线程库的高级异步操作)
      - [`QtConcurrent::run`](#QtConcurrent::run)
      - [`QFuture`](#QFuture)
      - [`QFuture::isFinished`](#QFuture::isFinished)
      - [`QFuture::result`](#QFuture::result)
      - [`QFutureWatcher`](#QFutureWatcher)
- [关于两个线程库的异同](#关于两个线程库的异同)
  - [`std::future` 的析构阻塞特性](#std::future的析构阻塞特性)
  - [``std::async` 在不同编译器实现下的多线程策略](#std::async在不同编译器实现下的多线程策略)
- [结语](#结语)

---

<div id="前言"></div>

# 前言

写这篇文章的契机是在我重构一段代码的过程中遇到的一个疑惑. 本来的目的是想使用 C++ 标准线程库中的高级异步操作代替 Qt 线程库中的对应功能. 但在测试过程中发现使用了标准库的版本 "似乎" 效率很低, 低到仿佛不是在进行异步操作. 于是便查阅了一些文章并且做了些测试, 最后发现我这里的主观猜测是错误的. 产生这种错误判断的原因在于标准库与 Qt 库指导思想上的差别导致的实现上的差别, 以至于代码上看似相同的操作可能会得到截然不同的结果.

> 文中 C++ 标准库源码示例参考 GNU GCC 9.3.0 的实现, Qt 线程库源码示例参考 Qt 5.14.2 MSVC 2017 的实现.

[<返回顶部>](#返回顶部)

---

<div id="基本使用方法介绍"></div>

# 基本使用方法介绍  

<div id="C++11标准线程库中的高级异步操作"></div>

### C++11 标准线程库中的高级异步操作 : `std::async`  

在 C++11 后引入 std 的标准线程库中, 封装了比 `std::thread` 更高级的异步操作. 即 `<future>` 中的 `std::async` 与 `std::future` . 该接口可供用户无需关心底层线程资源调用策略, 而直接进行异步操作.

[<返回顶部>](#返回顶部)

<div id="std::async"></div>

#### `std::async` 

函数原型为 :

```cpp
template<typename _Fn, typename... _Args>
future<__async_result_of<_Fn, _Args...>> async(launch __policy, _Fn&& __fn, _Args&&... __args);
```

第一个参数用于指定执行策略, 可选的策略有 4 种:

```cpp
// namespace std
enum class launch {
  async = 1,
  deferred = 2
};

std::async(std::launch::async, [](){});                          // 异步执行
std::async(std::launch::deferred, [](){});                       // 同步延迟执行
std::async(std::launch::async | std::launch::deferred, [](){});  // 由 OS 决定
std::async([](){});                                              // 默认 (同步延迟执行)
```

第二个参数为异步函数, 第三个参数起为异步函数的参数列表.

> C++ 11 标准线程库种的高级异步操作除了 `std::async` 还有 `std::promise` 与 `std::packaged_task`. `std::async` 相对于封装了 `std::thread` 与 `std::promise` 与 `std::packaged_task` . 后两者不再本文讨论范围内, 不多介绍.  

[<返回顶部>](#返回顶部)

<div id="std::future"></div>

#### `std::future` 

`std::future` 是 `std::async` , `std::promise` 与 `std::packaged_task` 的底层对象, 用于传递异步函数的返回值.  

[<返回顶部>](#返回顶部)

<div id="std::future::wait_for"></div>

#### `std::future::wait_for` 

通过 `std::future::wait_for` 可在等待结束后获得当前 future 对象的状态, 原型为 : 

```cpp
// class __basic_future
template<typename _Rep, typename _Period>
future_status wait_for(const chrono::duration<_Rep, _Period>& __rel) const {
  _State_base::_S_check(_M_state);
  return _M_state->wait_for(__rel);
}
```

`std::future_status` 状态包括: 

```cpp
enum class future_status {
  ready,    // 就绪
  timeout,  // 超时
  deferred  // 未就绪
};
```

[<返回顶部>](#返回顶部)

<div id="std::future::get"></div>

#### `std::future::get` 

通过 `std::future::get` 可获得异步函数的返回值.

当使用 `std::async` 以 `std::launch::async` 异步模式作业时, `std::future` 将在异步操作结束后获得其返回值. 常用的检查状态操作是设置 `std::future::wait_for` 方法参数为 0 秒, 直接检查当前状态.  

```cpp
std::future<bool> future = std::async(std::launch::async, [](){
  std::this_thread::sleep_for(std::chrono::seconds(1));
  return true;
});

while (true) {
  if (futrue.wait_for(std::chrono::seconds(0)) == std::future_status::ready){
    std::cout << futrue.get();
    break;
  }
}
```

当使用 `std::async` 以 `std::launch::deferred` 同步延迟模式作业时, 将在执行 `std::future::get` 时执行函数, 并立即获得返回值.

```cpp
std::future<bool> future = std::async(std::launch::deferred, [](){
  std::this_thread::sleep_for(std::chrono::seconds(1));
  return true;
});

std::cout << futrue.get();  // 阻塞于此, 直到执行完毕函数.
```

[<返回顶部>](#返回顶部)

<div id="Qt线程库的高级异步操作"></div>

### Qt 线程库的高级异步操作 : `QtConcurrent::run`  

Qt 的线程库非常好用, 也同样提供了高级的异步处理方法. 这些方法在头文件 `<QtConcurrent>` 中, 这里我们关心的主要是 `QtConcurrent::run` 方法与 `QFuture` 对象.  

[<返回顶部>](#返回顶部)

<div id="QtConcurrent::run"></div>

#### `QtConcurrent::run`  

该函数有多个重载, 其中一个原型为 :  

```cpp
template <typename T, typename Param1, typename Arg1>
QFuture<T> run(QThreadPool *pool, T (*functionPointer)(Param1), const Arg1 &arg1) {
  return (new StoredFunctorCall1<T, T (*)(Param1), Arg1>(functionPointer, arg1))->start(pool);
}
```

其余的重载函数区别主要在于对第一个参数设置缺省值与更长的参数列表.  

参数 1 用于指定用户自己所使用的线程池, 若不指定则默认用全局线程池. 参数 2 即异步函数. 参数 3 起为异步函数的参数列表.  

> 这里说到有针对参数列表长度的多个重载, 应该是 Qt 版本比较老导致的, 推测新版也会改用可变参数模板实现.  

[<返回顶部>](#返回顶部)

<div id="QFuture"></div>

#### `QFuture`  

与 `std::future` 类似, `QFuture` 用于获取 `QtConcurrent::run` 所执行异步函数的返回值.  

[<返回顶部>](#返回顶部)

<div id="QFuture::isFinished"></div>

#### `QFuture::isFinished`  

`QFuture::isFinished` 用于检查当前 `QFuture` 所对应异步操作的状态, 若执行完毕则返回 `true` . 原型为 :

```cpp
// qfuture.h
template <typename T>
class QFuture {
public:
  QFuture() : d(QFutureInterface<T>::canceledResult()) { }
  explicit QFuture(QFutureInterface<T> *p) : d(*p) { }
  bool isFinished() const { return d.isFinished(); }
......
};
// qfutureinterface.h
bool QFutureInterfaceBase::isFinished() const {
  return queryState(Finished);
}
bool QFutureInterfaceBase::queryState(State state) const {
  return d->state.loadRelaxed() & state;
}
```

> Qt 的源码比起 std 的源码更 Java like.  
好处是代码看起来更易懂, 但缺点是检查一个功能的实现总是要跳转好几个文件.  
并且比起基于 GP 实现编译期多态的 标准库, 基于 OOP 实现运行期多态的 Qt 库执行效率理论上更低.

[<返回顶部>](#返回顶部)

<div id="QFuture::result"></div>

#### `QFuture::result`  

`QFuture::result` 用户获取当前 `QFuture` 所对应异步操作的返回值.

```cpp
QFuture<bool> future = QtConcurrent::run([]()->bool {
  QThread::sleep(1);
  return true;
});

while (true) {
  if (future.isFinished()){
    qDebug() << futrue.value();
    break;
  }
}
```

> QFuture 比起 std::future 封装了更多实用的方法, 此处不多介绍.

[<返回顶部>](#返回顶部)

<div id="QFutureWatcher"></div>

#### `QFutureWatcher` 与 信号槽机制

`QFutureWatcher` 提供对 `QFuture` 状态检测的方法, 结合 Qt 的信号槽机制, 可以设置回调当异步函数执行完毕后立即获取返回值并加以处理.

```cpp
QFuture<bool> future = QtConcurrent::run([]()->bool {
  QThread::sleep(1);
  return true;
});

QFutureWatcher<bool> watcher;
watcher.setFuture(future);
QObject::connect(&watcher, &QFutureWatcher<bool>::finished, this, [&future]() {
  qDebug() << future.result();
}, Qt::QueuedConnection);
```

[<返回顶部>](#返回顶部)

---

<div id="关于两个线程库的异同"></div>

# 关于两个线程库的异同

上面介绍两套线程库的基本高级操作看似很相似, 但实际使用上有些不同, 其中一些差异很容易造成用户的误解.  

仅从表面来看异同点如下:  

- 相同点  
如上一大节中的示例, 基本操作就是利用异步方法执行一个异步函数, 并获得一个 future 对象.  
通过检查 future 对象的状态判断异步操作是否已执行完毕, 并通过 future 获取异步函数的返回值.  
- 不同点  
  1. `std::async` 除了异步模式外, 还可以设置同步延迟模式. 让函数在合适的地方由用户主动调用.
  2. `QFutureWatcher` 提供了更方便的检查 future 对象状态的方法, 并允许利用信号槽机制设置回调.

> 除此之外, Qt 的库总体来说使用上更方便一些.  
C++ 标准委员会始终秉持着自下而上效率为先的理念, 导致标准库封装的功能经常不太实用. (效率有时候也没有太高 orz)

上述这些异同只是流于表面的, 两个线程库有些功能设计上的根本区别导致看似相同的操作实际上将造成不同的结果. 使用时若不加以注意则可能你的代码将以与你预期所截然不同的情况运行. 下面介绍两个点.

[<返回顶部>](#返回顶部)

<div id="std::future的析构阻塞特性"></div>

### `std::future` 的析构阻塞特性  

首先, 请先仔细看看下面的代码: 

```cpp
// 获得当前时间的字符串, 忽略
std::string get_now() {
  auto tp = std::chrono::system_clock::now();
  auto tt = std::chrono::system_clock::to_time_t(tp);
  auto tm = localtime(&tt);
  char buf[9] = {0};
  sprintf(buf, "%02d:%02d:%02d", tm->tm_hour, tm->tm_min, tm->tm_sec);
  return std::string(buf);
}

// 标准库版本
void waste_time_std() {
  std::async(std::launch::async, []() {
    std::this_thread::sleep_for(std::chrono::minutes(10));
    std::cout << "std  " << get_now() << std::endl;
  });
}

// Qt 版本
void waste_time_qt() {
  QtConcurrent::run([]() {
    QThread::sleep(10 * 60);
    std::cout << "qt   " << get_now() << std::endl;
  });
}

// 测试用例
int main(int argc, char* argv[]) {
  std::cout << "main " << get_now() << std::endl;  // -> main 00:00:00
  waste_time_std();
  std::cout << "main " << get_now() << std::endl;
  waste_time_qt();
  std::cout << "main " << get_now() << std::endl;
  return 0;
}
```

这里我们利用两个线程库的高级异步操作分别等待了 10 分钟后打印当前时间. 由于不需要处理异步函数的返回值, 因此我们不接收对应的 future 对象.  
现在试想一下这段代码的执行时间与输出结果何如?  

依据上文的经验, 我们有理由推测执行时间大约在 10 分钟出头, 并且输出可能是这样的: 

```text
main 00:00:00
main 00:00:01
main 00:00:02
std  00:10:01
qt   00:10:02
```

而实际上这段代码将运行约 20 分钟出头, 且输出大致像这样:

```text
main 00:00:00
std  00:10:01
main 00:10:02
main 00:10:03
qt   00:20:04
```

造成这样结果的原因在于 `std::future` 对象在析构时将阻塞等待异步函数执行完毕, 而 `QFuture` 则不会进行这样的阻塞.  
代码中两个 future 对象在进行异步操作后返回, 由于我们没有用左值接收它们, 返回后的这两个右值将在当前行执行完毕后析构. `waste_time_std()` 处的阻塞现象就是来源于 `std::future` 析构时的阻塞. 而根据上文 `waste_time_qt()` 内的 `QFuture` 不会在析构时阻塞, 自然也不会造成 `waste_time_qt()` 处阻塞.  

我在前言中提到自己遇到标准库实现版本的效率 "似乎" 很低的原因也在于此.

现在我们将代码稍作修改, 如下:

```cpp
.....

// 标准库版本
std::future<void> waste_time_std() {
  return std::async(std::launch::async, []() {
    std::this_thread::sleep_for(std::chrono::minutes(10));
    std::cout << "std  " << get_now() << std::endl;
  });
}

// Qt 版本
......

// 测试用例
int main(int argc, char* argv[]) {
  std::cout << "main " << get_now() << std::endl;  // -> main 00:00:00
  auto future = waste_time_std();
  std::cout << "main " << get_now() << std::endl;
  waste_time_qt();
  std::cout << "main " << get_now() << std::endl;
  return 0;
}
```

`std::future` 对象的析构被延后至 main 函数返回时. 此时将阻塞至两个被我们调用的异步线程均执行完异步操作. 这样一来, 代码的执行时间与输出结果则正如我们最初所预料的情况了.  

这个结论是 100% 成立的. C++11 起的标准要求编译器必须对非指针非引用的实例返回值进行避免内存拷贝的优化. 而 C++11 以前也没有标准线程库, 自然也不具备反驳的前提条件了.

> 个人认为造成这种差异的根本原因还是在于标准线程库和 Qt 线程库封装时的指导思想.  
Qt 团队希望尽可能地隐藏细节, 让用户只需要写好业务代码就好了;  
而 C++ 标准委员会希望用户无论何时都可以尽可能地控制资源, 因此即便开始了异步作业, 但 future 对象仍然保留对该线程一定程度上的控制权.

[<返回顶部>](#返回顶部)

<div id="std::async在不同编译器实现下的多线程策略"></div>

### `std::async` 在不同编译器实现下的多线程策略  

网上很多聊 C++ 各类线程库的文章总喜欢把 `std::async` 拎出来批评一番. 最主要的一个论点就是:  
" `std::async` 将在每次调用时都开辟新的线程, 当频繁的调用时开销可能将增大到无法忍受的地步. "  
根据上文, 我们已经知道了 Qt 线程库高级异步操作的多线程策略里是利用线程池的. 相比之下明显 Qt 或 Boost 的实现比标准库的实现更有优势. 从 CSDN 到 Stack Overflow 上的 "programmer KOL" 们基本上都秉持这个观点. 

这个结论本身并不是错误的, 只是大部分作者都忽略了得出这个结论的前提. 那就是 C++ 标准委员会至今没有对 `std::async` 调用时的多线程策略做出任何要求, 只规定了异步处理这个功能上的要求. 因此这个函数的多线程策略实际上是由各编译器自行决定的. 

经过我的测试, 发现大部分文章作者是基于 GNU 实现的版本得出的结论, 而 G++ 的实现就是每调用一次 `std::async` 就开辟一个新线程. 但微软实现的版本则不同, MSVC 的实现中 `std::async` 也引入了某种线程复用的机制. 请看下面的代码: 

```cpp
// main.cpp
#include <iostream>
#include <future>
#include <array>
static constexpr size_t MaxIndex = 100;
int main(int argc, char* argv[]) {
  std::array<std::future<void>, MaxIndex> futures;
  std::cout << "start" << std::endl;
  for (size_t index = 0; index != MaxIndex; index++) {
    futures[index] = std::async(std::launch::async, [index]() {
      std::this_thread::sleep_for(std::chrono::seconds(1));
      std::cout << index << " " << std::this_thread::get_id() << std::endl;
    });
  }
  std::cout << "end" << std::endl;
  return 0;
}
```

这段代码一次性执行了 100 次 `std::async` 标准线程库的异步操作, 每个异步操作都在等待 1 秒钟后输出本次异步操作对应的 index 与当前线程的 ID.  
我们将同一段代码分别使用 G++ 与 MSVC 编译器编译执行, 观察其输出结果.  

首先是 G++ 版本, 输出如下:  

```text
start
end
6 140380312651520
7 140380304258816
5 140380321044224
8 140380295866112
9 140380287473408
10 140380279080704
4 140380329436928
11 140380270688000
12 140380262295296
13 140380253902592
14 140380245509888
3 140380337829632
15 140380237117184
16 140380228724480
17 140380220331776
......
```

这里没有贴出完整的输出, 但我可以明确地告诉你, 这 100 个线程 ID 里没有出现重复项. 正如网上大部分文章所言.  

接下来我们再看一下 MSVC 版本:  

```text
start
end
2 12208  <-- 线程 12208
4 16120
0 7172
1 12820
3 9232
5 28080
6 20468
7 20100
8 26656
9 24788
10 12508
13 7172
12 16120
14 12820
11 12208  <-- 线程 12208
......
```

可见, 这里出现了重复的线程 ID , 并且贴出来的这几个线程 ID 在后续的输出里也重复出现了多次. 显然不是每一次调用都开辟新的线程, 而是使用了线程池或其他复用线程的机制. 若微软在底层用的是线程池, 经过简单的统计, 这个线程池在程序执行完毕后大致有 32 个线程的大小.  

这里我只测试了 G++ 与 MSVC 的实现, 我手上目前没有 Clang 的环境, 因此没做测试. 不过已经能够得出结论了. 

> 其实不同编译器实现的 C++ 之间还是有不少差异的. 甚至在标准模板库的数据结构中也存在一些差异.  
> 除了文中所说的这一个以外, 还有几个我目前已知的差异 (以截止 C++ 14 为准):  
> 1. MSVC 中支持 for each 语句, 而 G++ 不支持.  
> 2. G++ 允许数组声明时长度以变量方式设置, MSVC 则必须用常量.
> 3. G++ 允许外部构造结构体实例时在同一语句内设置实例内成员的值, MSVC 则必须构造完毕后再赋值.
> 4. 早期 G++ 中 std::list 底层数据结构是两个方向相反的单向链表, MSVC 中是一个双向链表.  
> (检查发现 GNU GCC 9.3.0 的 G++ 也改用一个双向链表实现了.)

[<返回顶部>](#返回顶部)

---

<div id="结语"></div>

# 结语

到此, 我们可以做一个简单的总结.  

首先是文中各实验结果的归纳:  
1. C++ 标准线程库与 Qt 线程库均提供了高级的异步操作工具, 但使用上 Qt 线程库的版本更方便.  
2. `std::async` 与 `QtConcurrent` 相比, 前者的优势是允许使用同步延迟操作的方式, 后者的优势是允许指定线程池.
2. `std::future` 与 `QFuture` 相比, 前者析构时将阻塞至对应的异步函数执行结束, 后者析构时直接完全放弃控制权.  
3. Qt 线程库比起 C++ 标准线程库多了 `QFutureWatcher` 用于更方便的监控异步操作的状态, 并且可以配合信号槽机制实现更多功能.  

其次是一些个人主观感受. 一个是不同库在使用时能明显感受到其代码的 "风格" , 由于这些 "风格" 的差异, 可能会导致不同库之间类似的操作可能会带来不同的结果. 还有一个是有时候比起大量查阅资料, 自己动手实验会得到更准确的结果, 也会节约很多时间. 网络博客上的文章大部分都没有告诉我们足够的前提条件与实验环境, 这时候需要自己辨别. 

最后说点私货, Qt 库中大部分功能实现使用起来比 C++ 标准库/标准模板库中的实现更好用, 但前者相当一部分功能的效率不如后者. 这里我根据目前自己测试过的各功能模块提一些使用选择上的建议. 下面全是我的判断, 你可以选择性接受:  
1. Qt 线程库内的并发机制效率只比标准线程库中略低, 但使用上更方便, 此处建议优先使用 Qt 线程库.  
2. . Qt 线程库内的同步机制效率比标准线程库中的实现效率低很多. 如 QMutex 相比于 std::mutex, 在 MSVC 编译器 -O2 级别优化下, 前者效率大约只是后者的 1/10. 建议使用标准线程库中的同步机制而非 Qt 的对应功能.  
3. Qt 基础库中各类容器与 STL 相比, 效率上相差不大, 可自行选择使用. 但两个库本身不是完全一一对应的, 详见下面两点.  
4. QString 比起饱受诟病的 std::string 更方便, 建议使用 QString 而非 std::string . 使用 std::string 时可以把它视作类似 QByteArray 的结构. (std::string 如果叫做 std::byte_array 就合理多了)  
5. QList 对应的 STL 容器不是 std::list, 应该是 std::vector. QList 不是双向链表实现的, 而是优化过内存分配策略的 QVector. 经过测试 QList 的效率非常高, 这里推荐使用 QList 代替 QVector/std::vector/std::list/std::forward_list.

> 这些对比是我自己在开发间隙测试得出的结论, 仅供参考. 后续把测试过程总结放上来.

[<返回顶部>](#返回顶部)