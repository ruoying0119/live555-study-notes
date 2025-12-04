# Live555源码学习笔记9——FramedSource 与数据获取机制

## 一、FramedSource 概述

### 1.1 FramedSource 的作用

FramedSource 是 Live555 中**最重要的数据源基类**，它定义了获取帧数据的标准接口。所有需要产生帧数据的 Source 都继承自它。

```
┌─────────────────────────────────────────────────────────────────┐
│                    FramedSource 在系统中的位置                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   MediaSink (如 RTPSink)                                        │
│        │                                                        │
│        │ 调用 source->getNextFrame()                            │
│        ▼                                                        │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                    FramedSource                         │   │
│   │                                                         │   │
│   │  • 定义了 getNextFrame() 接口                           │   │
│   │  • 管理异步数据获取流程                                 │   │
│   │  • 提供帧信息（大小、时间戳、时长）                     │   │
│   │                                                         │   │
│   └─────────────────────────────────────────────────────────┘   │
│        △                                                        │
│        │ 继承                                                   │
│        │                                                        │
│   ┌────┴────────────────────────────────────────────────────┐   │
│   │                                                         │   │
│   │  RTPSource          - 从网络接收RTP数据                 │   │
│   │  ByteStreamFileSource - 从文件读取字节流                │   │
│   │  FramedFilter       - 过滤器基类（处理上游数据）        │   │
│   │    ├─ H264VideoStreamFramer - H. 264帧解析               │   │
│   │    ├─ MPEG4VideoStreamFramer - MPEG4帧解析              │   │
│   │    └─ ...                                                │   │
│   │                                                         │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 核心概念："帧"

在 Live555 中，"帧"是一个通用概念：

```
┌─────────────────────────────────────────────────────────────────┐
│                    "帧" 的含义                                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   【视频帧】                                                     │
│   • H.264/H.265: 一个 NAL 单元                                  │
│   • MPEG-4: 一个 VOP (Video Object Plane)                       │
│   • MJPEG: 一张 JPEG 图片                                       │
│                                                                 │
│   【音频帧】                                                     │
│   • AAC: 一个 ADTS 帧                                           │
│   • MP3: 一个 MP3 帧                                            │
│   • PCM: 一段采样数据                                           │
│                                                                 │
│   【其他】                                                       │
│   • 字节流：一块任意大小的数据                                  │
│   • RTP: 一个 RTP 包的有效载荷                                  │
│                                                                 │
│   💡 FramedSource 不关心具体含义，只负责提供数据块              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 二、FramedSource 类定义

### 2.1 完整类定义

```cpp
// 文件：FramedSource.hh

class FramedSource: public MediaSource {
public:
    // 通过名称查找
    static Boolean lookupByName(UsageEnvironment& env, 
                                char const* sourceName,
                                FramedSource*& resultSource);

    // 回调函数类型定义
    typedef void (afterGettingFunc)(void* clientData, 
                                    unsigned frameSize,
                                    unsigned numTruncatedBytes,
                                    struct timeval presentationTime,
                                    unsigned durationInMicroseconds);
    typedef void (onCloseFunc)(void* clientData);
    
    // ⭐ 核心方法：请求获取下一帧数据
    void getNextFrame(unsigned char* to,           // 数据存放位置
                      unsigned maxSize,             // 缓冲区最大大小
                      afterGettingFunc* afterGettingFunc,  // 获取成功回调
                      void* afterGettingClientData,
                      onCloseFunc* onCloseFunc,     // 源关闭回调
                      void* onCloseClientData);

    // 处理源关闭
    static void handleClosure(void* clientData);
    void handleClosure();

    // 停止获取帧
    void stopGettingFrames();

    // 获取最大帧大小（0表示未知）
    virtual unsigned maxFrameSize() const;

    // ⭐ 纯虚函数：子类必须实现，实际获取数据的逻辑
    virtual void doGetNextFrame() = 0;

    // 是否正在等待数据
    Boolean isCurrentlyAwaitingData() const { return fIsCurrentlyAwaitingData; }

    // 数据获取完成后调用
    static void afterGetting(FramedSource* source);

protected:
    FramedSource(UsageEnvironment& env);  // 抽象基类
    virtual ~FramedSource();

    // 子类可重写：停止获取帧时的清理逻辑
    virtual void doStopGettingFrames();

protected:
    // ⭐ 这些变量由 doGetNextFrame() 使用
    // 输入参数（调用者设置）
    unsigned char* fTo;      // 数据存放位置
    unsigned fMaxSize;       // 缓冲区最大大小
    
    // 输出参数（子类设置）
    unsigned fFrameSize;     // 实际帧大小
    unsigned fNumTruncatedBytes;  // 被截断的字节数
    struct timeval fPresentationTime;  // 显示时间戳
    unsigned fDurationInMicroseconds;  // 帧持续时间

private:
    virtual Boolean isFramedSource() const;  // 返回 True

private:
    afterGettingFunc* fAfterGettingFunc;
    void* fAfterGettingClientData;
    onCloseFunc* fOnCloseFunc;
    void* fOnCloseClientData;

    Boolean fIsCurrentlyAwaitingData;  // 是否正在等待数据
};
```

