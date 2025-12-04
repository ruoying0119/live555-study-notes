# Live555源码学习笔记12——RTSPClient 实现



## 一、RTSPClient 概述

### 1.1 RTSPClient 的作用

RTSPClient 是 Live555 中用于连接 RTSP 服务器并控制流媒体播放的客户端类：

```
┌─────────────────────────────────────────────────────────────────┐
│                    RTSPClient 功能概述                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   【核心功能】                                                   │
│   • 连接 RTSP 服务器                                            │
│   • 发送 RTSP 命令（OPTIONS/DESCRIBE/SETUP/PLAY等）             │
│   • 处理 RTSP 响应                                              │
│   • 管理媒体会话（MediaSession）                                │
│   • 支持认证（Digest Authentication）                           │
│   • 支持 RTSP over HTTP tunneling                               │
│   • 支持 RTP over TCP                                           │
│                                                                 │
│   【典型使用流程】                                               │
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                                                         │   │
│   │   1.  创建 RTSPClient                                    │   │
│   │      RTSPClient::createNew(env, url)                    │   │
│   │                     │                                   │   │
│   │                     ▼                                   │   │
│   │   2. 发送 DESCRIBE，获取 SDP                            │   │
│   │      sendDescribeCommand(responseHandler)               │   │
│   │                     │                                   │   │
│   │                     ▼                                   │   │
│   │   3.  创建 MediaSession                                  │   │
│   │      MediaSession::createNew(env, sdp)                  │   │
│   │                     │                                   │   │
│   │                     ▼                                   │   │
│   │   4. 对每个 Subsession 发送 SETUP                       │   │
│   │      sendSetupCommand(subsession, responseHandler)      │   │
│   │                     │                                   │   │
│   │                     ▼                                   │   │
│   │   5. 发送 PLAY，开始接收数据                            │   │
│   │      sendPlayCommand(session, responseHandler)          │   │
│   │                     │                                   │   │
│   │                     ▼                                   │   │
│   │   6. 播放完成后发送 TEARDOWN                            │   │
│   │      sendTeardownCommand(session, responseHandler)      │   │
│   │                                                         │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 类层次结构

```
┌─────────────────────────────────────────────────────────────────┐
│                    RTSPClient 相关类层次                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   Medium                                                        │
│       │                                                         │
│       ├─► RTSPClient                                            │
│       │   ├─ 发送 RTSP 命令                                     │
│       │   ├─ 处理 RTSP 响应                                     │
│       │   └─ 管理请求队列                                       │
│       │                                                         │
│       └─► MediaSession                                          │
│           ├─ 解析 SDP                                           │
│           ├─ 管理 MediaSubsession 链表                          │
│           └─ 客户端会话描述                                     │
│                                                                 │
│   MediaSubsession                                               │
│       ├─ 单个媒体轨道（音频/视频）                              │
│       ├─ 创建 RTPSource / RTCPInstance                          │
│       └─ 管理接收状态                                           │
│                                                                 │
│   ─────────────────────────────────────────────────────────────│
│                                                                 │
│   内嵌类：                                                       │
│                                                                 │
│   RTSPClient::RequestRecord                                     │
│       ├─ 存储待处理请求的信息                                   │
│       ├─ CSeq, 命令名, 回调函数等                               │
│       └─ 形成请求队列                                           │
│                                                                 │
│   RTSPClient::RequestQueue                                      │
│       ├─ 管理 RequestRecord 链表                                │
│       └─ 入队/出队/查找操作                                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 二、RTSPClient 类定义

### 2.1 主要接口

