---
title: "TypeScript vs JavaScript 深度对比 (Deep Dive)"
category: synthesis
tags: [typescript, javascript, comparison, type-system, runtime, migration, interop]
sources:
  - "TypeScript Docs - Type Compatibility"
  - "TypeScript Docs - Migrating from JavaScript"
  - "TypeScript Deep Dive (Basarat Ali Syed)"
  - "MDN JavaScript Reference"
summary: "TypeScript 与 JavaScript 深度对比 - 类型/函数/类/异步/模块/工具链/运行时/迁移/互操作的完整 side-by-side"
provenance:
  extracted: 0.92
  inferred: 0.06
  ambiguous: 0.02
base_confidence: 0.90
lifecycle: draft
lifecycle_changed: 2026-09-20
created: 2026-09-20
updated: 2026-09-20
---

# §13 TypeScript vs JavaScript 深度对比 (Deep Dive)

> 本章节专门深度对比 TypeScript 与 JavaScript 的核心差异。不是「选哪个」，而是「具体哪里不同」。涵盖 13 个维度，每个都有 side-by-side 代码示例。

## 1. 一句话关系

**TypeScript 是 JavaScript 的超集**——所有合法 JS 都是合法 TS，TS 加静态类型 + 现代特性，编译到纯 JS 在任何环境运行。

**关键洞察**：TS 不替代 JS，是渐进叠加——你可以在 JS 项目里**逐步加 TS 文件**。

## 2. 类型系统（最核心差异）

### 2.1 基础类型注解

```javascript
// JavaScript：动态类型（运行时才检查）
function add(a, b) {
    return a + b;
}
add(1, "2");           // "12"（字符串拼接，bug）
add(1, null);          // 1（null 转 0）
add(1, undefined);     // NaN
add({}, []);           // "object"
add("a", { toString: () => "b" });  // "ab"
```

```typescript
// TypeScript：静态类型（编译期检查）
function add(a: number, b: number): number {
    return a + b;
}
add(1, "2");           // ❌ 编译错误：'string' is not 'number'
add(1, 2);              // ✅ 3

// 运行时：完全相同的代码（类型擦除）
// 编译后：(a, b) => a + b（无类型信息）
```

**核心差异**：
- JS：运行时发现错误（runtime error）
- TS：编译期发现错误（compile error）

### 2.2 null/undefined 处理

```javascript
// JavaScript：null/undefined 是 bug 温床
function greet(name) {
    return `Hello, ${name.toUpperCase()}`;  // null → "Hello, NULL"
    // undefined → TypeError: Cannot read property of undefined
}
greet(null);
greet(undefined);  // 💥
```

```typescript
// TypeScript：类型系统强制处理 null
function greet(name: string | null): string {
    if (name === null) {
        return "Hello, Guest";
    }
    return `Hello, ${name.toUpperCase()}`;
}

// strictNullChecks: false（默认关闭）
function bad(name: string | null) {
    return name.toUpperCase();  // ⚠️ 警告但通过

// strictNullChecks: true（推荐）
function good(name: string | null) {
    return name.toUpperCase();  // ❌ 编译错误
}
```

### 2.3 类型推断对比

```javascript
// JS：没有类型推断（变量无类型）
const data = fetchUser();
data.name;           // IDE 无法帮助
data.nonExistent;    // 运行时 undefined

// ES2020+ 链式可选
data?.name?.toUpperCase();  // 手动 null check
```

```typescript
// TS：自动推断
const data = fetchUser();  // 类型：User | undefined（如果返回类型如此）
data.name;                   // ✅ IDE 智能提示
data.nonExistent;           // ❌ 编译错误

// 类型守卫
if (data) {
    data.name;              // ✅ data 类型缩窄为 User
}
```

## 3. 函数签名

### 3.1 参数类型 + 返回类型

```javascript
// JS：参数无类型
function process(data, callback) {
    const result = transform(data);
    callback(result);
}
// 不知道 data 是什么、callback 接受什么
```

