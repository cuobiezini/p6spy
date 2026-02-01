# P6Spy 核心功能与逻辑分析

本文档旨在梳理 P6Spy 的核心功能、基本用法和内部工作流程。

## 1. P6Spy 是什么？

P6Spy 是一个针对 Java 应用程序的 JDBC 代理框架。它能够在不修改任何应用程序代码的情况下，无缝地拦截数据库操作（如 SQL 查询、事务等）。通过拦截，P6Spy 可以实现以下核心功能：

*   **SQL 日志记录**：记录所有执行的 SQL 语句，包括参数、执行时间、结果等。这是最常用、最核心的功能。
*   **性能监控**：分析 SQL 执行耗时，帮助开发者定位慢查询。
*   **数据源代理**：可以代理任何符合 JDBC 规范的数据源。
*   **可扩展性**：提供事件监听机制，允许开发者编写自定义的监听器来处理 JDBC 事件，实现如 SQL 审计、自定义日志格式、性能告警等高级功能。

## 2. 如何使用 P6Spy？

使用 P6Spy 通常涉及以下两个步骤：

### 步骤 1：修改 JDBC 配置

将应用程序的 JDBC 配置修改为使用 P6Spy 提供的代理驱动。

*   **JDBC Driver**: 从原始的数据库驱动（如 `com.mysql.cj.jdbc.Driver`）修改为 `com.p6spy.engine.spy.P6SpyDriver`。
*   **JDBC URL**: 在原始的 JDBC URL 前面加上 `p6spy:` 前缀。

**示例 (以 MySQL 为例):**

**原始配置:**
```
jdbc.driver=com.mysql.cj.jdbc.Driver
jdbc.url=jdbc:mysql://localhost:3306/mydatabase
```

**修改后的配置:**
```
jdbc.driver=com.p6spy.engine.spy.P6SpyDriver
jdbc.url=jdbc:p6spy:mysql://localhost:3306/mydatabase
```

### 步骤 2：配置 P6Spy

在项目的 `src/main/resources` 目录下创建一个名为 `spy.properties` 的文件，用于配置 P6Spy 的行为。

**常用配置项:**

*   `realdriver`: 真实的数据库驱动类名。如果留空，P6Spy 会自动从 classpath 中查找合适的驱动。
*   `modulelist`: 指定要启用的 P6Spy 模块。最核心的模块是 `P6LogFactory`，用于记录 SQL。
*   `logMessageFormat`: 自定义日志输出的格式。
*   `logfile`: 指定日志文件的输出位置。默认为 `spy.log`。
*   `append`: 是否以追加模式写入日志文件。

**示例 `spy.properties`:**
```properties
# 指定 P6LogFactory 模块，用于记录 SQL
modulelist=com.p6spy.engine.logging.P6LogFactory

# 指定真实的数据库驱动（可选，P6Spy 会自动检测）
# realdriver=com.mysql.cj.jdbc.Driver

# 日志格式
logMessageFormat=com.p6spy.engine.spy.appender.MultiLineFormat

# 日志文件路径
logfile=logs/spy.log

# 是否追加日志
append=true
```

完成以上配置后，启动应用程序，P6Spy 就会自动开始工作，并将 SQL 日志输出到指定的文件中。

## 3. 核心逻辑梳理

P6Spy 的核心是基于 **装饰器模式 (Decorator Pattern)** 和 **Java SPI (Service Provider Interface)** 机制实现的。

### 3.1. 启动和驱动注册 (P6SpyDriver)

1.  **入口点**: `com.p6spy.engine.spy.P6SpyDriver` 是 P6Spy 的 JDBC 驱动实现。
2.  **自动注册**: 利用 Java 的 `java.sql.Driver` 机制，`P6SpyDriver` 在类加载时通过静态代码块将自身注册到 `DriverManager` 中。
3.  **URL 匹配**: `P6SpyDriver` 只接受以 `jdbc:p6spy:` 开头的 URL。
4.  **寻找真实驱动**: 当应用程序请求一个连接时，`P6SpyDriver` 的 `connect` 方法被调用。它会剥离 `p6spy:` 前缀，得到真实的 JDBC URL，然后在 `DriverManager` 中找到能够处理这个真实 URL 的实际数据库驱动（例如 `com.mysql.cj.jdbc.Driver`）。

### 3.2. 连接和语句的包装 (Wrapper)

