---
title: "TypeScript 泛型与高级类型"
category: synthesis
tags: [typescript, generics, conditional-types, mapped-types, infer-keyword, utility-types]
sources:
  - "TypeScript Docs - Generics"
  - "TypeScript Docs - Conditional Types"
  - "TypeScript Docs - Mapped Types"
  - "Effective TypeScript - Chapter 5: Generic"
summary: "TypeScript 泛型、约束、条件类型、映射类型、infer 关键字、Template Literal Types、内置工具类型"
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

# §04 TypeScript 泛型与高级类型

## 1. 泛型基础

```typescript
// 泛型函数
function identity<T>(value: T): T {
    return value;
}
identity<string>("hello");    // 显式类型
identity(42);                // 推断为 number

// 泛型接口
interface ApiResponse<T> {
    data: T;
    status: number;
    message: string;
}

const res: ApiResponse<User> = {
    data: { id: "1", name: "Mike" },
    status: 200,
    message: "OK"
};

// 泛型类
class Container<T> {
    constructor(public value: T) {}
    map<U>(fn: (value: T) => U): Container<U> {
        return new Container(fn(this.value));
    }
}

const c = new Container(42).map(x => x.toString());
c.value;  // "42"
```

## 2. 泛型约束

```typescript
// extends 约束
interface HasLength { length: number; }

function logLength<T extends HasLength>(value: T): number {
    return value.length;
}
logLength("hello");      // 5
logLength([1, 2, 3]);    // 3
logLength({ length: 10 }); // 10
// logLength(42);          // ❌ number 没有 length

// 多约束
function process<T extends string | number>(value: T): T {
    return value;
}

// 嵌套泛型约束
function getProperty<T, K extends keyof T>(obj: T, key: K): T[K] {
    return obj[key];
}
const user = { name: "Mike", age: 30 };
getProperty(user, "name");  // "Mike"
getProperty(user, "age");   // 30
// getProperty(user, "foo");  // ❌ keyof User 没有 "foo"
```

## 3. 条件类型（Conditional Types）

```typescript
// 三元类型
type IsString<T> = T extends string ? true : false;
type A = IsString<string>;  // true
type B = IsString<number>;  // false

// 实用例子：提取函数返回类型
type MyReturnType<T> = T extends (...args: any[]) => infer R ? R : never;
type C = MyReturnType<() => string>;  // string

// 提取 Promise unwrap
type Awaited<T> = T extends Promise<infer U> ? Awaited<U> : T;
type D = Awaited<Promise<string>>;     // string
type E = Awaited<Promise<Promise<number>>>;  // number（递归）
```

## 4. infer 关键字

```typescript
// 从复杂类型中提取子类型
type ArrayElement<T> = T extends (infer U)[] ? U : never;
type Str = ArrayElement<string[]>;  // string
type Num = ArrayElement<number[]>;  // number

// 实战：提取函数参数
type MyParameters<T> = T extends (...args: infer P) => any ? P : never;
type F = MyParameters<(a: string, b: number) => void>;  // [string, number]

// 实战：提取 Promise value
type PromiseValue<T> = T extends Promise<infer V> ? V : T;
type G = PromiseValue<Promise<User>>;  // User
```

## 5. 映射类型（Mapped Types）

```typescript
// 把所有属性变可选
type Partial<T> = {
    [K in keyof T]?: T[K];
};

interface User { name: string; age: number; }
type PartialUser = Partial<User>;
// = { name?: string; age?: number; }

// 把所有属性变只读
type Readonly<T> = {
    readonly [K in keyof T]: T[K];
};

// 把所有属性变 nullable
type Nullable<T> = {
    [K in keyof T]: T[K] | null;
};

// 修改 key（重命名）
type Getters<T> = {
    [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K];
};

interface User { name: string; age: number; }
type UserGetters = Getters<User>;
// = { getName: () => string; getAge: () => number; }
```

## 6. Template Literal Types（TS 4.1+）

```typescript
// 字符串模板类型
type EventName = `on${Capitalize<string>}`;
type E = EventName;  // "onClick" | "onHover" | ...（联合所有）

// 实战：API 路径类型
type ApiPath = `/api/${string}`;
const path: ApiPath = "/api/users";  // ✅
const bad: ApiPath = "/users";       // ❌

// 实战：CSS 属性
type CSSProperty = `${string}-${"top" | "bottom" | "left" | "right"}`;
const prop: CSSProperty = "margin-top";  // ✅

// 实战：事件类型
type HTMLEvent = `on${"click" | "focus" | "blur"}`;
// = "onclick" | "onfocus" | "onblur"
```

## 7. 内置工具类型

```typescript
interface User {
    id: string;
    name: string;
    email?: string;
    age: number;
}

// 常用工具
type U1 = Partial<User>;        // 所有属性可选
type U2 = Required<User>;      // 所有属性必填（去掉 ?）
type U3 = Readonly<User>;      // 所有属性只读
type U4 = Pick<User, "id" | "name">;     // 挑选字段
type U5 = Omit<User, "age">;            // 排除字段
type U6 = Record<"admin" | "user", User>; // 构造对象类型
type U7 = Pick<User, keyof User>;        // = User（全部）
type U8 = Exclude<"a" | "b" | "c", "a">; // "b" | "c"
type U9 = Extract<"a" | "b" | "c", "a" | "x">;  // "a"
type U10 = NonNullable<string | null>;   // string
type U11 = ReturnType<() => User>;      // User
type U12 = Parameters<(id: string) => void>;  // [string]
type U13 = InstanceType<typeof User>;   // User（类实例类型）
```

