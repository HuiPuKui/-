# EPollPoller
## 类
```cpp
#pragma once

#include <vector>
#include <sys/epoll.h>

#include "Poller.h"
#include "Timestamp.h"

/**
 *  epoll 的使用
 *  epoll_create
 *  epoll_ctl   add/modify/del
 *  epoll_wait
 */

class EPollPoller : public Poller
{
public:
    EPollPoller(EventLoop *loop);
    ~EPollPoller() override;

    // 重写基类 Poller 的抽象方法
    Timestamp poll(int timeoutMs, ChannelList *activeChannels) override;
    void updateChannel(Channel *channel) override;
    void removeChannel(Channel *channel) override;
private:
    static const int kInitEventListSize = 16;

    // 填写活跃的连接
    void fillActiveChannels(int numEvents, ChannelList *activeChannels) const;
    // 更新 channel 通道
    void update(int operation, Channel *channel);

    using EventList = std::vector<epoll_event>;

    int epollfd_; // epoll 的文件描述符
    EventList events_; // epoll 所属的事件循环
};
```
## 状态
```cpp
const int kNew = -1; // 刚创建的
const int kAdded = 1; // 已经被添加的
const int kDeleted = 2; // 已经被删除的
```
## EPollPoller(EventLoop *loop);
构造函数：设置epoll所属的事件循环，调用系统函数创建 epoll并得到epollfd，初始化放有事件的 vector 以初始大小      
如果 epoll 创建失败：要打一个 FATAL 日志
```cpp
EPollPoller::EPollPoller(EventLoop *loop)
  : Poller(loop)
  , epollfd_(::epoll_create1(EPOLL_CLOEXEC))
  , events_(kInitEventListSize)
{
    if (epollfd_ < 0)
    {
        LOG_FATAL("epoll_create error:%d \n", errno);
    }
}
```
## ~EPollPoller() override;
调用系统函数关闭 epoll
```cpp
EPollPoller::~EPollPoller() 
{
    ::close(epollfd_);
}
```
## Timestamp poll(int timeoutMs, ChannelList *activeChannels) override;
调用 epoll_wait 将所有发生的事件放入 events_ 中，并记录个数     
保存下来产生的错误      
保存当前时间戳      

如果成功返回了事件，那么就要调用 fillActiveChannel 将所有发生事件的channel返回  
另外需要判断一下这次发生了多少事件，如果事件个数和我们当时限定的个数相等就代表当前容器的大小可能会不够大了，就需要手动扩容一次      

如果事件个数为零的话，就打一个超时的日志        

否则即有错误发生，打印一个错误的日志
```cpp
Timestamp EPollPoller::poll(int timeoutMs, ChannelList *activeChannels)
{
    // 实际上应该用 LOG_DEBUG 输出日志更为合理
    LOG_INFO("func=%s => fd total count:%lu\n", __FUNCTION__, channels_.size());

    int numEvents = ::epoll_wait(epollfd_, &*events_.begin(), static_cast<int>(events_.size()), timeoutMs);
    int saveErrno = errno;
    Timestamp now(Timestamp::now());

    if (numEvents > 0)
    {
        LOG_INFO("%d events happened \n", numEvents);
        fillActiveChannels(numEvents, activeChannels);
        if (numEvents == events_.size())
        {
            events_.resize(events_.size() * 2);
        }
    }
    else if (numEvents == 0)
    {
        LOG_DEBUG("%s timeout! \n", __FUNCTION__);
    }
    else
    {
        if (saveErrno != EINTR)
        {
            errno = saveErrno;
            LOG_ERROR("EPollPoller::poll() err!");
        }
    }
    return now;
}
```
## void updateChannel(Channel *channel) override;
要更新 channel 的状态首先要得到目前的状态，再根据目前的状态去推导下一步要变成的状态     
如果 当前状态是 kNew 或者 kDeleted 那么下一个状态必然会变成 kAdded  
如果是 kNew 就要先将 channel 添加到 unordered_map 中

否则状态是 kAdded ，也就是在 poller 上注册过了，就判断当前channel的活动是什么       
如果目前没有活动就会被删除，并改为 kDeleted     
如果有活动就会标识修改
```cpp
void EPollPoller::updateChannel(Channel *channel)
{
    const int index = channel->index();
    LOG_INFO("func=%s fd=%d events=%d index=%d \n", __FUNCTION__, channel->fd(), channel->events(), index);

    if (index == kNew || index == kDeleted)
    {
        if (index == kNew)
        {
            int fd = channel->fd();
            channels_[fd] = channel;
        }

        channel->set_index(kAdded);
        update(EPOLL_CTL_ADD, channel);
    }
    else // channel 已经在 poller 上注册过了
    {
        int fd = channel->fd();
        if (channel->isNoneEvent())
        {
            update(EPOLL_CTL_DEL, channel);
            channel->set_index(kDeleted);
        }
        else
        {
            update(EPOLL_CTL_MOD, channel);
        }
    }
}
```
## void removeChannel(Channel *channel) override;
首先根据当前channel的文件描述符在 unordered_map 中删掉      
然后检查 channel 之前的状态，如果是已经添加的状态就调用一下 update 在epoll 中删除，最后将状态设置为 New
```cpp
void EPollPoller::removeChannel(Channel *channel) 
{
    int fd = channel->fd();
    channels_.erase(fd);
    
    LOG_INFO("func=%s fd=%d \n", __FUNCTION__, fd);

    int index = channel->index();
    if (index == kAdded)
    {
        update(EPOLL_CTL_DEL, channel);
    }
    channel->set_index(kNew);
}
```
## void fillActiveChannels(int numEvents, ChannelList *activeChannels) const;
这里会根据 epoll_wait 轮询出来的事件设置到所属channel中的revents中，并将channel放入 activeChannels返回
```cpp
void EPollPoller::fillActiveChannels(int numEvents, ChannelList *activeChannels) const
{
    for (int i = 0; i < numEvents; ++i)
    {
        Channel *channel = static_cast<Channel*>(events_[i].data.ptr);
        channel->set_revents(events_[i].events);
        activeChannels->push_back(channel); // EventLoop 就拿到了它的 poller 给它返回的所有发生事件的 channel 列表
    }
}
```
## void update(int operation, Channel *channel);
根据传入的参数：channel，和需要进行的操作，调用epoll的系统函数 epoll_ctl 来修改 channel 通道
```cpp
void EPollPoller::update(int operation, Channel *channel)
{
    epoll_event event;
    bzero(&event, sizeof event);

    int fd = channel->fd();

    event.events = channel->events();
    event.data.fd = fd;
    event.data.ptr = channel;
    
    if (::epoll_ctl(epollfd_, operation, fd, &event) < 0)
    {
        if (operation == EPOLL_CTL_DEL)
        {
            LOG_ERROR("epoll_ctl del error:%d \n", errno);
        }
        else
        {
            LOG_FATAL("epoll_ctl add/mod error:%d \n", errno);
        }
    }
}
```