```cpp
// 文件：RTSPClient.hh

class RTSPClient: public Medium {
public:
    // 创建 RTSPClient
    static RTSPClient* createNew(UsageEnvironment& env, 
                                 char const* rtspURL,
                                 int verbosityLevel = 0,
                                 char const* applicationName = NULL,
                                 portNumBits tunnelOverHTTPPortNum = 0,
                                 int socketNumToServer = -1);

    // 响应回调函数类型
    typedef void (responseHandler)(RTSPClient* rtspClient,
                                   int resultCode, 
                                   char* resultString);
    // resultCode: 0=成功, >0=RTSP错误码, <0=-errno

    // ⭐ 发送 RTSP 命令的方法（返回 CSeq）
    unsigned sendDescribeCommand(responseHandler* responseHandler, 
                                 Authenticator* authenticator = NULL);
    
    unsigned sendOptionsCommand(responseHandler* responseHandler, 
                                Authenticator* authenticator = NULL);
    
    unsigned sendAnnounceCommand(char const* sdpDescription, 
                                 responseHandler* responseHandler, 
                                 Authenticator* authenticator = NULL);
    
    unsigned sendSetupCommand(MediaSubsession& subsession, 
                              responseHandler* responseHandler,
                              Boolean streamOutgoing = False,
                              Boolean streamUsingTCP = False,
                              Boolean forceMulticastOnUnspecified = False,
                              Authenticator* authenticator = NULL);
    
    unsigned sendPlayCommand(MediaSession& session, 
                             responseHandler* responseHandler,
                             double start = 0.0f, double end = -1.0f, 
                             float scale = 1.0f,
                             Authenticator* authenticator = NULL);
    
    unsigned sendPlayCommand(MediaSubsession& subsession, 
                             responseHandler* responseHandler,
                             double start = 0. 0f, double end = -1.0f, 
                             float scale = 1. 0f,
                             Authenticator* authenticator = NULL);
    
    unsigned sendPauseCommand(MediaSession& session, 
                              responseHandler* responseHandler, 
                              Authenticator* authenticator = NULL);
    
    unsigned sendPauseCommand(MediaSubsession& subsession, 
                              responseHandler* responseHandler, 
                              Authenticator* authenticator = NULL);
    
    unsigned sendTeardownCommand(MediaSession& session, 
                                 responseHandler* responseHandler, 
                                 Authenticator* authenticator = NULL);
    
    unsigned sendTeardownCommand(MediaSubsession& subsession, 
                                 responseHandler* responseHandler, 
                                 Authenticator* authenticator = NULL);
    
    unsigned sendSetParameterCommand(MediaSession& session, 
                                     responseHandler* responseHandler,
                                     char const* parameterName, 
                                     char const* parameterValue,
                                     Authenticator* authenticator = NULL);
    
    unsigned sendGetParameterCommand(MediaSession& session, 
                                     responseHandler* responseHandler, 
                                     char const* parameterName,
                                     Authenticator* authenticator = NULL);

    // 更改响应处理器
    Boolean changeResponseHandler(unsigned cseq, 
                                  responseHandler* newResponseHandler);

    // 获取 socket
    int socketNum() const { return fInputSocketNum; }

    // 解析 RTSP URL
    Boolean parseRTSPURL(char const* url,
                         char*& username, char*& password, 
                         NetAddress& address, portNumBits& portNum, 
                         char const** urlSuffix = NULL);

    // 设置 User-Agent
    void setUserAgentString(char const* userAgentName);

    // 禁用 Basic 认证
    void disallowBasicAuthentication() { fAllowBasicAuthentication = False; }

    // 获取会话超时时间
    unsigned sessionTimeoutParameter() const { return fSessionTimeoutParameter; }

    // 获取 URL
    char const* url() const { return fBaseURL; }

    static unsigned responseBufferSize;

protected:
    RTSPClient(UsageEnvironment& env, char const* rtspURL,
               int verbosityLevel, char const* applicationName, 
               portNumBits tunnelOverHTTPPortNum, int socketNumToServer);
    virtual ~RTSPClient();

    void reset();
    void setBaseURL(char const* url);
    virtual unsigned sendRequest(RequestRecord* request);

private:
    // 请求队列
    RequestQueue fRequestsAwaitingConnection;
    RequestQueue fRequestsAwaitingHTTPTunneling;
    RequestQueue fRequestsAwaitingResponse;

    // 连接和数据处理
    int openConnection();
    static void connectionHandler(void*, int);
    void connectionHandler1();
    static void incomingDataHandler(void*, int);
    void incomingDataHandler1();
    void handleResponseBytes(int newBytesRead);

protected:
    int fVerbosityLevel;
    unsigned fCSeq;  // 序列号
    Authenticator fCurrentAuthenticator;
    Boolean fAllowBasicAuthentication;
    struct sockaddr_storage fServerAddress;

private:
    portNumBits fTunnelOverHTTPPortNum;
    char* fUserAgentHeaderStr;
    int fInputSocketNum, fOutputSocketNum;
    char* fBaseURL;
    unsigned char fTCPStreamIdCount;
    char* fLastSessionId;
    unsigned fSessionTimeoutParameter;
    char* fResponseBuffer;
    unsigned fResponseBytesAlreadySeen, fResponseBufferBytesLeft;
};
```

### 2.2 RequestRecord 内嵌类