1.  **包装 Connection**: `P6SpyDriver` 在获取到真实的 `Connection` 对象后，并不会直接返回它，而是使用 `com.p6spy.engine.wrapper.ConnectionWrapper` 将其包装起来。
2.  **包装 Statement**: 当应用程序在被包装的 `ConnectionWrapper` 上调用 `createStatement()` 或 `prepareStatement()` 时，`ConnectionWrapper` 并不会直接调用真实连接的方法，而是返回一个被包装过的 `StatementWrapper` 或 `PreparedStatementWrapper`。
3.  **层层包装**: 这种包装是层层递进的，`Connection` -> `Statement` -> `ResultSet`，所有核心的 JDBC 对象都被 P6Spy 的包装类（Wrapper）所代理。

### 3.3. 事件拦截与监听 (JdbcEventListener)

1.  **事件定义**: `com.p6spy.engine.event.JdbcEventListener` 是一个抽象类，定义了所有可以被监听的 JDBC 事件，例如 `onBeforeExecuteQuery`（执行查询前）、`onAfterExecuteQuery`（执行查询后）、`onBeforeCommit`（提交事务前）等。
2.  **事件触发**: 在每个包装类（如 `StatementWrapper`）的关键方法中，实际的数据库操作被一个 `try...catch...finally` 块包围。
    *   在 `try` 之前，调用 `jdbcEventListener` 的 `onBefore...` 方法。
    *   在 `finally` 中，调用 `jdbcEventListener` 的 `onAfter...` 方法，并传入执行耗时和可能发生的异常。
3.  **监听器加载**: P6Spy 使用 Java 的 SPI 机制来加载事件监听器。它会查找实现了 `com.p6spy.engine.spy.P6Factory` 接口的类，并通过这些工厂类获取具体的 `JdbcEventListener` 实例。
4.  **核心监听器**:
    *   `com.p6spy.engine.logging.LoggingEventListener`: 这是实现 SQL 日志记录的核心监听器。它监听 `onAfterAnyExecute` 等事件，当事件发生时，它会调用 `P6LogQuery.logElapsed()` 方法来记录日志。
    *   `com.p6spy.engine.outage.OutageJdbcEventListener`: 用于检测和报告长时间运行的 SQL 语句。

### 3.4. 日志记录 (P6LogQuery)

1.  **日志生成**: `com.p6spy.engine.common.P6LogQuery` 是负责生成和输出日志的核心工具类。
2.  **格式化**: 它根据 `spy.properties` 中配置的 `logMessageFormat` 来格式化日志消息。
3.  **输出**: 最终，格式化后的日志被发送到由 `spy.properties` 中 `appender` 配置所指定的输出目标（如文件、控制台等）。

### 流程图总结

```
应用程序
   |
   v
调用 DriverManager.getConnection("jdbc:p6spy:...")
   |
   v
P6SpyDriver.connect()
   | 1. 剥离 "p6spy:" 前缀
   | 2. 找到真实的 Driver (e.g., MySQL Driver)
   | 3. 调用真实 Driver.connect() 获取真实 Connection
   | 4. 创建 JdbcEventListener (e.g., LoggingEventListener)
   | 5. 返回 ConnectionWrapper(真实 Connection, eventListener)
   |
   v
应用程序调用 connection.prepareStatement(sql)
   |
   v
ConnectionWrapper.prepareStatement(sql)
   | 1. 调用真实 connection.prepareStatement(sql) 获取真实 PreparedStatement
   | 2. 返回 PreparedStatementWrapper(真实 PreparedStatement, eventListener)
   |
   v
应用程序调用 statement.executeQuery()
   |
   v
PreparedStatementWrapper.executeQuery()
   |
   |--> eventListener.onBeforeExecuteQuery()  // <-- 事件通知
   |
   |    (记录开始时间)
   |    真实 statement.executeQuery()
   |    (记录结束时间)
   |
   |--> eventListener.onAfterExecuteQuery(...) // <-- 事件通知
         |
         v
      LoggingEventListener
         |
         v
      P6LogQuery.logElapsed(...)  // 记录 SQL 和耗时
         |
         v
      根据 spy.properties 配置输出到文件或控制台
```

## 4. P6Spy 如何获取并解析完整的 SQL？

P6Spy 的巧妙之处在于它**并不需要一个真正的 SQL 解析器**。它通过拦截 JDBC 调用来“重组”出最终执行的 SQL。

### 4.1. 技术核心：拦截 `set` 方法

对于 `PreparedStatement`，其 SQL 模板（包含 `?` 占位符）在创建时就已经确定。P6Spy 的核心任务是捕获所有为这些占位符设置参数值的调用。

