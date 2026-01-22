# Live555源码学习笔记11——RTSPServer 实现

## 一、RTSP 协议概述

### 1.1 什么是 RTSP

RTSP（Real Time Streaming Protocol）是用于控制流媒体服务器的应用层协议：

```
┌─────────────────────────────────────────────────────────────────┐
│                    RTSP 协议特点                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   【基本信息】                                                   │
│   • 默认端口：554（RTSP）或 322（RTSPS/TLS）                    │
│   • 基于文本协议，类似 HTTP                                     │
│   • 用于控制媒体流的播放（PLAY/PAUSE/TEARDOWN等）               │
│   • 不传输媒体数据本身（那是 RTP 的工作）                       │
│                                                                 │
│   【主要命令】                                                   │
│   • OPTIONS   - 查询支持的方法                                  │
│   • DESCRIBE  - 获取媒体描述（SDP）                             │
│   • SETUP     - 建立传输通道                                    │
│   • PLAY      - 开始播放                                        │
│   • PAUSE     - 暂停播放                                        │
│   • TEARDOWN  - 结束会话                                        │
│                                                                 │
│   【典型会话流程】                                               │
│                                                                 │
│   Client                                    Server              │
│      │                                         │                │
│      │────── OPTIONS ──────────────────────────►│                │
│      │◄───── 200 OK (支持的方法) ──────────────│                │
│      │                                         │                │
│      │────── DESCRIBE rtsp://...   ─────────────►│                │
│      │◄───── 200 OK (SDP内容) ─────────────────│                │
│      │                                         │                │
│      │────── SETUP (track1, RTP端口) ──────────►│                │
│      │◄───── 200 OK (服务器RTP端口) ───────────│                │
│      │                                         │                │
│      │────── SETUP (track2, RTP端口) ──────────►│                │
│      │◄───── 200 OK ───────────────────────────│                │
│      │                                         │                │
│      │────── PLAY ─────────────────────────────►│                │
│      │◄───── 200 OK ───────────────────────────│                │
│      │                                         │                │
│      │◄═══════ RTP/RTCP 数据流 ════════════════│                │
│      │                                         │                │
│      │────── TEARDOWN ─────────────────────────►│                │
│      │◄───── 200 OK ───────────────────────────│                │
│      │                                         │                │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 RTSP 请求/响应格式

```
┌─────────────────────────────────────────────────────────────────┐
│                    RTSP 消息格式                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   【请求格式】                                                   │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │ DESCRIBE rtsp://192.168.1. 1:554/live. 264 RTSP/1.0       │   │
│   │ CSeq: 2                                                 │   │
│   │ User-Agent: Live555                                     │   │
│   │ Accept: application/sdp                                 │   │
│   │                                                         │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│   【响应格式】                                                   │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │ RTSP/1.0 200 OK                                         │   │
│   │ CSeq: 2                                                 │   │
│   │ Date: Sat, 01 Jan 2025 12:00:00 GMT                     │   │
│   │ Content-Type: application/sdp                           │   │
│   │ Content-Length: 460                                     │   │
│   │                                                         │   │
│   │ v=0                                                     │   │
│   │ o=- 12345 1 IN IP4 192. 168.1.1                          │   │
│   │ s=Live Stream                                           │   │
│   │ t=0 0                                                   │   │
│   │ m=video 0 RTP/AVP 96                                    │   │
│   │ a=rtpmap:96 H264/90000                                  │   │
│   │ a=control:track1                                        │   │
│   │ ...                                                      │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 二、RTSPServer 类层次结构

### 2.1 类继承关系

```
┌─────────────────────────────────────────────────────────────────┐
│                    RTSPServer 类层次                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   Medium                                                        │
│       │                                                         │
│       ▼                                                         │
│   GenericMediaServer                                            │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │ • 通用媒体服务器基类                                    │   │
│   │ • 管理 ServerMediaSession                               │   │
│   │ • 管理 ClientConnection 和 ClientSession               │   │
│   │ • 处理 Socket 连接                                      │   │
│   └───────────────────────────┬─────────────────────────────┘   │
│                               │                                 │
│                               ▼                                 │
│   RTSPServer                                                    │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │ • RTSP 协议具体实现                                     │   │
│   │ • 处理 RTSP 命令                                        │   │
│   │ • 支持认证                                              │   │
│   │ • 支持 RTP over TCP                                     │   │
│   │ • 支持 RTSP over HTTP tunneling                         │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│   内嵌类：                                                       │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                                                         │   │
│   │   RTSPClientConnection                                  │   │
│   │   ├─ 表示一个 TCP 连接                                  │   │
│   │   ├─ 处理 OPTIONS/DESCRIBE/REGISTER 等                  │   │
│   │   └─ 可能被多个 ClientSession 共享                      │   │
│   │                                                         │   │
│   │   RTSPClientSession                                     │   │
│   │   ├─ 表示一个 RTSP 会话                                 │   │
│   │   ├─ 处理 SETUP/PLAY/PAUSE/TEARDOWN 等                  │   │
│   │   └─ 管理流状态 (streamToken)                           │   │
│   │                                                         │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 2.2 核心组件关系

```
┌─────────────────────────────────────────────────────────────────┐
│                    RTSPServer 组件关系                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   RTSPServer                                                    │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                                                         │   │
│   │   fServerMediaSessions (HashTable)                      │   │
│   │   ┌─────────────────────────────────────────────────┐   │   │
│   │   │ "stream1" ──► ServerMediaSession                │   │   │
│   │   │ "stream2" ──► ServerMediaSession                │   │   │
│   │   │ "live"    ──► ServerMediaSession                │   │   │
│   │   └─────────────────────────────────────────────────┘   │   │
│   │                                                         │   │
│   │   fClientConnections (HashTable)                        │   │
│   │   ┌─────────────────────────────────────────────────┐   │   │
│   │   │ socket1 ──► RTSPClientConnection                │   │   │
│   │   │ socket2 ──► RTSPClientConnection                │   │   │
│   │   └─────────────────────────────────────────────────┘   │   │
│   │                                                         │   │
│   │   fClientSessions (HashTable)                           │   │
│   │   ┌─────────────────────────────────────────────────┐   │   │
│   │   │ "12345678" ──► RTSPClientSession                │   │   │
│   │   │ "87654321" ──► RTSPClientSession                │   │   │
│   │   └─────────────────────────────────────────────────┘   │   │
│   │                                                         │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│   ServerMediaSession                                            │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │ fStreamName: "live"                                     │   │
│   │ fSubsessions: ──► [Subsession1] ──► [Subsession2]       │   │
│   │                     (video)           (audio)           │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 三、GenericMediaServer 类