### 2.2 关键成员变量说明

```
┌─────────────────────────────────────────────────────────────────┐
│                    FramedSource 成员变量                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   【输入参数 - 由调用者(Sink)设置】                              │
│                                                                 │
│   fTo        ──► 数据存放的缓冲区地址                           │
│   fMaxSize   ──► 缓冲区的最大容量                               │
│                                                                 │
│   【输出参数 - 由 doGetNextFrame() 设置】                        │
│                                                                 │
│   fFrameSize ──► 实际读取/生成的帧大小                          │
│                  (必须 <= fMaxSize)                             │
│                                                                 │
│   fNumTruncatedBytes ──► 如果帧太大被截断，记录丢失的字节数     │
│                                                                 │
│   fPresentationTime ──► 帧的显示时间戳 (PTS)                    │
│                         通常是 gettimeofday() 或计算得出        │
│                                                                 │
│   fDurationInMicroseconds ──► 帧的持续时间（微秒）              │
│                               用于计算下一帧的时间              │
│                                                                 │
│   【状态变量】                                                   │
│                                                                 │
│   fIsCurrentlyAwaitingData ──► True = 正在等待数据              │
│                                False = 空闲                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 三、getNextFrame() 工作机制

### 3.1 调用流程

```cpp
void FramedSource::getNextFrame(unsigned char* to, unsigned maxSize,
                                afterGettingFunc* afterGettingFunc,
                                void* afterGettingClientData,
                                onCloseFunc* onCloseFunc,
                                void* onCloseClientData) {
    // 1. 检查是否已经在等待数据（防止重入）
    if (fIsCurrentlyAwaitingData) {
        envir() << "FramedSource::getNextFrame(): attempting to read more than once!\n";
        envir(). internalError();
    }

    // 2.  保存输入参数
    fTo = to;
    fMaxSize = maxSize;
    
    // 3.  保存回调函数
    fAfterGettingFunc = afterGettingFunc;
    fAfterGettingClientData = afterGettingClientData;
    fOnCloseFunc = onCloseFunc;
    fOnCloseClientData = onCloseClientData;
    
    // 4. 标记为正在等待数据
    fIsCurrentlyAwaitingData = True;

    // 5. 调用子类实现的 doGetNextFrame()
    doGetNextFrame();
}
```

### 3.2 完整的数据获取流程图

```
┌─────────────────────────────────────────────────────────────────┐
│                    getNextFrame() 完整流程                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   MediaSink (如 RTPSink)                                        │
│        │                                                        │
│        │ ① 调用 getNextFrame(buffer, size, callback, ...)       │
│        │    "给我一帧数据，放到buffer，完成后调callback"        │
│        ▼                                                        │
│   FramedSource::getNextFrame()                                  │
│        │                                                        │
│        │ ② 保存参数和回调                                       │
│        │    fTo = buffer                                        │
│        │    fMaxSize = size                                     │
│        │    fAfterGettingFunc = callback                        │
│        │                                                        │
│        │ ③ 标记 fIsCurrentlyAwaitingData = True                 │
│        │                                                        │
│        │ ④ 调用 doGetNextFrame()                                │
│        ▼                                                        │
│   子类::doGetNextFrame()                                        │
│        │                                                        │
│        │ ⑤ 获取数据（可能是同步或异步）                         │
│        │    - 从文件读取                                        │
│        │    - 从网络接收                                        │
│        │    - 从上游Source获取并处理                            │
│        │                                                        │
│        │ ⑥ 填充输出参数                                         │
│        │    fFrameSize = 实际大小                               │
│        │    fPresentationTime = 时间戳                          │
│        │    fDurationInMicroseconds = 持续时间                  │
│        │                                                        │
│        │ ⑦ 调用 afterGetting(this)                              │
│        ▼                                                        │
│   FramedSource::afterGetting()                                  │
│        │                                                        │
│        │ ⑧ 标记 fIsCurrentlyAwaitingData = False                │
│        │                                                        │
│        │ ⑨ 调用回调函数                                         │
│        │    (*fAfterGettingFunc)(fAfterGettingClientData,       │
│        │                         fFrameSize,                    │
│        │                         fNumTruncatedBytes,            │
│        │                         fPresentationTime,             │
│        │                         fDurationInMicroseconds)       │
│        ▼                                                        │
│   回到 Sink 的回调函数                                          │
│        │                                                        │
│        │ ⑩ 处理数据（如发送RTP包）                              │
│        │                                                        │
│        │ ⑪ 再次调用 getNextFrame() 请求下一帧                   │
│        ▼                                                        │
│   循环重复...                                                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 3.3 afterGetting() 实现

