# Linux 对齐最新架构 — 开发执行方案

> 目标：把 `src/Everywhere.Linux` 从旧架构迁移到 `main` 的新 Automation 架构，产出可提交给上游的 PR。
> 上游仓库：`Sylinko/Everywhere`（分支 `main`）
> 相关 issue：https://github.com/Sylinko/Everywhere/issues/520
>
> **本文档是给执行 agent 的实施指南。所有 API 签名均已在 main 最新 commit（`66f755f8`）核实。**

---

## 0. 现状基线（已核实）

| 项 | 值 |
|----|----|
| main HEAD | `66f755f8`（2026-10-22 02:12） |
| `dev/linux` HEAD | `c3a67026`（2026-02-06） |
| 差距 | main 领先 **934** 提交，dev/linux 独有 **13** |
| main 上 Linux 编译 | ❌ 53 个 CS0246 |
| dev/linux 编译 | ✅ 0 错误 |

### 三个已核实的关键事实

1. **`AutomationHostSession` 是跨平台共享的**，位于 `src/Everywhere.Core/ProcessIsolation/Automation/`，
   构造函数签名：
   ```csharp
   public AutomationHostSession(IVisualElementBackend backend)
   public AutomationHostSession(
       IVisualElementBackend backend,
       IVisualPickerResolver pickerResolver,
       ITextSelectionMonitorFactory textSelectionMonitorFactory)
   ```
   → **Linux 不需要重写 Host 逻辑**，只需实现 `IVisualElementBackend`。

2. **main 上已有 `src/Everywhere.Linux/Interop/X11Backend/X11VisualElement.cs`（5.9 KB）**，
   X11 原生层共 10 个文件（`X11Native.cs` 14.9 KB、`X11WindowManager.cs` 11.8 KB、
   `X11InputHandler.cs` 9.1 KB、`X11Context.cs` 10.1 KB 等）→ **X11 层可复用**。

3. **`ProcessRole` 枚举有三个值**：`Main` / `Input` / `Automation`。
   Windows 的 `Program.cs` 已实现三分支派发，Linux 的 `Program.cs` 目前**完全没有**这段。

---

## 1. 架构目标

对齐 Windows/macOS，Linux 需要提供三个进程角色：

```
Main 进程    → Avalonia UI、聊天、策略引擎
Input 进程   → 全局键鼠监听（X11InputHandler）
Automation 进程 → 视觉树自动化（AT-SPI）← 本次核心
```

Automation Host 内部结构（Windows 参考）：

```
ProcessRoleHostRunner.RunAsync(ProcessRole.Automation, args, peerVerifier, CreateAutomationHostSession)
  └── AutomationHostSession(
        new WindowsVisualElementBackend(),     // ← Linux 要换成 LinuxVisualElementBackend
        new WindowsVisualPickerResolver(),
        new WindowsTextSelectionMonitorFactory())
```

Linux 侧对应：

```
ProcessRoleHostRunner.RunAsync(ProcessRole.Automation, args, peerVerifier, CreateAutomationHostSession)
  └── AutomationHostSession(
        new LinuxVisualElementBackend(),       // ← 新建
        new LinuxVisualPickerResolver(),       // ← 新建（可选，见 §5）
        new LinuxTextSelectionMonitorFactory())// ← 新建（可选，见 §5）
```

---

## 2. 阶段划分

### Phase 0 — CI 与打包修复 cherry-pick（先做，零风险）

dev/linux 的 13 个独有提交里，**8 个与架构重构无关**，可直接 cherry-pick 到 main：

```bash
git checkout -b feat/linux-modernize origin/main

git cherry-pick 1049887b   # fix: hicolor icon resolution in build-appimage.sh
git cherry-pick 33e8ea3a    # build(Linux): standardize deb packing
git cherry-pick b933ea95    # Update linux-release.yml
git cherry-pick bf780b86    # ci: add tar.zst archive packaging
git cherry-pick 17761c82    # ci: fix dotnet publish problem in ci & appimage builder
git cherry-pick 984966f3    # fix: restore before publish
git cherry-pick 3bd19778    # fix: clean before appimage building
git cherry-pick c2536e27    # ci: fix dotnet publish problem in ci & appimage builder
```