### 3.1 类定义

```cpp
// 文件：GenericMediaServer.hh

#define REQUEST_BUFFER_SIZE 20000
#define RESPONSE_BUFFER_SIZE 20000

class GenericMediaServer: public Medium {
public:
    // 添加/管理 ServerMediaSession
    virtual void addServerMediaSession(ServerMediaSession* serverMediaSession);
    
    // 异步查找 ServerMediaSession
    virtual void lookupServerMediaSession(
        char const* streamName,
        lookupServerMediaSessionCompletionFunc* completionFunc,
        void* completionClientData,
        Boolean isFirstLookupInSession = True);
    
    // 移除 ServerMediaSession
    void removeServerMediaSession(ServerMediaSession* serverMediaSession);
    virtual void removeServerMediaSession(char const* streamName);
    
    // 关闭所有使用指定会话的客户端
    void closeAllClientSessionsForServerMediaSession(ServerMediaSession* sms);
    virtual void closeAllClientSessionsForServerMediaSession(char const* streamName);
    
    // 删除 ServerMediaSession（关闭客户端 + 移除）
    void deleteServerMediaSession(ServerMediaSession* serverMediaSession);
    virtual void deleteServerMediaSession(char const* streamName);
    
    // 客户端数量
    unsigned numClientSessions() const { return fClientSessions->numEntries(); }

protected:
    GenericMediaServer(UsageEnvironment& env, 
                       int ourSocketIPv4, int ourSocketIPv6, 
                       Port ourPort,
                       unsigned reclamationSeconds);
    virtual ~GenericMediaServer();
    
    void cleanup();  // 子类析构函数必须调用
    static int setUpOurSocket(UsageEnvironment& env, Port& ourPort, int domain);

    // 连接处理
    static void incomingConnectionHandlerIPv4(void*, int);
    static void incomingConnectionHandlerIPv6(void*, int);
    void incomingConnectionHandlerOnSocket(int serverSocket);

protected:
    // 必须由子类实现的纯虚函数
    virtual ClientConnection* createNewClientConnection(
        int clientSocket, struct sockaddr_storage const& clientAddr) = 0;
    virtual ClientSession* createNewClientSession(u_int32_t sessionId) = 0;

    // 查找 ClientSession
    ClientSession* lookupClientSession(u_int32_t sessionId);
    ClientSession* lookupClientSession(char const* sessionIdStr);

    // 获取 ServerMediaSession（同步版本）
    ServerMediaSession* getServerMediaSession(char const* streamName);

protected:
    int fServerSocketIPv4, fServerSocketIPv6;
    Port fServerPort;
    unsigned fReclamationSeconds;  // 会话超时时间
    HashTable* fServerMediaSessions;  // streamName -> ServerMediaSession
    HashTable* fClientConnections;    // socket -> ClientConnection
    HashTable* fClientSessions;       // sessionId -> ClientSession
};
```

### 3.2 ClientConnection 内嵌类

```cpp
// 文件：GenericMediaServer.hh

class GenericMediaServer::ClientConnection {
protected:
    ClientConnection(GenericMediaServer& ourServer,
                     int clientSocket, 
                     struct sockaddr_storage const& clientAddr,
                     Boolean useTLS);
    virtual ~ClientConnection();

    UsageEnvironment& envir() { return fOurServer.envir(); }
    void closeSockets();

    // 请求处理
    static void incomingRequestHandler(void*, int);
    void incomingRequestHandler();
    virtual void handleRequestBytes(int newBytesRead) = 0;  // 纯虚函数
    void resetRequestBuffer();

protected:
    GenericMediaServer& fOurServer;
    int fOurSocket;
    struct sockaddr_storage fClientAddr;
    unsigned char fRequestBuffer[REQUEST_BUFFER_SIZE];
    unsigned char fResponseBuffer[RESPONSE_BUFFER_SIZE];
    unsigned fRequestBytesAlreadySeen, fRequestBufferBytesLeft;

    // TLS 支持
    ServerTLSState fTLS;
    ServerTLSState* fInputTLS;
    ServerTLSState* fOutputTLS;
};
```