```cpp
// 文件：RTSPClient.hh

class RTSPClient::RequestRecord {
public:
    RequestRecord(unsigned cseq, char const* commandName, 
                  responseHandler* handler,
                  MediaSession* session = NULL, 
                  MediaSubsession* subsession = NULL, 
                  u_int32_t booleanFlags = 0,
                  double start = 0.0f, double end = -1.0f, 
                  float scale = 1. 0f, 
                  char const* contentStr = NULL);
    
    // 用于绝对时间的 PLAY 请求
    RequestRecord(unsigned cseq, responseHandler* handler,
                  char const* absStartTime, char const* absEndTime = NULL, 
                  float scale = 1.0f,
                  MediaSession* session = NULL, 
                  MediaSubsession* subsession = NULL);
    
    virtual ~RequestRecord();

    // 访问器
    RequestRecord*& next() { return fNext; }
    unsigned& cseq() { return fCSeq; }
    char const* commandName() const { return fCommandName; }
    MediaSession* session() const { return fSession; }
    MediaSubsession* subsession() const { return fSubsession; }
    u_int32_t booleanFlags() const { return fBooleanFlags; }
    double start() const { return fStart; }
    double end() const { return fEnd; }
    char const* absStartTime() const { return fAbsStartTime; }
    char const* absEndTime() const { return fAbsEndTime; }
    float scale() const { return fScale; }
    char* contentStr() const { return fContentStr; }
    responseHandler*& handler() { return fHandler; }

private:
    RequestRecord* fNext;
    unsigned fCSeq;
    char const* fCommandName;
    MediaSession* fSession;
    MediaSubsession* fSubsession;
    u_int32_t fBooleanFlags;
    double fStart, fEnd;
    char *fAbsStartTime, *fAbsEndTime;
    float fScale;
    char* fContentStr;
    responseHandler* fHandler;
};
```

---

## 三、MediaSession 类

### 3.1 类定义

MediaSession 表示客户端的媒体会话，从 SDP 创建：

```cpp
// 文件：MediaSession.hh

class MediaSession: public Medium {
public:
    // 从 SDP 创建
    static MediaSession* createNew(UsageEnvironment& env,
                                   char const* sdpDescription);

    static Boolean lookupByName(UsageEnvironment& env, char const* sourceName,
                                MediaSession*& resultSession);

    // 是否有子会话
    Boolean hasSubsessions() const { return fSubsessionsHead != NULL; }

    // 会话信息
    char* connectionEndpointName() const { return fConnectionEndpointName; }
    char const* CNAME() const { return fCNAME; }
    float& scale() { return fScale; }
    float& speed() { return fSpeed; }
    char* mediaSessionType() const { return fMediaSessionType; }
    char* sessionName() const { return fSessionName; }
    char* sessionDescription() const { return fSessionDescription; }
    char const* controlPath() const { return fControlPath; }

    // 播放时间范围
    double& playStartTime() { return fMaxPlayStartTime; }
    double& playEndTime() { return fMaxPlayEndTime; }
    char* absStartTime() const;
    char* absEndTime() const;

    // 根据 MIME 类型初始化
    Boolean initiateByMediaType(char const* mimeType,
                                MediaSubsession*& resultSubsession,
                                int useSpecialRTPoffset = -1);

    // 加密相关
    MIKEYState* getMIKEYState() const { return fMIKEYState; }
    SRTPCryptographicContext* getCrypto() const { return fCrypto; }

protected:
    MediaSession(UsageEnvironment& env);
    virtual ~MediaSession();

    virtual MediaSubsession* createNewMediaSubsession();
    Boolean initializeWithSDP(char const* sdpDescription);

    // SDP 解析方法
    Boolean parseSDPLine(char const* input, char const*& nextLine);
    Boolean parseSDPLine_s(char const* sdpLine);
    Boolean parseSDPLine_i(char const* sdpLine);
    Boolean parseSDPLine_c(char const* sdpLine);
    Boolean parseSDPAttribute_type(char const* sdpLine);
    Boolean parseSDPAttribute_control(char const* sdpLine);
    Boolean parseSDPAttribute_range(char const* sdpLine);

protected:
    char* fCNAME;  // 用于 RTCP
    
    // 子会话链表
    MediaSubsession* fSubsessionsHead;
    MediaSubsession* fSubsessionsTail;

    // 从 SDP 解析的字段
    char* fConnectionEndpointName;
    double fMaxPlayStartTime;
    double fMaxPlayEndTime;
    char* fAbsStartTime;
    char* fAbsEndTime;
    float fScale;
    float fSpeed;
    char* fMediaSessionType;
    char* fSessionName;
    char* fSessionDescription;
    char* fControlPath;
};
```