```typescript
// TS：完整签名
function process(data: UserData, callback: (result: TransformResult) => void): void {
    const result = transform(data);
    callback(result);
}

// 调用时 IDE 提示
```

### 3.2 可选参数 vs 默认参数

```javascript
// JS：默认参数（ES2015+）
function greet(name, greeting = "Hello") {
    return `${greeting}, ${name}`;
}
greet("Mike");          // "Hello, Mike"
greet("Mike", "Hi");    // "Hi, Mike"
greet("Mike", undefined); // "Hello, Mike"（undefined 触发默认值）

// JS：可选参数（无类型，靠 undefined 检查）
function old(name, greeting) {
    const g = greeting || "Hello";
    return `${g}, ${name}`;
}
```

```typescript
// TS：可选参数（`?`）
function greet(name: string, greeting?: string): string {
    return `${greeting ?? "Hello"}, ${name}`;
}

// TS：默认参数（推荐）
function greet2(name: string, greeting: string = "Hello"): string {
    return `${greeting}, ${name}`;
}

// TS：可选 + 默认（混合 - 不推荐）
function mixed(name: string, greeting: string = "Hello", suffix?: string): string {
    return `${greeting}, ${name}${suffix ?? ""}`;
}
```

### 3.3 函数重载

```javascript
// JS：单函数模式（用 arguments 或 rest）
function makeDate(...args) {
    if (args.length === 1) return new Date(args[0]);
    if (args.length === 3) return new Date(args[0], args[1] - 1, args[2]);
}
makeDate(1234567890);     // OK
makeDate(2026, 9, 20);   // OK
makeDate("2026-09-20");   // 💥 运行时错
```

```typescript
// TS：函数重载（编译期类型检查）
function makeDate(timestamp: number): Date;
function makeDate(year: number, month: number, day: number): Date;
function makeDate(yearOrTimestamp: number, month?: number, day?: number): Date {
    if (month !== undefined && day !== undefined) {
        return new Date(yearOrTimestamp, month - 1, day);
    }
    return new Date(yearOrTimestamp);
}

makeDate(1234567890);       // ✅ number 版本
makeDate(2026, 9, 20);      // ✅ year/month/day 版本
makeDate("2026-09-20");     // ❌ 编译错
```

## 4. 类与 OOP

### 4.1 基本类

```javascript
// JavaScript ES2022
class User {
    #age;  // 私有字段（#）
    
    constructor(name, age) {
        this.name = name;
        this.#age = age;
    }
    
    greet() {
        return `Hello, ${this.name}`;
    }
}
```

```typescript
// TypeScript
class User {
    private age: number;  // 编译期私有
    
    constructor(public name: string, age: number) {
        this.name = name;
        this.age = age;
    }
    
    greet(): string {
        return `Hello, ${this.name}`;
    }
    
    // 抽象方法、接口实现、装饰器...
}

// 继承
class Admin extends User {
    constructor(name: string, age: number, public role: string) {
        super(name, age);
    }
    
    override greet(): string {
        return `Hello, Admin ${this.name}`;
    }
}
```

### 4.2 抽象类（TS 独有）

```typescript
// TypeScript：抽象类（编译期 + 运行期双重）
abstract class Shape {
    abstract area(): number;
    abstract perimeter(): number;
    
    describe(): string {
        return `area=${this.area()}`;
    }
}

class Circle extends Shape {
    constructor(public radius: number) {
        super();
    }
    
    area(): number { return Math.PI * this.radius ** 2; }
    perimeter(): number { return 2 * Math.PI * this.radius; }
}

// new Shape();  // ❌ 编译错：不能实例化抽象类
```

```javascript
// JavaScript：需要手动抛错模拟
class Shape {
    area() { throw new Error("Not implemented"); }
    perimeter() { throw new Error("Not implemented"); }
}

class Circle extends Shape {
    constructor(radius) { super(); this.radius = radius; }
    area() { return Math.PI * this.radius ** 2; }
}
// 没有编译期检查
```