```cpp
void FramedSource::afterGetting(FramedSource* source) {
    // 重置等待标志
    source->fIsCurrentlyAwaitingData = False;
    
    // 调用回调函数，通知调用者数据已就绪
    if (source->fAfterGettingFunc != NULL) {
        (*(source->fAfterGettingFunc))(
            source->fAfterGettingClientData,
            source->fFrameSize,
            source->fNumTruncatedBytes,
            source->fPresentationTime,
            source->fDurationInMicroseconds
        );
    }
}
```

### 3. 4 handleClosure() 处理源关闭

```cpp
void FramedSource::handleClosure(void* clientData) {
    FramedSource* source = (FramedSource*)clientData;
    source->handleClosure();
}

void FramedSource::handleClosure() {
    // 重置等待标志
    fIsCurrentlyAwaitingData = False;
    
    // 调用关闭回调
    if (fOnCloseFunc != NULL) {
        (*fOnCloseFunc)(fOnCloseClientData);
    }
}
```

---

## 四、ByteStreamFileSource 示例

### 4.1 类定义

ByteStreamFileSource 是从文件读取字节流的 Source：

```cpp
// 文件：ByteStreamFileSource. hh

class ByteStreamFileSource: public FramedFileSource {
public:
    // 创建方法
    static ByteStreamFileSource* createNew(UsageEnvironment& env,
                                           char const* fileName,
                                           unsigned preferredFrameSize = 0,
                                           unsigned playTimePerFrame = 0);

    u_int64_t fileSize() const { return fFileSize; }

    // 定位方法
    void seekToByteAbsolute(u_int64_t byteNumber, u_int64_t numBytesToStream = 0);
    void seekToByteRelative(int64_t offset, u_int64_t numBytesToStream = 0);
    void seekToEnd();

protected:
    ByteStreamFileSource(UsageEnvironment& env, FILE* fid,
                         unsigned preferredFrameSize,
                         unsigned playTimePerFrame);
    virtual ~ByteStreamFileSource();

    static void fileReadableHandler(ByteStreamFileSource* source, int mask);
    void doReadFromFile();

private:
    // 重写的虚函数
    virtual void doGetNextFrame();
    virtual void doStopGettingFrames();

protected:
    u_int64_t fFileSize;

private:
    unsigned fPreferredFrameSize;   // 首选帧大小
    unsigned fPlayTimePerFrame;      // 每帧播放时间
    Boolean fFidIsSeekable;          // 文件是否可定位
    unsigned fLastPlayTime;
    Boolean fHaveStartedReading;
    Boolean fLimitNumBytesToStream;
    u_int64_t fNumBytesToStream;
};
```

