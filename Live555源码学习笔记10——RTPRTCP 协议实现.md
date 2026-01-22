# Live555源码学习笔记10——RTP/RTCP 协议实现

## 一、RTP/RTCP 协议概述

### 1.1 RTP 与 RTCP 的关系

```
┌─────────────────────────────────────────────────────────────────┐
│                    RTP/RTCP 协议关系                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   【RTP - Real-time Transport Protocol】                        │
│   • 实时传输协议                                                │
│   • 用于传输音视频数据                                          │
│   • 默认使用偶数端口（如 5000）                                 │
│                                                                 │
│   【RTCP - RTP Control Protocol】                               │
│   • RTP 控制协议                                                │
│   • 用于传输控制信息（质量反馈、同步等）                        │
│   • 默认使用 RTP端口+1（如 5001）                               │
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                                                         │   │
│   │   发送端                              接收端            │   │
│   │   ┌──────────┐                    ┌──────────┐          │   │
│   │   │ RTPSink  │ ═══RTP数据════════►│ RTPSource│          │   │
│   │   │          │      (端口N)        │          │          │   │
│   │   │          │                    │          │          │   │
│   │   │ RTCP     │ ◄══RTCP报告═══════►│ RTCP     │          │   │
│   │   │ Instance │     (端口N+1)       │ Instance │          │   │
│   │   └──────────┘                    └──────────┘          │   │
│   │                                                         │   │
│   │   发送 SR 报告 ──────────────────► 接收 SR               │   │
│   │   接收 RR 报告 ◄────────────────── 发送 RR 报告          │   │
│   │                                                         │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 RTP 包结构

```
┌─────────────────────────────────────────────────────────────────┐
│                    RTP 包结构 (RFC 3550)                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│    0                   1                   2                   3│
│    0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
│   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
│   |V=2|P|X|  CC   |M|     PT      |       sequence number         |
│   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
│   |                           timestamp                           |
│   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
│   |           synchronization source (SSRC) identifier            |
│   +=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
│   |            contributing source (CSRC) identifiers             |
│   |                             ....                              |
│   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
│   |                          payload ...                           |
│   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
│                                                                 │
│   【字段说明】                                                   │
│   V (2 bits)   : 版本号，固定为 2                               │
│   P (1 bit)    : 填充标志                                       │
│   X (1 bit)    : 扩展标志                                       │
│   CC (4 bits)  : CSRC 计数                                      │
│   M (1 bit)    : 标记位（帧结束等）                             │
│   PT (7 bits)  : 载荷类型（如 96 = H.264）                      │
│   Seq (16 bits): 序列号（每包递增）                             │
│   Timestamp    : RTP 时间戳                                     │
│   SSRC         : 同步源标识符                                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 1.3 Live555 中的 RTP 相关类

```
┌─────────────────────────────────────────────────────────────────┐
│                    RTP 相关类层次                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   【发送端】                                                     │
│                                                                 │
│   MediaSink                                                     │
│       │                                                         │
│       ▼                                                         │
│   RTPSink (抽象基类)                                            │
│       │                                                         │
│       ▼                                                         │
│   MultiFramedRTPSink (通用实现)                                 │
│       │                                                         │
│       ├─► H264VideoRTPSink                                      │
│       ├─► H265VideoRTPSink                                      │
│       ├─► MPEG4GenericRTPSink                                   │
│       └─► SimpleRTPSink                                         │
│                                                                 │
│   ─────────────────────────────────────────────────────────────│
│                                                                 │
│   【接收端】                                                     │
│                                                                 │
│   FramedSource                                                  │
│       │                                                         │
│       ▼                                                         │
│   RTPSource (抽象基类)                                          │
│       │                                                         │
│       ▼                                                         │
│   MultiFramedRTPSource (通用实现)                               │
│       │                                                         │
│       ├─► H264VideoRTPSource                                    │
│       ├─► H265VideoRTPSource                                    │
│       ├─► MPEG4GenericRTPSource                                 │
│       └─► SimpleRTPSource                                       │
│                                                                 │
│   ─────────────────────────────────────────────────────────────│
│                                                                 │
│   【控制协议】                                                   │
│                                                                 │
│   RTCPInstance - RTCP 协议处理                                  │
│   RTPInterface - RTP/RTCP 网络接口封装                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 二、RTPInterface 类

### 2.1 类定义

RTPInterface 封装了 RTP/RTCP 的网络传输，支持 UDP 和 TCP（RTP over TCP）：

```cpp
// 文件：RTPInterface.hh

// 辅助处理函数类型
typedef void AuxHandlerFunc(void* clientData, unsigned char* packet,
                            unsigned& packetSize);

class RTPInterface {
public:
    RTPInterface(Medium* owner, Groupsock* gs);
    virtual ~RTPInterface();

    // 获取 Groupsock
    Groupsock* gs() const { return fGS; }

    // TCP 流支持（RTP over TCP）
    void setStreamSocket(int sockNum, unsigned char streamChannelId, 
                         TLSState* tlsState);
    void addStreamSocket(int sockNum, unsigned char streamChannelId, 
                         TLSState* tlsState);
    void removeStreamSocket(int sockNum, unsigned char streamChannelId);

