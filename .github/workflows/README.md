# CI 构建说明（`.github/workflows/`）

这个目录放的是 GitHub Actions 工作流。**改脚本之前请先读完本文件** —— 里面记的都是已经踩过、
已经用真实 CI 跑验证过的坑，照着写能省掉大量试错。

---

## 0. 先看这里：我该用哪个脚本

| 我想要什么 | 用哪个 | 怎么触发 |
| --- | --- | --- |
| **一个能双击运行的 exe**（最常见） | `build.yml` | 不用管，push 就自动跑；下载 `ArsenalImageMounter-GUI-net10-x64-single` 这个 artifact |
| 体积最小的版本（依赖系统自带 .NET Framework） | `build.yml` | 同上，下载 `ArsenalImageMounter-GUI-net48` |
| 自包含的文件夹版（不打包成单文件） | `build.yml` | 同上，下载 `ArsenalImageMounter-GUI-net10-x64` |
| 发版，让别人直接下 exe | `build.yml` | 打一个 `v*` tag，会自动挂到 Release 页面 |
| 命令行工具 `aim_cli`、Analyze、PowerShell 模块、NuGet 包 | `build-all.yml` | **不会自动触发**，去 Actions 页面手动 `Run workflow` |
| 跑单元测试 | `build-all.yml` | 手动触发时勾上 `run_tests` |

一句话区别：**`build.yml` = 日常自动出 GUI 成品；`build-all.yml` = 手动出全套（含 CLI / 包 / 测试）。**

---

## 1. 两个工作流，各管什么

| 文件 | 触发方式 | 产出 | 用途 |
| --- | --- | --- | --- |
| `build.yml` | `push` / `pull_request` / 打 `v*` tag / 手动 | 可视化界面成品（MountTool + GUISetup），3 个变体 | **默认工作流**。平时只管这一个。 |
| `build-all.yml` | 手动 `workflow_dispatch` / 每周日 cron | 全部可发布的托管产物（CLI、Analyze、PowerShell 模块、NuGet 包、两个 Setup） | 发版或想一次性出全套时手动跑。 |

`build.yml` 的三个变体：

| 变体 | TFM / RID | 形态 | 解压后体积 | 目标机要求 |
| --- | --- | --- | --- | --- |
| `net10-x64-single` **（默认）** | `net10.0-windows` / `win-x64` | **单文件 exe**，双击即用 | 约 115 MB（2 个 exe 各约 57 MB） | 无，自包含 |
| `net48` | `net48` / 无 RID | 文件夹（exe + 一堆 dll） | 约 13 MB | .NET Framework 4.8（Win10 1903+ 自带） |
| `net10-x64` | `net10.0-windows` / `win-x64` | 文件夹（exe + 一堆 dll） | 约 103 MB | 无，自包含 |

`net10-x64-single` 是矩阵里的**第一项**，也就是默认产物。打 `v*` tag 时 `release` job 还会把
zip 里那两个 `.exe` 单独拎出来挂到 Release 页面，这样别人可以直接下 exe，不用先解压。

---

## 2. 仓库真实结构（写脚本前必须知道的部分）

```
Managed Source/
  ArsenalImageMounter.slnx          ← 新式 slnx，不是 .sln
  Directory.Build.props             ← 全局 TFM / OutputPath / 版本号都在这里
  Arsenal.ImageMounter.csproj       ← 类库，PackageId = ArsenalRecon.ImageMounter
  Arsenal.ImageMounter.Forms.csproj ← 类库（无 Main！）
  Arsenal.ImageMounter.Native.csproj
  Arsenal.ImageMounter.PowerShell.csproj
  ArsenalImageMounterService.csproj     ← Exe，AssemblyName = aim_cli
  ArsenalImageMounterAnalyze.csproj     ← Exe
  ArsenalImageMounterCLISetup.csproj    ← Exe（WinForms）
  ArsenalImageMounterGUISetup.vbproj    ← WinExe，纯驱动安装/卸载器
  ArsenalImageMounterMountTool.vbproj   ← WinExe，真正的可视化挂载工具
  TestProject.csproj
Unmanaged Source/
  ArsenalImageMounter.sln           ← vcxproj，需要 WDK 8.1+，**不在 CI 里**
DriverSetup/
  DriverFiles.zip                   ← 2.98 MB，被内嵌进 Setup / MountTool 的 exe
Graphical applications/
  Graphical application moved.txt    ← 只有这个，没有主 GUI 程序
```

