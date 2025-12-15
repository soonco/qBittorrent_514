# LogMsg 技术文档

## 一、概述

`LogMsg` 是 qBittorrent 项目中的全局日志记录辅助函数，用于向应用程序的日志系统添加消息。它是对 `Logger` 单例类的简化封装，提供了便捷的日志记录接口。

**定义位置：**
- 声明：`src/base/logger.h:104`
- 实现：`src/base/logger.cpp:125-128`

---

## 二、函数签名

```cpp
void LogMsg(const QString &message, const Log::MsgType &type = Log::NORMAL);
```

**参数说明：**
- `message`：日志消息内容（QString 类型）
- `type`：日志消息类型（可选，默认为 `Log::NORMAL`）

---

## 三、日志消息类型

日志系统支持以下消息类型（定义在 `logger.h:42-49`）：

```cpp
enum MsgType
{
    ALL = -1,          // 所有类型（用于过滤）
    NORMAL = 0x1,      // 普通消息
    INFO = 0x2,        // 信息消息
    WARNING = 0x4,     // 警告消息
    CRITICAL = 0x8     // 严重错误消息
};
```

**类型说明：**
- **NORMAL**：普通操作日志，如 torrent 添加、完成等
- **INFO**：系统信息，如配置状态、功能开关状态
- **WARNING**：警告信息，如配置错误、操作失败但不致命
- **CRITICAL**：严重错误，如系统故障、数据不一致

---

## 四、实现原理

### 4.1 核心架构

LogMsg 基于单例模式的 `Logger` 类实现，整体架构如下：

```
LogMsg (辅助函数)
    ↓
Logger::instance()->addMessage()
    ↓
存储到循环缓冲区 (boost::circular_buffer_space_optimized)
    ↓
发射 Qt 信号 newLogMessage()
    ↓
订阅者接收日志（GUI、文件记录器等）
```

### 4.2 Logger 类核心实现

#### 单例管理（logger.cpp:48-71）

```cpp
Logger *Logger::m_instance = nullptr;

void Logger::initInstance()
{
    if (!m_instance)
        m_instance = new Logger;
}

void Logger::freeInstance()
{
    delete m_instance;
    m_instance = nullptr;
}

Logger *Logger::instance()
{
    return m_instance;
}
```

#### 消息添加（logger.cpp:73-81）

```cpp
void Logger::addMessage(const QString &message, const Log::MsgType &type)
{
    QWriteLocker locker(&m_lock);
    const Log::Msg msg = {m_msgCounter++, type, QDateTime::currentSecsSinceEpoch(), message};
    m_messages.push_back(msg);
    locker.unlock();

    emit newLogMessage(msg);
}
```

#### LogMsg 实现（logger.cpp:125-128）

```cpp
void LogMsg(const QString &message, const Log::MsgType &type)
{
    Logger::instance()->addMessage(message, type);
}
```

### 4.3 数据结构

#### 日志消息结构（logger.h:52-58）

```cpp
struct Msg
{
    int id = -1;                // 消息唯一 ID（自增）
    MsgType type = ALL;         // 消息类型
    qint64 timestamp = -1;      // Unix 时间戳（秒）
    QString message;            // 消息内容
};
```

**存储机制：**
- 使用 `boost::circular_buffer_space_optimized` 循环缓冲区
- 最大容量：20000 条消息（`MAX_LOG_MESSAGES`，logger.h:38）
- 超出容量时自动覆盖最旧的消息

### 4.4 线程安全

Logger 使用 `QReadWriteLock` 保证线程安全（logger.h:98）：
- **写操作**（addMessage）：使用 `QWriteLocker` 独占锁
- **读操作**（getMessages）：使用 `QReadLocker` 共享锁
- 支持多线程并发读取，写入时互斥

---

## 五、使用方法

### 5.1 基本用法

#### 普通日志

```cpp
LogMsg(tr("Torrent added successfully"));
```

#### 信息日志

```cpp
LogMsg(tr("Distributed Hash Table (DHT) support: ON"), Log::INFO);
```

#### 警告日志

```cpp
LogMsg(tr("Failed to load configuration file"), Log::WARNING);
```

#### 严重错误日志

```cpp
LogMsg(tr("Database corruption detected"), Log::CRITICAL);
```

