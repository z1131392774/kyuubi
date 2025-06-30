# Kyuubi 会话元数据管理补丁可行性报告

## 概述

本报告分析了 `725e20cdd30add023f68a14cc1146d6e7d74a2a6.patch` 补丁在当前 Kyuubi 代码库中的可行性。该补丁旨在将 SQL
会话元数据保存到 MetadataManager 中，实现会话信息的持久化存储。

## 补丁功能分析

### 主要功能

- **会话持久化存储**：将会话元数据保存到 MetadataStore，即使服务重启也能查看会话历史
- **API 层增强**：修改 REST API 从持久化存储而非仅内存获取会话列表
- **生命周期管理**：在会话创建和关闭时自动管理元数据状态

### 涉及文件

1. `SessionsResource.scala` - API 接口层修改
2. `MetadataManager.scala` - 元数据管理器扩展
3. `KyuubiSessionImpl.scala` - 会话实现层集成
4. `KyuubiSessionManager.scala` - 会话管理器增强

## 兼容性评估结果

### ❌ 不能直接合并

由于补丁创建于 2022年11月，距今已过去2年多时间，当前代码库结构发生了重大变化，存在以下主要冲突：

#### 1. SessionsResource.scala 重大冲突

**当前版本实现（现有代码）**：

```scala
@ApiResponse(
  responseCode = "200",
  content = Array(new Content(
    mediaType = MediaType.APPLICATION_JSON,
    array = new ArraySchema(schema = new Schema(implementation = classOf[SessionData])))),
  description = "get the list of all live sessions")
@GET
def sessions(): Seq[SessionData] = {
  sessionManager.allSessions()
    .map(session => ApiUtils.sessionData(session.asInstanceOf[KyuubiSession])).toSeq
}
```

**补丁版本实现（期望修改后）**：

```scala
@GET
def sessions(): Seq[SessionData] = {
  val sessions = sessionManager.getActiveSessionsFromMetadataStore()
  sessions.map { metadata =>
    new SessionData(
      metadata.identifier,
      metadata.username,
      metadata.ipAddress,
      metadata.requestConf.asJava,
      metadata.createTime,
      0,
      0)
  }
}
```

**冲突分析**：

- **构造方式差异**：当前版本使用 `ApiUtils.sessionData()` 工厂方法，补丁版本手动构造 SessionData
- **数据源差异**：当前版本从内存会话 (`allSessions()`) 获取，补丁版本从元数据存储 (`getActiveSessionsFromMetadataStore()`)
  获取
- **类型转换**：当前版本需要转换 `KyuubiSession`，补丁版本直接使用 `Metadata` 对象
- **依赖缺失**：补丁版本需要导入 `KyuubiSessionManager` 类型转换

#### 2. MetadataManager.scala 缺失核心方法

**当前版本状态**：

```scala
// 当前版本中不存在以下方法
def getSessions(state: String, from: Int, size: Int): Seq[Metadata] = {
  // 此方法在当前版本中完全不存在
}
```

**补丁要求添加**：

```scala
def getSessions(state: String, from: Int, size: Int): Seq[Metadata] = {
  val filter = MetadataFilter(state = state)
  withMetadataRequestMetrics(_metadataStore.getMetadataList(filter, from, size, false))
}
```

**冲突分析**：

- **方法缺失**：当前版本的 MetadataManager 主要处理 Batch 操作，缺少会话查询支持
- **过滤器依赖**：需要确保 `MetadataFilter` 支持 `state` 字段过滤
- **存储后端**：需要验证 `_metadataStore.getMetadataList()` 方法的第4个参数含义

#### 3. KyuubiSessionManager.scala 部分兼容

**当前版本已有方法**：

```scala
// 这些方法在当前版本中已存在
def insertMetadata(metadata: Metadata): Unit = {
  /* 已实现 */
}
def updateMetadata(metadata: Metadata): Unit = {
  /* 已实现 */
}
```

**补丁要求新增**：

```scala
def getActiveSessionsFromMetadataStore(): Seq[Metadata] = {
  metadataManager.map(_.getSessions(SessionState.ACTIVE.toString, 0, Int.MaxValue))
    .getOrElse(Seq.empty)
}
```

**冲突分析**：

- **方法缺失**：`getActiveSessionsFromMetadataStore()` 方法不存在
- **依赖链**：该方法依赖 `MetadataManager.getSessions()`，而后者也不存在
- **状态枚举**：需要确认 `SessionState.ACTIVE` 在当前版本中的定义

#### 4. KyuubiSessionImpl.scala 生命周期集成缺失

**补丁要求在 `open()` 方法中添加**：

```scala
// 在会话打开时插入元数据
val metaData = Metadata(
  identifier = handle.identifier.toString,
  sessionType = sessionType,
  realUser = user,
  username = user,
  ipAddress = ipAddress,
  kyuubiInstance = KyuubiRestFrontendService.getConnectionUrl,
  state = SessionState.ACTIVE.toString,
  requestName = engine.defaultEngineName,
  requestConf = normalizedConf,
  createTime = createTime,
  engineType = sessionConf.get(ENGINE_TYPE))
sessionManager.insertMetadata(metaData)
```