**验收标准**：`dotnet build` 在 Linux 上成功；`.github/workflows/linux-release.yml` 能跑通。

**注意**：`linux-release.yml` 在 main 上状态是 `disabled_manually`，需确认是否需要重新启用——
**这一项必须先问维护者**，不要自行启用。

---

### Phase 1 — 让 Linux 编译通过（目标：0 error）

**背景**：main 上 `src/Everywhere.Linux` 引用了三个已被 Visual Context 重构删除的类型：

```
CS0246: IVisualElement
CS0246: IVisualElementContext
CS0246: VisualElementSiblingAccessor
```

上游 `docs/Automation/07-Migration.md` 记录了映射：

| 旧（Linux 仍在用） | 新 |
|---|---|
| `Everywhere.Interop.IVisualElement` | `Everywhere.Automation.VisualElement` |
| `IVisualElementContext` | `IVisualElementBackend` + `AutomationHostSession` |
| `VisualElementSiblingAccessor` | `VisualElement.CreateEnumerator()` |

**步骤**：

1. `src/Everywhere.Linux/Everywhere.Linux.csproj` 增加：
   ```xml
   <ProjectReference Include="..\Everywhere.Automation\Everywhere.Automation.csproj" />
   ```

2. 删除旧契约文件（这些类型在新架构下已无对应物）：
   ```
   src/Everywhere.Linux/Interop/VisualElementContext.cs
   src/Everywhere.Linux/Interop/VisualElementContext.Picker.cs
   src/Everywhere.Linux/Interop/VisualElementContext.Screenshot.cs
   ```
   ⚠️ 删除前先确认 `ScreenSelectionSession.cs`（6.7 KB）是否被引用 —— 它对应 macOS 的
   `MacScreenSelectionService`，**可能需要保留并改名**。

3. `Interop/AtspiService.cs`（36.8 KB，单体）—— 本阶段先**保留**，只做最小改动让它编译：
   - 旧 `IVisualElement` 相关方法签名降级/暂时移除
   - `X11WindowBackend.cs` 中实现旧接口的部分暂时注释

4. `Interop/IWindowBackend.cs`（1.1 KB）—— 检查是否还需实现旧接口。

**验收标准**：
```bash
dotnet build src/Everywhere.Linux/Everywhere.Linux.csproj -c Release -r linux-x64
# → 0 Error(s)
```

**⚠️ 本阶段结束时必须停下来，向维护者发一条进度同步再继续。**
理由：上游 review 标准高（维护者明说会深度 review），中途同步可避免大方向跑偏。

---

### Phase 2 — AT-SPI 模块化移植

**目标资产**：`dev/linux` 的 `53c58bfe` 提交把 36.8 KB 单体 `AtspiService.cs` 拆成 10 个模块：

```
src/Everywhere.Linux/Interop/AtspiService/
├── AtspiService.cs                        (240 B)
├── Core/AtspiCacheManager.cs              (75 B)
├── Core/AtspiContext.cs                   (95 B)
├── Core/AtspiEventhub.cs                  (252 B)
├── Elements/AtspiElement.cs               (646 B)
├── Elements/AtspiElement.Actions.cs       (223 B)
├── Elements/ElementFactory.cs             (245 B)
├── Native/AtspiNative.cs                  (190 B)
├── Native/AtspiEnums.cs                   (134 B)
└── Native/SafeAtspiHandle.cs              (55 B)
```

**移植方式**（不要 merge 整个分支）：
```bash
git checkout origin/dev/linux -- src/Everywhere.Linux/Interop/AtspiService/
git checkout origin/dev/linux -- src/Everywhere.Linux/Interop/X11Backend/
```
再手工解决冲突。

**分层判断**：

