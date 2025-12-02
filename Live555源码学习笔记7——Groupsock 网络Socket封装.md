# Live555源码学习笔记7——Groupsock 网络Socket封装

## 一、模块概述

### 1.1 Groupsock 模块的作用

Groupsock 模块是 Live555 网络层的核心，它封装了 UDP Socket 的操作，并提供了对**组播（Multicast）**的完整支持。

```c++
┌─────────────────────────────────────────────────────────────────┐
│                   Groupsock 模块在系统中的位置                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   liveMedia 层                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │ RTPSink      - 需要Groupsock发送RTP数据                 │   │
│   │ RTPSource    - 需要Groupsock接收RTP数据                 │   │
│   │ RTCPInstance - 需要Groupsock收发RTCP数据                │   │
│   └─────────────────────────────────────────────────────────┘   │
│                              │                                  │
│                              │ 使用                             │
│                              ▼                                  │
│   Groupsock 模块 ⭐ 本节内容                                    │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                                                         │   │
│   │   NetInterface   - 网络接口基类                         │   │
│   │   Socket         - Socket基类（UDP）                    │   │
│   │   OutputSocket   - 仅发送的Socket                       │   │
│   │   Groupsock      - 完整的组播/单播Socket ⭐              │   │
│   │   GroupsockHelper- 辅助函数                             │   │
│   │                                                         │   │
│   └─────────────────────────────────────────────────────────┘   │
│                              │                                  │
│                              ▼                                  │
│   NetAddress 模块（上一课）                                     │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │ NetAddress / Port / AddressString                       │   │
│   └─────────────────────────────────────────────────────────┘   │
│                              │                                  │
│                              ▼                                  │
│   操作系统 Socket API                                           │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │ socket() / bind() / sendto() / recvfrom()               │   │
│   │ setsockopt() / getsockopt() / close()                   │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 文件结构

```
groupsock/
├── include/
│   ├── Groupsock.hh          ⭐ 核心类定义
│   ├── NetInterface.hh       Socket基类
│   ├── GroupEId.hh           组端点标识
│   ├── GroupsockHelper.hh    辅助函数
│   └── IOHandlers.hh         IO处理器
├── Groupsock.cpp             Groupsock实现
├── NetInterface.cpp          Socket实现
├── GroupEId.cpp              GroupEId实现
├── GroupsockHelper.cpp       辅助函数实现
└── IOHandlers.cpp            IO处理器实现
```

### 1.3 类继承结构

```
┌─────────────────────────────────────────────────────────────────┐
│                    类继承结构                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   NetInterface                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │ 网络接口抽象基类                                        │   │
│   │ + DefaultUsageEnvironment: static                       │   │
│   └───────────────────────────┬─────────────────────────────┘   │
│                               │                                 │
│                               ▼                                 │
│   Socket                                                        │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │ UDP Socket封装                                          │   │
│   │ - fSocketNum: int (文件描述符)                          │   │
│   │ - fPort: Port                                           │   │
│   │ - fFamily: int (AF_INET/AF_INET6)                       │   │
│   │ + handleRead(): 纯虚函数                                │   │
│   └───────────────────────────┬─────────────────────────────┘   │
│                               │                                 │
│                               ▼                                 │
│   OutputSocket                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │ 专用于发送的Socket                                      │   │
│   │ - fSourcePort: Port                                     │   │
│   │ - fLastSentTTL: unsigned                                │   │
│   │ + write(): 发送数据                                     │   │
│   └───────────────────────────┬─────────────────────────────┘   │
│                               │                                 │
│                               ▼                                 │
│   Groupsock ⭐                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │ 完整的组播/单播Socket                                   │   │
│   │ - fDests: destRecord* (目标地址链表)                    │   │
│   │ - fIncomingGroupEId: GroupEId                           │   │
│   │ + output(): 发送到所有目标                              │   │
│   │ + handleRead(): 接收数据                                │   │
│   │ + addDestination(): 添加目标                            │   │
│   │ + removeDestination(): 移除目标                         │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 二、NetInterface 基类

### 2.1 类定义

```cpp
// 文件：NetInterface.hh

class NetInterface {
public:
    virtual ~NetInterface();

    // 全局默认使用环境
    // 如果设置了这个值，新创建的接口会优先使用它
    static UsageEnvironment* DefaultUsageEnvironment;

protected:
    NetInterface();  // 抽象基类，不能直接实例化
};
```

