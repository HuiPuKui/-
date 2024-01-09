# InetAddress
## 类
```cpp
#pragma once

#include <arpa/inet.h>
#include <netinet/in.h>
#include <string>

// 封装socket地址类型

class InetAddress
{
public:
    explicit InetAddress(uint16_t port = 0, std::string ip = "127.0.0.1");

    explicit InetAddress(const sockaddr_in &addr) : addr_(addr) { }
    
    std::string toIp() const;
    std::string toIpPort() const;
    uint16_t toPort() const;

    const sockaddr_in* getSockAddr() const {return &addr_;}
    void setSockAddr(const sockaddr_in &addr) { addr_ = addr;}
private:
    sockaddr_in addr_; // 套接字的地址
};
```
## explicit InetAddress(uint16_t port = 0, std::string ip = "127.0.0.1");
构造函数：根据端口号和IP地址构造。首先清空addr，之后设置 IPv4、端口、地址
```cpp
InetAddress::InetAddress(uint16_t port, std::string ip)
{
    bzero(&addr_, sizeof addr_);
    addr_.sin_family = AF_INET;
    addr_.sin_port = htons(port);
    addr_.sin_addr.s_addr = inet_addr(ip.c_str());
}
```
## explicit InetAddress(const sockaddr_in &addr);
构造函数：根据addr构造。
```cpp
InetAddress::InetAddress(const sockaddr_in &addr) : addr_(addr) 
{

}
```
## std::string toIp() const;
将IP从用于网络传输的数值格式转换为点分十进制的IP地址格式
```cpp
std::string InetAddress::toIp() const
{
    char buf[64] = {0};
    ::inet_ntop(AF_INET, &addr_.sin_addr.s_addr, buf, sizeof buf);
    return buf;
}
```
## std::string toIpPort() const;
调用 `::inet_ntop` 和 `ntohs` 将IP地址和端口号都写到 buf 中并返回
```cpp
std::string InetAddress::toIpPort() const
{
    char buf[64] = {0};
    ::inet_ntop(AF_INET, &addr_.sin_addr, buf, sizeof buf);
    size_t end = strlen(buf);
    uint16_t port = ntohs(addr_.sin_port);

    sprintf(buf+end, ":%u", port);
    return buf;
    // ip:port
}
```
## uint16_t toPort() const;
将一个十六位数由网络字节序转换为主机字节序
```cpp
uint16_t InetAddress::toPort() const
{
    return ntohs(addr_.sin_port);
}
```
## const sockaddr_in* getSockAddr() const;
取到保存套接字地址的结构体
```cpp
const sockaddr_in* InetAddress::getSockAddr() const
{
    return &addr_;
}
```
## void setSockAddr(const sockaddr_in &addr) { addr_ = addr;}
设置保存套接字地址的结构体
```cpp
void InetAddress::setSockAddr(const sockaddr_in &addr) 
{
    addr_ = addr;
}
```