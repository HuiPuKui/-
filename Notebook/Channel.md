# Channel
## ��
```cpp
#pragma once

#include <functional>
#include <memory>

#include "noncopyable.h"
#include "Timestamp.h"

class EventLoop;

/**
 * �������EventLoop��Channel��Poller֮��Ĺ�ϵ <- Reactor ģ���϶�Ӧ Demultiplex
 * Channel ���Ϊͨ������װ�� sockfd �������Ȥ�� event���� EPOLLIN��EPOLLOUT �¼�
 * ������ poller ���صľ����¼� 
 */
class Channel : noncopyable
{
public:
    using EventCallback = std::function<void()>;
    using ReadEventCallback = std::function<void(Timestamp)>;

    Channel(EventLoop *loop, int fd);
    ~Channel();

    // fd �õ� poller ֪ͨ�Ժ󣬴����¼��ģ�������Ӧ�Ļص�����
    void handleEvent(Timestamp receiveTime);

    // ���ûص���������
    void setReadCallback(ReadEventCallback cb);
    void setWriteCallback(EventCallback cb);
    void setCloseCallback(EventCallback cb);
    void setErrorCallback(EventCallback cb);

    // ��ֹ�� channel ���ֶ� remove ����channel ����ִ�лص�����
    void tie(const std::shared_ptr<void>&);

    int fd();
    int events() const;
    void set_revents(int revt);
    
    // ���� fd ��Ӧ���¼�״̬
    void enableReading();
    void disableReading();
    void enableWriting();
    void disableWriting();
    void disableAll();
    
    // ���� fd ��ǰ���¼�״̬
    bool isNoneEvent() const;
    bool isWriting() const;
    bool isReading() const;

    int index();
    void set_index(int idx);

    // one loop per thread
    // ��ǰ Channel �����ĸ� EventLoop
    EventLoop *ownerLoop();
    void remove();
private:
    void update();
    void handleEventWithGuard(Timestamp receiveTime);

    // ����Ȥ���¼���Ϣ��״̬������
    static const int kNoneEvent; // ���¼�
    static const int kReadEvent; // ���¼�
    static const int kWriteEvent; // д�¼�

    EventLoop *loop_; // �¼�ѭ��
    const int fd_;    // fd, Poller �����Ķ���
    int events_; // ע�� fd ����Ȥ���¼�
    int revents_; // poller ���صľ��巢�����¼�
    int index_; // channel Ŀǰ��״̬

    std::weak_ptr<void> tie_;
    bool tied_;

    // ��Ϊ Channel ͨ�������ܹ���֪ fd ���շ����ľ�����¼� revents��������������þ����¼��Ļص�����
    ReadEventCallback readCallback_; // ���¼��Ļص�����
    EventCallback writeCallback_; // д�¼��Ļص�����
    EventCallback closeCallback_; // �ر��¼��Ļص�����
    EventCallback errorCallback_; // ���������¼��Ļص�����
};
```
## ��ʼ��
```cpp
const int Channel::kNoneEvent = 0;
const int Channel::kReadEvent = EPOLLIN | EPOLLPRI;
const int Channel::kWriteEvent = EPOLLOUT;
```
## Channel(EventLoop *loop, int fd);
������channel �������¼�ѭ����channel ���ĵ��׽��ֵ��ļ�������
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
�޲���
```cpp
Channel::~Channel()
{

}
```
## void handleEvent(Timestamp receiveTime);
ͨ���ж� tied_ �����ڴ����¼�ʱ��ʱ������Ч�ı������ж�
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
���ö��¼��Ļص�����
```cpp
void Channel::setReadCallback(ReadEventCallback cb)
{
    readCallback_ = std::move(cb);
}
```
## void setWriteCallback(EventCallback cb);
����д�¼��Ļص�����
```cpp
void Channel::setWriteCallback(EventCallback cb)
{
    writeCallback_ = std::move(cb);
}
```
## void setCloseCallback(EventCallback cb);
���ùر��¼��Ļص�����
```cpp
void Channel::setCloseCallback(EventCallback cb)
{
    closeCallback_ = std::move(cb);
}
```
## void setErrorCallback(EventCallback cb);
���ô����¼��Ļص�����
```cpp
void Channel::setErrorCallback(EventCallback cb)
{
    errorCallback_ = std::move(cb);
}
```
## void tie(const std::shared_ptr<void>&);
��
```cpp
// channel �� tie ����ʲôʱ����ù���һ�� TcpConnection �����Ӵ�����ʱ�� TcpConnection => Channel
void Channel::tie(const std::shared_ptr<void> &obj)
{
    tie_ = obj;
    tied_ = true;
}
```
## int fd();
�õ� fd_
```cpp
int Channel::fd()
{
    return fd_;
}
```
## int events() const;
�õ�����Ȥ���¼�
```cpp
int Channel::events() const
{
    return events_;
}
```
## void set_revents(int revt);
���÷��ص��¼�
```cpp
void Channel::set_revents(int revt)
{
    revents_ = revt;
}
```
## void enableReading();
���öԶ�����Ȥ
```cpp
void Channel::enableReading()
{
    events_ |= kReadEvent;
    update();
}
```
## void disableReading();
���öԶ�������Ȥ
```cpp
void Channel::disableReading()
{
    events_ &= ~kReadEvent;
    update();
}
```
## void enableWriting();
���ö�д����Ȥ
```cpp
void Channel::enableWriting()
{
    events_ |= kWriteEvent;
    update();
}
```
## void disableWriting();
���ö�д������Ȥ
```cpp
void Channel::disableWriting()
{
    events_ &= ~kWriteEvent;
    update();
}
```
## void disableAll();
���ö������¼���������Ȥ
```cpp
void Channel::disableAll()
{
    events_ = kNoneEvent;
    update();
}
```
## bool isNoneEvent() const;
�Ƿ�������¼���������Ȥ
```cpp
bool Channel::isNoneEvent() const
{
    return events_ == kNoneEvent;
}
```
## bool isWriting() const;
�Ƿ��д����Ȥ
```cpp
bool Channel::isWriting() const
{
    return events_ & kWriteEvent;
}
```
## bool isReading() const;
�Ƿ�Զ�����Ȥ
```cpp
bool Channel::isReading() const
{
    return events_ & kReadEvent;
}
```
## int index();
�õ�Ŀǰ��״̬
```cpp
int Channel::index()
{
    return index_;
}
```
## void set_index(int idx);
����״̬
```cpp
void Channel::set_index(int idx)
{
    index_ = idx;
}
```
## EventLoop *ownerLoop();
�õ��� channel �������¼�ѭ��
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
    // ͨ�� Channel ������ EventLoop������ poller ����Ӧ������ע�� fd �� events �¼�
    loop_->updateChannel(this);
}
```
## void handleEventWithGuard(Timestamp receiveTime);
���� poller ֪ͨ�� channel �����ľ����¼����� channel ������þ���Ļص�����
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