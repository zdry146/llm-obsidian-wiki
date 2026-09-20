---
title: "TypeScript 构建与打包"
category: synthesis
tags: [typescript, build, bundler, webpack, vite, esbuild, swc, tsc]
sources:
  - "TypeScript Docs - Project References"
  - "esbuild Docs (esbuild.github.io)"
  - "Vite Docs (vitejs.dev)"
  - "Webpack Docs (webpack.js.org)"
summary: "TypeScript 构建工具对比：tsc / esbuild / swc / Webpack / Vite / Rollup + monorepo + 库打包"
provenance:
  extracted: 0.88
    inferred: 0.10
    ambiguous: 0.02
base_confidence: 0.86
lifecycle: draft
lifecycle_changed: 2026-09-20
created: 2026-09-20
updated: 2026-09-20
---

# §08 TypeScript 构建与打包

## 1. 5 大构建工具对比

| 工具 | 语言 | 类型检查 | 速度 | 适用场景 |
|------|------|---------|------|---------|
| **tsc** | TypeScript | ✅ | 慢（基线） | 类型检查 + 库编译 |
| **esbuild** | Go | ❌ | **~100x tsc** | dev 编译 + 大型项目 |
| **swc** | Rust | ❌ | ~80x tsc | Webpack 替代 loader |
| **Webpack** | JS | ❌ | 慢 | 传统打包（生态成熟） |
| **Vite** | JS (esbuild + Rollup) | ❌ | 极快 | dev + 现代构建 |
| **Rollup** | JS | ❌ | 中 | **库打包首选** |

## 2. tsc（官方编译器）

```bash
# 编译（生成 JS + .d.ts）
tsc

# 监听模式（dev）
tsc --watch

# 性能：~10s 编译 1000 个文件
# 优点：完整类型检查 + 类型声明生成
# 缺点：相对慢
```

**适用**：库发布（需要 .d.ts）、CI 类型检查。

## 3. esbuild（推荐 dev）

```bash
npm i -D esbuild
```

```javascript
// esbuild.config.js
import { build } from "esbuild";

await build({
    entryPoints: ["src/index.ts"],
    bundle: true,
    outfile: "dist/index.js",
    platform: "node",                  // 或 "browser"
    format: "esm",                     // 或 "cjs"
    target: "es2022",
    sourcemap: true,
    minify: true,
    treeShaking: true
});
```

**速度基准**（Vite 官方数据）：
- esbuild：**~100ms** 启动
- tsc：~10s 启动
- Webpack：~30s 启动

**esbuild 限制**：
- ❌ 不做类型检查（仅擦除）
- ❌ 不生成 .d.ts
- ✅ 仅编译 + bundle

## 4. swc（Rust 写的超快编译器）

```bash
npm i -D @swc/core @swc/cli
```

```json
// .swcrc
{
    "jsc": {
        "parser": { "syntax": "typescript" },
        "target": "es2022"
    },
    "module": { "type": "es6" }
}
```

**适用**：Webpack/SWC loader、Next.js 内置、Rust 生态。

## 5. Webpack（传统打包）

```bash
npm i -D webpack webpack-cli ts-loader
```

```javascript
// webpack.config.js
module.exports = {
    mode: "development",  // 或 "production"
    entry: "./src/index.ts",
    output: {
        path: __dirname + "/dist",
        filename: "bundle.js"
    },
    resolve: {
        extensions: [".ts", ".js"]
    },
    module: {
        rules: [
            {
                test: /\.ts$/,
                use: "ts-loader",     // 或 swc-loader（更快）
                exclude: /node_modules/
            }
        ]
    }
};
```

**优缺点**：
- ✅ 生态最丰富（loader/plugin 无数）
- ✅ 完整打包方案（CSS/Asset/Split）
- ❌ 慢（dev 启动 ~30s）
- ❌ 配置复杂

## 6. Vite（现代首选）

```bash
npm create vite@latest my-app -- --template vanilla-ts
```

```typescript
// vite.config.ts
import { defineConfig } from "vite";

export default defineConfig({
    // dev: esbuild（极速 HMR）
    // build: Rollup（生产打包）
    build: {
        target: "es2022",
        sourcemap: true,
        rollupOptions: {
            output: {
                manualChunks: {
                    vendor: ["react", "react-dom"]
                }
            }
        }
    }
});
```

**实战优势**：
- ✅ dev 启动 ~50ms（按需编译）
- ✅ HMR 极快（模块热替换）
- ✅ 生产 Rollup 打包（成熟）
- ✅ 内置 TS、CSS、Asset 支持
- ✅ 零配置可用

## 7. Rollup（库打包首选）

```bash
npm i -D rollup @rollup/plugin-typescript @rollup/plugin-node-resolve
```

```javascript
// rollup.config.js
import typescript from "@rollup/plugin-typescript";
import resolve from "@rollup/plugin-node-resolve";

export default {
    input: "src/index.ts",
    output: [
        { file: "dist/index.cjs", format: "cjs" },
        { file: "dist/index.mjs", format: "es" }
    ],
    plugins: [
        resolve(),
        typescript({ tsconfig: "./tsconfig.build.json" })
    ],
    external: ["react", "react-dom"]  // 不打包到 bundle
};
```

**优势**：
- ✅ Tree-shaking 最好（ESM 原生）
- ✅ 输出可读（库友好）
- ✅ 多格式输出（cjs + esm）

