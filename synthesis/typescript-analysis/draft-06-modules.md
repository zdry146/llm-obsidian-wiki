---
title: "TypeScript 模块系统"
category: synthesis
tags: [typescript, modules, esm, commonjs, imports, exports, namespaces]
sources:
  - "TypeScript Docs - Modules"
  - "MDN - ES Modules"
  - "TypeScript Docs - Namespaces"
summary: "TypeScript 模块：ESM vs CommonJS、import/export 各种形式、type-only、namespace、动态 import"
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

# §06 TypeScript 模块系统

## 1. ESM vs CommonJS

| 维度 | ESM（推荐）| CommonJS（Node.js 旧） |
|------|-----------|---------------------|
| **语法** | `import`/`export` | `require`/`module.exports` |
| **加载** | 静态分析 + 异步 | 同步 |
| **tree-shaking** | ✅ | ❌ |
| **浏览器** | ✅ 原生支持 | ❌ 需要打包 |
| **Node.js** | ✅ 14+ 支持 | ✅ 默认 |
| **导出形式** | 命名 + 默认 | 任意 |

**实战**：现代项目全用 ESM。

## 2. export 6 种形式

```typescript
// 1. 命名导出
export const PI = 3.14;
export function add(a: number, b: number): number { return a + b; }

// 2. 默认导出（每个文件最多 1 个）
export default class UserService { }

// 3. 一起
export const name = "Mike";
export default { name };

// 4. 重新导出（barrel）
export { add, subtract } from "./math";
export * from "./utils";
export * as utils from "./utils";      // 命名空间导出

// 5. 类型导出
export type { User, Post } from "./types";

// 6. 批量导出
const a = 1;
const b = 2;
const c = 3;
export { a, b, c };
```

## 3. import 9 种形式

```typescript
// 1. 默认导入
import UserService from "./user-service";

// 2. 命名导入
import { User, Post } from "./types";

// 3. 别名
import { User as UserModel } from "./types";

// 4. 命名空间导入
import * as utils from "./utils";
utils.formatDate();

// 5. 副作用导入（执行模块代码）
import "./polyfill";

// 6. 类型导入（不引入运行时）
import type { User } from "./types";

// 7. 混合
import UserService, { type Config } from "./user-service";

// 8. 动态导入
const module = await import("./heavy-module");

// 9. CommonJS 互操作
import utils = require("./utils");          // ts 特有
const utils = require("./utils");          // 通用
```

## 4. type-only 导入/导出（关键）

```typescript
// 普通导入：运行时 + 类型
import { User } from "./types";
// 编译后：const types_1 = require("./types")  // ← 引入运行时

// type-only 导入：仅类型，编译擦除
import type { User } from "./types";
// 编译后：什么都没有 ← 完全擦除

// 内联语法（推荐）
import { type User, value } from "./types";

// 实战：
// - 库作者：所有类型导出用 `export type`
// - 应用开发者：导入类型用 `import type`（特别在 monorepo）
```

**为什么用 type-only**：
1. **更小的 bundle**：类型不引入运行时
2. **更清晰的意图**：明确这是类型
3. **避免循环依赖**：类型可循环，实际不行
4. **更好的 tree-shaking**

## 5. barrel 文件（聚合导出）

```typescript
// src/components/index.ts
export { Button } from "./Button";
export { Input } from "./Input";
export { Modal } from "./Modal";
export type { ButtonProps } from "./Button";
export type { InputProps } from "./Input";

// 使用：单次导入多个
import { Button, Input, type ButtonProps } from "./components";
```

**实战**：每个目录加 `index.ts` 聚合导出，避免深层路径（`../../../components/Button`）。

## 6. 动态 import（懒加载）

```typescript
// 基础用法
async function loadHeavyModule() {
    const module = await import("./heavy-module");
    return module.doSomething();
}

// 实战：路由懒加载（React）
const HomePage = lazy(() => import("./pages/HomePage"));
const AboutPage = lazy(() => import("./pages/AboutPage"));

// 实战：按需加载 polyfill
if (window.Intl === undefined) {
    await import("intl-polyfill");
}

// 实战：条件加载（服务端 vs 客户端）
const api = typeof window === "undefined"
    ? (await import("./server-api")).ServerAPI
    : (await import("./browser-api")).BrowserAPI;
```

## 7. 命名空间（namespace，传统）

```typescript
// ⚠️ 老式写法，现代项目用 ESM
namespace Utils {
    export function format(date: Date): string {
        return date.toISOString();
    }
    
    export namespace Validation {
        export function isEmail(s: string): boolean {
            return /^.+@.+\..+$/.test(s);
        }
    }
}

// 使用
Utils.format(new Date());
Utils.Validation.isEmail("mike@example.com");
```

