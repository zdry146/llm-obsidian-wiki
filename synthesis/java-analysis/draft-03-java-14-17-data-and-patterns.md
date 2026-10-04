---
title: "Java 14-17 数据 + 模式匹配"
category: synthesis
tags: [java, java-14, java-15, java-16, java-17, records, sealed-classes, pattern-matching, switch-expression, text-blocks]
sources:
  - "JEP 395 - Records"
  - "JEP 409 - Sealed Classes"
  - "JEP 394 - Pattern Matching for instanceof"
  - "JEP 361 - Switch Expressions"
  - "JEP 378 - Text Blocks"
summary: "Java 14-17 - records + sealed classes + pattern matching + switch 表达式 + text blocks（数据载体范式）"
provenance:
  extracted: 0.90
  inferred: 0.08
  ambiguous: 0.02
base_confidence: 0.91
lifecycle: draft
lifecycle_changed: 2026-10-04
created: 2026-10-03
updated: 2026-10-04
---

# §03 Java 14-17 数据 + 模式匹配

> Java 14 (2020-03) → Java 17 (2021-09, LTS) 完成了 Java 的"数据 + 模式匹配"现代化。这 4 个版本里最关键的 4 个特性：**records**（数据载体）、**sealed classes**（封闭类型层次）、**pattern matching**（类型驱动控制流）、**switch 表达式**（多态分支）。

## 1. records（JEP 395, Java 16 final）

```java
// Java 8 之前 - 60 行 POJO
public final class User {
    private final String name;
    private final int age;
    public User(String name, int age) {
        this.name = name;
        this.age = age;
    }
    public String getName() { return name; }
    public int getAge() { return age; }
    @Override public boolean equals(Object o) { ... }
    @Override public int hashCode() { ... }
    @Override public String toString() { ... }
}

// Java 16+ - 一行
public record User(String name, int age) {}
// 自动获得：name()、age()、equals()、hashCode()、toString()
```

### records 的限制

- **自动 final** — 不可继承（不能 `extends User`）
- **字段是 final** — 不能 `user.name = "..."`
- **可以有额外方法**：
  ```java
  public record User(String name, int age) {
      // 紧凑构造器（可校验）
      public User {
          if (age < 0) throw new IllegalArgumentException("age < 0");
      }

      // 实例方法
      public boolean isAdult() { return age >= 18; }

      // 静态字段
      public static final User ANONYMOUS = new User("Anonymous", 0);
  }
  ```

### records 的应用场景

```java
// DTO / VO
public record OrderItem(String productId, int quantity, BigDecimal price) {}

// 多返回值（替代 Tuple）
public record UserAndPosts(User user, List<Post> posts) {}

// 值对象
public record Money(BigDecimal amount, Currency currency) {
    public Money add(Money other) {
        if (currency != other.currency) throw new IllegalArgumentException();
        return new Money(amount.add(other.amount), currency);
    }
}
```

**反模式**：
```java
// ❌ record 当可变 bean
record User(String name) {
    public void setName(String n) { ... }  // 破坏 record 不变性
}

// ❌ record 继承
record AdminUser(String name, int level) extends User(name, 0) {}  // ❌ record 不能 extends
```

## 2. sealed classes（JEP 409, Java 17 final）

```java
// 封闭类：明确列出允许的子类
public sealed class Shape
    permits Circle, Rectangle, Square {

    public abstract double area();
}

public final class Circle extends Shape {
    private final double radius;
    public Circle(double radius) { this.radius = radius; }
    @Override public double area() { return Math.PI * radius * radius; }
}

public final class Rectangle extends Shape {
    private final double w, h;
    public Rectangle(double w, double h) { this.w = w; this.h = h; }
    @Override public double area() { return w * h; }
}

public final class Square extends Shape {
    private final double side;
    public Square(double side) { this.side = side; }
    @Override public double area() { return side * side; }
}
```

### sealed 子类的 3 种修饰符

| 修饰符 | 含义 |
|---|---|
| `final` | 不能被继承（最严格）|
| `sealed` | 可以再被自己的 permits 列表限制继承 |
| `non-sealed` | 放开，可以被任意继承 |

