# Live555源码学习笔记4——DelayQueue延时队列

## 一、模块概述

### 1.1 DelayQueue 的作用

DelayQueue（延时队列）是 Live555 任务调度的核心数据结构，用于管理所有需要延时执行的任务。

```
┌─────────────────────────────────────────────────────────────────┐
│                    DelayQueue 在系统中的位置                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   用户代码                                                       │
│      │                                                          │
│      │ scheduleDelayedTask(5000000, callback, data)            │
│      │ "5秒后执行callback"                                      │
│      ▼                                                          │
│   BasicTaskScheduler                                            │
│      │                                                          │
│      │ 创建AlarmHandler，添加到队列                             │
│      ▼                                                          │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                    DelayQueue                           │   │
│   │                                                         │   │
│   │   存储所有待执行的定时任务                               │   │
│   │   按到期时间排序                                        │   │
│   │   提供查询"下一个任务还有多久到期"                      │   │
│   └─────────────────────────────────────────────────────────┘   │
│      │                                                          │
│      │ 任务到期时                                               │
│      ▼                                                          │
│   handleAlarm() -> handleTimeout() -> callback(data)           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 文件结构

```
BasicUsageEnvironment/
├── include/
│   └── DelayQueue.hh      # 头文件（类型定义）
└── DelayQueue.cpp         # 实现文件
```

---

## 二、时间相关类

### 2.1 Timeval类（时间值基类）

```cpp
// 文件：DelayQueue.hh

// 时间基础类型（秒）
#ifdef TIME_BASE
typedef TIME_BASE time_base_seconds;
#else
typedef long time_base_seconds;
#endif

// Timeval 可以表示绝对时间或时间间隔
class Timeval {
public:
    // 获取秒和微秒部分
    time_base_seconds seconds() const { return fTv.tv_sec; }
    time_base_seconds useconds() const { return fTv.tv_usec; }

    // 比较运算符
    int operator>=(Timeval const& arg2) const;
    int operator<=(Timeval const& arg2) const { return arg2 >= *this; }
    int operator<(Timeval const& arg2) const { return ! (*this >= arg2); }
    int operator>(Timeval const& arg2) const { return arg2 < *this; }
    int operator==(Timeval const& arg2) const;
    int operator!=(Timeval const& arg2) const { return !(*this == arg2); }

    // 算术运算符
    void operator+=(class DelayInterval const& arg2);
    void operator-=(class DelayInterval const& arg2);

protected:
    Timeval(time_base_seconds seconds, time_base_seconds useconds) {
        fTv.tv_sec = seconds;
        fTv.tv_usec = useconds;
    }

private:
    struct timeval fTv;  // 使用POSIX的timeval结构
};
```

### 2.2 DelayInterval类（时间间隔）

```cpp
// 文件：DelayQueue.hh

class DelayInterval: public Timeval {
public:
    DelayInterval(time_base_seconds seconds, time_base_seconds useconds)
        : Timeval(seconds, useconds) {}
};

// 预定义的常用时间间隔
extern DelayInterval const DELAY_ZERO;    // 0
extern DelayInterval const DELAY_SECOND;  // 1秒
extern DelayInterval const DELAY_MINUTE;  // 1分钟
extern DelayInterval const DELAY_HOUR;    // 1小时
extern DelayInterval const DELAY_DAY;     // 1天
```

**实现：**

```cpp
// 文件：DelayQueue.cpp

const DelayInterval DELAY_ZERO(0, 0);
const DelayInterval DELAY_SECOND(1, 0);
const DelayInterval DELAY_MINUTE = 60*DELAY_SECOND;
const DelayInterval DELAY_HOUR = 60*DELAY_MINUTE;
const DelayInterval DELAY_DAY = 24*DELAY_HOUR;
const DelayInterval ETERNITY(INT_MAX, MILLION-1);  // 内部使用，表示"永远"
```

### 2.3 _EventTime类（事件时间/绝对时间）

```cpp
// 文件：DelayQueue.hh

