# PROMPT — Everywhere Linux 架构对齐（交给开发 Agent）

把下面整段原样发给执行 agent。

---

## 你的任务

把 [Sylinko/Everywhere](https://github.com/Sylinko/Everywhere) 的 `src/Everywhere.Linux` 从旧架构迁移到 `main` 分支的新 Automation 架构，产出可提交给上游的 PR。

上游 `main` 分支的 Linux 代码**当前无法编译**（53 个 CS0246 错误）——因为 2026-09-30 的 Visual Context 重构删除了 `IVisualElement` / `IVisualElementContext` / `VisualElementSiblingAccessor`，而 Linux 侧从未跟随迁移。这是你的工作起点。

**相关 issue**：https://github.com/Sylinko/Everywhere/issues/520（维护者 DearVa 已回复，明确欢迎贡献并承诺指导；他也明确说 review 耗时长、会深度审查甚至自行改写）

---

## 第一步：读方案文档（必做，先读再动手）

完整执行方案已备好，**包含所有已核实的 API 签名、文件清单、验收标准**：

```
https://github.com/vegetable-kun/Everywhere-Linux/blob/main/docs/LINUX-ALIGNMENT-PLAN.md
```

本地（服务器上）：
```
/home/ecs-assist-user/everywhere-linux-repo/docs/LINUX-ALIGNMENT-PLAN.md
```

> 该仓库是 **PRIVATE**。用 `gh` 访问：
> ```bash
> gh api 'repos/vegetable-kun/Everywhere-Linux/contents/docs/LINUX-ALIGNMENT-PLAN.md' \
>   -H 'Accept: application/vnd.github.raw'
> ```

**方案文档里的 API 签名全部在 main HEAD (`66f755f8`) 核实过**，直接用，不要重新猜。但你要**自己验证当前 main 是否又有新提交**——main 每天都在动。

---

## 执行环境

**推荐在服务器上开发**（`/home/ecs-assist-user/Everywhere`），因为它能直连 GitHub 和私有 NuGet 源。

| 项 | 值 |
|----|----|
| 仓库路径 | `/home/ecs-assist-user/Everywhere` |
| .NET SDK | `/home/ecs-assist-user/.dotnet`（10.0.401，main 必需） |
| 环境变量 | `export DOTNET_ROOT=/home/ecs-assist-user/.dotnet` + `export PATH=$PATH:$DOTNET_ROOT` |
| 系统 | Ubuntu Server 24.04, x86_64, **1.6 GB RAM** |
| 图形 | **无头环境**（`DISPLAY` 空）→ 你**不能**在此运行 GUI |

```bash
export DOTNET_ROOT=/home/ecs-assist-user/.dotnet
export PATH=$PATH:$DOTNET_ROOT
export DOTNET_CLI_TELEMETRY_OPTOUT=1 DOTNET_NOLOGO=1
cd /home/ecs-assist-user/Everywhere
```

**内存是主要约束**：1.6 GB 上跑 .NET 10 构建峰值接近上限，构建耗时 **4–5 分钟**，务必用后台任务 + 日志落盘，不要用前台长超时。

**GUI 验证需要用户的 X11 主机**（Ubuntu + GNOME X11）。到了需要真机验证的阶段，把产物打包交给用户去跑——不要自己猜运行结果。

---

## 已知的环境事实（不必重新排查）

**✅ 私有 NuGet 源在服务器上直连正常**：`https://nuget.sylinko.com/v3/index.json` 返回 HTTP 200，`dotnet restore` 可直接下载 9 个 `999.*` 私有包。**不需要**镜像、不需要本地文件夹源、不需要 `-k`。

**✅ GitHub 直连正常**：HTTP 200，clone/push 无障碍。

**✅ 子模块已初始化**：6 个子模块（`3rd/` 下：`ClassicDiagnostics.Avalonia`、`Motion.Avalonia`、`semantic-kernel`、`shad-ui`、`EverythingNetCore`、`anthropic-sdk-csharp`）。若报 `MSB3202 ... project file was not found`，跑 `git submodule update --init --recursive`。

**⚠️ 用户的 X11 主机连不上 GitHub**——若需要在那边构建，把包或产物用 tar 传过去，不要假设那边能 clone。

**⚠️ `main` 的 `global.json` 钉死 SDK 10.0.401**（`allowPrerelease: false`，无 `rollForward` → 默认 `latestPatch`，同 feature band 内必须 ≥ 401）。不要装 10.0.100。

**⚠️ `docs/build.md` 里写的 SDK "10.202" 是文档笔误**，以 `global.json` 为准。

---

## 执行阶段

方案文档里有完整的 Phase 0–4 细节。这里给的是**执行纪律**：

### Phase 0 — cherry-pick CI/打包修复（零风险，先做）

从 `dev/linux` cherry-pick 8 个与架构无关的提交到 `origin/main` 新分支：

```bash
git checkout -B feat/linux-modernize origin/main
git cherry-pick 1049887b 33e8ea3a b933ea95 bf780b86 \
                 17761c82 984966f3 3bd19778 c2536e27
```

**完成标准**：`dotnet build src/Everywhere.Linux/Everywhere.Linux.csproj -c Release -r linux-x64` 输出 `0 Error(s)`（功能可以是 stub）。

> ⚠️ `linux-release.yml` 在 main 上是 `disabled_manually` 状态。**不要自行启用**——这需要维护者同意，写进 issue 评论里问他。

### Phase 1 — 迁移到新契约，让编译通过

- `Everywhere.Linux.csproj` 加 `Everywhere.Automation` 项目引用
- 删除 3 个旧契约文件（`VisualElementContext.cs` / `.Picker.cs` / `.Screenshot.cs`）
- 检查 `ScreenSelectionSession.cs` 是否需保留（它对应 macOS 的 `MacScreenSelectionService`）

**完成标准**：Linux 项目 0 error 编译通过。

**⛔ 完成 Phase 1 后必须停下来**：在 issue #520 下发一条评论同步进度（附编译日志），**然后才继续 Phase 2**。这是硬性要求——维护者会深度 review，中途同步能避免整条路走偏。

### Phase 2 — 移植 AT-SPI 模块化

从 `dev/linux` 的 `53c58bfe` 提交取 10 个模块（`src/Everywhere.Linux/Interop/AtspiService/`）。

**不要 merge 整个 `dev/linux` 分支**——它落后 main 934 提交，merge 有 4 个冲突，且不会自动解决 53 个 CS0246。用 `git checkout origin/dev/linux -- <path>` 逐路径取。

8/10 模块可直接用（`Native/*`、`Core/AtspiCacheManager`、`Core/AtspiEventhub`），只有 `Elements/AtspiElement.cs` 和 `ElementFactory.cs` 需要适配新契约。

**完成标准**：AT-SPI 分层落地，编译通过。

### Phase 3 — 实现 `LinuxVisualElementBackend`（核心）

新建 `src/Everywhere.Linux/Automation/LinuxVisualElementBackend.cs`，实现 `Everywhere.Automation/IVisualElementBackend`。

**关键策略：先只做 `TopLevel` 切片**，暂不实现 `Direct` 和 `Screen`。理由：macOS 的 `TopLevelWindowEnumerator.cs` 只有 7.2 KB、Windows 的 5.8 KB——这是个能独立 review 的小切片。小 PR 是降低被拒风险的唯一手段。

实现时注意：
- `VisualElement` 子类必须实现 4 个抽象成员：`QueryCore` / `CreateEnumeratorCore` / `CaptureCoreAsync` / `ReleaseCore`
- 元素身份用 `VisualElementIdentityMap` 管理（参考 macOS 的 `GetOrCreateAXElement`）
- `VisualElementQueryRequest.Default` 的文本上限是 **4096** 字符，不要超过
- **`Query()` 返回后不得持有 context / retention / element 引用**（接口 XML doc 明确要求）

**完成标准**：能通过 AT-SPI 读到当前活动窗口标题。

### Phase 4 — 进程角色派发

main 的架构是三进程：`Main` / `Input` / `Automation`。Linux 的 `Program.cs` 目前完全没有这段。

参照 `src/Everywhere.Windows/Program.cs` 和 `src/Everywhere.Mac/Program.cs`。

`AutomationHostSession` 在 `src/Everywhere.Core/ProcessIsolation/Automation/`（**跨平台共享，不要重写**），构造函数收 `IVisualElementBackend`。

**⚠️ 线程模型需要你实测确认**：Windows 要求 MTA 线程，macOS 要求主线程。Linux 上 AT-SPI 走 D-Bus，线程安全性需要验证。**先实验再下结论**，不要假设。

**完成标准**：Automation Host 能独立进程启动并响应 RPC。

---

## 硬性约束

**🔴 分支与 PR**

- 基线始终是 `origin/main`，不要基于 `dev/linux`
- 拆成多个小 PR（PR-1 = Phase 0，PR-2 = Phase 1，PR-3 = Phase 2+3）
- **不要一次性提交全部改动**

**🔴 安全**

- **不要**用 `-k` 或关闭 TLS 校验
- **不要**用 `--ignore-failed-sources` 掩盖下载失败
- **不要**关闭 cert 校验来"绕过"问题

**🔴 范围**

- PR 里只改 Linux 相关文件 + 必要的 csproj 引用
- **不要**在 PR 里夹带无关重构、格式化、或"顺手"改的东西

**🔴 沟通**

- Phase 1 完成后发 issue #520 评论同步进度
- 下面三件事**先问维护者再做**：启用被禁用的 CI workflow、决定 AT-SPI 拆分是否可直接采用、决定 PR 拆分粒度
- issue 和 PR 用**英文**撰写
- 上游 issue 模板要求勾选「已搜索重复 issue」并填 `App Version` 字段

**🔴 版本控制**

- 不要 force push 到任何共享分支
- 每个 Phase 一个 commit（或按方案文档的建议拆分），commit message 用仓库风格：`type(scope): description`

---

## 交付物

每个 Phase 结束时，产出：

1. **可编译的代码**（`0 Error(s)` 为硬指标）
2. **验证证据**——编译日志、测试输出。`dotnet build` 和 `dotnet publish` 都要实际跑过并把输出贴出来
3. **commit**（在 `feat/linux-modernize` 分支上）
4. **进度同步**——Phase 1 完成后在 issue #520 下评论

**最终目标**：一个能提交给上游的 PR。提交前先给我过目——**不要直接开 PR**，除非我明确说要开。

---

## 报告要求

每个阶段结束时，用这个结构报告：

```
## Phase N 完成
**改了什么**：文件清单 + 每个文件的作用
**验证**：编译命令 + 实际输出的关键行
**commit**：hash + message
**遇到的问题**：具体现象 + 根因 + 解法
**需要决策的点**：需要我或维护者拍板的地方
**下一步**：Phase N+1 的具体动作
```

---

## 开工前先做这三件事

1. 读方案文档 `docs/LINUX-ALIGNMENT-PLAN.md`（全文）
2. 检查 main 是否又有新提交：
   ```bash
   git fetch origin main
   git log origin/main --oneline -3
   ```
   如果有新提交，方案里引用的文件路径和行号可能变了——**自己重新核实 API 签名**
3. 验证环境可用：
   ```bash
   export DOTNET_ROOT=/home/ecs-assist-user/.dotnet
   export PATH=$PATH:$DOTNET_ROOT
   dotnet --version        # 期望 10.0.401
   cd /home/ecs-assist-user/Everywhere
   git submodule update --init --recursive
   ```

---

## 需要你回答的问题（开工前先答，别猜）

1. 方案的 Phase 划分和优先级你认可吗？有要调整的吗？
2. 你打算在服务器上做，还是有更好的执行位置？
3. Phase 3 的 `TopLevel` 切片，你觉得 AT-SPI 实现上可行吗？有没有我漏掉的技术障碍？

答完这三个就开始 Phase 0。
