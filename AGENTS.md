# AGENTS.md

DeepSeek Harness 是一个基于插件的智能体运行框架，构建于项目内置的 Cordis 之上：**一切皆插件**。修改`packages/`前请阅读 [docs/architecture.md](docs/architecture.md)；编写文档时请遵循[docs/AGENTS.md](docs/AGENTS.md)。

## 预发布立场：基础涉及优先于影响范围

**首次发布带标签的版本时，请删除本节。**当前没有外部使用者，因此应优先选择正确的基础设计，而不是添加兼容层：可以自由重命名或重新组织包，但必须同时更新所有引用。后端会拒绝旧的磁盘格式。SQLite 使用单调递增的 `SCHEMA_VERSION`；`dsh-session` 的 `SESSION_FORMAT_VERSION` 保持为 `0`，且不作兼容性承诺。

## Repository layout

```
vendor/      内置的 Cordis 源码——清单与同步流程见 vendor/README.md
packages/    @deepseek-ai/dsh-<pkg> 工作区，位于 packages/<group>/<pkg>/
  core/        产品 API 主干: session, system-prompt, tools, agent, agent-loop
  api/         远程 BFF 组装与 Typert RPC 网关
  typert/      类型图生成器、加载器与运行时注册表
  llm/         LLM 能力: Service Definition/Consumer + DeepSeek providers
  e2b/         E2B POC: sandbox + FS/subprocess 适配器
  shell/        bash 能力: Service Definition + local/pwsh providers + shell Consumers
  subprocess/  subprocess 能力 + 本地进程树 provider
  terminal/         持久化终端会话
  fs/          文件系统能力与策略
  lsp/         语言服务器能力
  skill/       skill provider 注册表 + 本地实现 + 目录/加载tool
  web/         web 能力: Service Definition + search/fetch providers + tool Consumer
  compaction/     上下文压缩能力 + 基础 provider
  context/     请求上下文插件
  subagent/    子智能体能力: Service Definition + providers + 委派 Consumers
  bundle/      可安装的 dsh --profile 补丁层 bundles
  workflow/    工作流能力 + worker-thread provider + tool Consumer
  todo/        todo_write tool
  plan/        作为日志状态记录的计划模式
  preset/      通过preset cordis.yml文件组合每个会话的智能体
  guard/       循环-清理 + 工具-超时 插件
  self-modification/  智能体检查并挂载自身插件
  hooks/       Claude Code/Codex hook bridges + wire-protocol 库
  session/     持久化会话数据: persistence, projection, titles, telemetry
  identity/    匿名身份
  settings/    user-settings capability + file provider
  credentials/ 凭证/授权能力 + env/.env provider
  acp/         仅用于自动化的 Agent Client Protocol 服务器
  interaction/ 审批/交互能力, permission, commands, ask-user
  boot/        共享的应用程序入口粘合层
  sdk/         JSON-RPC 协议, 服务器和 TypeScript 客户端
  examples/    示例 bundles (agent-spine + CLI/ACP/JSON-RPC 程序)
  experimental/ 不纳入正式发布的私有原型
  support/     dev/test 基础设施
  util/        零依赖工具
python/      Python SDK 和内置运行时 (见 python/README.md)
native/      @deepseek-ai/node-addon-landlock-run 的权威源码(见 native/README.md)
examples/    基于 packages/examples 的可运行 cordis.yml 叶节点 (见 examples/AGENTS.md)
.agents/     智能体工作流和 Agent Notes (`notes/`)
docs/        架构、 生成目录, 事后分析, cookbook (见 docs/AGENTS.md)
scripts/     仓库门禁与生成器
website/     从选定双语 docs/ 源文件投影而成的 VitePress 网站
```

包分组: [packages/README.md](packages/README.md).

## Commands

