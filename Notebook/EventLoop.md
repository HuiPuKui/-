# EventLoop
## 类
```cpp
#pragma once

#include <functional>
#include <vector>
#include <atomic>
#include <memory>
#include <mutex>

#include "noncopyable.h"
#include "Timestamp.h"
#include "CurrentThread.h"

class Channel;
class Poller;

// 时间循环类：主要包含了两个大模块 Channel     Poller  (epoll 的抽象)
class EventLoop : noncopyable
{
public:
    using Functor = std::function<void()>;

    EventLoop();
    ~EventLoop();

    // 开启事件循环
    void loop();
    // 退出事件循环
    void quit();

    Timestamp pollReturnTime() const;

    // 在当前 loop 中执行
    void runInLoop(Functor cb);
    // 把 cb 放入队列中，唤醒 loop 所在的线程，执行 cb
    void queueInLoop(Functor cb);

    // 用来唤醒 loop 所在的线程的
    void wakeup();

    // EventLoop 的方法 => Poller 的方法
    void updateChannel(Channel *channel);
    void removeChannel(Channel *channel);
    bool hasChannel(Channel *channel);

    // 判断 EventLoop 对象是否在对象自己的线程里面
    bool isInLoopThread() const;
private:
    void handleRead(); // wake up
    void doPendingFunctors(); // 执行回调

    using ChannelList = std::vector<Channel*>;

    std::atomic_bool looping_; // 原子操作，通过 CAS 实现的
    std::atomic_bool quit_; // 标志退出 loop 循环
    
    const pid_t threadId_; // 记录当前 loop 所在线程的 id
    Timestamp pollReturnTime_; // poller 返回发生事件的 channels 的时间点
    std::unique_ptr<Poller> poller_;

    int wakeupFd_; // 主要作用：当 mainLoop 获取一个新用户的 channel，通过轮询算法选择一个 subLoop，通过该成员唤醒 subLoop 处理 channel
    std::unique_ptr<Channel> wakeupChannel_;

    ChannelList activeChannels_;

    std::atomic_bool callingPendingFunctors_; // 标识当前 loop 是否有需要执行的回调操作
    std::vector<Functor> pendingFunctors_; // 存储 loop 需要执行的所有的回调操作
    std::mutex mutex_; // 互斥锁，用来保护上面 vector 容器的线程安全操作
};
```
## 初始化
```cpp
// 防止一个线程创建多个 EventLoop  thread_local
__thread EventLoop *t_loopInThisThread = nullptr;

// 定义默认的 Poller IO 复用接口的超时时间
const int kPollTimeMs = 10000;
```
## int createEventfd();
创建一个 wakeupFd，用来 nodify 唤醒 subReactor 处理新来的 channel
```cpp
int createEventfd()
{
    int evtfd = ::eventfd(0, EFD_NONBLOCK | EFD_CLOEXEC);
    if (evtfd < 0)
    {
        LOG_FATAL("eventfd err:%d \n", errno);
    }
    return evtfd;
}
```
## EventLoop();
threadId_：调用系统函数得到当前线程的 Id    
poller_：默认采用 epoll     
wakeupFd_：专门用来唤醒的文件描述符     
wakeupChannel_：专门用来唤醒的 Channel      
设置 wakeupFd 感兴趣的事件以及事件发生后需要调用的回调函数      
```cpp
EventLoop::EventLoop()
  : looping_(false)
  , quit_(false)
  , callingPendingFunctors_(false)
  , threadId_(CurrentThread::tid())
  , poller_(Poller::newDefaultPoller(this))
  , wakeupFd_(createEventfd())
  , wakeupChannel_(new Channel(this, wakeupFd_))
{
    LOG_DEBUG("EventLoop created %p in thread %d \n", this, threadId_);
    if (t_loopInThisThread)
    {
        LOG_FATAL("Another EventLoop %p exists in this thread %d \n", t_loopInThisThread, threadId_);
    }
    else
    {
        t_loopInThisThread = this;
    }

    // 设置 wakeupfd 的事件类型以及发生事件后的回调操作
    wakeupChannel_->setReadCallback(std::bind(&EventLoop::handleRead, this));
    // 每一个 EventLoop 都将监听 wakeupchannel 的 EPOLLIN 读事件了
    wakeupChannel_->enableReading();
}
```
## ~EventLoop();
取消掉 channel 感兴趣的事件     
在 poller 中移除 channel        
关闭文件描述符      
将这个线程中的 loop 的指针置空      
```cpp
EventLoop::~EventLoop()
{
    wakeupChannel_->disableAll();
    wakeupChannel_->remove();
    ::close(wakeupFd_);
    t_loopInThisThread = nullptr;
}
```
## void loop();
首先标记事件循环开始，未退出        
之后开始循环，循环中不断通过 poller 轮询监听是否有事件发生 ，并将所有事件的 channel 放入 activeChannels 中      
在一次返回之后，遍历 activeChannels，执行 每个 channel 对应的回调函数       
执行完回调之后还要看 pendingFunctors 中保存的未执行的回调，将其执行，执行结束后开启下一轮循环       
循环结束后将标志位设为 false        
```cpp
void EventLoop::loop()
{
    looping_ = true;
    quit_ = false;

    LOG_INFO("EventLoop %p start looping \n", this);

    while (!quit_)
    {
        activeChannels_.clear();
        // 监听两类 fd 一种是 client 的 fd，一种 wakeupfd
        pollReturnTime_ = poller_->poll(kPollTimeMs, &activeChannels_);
        for (Channel *channel : activeChannels_)
        {
            // Poller 监听哪些 channel 发生事件了，然后上报给 EventLoop，通知 channel 处理相应的事件
            channel->handleEvent(pollReturnTime_);
        }
        // 执行当前 EventLoop 事件循环需要处理的回调操作
        /**
         * IO 线程 mainLoop accept fd 《= channel subloop
         * mainLoop 事先注册一个回调 cb（需要 subloop 来执行） wakeup  subloop 后执行下面的方法，执行之前 mainloop 注册的 cb 操作
         */
        doPendingFunctors();
    }

    LOG_INFO("EventLoop %p stop looping. \n", this);
    looping_ = false;
}
```
## void quit();
退出事件循环：      
改变标志位为 false      
1. 在自己的线程中调用 quit：直接执行
2. 在非 loop 线程中调用 loop 的 quit：需要进行唤醒
```cpp
void EventLoop::quit()
{
    quit_ = true;

    if (!isInLoopThread()) // 如果是在其他线程中，调用 quit   在一个 subloop（woker） 中，调用了 mainLoop（IO） 的 quit
    {
        wakeup();
    }
}
```
## Timestamp pollReturnTime() const;
获得 poller 返回事件的时间点
```cpp
Timestamp EventLoop::pollReturnTime() const
{
    return pollReturnTime_;
}
```
## void runInLoop(Functor cb);
需要看是否在当前线程中执行：    
是：直接调用回调函数        
否：执行 queueInLoop()
```cpp
void EventLoop::runInLoop(Functor cb)
{
    if (isInLoopThread()) // 在当前的 loop 线程中执行 cb
    {
        cb();
    }
    else // 在非 loop 线程中执行 cb，就需要唤醒 loop 所在线程，执行 cb
    {
        queueInLoop(cb);
    }
}
```
## void queueInLoop(Functor cb);
首先要上锁因为要操作队列了   
唤醒
```cpp
void EventLoop::queueInLoop(Functor cb)
{
    {
        std::unique_lock<std::mutex> lock(mutex_);
        pendingFunctors_.emplace_back(cb);
    }

    // 唤醒相应的，需要执行上面回调操作的 loop 的线程
    // callingPendingFunctors_ 的意思是：当前 loop 正在执行回调，但是 loop 又有了新的回调
    if (!isInLoopThread() || callingPendingFunctors_)
    {
        wakeup(); // 唤醒 loop 所在线程
    }
}
```
## void wakeup();
随便向要唤醒的 fd 写一个数据，wakeupChannel 就会发生一个读事件，loop 就会被唤醒
```cpp
void EventLoop::wakeup()
{
    uint64_t one = 1;
    ssize_t n = write(wakeupFd_, &one, sizeof one);
    if (n != sizeof one)
    {
        LOG_ERROR("EventLoop::wakeup() writes %lu bytes instead of 8 \n", n);
    }
}
```
## void updateChannel(Channel *channel);
EventLoop => Poller
```cpp
void EventLoop::updateChannel(Channel *channel)
{
    poller_->updateChannel(channel);
}
```
## void removeChannel(Channel *channel);
EventLoop => Poller
```cpp
void EventLoop::removeChannel(Channel *channel)
{
    poller_->removeChannel(channel);
}
```
## bool hasChannel(Channel *channel);
EventLoop => Poller
```cpp
bool EventLoop::hasChannel(Channel *channel)
{
    return poller_->hasChannel(channel);
}
```
## bool isInLoopThread() const;
判断是否执行在当前线程
```cpp
bool EventLoop::isInLoopThread() const
{
    return threadId_ == CurrentThread::tid();
}
```
## void handleRead();
唤醒事件专门的读回调函数
```cpp
void EventLoop::handleRead()
{
    uint64_t one = 1;
    ssize_t n = read(wakeupFd_, &one, sizeof one);
    if (n != sizeof one)
    {
        LOG_ERROR("EventLoop::handleRead() reads %ld bytes instead of 8", n);
    }
}
```
## void doPendingFunctors(); 
这个地方很妙：不是直接操作原来的队列，而是选择上锁然后直接交换      
这样可以拿到之前需要进行的回调，而且不会影响在执行过程中新来的放入队列
```cpp
void EventLoop::doPendingFunctors()
{
    std::vector<Functor> functors;
    callingPendingFunctors_ = true;

    {
        std::unique_lock<std::mutex> lock(mutex_);
        functors.swap(pendingFunctors_);
    }

    for (const Functor &functor : functors)
    {
        functor(); // 执行当前 loop 需要执行的回调操作
    }

    callingPendingFunctors_ = false;
}
```