

# Live555源码调试笔记1——Live555事件驱动Demo

在学习完基础层（`BasicUsageEnvironment`）和网络层（`groupsock`）后，我想先抛开后面复杂的媒体层来写一个Demo来实战理解Live555“**事件驱动**”的核心机制。

#### Demo设计思路：UDP心跳收发器

这个程序不传输复杂的视频，只传输简单的文本字符串，但它完整地演示了Live555的核心三要素：**定时器任务**、**Socket 读取事件**、**事件循环**。程序中有两个角色：

1. **发送者 (Sender)**：利用`scheduleDelayedTask`（定时器），每隔1秒向组播地址发送一句 "Hello Live555"。
2. **接收者 (Receiver)**：利用`setBackgroundHandling`（Socket 事件），监听组播端口。当有数据到达时，触发回调打印出来。

**涉及的核心类：**

- `BasicTaskScheduler`：整个程序的发动机。
- `BasicUsageEnvironment`：提供日志和环境上下文。
- `Groupsock`：封装 UDP/组播 Socket 操作。

#### 代码实现

```c++
#include "liveMedia.hh"
#include "BasicUsageEnvironment.hh"
#include "GroupsockHelper.hh"

// 定义组播地址和端口
char const* multicastAddressStr = "239.255.42.42";
Port const multicastPort(8888);
const unsigned char ttl = 255;

// 全局环境指针
UsageEnvironment* env;

// =============================================================
//  接收端逻辑：处理 Socket 可读事件
// =============================================================
#define MAX_PACKET_SIZE 2048

// 这是核心回调函数：当 Socket 有数据时，TaskScheduler 会调用它
void incomingDataHandler(void* clientData, int mask) {
    Groupsock* socket = (Groupsock*)clientData;
    
    unsigned char buffer[MAX_PACKET_SIZE];
    struct sockaddr_storage fromAddress;
    
    // 1. 读取数据 (非阻塞，因为 select 告诉我们需要读了)
    // handleRead 是 Groupsock 的方法，封装了 recvfrom
    unsigned bytesRead = 0;
    if (socket->handleRead(buffer, MAX_PACKET_SIZE, bytesRead, fromAddress)) {
        // 确保字符串结束符
        if (bytesRead < MAX_PACKET_SIZE) buffer[bytesRead] = '\0';
        
        // 2. 打印接收到的信息
        *env << "[Receiver] Got " << bytesRead << " bytes: " << (char*)buffer << "\n";
    }
}

// =============================================================
//  发送端逻辑：定时发送任务
// =============================================================
void periodicSendTask(void* clientData) {
    Groupsock* socket = (Groupsock*)clientData;
    
    // 1. 构造数据
    static int counter = 0;
    char sendBuffer[100];
    sprintf(sendBuffer, "Hello Live555 heartbeat #%d", ++counter);
    
    // 2. 发送数据
    // output 直接把数据发给 Groupsock 绑定的组播地址
    socket->output(*env, (unsigned char*)sendBuffer, strlen(sendBuffer));
    *env << "[Sender] Sent: " << sendBuffer << "\n";
    
    // 3. 重新调度自己：1秒(1000000微秒)后再次执行
    // 这就是 Live555 的"死循环"驱动方式
    env->taskScheduler().scheduleDelayedTask(1000000, periodicSendTask, socket);
}

int main(int argc, char** argv) {
    // 1. 初始化环境 (必须步骤)
    TaskScheduler* scheduler = BasicTaskScheduler::createNew();
    env = BasicUsageEnvironment::createNew(*scheduler);

    // 2. 创建网络地址对象
    struct sockaddr_storage addr;
    // 使用 GroupsockHelper 中的工具函数解析 IP
    NetAddressList addresses(multicastAddressStr);
    if (addresses.numAddresses() == 0) {
        *env << "Failed to find address for " << multicastAddressStr << "\n";
        return 1;
    }
    // 获取第一个地址
    // 注意：这里简化了 IP 版本处理，假设是 IPv4
    NetAddress const& netAddr = *addresses.firstAddress(); 
    // Live555 的 IP 地址处理稍微有点繁琐，这里简化处理，实际要构建 sockaddr_storage
    // 为了演示方便，我们利用 Groupsock 构造函数自动处理
    
    struct in_addr ipAddr;
    ipAddr.s_addr = *(unsigned*)(netAddr.data()); 
    
    struct sockaddr_in* ipv4Addr = (struct sockaddr_in*)&addr;
    ipv4Addr->sin_family = AF_INET;
    ipv4Addr->sin_port = multicastPort.num();
    ipv4Addr->sin_addr = ipAddr;

    // 3. 创建 Groupsock 对象
    // 这个对象封装了 socket(), bind(), setsockopt(IP_ADD_MEMBERSHIP) 等底层操作
    Groupsock rtpGroupsock(*env, addr, multicastPort, ttl);
    rtpGroupsock.multicastSendOnly(); // 如果只想发不想收可以调这个，代码内部是注释了的，所以可以接收

    // =========================================================
    //  关键步骤 A：注册接收处理 (Input)
    // =========================================================
    // 获取底层 Socket 文件描述符
    int socketNum = rtpGroupsock.socketNum();
    
    // 告诉调度器：盯着这个 socketNum，当它 "SOCKET_READABLE" 时，
    // 调用 incomingDataHandler，并把 rtpGroupsock 指针传过去
    env->taskScheduler().setBackgroundHandling(
        socketNum, 
        SOCKET_READABLE, 
        incomingDataHandler, 
        &rtpGroupsock
    );

    // =========================================================
    //  关键步骤 B：启动定时发送任务 (Timer)
    // =========================================================
    // 启动第一次发送，后续会在回调里自我调度
    env->taskScheduler().scheduleDelayedTask(0, periodicSendTask, &rtpGroupsock);

    *env << "Demo started. Press Ctrl+C to exit...\n";

    // 4. 进入事件循环 (心脏开始跳动)
    // 这一行代码之后，main 函数就再也不会往下执行了
    // 所有的逻辑都将在 SingleStep() -> select() 返回后被分发执行
    env->taskScheduler().doEventLoop();

    return 0; // Unreachable
}
```

