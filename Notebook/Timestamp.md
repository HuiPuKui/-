# Timestamp
## 类
```cpp
#pragma once

#include <iostream>

// 时间类
class Timestamp
{
public:
    Timestamp();
    explicit Timestamp(int64_t microSecondsSinceEpoch);
    static Timestamp now();
    std::string toString() const;
private:
    int64_t microSecondsSinceEpoch_; // 时间戳
};
```
## Timestamp();
无参构造函数，默认时间戳为 0
```cpp
Timestamp::Timestamp() : microSecondsSinceEpoch_(0) 
{

}
```
## explicit Timestamp(int64_t microSecondsSinceEpoch);
一个参数的构造函数，根据参数设置时间戳
```cpp
Timestamp::Timestamp(int64_t microSecondsSinceEpoch) : microSecondsSinceEpoch_(microSecondsSinceEpoch) 
{

}
```
## static Timestamp now();
获取当前时间
```cpp
Timestamp Timestamp::now()
{
    return Timestamp(time(NULL));
}
```
## std::string toString() const;
将当前的时间戳变成我们好理解的年月日时分秒
tm 是一个结构体，通过使用时间戳构造之后，调用它的成员得到我们需要的信息
```cpp
std::string Timestamp::toString() const
{
    char buf[128] = {0};
    tm *tm_time = localtime(&microSecondsSinceEpoch_);
    snprintf(buf, 128, "%4d/%02d/%02d %02d:%02d:%02d", tm_time->tm_year + 1900, tm_time->tm_mon + 1, tm_time->tm_mday, tm_time->tm_hour, tm_time->tm_min, tm_time->tm_sec);
    return buf;
}
```