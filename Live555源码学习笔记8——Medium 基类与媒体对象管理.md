# Live555源码学习笔记8——Medium 基类与媒体对象管理

## 一、liveMedia 模块概述

### 1.1 liveMedia 的作用

liveMedia 是 Live555 最核心、最庞大的模块，实现了所有媒体相关的功能：

```
┌─────────────────────────────────────────────────────────────────┐
│                    liveMedia 模块功能                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                    核心基类                              │   │
│   │  Medium         - 所有媒体对象的基类                    │   │
│   │  MediaSource    - 数据源基类                            │   │
│   │  MediaSink      - 数据消费者基类                        │   │
│   │  FramedSource   - 帧数据源基类                          │   │
│   │  FramedFilter   - 帧过滤器基类                          │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                    RTP/RTCP 协议                         │   │
│   │  RTPSource      - RTP数据接收                           │   │
│   │  RTPSink        - RTP数据发送                           │   │
│   │  RTCPInstance   - RTCP协议处理                          │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                    RTSP 协议                             │   │
│   │  RTSPServer     - RTSP服务端                            │   │
│   │  RTSPClient     - RTSP客户端                            │   │
│   │  ServerMediaSession - 服务端媒体会话                    │   │
│   │  MediaSession   - 客户端媒体会话                        │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                    媒体格式支持                          │   │
│   │  H.264/H.265 视频编解码                                 │   │
│   │  AAC/MP3/G. 711 音频编解码                               │   │
│   │  MPEG-TS/PS 容器格式                                    │   │
│   │  Matroska/Ogg 容器格式                                  │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 核心文件结构

```
liveMedia/
├── include/
│   ├── Media.hh              ⭐ Medium基类
│   ├── MediaSource. hh        ⭐ 数据源基类
│   ├── MediaSink.hh          ⭐ 数据消费者基类
│   ├── FramedSource.hh       帧数据源
│   ├── FramedFilter. hh       帧过滤器
│   ├── RTPSource.hh          RTP接收
│   ├── RTPSink. hh            RTP发送
│   ├── RTSPServer.hh         RTSP服务端
│   ├── RTSPClient.hh         RTSP客户端
│   └── ...  (100+个头文件)
└── *. cpp                     对应的实现文件
```

---

## 二、Medium 基类

### 2.1 类定义

Medium 是 liveMedia 中**所有媒体对象**的基类：

```cpp
// 文件：Media.hh

#define mediumNameMaxLen 30

class Medium {
public:
    // 静态方法：通过名称查找媒体对象
    static Boolean lookupByName(UsageEnvironment& env,
                                char const* mediumName,
                                Medium*& resultMedium);
    
    // 静态方法：关闭（删除）媒体对象
    static void close(UsageEnvironment& env, char const* mediumName);
    static void close(Medium* medium);  // 通过指针关闭
    
    // 获取使用环境
    UsageEnvironment& envir() const { return fEnviron; }
    
    // 获取媒体对象名称
    char const* name() const { return fMediumName; }

    // 类型判断（虚函数，子类可重写）
    virtual Boolean isSource() const;
    virtual Boolean isSink() const;
    virtual Boolean isRTCPInstance() const;
    virtual Boolean isRTSPClient() const;
    virtual Boolean isRTSPServer() const;
    virtual Boolean isMediaSession() const;
    virtual Boolean isServerMediaSession() const;

protected:
    friend class MediaLookupTable;
    Medium(UsageEnvironment& env);  // 抽象基类
    virtual ~Medium();               // 通过close()删除，不能直接delete