| 模块 | 是否直接可用 | 说明 |
|---|---|---|
| `Native/AtspiNative.cs` | ✅ 直接用 | 纯 P/Invoke 封装 |
| `Native/AtspiEnums.cs` | ✅ 直接用 | 纯枚举 |
| `Native/SafeAtspiHandle.cs` | ✅ 直接用 | 纯句柄管理 |
| `Core/AtspiCacheManager.cs` | ✅ 直接用 | 与契约无关 |
| `Core/AtspiEventhub.cs` | ✅ 直接用 | 事件订阅，与契约无关 |
| `Elements/AtspiElement.cs` | ⚠️ **需重写** | 旧 `IVisualElement` 实现 → 需改为新 `VisualElement` 子类 |
| `Elements/ElementFactory.cs` | ⚠️ **需重写** | 工厂需接入 `VisualElementIdentityMap` |
| `Core/AtspiContext.cs` | ⚠️ 需适配 | 对应新 `VisualContext` |

---

### Phase 3 — 实现 `LinuxVisualElementBackend`（核心）

**新建文件**：`src/Everywhere.Linux/Automation/LinuxVisualElementBackend.cs`

**必须实现的接口**（`Everywhere.Automation/IVisualElementBackend.cs`）：

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

**契约要点**（XML doc 明确要求）：
- 实现可以持有共享的原生客户端
- **方法返回后不得保留 context / retention / element 引用**
- `retention` 决定返回元素的归属批次

#### 3.1 需要支持的四种 locator

`VisualElementLocator`（`VisualElementQueryContracts.cs`）：

| Locator | 工厂方法 | 语义 | Linux 实现 |
|---|---|---|---|
| `Default` | `VisualElementLocator.Default` | 无锚点 | AT-SPI 根遍历 |
| `Focused` | `VisualElementLocator.Focused` | 焦点元素 | `atspi_get_focused_element` |
| `Pointer` | `VisualElementLocator.Pointer` | 指针位置 | X11 `XQueryPointer` |
| 坐标点 | `VisualElementLocator.FromPoint(PixelPoint)` | 指定坐标 | X11 root → AT-SPI 命中测试 |
| 原生窗口 | `VisualElementLocator.FromNativeWindow(nint)` | X11 window id | AT-SPI frame 查找 |

#### 3.2 三种 resolution

| Resolution | macOS 参考实现 | Linux 对应 |
|---|---|---|
| `Direct` | `QueryDirect` | AT-SPI element 直接取 |
| `TopLevel` | `QueryTopLevel` | 沿 parent 链上溯到 window frame |
| `Screen` | `QueryScreen` | X11 屏幕 + `ScreenSelectionSession` |

> **建议切片顺序**：先只实现 `TopLevel`（对应 macOS 的 `TopLevelWindowEnumerator.cs` 7.2 KB
> / Windows 的 `TopLevelWindowEnumerator.cs` 5.8 KB），跑通后再补 `Direct` 和 `Screen`。
> 切片小 → review 快 → 更容易被接受。

#### 3.3 `VisualElement` 子类必须实现的抽象成员

`Everywhere.Automation/VisualElement.cs`（17.2 KB）：

```csharp
public abstract class VisualElement
{
    protected VisualElement(VisualElementIdentity identity, string id);

    // 必须实现：
    protected abstract VisualElementQueryResult QueryCore(VisualElementQueryRequest request);
    protected abstract IVisualElementCursor CreateEnumeratorCore(...);
    protected abstract Task<IVisualElementCapture> CaptureCoreAsync(CancellationToken ct);
    protected abstract void ReleaseCore();

    // 可选覆写（有默认实现）：
    protected virtual VisualElementTextReadResult ReadTextCore(int offset, int maxCharacters, int maximumProbeCharacters);
    protected virtual void InvokeCore();
    protected virtual void SetTextCore(string text);
    protected virtual void FocusCore();
    protected virtual void SendKeyGestureCore(KeyGesture keyGesture);
    protected virtual string? GetSelectedTextCore(int maxCharacters);
    protected virtual VisualElement AdoptCore(VisualElementRetention destinationRetention);
    protected virtual bool TryConvertPlatformException(Exception ex, out Exception? converted);
}
```

