# 交接文档：Everywhere Linux 平台开发

> **给接手方（另一个 AI agent 或开发者）的完整上下文。**
> 目标：让 `Sylinko/Everywhere` 的 Linux 支持跟上 `main` 的新架构，并评估 Wayland 可行性。
>
> **接手第一步：先执行「§9 验证清单」的第一条命令，确认本文档描述的状态属实，再继续。**

---

## 1. 任务目标

| 优先级 | 目标 | 可检验的完成标准 |
|--------|------|------------------|
| **P0** | 让 `src/Everywhere.Linux` 在 `main` 分支上**编译通过** | `dotnet build src/Everywhere.Linux/Everywhere.Linux.csproj -c Release -r linux-x64` → **0 errors** |
| **P1** | 让 Linux 实现**新架构**（`Everywhere.Automation`）的核心能力 | 屏幕上下文感知在 X11 下可返回 `VisualElement` 快照 |
| **P2** | **Wayland 可行性调研**（不要求实现） | 产出一份技术结论文档，说明可行/不可行/部分可行的边界与理由 |
| **P3** | 向**上游**提交贡献 | 向 issue #520 补充信息，或开 PR |

> ⚠️ **P0 是 P1 的前置**。当前 `main` 的 Linux 代码引用了 8 个已删除类型，不修好无法进入新架构工作。
> ⚠️ **P2 是上游明确提出的开放问题**，见 §4。**这是本任务里性价比最高的一项**——不需要写大量代码，且直接回应维护者。

---

## 2. 现状快照（截至 2026-10-21）

### 2.1 上游仓库状态

| 项 | 值 |
|----|----|
| 仓库 | https://github.com/Sylinko/Everywhere |
| `main` HEAD | `66f755f80`（2026-10-21） |
| `dev/linux` HEAD | `c3a670262`（**2026-02-05，停滞 8 个月**） |
| `main` vs `dev/linux` | `diverged`；main **ahead 933**，dev/linux **ahead 13** |
| 合并记录 | **从未合并**；无任何 `dev/linux → main` 的 PR |
| `dev/linux → main` 的反向 PR | PR #272（`base: dev-linux`, `head: main`），**未合并** |
| Linux CI | workflow `disabled_manually`，历史运行 **0 次** |
| 历史 release 的 Linux 产物 | **从未有过**（如 `v0.9.0-canary.20261010.25` 仅 Windows + macOS） |

### 2.2 两个分支的关键差异

| 维度 | `main` | `dev/linux` |
|------|-------|-------------|
| 解决方案文件 | `Everywhere.Linux.slnx` | **`Everywhere.Linux.slnf`**（solution filter） |
| `global.json` SDK | `10.0.401`（严格，`latestPatch` 默认） | `10.0.100` + **`rollForward: latestMajor`**（宽松） |
| NuGet 私有源 | **需要** 9 个 `999.*` 私有包 | **不需要**，仅 `nuget.org` |
| 编译结果 | ❌ 53 个 CS0246 | ✅ **0 errors**, 4 warnings |
| 架构代际 | 新（`Everywhere.Automation`） | 旧（`Everywhere.Interop.IVisualElement`） |

> 🔑 **这两个分支的「可编译」和「新架构」是互斥的**。`dev/linux` 能编译是因为它还在用旧架构；`main` 用新架构但 Linux 没跟上。

### 2.3 `main` 上的具体破坏点

**错误类型**：8 个已删除的类型，分布在 `src/Everywhere.Linux/Interop/`：

| 旧类型（Linux 仍在用） | 已被替换为 |
|---|---|
| `Everywhere.Interop.IVisualElement` | `Everywhere.Automation.VisualElement` |
| `VisualElementSiblingAccessor` | `VisualElement.CreateEnumerator` |
| `IVisualElementContext` | `IVisualElementBackend` + `VisualContext` 模型 |

**受影响文件**（`main` 分支）：

