构建环境
========

系统      : Ubuntu Server 24.04 x86_64
内核      : Linux 6.8.0-124-generic
内存      : 1.6 GB total / 约 950 MB available（构建峰值接近上限）
磁盘      : 40 GB
.NET SDK  : 10.0.100（安装于 ~/.dotnet，rollForward=latestMajor）
源码分支  : dev/linux @ c3a67026218eba3d7c47674c8884f97a1e919c32
子模块    : 6 个（3rd/ 目录下）
           - WritableJsonConfiguration  @ feceb807eca06df78ca97b1f1c21262f046a3208
           - shad-ui                   @ f096c16202f645c2e73db340a46090fdaf3a9097
           - MessagePack-CSharp        @ 41869d099939881d9a8bf05eeef02018d73a9b6c
           - EverythingNetCore         @ 35e4a8874d50598fe6bb488177e115bc5c476822
           - semantic-kernel           @ dc7c1c048488a8b611b5382d0d7efe05608a9fa0
           - anthropic-sdk-csharp      @ f3254936a99bcd726f82fe8b28cd921206b6c726

注意：服务器为无头环境（XDG_SESSION_TYPE=tty，无 DISPLAY），
      因此仅完成构建，未实际运行 GUI。


步骤与耗时
==========

1. 克隆仓库（--no-recurse-submodules 快速版）        ~40s
2. git fetch origin dev/linux:dev/linux            ~20s
3. 切换到 dev/linux，验证 SDK 解析                  <1s
   $ dotnet --version
   10.0.100

4. 还原依赖（需先补齐子模块）                       ~5 min
   $ git submodule update --init --recursive
   $ dotnet restore Everywhere.Linux.slnf -r linux-x64
   Restored Everywhere.Watchdog    (in 2.21 min)
   Restored Everywhere.Linux       (in 2.3 min)
   Restored Everywhere.Tests       (in 4.51 min)
   ... 共 21 个项目，0 错误

   注意：dev/linux 使用 .slnf（solution filter）而非 .slnx。
   使用 Everywhere.Linux.slnx 会报：
     MSBUILD : error MSB1009: Project file does not exist.

5. 构建                                               4 min 23 s
   $ dotnet build src/Everywhere.Linux/Everywhere.Linux.csproj \
       -c Release -r linux-x64 --no-restore
   Build succeeded.
       4 Warning(s)
       0 Error(s)

6. 发布（self-contained）                              ~2 min
   $ dotnet publish src/Everywhere.Linux/Everywhere.Linux.csproj \
       -c Release -r linux-x64 -o ./publish \
       --self-contained true --no-restore

7. 产物
   368 个文件 / 163 MB
   自带 .NET 运行时（libcoreclr.so / libhostfxr.so / System.Private.CoreLib.dll），
   目标机器无需预装 .NET。


构建警告（4 条，均不影响产物）
================================

CS9113  Everywhere.Core/AI/CustomAssistant.cs(204,79)
        Parameter 'owner' is unread.
CS0168  Everywhere.Core/Interop/EnvironmentVariableUtilities.cs(43,26)
        The variable 'e' is declared but never used.
CS0649  Everywhere.Linux/Interop/X11Backend/Components/X11SelectionHandler.cs(18,17)
        Field 'X11SelectionHandler._xiOpcode' is never assigned to.
CA2024  Everywhere.Linux/Chat/Plugins/FdFindPlugin.cs(119,21)
        Do not use 'process.StandardOutput.EndOfStream' in an async method.


运行时依赖
==========

$ ldd dist/Everywhere   → 无缺失

必须：
  libX11.so.6        X11 窗口交互
  libatspi.so.0      AT-SPI 无障碍树（屏幕上下文感知）
其他：
  libglib-2.0.so.0   GirCore.GObject-2.0 绑定依赖
  libfontconfig.so.1 字体配置

Debian/Ubuntu:
  sudo apt install -y libx11-6 libatspi2.0-0 libglib2.0-0 libfontconfig1
Arch:
  sudo pacman -S --needed libx11 at-spi2-core glib2 fontconfig

环境变量：
  DISPLAY 必须有效；程序在非 X11 会话下会直接抛异常退出。
  AT-SPI 辅助功能服务须在运行，否则屏幕上下文感知受限。