    // 发送数据包
    Boolean sendPacket(unsigned char* packet, unsigned packetSize);

    // 开始/停止网络读取
    void startNetworkReading(TaskScheduler::BackgroundHandlerProc* handlerProc);
    void stopNetworkReading();

    // 读取数据包
    Boolean handleRead(unsigned char* buffer, unsigned bufferMaxSize,
                       unsigned& bytesRead, 
                       struct sockaddr_storage& fromAddress,
                       int& tcpSocketNum, 
                       unsigned char& tcpStreamChannelId,
                       Boolean& packetReadWasIncomplete);

    // 设置辅助读取处理器
    void setAuxilliaryReadHandler(AuxHandlerFunc* handlerFunc,
                                  void* handlerClientData);

    UsageEnvironment& envir() const { return fOwner->envir(); }

private:
    Medium* fOwner;
    Groupsock* fGS;
    class tcpStreamRecord* fTCPStreams;  // TCP流记录链表
    
    // TCP 读取状态
    unsigned short fNextTCPReadSize;
    int fNextTCPReadStreamSocketNum;
    unsigned char fNextTCPReadStreamChannelId;
    TLSState* fNextTCPReadTLSState;
    
    TaskScheduler::BackgroundHandlerProc* fReadHandlerProc;
    AuxHandlerFunc* fAuxReadHandlerFunc;
    void* fAuxReadHandlerClientData;
};
```

### 2. 2 UDP vs TCP 传输

```
┌─────────────────────────────────────────────────────────────────┐
│                    RTP 传输方式                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   【UDP 传输】（默认方式）                                       │
│                                                                 │
│   • 使用 Groupsock 发送/接收                                    │
│   • RTP 端口通常为偶数（如 5004）                               │
│   • RTCP 端口 = RTP端口 + 1（如 5005）                          │
│   • 支持单播和组播                                              │
│                                                                 │
│   ┌──────────────┐          UDP           ┌──────────────┐      │
│   │   RTPSink    │ ═══════════════════════►│  RTPSource   │      │
│   │  (端口5004)  │                        │  (端口5004)  │      │
│   └──────────────┘                        └──────────────┘      │
│                                                                 │
│   ─────────────────────────────────────────────────────────────│
│                                                                 │
│   【TCP 传输】（RTP over TCP，RFC 2326 Section 10.12）          │
│                                                                 │
│   • 复用 RTSP 的 TCP 连接                                       │
│   • 使用 interleaved 帧封装                                     │
│   • 适合穿越防火墙                                              │
│                                                                 │
│   TCP Interleaved Frame:                                        │
│   ┌────────┬───────────┬──────────────────────────────────┐     │
│   │  '$'   │ Channel   │  Length   │     RTP/RTCP Data    │     │
│   │ (1B)   │   (1B)    │   (2B)    │     (N bytes)        │     │
│   └────────┴───────────┴───────────┴──────────────────────┘     │
│                                                                 │
│   Channel 0: RTP 数据                                           │
│   Channel 1: RTCP 数据                                          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 三、RTPSink 类

### 3.1 类定义

RTPSink 是 RTP 发送端的基类：

```cpp
// 文件：RTPSink.hh

class RTPSink: public MediaSink {
public:
    static Boolean lookupByName(UsageEnvironment& env, 
                                char const* sinkName,
                                RTPSink*& resultSink);

    // 获取 Groupsock
    Groupsock const& groupsockBeingUsed() const;
    Groupsock& groupsockBeingUsed();

    // RTP 参数
    unsigned char rtpPayloadType() const { return fRTPPayloadType; }
    unsigned rtpTimestampFrequency() const { return fTimestampFrequency; }
    void setRTPTimestampFrequency(unsigned freq);
    char const* rtpPayloadFormatName() const { return fRTPPayloadFormatName; }
    unsigned numChannels() const { return fNumChannels; }

    // SRTP 设置
    void setupForSRTP(Boolean useEncryption, u_int32_t roc);

    // SDP 生成
    virtual char const* sdpMediaType() const;  // "video" 或 "audio"
    virtual char* rtpmapLine() const;          // a=rtpmap:... 
    virtual char const* auxSDPLine();          // a=fmtp:...

    // 当前状态
    u_int16_t currentSeqNo() const { return fSeqNo; }
    u_int32_t SSRC() const { return fSSRC; }

    // 统计信息
    RTPTransmissionStatsDB& transmissionStatsDB() const;
    void getTotalBitrate(unsigned& outNumBytes, double& outElapsedTime);

    // TCP 传输支持
    void setStreamSocket(int sockNum, unsigned char streamChannelId, 
                         TLSState* tlsState);
    void addStreamSocket(int sockNum, unsigned char streamChannelId, 
                         TLSState* tlsState);
    void removeStreamSocket(int sockNum, unsigned char streamChannelId);

protected:
    RTPSink(UsageEnvironment& env,
            Groupsock* rtpGS,
            unsigned char rtpPayloadType,
            u_int32_t rtpTimestampFrequency,
            char const* rtpPayloadFormatName,
            unsigned numChannels);
    virtual ~RTPSink();

    // RTCP 友元访问
    friend class RTCPInstance;
    u_int32_t convertToRTPTimestamp(struct timeval tv);
    unsigned packetCount() const { return fPacketCount; }
    unsigned octetCount() const { return fOctetCount; }

protected:
    RTPInterface fRTPInterface;           // 网络接口
    unsigned char fRTPPayloadType;        // 载荷类型
    unsigned fPacketCount, fOctetCount;   // 统计计数
    u_int32_t fCurrentTimestamp;          // 当前时间戳
    u_int16_t fSeqNo;                     // 序列号

private:
    u_int32_t fSSRC;                      // 同步源标识
    u_int32_t fTimestampBase;             // 时间戳基准
    unsigned fTimestampFrequency;         // 时间戳频率
    char const* fRTPPayloadFormatName;    // 载荷格式名
    unsigned fNumChannels;                // 声道数
    
    RTPTransmissionStatsDB* fTransmissionStatsDB;  // 发送统计
};
```

