

# Live555源码调试笔记2——Source-Sink数据流模型

#### Demo设计思路

在搞清楚Live555的事件驱动模型之后，再来看Live555在媒体流处理的核心：**Source-Sink 数据流模型**。所有的 RTSP 服务器（包括 VLC、FFmpeg 的实现）在数据传输层几乎都遵循这个模式。

在上一个Demo的基础上，用Source/Sink模式重写接收端：

* **生产者 (Source)**：使用 `BasicUDPSource`（Live555 自带类，它封装了 UDP Socket 的读取）。
* **消费者 (Sink)**：我们需要手写一个 `DummySink`（模拟 `RTPSink` 的行为），它负责不断地向 Source **“索要”**数据。

##### 代码如下：

```c++
#include "liveMedia.hh"
#include "BasicUsageEnvironment.hh"
#include "GroupsockHelper.hh"
#include "BasicUDPSource.hh" // 必须引入这个

// 定义组播地址和端口
char const* multicastAddressStr = "239.255.42.42";
Port const multicastPort(8888);
const unsigned char ttl = 255;

UsageEnvironment* env;

// =============================================================
//  Sink 类定义：模拟媒体播放器或文件写入器
//  核心逻辑：startPlaying -> getNextFrame -> 回调 -> 处理数据 -> getNextFrame ...
// =============================================================
class DummySink: public MediaSink {
public:
  static DummySink* createNew(UsageEnvironment& env) {
    return new DummySink(env);
  }

protected:
  DummySink(UsageEnvironment& env): MediaSink(env) {
      // 1. 准备一个缓冲区来接收数据
      fReceiveBufferSize = 2048;
      fReceiveBuffer = new unsigned char[fReceiveBufferSize];
  }
  virtual ~DummySink() { delete[] fReceiveBuffer; }

  // [关键点 1] 开始运行
  // 当你调用 sink->startPlaying() 时，基类会最终调用这个虚函数
  virtual Boolean continuePlaying() {
    if (fSource == NULL) return False;

    // [关键点 2] 主动向 Source 伸手要数据 (Pull)
    // 参数：
    // 1. 接收缓冲区
    // 2. 缓冲区最大大小
    // 3. 回调函数：数据到了告诉我 (afterGettingFrame)
    // 4. clientData：我是谁 (this)
    // 5. onClose：源关闭了告诉我
    fSource->getNextFrame(fReceiveBuffer, fReceiveBufferSize,
                          afterGettingFrame, this,
                          onSourceClosure, this);
    return True;
  }

  // [关键点 3] 数据到达回调 (静态函数)
  static void afterGettingFrame(void* clientData, unsigned frameSize,
                                unsigned numTruncatedBytes,
                                struct timeval presentationTime,
                                unsigned durationInMicroseconds) {
    DummySink* sink = (DummySink*)clientData;
    sink->onFrameReceived(frameSize, presentationTime);
  }

  // 处理接收到的帧
  void onFrameReceived(unsigned frameSize, struct timeval presentationTime) {
    // 打印数据
    fReceiveBuffer[frameSize] = '\0';
    envir() << "[DummySink] Pull got " << frameSize << " bytes: " << (char*)fReceiveBuffer << "\n";

    // [关键点 4] 形成闭环：处理完这一帧，立马去拉下一帧！
    // 如果不调用这个，数据流就会在这里中断
    continuePlaying();
  }
  
  static void onSourceClosure(void* clientData) {
      printf("Source closed\n");
  }

private:
  unsigned char* fReceiveBuffer;
  unsigned fReceiveBufferSize;
};

// =============================================================
//  发送端逻辑 (保持不变，为了产生数据)
// =============================================================
void periodicSendTask(void* clientData) {
    Groupsock* socket = (Groupsock*)clientData;
    static int counter = 0;
    char sendBuffer[100];
    sprintf(sendBuffer, "Hello Live555 heartbeat #%d", ++counter);
    
    socket->output(*env, (unsigned char*)sendBuffer, strlen(sendBuffer));
    *env << "[Sender] Sent: " << sendBuffer << "\n";
    
    env->taskScheduler().scheduleDelayedTask(1000000, periodicSendTask, socket);
}

int main(int argc, char** argv) {
    TaskScheduler* scheduler = BasicTaskScheduler::createNew();
    env = BasicUsageEnvironment::createNew(*scheduler);

    // 1. 网络地址设置 (和之前一样)
    struct sockaddr_storage addr;
    NetAddressList addresses(multicastAddressStr);
    if (addresses.numAddresses() == 0) return 1;
    NetAddress const& netAddr = *addresses.firstAddress(); 
    struct in_addr ipAddr; ipAddr.s_addr = *(unsigned*)(netAddr.data()); 
    struct sockaddr_in* ipv4Addr = (struct sockaddr_in*)&addr;
    ipv4Addr->sin_family = AF_INET;
    ipv4Addr->sin_port = multicastPort.num();
    ipv4Addr->sin_addr = ipAddr;

    // 2. 创建 Groupsock
    Groupsock rtpGroupsock(*env, addr, multicastPort, ttl);

    // =========================================================
    //  关键变化：使用 Source -> Sink 模式
    // =========================================================
    
    // A. 创建 Source (生产者)
    // BasicUDPSource 会自动管理底层的 Socket 读取事件
    BasicUDPSource* source = BasicUDPSource::createNew(*env, &rtpGroupsock);

    // B. 创建 Sink (消费者)
    DummySink* sink = DummySink::createNew(*env);

    // C. 启动数据流！
    // 这一句就像接通了水管的阀门
    *env << "Starting Sink (Pull Model)...\n";
    sink->startPlaying(*source, NULL, NULL);

    // =========================================================

    // 启动发送任务 (为了测试)
    env->taskScheduler().scheduleDelayedTask(0, periodicSendTask, &rtpGroupsock);

    env->taskScheduler().doEventLoop();
    return 0;
}
```