### 2.2 实现

```cpp
// 文件：NetInterface.cpp

UsageEnvironment* NetInterface::DefaultUsageEnvironment = NULL;

NetInterface::NetInterface() {
}

NetInterface::~NetInterface() {
}
```

---

## 三、Socket 类

### 3.1 类定义

```cpp
// 文件：NetInterface.hh

class Socket: public NetInterface {
public:
    virtual ~Socket();
    
    // 重置Socket（关闭并将fSocketNum设为-1）
    void reset();

    // 纯虚函数：处理数据读取（子类必须实现）
    virtual Boolean handleRead(unsigned char* buffer, unsigned bufferMaxSize,
                               unsigned& bytesRead,
                               struct sockaddr_storage& fromAddress) = 0;

    // 获取Socket信息
    int socketNum() const { return fSocketNum; }
    Port port() const { return fPort; }
    UsageEnvironment& env() const { return fEnv; }

    // 调试级别（静态成员，影响所有Socket）
    static int DebugLevel;

protected:
    Socket(UsageEnvironment& env, Port port, int family);
    Boolean changePort(Port newPort);

private:
    int fSocketNum;           // Socket文件描述符
    UsageEnvironment& fEnv;   // 使用环境引用
    Port fPort;               // 绑定的端口
    int fFamily;              // 地址族 (AF_INET 或 AF_INET6)
};
```

### 3.2 实现

```cpp
// 文件：NetInterface.cpp

int Socket::DebugLevel = 1;  // 默认调试级别

Socket::Socket(UsageEnvironment& env, Port port, int family)
    : fEnv(DefaultUsageEnvironment != NULL ? *DefaultUsageEnvironment : env),
      fPort(port), 
      fFamily(family) {
    // 调用辅助函数创建UDP Socket
    fSocketNum = setupDatagramSocket(fEnv, port, family);
}

void Socket::reset() {
    if (fSocketNum >= 0) {
        closeSocket(fSocketNum);
    }
    fSocketNum = -1;
}

Socket::~Socket() {
    reset();
}

Boolean Socket::changePort(Port newPort) {
    int oldSocketNum = fSocketNum;
    
    // 保存旧的缓冲区大小
    unsigned oldReceiveBufferSize = getReceiveBufferSize(fEnv, fSocketNum);
    unsigned oldSendBufferSize = getSendBufferSize(fEnv, fSocketNum);
    
    // 关闭旧Socket
    closeSocket(fSocketNum);

    // 创建新Socket，绑定新端口
    fSocketNum = setupDatagramSocket(fEnv, newPort, fFamily);
    if (fSocketNum < 0) {
        fEnv.taskScheduler().turnOffBackgroundReadHandling(oldSocketNum);
        return False;
    }

    // 恢复缓冲区大小
    setReceiveBufferTo(fEnv, fSocketNum, oldReceiveBufferSize);
    setSendBufferTo(fEnv, fSocketNum, oldSendBufferSize);
    
    // 如果Socket号变了，迁移事件处理
    if (fSocketNum != oldSocketNum) {
        fEnv.taskScheduler().moveSocketHandling(oldSocketNum, fSocketNum);
    }
    
    return True;
}
```

---

## 四、OutputSocket 类

### 4.1 类定义

```cpp
// 文件：Groupsock.hh

class OutputSocket: public Socket {
public:
    OutputSocket(UsageEnvironment& env, int family);
    virtual ~OutputSocket();

    // 发送数据到指定地址
    virtual Boolean write(struct sockaddr_storage const& addressAndPort, 
                          u_int8_t ttl,
                          unsigned char* buffer, unsigned bufferSize);

protected:
    OutputSocket(UsageEnvironment& env, Port port, int family);
    portNumBits sourcePortNum() const { return fSourcePort. num(); }

private:
    virtual Boolean handleRead(unsigned char* buffer, unsigned bufferMaxSize,
                               unsigned& bytesRead,
                               struct sockaddr_storage& fromAddressAndPort);

private:
    Port fSourcePort;        // 源端口（发送后获取）
    unsigned fLastSentTTL;   // 上次发送使用的TTL
};
```

### 4.2 write() 实现

