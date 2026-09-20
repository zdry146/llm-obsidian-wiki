---
title: "TypeScript 类与面向对象"
category: synthesis
tags: [typescript, classes, oop, modifiers, abstract, implements, decorators]
sources:
  - "TypeScript Docs - Classes"
  - "TypeScript Docs - Decorators"
  - "MDN - Classes"
summary: "TypeScript 类：访问修饰符、abstract、implements、override、装饰器（实验性）、私有字段"
provenance:
  extracted: 0.90
  inferred: 0.08
  ambiguous: 0.02
base_confidence: 0.88
lifecycle: draft
lifecycle_changed: 2026-09-20
created: 2026-09-20
updated: 2026-09-20
---

# §05 TypeScript 类与面向对象

## 1. 基本类

```typescript
class User {
    // 实例字段
    name: string;
    age: number;
    
    // 构造器
    constructor(name: string, age: number) {
        this.name = name;
        this.age = age;
    }
    
    // 方法
    greet(): string {
        return `Hello, ${this.name}!`;
    }
    
    // 静态字段
    static species = "Homo sapiens";
    
    // 静态方法
    static createAnonymous(): User {
        return new User("Anonymous", 0);
    }
}

const u = new User("Mike", 30);
u.greet();                     // "Hello, Mike!"
User.species;                  // "Homo sapiens"
User.createAnonymous();        // User { name: "Anonymous" }
```

## 2. 访问修饰符（5 种）

```typescript
class Account {
    public name: string;          // 公开（默认）
    private balance: number;      // 类内部访问
    protected password: string;   // 子类访问
    readonly id: string;          // 只读
    
    constructor(name: string, balance: number) {
        this.name = name;
        this.balance = balance;
        this.id = crypto.randomUUID();
    }
    
    deposit(amount: number) {
        if (amount <= 0) throw new Error("Invalid amount");
        this.balance += amount;
    }
    
    protected log(message: string) {
        console.log(`[${this.name}] ${message}`);
    }
}

const acc = new Account("Mike", 1000);
acc.deposit(500);                 // ✅
// acc.balance = 999999;          // ❌ private
// acc.log("test");               // ❌ protected

// ✅ readonly
// acc.id = "new";                // ❌ 只读
```

**5 种修饰符**：

| 修饰符 | 范围 |
|--------|------|
| `public`（默认）| 所有 |
| `private` | 类内部 |
| `protected` | 类内部 + 子类 |
| `readonly` | 不可写 |
| `#name`（ES 私有字段）| 类内部（运行时真私有） |

## 3. 抽象类（abstract）

```typescript
abstract class Shape {
    abstract area(): number;
    abstract perimeter(): number;
    
    // 具体方法
    describe(): string {
        return `${this.constructor.name}: area=${this.area()}, perimeter=${this.perimeter()}`;
    }
}

class Circle extends Shape {
    constructor(public radius: number) {
        super();
    }
    
    area(): number {
        return Math.PI * this.radius ** 2;
    }
    
    perimeter(): number {
        return 2 * Math.PI * this.radius;
    }
}

// const s = new Shape();  // ❌ 不能实例化抽象类
const c = new Circle(5);
c.describe();  // "Circle: area=78.5..., perimeter=31.4..."
```

## 4. implements（实现接口）

```typescript
interface Serializable {
    serialize(): string;
    deserialize(data: string): void;
}

interface Loggable {
    log(level: string, message: string): void;
}

// 多接口实现
class User implements Serializable, Loggable {
    constructor(public name: string, public age: number) {}
    
    serialize(): string {
        return JSON.stringify(this);
    }
    
    deserialize(data: string): void {
        const obj = JSON.parse(data);
        this.name = obj.name;
        this.age = obj.age;
    }
    
    log(level: string, message: string): void {
        console.log(`[${level}] ${this.name}: ${message}`);
    }
}
```

## 5. 继承（extends）