### 3.3 ClientSession 内嵌类

```cpp
// 文件：GenericMediaServer.hh

class GenericMediaServer::ClientSession {
protected:
    ClientSession(GenericMediaServer& ourServer, u_int32_t sessionId);
    virtual ~ClientSession();

    UsageEnvironment& envir() { return fOurServer.envir(); }
    
    // 活跃性检查
    void noteLiveness();
    static void noteClientLiveness(ClientSession* clientSession);
    static void livenessTimeoutTask(ClientSession* clientSession);

protected:
    GenericMediaServer& fOurServer;
    u_int32_t fOurSessionId;
    ServerMediaSession* fOurServerMediaSession;
    TaskToken fLivenessCheckTask;
};
```

---

## 四、RTSPServer 类

### 4.1 类定义

```cpp
// 文件：RTSPServer.hh

class RTSPServer: public GenericMediaServer {
public:
    static RTSPServer* createNew(UsageEnvironment& env, 
                                 Port ourPort = 554,
                                 UserAuthenticationDatabase* authDatabase = NULL,
                                 unsigned reclamationSeconds = 65);
    
    static Boolean lookupByName(UsageEnvironment& env, char const* name,
                                RTSPServer*& resultServer);

    // 生成 RTSP URL
    char* rtspURL(ServerMediaSession const* serverMediaSession,
                  int clientSocket = -1, Boolean useIPv6 = False) const;
    char* rtspURLPrefix(int clientSocket = -1, Boolean useIPv6 = False) const;

    // 认证数据库
    UserAuthenticationDatabase* setAuthenticationDatabase(
        UserAuthenticationDatabase* newDB);

    // 禁用 RTP over TCP
    void disableStreamingRTPOverTCP() { fAllowStreamingRTPOverTCP = False; }

    // RTSP over HTTP 隧道
    Boolean setUpTunnelingOverHTTP(Port httpPort);
    portNumBits httpServerPortNum() const;

    // TLS 设置
    void setTLSState(char const* certFileName, char const* privKeyFileName,
                     Boolean weServeSRTP = True, Boolean weEncryptSRTP = True);

protected:
    RTSPServer(UsageEnvironment& env,
               int ourSocketIPv4, int ourSocketIPv6, Port ourPort,
               UserAuthenticationDatabase* authDatabase,
               unsigned reclamationSeconds);
    virtual ~RTSPServer();

    // 可重写的虚函数
    virtual char const* allowedCommandNames();
    virtual UserAuthenticationDatabase* getAuthenticationDatabaseForCommand(
        char const* cmdName);
    virtual Boolean specialClientAccessCheck(int clientSocket,
                                             struct sockaddr_storage const& clientAddr,
                                             char const* urlSuffix);

public:
    virtual Boolean isRTSPServer() const;
    virtual void addServerMediaSession(ServerMediaSession* serverMediaSession);

protected:
    // 创建客户端连接/会话
    virtual ClientConnection* createNewClientConnection(
        int clientSocket, struct sockaddr_storage const& clientAddr);
    virtual ClientSession* createNewClientSession(u_int32_t sessionId);

private:
    UserAuthenticationDatabase* fAuthDB;
    Boolean fAllowStreamingRTPOverTCP;
    Boolean fOurConnectionsUseTLS;
    int fHTTPServerSocketIPv4, fHTTPServerSocketIPv6;
    Port fHTTPServerPort;
    HashTable* fClientConnectionsForHTTPTunneling;
    HashTable* fTCPStreamingDatabase;
};
```

### 4.2 RTSPClientConnection 内嵌类

```cpp
// 文件：RTSPServer.hh

class RTSPServer::RTSPClientConnection: public GenericMediaServer::ClientConnection {
protected:
    RTSPClientConnection(RTSPServer& ourServer,
                         int clientSocket, 
                         struct sockaddr_storage const& clientAddr,
                         Boolean useTLS = False);
    virtual ~RTSPClientConnection();

protected:
    // 重写的虚函数
    virtual void handleRequestBytes(int newBytesRead);

    // RTSP 命令处理（虚函数，可重写）
    virtual void handleCmd_OPTIONS();
    virtual void handleCmd_GET_PARAMETER(char const* fullRequestStr);
    virtual void handleCmd_SET_PARAMETER(char const* fullRequestStr);
    virtual void handleCmd_DESCRIBE(char const* urlPreSuffix, 
                                    char const* urlSuffix, 
                                    char const* fullRequestStr);
    virtual void handleCmd_REGISTER(char const* cmd, char const* url, 
                                    char const* urlSuffix, 
                                    char const* fullRequestStr,
                                    Boolean reuseConnection, 
                                    Boolean deliverViaTCP, 
                                    char const* proxyURLSuffix);
    virtual void handleCmd_bad();
    virtual void handleCmd_notSupported();
    virtual void handleCmd_notFound();
    virtual void handleCmd_sessionNotFound();
    virtual void handleCmd_unsupportedTransport();

    // 认证检查
    Boolean authenticationOK(char const* cmdName, char const* urlSuffix, 
                             char const* fullRequestStr);

    // 响应设置
    void setRTSPResponse(char const* responseStr);
    void setRTSPResponse(char const* responseStr, u_int32_t sessionId);
    void setRTSPResponse(char const* responseStr, char const* contentStr);
    void setRTSPResponse(char const* responseStr, u_int32_t sessionId, 
                         char const* contentStr);

protected:
    RTSPServer& fOurRTSPServer;
    int& fClientInputSocket;
    int fClientOutputSocket;
    Boolean fIsActive;
    unsigned char* fLastCRLF;
    char const* fCurrentCSeq;
    Authenticator fCurrentAuthenticator;
};
```

