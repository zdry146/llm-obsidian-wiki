---
title: "Kotlin DSL 与作用域函数"
category: synthesis
tags: [kotlin, dsl, scope-functions, apply, with, run, also, let, receiver]
sources:
  - "Kotlin Docs - Scope Functions"
  - "Kotlin Docs - Type-Safe Builders"
summary: "Kotlin 5 大作用域函数 apply/with/run/also/let + Receiver 类型 + DSL 构建器"
provenance:
  extracted: 0.90
  inferred: 0.08
  ambiguous: 0.02
base_confidence: 0.88
lifecycle: draft
lifecycle_changed: 2026-09-13
created: 2026-09-13
updated: 2026-09-13
---

# §04 Kotlin DSL 与作用域函数

## 1. 5 大作用域函数

| 函数 | Receiver | 返回值 | 用途 |
|------|----------|--------|------|
| **`apply`** | `T` (this) | `T` | 配置对象，返回对象自身 |
| **`with`** | `T` (this) | `R` | 对对象做一组操作，返回结果 |
| **`run`** | `T` (this) | `R` | 配置 + 计算，lambda 返回结果 |
| **`also`** | `T` (it) | `T` | 副作用（log/debug），返回对象自身 |
| **`let`** | `T` (it) | `R` | 非空操作 + null safe，返回结果 |

## 2. apply（最常用）

```kotlin
// ✅ 配置对象，返回对象自身
val client = OkHttpClient.Builder()
    .connectTimeout(10, TimeUnit.SECONDS)
    .readTimeout(30, TimeUnit.SECONDS)
    .build()
    .apply {
        // 这里直接访问 client 的字段
    }

// 经典用法：Builder 链
val client = OkHttpClient.Builder()
    .connectTimeout(10, TimeUnit.SECONDS)
    .apply {
        // 在 builder 上下文里调用其方法
        readTimeout(30, TimeUnit.SECONDS)
        writeTimeout(30, TimeUnit.SECONDS)
        addInterceptor(AuthInterceptor(tokenProvider))
    }
    .build()
```

**核心洞察**：**apply = Java 风格的 builder 链的 Kotlin 替代**——更简洁。

## 3. with（无返回值操作）

```kotlin
// 对对象做一组操作，返回最后一个表达式的值
val result = with(file) {
    val text = readText()
    val lines = text.lines()
    lines.filter { it.isNotBlank() }
}

// ❌ Java 风格
File file = ...;
List<String> result;
try {
    String text = file.readText();
    List<String> lines = text.lines();
    result = lines.filter(line -> !line.isBlank());
} catch (IOException e) { ... }

// ✅ Kotlin with 风格
val file = File("data.txt")
val result = with(file) {
    val text = readText()
    text.lines().filter { it.isNotBlank() }
}
```

## 4. run（apply + lambda return）

```kotlin
// 配置 + 计算返回值
val name = User(name = "Mike", age = 30).run {
    if (age >= 18) "Adult: $name" else "Minor: $name"
}

// 安全调用
val length = user?.run {
    name.length
} ?: 0
```

## 5. also（副作用，调试/日志）

```kotlin
// 也返回对象，但 lambda 接收 it（不是 this）
val user = fetchUser()
    .also { log.debug("Fetched: $it") }   // log
    .also { cache.put(it.id, it) }        // cache

// 等价 Java
User user = fetchUser();
log.debug("Fetched: " + user);
cache.put(user.getId(), user);
```

## 6. let（null 安全）

```kotlin
// 标准模式：nullable 处理
val name: String? = getName()
val length = name?.let {
    log.info("Name: $it")
    it.length
} ?: 0

// 链式 nullable 操作
val result = user?.let { u ->
    address?.let { a ->
        a.format(u)
    }
}

// early return
fun process(user: User?) {
    user ?: return
    // user 非空，可直接使用
    println(user.name)
}
```

## 7. 作用域函数对比

```kotlin
data class Config(var host: String = "", var port: Int = 0)

// apply - 配置对象
val c1 = Config().apply {
    host = "localhost"
    port = 50051
}  // 返回 Config

// with - 返回 lambda 结果
val c2: String = with(Config()) {
    host = "localhost"
    port = 50051
    "Config(host=$host, port=$port)"  // 返回这个字符串
}

// run - 配置 + 返回 lambda 结果
val c3 = Config().run {
    host = "localhost"
    port = 50051
    "Configured"
}  // 返回 "Configured"

// also - 副作用
val c4 = Config().also {
    it.host = "localhost"
    it.port = 50051
    log.info("Configured")
}  // 返回 Config（但执行了 log）

// let - null safe
val c5: Config? = getConfig()?.let {
    it.host = "localhost"
    it.port = 50051
    it
}  // 返回 Config?（可能为 null）
```

## 8. 何时用哪个？

```
问 3 个问题：
  1. 需要返回对象还是 lambda 结果？
     - 对象本身 → apply / also
     - lambda 结果 → run / with / let
  2. 配置（this）还是副作用（it）？
     - 配置 → apply / with / run
     - 副作用 / log → also
  3. 是否处理 nullable？
     - 是 → let / ?.also
     - 否 → apply / with / run
```

| 场景 | 推荐 |
|------|------|
| **Builder 配置** | `apply` |
| **多步操作返回结果** | `with` |
| **配置后计算** | `run` |
| **log/debug** | `also` |
| **null 安全** | `let` |