### 4.2 doGetNextFrame() 实现

```cpp
void ByteStreamFileSource::doGetNextFrame() {
    if (feof(fFid) || ferror(fFid) || 
        (fLimitNumBytesToStream && fNumBytesToStream == 0)) {
        // 文件结束或错误
        handleClosure();
        return;
    }

    // 确定要读取的大小
    unsigned numBytesToRead = fPreferredFrameSize > 0 
                              ? fPreferredFrameSize 
                              : fMaxSize;
    
    if (fLimitNumBytesToStream && numBytesToRead > fNumBytesToStream) {
        numBytesToRead = (unsigned)fNumBytesToStream;
    }

    // 使用异步读取（通过事件循环）
    if (!fHaveStartedReading) {
        // 第一次读取，注册文件可读事件
        envir(). taskScheduler().setBackgroundHandling(
            fileno(fFid),
            SOCKET_READABLE,
            (TaskScheduler::BackgroundHandlerProc*)&fileReadableHandler,
            this
        );
        fHaveStartedReading = True;
    }
}

void ByteStreamFileSource::doReadFromFile() {
    // 实际读取文件
    fFrameSize = fread(fTo, 1, fMaxSize, fFid);
    
    if (fFrameSize == 0) {
        handleClosure();
        return;
    }

    if (fLimitNumBytesToStream) {
        fNumBytesToStream -= fFrameSize;
    }

    // 设置时间戳
    gettimeofday(&fPresentationTime, NULL);
    
    // 设置持续时间
    fDurationInMicroseconds = fPlayTimePerFrame;

    // 通知数据已就绪
    FramedSource::afterGetting(this);
}
```

---

## 五、FramedFilter 过滤器

### 5.1 类定义

FramedFilter 是连接上下游 Source 的过滤器基类：

```cpp
// 文件：FramedFilter.hh

class FramedFilter: public FramedSource {
public:
    // 获取输入源
    FramedSource* inputSource() const { return fInputSource; }

    // 更换输入源
    void reassignInputSource(FramedSource* newInputSource) { 
        fInputSource = newInputSource; 
    }

    // 分离输入源（防止析构时关闭输入源）
    void detachInputSource();

protected:
    FramedFilter(UsageEnvironment& env, FramedSource* inputSource);
    virtual ~FramedFilter();

protected:
    // 可重写的虚函数
    virtual char const* MIMEtype() const;
    virtual void getAttributes() const;
    virtual void doStopGettingFrames();

protected:
    FramedSource* fInputSource;  // 上游输入源
};
```

### 5.2 过滤器链结构