1.  **拦截 `setXXX` 调用**: 当应用程序调用 `preparedStatement.setString(1, "John")`、`setInt(2, 42)` 等方法时，实际上调用的是 `PreparedStatementWrapper` 上的对应方法。
2.  **存储参数**: 在 `PreparedStatementWrapper` 内部，每个 `setXXX` 方法在将调用委托给真实的 `PreparedStatement` 之后，会立刻触发一个 `onAfterPreparedStatementSet` 事件。
3.  **记录参数值**: `PreparedStatementInformation` 对象会监听这个事件，并将参数的**位置**（例如 `1`）和**值**（例如 `"John"`）存储在一个内部的 `Map<Integer, Value>` 中。

**代码示例 (`PreparedStatementWrapper.java`):**
```java
@Override
public void setString(int parameterIndex, String x) throws SQLException {
    SQLException e = null;
    try {
        delegate.setString(parameterIndex, x); // 1. 调用真实方法
    } catch (SQLException sqle) {
        e = sqle;
        throw e;
    } finally {
        // 2. 触发事件，将参数值 (x) 传递出去
        eventListener.onAfterPreparedStatementSet(statementInformation, parameterIndex, x, e);
    }
}
```

### 4.2. SQL 重组：简单的字符串替换

当需要生成完整的、可执行的 SQL 语句时（通常是在 `execute` 方法被调用后，准备记录日志时），P6Spy 会执行以下操作：

1.  **获取原始 SQL**: 从 `PreparedStatementInformation` 中获取原始的、包含 `?` 的 SQL 模板。
2.  **获取参数 Map**: 从同一个对象中获取之前存储的所有参数值。
3.  **遍历和替换**: `getSqlWithValues()` 方法会遍历 SQL 模板字符串。当遇到一个 `?` 时，它就从参数 `Map` 中按顺序取出一个值，并将其转换为字符串，替换掉这个 `?`。
4.  **类型处理**: `Value` 类负责将不同类型的 Java 对象（如 `String`, `Integer`, `Date`）格式化为符合 SQL 语法的字符串（例如，字符串会用单引号包裹，日期会格式化为特定的字符串格式）。

**代码示例 (`PreparedStatementInformation.java`):**
```java
@Override
public String getSqlWithValues() {
    final StringBuilder sb = new StringBuilder();
    final String statementQuery = getStatementQuery();

    int currentParameter = 0;
    for (int pos = 0; pos < statementQuery.length(); pos++) {
        char character = statementQuery.charAt(pos);
        if (character == '?') {
            // 遇到 '?'，从 Map 中获取参数并替换
            Value value = parameterValues.get(currentParameter);
            sb.append(value != null ? value.toString() : new Value().toString());
            currentParameter++;
        } else {
            sb.append(character);
        }
    }

    return sb.toString();
}
```

### 4.3. 总结

| 技术/模式 | 在 P6Spy 中的应用 |
| :--- | :--- |
| **装饰器模式** | `ConnectionWrapper`, `PreparedStatementWrapper` 等类包装了真实的 JDBC 对象，附加了事件通知的功能。 |
| **事件监听** | `JdbcEventListener` 定义了一系列事件钩子，允许在 JDBC 方法执行前后插入自定义逻辑。 |
| **方法拦截** | 在包装类中重写 `setXXX` 和 `executeXXX` 等关键方法，从而捕获 SQL 参数和执行时机。 |
| **字符串替换** | 通过简单的遍历和替换 `?` 占位符的方式，将捕获到的参数值“填空”到原始 SQL 模板中，从而重组出完整的 SQL 语句。 |

这种方法避免了引入重量级的 SQL 解析库，使得 P6Spy 保持了轻量级和高性能，同时又能准确地捕获到最终执行的 SQL 语句。

---

## 5. 源代码学习路径 (从入口开始)

本节提供一个自顶向下的源代码学习路径，帮助您理解 P6Spy 的完整工作流程。

### 第 1 站：驱动入口 - `P6SpyDriver.java`

这是 P6Spy 世界的起点。当您的应用程序通过 `DriverManager` 请求一个以 `jdbc:p6spy:` 开头的连接时，就是这个类在响应。

