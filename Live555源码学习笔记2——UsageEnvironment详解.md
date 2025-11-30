# Live555源码学习笔记2——UsageEnvironment详解

## 一、模块概述

### 1.1 模块定位

`UsageEnvironment`是Live555的**最底层模块**，它定义了整个框架运行所需的基础设施接口：

```
┌─────────────────────────────────────────────────────────────────┐
│                    UsageEnvironment 模块定位                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   liveMedia / groupsock / 应用程序                              │
│           │                                                     │
│           │ 依赖                                                │
│           ▼                                                     │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │              UsageEnvironment (抽象层)                   │   │
│   │                                                         │   │
│   │  提供：                                                  │   │
│   │  • 任务调度器 (TaskScheduler)                           │   │
│   │  • 日志/消息输出                                        │   │
│   │  • 错误处理                                             │   │
│   │  • 哈希表等工具类                                       │   │
│   └─────────────────────────────────────────────────────────┘   │
│           │                                                     │
│           │ 由子类实现                                          │
│           ▼                                                     │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │           BasicUsageEnvironment (实现层)                 │   │
│   │                                                         │   │
│   │  实现：                                                  │   │
│   │  • 基于 select() 的事件循环                             │   │
│   │  • 标准输出的日志打印                                   │   │
│   │  • 延时队列管理                                         │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 文件结构

```
UsageEnvironment/
├── include/
│   ├── UsageEnvironment.hh      ⭐ 核心头文件
│   ├── UsageEnvironment_version.hh  版本信息
│   ├── HashTable.hh             哈希表抽象接口
│   ├── Boolean.hh               布尔类型定义
│   ├── strDup.hh                字符串复制工具
│   └── NetCommon.h              网络通用定义
├── UsageEnvironment. cpp         抽象类部分实现
├── HashTable.cpp                哈希表部分实现
└── strDup.cpp                   字符串工具实现
```

---

## 二、UsageEnvironment 类详解

### 2.1 完整类定义

```cpp
// 文件：UsageEnvironment/include/UsageEnvironment.hh

class TaskScheduler; // 前向声明

// UsageEnvironment 是一个抽象基类
class UsageEnvironment {
public:
    // ============ 对象生命周期管理 ============
    Boolean reclaim();
        // 回收（删除）本对象，返回True表示成功删除

    // ============ 任务调度器访问 ============
    TaskScheduler& taskScheduler() const { return fScheduler; }

    // ============ 结果消息处理 ============
    typedef char const* MsgString;
    
    virtual MsgString getResultMsg() const = 0;              // 获取消息
    virtual void setResultMsg(MsgString msg) = 0;            // 设置单条消息
    virtual void setResultMsg(MsgString msg1, 
                              MsgString msg2) = 0;           // 设置两条消息
    virtual void setResultMsg(MsgString msg1, 
                              MsgString msg2, 
                              MsgString msg3) = 0;           // 设置三条消息
    virtual void setResultErrMsg(MsgString msg, 
                                 int err = 0) = 0;           // 设置错误消息
    virtual void appendToResultMsg(MsgString msg) = 0;       // 追加消息
    virtual void reportBackgroundError() = 0;                // 报告后台错误

    virtual void internalError();  // 处理内部错误（通常是abort）

    // ============ 错误码 ============
    virtual int getErrno() const = 0;

    // ============ 控制台输出（日志）============
    virtual UsageEnvironment& operator<<(char const* str) = 0;
    virtual UsageEnvironment& operator<<(int i) = 0;
    virtual UsageEnvironment& operator<<(unsigned u) = 0;
    virtual UsageEnvironment& operator<<(double d) = 0;
    virtual UsageEnvironment& operator<<(void* p) = 0;

    // ============ 私有数据指针（供其他模块使用）============
    void* liveMediaPriv;   // liveMedia模块使用
    void* groupsockPriv;   // groupsock模块使用

protected:
    // 构造函数是protected，说明这是抽象基类，不能直接实例化
    UsageEnvironment(TaskScheduler& scheduler);
    virtual ~UsageEnvironment();  // 通过reclaim()删除

private:
    TaskScheduler& fScheduler;  // 持有任务调度器的引用
};
```

### 2.2 核心方法说明

#### 2.2.1 任务调度器访问

```cpp
TaskScheduler& taskScheduler() const { return fScheduler; }
```

这是最重要的方法！整个Live555的事件驱动都通过这个调度器实现：

```c++
// 使用示例
UsageEnvironment* env = ...;
// 调度一个延时任务
env->taskScheduler().scheduleDelayedTask(1000000, myCallback, myData);
// 监听Socket
env->taskScheduler().setBackgroundHandling(sockfd, SOCKET_READABLE, onReadable, myData);
// 进入事件循环
env->taskScheduler().doEventLoop();
```

#### 2.2.2 消息处理系统

```cpp
// 设置结果消息
env->setResultMsg("Connection failed");
env->setResultMsg("Error: ", "file not found");