```
┌─────────────────────────────────────────────────────────────────┐
│                    过滤器链结构                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   【典型的 H.264 视频处理链】                                    │
│                                                                 │
│   ┌─────────────────┐                                           │
│   │ ByteStreamFile  │  读取 H.264 文件的原始字节                │
│   │ Source          │                                           │
│   └────────┬────────┘                                           │
│            │ fInputSource                                       │
│            ▼                                                    │
│   ┌─────────────────┐                                           │
│   │ H264VideoStream │  解析字节流，提取 NAL 单元                │
│   │ Framer          │  添加时间戳                               │
│   └────────┬────────┘                                           │
│            │ fInputSource                                       │
│            ▼                                                    │
│   ┌─────────────────┐                                           │
│   │ H264FUAFragmen  │  将大的 NAL 分片（可选）                  │
│   │ ter             │                                           │
│   └────────┬────────┘                                           │
│            │                                                    │
│            ▼                                                    │
│   ┌─────────────────┐                                           │
│   │ H264VideoRTP    │  打包成 RTP 发送                          │
│   │ Sink            │                                           │
│   └─────────────────┘                                           │
│                                                                 │
│   数据流向: FileSource → Framer → Sink                          │
│   请求方向: Sink → Framer → FileSource                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 5.3 过滤器的数据获取流程

```
┌─────────────────────────────────────────────────────────────────┐
│                    过滤器数据获取流程                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   RTPSink                                                       │
│      │                                                          │
│      │ ① getNextFrame(buffer, callback)                         │
│      ▼                                                          │
│   H264VideoStreamFramer (FramedFilter)                          │
│      │                                                          │
│      │ ② doGetNextFrame()                                       │
│      │    需要从上游获取数据                                    │
│      │                                                          │
│      │ ③ fInputSource->getNextFrame(                            │
│      │       tempBuffer, myCallback)                            │
│      ▼                                                          │
│   ByteStreamFileSource                                          │
│      │                                                          │
│      │ ④ doGetNextFrame()                                       │
│      │    从文件读取数据                                        │
│      │                                                          │
│      │ ⑤ afterGetting() → myCallback                            │
│      ▼                                                          │
│   回到 H264VideoStreamFramer::myCallback()                      │
│      │                                                          │
│      │ ⑥ 处理数据（解析NAL单元等）                              │
│      │                                                          │
│      │ ⑦ 将处理后的数据复制到 fTo                               │
│      │    设置 fFrameSize, fPresentationTime 等                 │
│      │                                                          │
│      │ ⑧ afterGetting(this) → 原始callback                      │
│      ▼                                                          │
│   回到 RTPSink 的回调                                           │
│      │                                                          │
│      │ ⑨ 处理帧数据，发送 RTP 包                                │
│      │                                                          │
│      │ ⑩ 再次调用 getNextFrame()                                │
│      ▼                                                          │
│   循环重复...                                                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 六、H264or5VideoStreamFramer 示例

### 6.1 类定义

```cpp
// 文件：H264or5VideoStreamFramer. hh

class H264or5VideoStreamFramer: public MPEGVideoStreamFramer {
public:
    // 获取 VPS/SPS/PPS（视频参数）
    void getVPSandSPSandPPS(u_int8_t*& vps, unsigned& vpsSize,
                            u_int8_t*& sps, unsigned& spsSize,
                            u_int8_t*& pps, unsigned& ppsSize) const;

    // 设置 VPS/SPS/PPS
    void setVPSandSPSandPPS(u_int8_t* vps, unsigned vpsSize,
                            u_int8_t* sps, unsigned spsSize,
                            u_int8_t* pps, unsigned ppsSize);

protected:
    H264or5VideoStreamFramer(int hNumber,  // 264 或 265
                             UsageEnvironment& env, 
                             FramedSource* inputSource,
                             Boolean createParser,
                             Boolean includeStartCodeInOutput,
                             Boolean insertAccessUnitDelimiters);
    virtual ~H264or5VideoStreamFramer();

    // NAL 单元类型判断
    Boolean isVPS(u_int8_t nal_unit_type);
    Boolean isSPS(u_int8_t nal_unit_type);
    Boolean isPPS(u_int8_t nal_unit_type);
    Boolean isVCL(u_int8_t nal_unit_type);

protected:
    // 重写的虚函数
    virtual void doGetNextFrame();

protected:
    int fHNumber;  // 264 或 265
    Boolean fIncludeStartCodeInOutput;
    Boolean fInsertAccessUnitDelimiters;
    
    // 保存最近看到的参数集
    u_int8_t* fLastSeenVPS;
    unsigned fLastSeenVPSSize;
    u_int8_t* fLastSeenSPS;
    unsigned fLastSeenSPSSize;
    u_int8_t* fLastSeenPPS;
    unsigned fLastSeenPPSSize;
    
    struct timeval fNextPresentationTime;
};
```