**补丁要求在 `close()` 方法中添加**：

```scala
// 在会话关闭时更新元数据状态
val metadataToUpdate = Metadata(
  identifier = handle.identifier.toString,
  state = SessionState.TERMINATED.toString,
  endTime = lastAccessTime)
sessionManager.updateMetadata(metadataToUpdate)
```

**冲突分析**：

- **导入缺失**：需要导入 `KyuubiRestFrontendService`、`Metadata` 等类
- **字段依赖**：`Metadata` 构造需要的字段可能在当前版本中有变化
- **生命周期钩子**：需要确保在正确的位置插入元数据管理代码

#### 5. 类型定义和导入差异

**当前版本导入**：

```scala
import org.apache.kyuubi.server.api.{ApiRequestContext, ApiUtils}
import org.apache.kyuubi.session.{KyuubiSession, SessionHandle}
```

**补丁版本需要额外导入**：

```scala
import org.apache.kyuubi.session.{KyuubiSession, KyuubiSessionManager, SessionHandle}
import org.apache.kyuubi.server.KyuubiRestFrontendService
import org.apache.kyuubi.server.metadata.api.Metadata
```

**冲突分析**：

- **类型转换**：补丁需要将 `sessionManager` 强制转换为 `KyuubiSessionManager`
- **服务依赖**：`KyuubiRestFrontendService.getConnectionUrl` 方法可能不存在或签名已变
- **元数据API**：`Metadata` 类的构造函数签名可能已发生变化

### 兼容性风险等级评估

#### 🔴 高风险冲突

1. **API行为变更**：从内存数据源切换到持久化数据源，可能影响现有客户端
2. **性能影响**：每次API调用都需要数据库查询，而非内存访问
3. **数据一致性**：内存会话和持久化会话可能出现不同步

#### 🟡 中风险冲突

1. **方法签名变化**：`Metadata` 构造函数可能需要适配当前版本
2. **依赖链断裂**：多个缺失的方法需要按顺序实现
3. **类型安全**：强制类型转换可能在运行时失败

#### 🟢 低风险冲突

1. **导入语句**：可以通过添加必要的 import 解决
2. **代码风格**：可以适配当前项目的编码规范

### 具体冲突代码位置

| 文件                           | 行号范围       | 冲突类型   | 解决复杂度 |
|------------------------------|------------|--------|-------|
| `SessionsResource.scala`     | 51-65      | 重构现有方法 | 高     |
| `MetadataManager.scala`      | 156+       | 新增方法   | 中     |
| `KyuubiSessionManager.scala` | 262+       | 新增方法   | 中     |
| `KyuubiSessionImpl.scala`    | 105+, 225+ | 生命周期集成 | 高     |

## 适配实施方案

### 总体策略

采用**分阶段重构**方式，按依赖关系逐步实现功能，确保每个阶段都能独立测试和验证。

### 🔧 阶段1：MetadataManager 扩展

#### 1.1 目标

在 MetadataManager 中添加会话查询支持，为后续功能提供数据访问基础。

#### 1.2 实现代码

**文件**: `kyuubi-server/src/main/scala/org/apache/kyuubi/server/metadata/MetadataManager.scala`

**在现有类中添加以下方法**：

```scala
/**
 * Get sessions by state with pagination support
 * @param state Session state filter (e.g., "ACTIVE", "TERMINATED")
 * @param from  Start index for pagination
 * @param size  Maximum number of results to return
 * @return Sequence of session metadata
 */
def getSessions(state: String, from: Int, size: Int): Seq[Metadata] = {
  require(state != null && state.nonEmpty, "State parameter cannot be null or empty")
  require(from >= 0, "From parameter must be non-negative")
  require(size > 0, "Size parameter must be positive")

  val filter = MetadataFilter(
    state = Some(state),
    sessionType = Some("SQL") // 过滤SQL会话
  )
  withMetadataRequestMetrics(_metadataStore.getMetadataList(filter, from, size, false))
}

/**
 * Get all active sessions without pagination
 * @return Sequence of active session metadata
 */
def getActiveSessions(): Seq[Metadata] = {
  getSessions("ACTIVE", 0, Int.MaxValue)
}

/**
 * Get session count by state
 * @param state Session state to count
 * @return Number of sessions in the specified state
 */
def getSessionCount(state: String): Int = {
  val filter = MetadataFilter(state = Some(state))
  withMetadataRequestMetrics(_metadataStore.getMetadataCount(filter))
}
```

#### 1.3 依赖验证

**检查 MetadataFilter 支持**：

