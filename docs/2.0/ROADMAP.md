# 2.0 主干迁移与实施指导（OpenCode 基线）

- Status: active
- Date: 2026-09-12
- Baseline: `opencode` `dev` @ `95daf90670b7c039c436c85537da5fbfe2205b41` (version `1.18.30`)
- Product mainline branch: `main`
- Upstream sync branch: `dev`
- Decision source: https://github.com/ericapaeus/vertent2/issues/614

> 本文用于指导 2.0 在 opencode 最新代码基础上的迁移与实施，不授权直接实施。
> 实施前仍需拆分方案、做 spike 验证，并按项目规范评审。

---

## 1. 背景与结论

2.0 不再继续在 vertent2 内的 1.14.29 副本上做增强，而是迁移到 opencode 最新主干：

- 仓库：`/Users/share/ai/opencode`
- 基线：`dev` 分支 `95daf9067`（opencode `1.18.30`）
- 产品主干：`main`
- 上游同步：`dev` 只用于同步 `source` / `upstream`，不在上面做产品提交

结论：

1. opencode 最新代码已提供 2.0 所需的工作区身份、workspace adapter、插件 API、server、desktop 和多工作区 UI 基础。
2. vertent2 中的插件宿主和自动化 JobEngine 不能整包复制；它们是 Fastify/Drizzle/自有服务栈下的实现，必须作为参考实现重新适配。
3. 2.0 的产品代码应尽量放在独立 package 中，对 `packages/opencode` / `packages/core` / `packages/app` / `packages/desktop` 的改动要尽量薄，否则会再次形成难同步的分叉。

---

## 2. 分支与基线策略

- `dev`：上游同步分支，只做 fetch/merge，不承载产品提交。
- `main`：2.0 产品主干，从基线提交创建。
- `spike/*`：验证性分支，从 `main` 出。
- `feat/*`、`fix/*`：正式实施分支，从 `main` 出，走 PR/审查。
- 基线 tag：`baseline/2.0-opencode-1.18.30`。

同步方式建议：

- 每个上游 release，或至少每 1-2 周，把上游 `dev` 合并进 `main`。
- 合并前先跑一次冲突预演，记录冲突文件数、涉及包、处置结论。
- 不允许 `main` 长期落后于上游超过一个发布周期而不处理。

---

## 3. 目标架构

```text
Desktop / Control Plane（共享）
  ├─ 登录页/静态壳、账号索引、插件包管理、模板/缓存/更新
  └─ 登录成功后启动用户运行时

用户运行时（每用户独立进程，单活跃）
  ├─ 用户数据根：~/.hezor/<userId>/
  ├─ 内嵌 OpenCode server / SDK
  ├─ 模型/provider、Hezor 凭据
  ├─ 插件宿主：plugin loader + manifest + UI bridge + runtime inject
  ├─ Automation：持久化 JobEngine
  ├─ My Workspace / Knowledge 等第一方插件
  └─ 文件级会话授权标记
```

关键原则：

- 对话是基础能力，不是独立频道。
- 工作区身份使用 OpenCode `WorkspaceID` / `workspace` 体系。
- 插件关系是“插件 → 目录”，同一目录可被多个插件使用，最终解析到同一个 workspace。
- 自动化按用户运行，用户运行时和所需工作区 ready 后才激活。
- OpenCode 在用户运行时进程内嵌；工作区按目录惰性初始化。

---

## 4. 建议包结构

优先使用现有包，不足再新增：

- `packages/opencode`：尽量保持上游原貌。
- `packages/app`、`packages/desktop`、`packages/server`、`packages/session-ui`：作为产品 UI/壳层复用；产品改动尽量薄。
- `packages/vertent-core`：现有插件/宿主雏形，重做为插件契约与生命周期核心。
- `packages/vertent-server`：插件 host / 产品 API 边界。
- `packages/vertent-ui`：PluginFrame / PluginBridgeHost / 插件前端桥。
- `packages/vertent-automation`（建议新增）：持久化 JobEngine、schedules、handler、恢复。
- `packages/vertent-runtime`（建议新增）：控制面/用户运行时/登录/数据目录编排。
- `packages/my-workspace`（建议新增）：首屏“我的工作区”插件。
- 知识库、发布等插件：迁移为独立插件包，不再走 server 直注入。

---

## 5. 上游优先使用的扩展点

1. **Workspace adapter**
   - `experimental_workspace.register` 可注册自定义 adaptor。
   - 任意本地目录工作区应实现为 `local` 类型 adaptor，不修改 control-plane core。

2. **Plugin v2 transform**
   - agent / skill / command / catalog / integration 等通过 `@opencode-ai/plugin/v2` 注册。
   - 插件启停优先使用 scope/dispose，不要依赖全局 dispose。

3. **工具注入**
   - tool registry 扫描 config 目录 `{tool,tools}/*.{js,ts}`。
   - 插件 runtime tool 优先通过 config 目录注入；只有在需要运行时动态注册/注销时才评估最小上游补丁。

4. **Server / SDK**
   - 用户运行时通过 `Server.listen()` 或 `Default().app.fetch` 使用 OpenCode。
   - 插件 host 通过官方 SDK 创建 session，不要裸改内部实例状态。

5. **Desktop sidecar**
   - 复用 upstream desktop 的 utilityProcess sidecar 机制。
   - 用户级隔离通过启动不同 userDataPath/XDG/OPENCODE_DB 的用户运行时实现。

---

## 6. 参考资产处置

### 从 vertent2 吸收的

