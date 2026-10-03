---
title: "TypeScript 函数与异步"
category: synthesis
tags: [typescript, functions, arrow-functions, async-await, promise, generics]
sources:
  - "TypeScript Docs - Functions"
  - "TypeScript Docs - More on Functions"
  - "MDN - async function"
summary: "TypeScript 函数：函数类型、可选/默认/剩余参数、函数重载、async/await、Promise、迭代器"
provenance:
  extracted: 0.90
  inferred: 0.08
    ambiguous: 0.02
base_confidence: 0.90
lifecycle: draft
lifecycle_changed: 2026-10-03
created: 2026-09-20
updated: 2026-10-03
---

# §02 TypeScript 函数与异步

## 1. 函数定义 3 种方式

```typescript
// 1. 函数声明（可提升）
function add(a: number, b: number): number {
    return a + b;
}

// 2. 函数表达式（const 推荐，更严格）
const subtract = (a: number, b: number): number => a - b;

// 3. 方法简写
const obj = {
    name: "Mike",
    greet(): string {
        return `Hello, ${this.name}!`;
    }
};
```

## 2. 函数类型

```typescript
// 类型别名定义函数类型
type BinaryOp = (a: number, b: number) => number;

const add: BinaryOp = (a, b) => a + b;
const mul: BinaryOp = (a, b) => a * b;

// 接口定义函数类型
interface Handler {
    (req: Request, res: Response): void;
}

// 函数作为参数
function apply(op: BinaryOp, a: number, b: number): number {
    return op(a, b);
}

apply(add, 1, 2);  // 3
```

## 3. 可选参数与默认参数

```typescript
// 可选参数（参数后加 ?）
function greet(name: string, greeting?: string): string {
    return `${greeting ?? "Hello"}, ${name}`;
}
greet("Mike");              // "Hello, Mike"
greet("Mike", "Hi");        // "Hi, Mike"

// 默认参数（推荐，ES6+）
function greet2(name: string, greeting: string = "Hello"): string {
    return `${greeting}, ${name}`;
}

// ⚠️ 可选参数必须在必选参数之后
function bad(a?: number, b: number): void {}  // ❌

// ✅ 正确：必选在前，可选在后
function good(a: number, b?: number): void {}
```

## 4. 剩余参数与展开

```typescript
// 剩余参数
function sum(prefix: string, ...numbers: number[]): string {
    return `${prefix}: ${numbers.reduce((a, b) => a + b, 0)}`;
}
sum("Sum", 1, 2, 3);  // "Sum: 6"

// 元组类型推断（TS 4.0+）
function log(...args: [...string[], number]) {
    // 至少一个数字，前面任意字符串
}
log("a", "b", 1);  // OK
log(1);            // ❌ 至少需要一个 string
```

### 4.1 综合拆解：函数声明的 6 种形态

#### ① 函数声明（Function Declaration）

```typescript
function add(a: number, b: number): number {
    return a + b;
}
```

- **`function` 关键字 + 函数名** = 声明式
- **会 hoist（提升）**：可以在声明之前调用
  ```typescript
  add(1, 2);  // ✅ 即使 add 在下面声明也能跑
  function add(a, b) { return a + b; }
  ```
- 适用：模块顶层 / 公共工具函数

#### ② 箭头函数（Function Expression）

```typescript
const add2 = (a: number, b: number): number => a + b;
```

- **没有名字，赋值给变量**
- **不会 hoist**：必须先声明后使用
- 单表达式时 `{ return ... }` 可省略为 `=> ...`
- **不能**当构造函数用（`new add2()` 报错）
- `this` 是**词法绑定**（定义时的外层 `this`），不是动态绑定

**function vs 箭头 关键差异**：

| 维度 | `function add() {}` | `const add = () => {}` |
|---|---|---|
| Hoist | ✅ | ❌ |
| `this` 绑定 | 动态（运行时） | 词法（定义时） |
| `arguments` 对象 | ✅ | ❌ |
| 能 `new` 吗 | ✅ | ❌ |
| 适合做 | 普通函数、构造器 | 回调、短小表达式 |

#### ③ 函数类型别名

```typescript
type AddFn = (a: number, b: number) => number;
const add3: AddFn = (a, b) => a + b;
```

- `type AddFn = ...` 定义**函数类型别名**
- 关键点：**函数类型的写法 `(args) => returnType`** —— 长得跟箭头函数一样但语义不同
- 变量侧 `add3` 不需要再写参数类型（自动从 `AddFn` 推断）
- 适用：复杂函数签名复用

对比 `interface` 写法：
```typescript
interface AddFnI {
    (a: number, b: number): number;   // 用 : 返回类型，不是 =>
}
```

`type` vs `interface` 在函数类型上**基本可互换**，但 `type` 还能表达 union / intersection / 工具类型，能力更广。

