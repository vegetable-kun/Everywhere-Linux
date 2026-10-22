# 给接手 Agent 的启动提示词

> **用法**：把本文件全文（连同 `HANDOFF.md`）提供给新的 agent 会话，或直接粘贴下面的代码块。
> 前提：新 agent 能访问 GitHub 仓库 `vegetable-kun/Everywhere-Linux`（**private**，需要 `vegetable-kun` 账号的读权限）。

---

## 版本 A：完整上下文（推荐）

```text
你要接手 Sylinko/Everywhere（https://github.com/Sylinko/Everywhere）项目的 Linux 平台开发。

请先阅读交接文档，它包含全部实测数据、失败死路和验收标准：
  https://github.com/vegetable-kun/Everywhere-Linux/blob/main/HANDOFF.md
（仓库为 private，需先 gh auth login / 配置访问权限）

同时可参考同仓库的 README.md（构建过程记录）和 build-logs/（构建环境）。

任务目标，按优先级：
  P2  Wayland 可行性调研  ← 建议第一个做，投入小、直接回应上游维护者提出的开放问题
  P0  让 src/Everywhere.Linux 在 main 分支上编译通过（当前 53 个 CS0246）
  P1  实现新架构 IVisualElementBackend（参考 src/Everywhere.Mac/Automation/MacVisualElementBackend.cs）
  P3  向上游提交贡献（issue #520 已开，https://github.com/Sylinko/Everywhere/issues/520）

开始任何编码之前，请先执行 HANDOFF.md「§9 验证清单」的第 ① ④ 两条命令，
确认上游状态和编译基线仍与文档描述一致。若已变化，先更新文档再继续。

关键约束（务必遵守）：
  - 禁止整分支 merge dev/linux 到 main（已实测：4 冲突 + 53 错误不消失）
  - 禁止用 -k 或关闭证书校验绕过 TLS
  - 禁止用 --ignore-failed-sources 掩盖包下载失败
  - 不要重新尝试已失败的路径，先读 HANDOFF.md「§6 失败死路」
  - 提交上游前先出英文草稿对齐仓库 issue 模板，署名 vegetable-kun

完成每个阶段后，请更新 HANDOFF.md 的对应章节，保持文档与实际状态同步。
```

---

## 版本 B：精简版（如果只想要核心信息）

```text
接手 Sylinko/Everywhere 的 Linux 平台开发。

上下文文档：https://github.com/vegetable-kun/Everywhere-Linux/blob/main/HANDOFF.md（private）

一句话现状：main 分支的 Linux 代码停留在旧架构，引用了 8 个已删除类型（53 个 CS0246），
且 csproj 缺 Everywhere.Cloud / Everywhere.ProcessIsolation 两个项目引用。
dev/linux 分支能编译（0 errors），但它是 2026-02 的旧架构版本，落后 main 933 提交。

建议第一个任务：Wayland 可行性调研（上游维护者在 issue #520 明确提出这是开放挑战）。
产出：一份技术结论文档，说明哪些能力可用 xdg-desktop-portal 获得、哪些在 Wayland 协议层根本不可得。

参考实现是 macOS（src/Everywhere.Mac/Automation/MacVisualElementBackend.cs），不是 Windows。
禁止整分支 merge dev/linux 到 main。
```

---

## 需要准备的访问权限

| 需要 | 用途 | 怎么配 |
|------|------|--------|
| `vegetable-kun` 账号的 repo 读权限 | 读 `HANDOFF.md` / `README.md` / `build-logs/` | 用该账号登录，或把仓库改为 public，或把 `HANDOFF.md` 单独发给新 agent |
| GitHub API token | 查上游 issue / commit | `gh auth login` |
| 对 `Sylinko/Everywhere` 的 fork 权限 | 提 PR | `gh repo fork Sylinko/Everywhere --clone` |
| 上游 issue #520 的评论权限 | 与维护者沟通 | 用 `vegetable-kun` 账号 |

---

## 如果新 agent 与当前 agent 在**同一台机器**

它可以直接读本地文件，无需 GitHub 权限：

```bash
# 交接文档
cat /home/ecs-assist-user/everywhere-linux-repo/HANDOFF.md

# 源码仓库（已在 dev/linux 分支）
cd /home/ecs-assist-user/Everywhere
git log --oneline -3

# 已构建产物（可直接运行验证）
cd /home/ecs-assist-user/everywhere-publish && ./Everywhere
```

---

## 如果新 agent 在**另一台机器**

1. 取得 `vegetable-kun/Everywhere-Linux` 的读权限（或直接接收 `HANDOFF.md` 文件）
2. clone 上游源码：
   ```bash
   git clone --recursive https://github.com/Sylinko/Everywhere.git
   ```
3. 按 `HANDOFF.md` §9 验证清单确认基线
4. 按 §10 下一步执行

---

*配套文件：`HANDOFF.md`（完整交接文档）· `README.md`（构建记录）· `build-logs/`（环境数据）*