```
src/Everywhere.Linux/Interop/VisualElementContext.cs
src/Everywhere.Linux/Interop/VisualElementContext.Picker.cs
src/Everywhere.Linux/Interop/VisualElementContext.Screenshot.cs
src/Everywhere.Linux/Interop/X11WindowBackend.cs
src/Everywhere.Linux/Interop/X11Backend/X11VisualElement.cs
src/Everywhere.Linux/Interop/IWindowBackend.cs
src/Everywhere.Linux/Interop/AtspiService.cs
```

**结构性缺陷（不只是改名能解决的）**：

`src/Everywhere.Linux/Everywhere.Linux.csproj` 的 `ProjectReference` 只有 3 项：

```xml
<ProjectReference Include="..\Everywhere.Core\Everywhere.Core.csproj" />
<ProjectReference Include="..\Everywhere.I18N.Abstractions\Everywhere.I18N.Abstractions.csproj" />
<ProjectReference Include="..\Everywhere.I18N.SourceGenerator\Everywhere.I18N.SourceGenerator.csproj" ... />
```

对比 `src/Everywhere.Mac/Everywhere.Mac.csproj`：

```xml
<ProjectReference Include="..\Everywhere.Cloud\Everywhere.Cloud.csproj"/>
<ProjectReference Include="..\Everywhere.Core\Everywhere.Core.csproj"/>
<ProjectReference Include="..\Everywhere.I18N.Abstractions\Everywhere.I18N.Abstractions.csproj"/>
<ProjectReference Include="..\Everywhere.I18N.SourceGenerator\Everywhere.I18N.SourceGenerator.csproj" .../>
<ProjectReference Include="..\Everywhere.ProcessIsolation\Everywhere.ProcessIsolation.csproj"/>
```

> ⚠️ **Linux 缺 `Everywhere.Cloud` 和 `Everywhere.ProcessIsolation` 两个引用。**
> `Everywhere.Cloud` 提供模型 API 调用，`Everywhere.ProcessIsolation` 提供新的进程隔离模型——**这解释了为什么 macOS 的后端能直接启动 Hosts，而 Linux 不行**。
> **只做类型重命名无法让 Linux 编译通过。**

**另一个容易踩的坑**：`src/Everywhere.Linux/Interop/IWindowBackend.cs` 在 `main` 上**仍然返回旧接口**：

```csharp
IVisualElement? GetFocusedWindowElement();
IVisualElement GetWindowElementAt(PixelPoint point);
IVisualElement? GetWindowElementByInfo(int pid, PixelRect rect);
IVisualElement GetScreenElement();
IEnumerable<IVisualElement> Screens { get; }
```

→ 整个 `IWindowBackend` 契约本身都要重设计，不是替换几个类型名。**这是本任务最大的工作量所在。**

### 2.4 新架构接口（已从 `main` 提取的权威签名）

`src/Everywhere.Automation/IVisualElementBackend.cs`：

```csharp
namespace Everywhere.Automation;

public interface IVisualElementBackend : IDisposable
{
    VisualElementQueryResult? Query(
        VisualElementRetention retention,
        VisualElementLocator locator,
        VisualElementResolution resolution = VisualElementResolution.Direct,
        VisualElementQueryRequest? request = null);
}
```

**XML 文档要点（实现时必须遵守）**：

- `retention` — 选择目标 `VisualContext` 的所有权批次；实现**不得**在调用返回后持有 context / retention / element
- `locator` — 查询执行时采样的源；`VisualElementLocator.Default` 表示无锚点
- `resolution` — 拓扑解析结果（Direct / 上溯 / 下钻）
- `request` — 有界标量查询；`null` 表示用 `VisualElementQueryRequest.Default`

**新架构项目清单**（`src/Everywhere.Automation/`）：

```
Everywhere.Automation.csproj
IVisualElementBackend.cs        ← Linux 要实现的核心接口
IVisualElementCapture.cs
IVisualElementCursor.cs
IVisualElementEnumerator.cs
VisualElement.cs
VisualContext.cs
VisualContextPlatformOptions.cs
VisualContextSnapshot.cs
VisualContextSnapshotLimits.cs
VisualContextSnapshotter.cs
VisualContextTraverseDirections.cs
EmptyVisualElementEnumerator.cs
I18N/Strings.resx  I18N/Strings.zh-hans.resx
```

