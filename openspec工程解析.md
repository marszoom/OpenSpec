# OpenSpec 工程解析

## 1. 工程概览

OpenSpec 是一个 AI 原生、规范驱动开发工具，核心形态是 Node.js CLI。它负责管理项目中的需求规格、变更提案、实现任务和归档流程，并向 Claude、Cursor、OpenCode、Codex、Copilot 等 AI 编程工具生成对应的 commands 和 skills。

核心目标不是运行应用业务，而是帮助 AI 和开发者按以下流程协作：

```text
openspec init
    ↓
创建 openspec/ 目录与 AI 工具集成
    ↓
explore / propose
    ↓
生成 proposal、design、specs、tasks
    ↓
apply / verify
    ↓
sync / archive
    ↓
更新主规格并归档变更
```

主要工作流包括：

- `explore`：探索问题和方案
- `propose`：创建变更提案
- `new` / `continue` / `ff`：管理变更文档
- `apply`：按任务实现代码
- `update`：更新变更计划
- `verify`：验证实现结果
- `sync`：同步 delta spec
- `archive`：归档变更并更新主规格
- `bulk-archive`：批量归档
- `onboard`：帮助新用户理解项目

CLI 还提供初始化、配置、校验、状态查看、存储管理、Schema 管理、Shell 补全和诊断等功能。

## 2. 主要目录结构

```text
OpenSpec/
├─ bin/
│  └─ openspec.js              # 发布包实际执行入口
├─ src/
│  ├─ cli/                     # Commander CLI 根入口
│  ├─ commands/                # 面向 CLI 的命令实现
│  ├─ core/                    # 核心领域逻辑
│  │  ├─ command-generation/   # 各 AI 工具命令适配器
│  │  ├─ templates/             # 工作流和 skill 模板
│  │  ├─ validation/            # 规格、任务、变更校验
│  │  ├─ store/                 # 外部 Store、注册表、Git 操作
│  │  ├─ schemas/               # Zod 数据结构
│  │  ├─ parsers/               # Markdown 和规格解析
│  │  ├─ artifact-graph/        # 工件依赖图和状态解析
│  │  ├─ completions/           # Bash、PowerShell、Zsh、Fish 补全
│  │  ├─ init.ts                # 初始化和 AI 工具文件生成
│  │  └─ update.ts              # 工具集成更新、迁移和清理
│  ├─ utils/                    # 文件系统、路径、进程、差异等工具
│  ├─ prompts/                  # 交互式输入
│  ├─ ui/                       # 终端界面
│  └─ telemetry/                # 匿名遥测
├─ openspec/
│  ├─ changes/                  # 变更提案和进行中的变更
│  ├─ specs/                    # 主规格
│  ├─ config.yaml               # 当前工程配置
│  ├─ work/                     # 规划工作
│  ├─ initiatives/              # 长期计划
│  └─ explorations/             # 探索记录
├─ schemas/
│  └─ spec-driven/              # 内置规范 Schema
├─ skills/                      # 生成后的 skill 分发产物
├─ test/
│  ├─ cli-e2e/                  # CLI 端到端测试
│  ├─ commands/                 # 命令测试
│  ├─ core/                     # 核心逻辑测试
│  ├─ templates/                # 模板和生成结果一致性测试
│  └─ specs/                    # 规格行为测试
├─ docs/                        # 旧版文档树
├─ docs-lab/                    # 当前文档源文件
├─ website/                     # 独立的 Next.js/Fumadocs 文档站
├─ scripts/                     # skill 生成、哈希校验、发布辅助脚本
├─ .changeset/                  # Changesets 发布元数据
└─ dist/                        # TypeScript 构建产物
```

当前工作区执行过 `openspec init`，并存在：

```text
.opencode/
├─ commands/
└─ skills/
```

这些是 OpenCode 的项目本地生成文件，通常被 Git 忽略，不应手工修改。生成源主要位于 `src/core/templates/` 和 `src/core/command-generation/`。

## 3. 关键代码链路

### 3.1 CLI 启动链

```text
bin/openspec.js
  → dist/cli/index.js
  → src/cli/index.ts
  → src/commands/*
  → src/core/*
```

- `src/cli/index.ts`：注册 Commander 命令、处理全局选项、JSON 输出、遥测和错误
- `src/commands/`：CLI 命令层
- `src/core/`：领域逻辑和文件操作

### 3.2 初始化链路

`openspec init` 大致执行：

```text
读取参数
  → 确定项目 root
  → 读取全局配置和 profile
  → 选择 AI 工具
  → 读取工作流模板
  → 使用工具适配器转换
  → 生成 commands、skills 和配置文件
```

核心实现是 [src/core/init.ts](src/core/init.ts)。

### 3.3 更新链路

`openspec update` 会：

