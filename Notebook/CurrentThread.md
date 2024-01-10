# CurrentThread
## 类
```cpp
#pragma once

#include <unistd.h>
#include <sys/syscall.h>

namespace CurrentThread
{
    extern __thread int t_cachedTid; // 保存线程 Id

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
根据 Linux 系统调用获取当前线程的 tid 值
```cpp
void cacheTid()
{
    if (t_cachedTid == 0)
    {
        // 根据 Linux 系统调用获取当前线程的 tid 值
        t_cachedTid = static_cast<pid_t>(::syscall(SYS_gettid));
    }
}
```
## inline int tid();
在编译层面优化了一下，当 t_cachedTid == 0 时获取线程 Id，返回线程 Id
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