### 4.3 RTSPClientSession 内嵌类

```cpp
// 文件：RTSPServer.hh

class RTSPServer::RTSPClientSession: public GenericMediaServer::ClientSession {
protected:
    RTSPClientSession(RTSPServer& ourServer, u_int32_t sessionId);
    virtual ~RTSPClientSession();

    // RTSP 命令处理（虚函数，可重写）
    virtual void handleCmd_SETUP(RTSPClientConnection* ourClientConnection,
                                 char const* urlPreSuffix, 
                                 char const* urlSuffix, 
                                 char const* fullRequestStr);
    virtual void handleCmd_withinSession(RTSPClientConnection* ourClientConnection,
                                         char const* cmdName,
                                         char const* urlPreSuffix, 
                                         char const* urlSuffix,
                                         char const* fullRequestStr);
    virtual void handleCmd_TEARDOWN(RTSPClientConnection* ourClientConnection,
                                    ServerMediaSubsession* subsession);
    virtual void handleCmd_PLAY(RTSPClientConnection* ourClientConnection,
                                ServerMediaSubsession* subsession, 
                                char const* fullRequestStr);
    virtual void handleCmd_PAUSE(RTSPClientConnection* ourClientConnection,
                                 ServerMediaSubsession* subsession);
    virtual void handleCmd_GET_PARAMETER(RTSPClientConnection* ourClientConnection,
                                         ServerMediaSubsession* subsession, 
                                         char const* fullRequestStr);
    virtual void handleCmd_SET_PARAMETER(RTSPClientConnection* ourClientConnection,
                                         ServerMediaSubsession* subsession, 
                                         char const* fullRequestStr);

protected:
    RTSPServer& fOurRTSPServer;
    Boolean fIsMulticast, fStreamAfterSETUP;
    unsigned char fTCPStreamIdCount;
    unsigned fNumStreamStates;
    
    // 流状态数组
    struct streamState {
        ServerMediaSubsession* subsession;
        int tcpSocketNum;
        void* streamToken;
    } * fStreamStates;
};
```

---

## 五、ServerMediaSession 类

### 5.1 类定义

```cpp
// 文件：ServerMediaSession.hh

class ServerMediaSession: public Medium {
public:
    static ServerMediaSession* createNew(UsageEnvironment& env,
                                         char const* streamName = NULL,
                                         char const* info = NULL,
                                         char const* description = NULL,
                                         Boolean isSSM = False,
                                         char const* miscSDPLines = NULL);

    static Boolean lookupByName(UsageEnvironment& env,
                                char const* mediumName,
                                ServerMediaSession*& resultSession);

    // 生成 SDP 描述
    char* generateSDPDescription(int addressFamily);

    // 流名称
    char const* streamName() const { return fStreamName; }

    // 添加子会话
    Boolean addSubsession(ServerMediaSubsession* subsession);
    unsigned numSubsessions() const { return fSubsessionCounter; }

    // 时长相关
    void testScaleFactor(float& scale);
    float duration() const;

    // 引用计数
    unsigned referenceCount() const { return fReferenceCount; }
    void incrementReferenceCount() { ++fReferenceCount; }
    void decrementReferenceCount();
    Boolean& deleteWhenUnreferenced() { return fDeleteWhenUnreferenced; }

    // 删除所有子会话
    void deleteAllSubsessions();

    Boolean streamingUsesSRTP;      // 默认 False
    Boolean streamingIsEncrypted;   // 默认 False

protected:
    ServerMediaSession(UsageEnvironment& env, char const* streamName,
                       char const* info, char const* description,
                       Boolean isSSM, char const* miscSDPLines);
    virtual ~ServerMediaSession();

private:
    Boolean fIsSSM;
    ServerMediaSubsession* fSubsessionsHead;
    ServerMediaSubsession* fSubsessionsTail;
    unsigned fSubsessionCounter;
    char* fStreamName;
    char* fInfoSDPString;
    char* fDescriptionSDPString;
    char* fMiscSDPLines;
    struct timeval fCreationTime;
    unsigned fReferenceCount;
    Boolean fDeleteWhenUnreferenced;
};
```

### 5.2 ServerMediaSubsession 类

