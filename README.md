# Everywhere — Linux 自构建产物

> **非官方构建。** 源码版权归 [Sylinko Inc.](https://github.com/Sylinko/Everywhere) 所有，本仓库仅保存**自行编译的 Linux 二进制产物**与构建记录。
> 上游项目采用 **BSL-1.1** 许可证（见 `LICENSE`）。本仓库仅用于个人学习与非商业用途。

本仓库保存了 [Sylinko/Everywhere](https://github.com/Sylinko/Everywhere) 的 **Linux 自包含构建产物**，以及从零开始完成该构建的完整排查与操作记录。

| 项目 | 值 |
|------|-----|
| 上游仓库 | [Sylinko/Everywhere](https://github.com/Sylinko/Everywhere) |
| 构建分支 | `dev/linux`（commit `c3a67026218eba3d7c47674c8884f97a1e919c32`） |
| 源码日期 | 2026-02-05 |
| 目标运行时 | `linux-x64`（.NET 10，self-contained） |
| 产物大小 | 约 163 MB / 367 个文件 |
| 构建主机 | Ubuntu Server 24.04, x86_64, 1.6 GB RAM, 无头环境 |

---

## 📦 产物说明

```
dist/            # 自包含构建产物（自包含 .NET runtime，无需预装 .NET）
├── Everywhere              # 主程序（ELF 64-bit PIE executable）
├── Everywhere.dll          # 主程序集
├── libcoreclr.so           # 自带 .NET runtime
├── libSkiaSharp.so         # 图形渲染
├── libHarfBuzzSharp.so     # 文本排版
├── Everywhere.Watchdog     # 守护进程
└── *.dll / *.so            # 依赖库
```

**self-contained**：产物自带 .NET 运行时，目标机器**无需安装 .NET SDK 或 runtime**。

### 运行

```bash
cd dist
./Everywhere
```

### 系统依赖

| 库 | 用途 |
|----|------|
| `libX11.so.6` | X11 窗口交互（**必需**） |
| `libatspi.so.0` | AT-SPI 无障碍树，屏幕上下文感知（**必需**） |
| `libglib-2.0.so.0` | GLib（GirCore 绑定依赖） |
| `libfontconfig.so.1` | 字体配置 |

Debian / Ubuntu：

```bash
sudo apt install -y libx11-6 libatspi2.0-0 libglib2.0-0 libfontconfig1
```

Arch：

```bash
sudo pacman -S --needed libx11 at-spi2-core glib2 fontconfig
```

### ⚠️ 运行环境要求

- **必须 X11 会话。** 程序在启动时检测会话类型，**非 X11 会话会直接抛异常退出**（`Program.cs` 中有硬检查）。
- **Wayland 不支持。** 上游文档指出 Wayland 因缺少 accessibility / UI automation / screen capture 接口而极难实现；当前产物仅支持 X11。
- 需要 **AT-SPI 辅助功能服务**在运行，否则屏幕上下文感知功能受限。
- 首次启动需配置模型 API（OpenAI / Claude / DeepSeek / 本地 Ollama 等任选）。

---

## 📖 构建记录（完整 handoff）

以下是从零开始完成本次构建的**全部步骤、踩过的坑与最终解法**。

### 背景

初始需求是「分析 `sylinko/everywhere` 的 Linux 兼容性并在本机尝试运行」。官方 README 将 Linux 标为 🚧 *In Development*，`docs/build.md` 标注 `not in active development yet`。

在 `main` 分支上构建失败，排查后发现 **Linux 目标代码停留在旧架构、从未跟随上游重构**，当前无法编译。转而验证 `dev/linux` 分支后成功。

### 问题链（按时间顺序）

#### ❶ `.NET` SDK 版本选择

- `global.json` 在 `main` 上钉死 `10.0.401`（`allowPrerelease: false`，**无** `rollForward` → 走默认 `latestPatch` 策略，同一 feature band 内必须 ≥ 401）。
- ⚠️ `docs/build.md` 中写的 `10.202` **不是合法的 SDK 版本号**，属文档滞后／笔误。**以 `global.json` 为准。**
- `dev/linux` 分支的 `global.json` 则是 `10.0.100` + `rollForward: latestMajor`，约束宽松得多。

#### ❷ `PartialChain` — TLS 证书链不完整

**现象**

```
未能从 "https://nuget.sylinko.com/v3-flatcontainer/porta.pty/999.20261001.1/porta.pty.999.20261001.1.nupkg" 下载包
The SSL connection could not be established...
The remote certificate is invalid because of errors in the certificate chain: PartialChain
```

**根因**

`nuget.sylinko.com`（Cloudflare 代理）发送的证书链不完整：

```
sylinko.com（叶证书）
  └─ WE1（中间证书）
       └─ GTS Root R4【交叉签名版】 ← 服务器只发到这里
            └─ GlobalSign Root CA   ← 缺失，链无法终止
```

服务器发出的是 GTS Root R4 的**交叉签名版**（SHA256 指纹 `76:B2:7B:80:...`），而系统信任锚是**自签名版**（指纹 `34:9D:FA:40:...`）—— 同名不同指纹，导致链构建失败。

**解法**（按成本从低到高）：

```bash
# 首选：刷新系统 CA 存储
sudo apt update && sudo apt install --reinstall ca-certificates
sudo update-ca-certificates --fresh
```

若仍失败，可将自签名版 GTS Root R4 装成信任锚：

```bash
sudo curl -fsSL https://pki.goog/repo/certs/gtsr4.pem -o /usr/local/share/ca-certificates/gts-root-r4.crt
sudo update-ca-certificates
```

> ❌ **不要**使用 `-k` 或关闭证书校验 —— 那等于对整条供应链裸奔。

#### ❸ `Connection refused (github.com:443)` — 私有源其实是跳转器

**现象**：修复 TLS 后报错**发生变化**：

```
未能从 "https://nuget.sylinko.com/v3-flatcontainer/.../xxx.nupkg" 下载包
Connection refused (github.com:443)
```

**根因** — 关键发现：

`nuget.sylinko.com` **并非真正的 NuGet 源，而是一个 HTTP 307 跳转器**：

```
nuget.sylinko.com/xxx.nupkg
  → 307 → github.com/Sylinko/.../releases/download/...
  → 302 → release-assets.githubusercontent.com/...
  → 200 → 真正的 nupkg 字节
```

包实际托管在 **GitHub Releases**。国内网络环境下 `github.com:443` 不可达，链路断在第二跳。

**解法（`main` 分支专用）** — 本地文件夹源：

1. 经 GitHub 加速镜像抓取 nupkg（实测可用：`gh-proxy.com` / `ghfast.top` / `ghproxy.net`）
2. 配置本地文件夹源 + `packageSourceMapping`（**按包名匹配**，必须复制上游的包名映射段，否则包找不到）

涉及的 9 个私有包（`main` 分支，`Directory.Packages.props`）：

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

> 💡 **关键转折**：切到 `dev/linux` 分支后，**这 9 个私有包根本不需要** —— 该分支的 `nuget.config` 只有 `nuget.org`，私有包体系是 `main` 后来才引入的。**所有 NuGet 绕行方案在 `dev/linux` 上都不必要。**

#### ❹ 53 个 `CS0246` 编译错误 — `main` 分支的 Linux 代码是孤儿

**现象**

```
Everywhere.Linux net10.0 linux-x64 失败，出现 53 错误
error CS0246: 未能找到类型或命名空间名 "IVisualElement"
error CS0246: 未能找到类型或命名空间名 "IVisualElementContext"
error CS0246: 未能找到类型或命名空间名 "VisualElementSiblingAccessor"
```

**根因**

这三个类型在 2026-09-30 合并的 **Visual Context 重构**（上游 PR #509）中已被删除／重命名，但 `src/Everywhere.Linux` **未跟随迁移**。全仓库检索确认它们**只有引用、没有定义**。

上游 `docs/Automation/07-Migration.md` 记录了迁移映射：

| 旧（Linux 仍在引用） | 新目标 |
|---|---|
| `Everywhere.Interop.IVisualElement` | `Everywhere.Automation.VisualElement` |
| `VisualElementSiblingAccessor` | `VisualElement.CreateEnumerator` |
| `IVisualElementContext` | Automation Host `IVisualElementBackend` + Main 侧服务 |

同一文档亦明确写道：

> Linux `IVisualElementContext` / `VisualElementContext` — *retained only as a platform migration source until native Context work*

**结构性缺陷**：`Everywhere.Linux.csproj` **未引用** `Everywhere.Automation`（替换类型所在项目），仅做类型重命名无法修复。

受影响文件（均在 `src/Everywhere.Linux/Interop/`）：
`VisualElementContext.cs`、`VisualElementContext.Picker.cs`、`VisualElementContext.Screenshot.cs`、`X11WindowBackend.cs`、`X11Backend/X11VisualElement.cs`、`IWindowBackend.cs`、`AtspiService.cs`

#### ❺ `dev/linux` 分支 — 可编译，但停滞

对 `dev/linux` 与 `main` 的实测对比：

| 检查项 | 结果 |
|--------|------|
| `dev/linux` 是否已合并到 `main` | ❌ **从未合并**（`compare` 状态 = `diverged`；无任何 `dev/linux → main` 的 PR） |
| 提交差距 | main **ahead 933** / dev/linux **ahead 13** |
| 最后提交 | **2026-02-05**（停滞 8 个月） |
| 分支 CI 运行次数 | **0** |
| `main` 上提及 `dev/linux` 的 merge commit | **0 条** |

> 💡 **`dev/linux` 并非废弃的垃圾分支** —— 其独有的 13 个提交中包含**实质性的重构成果**：
>
> ```
> 53c58bfe3  refactor: complete decomposition of at-api2 service for better modularity
> ```
>
> 将单体 `AtspiService.cs`（1054 行）**拆分为 13 个职责清晰的模块**：
>
> ```
> src/Everywhere.Linux/Interop/AtspiService/
> ├── AtspiService.cs                (240)
> ├── Core/AtspiCacheManager.cs      (75)
> ├── Core/AtspiContext.cs           (95)
> ├── Core/AtspiEventhub.cs          (252)
> ├── Elements/AtspiElement.cs       (646)
> ├── Elements/AtspiElement.Actions.cs (223)
> ├── Elements/ElementFactory.cs     (245)
> ├── Native/AtspiNative.cs          (190)
> ├── Native/AtspiEnums.cs           (134)
> └── Native/SafeAtspiHandle.cs      (55)
> ```
>
> 这份拆分**至今未进入 `main`**，是未来迁移到新架构的现成起点。

### ✅ 最终可行路径

| 分支 | 解决方案文件 | 私有源 | SDK | 结果 |
|------|-------------|--------|-----|------|
| `main` | `Everywhere.Linux.slnx` | ✅ 需要 9 个私有包 | 10.0.401（严格） | ❌ 53 个 CS0246 |
| **`dev/linux`** | **`Everywhere.Linux.slnf`** | ❌ **只需 nuget.org** | 10.0.100（宽松） | ✅ **构建成功** |

> ⚠️ **`.slnx` vs `.slnf`**：`main` 使用新版 solution 格式 `.slnx`；`dev/linux` 使用 `.slnf`（solution filter，配合 `Everywhere.sln`）。在 `dev/linux` 上用 `.slnx` 会报 `MSB1009: Project file does not exist`。

### 完整复现命令

```bash
# 1. 克隆（含子模块，dev/linux 的子模块在 3rd/ 目录）
git clone --recursive https://github.com/Sylinko/Everywhere.git
cd Everywhere
git checkout dev/linux

# 2. 安装 .NET SDK（dev/linux 要求 10.0.100，rollForward=latestMajor）
curl -fsSL https://dot.net/v1/dotnet-install.sh | bash -s -- --version 10.0.100 --install-dir "$HOME/.dotnet"
export DOTNET_ROOT="$HOME/.dotnet"
export PATH="$PATH:$HOME/.dotnet"
dotnet --version     # 期望 10.0.100

# 3. 还原（只需公共源，无需任何私有源配置）
dotnet restore Everywhere.Linux.slnf -r linux-x64

# 4. 构建
dotnet publish src/Everywhere.Linux/Everywhere.Linux.csproj \
  -c Release -r linux-x64 -o ./publish \
  --self-contained true --no-restore

# 5. 运行（必须在 X11 会话内）
./publish/Everywhere
```

### 构建环境记录

| 项 | 值 |
|----|----|
| 系统 | Ubuntu Server 24.04 x86_64 |
| 内存 | 1.6 GB（构建耗时 4:23，峰值接近上限） |
| SDK | 10.0.100（`~/.dotnet`） |
| 分支 | `dev/linux` @ `c3a67026` |
| 还原耗时 | ~2.3 min（首次，含 6 个子模块 clone） |
| 构建结果 | **0 errors**, 4 warnings |
| 运行时依赖 | `libX11.so.6` / `libatspi.so.0` / `libglib-2.0.so.0` / `libfontconfig.so.1` |
| 产物 | 163 MB，self-contained（含 `libcoreclr.so`） |

---

## 📌 上游 issue 与后续方向

### 已提交 issue

**[Sylinko/Everywhere#520](https://github.com/Sylinko/Everywhere/issues/520)** — `[Feat] Linux: is Everywhere.Linux still intended to be maintained? (does not compile on main)`

维护者 **DearVa** 已回复，态度积极：

- 承认 Linux 代码主要来自社区贡献者（`@jimfengnpu`），此前每次重构都会尽量保证 Linux 可编译，本次因重构规模无法用简单 patch 解决
- Linux 支持已列入 **future TODO list**
- **明确欢迎贡献**并承诺提供指导
- 指出两个挑战：**Wayland 移植难度**、**代码质量标准高**（深度 review + 虚拟机测试 + 可能自行改写，review 耗时长）

### 上游 Linux 支持现状（实测）

| 检查项 | 结果 |
|--------|------|
| `(Linux) Build and Release` CI | **`disabled_manually`**，历史运行 **0 次** |
| 所有 release 的 Linux 产物 | ❌ **从未有过**（如 `v0.9.0-canary.20261010.25` 仅 Windows + macOS） |
| `dev/linux` 分支 | 存在但停滞 8 个月，落后 `main` 933 提交，自身 CI 0 次运行 |

### 建议的后续贡献路径

| 阶段 | 内容 | 工作量 |
|------|------|--------|
| **C** | **Wayland 可行性调研** —— 回答维护者自己提出的开放问题：at-spi2 在 Wayland 下的现状、`xdg-desktop-portal` 的 `ScreenCast`/`RemoteDesktop` 能否补足屏幕捕获与输入、Wayland 协议禁止全局窗口枚举如何规避 | 1–2 天，无需跑通编译 |
| **A** | **窄切片 PR** —— 仅实现 `Query(Default + TopLevel)`，打通 `list_windows` 全链路（Backend → Host Session → RPC → Main → Agent），先摸清 review 标准 | 约 500–800 行，1–2 周 |
| **B** | **完整迁移** —— 完整 `LinuxVisualElementBackend` + AT-SPI 元素模型，移植 `dev/linux` 的 AT-SPI 模块化拆分 | 数周+ |

**关键技术点**

- Linux 侧需实现的新接口：
  ```csharp
  public interface IVisualElementBackend : IDisposable {
      VisualElementQueryResult? Query(VisualElementRetention retention,
          VisualElementLocator locator, VisualElementResolution resolution = Direct,
          VisualElementQueryRequest? request = null);
  }
  ```
- **可复用**：X11 原生层（`X11Native.cs` / `X11WindowManager.cs` / `X11InputHandler.cs` 等）、`Everywhere.Automation`（~140 KB）、`AutomationHostSession`（已在 `Everywhere.Core`）
- **需重写**：`AtspiService` 相关（AT-SPI 无障碍树 → 新的 `VisualElement` 契约）
- ⭐ **参考模板是 macOS 而非 Windows** —— 上游文档明确写 macOS *"starts its Hosts directly rather than exposing Windows service mode"*

---

## 📁 仓库内容

```
.
├── README.md          # 本文件
├── LICENSE            # 上游 BSL-1.1 原文
├── NOTICE             # 来源与构建方式声明
├── dist/              # 构建产物（git-lfs 追踪）
└── build-logs/        # 构建日志留档
```

---

## 📄 许可证与归属

本仓库内容遵循上游 [Sylinko/Everywhere](https://github.com/Sylinko/Everywhere) 的 **Business Source License 1.1**（见 `LICENSE`）：

- Licensor：Sylinko Inc.
- Change Date：首次公开发布后 4 年
- Change License：Apache License, Version 2.0
- Additional Use Grant：允许内部使用、非商业教育与研究；**禁止 Competing Use**

本仓库为**非官方构建**，仅保存自行编译的二进制产物与构建记录，**不构成官方分发渠道**。版权归 Sylinko Inc. 所有。
