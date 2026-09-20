---
title: "TypeScript 已知坑与陷阱"
category: synthesis
tags: [typescript, pitfalls, any, type-assertion, decorators, gotchas]
sources:
  - "TypeScript GitHub Issues (Microsoft/TypeScript)"
  - "Effective TypeScript (Dan Vanderkam) - Common Pitfalls"
  - "作者实战经验"
summary: "TypeScript 10 大陷阱：any 滥用、类型断言、enum 性能、装饰器实验性、严格模式、循环依赖"
provenance:
  extracted: 0.85
    inferred: 0.12
    ambiguous: 0.03
base_confidence: 0.85
lifecycle: draft
lifecycle_changed: 2026-09-20
created: 2026-09-20
updated: 2026-09-20
---

# §10 TypeScript 已知坑与陷阱

## 1. 【最常见】any 滥用 → 失去类型安全

```typescript
// ❌ 危险：完全失去类型保护
function process(data: any) {
    return data.user.name.toUpperCase();  // 运行时崩
}

// ✅ 安全：unknown + 类型守卫
function process(data: unknown) {
    if (isUser(data)) {
        return data.user.name.toUpperCase();
    }
}

function isUser(obj: unknown): obj is { user: { name: string } } {
    return (
        typeof obj === "object" &&
        obj !== null &&
        "user" in obj &&
        typeof (obj as any).user === "object"
    );
}
```

**任何 any 都需注释**：`// eslint-disable-next-line @typescript-eslint/no-explicit-any`

## 2. 【致命】类型断言 vs 类型守卫

```typescript
// ❌ 类型断言：编译通过，运行时崩
function bad(value: unknown) {
    const str = value as string;
    return str.toUpperCase();   // runtime NPE（value 可能 undefined）
}

// ✅ 类型守卫：编译 + 运行时双重安全
function good(value: unknown) {
    if (typeof value === "string") {
        return value.toUpperCase();
    }
}

// ✅ 双重断言（最后手段）
const data = JSON.parse(input) as unknown as User;
```

## 3. 【坑】! 非空断言滥用

```typescript
// ❌ 危险：你确定 user 不为 null？
function process(user: User | null) {
    console.log(user!.name);  // runtime NPE if null
}

// ✅ 显式检查
function process(user: User | null) {
    if (user !== null) {
        console.log(user.name);
    }
}
```

**实战**：允许 `!` 仅用于已检查但 TS 推断不到的场景（如刚 `findById` 后立刻用）。

## 4. 【坑】enum vs 字面量联合

```typescript
// ❌ enum：编译生成 IIFE（运行时占用）
enum Status { Active, Inactive }
console.log(Status.Active);  // 0（编译擦除！TS 编译成对象）
// 输出文件包含：var Status; (function (Status) { Status[Status["Active"] = 0] = "Active"; })(Status || (Status = {}));

// ✅ 字面量联合：纯类型
type Status = "active" | "inactive";
const status: Status = "active";
// 编译后：完全擦除，零运行时开销
```

**实战**：新项目用字面量联合，老 enum 兼容时保留。

## 5. 【坑】interface 声明合并（意外行为）

```typescript
// 同一接口多次声明会合并
interface User {
    id: string;
}
interface User {
    name: string;
}
// User = { id: string; name: string }

// ⚠️ 意外场景：第三方库扩展
// 别人可能添加字段，破坏你的代码
```

## 6. 【坑】TypeScript 不做运行时检查

```typescript
// 编译期有类型，运行时没有
interface User { name: string; }

// JSON.parse 不知道是 User
const data: User = JSON.parse(input);   // ❌ runtime data 可能是 {}

// ✅ 用 zod / yup / valibot 做运行时验证
import { z } from "zod";

const UserSchema = z.object({
    name: z.string(),
    age: z.number()
});

type User = z.infer<typeof UserSchema>;

const data = UserSchema.parse(JSON.parse(input));   // runtime check
```

## 7. 【坑】装饰器实验性（TS 5.0 前）

```typescript
// TS 5.0 前需要 experimentalDecorators
{
    "compilerOptions": {
        "experimentalDecorators": true,
        "emitDecoratorMetadata": true
    }
}

// TS 5.0+ 标准化（Stage 3 ECMAScript），但需 target: ES2022+
```

**实战**：Angular / NestJS 仍用 experimentalDecorators（生态滞后）。

## 8. 【坑】strictPropertyInitialization

```typescript
class User {
    name: string;   // ❌ strict 模式报错：未初始化
    
    constructor() {
        this.name = "default";  // ✅ 必须初始化
    }
}

// 或显式 !
class User {
    name!: string;  // 告诉 TS：会初始化（unsafe）
}
```