```cpp
// 文件：Groupsock.cpp

Boolean OutputSocket::write(struct sockaddr_storage const& addressAndPort, 
                            u_int8_t ttl,
                            unsigned char* buffer, unsigned bufferSize) {
    // TTL优化：如果TTL没变，跳过setsockopt调用
    if ((unsigned)ttl == fLastSentTTL) {
        if (! writeSocket(env(), socketNum(), addressAndPort, buffer, bufferSize)) {
            return False;
        }
    } else {
        if (!writeSocket(env(), socketNum(), addressAndPort, ttl, buffer, bufferSize)) {
            return False;
        }
        fLastSentTTL = (unsigned)ttl;
    }

    // 第一次发送后，获取内核分配的源端口
    if (sourcePortNum() == 0) {
        if (!getSourcePort(env(), socketNum(), addressAndPort. ss_family, fSourcePort)) {
            if (DebugLevel >= 1) {
                env() << *this << ": failed to get source port\n";
            }
            return False;
        }
    }

    return True;
}
```

---

## 五、GroupEId 类（组端点标识）

### 5.1 类定义

```cpp
// 文件：GroupEId.hh

class GroupEId {
public:
    // 源无关组播 (ASM)
    GroupEId(struct sockaddr_storage const& groupAddr,
             portNumBits portNum, u_int8_t ttl);
    
    // 源特定组播 (SSM)
    GroupEId(struct sockaddr_storage const& groupAddr,
             struct sockaddr_storage const& sourceFilterAddr,
             portNumBits portNum);

    struct sockaddr_storage const& groupAddress() const { return fGroupAddress; }
    struct sockaddr_storage const& sourceFilterAddress() const { return fSourceFilterAddress; }
    Boolean isSSM() const;
    portNumBits portNum() const;
    u_int8_t ttl() const { return fTTL; }

private:
    struct sockaddr_storage fGroupAddress;
    struct sockaddr_storage fSourceFilterAddress;
    u_int8_t fTTL;
};
```

### 5.2 ASM vs SSM 组播

```
┌─────────────────────────────────────────────────────────────────┐
│                    ASM vs SSM 组播                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   【ASM - Any Source Multicast（任意源组播）】                   │
│                                                                 │
│   • 接收来自任何源的组播数据                                    │
│   • 只需指定：组播地址 + 端口 + TTL                             │
│   • 使用 socketJoinGroup() 加入                                 │
│                                                                 │
│         源A ──┐                                                 │
│               ├──► 组播组 239.1.1.1 ──► 接收方                  │
│         源B ──┘         (接收所有源)                            │
│                                                                 │
│   ─────────────────────────────────────────────────────────────│
│                                                                 │
│   【SSM - Source Specific Multicast（源特定组播）】              │
│                                                                 │
│   • 只接收指定源的组播数据                                      │
│   • 需要指定：组播地址 + 源地址 + 端口                          │
│   • 使用 socketJoinGroupSSM() 加入                              │
│   • 更安全，避免恶意源攻击                                      │
│                                                                 │
│         源A (10.0.0. 1) ──► 组播组 232.1.1. 1 ──► 接收方          │
│         源B (10.0.0.2) ──► 组播组 232. 1.1.1 ──✗ (被过滤)        │
│                           (只接收源A)                           │
│                                                                 │
│   SSM地址范围：232.0.0.0/8 (IPv4)                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 六、destRecord 类（目标记录）

### 6.1 类定义

```cpp
// 文件：Groupsock.hh

class destRecord {
public:
    destRecord(struct sockaddr_storage const& addr, Port const& port, 
               u_int8_t ttl, unsigned sessionId, destRecord* next);
    virtual ~destRecord();

public:
    destRecord* fNext;        // 链表下一个节点
    GroupEId fGroupEId;       // 目标端点信息（地址+端口+TTL）
    unsigned fSessionId;      // 会话ID（用于标识和移除）
};
```

### 6.2 实现

```cpp
// 文件：Groupsock. cpp

destRecord::destRecord(struct sockaddr_storage const& addr, Port const& port, 
                       u_int8_t ttl, unsigned sessionId, destRecord* next)
    : fNext(next), 
      fGroupEId(addr, port. num(), ttl), 
      fSessionId(sessionId) {
}