// 设置带系统错误码的消息
env->setResultErrMsg("Socket error");  // 会自动追加errno描述

// 获取消息
const char* msg = env->getResultMsg();

// 追加消息
env->appendToResultMsg(", retrying.. .");

// 报告后台错误（在事件循环中发生的错误）
env->reportBackgroundError();
```

#### 2.2.3 日志输出

重载的`<<`运算符让日志输出像`cout`一样方便：

```cpp
// 输出日志
*env << "Server started on port " << 8554 << "\n";
*env << "Client connected from " << clientIP << "\n";
*env << "Bitrate: " << 2. 5 << " Mbps\n";
```

### 2.3 设计模式分析

#### 为什么使用抽象类？

```
┌─────────────────────────────────────────────────────────────────┐
│                    依赖倒置原则（DIP）                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  【不好的设计】直接依赖具体实现                                   │
│                                                                 │
│   liveMedia ─────────► BasicUsageEnvironment                    │
│                        (具体实现)                               │
│                                                                 │
│   问题：如果要换一种实现（如GUI环境），必须修改liveMedia代码      │
│                                                                 │
│  ─────────────────────────────────────────────────────────────  │
│                                                                 │
│  【好的设计】依赖抽象接口                                        │
│                                                                 │
│   liveMedia ─────────► UsageEnvironment (抽象接口)              │
│                              ▲                                  │
│                              │ 实现                             │
│                              │                                  │
│               ┌──────────────┴──────────────┐                   │
│               │                             │                   │
│   BasicUsageEnvironment            GuiUsageEnvironment          │
│   (控制台实现)                      (GUI实现)                   │
│                                                                 │
│   好处：可以随时切换不同的实现，而不需要修改上层代码              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 三、TaskScheduler 类详解

### 3.1 完整类定义

```cpp
// 文件：UsageEnvironment/include/UsageEnvironment.hh

// ============ 类型定义 ============
typedef void TaskFunc(void* clientData);     // 任务回调函数类型
typedef void* TaskToken;                      // 任务标识符
typedef u_int32_t EventTriggerId;            // 事件触发器ID

// 事件循环监视变量（用于退出循环）
#ifndef NO_STD_LIB
typedef std::atomic_char EventLoopWatchVariable;  // 线程安全
#else
typedef char volatile EventLoopWatchVariable;
#endif

class TaskScheduler {
public:
    virtual ~TaskScheduler();

    // ============ 1. 延时任务调度 ============
    virtual TaskToken scheduleDelayedTask(int64_t microseconds,
                                          TaskFunc* proc,
                                          void* clientData) = 0;
    
    virtual void unscheduleDelayedTask(TaskToken& prevTask) = 0;
    
    virtual void rescheduleDelayedTask(TaskToken& task,
                                       int64_t microseconds,
                                       TaskFunc* proc,
                                       void* clientData);

    // ============ 2. Socket后台处理 ============
    typedef void BackgroundHandlerProc(void* clientData, int mask);
    
    // 条件标志位
    #define SOCKET_READABLE    (1<<1)  // 0x02
    #define SOCKET_WRITABLE    (1<<2)  // 0x04
    #define SOCKET_EXCEPTION   (1<<3)  // 0x08
    
    virtual void setBackgroundHandling(int socketNum,
                                       int conditionSet,
                                       BackgroundHandlerProc* handlerProc,
                                       void* clientData) = 0;
    
    void disableBackgroundHandling(int socketNum) {
        setBackgroundHandling(socketNum, 0, NULL, NULL);
    }
    
    virtual void moveSocketHandling(int oldSocketNum, 
                                    int newSocketNum) = 0;

    // ============ 3.  事件循环 ============
    virtual void doEventLoop(EventLoopWatchVariable* watchVariable = NULL) = 0;

    // ============ 4.  事件触发器（跨线程通信）============
    virtual EventTriggerId createEventTrigger(TaskFunc* eventHandlerProc) = 0;
    virtual void deleteEventTrigger(EventTriggerId eventTriggerId) = 0;
    virtual void triggerEvent(EventTriggerId eventTriggerId, 
                              void* clientData = NULL) = 0;

    // ============ 废弃的兼容方法 ============
    void turnOnBackgroundReadHandling(int socketNum, 
                                      BackgroundHandlerProc* handlerProc,
                                      void* clientData) {
        setBackgroundHandling(socketNum, SOCKET_READABLE, handlerProc, clientData);
    }
    void turnOffBackgroundReadHandling(int socketNum) {
        disableBackgroundHandling(socketNum);
    }

    virtual void internalError();

protected:
    TaskScheduler();  // 抽象基类
};
```