### 3.2 关键字段说明

```
┌─────────────────────────────────────────────────────────────────┐
│                    RTPSink 关键字段                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   【RTP 参数】                                                   │
│                                                                 │
│   fRTPPayloadType (PT)                                          │
│   ├─ 动态类型: 96-127（H.264常用96）                            │
│   └─ 静态类型: 0-95（如PCMU=0, PCMA=8）                         │
│                                                                 │
│   fTimestampFrequency                                           │
│   ├─ 视频: 90000 Hz（90kHz时钟）                                │
│   └─ 音频: 采样率（如 44100, 48000）                            │
│                                                                 │
│   fSSRC (Synchronization Source)                                │
│   └─ 随机生成的32位标识符，标识发送源                           │
│                                                                 │
│   fSeqNo (Sequence Number)                                      │
│   └─ 16位序列号，每发一包递增1                                  │
│                                                                 │
│   fCurrentTimestamp                                             │
│   └─ 当前帧的RTP时间戳                                          │
│                                                                 │
│   【统计信息】                                                   │
│                                                                 │
│   fPacketCount: 已发送的包数量                                  │
│   fOctetCount:  已发送的字节数（不含RTP头）                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 四、MultiFramedRTPSink 类

### 4.1 类定义

MultiFramedRTPSink 是最常用的 RTPSink 实现，支持多帧打包：

```cpp
// 文件：MultiFramedRTPSink.hh

class MultiFramedRTPSink: public RTPSink {
public:
    // 设置包大小
    void setPacketSizes(unsigned preferredPacketSize, unsigned maxPacketSize);

    // 发送错误回调
    typedef void (onSendErrorFunc)(void* clientData);
    void setOnSendErrorFunc(onSendErrorFunc* func, void* data);

protected:
    MultiFramedRTPSink(UsageEnvironment& env,
                       Groupsock* rtpgs,
                       unsigned char rtpPayloadType,
                       unsigned rtpTimestampFrequency,
                       char const* rtpPayloadFormatName,
                       unsigned numChannels = 1);
    virtual ~MultiFramedRTPSink();

    // 子类可重写的虚函数
    virtual void doSpecialFrameHandling(unsigned fragmentationOffset,
                                        unsigned char* frameStart,
                                        unsigned numBytesInFrame,
                                        struct timeval framePresentationTime,
                                        unsigned numRemainingBytes);
    virtual Boolean allowFragmentationAfterStart() const;
    virtual Boolean allowOtherFramesAfterLastFragment() const;
    virtual Boolean frameCanAppearAfterPacketStart(unsigned char const* frameStart,
                                                   unsigned numBytesInFrame) const;
    virtual unsigned specialHeaderSize() const;
    virtual unsigned frameSpecificHeaderSize() const;
    virtual unsigned computeOverflowForNewFrame(unsigned newFrameSize) const;

    // 辅助函数
    Boolean isFirstPacket() const { return fIsFirstPacket; }
    Boolean isFirstFrameInPacket() const { return fNumFramesUsedSoFar == 0; }
    unsigned curFragmentationOffset() const { return fCurFragmentationOffset; }
    void setMarkerBit();
    void setTimestamp(struct timeval framePresentationTime);
    void setSpecialHeaderWord(unsigned word, unsigned wordPosition = 0);
    void setSpecialHeaderBytes(unsigned char const* bytes, unsigned numBytes,
                               unsigned bytePosition = 0);

public:
    virtual void stopPlaying();

protected:
    virtual Boolean continuePlaying();

private:
    void buildAndSendPacket(Boolean isFirstPacket);
    void packFrame();
    void sendPacketIfNecessary();
    static void sendNext(void* firstArg);
    