**实战**：命名空间用于：
- 全局类型声明（`declare global { ... }`）
- 老代码兼容
- 内部模块组织（少见）

## 8. 模块解析策略

`tsconfig.json` 配置：

```json
{
    "compilerOptions": {
        "moduleResolution": "node",          // 或 "bundler" / "node16" / "nodenext"
        "module": "ESNext",                  // 或 "CommonJS"
        "baseUrl": "./src",
        "paths": {
            "@/*": ["*"],                    // 路径别名
            "@components/*": ["components/*"]
        }
    }
}
```

**moduleResolution 选项**（TS 5.x）：
- `node`：经典 Node.js（Node 10-）
- `node16` / `nodenext`：现代 Node.js（Node 16+）
- `bundler`：Webpack/Vite/esbuild 友好（推荐）
- `classic`：老式（不推荐）

## 9. 实战：路径别名

```json
// tsconfig.json
{
    "compilerOptions": {
        "baseUrl": ".",
        "paths": {
            "@/*": ["src/*"],
            "@components/*": ["src/components/*"],
            "@utils/*": ["src/utils/*"]
        }
    }
}
```

```typescript
// ✅ 推荐：路径别名
import { Button } from "@components/Button";
import { format } from "@utils/date";

// ❌ 反模式：相对路径地狱
import { Button } from "../../../components/Button";
```

## 10. 模块声明（.d.ts）

```typescript
// types/my-lib.d.ts
declare module "my-lib" {
    export function doSomething(): void;
    export class MyClass {
        constructor(value: string);
        getValue(): string;
    }
}

// 使用（类型 + 实现由 JS 提供）
import { MyClass } from "my-lib";
```

**实战**：
- DefinitelyTyped（@types/*）就是这种 .d.ts
- 没有官方类型的库自己写 .d.ts
- `declare module` 是 ambient declaration

## 11. 默认导出 vs 命名导出

```typescript
// 默认导出：1 个主要类/函数
// user-service.ts
export default class UserService { }

// 导入（可任意命名）
import UserService from "./user-service";
import US from "./user-service";  // ✅ 重命名

// 命名导出：多个相关项
// utils.ts
export function format() { }
export function parse() { }

// 导入（必须用原名）
import { format, parse } from "./utils";

// 实战选择：
// - 一个文件 1 个主类 → default
// - 一个文件多个工具 → named
// - 库作者推荐：只 named export（更好 tree-shake）
```

## 12. 实战案例

### 12.1 Node.js 库导出策略

```typescript
// 单入口
// src/index.ts
export * from "./user";
export * from "./post";
export * from "./comment";
// 用户：import { User, Post } from "my-lib"
```

### 12.2 避免循环依赖

```typescript
// ❌ 循环依赖
// a.ts: import { b } from "./b"
// b.ts: import { a } from "./a"  // 循环！

// ✅ 解决 1：lazy import
// b.ts:
async function getA() {
    const { a } = await import("./a");  // 运行时加载
    return a;
}

// ✅ 解决 2：抽出共享
// common.ts: 共享类型/常量
// a.ts: import { Common } from "./common"
// b.ts: import { Common } from "./common"
```

### 12.3 类型声明合并

```typescript
// types/express.d.ts
declare global {
    namespace Express {
        interface Request {
            user?: User;
        }
    }
}

// 使用：express.Request 现在有 user 字段
```

## 13. 与 Kotlin/JS 对照

| 维度 | TypeScript ESM | Kotlin |
|------|-----------|--------|
| 模块声明 | `export` | `internal`/`public` |
| 路径 | 相对 + 别名 | 包名 + 相对 |
| 树摇 | ✅ | ✅ |
| 懒加载 | `import()` | `by lazy` |
| 可见性 | `export`/`default` | `internal`/`public` |

## 14. 关键设计原则

1. **ESM 优先**——现代项目标配
2. **barrel 文件**——简化导入路径
3. **type-only 导入**——库作者必备
4. **动态 import**——按需加载
5. **路径别名**——避免相对路径地狱
6. **避免循环依赖**——用 lazy import 或抽公共

## 15. 反模式

❌ **默认导出滥用**——不利于 tree-shaking
❌ **相对路径深**——> 3 层用别名
❌ **运行时导入类型**——编译不擦除
❌ **循环依赖**——用 dynamic import 解决
❌ **namespace 滥用**——现代 ESM 优先
❌ **CommonJS 新项目**——失去 tree-shaking

## 相关笔记

- **工具链**: [[draft-07-tooling]]
- **构建与打包**: [[draft-08-build-bundlers]]
- **最佳实践**: [[draft-09-best-practices]]
- **综合入口**: [[summary]]