**元素身份管理**（macOS 的做法，见 `MacVisualElementBackend.GetOrCreateAXElement`）：

```csharp
internal AXVisualElement GetOrCreateAXElement(VisualElementRetention retention, AXUIElement nativeElement)
{
    var context = retention.Context;
    var identity = new AXUIElementIdentity(nativeElement.NativeHandle);
    return context.GetIdentityMap(AXUIElementIdentityComparer.Shared).GetOrAdd(
        retention,
        identity,
        (Backend: this, NativeElement: nativeElement, IsSystemWide: ...),
        static (elementIdentity, state) => AXVisualElement.Create(...));
}
```

Linux 版对应：`AtspiElementIdentity`（用 AT-SPI 指针/句柄做 key）
+ `VisualElementIdentityMap<AtspiElementIdentity>`。

#### 3.4 `VisualElementQueryRequest` 默认值

```csharp
public static VisualElementQueryRequest Default => new(VisualElementFields.All, 4_096);
```
→ **文本截断上限 4096 字符**，实现时不要超过。

---

### Phase 4 — 进程角色派发

**参照 Windows `Program.cs`**（`src/Everywhere.Windows/Program.cs`）：

```csharp
var role = ProcessRoleCommandLine.Parse(args);
if (role is not ProcessRole.Main)
{
    return await (role is ProcessRole.Input ?
        ProcessRoleHostRunner.RunAsync(role, args, peerVerifier,
            static () => new WindowsInputHostSession()) :
        RunAutomationHostAsync(args, peerVerifier)).ConfigureAwait(false);
}

private static Task<int> RunAutomationHostAsync(string[] args, INamedPipePeerVerifier peerVerifier) =>
    Task.Run(async () =>
    {
        // Windows: 断言 MTA 线程
        // Linux:   断言主线程（X11 需在主线程初始化）
        return await ProcessRoleHostRunner
            .RunAsync(ProcessRole.Automation, args, peerVerifier, CreateAutomationHostSession)
            .ConfigureAwait(false);
    });

private static IProcessRoleSession CreateAutomationHostSession() =>
    new AutomationHostSession(
        new LinuxVisualElementBackend(),
        new LinuxVisualPickerResolver(),
        new LinuxTextSelectionMonitorFactory());
```

**Linux 侧需要**：
- `src/Everywhere.Linux/ProcessIsolation/LinuxNamedPipePeerVerifier.cs` —— **已存在**（2.4 KB）✅
- Input Host：Linux 需提供 `IProcessRoleSession` 实现（参考 `X11InputHandler.cs` 9.1 KB）

**注意线程模型差异**：
- Windows Automation Host 要求 **MTA 线程**
- macOS 要求 **主线程**（AppKit 限制）
- **Linux 需实测**：AT-SPI 的 dbus 调用在哪个线程安全。建议先在主线程试，若 X11 初始化冲突再调整。
  这是**需要实验确认**的点，不要假设。

---

## 3. 提交策略

### 分支与 PR

```
分支名：feat/linux-automation-backend
基础：origin/main
```

**建议拆成多个小 PR**，而不是一个巨型 PR：

| PR | 内容 | 依赖 |
|---|------|------|
| PR-1 | Phase 0：CI/打包修复 cherry-pick | 无 |
| PR-2 | Phase 1：编译修复（0 error） | PR-1 |
| PR-3 | Phase 2+3：AT-SPI 后端（`TopLevel` 切片） | PR-2 |

理由：维护者明说 review 耗时长、会深度审查甚至自行改写。**小 PR 是降低被拒风险的唯一手段。**

### 提交信息规范

参考仓库现有风格（`type(scope): description`）：
```
feat(linux): implement LinuxVisualElementBackend with AT-SPI top-level query
fix(linux): restore Linux build by migrating to Automation contracts
build(linux): standardize deb packing
```

### PR 描述模板

上游 issue 模板要求（见 `.github/ISSUE_TEMPLATE/feature-request.yml`）：
- 勾选「已搜索重复 issue」
- 填 `App Version`
- 英文撰写

