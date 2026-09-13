---
title: "Kotlin 集合与函数式"
category: synthesis
tags: [kotlin, collections, list, map, set, sequence, functional, filter, map]
sources:
  - "Kotlin Docs - Collections"
  - "Kotlin Docs - Sequences"
  - "Kotlin Docs - Scope Functions"
summary: "Kotlin 集合：List/Map/Set、Sequence vs Iterable、filter/map/reduce/fold、集合转换"
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

# §05 Kotlin 集合与函数式

## 1. 三种集合构造

```kotlin
// List（不可变）
val list1 = listOf(1, 2, 3, 4)               // List<Int>
List2 = listOf("a", "b", "c")                 // List<String>

// MutableList（可变）
val mutList = mutableListOf(1, 2, 3)
mutList.add(4)
mutList.remove(1)

// Map
val map = mapOf("a" to 1, "b" to 2, "c" to 3)
val mutableMap = mutableMapOf("a" to 1)
mutableMap["b"] = 2

// Set
val set = setOf(1, 2, 3, 2, 1)              // [1, 2, 3]（去重）
val mutableSet = mutableSetOf(1, 2, 3)
```

**核心洞察**：**默认不可变，可变需显式声明**（类似 val/var）。

## 2. vs Java

| 维度 | Kotlin | Java |
|------|--------|------|
| **默认** | 不可变 | 可变 |
| **构造** | `listOf(...)` | `Arrays.asList(...)` |
| **可变** | `mutableListOf(...)` | `ArrayList<>()` |
| **类型** | `List<Int>` | `List<Integer>` |
| **空集合** | `emptyList<Int>()` | `Collections.emptyList()` |

## 3. 不可变 vs 可变

```kotlin
// ✅ 推荐：不可变 + copy
val original = listOf(1, 2, 3)
val added = original + 4                    // 返回新 list，original 不变
val removed = original - 2                  // 返回新 list

// ❌ 不推荐：到处用 mutable
val mut = mutableListOf(1, 2, 3)
mut.add(4)
mut.remove(2)                                // 副作用
```

**核心原则**：**不可变集合 = 函数式 + 线程安全**。

## 4. 函数式操作（核心）

### 4.1 转换

```kotlin
val numbers = listOf(1, 2, 3, 4, 5)

// map：每个元素转换
val doubled = numbers.map { it * 2 }              // [2, 4, 6, 8, 10]

// filter：过滤
val evens = numbers.filter { it % 2 == 0 }        // [2, 4]

// mapNotNull：转换 + 过滤 null
val strings = listOf("1", "2", "x", "3", null)
val parsed = strings.mapNotNull { it?.toIntOrNull() }  // [1, 2, 3]

// flatMap：展平
val nested = listOf(listOf(1, 2), listOf(3, 4))
val flat = nested.flatMap { it }                  // [1, 2, 3, 4]

// flatten：直接展平
val flat = nested.flatten()
```

### 4.2 聚合

```kotlin
val numbers = listOf(1, 2, 3, 4, 5)

// sum
val total = numbers.sum()                          // 15

// average
val avg = numbers.average()                        // 3.0

// count
val count = numbers.count { it > 3 }               // 2

// max / min
val max = numbers.max()                            // 5
val min = numbers.min()                            // 1

// maxBy / minBy（按 lambda）
data class User(val name: String, val age: Int)
val oldest = users.maxBy { it.age }                // User(name=..., age=99)

// reduce / fold
val sum = numbers.reduce { acc, x -> acc + x }     // 15
val sum2 = numbers.fold(0) { acc, x -> acc + x }   // 15（初始值 0）

// joinToString
val joined = numbers.joinToString(", ")            // "1, 2, 3, 4, 5"
```

### 4.3 查找

```kotlin
val users = listOf(User("Alice", 30), User("Bob", 25))

// find：第一个匹配
val alice = users.find { it.name == "Alice" }    // User(name=Alice, age=30)
val notFound = users.find { it.name == "x" }      // null

// firstOrNull / first
val firstAdult = users.firstOrNull { it.age >= 18 }

// any / all / none
val anyAdult = users.any { it.age >= 18 }          // true
val allAdult = users.all { it.age >= 18 }          // true
val noneMinor = users.none { it.age < 18 }         // true
```

### 4.4 分组与分区

```kotlin
val users = listOf(User("Alice", 30), User("Bob", 17), User("Charlie", 25))

// groupBy：按 lambda 分组
val byAge = users.groupBy { it.age >= 18 }
// {true=[Alice, Charlie], false=[Bob]}

// partition：分两组
val (adults, minors) = users.partition { it.age >= 18 }
// adults = [Alice, Charlie]
// minors = [Bob]

// associateBy：用 lambda 作为 key
val byName = users.associateBy { it.name }
// {Alice=User(Alice,30), Bob=User(Bob,17), ...}
```

### 4.5 排序

```kotlin
val numbers = listOf(3, 1, 4, 1, 5)

// sorted：自然顺序
val sorted = numbers.sorted()                      // [1, 1, 3, 4, 5]

// sortedDescending
val desc = numbers.sortedDescending()              // [5, 4, 3, 1, 1]

// sortedBy：按 lambda
val users = listOf(User("Alice", 30), User("Bob", 25))
val byAge = users.sortedBy { it.age }              // [Bob, Alice]
```

## 5. Sequence vs Iterable（性能关键）