### `Directory.Build.props` 关键属性

```xml
<TargetFrameworks>net48;netstandard2.1;net10.0;net11.0</TargetFrameworks>
<OutputPath>..\bin\$(Configuration)\</OutputPath>
<FileVersion>3.13.369.0</FileVersion>
<EnableWindowsTargeting>true</EnableWindowsTargeting>
<PackageOutputPath>$(LocalNuGetPath)</PackageOutputPath>   <!-- 未定义 → 空 -->
```

各项目实际 TFM（覆盖上面的默认值）：

| 项目 | TargetFrameworks |
| --- | --- |
| `Arsenal.ImageMounter` | 继承 `net48;netstandard2.1;net10.0;net11.0` |
| `Arsenal.ImageMounter.Forms` | `net48;net11.0-windows;net10.0-windows` |
| `Arsenal.ImageMounter.PowerShell` | `net48;net10.0;net11.0` |
| `Arsenal.ImageMounter.Native` | 清空 TargetFrameworks，只 `net11.0` |
| `ArsenalImageMounterService`（aim_cli） | 继承 |
| `ArsenalImageMounterAnalyze` | 继承 |
| `ArsenalImageMounterCLISetup` | `net48;net11.0-windows;net10.0-windows` |
| `ArsenalImageMounterGUISetup` | `net48;net11.0-windows;net10.0-windows` |
| `ArsenalImageMounterMountTool` | `net48;net11.0-windows;net10.0-windows` |
| `TestProject` | `net11.0-windows;net10.0-windows;net48` |

注意 PowerShell 模块和 Analyze **没有** `net8.0` / `net9.0`，仓库里的 `Managed Source/publishps.cmd`
还在循环 `net48 net8.0 net9.0 net10.0`，是过时的，别照抄。

---

## 3. 为什么必须用 .NET 11 SDK

`Directory.Build.props` 的 `TargetFrameworks` 里含 `net11.0`。用 9.x SDK 会直接：

```
error NETSDK1045: 当前 .NET SDK 不支持将 .NET 11.0 作为目标。
```

所以 `DOTNET_SDK_VERSION` 固定 `11.0.x`，`Preflight` 步骤还会再校验一次，缺了就立刻报错退出，
免得等到 publish 才炸。

---

## 4. "GUI 程序"到底是什么

仓库里**没有** `ArsenalImageMounter.exe` 主程序。`Graphical applications/` 下只有一个
`Graphical application moved.txt`，带商业功能的 GUI 已经搬到 <https://ArsenalRecon.com>。

仓库内真正的可视化工具是 `Managed Source/ArsenalImageMounterMountTool`：

- `ArsenalImageMounterMountTool.vbproj` 里 `<OutputType>WinExe</OutputType>`
- `MainModule.vb` 有 `Public Sub Main()`
- `MainForm.Designer.vb` 是完整挂载界面（Mount raw / DiscUtils / libewf / libaff4 / libqcow、
  设备列表、RAM disk、Rescan SCSI bus、Remove all…）
- `MainForm.vb:804` `GetEmbeddedDriverVersion()` 和 `MainForm.vb:814` `InstallDriver()`
  从内嵌的 `DriverFiles.zip` 装驱动 —— 所以它自己就能完成驱动安装，不需要额外的开发工具。

这就是"可视化界面 + 不含开发工具"的那个成品。