### 2.5 ⭐ 参考实现是 macOS，不是 Windows

上游 `docs/Automation/07-Migration.md` 明确写道：

> macOS *"starts its Hosts directly rather than exposing Windows service mode"*

→ **Linux 应对标 macOS 的「直接启动 Hosts」模式**，而不是 Windows 的 service mode。

**macOS 参考文件**（`main` 分支，`src/Everywhere.Mac/Automation/`）：

```
MacVisualElementBackend.cs        ← Linux 的直接模板
AXSystemWideVisualElement.cs      ← 系统级特殊根节点
AXVisualElement.cs
AXUIElementIdentity.cs
AXWindowResolver.cs
CGDisplayEnumerator.cs
CGDisplaySiblingEnumerator.cs
CGDisplayVisualElement.cs
MacVisualPickerResolver.cs
TopLevelWindowEnumerator.cs
AXFailureExtensions.cs
```

**参考要点**（从 `AXSystemWideVisualElement.cs` 摘录）：

```csharp
protected override VisualElementQueryResult QueryCore(VisualElementQueryRequest request)
{
    // 特殊根节点拒绝普通标量批次 → 直接返回 InvalidUIElement
    // 它的 Context identity 和 special-root type 是 Backend 已知的本地事实
    // → 查询它们时不得发起 provider RPC
    var availableFields = VisualElementFields.None;
    ...
    return new VisualElementQueryResult(this, snapshot, availableFields,
        request.RequestedFields & ~availableFields, null);
}
```

> 🔑 **「哪些字段是 Backend 本地已知事实、哪些必须走 provider RPC」是实现新后端的核心设计约束**，macOS 这里有完整示范。

### 2.6 `dev/linux` 的 AT-SPI 模块化拆分（未合并，值得移植）

`dev/linux` 独有提交 `53c58bfe3`：

```
refactor: complete decomposition of at-api2 service for better modularity
```

将单体 `AtspiService.cs`（1054 行）拆为 13 个模块：

```
src/Everywhere.Linux/Interop/AtspiService/
├── AtspiService.cs                  (240)
├── Core/AtspiCacheManager.cs        (75)
├── Core/AtspiContext.cs             (95)
├── Core/AtspiEventhub.cs            (252)
├── Elements/AtspiElement.cs         (646)
├── Elements/AtspiElement.Actions.cs (223)
├── Elements/ElementFactory.cs       (245)
├── Native/AtspiNative.cs            (190)
├── Native/AtspiEnums.cs             (134)
└── Native/SafeAtspiHandle.cs        (55)
```

> 💡 **这是未来迁移的现成起点**，不是需要从零重写的东西。移植时请**逐个模块**移植（不要整分支 merge——见 §6）。

---

## 3. 上游沟通现状

### Issue #520（已开，状态 open，1 条评论）

**地址**：https://github.com/Sylinko/Everywhere/issues/520
**标题**：`[Feat] Linux: is Everywhere.Linux still intended to be maintained? (does not compile on main)`
**作者账号**：`vegetable-kun`

**维护者 DearVa 的回复要点**（2026-10-11，已完整阅读）：

1. 他目前是**唯一的核心维护者**，主开发机是 Windows + Mac 备份，Linux 暂无力维护
2. Linux 代码主要由 **@jimfengnpu** 贡献（非 C#/Linux 专家，纯个人兴趣）
3. 他**每次重构核心接口都会尽力保证 Linux 能编译**，但这次 VisualContext 组件大重构**无法用简单 patch 解决**
4. 已加入 **future TODO list**
5. **明确欢迎贡献**，承诺提供指导
6. 指出两个挑战：**Wayland 移植难度**、**代码质量标准高**（深度 review + 虚拟机测试 + 可能自行改写，review 耗时长）

### ⚠️ 沟通策略建议