*   **重点关注**:
    *   **静态代码块**: `DriverManager.registerDriver(P6SpyDriver.instance);` - 这是 P6Spy 作为 JDBC 驱动被 Java 发现的关键。
    *   `acceptsURL(String url)`: 判断是否应该由 P6Spy 处理该连接请求。
    *   `connect(String url, Properties properties)`: 这是核心中的核心。请仔细阅读此方法：
        1.  它如何调用 `findPassthru(url)` 来找到真正的数据库驱动。
        2.  它如何加载 `JdbcEventListenerFactory`。
        3.  它如何调用真实驱动的 `connect()` 方法来获取一个真实的 `Connection`。
        4.  最关键的一步：它如何使用 `ConnectionWrapper.wrap(...)` 来包装这个真实的 `Connection`，并返回一个代理连接。

### 第 2 站：核心代理 - `ConnectionWrapper.java`

这是第一个代理类，它掌管着所有从 `Connection` 对象派生出的操作。

*   **重点关注**:
    *   **构造函数**: 理解它如何持有一个真实的 `Connection` (名为 `delegate`) 和一个 `JdbcEventListener`。
    *   `createStatement()` 和 `prepareStatement(String sql)`: 这两个方法是代理模式的完美体现。它们没有直接返回真实 `Statement`，而是返回了被再次包装的 `StatementWrapper` 或 `PreparedStatementWrapper`。这是实现拦截链的关键。
    *   `commit()` 和 `rollback()`: 观察它们是如何在调用真实 `delegate` 的方法前后，分别触发 `onBeforeCommit`/`onAfterCommit` 等事件的。这展示了事务是如何被监控的。

### 第 3 站：语句拦截 - `PreparedStatementWrapper.java`

这是最繁忙的代理类，几乎所有的 SQL 参数设置和执行都在这里被拦截。

*   **重点关注**:
    *   **所有的 `setXXX(int parameterIndex, ...)` 方法**: 仔细观察其中一个，例如 `setString()`。您会发现一个固定的模式：
        1.  调用 `delegate.setString(...)`。
        2.  在 `finally` 块中，调用 `eventListener.onAfterPreparedStatementSet(...)`，将参数的索引和值传递出去。
    *   `executeQuery()` 和 `executeUpdate()`: 与 `ConnectionWrapper` 中的 `commit` 类似，这些方法在调用真实 `delegate` 的方法前后，触发了 `onBeforeExecuteQuery` 和 `onAfterExecuteQuery` 事件，并精确计算了 SQL 的执行耗时。

### 第 4 站：信息载体 - `PreparedStatementInformation.java`

这个类本身不是代理，而是一个数据容器。它与 `PreparedStatementWrapper` 一一对应，负责收集关于单个 `PreparedStatement` 的所有信息。

*   **重点关注**:
    *   `setParameterValue(int position, Object value)`: 这个方法被 `onAfterPreparedStatementSet` 事件触发，负责将 `PreparedStatementWrapper` 捕获到的参数值存入一个内部的 `Map` 中。
    *   `getSqlWithValues()`: 这是将所有信息汇总的地方。它拿出原始的 SQL 模板和存储的参数 `Map`，通过简单的字符串替换，拼装出最终可读的、完整的 SQL 语句。

### 第 5 站：事件中心 - `JdbcEventListener.java` 和 `LoggingEventListener.java`

`JdbcEventListener` 定义了所有事件的“契约”，而 `LoggingEventListener` 则是这些事件的具体消费者。

*   **重点关注**:
    *   **`JdbcEventListener.java`**: 快速浏览一遍，了解 P6Spy 提供了哪些可以被监听的事件钩子（`onBefore...`, `onAfter...`）。
    *   **`LoggingEventListener.java`**:
        1.  `onAfterAnyExecute(...)`: 这是最核心的日志触发点。无论 `executeQuery`, `executeUpdate` 还是 `execute` 被调用，最终都会触发这个事件。
        2.  `logElapsed(...)`: 此方法是日志记录的最终委托者，它调用了 `P6LogQuery.logElapsed()`。

### 第 6 站：日志输出 - `P6LogQuery.java`

这是日志记录流程的最后一站。

*   **重点关注**:
    *   `logElapsed(...)`: 这个静态方法接收来自 `LoggingEventListener` 的所有信息（连接ID、耗时、SQL信息等）。
    *   它会进一步调用 `P6SpyOptions.getActiveInstance().getLogMessageFormat().formatMessage(...)` 来获取格式化器。
    *   最终通过 `log()` 方法将格式化后的字符串输出到 `spy.properties` 中配置的 `appender`。

### 学习建议