### 5.2 实际使用示例

#### 示例 1：应用启动日志（application.cpp:315-316）

```cpp
LogMsg(tr("qBittorrent %1 started. Process ID: %2", "qBittorrent v3.2.0alpha started")
    .arg(QStringLiteral(QBT_VERSION), QString::number(QCoreApplication::applicationPid())));
```

#### 示例 2：配置信息日志（application.cpp:319）

```cpp
LogMsg(tr("Running in portable mode. Auto detected profile folder at: %1")
    .arg(profileDir.toString()));
```

#### 示例 3：警告日志（application.cpp:321）

```cpp
LogMsg(tr("Redundant command line flag detected: \"%1\". Portable mode implies relative fastresume.")
    .arg(u"--relative-fastresume"_s), Log::WARNING);
```

#### 示例 4：外部程序执行日志（application.cpp:649）

```cpp
const QString logMsg = tr("Running external program. Torrent: \"%1\". Command: `%2`");
LogMsg(logMsg.arg(torrent->name(), program));
```

#### 示例 5：系统配置日志（sessionimpl.cpp:1704-1710）

```cpp
LogMsg(tr("Peer ID: \"%1\"").arg(QString::fromStdString(peerId)), Log::INFO);
LogMsg(tr("HTTP User-Agent: \"%1\"").arg(USER_AGENT), Log::INFO);
LogMsg(tr("Distributed Hash Table (DHT) support: %1")
    .arg(isDHTEnabled() ? tr("ON") : tr("OFF")), Log::INFO);
LogMsg(tr("Local Peer Discovery support: %1")
    .arg(isLSDEnabled() ? tr("ON") : tr("OFF")), Log::INFO);
LogMsg(tr("Peer Exchange (PeX) support: %1")
    .arg(isPeXEnabled() ? tr("ON") : tr("OFF")), Log::INFO);
```

#### 示例 6：错误日志（application.cpp:1219）

```cpp
const QString logMessage = tr("Failed to set physical memory (RAM) usage limit. Error code: %1. Error message: \"%2\"");
LogMsg(logMessage.arg(QString::number(errorCode), message), Log::WARNING);
```

#### 示例 7：Torrent 操作日志（sessionimpl.cpp:2347-2363）

```cpp
LogMsg(u"%1 %2 %3"_s.arg(description, tr("Removing torrent."), torrentName));
LogMsg(u"%1 %2 %3"_s.arg(description, tr("Removing torrent and deleting its content."), torrentName));
LogMsg(u"%1 %2 %3"_s.arg(description, tr("Torrent stopped."), torrentName));
```

### 5.3 国际化支持

LogMsg 完全支持 Qt 国际化（i18n）：

```cpp
LogMsg(tr("Torrent: %1, sending mail notification").arg(torrent->name()));
```

使用 `tr()` 函数包裹消息字符串，确保消息可以被翻译成不同语言。

---

## 六、日志消费

### 6.1 获取日志消息

应用程序可以通过以下方式获取日志：

```cpp
QList<Log::Msg> Logger::getMessages(int lastKnownId = -1) const
```

**参数说明：**
- `lastKnownId = -1`：获取所有消息
- `lastKnownId >= 0`：获取 ID 大于 lastKnownId 的新消息（增量获取）

**实现原理（logger.cpp:93-107）：**

```cpp
QList<Log::Msg> Logger::getMessages(const int lastKnownId) const
{
    const QReadLocker locker(&m_lock);

    const int diff = m_msgCounter - lastKnownId - 1;
    const int size = static_cast<int>(m_messages.size());

    if ((lastKnownId == -1) || (diff >= size))
        return loadFromBuffer(m_messages);

    if (diff <= 0)
        return {};

    return loadFromBuffer(m_messages, (size - diff));
}
```

### 6.2 实时监听

通过 Qt 信号槽机制实时接收日志：

```cpp
connect(Logger::instance(), &Logger::newLogMessage, 
        this, [](const Log::Msg &msg) {
    qDebug() << "New log:" << msg.message;
});
```

**信号定义（logger.h:88）：**

```cpp
signals:
    void newLogMessage(const Log::Msg &message);
```

### 6.3 文件日志记录

