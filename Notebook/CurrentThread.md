# CurrentThread
## ��
```cpp
#pragma once

#include <unistd.h>
#include <sys/syscall.h>

namespace CurrentThread
{
    extern __thread int t_cachedTid; // �����߳� Id

    void cacheTid();

    inline int tid()
    {
        if (__builtin_expect(t_cachedTid == 0, 0))
        {
            cacheTid();
        }
        return t_cachedTid;
    }
}
```
## void cacheTid();
���� Linux ϵͳ���û�ȡ��ǰ�̵߳� tid ֵ
```cpp
void cacheTid()
{
    if (t_cachedTid == 0)
    {
        // ���� Linux ϵͳ���û�ȡ��ǰ�̵߳� tid ֵ
        t_cachedTid = static_cast<pid_t>(::syscall(SYS_gettid));
    }
}
```
## inline int tid();
�ڱ�������Ż���һ�£��� t_cachedTid == 0 ʱ��ȡ�߳� Id�������߳� Id
```cpp
inline int tid()
{
    if (__builtin_expect(t_cachedTid == 0, 0))
    {
        cacheTid();
    }
    return t_cachedTid;
}
```