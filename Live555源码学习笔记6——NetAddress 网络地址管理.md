# Live555源码学习笔记6——NetAddress 网络地址管理

## 一、模块概述

### 1.1 NetAddress 模块的作用

NetAddress 模块负责**网络地址的表示、解析和管理**，是 groupsock 网络层的基础设施。它封装了IPv4和IPv6地址的操作，提供了统一的接口。

```c++
┌─────────────────────────────────────────────────────────────────┐
│                  NetAddress 模块在系统中的位置                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   liveMedia 层                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │ RTSPServer / RTSPClient / RTPSink / RTPSource           │   │
│   │     需要处理网络地址（客户端IP、组播地址等）             │   │
│   └─────────────────────────────────────────────────────────┘   │
│                              │                                  │
│                              ▼                                  │
│   groupsock 层 - Groupsock                                      │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │ 需要管理目标地址、组播地址                              │   │
│   └─────────────────────────────────────────────────────────┘   │
│                              │                                  │
│                              ▼                                  │
│   NetAddress 模块                                    │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                                                         │   │
│   │   NetAddress      - IP地址封装（IPv4/IPv6）             │   │
│   │   NetAddressList  - 地址列表（DNS解析结果）             │   │
│   │   Port            - 端口号封装                          │   │
│   │   AddressString   - IP地址转字符串                      │   │
│   │   AddressPortLookupTable - 地址端口查找表               │   │
│   │                                                         │   │
│   └─────────────────────────────────────────────────────────┘   │
│                              │                                  │
│                              ▼                                  │
│   操作系统网络 API                                              │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │ sockaddr_in / sockaddr_in6 / sockaddr_storage           │   │
│   │ inet_pton() / inet_ntop() / getaddrinfo()               │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 文件结构

```
groupsock/
├── include/
│   ├── NetAddress. hh      ⭐重点
│   └── NetCommon.h        网络通用定义
└── NetAddress.cpp         实现文件
```

### 1.3 为什么需要封装网络地址？

```
┌─────────────────────────────────────────────────────────────────┐
│                   网络地址封装的必要性                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   【原生API的问题】                                              │
│                                                                 │
│   1.  IPv4 和 IPv6 使用不同的结构体：                            │
│      - IPv4: struct sockaddr_in  (16字节)                       │
│      - IPv6: struct sockaddr_in6 (28字节)                       │
│                                                                 │
│   2. 字节序问题：                                               │
│      - 网络字节序 (Big-Endian)                                  │
│      - 主机字节序 (可能是 Little-Endian)                        │
│                                                                 │
│   3. 地址字符串解析：                                           │
│      - "192.168.1.1" → 二进制                                   │
│      - "fe80::1" → 二进制                                       │
│                                                                 │
│   4. DNS 解析返回多个地址：                                     │
│      - 需要管理地址列表                                         │
│                                                                 │
│   【Live555 的解决方案】                                         │
│                                                                 │
│   - NetAddress: 统一封装 IPv4/IPv6 地址                         │
│   - Port: 处理端口号的字节序转换                                │
│   - NetAddressList: 管理 DNS 解析返回的多个地址                 │
│   - AddressString: 将地址转为可读字符串                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 二、基础类型定义

### 2.1 NetCommon.h 中的类型

```cpp
// 文件：groupsock/include/NetCommon. h

// IPv4 地址类型（32位）
typedef u_int32_t ipv4AddressBits;

// IPv6 地址类型（128位）
typedef u_int8_t ipv6AddressBits[16];

// 端口号类型（16位）
typedef u_int16_t portNumBits;
```

### 2.2 sockaddr 结构体回顾

