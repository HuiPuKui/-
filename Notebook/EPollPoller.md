# EPollPoller
## ��
```cpp
#pragma once

#include <vector>
#include <sys/epoll.h>

#include "Poller.h"
#include "Timestamp.h"

/**
 *  epoll ��ʹ��
 *  epoll_create
 *  epoll_ctl   add/modify/del
 *  epoll_wait
 */

class EPollPoller : public Poller
{
public:
    EPollPoller(EventLoop *loop);
    ~EPollPoller() override;

    // ��д���� Poller �ĳ��󷽷�
    Timestamp poll(int timeoutMs, ChannelList *activeChannels) override;
    void updateChannel(Channel *channel) override;
    void removeChannel(Channel *channel) override;
private:
    static const int kInitEventListSize = 16;

    // ��д��Ծ������
    void fillActiveChannels(int numEvents, ChannelList *activeChannels) const;
    // ���� channel ͨ��
    void update(int operation, Channel *channel);

    using EventList = std::vector<epoll_event>;

    int epollfd_; // epoll ���ļ�������
    EventList events_; // epoll �������¼�ѭ��
};
```
## ״̬
```cpp
const int kNew = -1; // �մ�����
const int kAdded = 1; // �Ѿ�����ӵ�
const int kDeleted = 2; // �Ѿ���ɾ����
```
## EPollPoller(EventLoop *loop);
���캯��������epoll�������¼�ѭ��������ϵͳ�������� epoll���õ�epollfd����ʼ�������¼��� vector �Գ�ʼ��С      
��� epoll ����ʧ�ܣ�Ҫ��һ�� FATAL ��־
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
����ϵͳ�����ر� epoll
```cpp
EPollPoller::~EPollPoller() 
{
    ::close(epollfd_);
}
```
## Timestamp poll(int timeoutMs, ChannelList *activeChannels) override;
���� epoll_wait �����з������¼����� events_ �У�����¼����     
�������������Ĵ���      
���浱ǰʱ���      

����ɹ��������¼�����ô��Ҫ���� fillActiveChannel �����з����¼���channel����  
������Ҫ�ж�һ����η����˶����¼�������¼����������ǵ�ʱ�޶��ĸ�����Ⱦʹ���ǰ�����Ĵ�С���ܻ᲻�����ˣ�����Ҫ�ֶ�����һ��      

����¼�����Ϊ��Ļ����ʹ�һ����ʱ����־        

�����д���������ӡһ���������־
```cpp
Timestamp EPollPoller::poll(int timeoutMs, ChannelList *activeChannels)
{
    // ʵ����Ӧ���� LOG_DEBUG �����־��Ϊ����
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
Ҫ���� channel ��״̬����Ҫ�õ�Ŀǰ��״̬���ٸ���Ŀǰ��״̬ȥ�Ƶ���һ��Ҫ��ɵ�״̬     
��� ��ǰ״̬�� kNew ���� kDeleted ��ô��һ��״̬��Ȼ���� kAdded  
����� kNew ��Ҫ�Ƚ� channel ��ӵ� unordered_map ��

����״̬�� kAdded ��Ҳ������ poller ��ע����ˣ����жϵ�ǰchannel�Ļ��ʲô       
���Ŀǰû�л�ͻᱻɾ��������Ϊ kDeleted     
����л�ͻ��ʶ�޸�
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
    else // channel �Ѿ��� poller ��ע�����
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
���ȸ��ݵ�ǰchannel���ļ��������� unordered_map ��ɾ��      
Ȼ���� channel ֮ǰ��״̬��������Ѿ���ӵ�״̬�͵���һ�� update ��epoll ��ɾ�������״̬����Ϊ New
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
�������� epoll_wait ��ѯ�������¼����õ�����channel�е�revents�У�����channel���� activeChannels����
```cpp
void EPollPoller::fillActiveChannels(int numEvents, ChannelList *activeChannels) const
{
    for (int i = 0; i < numEvents; ++i)
    {
        Channel *channel = static_cast<Channel*>(events_[i].data.ptr);
        channel->set_revents(events_[i].events);
        activeChannels->push_back(channel); // EventLoop ���õ������� poller �������ص����з����¼��� channel �б�
    }
}
```
## void update(int operation, Channel *channel);
���ݴ���Ĳ�����channel������Ҫ���еĲ���������epoll��ϵͳ���� epoll_ctl ���޸� channel ͨ��
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