    // 获取下一个任务的Token（用于异步操作）
    TaskToken& nextTask() { return fNextTask; }

private:
    UsageEnvironment& fEnviron;          // 使用环境引用
    char fMediumName[mediumNameMaxLen];  // 媒体对象名称
    TaskToken fNextTask;                  // 下一个待执行任务
};
```

### 2.2 设计要点

```
┌─────────────────────────────────────────────────────────────────┐
│                    Medium 设计要点                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   1. 【抽象基类】                                                │
│      - 构造函数是 protected                                     │
│      - 不能直接实例化 Medium                                    │
│      - 必须通过子类创建对象                                     │
│                                                                 │
│   2.  【名称管理】                                                │
│      - 每个 Medium 对象有唯一的名称                             │
│      - 名称由 MediaLookupTable 自动生成                         │
│      - 可以通过名称查找对象                                     │
│                                                                 │
│   3. 【生命周期管理】                                            │
│      - 析构函数是 protected                                     │
│      - 不能直接 delete，必须使用 Medium::close()                │
│      - close() 会从查找表中移除并删除对象                       │
│                                                                 │
│   4. 【类型系统】                                                │
│      - 提供 isXxx() 虚函数判断类型                              │
│      - 避免使用 dynamic_cast                                    │
│      - 性能更好，不依赖 RTTI                                    │
│                                                                 │
│   5. 【异步任务】                                                │
│      - fNextTask 存储下一个待执行任务                           │
│      - 用于调度延时操作                                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 2.3 类继承体系

```
┌─────────────────────────────────────────────────────────────────┐
│                    Medium 继承体系                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                          Medium                                 │
│                            │                                    │
│         ┌──────────────────┼──────────────────┐                 │
│         │                  │                  │                 │
│         ▼                  ▼                  ▼                 │
│    MediaSource        MediaSink         RTSPClient             │
│         │                  │             RTSPServer             │
│         ▼                  ▼             MediaSession           │
│    FramedSource       RTPSink           ServerMediaSession      │
│         │             FileSink          RTCPInstance            │
│    ┌────┼────┐                                                  │
│    │    │    │                                                  │
│    ▼    ▼    ▼                                                  │
│  RTP   File  Framed                                             │
│ Source Source Filter                                            │
│                  │                                              │
│            ┌─────┴─────┐                                        │
│            │           │                                        │
│            ▼           ▼                                        │
│        H264Video   MPEG1or2                                     │
│        StreamFramer VideoFramer                                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 三、MediaLookupTable 媒体查找表

### 3.1 类定义

MediaLookupTable 管理所有 Medium 对象，提供名称查找功能：

```cpp
// 文件：Media.hh

class MediaLookupTable {
public:
    // 获取当前环境的查找表（单例模式）
    static MediaLookupTable* ourMedia(UsageEnvironment& env);
    
    // 获取内部哈希表（用于遍历）
    HashTable const& getTable() { return *fTable; }

protected:
    MediaLookupTable(UsageEnvironment& env);
    virtual ~MediaLookupTable();

private:
    friend class Medium;

    // 通过名称查找媒体对象
    Medium* lookup(char const* name) const;
    
    // 添加新的媒体对象
    void addNew(Medium* medium, char* mediumName);
    
    // 移除媒体对象
    void remove(char const* name);
    
    // 生成新的唯一名称
    void generateNewName(char* mediumName, unsigned maxLen);

private:
    UsageEnvironment& fEnv;
    HashTable* fTable;       // 哈希表存储 name -> Medium*
    unsigned fNameGenerator; // 名称生成计数器
};
```

### 3.2 名称生成机制

```
┌─────────────────────────────────────────────────────────────────┐
│                    名称生成机制                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   MediaLookupTable::generateNewName()                           │
│                                                                 │
│   生成格式：liveMedia + 递增数字                                │
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                                                         │   │
│   │   第1个对象: "liveMedia0"                               │   │
│   │   第2个对象: "liveMedia1"                               │   │
│   │   第3个对象: "liveMedia2"                               │   │
│   │   ...                                                   │   │
│   │   第N个对象: "liveMediaN-1"                             │   │
│   │                                                         │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│   实现代码：                                                    │
│   sprintf(mediumName, "liveMedia%d", fNameGenerator++);         │
│                                                                 │
│   用途：                                                        │
│   - 确保每个对象有唯一标识                                      │
│   - 支持通过名称查找和关闭对象                                  │
│   - 调试时便于识别对象                                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 3.3 查找表结构

