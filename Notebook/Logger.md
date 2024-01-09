# Logger
## ��
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

// ������־���� INFO ERROR FATAL DEBUG
enum LogLevel
{
    INFO, //��ͨ��Ϣ
    ERROR, //������Ϣ
    FATAL, //core��Ϣ
    DEBUG // ������Ϣ
};

// ���һ����־��
class Logger : noncopyable
{
public:
    // ��ȡ��־Ψһ��ʵ������
    static Logger& instance();
    // ������־����
    void setLogLevel(int level);
    // д��־
    void log(std::string msg);
private:
    int logLevel_; // ��־����
    Logger(){ }
};
```
## LOG_INFO(logmsgFormat, ...)
��־-��ͨ��Ϣ
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
��־-������Ϣ
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
��־-core��Ϣ
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
��־-������Ϣ
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
������־����
```cpp
enum LogLevel
{
    INFO, //��ͨ��Ϣ
    ERROR, //������Ϣ
    FATAL, //core��Ϣ
    DEBUG // ������Ϣ
};
```
## static Logger& instance();
����ģʽ����ȡ��־Ψһ��ʵ������
```cpp
Logger& Logger::instance()
{
    static Logger logger;
    return logger;
}
```
## void setLogLevel(int level);
������־����
```cpp
void Logger::setLogLevel(int level)
{
    logLevel_ = level;
}
```
## void log(std::string msg);
д��־��ͨ��ʹ�ú�-LOG_XXXX() ��ʱ�����������־�������ʹ�ӡ��������������
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

    // ��ӡʱ�� �� msg
    std::cout << Timestamp::now().toString() << " : " << msg << std::endl;
}
```
## Logger();
��Ϊǰ��д�ɵ���ģʽ�ˣ��������캯��Ҫ��Ϊ˽��
```cpp
Logger()
{
    
}
```