### 3.2 四大核心功能详解

#### 3.2. 1 延时任务调度

```cpp
// 调度一个延时任务
TaskToken scheduleDelayedTask(int64_t microseconds,  // 延时（微秒）
                              TaskFunc* proc,         // 回调函数
                              void* clientData);      // 用户数据

// 取消延时任务
void unscheduleDelayedTask(TaskToken& prevTask);

// 重新调度任务（先取消再调度）
void rescheduleDelayedTask(TaskToken& task,
                           int64_t microseconds,
                           TaskFunc* proc,
                           void* clientData);
```

**使用示例：超时检测**

```cpp
class MyConnection {
public:
    void startTimeoutCheck() {
        // 5秒后检查超时
        fTimeoutTask = env->taskScheduler().scheduleDelayedTask(
            5000000,              // 5秒 = 5,000,000微秒
            timeoutHandler,       // 静态回调函数
            this                  // 传递this指针
        );
    }
    
    void onDataReceived() {
        // 收到数据，重置超时计时器
        env->taskScheduler().rescheduleDelayedTask(
            fTimeoutTask,
            5000000,
            timeoutHandler,
            this
        );
    }
    
    void close() {
        // 关闭前取消定时器
        env->taskScheduler().unscheduleDelayedTask(fTimeoutTask);
    }
    
private:
    // 静态回调函数（必须是静态的，因为回调函数签名固定）
    static void timeoutHandler(void* clientData) {
        MyConnection* self = (MyConnection*)clientData;
        self->handleTimeout();
    }
    
    void handleTimeout() {
        *env << "Connection timeout!\n";
        close();
    }
    
    TaskToken fTimeoutTask;
    UsageEnvironment* env;
};
```

#### 3.2.2 Socket后台处理

```cpp
// 设置Socket后台处理
void setBackgroundHandling(int socketNum,           // Socket描述符
                           int conditionSet,        // 监听条件
                           BackgroundHandlerProc* handlerProc,  // 回调
                           void* clientData);       // 用户数据

// 监听条件可以组合使用
#define SOCKET_READABLE    (1<<1)  // Socket可读
#define SOCKET_WRITABLE    (1<<2)  // Socket可写
#define SOCKET_EXCEPTION   (1<<3)  // Socket异常
```

**使用示例：接收网络数据**

```cpp
class MyServer {
public:
    void startListening(int listenSocket) {
        fListenSocket = listenSocket;
        
        // 监听Socket可读事件（有新连接到来）
        env->taskScheduler().setBackgroundHandling(
            fListenSocket,
            SOCKET_READABLE,
            incomingConnectionHandler,
            this
        );
    }
    
    void stopListening() {
        // 取消监听
        env->taskScheduler().disableBackgroundHandling(fListenSocket);
    }
    
private:
    static void incomingConnectionHandler(void* clientData, int mask) {
        MyServer* self = (MyServer*)clientData;
        
        // mask告诉我们是什么事件
        if (mask & SOCKET_READABLE) {
            self->handleNewConnection();
        }
    }
    
    void handleNewConnection() {
        int clientSocket = accept(fListenSocket, NULL, NULL);
        // 处理新连接... 
    }
    
    int fListenSocket;
    UsageEnvironment* env;
};
```

#### 3.2.3 事件循环

```cpp
// 进入事件循环
void doEventLoop(EventLoopWatchVariable* watchVariable = NULL);
```

**工作原理：**