```java
public sealed class Animal permits Dog, Cat {
    void speak();
}

public sealed class Dog extends Animal permits Husky, Beagle { }  // 再封闭
public non-sealed class Cat extends Animal { }                    // 完全开放
public final class Husky extends Dog { }                          // 终止
```

### 与 record 的天然组合

```java
public sealed interface Expr
    permits Constant, Add, Multiply, Negate {

    double eval();
}

public record Constant(double value) implements Expr {
    @Override public double eval() { return value; }
}

public record Add(Expr left, Expr right) implements Expr {
    @Override public double eval() { return left.eval() + right.eval(); }
}

public record Multiply(Expr left, Expr right) implements Expr {
    @Override public double eval() { return left.eval() * right.eval(); }
}

public record Negate(Expr expr) implements Expr {
    @Override public double eval() { return -expr.eval(); }
}
```

### 2.1 Sealed classes 实战拆解

#### ① permits 子句的两种写法

```java
// 写法 1：独立子类（Java 17+ 可跨文件）
public sealed class Shape permits Circle, Rectangle, Square { ... }

// 文件 A
public final class Circle extends Shape { ... }

// 文件 B
public final class Rectangle extends Shape { ... }

// 文件 C
public final class Square extends Shape { ... }
```

```java
// 写法 2：嵌套子类（隐式 permits）
public sealed class Shape {
    public static final class Circle extends Shape { ... }
    public static final class Rectangle extends Shape { ... }
    public static final class Square extends Shape { ... }
}
```

**注意**：Java 17+ permits 子类**可以跨文件 / 跨包**（只要同模块），Kotlin sealed class 一直要求同 package。Java 这点更宽松。

#### ② 3 种子类修饰符对比

```java
// final - 终止继承（最严格）
public final class Circle extends Shape { ... }

// sealed - 再封闭一层（多级 ADT）
public sealed class Dog extends Animal permits Husky, Beagle { ... }
public final class Husky extends Dog { ... }
public final class Beagle extends Dog { ... }

// non-sealed - 完全开放
public non-sealed class Cat extends Animal { ... }  // 可以被任何人继承
```

**对比表**：

| 修饰符 | 继承性 | 适用场景 |
|---|---|---|
| `final` | ❌ 完全终止 | 大多数子类（不需要再扩展）|
| `sealed` | ✅ 限定继承 | 多级 ADT（递归结构）|
| `non-sealed` | ✅ 完全开放 | 框架扩展点 |

#### ③ permits 列表的硬约束

```java
// ❌ 编译错误：子类不在 permits 列表
public sealed class Shape permits Circle, Rectangle {}
public class Triangle extends Shape {}  // ❌ Triangle 不在 permits

// ❌ 编译错误：子类与 sealed 不同模块
// module-info 里 modules 不匹配

// ❌ 编译错误：子类没有兼容修饰符
public sealed class Shape permits Circle {}
public class Circle extends Shape {}  // ❌ Circle 必须 final / sealed / non-sealed
```

#### ④ exhaustive switch 的编译器魔法

```java
// Java 21+ switch pattern（standard）
static double area(Shape shape) {
    return switch (shape) {
        case Circle c    -> Math.PI * c.radius() * c.radius();
        case Rectangle r -> r.width() * r.height();
        case Square s    -> s.side() * s.side();
        // 编译器验证已穷举 — 不需要 default
        // 加一个 Circle 子类（如 Triangle）就编译失败
    };
}
```

**关键**：sealed permits + switch pattern = **编译期穷举保证**。漏 case / 多 case / 拼写错都会被编译器抓住。

#### ⑤ sealed + record 联合（AST 完整例子）

```java
public sealed interface Expr
    permits Constant, Add, Negate {

    double eval();
}

public record Constant(double value) implements Expr {
    @Override public double eval() { return value; }
}

public record Add(Expr left, Expr right) implements Expr {
    @Override public double eval() { return left.eval() + right.eval(); }
}

public record Negate(Expr expr) implements Expr {
    @Override public double eval() { return -expr.eval(); }
}

// 客户端：穷举 switch + record pattern 解构
static String render(Expr e) {
    return switch (e) {
        case Constant(double v)       -> String.valueOf(v);
        case Add(Expr l, Expr r)      -> render(l) + " + " + render(r);
        case Negate(Expr expr)        -> "-" + render(expr);
    };
}
```