- `plugin-api` 类型契约；
- `plugin-frontend-api` postMessage 协议；
- 事务化 install/update/uninstall 与回滚；
- `dataDirectoryId` / `replaces` 数据身份解耦；
- workspace 路径逃逸、Zip-Slip、manifest 校验；
- JobEngine：lease/retry/verifier/dedupe/schedule/启动恢复；
- Hezor provider 与专属模型接入经验。

### 不直接迁移的

- Fastify 路由与 `coreServices` 装配；
- Drizzle core DB 与全局单例；
- 现有 plugin-loader 对固定默认 workspace 的假设；
- 旧“对话频道”导航与页面结构；
- 旧全局设备级数据目录与 JWT 模型。

### 参考分支

- `hm0905`：早期插件宿主 MVP，可参考 PluginFrame / runtime inject / capability-center 思路，但需按本文重做。
- `plugin-impl`：过旧，只作历史参考。
- `source/2.0`：早期上游探索，不采用。

---

## 7. 实施顺序

### P0：基线、仓库纪律与 Spike

- 冻结基线 commit/tag、建立 `main`。
- 建本文档。
- 完成第 8 节 spike，证明关键扩展点可行。

### P1：插件契约与宿主最小闭环

- 插件 manifest / PluginContext / frontend SDK 收敛。
- loader、registry、安装事务、shutdown、热更新最小闭环。
- hello/internal plugin 全链路。

### P2：用户运行时与工作区

- 控制面/用户运行时启动编排。
- `~/.hezor/<userId>/` 数据目录、登录/离线、Hezor 凭据。
- local workspace adaptor、workspace registry 与插件绑定。
- 文件级会话授权。

### P3：Automation

- JobEngine schema/迁移/恢复。
- 插件 `ctx.jobs`、schedules、verifier。
- 用户运行时 + 工作区 ready 后 activation。
- 内部/外部任务模型。

### P4：第一方插件

- `my-workspace` 首屏插件及 AGENTS.md / skills / tools。
- 知识库外部插件、专属工作区、查询 tool、一次性迁移。
- 专用模型/OpenCode 原生模型共存。

### P5：UI 模式、发布与迁移

- tab / split / fullscreen。
- 桌面打包、自动更新、旧数据迁移引导。
- 测试矩阵、跨平台验收。

---

## 8. Spike 清单与退出标准

Spike 分支：`spike/2.0-foundation`

必须验证：

1. 从当前 upstream dev 启动 opencode server。
2. 注册一个 `local` workspace adaptor，把任意目录登记为 workspace。
3. 一个 hello 插件可加载：server routes + UI iframe + runtime agent/skill/tool。
4. 插件 runtime tool 能被 OpenCode 会话调用。
5. JobEngine 最小切片：一个持久化定时 job，可执行、可取消、可恢复。
6. 用户运行时可按 `~/.hezor/<userId>/` 启动，OpenCode DB/配置路径隔离。
7. 登录/离线身份不影响 OpenCode 原生模型能力。

退出标准：

- 上述 7 项有可复现证据；
- 不需要大范围修改 `packages/opencode` core；
- 若必须修改，能列出明确 patch 列表、范围和回馈 upstream 计划。

---

## 9. 上游同步与补丁纪律

- 每个必须的上游补丁登记：文件、原因、范围、替代方案、回馈上游状态。
- 能通过插件 API / config / adapter 解决的，不进入 core patch。
- 不允许把 vertent2 的 Fastify/Drizzle 依赖带入上游包。
- 每次同步 merge 后运行：typecheck、相关 package tests、插件/自动化 smoke。
- 冲突预算一旦超限，先停止新功能扩张，收敛分叉面。

---

## 10. 决策索引

2.0 产品与架构决策记录见：

- vertent2#614: https://github.com/ericapaeus/vertent2/issues/614

当前已确认方向摘要：

- Hezor 账号唯一登录，允许缓存身份离线进系统；
- 控制面 + 每用户独立运行时进程；
- 用户数据在 `~/.hezor/<userId>/`，共享区只放通用资源；
- 无系统默认工作区；“我的工作区”是插件自己的独立工作区；
- 插件与工作区关系是“插件 → 目录”，同一目录可被多个插件使用；
- 工作区身份使用 OpenCode WorkspaceID；
- 对话是基础能力，无独立对话频道；
- 自动化按用户运行时，在运行时和所需工作区 ready 后激活；
- opencode.db 保持 SQLite；
- 专用模型由 Hezor 平台授权，本地不维护第二份 allowlist；
- 文件级会话授权根在用户目录；
- 知识库为外部插件、专属工作区、查询 tool、一次性迁移。

---

## 11. 待完善

1. `my-workspace` 文件 schema 与 AGENTS.md 组织规则；
2. tab / split / fullscreen 的 manifest 与交互契约；
3. 知识库查询 tool 的输入输出与权限边界；
4. 插件专属工作区默认路径与生命周期；
5. 多缓存账号离线进入的选择策略；
6. 用户运行时切换时在途 job/session/tunnel/SSE 的停止语义；
7. 旧数据迁移引导与回滚细节；
8. 外部插件信任与权限模型；
9. 上游默认模型/default agent 回退策略。

---

## 12. 风险

- 上游 dev 迭代快，main 同步若失守会再次变成顽固分叉。
- upstream plugin v2 与 vertent2 插件契约差异大，移植工作量被低估。
- OpenCode workspace 与 session 的内部状态变更可能影响插件 UI/bridge。
- 用户级隔离、桌面进程编排、文件级授权都属于上游没有的能力，需要自研。
- 旧 vertent2 与新 2.0 并行期间要避免对外发布双机制。