```scala
// 验证 MetadataFilter 是否支持 state 字段
// 如果不支持，需要先扩展 MetadataFilter 类
case class MetadataFilter(
                           identifier: Option[String] = None,
                           sessionType: Option[String] = None,
                           state: Option[String] = None, // 确保此字段存在
                           // ... 其他字段
                         )
```

#### 1.4 测试验证

```scala
// 单元测试示例
class MetadataManagerSessionSuite extends KyuubiFunSuite {
  test("getSessions should return filtered results") {
    val manager = new MetadataManager()
    manager.initialize(conf)

    // 测试基本查询
    val activeSessions = manager.getSessions("ACTIVE", 0, 10)
    assert(activeSessions.forall(_.state == "ACTIVE"))

    // 测试分页
    val firstPage = manager.getSessions("ACTIVE", 0, 5)
    val secondPage = manager.getSessions("ACTIVE", 5, 5)
    assert(firstPage.size <= 5)
    assert(secondPage.size <= 5)
  }
}
```

### 🔧 阶段2：KyuubiSessionManager 增强

#### 2.1 目标

在会话管理器中添加元数据存储访问方法，连接内存会话和持久化存储。

#### 2.2 实现代码

**文件**: `kyuubi-server/src/main/scala/org/apache/kyuubi/session/KyuubiSessionManager.scala`

**在现有类中添加以下方法**：

```scala
/**
 * Get active sessions from metadata store
 * 从元数据存储获取活跃会话，而非内存
 */
def getActiveSessionsFromMetadataStore(): Seq[Metadata] = {
  metadataManager match {
    case Some(manager) =>
      try {
        manager.getSessions("ACTIVE", 0, Int.MaxValue)
      } catch {
        case e: Exception =>
          warn(s"Failed to get sessions from metadata store: ${e.getMessage}")
          Seq.empty
      }
    case None =>
      warn("MetadataManager is not available, returning empty session list")
      Seq.empty
  }
}

/**
 * Get sessions from metadata store with pagination
 */
def getSessionsFromMetadataStore(state: String, from: Int, size: Int): Seq[Metadata] = {
  metadataManager match {
    case Some(manager) => manager.getSessions(state, from, size)
    case None => Seq.empty
  }
}

/**
 * Sync memory sessions to metadata store
 * 同步内存中的会话信息到元数据存储
 */
def syncSessionsToMetadataStore(): Unit = {
  metadataManager.foreach { manager =>
    allSessions().foreach { session =>
      val kyuubiSession = session.asInstanceOf[KyuubiSession]
      try {
        val metadata = createSessionMetadata(kyuubiSession)
        manager.insertMetadata(metadata)
      } catch {
        case e: Exception =>
          warn(s"Failed to sync session ${session.handle} to metadata store: ${e.getMessage}")
      }
    }
  }
}

/**
 * Create metadata object from KyuubiSession
 */
private def createSessionMetadata(session: KyuubiSession): Metadata = {
  Metadata(
    identifier = session.handle.identifier.toString,
    sessionType = session.sessionType.toString,
    realUser = session.user,
    username = session.user,
    ipAddress = session.ipAddress,
    kyuubiInstance = getKyuubiInstance(),
    state = if (session.isOpen) "ACTIVE" else "TERMINATED",
    requestName = session.engine.defaultEngineName,
    requestConf = session.conf.asScala.toMap,
    createTime = session.createTime,
    endTime = if (session.isOpen) 0L else session.lastAccessTime,
    engineType = session.sessionConf.get(ENGINE_TYPE).getOrElse("SPARK_SQL")
  )
}

/**
 * Get Kyuubi instance identifier
 */
private def getKyuubiInstance(): String = {
  // 适配当前版本的获取方式
  try {
    // 尝试获取连接URL，如果失败则使用默认值
    conf.get(KyuubiConf.FRONTEND_CONNECTION_URL_USE_IP) match {
      case true => s"${Utils.findLocalInetAddress.getHostAddress}:${conf.get(KyuubiConf.FRONTEND_THRIFT_BINARY_BIND_PORT)}"
      case false => s"${Utils.findLocalInetAddress.getHostName}:${conf.get(KyuubiConf.FRONTEND_THRIFT_BINARY_BIND_PORT)}"
    }
  } catch {
    case _: Exception => "unknown"
  }
}
```

#### 2.3 配置项添加

```scala
// 在 KyuubiConf 中添加控制开关
object KyuubiConf {
  val SESSION_METADATA_STORE_ENABLED: ConfigEntry[Boolean] = buildConf("kyuubi.session.metadata.store.enabled")
    .doc("Enable session metadata storage for persistence")
    .version("1.9.0")
    .booleanConf
    .createWithDefault(false)
}
```

#### 2.4 测试验证

```scala
test("getActiveSessionsFromMetadataStore should return persisted sessions") {
  val sessionManager = new KyuubiSessionManager("test")
  sessionManager.initialize(conf)

  // 创建测试会话
  val handle = sessionManager.openSession(
    TProtocolVersion.HIVE_CLI_SERVICE_PROTOCOL_V1,
    "test_user", "", "localhost", Map.empty
  )

  // 验证可以从元数据存储获取
  val sessions = sessionManager.getActiveSessionsFromMetadataStore()
  assert(sessions.exists(_.identifier == handle.identifier.toString))
}
```