**实战模式**：
1. `sealed interface` 定义有限类型
2. 每个子类是 `record`（不可变数据载体）
3. 操作时用 `switch + record pattern` 解构
4. 加新子类 → 编译器把"漏处理"暴露在 build 时

#### ⑥ sealed interface vs sealed class

```java
// sealed interface - 可多实现（推荐）
public sealed interface JsonValue
    permits JsonNull, JsonNumber, JsonString, JsonArray, JsonObject {}

// 一个子类可同时实现多个 sealed interface
public record JsonObject(Map<String, JsonValue> entries)
    implements JsonValue, Iterable<Map.Entry<String, JsonValue>> {}

// sealed class - 单继承
public sealed class Result<T>
    permits Success, Failure { ... }  // Success 不能同时 extends Other
```

**选型**：**优先用 `sealed interface`**（更灵活，不锁死单继承）。

#### ⑦ sealed vs enum 对比

| 维度 | enum | sealed |
|---|---|---|
| 子类数 | 固定（编译时）| 固定（编译时）|
| 子类可携带不同字段 | ❌ | ✅ |
| `instanceof` pattern | ❌ | ✅ |
| exhaustive switch | ✅ | ✅ |
| 可继承自己的子类 | ❌ | ✅（用 sealed 修饰符）|
| 适用 | 简单状态（color/role）| 复杂数据（AST/Result/Event）|
| 实例数 | 枚举值 = 实例 | 每个 record 可创建多个实例 |

**经验法则**：
- **enum**：所有实例字段相同（`Status.ACTIVE("Active")`）
- **sealed**：每个 record 子类字段不同（`Add(Expr, Expr)` vs `Negate(Expr)`）

---

**核心洞察**：sealed class 是 Java 的 **ADT（Algebraic Data Type）**——把"封闭的有限可能性"编码进类型系统。配合 record + pattern matching switch，**编译器帮你验证穷举**，传统 OOP 的 `instanceof` + cast + 漏 case 全部消失。这是 Java 16-17 给"类型驱动开发"的杀手锏，**所有 AST / Result / Event / State 设计都应该重新评估能否改写成 sealed + record**。

## 3. pattern matching instanceof（JEP 394, Java 16 final）

```java
// Java 15 之前
if (obj instanceof String) {
    String s = (String) obj;       // 显式转型
    System.out.println(s.length());
}

// Java 16+
if (obj instanceof String s) {     // pattern binding - 自动转型 + 绑定
    System.out.println(s.length());
}

// 复杂场景
if (obj instanceof Integer i && i > 0) {
    System.out.println("positive: " + i);
}
```

**模式匹配的演进**：

```
Java 16:  instanceof pattern (final)
Java 17:  switch pattern (preview 1)
Java 21:  switch pattern (final) + record pattern (final)
Java 25:  primitive pattern (preview 3)
```

## 4. switch 表达式（JEP 361, Java 14 final）

```java
// Java 14 之前 - switch 是语句
String label;
switch (day) {
    case MONDAY: label = "Mon"; break;
    case TUESDAY: label = "Tue"; break;
    case WEDNESDAY: label = "Wed"; break;
    default: label = "Other"; break;
}

// Java 14+ - switch 是表达式
String label = switch (day) {
    case MONDAY -> "Mon";
    case TUESDAY -> "Tue";
    case WEDNESDAY -> "Wed";
    default -> "Other";
};

// 多值匹配
String label = switch (day) {
    case MONDAY, TUESDAY, WEDNESDAY, THURSDAY, FRIDAY -> "Weekday";
    case SATURDAY, SUNDAY -> "Weekend";
};

// 块体 switch（Java 14+）
String label = switch (day) {
    case MONDAY -> {
        System.out.println("Monday blues");
        yield "Mon";  // yield 返回值
    }
    default -> "Other";
};
```