## 9. 【坑】import 循环依赖

```typescript
// a.ts
import { b } from "./b";
export const a = () => b();

// b.ts
import { a } from "./a";  // ❌ 循环
export const b = () => a();
```

**解决**：
- 抽公共到 `common.ts`
- 用 dynamic `import()` 延迟加载
- 重构为事件/回调

## 10. 【坑】CommonJS 互操作

```typescript
// TS 默认编译 ESM
import _ from "lodash";

// CommonJS 库的导出：default 可能是整个 module.exports
// TS 编译报错或运行时 undefined

// 解决 1：allowSyntheticDefaultImports
// tsconfig.json: { "allowSyntheticDefaultImports": true, "esModuleInterop": true }

// 解决 2：命名空间导入
import * as _ from "lodash";
```

## 11. 【坑】类型与运行时脱节

```typescript
// 接口仅编译期，运行时没有
interface User {
    name: string;
}

// 同一份 User 可以从 JSON、网络、DB 来——类型无法保证
function isUser(obj: unknown): obj is User {
    return typeof obj === "object" && obj !== null && "name" in obj;
    // 不验证 name 是否真的是 string（运行时类型错误）
}

// ✅ 用 zod 严格验证
const UserSchema = z.object({ name: z.string() });
const user: User = UserSchema.parse(input);  // 严格 runtime 验证
```

## 12. 【坑】Promise.all 错误处理

```typescript
// ❌ 一个失败，全部失败
const [a, b, c] = await Promise.all([
    fetch1(),   // 失败
    fetch2(),
    fetch3()
]);
// 抛出 fetch1 的错误

// ✅ 用 Promise.allSettled（不抛错）
const results = await Promise.allSettled([fetch1(), fetch2(), fetch3()]);
results.forEach(r => {
    if (r.status === "fulfilled") console.log(r.value);
    else console.error(r.reason);
});
```

## 13. 【坑】tsconfig path 别名在运行时失效

```json
// tsconfig.json
{
    "compilerOptions": {
        "baseUrl": ".",
        "paths": { "@/*": ["src/*"] }
    }
}
```

```typescript
// 编译 OK，运行时 ✗
import { foo } from "@/utils/foo";  // ❌ Node.js 找不到 @/
```

**解决**：用 bundler（Vite/Webpack）解析，或运行时注册（tsconfig-paths/register）。

## 14. 【坑】circular type definitions

```typescript
// 循环引用类型
type A = { b?: B };
type B = { a?: A };

const a: A = {};
const b: B = { a };   // runtime OK，但类型可能无限递归
```

**实战**：通常没问题（TS 编译器处理），但 JSON.stringify 会爆栈。

## 15. 【工具检测】

```bash
# 类型覆盖率
npx type-coverage

# any 检测
npx ts-unused-exports

# 安全 lint
npx eslint --plugin security

# 性能分析
npx tsc --extendedDiagnostics
```

## 16. 实战 Checklist

- [ ] 项目第一天启用 strict + 额外严格选项
- [ ] 禁止 any（除非显式禁用）
- [ ] 公共 API 用 interface + JSDoc
- [ ] 运行时验证用 zod / yup
- [ ] CommonJS 互操作配置好
- [ ] 路径别名 runtime 解决
- [ ] 装饰器版本对齐
- [ ] 循环依赖检测

## 17. 反模式 Top 10

1. ❌ any 替代 unknown
2. ❌ 类型断言 narrow
3. ❌ ! 非空断言滥用
4. ❌ enum 替代字面量联合
5. ❌ interface 在不需要合并时用 type（不能合并 type）
6. ❌ 类型与运行时混淆
7. ❌ 不验证外部数据（JSON / DB）
8. ❌ 循环依赖
9. ❌ tsconfig path 别名不解决
10. ❌ 装饰器版本不一致

## 18. 推荐工具链

```
- 编译：tsc（CI） + esbuild/Vite（dev）
- Lint：ESLint + typescript-eslint
- 格式：Prettier
- 类型验证：zod / valibot
- 测试：vitest（推荐）/ Jest
- 监视：tsc --watch + tsc-files
- 提交前：husky + lint-staged
```

## 相关笔记

- **核心语法**: [[draft-01-syntax-core]]
- **类型系统**: [[draft-03-type-system]]
- **泛型**: [[draft-04-generics]]
- **最佳实践**: [[draft-09-best-practices]]
- **综合入口**: [[summary]]