    static void afterGettingFrame(void* clientData,
                                  unsigned numBytesRead, 
                                  unsigned numTruncatedBytes,
                                  struct timeval presentationTime,
                                  unsigned durationInMicroseconds);
    void afterGettingFrame1(unsigned numBytesRead, 
                            unsigned numTruncatedBytes,
                            struct timeval presentationTime,
                            unsigned durationInMicroseconds);

private:
    OutPacketBuffer* fOutBuf;           // 输出缓冲区
    Boolean fNoFramesLeft;               // 是否还有帧
    unsigned fNumFramesUsedSoFar;        // 当前包中的帧数
    unsigned fCurFragmentationOffset;    // 当前分片偏移
    Boolean fPreviousFrameEndedFragmentation;
    Boolean fIsFirstPacket;              // 是否第一个包
    struct timeval fNextSendTime;        // 下次发送时间
    unsigned fOurMaxPacketSize;          // 最大包大小
};
```

### 4.2 RTP 打包流程

```
┌─────────────────────────────────────────────────────────────────┐
│                    MultiFramedRTPSink 打包流程                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   【流程概述】                                                   │
│                                                                 │
│   startPlaying()                                                │
│        │                                                        │
│        ▼                                                        │
│   continuePlaying()                                             │
│        │                                                        │
│        ▼                                                        │
│   buildAndSendPacket(isFirstPacket=True)                        │
│        │                                                        │
│        │  ┌──────────────────────────────────────────────┐      │
│        │  │           packFrame() 循环                   │      │
│        │  │                                              │      │
│        │  │  ① 调用 source->getNextFrame()              │      │
│        │  │           │                                  │      │
│        │  │           ▼                                  │      │
│        │  │  ② afterGettingFrame() 回调                 │      │
│        │  │           │                                  │      │
│        │  │           ▼                                  │      │
│        │  │  ③ 检查帧能否放入当前包                     │      │
│        │  │      ├─ 能: 添加到 fOutBuf                  │      │
│        │  │      │      继续 packFrame()                │      │
│        │  │      │                                      │      │
│        │  │      └─ 不能: 或包已满                      │      │
│        │  │             ↓                               │      │
│        │  └──────────────────────────────────────────────┘      │
│        │                                                        │
│        ▼                                                        │
│   sendPacketIfNecessary()                                       │
│        │                                                        │
│        │  ④ 发送 RTP 包                                        │
│        │     fRTPInterface.sendPacket()                         │
│        │                                                        │
│        │  ⑤ 调度下一次发送                                     │
│        │     scheduleDelayedTask(sendNext)                      │
│        │                                                        │
│        ▼                                                        │
│   sendNext()                                                    │
│        │                                                        │
│        └──► buildAndSendPacket(isFirstPacket=False)             │
│             (循环继续...)                                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 4.3 RTP 包结构示例

```
┌─────────────────────────────────────────────────────────────────┐
│                    RTP 包结构示例                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   【单帧打包】一个 RTP 包包含一个完整帧                         │
│                                                                 │
│   ┌────────────────┬────────────────────────────────────────┐   │
│   │   RTP Header   │              Frame Data                 │   │
│   │   (12 bytes)   │              (完整帧)                   │   │
│   └────────────────┴────────────────────────────────────────┘   │
│                                                                 │
│   【多帧打包】一个 RTP 包包含多个帧（如音频）                   │
│                                                                 │
│   ┌────────────────┬─────────┬─────────┬─────────┬──────────┐   │
│   │   RTP Header   │ Frame 1 │ Frame 2 │ Frame 3 │   ...     │   │
│   │   (12 bytes)   │         │         │         │          │   │
│   └────────────────┴─────────┴─────────┴─────────┴──────────┘   │
│                                                                 │
│   【分片打包】大帧分成多个 RTP 包（如 H.264 FU-A）              │
│                                                                 │
│   包1 (M=0):                                                    │
│   ┌────────────────┬──────────────┬─────────────────────────┐   │
│   │   RTP Header   │ FU Indicator │  Frame Fragment 1       │   │
│   │   M=0          │ + FU Header  │  (Start)                │   │
│   └────────────────┴──────────────┴─────────────────────────┘   │
│                                                                 │
│   包2 (M=0):                                                    │
│   ┌────────────────┬──────────────┬─────────────────────────┐   │
│   │   RTP Header   │ FU Indicator │  Frame Fragment 2       │   │
│   │   M=0          │ + FU Header  │  (Middle)               │   │
│   └────────────────┴──────────────┴─────────────────────────┘   │
│                                                                 │
│   包N (M=1):                                                    │
│   ┌────────────────┬──────────────┬─────────────────────────┐   │
│   │   RTP Header   │ FU Indicator │  Frame Fragment N       │   │
│   │   M=1 (结束)   │ + FU Header  │  (End)                  │   │
│   └────────────────┴──────────────┴─────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 五、RTPSource 类

### 5.1 类定义

RTPSource 是 RTP 接收端的基类：

```cpp
// 文件：RTPSource.hh

class RTPSource: public FramedSource {
public:
    static Boolean lookupByName(UsageEnvironment& env, 
                                char const* sourceName,
                                RTPSource*& resultSource);

    // 当前包信息
    Boolean curPacketMarkerBit() const { return fCurPacketMarkerBit; }
    unsigned char rtpPayloadFormat() const { return fRTPPayloadFormat; }
    u_int16_t curPacketRTPSeqNum() const { return fCurPacketRTPSeqNum; }