### 4.3 访问修饰符（TS 独有）

```typescript
class Account {
    public name: string;          // 默认公开
    private balance: number;      // 类内部访问
    protected password: ***;  // 子类访问
    readonly id: string;         // 只读
    
    constructor(name: string, balance: number) {
        this.name = name;
        this.balance = balance;
        this.id = crypto.randomUUID();
    }
}
```

```javascript
// JavaScript：只有 # 私有字段（运行时）
class Account {
    #balance;  // 真私有
    
    constructor(name, balance) {
        this.name = name;          // 公开
        this._password = "";       // 约定私有（不强制）
    }
}
```

**差异**：
- TS `private`：编译期检查（运行时仍可访问）
- JS `#`：运行时真私有（编译期无法 bypass）

## 5. 异步编程

### 5.1 Promise（都支持，相同）

```javascript
// JavaScript
async function fetchUser(id) {
    const res = await fetch(`/api/users/${id}`);
    if (!res.ok) throw new Error(`HTTP ${res.status}`);
    return await res.json();
}
```

```typescript
// TypeScript（语法相同，加类型）
async function fetchUser(id: string): Promise<User> {
    const res = await fetch(`/api/users/${id}`);
    if (!res.ok) throw new Error(`HTTP ${res.status}`);
    return (await res.json()) as User;  // 类型断言
}
```

### 5.2 async/await（都支持）

```javascript
// JavaScript
async function loadDashboard() {
    const user = await fetchUser();
    const posts = await fetchPosts();
    return { user, posts };
}
```

```typescript
// TypeScript（同样的语法 + 类型推断）
async function loadDashboard(): Promise<Dashboard> {
    const [user, posts] = await Promise.all([
        fetchUser(),
        fetchPosts()
    ]);
    // user: User, posts: Post（自动推断）
    return { user, posts };
}
```

**实战**：TS 主要提供**类型安全**——异步控制流语法相同。

### 5.3 TS 额外的类型化工具

```typescript
// 类型安全的 fetch 封装
async function safeFetch<T>(url: string): Promise<T> {
    const res = await fetch(url);
    if (!res.ok) throw new HttpError(res.status);
    return res.json() as Promise<T>;
}

const user = await safeFetch<User>("/api/users/1");
// 编译期：user 是 User 类型
```

## 6. 模块系统

### 6.1 ESM（都支持）

```javascript
// JavaScript ESM
// math.js
export function add(a, b) { return a + b; }
export default class Calculator { }

// main.js
import Calculator, { add } from "./math.js";
```

```typescript
// TypeScript ESM（语法几乎相同）
// math.ts
export function add(a: number, b: number): number {
    return a + b;
}
export default class Calculator { }

// main.ts
import Calculator, { add } from "./math.ts";
```

**核心差异**：
- TS 文件扩展名 `.ts`（编译后 `.js`）
- TS 可编译为 ESM/CJS/UMD

### 6.2 Type-only 导入（TS独有）

```typescript
// TypeScript：type-only 导入（编译擦除）
import type { User, Post } from "./types";
import { type Config, default as Api } from "./api";

// 编译后完全擦除（节省 bundle size）
// type User = ...;（不引入任何运行时代码）
```

```javascript
// JavaScript 没有类型，无此概念
```

## 7. 工具链

### 7.1 编译/构建

| 工具 | JavaScript | TypeScript |
|------|-----------|------------|
| **直接运行** | ✅ Node.js / 浏览器 | ❌ 需编译 |
| **构建工具** | Vite（无需 tsc）| tsc + Vite/esbuild |
| **类型检查** | 无 | tsc（CI 必跑）|
| **代码压缩** | esbuild | esbuild（编译后）|
| **包大小** | 基线 | +5-10%（类型擦除后）|