### 🔧 阶段3：KyuubiSessionImpl 生命周期集成

#### 3.1 目标

在会话的创建和关闭生命周期中集成元数据管理。

#### 3.2 实现代码

**文件**: `kyuubi-server/src/main/scala/org/apache/kyuubi/session/KyuubiSessionImpl.scala`

**3.2.1 添加必要的导入**：

```scala
import org.apache.kyuubi.server.metadata.api.Metadata
import org.apache.kyuubi.config.KyuubiConf._
```

**3.2.2 在 `open()` 方法中添加元数据插入逻辑**：

```scala
override def open(): Unit = {
  // ... 现有的 open 逻辑 ...

  checkSessionAccessPathURIs()

  // 在会话成功打开后插入元数据
  if (sessionConf.get(SESSION_METADATA_STORE_ENABLED)) {
    insertSessionMetadata()
  }

  // we should call super.open before running launch engine operation
  super.open()

  // ... 现有的引擎启动逻辑 ...
}

/**
 * Insert session metadata to store
 */
private def insertSessionMetadata(): Unit = {
  try {
    val metaData = Metadata(
      identifier = handle.identifier.toString,
      sessionType = sessionType.toString,
      realUser = user,
      username = user,
      ipAddress = ipAddress,
      kyuubiInstance = getKyuubiInstanceUrl(),
      state = "ACTIVE",
      requestName = engine.defaultEngineName,
      requestConf = normalizedConf,
      createTime = createTime,
      engineType = sessionConf.get(ENGINE_TYPE).getOrElse("SPARK_SQL")
    )
    sessionManager.asInstanceOf[KyuubiSessionManager].insertMetadata(metaData)
    info(s"Session metadata inserted for ${handle.identifier}")
  } catch {
    case e: Exception =>
      warn(s"Failed to insert session metadata for ${handle.identifier}: ${e.getMessage}")
    // 不抛出异常，避免影响会话创建
  }
}

/**
 * Get Kyuubi instance URL
 */
private def getKyuubiInstanceUrl(): String = {
  try {
    // 适配当前版本的实现
    sessionManager.asInstanceOf[KyuubiSessionManager].getKyuubiInstance()
  } catch {
    case _: Exception => "unknown"
  }
}
```

**3.2.3 在 `close()` 方法中添加状态更新逻辑**：

```scala
override def close(): Unit = {
  // 在关闭前更新元数据状态
  if (sessionConf.get(SESSION_METADATA_STORE_ENABLED)) {
    updateSessionMetadata()
  }

  super.close()
  sessionManager.credentialsManager.removeSessionCredentialsEpoch(handle.identifier.toString)

  // ... 现有的客户端关闭逻辑 ...
}

/**
 * Update session metadata on close
 */
private def updateSessionMetadata(): Unit = {
  try {
    val metadataToUpdate = Metadata(
      identifier = handle.identifier.toString,
      state = "TERMINATED",
      endTime = lastAccessTime
    )
    sessionManager.asInstanceOf[KyuubiSessionManager].updateMetadata(metadataToUpdate)
    info(s"Session metadata updated for ${handle.identifier}")
  } catch {
    case e: Exception =>
      warn(s"Failed to update session metadata for ${handle.identifier}: ${e.getMessage}")
    // 不抛出异常，避免影响会话关闭
  }
}
```

#### 3.3 错误处理和监控

```scala
/**
 * Async metadata operation with error handling
 */
private def executeMetadataOperation(operation: => Unit): Unit = {
  if (sessionConf.get(SESSION_METADATA_STORE_ENABLED)) {
    Future {
      try {
        operation
      } catch {
        case e: Exception =>
          MetricsSystem.incrementCounter(MetricsConstants.METADATA_OPERATION_FAIL)
          error(s"Metadata operation failed for session ${handle.identifier}", e)
      }
    }(ExecutionContext.global)
  }
}
```

### 🔧 阶段4：SessionsResource API适配

#### 4.1 目标

修改 REST API 实现，支持从元数据存储获取会话列表，同时保持向后兼容。

#### 4.2 实现代码

**文件**: `kyuubi-server/src/main/scala/org/apache/kyuubi/server/api/v1/SessionsResource.scala`

**4.2.1 添加配置控制的会话列表实现**：

