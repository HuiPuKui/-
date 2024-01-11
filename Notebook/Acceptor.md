# Acceptor
## 类
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

    EventLoop *loop_; // Acceptor 用的就是用户定义的那个 baseLoop，也称作 mainLoop
    Socket acceptSocket_; // 专门用于接受连接的套接字
    Channel acceptChannel_; // 专门接受连接的通道，监听 conn fd
    NewConnectionCallback newConnectionCallback_; // 新连接建立的回调函数
    bool listenning_; // 监听状态
};
```
## int createNonblocking();
创建套接字
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
构造函数：创建 sockfd，待后续交给 TcpServer 来 start 监听套接字
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
    // TcpServer::start()  Acceptor.listen 有新用户连接，要执行一个回调（connfd => channel => subLoop）
    // baseLoop => acceptChannel_(listenfd)
    acceptChannel_.setReadCallback(std::bind(&Acceptor::handleRead, this));
}
```
## ~Acceptor();
析构函数：取消掉 Channel 感兴趣的事件，并将 Channel 移除
```cpp
Acceptor::~Acceptor()
{
    acceptChannel_.disableAll();
    acceptChannel_.remove();
}
```
## void setNewConnectionCallback(const NewConnectionCallback &cb);
设置新连接建立的回调函数
```cpp
void Acceptor::setNewConnectionCallback(const NewConnectionCallback &cb)
{
    newConnectionCallback_ = std::move(cb);
}
```
## bool listenning() const;
返回是否正在监听
```cpp
bool Acceptor::listenning() const
{
    return listenning_;
}
```
## void listen();
开启监听，并设置感兴趣事件为读事件
```cpp
void Acceptor::listen()
{
    listenning_ = true;
    acceptSocket_.listen(); // listen
    acceptChannel_.enableReading(); // 
}
```
## void handleRead();
listenfd 有事件发生了，就是有新用户连接了       
会创建一个对端的套接字，并调用回调（轮询找到 subLoop，唤醒，分发当前的新客户端的 Channel）
```cpp
void Acceptor::handleRead()
{
    InetAddress peerAddr;
    int connfd = acceptSocket_.accept(&peerAddr);
    if (connfd >= 0)
    {
        if (newConnectionCallback_)
        {
            newConnectionCallback_(connfd, peerAddr); // 轮询找到 subLoop，唤醒，分发当前的新客户端的 Channel
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