```
┌─────────────────────────────────────────────────────────────────┐
│                   sockaddr 结构体家族                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   【通用结构】                                                   │
│   struct sockaddr {                                             │
│       sa_family_t sa_family;   // 地址族 (AF_INET/AF_INET6)     │
│       char sa_data[14];        // 地址数据                      │
│   };                                                            │
│                                                                 │
│   【IPv4 结构】                                                  │
│   struct sockaddr_in {                                          │
│       sa_family_t    sin_family;   // AF_INET                   │
│       in_port_t      sin_port;     // 端口号（网络字节序）      │
│       struct in_addr sin_addr;     // IPv4地址                  │
│       char           sin_zero[8];  // 填充                      │
│   };                                                            │
│                                                                 │
│   【IPv6 结构】                                                  │
│   struct sockaddr_in6 {                                         │
│       sa_family_t     sin6_family;   // AF_INET6                │
│       in_port_t       sin6_port;     // 端口号（网络字节序）    │
│       uint32_t        sin6_flowinfo; // 流信息                  │
│       struct in6_addr sin6_addr;     // IPv6地址（128位）       │
│       uint32_t        sin6_scope_id; // 作用域ID                │
│   };                                                            │
│                                                                 │
│   【存储结构 - 可容纳任何地址】                                  │
│   struct sockaddr_storage {                                     │
│       sa_family_t ss_family;                                    │
│       // 足够大的空间存储 IPv4 或 IPv6                          │
│       char __ss_padding[128 - sizeof(sa_family_t)];             │
│   };                                                            │
│                                                                 │
│   💡 Live555 主要使用 sockaddr_storage 来统一处理地址           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 三、NetAddress 类

### 3.1 类定义

```cpp
// 文件：NetAddress.hh

class NetAddress {
public:
    // 构造函数
    NetAddress(u_int8_t const* data,
               unsigned length = 4 /* 默认IPv4(4字节); IPv6用16 */);
    NetAddress(unsigned length = 4);  // 全零地址
    NetAddress(NetAddress const& orig);  // 拷贝构造
    
    // 赋值运算符
    NetAddress& operator=(NetAddress const& rightSide);
    
    virtual ~NetAddress();
    
    // 访问器
    unsigned length() const { return fLength; }
    u_int8_t const* data() const { return fData; }  // 网络字节序
    
private:
    void assign(u_int8_t const* data, unsigned length);
    void clean();
    
    unsigned fLength;    // 地址长度：4(IPv4) 或 16(IPv6)
    u_int8_t* fData;     // 地址数据（动态分配）
};
```

### 3.2 实现详解

```cpp
// 文件：NetAddress. cpp

// 从数据构造
NetAddress::NetAddress(u_int8_t const* data, unsigned length) {
    assign(data, length);
}

// 构造全零地址
NetAddress::NetAddress(unsigned length) {
    fData = new u_int8_t[length];
    if (fData == NULL) {
        fLength = 0;
        return;
    }
    
    // 初始化为全零
    for (unsigned i = 0; i < length; ++i) fData[i] = 0;
    fLength = length;
}

// 拷贝构造
NetAddress::NetAddress(NetAddress const& orig) {
    assign(orig.data(), orig. length());
}

// 赋值运算符
NetAddress& NetAddress::operator=(NetAddress const& rightSide) {
    if (&rightSide != this) {  // 防止自赋值
        clean();
        assign(rightSide.data(), rightSide. length());
    }
    return *this;
}

// 析构函数
NetAddress::~NetAddress() {
    clean();
}

// 分配并复制数据
void NetAddress::assign(u_int8_t const* data, unsigned length) {
    fData = new u_int8_t[length];
    if (fData == NULL) {
        fLength = 0;
        return;
    }
    
    for (unsigned i = 0; i < length; ++i) fData[i] = data[i];
    fLength = length;
}

// 清理
void NetAddress::clean() {
    delete[] fData;
    fData = NULL;
    fLength = 0;
}
```

### 3.3 使用示例

```cpp
// 创建IPv4地址
u_int8_t ipv4Data[4] = {192, 168, 1, 100};
NetAddress addr4(ipv4Data, 4);

// 创建IPv6地址
u_int8_t ipv6Data[16] = {0xfe, 0x80, 0, 0, 0, 0, 0, 0, 
                          0, 0, 0, 0, 0, 0, 0, 1};