应用程序使用 `FileLogger` 类将日志持久化到文件（application.cpp:329）：

```cpp
if (isFileLoggerEnabled())
    m_fileLogger = new FileLogger(
        fileLoggerPath(), 
        isFileLoggerBackup(), 
        fileLoggerMaxSize(), 
        isFileLoggerDeleteOld(), 
        fileLoggerAge(), 
        static_cast<FileLogger::FileLogAgeType>(fileLoggerAgeType())
    );
```

**文件日志配置：**
- 默认路径：`<DataFolder>/logs/`
- 默认大小限制：65 KB（可配置范围：1 KB - 1000 MB）
- 支持日志轮转和自动清理
- 支持按时间清理旧日志（天/月/年）

---

## 七、初始化和清理

### 7.1 初始化

在应用程序启动时初始化 Logger（application.cpp:281）：

```cpp
Logger::initInstance();
```

**注意：** 必须在使用 `LogMsg` 之前调用 `initInstance()`。

### 7.2 清理

在应用程序退出时释放 Logger（application.cpp:1395）：

```cpp
Logger::freeInstance();
```

**完整的清理顺序（application.cpp:1334-1410）：**

```cpp
void Application::cleanup()
{
    LogMsg(tr("qBittorrent termination initiated"));
    
    // ... 清理其他组件 ...
    
    LogMsg(tr("qBittorrent is now ready to exit"));
    Logger::freeInstance();
    delete m_fileLogger;
}
```

### 7.3 元类型注册

为了在 Qt 信号槽中传递日志对象，需要注册元类型（application.cpp:269-270）：

```cpp
qRegisterMetaType<Log::Msg>("Log::Msg");
qRegisterMetaType<Log::Peer>("Log::Peer");
```

---

## 八、最佳实践

### 8.1 选择合适的日志级别

#### NORMAL - 用户操作、业务逻辑事件

```cpp
LogMsg(tr("Torrent '%1' added").arg(name));
LogMsg(tr("Download completed: %1").arg(torrentName));
LogMsg(tr("Saving resume data completed."));
```

#### INFO - 系统状态、配置信息

```cpp
LogMsg(tr("DHT enabled: %1").arg(enabled ? "ON" : "OFF"), Log::INFO);
LogMsg(tr("Peer ID: \"%1\"").arg(peerId), Log::INFO);
LogMsg(tr("Using config directory: %1").arg(configPath), Log::INFO);
```

#### WARNING - 非致命错误、配置问题

```cpp
LogMsg(tr("Failed to load theme: %1").arg(path), Log::WARNING);
LogMsg(tr("Restart is required to toggle Peer Exchange (PeX) support"), Log::WARNING);
LogMsg(tr("Set app style failed. Unknown style: \"%1\"").arg(styleName), Log::WARNING);
```

#### CRITICAL - 严重错误、数据损坏

```cpp
LogMsg(tr("Database corruption detected"), Log::CRITICAL);
LogMsg(tr("Failed to initialize BitTorrent session"), Log::CRITICAL);
```

### 8.2 消息格式建议

#### 1. 使用描述性消息

```cpp
// 好的示例
LogMsg(tr("Failed to resume torrent. Torrent: \"%1\". Reason: \"%2\"")
    .arg(torrentName, reason));

// 避免这样
LogMsg(tr("Error: %1").arg(code));
```

#### 2. 包含上下文信息

```cpp
LogMsg(tr("Running external program. Torrent: \"%1\". Command: `%2`")
    .arg(torrent->name(), program));

LogMsg(tr("Failed to save search history. File: \"%1\". Error: \"%2\"")
    .arg(filePath, errorMessage));
```

#### 3. 使用一致的格式

- **文件路径**用双引号：`"path/to/file"`
- **命令**用反引号：`` `command` ``
- **参数名**用冒号分隔：`Name: "value"`
- **操作结果**明确说明：`"succeeded"` / `"failed"`

```cpp
LogMsg(tr("File: \"%1\". Size: %2. Status: %3")
    .arg(filePath, sizeStr, status));
```

#### 4. 提供可操作的信息