```scala
@ApiResponse(
  responseCode = "200",
  content = Array(new Content(
    mediaType = MediaType.APPLICATION_JSON,
    array = new ArraySchema(schema = new Schema(implementation = classOf[SessionData])))),
  description = "get the list of all live sessions")
@GET
def sessions(): Seq[SessionData] = {
  val useMetadataStore = sessionManager.conf.get(KyuubiConf.SESSION_METADATA_STORE_ENABLED)

  if (useMetadataStore) {
    getSessionsFromMetadataStore()
  } else {
    getSessionsFromMemory()
  }
}

/**
 * Get sessions from metadata store (new implementation)
 */
private def getSessionsFromMetadataStore(): Seq[SessionData] = {
  try {
    val kyuubiSessionManager = sessionManager.asInstanceOf[KyuubiSessionManager]
    val sessions = kyuubiSessionManager.getActiveSessionsFromMetadataStore()

    sessions.map { metadata =>
      // 使用 ApiUtils 保持一致的数据格式
      ApiUtils.sessionDataFromMetadata(metadata)
    }
  } catch {
    case e: Exception =>
      error(s"Failed to fetch sessions from metadata store: ${e.getMessage}")
      Seq.empty
  }
}

/**
 * Get sessions from memory (legacy implementation)
 */
private def getSessionsFromMemory(): Seq[SessionData] = {
  sessionManager.allSessions().map { session =>
    ApiUtils.sessionData(session.asInstanceOf[KyuubiSession])
  }.toSeq
}
```

#### 4.2.2 添加ApiUtils扩展方法**：

```scala
// 在 ApiUtils 对象中添加新方法
object ApiUtils {
  // ... 现有方法 ...

  /**
   * Convert metadata to SessionData
   */
  def sessionDataFromMetadata(metadata: Metadata): SessionData = {
    new SessionData(
      metadata.identifier,
      metadata.username,
      metadata.ipAddress,
      metadata.requestConf.asJava,
      metadata.createTime,
      metadata.endTime - metadata.createTime, // duration
      0L // noOperationTime - 从元数据无法获取，设为0
    )
  }
}
```

#### 4.2.3 向后兼容性保证**：

```scala
// 添加配置迁移检查
private def validateMetadataStoreConfiguration(): Unit = {
  val isEnabled = sessionManager.conf.get(KyuubiConf.SESSION_METADATA_STORE_ENABLED)
  if (isEnabled) {
    // 验证元数据存储配置是否正确
    sessionManager.asInstanceOf[KyuubiSessionManager].metadataManager match {
      case Some(_) =>
        info("Metadata store is properly configured for session persistence")
      case None =>
        warn("Session metadata store is enabled but MetadataManager is not available, falling back to memory-based sessions")
    }
  }
}
```

#### 4.3 测试验证

```scala
class SessionsResourceMetadataSuite extends KyuubiFunSuite {
  test("sessions API should work with metadata store enabled") {
    withKyuubiConf(Map(
      KyuubiConf.SESSION_METADATA_STORE_ENABLED.key -> "true"
    )) { conf =>
      val sessionManager = new KyuubiSessionManager("test")
      sessionManager.initialize(conf)

      val resource = new SessionsResource()

      // 创建测试会话
      val handle = sessionManager.openSession(
        TProtocolVersion.HIVE_CLI_SERVICE_PROTOCOL_V1,
        "test_user", "", "localhost", Map.empty
      )

      // 验证API返回正确数据
      val sessions = resource.sessions()
      assert(sessions.nonEmpty)
      assert(sessions.exists(_.getIdentifier == handle.identifier.toString))
    }
  }

  test("sessions API should fallback gracefully when metadata store fails") {
    // 测试元数据存储失败时的降级行为
    val mockSessionManager = mock(classOf[KyuubiSessionManager])
    when(mockSessionManager.conf.get(KyuubiConf.SESSION_METADATA_STORE_ENABLED)).thenReturn(true)
    when(mockSessionManager.getActiveSessionsFromMetadataStore()).thenThrow(new RuntimeException("DB connection failed"))

    val resource = new SessionsResource()

    // 应该返回空列表而不是抛出异常
    val sessions = resource.sessions()
    assert(sessions.isEmpty)
  }
}
```

## 风险评估

### 高风险项

- **数据一致性**：内存会话与持久化会话可能不同步
- **性能影响**：每次会话操作都需要数据库访问
- **事务管理**：元数据操作需要适当的错误处理

### 中风险项

- **API 兼容性**：现有客户端可能依赖当前 API 行为
- **配置依赖**：需要确保 MetadataStore 正确配置

### 低风险项

- **代码维护性**：新增代码遵循现有架构模式
- **测试覆盖**：现有测试框架可以支持新功能

## 建议

### 推荐方案

1. **重新实现**：基于当前代码结构重新实现该功能，而非直接应用补丁
2. **分阶段实施**：按照上述4个阶段逐步实现，每个阶段独立测试
3. **向后兼容**：保持现有 API 行为，新增配置开关控制功能启用

### 实施优先级

1. **P0**：MetadataManager 和 KyuubiSessionManager 扩展
2. **P1**：KyuubiSessionImpl 生命周期集成
3. **P2**：SessionsResource API 适配
4. **P3**：配置和监控集成

### 测试建议

