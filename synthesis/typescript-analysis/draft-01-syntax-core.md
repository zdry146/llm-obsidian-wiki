---
title: "TypeScript 核心语法与类型基础"
category: synthesis
tags: [typescript, syntax, type-annotations, any, unknown, literal-types, type-inference]
sources:
  - "TypeScript Docs - Basic Types"
  - "TypeScript Handbook - Everyday Types"
  - "Effective TypeScript (Dan Vanderkam)"
summary: "TypeScript 类型注解、基础类型、字面量类型、类型推断、any vs unknown 区别"
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

# §01 TypeScript 核心语法与类型基础

## 1. 类型注解（Type Annotations）

```typescript
// 基本注解
let name: string = "Mike";
let age: number = 30;
let active: boolean = true;

// 函数参数 + 返回类型
function greet(name: string): string {
    return `Hello, ${name}!`;
}

// 变量类型推断（不写也 OK）
let inferred = "Hello";  // 推断为 string
// inferred = 123;       // ❌ 编译错误
```

## 2. 基础类型（Primitive Types）

```typescript
let str: string = "hello";
let num: number = 42;
let bool: boolean = true;
let nul: null = null;
let undef: undefined = undefined;
let sym: symbol = Symbol("id");
let big: bigint = 100n;            // ES2020

// 特殊类型
let anything: any = "anything";     // ⚠️ 绕过类型检查
let unknownVar: unknown = ...;      // ✅ 安全的 top type
let neverVar: never = (() => { throw new Error() })();  // 永不返回
let voidVar: void = undefined;      // 函数无返回值
```

## 3. 数组与元组

```typescript
// 数组
let numbers: number[] = [1, 2, 3];
let strings: Array<string> = ["a", "b"];     // 泛型写法

// 元组（固定长度 + 不同类型）
let tuple: [string, number] = ["Mike", 30];
tuple[0];  // "Mike"
tuple[1];  // 30

// 命名字段（元组更可读，TS 4.0+）
let user: [name: string, age: number] = ["Mike", 30];

// 可变元组（TS 4.0+）
let flexible: [string, ...number[]] = ["count", 1, 2, 3];
```

## 4. 对象类型

```typescript
// 对象字面量类型
let user: { name: string; age: number } = { name: "Mike", age: 30 };

// 可选属性
let config: { host: string; port?: number } = { host: "localhost" };

// 只读属性
let point: { readonly x: number; readonly y: number } = { x: 1, y: 2 };

// 索引签名
let dict: { [key: string]: number } = { age: 30, count: 5 };
```

## 5. 字面量类型（Literal Types）

```typescript
// 字符串字面量
let status: "pending" | "success" | "error" = "pending";
// status = "unknown";  // ❌ 只能是这三个值

// 数字字面量
let httpStatus: 200 | 404 | 500 = 200;

// 布尔字面量
let enabled: true = true;

// 字面量推断
const constant = "hello";  // 推断为 "hello"（非 string）
// constant = "world";    // ❌ 必须是 "hello"
```

## 6. 类型推断 vs 类型注解

```typescript
// ✅ 推荐：依赖类型推断（少写代码）
let count = 100;          // 推断为 number
let items = [1, 2, 3];    // 推断为 number[]
let user = { name: "Mike", age: 30 };  // 推断为 { name: string; age: number }

// ⚠️ 必要时显式注解
function add(a: number, b: number): number {  // 参数返回显式
    return a + b;
}

// 复杂场景显式注解
const map: Map<string, User> = new Map();
```

## 7. any vs unknown（关键区别）

```typescript
// ❌ any：绕过所有类型检查
let x: any = "hello";
x.foo();            // 编译通过（运行时可能 crash）
x.bar.baz[0];       // 编译通过

// ✅ unknown：安全的 top type（必须先 narrow）
let y: unknown = "hello";
// y.toUpperCase();  // ❌ 编译错误：unknown 不能直接访问
if (typeof y === "string") {
    y.toUpperCase();  // ✅ 类型缩窄后 OK
}
```

**核心区别**：
| `any` | `unknown` |
|------|-----------|
| ⚠️ 关闭类型检查 | ✅ 保留类型检查 |
| 编译通过任何操作 | 必须 narrow 后才能用 |
| TS 4+ 警告 | TS 4+ 推荐 |
| 遗留代码常用 | 新代码首选 |