1.  **使用 Debugger**: 设置一个断点在 `P6SpyDriver.connect()`，然后以 Debug 模式运行一个使用 P6Spy 的简单 Java 程序。单步执行，亲眼观察 P6Spy 是如何一步步创建代理、触发事件和记录日志的。
2.  **自顶向下**: 严格按照上述 1-6 站的顺序阅读代码，这与 P6Spy 的实际执行流程完全一致。
3.  **先主后次**: 先理解 `PreparedStatement` 的流程，因为它是最常见的。之后再去看 `Statement` 和 `CallableStatement` 的实现，您会发现逻辑大同小异。

---

## 6. 深度解析：PreparedStatementWrapper 的拦截原理

`PreparedStatementWrapper` 是 P6Spy 拦截机制的核心体现。理解它的工作原理，就能掌握 P6Spy 的精髓。其原理可以概括为 **“装饰与通知”**。

### 6.1. 角色定义

*   **被装饰者 (Delegate)**: 这是一个由真实 JDBC 驱动创建的、原始的 `java.sql.PreparedStatement` 对象。它负责与数据库进行实际的通信。
*   **装饰者 (Wrapper)**: `PreparedStatementWrapper` 本身。它实现了 `java.sql.PreparedStatement` 接口，因此对于应用程序来说，它看起来就和一个普通的 `PreparedStatement` 一模一样。它的内部持有一个 `delegate` 对象的引用。
*   **通知中心 (EventListener)**: 一个 `JdbcEventListener` 的实例。它负责接收 `PreparedStatementWrapper` 发送的各种事件通知。

### 6.2. 拦截两大类关键方法

`PreparedStatementWrapper` 的工作重点是拦截两类关键的 JDBC 方法：**参数设置方法**和**语句执行方法**。

#### 6.2.1. 拦截参数设置 (`setXXX`)

这是 P6Spy 能够捕获 SQL 参数的关键。

**场景**: 应用程序调用 `preparedStatement.setString(1, "some_value");`

**执行流程**:

1.  **调用 Wrapper**: 应用程序实际上调用的是 `PreparedStatementWrapper` 的 `setString` 方法。
2.  **委托给 Delegate**: Wrapper 内部首先会调用 `delegate.setString(1, "some_value");`。这一步是必须的，因为它需要让真实的 `PreparedStatement` 对象去完成参数的设置。
3.  **发布事件 (核心)**: 在 `finally` 块中，Wrapper 会立即调用 `eventListener.onAfterPreparedStatementSet(...)` 方法。这是一个关键的“通知”步骤，它将刚刚设置的参数信息（索引 `1` 和值 `"some_value"`）广播出去。
4.  **信息存储**: `PreparedStatementInformation` 对象作为事件的接收者之一，会将这些参数信息存入一个内部的 `Map` 中，以备后续使用。

**伪代码**:
```java
// 在 PreparedStatementWrapper.java 中
public void setString(int parameterIndex, String value) {
    try {
        // 1. 委托给真实对象
        this.delegate.setString(parameterIndex, value);
    } finally {
        // 2. 发布参数已设置的事件
        this.eventListener.onAfterPreparedStatementSet(this.statementInformation, parameterIndex, value);
    }
}
```

通过重写所有 `setXXX` 方法并遵循这个“委托 -> 通知”模式，`PreparedStatementWrapper` 确保了没有任何一个参数设置操作会被遗漏。

#### 6.2.2. 拦截语句执行 (`executeXXX`)

这是 P6Spy 能够记录 SQL 执行耗时和最终 SQL 语句的关键。

**场景**: 应用程序调用 `preparedStatement.executeUpdate();`

**执行流程**:

1.  **调用 Wrapper**: 应用程序调用的是 `PreparedStatementWrapper` 的 `executeUpdate` 方法。
2.  **执行前通知**: 在调用真实方法之前，Wrapper 首先调用 `eventListener.onBeforeExecuteUpdate(...)`。这可以用于标记 SQL 即将执行。
3.  **计时与执行**:
    *   记录一个开始时间戳 `long start = System.nanoTime();`。
    *   在 `try` 块中，调用 `delegate.executeUpdate();` 来实际执行 SQL。
4.  **执行后通知 (核心)**: 在 `finally` 块中，Wrapper 会调用 `eventListener.onAfterExecuteUpdate(...)`。它会传入以下关键信息：
    *   `statementInformation`: 包含了原始 SQL 和所有已捕获参数的那个信息对象。
    *   `System.nanoTime() - start`: 精确的 SQL 执行耗时。
    *   执行过程中可能抛出的 `SQLException`。
