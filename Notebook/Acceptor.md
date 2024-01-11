# Acceptor
## ��
```cpp
#pragma once

#include <functional>

#include "noncopyable.h"
#include "Socket.h"
#include "Channel.h"

class EventLoop;
class InetAddress;

class Acceptor : noncopyable
{
public:
    using NewConnectionCallback = std::function<void(int sockfd, const InetAddress&)>;
    Acceptor(EventLoop *loop, const InetAddress &listenAddr, bool reuseport);
    ~Acceptor();

    void setNewConnectionCallback(const NewConnectionCallback &cb);
    
    bool listenning() const;
    void listen();
private:
    void handleRead();

    EventLoop *loop_; // Acceptor �õľ����û�������Ǹ� baseLoop��Ҳ���� mainLoop
    Socket acceptSocket_; // ר�����ڽ������ӵ��׽���
    Channel acceptChannel_; // ר�Ž������ӵ�ͨ�������� conn fd
    NewConnectionCallback newConnectionCallback_; // �����ӽ����Ļص�����
    bool listenning_; // ����״̬
};
```
## int createNonblocking();
�����׽���
```cpp
static int createNonblocking()
{
    int sockfd = ::socket(AF_INET, SOCK_STREAM | SOCK_NONBLOCK | SOCK_CLOEXEC, 0);
    if (sockfd < 0)
    {
        LOG_FATAL("%s:%s:%d listen socket create err:%d \n", __FILE__, __FUNCTION__, __LINE__, errno);
    }
    return sockfd;
}
```
## Acceptor(EventLoop *loop, const InetAddress &listenAddr, bool reuseport);
���캯�������� sockfd������������ TcpServer �� start �����׽���
```cpp
Acceptor::Acceptor(EventLoop *loop, const InetAddress &listenAddr, bool reuseport)
  : loop_(loop)
  , acceptSocket_(createNonblocking())
  , acceptChannel_(loop, acceptSocket_.fd())
  , listenning_(false)
{
    acceptSocket_.setReuseAddr(true);
    acceptSocket_.setReusePort(true);
    acceptSocket_.bindAddress(listenAddr); // bind
    // TcpServer::start()  Acceptor.listen �����û����ӣ�Ҫִ��һ���ص���connfd => channel => subLoop��
    // baseLoop => acceptChannel_(listenfd)
    acceptChannel_.setReadCallback(std::bind(&Acceptor::handleRead, this));
}
```
## ~Acceptor();
����������ȡ���� Channel ����Ȥ���¼������� Channel �Ƴ�
```cpp
Acceptor::~Acceptor()
{
    acceptChannel_.disableAll();
    acceptChannel_.remove();
}
```
## void setNewConnectionCallback(const NewConnectionCallback &cb);
���������ӽ����Ļص�����
```cpp
void Acceptor::setNewConnectionCallback(const NewConnectionCallback &cb)
{
    newConnectionCallback_ = std::move(cb);
}
```
## bool listenning() const;
�����Ƿ����ڼ���
```cpp
bool Acceptor::listenning() const
{
    return listenning_;
}
```
## void listen();
���������������ø���Ȥ�¼�Ϊ���¼�
```cpp
void Acceptor::listen()
{
    listenning_ = true;
    acceptSocket_.listen(); // listen
    acceptChannel_.enableReading(); // 
}
```
## void handleRead();
listenfd ���¼������ˣ����������û�������       
�ᴴ��һ���Զ˵��׽��֣������ûص�����ѯ�ҵ� subLoop�����ѣ��ַ���ǰ���¿ͻ��˵� Channel��
```cpp
void Acceptor::handleRead()
{
    InetAddress peerAddr;
    int connfd = acceptSocket_.accept(&peerAddr);
    if (connfd >= 0)
    {
        if (newConnectionCallback_)
        {
            newConnectionCallback_(connfd, peerAddr); // ��ѯ�ҵ� subLoop�����ѣ��ַ���ǰ���¿ͻ��˵� Channel
        }
        else
        {
            ::close(connfd);
        }
    }
    else
    {
        LOG_ERROR("%s:%s:%d accept err:%d \n", __FILE__, __FUNCTION__, __LINE__, errno);
        if (errno == EMFILE)
        {
            LOG_ERROR("%s:%s:%d sockfd reached limit \n", __FILE__, __FUNCTION__, __LINE__);
        }
    }
}
```