#### 运行与调试

##### 观察请求阶段

在`DummySink::continuePlaying`打上断点，运行调试如下：

![image-20260123170535748](https://gitee.com/ruoying0119/picture/raw/master/images/20260123170535798.png)通过这个调用栈可以看到，现在的 `continuePlaying` 是由 main 函数“同步”调用的，而不是由事件循环调用的。

* #2 `main`: 代码里写了 `sink->startPlaying(...)`。
* #1 `MediaSink::startPlaying`: 这是基类的代码。它的逻辑很简单，就是初始化一下，然后立即调用子类的 continuePlaying。
* #0 `DummySink::continuePlaying`: 也就是现在停下的地方。

然后单步执行后，直到调用 `fSource->getNextFrame()`，再次单步执行后，程序的执行瞬间跨过了`getNextFrame`，程序回到了main函数，没有任何卡顿。调用`fSource->getNextFrame()`后发生了如下过程：

* `BasicUDPSource` 收到请求，它并没有去读 Socket（因为可能没数据），它只是在心里记了一个账：“Sink 想要数据了，回调函数是 `afterGettingFrame`”。
* 然后它**立即返回**了。

##### 观察回调阶段

在`DummySink::afterGettingFrame`打上断点然后继续运行，调试如下：

![image-20260123172715497](https://gitee.com/ruoying0119/picture/raw/master/images/20260123172715549.png)

`doEventLoop`正在运行然后网络包到达，调度器将回调叫醒，整个过程如下：

1. #4 `SingleStep`(老板)：发现 UDP Socket 变红（Readable）。
2. #3 `incomingPacketHandler`(适配器)**：调度器调用的标准 C 格式回调。
3. #2 `incomingPacketHandler1`(干活的人)**：这是 `BasicUDPSource` 真正的成员函数。它调用系统 API`recvfrom`从内核读取了26字节的数据（"Hello Live555..."）。
4. #1 `FramedSource::afterGetting`(信使)：它发现之前有一笔“记账”（Sink 请求过数据），于是它履行承诺，准备调用回调。
5. #0 `afterGettingFrame` (消费者)：数据交到了你的手里。

再次继续单步执行，`afterGettingFrame`继续执行，然后执行`sink->onFrameReceived`，完成取一次的操作。继续单步执行如下：

![image-20260123174550148](https://gitee.com/ruoying0119/picture/raw/master/images/20260123174646672.png)由于`DummySink::continuePlaying`中调用了`continuePlaying()`，它会立刻去拉下一帧。此时Sink说“我要读第二个包，有货了叫我”，控制权交给调度器，Sink此时在等数据，调度器并没有傻等，它发现定时器到了，然后先去执行了发送任务，然后再次回到前面的流程。