```
┌─────────────────────────────────────────────────────────────────┐
│                    MediaLookupTable 结构                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   UsageEnvironment                                              │
│        │                                                        │
│        │ liveMediaPriv                                          │
│        ▼                                                        │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                    _Tables                              │   │
│   │                                                         │   │
│   │   mediaTable ─────► MediaLookupTable                    │   │
│   │                          │                              │   │
│   │                          ▼                              │   │
│   │                    ┌──────────────────────────────────┐ │   │
│   │                    │         HashTable                │ │   │
│   │                    │                                  │ │   │
│   │                    │  "liveMedia0" → RTSPServer*      │ │   │
│   │                    │  "liveMedia1" → RTPSink*         │ │   │
│   │                    │  "liveMedia2" → H264Source*      │ │   │
│   │                    │  "liveMedia3" → RTCPInstance*    │ │   │
│   │                    │  ...                              │ │   │
│   │                    │                                  │ │   │
│   │                    └──────────────────────────────────┘ │   │
│   │                                                         │   │
│   │   socketTable ────► groupsock的Socket表                 │   │
│   │                                                         │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 四、_Tables 类

### 4.1 类定义

_Tables 是存储在 UsageEnvironment 中的全局表结构：

```cpp
// 文件：Media.hh

class _Tables {
public:
    // 获取或创建 _Tables 实例
    static _Tables* getOurTables(UsageEnvironment& env, 
                                  Boolean createIfNotPresent = True);
    
    // 尝试回收（当不再使用时删除自己）
    void reclaimIfPossible();

    MediaLookupTable* mediaTable;  // 媒体对象查找表
    void* socketTable;              // Socket查找表（给groupsock用）

protected:
    _Tables(UsageEnvironment& env);
    virtual ~_Tables();

private:
    UsageEnvironment& fEnv;
};
```

### 4.2 与 UsageEnvironment 的关系

```cpp
// _Tables 存储在 UsageEnvironment 的 liveMediaPriv 字段中

_Tables* _Tables::getOurTables(UsageEnvironment& env, 
                                Boolean createIfNotPresent) {
    // 从 env 的私有数据获取
    if (env.liveMediaPriv == NULL && createIfNotPresent) {
        env.liveMediaPriv = new _Tables(env);
    }
    return (_Tables*)(env.liveMediaPriv);
}
```

---

## 五、Medium 的创建与销毁

### 5.1 创建流程

```cpp
// Medium 构造函数
Medium::Medium(UsageEnvironment& env)
    : fEnviron(env), fNextTask(NULL) {
    
    // 1. 获取或创建查找表
    MediaLookupTable* table = MediaLookupTable::ourMedia(env);
    
    // 2. 生成唯一名称
    table->generateNewName(fMediumName, mediumNameMaxLen);
    
    // 3. 将自己添加到查找表
    table->addNew(this, fMediumName);
}
```

**创建流程图：**

```
┌─────────────────────────────────────────────────────────────────┐
│                    Medium 创建流程                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   用户代码                                                       │
│      │                                                          │
│      │  new RTSPServer::createNew(env, ...)                     │
│      ▼                                                          │
│   RTSPServer 构造函数                                           │
│      │                                                          │
│      │  调用父类构造函数                                        │
│      ▼                                                          │
│   Medium 构造函数                                               │
│      │                                                          │
│      ├──► 获取 MediaLookupTable                                 │
│      │         │                                                │
│      │         ▼                                                │
│      │    MediaLookupTable::ourMedia(env)                       │
│      │         │                                                │
│      │         ▼                                                │
│      │    如果不存在则创建 _Tables                              │
│      │                                                          │
│      ├──► 生成唯一名称 "liveMediaN"                             │
│      │                                                          │
│      └──► 添加到哈希表                                          │
│                table->addNew(this, fMediumName)                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 销毁流程

```cpp
// 通过名称关闭
void Medium::close(UsageEnvironment& env, char const* name) {
    MediaLookupTable* table = MediaLookupTable::ourMedia(env);
    Medium* medium = table->lookup(name);
    
    if (medium != NULL) {
        table->remove(name);  // 从表中移除
        delete medium;        // 调用析构函数
    }
}

// 通过指针关闭
void Medium::close(Medium* medium) {
    if (medium == NULL) return;
    close(medium->envir(), medium->name());
}
```