1. **单元测试**：覆盖每个新增方法的基本功能
2. **集成测试**：验证会话生命周期的元数据管理
3. **性能测试**：评估持久化操作对系统性能的影响
4. **兼容性测试**：确保现有客户端正常工作

## 补丁原始内容

```diff
From 725e20cdd30add023f68a14cc1146d6e7d74a2a6 Mon Sep 17 00:00:00 2001
From: Tianlin Liao <tiliao@ebay.com>
Date: Thu, 3 Nov 2022 15:45:32 +0800
Subject: [PATCH] [KYUUBI #3723] save the SQL session metadata into
 MetadataManager

---
 .../server/api/v1/SessionsResource.scala      | 24 +++++++++----------
 .../server/metadata/MetadataManager.scala     |  5 ++++
 .../kyuubi/session/KyuubiSessionImpl.scala    | 24 +++++++++++++++++++
 .../kyuubi/session/KyuubiSessionManager.scala |  5 ++++
 4 files changed, 46 insertions(+), 12 deletions(-)

diff --git a/kyuubi-server/src/main/scala/org/apache/kyuubi/server/api/v1/SessionsResource.scala b/kyuubi-server/src/main/scala/org/apache/kyuubi/server/api/v1/SessionsResource.scala
index 2a570ccbd8a..b40cd09333c 100644
--- a/kyuubi-server/src/main/scala/org/apache/kyuubi/server/api/v1/SessionsResource.scala
+++ b/kyuubi-server/src/main/scala/org/apache/kyuubi/server/api/v1/SessionsResource.scala
@@ -35,14 +35,13 @@ import org.apache.kyuubi.client.api.v1.dto._
 import org.apache.kyuubi.events.KyuubiEvent
 import org.apache.kyuubi.operation.OperationHandle
 import org.apache.kyuubi.server.api.ApiRequestContext
-import org.apache.kyuubi.session.KyuubiSession
-import org.apache.kyuubi.session.SessionHandle
+import org.apache.kyuubi.session.{KyuubiSession, KyuubiSessionManager, SessionHandle}
 
 @Tag(name = "Session")
 @Produces(Array(MediaType.APPLICATION_JSON))
 private[v1] class SessionsResource extends ApiRequestContext with Logging {
   implicit def toSessionHandle(str: String): SessionHandle = SessionHandle.fromUUID(str)
-  private def sessionManager = fe.be.sessionManager
+  private def sessionManager = fe.be.sessionManager.asInstanceOf[KyuubiSessionManager]
 
   @ApiResponse(
     responseCode = "200",
@@ -52,16 +51,17 @@ private[v1] class SessionsResource extends ApiRequestContext with Logging {
     description = "get the list of all live sessions")
   @GET
   def sessions(): Seq[SessionData] = {
-    sessionManager.allSessions().map { session =>
+    val sessions = sessionManager.getActiveSessionsFromMetadataStore()
+    sessions.map { metadata =>
       new SessionData(
-        session.handle.identifier.toString,
-        session.user,
-        session.ipAddress,
-        session.conf.asJava,
-        session.createTime,
-        session.lastAccessTime - session.createTime,
-        session.getNoOperationTime)
-    }.toSeq
+        metadata.identifier,
+        metadata.username,
+        metadata.ipAddress,
+        metadata.requestConf.asJava,
+        metadata.createTime,
+        0,
+        0)
+    }
   }
 
   @ApiResponse(
diff --git a/kyuubi-server/src/main/scala/org/apache/kyuubi/server/metadata/MetadataManager.scala b/kyuubi-server/src/main/scala/org/apache/kyuubi/server/metadata/MetadataManager.scala
index c7946950c20..25e0e6f8bcf 100644
--- a/kyuubi-server/src/main/scala/org/apache/kyuubi/server/metadata/MetadataManager.scala
+++ b/kyuubi-server/src/main/scala/org/apache/kyuubi/server/metadata/MetadataManager.scala
@@ -156,6 +156,11 @@ class MetadataManager extends AbstractService("MetadataManager") {
     withMetadataRequestMetrics(_metadataStore.getMetadataList(filter, from, size, true))
   }
 
+  def getSessions(state: String, from: Int, size: Int): Seq[Metadata] = {
+    val filter = MetadataFilter(state = state)
+    withMetadataRequestMetrics(_metadataStore.getMetadataList(filter, from, size, false))
+  }
+
   def updateMetadata(metadata: Metadata, retryOnError: Boolean = true): Unit = {
     try {
       withMetadataRequestMetrics(_metadataStore.updateMetadata(metadata))
diff --git a/kyuubi-server/src/main/scala/org/apache/kyuubi/session/KyuubiSessionImpl.scala b/kyuubi-server/src/main/scala/org/apache/kyuubi/session/KyuubiSessionImpl.scala
index c1e17deb3ea..43039a01869 100644
--- a/kyuubi-server/src/main/scala/org/apache/kyuubi/session/KyuubiSessionImpl.scala
+++ b/kyuubi-server/src/main/scala/org/apache/kyuubi/session/KyuubiSessionImpl.scala
@@ -34,6 +34,8 @@ import org.apache.kyuubi.metrics.MetricsConstants._
 import org.apache.kyuubi.metrics.MetricsSystem
 import org.apache.kyuubi.operation.{Operation, OperationHandle}
 import org.apache.kyuubi.operation.log.OperationLog
+import org.apache.kyuubi.server.KyuubiRestFrontendService
+import org.apache.kyuubi.server.metadata.api.Metadata
 import org.apache.kyuubi.service.authentication.InternalSecurityAccessor
 import org.apache.kyuubi.session.SessionType.SessionType
 
@@ -103,6 +105,21 @@ class KyuubiSessionImpl(
 
     checkSessionAccessPathURIs()
 
+    // insert session to metastore
+    val metaData = Metadata(
+      identifier = handle.identifier.toString,
+      sessionType = sessionType,
+      realUser = user,
+      username = user,
+      ipAddress = ipAddress,
+      kyuubiInstance = KyuubiRestFrontendService.getConnectionUrl,
+      state = SessionState.ACTIVE.toString,
+      requestName = engine.defaultEngineName,
+      requestConf = normalizedConf,
+      createTime = createTime,
+      engineType = sessionConf.get(ENGINE_TYPE))
+    sessionManager.insertMetadata(metaData)
+
     // we should call super.open before running launch engine operation
     super.open()
 
@@ -208,6 +225,13 @@ class KyuubiSessionImpl(
   override def close(): Unit = {
     super.close()
     sessionManager.credentialsManager.removeSessionCredentialsEpoch(handle.identifier.toString)
+
+    val metadataToUpdate = Metadata(
+      identifier = handle.identifier.toString,
+      state = SessionState.TERMINATED.toString,
+      endTime = lastAccessTime)
+    sessionManager.updateMetadata(metadataToUpdate)
+
     try {
       if (_client != null) _client.closeSession()
     } finally {
diff --git a/kyuubi-server/src/main/scala/org/apache/kyuubi/session/KyuubiSessionManager.scala b/kyuubi-server/src/main/scala/org/apache/kyuubi/session/KyuubiSessionManager.scala
index ff051019951..563dfb2bcb6 100644
--- a/kyuubi-server/src/main/scala/org/apache/kyuubi/session/KyuubiSessionManager.scala
+++ b/kyuubi-server/src/main/scala/org/apache/kyuubi/session/KyuubiSessionManager.scala
@@ -262,6 +262,11 @@ class KyuubiSessionManager private (name: String) extends SessionManager(name) {
     }
   }
 
+  def getActiveSessionsFromMetadataStore(): Seq[Metadata] = {
+    metadataManager.map(_.getSessions(SessionState.ACTIVE.toString, 0, Int.MaxValue))
+      .getOrElse(Seq.empty)
+  }
+
   override protected def isServer: Boolean = true
 
   private def initSessionLimiter(conf: KyuubiConf): Unit = {
```