```
┌─────────────────────────────────────────────────────────────────┐
│                    doEventLoop() 工作流程                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   while (*watchVariable == 0) {  // 循环直到退出标志被设置       │
│       │                                                         │
│       │  ┌───────────────────────────────────────────────────┐  │
│       ├─►│ 1. 计算下一个定时任务的到期时间                    │  │
│       │  └───────────────────────────────────────────────────┘  │
│       │                                                         │
│       │  ┌───────────────────────────────────────────────────┐  │
│       ├─►│ 2. 调用 select() 等待事件                         │  │
│       │  │    - 等待Socket就绪 或 超时                       │  │
│       │  └───────────────────────────────────────────────────┘  │
│       │                                                         │
│       │  ┌───────────────────────────────────────────────────┐  │
│       ├─►│ 3. 处理就绪的Socket                               │  │
│       │  │    - 调用对应的回调函数                           │  │
│       │  └───────────────────────────────────────────────────┘  │
│       │                                                         │
│       │  ┌───────────────────────────────────────────────────┐  │
│       ├─►│ 4.  处理事件触发器                                 │  │
│       │  └───────────────────────────────────────────────────┘  │
│       │                                                         │
│       │  ┌───────────────────────────────────────────────────┐  │
│       └─►│ 5. 处理到期的延时任务                             │  │
│          └───────────────────────────────────────────────────┘  │
│   }                                                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**使用示例：**

```cpp
int main() {
    TaskScheduler* scheduler = BasicTaskScheduler::createNew();
    UsageEnvironment* env = BasicUsageEnvironment::createNew(*scheduler);
    
    // 设置退出标志
    EventLoopWatchVariable watchVariable = 0;
    
    // 注册信号处理
    signal(SIGINT, [](int) { watchVariable = 1; });
    
    // 创建服务器等... 
    
    // 进入事件循环（会一直运行）
    env->taskScheduler().doEventLoop(&watchVariable);
    
    // 退出后清理
    env->reclaim();
    delete scheduler;
    
    return 0;
}
```

#### 3.2.4 事件触发器（跨线程通信）

这是Live555中**唯一可以从外部线程安全调用**的机制：

```cpp
// 创建事件触发器
EventTriggerId createEventTrigger(TaskFunc* eventHandlerProc);

// 删除事件触发器
void deleteEventTrigger(EventTriggerId eventTriggerId);

// 触发事件（可以从其他线程调用！）
void triggerEvent(EventTriggerId eventTriggerId, void* clientData = NULL);
```

**使用示例：视频采集线程通知主线程**

```cpp
class VideoCapture {
public:
    void start() {
        // 在主线程创建触发器
        fNewFrameTriggerId = env->taskScheduler().createEventTrigger(
            onNewFrameAvailable
        );
        
        // 启动采集线程
        fCaptureThread = std::thread(&VideoCapture::captureLoop, this);
    }
    
    void stop() {
        fRunning = false;
        fCaptureThread.join();
        env->taskScheduler().deleteEventTrigger(fNewFrameTriggerId);
    }
    
private:
    // 采集线程（在另一个线程中运行）
    void captureLoop() {
        while (fRunning) {
            // 采集一帧
            Frame* frame = captureOneFrame();
            
            // 存储帧数据
            fFrameQueue.push(frame);
            
            // 通知主线程（这是唯一可以跨线程调用的Live555函数！）
            env->taskScheduler().triggerEvent(fNewFrameTriggerId, this);
        }
    }
    
    // 在主线程中被调用
    static void onNewFrameAvailable(void* clientData) {
        VideoCapture* self = (VideoCapture*)clientData;
        
        // 从队列取出帧
        Frame* frame = self->fFrameQueue. pop();
        
        // 处理帧（发送到网络等）
        self->processFrame(frame);
    }
    
    EventTriggerId fNewFrameTriggerId;
    std::thread fCaptureThread;
    bool fRunning;
    ThreadSafeQueue<Frame*> fFrameQueue;
    UsageEnvironment* env;
};
```

---

## 四、回调函数机制详解

### 4. 1 为什么需要回调？

```
┌─────────────────────────────────────────────────────────────────┐
│                    同步 vs 异步（回调）                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  【同步方式】阻塞等待                                            │
│                                                                 │
│   while (1) {                                                   │
│       data = recv(socket);  // 阻塞！可能等很久                  │
│       process(data);                                            │
│   }                                                             │
│                                                                 │
│   问题：等待期间无法做其他事情                                   │
│                                                                 │
│  ─────────────────────────────────────────────────────────────  │
│                                                                 │
│  【异步方式】回调                                                │
│                                                                 │
│   // 注册回调，立即返回                                          │
│   setBackgroundHandling(socket, READABLE, myCallback, data);    │
│                                                                 │
│   // 进入事件循环                                                │
│   doEventLoop();  // 当socket可读时，自动调用myCallback          │
│                                                                 │
│   好处：可以同时处理多个Socket和定时任务                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 回调函数的签名