class _EventTime: public Timeval {
public:
    _EventTime(unsigned secondsSinceEpoch = 0,
               unsigned usecondsSinceEpoch = 0)
        // 使用Unix纪元：1970年1月1日
        : Timeval(secondsSinceEpoch, usecondsSinceEpoch) {}
};

// 获取当前时间
_EventTime TimeNow();

extern _EventTime const THE_END_OF_TIME;
```

**实现：**

```cpp
// 文件：DelayQueue.cpp

_EventTime TimeNow() {
    struct timeval tvNow;
    gettimeofday(&tvNow, NULL);  // POSIX函数获取当前时间
    return _EventTime(tvNow. tv_sec, tvNow. tv_usec);
}

const _EventTime THE_END_OF_TIME(INT_MAX);
```

### 2.4 时间运算实现

```cpp
// 文件：DelayQueue.cpp

static const int MILLION = 1000000;  // 100万（微秒/秒）

// 比较运算：判断 this >= arg2
int Timeval::operator>=(const Timeval& arg2) const {
    return seconds() > arg2.seconds()
        || (seconds() == arg2.seconds() && useconds() >= arg2.useconds());
}

// 加法：时间 += 间隔
void Timeval::operator+=(const DelayInterval& arg2) {
    secs() += arg2.seconds();
    usecs() += arg2.useconds();
    
    // 处理微秒溢出（>=1000000）
    if (useconds() >= MILLION) {
        usecs() -= MILLION;
        ++secs();
    }
}

// 减法：时间 -= 间隔
void Timeval::operator-=(const DelayInterval& arg2) {
    secs() -= arg2.seconds();
    usecs() -= arg2.useconds();
    
    // 处理微秒借位（<0）
    if ((int)useconds() < 0) {
        usecs() += MILLION;
        --secs();
    }
    
    // 防止负数结果
    if ((int)seconds() < 0) {
        secs() = usecs() = 0;
    }
}

// 两个时间的差（返回间隔）
DelayInterval operator-(const Timeval& arg1, const Timeval& arg2) {
    time_base_seconds secs = arg1.seconds() - arg2.seconds();
    time_base_seconds usecs = arg1.useconds() - arg2.useconds();

    if ((int)usecs < 0) {
        usecs += MILLION;
        --secs;
    }
    
    if ((int)secs < 0)
        return DELAY_ZERO;  // 负数返回0
    else
        return DelayInterval(secs, usecs);
}
```

**时间类继承关系图：**

```
┌─────────────────────────────────────────────────────────────────┐
│                       时间类继承关系                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                      Timeval (基类)                             │
│                    ┌─────────────────┐                          │
│                    │ - fTv: timeval  │                          │
│                    │ + seconds()     │                          │
│                    │ + useconds()    │                          │
│                    │ + operator>=    │                          │
│                    │ + operator+=    │                          │
│                    └────────┬────────┘                          │
│                             │                                   │
│               ┌─────────────┴─────────────┐                     │
│               │                           │                     │
│               ▼                           ▼                     │
│      DelayInterval                   _EventTime                 │
│    ┌─────────────────┐           ┌─────────────────┐            │
│    │ 表示时间间隔    │           │ 表示绝对时间    │            │
│    │ 如：500ms       │           │ 如：时间戳      │            │
│    └─────────────────┘           └─────────────────┘            │
│                                                                 │
│    用途：                        用途：                         │
│    - 定时任务的延时              - 记录上次同步时间             │
│    - select超时时间              - 获取当前时间                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 三、DelayQueueEntry类（队列条目）

### 3.1 类定义