```sh
pnpm install            # 安装 pnpm 工作区依赖；要求 node ^22.19 或 >=24
pnpm run clean           # 删除构建输出及已删除包留下的安全残留
pnpm run test           # 运行 Vitest 单元测试
pnpm run test:coverage  # CI 覆盖率门禁：packages/*/*/src 中每个文件达到 100%
pnpm run test:e2e       # 真实 API 测试；没有 DEEPSEEK_API_KEY 时自动跳过
pnpm run test:snapshot  # 无密钥 ACP/headless 回放并与预期输出比较；过滤：-t <name>
pnpm run test:snapshot:record  # 重新记录预期输出（需要密钥）
pnpm run typecheck
pnpm run lint
pnpm run duplication    # 检测跨文件 TypeScript 克隆代码
pnpm run build          # tsc 输出 lib/types，tsdown 打包运行时代码
pnpm run hygiene        # knip、publint、工作区约束与 NodeNext 消费方检查
pnpm run check:windows-wine  # 仅在诊断已知 Windows 故障时运行（需要 wine）；该信号由 CI 负责
pnpm run doc-sync       # 运行全部文档门禁；叶级清单见 scripts/run-gates.ts
pnpm run website:build  # 构建 VitePress，同时检查失效链接
pnpm dsh --profile headless "task"  # 从源码运行单个任务（需要 DEEPSEEK_API_KEY）
pnpm run demo:cordis    # 智能体修改自身运行时（需要密钥）
pnpm run demo:acp       # ACP 自动化服务器（需要 DEEPSEEK_API_KEY）
```

### Host sandbox 故障

当必需的`gh`，`pnpm`、构建、测试或生成器命令因智能体沙箱阻止凭据、网络、IPC、文件监视或嵌套`sandbox-exec`而失败时，应先使用范围最小的主机权限提升原样重试，再诊断身份验证或项目故障。必须先取得沙箱导致故障的证据；不得绕过真实的测试失败，也不得绕过正在接受测试的产品沙箱。

### 在本地运行相关检查

推送前通过 [dsh-pre-push-checks](.agents/skills/dsh-pre-push-checks/SKILL.md)  运行检查：只报告实际运行过的命令。执行 `gh stack sync` 后立即验证；检查通过前不得合并。

- 检查证据必须匹配变更范围：作为变更运行聚焦测试；模型或用户输出变更运行快照；文档变更运行`doc-sync`；发布路径变更运行 build、hygiene 和构建产物冒烟测试；真实 provider 行为变更运行真实 API e2e。
- 不得默认运行完整测试套件，也不得因为提交或推送而重复已通过的检查。全量覆盖率和平台矩阵由 CI负责；只有用户明确要求、正在诊断CI，或变更确实取法缩小到仓库局部时，才在本地完整演练。
- CI 覆盖率门禁是`test:coverage`, 不是 `test` ([说明](docs/testing.md))。

## 密钥 / .env