### 3.2 MediaSubsession 类

```cpp
// 文件：MediaSession.hh

class MediaSubsession {
public:
    // 父会话
    MediaSession& parentSession() { return fParent; }
    
    // 媒体信息
    unsigned short clientPortNum() const { return fClientPortNum; }
    unsigned char rtpPayloadFormat() const { return fRTPPayloadFormat; }
    char const* savedSDPLines() const { return fSavedSDPLines; }
    char const* mediumName() const { return fMediumName; }      // "video"/"audio"
    char const* codecName() const { return fCodecName; }        // "H264"/"AAC"
    char const* protocolName() const { return fProtocolName; }  // "RTP"
    char const* controlPath() const { return fControlPath; }

    // 视频特定
    unsigned short videoWidth() const { return fVideoWidth; }
    unsigned short videoHeight() const { return fVideoHeight; }
    unsigned videoFPS() const { return fVideoFPS; }
    
    // 音频特定
    unsigned numChannels() const { return fNumChannels; }

    // RTP/RTCP 对象
    RTPSource* rtpSource() { return fRTPSource; }
    RTCPInstance* rtcpInstance() { return fRTCPInstance; }
    unsigned rtpTimestampFrequency() const { return fRTPTimestampFrequency; }
    
    // 读取源（可能添加了过滤器）
    FramedSource* readSource() { return fReadSource; }
    void addFilter(FramedFilter* filter);

    // 播放时间
    double playStartTime() const;
    double playEndTime() const;

    // ⭐ 初始化（创建 RTPSource 等）
    Boolean initiate(int useSpecialRTPoffset = -1);
    void deInitiate();  // 销毁

    // 设置客户端端口
    Boolean setClientPortNum(unsigned short portNum);

    // 获取连接地址
    void getConnectionEndpointAddress(struct sockaddr_storage& addr) const;
    
    // 设置目标地址（SETUP 后调用）
    void setDestinations(struct sockaddr_storage const& defaultDestAddress);

    // RTSP 会话 ID
    char const* sessionId() const { return fSessionId; }
    void setSessionId(char const* sessionId);

    // 公共字段
    unsigned short serverPortNum;          // 服务器 RTP 端口
    unsigned char rtpChannelId, rtcpChannelId;  // TCP 通道 ID
    MediaSink* sink;   // 调用者可用于跟踪播放器
    void* miscPtr;     // 调用者自用

    // 从 RTP-Info 头解析的参数
    struct {
        u_int16_t seqNum;
        u_int32_t timestamp;
        Boolean infoIsNew;
    } rtpInfo;

    // 计算 NPT（正常播放时间）
    double getNormalPlayTime(struct timeval const& presentationTime);

    // SDP 属性访问
    char const* attrVal_str(char const* attrName) const;
    unsigned attrVal_int(char const* attrName) const;

    // 常用 fmtp 属性
    char const* fmtp_config() const;
    char const* fmtp_spropparametersets() const;

protected:
    MediaSubsession(MediaSession& parent);
    virtual ~MediaSubsession();

    // SDP 解析
    Boolean parseSDPLine_c(char const* sdpLine);
    Boolean parseSDPLine_b(char const* sdpLine);
    Boolean parseSDPAttribute_rtpmap(char const* sdpLine);
    Boolean parseSDPAttribute_control(char const* sdpLine);
    Boolean parseSDPAttribute_fmtp(char const* sdpLine);

    // 创建源对象
    virtual Boolean createSourceObjects(int useSpecialRTPoffset);

protected:
    MediaSession& fParent;
    MediaSubsession* fNext;

    // 从 SDP 解析的字段
    char* fConnectionEndpointName;
    unsigned short fClientPortNum;
    unsigned char fRTPPayloadFormat;
    char* fMediumName;
    char* fCodecName;
    char* fProtocolName;
    unsigned fRTPTimestampFrequency;
    char* fControlPath;

    // 视频/音频参数
    unsigned short fVideoWidth, fVideoHeight;
    unsigned fVideoFPS;
    unsigned fNumChannels;
    float fScale, fSpeed;
    unsigned fBandwidth;

    // initiate() 创建的对象
    Groupsock* fRTPSocket;
    Groupsock* fRTCPSocket;
    RTPSource* fRTPSource;
    RTCPInstance* fRTCPInstance;
    FramedSource* fReadSource;

    // 会话 ID
    char* fSessionId;
};
```

