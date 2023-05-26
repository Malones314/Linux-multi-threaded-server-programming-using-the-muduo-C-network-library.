# Linux多线程服务端编程-使用muduo C++ 网络库
## 多线程编程

### 线程安全的对象生命期管理
可以使用boost库中的shared_ptr和weak_otr完美解决析对象析构时可能存在的race condition(竞态条件)
#### 析构函数遇到多线程的情况
1. 可能出现的几种race condition：

    1.1 即将析构一个对象时，如何知道是否有别的线程正在执行该对象的成员函数

    1.2 如何保证正在执行成员函数期间，对象不会在另一个线程里被析构

    1.3 在调用一个对象的成员函数之前，如何知道这个对象还活着，他的析构函数会不会正好执行到一半

这些问题可以通过使用```shared_ptr```来一劳永逸地解决。

2. 一个线程安全的class应该有以下三个条件：

    2.1 多个线程同时访问时，有正确的表现

    2.2 不论操作系统如何调度这些线程，不论这些线程的执行顺序如何交织

    2.3 调试端代码无需额外的同步或其他协调工作

根据这三个条件，```string```，```vector```，```map```等都不是线程安全的class，因为他们需要在外部加锁才能提供多个线程同时访问。
 
3. 对象的构造要做到线程安全，**唯一**的要求是在构造期间不要泄漏```this```指针，即：
    1. 不要在构造期间注册任何回调
    2. 不要在构造函数中把this传递给跨线程的对象
    3. 即使在构造函数的最后一行也不要泄漏```this```指针 （因为该类可能是基类，先于派生类构造，可能执行完最后一行后还会继续执行派生类的构造）

不要这么做：
```cpp
class Foo : public Observer{
  public:
    Foo( Observer* s){
      s->register_(this); 
    }

    virtual void update();
};
```
应当选择这种做法：
```cpp
class Foo : public Observer{
  public:
    Foo();
    virtual void update();

    void observe( Observer* s){
      s->register_(this);
    }
};
Foo* pFoo = new Foo;
Observer* s = getSubject();
pFoo->observe(s);   //二段式构造（构造函数+initialize()，有时是解决问题的好办法 ）。或者直接写 s->register_(pFoo);
```
4. 析构时很复杂
作为数据成员的mutex不能保护析构。因为析构函数会把mutex成员变量给析构了，所以无法使用mutex实现线程安全。
而且，对于基类对象，当调用到基类析构函数的时候，派生类对象的那部分已经析构好了，基类对象所拥有的mutexlock不能保护整个析构过程。

为了保证析构函数的线程安全性，可以采用以下几种方法：
```cpp
1. 使用互斥锁：在析构函数中使用互斥锁来保护共享资源，以确保同一时间只有一个线程可以访问该资源。
class Foo {
public:
  ~Foo() {
    std::lock_guard<std::mutex> lock(mutex_);
    // 释放资源
  }
private:
  std::mutex mutex_;
};
```
这段代码使用了C++11中的```std::lock_guard<std::mutex>```来保护共享资源```mutex_```，以确保同一时间只有一个线程可以访问该资源。```std::lock_guard```是一个RAII类，它在构造函数中获取锁，在析构函数中释放锁，从而保证了锁的正确使用。

如果你需要在多线程环境下访问共享资源，可以考虑使用```std::lock_guard```来保证线程安全性。

```std::lock_guard```的主要目的是简化互斥量的使用，确保在退出作用域时互斥量自动解锁，从而避免忘记手动解锁导致的死锁或资源泄漏。
当创建```std::lock_guard```对象时，它会自动锁定互斥量。一旦到达```std::lock_guard```对象的作用域结束，**无论是正常执行还是异常退出**，```std::lock_guard```对象的析构函数将自动被调用，释放互斥量的锁。这样确保了互斥量在合适的时候被解锁，避免了手动解锁的疏忽。

```cpp
2. 2. 使用原子操作：在析构函数中使用原子操作来保护共享资源，以确保同一时间只有一个线程可以修改该资源。
class Foo {
public:
  ~Foo() {
    std::atomic<bool>& flag = getFlag();
    while (flag.exchange(true)) {
      std::this_thread::yield();
    }
    // 释放资源
    flag.store(false);
  }
private:
  std::atomic<bool>& getFlag() {
    static std::atomic<bool> flag(false);
    return flag;
  }
};
```
```std::atomic```是C++标准库中提供的一个模板类，用于实现原子操作。它是为了在多线程环境下对共享数据进行安全访问而设计的。
```std::atomic```可以用于实现对特定类型的原子操作，例如读取、写入、交换和比较等。它保证了这些操作的原子性，即这些操作要么完全执行，要么完全不执行，不会发生中间状态的干扰。

```std::atomic```的主要特点和用途如下：

**1**.原子性操作：```std::atomic```提供了一组函数和操作符，可以对共享变量进行原子操作。这意味着多个线程可以同时对同一变量进行读写操作，而不会导致数据竞争或未定义的行为。

**2**.内存顺序：```std::atomic```还提供了一些内存顺序的控制机制，用于指定对共享变量的读写操作在多线程间的可见性和顺序性。可以使用不同的内存顺序来平衡性能和一致性需求。

**3**.支持的类型：```std::atomic```可以用于各种内置数据类型（如整数、指针）以及一些特定的自定义类型。可以通过模板参数来指定要操作的数据类型。

使用```std::atomic```的一般步骤如下：
**1**定义一个std::atomic对象，指定要进行原子操作的数据类型。
**2**使用提供的成员函数或操作符来执行原子操作，如加载（load）、存储（store）、交换（exchange）、比较并交换（compare_exchange_weak/compare_exchange_strong）等。