**关键差异**：

| 维度 | switch 语句（Java 14 前）| switch 表达式（Java 14+）|
|---|---|---|
| 有返回值？ | ❌ | ✅ |
| 默认 fall-through？ | ✅ | ❌（每个 case 必须穷尽）|
| 多值匹配 | ❌ | ✅ |
| `yield` | ❌ | ✅ |

## 5. text blocks（JEP 378, Java 15 final）

```java
// Java 14 之前 - 多行字符串拼凑
String json = "{\n" +
              "  \"name\": \"Mike\",\n" +
              "  \"age\": 30\n" +
              "}";

// Java 15+
String json = """
        {
          "name": "Mike",
          "age": 30
        }
        """;

// 注意：
// - 开头 """ 后面必须换行
// - 缩进按 closing """ 左侧对齐计算（自动剥离公共缩进）
// - 可以包含 " 字符（不需要转义）
// - 可以用 \ 续行
String html = """
        <html>
            <body>
                <p>Hello, World!</p>
            </body>
        </html>
        """;
```

**自动缩进处理**：Java 编译器自动检测 closing `"""` 左侧的空白，剥离所有行的同等缩进。

## 6. pattern matching switch（JEP 406, Java 17 preview）

```java
// Java 17 preview - 类型驱动的 switch
static String formatter(Object obj) {
    return switch (obj) {
        case Integer i -> "Integer: " + i;
        case Long l    -> "Long: " + l;
        case Double d  -> "Double: " + d;
        case String s  -> "String: " + s;
        default        -> "Unknown: " + obj;
    };
}

// null 安全
static String formatter(Object obj) {
    return switch (obj) {
        case null        -> "null";
        case Integer i   -> "Integer: " + i;
        case String s    -> "String: " + s;
        default          -> obj.toString();
    };
}
```

**关键点**：
- switch 现在支持**类型 pattern**（`case Integer i`）
- `case null` 显式处理 null（不会被 NullPointerException）
- 配合 sealed class 可实现 **exhaustive switch**（编译器知道所有子类）

```java
static double area(Shape shape) {
    return switch (shape) {
        case Circle c    -> Math.PI * c.radius() * c.radius();
        case Rectangle r -> r.width() * r.height();
        case Square s    -> s.side() * s.side();
        // 没有 default，编译器验证已穷举 sealed permits
    };
}
```

> Java 21 后这个特性**正式落地**，详见 [[draft-04-java-21-lts-virtual-threads#3-pattern-matching-switch-standard]]。

## 7. 关键设计原则

1. **record 替代 90% 的 POJO** — DTO / VO / 值对象 / 多返回值都用 record
2. **sealed 表达封闭领域** — 当所有子类已知且有限时（AST / Expr / Result / Status）
3. **instanceof pattern 替代手动转型** — 不再写 `(String) obj`
4. **switch 表达式替代 if-else 链** — 多分支场景可读性 + 穷尽性
5. **text blocks 用于 JSON / HTML / SQL** — 但单行字符串仍用双引号

## 8. 反模式

❌ **record 当 entity / ORM 实体** — JPA / Hibernate 需要可变 setter
❌ **sealed + 非 final 子类** — 失去了封闭的意义
❌ **switch 表达式不写 default** — 即使你认为"不会发生"
❌ **text blocks 用错缩进** — closing `"""` 的缩进决定公共缩进
❌ **instanceof pattern 不绑定** — `if (obj instanceof String)`（无绑定名）警告
❌ **传统 switch 继续用** — 新代码统一用 switch 表达式

## 相关笔记

- **Java 8 基础**: [[draft-01-java-8-foundations]]
- **Java 21 LTS**: [[draft-04-java-21-lts-virtual-threads]]
- **Java 22-25**: [[draft-05-java-22-25-modern-concurrency]]
- **Kotlin data class**: [[../kotlin-analysis/draft-06-oop#2-data-class杀手特性]]
- **TypeScript type narrowing**: [[../typescript-analysis/draft-03-type-system]]