    // RTCP 同步
    virtual Boolean hasBeenSynchronizedUsingRTCP();

    // Groupsock 访问
    Groupsock* RTPgs() const { return fRTPInterface.gs(); }

    // 包重排序阈值
    virtual void setPacketReorderingThresholdTime(unsigned uSeconds) = 0;

    // SRTP 支持
    void setCrypto(SRTPCryptographicContext* crypto);

    // SSRC 信息
    u_int32_t SSRC() const { return fSSRC; }
    u_int32_t lastReceivedSSRC() const { return fLastReceivedSSRC; }
    unsigned timestampFrequency() const { return fTimestampFrequency; }

    // 接收统计
    RTPReceptionStatsDB& receptionStatsDB() const;

    // RTCP 控制
    Boolean& enableRTCPReports() { return fEnableRTCPReports; }
    void registerForMultiplexedRTCPPackets(class RTCPInstance* rtcpInstance);

    // TCP 传输支持
    void setStreamSocket(int sockNum, unsigned char streamChannelId, 
                         TLSState* tlsState);

protected:
    RTPSource(UsageEnvironment& env, Groupsock* RTPgs,
              unsigned char rtpPayloadFormat, 
              u_int32_t rtpTimestampFrequency);
    virtual ~RTPSource();

protected:
    RTPInterface fRTPInterface;
    u_int16_t fCurPacketRTPSeqNum;        // 当前包序列号
    u_int32_t fCurPacketRTPTimestamp;     // 当前包时间戳
    Boolean fCurPacketMarkerBit;          // 当前包标记位
    Boolean fCurPacketHasBeenSynchronizedUsingRTCP;
    u_int32_t fLastReceivedSSRC;          // 最近收到的SSRC
    class RTCPInstance* fRTCPInstanceForMultiplexedRTCPPackets;
    SRTPCryptographicContext* fCrypto;

private:
    unsigned char fRTPPayloadFormat;
    unsigned fTimestampFrequency;
    u_int32_t fSSRC;                      // 我们的SSRC
    Boolean fEnableRTCPReports;

    RTPReceptionStatsDB* fReceptionStatsDB;  // 接收统计
};
```

### 5.2 接收统计类

```cpp
// 接收统计数据库
class RTPReceptionStatsDB {
public:
    unsigned totNumPacketsReceived() const;
    unsigned numActiveSourcesSinceLastReset() const;
    void reset();

    class Iterator { /* 迭代器 */ };

    // 记录收到的包
    void noteIncomingPacket(u_int32_t SSRC, u_int16_t seqNum,
                            u_int32_t rtpTimestamp,
                            unsigned timestampFrequency,
                            Boolean useForJitterCalculation,
                            struct timeval& resultPresentationTime,
                            Boolean& resultHasBeenSyncedUsingRTCP,
                            unsigned packetSize);

    // 记录收到的 SR
    void noteIncomingSR(u_int32_t SSRC,
                        u_int32_t ntpTimestampMSW, 
                        u_int32_t ntpTimestampLSW,
                        u_int32_t rtpTimestamp);

    RTPReceptionStats* lookup(u_int32_t SSRC) const;
};

// 单个源的接收统计
class RTPReceptionStats {
public:
    u_int32_t SSRC() const;
    unsigned numPacketsReceivedSinceLastReset() const;
    unsigned totNumPacketsReceived() const;
    double totNumKBytesReceived() const;
    unsigned totNumPacketsExpected() const;
    unsigned baseExtSeqNumReceived() const;
    unsigned highestExtSeqNumReceived() const;
    unsigned jitter() const;  // 抖动（用于RTCP RR）
    unsigned lastReceivedSR_NTPmsw() const;
    unsigned lastReceivedSR_NTPlsw() const;
    struct timeval const& lastReceivedSR_time() const;
};
```

---

## 六、RTCPInstance 类

### 6.1 类定义

RTCPInstance 处理 RTCP 协议：

```cpp
// 文件：RTCP.hh

class RTCPInstance: public Medium {
public:
    static RTCPInstance* createNew(UsageEnvironment& env, 
                                   Groupsock* RTCPgs,
                                   unsigned totSessionBW,  // kbps
                                   unsigned char const* cname,
                                   RTPSink* sink,
                                   RTPSource* source,
                                   Boolean isSSMTransmitter = False,
                                   SRTPCryptographicContext* crypto = NULL);

    unsigned numMembers() const;
    unsigned totSessionBW() const { return fTotSessionBW; }

    // 设置事件处理器
    void setByeHandler(TaskFunc* handlerTask, void* clientData,
                       Boolean handleActiveParticipantsOnly = True);
    void setByeWithReasonHandler(ByeWithReasonHandlerFunc* handlerTask, 
                                 void* clientData,
                                 Boolean handleActiveParticipantsOnly = True);
    void setSRHandler(TaskFunc* handlerTask, void* clientData);
    void setRRHandler(TaskFunc* handlerTask, void* clientData);
    void setSpecificRRHandler(struct sockaddr_storage const& fromAddress, 
                              Port fromPort,
                              TaskFunc* handlerTask, void* clientData);
    void setAppHandler(RTCPAppHandlerFunc* handlerTask, void* clientData);