**销毁流程图：**

```
┌─────────────────────────────────────────────────────────────────┐
│                    Medium 销毁流程                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   用户代码                                                       │
│      │                                                          │
│      │  Medium::close(rtspServer)                               │
│      ▼                                                          │
│   Medium::close(Medium* medium)                                 │
│      │                                                          │
│      │  获取名称和环境                                          │
│      ▼                                                          │
│   Medium::close(env, name)                                      │
│      │                                                          │
│      ├──► 从 MediaLookupTable 查找对象                          │
│      │                                                          │
│      ├──► 从哈希表中移除                                        │
│      │         table->remove(name)                              │
│      │                                                          │
│      └──► 删除对象                                              │
│               delete medium                                     │
│                    │                                            │
│                    ▼                                            │
│              调用 ~Medium()                                     │
│                    │                                            │
│                    ▼                                            │
│              调用子类析构函数                                   │
│                                                                 │
│   ⚠️ 注意：不要直接 delete Medium 对象！                        │
│            必须使用 Medium::close()                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 5.3 通过名称查找

```cpp
Boolean Medium::lookupByName(UsageEnvironment& env,
                             char const* mediumName,
                             Medium*& resultMedium) {
    resultMedium = MediaLookupTable::ourMedia(env)->lookup(mediumName);
    
    if (resultMedium == NULL) {
        env.setResultMsg("Medium not found: ", mediumName);
        return False;
    }
    
    return True;
}
```

---

## 六、MediaSource 基类

### 6.1 类定义

MediaSource 是所有**数据源**的基类：

```cpp
// 文件：MediaSource.hh

class MediaSource: public Medium {
public:
    // 通过名称查找
    static Boolean lookupByName(UsageEnvironment& env, 
                                char const* sourceName,
                                MediaSource*& resultSource);
    
    // 获取属性（返回到 env 的结果消息中）
    virtual void getAttributes() const;
    
    // 获取 MIME 类型
    virtual char const* MIMEtype() const;

    // 类型判断（子类可重写）
    virtual Boolean isFramedSource() const;
    virtual Boolean isRTPSource() const;
    virtual Boolean isMPEG1or2VideoStreamFramer() const;
    virtual Boolean isMPEG4VideoStreamFramer() const;
    virtual Boolean isH264VideoStreamFramer() const;
    virtual Boolean isH265VideoStreamFramer() const;
    virtual Boolean isDVVideoStreamFramer() const;
    virtual Boolean isJPEGVideoSource() const;
    virtual Boolean isAMRAudioSource() const;
    virtual Boolean isMPEG2TransportStreamMultiplexor() const;

protected:
    MediaSource(UsageEnvironment& env);  // 抽象基类
    virtual ~MediaSource();

private:
    // 重写 Medium 的虚函数
    virtual Boolean isSource() const;  // 返回 True
};
```

### 6.2 MediaSource 继承体系

```
┌─────────────────────────────────────────────────────────────────┐
│                    MediaSource 继承体系                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                       MediaSource                               │
│                            │                                    │
│              ┌─────────────┴─────────────┐                      │
│              │                           │                      │
│              ▼                           ▼                      │
│        FramedSource                  其他特殊Source             │
│              │                                                  │
│    ┌─────────┼─────────┬─────────────┐                          │
│    │         │         │             │                          │
│    ▼         ▼         ▼             ▼                          │
│ RTPSource  File     Framed        ByteStream                    │
│            Source   Filter        FileSource                    │
│    │                   │                                        │
│    │         ┌─────────┼─────────┐                              │
│    │         │         │         │                              │
│    ▼         ▼         ▼         ▼                              │
│ MultiFramed H264Video MPEG4Video MPEG1or2                       │
│ RTPSource  StreamFramer StreamFramer VideoFramer                │
│                                                                 │
│   ⭐ FramedSource 是最重要的中间类                              │
│      定义了获取帧数据的接口                                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 七、MediaSink 基类

### 7. 1 类定义

MediaSink 是所有**数据消费者**的基类：

