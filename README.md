# 开发参考视频地址

[\[C++高级教程\]从零开始开发服务器框架(sylar)](https://www.bilibili.com/video/av53602631/?from=www.sylar.top "")

# 模块内容：

## 1.日志模块*

支持流式日志风格写日志和格式化风格写日志，支持日志格式自定义，日志级别，多日志分离等等功能
流式日志使用：SYLAR_LOG_INFO(g_logger) << "this is a log";
格式化日志使用：SYLAR_LOG_FMT_INFO(g_logger, "%s", "this is a log");
支持时间,线程id,线程名称,日志级别,日志名称,文件名,行号等内容的自由配置

## 2.配置模块*

采用约定由于配置的思想。定义即可使用。不需要单独去解析。支持变更通知功能。使用YAML文件做为配置内容。支持级别格式的数据类型，支持STL容器(vector,list,set,map等等),支持自定义类型的支持（需要实现序列化和反序列化方法)使用方式如下：

```cpp
static sylar::ConfigVar<int>::ptr g_tcp_connect_timeout =
	sylar::Config::Lookup("tcp.connect.timeout", 5000, "tcp connect timeout");
```

定义了一个tcp连接超时参数，可以直接使用 g_tcp_connect_timeout->getValue() 获取参数的值，当配置修改重新加载，该值自动更新
上述配置格式如下：

```sh
tcp:
    connect:
            timeout: 10000
```

## 3.线程模块

### 问题：封装了pthread

1为什么不适用c++11里面的thread
本框架是使用C++11开发，不使用thread，是因为thread其实也是基于pthread实现的。并且C++11里面没有提供读写互斥量，RWMutex，Spinlock等，在高并发场景，这些对象是经常需要用到的。所以选择了自己封装pthread

线程模块，封装了pthread里面的一些常用功能，Thread,Semaphore,Mutex,RWMutex,Spinlock等对象，可以方便开发中对线程日常使用`Thread`：线程类，构造函数传入线程入口函数和线程名称，线程入口函数类型为void()，如果带参数，则需要用std::bind进行绑定。线程类构造之后线程即开始运行，构造函数在线程真正开始运行之后返回。

线程同步类（这部分被拆分到mutex.h)中：

`Semaphore`: 计数信号量，基于sem_t实现
`Mutex`: 互斥锁，基于pthread_mutex_t实现
`RWMutex`: 读写锁，基于pthread_rwlock_t实现
`Spinlock`: 自旋锁，基于pthread_spinlock_t实现
`CASLock`: 原子锁，基于std::atomic_flag实现

线程模块总体比较简单，在对pthread相关的接口有一定了解的情况下，参考源码应该不难理解，这里说几个重点：

\1. 为什么不直接使用C++11提供的thread类。按sylar的描述，因为thread其实也是基于pthread实现的。并且C++11里面没有提供读写互斥量，RWMutex，Spinlock等，在高并发场景，这些对象是经常需要用到的，所以选择自己封装pthread。

\2. 关于线程入口函数。sylar的线程只支持void(void)类型的入口函数，不支持给线程传参数，但实际使用时可以结合std::bind来绑定参数，这样就相当于支持任何类型和数量的参数。

\3. 关于子线程的执行时机。sylar的线程类可以保证在构造完成之后线程函数一定已经处于运行状态，这是通过一个信号量来实现的，构造函数在创建线程后会一直阻塞，直到线程函数运行并且通知信号量，构造函数才会返回，而构造函数一旦返回，就说明线程函数已经在执行了。

\4. 关于线程局部变量。sylar的每个线程都有两个线程局部变量，一个用于存储当前线程的Thread指针，另一个存储线程名称，通过Thread::GetThis()可以拿到当前线程的指针。

\5. 关于范围锁。sylar大量使用了范围锁来实现互斥，范围锁是指用类的构造函数来加锁，用析造函数来释放锁。这种方式可以简化锁的操作，也可以避免忘记解锁导致的死锁问题，以下是一个范围锁的示例和说明：

sylar::Mutex mutex;

{
    sylar::Mutex::Lock lock(mutex); // 定义lock对象，类型为sylar::Mutex::Lock，传入互斥量，在构造函数中完成加锁操作，如果该锁已经被持有，那构造lock时就会阻塞，直到锁被释放
    //临界区操作
    ...
    // 大括号范围结束，所有在该范围内定义的自动变量都会被回收，lock对象被回收时触发析构函数，在析构函数中释放锁
}

## 4.协程模块

协程：用户态的线程，相当于线程中的线程，更轻量级。后续配置socket hook，可以把复杂的异步调用，封装成同步操作。降低业务逻辑的编写复杂度。
目前该协程是基于ucontext_t来实现的，后续将支持采用boost.context里面的fcontext_t的方式实现

## 5.协程调度模块

协程调度器，管理协程的调度，内部实现为一个线程池，支持协程在多线程中切换，也可以指定协程在固定的线程中执行。是一个N-M的协程调度模型，N个线程，M个协程。重复利用每一个线程。

## 6.IO协程调度模块

继承与协程调度器，封装了epoll（Linux），并支持定时器功能（使用epoll实现定时器，精度毫秒级）,支持Socket读写时间的添加，删除，取消功能。支持一次性定时器，循环定时器，条件定时器等功能

## 7.Hook模块

hook系统底层和socket相关的API，socket io相关的API，以及sleep系列的API。hook的开启控制是线程粒度的。可以自由选择。通过hook模块，可以使一些不具异步功能的API，展现出异步的性能。如（mysql）

## 8.Socket模块

封装了Socket类，提供所有socket API功能，统一封装了地址类，将IPv4，IPv6，Unix地址统一起来。并且提供域名，IP解析功能。

## 9.ByteArray序列化模块

ByteArray二进制序列化模块，提供对二进制数据的常用操作。读写入基础类型int8_t,int16_t,int32_t,int64_t等，支持Varint,std::string的读写支持,支持字节序转化,支持序列化到文件，以及从文件反序列化等功能

## 10.TcpServer模块

基于Socket类，封装了一个通用的TcpServer的服务器类，提供简单的API，使用便捷，可以快速绑定一个或多个地址，启动服务，监听端口，accept连接，处理socket连接等功能。具体业务功能更的服务器实现，只需要继承该类就可以快速实现

## 11.Stream模块

封装流式的统一接口。将文件，socket封装成统一的接口。使用的时候，采用统一的风格操作。基于统一的风格，可以提供更灵活的扩展。目前实现了SocketStream

## 12.HTTP模块

采用Ragel（有限状态机，性能媲美汇编），实现了HTTP/1.1的简单协议实现和uri的解析。基于SocketStream实现了HttpConnection(HTTP的客户端)和HttpSession(HTTP服务器端的链接）。基于TcpServer实现了HttpServer。提供了完整的HTTP的客户端API请求功能，HTTP基础API服务器功能

## 13.Servlet模块

仿照java的servlet，实现了一套Servlet接口，实现了ServletDispatch，FunctionServlet。NotFoundServlet。支持uri的精准匹配，模糊匹配等功能。和HTTP模块，一起提供HTTP服务器功能

## 14.其他相关

联系方式：
 QQ：564628276
 邮箱：564628276@qq.com
 微信：sylar-yin
 QQ群：8151915（sylar技术群）
个人主页：www.sylar.top
github:https://github.com/sylar-yin/sylar