NetAddress addr6(ipv6Data, 16);

// 创建全零地址
NetAddress zeroAddr4(4);   // 0.0.0. 0
NetAddress zeroAddr6(16);  // ::

// 拷贝地址
NetAddress addrCopy = addr4;
```

---

## 四、Port类

### 4.1 类定义

```cpp
// 文件：NetAddress.hh

typedef u_int16_t portNumBits;

class Port {
public:
    Port(portNumBits num /* 主机字节序 */);
    
    portNumBits num() const { return fPortNum; }  // 返回网络字节序

private:
    portNumBits fPortNum;  // 存储为网络字节序
};
```

### 4.2 实现

```cpp
// 文件：NetAddress.cpp

Port::Port(portNumBits num /* 主机字节序 */) {
    fPortNum = htons(num);  // 转换为网络字节序存储
}

// 输出运算符重载
UsageEnvironment& operator<<(UsageEnvironment& s, const Port& p) {
    return s << ntohs(p. num());  // 转换回主机字节序输出
}
```

### 4. 3 字节序说明

```
┌─────────────────────────────────────────────────────────────────┐
│                       字节序（Endianness）                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   端口号 8080 (十六进制: 0x1F90)                                │
│                                                                 │
│   【大端序 Big-Endian】（网络字节序）                           │
│   高字节在前，低字节在后                                        │
│   内存: [0x1F] [0x90]                                           │
│          低地址  高地址                                          │
│                                                                 │
│   【小端序 Little-Endian】（x86/x64主机常用）                   │
│   低字节在前，高字节在后                                        │
│   内存: [0x90] [0x1F]                                           │
│          低地址  高地址                                          │
│                                                                 │
│   【转换函数】                                                   │
│   htons() - Host TO Network Short (16位)                        │
│   htonl() - Host TO Network Long (32位)                         │
│   ntohs() - Network TO Host Short (16位)                        │
│   ntohl() - Network TO Host Long (32位)                         │
│                                                                 │
│   💡 Port类自动处理字节序转换，使用者无需关心                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 4.4 使用示例

```cpp
// 创建端口对象（传入主机字节序）
Port rtspPort(554);
Port rtpPort(5000);

// 获取端口号（返回网络字节序）
portNumBits portInNetworkOrder = rtspPort.num();

// 用于socket操作
struct sockaddr_in addr;
addr.sin_port = rtspPort. num();  // 直接使用，已是网络字节序

// 输出调试
env << "RTSP Port: " << rtspPort << "\n";  // 输出 "554"
```

---

## 五、NetAddressList类

### 5.1 类定义

NetAddressList 用于存储DNS解析返回的多个地址：

```cpp
// 文件：NetAddress.hh

class NetAddressList {
public:
    // 从主机名构造（执行DNS解析）
    NetAddressList(char const* hostname, int addressFamily = AF_UNSPEC);
    
    NetAddressList(NetAddressList const& orig);
    NetAddressList& operator=(NetAddressList const& rightSide);
    virtual ~NetAddressList();
    
    // 获取地址数量
    unsigned numAddresses() const { return fNumAddresses; }
    
    // 获取第一个地址
    NetAddress const* firstAddress() const;
    
    // 迭代器 - 用于遍历所有地址
    class Iterator {
    public:
        Iterator(NetAddressList const& addressList);
        NetAddress const* nextAddress();  // 返回NULL表示结束
    private:
        NetAddressList const& fAddressList;
        unsigned fNextIndex;
    };
    
private:
    void assign(unsigned numAddresses, NetAddress** addressArray);
    void clean();
    
    friend class Iterator;
    unsigned fNumAddresses;
    NetAddress** fAddressArray;  // 地址指针数组
};
```

### 5.2 构造函数实现（DNS解析）