## 8. void vs never

```typescript
// void：函数无返回值
function log(msg: string): void {
    console.log(msg);
    // 无 return
}

// never：函数永不返回（抛异常或死循环）
function throwError(msg: string): never {
    throw new Error(msg);
}

function infiniteLoop(): never {
    while (true) { /* ... */ }
}

// 实战：exhaustive check
function assertNever(value: never): never {
    throw new Error(`Unhandled case: ${value}`);
}

type Shape = "circle" | "square";
function area(shape: Shape): number {
    switch (shape) {
        case "circle": return Math.PI;
        case "square": return 1;
        default: return assertNever(shape);  // 编译期穷尽检查
    }
}
```

## 9. 类型断言（Type Assertion）

```typescript
// as 语法（推荐）
let value: unknown = "hello";
let len = (value as string).length;

// 双重断言（避免类型错误）
let s = (value as unknown) as string;

// 非空断言（避免 null 检查）
function process(value: string | null) {
    // 编译错误：value 可能为 null
    // console.log(value.length);

    // ✅ 非空断言（你确定不为 null）
    console.log(value!.length);
}

// ❌ 不推荐：滥用类型断言会绕过类型安全
let user = {} as User;  // 即使 user 没有 user 的字段
```

## 10. 类型注解最佳实践

```typescript
// ✅ 函数签名：参数 + 返回类型显式
function fetchUser(id: string): Promise<User> { ... }

// ✅ 公共 API 显式
export interface User { id: string; name: string }

// ✅ 复杂类型显式
const config: HttpConfig = {
    timeout: 5000,
    retries: 3
};

// ✅ 局部变量 + 简单类型：依赖推断
let count = 0;
let name = "Mike";

// ❌ 不推荐：到处显式（冗余）
let count: number = 0;
let name: string = "Mike";
```

## 11. 类型别名 vs 接口

```typescript
// 类型别名（type）：灵活（联合/交叉/原始）
type Status = "pending" | "success" | "error";
type User = {
    id: string;
    name: string;
};

// 接口（interface）：可扩展、合并
interface User {
    id: string;
    name: string;
}

interface User {           // ✅ 自动合并
    email: string;
}

// 实战选择：
// - 对象类型用 interface（更灵活）
// - 联合/工具类型用 type（更强大）
```

## 12. 数组 vs Set vs Map 选择

```typescript
// 数组：有序 + 可重复
let list: number[] = [1, 2, 2, 3];

// Set：唯一 + 快速查找
let set: Set<number> = new Set([1, 2, 3]);

// Map：键值对（任意类型 key）
let map: Map<string, User> = new Map();

// 选型：
// - 顺序 + 重复 → Array
// - 唯一 + 查找 → Set
// - 键值 → Map
```

## 13. 函数基础（预览）

```typescript
// 函数声明
function add(a: number, b: number): number {
    return a + b;
}

// 函数表达式
const add2 = (a: number, b: number): number => a + b;

// 类型别名
type AddFn = (a: number, b: number) => number;
const add3: AddFn = (a, b) => a + b;

// 可选参数
function greet(name: string, greeting?: string): string {
    return `${greeting ?? "Hello"}, ${name}!`;
}

// 默认参数
function greet2(name: string, greeting: string = "Hello"): string {
    return `${greeting}, ${name}!`;
}

// 剩余参数
function sum(...numbers: number[]): number {
    return numbers.reduce((a, b) => a + b, 0);
}
```

详见 [[draft-02-functions]]。

## 14. 关键设计原则

1. **依赖类型推断**——能少写就少写
2. **避免 any**——用 unknown + 类型守卫
3. **公共 API 显式**——内部推断
4. **接口优先**——对象类型
5. **类型别名辅助**——联合/工具类型

## 15. 反模式

❌ **到处显式注解**——冗余
❌ **滥用 any**——关闭类型检查
❌ **类型断言滥用**——绕过类型安全
❌ **! 非空断言滥用**——运行时 NPE 风险
❌ **never 误用**——必须永不返回才用

## 相关笔记

- **类型系统**: [[draft-03-type-system]]
- **泛型**: [[draft-04-generics]]
- **函数**: [[draft-02-functions]]
- **最佳实践**: [[draft-09-best-practices]]
- **综合入口**: [[summary]]