```cpp
// 文件：DelayQueue.hh

class DelayQueueEntry {
public:
    virtual ~DelayQueueEntry();

    // 获取任务标识（用于查找和取消任务）
    intptr_t token() { return fToken; }

protected:
    // 构造函数是protected，这是抽象基类
    DelayQueueEntry(DelayInterval delay, intptr_t token);

    // 超时处理（子类重写来执行实际任务）
    virtual void handleTimeout();

private:
    friend class DelayQueue;  // DelayQueue需要访问私有成员
    
    // 双向链表指针
    DelayQueueEntry* fNext;
    DelayQueueEntry* fPrev;
    
    // ⭐ 关键：存储的是"增量时间"，不是绝对到期时间
    DelayInterval fDeltaTimeRemaining;
    
    // 任务唯一标识
    intptr_t fToken;
};
```

### 3.2 构造函数和析构函数

```cpp
// 文件：DelayQueue.cpp

DelayQueueEntry::DelayQueueEntry(DelayInterval delay, intptr_t token)
    : fDeltaTimeRemaining(delay), fToken(token) {
    // 初始化为自己指向自己（单节点循环）
    fNext = fPrev = this;
}

DelayQueueEntry::~DelayQueueEntry() {
}

// 默认的超时处理：删除自己
void DelayQueueEntry::handleTimeout() {
    delete this;
}
```

### 3.3 增量时间存储方式（核心设计）

这是DelayQueue最巧妙的设计！不存储绝对到期时间，而是存储**相对于前一个节点的增量时间**。

```
┌─────────────────────────────────────────────────────────────────┐
│                     增量时间存储方式                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  【绝对时间存储方式】（Live555没有采用）                         │
│                                                                 │
│   任务A: 到期时间 = 100ms                                       │
│   任务B: 到期时间 = 300ms                                       │
│   任务C: 到期时间 = 500ms                                       │
│                                                                 │
│   问题：每次需要更新所有任务的剩余时间                          │
│                                                                 │
│  ─────────────────────────────────────────────────────────────  │
│                                                                 │
│  【增量时间存储方式】（Live555采用）⭐                           │
│                                                                 │
│   [head] ──► [任务A] ──► [任务B] ──► [任务C] ──► [哨兵]         │
│              100ms       200ms       200ms      ETERNITY        │
│                                                                 │
│   任务A: 增量 = 100ms (距离现在100ms)                           │
│   任务B: 增量 = 200ms (距离任务A 200ms，即距离现在300ms)        │
│   任务C: 增量 = 200ms (距离任务B 200ms，即距离现在500ms)        │
│                                                                 │
│   优点：                                                        │
│   ✓ 时间流逝时，只需更新第一个节点                              │
│   ✓ 插入/删除节点时，只需调整相邻节点                           │
│   ✓ 判断是否到期：只看第一个节点是否为0                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**时间流逝的处理：**

```
┌─────────────────────────────────────────────────────────────────┐
│                     时间流逝示例                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  初始状态（T=0）：                                               │
│  [任务A: 100ms] ──► [任务B: 200ms] ──► [任务C: 200ms]           │
│                                                                 │
│  经过50ms后（T=50ms）：                                          │
│  [任务A: 50ms] ──► [任务B: 200ms] ──► [任务C: 200ms]            │
│  （只需要减少第一个节点！）                                      │
│                                                                 │
│  又经过50ms后（T=100ms）：                                       │
│  [任务A: 0ms] ──► [任务B: 200ms] ──► [任务C: 200ms]             │
│  （任务A到期！执行并移除）                                       │
│                                                                 │
│  移除任务A后：                                                   │
│  [任务B: 200ms] ──► [任务C: 200ms]                              │
│  （任务B现在是第一个，它的200ms就是距离现在的时间）              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 四、DelayQueue类（延时队列）

### 4.1 类定义

