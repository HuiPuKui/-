# Channel
## 类
```cpp
#pragma once

#include <functional>
#include <memory>

#include "noncopyable.h"
#include "Timestamp.h"

class EventLoop;

/**
 * 理清楚：EventLoop、Channel、Poller之间的关系 <- Reactor 模型上对应 Demultiplex
 * Channel 理解为通道，封装了 sockfd 和其感兴趣的 event，如 EPOLLIN、EPOLLOUT 事件
 * 还绑定了 poller 返回的具体事件 
 */
class Channel : noncopyable
{
public:
    using EventCallback = std::function<void()>;
    using ReadEventCallback = std::function<void(Timestamp)>;

    Channel(EventLoop *loop, int fd);
    ~Channel();

    // fd 得到 poller 通知以后，处理事件的，调用相应的回调方法
    void handleEvent(Timestamp receiveTime);

    // 设置回调函数对象
    void setReadCallback(ReadEventCallback cb);
    void setWriteCallback(EventCallback cb);
    void setCloseCallback(EventCallback cb);
    void setErrorCallback(EventCallback cb);

    // 防止当 channel 被手动 remove 掉，channel 还在执行回调操作
    void tie(const std::shared_ptr<void>&);

    int fd();
    int events() const;
    void set_revents(int revt);
    
    // 设置 fd 相应的事件状态
    void enableReading();
    void disableReading();
    void enableWriting();
    void disableWriting();
    void disableAll();
    
    // 返回 fd 当前的事件状态
    bool isNoneEvent() const;
    bool isWriting() const;
    bool isReading() const;

    int index();
    void set_index(int idx);

    // one loop per thread
    // 当前 Channel 属于哪个 EventLoop
    EventLoop *ownerLoop();
    void remove();
private:
    void update();
    void handleEventWithGuard(Timestamp receiveTime);

    // 感兴趣的事件信息的状态的描述
    static const int kNoneEvent; // 无事件
    static const int kReadEvent; // 读事件
    static const int kWriteEvent; // 写事件

    EventLoop *loop_; // 事件循环
    const int fd_;    // fd, Poller 监听的对象
    int events_; // 注册 fd 感兴趣的事件
    int revents_; // poller 返回的具体发生的事件
    int index_; // channel 目前的状态

    std::weak_ptr<void> tie_;
    bool tied_;

    // 因为 Channel 通道里面能够获知 fd 最终发生的具体的事件 revents，所以它负责调用具体事件的回调操作
    ReadEventCallback readCallback_; // 读事件的回调函数
    EventCallback writeCallback_; // 写事件的回调函数
    EventCallback closeCallback_; // 关闭事件的回调函数
    EventCallback errorCallback_; // 发生错误事件的回调函数
};
```
## 初始化
```cpp
const int Channel::kNoneEvent = 0;
const int Channel::kReadEvent = EPOLLIN | EPOLLPRI;
const int Channel::kWriteEvent = EPOLLOUT;
```
## Channel(EventLoop *loop, int fd);
参数：channel 所属的事件循环、channel 关心的套接字的文件描述符
```cpp
Channel::Channel(EventLoop *loop, int fd) 
  : loop_(loop)
  , fd_(fd)
  , events_(0)
  , revents_(0)
  , index_(-1)
  , tied_(false)
{
}
```
## ~Channel();
无操作
```cpp
Channel::~Channel()
{

}
```
## void handleEvent(Timestamp receiveTime);
通过判断 tied_ 可以在处理事件时及时进行有效的保护和判断
```cpp
void Channel::handleEvent(Timestamp receiveTime)
{
    if (tied_)
    {
        std::shared_ptr<void> guard = tie_.lock();
        if (guard)
        {
            handleEventWithGuard(receiveTime);
        }
    }
    else
    {
        handleEventWithGuard(receiveTime);
    }
}
```
## void setReadCallback(ReadEventCallback cb);
设置读事件的回调函数
```cpp
void Channel::setReadCallback(ReadEventCallback cb)
{
    readCallback_ = std::move(cb);
}
```
## void setWriteCallback(EventCallback cb);
设置写事件的回调函数
```cpp
void Channel::setWriteCallback(EventCallback cb)
{
    writeCallback_ = std::move(cb);
}
```
## void setCloseCallback(EventCallback cb);
设置关闭事件的回调函数
```cpp
void Channel::setCloseCallback(EventCallback cb)
{
    closeCallback_ = std::move(cb);
}
```
## void setErrorCallback(EventCallback cb);
设置错误事件的回调函数
```cpp
void Channel::setErrorCallback(EventCallback cb)
{
    errorCallback_ = std::move(cb);
}
```
## void tie(const std::shared_ptr<void>&);
绑定
```cpp
// channel 的 tie 方法什么时候调用过？一个 TcpConnection 新连接创建的时候 TcpConnection => Channel
void Channel::tie(const std::shared_ptr<void> &obj)
{
    tie_ = obj;
    tied_ = true;
}
```
## int fd();
得到 fd_
```cpp
int Channel::fd()
{
    return fd_;
}
```
## int events() const;
得到感兴趣的事件
```cpp
int Channel::events() const
{
    return events_;
}
```
## void set_revents(int revt);
设置返回的事件
```cpp
void Channel::set_revents(int revt)
{
    revents_ = revt;
}
```
## void enableReading();
设置对读感兴趣
```cpp
void Channel::enableReading()
{
    events_ |= kReadEvent;
    update();
}
```
## void disableReading();
设置对读不感兴趣
```cpp
void Channel::disableReading()
{
    events_ &= ~kReadEvent;
    update();
}
```
## void enableWriting();
设置对写感兴趣
```cpp
void Channel::enableWriting()
{
    events_ |= kWriteEvent;
    update();
}
```
## void disableWriting();
设置对写不感兴趣
```cpp
void Channel::disableWriting()
{
    events_ &= ~kWriteEvent;
    update();
}
```
## void disableAll();
设置对所有事件都不感兴趣
```cpp
void Channel::disableAll()
{
    events_ = kNoneEvent;
    update();
}
```
## bool isNoneEvent() const;
是否对所有事件都不感兴趣
```cpp
bool Channel::isNoneEvent() const
{
    return events_ == kNoneEvent;
}
```
## bool isWriting() const;
是否对写感兴趣
```cpp
bool Channel::isWriting() const
{
    return events_ & kWriteEvent;
}
```
## bool isReading() const;
是否对读感兴趣
```cpp
bool Channel::isReading() const
{
    return events_ & kReadEvent;
}
```
## int index();
得到目前的状态
```cpp
int Channel::index()
{
    return index_;
}
```
## void set_index(int idx);
设置状态
```cpp
void Channel::set_index(int idx)
{
    index_ = idx;
}
```
## EventLoop *ownerLoop();
得到本 channel 所属的事件循环
```cpp
EventLoop *Channel::ownerLoop()
{
    return loop_;
}
```
## void remove();
channel => EventLoop => poller
```cpp
void Channel::remove()
{
    loop_->removeChannel(this);
}
```
## void update();
channel => EventLoop => poller
```cpp
void Channel::update()
{
    // 通过 Channel 所属的 EventLoop，调用 poller 的相应方法，注册 fd 的 events 事件
    loop_->updateChannel(this);
}
```
## void handleEventWithGuard(Timestamp receiveTime);
根据 poller 通知的 channel 发生的具体事件，由 channel 负责调用具体的回调函数
```cpp
void Channel::handleEventWithGuard(Timestamp receiveTime)
{
    LOG_INFO("Channel handleEvent revents:%d\n", revents_);

    if ((revents_ & EPOLLHUP) && !(revents_ & EPOLLIN))
    {
        if (closeCallback_)
        {
            closeCallback_();
        }
    }

    if (revents_ & EPOLLERR)
    {
        if (errorCallback_)
        {
            errorCallback_();
        }
    }

    if (revents_ & (EPOLLIN | EPOLLPRI))
    {
        if (readCallback_)
        {
            readCallback_(receiveTime);
        }
    }

    if (revents_ & EPOLLOUT)
    {
        if (writeCallback_)
        {
            writeCallback_();
        }
    }
}
```