```cpp
// 好的示例 - 告诉用户如何解决
LogMsg(tr("Failed to set physical memory (RAM) usage hard limit. "
          "Requested size: %1. System hard limit: %2. "
          "Error code: %3. Error message: \"%4\"")
    .arg(requestedSize, systemLimit, errorCode, errorMsg), Log::WARNING);

// 避免这样 - 信息不足
LogMsg(tr("Memory limit failed"), Log::WARNING);
```

### 8.3 性能考虑

#### 1. 避免频繁日志

```cpp
// 避免在循环中记录日志
for (const auto &torrent : torrents) {
    // 不要：LogMsg(tr("Processing: %1").arg(torrent->name()));
}

// 改为批量记录
LogMsg(tr("Processing %1 torrents...").arg(torrents.size()));
```

#### 2. 延迟字符串构造

```cpp
// 只在需要时构造复杂字符串
if (Logger::instance()) {
    QString detailedInfo = buildDetailedDebugInfo();  // 昂贵的操作
    LogMsg(detailedInfo, Log::INFO);
}
```

#### 3. 使用合适的缓冲区大小

默认 20000 条足够大多数场景。如需调整，修改 `logger.h:38`：

```cpp
inline const int MAX_LOG_MESSAGES = 20000;
```

### 8.4 调试技巧

#### 1. 临时调试日志

```cpp
#ifdef QBT_DEBUG
    LogMsg(tr("[DEBUG] Variable value: %1").arg(debugValue), Log::INFO);
#endif
```

#### 2. 条件日志

```cpp
if (Preferences::instance()->isVerboseLoggingEnabled()) {
    LogMsg(tr("Detailed operation info: %1").arg(details), Log::INFO);
}
```

#### 3. 日志过滤

在 GUI 中可以按类型过滤日志（mainwindow.cpp:366）：

```cpp
const Log::MsgTypes flags = executionLogMsgTypes();
// 可以组合多个类型：Log::NORMAL | Log::INFO | Log::WARNING
```

---

## 九、高级特性

### 9.1 Peer 日志系统

除了普通消息，Logger 还支持 Peer 相关的日志：

```cpp
void Logger::addPeer(const QString &ip, const bool blocked, const QString &reason = {});
```

**Peer 日志结构（logger.h:60-67）：**

```cpp
struct Peer
{
    int id = -1;
    bool blocked = false;
    qint64 timestamp = -1;
    QString ip;
    QString reason;
};
```

**使用示例（peerlistwidget.cpp:356）：**

```cpp
LogMsg(tr("Peer \"%1\" is manually banned").arg(ip));
```

### 9.2 日志类型标志

`Log::MsgTypes` 是一个标志类型，支持位运算：

```cpp
Q_DECLARE_FLAGS(MsgTypes, MsgType)
Q_DECLARE_OPERATORS_FOR_FLAGS(Log::MsgTypes)
```

**使用示例：**

```cpp
Log::MsgTypes types = Log::NORMAL | Log::INFO;
types.setFlag(Log::WARNING, true);   // 添加 WARNING
types.testFlag(Log::INFO);           // 测试是否包含 INFO
```

### 9.3 WebUI 日志访问

日志可以通过 WebUI API 访问，支持：
- 获取指定范围的日志
- 实时日志流
- 按类型过滤

---

## 十、故障排查

### 10.1 常见问题

#### 问题 1：LogMsg 调用无效果

**原因：** Logger 未初始化

**解决：**
```cpp
Logger::initInstance();  // 在使用前调用
```

#### 问题 2：日志丢失

**原因：** 循环缓冲区溢出（超过 20000 条）

**解决：**
- 增加 `MAX_LOG_MESSAGES` 值
- 使用文件日志持久化
- 及时清理或导出日志

#### 问题 3：日志文件未生成

**原因：** 文件日志未启用或路径无权限

**解决：**
```cpp
// 检查配置
if (!isFileLoggerEnabled()) {
    setFileLoggerEnabled(true);
}

// 检查路径权限
QFileInfo info(fileLoggerPath().data());
if (!info.isWritable()) {
    // 更改路径或修复权限
}
```

### 10.2 调试方法

#### 方法 1：检查 Logger 状态

```cpp
if (Logger::instance()) {
    qDebug() << "Logger is initialized";
    qDebug() << "Message count:" << Logger::instance()->getMessages().size();
} else {
    qDebug() << "Logger is NOT initialized";
}
```