```cpp
// 文件：MediaSink. hh

class MediaSink: public Medium {
public:
    // 通过名称查找
    static Boolean lookupByName(UsageEnvironment& env, 
                                char const* sinkName,
                                MediaSink*& resultSink);

    // 播放完成回调类型
    typedef void (afterPlayingFunc)(void* clientData);
    
    // 开始播放（从 source 获取数据）
    Boolean startPlaying(MediaSource& source,
                         afterPlayingFunc* afterFunc,
                         void* afterClientData);
    
    // 停止播放
    virtual void stopPlaying();

    // 类型判断
    virtual Boolean isRTPSink() const;

    // 获取数据源
    FramedSource* source() const { return fSource; }

protected:
    MediaSink(UsageEnvironment& env);  // 抽象基类
    virtual ~MediaSink();

    // 检查源是否兼容（startPlaying调用）
    virtual Boolean sourceIsCompatibleWithUs(MediaSource& source);
    
    // 继续播放（纯虚函数，子类必须实现）
    virtual Boolean continuePlaying() = 0;

    // 源关闭回调
    static void onSourceClosure(void* clientData);
    void onSourceClosure();

    FramedSource* fSource;  // 数据源指针

private:
    virtual Boolean isSink() const;  // 返回 True

private:
    afterPlayingFunc* fAfterFunc;     // 播放完成回调
    void* fAfterClientData;           // 回调用户数据
};
```

### 7.2 Source-Sink 数据流模型

```
┌─────────────────────────────────────────────────────────────────┐
│                    Source-Sink 数据流模型                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   【概念说明】                                                   │
│                                                                 │
│   Source（源）：产生数据的对象                                  │
│   Sink（汇）：消费数据的对象                                    │
│                                                                 │
│   数据从 Source 流向 Sink                                       │
│                                                                 │
│   ─────────────────────────────────────────────────────────────│
│                                                                 │
│   【典型的数据流】                                               │
│                                                                 │
│   ┌──────────────┐      ┌──────────────┐      ┌──────────────┐ │
│   │ FileSource   │ ───► │ VideoFramer  │ ───► │ RTPSink      │ │
│   │ (读取文件)   │      │ (帧解析)     │      │ (RTP发送)    │ │
│   └──────────────┘      └──────────────┘      └──────────────┘ │
│                                                                 │
│   ByteStreamFileSource → H264VideoStreamFramer → H264RTPSink   │
│                                                                 │
│   ─────────────────────────────────────────────────────────────│
│                                                                 │
│   【工作原理】                                                   │
│                                                                 │
│   1.  Sink 调用 startPlaying(source, ...)                        │
│                                                                 │
│   2.  Sink 调用 source->getNextFrame(buffer, callback)           │
│      "我要一帧数据，放到buffer，完成后调用callback"             │
│                                                                 │
│   3. Source 产生数据后，调用 callback                           │
│      "数据已经准备好了"                                         │
│                                                                 │
│   4. Sink 在 callback 中处理数据                                │
│                                                                 │
│   5. Sink 再次调用 getNextFrame() 请求下一帧                    │
│                                                                 │
│   6. 重复步骤 3-5 直到结束                                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 7.3 startPlaying() 流程

```cpp
Boolean MediaSink::startPlaying(MediaSource& source,
                                afterPlayingFunc* afterFunc,
                                void* afterClientData) {
    // 1. 检查是否已经在播放
    if (fSource != NULL) {
        envir().setResultMsg("This sink is already playing");
        return False;
    }

    // 2.  检查源类型是否兼容
    if (! sourceIsCompatibleWithUs(source)) {
        envir(). setResultMsg("MediaSink::startPlaying(): source is not compatible!");
        return False;
    }
    
    // 3. 将源转换为 FramedSource
    fSource = (FramedSource*)&source;
    
    // 4.  保存回调
    fAfterFunc = afterFunc;
    fAfterClientData = afterClientData;
    
    // 5. 开始播放（调用子类实现）
    return continuePlaying();
}
```

---

## 八、OutPacketBuffer 类

### 8.1 类定义

OutPacketBuffer 是 Sink 用于构建输出包的缓冲区：

```cpp
// 文件：MediaSink.hh