## 结论

该补丁提供了有价值的会话持久化功能，但由于代码库的重大变化，**不能直接应用**
。建议采用重新实现的方式，基于当前代码结构实现相同功能。预计需要2-3个开发周期完成完整实施，包括开发、测试和部署。

### 最终建议

1. **不要直接应用补丁**：由于代码结构差异太大，强制应用会引入大量错误
2. **采用分阶段重构**：按照本报告提出的4个阶段逐步实现功能
3. **优先级排序**：先实现核心的元数据管理功能，后实现API层适配
4. **充分测试**：每个阶段都需要完整的单元测试和集成测试
5. **配置化控制**：通过配置开关控制功能启用，确保向后兼容

### 价值评估

尽管实施复杂度较高，该功能具有显著价值：

- 提升系统可观测性和可维护性
- 支持会话信息的持久化和恢复
- 为未来的会话管理功能提供基础

**总体评估：功能有价值，但需要重新实现而非直接合并。**

## 实施计划详细步骤

#### Phase 1: 基础设施准备 (第1-2周)

1. **环境配置**
   ```bash
   # 1. 创建功能分支
   git checkout -b feature/session-metadata-persistence
   
   # 2. 备份相关文件
   cp kyuubi-server/src/main/scala/org/apache/kyuubi/server/metadata/MetadataManager.scala MetadataManager.scala.backup
   cp kyuubi-server/src/main/scala/org/apache/kyuubi/session/KyuubiSessionManager.scala KyuubiSessionManager.scala.backup
   ```

2. **依赖检查和准备**
   ```scala
   // 验证当前MetadataFilter是否支持state字段
   // 如不支持，先扩展MetadataFilter
   ```

#### Phase 2: 核心实现 (第3-5周)

1. **阶段1实施** (第3周)
    - 实现MetadataManager扩展
    - 单元测试验证
    - 集成测试确认

2. **阶段2实施** (第4周)
    - 实现KyuubiSessionManager增强
    - 配置项添加
    - 测试验证