```cpp
// 文件：DelayQueue.hh

class DelayQueue: public DelayQueueEntry {
public:
    DelayQueue();
    virtual ~DelayQueue();

    // 添加任务到队列
    void addEntry(DelayQueueEntry* newEntry);
    
    // 更新任务的延时
    void updateEntry(DelayQueueEntry* entry, DelayInterval newDelay);
    void updateEntry(intptr_t tokenToFind, DelayInterval newDelay);
    
    // 从队列移除任务（不删除）
    void removeEntry(DelayQueueEntry* entry);
    DelayQueueEntry* removeEntry(intptr_t tokenToFind);

    // 获取下一个任务的到期时间（用于select超时）
    DelayInterval const& timeToNextAlarm();
    
    // 处理到期的任务
    void handleAlarm();

private:
    // 获取队首（第一个真正的任务）
    DelayQueueEntry* head() { return fNext; }
    
    // 根据Token查找任务
    DelayQueueEntry* findEntryByToken(intptr_t token);
    
    // 同步时间（更新增量时间）
    void synchronize();

    // 上次同步的时间点
    _EventTime fLastSyncTime;
};
```

**注意：DelayQueue继承自DelayQueueEntry！**

这是一个巧妙的设计：DelayQueue本身作为链表的**哨兵节点**（sentinel），简化了边界条件的处理。

```
┌─────────────────────────────────────────────────────────────────┐
│                   DelayQueue 链表结构                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   DelayQueue对象本身是哨兵节点，永远在链表末尾                   │
│   其 fDeltaTimeRemaining = ETERNITY（无穷大）                   │
│                                                                 │
│        ┌──────────────────────────────────────────────────┐     │
│        │                                                  │     │
│        ▼                                                  │     │
│   ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐     │
│   │DelayQueue│◄──►│ 任务A  │◄──►│ 任务B  │◄──►│ 任务C  │     │
│   │ (哨兵)  │    │ 100ms  │    │ 200ms  │    │ 200ms  │     │
│   │ETERNITY │    │        │    │        │    │        │     │
│   └─────────┘    └─────────┘    └─────────┘    └─────────┘     │
│        ▲                                              │         │
│        └──────────────────────────────────────────────┘         │
│                      (双向循环链表)                              │
│                                                                 │
│   head() 返回 fNext，即第一个真正的任务（任务A）                │
│   当 head() == this 时，表示队列为空                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 构造函数和析构函数

```cpp
// 文件：DelayQueue.cpp

DelayQueue::DelayQueue()
    : DelayQueueEntry(ETERNITY, 0) {  // 哨兵节点的延时是"永远"
    fLastSyncTime = TimeNow();  // 记录创建时间
}

DelayQueue::~DelayQueue() {
    // 删除所有剩余的任务
    while (fNext != this) {
        DelayQueueEntry* entryToRemove = fNext;
        removeEntry(entryToRemove);
        delete entryToRemove;
    }
}
```

### 4.3 添加任务：addEntry() 

```cpp
// 文件：DelayQueue. cpp

void DelayQueue::addEntry(DelayQueueEntry* newEntry) {
    // 首先同步时间（更新现有任务的增量时间）
    synchronize();

    // 找到合适的插入位置（保持按到期时间排序）
    DelayQueueEntry* cur = head();
    
    // 遍历找到第一个比newEntry晚到期的节点
    while (newEntry->fDeltaTimeRemaining >= cur->fDeltaTimeRemaining) {
        // 减去经过的节点的增量时间
        newEntry->fDeltaTimeRemaining -= cur->fDeltaTimeRemaining;
        cur = cur->fNext;
    }

    // 调整后续节点的增量时间
    cur->fDeltaTimeRemaining -= newEntry->fDeltaTimeRemaining;

    // 将newEntry插入到cur之前
    newEntry->fNext = cur;
    newEntry->fPrev = cur->fPrev;
    cur->fPrev = newEntry->fPrev->fNext = newEntry;
}
```

**插入示例：**

```
┌─────────────────────────────────────────────────────────────────┐
│                     addEntry() 插入示例                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  初始队列：                                                      │
│  [哨兵] ──► [任务A: 100ms] ──► [任务C: 300ms] ──► [哨兵]        │
│                                                                 │
│  插入任务B（延时250ms）：                                        │
│                                                                 │
│  步骤1：从head开始遍历                                          │
│         newEntry. delta = 250ms                                  │
│         cur = 任务A (delta=100ms)                               │
│         250ms >= 100ms → newEntry.delta = 250-100 = 150ms       │
│         cur = 任务C (delta=300ms)                               │
│         150ms < 300ms → 找到插入位置！                          │
│                                                                 │
│  步骤2：调整任务C的增量                                         │
│         任务C. delta = 300 - 150 = 150ms                         │
│                                                                 │
│  步骤3：插入任务B                                               │
│                                                                 │
│  结果：                                                         │
│  [哨兵] ──► [任务A: 100ms] ──► [任务B: 150ms] ──► [任务C: 150ms]│
│                                                                 │                   
└─────────────────────────────────────────────────────────────────┘
```

### 4.4 移除任务：removeEntry()

```cpp
// 文件：DelayQueue.cpp