PR 描述应包含：
1. 动机（对应 issue #520）
2. 改动范围
3. **测试环境**：Ubuntu 版本、GNOME 版本、X11、AT-SPI 状态
4. **已知限制**（明确写出不支持的部分）
5. 引用 `docs/Automation/07-Migration.md` 的迁移映射

---

## 4. 沟通节奏

**维护者 DearVa 已明确表示欢迎贡献并承诺指导**（issue #520 回复）。

| 时机 | 动作 |
|------|------|
| Phase 0 完成后 | 发 issue #520 评论，说明 CI/打包修复已就绪，询问是否直接开 PR |
| Phase 1 完成后 | **必须**发评论同步「已 0 error，但功能是 stub」，附上编译日志 |
| Phase 2 开始前 | 询问 AT-SPI 模块化拆分的移植方式是否合适（是直接用还是他另有安排） |
| Phase 3 提交前 | 告知切片策略（先 TopLevel），降低他 review 负担 |

> ⚠️ **不要一次性提交全部改动。** 上游 934 提交的漂移速度很快，PR 越大越难 rebase。

---

## 5. 可选组件（按工作量决定是否纳入首个 PR）

`AutomationHostSession` 的构造函数有三个重载。若只提供 backend：

```csharp
public AutomationHostSession(IVisualElementBackend backend)
```
则 Picker 和 TextSelection 功能为空。

| 组件 | macOS 参考 | 大小 | 建议 |
|------|----------|------|------|
| `IVisualPickerResolver` | `MacVisualPickerResolver.cs` | 2.1 KB | ⭐ 第二个切片 |
| `ITextSelectionMonitorFactory` | macOS 无独立文件 | — | ⭐ 第三个切片（`AutomationTextSelectionWatcher.cs` 28.7 KB，共享库里） |

**首个 PR 只做 backend**，用 `AutomationHostSession(backend)` 单参构造。

---

## 6. 验收清单

### 每个阶段结束必须验证

```bash
# 编译
dotnet build src/Everywhere.Linux/Everywhere.Linux.csproj -c Release -r linux-x64

# 发布
dotnet publish src/Everywhere.Linux/Everywhere.Linux.csproj \
  -c Release -r linux-x64 -o ./publish --self-contained true

# 运行时依赖
ldd publish/Everywhere | grep "not found"    # 应无输出
```

### 首个 PR 的完整验收

- [ ] `dotnet build` 0 error
- [ ] `dotnet publish` 成功，产物含 `libcoreclr.so`
- [ ] 在真实 X11 会话启动，`Ctrl+Shift+E` 能唤起窗口
- [ ] AT-SPI 能读到当前活动窗口标题（用一个最小测试程序验证）
- [ ] `docs/Automation/07-Migration.md` 中所有映射项均已处理
- [ ] PR 描述含测试环境 + 已知限制
- [ ] 已发 issue #520 评论同步进度

---

## 7. 关键参考文件索引

| 用途 | 路径 |
|------|------|
| 核心接口 | `src/Everywhere.Automation/IVisualElementBackend.cs` |
| 查询契约 | `src/Everywhere.Automation/VisualElementQueryContracts.cs` (11.2 KB) |
| 元素基类 | `src/Everywhere.Automation/VisualElement.cs` (17.2 KB) |
| 身份映射 | `src/Everywhere.Automation/VisualElementIdentityMap.cs` (7.4 KB) |
| 保留批次 | `src/Everywhere.Automation/VisualElementRetention.cs` (2.0 KB) |
| 共享 Host | `src/Everywhere.Core/ProcessIsolation/Automation/AutomationHostSession.cs` (23.7 KB) |
| Host 启动器 | `src/Everywhere.ProcessIsolation/Hosting/ProcessRoleHostRunner.cs` (16.2 KB) |
| 角色枚举 | `src/Everywhere.ProcessIsolation/Roles/ProcessRole.cs` |
| **macOS 参照** | `src/Everywhere.Mac/Automation/MacVisualElementBackend.cs` (13.7 KB) |
| macOS 角色派发 | `src/Everywhere.Mac/Program.cs` |
| Windows 参照 | `src/Everywhere.Windows/Automation/WindowsVisualElementBackend.cs` (10.7 KB) |
| Windows 角色派发 | `src/Everywhere.Windows/Program.cs` |
| X11 可复用层 | `src/Everywhere.Linux/Interop/X11Backend/` (10 文件) |
| AT-SPI 拆分源 | `dev/linux` 分支 `src/Everywhere.Linux/Interop/AtspiService/` (10 文件) |
| 迁移映射文档 | `docs/Automation/07-Migration.md` |
| 架构文档 | `docs/Automation/02-Architecture.md` |