| 做法 | 理由 |
|------|------|
| **先在 #520 下回复，再动手写代码** | 维护者已表态欢迎贡献，**先对齐方向再投入**能避免白做 |
| **先做 P2（Wayland 调研）** | 直接回应他提出的开放问题，投入小、可见度高 |
| **PR 前先开 draft** 或先问 | 他明说 review 耗时长，提前沟通可减少返工 |
| **联系 @jimfengnpu** | 他是 AT-SPI 代码的原始作者，可能仍有参与意愿 |
| **保持英文、保持谦逊** | 沿用 #520 的语气；只提问、不提「建议你们删代码」类主张 |

---

## 4. Wayland 调研（P2，建议第一个做）

### 上游态度

DearVa 在 #520 明确把 Wayland 列为**尚未解决的开放挑战**——这正是他希望有人回答的问题。

### 硬性技术约束（已从源码确认）

`src/Everywhere.Linux/Interop/IWindowBackend.cs` 的类注释：

```csharp
/// <summary>
/// Abstract display backend contract (X11[Xorg] / Wayland Compositor / other).
/// </summary>
```

→ 接口**设计时已预留 Wayland**，但 `main` 分支的实现只有 `X11WindowBackend`。

### Wayland 的根本性障碍（调研必须回答的核心问题）

| 能力 | X11 | Wayland | 备注 |
|------|-----|---------|------|
| **枚举窗口** | ✅ `_NET_CLIENT_LIST` / `_NET_WM_NAME` | ❌ **协议禁止** | compositor 不暴露全局窗口列表；只能从已知 surface 反推 |
| **读取其他应用的无障碍树** | ✅ AT-SPI | ⚠️ AT-SPI 在 Wayland 下可用，但**只能查询应用自己暴露的部分** | 需要目标应用启用 AT-SPI bridge（GTK/Qt 默认启用，Electron/Chromium 需额外配置） |
| **读取窗口几何/标题** | ✅ | ❌ 无法查询其他进程的窗口 | 只能通过焦点/surface 事件间接获取 |
| **屏幕截图** | ✅ XGetImage | ⚠️ 需 `xdg-desktop-portal` `ScreenCast` API | 用户必须交互授权（权限对话框） |
| **模拟键盘输入** | ✅ XTest | ⚠️ 需 `xdg-desktop-portal` `RemoteDesktop` API | 同样需要用户授权 |
| **全局热键** | ✅ XGrabKey | ❌ **协议禁止** | 需 compositor 扩展（各 compositor 不同），或用 portal / 独立守护进程 |

> 🔑 **调研的核心结论应该回答**：**在 Wayland 下，哪些能力可以用 portal 获得（且代价是什么），哪些能力在协议层面根本不可得？**

### 建议的调研维度

1. **portal 可覆盖的部分**：`ScreenCast` / `RemoteDesktop` 的实际能力边界、各 compositor（GNOME / KDE Plasma / wlroots / sway）的实现差异
2. **不可覆盖的部分**：全局窗口枚举、全局热键、跨应用无障碍树的读取
3. **替代架构**：是否需要「AT-SPI + portal + compositor 扩展」混合方案；是否有现成开源项目可参考（如 `wlrctl`、`swaymsg`、`ydotool`、`wtype`、`libei`/`wlr-protocols` 的 input-inhibit 等）
4. **结论建议**：是否建议 **X11 优先 + Wayland 部分支持** 的分层策略，并向上游提出「先保证 X11 正确性」的合理路线

### 可选参考

- `xdg-desktop-portal` 规范（ScreenCast / RemoteDesktop / InputCapture）
- `libei`（EI 协议，Wayland 输入模拟的新标准）
- AT-SPI2 在 Wayland 下的实现现状
- 各 compositor 的私有 IPC 协议（GNOME D-Bus / KDE KWin scripting / wlroots IPC）

---

## 5. 环境与依赖

### 5.1 开发主机（用户本机）

| 项 | 值 |
|----|----|
| 系统 | Ubuntu + GNOME **X11**（用户明确确认过，非 Wayland） |
| 用户 / 家目录 | `cxk` / `/home/cxk` |
| .NET SDK | `/home/cxk/app/dotnet`，**10.0.401** |
| 仓库位置 | `/home/cxk/git-design/everywhere` |
| 网络 | 直连 `github.com:443` **被拒**（Connection refused），需走加速镜像 |

