# EventLoop
## ��
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

// ʱ��ѭ���ࣺ��Ҫ������������ģ�� Channel     Poller  (epoll �ĳ���)
class EventLoop : noncopyable
{
public:
    using Functor = std::function<void()>;

    EventLoop();
    ~EventLoop();

    // �����¼�ѭ��
    void loop();
    // �˳��¼�ѭ��
    void quit();

    Timestamp pollReturnTime() const;

    // �ڵ�ǰ loop ��ִ��
    void runInLoop(Functor cb);
    // �� cb ��������У����� loop ���ڵ��̣߳�ִ�� cb
    void queueInLoop(Functor cb);

    // �������� loop ���ڵ��̵߳�
    void wakeup();

    // EventLoop �ķ��� => Poller �ķ���
    void updateChannel(Channel *channel);
    void removeChannel(Channel *channel);
    bool hasChannel(Channel *channel);

    // �ж� EventLoop �����Ƿ��ڶ����Լ����߳�����
    bool isInLoopThread() const;
private:
    void handleRead(); // wake up
    void doPendingFunctors(); // ִ�лص�

    using ChannelList = std::vector<Channel*>;

    std::atomic_bool looping_; // ԭ�Ӳ�����ͨ�� CAS ʵ�ֵ�
    std::atomic_bool quit_; // ��־�˳� loop ѭ��
    
    const pid_t threadId_; // ��¼��ǰ loop �����̵߳� id
    Timestamp pollReturnTime_; // poller ���ط����¼��� channels ��ʱ���
    std::unique_ptr<Poller> poller_;

    int wakeupFd_; // ��Ҫ���ã��� mainLoop ��ȡһ�����û��� channel��ͨ����ѯ�㷨ѡ��һ�� subLoop��ͨ���ó�Ա���� subLoop ���� channel
    std::unique_ptr<Channel> wakeupChannel_;

    ChannelList activeChannels_;

    std::atomic_bool callingPendingFunctors_; // ��ʶ��ǰ loop �Ƿ�����Ҫִ�еĻص�����
    std::vector<Functor> pendingFunctors_; // �洢 loop ��Ҫִ�е����еĻص�����
    std::mutex mutex_; // �������������������� vector �������̰߳�ȫ����
};
```
## ��ʼ��
```cpp
// ��ֹһ���̴߳������ EventLoop  thread_local
__thread EventLoop *t_loopInThisThread = nullptr;

