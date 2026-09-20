---
title: "TypeScript 类型系统 (Union / Intersection / Narrowing)"
category: synthesis
tags: [typescript, type-system, union-types, intersection-types, type-narrowing, type-guards, discriminated-unions]
sources:
  - "TypeScript Docs - Narrowing"
  - "TypeScript Docs - Literal Types"
  - "Effective TypeScript (Dan Vanderkam) - Chapter 4: Type Design"
summary: "TypeScript 类型系统核心：联合类型、交叉类型、类型缩窄、类型守卫、辨别联合类型、never 穷尽检查"
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

# §03 TypeScript 类型系统（Union / Intersection / Narrowing）

## 1. 联合类型（Union Types）

```typescript
// 基础联合
type Status = "pending" | "success" | "error";
type ID = string | number;

// 函数参数
function printId(id: ID): void {
    console.log(id);
}
printId("abc123");
printId(12345);

// ⚠️ 联合类型只能访问共有成员
type Cat = { name: string; meow(): void };
type Dog = { name: string; bark(): void };
type Pet = Cat | Dog;

function petName(pet: Pet): string {
    return pet.name;          // ✅ name 是共有
    // pet.meow();            // ❌ Pet 上没有 meow
}
```

## 2. 辨别联合类型（Discriminated Unions）

```typescript
// 关键模式：每个类型有共同字面量字段（discriminant）
type Shape =
    | { kind: "circle"; radius: number }
    | { kind: "square"; side: number }
    | { kind: "rectangle"; width: number; height: number };

function area(shape: Shape): number {
    switch (shape.kind) {
        case "circle":
            return Math.PI * shape.radius ** 2;
        case "square":
            return shape.side ** 2;
        case "rectangle":
            return shape.width * shape.height;
        // 编译器确保 exhaustive（增加 kind 时会报错）
    }
}
```

**核心洞察**：辨别联合类型是 TS 模式匹配的事实标准——比继承更灵活，比 enum 更轻量。

## 3. 交叉类型（Intersection Types）

```typescript
// 合并多个类型
type Person = { name: string; age: number };
type Employee = { id: string; salary: number };

type Worker = Person & Employee;
const w: Worker = {
    name: "Mike",
    age: 30,
    id: "E001",
    salary: 100000
};

// 实战：mixin 模式
type WithTimestamps<T> = T & {
    createdAt: Date;
    updatedAt: Date;
};

type UserWithMeta = WithTimestamps<User>;
// = User & { createdAt, updatedAt }

// ⚠️ 交叉类型冲突
type A = { x: number };
type B = { x: string };
type C = A & B;          // { x: never } （number & string = never）
```

## 4. 类型缩窄（Narrowing）

```typescript
// TS 自动根据控制流 narrow 类型
function process(value: string | number) {
    if (typeof value === "string") {
        return value.toUpperCase();  // ✅ value 是 string
    }
    return value.toFixed(2);         // ✅ value 是 number
}

// 多种 narrow 方式
function narrow(x: string | number | null | undefined) {
    if (x === null) return "null";
    if (x === undefined) return "undefined";
    if (typeof x === "string") return x.length;
    return x.toFixed(2);
}
```

**Narrowing 触发器**：
- `typeof` 检查
- `instanceof` 检查
- `in` 操作符
- 字面量相等（`=== "literal"`）
- 真值检查（`if (x)`）
- 赋值（let 类型变窄）
- 控制流分析（return / break）

## 5. 类型守卫函数（Type Guards）

```typescript
// 自定义类型守卫（谓词：value is Type）
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
function handle(value: unknown) {
    if (isString(value)) {
        console.log(value.length);  // value 是 string
    } else if (isUser(value)) {
        console.log(value.id);     // value 是 User
    }
}

// 数组过滤 + 类型守卫
const users: (User | string)[] = [...];
const validUsers: User[] = users.filter(isUser);  // ✅ 类型自动 narrow
```

## 6. 辨别联合 + 穷尽检查（never）

```typescript
type Action =
    | { type: "increment"; payload: number }
    | { type: "decrement"; payload: number }
    | { type: "reset" };

function reducer(state: number, action: Action): number {
    switch (action.type) {
        case "increment": return state + action.payload;
        case "decrement": return state - action.payload;
        case "reset": return 0;
        // 如果将来加 "multiply"，编译器会报错（缺 case）
    }
}

// 显式 exhaustive 检查
function assertNever(value: never): never {
    throw new Error(`Unhandled case: ${JSON.stringify(value)}`);
}

function reducer2(state: number, action: Action): number {
    switch (action.type) {
        case "increment": return state + action.payload;
        case "decrement": return state - action - 1;
        case "reset": return 0;
        default: return assertNever(action);  // Action 增加类型时会报错
    }
}
```