```cpp
// 文件：NetAddress.cpp

NetAddressList::NetAddressList(char const* hostname, int addressFamily)
    : fNumAddresses(0), fAddressArray(NULL) {
    if (hostname == NULL) return;

    // 首先检查是否是IP地址字符串（而不是主机名）
    
    // 尝试解析为IPv4地址
    ipv4AddressBits addr4;
    if (addressFamily != AF_INET6 && 
        inet_pton(AF_INET, hostname, (u_int8_t*)&addr4) == 1) {
        // 成功解析为IPv4地址
        fNumAddresses = 1;
        fAddressArray = new NetAddress*[fNumAddresses];
        fAddressArray[0] = new NetAddress((u_int8_t*)&addr4, sizeof addr4);
        return;
    }

    // 尝试解析为IPv6地址
    ipv6AddressBits addr6;
    if (addressFamily != AF_INET && 
        inet_pton(AF_INET6, hostname, (u_int8_t*)&addr6) == 1) {
        // 成功解析为IPv6地址
        fNumAddresses = 1;
        fAddressArray = new NetAddress*[fNumAddresses];
        fAddressArray[0] = new NetAddress((u_int8_t*)&addr6, sizeof addr6);
        return;
    }
    
    // 不是IP地址字符串，尝试DNS解析
    // 使用 getaddrinfo() 函数
    struct addrinfo addrinfoHints;
    memset(&addrinfoHints, 0, sizeof addrinfoHints);
    addrinfoHints.ai_flags = AI_ADDRCONFIG;
    
    struct addrinfo* addrinfoResultPtr = NULL;
    int result = -1;
    
    // 首先尝试IPv4
    if (addressFamily != AF_INET6) {
        addrinfoHints.ai_family = AF_INET;
        result = getaddrinfo(hostname, NULL, &addrinfoHints, &addrinfoResultPtr);
    }
    
    // 如果IPv4失败，尝试IPv6
    if (addressFamily != AF_INET && (result != 0 || addrinfoResultPtr == NULL)) {
        addrinfoHints.ai_family = AF_INET6;
        result = getaddrinfo(hostname, NULL, &addrinfoHints, &addrinfoResultPtr);
    }
    
    if (result != 0 || addrinfoResultPtr == NULL) return;  // 解析失败

    // 第一遍：计算地址数量
    const struct addrinfo* p = addrinfoResultPtr;
    while (p != NULL) {
        if (p->ai_family == AF_INET || p->ai_family == AF_INET6) {
            ++fNumAddresses;
        }
        p = p->ai_next;
    }

    // 分配数组
    fAddressArray = new NetAddress*[fNumAddresses];
    
    // 第二遍：填充地址数组
    unsigned i = 0;
    p = addrinfoResultPtr;
    while (p != NULL) {
        if (p->ai_family == AF_INET) {
            // IPv4地址
            struct sockaddr_in* addr = (struct sockaddr_in*)p->ai_addr;
            fAddressArray[i++] = new NetAddress(
                (u_int8_t const*)&addr->sin_addr. s_addr, 
                sizeof(ipv4AddressBits)
            );
        } else if (p->ai_family == AF_INET6) {
            // IPv6地址
            struct sockaddr_in6* addr = (struct sockaddr_in6*)p->ai_addr;
            fAddressArray[i++] = new NetAddress(
                (u_int8_t const*)&addr->sin6_addr. s6_addr, 
                sizeof(ipv6AddressBits)
            );
        }
        p = p->ai_next;
    }

    // 释放getaddrinfo分配的内存
    freeaddrinfo(addrinfoResultPtr);
}
```

### 5.3 迭代器实现

```cpp
// 文件：NetAddress.cpp

NetAddressList::Iterator::Iterator(NetAddressList const& addressList)
    : fAddressList(addressList), fNextIndex(0) {
}

NetAddress const* NetAddressList::Iterator::nextAddress() {
    if (fNextIndex >= fAddressList.numAddresses()) {
        return NULL;  // 没有更多地址了
    }
    return fAddressList. fAddressArray[fNextIndex++];
}
```

### 5.4 使用示例