void DelayQueue::removeEntry(DelayQueueEntry* entry) {
    if (entry == NULL || entry->fNext == NULL) return;

    // 将被删除节点的增量时间加到下一个节点
    entry->fNext->fDeltaTimeRemaining += entry->fDeltaTimeRemaining;
    
    // 从链表中移除
    entry->fPrev->fNext = entry->fNext;
    entry->fNext->fPrev = entry->fPrev;
    
    // 标记为已移除（防止重复移除）
    entry->fNext = entry->fPrev = NULL;
}

// 根据Token移除
DelayQueueEntry* DelayQueue::removeEntry(intptr_t tokenToFind) {
    DelayQueueEntry* entry = findEntryByToken(tokenToFind);
    removeEntry(entry);
    return entry;
}
```

**移除示例：**

```
┌─────────────────────────────────────────────────────────────────┐
│                     removeEntry() 移除示例                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  原队列：                                                        │
│  [A: 100ms] ──► [B: 150ms] ──► [C: 150ms]                       │
│                                                                 │
│  移除任务B：                                                     │
│                                                                 │
│  步骤1：将B的增量加到C                                          │
│         C.delta = 150 + 150 = 300ms                             │
│                                                                 │
│  步骤2：从链表移除B                                             │
│                                                                 │
│  结果：                                                         │
│  [A: 100ms] ──► [C: 300ms]                                      │
│                                                                 │
│  验证：                                                         │
│  任务C原来是 100+150+150 = 400ms 后到期                         │
│  现在是 100+300 = 400ms 后到期 ✓ 正确！                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 4.5 时间同步：synchronize() 

这是保证增量时间正确的关键函数：

```cpp
// 文件：DelayQueue.cpp

void DelayQueue::synchronize() {
    // 计算自上次同步以来经过的时间
    _EventTime timeNow = TimeNow();
    
    // 处理系统时钟回退的情况
    if (timeNow < fLastSyncTime) {
        fLastSyncTime = timeNow;
        return;
    }
    
    DelayInterval timeSinceLastSync = timeNow - fLastSyncTime;
    fLastSyncTime = timeNow;

    // 从队首开始，减去经过的时间
    DelayQueueEntry* curEntry = head();
    
    while (timeSinceLastSync >= curEntry->fDeltaTimeRemaining) {
        // 这个节点的时间已经"用完"，标记为0
        timeSinceLastSync -= curEntry->fDeltaTimeRemaining;
        curEntry->fDeltaTimeRemaining = DELAY_ZERO;
        curEntry = curEntry->fNext;
    }
    
    // 最后一个未完全"用完"的节点，减去剩余时间
    curEntry->fDeltaTimeRemaining -= timeSinceLastSync;
}
```

**同步示例：**