```typescript
class Animal {
    constructor(public name: string) {}
    
    speak(): void {
        console.log(`${this.name} makes a sound`);
    }
}

class Dog extends Animal {
    constructor(name: string, public breed: string) {
        super(name);                  // 必须调 super
    }
    
    // 重写方法
    override speak(): void {
        console.log(`${this.name} barks`);
    }
    
    // 子类特有方法
    fetch(): void {
        console.log(`${this.name} fetches the ball`);
    }
}

const d = new Dog("Buddy", "Labrador");
d.speak();   // "Buddy barks"
d.fetch();   // "Buddy fetches the ball"
```

**TS 4.3+ 的 `override` 关键字**：显式标记重写方法，避免拼写错误导致意外新建方法。

## 6. getter / setter

```typescript
class Temperature {
    private _celsius: number;
    
    constructor(celsius: number) {
        this._celsius = celsius;
    }
    
    get fahrenheit(): number {
        return this._celsius * 9/5 + 32;
    }
    
    set fahrenheit(value: number) {
        this._celsius = (value - 32) * 5/9;
    }
}

const t = new Temperature(100);
console.log(t.fahrenheit);  // 212
t.fahrenheit = 32;
console.log(t._celsius);     // 0
```

## 7. 泛型类

```typescript
class Container<T> {
    constructor(private value: T) {}
    
    getValue(): T {
        return this.value;
    }
    
    map<U>(fn: (value: T) => U): Container<U> {
        return new Container(fn(this.value));
    }
}

const c1 = new Container(42).map(x => x.toString());
// Container<string>

// 多类型参数 + 约束
class KeyValuePair<K extends string, V> {
    constructor(public key: K, public value: V) {}
    
    toString(): string {
        return `${this.key}=${this.value}`;
    }
}

const kv = new KeyValuePair("name", "Mike");
kv.toString();  // "name=Mike"
```

## 8. this 类型（流畅接口）

```typescript
class Calculator {
    private value: number = 0;
    
    add(n: number): this {
        this.value += n;
        return this;
    }
    
    multiply(n: number): this {
        this.value *= n;
        return this;
    }
    
    getValue(): number {
        return this.value;
    }
}

// 链式调用
const result = new Calculator()
    .add(5)       // Calculator
    .multiply(3)  // Calculator
    .add(10)      // Calculator
    .getValue();  // 25
```

## 9. 私有字段（ES2022 `#`）

```typescript
class Counter {
    // 真私有（运行时不可访问）
    #count = 0;
    
    increment(): void {
        this.#count++;
    }
    
    getCount(): number {
        return this.#count;
    }
}

const c = new Counter();
c.increment();
// c.#count          // ❌ 编译错（语法层面）
// c["#count"]        // ❌ 运行错（真私有）
c.getCount();          // 1
```

**TypeScript `private` vs ES `#`**：
- `private` 编译期检查，运行时可访问（仅约定）
- `#` 编译期 + 运行期都不可访问（真正私有）

## 10. 装饰器（Decorators，TS 5.0+ 标准化）

```typescript
// 启用：tsconfig.json 加 "experimentalDecorators": true
// TS 5.0+ 也支持 ECMAScript Stage 3 装饰器（无需 experimentalDecorators）

// 类装饰器
function sealed(constructor: Function) {
    Object.seal(constructor);
    Object.seal(constructor.prototype);
}

@sealed
class Greeter {
    greeting: string;
    constructor(message: string) {
        this.greeting = message;
    }
    greet() {
        return "Hello, " + this.greeting;
    }
}

// 方法装饰器（实战：日志 + 性能监控）
function log(target: any, key: string, descriptor: PropertyDescriptor) {
    const original = descriptor.value;
    descriptor.value = function (...args: any[]) {
        const start = Date.now();
        const result = original.apply(this, args);
        console.log(`${key} took ${Date.now() - start}ms`);
        return result;
    };
    return descriptor;
}

class Calculator {
    @log
    add(a: number, b: number): number {
        return a + b;
    }
}

new Calculator().add(1, 2);  // 输出 "add took 0ms"
```

**实战装饰器**：
- Angular / NestJS 大量使用装饰器（`@Controller`、`@Injectable`）
- 类装饰器 / 方法装饰器 / 属性装饰器 / 参数装饰器
- 装饰器是元编程工具，谨慎使用

