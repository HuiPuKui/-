# Thread
## ��
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

    bool started_; // �Ƿ�ʼ
    bool joined_; // �Ƿ� join �� A��ִ�� B.join()   A �߳���Ҫ�ȴ� B �߳����н������ܼ�������
    std::shared_ptr<std::thread> thread_; // �߳�
    pid_t tid_; // �̺߳�
    ThreadFunc func_; // ����
    std::string name_; // ��
    static std::atomic_int numCreated_; // �����ĸ���
};
```
## ��ʼ��
```cpp
std::atomic_int Thread::numCreated_(0);
```
## Thread(ThreadFunc, const std::string &name = std::string());
���캯����ʼ��һϵ�г�Ա�����û���ָ�Ĭ������һ������
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
�������߳�
```cpp
Thread::~Thread()
{
    if (started_ && !joined_)
    {
        thread_->detach(); // thread ���ṩ�����÷����̵߳ķ���
    }
}
```
## void start();
����һ���µ��߳�
```cpp
void Thread::start() // һ�� Thread ���󣬼�¼�ľ���һ�����̵߳���ϸ��Ϣ
{
    started_ = true;
    sem_t sem;
    sem_init(&sem, false, 0);
    // �����߳�
    thread_ = std::shared_ptr<std::thread>(new std::thread([&](){
        // ��ȡ�̵߳� tid ֵ
        tid_ = CurrentThread::tid();
        sem_post(&sem);
        // ����һ�����̣߳�ר��ִ���̺߳���
        func_(); 
    }));

    // �������ȴ���ȡ�����´������̵߳� tid ֵ
    sem_wait(&sem);
}
```
## void join();
join �������Ϊ true��������ϵͳ�� join ����
```cpp
void Thread::join()
{
    joined_ = true;
    thread_->join();
}
```
## bool started() const;
�Ƿ�ʼ
```cpp
bool Thread::started() const
{
    return started_;
}
```
## pid_t tid() const;
�õ��̺߳�
```cpp
pid_t Thread::tid() const
{
    return tid_;
}
```
## const std::string& name() const;
�õ��̵߳�����
```cpp
const std::string& Thread::name() const
{
    return name_;
}
```
## static int numCreated();
�õ������ĸ���
```cpp
int Thread::numCreated()
{
    return numCreated_;
}
```
## void setDefaultName();
ʹ�ø�����ΪĬ������
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