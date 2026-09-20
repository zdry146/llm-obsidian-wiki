---
title: "TypeScript 工具链"
category: synthesis
tags: [typescript, tooling, tsc, ts-node, tsx, eslint, prettier, vscode]
sources:
  - "TypeScript Docs - tsc CLI"
  - "TypeScript Docs - tsconfig"
  - "ESLint Docs"
  - "Prettier Docs"
summary: "TypeScript 工具链：tsc / ts-node / tsx / ESLint / Prettier / IDE 集成 + tsconfig 完整配置"
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

# §07 TypeScript 工具链

## 1. tsc（TypeScript 编译器）

```bash
# 编译（默认 tsconfig.json）
tsc

# 指定配置文件
tsc --project tsconfig.build.json

# 监视模式（dev 用）
tsc --watch

# 不生成文件（只类型检查）
tsc --noEmit

# 增量编译
tsc --incremental

# 输出 sourcemap
tsc --sourceMap

# 显示详细错误
tsc --pretty

# 跳过 lib 检查（加速）
tsc --skipLibCheck
```

## 2. tsconfig.json 完整配置

```json
{
    "compilerOptions": {
        // 目标与模块
        "target": "ES2022",                    // 编译目标
        "module": "ESNext",                    // 模块系统
        "moduleResolution": "Bundler",         // 解析策略（推荐）
        "lib": ["ES2022", "DOM", "DOM.Iterable"],
        
        // 输出
        "outDir": "./dist",
        "rootDir": "./src",
        "sourceMap": true,
        "declaration": true,                   // 生成 .d.ts
        "declarationMap": true,
        "removeComments": false,
        
        // 严格模式（核心）
        "strict": true,                        // 启用所有严格选项
        "noImplicitAny": true,                 // 隐式 any 报错
        "strictNullChecks": true,              // null 检查
        "strictFunctionTypes": true,
        "strictBindCallApply": true,
        "strictPropertyInitialization": true,
        "noImplicitThis": true,
        "alwaysStrict": true,
        "useUnknownInCatchVariables": true,     // catch 默认 unknown
        
        // 额外严格选项
        "noUnusedLocals": true,                // 未使用变量
        "noUnusedParameters": true,           // 未使用参数
        "noImplicitReturns": true,             // 函数必须 return
        "noFallthroughCasesInSwitch": true,    // switch 必 break
        "noUncheckedIndexedAccess": true,      // 数组访问可能 undefined
        "noPropertyAccessFromIndexSignature": true,
        "exactOptionalPropertyTypes": true,     // 区分 ? 和 | undefined
        
        // 互操作
        "esModuleInterop": true,               // 启用 ESM/CJS 互操作
        "allowSyntheticDefaultImports": true,
        "forceConsistentCasingInFileNames": true,
        
        // 解析
        "baseUrl": ".",
        "paths": {
            "@/*": ["src/*"]
        },
        "resolveJsonModule": true,             // 导入 JSON
        
        // 跳过检查（性能）
        "skipLibCheck": true,                  // 跳过 .d.ts 检查（加速 ~5x）
        
        // 类型生成
        "isolatedModules": true,               // 单文件可独立编译（esbuild/swc 用）
        "verbatimModuleSyntax": true,          // 强制 type-only import
    },
    "include": ["src/**/*"],
    "exclude": ["node_modules", "dist", "build"]
}
```

**严格模式推荐**（项目初始化就用）：

```json
{
    "compilerOptions": {
        "strict": true,
        "noUncheckedIndexedAccess": true,
        "noImplicitReturns": true,
        "noFallthroughCasesInSwitch": true,
        "noUnusedLocals": true,
        "noUnusedParameters": true,
        "exactOptionalPropertyTypes": true
    }
}
```

## 3. ts-node 与 tsx（运行时执行 TS）

### ts-node

```bash
# 安装
npm i -D ts-node typescript

# 运行 TS 文件
ts-node src/index.ts

# 监视模式
ts-node --watch src/index.ts

# 配置（tsconfig.json + ts-node 子配置）
ts-node --project tsconfig.json src/index.ts
```

### tsx（更快，推荐）

```bash
# 安装
npm i -D tsx

# 运行
tsx src/index.ts

# 监视
tsx watch src/index.ts

# 优势：基于 esbuild，启动 ~50ms（ts-node ~2s）
```

**实战选择**：新项目用 tsx，老项目保留 ts-node。

## 4. ESLint（TypeScript 集成）

```bash
npm i -D eslint @typescript-eslint/parser @typescript-eslint/eslint-plugin
```

```javascript
// eslint.config.js（ESLint 9 flat config）
import tseslint from "typescript-eslint";

export default tseslint.config(
    tseslint.configs.recommended,
    {
        rules: {
            "@typescript-eslint/no-unused-vars": "error",
            "@typescript-eslint/no-explicit-any": "warn",
            "@typescript-eslint/explicit-function-return-type": "off",
            "@typescript-eslint/no-non-null-assertion": "warn"
        }
    }
);
```