**实战**：
- TS 项目：`tsc --noEmit`（CI 检查）+ `esbuild` / `Vite`（dev/build）
- JS 项目：直接 `node` / `vite`

### 7.2 IDE 智能提示

```javascript
// JavaScript：弱智能提示
const user = await fetchUser("123");
user.naem;  // IDE 不提示错误，运行时 undefined
```

```typescript
// TypeScript：完整智能提示
const user = await fetchUser("123");
// IDE 显示：user: User
user.naem;   // ❌ IDE 标红：Did you mean 'name'?
user.name;   // ✅ 自动补全
```

**实战差异**：
- JS IDE：基于语法推断（弱）
- TS IDE：基于类型系统（强）

### 7.3 重构能力

```javascript
// JS：手动全局搜索
function User(name) { this.name = name; }
// 改字段名：grep → 改所有引用 → 测试 → 担心漏改
```

```typescript
// TS：自动重构
class User {
    constructor(public name: string) {}
}
// 重命名字段：IDE 自动修改所有引用，编译检查 + 测试保证
```

## 9. 运行时性能

### 9.1 运行时开销

```javascript
// JavaScript：直接执行
node app.js
```

```typescript
// TypeScript：先编译后执行
tsc → app.js
node app.js

// 或 dev：tsx 直接运行
tsx app.ts
```

**性能对比**：

| 维度 | JavaScript | TypeScript |
|------|-----------|------------|
| **启动** | 快（基线）| 同 JS（编译后相同）|
| **执行** | 基线 | **同 JS**（类型擦除）|
| **内存** | 基线 | 同 JS |
| **包大小** | 基线 | +5-10%（运行时声明）|

**核心洞察**：TS 的运行时性能 = JS 的运行时性能**。类型在编译后完全擦除**。

### 9.2 编译开销

```
TS 编译时间（1000 文件）:
- tsc:         ~10s（基线）
- esbuild:     ~0.5s（~20x tsc）
- swc:         ~0.3s（~33x tsc）
- tsx (dev):   ~0.05s（~200x tsc）
```

**实战**：现代项目 dev 用 esbuild/tsx（快），CI 用 tsc（完整类型检查）。

## 10. 生态系统

### 10.1 包大小对比

```bash
# TypeScript 引入带来的间接开销：
# 1. .d.ts 类型声明（npm 上 @types/*）
# 2. TypeScript 本身（devDependency）

# 但运行时无额外开销（类型擦除）
```

### 10.2 DefinitelyTyped（社区类型）

```typescript
// JavaScript 库：手动写 .d.ts（如果作者没提供）
// 或安装社区类型：@types/library-name

// TypeScript：自动获得 IDE 提示
import express from "express";  // 类型来自 @types/express
const app = express();
app.get("/", (req: Request, res: Response) => { /* 完整提示 */ });
```

### 10.3 谁在用

| 公司/项目 | JavaScript | TypeScript |
|---------|-----------|------------|
| GitHub Octoverse 2024 | ✓ | **#4 大语言** |
| React | ✅ | ✅ 默认 |
| Vue | ✅ | ✅ 推荐 |
| Angular | ❌ | ✅ 强制 |
| Next.js | ❌ | ✅ 默认 |
| VS Code | ⚠️ | ✅ TypeScript 写 |
| Deno | ❌ | ✅ 原生 |
| Bun | ❌ | ✅ 原生 |
| 字节跳动 / B站 | ✅ | ✅ 主流 |

## 11. 互操作

### 11.1 TS 调用 JS（allowJs: true）

```typescript
// tsconfig.json
{
    "compilerOptions": {
        "allowJs": true,
        "checkJs": true   // 也对 JS 文件类型检查
    }
}
```

```typescript
// 在 .ts 文件里 import JS
// legacy.js（无类型）
export function oldApi(data) { /* ... */ }

// index.ts
import { oldApi } from "./legacy.js";
// data 类型：any（无类型信息）
```

