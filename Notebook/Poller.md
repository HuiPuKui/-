# Poller
## ��
```cpp
#pragma once

#include <vector>
#include <unordered_map>

#include "noncopyable.h"
#include "Timestamp.h"

class Channel;
class EventLoop;

// muduo ���ж�·�¼��ַ����ĺ��� IO ����ģ��
class Poller : noncopyable
{
public:
    using ChannelList = std::vector<Channel*>;

    Poller(EventLoop *loop);
    virtual ~Poller() = default;

    // ������ IO ���ñ���ͬ��Ľӿ�
    virtual Timestamp poll(int timeoutMs, ChannelList *activeChannels) = 0;
    virtual void updateChannel(Channel *channel) = 0;
    virtual void removeChannel(Channel *channel) = 0;

    // �жϲ��� channel �Ƿ��ڵ�ǰ Poller ����
    bool hasChannel(Channel *channel) const;

    // EventLoop ����ͨ���ýӿڻ�ȡĬ�ϵ� IO ���þ���ʵ��
    static Poller *newDefaultPoller(EventLoop *loop);
protected:
    // map �� key��sockfd   value��sockfd ������ channel ͨ������
    using ChannelMap = std::unordered_map<int, Channel*>;
    ChannelMap channels_;
private:
    EventLoop *ownerLoop_; // ���� Poller �������¼�ѭ�� EventLoop
};
```
## �麯��������������ʵ�֣�
```cpp
virtual Timestamp poll(int timeoutMs, ChannelList *activeChannels) = 0;
virtual void updateChannel(Channel *channel) = 0;
virtual void removeChannel(Channel *channel) = 0;
virtual ~Poller() = default;
```
## Poller(EventLoop *loop);
���캯����ʹ���¼�ѭ������
```cpp
Poller::Poller(EventLoop *loop)
  : ownerLoop_(loop)
{
}
```
## bool hasChannel(Channel *channel) const;
�ж� channel �Ƿ��� unordered_map ��
```cpp
bool Poller::hasChannel(Channel *channel) const
{
    auto it = channels_.find(channel->fd());
    return it != channels_.end() && it->second == channel;
}
```
## static Poller *newDefaultPoller(EventLoop *loop);
�ú���λ�� DefaultPoller.cc     
���û�Ĭ������ EPollPoller
```cpp
#include <stdlib.h>

#include "EPollPoller.h"
#include "Poller.h"

Poller *Poller::newDefaultPoller(EventLoop *loop)
{
    if (::getenv("MUDUO_USE_POLL"))
    {
        return nullptr; // ���� poll ��ʵ��
    }
    else
    {
        return new EPollPoller(loop); // ���� epoll ��ʵ��
    }
}
```