---

## 四、RTSP 命令发送流程

### 4.1 命令发送机制

```
┌─────────────────────────────────────────────────────────────────┐
│                    RTSP 命令发送流程                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   用户调用 sendDescribeCommand(responseHandler)                 │
│        │                                                        │
│        ▼                                                        │
│   创建 RequestRecord                                            │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │ fCSeq = ++fCSeq (如 1)                                  │   │
│   │ fCommandName = "DESCRIBE"                               │   │
│   │ fHandler = responseHandler                              │   │
│   │ fSession = NULL                                         │   │
│   │ fSubsession = NULL                                      │   │
│   └─────────────────────────────────────────────────────────┘   │
│        │                                                        │
│        ▼                                                        │
│   sendRequest(request)                                          │
│        │                                                        │
│        │  检查连接状态                                          │
│        │  ├─ 未连接：加入 fRequestsAwaitingConnection          │
│        │  │          调用 openConnection()                     │
│        │  │                                                    │
│        │  └─ 已连接：直接发送                                  │
│        ▼                                                        │
│   构造 RTSP 请求字符串                                          │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │ DESCRIBE rtsp://server/stream RTSP/1.0\r\n              │   │
│   │ CSeq: 1\r\n                                             │   │
│   │ User-Agent: Live555\r\n                                 │   │
│   │ Accept: application/sdp\r\n                             │   │
│   │ \r\n                                                    │   │
│   └─────────────────────────────────────────────────────────┘   │
│        │                                                        │
│        ▼                                                        │
│   write() 发送到服务器                                          │
│        │                                                        │
│        ▼                                                        │
│   将 request 加入 fRequestsAwaitingResponse                     │
│        │                                                        │
│        ▼                                                        │
│   返回 CSeq (1)                                                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 响应处理流程

```
┌─────────────────────────────────────────────────────────────────┐
│                    RTSP 响应处理流程                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   服务器响应到达                                                │
│        │                                                        │
│        ▼                                                        │
│   incomingDataHandler() 被触发                                  │
│        │                                                        │
│        ▼                                                        │
│   incomingDataHandler1()                                        │
│        │                                                        │
│        │  read() 读取数据到 fResponseBuffer                     │
│        ▼                                                        │
│   handleResponseBytes()                                         │
│        │                                                        │
│        │  ① 解析响应行                                          │
│        │     "RTSP/1.0 200 OK"                                  │
│        │     提取 responseCode = 200                            │
│        │                                                        │
│        │  ② 解析 CSeq 头                                        │
│        │     "CSeq: 1"                                          │
│        │     提取 cseq = 1                                      │
│        │                                                        │
│        │  ③ 从 fRequestsAwaitingResponse 查找请求               │
│        │     request = findByCSeq(1)                            │
│        │                                                        │
│        │  ④ 解析其他头部                                        │
│        │     Content-Type, Content-Length, Session 等           │
│        │                                                        │
│        │  ⑤ 处理特定命令的响应                                  │
│        │     ├─ DESCRIBE: 提取 SDP                              │
│        │     ├─ SETUP: 调用 handleSETUPResponse()               │
│        │     └─ PLAY: 调用 handlePLAYResponse()                 │
│        │                                                        │
│        │  ⑥ 调用用户的回调函数                                  │
│        ▼                                                        │
│   (*request->handler())(this, responseCode, resultString)       │
│        │                                                        │
│        ▼                                                        │
│   删除 request                                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 五、MediaSession 创建流程

### 5.1 从 SDP 创建 MediaSession