- 读取项目和全局配置
- 检查已配置的 AI 工具
- 迁移旧目录
- 检测生成文件漂移
- 根据 profile 和 delivery 模式重新生成文件
- 清理旧版 OpenSpec 管理文件
- 更新 GitHub Copilot cloud-agent 文件

核心实现是 [src/core/update.ts](src/core/update.ts)。

### 3.4 OpenCode 集成

OpenCode 的配置位于 [src/core/config.ts](src/core/config.ts)，主要输出：

```text
.opencode/skills/openspec-*/SKILL.md
.opencode/commands/opsx-<id>.md
```

适配器是 [src/core/command-generation/adapters/opencode.ts](src/core/command-generation/adapters/opencode.ts)，特点包括：

- 使用 YAML frontmatter 中的 `description`
- 使用 `/opsx-<id>` 形式的命令
- 通过 `$ARGUMENTS` 或位置参数传递命令参数
- 不使用 `/opsx:<id>` 这种命名空间形式

## 4. 依赖分析

### 4.1 生产依赖

依赖定义在 [package.json](package.json)：

| 依赖 | 用途 |
|---|---|
| `commander` | CLI 命令树、参数和选项解析 |
| `@inquirer/core`、`@inquirer/prompts` | 交互式问答、选择和确认 |
| `chalk` | 终端彩色输出 |
| `ora` | 进度 spinner |
| `yaml` | YAML 配置、Store 元数据和注册表解析 |
| `zod` | 配置、输入、Schema 和 Store 数据校验 |
| `fast-glob` | 搜索工作流、模板和生成文件 |
| `diff` | 规格和 delta spec 差异处理 |
| `cross-spawn` | 跨平台启动外部命令和编辑器 |

运行环境要求：

- Node.js `>=20.19.0`
- pnpm `10.34.5`
- 纯 ESM
- TypeScript `NodeNext`
- 所有相对导入必须使用 `.js` 扩展名

### 4.2 开发依赖

| 依赖 | 用途 |
|---|---|
| `typescript` | 严格模式 TypeScript 编译 |
| `vitest`、`@vitest/ui` | 单元测试、集成测试和测试 UI |
| `eslint`、`typescript-eslint` | 源码检查 |
| `@types/node` | Node.js API 类型 |
| `@changesets/cli` | 版本管理和发布 |
| `@changesets/changelog-github` | GitHub Release 日志 |
| `smol-toml` | TOML 相关测试和开发支持 |

### 4.3 Website 依赖

`website/` 是独立项目，不属于根 workspace，拥有自己的 lockfile 和安装流程。

主要技术：

- Next.js 16
- React 19
- Fumadocs
- Tailwind CSS 4
- PostCSS
- `beautiful-mermaid`
- `lucide-react`
- `zod`

网站通过 `docs.sync.config.mjs` 从 `docs/` 和 `docs-lab/` 同步文档内容。

## 5. 配置和数据存储

### 5.1 项目配置

项目级配置位于：

```text
openspec/config.yaml
```

用于配置：

- Schema
- 文档上下文
- 规则
- 工作流操作
- 外部 Store
- 引用关系
- GitHub Copilot cloud-agent

相关实现：[src/core/project-config.ts](src/core/project-config.ts)。

### 5.2 全局配置

全局配置通常位于：

- Windows：`%APPDATA%/openspec/config.json`
- macOS/Linux：`~/.config/openspec/config.json`
- 如果设置 `XDG_CONFIG_HOME`，优先使用该目录

全局配置包含：

- profile
- delivery 模式
- 工作流列表
- 默认 Store
- 遥测开关
- Shell completion 状态

相关实现：[src/core/global-config.ts](src/core/global-config.ts)。

### 5.3 Store

Store 支持将规格和变更放置在独立目录中，通过 registry 和 pointer 关联项目。

当前 Store 的 Git 支持主要用于本地初始化、状态探测和提交管理，并不自动负责远程 clone、pull 或 push。

## 6. 构建、测试和发布

根项目常用命令：

```bash
pnpm install
pnpm build
pnpm test
pnpm exec tsc --noEmit
pnpm lint
```

完整 CI 流程基本对应：

```text
pnpm build
pnpm test
pnpm exec tsc --noEmit
pnpm lint
```

重点注意：

1. `dist/` 是构建输出，CLI E2E 测试通常执行 `dist/` 中的代码。
2. 修改源码后，如果不重新执行 `pnpm build`，测试可能继续使用旧代码。
3. `tsconfig.json` 排除了 `test/`，因此 `tsc --noEmit` 不会检查测试代码类型。
4. Windows 是正式 CI 平台，路径测试必须兼容反斜杠、大小写和 junction/symlink。
5. `skills/` 和部分规格文档是生成或规范化产物，不能直接手工修改。