    // 发送 APP 包
    void sendAppPacket(u_int8_t subtype, char const* name,
                       u_int8_t* appDependentData, 
                       unsigned appDependentDataSize);

    // 发送 BYE
    void sendBYE(char const* reason = NULL);

    // TCP 传输支持
    void setStreamSocket(int sockNum, unsigned char streamChannelId, 
                         TLSState* tlsState);
    void addStreamSocket(int sockNum, unsigned char streamChannelId, 
                         TLSState* tlsState);

    Groupsock* RTCPgs() const { return fRTCPInterface.gs(); }

protected:
    RTCPInstance(UsageEnvironment& env, Groupsock* RTPgs, 
                 unsigned totSessionBW,
                 unsigned char const* cname,
                 RTPSink* sink, RTPSource* source,
                 Boolean isSSMTransmitter,
                 SRTPCryptographicContext* crypto);
    virtual ~RTCPInstance();

private:
    // 构建和发送报告
    Boolean addReport(Boolean alwaysAdd = False);
    void addSR();   // Sender Report
    void addRR();   // Receiver Report
    void addSDES(); // Source Description
    void addBYE(char const* reason);
    void sendBuiltPacket();

    // 定时发送
    static void onExpire(RTCPInstance* instance);
    void onExpire1();

    // 接收处理
    static void incomingReportHandler(RTCPInstance* instance, int mask);
    void processIncomingReport(unsigned packetSize, 
                               struct sockaddr_storage const& fromAddress,
                               int tcpSocketNum, 
                               unsigned char tcpStreamChannelId);

private:
    OutPacketBuffer* fOutBuf;
    RTPInterface fRTCPInterface;
    unsigned fTotSessionBW;
    RTPSink* fSink;
    RTPSource* fSource;
    Boolean fIsSSMTransmitter;
    SRTPCryptographicContext* fCrypto;

    SDESItem fCNAME;
    RTCPMemberDatabase* fKnownMembers;
    
    // 发送调度
    double fAveRTCPSize;
    double fPrevReportTime;
    double fNextReportTime;

    // 事件处理器
    TaskFunc* fByeHandlerTask;
    void* fByeHandlerClientData;
    TaskFunc* fSRHandlerTask;
    void* fSRHandlerClientData;
    TaskFunc* fRRHandlerTask;
    void* fRRHandlerClientData;
    RTCPAppHandlerFunc* fAppHandlerTask;
    void* fAppHandlerClientData;
};
```

### 6.2 RTCP 包类型

```cpp
// RTCP 包类型常量
const unsigned char RTCP_PT_SR   = 200;  // Sender Report
const unsigned char RTCP_PT_RR   = 201;  // Receiver Report
const unsigned char RTCP_PT_SDES = 202;  // Source Description
const unsigned char RTCP_PT_BYE  = 203;  // Goodbye
const unsigned char RTCP_PT_APP  = 204;  // Application-Defined

// SDES 项类型
const unsigned char RTCP_SDES_END   = 0;
const unsigned char RTCP_SDES_CNAME = 1;  // Canonical Name
const unsigned char RTCP_SDES_NAME  = 2;  // User Name
const unsigned char RTCP_SDES_EMAIL = 3;  // Email
const unsigned char RTCP_SDES_PHONE = 4;  // Phone
const unsigned char RTCP_SDES_LOC   = 5;  // Location
const unsigned char RTCP_SDES_TOOL  = 6;  // Tool
const unsigned char RTCP_SDES_NOTE  = 7;  // Note
```

### 6.3 RTCP 工作机制

```
┌─────────────────────────────────────────────────────────────────┐
│                    RTCP 工作机制                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   【发送端 (有 RTPSink)】                                       │
│                                                                 │
│   定期发送 SR (Sender Report):                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  • NTP 时间戳 (绝对时间)                                │   │
│   │  • RTP 时间戳 (相对时间)                                │   │
│   │  • 发送的包数量                                         │   │
│   │  • 发送的字节数                                         │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│   ─────────────────────────────────────────────────────────────│
│                                                                 │
│   【接收端 (有 RTPSource)】                                     │
│                                                                 │
│   定期发送 RR (Receiver Report):                                │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  • SSRC (报告的发送源)                                  │   │
│   │  • 丢包率                                               │   │
│   │  • 累计丢包数                                           │   │
│   │  • 收到的最高序列号                                     │   │
│   │  • 抖动 (Jitter)                                        │   │
│   │  • 上次 SR 时间戳                                       │   │
│   │  • 从 SR 到现在的延迟                                   │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│   ─────────────────────────────────────────────────────────────│
│                                                                 │
│   【SDES (Source Description)】                                 │
│                                                                 │
│   包含在 SR/RR 后面:                                            │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  • CNAME: 规范名称 (如 "user@host")                     │   │
│   │  • NAME:  用户名                                        │   │
│   │  • EMAIL: 邮箱                                          │   │
│   │  • TOOL:  工具名 (如 "Live555")                         │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│   ─────────────────────────────────────────────────────────────│
│                                                                 │
│   【BYE】                                                        │
│                                                                 │
│   会话结束时发送:                                               │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │  • SSRC                                                 │   │
│   │  • 可选的离开原因                                       │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 七、RTP 发送统计