---

## 8. 反模式（务必避免）

| ❌ 不要 | ✅ 应该 |
|---|---|
| merge `dev/linux` 到 main | 从 main 开新分支，按需 cherry-pick |
| 一次性提交全部改动 | 拆成 PR-1 / PR-2 / PR-3 |
| 自己启用被禁用的 CI workflow | 先问维护者 |
| 自行决定 AT-SPI 拆分是否可用 | 先问，他可能另有安排 |
| 关闭 TLS 校验 / 用 `-k` | 修 CA 证书链 |
| 用 `--ignore-failed-sources` 掩盖失败 | 正常解决源问题 |
| 在 PR 里改无关文件 | 严格限定范围 |

---

## 9. 参考：main 上 Linux 的现状清单

```
src/Everywhere.Linux/
├── Chat/                        ← 已有
├── Common/LinuxUpdateHandler.cs (3.4 KB)
├── I18N/
├── Interop/
│   ├── AtspiService.cs           (36.8 KB)  ← 旧单体，Phase 2 拆
│   ├── IEventHelper.cs           (1.0 KB)
│   ├── IWindowBackend.cs         (1.1 KB)
│   ├── LinuxNativeMessageBox.cs  (8.5 KB)
│   ├── NativeHelper.cs           (2.9 KB)
│   ├── ScreenSelectionSession.cs (6.7 KB)  ← 对应 macOS ScreenSelectionService
│   ├── ShortcutListener.cs       (4.3 KB)
│   ├── VisualElementContext.cs        (4.2 KB) ← 删除
│   ├── VisualElementContext.Picker.cs (2.7 KB) ← 删除
│   ├── VisualElementContext.Screenshot.cs (6.8 KB) ← 删除
│   ├── X11WindowBackend.cs       (7.8 KB)
│   └── X11Backend/               ← 10 文件，可复用
│       ├── X11AtomCache.cs       (1.4 KB)
│       ├── X11CapturedBitmapData.cs (3.6 KB)
│       ├── X11Context.cs         (10.1 KB)
│       ├── X11CoreServices.cs    (2.6 KB)
│       ├── X11InputHandler.cs    (9.1 KB)
│       ├── X11Native.cs          (14.9 KB)
│       ├── X11Screenshot.cs      (2.5 KB)
│       ├── X11SelectionHandler.cs (6.6 KB)
│       ├── X11VisualElement.cs   (5.9 KB)  ← 新架构已有部分实现
│       └── X11WindowManager.cs   (11.8 KB)
├── ProcessIsolation/
│   └── LinuxNamedPipePeerVerifier.cs (2.4 KB) ← 已有 ✅
├── Everywhere.Linux.csproj
├── Global.cs
├── PKGBUILD
├── Program.cs                    ← Phase 4 加角色派发
├── build-appimage.sh
└── packaging_deb.sh
```

**需要新建的**（Phase 3/4）：
```
src/Everywhere.Linux/Automation/LinuxVisualElementBackend.cs
src/Everywhere.Linux/Automation/AtspiVisualElement.cs
src/Everywhere.Linux/Automation/AtspiElementIdentity.cs
src/Everywhere.Linux/Automation/LinuxVisualPickerResolver.cs      (可选)
src/Everywhere.Linux/Automation/LinuxTextSelectionMonitorFactory.cs (可选)
src/Everywhere.Linux/ProcessIsolation/LinuxInputHostSession.cs     (Phase 4)
```