#### 方法 2：监听所有日志

```cpp
connect(Logger::instance(), &Logger::newLogMessage, 
        [](const Log::Msg &msg) {
    qDebug() << "[" << msg.id << "]" 
             << QDateTime::fromSecsSinceEpoch(msg.timestamp)
             << msg.type << ":" << msg.message;
});
```

#### 方法 3：导出日志到控制台

```cpp
for (const auto &msg : Logger::instance()->getMessages()) {
    std::cout << msg.message.toStdString() << std::endl;
}
```

---

## 十一、相关文件

| 文件 | 说明 | 关键内容 |
|------|------|----------|
| `src/base/logger.h` | Logger 类声明 | 日志类型定义、Logger 接口 |
| `src/base/logger.cpp` | Logger 类实现 | LogMsg 实现、消息管理 |
| `src/app/application.cpp` | 应用程序主类 | Logger 初始化、使用示例 |
| `src/app/filelogger.h` | 文件日志记录器声明 | 文件日志配置 |
| `src/app/filelogger.cpp` | 文件日志记录器实现 | 日志文件写入、轮转 |
| `src/gui/mainwindow.cpp` | 主窗口 | 日志过滤、GUI 显示 |
| `src/gui/executionlogwidget.cpp` | 日志显示组件 | 日志 UI 实现 |

---

## 十二、API 参考

### Logger 类完整接口

```cpp
class Logger final : public QObject
{
    Q_OBJECT
    Q_DISABLE_COPY_MOVE(Logger)

public:
    static void initInstance();
    static void freeInstance();
    static Logger *instance();

    void addMessage(const QString &message, const Log::MsgType &type = Log::NORMAL);
    void addPeer(const QString &ip, bool blocked, const QString &reason = {});
    QList<Log::Msg> getMessages(int lastKnownId = -1) const;
    QList<Log::Peer> getPeers(int lastKnownId = -1) const;

signals:
    void newLogMessage(const Log::Msg &message);
    void newLogPeer(const Log::Peer &peer);

private:
    Logger();
    ~Logger() = default;

    static Logger *m_instance;
    boost::circular_buffer_space_optimized<Log::Msg> m_messages;
    boost::circular_buffer_space_optimized<Log::Peer> m_peers;
    mutable QReadWriteLock m_lock;
    int m_msgCounter = 0;
    int m_peerCounter = 0;
};
```

### 辅助函数

```cpp
void LogMsg(const QString &message, const Log::MsgType &type = Log::NORMAL);
```

---

## 十三、总结

`LogMsg` 是 qBittorrent 日志系统的核心接口，具有以下特点：

### 优势

✅ **简单易用**：全局函数，无需显式获取 Logger 实例  
✅ **线程安全**：使用读写锁保护，支持多线程并发  
✅ **类型丰富**：支持 4 种日志级别（NORMAL、INFO、WARNING、CRITICAL）  
✅ **高效存储**：循环缓冲区，自动管理内存  
✅ **实时通知**：通过 Qt 信号机制实时分发日志  
✅ **持久化支持**：可配置文件日志记录器  
✅ **国际化友好**：完全支持 Qt 翻译系统  
✅ **灵活过滤**：支持按类型、时间、ID 过滤日志  

### 适用场景

- ✔️ 记录用户操作和业务事件
- ✔️ 追踪系统状态和配置变化
- ✔️ 诊断错误和性能问题
- ✔️ 审计和合规性记录
- ✔️ 开发调试和测试

### 注意事项

- ⚠️ 必须先调用 `Logger::initInstance()` 初始化
- ⚠️ 避免在高频循环中记录日志
- ⚠️ 循环缓冲区有容量限制（20000 条）
- ⚠️ 重要日志应启用文件持久化
- ⚠️ 注意日志消息的国际化处理

通过合理使用 `LogMsg`，可以有效追踪应用程序运行状态、诊断问题和记录重要事件，为软件的开发、测试和维护提供强有力的支持。

---

**文档版本：** 1.0  
**生成日期：** 2025-12-15  
**基于代码版本：** qBittorrent 5.14  
**作者：** AI Assistant  
**适用范围：** qBittorrent 开发者、维护者、技术文档编写者