**`Arsenal.ImageMounter.Forms` 是类库**，目录里只有 `AsyncMessageBox` / `DriverSetup` /
`FormsExtensions` / `NativeWindowHandle` / `RAMDiskForm` 和 csproj，**没有 `Program.cs`、没有 `Main`**。
用 `-p:OutputType=WinExe` 强行发布它只会得到一个跑不起来的 exe。

---

## 5. 踩过的坑（每一条都是真实 CI 失败换来的）

### 5.1 单文件发布必须带 `IncludeNativeLibrariesForSelfExtract`

只加 `-p:PublishSingleFile=true` 是不够的。`libewf.dll` / `libqcow.dll` / `libaff4.dll` /
`bzip2.dll` 这些是 P/Invoke 加载的**原生库**，默认被排除在单文件之外，产物看着是"一个 exe"，
一运行就找不到 DLL。必须同时加：

```
-p:PublishSingleFile=true
-p:IncludeNativeLibrariesForSelfExtract=true
-p:EnableCompressionInSingleFile=true
```

代价是首次启动会把原生库自解压到 `%TEMP%\.net\`，慢一两秒，属于固有行为。

### 5.2 `GeneratePackageOnBuild=false` 必须加

Release 配置下各库项目默认 `GeneratePackageOnBuild=true`，而 `Directory.Build.props` 把
`PackageOutputPath` 设成了 `$(LocalNuGetPath)`（未定义 → 空路径）。publish 时被顺带触发的 pack
会因为找不到输出目录而报错。所有 publish 命令都带上 `-p:GeneratePackageOnBuild=false`。

### 5.3 `dotnet pack` 不能单独跑

`dotnet pack Arsenal.ImageMounter.csproj` 直接失败：

```
error NU5026: 磁盘上找不到要打包的文件"...\bin\Release\net11.0\Arsenal.ImageMounter.runtimeconfig.json"
```

原因同上：`GeneratePackageOnBuild` 场景下 pack 只对**当前已构建的 TFM**有输出。正确顺序是
**先 `dotnet build`（构建全部 TFM），再 `dotnet pack --no-build`**，见 `build-all.yml` 的 `packages` job。

### 5.4 `net48` 不能带 `-r` / `--self-contained`

.NET Framework 没有 RID 概念，也没有自包含发布。脚本里用 `if ($rid) { ... }` 判断，
`net48` 那一行的 `rid` 留空字符串。

### 5.5 Analyze 的 TFM 不能带 `-windows` 后缀

`ArsenalImageMounterAnalyze.csproj` 没有覆盖 `TargetFrameworks`，继承的是
`net48;netstandard2.1;net10.0;net11.0`。写 `-f net10.0-windows` 会报：

```
error NETSDK1005: 资产文件"...\obj\project.assets.json"没有"net10.0-windows"的目标
```

必须用 `-f net10.0`（RID 仍然可以是 `win-x86`）。

### 5.6 YAML 块标量里不能用 PowerShell here-string

`run: |` 里的 here-string 结束符 `"@` **必须顶格**，而 YAML 块标量要求所有续行都有缩进，
顶格会直接把 YAML 解析搞崩：

```
yaml.scanner.ScannerError: while scanning a simple key, line 158, column 1,
could not find expected ':'
```

统一改成**字符串数组**：

```powershell
$lines = @(
  'line 1'
  'line 2'
)
Set-Content -LiteralPath "$dist/README.txt" -Value $lines -Encoding UTF8
```

### 5.7 上传大文件优先用 7-Zip

产物可达 130 MB，`Compress-Archive` 在这个量级上很慢且吃内存。`windows-latest` 自带
`C:\Program Files\7-Zip\7z.exe`，优先用它，失败再回退 `Compress-Archive`。

### 5.8 打包前先 `Push-Location dist`

否则 zip 里的顶层会带 `dist/...` 前缀，解压出来多一层目录。

### 5.9 `Get-ChildItem -Include ... | Remove-Item` 在结果为空时会崩

PowerShell 5.1 下会抛 `Remove-Item : Object reference not set to an instance of an object.`。
先收集成数组再逐个删：