```
┌─────────────────────────────────────────────────────────────────┐
│                     synchronize() 同步示例                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  上次同步时间：T=0                                               │
│  队列状态：[A: 100ms] ──► [B: 150ms] ──► [C: 200ms]             │
│                                                                 │
│  现在时间：T=120ms（经过了120ms）                                │
│                                                                 │
│  同步过程：                                                      │
│  timeSinceLastSync = 120ms                                      │
│                                                                 │
│  处理任务A (delta=100ms)：                                       │
│    120ms >= 100ms → 任务A到期                                   │
│    A.delta = 0                                                  │
│    timeSinceLastSync = 120 - 100 = 20ms                         │
│                                                                 │
│  处理任务B (delta=150ms)：                                       │
│    20ms < 150ms → 任务B未到期                                   │
│    B.delta = 150 - 20 = 130ms                                   │
│                                                                 │
│  结果：                                                         │
│  [A: 0ms] ──► [B: 130ms] ──► [C: 200ms]                         │
│                                                                 │
│  任务A的delta=0表示已到期，等待handleAlarm()处理                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 4.6 获取下一个任务到期时间：timeToNextAlarm()

```cpp
// 文件：DelayQueue.cpp

DelayInterval const& DelayQueue::timeToNextAlarm() {
    // 快速路径：如果队首已经到期，直接返回0
    if (head()->fDeltaTimeRemaining == DELAY_ZERO) {
        return DELAY_ZERO;
    }

    // 否则需要同步时间后返回
    synchronize();
    return head()->fDeltaTimeRemaining;
}
```

**这个函数在SingleStep()中被调用，用于计算select()的超时时间。**

### 4.7 处理到期任务：handleAlarm()

```cpp
// 文件：DelayQueue.cpp

void DelayQueue::handleAlarm() {
    // 如果队首不是0，先同步
    if (head()->fDeltaTimeRemaining != DELAY_ZERO) {
        synchronize();
    }

    // 如果队首到期了（delta == 0）
    if (head()->fDeltaTimeRemaining == DELAY_ZERO) {
        // 取出队首任务
        DelayQueueEntry* toRemove = head();
        
        // 先从队列移除（防止handleTimeout中再次操作队列时出问题）
        removeEntry(toRemove);

        // 调用任务的超时处理函数
        toRemove->handleTimeout();
    }
}
```

### 4.8 查找任务：findEntryByToken()

```cpp
// 文件：DelayQueue.cpp

DelayQueueEntry* DelayQueue::findEntryByToken(intptr_t tokenToFind) {
    DelayQueueEntry* cur = head();
    
    // 遍历链表查找
    while (cur != this) {  // this是哨兵节点
        if (cur->token() == tokenToFind) {
            return cur;
        }
        cur = cur->fNext;
    }

    return NULL;  // 未找到
}
```

---

## 五、AlarmHandler类（与DelayQueue配合使用）

在`BasicTaskScheduler0. cpp`中定义，是DelayQueueEntry的子类：

```cpp
// 文件：BasicTaskScheduler0.cpp

class AlarmHandler: public DelayQueueEntry {
public:
    AlarmHandler(TaskFunc* proc, void* clientData, 
                 DelayInterval timeToDelay, intptr_t token)
        : DelayQueueEntry(timeToDelay, token),
          fProc(proc), 
          fClientData(clientData) {
    }

private:
    // 重写handleTimeout，执行用户回调
    virtual void handleTimeout() {
        (*fProc)(fClientData);              // 执行用户的回调函数
        DelayQueueEntry::handleTimeout();   // 调用基类（删除自己）
    }

private:
    TaskFunc* fProc;       // 用户注册的回调函数
    void* fClientData;     // 用户数据
};
```

**类关系图：**

```
┌─────────────────────────────────────────────────────────────────┐
│                    DelayQueue 相关类图                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│                    DelayQueueEntry                              │
│                  ┌───────────────────┐                          │
│                  │ - fNext, fPrev    │                          │
│                  │ - fDeltaTimeRemaining                        │
│                  │ - fToken          │                          │
│                  │ + handleTimeout() │ ◄─── 虚函数              │
│                  └─────────┬─────────┘                          │
│                            │                                    │
│              ┌─────────────┴─────────────┐                      │
│              │                           │                      │
│              ▼                           ▼                      │
│        DelayQueue                   AlarmHandler                │
│     ┌───────────────────┐      ┌───────────────────┐           │
│     │ 作为哨兵节点      │      │ - fProc           │           │
│     │ - fLastSyncTime   │      │ - fClientData     │           │
│     │ + addEntry()      │      │ + handleTimeout() │           │
│     │ + removeEntry()   │      │   执行用户回调    │           │
│     │ + handleAlarm()   │      └───────────────────┘           │
│     │ + synchronize()   │                                      │
│     └───────────────────┘                                      │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 六、使用示例

