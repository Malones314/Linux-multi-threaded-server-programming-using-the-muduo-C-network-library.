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