class OutPacketBuffer {
public:
    OutPacketBuffer(unsigned preferredPacketSize, 
                    unsigned maxPacketSize,
                    unsigned maxBufferSize = 0);
    ~OutPacketBuffer();

    static unsigned maxSize;  // 全局最大尺寸
    static void increaseMaxSizeTo(unsigned newMaxSize);

    // 当前位置和可用空间
    unsigned char* curPtr() const;
    unsigned totalBytesAvailable() const;
    unsigned totalBufferSize() const;
    
    // 包操作
    unsigned char* packet() const;
    unsigned curPacketSize() const;

    // 数据入队
    void increment(unsigned numBytes);
    void enqueue(unsigned char const* from, unsigned numBytes);
    void enqueueWord(u_int32_t word);
    
    // 数据插入（到指定位置）
    void insert(unsigned char const* from, unsigned numBytes, unsigned toPosition);
    void insertWord(u_int32_t word, unsigned toPosition);
    
    // 数据提取
    void extract(unsigned char* to, unsigned numBytes, unsigned fromPosition);
    u_int32_t extractWord(unsigned fromPosition);

    // 包大小检查
    Boolean isPreferredSize() const;
    Boolean wouldOverflow(unsigned numBytes) const;
    unsigned numOverflowBytes(unsigned numBytes) const;
    Boolean isTooBigForAPacket(unsigned numBytes) const;

    // 溢出数据管理
    void setOverflowData(unsigned overflowDataOffset,
                         unsigned overflowDataSize,
                         struct timeval const& presentationTime,
                         unsigned durationInMicroseconds);
    unsigned overflowDataSize() const;
    Boolean haveOverflowData() const;
    void useOverflowData();

    // 重置
    void resetPacketStart();
    void resetOffset();
    void resetOverflowData();

private:
    unsigned fPacketStart, fCurOffset, fPreferred, fMax, fLimit;
    unsigned char* fBuf;

    // 溢出数据信息
    unsigned fOverflowDataOffset, fOverflowDataSize;
    struct timeval fOverflowPresentationTime;
    unsigned fOverflowDurationInMicroseconds;
};
```

### 8.2 缓冲区结构

```
┌─────────────────────────────────────────────────────────────────┐
│                    OutPacketBuffer 结构                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   fBuf (缓冲区)                                                 │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                                                         │   │
│   │    fPacketStart           fCurOffset                    │   │
│   │         │                      │                        │   │
│   │         ▼                      ▼                        │   │
│   │   ┌─────┬──────────────────────┬────────────────────┐   │   │
│   │   │ ...  │ 已写入数据           │ 可用空间          │   │   │
│   │   └─────┴──────────────────────┴────────────────────┘   │   │
│   │   0                                               fLimit│   │
│   │                                                         │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│   【各字段含义】                                                 │
│                                                                 │
│   fPacketStart: 当前包的起始位置                                │
│   fCurOffset:   当前写入位置（相对于fPacketStart）              │
│   fPreferred:   首选包大小（达到这个大小就可以发送）            │
│   fMax:         单个包的最大大小                                │
│   fLimit:       缓冲区总大小                                    │
│                                                                 │
│   【溢出数据】                                                   │
│                                                                 │
│   当一帧数据太大，放不进一个包时：                              │
│   - 先发送一部分                                                │
│   - 剩余部分存为 "溢出数据"                                     │
│   - 下次发送时先处理溢出数据                                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 九、使用示例

### 9.1 创建和关闭 Medium 对象

```cpp
// 创建（通过子类的 createNew 方法）
RTSPServer* rtspServer = RTSPServer::createNew(env, 8554);

// 检查创建是否成功
if (rtspServer == NULL) {
    env << "Failed to create RTSP server: " << env.getResultMsg() << "\n";
    return;
}

// 获取名称
env << "Server name: " << rtspServer->name() << "\n";

// 通过名称查找
Medium* medium;
if (Medium::lookupByName(env, rtspServer->name(), medium)) {
    env << "Found medium: " << medium->name() << "\n";
}

// 关闭（通过 Medium::close）
Medium::close(rtspServer);  // ✓ 正确方式
// delete rtspServer;       // ✗ 错误！不要直接delete
```