```cpp
// DNS解析主机名
NetAddressList addressList("www.example.com");

if (addressList.numAddresses() == 0) {
    env << "DNS resolution failed!\n";
    return;
}

env << "Found " << addressList. numAddresses() << " addresses:\n";

// 使用迭代器遍历所有地址
NetAddressList::Iterator iter(addressList);
NetAddress const* addr;
while ((addr = iter.nextAddress()) != NULL) {
    if (addr->length() == 4) {
        // IPv4地址
        u_int8_t const* data = addr->data();
        env << "  IPv4: " << (int)data[0] << "." << (int)data[1] << "."
            << (int)data[2] << "." << (int)data[3] << "\n";
    } else {
        env << "  IPv6 address\n";
    }
}

// 或者只使用第一个地址
NetAddress const* firstAddr = addressList.firstAddress();
```

---

## 六、AddressString类

### 6.1 类定义

AddressString 将 IP 地址转换为可读的字符串格式：

```cpp
// 文件：NetAddress.hh

class AddressString {
public:
    // 从各种格式的IPv4地址构造
    AddressString(struct sockaddr_in const& addr);
    AddressString(struct in_addr const& addr);
    AddressString(ipv4AddressBits const& addr);

    // 从各种格式的IPv6地址构造
    AddressString(struct sockaddr_in6 const& addr);
    AddressString(struct in6_addr const& addr);
    AddressString(ipv6AddressBits const& addr);

    // 从通用sockaddr_storage构造
    AddressString(struct sockaddr_storage const& addr);

    virtual ~AddressString();

    // 获取字符串
    char const* val() const { return fVal; }

private:
    void init(ipv4AddressBits const& addr);  // IPv4初始化
    void init(ipv6AddressBits const& addr);  // IPv6初始化

    char* fVal;  // 结果字符串
};
```

### 6.2 实现

```cpp
// 文件：NetAddress.cpp

// 从sockaddr_storage构造
AddressString::AddressString(struct sockaddr_storage const& addr) {
    switch (addr.ss_family) {
        case AF_INET: {
            init(((sockaddr_in&)addr).sin_addr. s_addr);
            break;
        }
        case AF_INET6: {
            init(((sockaddr_in6&)addr). sin6_addr. s6_addr);
            break;
        }
        default: {
            fVal = new char[200];
            sprintf(fVal, "(unknown address family %d)", addr.ss_family);
            break;
        }
    }
}

// IPv4初始化
void AddressString::init(ipv4AddressBits const& addr) {
    fVal = new char[INET_ADDRSTRLEN];  // 16字节足够
    inet_ntop(AF_INET, &addr, fVal, INET_ADDRSTRLEN);
}

// IPv6初始化
void AddressString::init(ipv6AddressBits const& addr) {
    fVal = new char[INET6_ADDRSTRLEN];  // 46字节足够
    inet_ntop(AF_INET6, &addr, fVal, INET6_ADDRSTRLEN);
}

AddressString::~AddressString() {
    delete[] fVal;
}
```

### 6.3 使用示例

```cpp
// 从sockaddr_in转换
struct sockaddr_in addr4;
addr4.sin_family = AF_INET;
inet_pton(AF_INET, "192.168.1.100", &addr4. sin_addr);

AddressString str4(addr4);
env << "IPv4 Address: " << str4.val() << "\n";  // 输出 "192.168.1.100"

// 从sockaddr_storage转换
struct sockaddr_storage storageAddr;
// ... 填充地址 ...
AddressString strStorage(storageAddr);
env << "Address: " << strStorage.val() << "\n";

// 在调试输出中常用
void logConnection(struct sockaddr_storage const& clientAddr) {
    env << "Client connected from " << AddressString(clientAddr).val() << "\n";
}
```

---

## 七、AddressPortLookupTable类

### 7.1 类定义

这是一个用于按（地址1, 地址2, 端口）查找对象的哈希表：