destRecord::~destRecord() {
    delete fNext;  // 递归删除整个链表
}
```

### 6.3 目标链表结构

```
┌─────────────────────────────────────────────────────────────────┐
│                    destRecord 链表结构                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   Groupsock::fDests                                             │
│         │                                                       │
│         ▼                                                       │
│   ┌─────────────────┐      ┌─────────────────┐                  │
│   │  destRecord #1  │─────►│  destRecord #2  │─────► NULL       │
│   │                 │      │                 │                  │
│   │ addr: 10.0.0.1  │      │ addr: 10.0.0.2  │                  │
│   │ port: 5000      │      │ port: 5000      │                  │
│   │ ttl: 128        │      │ ttl: 128        │                  │
│   │ sessionId: 1    │      │ sessionId: 2    │                  │
│   └─────────────────┘      └─────────────────┘                  │
│                                                                 │
│   用途：                                                        │
│   • 组播时只有一个目标                                          │
│   • 多单播时可以有多个目标                                      │
│   • output() 会遍历发送到每个目标                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 七、Groupsock 类（核心）

### 7.1 类定义

```cpp
// 文件：Groupsock.hh

class Groupsock: public OutputSocket {
public:
    // 构造函数1：源无关组播 (ASM)
    Groupsock(UsageEnvironment& env, struct sockaddr_storage const& groupAddr,
              Port port, u_int8_t ttl);
    
    // 构造函数2：源特定组播 (SSM)
    Groupsock(UsageEnvironment& env, struct sockaddr_storage const& groupAddr,
              struct sockaddr_storage const& sourceFilterAddr, Port port);

    virtual ~Groupsock();

    // 修改目标参数
    void changeDestinationParameters(struct sockaddr_storage const& newDestAddr,
                                     Port newDestPort, int newDestTTL,
                                     unsigned sessionId = 0);

    // 多目标支持
    virtual void addDestination(struct sockaddr_storage const& addr, 
                                Port const& port, unsigned sessionId);
    virtual void removeDestination(unsigned sessionId);
    void removeAllDestinations();
    Boolean hasMultipleDestinations() const;

    // 获取组播信息
    struct sockaddr_storage const& groupAddress() const;
    struct sockaddr_storage const& sourceFilterAddress() const;
    Boolean isSSM() const;
    u_int8_t ttl() const;

    // 设置为仅发送模式
    void multicastSendOnly();

    // 发送数据到所有目标
    virtual Boolean output(UsageEnvironment& env, 
                           unsigned char* buffer, unsigned bufferSize);

    // 接收数据
    virtual Boolean handleRead(unsigned char* buffer, unsigned bufferMaxSize,
                               unsigned& bytesRead,
                               struct sockaddr_storage& fromAddressAndPort);

    // 流量统计
    static NetInterfaceTrafficStats statsIncoming;
    static NetInterfaceTrafficStats statsOutgoing;
    NetInterfaceTrafficStats statsGroupIncoming;
    NetInterfaceTrafficStats statsGroupOutgoing;

protected:
    destRecord* fDests;
private:
    GroupEId fIncomingGroupEId;
};
```

### 7.2 ASM 构造函数

```cpp
// 文件：Groupsock.cpp

// 源无关组播构造函数
Groupsock::Groupsock(UsageEnvironment& env, 
                     struct sockaddr_storage const& groupAddr,
                     Port port, u_int8_t ttl)
    : OutputSocket(env, port, groupAddr.ss_family),
      fDests(new destRecord(groupAddr, port, ttl, 0, NULL)),
      fIncomingGroupEId(groupAddr, port. num(), ttl) {
    
    // 加入组播组
    if (! socketJoinGroup(env, socketNum(), groupAddr)) {
        if (DebugLevel >= 1) {
            env << *this << ": failed to join group: "
                << env.getResultMsg() << "\n";
        }
    }

    // 确保能获取到自己的IP地址
    if (!weHaveAnIPAddress(env)) {
        if (DebugLevel >= 0) {
            env << "Unable to determine our source address: "
                << env.getResultMsg() << "\n";
        }
    }

    if (DebugLevel >= 2) env << *this << ": created\n";
}
```

### 7.3 SSM 构造函数

