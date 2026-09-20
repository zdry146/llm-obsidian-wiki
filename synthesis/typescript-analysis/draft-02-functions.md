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
base_confidence: 0.88
lifecycle: draft
lifecycle_changed: 2026-09-20
created: 2026-09-20
updated: 2026-09-20
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