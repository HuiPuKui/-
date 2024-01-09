# Timestamp
## ��
```cpp
#pragma once

#include <iostream>

// ʱ����
class Timestamp
{
public:
    Timestamp();
    explicit Timestamp(int64_t microSecondsSinceEpoch);
    static Timestamp now();
    std::string toString() const;
private:
    int64_t microSecondsSinceEpoch_; // ʱ���
};
```
## Timestamp();
�޲ι��캯����Ĭ��ʱ���Ϊ 0
```cpp
Timestamp::Timestamp() : microSecondsSinceEpoch_(0) 
{

}
```
## explicit Timestamp(int64_t microSecondsSinceEpoch);
һ�������Ĺ��캯�������ݲ�������ʱ���
```cpp
Timestamp::Timestamp(int64_t microSecondsSinceEpoch) : microSecondsSinceEpoch_(microSecondsSinceEpoch) 
{

}
```
## static Timestamp now();
��ȡ��ǰʱ��
```cpp
Timestamp Timestamp::now()
{
    return Timestamp(time(NULL));
}
```
## std::string toString() const;
����ǰ��ʱ���������Ǻ�����������ʱ����
tm ��һ���ṹ�壬ͨ��ʹ��ʱ�������֮�󣬵������ĳ�Ա�õ�������Ҫ����Ϣ
```cpp
std::string Timestamp::toString() const
{
    char buf[128] = {0};
    tm *tm_time = localtime(&microSecondsSinceEpoch_);
    snprintf(buf, 128, "%4d/%02d/%02d %02d:%02d:%02d", tm_time->tm_year + 1900, tm_time->tm_mon + 1, tm_time->tm_mday, tm_time->tm_hour, tm_time->tm_min, tm_time->tm_sec);
    return buf;
}
```