```cpp
// 源特定组播构造函数
Groupsock::Groupsock(UsageEnvironment& env, 
                     struct sockaddr_storage const& groupAddr,
                     struct sockaddr_storage const& sourceFilterAddr,
                     Port port)
    : OutputSocket(env, port, groupAddr.ss_family),
      fDests(new destRecord(groupAddr, port, 255, 0, NULL)),
      fIncomingGroupEId(groupAddr, sourceFilterAddr, port.num()) {
    
    // 首先尝试SSM加入
    if (! socketJoinGroupSSM(env, socketNum(), groupAddr, sourceFilterAddr)) {
        if (DebugLevel >= 3) {
            env << *this << ": SSM join failed - trying regular join\n";
        }
        // SSM失败则尝试普通加入
        if (!socketJoinGroup(env, socketNum(), groupAddr)) {
            if (DebugLevel >= 1) {
                env << *this << ": failed to join group\n";
            }
        }
    }

    if (DebugLevel >= 2) env << *this << ": created\n";
}
```

### 7.4 析构函数

```cpp
Groupsock::~Groupsock() {
    // 离开组播组
    if (isSSM()) {
        if (!socketLeaveGroupSSM(env(), socketNum(), groupAddress(), 
                                  sourceFilterAddress())) {
            socketLeaveGroup(env(), socketNum(), groupAddress());
        }
    } else {
        socketLeaveGroup(env(), socketNum(), groupAddress());
    }

    // 删除目标链表
    delete fDests;

    if (DebugLevel >= 2) env() << *this << ": deleting\n";
}
```

### 7.5 output() - 发送数据

```cpp
Boolean Groupsock::output(UsageEnvironment& env, 
                          unsigned char* buffer, unsigned bufferSize) {
    do {
        // 发送到每一个目标
        Boolean writeSuccess = True;
        for (destRecord* dests = fDests; dests != NULL; dests = dests->fNext) {
            if (! write(dests->fGroupEId.groupAddress(), 
                       dests->fGroupEId.ttl(), 
                       buffer, bufferSize)) {
                writeSuccess = False;
                break;
            }
        }
        if (! writeSuccess) break;
        
        // 更新统计
        statsOutgoing.countPacket(bufferSize);
        statsGroupOutgoing.countPacket(bufferSize);

        if (DebugLevel >= 3) {
            env << *this << ": wrote " << bufferSize 
                << " bytes, ttl " << (unsigned)ttl() << "\n";
        }
        return True;
    } while (0);

    if (DebugLevel >= 0) {
        UsageEnvironment::MsgString msg = strDup(env.getResultMsg());
        env. setResultMsg("Groupsock write failed: ", msg);
        delete[] (char*)msg;
    }
    return False;
}
```

### 7.6 handleRead() - 接收数据

```cpp
Boolean Groupsock::handleRead(unsigned char* buffer, unsigned bufferMaxSize,
                              unsigned& bytesRead,
                              struct sockaddr_storage& fromAddressAndPort) {
    bytesRead = 0;

    // 从Socket读取数据
    int numBytes = readSocket(env(), socketNum(), buffer, bufferMaxSize, 
                              fromAddressAndPort);
    if (numBytes < 0) {
        if (DebugLevel >= 0) {
            UsageEnvironment::MsgString msg = strDup(env(). getResultMsg());
            env().setResultMsg("Groupsock read failed: ", msg);
            delete[] (char*)msg;
        }
        return False;
    }

    // SSM模式：检查源地址是否匹配
    if (isSSM() && !(fromAddressAndPort == sourceFilterAddress())) {
        return True;  // 源不匹配，忽略数据
    }

    bytesRead = numBytes;

    // 更新统计（排除自己发送的回环包）
    if (! wasLoopedBackFromUs(env(), fromAddressAndPort)) {
        statsIncoming.countPacket(bytesRead);
        statsGroupIncoming. countPacket(bytesRead);
    }
    
    if (DebugLevel >= 3) {
        env() << *this << ": read " << bytesRead << " bytes from " 
              << AddressString(fromAddressAndPort). val() << "\n";
    }

    return True;
}
```

### 7.7 addDestination() - 添加目标