5.  **日志生成**: `LoggingEventListener` 在接收到这个 `onAfter` 事件后，会从 `statementInformation` 对象中调用 `getSqlWithValues()` 来获取完整的、带参数的 SQL 语句，然后连同执行耗时一起，交给 `P6LogQuery` 进行最终的格式化和输出。

**伪代码**:
```java
// 在 PreparedStatementWrapper.java 中
public int executeUpdate() throws SQLException {
    long start = 0;
    try {
        // 1. 执行前通知
        this.eventListener.onBeforeExecuteUpdate(this.statementInformation);
        // 2. 开始计时
        start = System.nanoTime();
        // 3. 委托给真实对象执行
        return this.delegate.executeUpdate();
    } finally {
        // 4. 执行后通知，并附上耗时
        long elapsed = System.nanoTime() - start;
        this.eventListener.onAfterExecuteUpdate(this.statementInformation, elapsed);
    }
}
```

### 6.3. 原理总结

`PreparedStatementWrapper` 的拦截原理可以总结为：

*   **身份伪装**: 它实现了 `PreparedStatement` 接口，让应用程序毫无察觉地使用它。
*   **工作委托**: 它将所有真实的数据库操作全部委托给其内部持有的 `delegate`（真实 `PreparedStatement` 对象）。
*   **附加职责**: 在委托真实操作的前后，它会执行额外的“附加职责”——**发布事件**。
    *   对于 `setXXX` 方法，它在操作后发布“参数已设置”的事件。
    *   对于 `executeXXX` 方法，它在操作前后发布“执行前/后”的事件，并附加计时功能。

通过这种非侵入式的“装饰与通知”机制，`PreparedStatementWrapper` 成功地将一个封闭的 JDBC 操作流程，变成了一个可观察、可度量的开放过程，从而实现了对 SQL 语句的全面监控。

---

## 7. 核心配置文件：spy.properties 详解

`spy.properties` 文件是 P6Spy 的大脑，它控制着 P6Spy 的所有行为。当 P6Spy 启动时，它会自动在 classpath 的根目录下寻找这个文件并加载其中的配置。

### 7.1. 模块化配置 (`modulelist`)

P6Spy 采用模块化设计，每个模块提供特定的功能。`modulelist` 属性用于激活一个或多个模块，多个模块之间用逗号分隔。

*   **`com.p6spy.engine.spy.P6SpyFactory`**: **(不推荐直接使用)** 这是最基础的模块，只提供事件通知，不包含任何日志记录逻辑。
*   **`com.p6spy.engine.logging.P6LogFactory`**: **(最常用)** 激活 SQL 日志记录功能。它会注册 `LoggingEventListener` 来监听 JDBC 事件并生成日志。
*   **`com.p6spy.engine.outage.P6OutageFactory`**: 激活 SQL 执行超时检测功能。当一个 SQL 的执行时间超过设定的阈值时，会记录一条 "outage" 日志。

### 7.2. 驱动配置 (`realdriver`, `deregisterdrivers`)

*   `realdriver`: 显式指定真实的 JDBC 驱动类名。**通常情况下无需配置此项**，因为 P6Spy 会自动扫描 classpath 并找到合适的驱动。只有在 classpath 中存在多个驱动或自动检测失败时才需要手动指定。
*   `deregisterdrivers`: (布尔值, 默认为 `false`) 是否在 P6Spy 注册真实驱动后，将它们从 `DriverManager` 中移除。在某些复杂的类加载环境中，这可以避免驱动被重复注册。

### 7.3. 日志输出与格式化

这些配置项只有在激活了 `P6LogFactory` 模块后才生效。

*   `logfile`: 指定日志文件的输出路径和名称。默认为项目根目录下的 `spy.log`。
*   `append`: (布尔值, 默认为 `true`) 如果日志文件已存在，是追加内容 (`true`) 还是覆盖旧内容 (`false`)。
*   `logMessageFormat`: 指定日志的格式化器类名。P6Spy 内置了两种格式：
    *   `com.p6spy.engine.spy.appender.SingleLineFormat`: 将 SQL 和所有信息输出在一行内。
    *   `com.p6spy.engine.spy.appender.MultiLineFormat`: 将 SQL 格式化为多行，更易于阅读。