### 7.1 RTPTransmissionStatsDB

```cpp
// 发送统计数据库
class RTPTransmissionStatsDB {
public:
    unsigned numReceivers() const { return fNumReceivers; }

    class Iterator {
    public:
        Iterator(RTPTransmissionStatsDB& receptionStatsDB);
        RTPTransmissionStats* next();
    };

    // 记录收到的 RR
    void noteIncomingRR(u_int32_t SSRC, 
                        struct sockaddr_storage const& lastFromAddress,
                        unsigned lossStats, 
                        unsigned lastPacketNumReceived,
                        unsigned jitter, 
                        unsigned lastSRTime, 
                        unsigned diffSR_RRTime);

    void removeRecord(u_int32_t SSRC);
    RTPTransmissionStats* lookup(u_int32_t SSRC) const;

private:
    unsigned fNumReceivers;
    RTPSink& fOurRTPSink;
    HashTable* fTable;
};

// 单个接收者的统计
class RTPTransmissionStats {
public:
    u_int32_t SSRC() const;
    struct sockaddr_storage const& lastFromAddress() const;
    unsigned lastPacketNumReceived() const;
    unsigned firstPacketNumReported() const;
    unsigned totNumPacketsLost() const;
    unsigned jitter() const;
    unsigned roundTripDelay() const;  // RTT
    struct timeval const& timeCreated() const;
    struct timeval const& lastTimeReceived() const;
    unsigned packetsReceivedSinceLastRR() const;
    u_int8_t packetLossRatio() const;
};
```

---

## 八、使用示例

### 8.1 创建 RTP 发送端

```cpp
// 创建 RTP Groupsock
struct sockaddr_in destAddr;
destAddr.sin_family = AF_INET;
destAddr.sin_addr.s_addr = inet_addr("239.255.42.42");  // 组播地址
Port rtpPort(5004);
unsigned char ttl = 7;

Groupsock* rtpGroupsock = new Groupsock(env, 
                                         *(sockaddr_storage*)&destAddr, 
                                         rtpPort, ttl);

// 创建 RTCP Groupsock
Port rtcpPort(5005);
Groupsock* rtcpGroupsock = new Groupsock(env, 
                                          *(sockaddr_storage*)&destAddr, 
                                          rtcpPort, ttl);

// 创建 H.264 RTP Sink
unsigned char payloadType = 96;
RTPSink* rtpSink = H264VideoRTPSink::createNew(env, rtpGroupsock, payloadType);

// 创建 RTCP Instance
unsigned estimatedSessionBW = 500;  // 500 kbps
unsigned char* cname = (unsigned char*)"my_stream@my_host";
RTCPInstance* rtcpInstance = RTCPInstance::createNew(env, 
                                                      rtcpGroupsock,
                                                      estimatedSessionBW, 
                                                      cname,
                                                      rtpSink, 
                                                      NULL,  // 没有 RTPSource
                                                      False);

// 开始发送
rtpSink->startPlaying(*videoSource, afterPlaying, NULL);
```

### 8.2 创建 RTP 接收端

```cpp
// 创建 RTP Groupsock
Port rtpPort(5004);
Groupsock* rtpGroupsock = new Groupsock(env, groupAddr, rtpPort, ttl);

// 创建 RTCP Groupsock  
Port rtcpPort(5005);
Groupsock* rtcpGroupsock = new Groupsock(env, groupAddr, rtcpPort, ttl);

// 创建 H.264 RTP Source
unsigned char payloadType = 96;
RTPSource* rtpSource = H264VideoRTPSource::createNew(env, rtpGroupsock, 
                                                      payloadType, 90000);

// 创建 RTCP Instance
unsigned estimatedSessionBW = 500;
RTCPInstance* rtcpInstance = RTCPInstance::createNew(env, 
                                                      rtcpGroupsock,
                                                      estimatedSessionBW, 
                                                      cname,
                                                      NULL,  // 没有 RTPSink
                                                      rtpSource,
                                                      False);

// 设置 BYE 处理器
rtcpInstance->setByeHandler(handleBye, NULL);

// 创建 FileSink 保存数据
MediaSink* fileSink = FileSink::createNew(env, "output.264");
fileSink->startPlaying(*rtpSource, afterPlaying, NULL);
```

### 8.3 设置 RTCP 事件处理

```cpp
// BYE 处理器
void handleBye(void* clientData) {
    env << "Received RTCP BYE\n";
    // 执行清理操作
}

// 带原因的 BYE 处理器
void handleByeWithReason(void* clientData, char const* reason) {
    if (reason != NULL) {
        env << "Received RTCP BYE with reason: " << reason << "\n";
        delete[] reason;
    }
}

// SR 处理器
void handleSR(void* clientData) {
    env << "Received RTCP SR\n";
}

// RR 处理器
void handleRR(void* clientData) {
    env << "Received RTCP RR\n";
    
    // 可以查询统计信息
    RTPTransmissionStatsDB& statsDB = rtpSink->transmissionStatsDB();
    env << "Number of receivers: " << statsDB.numReceivers() << "\n";
}

// 设置处理器
rtcpInstance->setByeHandler(handleBye, NULL);
rtcpInstance->setByeWithReasonHandler(handleByeWithReason, NULL);
rtcpInstance->setSRHandler(handleSR, NULL);
rtcpInstance->setRRHandler(handleRR, NULL);
```