```cpp
// 文件：NetAddress.hh

class AddressPortLookupTable {
public:
    AddressPortLookupTable();
    virtual ~AddressPortLookupTable();
    
    // 添加条目
    void* Add(struct sockaddr_storage const& address1,
              struct sockaddr_storage const& address2,
              Port port, void* value);
    
    // 简化版本（只用一个地址）
    void* Add(struct sockaddr_storage const& address1,
              Port port, void* value) {
        return Add(address1, nullAddress(), port, value);
    }

    // 移除条目
    Boolean Remove(struct sockaddr_storage const& address1,
                   struct sockaddr_storage const& address2,
                   Port port);
    Boolean Remove(struct sockaddr_storage const& address1, Port port) {
        return Remove(address1, nullAddress(), port);
    }

    // 查找条目
    void* Lookup(struct sockaddr_storage const& address1,
                 struct sockaddr_storage const& address2,
                 Port port);
    void* Lookup(struct sockaddr_storage const& address1, Port port) {
        return Lookup(address1, nullAddress(), port);
    }

    void* RemoveNext() { return fTable->RemoveNext(); }

    // 迭代器
    class Iterator {
    public:
        Iterator(AddressPortLookupTable& table);
        virtual ~Iterator();
        void* next();
    private:
        HashTable::Iterator* fIter;
    };
    
private:
    friend class Iterator;
    HashTable* fTable;
};
```

### 7.2 实现要点

```cpp
// 文件：NetAddress.cpp

// 键的构造：两个IPv6地址 + 一个端口
// 每个IPv6地址占4个int（128位），共需要9个int
#define NUM_RECORDS_IN_KEY_FOR_EACH_ADDRESS (sizeof(struct in6_addr)/sizeof(int))
#define NUM_RECORDS_IN_KEY_TOTAL (2*NUM_RECORDS_IN_KEY_FOR_EACH_ADDRESS + 1)

AddressPortLookupTable::AddressPortLookupTable()
    : fTable(HashTable::create(NUM_RECORDS_IN_KEY_TOTAL)) {
}

// 从地址构建键
static void setKeyFromAddress(int*& key, struct sockaddr_storage const& address) {
    if (address.ss_family == AF_INET) {
        // IPv4地址只用最后一个int，前3个置0
        *key++ = 0;
        *key++ = 0;
        *key++ = 0;
        *key++ = ((sockaddr_in const&)address).sin_addr.s_addr;
    } else {
        // IPv6地址使用全部128位
        struct sockaddr_in6 const& addr6 = (struct sockaddr_in6&)address;
        u_int8_t const* s6a = addr6.sin6_addr.s6_addr;
        *key++ = (s6a[0]<<24)|(s6a[1]<<16)|(s6a[2]<<8)|s6a[3];
        *key++ = (s6a[4]<<24)|(s6a[5]<<16)|(s6a[6]<<8)|s6a[7];
        *key++ = (s6a[8]<<24)|(s6a[9]<<16)|(s6a[10]<<8)|s6a[11];
        *key++ = (s6a[12]<<24)|(s6a[13]<<16)|(s6a[14]<<8)|s6a[15];
    }
}

// 构建完整的键
static void setKey(int* key,
                   struct sockaddr_storage const& address1,
                   struct sockaddr_storage const& address2,
                   Port port) {
    setKeyFromAddress(key, address1);
    setKeyFromAddress(key, address2);
    *key = (int)port. num();
}

void* AddressPortLookupTable::Lookup(struct sockaddr_storage const& address1,
                                     struct sockaddr_storage const& address2,
                                     Port port) {
    int key[NUM_RECORDS_IN_KEY_TOTAL];
    setKey(key, address1, address2, port);
    return fTable->Lookup((char*)key);
}
```

### 7.3 使用场景

AddressPortLookupTable 主要用于 **GroupsockLookupTable**，用于根据组播地址和端口查找对应的 Groupsock 对象。

```cpp
// 在 GroupsockLookupTable 中的使用
class GroupsockLookupTable {
private:
    AddressPortLookupTable fTable;
    
public:
    Groupsock* Lookup(struct sockaddr_storage const& groupAddress, Port port) {
        return (Groupsock*)fTable.Lookup(groupAddress, port);
    }
};
```