```cpp
void Groupsock::addDestination(struct sockaddr_storage const& addr, 
                               Port const& port, unsigned sessionId) {
    // 检查是否已存在相同的目标
    for (destRecord* dest = fDests; dest != NULL; dest = dest->fNext) {
        if (dest->fSessionId == sessionId &&
            dest->fGroupEId.groupAddress() == addr &&
            dest->fGroupEId.portNum() == port.num()) {
            return;  // 已存在，不重复添加
        }
    }
    
    // 添加到链表头部
    fDests = createNewDestRecord(addr, port, 255, sessionId, fDests);
}
```

### 7.8 removeDestination() - 移除目标

```cpp
void Groupsock::removeDestination(unsigned sessionId) {
    removeDestinationFrom(fDests, sessionId);
}

void Groupsock::removeDestinationFrom(destRecord*& dests, unsigned sessionId) {
    destRecord** destsPtr = &dests;
    while (*destsPtr != NULL) {
        if (sessionId == (*destsPtr)->fSessionId) {
            // 找到要删除的节点
            destRecord* next = (*destsPtr)->fNext;
            (*destsPtr)->fNext = NULL;  // 防止递归删除
            delete (*destsPtr);
            *destsPtr = next;
        } else {
            destsPtr = &((*destsPtr)->fNext);
        }
    }
}
```

---

## 八、GroupsockHelper 辅助函数

### 8.1 Socket 创建

```cpp
// 文件：GroupsockHelper.hh

// 创建UDP Socket
int setupDatagramSocket(UsageEnvironment& env, Port port, int domain);

// 创建TCP Socket
int setupStreamSocket(UsageEnvironment& env, Port port, int domain,
                      Boolean makeNonBlocking = True, 
                      Boolean setKeepAlive = False);
```

### 8.2 数据读写

```cpp
// 读取数据
int readSocket(UsageEnvironment& env,
               int socket, unsigned char* buffer, unsigned bufferSize,
               struct sockaddr_storage& fromAddress);

// 发送数据（带TTL设置）
Boolean writeSocket(UsageEnvironment& env,
                    int socket, struct sockaddr_storage const& addressAndPort,
                    u_int8_t ttlArg,
                    unsigned char* buffer, unsigned bufferSize);

// 发送数据（不设置TTL，性能更好）
Boolean writeSocket(UsageEnvironment& env,
                    int socket, struct sockaddr_storage const& addressAndPort,
                    unsigned char* buffer, unsigned bufferSize);
```

### 8. 3 缓冲区管理

```cpp
unsigned getSendBufferSize(UsageEnvironment& env, int socket);
unsigned getReceiveBufferSize(UsageEnvironment& env, int socket);
unsigned setSendBufferTo(UsageEnvironment& env, int socket, unsigned requestedSize);
unsigned setReceiveBufferTo(UsageEnvironment& env, int socket, unsigned requestedSize);
unsigned increaseSendBufferTo(UsageEnvironment& env, int socket, unsigned requestedSize);
unsigned increaseReceiveBufferTo(UsageEnvironment& env, int socket, unsigned requestedSize);
```

### 8.4 Socket选项

```cpp
Boolean makeSocketNonBlocking(int sock);
Boolean makeSocketBlocking(int sock, unsigned writeTimeoutInMilliseconds = 0);
Boolean setSocketKeepAlive(int sock);
void ignoreSigPipeOnSocket(int socketNum);
```

### 8.5 组播操作

```cpp
// 加入组播组 (ASM)
Boolean socketJoinGroup(UsageEnvironment& env, int socket,
                        struct sockaddr_storage const& groupAddress);

// 离开组播组 (ASM)
Boolean socketLeaveGroup(UsageEnvironment&, int socket,
                         struct sockaddr_storage const& groupAddress);

// 加入组播组 (SSM)
Boolean socketJoinGroupSSM(UsageEnvironment& env, int socket,
                           struct sockaddr_storage const& groupAddress,
                           struct sockaddr_storage const& sourceFilterAddr);

// 离开组播组 (SSM)
Boolean socketLeaveGroupSSM(UsageEnvironment&, int socket,
                            struct sockaddr_storage const& groupAddress,
                            struct sockaddr_storage const& sourceFilterAddr);
```

### 8.6 地址查询

```cpp
// 获取本机IP地址
ipv4AddressBits ourIPv4Address(UsageEnvironment& env);
ipv6AddressBits const& ourIPv6Address(UsageEnvironment& env);

Boolean weHaveAnIPv4Address(UsageEnvironment& env);
Boolean weHaveAnIPv6Address(UsageEnvironment& env);
Boolean weHaveAnIPAddress(UsageEnvironment& env);

// 获取源端口
Boolean getSourcePort(UsageEnvironment& env, int socket, int domain, Port& port);
```

