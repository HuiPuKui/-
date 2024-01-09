# Logger
## 类
```cpp
#pragma once

#include "noncopyable.h"

#include <string>

// LOG_INFO("%s %d", arg1, arg2)
#define LOG_INFO(logmsgFormat, ...) \
    do \
    {  \
        Logger &logger = Logger::instance(); \
        logger.setLogLevel(INFO); \
        char buf[1024] = {0}; \
        snprintf(buf, 1024, logmsgFormat, ##__VA_ARGS__); \
        logger.log(buf); \
    } while(0)

#define LOG_ERROR(logmsgFormat, ...) \
    do \
    {  \
        Logger &logger = Logger::instance(); \
        logger.setLogLevel(ERROR); \
        char buf[1024] = {0}; \
        snprintf(buf, 1024, logmsgFormat, ##__VA_ARGS__); \
        logger.log(buf); \
    } while(0)

#define LOG_FATAL(logmsgFormat, ...) \
    do \
    {  \
        Logger &logger = Logger::instance(); \
        logger.setLogLevel(FATAL); \
        char buf[1024] = {0}; \
        snprintf(buf, 1024, logmsgFormat, ##__VA_ARGS__); \
        logger.log(buf); \
        exit(-1); \
    } while(0)

#ifdef MUDBUG
#define LOG_DEBUG(logmsgFormat, ...) \
    do \
    {  \
        Logger &logger = Logger::instance(); \
        logger.setLogLevel(DEBUG); \
        char buf[1024] = {0}; \
        snprintf(buf, 1024, logmsgFormat, ##__VA_ARGS__); \
        logger.log(buf); \
    } while(0)
#else
    #define LOG_DEBUG(logmsgFormat, ...)
#endif

// 定义日志级别 INFO ERROR FATAL DEBUG
enum LogLevel
{
    INFO, //普通信息
    ERROR, //错误信息
    FATAL, //core信息
    DEBUG // 调试信息
};

// 输出一个日志类
class Logger : noncopyable
{
public:
    // 获取日志唯一的实例对象
    static Logger& instance();
    // 设置日志级别
    void setLogLevel(int level);
    // 写日志
    void log(std::string msg);
private:
    int logLevel_; // 日志级别
    Logger(){ }
};
```
## LOG_INFO(logmsgFormat, ...)
日志-普通信息
```cpp
#define LOG_INFO(logmsgFormat, ...) \
    do \
    {  \
        Logger &logger = Logger::instance(); \
        logger.setLogLevel(INFO); \
        char buf[1024] = {0}; \
        snprintf(buf, 1024, logmsgFormat, ##__VA_ARGS__); \
        logger.log(buf); \
    } while(0)
```
## LOG_ERROR(logmsgFormat, ...)
日志-错误信息
```cpp
#define LOG_ERROR(logmsgFormat, ...) \
    do \
    {  \
        Logger &logger = Logger::instance(); \
        logger.setLogLevel(ERROR); \
        char buf[1024] = {0}; \
        snprintf(buf, 1024, logmsgFormat, ##__VA_ARGS__); \
        logger.log(buf); \
    } while(0)
```
## LOG_FATAL(logmsgFormat, ...)
日志-core信息
```cpp
#define LOG_FATAL(logmsgFormat, ...) \
    do \
    {  \
        Logger &logger = Logger::instance(); \
        logger.setLogLevel(FATAL); \
        char buf[1024] = {0}; \
        snprintf(buf, 1024, logmsgFormat, ##__VA_ARGS__); \
        logger.log(buf); \
        exit(-1); \
    } while(0)
```
## LOG_DEBUG(logmsgFormat, ...)
日志-调试信息
```cpp
#ifdef MUDBUG
#define LOG_DEBUG(logmsgFormat, ...) \
    do \
    {  \
        Logger &logger = Logger::instance(); \
        logger.setLogLevel(DEBUG); \
        char buf[1024] = {0}; \
        snprintf(buf, 1024, logmsgFormat, ##__VA_ARGS__); \
        logger.log(buf); \
    } while(0)
#else
    #define LOG_DEBUG(logmsgFormat, ...)
#endif
```
## enum LogLevel
四种日志级别
```cpp
enum LogLevel
{
    INFO, //普通信息
    ERROR, //错误信息
    FATAL, //core信息
    DEBUG // 调试信息
};
```
## static Logger& instance();
单例模式，获取日志唯一的实例对象
```cpp
Logger& Logger::instance()
{
    static Logger logger;
    return logger;
}
```
## void setLogLevel(int level);
设置日志级别
```cpp
void Logger::setLogLevel(int level)
{
    logLevel_ = level;
}
```
## void log(std::string msg);
写日志：通过使用宏-LOG_XXXX() 的时候会调用设计日志级别函数和打印函数（本函数）
```cpp
void Logger::log(std::string msg)
{
    switch (logLevel_)
    {
    case INFO:
        std::cout << "[INFO]";
        break;
    case ERROR:
        std::cout << "[ERROR]";
    case FATAL:
        std::cout << "[FATAL]";
    case DEBUG:
        std::cout << "[DEBUG]";
    default:
        break;
    }

    // 打印时间 和 msg
    std::cout << Timestamp::now().toString() << " : " << msg << std::endl;
}
```
## Logger();
因为前面写成单例模式了，索引构造函数要设为私有
```cpp
Logger()
{
    
}
```