*   `appender`: 指定日志的输出目的地类名。默认为 `com.p6spy.engine.spy.appender.FileLogger`，即输出到文件。可以替换为：
    *   `com.p6spy.engine.spy.appender.StdoutLogger`: 输出到标准控制台 (`System.out`)。
    *   `com.p6spy.engine.spy.appender.Slf4JLogger`: 委托给 SLF4J 进行输出，可以与您项目中的 Logback, Log4j2 等日志框架集成。
    *   `com.p6spy.engine.spy.appender.NullLogger`: 不输出任何日志，相当于临时禁用日志功能。
*   `dateformat`: 自定义日志中时间的格式，遵循 `java.text.SimpleDateFormat` 的规范。

### 7.4. 过滤与筛选

*   `filter`: (布尔值, 默认为 `false`) 是否启用 SQL 过滤功能。
*   `include`: 当 `filter=true` 时生效，指定一个以逗号分隔的表名列表。只有操作这些表的 SQL 才会被记录。
*   `exclude`: 当 `filter=true` 时生效，指定一个以逗号分隔的表名列表。操作这些表的 SQL 将不会被记录。
*   `excludecategories`: 指定不记录哪些类别的日志，例如 `commit`, `rollback`, `result` 等。

### 7.5. 其他高级配置

*   `outagedetection`: (布尔值, 默认为 `false`) 是否启用 SQL 超时检测。需要配合 `P6OutageFactory` 模块使用。
*   `outagedetectioninterval`: (秒) 当 `outagedetection=true` 时，定义 SQL 执行时间的阈值。超过这个时间的 SQL 将被认为是 "outage"。
*   `jndicontextfactory`, `jndicontextproviderurl`, `jndicontextcustom`: 用于在 JNDI 环境下配置数据源。

### 7.6. 完整配置示例

```properties
# 模块列表：同时激活日志记录和超时检测
modulelist=com.p6spy.engine.logging.P6LogFactory,com.p6spy.engine.outage.P6OutageFactory

# Appender: 将日志委托给 SLF4J，而不是直接写文件
# 这样可以和项目本身的日志框架（如 Logback）统一管理
appender=com.p6spy.engine.spy.appender.Slf4JLogger

# 日志格式：使用易于阅读的多行格式
logMessageFormat=com.p6spy.engine.spy.appender.MultiLineFormat

# 日志中时间的格式
dateformat=yyyy-MM-dd HH:mm:ss

# 过滤配置：不记录所有对 'flyway_schema_history' 表的操作
filter=true
exclude=flyway_schema_history

# 类别过滤：不记录 'commit' 和 'result' 相关的日志，只关注 SQL 执行本身
excludecategories=commit,result

# 超时检测：执行时间超过 2 秒的 SQL 将被单独标记为 "outage"
outagedetection=true
outagedetectioninterval=2
```

---

## 8. 高级应用：实现 SQL 审计

P6Spy 的强大之处不仅在于开箱即用的 SQL 日志，更在于其优秀的可扩展性。通过实现自定义的 `JdbcEventListener`，我们可以构建各种高级功能，例如 SQL 审计、性能告警、甚至是防止高危 SQL 执行等。

本节将通过一个完整的例子，教您如何实现一个简单的 SQL 审计功能。

**审计目标**：记录所有对数据库执行的 `UPDATE` 和 `DELETE` 操作，并将操作人、操作时间、执行的 SQL 语句输出到独立的审计日志中。

### 步骤 1：创建自定义的事件监听器

首先，我们需要创建一个类，继承自 `com.p6spy.engine.event.SimpleJdbcEventListener`。继承 `SimpleJdbcEventListener` 而不是 `JdbcEventListener` 的好处是，我们只需要重写我们关心的事件方法即可。