```cpp
// 文件：ServerMediaSession.hh

class ServerMediaSubsession: public Medium {
public:
    unsigned trackNumber() const { return fTrackNumber; }
    char const* trackId();
    
    // 纯虚函数 - 子类必须实现
    virtual char const* sdpLines(int addressFamily) = 0;
    
    virtual void getStreamParameters(
        unsigned clientSessionId,                           // in
        struct sockaddr_storage const& clientAddress,       // in
        Port const& clientRTPPort,                          // in
        Port const& clientRTCPPort,                         // in
        int tcpSocketNum,                                   // in (-1 = UDP)
        unsigned char rtpChannelId,                         // in (TCP用)
        unsigned char rtcpChannelId,                        // in (TCP用)
        TLSState* tlsState,                                 // in (TCP用)
        struct sockaddr_storage& destinationAddress,        // in out
        u_int8_t& destinationTTL,                           // in out
        Boolean& isMulticast,                               // out
        Port& serverRTPPort,                                // out
        Port& serverRTCPPort,                               // out
        void*& streamToken) = 0;                            // out
    
    virtual void startStream(
        unsigned clientSessionId, void* streamToken,
        TaskFunc* rtcpRRHandler, void* rtcpRRHandlerClientData,
        unsigned short& rtpSeqNum, unsigned& rtpTimestamp,
        ServerRequestAlternativeByteHandler* serverRequestAlternativeByteHandler,
        void* serverRequestAlternativeByteHandlerClientData) = 0;
    
    virtual void pauseStream(unsigned clientSessionId, void* streamToken);
    virtual void seekStream(unsigned clientSessionId, void* streamToken, 
                            double& seekNPT, double streamDuration, 
                            u_int64_t& numBytes);
    virtual void setStreamScale(unsigned clientSessionId, void* streamToken, 
                                float scale);
    virtual void deleteStream(unsigned clientSessionId, void*& streamToken);
    
    virtual void getRTPSinkandRTCP(void* streamToken,
                                   RTPSink*& rtpSink, 
                                   RTCPInstance*& rtcp) = 0;

    virtual void testScaleFactor(float& scale);
    virtual float duration() const;

protected:
    ServerMediaSubsession(UsageEnvironment& env);
    virtual ~ServerMediaSubsession();

    char const* rangeSDPLine() const;

protected:
    ServerMediaSession* fParentSession;

private:
    ServerMediaSubsession* fNext;
    unsigned fTrackNumber;
    char const* fTrackId;
};
```

---

## 六、RTSP 请求处理流程

### 6.1 整体流程

```
┌─────────────────────────────────────────────────────────────────┐
│                    RTSP 请求处理流程                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   客户端连接                                                    │
│       │                                                         │
│       ▼                                                         │
│   incomingConnectionHandlerOnSocket()                           │
│       │                                                         │
│       │  创建 RTSPClientConnection                              │
│       │  注册 socket 可读事件                                   │
│       ▼                                                         │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │            incomingRequestHandler()                     │   │
│   │                      │                                  │   │
│   │                      ▼                                  │   │
│   │            handleRequestBytes()                         │   │
│   │                      │                                  │   │
│   │         ┌────────────┼────────────┐                     │   │
│   │         │            │            │                     │   │
│   │         ▼            ▼            ▼                     │   │
│   │    OPTIONS      DESCRIBE      SETUP                     │   │
│   │    handleCmd_   handleCmd_   handleCmd_                 │   │
│   │    OPTIONS()    DESCRIBE()   SETUP()                    │   │
│   │                                   │                     │   │
│   │                                   ▼                     │   │
│   │                          创建/查找                      │   │
│   │                          RTSPClientSession              │   │
│   │                                   │                     │   │
│   │    ┌──────────────────────────────┘                     │   │
│   │    │                                                    │   │
│   │    ▼                                                    │   │
│   │  PLAY / PAUSE / TEARDOWN                                │   │
│   │  handleCmd_PLAY() / handleCmd_PAUSE()                   │   │
│   │  handleCmd_TEARDOWN()                                   │   │
│   │                                                         │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

客户端请求 (SETUP)
    |
    ▼
incomingRequestHandler() -> handleRequestBytes()
    |
    ▼
handleCmd_SETUP()  <-- 【关键入口】
    |
    +--- Step 1: 检查 URL 是否带 Session ID？
    |     |-- 是: 查找现有的 RTSPClientSession
    |     `-- 否: 创建新的 RTSPClientSession (new RTSPClientSession)
    |
    +--- Step 2: 查找 ServerMediaSession (资源) 和 Subsession (轨道)
    |
    +--- Step 3: 【核心】调用 subsession->getStreamParameters()
    |     |
    |     +-- 创建 Source (ByteStreamFileSource + H264Framer)
    |     `-- 创建 Sink (H264VideoRTPSink)
    |
    +--- Step 4: 将 Source/Sink 存入 RTSPClientSession 的 StreamState 中
    |
    ▼
返回 200 OK (带 Session ID)
    |
    | (时间流逝...)
    ▼
客户端请求 (PLAY)
    |
    ▼
handleCmd_PLAY()
    |
    +--- Step 1: 根据 Session ID 找到 RTSPClientSession
    |
    +--- Step 2: 遍历内部所有的 StreamState
    |
    +--- Step 3: 调用 streamState.sink->startPlaying()  <-- 【点火】
    |
    ▼
数据流开始传输
```