```cpp
// 定时任务回调
typedef void TaskFunc(void* clientData);

// Socket处理回调
typedef void BackgroundHandlerProc(void* clientData, int mask);
```

**为什么用 `void*` ？**

- C风格的回调函数需要固定的函数签名
- `void*` 是"万能指针"，可以传递任何类型的数据
- 在回调函数内部需要将 `void*` 转回实际类型

### 4.3 回调函数的典型实现模式

```cpp
class MyClass {
public:
    void startOperation() {
        // 注册回调时传递this指针
        env->taskScheduler().scheduleDelayedTask(
            1000000,
            staticCallback,  // 必须是静态函数
            this             // 传递this
        );
    }
    
private:
    // 静态回调函数（作为"桥梁"）
    static void staticCallback(void* clientData) {
        // 将void*转回MyClass*
        MyClass* self = (MyClass*)clientData;
        // 调用成员函数
        self->memberCallback();
    }
    
    // 实际的处理函数（可以访问成员变量）
    void memberCallback() {
        // 这里可以访问this指针和所有成员变量
        fCounter++;
        *env << "Callback called, counter = " << fCounter << "\n";
    }
    
    int fCounter = 0;
    UsageEnvironment* env;
};
```

### 4.4 回调函数注意事项

```
┌─────────────────────────────────────────────────────────────────┐
│                    回调函数注意事项                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ⚠️  1. 回调函数必须快速返回                                    │
│      - Live555是单线程的                                        │
│      - 回调函数阻塞会导致整个程序卡住                           │
│      - 耗时操作应该拆分成多个小任务                             │
│                                                                 │
│  ⚠️  2.  不要在回调中删除正在使用的对象                          │
│      - 可能导致野指针                                           │
│      - 应该调度一个延时任务来删除                               │
│                                                                 │
│  ⚠️  3. 注意对象生命周期                                        │
│      - 确保回调执行时，传入的clientData仍然有效                 │
│      - 对象销毁前要取消所有注册的回调                           │
│                                                                 │
│  ⚠️  4. 静态函数 vs 成员函数                                    │
│      - 回调必须是静态函数或全局函数                             │
│      - 通过clientData传递this指针来访问成员                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 五、辅助类介绍

### 5.1 Boolean 类型

```cpp
// 文件：UsageEnvironment/include/Boolean.hh

#ifndef _BOOLEAN_HH
#define _BOOLEAN_HH

typedef unsigned char Boolean;
#define False 0
#define True 1

#endif
```

Live555 使用自定义的`Boolean`类型而不是C++的`bool`，这是为了：
- 兼容早期不支持`bool`的编译器
- 保证跨平台的一致性

### 5.2 strDup 字符串工具

```cpp
// 文件：UsageEnvironment/include/strDup.hh

// 复制字符串（分配新内存）
char* strDup(char const* str);

// 复制字符串，指定最大长度
char* strDupSize(char const* str, size_t& resultBufSize);
```

**使用示例：**

```cpp
const char* original = "Hello";
char* copy = strDup(original);  // 分配新内存并复制

// 使用完毕后需要释放
delete[] copy;
```

---

## 六、小结

### 6.1 要点

1. **UsageEnvironment** 是抽象基类，定义了框架运行所需的基础设施接口
2. **TaskScheduler** 提供四大核心功能：
   - 延时任务调度
   - Socket后台处理
   - 事件循环
   - 事件触发器
3. **回调机制** 是 Live555 事件驱动的核心
4. **单线程模型**：所有回调在同一线程中串行执行

### 6.2 类关系图

```
┌─────────────────────────────────────────────────────────────────┐
│                       类关系图                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   UsageEnvironment (抽象类)                                     │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │ + taskScheduler(): TaskScheduler&                       │   │
│   │ + setResultMsg(msg): void                               │   │
│   │ + operator<<(str): UsageEnvironment&                    │   │
│   │ - fScheduler: TaskScheduler&                            │   │
│   └─────────────────────────────────────────────────────────┘   │
│                              │ 聚合                             │
│                              ▼                                  │
│   TaskScheduler (抽象类)                                        │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │ + scheduleDelayedTask(... ): TaskToken                   │   │
│   │ + setBackgroundHandling(...): void                      │   │
│   │ + doEventLoop(...): void                                │   │
│   │ + createEventTrigger(...): EventTriggerId               │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