## 8. 分布式条件类型

```typescript
// 联合类型分发
type ToArray<T> = T extends unknown ? T[] : never;
type X = ToArray<string | number>;  // string[] | number[]（分发）
// 不是 (string | number)[]

// 过滤联合类型
type Filter<T, U> = T extends U ? T : never;
type StrOrNum = Filter<string | number | boolean, string | number>;
// = string | number
```

## 9. 实战案例

### 9.1 类型安全的 API 客户端

```typescript
// 路径 + 方法 + body 类型严格匹配
type ApiPath = "/users" | "/posts" | "/comments";
type HttpMethod = "GET" | "POST" | "PUT" | "DELETE";

interface PathParams {
    "/users": never;
    "/posts": { userId: string };
    "/comments": { postId: string };
}

interface PathResponse {
    "/users": User[];
    "/posts": Post[];
    "/comments": Comment[];
}

async function apiCall<P extends ApiPath, M extends HttpMethod>(
    path: P,
    method: M,
    params?: PathParams[P],
    body?: M extends "POST" | "PUT" ? any : never
): Promise<PathResponse[P]> {
    // 实现...
    return fetch(/* ... */) as any;
}

// 类型安全
const users = await apiCall("/users", "GET");          // User[]
const post = await apiCall("/posts", "GET", { userId: "123" });  // Post[]
// const bad = apiCall("/users", "POST", { postId: "..." });  // ❌ PathParams[/users] 是 never
```

### 9.2 深 Readonly

```typescript
type DeepReadonly<T> = {
    readonly [K in keyof T]: T[K] extends object ? DeepReadonly<T[K]> : T[K];
};

interface Config {
    server: { host: string; port: number };
    db: { url: string };
}

const config: DeepReadonly<Config> = {
    server: { host: "localhost", port: 8080 },
    db: { url: "mongodb://localhost" }
};
// config.server.host = "new";  // ❌ 编译错误
```

### 9.3 Pick by Type

```typescript
type FilterProperties<T, V> = {
    [K in keyof T as T[K] extends V ? K : never]: T[K];
};

interface User {
    id: string;
    name: string;
    age: number;
    createdAt: Date;
}

type StringFields = FilterProperties<User, string>;
// = { id: string; name: string; }
```

## 10. const type parameters（TS 5.0+）

```typescript
// 不推断为宽类型，而是字面量
function makeTuple<const T extends readonly string[]>(values: T): T {
    return values;
}
const t1 = makeTuple(["a", "b", "c"]);  // readonly ["a", "b", "c"]

// 之前会推断为 string[]
const t2 = makeTuple<string[]>(["a", "b"]);  // string[]
```

## 11. satisfies 操作符（TS 4.9+）

```typescript
// 检查类型但不改变推断
type Config = { port: number; host: string };

// 之前：显式注解 → 推断变窄
const c1: Config = { port: 8080, host: "localhost" };
// c1.port 类型 = number（宽了）

// 现在：satisfies → 推断保持字面量
const c2 = { port: 8080, host: "localhost" } satisfies Config;
// c2.port 类型 = 8080（保留字面量）
// 且保证是合法 Config
```

## 12. 关键设计原则

1. **泛型优于 any/unknown**——保留类型
2. **约束 T extends X**——缩小范围
3. **`infer` 提取子类型**——不用重复定义
4. **映射类型变换**——批量修改属性
5. **辨别联合 + never**——穷尽保证
6. **satisfies 检查但保留字面量**——TS 4.9+

## 13. 反模式

❌ **泛型约束过宽**（`extends any`）
❌ **过度使用 infer**——简单场景直接写
❌ **嵌套条件类型过深**——可读性差
❌ **循环约束**（A extends B, B extends A）
❌ **类型操作太多**——可读性下降

## 14. 内置工具速查

```typescript
// 修改类型
Partial<T>      // 可选
Required<T>    // 必填
Readonly<T>     // 只读
Mutable<T>      // 去掉 readonly

// 字段筛选
Pick<T, K>     // 选
Omit<T, K>     // 排除
Record<K, V>   // 构造对象

// 联合类型
Exclude<U, T>  // 排除
Extract<U, T>  // 提取
NonNullable<T>  // 去 null

// 函数相关
ReturnType<F>    // 返回类型
Parameters<F>   // 参数类型
InstanceType<C>  // 类实例类型
Awaited<T>       // Promise unwrap（递归）

// 字符串（TS 4.1+）
Uppercase<S>
Lowercase<S>
Capitalize<S>
Uncapitalize<S>
```

## 相关笔记

- **核心语法**: [[draft-01-syntax-core]]
- **类型系统**: [[draft-03-type-system]]
- **类与 OOP**: [[draft-05-classes-oop]]
- **最佳实践**: [[draft-09-best-practices]]
- **综合入口**: [[summary]]