### 6.2 DESCRIBE 命令处理

```
┌─────────────────────────────────────────────────────────────────┐
│                    DESCRIBE 处理流程                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   handleCmd_DESCRIBE()                                          │
│        │                                                        │
│        │  ① 解析 URL，提取 streamName                           │
│        ▼                                                        │
│   lookupServerMediaSession(streamName, callback)                │
│        │                                                        │
│        │  ② 异步查找 ServerMediaSession                         │
│        ▼                                                        │
│   DESCRIBELookupCompletionFunction()                            │
│        │                                                        │
│        │  ③ 回调                                                │
│        ▼                                                        │
│   handleCmd_DESCRIBE_afterLookup()                              │
│        │                                                        │
│        │  ④ 检查 session 是否存在                               │
│        │     如果不存在：handleCmd_notFound()                   │
│        │                                                        │
│        │  ⑤ 生成 SDP                                            │
│        │     session->generateSDPDescription()                  │
│        │                                                        │
│        │  ⑥ 构造响应                                            │
│        │     "RTSP/1.0 200 OK\r\n"                              │
│        │     "Content-Type: application/sdp\r\n"                │
│        │     "Content-Length: xxx\r\n"                          │
│        │     "\r\n"                                             │
│        │     "<SDP内容>"                                        │
│        ▼                                                        │
│   发送响应                                                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 6.3 SETUP 命令处理

```
┌─────────────────────────────────────────────────────────────────┐
│                    SETUP 处理流程                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   RTSPClientSession::handleCmd_SETUP()                          │
│        │                                                        │
│        │  ① 解析 Transport 头                                   │
│        │     - 客户端 RTP/RTCP 端口                             │
│        │     - 传输模式（UDP/TCP）                              │
│        ▼                                                        │
│   lookupServerMediaSession()                                    │
│        │                                                        │
│        │  ② 查找 ServerMediaSession                             │
│        ▼                                                        │
│   查找 ServerMediaSubsession（根据 trackId）                    │
│        │                                                        │
│        │  ③ 调用 subsession->getStreamParameters()              │
│        │     - 创建 RTPSink / RTCPInstance                      │
│        │     - 分配服务器端口                                   │
│        │     - 返回 streamToken                                 │
│        ▼                                                        │
│   保存 streamState                                              │
│        │                                                        │
│        │  ④ 构造响应                                            │
│        │     "RTSP/1.0 200 OK\r\n"                              │
│        │     "Session: <sessionId>\r\n"                         │
│        │     "Transport: RTP/AVP;unicast;"                      │
│        │     "client_port=xxx-xxx;server_port=yyy-yyy\r\n"      │
│        ▼                                                        │
│   发送响应                                                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 6.4 PLAY 命令处理

```
┌─────────────────────────────────────────────────────────────────┐
│                    PLAY 处理流程                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   RTSPClientSession::handleCmd_PLAY()                           │
│        │                                                        │
│        │  ① 解析 Range 头（可选）                               │
│        │     - 起始时间 (npt=0-)                                │
│        │     - 结束时间                                         │
│        ▼                                                        │
│   遍历所有 streamState                                          │
│        │                                                        │
│        │  ② 对每个子会话调用                                    │
│        │     subsession->seekStream() (如果需要)                │
│        │     subsession->startStream()                          │
│        │        │                                               │
│        │        └──► 启动 RTPSink::startPlaying()               │
│        │             启动 RTCPInstance                          │
│        ▼                                                        │
│   构造响应                                                       │
│        │                                                        │
│        │  ③ "RTSP/1.0 200 OK\r\n"                               │
│        │     "Session: <sessionId>\r\n"                         │
│        │     "RTP-Info: url=. ../track1;seq=xxx;rtptime=yyy,"    │
│        │     "url=.../track2;seq=xxx;rtptime=yyy\r\n"           │
│        ▼                                                        │
│   发送响应                                                       │
│        │                                                        │
│        ▼                                                        │
│   RTP/RTCP 数据开始流向客户端                                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 七、UserAuthenticationDatabase 类

### 7.1 类定义

```cpp
// 文件：GenericMediaServer.hh

class UserAuthenticationDatabase {
public:
    UserAuthenticationDatabase(char const* realm = NULL,
                               Boolean passwordsAreMD5 = False);
    virtual ~UserAuthenticationDatabase();

    // 添加/删除用户
    virtual void addUserRecord(char const* username, char const* password);
    virtual void removeUserRecord(char const* username);

    // 查找密码
    virtual char const* lookupPassword(char const* username);