真实 API 测试和演示读取`DEEPSEEK_API_KEY`、可选的`DEEPSEEK_BASE_URL`，以及仓库根目录的`.env`。在 cordis.yml 中，插件 `config` 和条目的 `disabled` 字段运行使用 `!!js`，绝不能使用 `!js`；其他元数据必须保持为字面值，因此条件组合也应使用 overlays（参见[入门说明](docs/cordis-primer.md#loader-configuration)）。绝不能提交凭据。没有密钥时，CI e2e会跳过；密钥策略由testing.md](docs/testing.md)规定。

## 约定

- 每个 npm 包都命名为 `@deepseek-ai/dsh-<name>`；内置包会重新设置 scope（参见[映射](docs/rescope.md)），并设置 `private: true`。每个 harness 包都将 `@deepseek-ai/cordis` 同时列为 peerDependency 和 devDependency。
- 全面使用 ESM（`"type": "module"`）。跨包导入使用包名，本地相对导入使用 `.ts` 扩展名。配置子进程在普通 Node 下运行构建后的 `lib/`；源码回归测试使用其声明的启动器（参见[测试策略](docs/testing.md#test-subprocess-launch-modes)）。`dsh` CLI 的源码启动通过 tsx 的纯 ESM hook（`node --import tsx/esm`）运行；它能到达的模块必须保持为 ESM，不能仅导出 CJS，因为支持的 Node 引擎范围无法统一使用 Node 原生 TypeScript 模式（参见[源码启动约定](.agents/notes/implemented/architecture/2026-07-29-dsh-source-launch-tsx-esm.md)）。Raw/Web `cordis.yml` 中的裸插件必须出现在其 resolver manifest 的 `dependencies` 中；`verify-cordis-config` 会强制检查。
- **注册属于 effect。** 所有贡献都必须通过 `ctx.effect()` 或 `ctx.on()` 注册；注册表的 `register()` 必须返回 disposer。
- **运行时不变式断言自身拥有的关系。** 应检查权威事件流或可变数据，而不是服务或方法是否存在、插件元数据或 effect 是否存在，也不能只检查固定的纯函数示例。没有合理关系可供检查时，附有说明的空 companion 才是正确做法（参见[包不变式规则](packages/AGENTS.md)）。
- **类型化事件使用声明合并**和可合并扩展的映射。事件 JSDoc 必须包含 `@mode` 和描述 payload 的 `@param`；不在 payload 中出现的作用域键必须标注 `@dshScopeScan unsupported`。公共服务方法必须记录参数和非 `void` 返回值。默认情况下，`SessionEventMap` 成员在读取时是必需的：不知道该成员类型的构建会拒绝日志，除非事件信封带有 `ignorable: true`；只有结构格式变更才增加 `SESSION_FORMAT_VERSION`（参见[机制说明](.agents/notes/implemented/architecture/2026-08-10-session-log-version-mechanism.md)）。
- **按判别标签执行 switch。** 封闭联合类型以 `assertNever` 结尾；可合并扩展的联合类型必须落入有文档说明的 default 分支。
- **Waterfall listener 必须调用 `next()`** 才能委派；不调用便直接返回会中断调用链（参见[语义](docs/cordis-primer.md#cordis-waterfall-semantics)）。
- **模型可见 ⟺ 已记录。** 任何进入模型请求的内容都必须能够从会话日志重建；新增模型可见输入时必须新增会话事件。
- **通过插件扩展，不修改循环。** 新行为应放在已有文档说明的扩展点上；修改 `agent-loop` 时必须同步更新 docs/architecture.md。
- **一个 capability seam 包含 Service Definition、Service Provider 和 Consumer 三种角色。** 它必须是完整能力，不能只包含一种角色；只有各角色独立演进时才能拆分（参见[术语表](docs/glossary.md#capability-seam)）。
- **优先使用维护良好的依赖，而不是自行实现，**前提是该依赖确实能删除项目自有代码和测试（参见[策略](.agents/notes/implemented/process/2026-07-26-dependencies-over-hand-rolling.md)）。
- **包边界上显式优于隐式。** 默认值必须由所属实现通过明确的 `resolve(request): Spec` 步骤解析，绝不能隐藏在 `run()` 内的 `?? default` 中；`dsh-shell` 的 request/spec 拆分是参考模板。
- **插件中不得硬编码可调参数。** 随部署而变化的选项必须是经过验证、可从 cordis.yml 修改的 `Config` 字段；`DEFAULT_*` 常量或测试 hook 不代表可配置性。协议常量、外部规范和安全不变式保持固定。
- **配置错误必须明确失败。** 能在加载时独立确定的错误应在加载时失败，否则应在最早能够解析的位置失败；绝不能静默跳过缺失的引用目标。
- **跨边界的不透明 ID 必须使用品牌类型，**即使用 `dsh-brand` 的 `Branded<B>`，绝不能使用裸 `string`。
- **在同一进程内的类型化边界上信任 TypeScript。** 不要仅为静态接口已要求的值增加运行时验证、回退行为或恶意输入测试；应在 parser/config、队列、模型/工具 JSON、持久化/文件、worker、进程和线协议边界进行验证。
- **源码平面与产物平面绝不能混用。** 静态门禁和测试通过 tsconfig `paths` 将工作区导入解析到 `src`，并且必须能在干净工作树上通过；使用构建后 `lib/` 的门禁必须声明该依赖（参见[布局](docs/development.md#typescript-project-layout)）。
- **保持编译器视图显式。** 除 `api/remotes` 外，每个包只属于一个 aggregate；仓库级程序必须以某个视图配置作为入口，绝不能使用根 solution（参见[布局](docs/development.md#typescript-project-layout)）。
- **空 `catch` 必须说明它吞掉了什么，**以及为何不会有其他错误到达该处；`try` 中只保留一条语句。
- 不要为代码中显而易见的事实编写注释。
- **并行值应保持对称。** 无法解释的不对称通常表示遗漏了提取工作。
- **测试描述行为，而不是“正确性”。** 改变过时行为时应同时修改对应测试；原因写入 PR。
- **非平凡变更必须在同一个 PR 中包含 Agent Note；**只有机械性或局部修改可以豁免（参见[适用范围](.agents/notes/README.md#when-to-write-one)）。归档 note 已冻结：绝不能编辑，也不能将其视为当前权威（参见[归档策略](.agents/notes/README.md#archiving-and-deletion)）。
-  **测试策略**见 [docs/testing.md](docs/testing.md)。每项非平凡的模型可见或产品用户可见行为变更，都必须通过同一个 PR 中真实可运行示例新增或更新无密钥快照；包测试、仅 e2e 断言和仅 mock fixture 均不能替代组装后应用的 transcript。Fixture 必须能在 macOS/Linux 上回放；应修复 fixture，而不是 normalizer。
- **工具的 UI 渲染意图属于其设计的一部分，**必须预先决定为 `generic`、`terminal` 或 `diff`，并确定 `locations`；展示方法必须是 `args` 的纯函数（参见 [cookbook](docs/cookbook/adding-a-tool.md)）。
- **为 capability seam、生命周期路径和 transcript 输出规划单元测试、e2e 与快照覆盖。** 缺失的快照框架支持必须包含在同一变更中。
- **两个 SDK 都必须投影 agent loop。** 修改 agent-loop、session-lifecycle 或 `SessionEventMap` 时，必须在同一个 PR 中更新 TypeScript 和 Python SDK 的预期输出；`pnpm run test` 不覆盖这两项（参见[适用范围](docs/testing.md#when-a-snapshot-test-is-required)）。
- **有意识地设计 PR 历史。** 拆分相互独立的变更；在传播前修复引入问题的 PR。独立 PR 和官方 stack 在评审后可以执行 merge-forward 或 rebase。重写历史时使用 `--force-with-lease`，远端发生移动时必须中止，绝不能使用裸 `--force`；正在进行的 merge-forward 必须保留检查点，再接入更新的基线（参见[原理](.agents/notes/implemented/process/2026-08-02-native-github-stacks-and-optional-rebases.md)）。
- **标签：**每个 PR 使用一个 `kind/*` 标签、所有相关的 `area/*` 标签，以及原生 Issue Type（参见[分类规则](.agents/notes/implemented/process/2026-08-08-unified-github-label-taxonomy.md)）。
- TODO 标记按紧急程度使用 `FIXME`、`TODO` 或 `XXX`（参见[语义](docs/development.md)）。
- 每个文件必须以且仅以一个换行符结尾；提交前由 `git diff --cached --check` 门禁检查。

## 防御性模式

处理生命周期、并发、子进程或 teardown 前，请阅读 [docs/defensive-patterns.md](docs/defensive-patterns.md)。

## 类型安全与文档

所有代码都必须在 `strict: true` 和 `noImplicitAny` 下编译；每个仍然存在的 `any` 都必须说明为何无法进一步收窄。每个模块和导出都必须为其非显而易见的约定提供简洁 JSDoc；函数形式的导出必须包含 `@param` 和 `@returns` ，并由 `verify-export-jsdoc` 强制检查。通过继承声明的成员、插件协议插槽和构造函数，应将文档保留在声明它们的 Service Definition、协议或类中。

注释和文档应描述完整约定与上下文，而不是推理过程。使用直接、具体的术语，不要使用比喻。写下 `contract`、`boundary`或`shape`前，应先判断是否有更准确的词语：使用 `response fields`、`JSON validation` 或 `ESM exports`，而不是 `response shape` 、`validation boundary` 或 `module shape`。只有在表达前置条件、后置条件、不变式、兼容性承诺，以及调用方、被调用方、实现者、provider、生产者或消费者所依赖的其他义务时，才使用 `contract`。仅在确实指进程、线协议、安全、事务或生命周期边界时使用 `boundary `。不要叙述控制流或测试，不要保留评审历史，也不要复述代码。保留行为、故障、时序、所有权和安全使用要求，并链接其原理。需要判断措辞时使用[dsh-prose-standard](.agents/skills/dsh-prose-standard/SKILL.md)。将可机械检查的不变式接入实际执行的顶层门禁，并证明每条发生变更的验收路径都会拒绝无效情况。使用范围狭窄且理由明确的例外，而不是全局禁用规则。

每次代码变更都必须伴随文档更新：同时更新受影响的 README 和JSDoc 约定。常规双语工作遵循[docs/AGENTS.md](docs/AGENTS.md)；只有用户明确要求时才能运行 `dsh-translate-docs`。当前状态表述、每个段落只占一个物理行、每项事实只有一个归属位置，以及字数预算等规则均由该文件规定。

## 编辑这些说明

根目录、`packages/` 和 `examples/` 中的 `CLAUDE.md` 都链接到相应的 `AGENTS.md`；请编辑真实文件。每条规则必须能够独立成立，同时链接到更高层文档。应在不损害清晰度的情况下压缩内容；当必要内容确实需要更多空间时，提高 `verify-doc-budgets` 上限。

## 内置源码策略

`vendor/` 中的包是固定版本的源码副本，其清单和上游 SHA 记录在 [vendor/README.md](vendor/README.md) 中。请按照其中的同步流程更新；重新应用或废弃已记录的本地修改；然后重新运行`pnpm run test && pnpm run build`。