> ⚠️ `github.com` 直连不通是**已验证的既有事实**，不是新问题。`git clone` 本身可能失败或极慢。

### 5.2 服务器（本次工作的执行环境）

| 项 | 值 |
|----|----|
| 系统 | Ubuntu Server 24.04 x86_64，Linux 6.8.0-124-generic |
| 内存 | **1.6 GB**（构建耗时 4:23，峰值接近上限，OOM 风险高） |
| 公网 IP | `8.163.87.180` |
| 已装 SDK | `~/.dotnet` → **10.0.100**（`dev/linux` 用） |
| 源码仓库 | `~/Everywhere`（当前在 `dev/linux` 分支） |
| 产物仓库 | `~/everywhere-linux-repo` → 已推送到 GitHub |

### 5.3 NuGet 私有源（`main` 分支的坑）

**关键事实**：`nuget.sylinko.com` **不是真正的 NuGet 源，是一个 HTTP 307 跳转器**：

```
nuget.sylinko.com/xxx.nupkg
  → 307 → github.com/Sylinko/.../releases/download/...
  → 302 → release-assets.githubusercontent.com/...
  → 200 → 真正的 nupkg
```

包实际托管在 **GitHub Releases**。

**`main` 分支需要的 9 个私有包**（`Directory.Packages.props`）：

| 包 | 版本 |
|----|------|
| `Porta.Pty` | 999.20261001.1 |
| `Porta.Pty.Native` | 999.20261001.1 |
| `MessagePack` | 999.20261001.1 |
| `MessagePack.Annotations` | 999.20261001.1 |
| `MessagePackAnalyzer` | 999.20261001.1 |
| `OfficeCLI` | 999.20261010.4 |
| `Microsoft.ML.Tokenizers` | 6.0.0 |
| `Microsoft.ML.Tokenizers.Data.O200kBase` | 6.0.0 |
| `EverythingNetCore` | 999.20261001.3 |

> ⚠️ **2026-10-21 最新检查：`https://nuget.sylinko.com/v3/index.json` 返回 HTTP 200**，网络环境可能已改善。
> **接手方第一步就应该直接试 `dotnet restore`**，如果能通就不需要任何绕行方案（§6 的方案全部作废）。
> 只有确认失败才启用 §6.2 的镜像方案。

---

## 6. 失败死路（**不要重走**，已实测）

### 6.1 ❌ 在 `main` 分支上试图编译

**结果**：53 个 `CS0246`（`IVisualElement` / `IVisualElementContext` / `VisualElementSiblingAccessor`）。
**原因**：上游 Visual Context 重构后 Linux 未迁移，**不是配置问题、不是 SDK 问题、不是包问题**。
**结论**：不要试图在 `main` 上「配好环境就能编译」。这是**上游代码缺陷**，必须改代码。

### 6.2 ❌ 把 `dev/linux` 整体 merge 进 `main`

**实测结果**：4 个冲突（2 个 content、2 个 modify/delete），且合并后：
- 旧类型引用**依然存在**
- `Everywhere.Automation` 项目引用**依然缺失**
- 53 个 CS0246 **不会自动消失**

**结论**：**禁止整分支 merge**。正确做法是**提取有价值的逻辑补丁**（如 §2.6 的 AT-SPI 模块化拆分），逐模块移植到新架构。

### 6.3 ❌ 在 `dev/linux` 上用 `.slnx`

**结果**：`MSBUILD : error MSB1009: Project file does not exist.`
**原因**：`dev/linux` 用 `.slnf`（solution filter）+ `Everywhere.sln`；`.slnx` 是 `main` 才有的新版格式。

### 6.4 ❌ 缺少子模块时直接 restore

**结果**：restore 失败。
**原因**：`dev/linux` 的子模块在 `3rd/` 目录下。
**修复**：`git submodule update --init --recursive`（6 个子模块，约 5 分钟）。

### 6.5 ❌ 用 `-k` 或关闭证书校验绕过 TLS

**明确禁止**。曾遇 `PartialChain`（证书链不完整），正确解法是刷新系统 CA：