### 6.2 工作原理

```
┌─────────────────────────────────────────────────────────────────┐
│                    H264VideoStreamFramer 工作原理                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   【输入】H. 264 字节流                                          │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │ 00 00 00 01 67 ...  | 00 00 00 01 68 ... | 00 00 01 65 .. │   │
│   │   起始码   SPS      起始码    PPS       起始码   IDR    │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│   【处理】                                                       │
│   1. 查找起始码 (00 00 00 01 或 00 00 01)                       │
│   2. 分离出每个 NAL 单元                                        │
│   3. 解析 NAL 类型（SPS/PPS/IDR/非IDR等）                       │
│   4. 保存 SPS/PPS（供 RTP 打包使用）                            │
│   5. 计算时间戳                                                 │
│                                                                 │
│   【输出】一次一个 NAL 单元                                     │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │ 第1帧: SPS NAL (67 ...)                                 │   │
│   │ 第2帧: PPS NAL (68 ...)                                 │   │
│   │ 第3帧: IDR NAL (65 ...)                                 │   │
│   │ 第4帧: 非IDR NAL (41 ...)                               │   │
│   │ ...                                                      │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│   💡 每次 getNextFrame() 返回一个完整的 NAL 单元                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 七、同步与异步数据获取

### 7.1 同步 vs 异步

```
┌─────────────────────────────────────────────────────────────────┐
│                    同步 vs 异步数据获取                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   【同步获取】                                                   │
│   doGetNextFrame() 立即完成，直接调用 afterGetting()            │
│                                                                 │
│   void MySource::doGetNextFrame() {                             │
│       // 立即生成数据                                           │
│       fFrameSize = generateData(fTo, fMaxSize);                 │
│       gettimeofday(&fPresentationTime, NULL);                   │
│       fDurationInMicroseconds = 40000; // 25fps                 │
│                                                                 │
│       // 直接调用完成回调                                       │
│       afterGetting(this);                                       │
│   }                                                             │
│                                                                 │
│   ─────────────────────────────────────────────────────────────│
│                                                                 │
│   【异步获取】                                                   │
│   doGetNextFrame() 注册事件，稍后调用 afterGetting()            │
│                                                                 │
│   void MySource::doGetNextFrame() {                             │
│       // 注册可读事件，等待数据到达                             │
│       envir().taskScheduler().setBackgroundHandling(            │
│           fSocketNum, SOCKET_READABLE,                          │
│           incomingDataHandler, this);                           │
│       // 不调用 afterGetting()，等回调                          │
│   }                                                             │
│                                                                 │
│   void MySource::incomingDataHandler(void* clientData, int) {   │
│       MySource* source = (MySource*)clientData;                 │
│       source->readData();                                       │
│   }                                                             │
│                                                                 │
│   void MySource::readData() {                                   │
│       fFrameSize = recv(fSocketNum, fTo, fMaxSize, 0);          │
│       gettimeofday(&fPresentationTime, NULL);                   │
│                                                                 │
│       // 数据就绪，调用完成回调                                 │
│       afterGetting(this);                                       │
│   }                                                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 7.2 使用定时器的异步获取

```cpp
// 定时产生帧（如模拟摄像头）
void MyLiveSource::doGetNextFrame() {
    // 计算下一帧的时间
    struct timeval now;
    gettimeofday(&now, NULL);
    
    int delayUs = calculateNextFrameDelay();
    
    // 注册定时任务
    nextTask() = envir().taskScheduler().scheduleDelayedTask(
        delayUs,
        (TaskFunc*)deliverFrame,
        this
    );
}

void MyLiveSource::deliverFrame(void* clientData) {
    MyLiveSource* source = (MyLiveSource*)clientData;
    
    if (! source->isCurrentlyAwaitingData()) return;
    
    // 从设备/编码器获取帧
    source->fFrameSize = source->captureFrame(source->fTo, source->fMaxSize);
    gettimeofday(&source->fPresentationTime, NULL);
    source->fDurationInMicroseconds = 40000;  // 25fps
    
    // 通知完成
    FramedSource::afterGetting(source);
}
```