### 6.1 调度一个延时任务

```cpp
// 用户代码
void myCallback(void* clientData) {
    printf("定时器到期！\n");
}

// 调度一个2秒后执行的任务
TaskToken token = env->taskScheduler(). scheduleDelayedTask(
    2000000,      // 2秒 = 2,000,000微秒
    myCallback,
    NULL
);
```

**内部流程：**

```
┌─────────────────────────────────────────────────────────────────┐
│              scheduleDelayedTask() 内部流程                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. 创建 AlarmHandler 对象                                      │
│     AlarmHandler* handler = new AlarmHandler(                   │
│         myCallback,                                             │
│         NULL,                                                   │
│         DelayInterval(2, 0),  // 2秒                            │
│         ++fTokenCounter       // 唯一标识                       │
│     );                                                          │
│                                                                 │
│  2. 添加到延时队列                                              │
│     fDelayQueue.addEntry(handler);                              │
│                                                                 │
│  3. 返回 Token                                                  │
│     return (void*)(handler->token());                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 6.2 取消延时任务

```cpp
// 取消之前调度的任务
env->taskScheduler().unscheduleDelayedTask(token);
```

### 6.3 周期性任务

```cpp
class PeriodicTask {
public:
    void start() {
        scheduleNext();
    }
    
    void stop() {
        env->taskScheduler().unscheduleDelayedTask(fTask);
    }
    
private:
    static void onTimer(void* clientData) {
        PeriodicTask* self = (PeriodicTask*)clientData;
        
        // 执行周期性工作
        self->doWork();
        
        // 重新调度下一次
        self->scheduleNext();
    }
    
    void scheduleNext() {
        fTask = env->taskScheduler().scheduleDelayedTask(
            1000000,  // 每秒执行一次
            onTimer,
            this
        );
    }
    
    void doWork() {
        printf("周期性任务执行中...\n");
    }
    
    TaskToken fTask;
    UsageEnvironment* env;
};
```



## 七、小结

### 7.1 要点

1. **时间类继承体系**：Timeval → DelayInterval / _EventTime
2. **增量时间存储**：每个节点存储相对于前一个节点的时间差
3. **哨兵节点设计**：DelayQueue 继承自 DelayQueueEntry，作为链表尾部哨兵
4. **时间同步机制**：synchronize() 保证增量时间的正确性
5. **AlarmHandler**：封装用户回调，继承 DelayQueueEntry

### 7.2 核心函数速查表

| 函数                | 功能             | 调用时机                    |
| ------------------- | ---------------- | --------------------------- |
| `addEntry()`        | 添加定时任务     | scheduleDelayedTask()       |
| `removeEntry()`     | 移除定时任务     | unscheduleDelayedTask()     |
| `timeToNextAlarm()` | 获取下次到期时间 | SingleStep() 计算select超时 |
| `handleAlarm()`     | 处理到期任务     | SingleStep() 末尾           |
| `synchronize()`     | 同步时间         | 被上述函数内部调用          |