```powershell
$junk = @(Get-ChildItem -LiteralPath $OutDir -Recurse -File |
          Where-Object { $_.Extension -in '.pdb', '.xml' })
foreach ($f in $junk) { Remove-Item -LiteralPath $f.FullName -Force -ErrorAction SilentlyContinue }
```

### 5.10 `$args` 是自动变量，函数里别当普通变量用

会报 `The variable '$args' cannot be retrieved because it has not been set`。改名成 `$dargs`，
splat 写成 `dotnet @dargs`。

### 5.11 本地跑 PS 5.1 时脚本要存成 UTF-8 **with BOM**

否则 `powershell.exe` 按 GBK 读中文注释，会出现一堆
`You must provide a value expression following the '-' operator` 之类的假语法错误。
（CI 里是 `pwsh` 7，没这个问题；但本地调试时很容易被这个坑住。）

### 5.12 测试用例有 3 个需要管理员权限

`TestProject` 共 16 个用例，其中 `Version.ValidateAuthenticodeWinPE64`、
`Win32API.PartitionTests`、`Win32API.QueryDefaultReparsePoints` 需要管理员 + 已安装驱动，
CI 里必然失败。所以 `build-all.yml` 的 `tests` job 是 `workflow_dispatch` 手动开启的，不阻塞主流程。

另外 `dotnet test -f net10.0-windows` 会因为 runner 上只装了 11.x 运行时直接崩
（`You must install or update .NET to run this application. Framework: 'Microsoft.NETCore.App', version '10.0.0'`），
测试统一用 `-f net11.0-windows`。

### 5.13 驱动是内嵌进 exe 的

`DriverFiles.zip` 有 2,977,579 字节，被 `<EmbeddedResource>` 编进 Setup 和 MountTool。
验证方法：产物里的 `ArsenalImageMounterGUISetup.dll` 约 2,992,128 字节，比 zip 略大，
说明确实打进去了。所以这些 exe 不需要联网、不需要旁边放驱动包。

### 5.14 `Arsenal.ImageMounter.Native` 的 nupkg 不含 dll 本体

它的 `IncludeBuildOutput=false`，只打包 `runtimes\**\native\*`。`runtimes/` 下有
`win-x86` / `win-x64` / `win-arm` / `win-arm64` 四个目录。

### 5.15 非托管部分不在 CI 里

`Unmanaged Source/ArsenalImageMounter.sln` 是 vcxproj，需要 **WDK 8.1+** 和完整 VS 工具链，
`windows-latest` 上的托管 SDK 构不出来。驱动本身用的是仓库里预编译好的 `DriverSetup/DriverFiles.zip`，
所以托管产物不需要现场编译驱动。

---

## 6. 想新增一个构建目标，怎么改

1. 先确认这个项目的**真实 TFM**（看它自己的 csproj/vbproj，没写就是继承 `Directory.Build.props`）。
2. 在 `build-all.yml` 的 `apps` job 的 `matrix.include` 里加一项：
   ```yaml
   - name: my-thing-net10-x64
     project: Managed Source/MyProject/MyProject.csproj
     tfm: net10.0
     rid: win-x64
     self_contained: 'true'
   ```
3. 如果是 GUI / 需要 Windows 桌面，TFM 得带 `-windows`，并且项目要有 `<OutputType>WinExe</OutputType>`
   和自己的 `Main`。
4. 改完**先本地或在一个分支上跑一遍** `workflow_dispatch`，确认 artifact 里有预期的 exe。

---

## 7. 相关文件

- `Managed Source/Directory.Build.props` —— 版本号、TFM、输出路径
- `DriverSetup/DriverFiles.zip` —— 驱动包，被内嵌
- `Managed Source/publishps.cmd` —— **已过时**（还在循环 net8.0/net9.0），以 `build-all.yml` 为准
- `How-to-build.md` —— **已过时**（写的是 .NET 6.0/7.0 和 VS2017-2022），以本文件为准