---

## 八、stopGettingFrames() 停止获取

### 8.1 实现

```cpp
void FramedSource::stopGettingFrames() {
    fIsCurrentlyAwaitingData = False;
    
    // 清空回调
    fAfterGettingFunc = NULL;
    fOnCloseFunc = NULL;
    
    // 调用子类的清理逻辑
    doStopGettingFrames();
}

// 默认实现（空）
void FramedSource::doStopGettingFrames() {
}
```

### 8.2 子类实现示例

```cpp
// ByteStreamFileSource 的实现
void ByteStreamFileSource::doStopGettingFrames() {
    // 取消文件读取的事件监听
    envir().taskScheduler().turnOffBackgroundReadHandling(fileno(fFid));
    fHaveStartedReading = False;
}

// FramedFilter 的实现
void FramedFilter::doStopGettingFrames() {
    // 也停止上游的获取
    if (fInputSource != NULL) {
        fInputSource->stopGettingFrames();
    }
}
```

---

## 九、使用示例

### 9.1 创建简单的数据处理链

```cpp
// 创建文件源
ByteStreamFileSource* fileSource = 
    ByteStreamFileSource::createNew(env, "test.264");
if (fileSource == NULL) {
    env << "Unable to open file\n";
    return;
}

// 创建 H. 264 帧解析器
H264VideoStreamFramer* framer = 
    H264VideoStreamFramer::createNew(env, fileSource);

// 创建 RTP Sink
Groupsock* rtpGroupsock = new Groupsock(env, destAddr, rtpPort, ttl);
RTPSink* rtpSink = H264VideoRTPSink::createNew(env, rtpGroupsock, 96);

// 开始播放
rtpSink->startPlaying(*framer, afterPlaying, NULL);
```

### 9.2 实现自定义 FramedSource

```cpp
class MyDeviceSource: public FramedSource {
public:
    static MyDeviceSource* createNew(UsageEnvironment& env) {
        return new MyDeviceSource(env);
    }
    
protected:
    MyDeviceSource(UsageEnvironment& env)
        : FramedSource(env) {
        // 初始化设备
    }
    
    virtual ~MyDeviceSource() {
        // 关闭设备
    }
    
    virtual void doGetNextFrame() {
        // 从设备获取一帧数据
        fFrameSize = readFromDevice(fTo, fMaxSize);
        
        if (fFrameSize == 0) {
            // 没有数据，稍后重试
            nextTask() = envir().taskScheduler(). scheduleDelayedTask(
                10000,  // 10ms后重试
                (TaskFunc*)tryAgain,
                this
            );
            return;
        }
        
        // 设置时间戳
        gettimeofday(&fPresentationTime, NULL);
        fDurationInMicroseconds = 40000;  // 25fps
        
        // 通知完成
        afterGetting(this);
    }
    
    virtual void doStopGettingFrames() {
        // 取消定时任务
        envir().taskScheduler().unscheduleDelayedTask(nextTask());
    }
    
private:
    static void tryAgain(void* clientData) {
        MyDeviceSource* source = (MyDeviceSource*)clientData;
        source->doGetNextFrame();
    }
    
    unsigned readFromDevice(unsigned char* buffer, unsigned maxSize) {
        // 实际的设备读取逻辑
        return 0;
    }
};
```

### 9.3 实现自定义 FramedFilter

