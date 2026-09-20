---
title: "TypeScript 最佳实践"
category: synthesis
tags: [typescript, best-practices, tsconfig, type-design, idioms]
sources:
  - "Effective TypeScript (Dan Vanderkam)"
  - "TypeScript Wiki - Style Guide"
  - "Microsoft TypeScript Blog"
summary: "TypeScript 13 条铁律 + tsconfig 推荐 + 类型设计 + 命名约定 + 类型 vs 接口"
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

# §09 TypeScript 最佳实践

## 1. 13 条铁律

1. **strict 模式必开**——项目第一天启用
2. **避免 any**——用 unknown + 类型守卫
3. **依赖类型推断**——能少写就少写
4. **type-only 导入**——库作者必备
5. **interface vs type**——按需选择
6. **辨别联合 + never**——穷尽保证
7. **public API 显式**——内部推断
8. **readonly 保护不变量**——避免意外修改
9. **as const 保留字面量**——避免 widening
10. **satisfies 检查但保留字面量**——TS 4.9+
11. **const 函数表达式**——更严格的 this
12. **路径别名优于相对路径**——避免深层导入
13. **避免循环依赖**——lazy import 或抽公共

## 2. tsconfig 推荐（项目第一天）

```json
{
    "compilerOptions": {
        "target": "ES2022",
        "module": "ESNext",
        "moduleResolution": "Bundler",
        "strict": true,
        "noUncheckedIndexedAccess": true,
        "noImplicitReturns": true,
        "noFallthroughCasesInSwitch": true,
        "noUnusedLocals": true,
        "noUnusedParameters": true,
        "exactOptionalPropertyTypes": true,
        "useUnknownInCatchVariables": true,
        "esModuleInterop": true,
        "forceConsistentCasingInFileNames": true,
        "skipLibCheck": true,
        "isolatedModules": true,
        "verbatimModuleSyntax": true
    }
}
```

**为什么？**——避免常见 bug（null、未定义返回、switch fall-through、未使用变量）。

## 3. 命名约定

```typescript
// 类 / 类型 / 接口：PascalCase
class UserService {}
interface User {}
type UserId = string;

// 变量 / 函数 / 方法：camelCase
const userName = "Mike";
function getUser(id: string): User {}
class UserRepo {
    findById(id: string): User { /* */ }
}

// 常量：UPPER_SNAKE_CASE
const MAX_CONNECTIONS = 100;
const API_BASE_URL = "https://api.example.com";

// 私有字段：下划线前缀（习惯，非必需）
class Service {
    private _cache: Map<string, unknown> = new Map();
}

// 布尔：用 is/has/can 前缀
const isActive = true;
const hasChildren = false;
const canSend = true;
```

## 4. 接口 vs 类型别名

```typescript
// ✅ 对象类型用 interface（可扩展 + 合并）
interface User {
    id: string;
    name: string;
}

interface User {
    email?: string;     // 自动合并
}

// ✅ 联合/交叉/工具类型用 type
type Status = "pending" | "success" | "error";
type UserWithMeta = User & { metadata: Record<string, string> };
type Awaited<T> = T extends Promise<infer U> ? Awaited<U> : T;
```

**实战选择**：

| 场景 | 用 |
|------|---|
| 对象类型 | `interface` |
| 联合/交叉类型 | `type` |
| 函数类型 | `type`（更简洁）|
| 工具类型（条件/映射） | `type` |

## 5. 类型设计 7 条原则

```typescript
// 1. 优先 type union 而非 enum
type Status = "active" | "inactive";     // ✅
enum Status2 { Active, Inactive };        // ❌（运行时占用）

// 2. 字面量类型 vs 宽类型
type ID = string;                         // ❌ 太宽
type ID2 = `user_${string}`;               // ✅ 模式

// 3. 辨别联合 > boolean flag
type AsyncState<T> =
    | { status: "loading" }
    | { status: "success"; data: T }
    | { status: "error"; error: string };   // ✅
// ❌ { loading: true, data: T, error: string }

// 4. readonly + immutable
interface User {
    readonly id: string;
    readonly name: string;
}

// 5. 命名空间 vs 嵌套对象
type Api = {
    user: { create(): Promise<void>; };
    post: { create(): Promise<void>; };
};
// 优于：namespace Api { namespace User { ... } }

// 6. 工具类型组合
type UserUpdate = Partial<Pick<User, "name" | "email">>;

// 7. satisfies 检查形状
const config = {
    port: 8080,
    host: "localhost"
} satisfies Config;                  // ✅ 检查但不改变字面量
```

## 6. 函数签名设计

```typescript
// ✅ 必填参数在前，可选/默认在后
function fetchUser(
    id: string,
    options: { includeProfile?: boolean; timeout?: number } = {}
): Promise<User> {
    // ...
}

// ❌ 反模式：可选参数在前
function fetchUser(
    options?: { includeProfile?: boolean },
    id: string         // 必填在可选后（不允许）
) {}

// ✅ 用对象代替多参数（参数 > 3）
interface FetchOptions {
    url: string;
    method?: "GET" | "POST";
    headers?: Record<string, string>;
    timeout?: number;
    retries?: number;
}
function fetch(options: FetchOptions): Promise<Response> {}

// ✅ 返回值尽量具体（不用 Promise<any>）
async function load(): Promise<User> { /* ... */ }

// ✅ never 标记永不返回的函数
function throwError(msg: string): never {
    throw new Error(msg);
}
```

