# Poller
## 类
```cpp
#pragma once

#include <vector>
#include <unordered_map>

#include "noncopyable.h"
#include "Timestamp.h"

class Channel;
class EventLoop;

// muduo 库中多路事件分发器的核心 IO 复用模块
class Poller : noncopyable
{
public:
    using ChannelList = std::vector<Channel*>;

    Poller(EventLoop *loop);
    virtual ~Poller() = default;

    // 给所有 IO 复用保留同意的接口
    virtual Timestamp poll(int timeoutMs, ChannelList *activeChannels) = 0;
    virtual void updateChannel(Channel *channel) = 0;
    virtual void removeChannel(Channel *channel) = 0;

    // 判断参数 channel 是否在当前 Poller 当中
    bool hasChannel(Channel *channel) const;

    // EventLoop 可以通过该接口获取默认的 IO 复用具体实现
    static Poller *newDefaultPoller(EventLoop *loop);
protected:
    // map 的 key：sockfd   value：sockfd 所属的 channel 通道类型
    using ChannelMap = std::unordered_map<int, Channel*>;
    ChannelMap channels_;
private:
    EventLoop *ownerLoop_; // 定义 Poller 所属的事件循环 EventLoop
};
```
## 虚函数（在派生类中实现）
```cpp
virtual Timestamp poll(int timeoutMs, ChannelList *activeChannels) = 0;
virtual void updateChannel(Channel *channel) = 0;
virtual void removeChannel(Channel *channel) = 0;
virtual ~Poller() = default;
```
## Poller(EventLoop *loop);
构造函数：使用事件循环构造
```cpp
Poller::Poller(EventLoop *loop)
  : ownerLoop_(loop)
{
}
```
## bool hasChannel(Channel *channel) const;
判断 channel 是否在 unordered_map 中
```cpp
bool Poller::hasChannel(Channel *channel) const
{
    auto it = channels_.find(channel->fd());
    return it != channels_.end() && it->second == channel;
}
```
## static Poller *newDefaultPoller(EventLoop *loop);
该函数位于 DefaultPoller.cc     
调用会默认生成 EPollPoller
```cpp
#include <stdlib.h>

#include "EPollPoller.h"
#include "Poller.h"

Poller *Poller::newDefaultPoller(EventLoop *loop)
{
    if (::getenv("MUDUO_USE_POLL"))
    {
        return nullptr; // 生成 poll 的实例
    }
    else
    {
        return new EPollPoller(loop); // 生成 epoll 的实例
    }
}
```