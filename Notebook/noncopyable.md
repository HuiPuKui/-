# noncopyable
## ��
���ã�noncopyable ���̳��Ժ������������������Ĺ������������������������޷����п�������͸�ֵ���� 
```cpp
#pragma once 

class noncopyable
{
public:
    noncopyable(const noncopyable&) = delete;
    noncopyable &operator = (const noncopyable&) = delete;
protected:
    noncopyable() = default;
    ~noncopyable() = default;
};
```