## 7. 异步代码

```typescript
// ✅ async/await 优于 Promise 链
async function loadUser(id: string): Promise<User> {
    try {
        const res = await fetch(`/api/users/${id}`);
        if (!res.ok) throw new Error(`HTTP ${res.status}`);
        return await res.json();
    } catch (e) {
        console.error("Failed", e);
        throw e;
    }
}

// ✅ 并行用 Promise.all
const [user, posts] = await Promise.all([
    fetchUser("1"),
    fetchPosts("1")
]);

// ✅ 不要 await 在 for 循环（串行）
// ❌ for (const id of ids) await fetch(id);

// ✅ 并行
await Promise.all(ids.map(id => fetch(id)));
```

## 8. 错误处理

```typescript
// ✅ 类型化错误
class HttpError extends Error {
    constructor(
        message: string,
        public status: number,
        public response?: Response
    ) {
        super(message);
        this.name = "HttpError";
    }
}

// ✅ 错误联合类型
type ApiResult<T> =
    | { success: true; data: T }
    | { success: false; error: HttpError };

// 使用
const result = await apiCall();
if (result.success) {
    console.log(result.data);
} else {
    console.error(result.error.status);
}
```

## 9. 测试

```typescript
// vitest（推荐，Vite 原生）
import { describe, it, expect } from "vitest";

describe("UserService", () => {
    it("should fetch user by id", async () => {
        const user = await userService.findById("1");
        expect(user.name).toBe("Mike");
    });
});

// Jest（老牌）
describe("UserService", () => {
    test("fetchUser returns user", async () => {
        const user = await fetchUser("1");
        expect(user.id).toBe("1");
    });
});
```

## 10. 命名空间 vs 模块

```typescript
// ⚠️ 命名空间：仅用于全局类型声明
declare global {
    namespace NodeJS {
        interface ProcessEnv {
            NODE_ENV: "development" | "production";
        }
    }
}

// ✅ 模块：现代项目主用
// src/utils/index.ts
export { formatDate } from "./date";
export { parseJson } from "./json";
```

## 11. 项目结构（推荐）

```
src/
├── index.ts              入口
├── config/               配置（env / db）
│   └── env.ts
├── types/                全局类型
│   ├── index.ts          barrel
│   └── api.ts
├── modules/              业务模块（feature）
│   ├── user/
│   │   ├── user-service.ts
│   │   ├── user-repo.ts
│   │   └── types.ts
│   └── order/
├── lib/                  通用工具（不依赖业务）
│   ├── http.ts
│   ├── logger.ts
│   └── date.ts
├── middleware/           Express/Fastify 中间件
├── errors/               自定义错误
└── tests/                测试
```

## 12. 文档（TSDoc）

```typescript
/**
 * 获取用户信息
 *
 * @param id - 用户 ID（必填）
 * @param options - 查询选项
 * @returns 用户对象 Promise
 * @throws {UserNotFoundError} 用户不存在
 *
 * @example
 * ```typescript
 * const user = await fetchUser("123", { includeProfile: true });
 * console.log(user.name);
 * ```
 */
async function fetchUser(id: string, options?: FetchOptions): Promise<User> {
    // ...
}
```

## 13. 实战检查清单

### 13.1 新项目

- [ ] `strict: true` + 额外严格选项
- [ ] ESLint + Prettier 配置
- [ ] 路径别名（`@/*`）
- [ ] Husky + lint-staged（commit 前检查）
- [ ] CI 跑 `tsc --noEmit` + `eslint`

### 13.2 Code Review

- [ ] 任何 `any` 都需注释说明
- [ ] 公共 API 显式类型
- [ ] 函数参数 ≤ 3 个（多则用对象）
- [ ] 联合类型用辨别联合
- [ ] switch 必用 `never` 兜底
- [ ] 类型断言 vs 类型守卫选对
- [ ] 错误处理有类型（自定义类 / 联合）

## 14. 反模式（Top 10）

1. ❌ `any` 滥用——用 unknown + 类型守卫
2. ❌ `!` 非空断言滥用——runtime NPE
3. ❌ 类型断言绕检查（`as` 乱用）
4. ❌ 联合类型不共享字段——无数访问
5. ❌ 不写 default + never——失去穷尽
6. ❌ 函数参数 > 3 个（应用对象）
7. ❌ 局部变量到处显式注解（冗余）
8. ❌ enum 滥用（用字面量联合）
9. ❌ 默认导出滥用（不利于 tree-shake）
10. ❌ 循环依赖（用 lazy import）

## 相关笔记

- **核心语法**: [[draft-01-syntax-core]]
- **类型系统**: [[draft-03-type-system]]
- **泛型**: [[draft-04-generics]]
- **已知坑**: [[draft-10-known-issues]]
- **综合入口**: [[summary]]