#### ④ 可选参数（`?`）

```typescript
function greet(name: string, greeting?: string): string {
    return `${greeting ?? "Hello"}, ${name}!`;
}
```

- 参数后加 `?` = **可传可不传**
- TS 推断 `greeting` 类型为 **`string | undefined`**（不是 `string`）
- 直接 `greeting.toUpperCase()` 报错，需要：
  - `greeting?.toUpperCase()` — optional chaining
  - `greeting ?? "Hello"` — **只对 `null`/`undefined` 给默认值**，比 `||` 精确
- **必须放在必填参数之后**：`(name?: string, greeting: string)` ❌ 编译报错

#### ⑤ 默认参数（`=`）

```typescript
function greet2(name: string, greeting: string = "Hello"): string {
    return `${greeting}, ${name}!`;
}
```

- 参数后 `= "Hello"` = **不传时使用默认值**
- TS **自动收窄类型**：`greeting` 在函数体内就是 `string`（不是 `string | undefined`），不需要 `??`

**`?` vs `=`**：

| 维度 | `greeting?: string` | `greeting: string = "Hello"` |
|---|---|---|
| 必须传？ | ❌ | ❌ |
| 函数体内类型 | `string \| undefined` | `string`（已收窄） |
| 默认值 | 无（手动给）| 自动 `"Hello"` |
| 适用 | "可有可无" | "有合理默认值，多数时候省略" |

#### ⑥ 剩余参数（`...rest`）

```typescript
function sum(...numbers: number[]): number {
    return numbers.reduce((a, b) => a + b, 0);
}
```

- `...numbers: number[]` = **把所有剩余参数打包成数组**
- 函数体内 `numbers` 就是 `number[]`
- **只能有一个**，且必须放在最后：
  ```typescript
  function f(...rest: number[], last: string)  // ❌
  function f(first: string, ...rest: number[])  // ✅
  ```
- 两种形态对比：
  - `...number[]` — 数组类型，每个元素都是 number（实践常用）
  - `...[string, number, boolean]` — tuple 类型，每个位置类型已知（TS 4.0+）

对比 ES5 时代（无 rest）：
```javascript
// 老写法：靠 arguments 对象
function sum() {
    var args = Array.prototype.slice.call(arguments);  // 必须手动转数组
    return args.reduce(function(a, b) { return a + b; }, 0);
}
```

---

**核心洞察**：TypeScript 函数语法 = **JS 函数语法 + 类型注解**。重点不是新概念，是把"运行时写法"补上"编译期类型契约"：`?` 表达"可省"、`=` 表达"有默认"、`...rest` 表达"打包成数组"，每个语法都有清晰的语义边界。学会用 `type AddFn = ...` 把函数签名抽出来，是 TS 类型体操的入门砖。

## 5. this 参数（TypeScript 独有）

```typescript
// 显式声明 this 类型
interface User {
    name: string;
    greet(this: User): string;
}

const user: User = {
    name: "Mike",
    greet() {
        return `Hello, ${this.name}`;  // this 是 User
    }
};

// 实战：回调中 this 类型
function bindEvent(handler: (this: HTMLButtonElement, e: MouseEvent) => void) {
    button.addEventListener("click", handler);
}
```

## 6. 函数重载（Overload）

```typescript
// 重载签名（多个 + 1 个实现）
function process(input: string): string;
function process(input: number): number;
function process(input: boolean): boolean;

// 实现签名（必须兼容所有重载）
function process(input: string | number | boolean): string | number | boolean {
    if (typeof input === "string") return input.toUpperCase();
    if (typeof input === "number") return input * 2;
    return !input;
}

process("hello");  // "HELLO"
process(5);        // 10
```

**实战：API 兼容多种输入**：

```typescript
function makeDate(timestamp: number): Date;
function makeDate(year: number, month: number, day: number): Date;
function makeDate(yearOrTimestamp: number, month?: number, day?: number): Date {
    if (month !== undefined && day !== undefined) {
        return new Date(yearOrTimestamp, month - 1, day);
    }
    return new Date(yearOrTimestamp);
}
```

## 7. 异步基础：Promise

```typescript
// 创建 Promise
const fetchUser = (id: string): Promise<User> => {
    return fetch(`/api/users/${id}`).then(res => res.json());
};

// 链式调用
fetchUser("123")
    .then(user => console.log(user.name))
    .catch(err => console.error(err))
    .finally(() => console.log("done"));

// Promise.all：并行等待
const [user, posts] = await Promise.all([
    fetchUser("123"),
    fetchPosts("123")
]);

// Promise.race：最快一个
const fastest = await Promise.race([
    fetch("https://api1.com"),
    fetch("https://api2.com")
]);

// Promise.allSettled：所有结果（成功/失败都返回）
const results = await Promise.allSettled([fetch1, fetch2]);
results.forEach(r => {
    if (r.status === "fulfilled") console.log(r.value);
    else console.error(r.reason);
});
```