// ����Ĭ�ϵ� Poller IO ���ýӿڵĳ�ʱʱ��
const int kPollTimeMs = 10000;
```
## int createEventfd();
����һ�� wakeupFd������ nodify ���� subReactor ���������� channel
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
threadId_������ϵͳ�����õ���ǰ�̵߳� Id    
poller_��Ĭ�ϲ��� epoll     
wakeupFd_��ר���������ѵ��ļ�������     
wakeupChannel_��ר���������ѵ� Channel      
���� wakeupFd ����Ȥ���¼��Լ��¼���������Ҫ���õĻص�����      
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

    // ���� wakeupfd ���¼������Լ������¼���Ļص�����
    wakeupChannel_->setReadCallback(std::bind(&EventLoop::handleRead, this));
    // ÿһ�� EventLoop �������� wakeupchannel �� EPOLLIN ���¼���
    wakeupChannel_->enableReading();
}
```
## ~EventLoop();
ȡ���� channel ����Ȥ���¼�     
�� poller ���Ƴ� channel        
�ر��ļ�������      
������߳��е� loop ��ָ���ÿ�      
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
���ȱ���¼�ѭ����ʼ��δ�˳�        
֮��ʼѭ����ѭ���в���ͨ�� poller ��ѯ�����Ƿ����¼����� �����������¼��� channel ���� activeChannels ��      
��һ�η���֮�󣬱��� activeChannels��ִ�� ÿ�� channel ��Ӧ�Ļص�����       
ִ����ص�֮��Ҫ�� pendingFunctors �б����δִ�еĻص�������ִ�У�ִ�н���������һ��ѭ��       
ѭ�������󽫱�־λ��Ϊ false        
```cpp
void EventLoop::loop()
{
    looping_ = true;
    quit_ = false;

    LOG_INFO("EventLoop %p start looping \n", this);

    while (!quit_)
    {
        activeChannels_.clear();
        // �������� fd һ���� client �� fd��һ�� wakeupfd
        pollReturnTime_ = poller_->poll(kPollTimeMs, &activeChannels_);
        for (Channel *channel : activeChannels_)
        {
            // Poller ������Щ channel �����¼��ˣ�Ȼ���ϱ��� EventLoop��֪ͨ channel ������Ӧ���¼�
            channel->handleEvent(pollReturnTime_);
        }
        // ִ�е�ǰ EventLoop �¼�ѭ����Ҫ����Ļص�����
        /**
         * IO �߳� mainLoop accept fd ��= channel subloop
         * mainLoop ����ע��һ���ص� cb����Ҫ subloop ��ִ�У� wakeup  subloop ��ִ������ķ�����ִ��֮ǰ mainloop ע��� cb ����
         */
        doPendingFunctors();
    }

    LOG_INFO("EventLoop %p stop looping. \n", this);
    looping_ = false;
}
```
## void quit();
�˳��¼�ѭ����      
�ı��־λΪ false      
1. ���Լ����߳��е��� quit��ֱ��ִ��
2. �ڷ� loop �߳��е��� loop �� quit����Ҫ���л���
```cpp
void EventLoop::quit()
{
    quit_ = true;

    if (!isInLoopThread()) // ������������߳��У����� quit   ��һ�� subloop��woker�� �У������� mainLoop��IO�� �� quit
    {
        wakeup();
    }
}
```
## Timestamp pollReturnTime() const;
��� poller �����¼���ʱ���
```cpp
Timestamp EventLoop::pollReturnTime() const
{
    return pollReturnTime_;
}
```
## void runInLoop(Functor cb);
��Ҫ���Ƿ��ڵ�ǰ�߳���ִ�У�    
�ǣ�ֱ�ӵ��ûص�����        
��ִ�� queueInLoop()
```cpp
void EventLoop::runInLoop(Functor cb)
{
    if (isInLoopThread()) // �ڵ�ǰ�� loop �߳���ִ�� cb
    {
        cb();
    }
    else // �ڷ� loop �߳���ִ�� cb������Ҫ���� loop �����̣߳�ִ�� cb
    {
        queueInLoop(cb);
    }
}
```
## void queueInLoop(Functor cb);
����Ҫ������ΪҪ����������   
����
```cpp
void EventLoop::queueInLoop(Functor cb)
{
    {
        std::unique_lock<std::mutex> lock(mutex_);
        pendingFunctors_.emplace_back(cb);
    }

    // ������Ӧ�ģ���Ҫִ������ص������� loop ���߳�
    // callingPendingFunctors_ ����˼�ǣ���ǰ loop ����ִ�лص������� loop �������µĻص�
    if (!isInLoopThread() || callingPendingFunctors_)
    {
        wakeup(); // ���� loop �����߳�
    }
}
```
## void wakeup();
�����Ҫ���ѵ� fd дһ�����ݣ�wakeupChannel �ͻᷢ��һ�����¼���loop �ͻᱻ����
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
�ж��Ƿ�ִ���ڵ�ǰ�߳�
```cpp
bool EventLoop::isInLoopThread() const
{
    return threadId_ == CurrentThread::tid();
}
```
## void handleRead();
�����¼�ר�ŵĶ��ص�����
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
����ط��������ֱ�Ӳ���ԭ���Ķ��У�����ѡ������Ȼ��ֱ�ӽ���      
���������õ�֮ǰ��Ҫ���еĻص������Ҳ���Ӱ����ִ�й����������ķ������
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
        functor(); // ִ�е�ǰ loop ��Ҫִ�еĻص�����
    }

    callingPendingFunctors_ = false;
}
```