## 11. 单例模式（TS 实战）

```typescript
class Database {
    private static instance: Database;
    
    private constructor(private connectionString: string) {}
    
    static getInstance(): Database {
        if (!Database.instance) {
            Database.instance = new Database("mongodb://localhost");
        }
        return Database.instance;
    }
    
    connect(): void {
        console.log(`Connected to ${this.connectionString}`);
    }
}

const db1 = Database.getInstance();
const db2 = Database.getInstance();
db1 === db2;  // true
```

## 12. 实战案例

### 12.1 Builder 模式

```typescript
class HttpRequestBuilder {
    private config: Partial<RequestInit> = {};
    
    method(method: string): this {
        this.config.method = method;
        return this;
    }
    
    header(key: string, value: string): this {
        this.config.headers = { ...this.config.headers, [key]: value };
        return this;
    }
    
    body(data: unknown): this {
        this.config.body = JSON.stringify(data);
        return this;
    }
    
    url(url: string): this {
        this._url = url;
        return this;
    }
    
    private _url: string = "";
    
    build(): { url: string; config: RequestInit } {
        return { url: this._url, config: this.config };
    }
}

const req = new HttpRequestBuilder()
    .url("https://api.example.com/users")
    .method("POST")
    .header("Content-Type", "application/json")
    .body({ name: "Mike" })
    .build();
```

### 12.2 抽象数据源

```typescript
abstract class DataSource<T> {
    abstract fetch(id: string): Promise<T | null>;
    abstract save(item: T): Promise<void>;
    abstract delete(id: string): Promise<void>;
}

class UserDataSource extends DataSource<User> {
    async fetch(id: string): Promise<User | null> {
        // 实现
        return await db.query("SELECT * FROM users WHERE id = ?", [id]);
    }
    
    async save(user: User): Promise<void> {
        await db.execute("INSERT INTO users ...", [user]);
    }
    
    async delete(id: string): Promise<void> {
        await db.execute("DELETE FROM users WHERE id = ?", [id]);
    }
}
```

### 12.3 Mixin（用交叉类型实现）

```typescript
type Constructor<T = {}> = new (...args: any[]) => T;

// Mixin 函数
function Timestamped<TBase extends Constructor>(Base: TBase) {
    return class extends Base {
        timestamp = Date.now();
    };
}

// 使用
class User {
    constructor(public name: string) {}
}

const TimestampedUser = Timestamped(User);
const u = new TimestampedUser("Mike");
u.name;       // "Mike"
u.timestamp;  // 1700000000000
```

## 13. 抽象类 vs 接口

```typescript
// 接口：纯契约，无实现
interface Animal {
    speak(): void;
}

// 抽象类：部分实现 + 契约
abstract class Animal {
    abstract speak(): void;
    
    // 共用实现
    describe(): string {
        return `I am a ${this.constructor.name}`;
    }
}

// 实战选择：
// - 仅契约 → interface
// - 部分实现 → abstract class
// - 多重继承 → interface（可多个）+ abstract class（单一）
```

## 14. 关键设计原则

1. **public 默认**——private/protected 按需用
2. **`readonly` 保护不变量**——属性不可变
3. **`override` 标记重写**——避免拼写错误
4. **抽象类部分实现**——不要全用接口
5. **this 返回支持链式**——Fluent API
6. **装饰器元编程**——Angular/NestJS 风格

## 15. 反模式

❌ **过度使用继承**——优先组合
❌ **过度使用装饰器**——难调试
❌ **深度类层次**（>3 层）——难维护
❌ **`private` 字段暴露 getter**——破坏封装
❌ **静态状态全局可变**——难测试

## 相关笔记

- **核心语法**: [[draft-01-syntax-core]]
- **类型系统**: [[draft-03-type-system]]
- **泛型**: [[draft-04-generics]]
- **模块**: [[draft-06-modules]]
- **最佳实践**: [[draft-09-best-practices]]
- **综合入口**: [[summary]]