```
┌─────────────────────────────────────────────────────────────────┐
│                    MediaSession 创建流程                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   DESCRIBE 响应中的 SDP:                                        │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │ v=0                                                     │   │
│   │ o=- 12345 1 IN IP4 192. 168.1.1                          │   │
│   │ s=Live Stream                                           │   │
│   │ c=IN IP4 192.168.1. 1                                    │   │
│   │ t=0 0                                                   │   │
│   │ a=control:*                                             │   │
│   │ a=range:npt=0-                                          │   │
│   │ m=video 0 RTP/AVP 96                                    │   │
│   │ a=rtpmap:96 H264/90000                                  │   │
│   │ a=fmtp:96 packetization-mode=1                          │   │
│   │ a=control:track1                                        │   │
│   │ m=audio 0 RTP/AVP 97                                    │   │
│   │ a=rtpmap:97 MPEG4-GENERIC/44100/2                       │   │
│   │ a=control:track2                                        │   │
│   └─────────────────────────────────────────────────────────┘   │
│        │                                                        │
│        ▼                                                        │
│   MediaSession::createNew(env, sdp)                             │
│        │                                                        │
│        ▼                                                        │
│   initializeWithSDP(sdp)                                        │
│        │                                                        │
│        │  ① 解析会话级 SDP 行                                   │
│        │     v=, o=, s=, c=, t=, a= (会话属性)                  │
│        │                                                        │
│        │  ② 遇到 m= 行，创建 MediaSubsession                    │
│        │     createNewMediaSubsession()                         │
│        │     加入链表                                           │
│        │                                                        │
│        │  ③ 解析媒体级 SDP 行                                   │
│        │     m=video 0 RTP/AVP 96                               │
│        │     a=rtpmap:96 H264/90000                             │
│        │     a=control:track1                                   │
│        │                                                        │
│        │  ④ 重复 ②③ 处理其他 m= 行                              │
│        ▼                                                        │
│   返回 MediaSession*                                            │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │ MediaSession                                            │   │
│   │ ├─ fSubsessionsHead                                     │   │
│   │ │   ├─ MediaSubsession (video, H264, track1)            │   │
│   │ │   │   ├─ fMediumName = "video"                        │   │
│   │ │   │   ├─ fCodecName = "H264"                          │   │
│   │ │   │   ├─ fRTPTimestampFrequency = 90000               │   │
│   │ │   │   └─ fControlPath = "track1"                      │   │
│   │ │   │                                                   │   │
│   │ │   └─► MediaSubsession (audio, MPEG4-GENERIC, track2)  │   │
│   │ │       ├─ fMediumName = "audio"                        │   │
│   │ │       ├─ fCodecName = "MPEG4-GENERIC"                 │   │
│   │ │       ├─ fRTPTimestampFrequency = 44100               │   │
│   │ │       └─ fControlPath = "track2"                      │   │
│   │ │                                                       │   │
│   │ └─ fSubsessionsTail ───────────────────────────┘        │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 MediaSubsession 初始化

```
┌─────────────────────────────────────────────────────────────────┐
│                    MediaSubsession::initiate() 流程              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   subsession->initiate()                                        │
│        │                                                        │
│        │  ① 创建 RTP Socket                                     │
│        │     fRTPSocket = new Groupsock(env, addr, rtpPort, ttl)│
│        │                                                        │
│        │  ② 创建 RTCP Socket                                    │
│        │     fRTCPSocket = new Groupsock(env, addr, rtcpPort)   │
│        │                                                        │
│        │  ③ 调用 createSourceObjects()                          │
│        │     根据 codecName 创建对应的 RTPSource                │
│        ▼                                                        │
│   createSourceObjects()                                         │
│        │                                                        │
│        │  根据 fCodecName 分支：                                │
│        │                                                        │
│        ├─ "H264":                                               │
│        │     fRTPSource = H264VideoRTPSource::createNew(...)    │
│        │     fReadSource = fRTPSource                           │
│        │                                                        │
│        ├─ "H265":                                               │
│        │     fRTPSource = H265VideoRTPSource::createNew(...)    │
│        │     fReadSource = fRTPSource                           │
│        │                                                        │
│        ├─ "MPEG4-GENERIC":                                      │
│        │     fRTPSource = MPEG4GenericRTPSource::createNew(... ) │
│        │     fReadSource = fRTPSource                           │
│        │                                                        │
│        └─ 其他编码...                                            │
│                                                                 │
│        │  ④ 创建 RTCPInstance                                   │
│        ▼                                                        │
│   fRTCPInstance = RTCPInstance::createNew(                      │
│       env, fRTCPSocket, bandwidth, cname,                       │
│       NULL,  // 没有 RTPSink                                    │
│       fRTPSource  // 有 RTPSource                               │
│   )                                                             │
│        │                                                        │
│        ▼                                                        │
│   返回 True (成功)                                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 六、完整的 RTSP 客户端示例

### 6.1 典型使用流程