### 11.2 JS 调用 TS（编译产物）

```typescript
// tsconfig.json
{
    "compilerOptions": {
        "declaration": true   // 生成 .d.ts
    }
}
```

```javascript
// 编译产物（dist/legacy.d.ts）
// export declare function oldApi(data: any): void;
```

```javascript
// 客户端 .js
// @ts-ignore（如果是 Node.js + tsc）
// 或者：import { oldApi } from "./legacy.js"（IDE 看 .d.ts）
```

## 12. 何时用 JS vs TypeScript

| 场景 | 推荐 |
|------|------|
| **新前端项目** | TypeScript（默认）|
| **新 Node.js 后端** | TypeScript（默认）|
| **库（给 JS 用户用）** | TypeScript（发 .d.ts + .js）|
| **简单脚本** | JavaScript（无需编译）|
| **Deno / Bun 项目** | TypeScript（原生支持）|
| **维护老 JS 项目** | 增量 TypeScript（allowJs）|
| **极小 prototype** | JavaScript（快速迭代）|

## 13. JS → TS 迁移指南

### 13.1 渐进式迁移

```bash
# 1. 安装
npm i -D typescript @types/node

# 2. 初始化配置
npx tsc --init
```

```json
// tsconfig.json（迁移配置）
{
    "compilerOptions": {
        "target": "ES2022",
        "module": "ESNext",
        "moduleResolution": "Bundler",
        "allowJs": true,           // 允许 .js 文件
        "checkJs": false,          // 暂时不检查 JS
        "strict": false,           // 先不严格
        "noEmit": true              // 仅类型检查，不生成
    },
    "include": ["src/**/*"]
}
```

### 13.2 重命名 .js → .ts（逐步）

```bash
# 第 1 步：核心模块（如 utils.js → utils.ts）
# 第 2 步：数据模型（如 user.js → user.ts）
# 第 3 步：业务逻辑（按依赖顺序）
# 第 4 步：入口（最后）
```

### 13.3 启用严格模式

```json
// 阶段 1：noImplicitAny（消除 any）
// 阶段 2：strictNullChecks（处理 null）
// 阶段 3：strict: true（其他严格选项）
```

```typescript
// 阶段 2 时的常见问题
function getUser(id) {  // 编译错：参数隐式 any
    // ...
}

// 加类型注解
function getUser(id: string): User {
    // ...
}
```

## 14. 关键速查表

```
                            JavaScript    TypeScript
启动速度                    快            同 JS（编译后）
运行时性能                  基线          同 JS
类型检查                    无            编译期
重构能力                    弱            强
IDE 智能提示                弱            强
null 安全                   无            ✅
联合类型                    无            ✅
泛型                        无            ✅
async/await                 ✅✅✅
类                          ES2022+       + 修饰符/abstract
模块                        ESM/CJS       ESM/CJS + type-only
工具链                      Vite/esbuild  tsc + Vite/esbuild
编译                        无            tsc（必需）
学习曲线                    平            平（JS 基础即可）
生态成熟度                  极高          高（与 JS 相同）
```

## 16. 一句话总结

**TypeScript = JavaScript + 静态类型**——100% 兼容，编译后**运行时完全相同**，但在编译期、IDE、重构、安全上有巨大提升。**2026 年的事实标准**：新 Web 项目默认 TypeScript。

## 关键洞察

1. **TS 是 JS 超集**——所有 JS 都是合法 TS
2. **运行时相同**——类型擦除，零开销
3. **编译期保护**——bug 提前发现
4. **IDE 智能**——完整类型提示
5. **渐进迁移**——可以逐步加 TS

## 相关笔记

- **核心语法**: [[draft-01-syntax-core]]
- **类型系统**: [[draft-03-type-system]]
- **泛型**: [[draft-04-generics]]
- **对比选型**: [[draft-11-comparison]]
- **最佳实践**: [[draft-09-best-practices]]
- **综合入口**: [[summary]]