##### 编译与运行

可以将Demo放在testProgs/目录下，注意在编译前需要修改该目录下的Makefile.tail文件。同时为了方便GDB调试，可以生成Debug版本的Makefile，源码刚好有这个配置，执行`./genMakefiles linux-gdb`然后再执行`make`，就得到可执行文件。

#### 调试与理解

##### Socket事件回调的调试与理解

在回调函数`void incomingDataHandler(void* clientData, int mask)`打上断点，当断点触发时调试信息如下：

![image-20260122112149620](C:/Users/Administrator/AppData/Roaming/Typora/typora-user-images/image-20260122112149620.png)

程序从main函数入口进入调度器的`doEventLoop`函数内部，这个函数的内部是一个while循环，它在循环中不停地调用`SingleStep`。这个函数的153行的代码如下：

```c++
(*handler->handlerProc)(handler->clientData, resultConditionSet);
```

`SingleStep`先调用了`select()`，发现Socket有数据可读；然后遍历链表找到负责这个Socket的handle；它通过函数指针 `(*handler->handlerProc)` 调用了注册的函数。

```c++
#0  incomingDataHandler (clientData=0x..., mask=2) at testBasicNet.cpp:19
```

再回到最上层，它的含义是：

* `clientData`: 这就是你在main里传进去的 `&rtpGroupsock` 指针。
* `mask=2`: 这是一个位掩码。在Live555里，`SOCKET_READABLE` 通常定义为 2。这告诉你：“我是因为**可读**事件才叫你的”

##### 定时器事件调试与理解

在`SingleStep`打一个断点，然后逐步执行，调试结果如下：

![image-20260122152611883](https://gitee.com/ruoying0119/picture/raw/master/images/20260122152611933.png)

在第71行，`timeToNextAlarm()` 计算出了距离下一次发送还有多久（这里是 1000ms）。select拿到这个时间去睡觉，因为这段时间内没有数据到达，所以select一觉睡醒，返回0（或者正数但没置位）。

![image-20260122153237278](https://gitee.com/ruoying0119/picture/raw/master/images/20260122153318042.png)

然后单步执行，发现光标一直在161-167之间跳，最后跳出了循环。这个过程发生了：

* 调度器很负责，它挨个询问了所有的 Socket：“刚才 select 醒了，是你们谁有动静吗？”
* `FD_ISSET` 告诉它：“不是我，我没动静。”
* 因为没有任何 Socket 没有可读可写事件，所以 `handler->handlerProc` 完全没有被执行。

执行到209行，屏幕立刻出现了Sent打印：

* 既然不是网络事件叫醒的，那肯定是时间到了。
* `handleAlarm()` 检查延时队列，发现 `periodicSendTask` 到时间了。
* 它立刻执行这个任务，打印了Log。