```bash
sudo apt update && sudo apt install --reinstall ca-certificates
sudo update-ca-certificates --fresh
```

证书链详情：`sylinko.com → WE1 → GTS Root R4`（交叉签名版），缺 `GlobalSign Root CA`。

### 6.6 ❌ 用 `--ignore-failed-sources` 掩盖包下载失败

**明确禁止**。曾遇 `Connection refused (github.com:443)`（因 307 跳转器），正解是本地文件夹源 + `packageSourceMapping`，**不是**忽略错误。

### 6.7 ❌ 把 `dev/linux` 的构建成功当作「Linux 版完成」

`dev/linux` 是 **2026-02 的旧架构**，落后 main **933 提交**。新架构的 Chat / 策略引擎改进**都没有**。
它是「**能跑的旧版**」，不是「main 的 Linux 版」。

---

## 7. 已产出的资产（可直接复用）

### 7.1 GitHub 仓库

**https://github.com/vegetable-kun/Everywhere-Linux**（**PRIVATE**）

| 内容 | 说明 |
|------|------|
| `dist/` | 368 文件 / 163 MB，self-contained Linux 产物（**git-lfs 追踪**） |
| `README.md` | 15 KB 完整构建 handoff |
| `LICENSE` | 上游 BSL-1.1 原文 |
| `NOTICE` | 来源与构建声明 |
| `build-logs/` | 构建环境、耗时、警告、运行时依赖 |
| **Release v0.1.0** | 额外提供 62 MB tar.gz（**不依赖 git-lfs**） |

**下载方式**：
```bash
# 方式 1：Release（不依赖 git-lfs，推荐）
gh release download v0.1.0 -R vegetable-kun/Everywhere-Linux

# 方式 2：git clone + LFS
git clone https://github.com/vegetable-kun/Everywhere-Linux.git
cd Everywhere-Linux && git lfs pull
```

> ⚠️ 若 clone 后 `dist/Everywhere` 显示为 132 字节 ASCII text → 那是 **LFS 指针文件**，说明 git-lfs 未安装或未拉取。装 `git-lfs` 后执行 `git lfs pull`。

### 7.2 产物运行方式（已验证可行）

```bash
cd ~/everywhere
./Everywhere
```

- 默认快捷键：**`Ctrl + Shift + E`**（源码 `ChatWindowSettings.cs` 确认）
- 配置目录：`~/.config/Everywhere/`
- 开机自启：`~/.config/autostart/Everywhere.desktop`
- 单实例互斥量：`com.sylinko.everywhere`
- 必需运行库：`libX11.so.6`、`libatspi.so.0`、`libglib-2.0.so.0`、`libfontconfig.so.1`
- **必须 X11 会话**（`DISPLAY` 为空 → 硬退出；非 x11 → 仅警告）
- **AT-SPI 必须运行**（`at-spi-bus-launcher`），否则屏幕上下文感知不可用

### 7.3 已知上游 issue

**https://github.com/Sylinko/Everywhere/issues/520**（open，1 条维护者评论）

---

## 8. 关键决策与理由（**含否决的替代方案**）

| 决策 | 理由 | 否决的替代方案 |
|------|------|---------------|
| 以 `dev/linux` 作为**可编译基线**，而非 `main` | `main` 有 53 个 CS0246，是代码缺陷，非环境问题 | 反复折腾 `main` 的 NuGet/SDK 配置——无法解决代码缺失 |
| **禁止**整分支 merge `dev/linux` → `main` | 实测 4 冲突 + 53 错误不消失 | 整分支 merge 后再逐个修冲突（工作量大且方向错） |
| 参考实现选 **macOS** 而非 Windows | 上游文档明确 macOS「直接启动 Hosts」，与 Linux 场景同构 | 照抄 Windows service mode（架构不匹配） |
| 仓库设 **private** | BSL-1.1 分发边界未定；private 可随时改 public，反之麻烦 | 直接公开（一旦公开无法完全撤回，且可能被缓存） |
| 二进制用 **git-lfs** | 163 MB / 367 文件入 git 会永久膨胀；LFS 是社区标准 | 直接入 git（仓库永久 ~330 MB） |
| 先做 **Wayland 调研**（P2）再写代码（P1） | 直接回应维护者提出的开放问题，投入小、可见度高，且能为 P1 定方向 | 直接开写迁移代码（可能方向与维护者预期不符） |