**核心洞察**：**`never` + 辨别联合 = 编译期穷尽保证**——新增 action 类型不写 switch 会编译失败。

## 7. 类型缩窄 vs 类型断言

```typescript
// ❌ 类型断言：绕过类型检查
function bad(value: unknown) {
    const str = value as string;
    str.toUpperCase();  // runtime NPE 风险
}

// ✅ 类型守卫：安全 narrow
function good(value: unknown) {
    if (typeof value === "string") {
        value.toUpperCase();  // 安全
    }
}
```

## 8. 字面量类型 + as const

```typescript
// 字面量推断
const config = {
    url: "https://api.example.com",
    method: "GET"
};
// config.method 推断为 string，不是字面量

// ✅ as const：变成字面量类型
const config2 = {
    url: "https://api.example.com",
    method: "GET"
} as const;
// config2.method 推断为 "GET"（字面量）
// config2.url 推断为 "https://api.example.com"
```

## 9. 实战案例

### 9.1 API 响应类型

```typescript
// 标准模式
type ApiResponse<T> =
    | { status: "success"; data: T }
    | { status: "error"; error: string }
    | { status: "loading" };

async function fetchData(): Promise<ApiResponse<User[]>> {
    try {
        const res = await fetch("/api/users");
        const data = await res.json();
        return { status: "success", data };
    } catch (e) {
        return { status: "error", error: String(e) };
    }
}

// 使用：编译器强制处理所有情况
async function handle() {
    const res = await fetchData();
    switch (res.status) {
        case "success":
            return res.data;          // User[]
        case "error":
            throw new Error(res.error);
        case "loading":
            return [];
        // 缺一不可
    }
}
```

### 9.2 状态机（discriminated union + never）

```typescript
type ConnectionState =
    | { status: "idle" }
    | { status: "connecting"; startedAt: Date }
    | { status: "connected"; sessionId: string; expiresAt: Date }
    | { status: "error"; reason: string };

class Connection {
    private state: ConnectionState = { status: "idle" };

    connect() {
        if (this.state.status !== "idle" && this.state.status !== "error") {
            throw new Error("Already connecting");
        }
        this.state = { status: "connecting", startedAt: new Date() };
    }

    // 编译器确保访问 connected.sessionId 是安全的
    getSession(): string {
        if (this.state.status === "connected") {
            return this.state.sessionId;  // ✅ 类型 narrow 后
        }
        throw new Error("Not connected");
    }
}
```

### 9.3 事件系统

```typescript
type Event =
    | { type: "click"; x: number; y: number }
    | { type: "keypress"; key: string }
    | { type: "resize"; width: number; height: number };

function handle(event: Event) {
    switch (event.type) {
        case "click":
            console.log(`Click at (${event.x}, ${event.y})`);
            break;
        case "keypress":
            console.log(`Key pressed: ${event.key}`);
            break;
        case "resize":
            console.log(`Resized to ${event.width}x${event.height}`);
            break;
        // 穷尽
    }
}
```

## 10. 类型谓词工具

```typescript
// TS 自带类型谓词
function isString(value: unknown): value is string { ... }
function isNumber(value: unknown): value is number { ... }
function isArray<T>(value: unknown): value is T[] { ... }

// 社区工具库：typeguard
// npm i typeguard
// import { is, isString, isNumber } from "typeguard";
```

## 11. 关键设计原则

1. **辨别联合 > enum**——更灵活，无运行时开销
2. **类型守卫 > 类型断言**——编译期保证
3. **`never` 实现穷尽检查**——关键安全网
4. **`as const` 用于字面量保留**——避免 widening
5. **discriminant 用字面量字段**——不要用 boolean

## 12. 反模式

❌ **boolean discriminant**（如 `{ isLoading, data, error }`）
❌ **类型断言 narrow**（不安全）
❌ **联合类型字段不共有**（导致无数访问限制）
❌ **switch 不写 default + never**（失去穷尽保证）
❌ **as const 滥用**（只在需要字面量保留时用）

## 相关笔记

- **核心语法**: [[draft-01-syntax-core]]
- **泛型**: [[draft-04-generics]]
- **类与 OOP**: [[draft-05-classes-oop]]
- **最佳实践**: [[draft-09-best-practices]]
- **综合入口**: [[summary]]