    // 获取认证域
    char const* realm() { return fRealm; }
    Boolean passwordsAreMD5() { return fPasswordsAreMD5; }

protected:
    HashTable* fTable;
    char* fRealm;
    Boolean fPasswordsAreMD5;
};
```

### 7.2 认证流程

```
┌─────────────────────────────────────────────────────────────────┐
│                    Digest 认证流程                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   Client                                    Server              │
│      │                                         │                │
│      │────── DESCRIBE (无认证) ────────────────►│                │
│      │                                         │                │
│      │◄───── 401 Unauthorized ─────────────────│                │
│      │       WWW-Authenticate: Digest          │                │
│      │       realm="Live555",                  │                │
│      │       nonce="abc123..."                 │                │
│      │                                         │                │
│      │  计算 response:                         │                │
│      │  HA1 = MD5(username:realm:password)     │                │
│      │  HA2 = MD5(method:uri)                  │                │
│      │  response = MD5(HA1:nonce:HA2)          │                │
│      │                                         │                │
│      │────── DESCRIBE ─────────────────────────►│                │
│      │       Authorization: Digest             │                │
│      │       username="user",                  │                │
│      │       realm="Live555",                  │                │
│      │       nonce="abc123.. .",                │                │
│      │       uri="rtsp://.. .",                 │                │
│      │       response="xyz789..."              │                │
│      │                                         │                │
│      │                               验证 response               │
│      │                                         │                │
│      │◄───── 200 OK (SDP) ─────────────────────│                │
│      │                                         │                │
└─────────────────────────────────────────────────────────────────┘
```

---

## 八、使用示例

### 8.1 创建基本的 RTSP 服务器

```cpp
#include "liveMedia.hh"
#include "BasicUsageEnvironment.hh"

int main() {
    // 1. 创建使用环境
    TaskScheduler* scheduler = BasicTaskScheduler::createNew();
    UsageEnvironment* env = BasicUsageEnvironment::createNew(*scheduler);
    
    // 2.  创建 RTSP 服务器
    RTSPServer* rtspServer = RTSPServer::createNew(*env, 554);
    if (rtspServer == NULL) {
        *env << "Failed to create RTSP server: " << env->getResultMsg() << "\n";
        return 1;
    }
    
    // 3. 创建 ServerMediaSession
    char const* streamName = "live";
    ServerMediaSession* sms = ServerMediaSession::createNew(
        *env, 
        streamName, 
        streamName,  // info
        "Live Stream"  // description
    );
    
    // 4. 添加 H. 264 视频子会话
    sms->addSubsession(H264VideoFileServerMediaSubsession::createNew(
        *env, 
        "test.264",  // 文件名
        False        // reuseFirstSource
    ));
    
    // 5. 将会话添加到服务器
    rtspServer->addServerMediaSession(sms);
    
    // 6.  打印 URL
    char* url = rtspServer->rtspURL(sms);
    *env << "Play this stream using: " << url << "\n";
    delete[] url;
    
    // 7. 进入事件循环
    env->taskScheduler(). doEventLoop();
    
    return 0;
}
```

### 8.2 添加用户认证

```cpp
// 创建认证数据库
UserAuthenticationDatabase* authDB = new UserAuthenticationDatabase("Live555");
authDB->addUserRecord("admin", "password123");
authDB->addUserRecord("user", "user456");

// 创建带认证的 RTSP 服务器
RTSPServer* rtspServer = RTSPServer::createNew(
    *env, 
    554,           // 端口
    authDB,        // 认证数据库
    65             // 超时时间（秒）
);
```

### 8.3 添加多个媒体轨道

```cpp
ServerMediaSession* sms = ServerMediaSession::createNew(
    *env, "movie", "movie", "Video and Audio Stream"
);

// 添加视频轨道
sms->addSubsession(H264VideoFileServerMediaSubsession::createNew(
    *env, "video.264", False
));

// 添加音频轨道
sms->addSubsession(ADTSAudioFileServerMediaSubsession::createNew(
    *env, "audio.aac", False
));

rtspServer->addServerMediaSession(sms);
```

### 8.4 自定义子会话（推流）

```cpp
class MyLiveServerMediaSubsession: public OnDemandServerMediaSubsession {
public:
    static MyLiveServerMediaSubsession* createNew(UsageEnvironment& env) {
        return new MyLiveServerMediaSubsession(env);
    }

protected:
    MyLiveServerMediaSubsession(UsageEnvironment& env)
        : OnDemandServerMediaSubsession(env, True /* reuseFirstSource */) {
    }
    
    virtual FramedSource* createNewStreamSource(unsigned clientSessionId,
                                                unsigned& estBitrate) {
        estBitrate = 500;  // kbps
        
        // 创建自定义的数据源
        MyDeviceSource* source = MyDeviceSource::createNew(envir());
        
        // 包装成 H.264 帧
        return H264VideoStreamFramer::createNew(envir(), source);
    }
    
    virtual RTPSink* createNewRTPSink(Groupsock* rtpGroupsock,
                                      unsigned char rtpPayloadTypeIfDynamic,
                                      FramedSource* inputSource) {
        return H264VideoRTPSink::createNew(envir(), rtpGroupsock, 
                                           rtpPayloadTypeIfDynamic);
    }
};