---

## 八、辅助函数

### 8.1 空地址和地址判断

```cpp
// 文件：NetAddress.hh / NetAddress.cpp

// 获取空地址（全零）
struct sockaddr_storage const& nullAddress(int addressFamily = AF_INET);

// 判断地址是否为空
Boolean addressIsNull(sockaddr_storage const& address);

// 获取地址结构的大小
SOCKLEN_T addressSize(sockaddr_storage const& address);

// 复制地址
void copyAddress(struct sockaddr_storage& to, NetAddress const* from);

// 比较地址（只比较地址部分，不比较端口）
Boolean operator==(struct sockaddr_storage const& left, 
                   struct sockaddr_storage const& right);
```

### 8.2 判断组播地址

```cpp
// 文件：NetAddress. cpp

Boolean IsMulticastAddress(struct sockaddr_storage const& address) {
    switch (address.ss_family) {
        case AF_INET: {
            // IPv4组播地址范围：224.0.0.0 - 239.255. 255.255
            // 但排除 224.0.0.0 - 224.0.0. 255（不可路由）
            ipv4AddressBits addressInNetworkOrder
                = htonl(((sockaddr_in const&)address).sin_addr.s_addr);
            return addressInNetworkOrder >  0xE00000FF &&
                   addressInNetworkOrder <= 0xEFFFFFFF;
        }
        case AF_INET6: {
            // IPv6组播地址以 0xFF 开头
            return ((sockaddr_in6 const&)address). sin6_addr. s6_addr[0] == 0xFF;
        }
        default: {
            return False;
        }
    }
}
```

**组播地址范围图解：**

```
┌─────────────────────────────────────────────────────────────────┐
│                      组播地址范围                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   【IPv4 组播地址】Class D: 224.0. 0.0 - 239.255. 255.255         │
│                                                                 │
│   ┌───────────────────────────────────────────────────────────┐ │
│   │ 224.0.0.0 - 224.0.0. 255    本地链路范围（不可路由）       │ │
│   │                             如：224.0.0. 1 = 所有主机      │ │
│   │                             Live555不认为这是组播        │ │
│   ├───────────────────────────────────────────────────────────┤ │
│   │ 224.0. 1.0 - 238.255.255. 255  全球范围组播                 │ │
│   │                               Live555认为是有效组播      │ │
│   ├───────────────────────────────────────────────────────────┤ │
│   │ 239.0.0.0 - 239.255.255. 255  管理范围组播（私有）         │ │
│   │                               常用于局域网流媒体          │ │
│   └───────────────────────────────────────────────────────────┘ │
│                                                                 │
│   【IPv6 组播地址】前缀 ff00::/8                                │
│                                                                 │
│   第一字节 = 0xFF 表示组播                                      │
│   例如：ff02::1 = 所有节点                                      │
│         ff02::2 = 所有路由器                                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 8.3 端口号操作

```cpp
// 获取sockaddr_storage中的端口号
portNumBits portNum(struct sockaddr_storage const& address) {
    switch (address.ss_family) {
        case AF_INET: {
            return ((sockaddr_in&)address).sin_port;
        }
        case AF_INET6: {
            return ((sockaddr_in6&)address).sin6_port;
        }
        default: {
            return 0;
        }
    }
}