### 9.2 类型判断

```cpp
Medium* medium = ... ;

// 判断类型
if (medium->isSource()) {
    MediaSource* source = (MediaSource*)medium;
    env << "MIME type: " << source->MIMEtype() << "\n";
    
    // 进一步判断
    if (source->isRTPSource()) {
        RTPSource* rtpSource = (RTPSource*)source;
        // ...
    }
}

if (medium->isSink()) {
    MediaSink* sink = (MediaSink*)medium;
    // ...
}

if (medium->isRTSPServer()) {
    RTSPServer* server = (RTSPServer*)medium;
    // ...
}
```

### 9.3 Source-Sink 数据流

```cpp
// 创建源
ByteStreamFileSource* fileSource = 
    ByteStreamFileSource::createNew(env, "video.264");

// 创建帧解析器（作为过滤器连接到源）
H264VideoStreamFramer* framer = 
    H264VideoStreamFramer::createNew(env, fileSource);

// 创建 RTP Sink
RTPSink* rtpSink = H264VideoRTPSink::createNew(env, rtpGroupsock, 96);

// 开始播放（数据从 framer 流向 rtpSink）
rtpSink->startPlaying(*framer, afterPlaying, NULL);

// 播放完成回调
void afterPlaying(void* clientData) {
    env << "Playback completed\n";
    
    // 停止播放
    rtpSink->stopPlaying();
    
    // 清理资源
    Medium::close(rtpSink);
    Medium::close(framer);
    Medium::close(fileSource);
}
```

---

## 十、小结

### 10.1 本节要点

1. **Medium** 是 liveMedia 所有媒体对象的基类
2. **MediaLookupTable** 管理所有 Medium 对象，通过名称查找
3. 创建对象使用子类的 `createNew()`，删除使用 `Medium::close()`
4. **MediaSource** 是数据生产者，**MediaSink** 是数据消费者
5. **Source-Sink 模型**：数据从 Source 流向 Sink
6. **OutPacketBuffer** 用于 Sink 构建输出数据包

### 10.2 核心类速查表

| 类                 | 功能                               |
| ------------------ | ---------------------------------- |
| `Medium`           | 所有媒体对象的基类                 |
| `MediaLookupTable` | 媒体对象查找表                     |
| `_Tables`          | 存储在 UsageEnvironment 中的全局表 |
| `MediaSource`      | 数据源基类                         |
| `MediaSink`        | 数据消费者基类                     |
| `OutPacketBuffer`  | 输出包缓冲区                       |

### 10.3 关键函数速查表

| 函数                           | 功能                 |
| ------------------------------ | -------------------- |
| `Medium::lookupByName()`       | 通过名称查找媒体对象 |
| `Medium::close()`              | 关闭（删除）媒体对象 |
| `Medium::isSource()`           | 判断是否为数据源     |
| `Medium::isSink()`             | 判断是否为数据消费者 |
| `MediaSink::startPlaying()`    | 开始播放             |
| `MediaSink::stopPlaying()`     | 停止播放             |
| `MediaSink::continuePlaying()` | 继续播放（子类实现） |

### 10.4 类关系图

```
┌─────────────────────────────────────────────────────────────────┐
│                    核心类关系图                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   UsageEnvironment                                              │
│        │                                                        │
│        │ liveMediaPriv                                          │
│        ▼                                                        │
│   _Tables ──────► MediaLookupTable ──────► HashTable            │
│                          │                     │                │
│                          │                     ▼                │
│                          │              name → Medium*          │
│                          │                                      │
│                          ▼                                      │
│                       Medium                                    │
│                          │                                      │
│            ┌─────────────┼─────────────┐                        │
│            │             │             │                        │
│            ▼             ▼             ▼                        │
│       MediaSource   MediaSink    RTSPServer                     │
│            │             │        RTSPClient                    │
│            ▼             ▼        MediaSession                  │
│      FramedSource    RTPSink     RTCPInstance                   │
│                      FileSink                                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