---

## 九、GroupsockLookupTable 类

用于管理和查找 Groupsock 对象：

```cpp
// 文件：Groupsock.hh

class GroupsockLookupTable {
public:
    // 获取或创建 Groupsock
    Groupsock* Fetch(UsageEnvironment& env, 
                     struct sockaddr_storage const& groupAddress,
                     Port port, u_int8_t ttl, Boolean& isNew);
    
    Groupsock* Fetch(UsageEnvironment& env, 
                     struct sockaddr_storage const& groupAddress,
                     struct sockaddr_storage const& sourceFilterAddr,
                     Port port, Boolean& isNew);

    // 查找（不创建）
    Groupsock* Lookup(struct sockaddr_storage const& groupAddress, Port port);
    Groupsock* Lookup(UsageEnvironment& env, int sock);

    // 移除
    Boolean Remove(Groupsock const* groupsock);

    // 迭代器
    class Iterator {
    public:
        Iterator(GroupsockLookupTable& groupsocks);
        Groupsock* next();
    private:
        AddressPortLookupTable::Iterator fIter;
    };

private:
    AddressPortLookupTable fTable;
};
```

---

## 十、使用示例

### 10.1 创建组播Groupsock

```cpp
void createMulticastGroupsock(UsageEnvironment& env) {
    // 设置组播地址
    struct sockaddr_in groupAddr;
    memset(&groupAddr, 0, sizeof(groupAddr));
    groupAddr.sin_family = AF_INET;
    inet_pton(AF_INET, "239.255.42.42", &groupAddr. sin_addr);
    
    Port port(5000);
    u_int8_t ttl = 7;
    
    // 创建Groupsock
    Groupsock* gs = new Groupsock(env, 
                                  *(struct sockaddr_storage*)&groupAddr,
                                  port, ttl);
    
    // 发送数据
    unsigned char data[] = "Hello, Multicast!";
    gs->output(env, data, sizeof(data));
    
    // 清理
    delete gs;
}
```

### 10.2 创建单播Groupsock并添加多个目标

```cpp
void createUnicastWithMultipleDestinations(UsageEnvironment& env) {
    // 创建一个Groupsock
    struct sockaddr_in localAddr;
    memset(&localAddr, 0, sizeof(localAddr));
    localAddr.sin_family = AF_INET;
    localAddr.sin_addr.s_addr = INADDR_ANY;
    
    Port port(5000);
    
    Groupsock* gs = new Groupsock(env, 
                                  *(struct sockaddr_storage*)&localAddr,
                                  port, 128);
    
    // 添加多个目标（实现多单播）
    struct sockaddr_in dest1, dest2;
    
    dest1.sin_family = AF_INET;
    inet_pton(AF_INET, "192.168.1. 10", &dest1. sin_addr);
    
    dest2.sin_family = AF_INET;
    inet_pton(AF_INET, "192. 168.1.20", &dest2. sin_addr);
    
    Port destPort(5000);
    
    gs->addDestination(*(struct sockaddr_storage*)&dest1, destPort, 1);
    gs->addDestination(*(struct sockaddr_storage*)&dest2, destPort, 2);
    
    // 发送数据（会发送到两个目标）
    unsigned char data[] = "Hello, all clients!";
    gs->output(env, data, sizeof(data));
    
    // 移除一个目标
    gs->removeDestination(1);
    
    delete gs;
}
```

### 10.3 接收组播数据

```cpp
void setupGroupsockForReceiving(UsageEnvironment& env, Groupsock* gs) {
    // 注册读取回调
    env.taskScheduler(). setBackgroundHandling(
        gs->socketNum(),
        SOCKET_READABLE,
        incomingDataHandler,
        gs
    );
}

void incomingDataHandler(void* clientData, int /*mask*/) {
    Groupsock* gs = (Groupsock*)clientData;
    
    unsigned char buffer[65536];
    unsigned bytesRead;
    struct sockaddr_storage fromAddr;
    
    if (gs->handleRead(buffer, sizeof(buffer), bytesRead, fromAddr)) {
        if (bytesRead > 0) {
            gs->env() << "Received " << bytesRead << " bytes from "
                      << AddressString(fromAddr).val() << "\n";
            // 处理数据... 
        }
    }
}
```