```java
// src/main/java/com/mycompany/p6spy/audit/AuditEventListener.java

package com.mycompany.p6spy.audit;

import com.p6spy.engine.common.StatementInformation;
import com.p6spy.engine.event.SimpleJdbcEventListener;

import java.text.SimpleDateFormat;
import java.util.Date;

public class AuditEventListener extends SimpleJdbcEventListener {

    @Override
    public void onAfterExecuteUpdate(StatementInformation statementInformation, long timeElapsedNanos, String sql, int rowCount, SQLException e) {
        // 我们只关心成功的、并且是 UPDATE 或 DELETE 的操作
        if (e == null) {
            String preparedSql = statementInformation.getSql();
            String finalSql = statementInformation.getSqlWithValues();

            // 简单的判断逻辑，实际场景可能需要更复杂的 SQL 解析
            if (isUpdateOrDelete(preparedSql)) {
                // 在真实场景中，操作人信息通常从线程上下文（ThreadLocal）中获取
                // 这里我们用一个硬编码的 "CurrentUser" 作为示例
                String currentUser = "CurrentUser"; 

                String auditMessage = new StringBuilder()
                    .append("[AUDIT] User '")
                    .append(currentUser)
                    .append("' performed a data modification. | Timestamp: ")
                    .append(new SimpleDateFormat("yyyy-MM-dd HH:mm:ss").format(new Date()))
                    .append(" | Rows Affected: ")
                    .append(rowCount)
                    .append(" | SQL: ")
                    .append(finalSql.replaceAll("\\s+", " ")) // 将 SQL 压缩到一行
                    .toString();

                // 将审计信息输出到控制台
                // 在真实项目中，您可能会使用独立的 Logger 将其写入到专门的审计日志文件
                System.out.println(auditMessage);
            }
        }
    }

    private boolean isUpdateOrDelete(String sql) {
        if (sql == null || sql.trim().isEmpty()) {
            return false;
        }
        String lowerCaseSql = sql.trim().toLowerCase();
        return lowerCaseSql.startsWith("update") || lowerCaseSql.startsWith("delete");
    }
}
```

### 步骤 2：创建自定义的工厂类

P6Spy 通过工厂模式来加载我们自定义的监听器。我们需要创建一个实现了 `com.p6spy.engine.spy.P6Factory` 接口的类。

```java
// src/main/java/com/mycompany/p6spy/audit/AuditFactory.java

package com.mycompany.p6spy.audit;

import com.p6spy.engine.event.JdbcEventListener;
import com.p6spy.engine.spy.P6Factory;
import com.p6spy.engine.spy.P6SpyOptions;

public class AuditFactory implements P6Factory {

    private static final JdbcEventListener AUDIT_EVENT_LISTENER = new AuditEventListener();

    @Override
    public P6SpyOptions getOptions(P6SpyOptions options) {
        // 我们可以在这里通过代码动态修改配置，但本例中不需要
        return options;
    }

    @Override
    public JdbcEventListener getJdbcEventListener() {
        // 返回我们自定义的监听器实例
        return AUDIT_EVENT_LISTENER;
    }
}
```

### 步骤 3：通过 SPI 机制注册工厂

这是让 P6Spy 发现我们自定义模块的关键一步。我们需要在 `src/main/resources/META-INF/services/` 目录下创建一个名为 `com.p6spy.engine.spy.P6Factory` 的文件。

文件的内容只有一行，就是我们刚刚创建的工厂类的全限定名：

```
com.mycompany.p6spy.audit.AuditFactory
```

### 步骤 4：修改 `spy.properties`

最后，我们需要更新 `spy.properties` 文件，将我们的自定义工厂类添加到 `modulelist` 中。P6Spy 会加载 `modulelist` 中所有的工厂，并将它们的监听器组合在一起。

```properties
# modulelist 现在包含两个模块：
# 1. P6LogFactory: 用于常规的 SQL 日志记录。
# 2. AuditFactory: 用于我们自定义的 SQL 审计。
# P6Spy 会将这两个模块的监听器（LoggingEventListener 和 AuditEventListener）合并成一个组合监听器。
modulelist=com.p6spy.engine.logging.P6LogFactory,com.mycompany.p6spy.audit.AuditFactory

# 其他配置保持不变...
appender=com.p6spy.engine.spy.appender.StdoutLogger
logMessageFormat=com.p6spy.engine.spy.appender.MultiLineFormat
```

### 运行与验证

完成以上步骤后，重新启动您的应用程序。当执行一个 `UPDATE` 或 `DELETE` 语句时，您会在控制台看到两种输出：

1.  由 `P6LogFactory` 生成的、标准格式的 SQL 执行日志。
2.  由我们自定义的 `AuditEventListener` 生成的、以 `[AUDIT]` 开头的审计日志。

**示例输出**:
```
// ... 标准 P6Spy 日志 ...
[AUDIT] User 'CurrentUser' performed a data modification. | Timestamp: 2023-10-28 10:30:00 | Rows Affected: 1 | SQL: UPDATE users SET city = 'New York' WHERE id = 1
// ... 标准 P6Spy 日志 ...
```

通过这个例子，您可以看到，利用 P6Spy 提供的事件机制，可以非常方便地在不修改业务代码的前提下，实现强大的、定制化的数据库操作监控功能。