## 8. async/await

```typescript
// async 函数自动返回 Promise<T>
async function fetchUser(id: string): Promise<User> {
    const res = await fetch(`/api/users/${id}`);
    if (!res.ok) {
        throw new Error(`HTTP ${res.status}`);
    }
    return res.json();
}

// 错误处理：try/catch
async function loadUser(id: string): Promise<User> {
    try {
        const user = await fetchUser(id);
        return user;
    } catch (e) {
        console.error("Failed to load user", e);
        throw e;  // 重新抛出
    }
}

// 并行：Promise.all + await
async function loadDashboard(): Promise<Dashboard> {
    const [user, posts, stats] = await Promise.all([
        fetchUser("123"),
        fetchPosts("123"),
        fetchStats("123")
    ]);
    return { user, posts, stats };
}
```

## 9. 异步迭代器（Async Iterable）

```typescript
// 异步生成器
async function* fetchPages(url: string): AsyncGenerator<Page> {
    let page = 1;
    while (true) {
        const res = await fetch(`${url}?page=${page}`);
        const data = await res.json();
        if (data.items.length === 0) break;
        yield data;
        page++;
    }
}

// 使用
for await (const page of fetchPages("/api/items")) {
    console.log(page);
}

// 异步迭代器（Array → Stream）
async function* toAsync<T>(items: T[]): AsyncIterableIterator<T> {
    for (const item of items) {
        yield item;
        await new Promise(r => setTimeout(r, 100));  // 模拟延迟
    }
}
```

## 10. 实战：HTTP 客户端

```typescript
class HttpClient {
    constructor(private baseURL: string) {}

    async get<T>(path: string): Promise<T> {
        const res = await fetch(`${this.baseURL}${path}`, {
            method: "GET",
            headers: { "Content-Type": "application/json" }
        });
        if (!res.ok) {
            throw new HttpError(res.status, res.statusText);
        }
        return res.json() as Promise<T>;
    }

    async post<T, B>(path: string, body: B): Promise<T> {
        const res = await fetch(`${this.baseURL}${path}`, {
            method: "POST",
            headers: { "Content-Type": "application/json" },
            body: JSON.stringify(body)
        });
        if (!res.ok) {
            throw new HttpError(res.status, res.statusText);
        }
        return res.json() as Promise<T>;
    }
}

// 使用
const client = new HttpClient("https://api.example.com");
const user = await client.get<User>("/users/123");
```

## 11. 高阶函数与柯里化

```typescript
// 高阶函数（接受/返回函数）
function withLogging<T extends (...args: any[]) => any>(fn: T): T {
    return ((...args: Parameters<T>) => {
        console.log(`Calling ${fn.name}`);
        const result = fn(...args);
        console.log(`Result: ${result}`);
        return result;
    }) as T;
}

const add = (a: number, b: number) => a + b;
const loggedAdd = withLogging(add);
loggedAdd(1, 2);  // 调用日志 + 结果日志

// 柯里化（部分应用）
function curry<A, B, C>(fn: (a: A, b: B) => C): (a: A) => (b: B) => C {
    return a => b => fn(a, b);
}

const addCurried = curry((a: number, b: number) => a + b);
const add5 = addCurried(5);
add5(3);  // 8
```

## 12. 类型守卫函数

```typescript
// 自定义类型守卫
function isString(value: unknown): value is string {
    return typeof value === "string";
}

function isUser(obj: unknown): obj is User {
    return (
        typeof obj === "object" &&
        obj !== null &&
        "id" in obj &&
        "name" in obj
    );
}

// 使用：自动 narrow
function process(value: string | number | null) {
    if (isString(value)) {
        return value.toUpperCase();  // value 是 string
    }
    if (typeof value === "number") {
        return value.toFixed(2);
    }
    return "unknown";
}
```

详见 [[draft-03-type-system#类型守卫]]。

## 13. 关键设计原则

1. **const 函数表达式优先**——更严格的 this 绑定
2. **默认参数优于可选参数**——更易理解
3. **重载提供更好类型推断**——用户体验
4. **async/await 优于 Promise 链**——可读性
5. **类型守卫用于 narrow**——类型安全

## 14. 反模式

❌ **function() {} vs () => {} 混用**——this 绑定混乱
❌ **可选参数在必选参数前**——编译错误
❌ **重载太多**——3 个以上考虑用对象参数
❌ **async 函数不用 try/catch**——异常吞噬
❌ **Promise.all vs for await 混淆**——选错模式

## 相关笔记

- **类型系统**: [[draft-03-type-system]]
- **泛型**: [[draft-04-generics]]
- **类与 OOP**: [[draft-05-classes-oop]]
- **最佳实践**: [[draft-09-best-practices]]
- **综合入口**: [[summary]]