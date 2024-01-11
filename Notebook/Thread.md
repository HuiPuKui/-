# Thread
## 类
```cpp
#pragma once

#include <functional>
#include <thread>
#include <memory>
#include <unistd.h>
#include <string>
#include <atomic>

#include "noncopyable.h"

class Thread : noncopyable
{
public:
    using ThreadFunc = std::function<void()>;

    explicit Thread(ThreadFunc, const std::string &name = std::string());
    ~Thread();

    void start();
    void join();

    bool started() const;
    pid_t tid() const;
    const std::string& name() const;

    static int numCreated();
private:
    void setDefaultName();

    bool started_; // 是否开始
    bool joined_; // 是否 join ： A中执行 B.join()   A 线程需要等待 B 线程运行结束才能继续运行
    std::shared_ptr<std::thread> thread_; // 线程
    pid_t tid_; // 线程号
    ThreadFunc func_; // 函数
    std::string name_; // 名
    static std::atomic_int numCreated_; // 创建的个数
};
```
## 初始化
```cpp
std::atomic_int Thread::numCreated_(0);
```
## Thread(ThreadFunc, const std::string &name = std::string());
构造函数初始化一系列成员，如果没名字给默认设置一个名字
```cpp
Thread::Thread(ThreadFunc func, const std::string &name)
  : started_(false)
  , joined_(false)
  , tid_(0)
  , func_(std::move(func))
  , name_(name)
{
    setDefaultName();
}
```
## ~Thread();
分离子线程
```cpp
Thread::~Thread()
{
    if (started_ && !joined_)
    {
        thread_->detach(); // thread 类提供的设置分离线程的方法
    }
}
```
## void start();
开启一个新的线程
```cpp
void Thread::start() // 一个 Thread 对象，记录的就是一个新线程的详细信息
{
    started_ = true;
    sem_t sem;
    sem_init(&sem, false, 0);
    // 开启线程
    thread_ = std::shared_ptr<std::thread>(new std::thread([&](){
        // 获取线程的 tid 值
        tid_ = CurrentThread::tid();
        sem_post(&sem);
        // 开启一个新线程，专门执行线程函数
        func_(); 
    }));

    // 这里必须等待获取上面新创建的线程的 tid 值
    sem_wait(&sem);
}
```
## void join();
join 标记设置为 true，并调用系统的 join 函数
```cpp
void Thread::join()
{
    joined_ = true;
    thread_->join();
}
```
## bool started() const;
是否开始
```cpp
bool Thread::started() const
{
    return started_;
}
```
## pid_t tid() const;
得到线程号
```cpp
pid_t Thread::tid() const
{
    return tid_;
}
```
## const std::string& name() const;
得到线程的名字
```cpp
const std::string& Thread::name() const
{
    return name_;
}
```
## static int numCreated();
得到创建的个数
```cpp
int Thread::numCreated()
{
    return numCreated_;
}
```
## void setDefaultName();
使用个数作为默认名字
```cpp
void Thread::setDefaultName()
{
    int num = ++numCreated_;
    if (name_.empty())
    {
        char buf[32] = {0};
        snprintf(buf, sizeof buf, "Thread%d", num);
        name_ = buf;
    }
}
```