## 9. DSL 构建器（Type-Safe Builders）

### 9.1 经典 HTML DSL

```kotlin
// HTML DSL（标准库示例）
val html = html {
    head {
        title { +"My Page" }
    }
    body {
        div(classes = "container") {
            h1 { +"Hello" }
            p { +"Welcome" }
        }
    }
}

// 实现
class Tag(val name: String) {
    val children = mutableListOf<Tag>()
    val attributes = mutableMapOf<String, String>()
    
    fun render(): String {
        val attrs = attributes.entries.joinToString(" ") { "${it.key}=\"${it.value}\"" }
        val childContent = children.joinToString("") { it.render() }
        return "<$name $attrs>$childContent</$name>"
    }
}

fun html(block: Tag.() -> Unit): Tag = Tag("html").apply(block)
fun Tag.head(block: Tag.() -> Unit) { children.add(Tag("head").apply(block)) }
fun Tag.body(block: Tag.() -> Unit) { children.add(Tag("body").apply(block)) }
// ...
```

### 9.2 Gradle Kotlin DSL

```kotlin
// build.gradle.kts
plugins {
    java
    kotlin("jvm") version "2.0.0"
}

dependencies {
    implementation("io.zdry:netlib-common:1.0.0")
    testImplementation(kotlin("test"))
}

tasks.withType<Test> {
    useJUnitPlatform()
}
```

### 9.3 OkHttp 风格 DSL（Kotlin 包装）

```kotlin
// 用 Kotlin 包装 OkHttpClient.Builder
fun httpClient(block: OkHttpClient.Builder.() -> Unit): OkHttpClient =
    OkHttpClient.Builder().apply(block).build()

// 使用
val client = httpClient {
    connectTimeout(5, TimeUnit.SECONDS)
    readTimeout(30, TimeUnit.SECONDS)
    addInterceptor(AuthInterceptor { tokenStore.getAccessToken() })
}
```

## 10. Receiver 类型（DSL 关键）

```kotlin
// 带 receiver 的函数类型：A.() -> Unit
fun buildString(capacity: Int = 16, block: StringBuilder.() -> Unit): String {
    val sb = StringBuilder(capacity)
    sb.block()        // 在 sb 上下文执行 block
    return sb.toString()
}

// 使用
val html = buildString {
    append("<html>")
    append("<body>")
    append("Hello")
    append("</body>")
    append("</html>")
}

// DSL 核心：lambda 接收 receiver 类型
// block 内部 this 是 StringBuilder
// 可以直接调用 sb 的方法，无需 sb.append()
```

**核心洞察**：**Receiver 类型让 DSL 像 builder 一样调用，但又保持类型安全**。

## 11. infix 函数 + DSL

```kotlin
// 中缀调用
infix fun <T> T.should(matcher: Matcher<T>): T = ...
infix fun <T> T.shouldNot(matcher: Matcher<T>): T = ...

// 测试 DSL
val result = fetchUser() should beValidUser
val config = httpClient {
    connectTimeout(5.seconds)
} shouldBeValidConfig
```

## 12. invoke operator（DSL 调用简化）

```kotlin
class HtmlBuilder {
    operator fun String.invoke(block: () -> String) = "<$this>${block()}</$this>"
}

val html = HtmlBuilder().apply {
    "html" { "body" { "h1" { "Hello" } } }
}
```

## 13. 实战：HTTP Client DSL

```kotlin
// netlib-common 的 HttpClientFactory 包装
class HttpBuilder {
    var connectTimeout: Duration = 10.seconds
    var readTimeout: Duration = 30.seconds
    var pingInterval: Duration = Duration.ZERO
    private val interceptors = mutableListOf<Interceptor>()
    
    fun interceptor(name: String, interceptor: Interceptor) {
        interceptors.add(interceptor)
    }
}

fun httpConfig(block: HttpBuilder.() -> Unit): HttpConfig {
    val builder = HttpBuilder().apply(block)
    return HttpConfig(
        connectTimeout = builder.connectTimeout,
        readTimeout = builder.readTimeout,
        pingInterval = builder.pingInterval,
        interceptors = builder.interceptors.toList()
    )
}

// 使用
val config = httpConfig {
    connectTimeout = 5.seconds
    readTimeout = 15.seconds
    interceptor("auth", AuthInterceptor { tokenStore.getAccessToken() })
    interceptor("metrics", MetricsInterceptor(meterRegistry))
}
```

## 14. 最佳实践

1. **apply**——对象配置（最常用）
2. **let**——nullable 处理
3. **also**——log/debug
4. **run**——计算 + 返回
5. **with**——多步操作（少用）

## 15. 反模式

❌ **混用 apply 和 also**——语义不清
❌ **长链作用域函数**——可读性差
❌ **let 里 return**——非局部返回（用 label）
❌ **过度用 receiver lambda**——简单场景用普通 lambda

## 相关笔记

- **核心语法**: [[draft-01-syntax-core]]
- **函数与扩展**: [[draft-02-functions]]
- **协程**: [[draft-03-coroutines]]
- **集合**: [[draft-05-collections]]
- **最佳实践**: [[draft-09-best-practices]]
- **netlib-common**: `~/.openclaw/workspace-developer/netlib-common/`
- **综合入口**: [[summary]]