```cpp
#include "liveMedia.hh"
#include "BasicUsageEnvironment.hh"

// 全局变量
UsageEnvironment* env;
RTSPClient* rtspClient;
MediaSession* session;
char eventLoopWatchVariable = 0;

// 前向声明
void continueAfterDESCRIBE(RTSPClient*, int, char*);
void continueAfterSETUP(RTSPClient*, int, char*);
void continueAfterPLAY(RTSPClient*, int, char*);
void setupNextSubsession(RTSPClient*);

int main(int argc, char** argv) {
    // 1. 创建使用环境
    TaskScheduler* scheduler = BasicTaskScheduler::createNew();
    env = BasicUsageEnvironment::createNew(*scheduler);
    
    // 2.  创建 RTSP 客户端
    char const* rtspURL = "rtsp://192.168.1. 100:554/live";
    rtspClient = RTSPClient::createNew(*env, rtspURL, 1, "testClient");
    
    if (rtspClient == NULL) {
        *env << "Failed to create RTSPClient: " << env->getResultMsg() << "\n";
        return 1;
    }
    
    // 3. 发送 DESCRIBE 命令
    rtspClient->sendDescribeCommand(continueAfterDESCRIBE);
    
    // 4. 进入事件循环
    env->taskScheduler(). doEventLoop(&eventLoopWatchVariable);
    
    return 0;
}

// DESCRIBE 响应处理
void continueAfterDESCRIBE(RTSPClient* rtspClient, int resultCode, char* resultString) {
    if (resultCode != 0) {
        *env << "Failed to get SDP: " << resultString << "\n";
        delete[] resultString;
        return;
    }
    
    // resultString 是 SDP 描述
    char* sdpDescription = resultString;
    *env << "Got SDP:\n" << sdpDescription << "\n";
    
    // 创建 MediaSession
    session = MediaSession::createNew(*env, sdpDescription);
    delete[] sdpDescription;
    
    if (session == NULL) {
        *env << "Failed to create MediaSession\n";
        return;
    }
    
    // 设置每个 Subsession
    setupNextSubsession(rtspClient);
}

// 迭代器（全局）
MediaSubsessionIterator* iter = NULL;

void setupNextSubsession(RTSPClient* rtspClient) {
    if (iter == NULL) {
        iter = new MediaSubsessionIterator(*session);
    }
    
    MediaSubsession* subsession = iter->next();
    
    if (subsession != NULL) {
        // 初始化 subsession
        if (! subsession->initiate()) {
            *env << "Failed to initiate subsession\n";
            setupNextSubsession(rtspClient);
            return;
        }
        
        *env << "Initiated subsession: " << subsession->mediumName() 
             << "/" << subsession->codecName() << "\n";
        
        // 发送 SETUP 命令
        rtspClient->sendSetupCommand(*subsession, continueAfterSETUP,
                                     False,  // streamOutgoing
                                     False); // streamUsingTCP (改为True启用RTP/TCP)
    } else {
        // 所有 subsession 都已设置，发送 PLAY
        delete iter;
        iter = NULL;
        
        rtspClient->sendPlayCommand(*session, continueAfterPLAY);
    }
}

// SETUP 响应处理
void continueAfterSETUP(RTSPClient* rtspClient, int resultCode, char* resultString) {
    if (resultCode != 0) {
        *env << "SETUP failed: " << resultString << "\n";
    } else {
        *env << "SETUP succeeded\n";
    }
    delete[] resultString;
    
    // 设置下一个 subsession
    setupNextSubsession(rtspClient);
}

// PLAY 响应处理
void continueAfterPLAY(RTSPClient* rtspClient, int resultCode, char* resultString) {
    if (resultCode != 0) {
        *env << "PLAY failed: " << resultString << "\n";
    } else {
        *env << "PLAY succeeded, now receiving stream.. .\n";
        
        // 创建 Sink 开始接收数据
        MediaSubsessionIterator iter2(*session);
        MediaSubsession* subsession;
        while ((subsession = iter2.next()) != NULL) {
            if (subsession->readSource() != NULL) {
                // 创建 FileSink 保存数据
                char filename[100];
                sprintf(filename, "%s-%s.dat", 
                        subsession->mediumName(), 
                        subsession->codecName());
                        
                subsession->sink = FileSink::createNew(*env, filename);
                
                if (subsession->sink != NULL) {
                    subsession->sink->startPlaying(
                        *(subsession->readSource()),
                        NULL, NULL  // 播放完成回调
                    );
                }
            }
        }
    }
    delete[] resultString;
}
```

### 6.2 使用认证

```cpp
// 创建认证器
Authenticator* auth = new Authenticator("username", "password");

// 发送需要认证的命令
rtspClient->sendDescribeCommand(continueAfterDESCRIBE, auth);

// 后续命令自动使用相同的认证信息
rtspClient->sendSetupCommand(*subsession, continueAfterSETUP);
```