Website 命令需要在 `website/` 目录执行：

```bash
pnpm install
pnpm run dev
pnpm run build
pnpm run types:check
pnpm start
```

发布采用 Changesets：

```bash
pnpm changeset
pnpm run release
```

通常由 Changesets 自动生成 Version Packages PR，再发布 npm 包。

## 7. 风险事项

### 7.1 高风险

#### 生成文件与源模板不一致

影响范围：

- `skills/`
- `.opencode/`
- 各种 AI 工具的 commands/skills
- Schema 文档和 parity hash

错误地直接修改生成文件会在下一次 `openspec update` 时被覆盖，或导致 parity 测试失败。

建议：

- 修改 `src/core/templates/`、`src/core/command-generation/` 等源文件
- 重新运行生成脚本
- 执行相关 parity 测试

#### 多平台路径和文件身份处理

OpenSpec 同时支持 Windows、macOS 和 Linux，并处理：

- 反斜杠和正斜杠
- 大小写不敏感文件系统
- junction/symlink
- Windows 短路径和长路径
- 外部 Store 和 pointer

应使用 `path.join`、`FileSystemUtils` 和路径 canonicalization，不应硬编码 Unix 路径。

参考：[test/AGENTS.md](test/AGENTS.md)。

#### CLI 对用户项目的批量文件操作

`openspec init` 和 `openspec update` 会创建、迁移、刷新或删除 OpenSpec 管理的文件。错误的工具检测、legacy migration 或 profile 配置可能造成意外文件变更。

尤其需要谨慎处理：

- `.opencode/`
- `.agents/`
- `.github/`
- `.claude/`
- legacy 工具目录
- GitHub Copilot cloud-agent 文件

### 7.2 中风险

#### 遥测与网络请求

CLI 默认采用 opt-out 遥测模式，会发送：

- command name
- OpenSpec version
- 随机 anonymous UUID

遥测实现位于 [src/telemetry/index.ts](src/telemetry/index.ts)。

可通过以下方式关闭：

```bash
OPENSPEC_TELEMETRY=0
DO_NOT_TRACK=1
```

CI 环境会自动关闭。源码声明不会发送路径和文件内容，但遥测仍属于外部网络行为，部署环境应明确设置策略。

#### 自动升级行为

`openspec update` 可以在用户确认后执行：

```text
npm install -g @fission-ai/openspec@latest
```

它只在交互环境中执行，但仍会改变全局 CLI 版本，并受 npm、网络、全局权限和生命周期脚本影响。

#### 依赖和供应链风险

项目使用多个 CLI、Markdown、解析和生成相关依赖。根项目已经在 `pnpm-workspace.yaml` 中维护若干安全 overrides。

需要区分：

- 生产依赖会进入发布包
- Vitest、ESLint、Changesets 等主要是开发依赖
- Website 依赖不会进入 CLI npm 包

审计时不应只依据完整 lockfile 的告警判断生产风险，应区分 `pnpm audit --prod` 和开发依赖。

#### Website 与根项目构建隔离

根目录和 `website/` 使用不同的依赖安装、lockfile 和构建流程。只执行根目录 CI 并不能证明网站构建正常。

### 7.3 低风险或维护性风险

#### 测试类型覆盖不完整

TypeScript 配置排除了 `test/`，Vitest 主要做运行时转译，因此测试文件中的类型问题可能直到运行时才暴露。

#### 文档存在新旧两套目录

- `docs/`：传统文档
- `docs-lab/`：当前文档源

新文档应优先放入 `docs-lab/`，并确认 `website/docs.sync.config.mjs` 是否发布该页面，否则页面可能存在但不会出现在文档站。

#### OpenCode 输出目录被忽略

`.opencode/` 通常是生成结果，Git 不跟踪它。排查 OpenCode 问题时需要同时检查：

- 源模板
- OpenCode adapter
- `openspec init/update` 的实际输出
- 相关生成测试

不能只依赖 Git diff 判断 OpenCode 文件是否变化。

## 8. 建议的维护顺序

修改 CLI 或核心逻辑时，建议按以下顺序验证：

```bash
pnpm exec vitest run test/path/to/relevant.test.ts
pnpm build
pnpm test
pnpm exec tsc --noEmit
pnpm lint
```

涉及 AI 工具生成内容时，追加：

```bash
pnpm build
pnpm generate:skills
pnpm exec vitest run test/core/templates/skillssh-parity.test.ts
```

涉及网站文档时，在 `website/` 中追加：

```bash
pnpm run types:check
pnpm run build
```

总体而言，该工程的最大风险不在传统 Web 服务暴露面，而在 **跨平台文件操作、生成文件一致性、AI 工具集成兼容性、配置迁移和 CLI 对用户项目的批量修改**。
