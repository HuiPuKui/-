# InetAddress
## ��
```cpp
#pragma once

#include <arpa/inet.h>
#include <netinet/in.h>
#include <string>

// ��װsocket��ַ����

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
    sockaddr_in addr_; // �׽��ֵĵ�ַ
};
```
## explicit InetAddress(uint16_t port = 0, std::string ip = "127.0.0.1");
���캯�������ݶ˿ںź�IP��ַ���졣�������addr��֮������ IPv4���˿ڡ���ַ
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
���캯��������addr���졣
```cpp
InetAddress::InetAddress(const sockaddr_in &addr) : addr_(addr) 
{

}
```
## std::string toIp() const;
��IP���������紫�����ֵ��ʽת��Ϊ���ʮ���Ƶ�IP��ַ��ʽ
```cpp
std::string InetAddress::toIp() const
{
    char buf[64] = {0};
    ::inet_ntop(AF_INET, &addr_.sin_addr.s_addr, buf, sizeof buf);
    return buf;
}
```
## std::string toIpPort() const;
���� `::inet_ntop` �� `ntohs` ��IP��ַ�Ͷ˿ںŶ�д�� buf �в�����
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
��һ��ʮ��λ���������ֽ���ת��Ϊ�����ֽ���
```cpp
uint16_t InetAddress::toPort() const
{
    return ntohs(addr_.sin_port);
}
```
## const sockaddr_in* getSockAddr() const;
ȡ�������׽��ֵ�ַ�Ľṹ��
```cpp
const sockaddr_in* InetAddress::getSockAddr() const
{
    return &addr_;
}
```
## void setSockAddr(const sockaddr_in &addr) { addr_ = addr;}
���ñ����׽��ֵ�ַ�Ľṹ��
```cpp
void InetAddress::setSockAddr(const sockaddr_in &addr) 
{
    addr_ = addr;
}
```