### 6.3 使用 RTP over TCP

```cpp
// SETUP 时指定使用 TCP
rtspClient->sendSetupCommand(*subsession, continueAfterSETUP,
                             False,  // streamOutgoing
                             True);  // streamUsingTCP = True
```

### 6.4 获取统计信息

```cpp
// 获取接收统计
RTPSource* rtpSource = subsession->rtpSource();
if (rtpSource != NULL) {
    RTPReceptionStatsDB& statsDB = rtpSource->receptionStatsDB();
    
    *env << "Total packets received: " << statsDB.totNumPacketsReceived() << "\n";
    
    RTPReceptionStatsDB::Iterator iter(statsDB);
    RTPReceptionStats* stats;
    while ((stats = iter. next()) != NULL) {
        *env << "SSRC: " << stats->SSRC() << "\n";
        *env << "Packets received: " << stats->totNumPacketsReceived() << "\n";
        *env << "Packets expected: " << stats->totNumPacketsExpected() << "\n";
        *env << "Jitter: " << stats->jitter() << "\n";
    }
}
```

---

## 七、小结

### 7.1 本节要点

1. **RTSPClient** 是 RTSP 客户端的核心类，发送命令并处理响应
2. **MediaSession** 从 SDP 创建，管理媒体会话
3. **MediaSubsession** 表示单个媒体轨道，创建 RTPSource
4.  RTSP 命令使用**异步回调机制**处理响应
5. **RequestRecord** 存储待处理请求，形成请求队列
6. 支持**认证**、**RTP over TCP**、**RTSP over HTTP**

### 7.2 核心类速查表

| 类                          | 功能                  |
| --------------------------- | --------------------- |
| `RTSPClient`                | RTSP 客户端，发送命令 |
| `RTSPClient::RequestRecord` | 存储请求信息          |
| `RTSPClient::RequestQueue`  | 请求队列管理          |
| `MediaSession`              | 客户端媒体会话        |
| `MediaSubsession`           | 单个媒体轨道          |
| `MediaSubsessionIterator`   | 遍历子会话            |

### 7.3 关键方法速查表

| 方法                          | 功能            |
| ----------------------------- | --------------- |
| `RTSPClient::createNew()`     | 创建客户端      |
| `sendDescribeCommand()`       | 发送 DESCRIBE   |
| `sendSetupCommand()`          | 发送 SETUP      |
| `sendPlayCommand()`           | 发送 PLAY       |
| `sendPauseCommand()`          | 发送 PAUSE      |
| `sendTeardownCommand()`       | 发送 TEARDOWN   |
| `MediaSession::createNew()`   | 从 SDP 创建会话 |
| `MediaSubsession::initiate()` | 初始化子会话    |

### 7.4 客户端工作流程图

```
┌─────────────────────────────────────────────────────────────────┐
│                    RTSP 客户端完整流程                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │ RTSPClient::createNew(env, url)                         │   │
│   └───────────────────────────┬─────────────────────────────┘   │
│                               │                                 │
│                               ▼                                 │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │ sendDescribeCommand() ──► 响应回调                      │   │
│   │                           │                             │   │
│   │                           ▼                             │   │
│   │ MediaSession::createNew(sdp) ──► 解析 SDP              │   │
│   └───────────────────────────┬─────────────────────────────┘   │
│                               │                                 │
│                               ▼                                 │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │ 遍历 MediaSubsession                                    │   │
│   │     │                                                   │   │
│   │     ├─► subsession->initiate()                          │   │
│   │     │   创建 RTPSocket, RTPSource, RTCPInstance         │   │
│   │     │                                                   │   │
│   │     └─► sendSetupCommand(subsession) ──► 响应回调       │   │
│   │         设置 serverPortNum 等                           │   │
│   └───────────────────────────┬─────────────────────────────┘   │
│                               │                                 │
│                               ▼                                 │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │ sendPlayCommand(session) ──► 响应回调                   │   │
│   │                              │                          │   │
│   │                              ▼                          │   │
│   │ 开始接收 RTP/RTCP 数据                                  │   │
│   │ subsession->readSource() ──► Sink                       │   │
│   └───────────────────────────┬─────────────────────────────┘   │
│                               │                                 │
│                               ▼                                 │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │ 播放结束或用户停止                                      │   │
│   │     │                                                   │   │
│   │     └─► sendTeardownCommand(session)                    │   │
│   │         关闭连接                                        │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

