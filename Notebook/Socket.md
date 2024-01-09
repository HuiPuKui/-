# Socket
## 类
```cpp
#pragma once

#include "noncopyable.h"

class InetAddress;

// 封装 Socket fd
class Socket : noncopyable
{
public:
    explicit Socket(int sockfd);

    ~Socket();

    int fd() const;
    void bindAddress(const InetAddress &localaddr);
    void listen();
    int accept(InetAddress *peeraddr);

    void shutdownWrite();

    void setTcpNoDelay(bool on);
    void setReuseAddr(bool on);
    void setReusePort(bool on);
    void setKeepAlive(bool on);
private:
    const int sockfd_; // 套接字的文件描述符
};
```
## explicit Socket(int sockfd);
Socket 类是对 socket fd 的进一步封装
```cpp
Socket::Socket(int sockfd) 
  : sockfd_(sockfd)
{

}
```

## ~Socket();
Socket 的析构函数，调用系统函数关闭文件描述符代表的 Socket
```cpp
Socket::~Socket()
{
    close(sockfd_);
}
```

## int fd() const;
可以取到 sockfd_ 具体值的函数
```cpp
int Socket::fd() const
{
    return sockfd_;
}
```

## void bindAddress(const InetAddress &localaddr);
绑定本地地址和套接字。如果绑定无错误发生会返回 0，否则返回 -1
```cpp
void Socket::bindAddress(const InetAddress &localaddr)
{
    if (0 != bind(sockfd_, (sockaddr*)localaddr.getSockAddr(), sizeof(sockaddr_in)))
    {
        LOG_FATAL("bind sockfd:%d fail \n", sockfd_);
    }
}
```
## void listen();
用 sockfd_ 代表的套接字进行监听，等待连接队列最大值设为 1024
```cpp
void Socket::listen()
{
    if (0 != ::listen(sockfd_, 1024))
    {
        LOG_FATAL("listen sockfd:%d fail \n", sockfd_);
    }
}
```
## int accept(InetAddress *peeraddr);
accept 取出在监听套接字 sockfd_ 请求队列里的第一个连接，新建一个已连接的套接字，并返回一个引用该套接字的新的文件描述符。如果连接成功了，就将新建的套接字保存在对端地址中，并返回引用新建套接字的文件描述符。
```cpp
int Socket::accept(InetAddress *peeraddr)
{
    sockaddr_in addr;
    socklen_t len = sizeof addr;
    bzero(&addr, sizeof addr);
    int connfd = ::accept4(sockfd_, (sockaddr*)&addr, &len, SOCK_NONBLOCK | SOCK_CLOEXEC);
    if (connfd >= 0)
    {
        peeraddr->setSockAddr(addr);
    }
    return connfd;
}
```
## void shutdownWrite();
第二个参数表示切断写
```cpp
void Socket::shutdownWrite()
{
    if (::shutdown(sockfd_, SHUT_WR) < 0)
    {
        LOG_ERROR("shutdownWrite error");
    }
}
```
## void setTcpNoDelay(bool on);
禁用 Nagle 算法，禁用 Nagle 可以避免连续发出包出现延迟，这对编写低延迟的网络服务很重要。
```cpp
void Socket::setTcpNoDelay(bool on)
{
    int optval = on ? 1 : 0;
    ::setsockopt(sockfd_, IPPROTO_TCP, TCP_NODELAY, &optval, sizeof optval);
}
```
## void setReuseAddr(bool on);
IP 地址复用
```cpp
void Socket::setReuseAddr(bool on)
{
    int optval = on ? 1 : 0;
    ::setsockopt(sockfd_, SOL_SOCKET, SO_REUSEADDR, &optval, sizeof optval);
}
```
## void setReusePort(bool on);
端口复用
```cpp
void Socket::setReusePort(bool on)
{
    int optval = on ? 1 : 0;
    ::setsockopt(sockfd_, SOL_SOCKET, SO_REUSEPORT, &optval, sizeof optval);
}
```
## void setKeepAlive(bool on);
长连接
```cpp
void Socket::setKeepAlive(bool on)
{
    int optval = on ? 1 : 0;
    ::setsockopt(sockfd_, SOL_SOCKET, SO_KEEPALIVE, &optval, sizeof optval);
}
```