---

## 十一、流量统计

### 11.1 NetInterfaceTrafficStats 类

```cpp
// 文件：NetInterface.hh

class NetInterfaceTrafficStats {
public:
    NetInterfaceTrafficStats();

    void countPacket(unsigned packetSize);

    float totNumPackets() const { return fTotNumPackets; }
    float totNumBytes() const { return fTotNumBytes; }
    Boolean haveSeenTraffic() const;

private:
    float fTotNumPackets;
    float fTotNumBytes;
};
```

### 11.2 使用统计

```cpp
// 全局统计（所有Groupsock共享）
float totalPacketsIn = Groupsock::statsIncoming.totNumPackets();
float totalBytesIn = Groupsock::statsIncoming. totNumBytes();
float totalPacketsOut = Groupsock::statsOutgoing. totNumPackets();
float totalBytesOut = Groupsock::statsOutgoing.totNumBytes();

// 单个Groupsock的统计
float groupPacketsIn = gs->statsGroupIncoming.totNumPackets();
float groupBytesIn = gs->statsGroupIncoming.totNumBytes();

env << "Total: " << totalPacketsIn << " packets, " << totalBytesIn << " bytes\n";
```

---

## 十二、小结

### 12.1 要点

1. **类继承结构**：NetInterface → Socket → OutputSocket → Groupsock
2. **Socket** 封装了UDP Socket的基本操作
3. **OutputSocket** 专门用于发送，优化了TTL设置
4. **Groupsock** 是核心类，支持组播和多单播
5. **destRecord** 链表支持发送到多个目标
6. **GroupsockHelper** 提供了底层 Socket 操作函数

### 12.2 核心类速查表

| 类                     | 功能                  |
| ---------------------- | --------------------- |
| `NetInterface`         | 网络接口抽象基类      |
| `Socket`               | UDP Socket封装        |
| `OutputSocket`         | 仅发送的Socket        |
| `Groupsock`            | 完整的组播/单播Socket |
| `GroupEId`             | 组端点标识            |
| `destRecord`           | 目标地址记录          |
| `GroupsockLookupTable` | Groupsock查找表       |

### 12.3 关键函数速查表

| 函数                    | 功能            |
| ----------------------- | --------------- |
| `setupDatagramSocket()` | 创建UDP Socket  |
| `socketJoinGroup()`     | 加入组播组(ASM) |
| `socketJoinGroupSSM()`  | 加入组播组(SSM) |
| `socketLeaveGroup()`    | 离开组播组      |
| `readSocket()`          | 读取数据        |
| `writeSocket()`         | 发送数据        |
| `output()`              | 发送到所有目标  |
| `handleRead()`          | 接收数据        |
| `addDestination()`      | 添加发送目标    |
| `removeDestination()`   | 移除发送目标    |

### 12.4 类关系图

```
┌─────────────────────────────────────────────────────────────────┐
│                     Groupsock 模块类图                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   NetInterface                                                  │
│        △                                                        │
│        │                                                        │
│   Socket                           GroupEId                     │
│   ┌─────────────────┐             ┌─────────────────┐          │
│   │ - fSocketNum    │             │ - fGroupAddress │          │
│   │ - fPort         │             │ - fSourceFilter │          │
│   │ - fFamily       │             │ - fTTL          │          │
│   └────────┬────────┘             └─────────────────┘          │
│            │                                                    │
│   OutputSocket                     destRecord                   │
│   ┌─────────────────┐             ┌─────────────────┐          │
│   │ - fSourcePort   │             │ - fNext         │          │
│   │ - fLastSentTTL  │             │ - fGroupEId     │          │
│   │ + write()       │             │ - fSessionId    │          │
│   └────────┬────────┘             └─────────────────┘          │
│            │                              ▲                     │
│   Groupsock                               │ 包含               │
│   ┌─────────────────┐─────────────────────┘                    │
│   │ - fDests        │                                          │
│   │ - fIncomingGrp  │                                          │
│   │ + output()      │                                          │
│   │ + handleRead()  │                                          │
│   │ + addDest()     │                                          │
│   └─────────────────┘                                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

