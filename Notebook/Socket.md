# Socket
## ��
```cpp
#pragma once

#include "noncopyable.h"

class InetAddress;

// ��װ Socket fd
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
    const int sockfd_; // �׽��ֵ��ļ�������
};
```
## explicit Socket(int sockfd);
Socket ���Ƕ� socket fd �Ľ�һ����װ
```cpp
Socket::Socket(int sockfd) 
  : sockfd_(sockfd)
{

}
```

## ~Socket();
Socket ����������������ϵͳ�����ر��ļ������������ Socket
```cpp
Socket::~Socket()
{
    close(sockfd_);
}
```

## int fd() const;
����ȡ�� sockfd_ ����ֵ�ĺ���
```cpp
int Socket::fd() const
{
    return sockfd_;
}
```

## void bindAddress(const InetAddress &localaddr);
�󶨱��ص�ַ���׽��֡�������޴������᷵�� 0�����򷵻� -1
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
�� sockfd_ ������׽��ֽ��м������ȴ����Ӷ������ֵ��Ϊ 1024
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
accept ȡ���ڼ����׽��� sockfd_ ���������ĵ�һ�����ӣ��½�һ�������ӵ��׽��֣�������һ�����ø��׽��ֵ��µ��ļ���������������ӳɹ��ˣ��ͽ��½����׽��ֱ����ڶԶ˵�ַ�У������������½��׽��ֵ��ļ���������
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
�ڶ���������ʾ�ж�д
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
���� Nagle �㷨������ Nagle ���Ա������������������ӳ٣���Ա�д���ӳٵ�����������Ҫ��
```cpp
void Socket::setTcpNoDelay(bool on)
{
    int optval = on ? 1 : 0;
    ::setsockopt(sockfd_, IPPROTO_TCP, TCP_NODELAY, &optval, sizeof optval);
}
```
## void setReuseAddr(bool on);
IP ��ַ����
```cpp
void Socket::setReuseAddr(bool on)
{
    int optval = on ? 1 : 0;
    ::setsockopt(sockfd_, SOL_SOCKET, SO_REUSEADDR, &optval, sizeof optval);
}
```
## void setReusePort(bool on);
�˿ڸ���
```cpp
void Socket::setReusePort(bool on)
{
    int optval = on ? 1 : 0;
    ::setsockopt(sockfd_, SOL_SOCKET, SO_REUSEPORT, &optval, sizeof optval);
}
```
## void setKeepAlive(bool on);
������
```cpp
void Socket::setKeepAlive(bool on)
{
    int optval = on ? 1 : 0;
    ::setsockopt(sockfd_, SOL_SOCKET, SO_KEEPALIVE, &optval, sizeof optval);
}
```