// 设置sockaddr_storage中的端口号
void setPortNum(struct sockaddr_storage& address, 
                portNumBits portNum /*网络字节序*/) {
    ((sockaddr_in&)address).sin_port = portNum;
    // sin_port和sin6_port在结构中的位置相同
}
```

---

## 九、完整使用示例

### 9.1 解析服务器地址并连接

```cpp
void connectToServer(UsageEnvironment& env, 
                     char const* serverName, 
                     unsigned short serverPort) {
    // 1. 解析服务器地址
    NetAddressList serverAddresses(serverName);
    
    if (serverAddresses.numAddresses() == 0) {
        env << "Failed to resolve " << serverName << "\n";
        return;
    }
    
    // 2. 获取第一个地址
    NetAddress const* serverAddr = serverAddresses.firstAddress();
    
    // 3.  构建sockaddr_storage
    struct sockaddr_storage serverSockAddr;
    memset(&serverSockAddr, 0, sizeof(serverSockAddr));
    copyAddress(serverSockAddr, serverAddr);
    
    // 4. 设置端口号
    Port port(serverPort);
    setPortNum(serverSockAddr, port. num());
    
    // 5.  输出调试信息
    env << "Connecting to " << AddressString(serverSockAddr). val() 
        << ":" << port << "\n";
    
    // 6. 创建Socket并连接... 
}
```

### 9.2 检查地址类型

```cpp
void analyzeAddress(UsageEnvironment& env, 
                    struct sockaddr_storage const& addr) {
    // 输出地址字符串
    env << "Address: " << AddressString(addr). val() << "\n";
    
    // 检查地址族
    if (addr.ss_family == AF_INET) {
        env << "  Type: IPv4\n";
    } else if (addr.ss_family == AF_INET6) {
        env << "  Type: IPv6\n";
    }
    
    // 检查是否为空
    if (addressIsNull(addr)) {
        env << "  Is null address\n";
    }
    
    // 检查是否为组播
    if (IsMulticastAddress(addr)) {
        env << "  Is multicast address\n";
    }
    
    // 获取端口号
    portNumBits port = portNum(addr);
    if (port != 0) {
        env << "  Port: " << ntohs(port) << "\n";
    }
}
```

---

## 十、小结

### 10.1 要点

1. **NetAddress** 统一封装 IPv4 和 IPv6 地址
2. **Port** 自动处理端口号的字节序转换
3. **NetAddressList** 管理 DNS 解析返回的多个地址
4. **AddressString** 将地址转换为可读字符串
5. **AddressPortLookupTable** 提供按地址+端口查找的哈希表
6. **IsMulticastAddress()** 判断是否为组播地址

### 10.2 核心类速查表

| 类/函数                  | 功能                         |
| ------------------------ | ---------------------------- |
| `NetAddress`             | IP地址封装（支持IPv4/IPv6）  |
| `Port`                   | 端口号封装（自动字节序转换） |
| `NetAddressList`         | DNS解析结果（地址列表）      |
| `AddressString`          | IP地址转字符串               |
| `AddressPortLookupTable` | 地址端口哈希表               |
| `IsMulticastAddress()`   | 判断组播地址                 |
| `addressIsNull()`        | 判断空地址                   |
| `nullAddress()`          | 获取空地址                   |

### 10.3 类关系图

```
┌─────────────────────────────────────────────────────────────────┐
│                      NetAddress 模块类图                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   NetAddress                    Port                            │
│   ┌────────────────────┐       ┌────────────────────┐           │
│   │ - fLength: unsigned│       │ - fPortNum         │           │
│   │ - fData: u_int8_t* │       │   (网络字节序)     │           │
│   │ + data(): u_int8_t*│       │ + num(): portNumBits│          │
│   │ + length(): unsigned│      └────────────────────┘           │
│   └────────────────────┘                                        │
│                                                                 │
│   NetAddressList                AddressString                   │
│   ┌────────────────────┐       ┌────────────────────┐           │
│   │ - fNumAddresses    │       │ - fVal: char*      │           │
│   │ - fAddressArray    │       │ + val(): char*     │           │
│   │ + numAddresses()   │       └────────────────────┘           │
│   │ + firstAddress()   │                                        │
│   │ + Iterator         │       AddressPortLookupTable           │
│   └────────────────────┘       ┌────────────────────┐           │
│                                │ - fTable: HashTable*│          │
│                                │ + Add()             │          │
│                                │ + Lookup()          │          │
│                                │ + Remove()          │          │
│                                └────────────────────┘           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