## 8. tsup（库打包零配置）

```bash
npm i -D tsup
```

```json
// package.json
{
    "scripts": {
        "build": "tsup src/index.ts --format esm,cjs --dts"
    }
}
```

**实战**：基于 esbuild，零配置库打包首选。

## 9. 实战：项目类型选择

### 9.1 库（npm publish）

```json
{
    "scripts": {
        "build": "tsup"
    }
}
```

- ✅ 推荐：**tsup**（零配置 + 快 + .d.ts 自动）
- 备选：Rollup（更细控制）

### 9.2 前端应用

```json
{
    "scripts": {
        "dev": "vite",
        "build": "tsc --noEmit && vite build"
    }
}
```

- ✅ 推荐：**Vite**（dev 快 + 生产成熟）
- 备选：Next.js（如果用 React 全栈）

### 9.3 后端 Node.js

```json
{
    "scripts": {
        "dev": "tsx watch src/index.ts",
        "build": "tsc"
    }
}
```

- ✅ 推荐：**tsc** + tsx（无需打包）
- 备选：esbuild + tsc（如果更快需要）

### 9.4 大型 monorepo

```json
{
    "scripts": {
        "build": "tsc --build"  // 用 project references
    }
}
```

- ✅ 推荐：**tsc project references** + esbuild（应用层）
- 备选：Nx / Turborepo（智能编排）

## 10. project references（大型项目）

```json
// tsconfig.json (root)
{
    "files": [],
    "references": [
        { "path": "./packages/core" },
        { "path": "./packages/web" },
        { "path": "./packages/api" }
    ]
}

// packages/core/tsconfig.json
{
    "compilerOptions": {
        "composite": true,        // 必须开
        "declaration": true,      // 生成 .d.ts
        "outDir": "./dist"
    }
}
```

```bash
# 增量编译（只编译修改的包）
tsc --build

# 强制全部重编译
tsc --build --force
```

## 11. monorepo 工具

| 工具 | 特点 | 速度 |
|------|------|------|
| **Turborepo** | 增量构建 + 远程缓存 | 快 |
| **Nx** | 完整 monorepo（构建 + 测试 + 部署）| 快 |
| **pnpm workspaces** | 简单 + 快 | 极快 |
| **npm workspaces** | 内置 | 慢 |

**实战**：pnpm workspaces + Turborepo = 当前最佳实践。

## 12. Bundle 优化技巧

```typescript
// 1. 树摇（默认开启）
// package.json: "sideEffects": false
{
    "sideEffects": false        // 告诉打包器可以安全删除未使用代码
}

// 2. 代码分割
const AdminPanel = lazy(() => import("./AdminPanel"));

// 3. 外部化大型依赖（库）
// rollup.config.js
external: ["react", "react-dom", "lodash"]

// 4. 生产压缩
{
    "build": "tsup --minify"
}
```

## 13. 实战：CI 构建

```yaml
# 1. 缓存 node_modules
- uses: actions/setup-node@v4
  with:
      cache: 'npm'

# 2. 类型检查（必须，tsc 是唯一完整类型检查工具）
- run: npm run type-check

# 3. 编译
- run: npm run build

# 4. 验证产物
- run: ls -la dist/

# 5. 测试打包（针对库）
- run: npm pack --dry-run
```

## 14. 关键工具决策

| 场景 | 推荐工具 |
|------|---------|
| **新前端项目** | Vite |
| **React + 全栈** | Next.js |
| **新 Node.js 后端** | tsx + tsc |
| **新库开发** | tsup |
| **库（精细控制）** | Rollup |
| **大型 monorepo** | tsc + Turborepo + pnpm |
| **微前端** | Module Federation + Vite/Webpack |
| **CLI 工具** | tsup + pkg |

## 15. 关键性能数据

```
构建时间对比（中等项目，1000 文件）：

  tsc (--build):        10s
  esbuild (冷启动):     0.5s   (~20x)
  swc:                 0.3s   (~33x)
  Webpack 5:           8s     (~1.25x)
  Vite (dev 启动):     0.5s   (~20x)
  Rollup:              4s
  tsup:                1s     (~10x)
```

**实战启示**：现代项目几乎不用纯 tsc（太慢），而是 **tsc 检查类型 + esbuild/swc/Vite 编译**。

## 16. 关键设计原则

1. **tsc 只管类型检查**——编译交给更快工具
2. **dev 用 esbuild/Vite**——HMR 极快
3. **库用 tsup/Rollup**——产物友好
4. **CI 必须跑 tsc**——完整类型检查
5. **monorepo 用 project references**——增量编译

## 17. 反模式

❌ **Webpack 5 dev**——太慢
❌ **tsc 编译应用**——太慢
❌ **不分离类型检查**——失去类型安全
❌ **外部化所有依赖**——bundle 过大
❌ **monorepo 不分项目**——构建慢

## 18. 实战推荐组合

```
新项目：Vite (前端) + tsx (后端) + tsc (CI 检查) + tsup (库)
老项目：Webpack 5 + ts-loader + 逐步迁移
库开发：tsup + tsc (生成 .d.ts)
monorepo：pnpm + tsc --build + Turborepo
```

## 相关笔记

- **工具链**: [[draft-07-tooling]]
- **模块系统**: [[draft-06-modules]]
- **最佳实践**: [[draft-09-best-practices]]
- **综合入口**: [[summary]]