---

## 9. 验证清单（接手方第一步照做）

### ① 确认上游状态未变

```bash
gh api repos/Sylinko/Everywhere/commits/main --jq '"\(.sha[0:9]) \(.commit.committer.date)"'
gh api repos/Sylinko/Everywhere/issues/520 --jq '"\(.state) comments=\(.comments) updated=\(.updated_at)"'
```
**期望**：`main` HEAD 接近 `66f755f80`（2026-10-21）；issue #520 仍 open。
**若已变化** → 重新核对 §2 的所有结论，尤其 `main` 是否已修好 Linux。

### ② 确认私有源是否已可直连（决定要不要绕行）

```bash
curl -s -o /dev/null -w "%{http_code}\n" --max-time 20 https://nuget.sylinko.com/v3/index.json
```
**期望**：200 → 直接 `dotnet restore` 即可，**跳过 §6 的所有绕行方案**。

### ③ 复现 `main` 的编译失败（确认 §2.3 属实）

```bash
git clone --recursive https://github.com/Sylinko/Everywhere.git
cd Everywhere && git checkout main
dotnet build src/Everywhere.Linux/Everywhere.Linux.csproj -c Release -r linux-x64
```
**期望**：53 个 `CS0246`。

### ④ 确认 `dev/linux` 仍可编译（回归基线）

```bash
git checkout dev/linux
dotnet restore Everywhere.Linux.slnf -r linux-x64
dotnet build src/Everywhere.Linux/Everywhere.Linux.csproj -c Release -r linux-x64 --no-restore
```
**期望**：`Build succeeded. 0 Error(s)`，4 warnings，耗时约 4:23。

### ⑤ 确认产物可运行（在有 X11 的机器上）

```bash
file dist/Everywhere          # 应为 ELF 64-bit executable，非 ASCII text
ldd dist/Everywhere | grep "not found"   # 应无输出
./Everywhere                  # 应弹窗；Ctrl+Shift+E 唤起
```

---

## 10. 下一步（按优先级，每条自包含可独立执行）

### 步骤 1（P2，最高性价比）— Wayland 调研

**产出**：一份技术结论文档，回答 §4 的四个调研维度。

**建议动作**：
1. 读 `xdg-desktop-portal` 规范中 `ScreenCast` / `RemoteDesktop` / `InputCapture` 三节
2. 查 `libei` / EI 协议在 Wayland 输入模拟上的现状
3. 调查 AT-SPI2 在 Wayland 下的实际覆盖率（哪些 toolkit 暴露、哪些不暴露）
4. 调查各 compositor 的私有 IPC 能否补足窗口枚举

**产出位置建议**：新建 `docs/wayland-feasibility.md`（或直接作为 issue #520 的后续评论）。

### 步骤 2 — 在 #520 下回复对齐方向

**要点**：告知维护者调研进展 + 打算从哪个窄切片开始；询问 `@jimfengnpu` 是否有意参与。

**语气**：沿用 #520 的谦逊风格；**只提问、不提要求**；提改动前先问。

### 步骤 3（P0）— 窄切片修复编译

**策略**（不要一次性改完）：
1. 先给 `Everywhere.Linux.csproj` 加 `Everywhere.Cloud` / `Everywhere.ProcessIsolation` 引用
2. 重新评估剩余错误数——**这会改变工作量估计**
3. 按 `docs/Automation/07-Migration.md` 的映射表逐个替换类型
4. 每个 commit 保持可编译

### 步骤 4（P1）— 实现 `LinuxVisualElementBackend`

**模板**：`src/Everywhere.Mac/Automation/MacVisualElementBackend.cs`
**约束**：遵守 §2.4 的 `Query()` 契约（不得跨调用持有 context/retention/element）
**建议首个切片**：只实现 `VisualElementResolution.Direct` + 顶层窗口枚举