// 使用
ServerMediaSession* sms = ServerMediaSession::createNew(*env, "live");
sms->addSubsession(MyLiveServerMediaSubsession::createNew(*env));
rtspServer->addServerMediaSession(sms);
```

---

## 九、小结

### 9.1 本节要点

1. **GenericMediaServer** 是通用服务器基类，管理会话和连接
2. **RTSPServer** 实现 RTSP 协议，处理各种命令
3. **RTSPClientConnection** 处理单个 TCP 连接，响应 OPTIONS/DESCRIBE
4. **RTSPClientSession** 管理 RTSP 会话，处理 SETUP/PLAY/TEARDOWN
5. **ServerMediaSession** 描述一个流媒体资源，包含多个子会话
6. **ServerMediaSubsession** 描述单个媒体轨道（音频/视频）

### 9.2 核心类速查表

| 类                           | 功能               |
| ---------------------------- | ------------------ |
| `GenericMediaServer`         | 通用媒体服务器基类 |
| `RTSPServer`                 | RTSP 服务器实现    |
| `RTSPClientConnection`       | 单个 TCP 连接      |
| `RTSPClientSession`          | RTSP 会话          |
| `ServerMediaSession`         | 流媒体资源描述     |
| `ServerMediaSubsession`      | 单个媒体轨道       |
| `UserAuthenticationDatabase` | 用户认证数据库     |

### 9.3 RTSP 命令处理速查表

| 命令          | 处理类               | 处理函数                    |
| ------------- | -------------------- | --------------------------- |
| OPTIONS       | RTSPClientConnection | `handleCmd_OPTIONS()`       |
| DESCRIBE      | RTSPClientConnection | `handleCmd_DESCRIBE()`      |
| SETUP         | RTSPClientSession    | `handleCmd_SETUP()`         |
| PLAY          | RTSPClientSession    | `handleCmd_PLAY()`          |
| PAUSE         | RTSPClientSession    | `handleCmd_PAUSE()`         |
| TEARDOWN      | RTSPClientSession    | `handleCmd_TEARDOWN()`      |
| GET_PARAMETER | 两者                 | `handleCmd_GET_PARAMETER()` |
| SET_PARAMETER | 两者                 | `handleCmd_SET_PARAMETER()` |

### 9.4 类关系图

```
┌─────────────────────────────────────────────────────────────────┐
│                    RTSPServer 类关系图                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   Medium                                                        │
│       │                                                         │
│       ▼                                                         │
│   GenericMediaServer                                            │
│       │    ┌────────────────────────┐                           │
│       │    │ ClientConnection       │                           │
│       │    │ ClientSession          │                           │
│       │    └────────────────────────┘                           │
│       ▼                                                         │
│   RTSPServer                                                    │
│       │    ┌────────────────────────┐                           │
│       │    │ RTSPClientConnection   │──► handleCmd_OPTIONS()    │
│       │    │                        │──► handleCmd_DESCRIBE()   │
│       │    │ RTSPClientSession      │──► handleCmd_SETUP()      │
│       │    │                        │──► handleCmd_PLAY()       │
│       │    └────────────────────────┘                           │
│       │                                                         │
│       │ uses                                                    │
│       ▼                                                         │
│   ServerMediaSession                                            │
│       │                                                         │
│       │ contains                                                │
│       ▼                                                         │
│   ServerMediaSubsession ──► [Subsession1] ──► [Subsession2]     │
│       │                        (video)          (audio)         │
│       │ creates                                                 │
│       ▼                                                         │
│   RTPSink + RTCPInstance                                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---





```
客户端           RTSPServer              RTSPClientConnection         RTSPClientSession            ServerMediaSession/Subsession              媒体链路
  |  TCP连接  →  (监听socket)   →   新建 Connection 对象
  |  DESCRIBE → handleRequestBytes()
                parseRequestString -> handleCmd_DESCRIBE()
                      |                       |
                      | lookupServerMediaSession(urlSuffix)
                      v                       |
              (DynamicRTSPServer::createNewSMS)  -- 创建 SMS + H264 Subsession
                      |
              session->generateSDPDescription()
                      |
                      v
           <------     回复 SDP 描述     ------


  |  SETUP track1 → handleRequestBytes()
                     -> (会话层) handleCmd_SETUP()
                          |
                          v
                subsession->getStreamParameters()
                     |   // 创建 ByteStreamFileSource
                     |   // 创建 H264VideoStreamFramer
                     |   // 创建 Groupsock(RTP/RTCP)
                     |   // 创建 H264VideoRTPSink
                     v
                 保存 streamToken 到 fStreamStates[track1]
           <------   回复 SETUP OK   ------


  |  PLAY      → handleRequestBytes()
                  -> handleCmd_withinSession("PLAY")
                        |
                        v
         对每个 track 调 subsession->startStream(streamToken)
                        |
                        v
         sink->startPlaying(*source, ...)  // 触发数据推流
                        |
         ByteStreamFileSource -> H264VideoStreamFramer -> H264VideoRTPSink -> UDP(RTP)


  |  TEARDOWN → handleRequestBytes()
                  -> handleCmd_withinSession("TEARDOWN")
                        |
                        v
              subsession->deleteStream(streamToken)
                        |
                        v
          关闭 sink/source/RTP socket，清理 session 状态
```





```
RTSPServer
  └─ 哈希表: urlSuffix -> ServerMediaSession (会话级：一条 URL，一个 SDP)
          └─ subsessions[] : ServerMediaSubsession (每个 track，一条 m= 行 + 源/汇工厂)
                 ├─ createNewStreamSource() → FramedSource (ByteStreamFileSource/H264VideoStreamFramer/摄像头...)
                 └─ createNewRTPSink()      → RTPSink     (H264VideoRTPSink/AACAudioRTPSink/...)
```