3. **阶段3实施** (第5周)
    - KyuubiSessionImpl生命周期集成
    - 错误处理完善
    - 性能测试

#### Phase 3: API集成 (第6周)

1. **阶段4实施**
    - SessionsResource API适配
    - ApiUtils扩展
    - 兼容性测试

#### Phase 4: 测试和优化 (第7-8周)

1. **完整集成测试**
2. **性能基准测试**
3. **生产环境验证**

### 风险评估和缓解策略

#### 高风险项及缓解方案

**1. 数据一致性风险**

- **风险**: 内存会话与持久化会话数据不同步
- **缓解**:
  ```scala
  // 实现同步机制
  def reconcileSessionData(): Unit = {
    val memorySessionIds = allSessions().map(_.handle.identifier.toString).toSet
    val persistedSessionIds = getActiveSessionsFromMetadataStore().map(_.identifier).toSet
    
    // 处理差异
    val orphanedInMemory = memorySessionIds -- persistedSessionIds
    val orphanedInStore = persistedSessionIds -- memorySessionIds
    
    orphanedInMemory.foreach { sessionId =>
      warn(s"Session $sessionId exists in memory but not in store, syncing...")
      // 同步到存储
    }
    
    orphanedInStore.foreach { sessionId =>
      warn(s"Session $sessionId exists in store but not in memory, cleaning...")
      // 清理存储中的记录
    }
  }
  ```

**2. 性能影响风险**

- **风险**: 每次API调用都访问数据库，性能下降
- **缓解**:
  ```scala
  // 实现缓存机制
  private val sessionCache = CacheBuilder.newBuilder()
    .maximumSize(1000)
    .expireAfterWrite(30, TimeUnit.SECONDS)
    .build[String, Seq[Metadata]]()
    
  def getCachedSessions(): Seq[Metadata] = {
    sessionCache.get("active_sessions", new Callable[Seq[Metadata]] {
      override def call(): Seq[Metadata] = {
        metadataManager.get.getSessions("ACTIVE", 0, Int.MaxValue)
      }
    })
  }
  ```

**3. 向后兼容性风险**

- **风险**: 现有客户端可能依赖特定的API行为
- **缓解**:
  ```scala
  // 实现特性开关和渐进式迁移
  private def shouldUseMetadataStore(): Boolean = {
    val isEnabled = conf.get(KyuubiConf.SESSION_METADATA_STORE_ENABLED)
    val hasValidManager = metadataManager.isDefined
    
    isEnabled && hasValidManager
  }
  ```

#### 中风险项及缓解方案

**1. 配置复杂性**

- **风险**: 用户配置错误导致功能异常
- **缓解**: 提供配置验证和自动修复机制

**2. 依赖版本冲突**

- **风险**: 新功能与现有组件版本不兼容
- **缓解**: 详细的版本兼容性测试

### 成功指标

#### 功能指标

- [ ] 所有现有单元测试通过
- [ ] 新增单元测试覆盖率 ≥ 80%
- [ ] 集成测试全部通过
- [ ] API兼容性测试通过

#### 性能指标

- [ ] 会话列表API响应时间 < 500ms (P95)
- [ ] 会话创建性能下降 < 10%
- [ ] 内存使用增长 < 5%

#### 可靠性指标

- [ ] 元数据存储故障时系统正常降级
- [ ] 数据一致性检查通过率 100%
- [ ] 零生产事故部署

### 回滚计划

#### 即时回滚 (配置级别)

```scala
// 通过配置快速禁用功能
kyuubi.session.metadata.store.enabled = false
```

#### 代码回滚 (紧急情况)

```bash
# 回滚到原始实现
git revert <commit-hash>
git push origin main
```

#### 数据清理 (完全回滚)

```sql
-- 清理元数据表中的会话记录
DELETE FROM metadata_store WHERE session_type = 'SQL';
```

### 监控和告警

#### 关键监控指标

```scala
// 添加自定义指标
object SessionMetadataMetrics {
  val METADATA_OPERATION_SUCCESS = Counter.build()
    .name("session_metadata_operations_total")
    .help("Total session metadata operations")
    .labelNames("operation", "status")
    .register()

  val METADATA_OPERATION_DURATION = Histogram.build()
    .name("session_metadata_operation_duration_seconds")
    .help("Session metadata operation duration")
    .labelNames("operation")
    .register()
}
```

#### 告警规则

```yaml
- alert: SessionMetadataHighFailureRate
  expr: rate(session_metadata_operations_total{status="failed"}[5m]) > 0.1
  for: 2m
  labels:
    severity: warning
  annotations:
    summary: "High session metadata operation failure rate"

- alert: SessionMetadataSlowOperations
  expr: histogram_quantile(0.95, session_metadata_operation_duration_seconds) > 1.0
  for: 5m
  labels:
    severity: critical
  annotations:
    summary: "Session metadata operations are slow"
```