## 5. Prettier（格式化）

```json
// .prettierrc
{
    "semi": true,
    "singleQuote": false,
    "tabWidth": 4,
    "printWidth": 100,
    "trailingComma": "es5",
    "arrowParens": "always",
    "endOfLine": "lf"
}
```

```bash
# 与 ESLint 集成（避免冲突）
npm i -D eslint-config-prettier

# 格式化
npx prettier --write "src/**/*.ts"

# 检查
npx prettier --check "src/**/*.ts"
```

## 6. VS Code 配置（推荐）

```json
// .vscode/settings.json
{
    "typescript.tsdk": "node_modules/typescript/lib",
    "typescript.enablePromptUseSourceMaps": true,
    
    // 保存时格式化
    "editor.formatOnSave": true,
    "editor.defaultFormatter": "esbenp.prettier-vscode",
    
    // ESLint 自动修复
    "editor.codeActionsOnSave": {
        "source.fixAll.eslint": "explicit"
    },
    
    // TypeScript 智能提示
    "typescript.suggest.autoImports": true,
    "typescript.updateImportsOnFileMove.enabled": "always"
}
```

## 7. npm scripts（标准）

```json
{
    "scripts": {
        "build": "tsc",
        "dev": "tsx watch src/index.ts",
        "start": "node dist/index.js",
        "type-check": "tsc --noEmit",
        "lint": "eslint . --ext .ts",
        "lint:fix": "eslint . --ext .ts --fix",
        "format": "prettier --write \"src/**/*.ts\"",
        "format:check": "prettier --check \"src/**/*.ts\"",
        "test": "vitest",
        "test:watch": "vitest watch"
    }
}
```

## 8. 实战：CI/CD 集成

```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]

jobs:
    test:
        runs-on: ubuntu-latest
        steps:
            - uses: actions/checkout@v4
            - uses: actions/setup-node@v4
              with:
                  node-version: 20
                  cache: npm
            
            - run: npm ci
            
            # 1. 类型检查
            - run: npm run type-check
            
            # 2. Lint
            - run: npm run lint
            
            # 3. 格式化检查
            - run: npm run format:check
            
            # 4. 单元测试
            - run: npm test
            
            # 5. 编译
            - run: npm run build
```

## 9. 类型生成（从后端 Schema）

```bash
# 从 OpenAPI 生成 TS 类型
npm i -D openapi-typescript
npx openapi-typescript https://api.example.com/openapi.json -o src/types/api.ts

# 从 GraphQL 生成
npm i -D @graphql-codegen/cli
npx graphql-codegen --config codegen.ts

# 从 Prisma schema 生成
npx prisma generate
```

## 10. 调试技巧

```bash
# tsc 报错信息解读
tsc --pretty false              # 单行错误信息（CI 用）

# 只检查某个文件
tsc --noEmit src/specific-file.ts

# 输出最常见的报错类型
tsc --diagnostics

# 详细错误位置
tsc --extendedDiagnostics
```

## 11. 实战：ts-node vs tsx vs node-loader 对比

| 工具 | 启动速度 | 配置复杂度 | 性能 | 推荐场景 |
|------|---------|----------|------|---------|
| **ts-node** | ~2s | 中 | 中 | 老项目 |
| **tsx** | ~50ms | 低 | 高 | **新项目首选** |
| **swc-node** | ~30ms | 中 | 极高 | 大型 monorepo |
| **node --loader** | 慢 | 高 | 低 | 不推荐 |

## 12. 关键设计原则

1. **strict: true**——项目第一天就启用
2. **skipLibCheck: true**——加速 ~5x
3. **verbatimModuleSyntax**——强制 type-only
4. **noUncheckedIndexedAccess**——数组安全
5. **exactOptionalPropertyTypes**——区分可选和 undefined
6. **tsc + ESLint + Prettier**——三位一体

## 13. 反模式

❌ **不启用 strict**——失去类型安全
❌ **any 滥用**——用 unknown + 类型守卫
❌ **skipLibCheck: false**——编译慢 5x
❌ **CommonJS 新项目**——失去 tree-shaking
❌ **手动 .d.ts 太多**——优先 @types/*

## 14. 关键调试技巧

```bash
# 类型不匹配？看具体类型
tsc --traceResolution          # 解析追踪

# 找不到模块？检查路径
tsc --listFiles                # 列出所有文件

# 性能问题？
tsc --extendedDiagnostics       # 各阶段耗时

# IDE 看不到类型？
Ctrl+Shift+P → "TypeScript: Restart TS Server"
```

## 相关笔记

- **核心语法**: [[draft-01-syntax-core]]
- **构建与打包**: [[draft-08-build-bundlers]]
- **最佳实践**: [[draft-09-best-practices]]
- **已知坑**: [[draft-10-known-issues]]
- **综合入口**: [[summary]]