---

## 九、小结

### 9.1 本节要点

1. **RTPInterface** 封装了 UDP 和 TCP (RTP over TCP) 传输
2. **RTPSink** 是 RTP 发送端基类，**MultiFramedRTPSink** 是通用实现
3. **RTPSource** 是 RTP 接收端基类，继承自 FramedSource
4. **RTCPInstance** 处理 RTCP 协议（SR/RR/SDES/BYE）
5.  RTCP 提供质量反馈（丢包率、抖动）和同步信息

### 9.2 核心类速查表

| 类                       | 功能                  |
| ------------------------ | --------------------- |
| `RTPInterface`           | RTP/RTCP 网络接口封装 |
| `RTPSink`                | RTP 发送端基类        |
| `MultiFramedRTPSink`     | 通用 RTP 发送实现     |
| `RTPSource`              | RTP 接收端基类        |
| `RTCPInstance`           | RTCP 协议处理         |
| `RTPTransmissionStatsDB` | 发送统计              |
| `RTPReceptionStatsDB`    | 接收统计              |

### 9.3 RTCP 包类型速查表

| 类型 | 值   | 说明                          |
| ---- | ---- | ----------------------------- |
| SR   | 200  | Sender Report（发送端报告）   |
| RR   | 201  | Receiver Report（接收端报告） |
| SDES | 202  | Source Description（源描述）  |
| BYE  | 203  | Goodbye（会话结束）           |
| APP  | 204  | Application-Defined（自定义） |

### 9.4 类关系图

```
┌─────────────────────────────────────────────────────────────────┐
│                    RTP/RTCP 类关系图                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   MediaSink                       FramedSource                  │
│       │                               │                         │
│       ▼                               ▼                         │
│   RTPSink ◄─────────────────────► RTPSource                     │
│       │                               │                         │
│       │     fRTPInterface             │     fRTPInterface       │
│       ▼                               ▼                         │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                    RTPInterface                         │   │
│   │    • Groupsock* (UDP)                                   │   │
│   │    • tcpStreamRecord* (TCP)                             │   │
│   │    • sendPacket() / handleRead()                        │   │
│   └─────────────────────────────────────────────────────────┘   │
│                               │                                 │
│                               ▼                                 │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                    RTCPInstance                         │   │
│   │    • RTPSink* fSink                                     │   │
│   │    • RTPSource* fSource                                 │   │
│   │    • 发送 SR/RR/SDES/BYE                                │   │
│   │    • 处理接收的 RTCP 报告                               │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---





```

H264VideoRTPSink::startPlaying(FramedSource& source, ...)
  └─> MultiFramedRTPSink::startPlaying(...)
        └─> MultiFramedRTPSink::continuePlaying()
              └─> 调 source.getNextFrame(...)
                  // 这里的 source 实际是 H264VideoStreamFramer

MultiFramedRTPSink::continuePlaying()
  │
  │ 调用（FramedSource 公共接口）
  ▼
H264VideoStreamFramer::getNextFrame(u_int8_t* to, unsigned maxSize,
                                    afterGettingFunc*, onCloseFunc*, clientData)
  └─> FramedSource::getNextFrame(...)  // 基类做一些状态设置
        └─> H264VideoStreamFramer::doGetNextFrame()
              // 这里是 H264/H265 专用：
              //   - 看当前缓冲里有没有完整 NALU/Frame
              //   - 如果不够，就向下游 ByteStreamFileSource 再要数据
              
H264VideoStreamFramer::doGetNextFrame()
  │
  │ 如果当前还没有完整的一帧 H.264 数据：
  │   → 向下游 inputSource（ByteStreamFileSource）调用 getNextFrame()
  ▼
ByteStreamFileSource::getNextFrame(to, maxSize, afterGettingFunc*, onClose*, clientData)
  └─> FramedSource::getNextFrame(...)
        └─> ByteStreamFileSource::doGetNextFrame()
              └─> 可能异步/同步进入 doReadFromFile()
```



```
MultiFramedRTPSink::afterGettingFrame1()
  │
  │ 1. 调专用的 doSpecialFrameHandling()（在 H264or5VideoRTPSink 里）
  │    - 判断帧大小，决定单 NALU / FU-A 分片 / STAP-A 等
  │    - 填 payload NALU 头、FU indicator、FU header 等
  │ 2. 把 RTP 头 + 负载写入 OutPacketBuffer
  │
  ▼
MultiFramedRTPSink::sendPacketIfNecessary()
  └─> RTPSink::sendPacket()
        └─> Groupsock::output(envir(), packet, packetSize)
              └─> ::sendto(udpSocket, packet, packetSize, ...)
```