### 步骤 5 — 移植 `dev/linux` 的 AT-SPI 模块化拆分

**逐模块移植**（见 §2.6 的文件清单），**不要整分支 merge**（见 §6.2）。

---

## 11. 置信度

| 部分 | 置信度 | 说明 |
|------|--------|------|
| 上游状态与差异分析 | **95%** | 全部由 `gh api` 实测提取，非推断 |
| 编译失败根因（53 CS0246） | **95%** | 已复现，且定位到缺失的项目引用 |
| 新架构接口签名 | **100%** | 直接从 `main` 提取原文 |
| macOS 参考实现 | **90%** | 文件清单和模式已确认；具体移植细节需读完整代码 |
| Wayland 障碍分析 | **75%** | 基于协议规范的已知事实，**各 compositor 的实际覆盖度需实测验证** |
| 私有源绕行方案 | **60%** | 最新检查 HTTP 200，**可能已不需要**；§5.3 已注明先试直连 |
| 工作量估计 | **50%** | 缺少 `Everywhere.Cloud` / `Everywhere.ProcessIsolation` 引用的连锁影响尚未实测 |

**低置信区域，接手方需重点复核**：
1. 加入两个 ProjectReference 后的**真实错误数**（这是最关键的不确定项）
2. Wayland 在各 compositor 上的**实际**能力覆盖度
3. 私有源是否真的已恢复直连

---

## 12. 环境速查（可直接复制）

```bash
# ---- 服务器 ----
export DOTNET_ROOT=/home/ecs-assist-user/.dotnet
export PATH=$PATH:$HOME/.dotnet
cd ~/Everywhere
dotnet --version          # 10.0.100

# ---- 用户本机 ----
export PATH=$PATH:/home/cxk/app/dotnet
cd /home/cxk/git-design/everywhere
dotnet --version          # 10.0.401
```

**磁盘提示**：服务器磁盘曾用到 95%（40 GB 中仅剩 1.9 GB）。清理 `~/.nuget`、`~/.dotnet`、仓库内 `bin/obj` 可释放数 GB。

---

## 附录 A：相关上游文件索引

**新架构**
```
src/Everywhere.Automation/                    ← 新契约所在项目
docs/Automation/07-Migration.md              ← 官方迁移映射表（必读）
```

**Linux 现状（main）**
```
src/Everywhere.Linux/Everywhere.Linux.csproj          ← 缺两个 ProjectReference
src/Everywhere.Linux/Interop/IWindowBackend.cs        ← 契约本身需重设计
src/Everywhere.Linux/Interop/X11WindowBackend.cs
src/Everywhere.Linux/Interop/X11Backend/X11VisualElement.cs
src/Everywhere.Linux/Interop/VisualElementContext.cs
src/Everywhere.Linux/Interop/VisualElementContext.Picker.cs
src/Everywhere.Linux/Interop/VisualElementContext.Screenshot.cs
src/Everywhere.Linux/Interop/AtspiService.cs          ← 单体，dev/linux 已拆分
```

**macOS 参考**
```
src/Everywhere.Mac/Automation/MacVisualElementBackend.cs   ← 直接模板
src/Everywhere.Mac/Automation/AXSystemWideVisualElement.cs
src/Everywhere.Mac/Automation/TopLevelWindowEnumerator.cs
```

**旧架构（dev/linux）**
```
src/Everywhere.Linux/Interop/AtspiService/            ← 模块化版本（未合并）
src/Everywhere.Linux/Interop/X11Backend/               ← 重组后的 X11 后端
```

## 附录 B：GitHub 账号与协作约定

| 项 | 值 |
|----|----|
| 账号 | `vegetable-kun` |
| 提交签名 | `vegetable-kun <vegetable-kun@users.noreply.github.com>` |
| Issue 语言 | **英文** |
| Issue 模板 | 遵循仓库 `feature-request.yml`（含 `Use the search function...` 勾选项 + `App Version` 字段） |
| 提交前 | **先出草稿给用户过目** → 确认署名/语言 → 再 `gh` 推送 |

---

*文档生成时间：2026-10-21 | 所有状态数据经 `gh api` / `git` 实测提取*