```cpp
class MyVideoFilter: public FramedFilter {
public:
    static MyVideoFilter* createNew(UsageEnvironment& env, 
                                    FramedSource* inputSource) {
        return new MyVideoFilter(env, inputSource);
    }
    
protected:
    MyVideoFilter(UsageEnvironment& env, FramedSource* inputSource)
        : FramedFilter(env, inputSource) {
    }
    
    virtual ~MyVideoFilter() {
    }
    
    virtual void doGetNextFrame() {
        // 从上游获取数据
        fInputSource->getNextFrame(
            fTo, fMaxSize,
            afterGettingFrame, this,
            FramedSource::handleClosure, this
        );
    }
    
private:
    static void afterGettingFrame(void* clientData, 
                                  unsigned frameSize,
                                  unsigned numTruncatedBytes,
                                  struct timeval presentationTime,
                                  unsigned durationInMicroseconds) {
        MyVideoFilter* filter = (MyVideoFilter*)clientData;
        filter->afterGettingFrame1(frameSize, numTruncatedBytes,
                                   presentationTime, durationInMicroseconds);
    }
    
    void afterGettingFrame1(unsigned frameSize,
                            unsigned numTruncatedBytes,
                            struct timeval presentationTime,
                            unsigned durationInMicroseconds) {
        // 处理数据（例如添加水印、转码等）
        processFrame(fTo, frameSize);
        
        // 设置输出参数
        fFrameSize = frameSize;
        fNumTruncatedBytes = numTruncatedBytes;
        fPresentationTime = presentationTime;
        fDurationInMicroseconds = durationInMicroseconds;
        
        // 通知完成
        afterGetting(this);
    }
    
    void processFrame(unsigned char* data, unsigned size) {
        // 实际的处理逻辑
    }
};
```

---

## 十、小结

### 10.1 本节要点

1. **FramedSource** 是所有帧数据源的基类
2. **getNextFrame()** 是获取帧的标准接口（异步模式）
3.  **doGetNextFrame()** 是子类必须实现的纯虚函数
4. **afterGetting()** 在数据就绪后调用，触发回调
5. **FramedFilter** 用于连接上下游，形成处理链
6. 支持**同步**和**异步**两种数据获取模式

### 10. 2 核心类速查表

| 类                      | 功能                     |
| ----------------------- | ------------------------ |
| `FramedSource`          | 帧数据源基类             |
| `FramedFilter`          | 过滤器基类（连接上下游） |
| `ByteStreamFileSource`  | 从文件读取字节流         |
| `H264VideoStreamFramer` | H.264 NAL 单元解析       |
| `H265VideoStreamFramer` | H.265 NAL 单元解析       |

### 10.3 关键函数速查表

| 函数                    | 功能                   |
| ----------------------- | ---------------------- |
| `getNextFrame()`        | 请求获取下一帧（异步） |
| `doGetNextFrame()`      | 子类实现：实际获取数据 |
| `afterGetting()`        | 数据就绪后调用         |
| `handleClosure()`       | 处理源关闭             |
| `stopGettingFrames()`   | 停止获取帧             |
| `doStopGettingFrames()` | 子类实现：停止时的清理 |

### 10.4 数据流与请求流

```
┌─────────────────────────────────────────────────────────────────┐
│                    数据流 vs 请求流                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   FileSource ──────► Framer ──────► Sink                        │
│       │                                 │                       │
│       │          数据流方向 ──►         │                       │
│       │                                 │                       │
│       │         ◄── 请求流方向          │                       │
│       │                                 │                       │
│       │                                 │                       │
│       │   ⑤ afterGetting()             │                       │
│       │   ────────────────────►        │                       │
│       │                                │                       │
│       │   ④ doGetNextFrame()          │                       │
│       │   ◄────────────────────       │                       │
│       │                               │                        │
│       │                   ② getNextFrame()                     │
│       │                   ◄────────────────────                │
│       │                                                        │
│       │                   ① startPlaying()                     │
│       │                   ◄────────────────────                │
│                                                                 │
│   💡 Sink 主动"拉取"数据，Source 被动"推送"                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