```kotlin
// Iterable：每个操作都立即执行完整列表
val result = listOf(1, 2, 3, 4, 5)
    .filter { it > 2 }                  // 创建新 list [3, 4, 5]
    .map { it * 2 }                    // 再创建 [6, 8, 10]
    .take(1)                            // [6]
// 内存峰值：3 个中间 list

// Sequence：懒求值，按需计算
val result = listOf(1, 2, 3, 4, 5)
    .asSequence()
    .filter { it > 2 }                  // 流水线
    .map { it * 2 }                    // 流水线
    .take(1)                            // 找到第一个就停
    .toList()
// 内存峰值：1 个元素
```

**关键洞察**：
- `Iterable`：每步都是「完整列表」（适合小数据）
- `Sequence`：懒求值（适合大数据链 + 早期终止）

**何时用 Sequence**：
- 链很长（>3 个操作）
- 数据量大
- 有 early termination（`take`、`first`）
- 元素处理耗时

## 6. 实战：Sequence 优化

```kotlin
// ❌ 慢：3 个中间 list
fun findActiveUsers(users: List<User>): List<String> {
    return users
        .filter { it.isActive }        // 完整 list
        .map { it.name }               // 完整 list
        .filter { it.startsWith("A") } // 完整 list
}

// ✅ 快：1 个流水线
fun findActiveUsers(users: List<User>): List<String> {
    return users
        .asSequence()
        .filter { it.isActive }
        .map { it.name }
        .filter { it.startsWith("A") }
        .toList()
}
```

## 7. 集合操作符（数学集合）

```kotlin
val a = setOf(1, 2, 3, 4)
val b = setOf(3, 4, 5, 6)

// 并集
val union = a union b                      // [1, 2, 3, 4, 5, 6]

// 交集
val intersect = a intersect b              // [3, 4]

// 差集
val diff = a subtract b                     // [1, 2]
```

## 8. Map 操作

```kotlin
val map = mapOf("a" to 1, "b" to 2, "c" to 3)

// 遍历
map.forEach { (k, v) -> println("$k -> $v") }

// 转换
val doubled = map.mapValues { (_, v) -> v * 2 }  // {a=2, b=4, c=6}
val keys = map.mapKeys { it.key.uppercase() }   // {A=1, B=2, C=3}

// 过滤
val filtered = map.filterValues { it > 1 }      // {b=2, c=3}
val filtered2 = map.filterKeys { it != "a" }     // {b=2, c=3}

// getOrDefault
val port = map.getOrDefault("port", 8080)

// getOrPut
val cache = mutableMapOf<String, User>()
val user = cache.getOrPut("key") { fetchUser() }
```

## 9. List 特有操作

```kotlin
val list = listOf(1, 2, 3, 4, 5)

// subList
val sub = list.subList(1, 3)               // [2, 3]

// chunked：分块
val chunks = list.chunked(2)               // [[1, 2], [3, 4], [5]]

// windowed：滑动窗口
val windows = list.windowed(3)             // [[1, 2, 3], [2, 3, 4], [3, 4, 5]]

// zip
val names = listOf("a", "b", "c")
val ages = listOf(1, 2, 3)
val zipped = names.zip(ages)               // [(a, 1), (b, 2), (c, 3)]
val map = names.zip(ages).toMap()          // {a=1, b=2, c=3}

// unzip
val (unzippedNames, unzippedAges) = zipped.unzip()

// indexed：加索引
val indexed = list.mapIndexed { i, v -> "$i:$v" }   // ["0:1", "1:2", ...]
```

## 10. 空集合与安全调用

```kotlin
val empty = emptyList<Int>()                // []
val nullList: List<Int>? = null

// 安全调用
val size = nullList?.size ?: 0             // 0（null 时返回 0）
val first = nullList?.firstOrNull()        // null

// orEmpty
val items: List<String>? = null
val size = items.orEmpty().size            // 0
```

## 11. 集合 vs 可变集合（性能）

```kotlin
// ✅ 不可变 + 派生（每次都是新对象）
val base = listOf(1, 2, 3)
val withFour = base + 4                    // [1, 2, 3, 4]
val withoutTwo = base - 2                  // [1, 3]

// ✅ 可变 + 原地修改（无新对象）
val mut = mutableListOf(1, 2, 3)
mut.add(4)                                 // mut = [1, 2, 3, 4]

// 选择：
// - 频繁派生 → 不可变
// - 频繁修改 → 可变
// - 多线程 → 不可变（线程安全）
```

## 12. 实操：HTTP 响应解析

```kotlin
fun parseUsers(json: String): List<User> {
    return Json.decodeFromString<List<User>>(json)
        .asSequence()
        .filter { it.isActive }
        .map { it.toDto() }
        .distinctBy { it.email }       // 去重
        .sortedBy { it.lastLogin }     // 排序
        .take(100)                     // 限制
        .toList()
}
```

## 13. 关键设计原则

1. **默认不可变**——除非必要
2. **Sequence vs Iterable**——大数据用 Sequence
3. **函数式组合**——`filter/map/reduce/fold`
4. **scope function**——`apply/let/run/with/also`
5. **不可变 + 派生**——无副作用

## 14. 反模式

❌ **到处用 mutableList**——优先不可变
❌ **不区分 Sequence vs Iterable**——大数据慢
❌ **长链 Iterable**——内存峰值高
❌ **lambda 内有副作用**——map/filter 应纯函数

## 相关笔记

- **核心语法**: [[draft-01-syntax-core]]
- **函数与扩展**: [[draft-02-functions]]
- **DSL**: [[draft-04-dsl-builders]]
- **OOP**: [[draft-06-oop]]
- **综合入口**: [[summary]]