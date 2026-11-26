# CI 构建说明

这个目录下有两个工作流，都是对着仓库**实际结构**写的（不是猜的）。改之前请先读完这份文件，尤其是[踩坑记录](#踩坑记录)那节——里面每一条都是在真实构建里撞出来的，重复踩一遍很浪费时间。

| 文件 | 显示名 | 触发 | 产出 |
|---|---|---|---|
| `build.yml` | `build-gui` | push / PR / tag / 手动 | 可视化界面成品（MountTool + GUISetup） |
| `build-all.yml` | `build-all` | 手动 + 每周日 | 全量托管产物（CLI、Analyze、NuGet 包、PowerShell 模块） |

`build.yml` 是默认那个，推代码就自动跑。

---

## 一、仓库真实结构（只列和构建有关的）

### 解决方案文件

| 文件 | 类型 | 说明 |
|---|---|---|
| `Managed Source/ArsenalImageMounter.slnx` | 新式 slnx | 托管代码，平台 `Any CPU` / `x64` / `x86` |
| `Unmanaged Source/ArsenalImageMounter.sln` | 传统 sln | 非托管（vcxproj），**需要 WDK 8.1+**，CI 里不构建 |

注意是 `.slnx` 不是 `.sln`，老版本 `dotnet` 命令不认。

### `Managed Source/Directory.Build.props` 关键属性

```xml
<TargetFrameworks>net48;netstandard2.1;net10.0;net11.0</TargetFrameworks>
<OutputPath>..\bin\$(Configuration)\</OutputPath>
<EnableWindowsTargeting>true</EnableWindowsTargeting>
<PackageOutputPath>$(LocalNuGetPath)</PackageOutputPath>
<FileVersion>3.13.369.0</FileVersion>
```

`$(LocalNuGetPath)` **没有被定义**，求值结果为空字符串——这是后面 `dotnet pack` 那个坑的根源之一。

### 各项目 TFM 与输出类型

| 项目 | TFM | 类型 | 备注 |
|---|---|---|---|
| `Arsenal.ImageMounter` | 继承四件套 | 类库 | `PackageId=ArsenalRecon.ImageMounter`，Release 下 `GeneratePackageOnBuild=true` |
| `Arsenal.ImageMounter.Forms` | `net48;net11.0-windows;net10.0-windows` | 类库 | **没有 `Main`**，别想发布成 exe |
| `Arsenal.ImageMounter.Native` | `net11.0` | 纯打包 | `IncludeBuildOutput=false`，只打 `runtimes/**/native/*` |
| `Arsenal.ImageMounter.PowerShell` | `net48;net10.0;net11.0` | 模块 | 不含 net8.0/net9.0 |
| `ArsenalImageMounterService` | 继承四件套 | Exe | **`AssemblyName=aim_cli`** |
| `ArsenalImageMounterAnalyze` | 继承四件套 | Exe | slnx 里固定 x86 |
| `ArsenalImageMounterCLISetup` | `net48;net11.0-windows;net10.0-windows` | Exe (WinForms) | 内嵌 `DriverSetup/DriverFiles.zip` |
| `ArsenalImageMounterGUISetup` | `net48;net11.0-windows;net10.0-windows` | WinExe | 内嵌 `DriverFiles.zip` + `EULA.txt` |
| `ArsenalImageMounterMountTool` | `net48;net11.0-windows;net10.0-windows` | WinExe | 内嵌 `DriverFiles.zip`，**仓库里唯一真正的 GUI** |
| `TestProject` | `net11.0-windows;net10.0-windows;net48` | xunit | 16 个用例，3 个要管理员 + 已装驱动 |
| `ArsenalImageMounterHelpBuilder` | — | shfbproj | Sandcastle，slnx 里 `Build=false` |

### 关于「GUI 程序」

`Graphical applications/` 目录下**只有一个 `Graphical application moved.txt`**。大家熟悉的 `ArsenalImageMounter.exe` 已经移到 [ArsenalRecon.com](https://ArsenalRecon.com) 的商业版里了，开源仓库中不存在，CI 怎么改都变不出来。

仓库内真正的可视化工具是 `ArsenalImageMounterMountTool`：

- `ArsenalImageMounterMountTool.vbproj` 里 `<OutputType>WinExe</OutputType>`、`<UseWindowsForms>true</UseWindowsForms>`
- `MainModule.vb` 有 `Public Sub Main()`
- `MainForm.Designer.vb` 是完整挂载界面（Mount raw / DiscUtils / libewf / libaff4 / libqcow、设备列表、RAM disk、Rescan SCSI bus）
- `MainForm.vb:804 GetEmbeddedDriverVersion()`、`MainForm.vb:814 InstallDriver()` 用 `GetManifestResourceStream(GetType(MainForm), "DriverFiles.zip")` 自动装驱动

所以它就是「可视化界面 + 不含开发工具」的成品。

---

## 二、为什么必须 .NET 11 SDK

`Directory.Build.props` 的 `TargetFrameworks` 里有 `net11.0`。用 9.x SDK 会直接：

```
error NETSDK1045: 当前 .NET SDK 不支持将 .NET 11.0 设置为目标。
```

所以两个工作流都固定：

```yaml
env:
  DOTNET_SDK_VERSION: '11.0.x'
```

并且 `Preflight` 步骤会显式检查 `dotnet --list-sdks` 里有没有 `11.` 开头，没有就直接 fail 并给出人话报错，而不是等到编译阶段报 NETSDK1045。

> 顺带：`How-to-build.md` 里写的「.NET 6.0 / 7.0 SDK、VS2017-2022」已经过时了；`Managed Source/publishps.cmd` 里 `for %%a in (net48 net8.0 net9.0 net10.0)` 的循环也过时（net8.0/net9.0 已 EOL，且项目根本不产出这两个 TFM）。别照抄。

---

## 三、产物长什么样

### `build-gui`（`build.yml`）

矩阵三项，各出一个 zip：

| 变体 | TFM | RID | 形态 | 大小 |
|---|---|---|---|---|
| `net48` | `net48` | 无 | 文件夹（exe + dll） | ≈ 13 MB |
| `net10-x64` | `net10.0-windows` | `win-x64` | 文件夹（exe + dll） | ≈ 103 MB |
| `net10-x64-single` | `net10.0-windows` | `win-x64` | **单个 exe** | ≈ 100 MB |

前两个是常规发布。第三个加了 `PublishSingleFile` + `IncludeNativeLibrariesForSelfExtract`，打出来是一个自解压的单文件 exe，**不需要任何运行时**，拷到机器上双击就能用。

> 单文件版的代价：首次启动会把原生库解压到 `%TEMP%\.net\` 下，启动慢一两秒；`net48` 变体做不了单文件（.NET Framework 没有这个能力）。

zip 内部结构（前两个变体）：

```
ArsenalImageMounter-GUI-<variant>/
├── MountTool/
│   ├── ArsenalImageMounterMountTool.exe    ← 双击运行
│   ├── Arsenal.ImageMounter.dll
│   ├── Arsenal.ImageMounter.Forms.dll
│   └── libewf.dll / libqcow.dll / libaff4.dll / bzip2.dll / ... （原生库）
├── DriverSetup/
│   └── ArsenalImageMounterGUISetup.exe     ← 纯驱动安装/卸载器
└── README.txt
```

单文件变体的 zip 里只有：

```
ArsenalImageMounter-GUI-net10-x64-single/
├── ArsenalImageMounterMountTool.exe        ← 一个文件，双击运行
├── ArsenalImageMounterGUISetup.exe         ← 一个文件
└── README.txt
```

tag（`refs/tags/v*`）触发时，`release` job 会自动把 zip 挂到 GitHub Release。

### `build-all`（`build-all.yml`）

8 个 artifact：`gui-net48`、`guisetup-net48`、`gui-net10-x64`、`clisetup-net10-x86`、`aim_cli-net48`、`aim_cli-net10-x64`、`analyze-net10-x86`、`packages`。

`packages` 里是 `ArsenalRecon.ImageMounter.3.13.369.nupkg` + `.snupkg`。

---

## 四、踩坑记录

以下全部在真实构建里复现过，按「症状 → 原因 → 解法」记。

### 1. `NETSDK1045`：当前 SDK 不支持目标框架

**原因**：用了 9.x 或更低的 SDK，而 `Directory.Build.props` 要求 `net11.0`。
**解法**：`DOTNET_SDK_VERSION: '11.0.x'`。

### 2. 把 `Arsenal.ImageMounter.Forms` 发布成 exe 失败

**原因**：那个项目目录里只有 `AsyncMessageBox` / `DriverSetup` / `FormsExtensions` / `NativeWindowHandle` / `RAMDiskForm` 和 csproj，**没有 `Program.cs`，没有 `Main`**，它是类库。原工作流这么干：

```powershell
Get-ChildItem -Path "Managed Source" -Recurse -Filter "*Forms*.csproj" | Select-Object -First 1
dotnet publish ... -p:OutputType=WinExe
```

强行改 `OutputType` 只会在链接阶段报找不到入口点。
**解法**：构建 `ArsenalImageMounterMountTool`（真有 `Main` 的那个）。

### 3. `dotnet pack` 单独跑必然 `NU5026`

```
error NU5026: 磁盘上找不到要打包的文件 "...\bin\Release\net11.0\Arsenal.ImageMounter.runtimeconfig.json"
```

**原因**：`GeneratePackageOnBuild=true` 时 pack 只针对**当前已构建的 TFM** 有输出，而 `TargetFrameworks` 有四个（`net48;netstandard2.1;net10.0;net11.0`），没构建过的 TFM 目录是空的。`$(LocalNuGetPath)` 为空让情况更糟。
**解法**：两步走——

```powershell
dotnet build "Managed Source/Arsenal.ImageMounter/Arsenal.ImageMounter.csproj" -c Release
dotnet pack  "Managed Source/Arsenal.ImageMounter/Arsenal.ImageMounter.csproj" -c Release --no-build `
  -p:LocalNuGetPath=<输出目录>
```

`build-all.yml` 的 `packages` job 就是这么写的。

### 4. `ArsenalImageMounterAnalyze` 的 TFM 不能带 `-windows`

```
error NETSDK1005: 资产文件 "...\ArsenalImageMounterAnalyze\obj\project.assets.json" 没有 "net10.0-windows" 的目标。
```

**原因**：它**没有**覆盖 `TargetFrameworks`，继承的是 `net48;netstandard2.1;net10.0;net11.0`，里面根本没有 `-windows` 变体。别的 WinForms 项目自己写了 `net10.0-windows`，容易顺手抄错。
**解法**：`-f net10.0`（配 `-r win-x86`）。

### 5. `net48` 不能带 `-r` / `--self-contained`

**原因**：RID 和自包含是 .NET Core / .NET 5+ 的概念，.NET Framework 不支持。
**解法**：脚本里判断，只有非 `net4*` 才加：

```powershell
if ($rid) { $dargs += @('-r', $rid, '--self-contained', 'true') }
```

### 6. 每个矩阵项都重复触发打包

**原因**：Release 配置下 `GeneratePackageOnBuild=true`，每跑一次 publish 都顺手 pack 一遍，产物还会往空路径 `$(LocalNuGetPath)` 里塞。
**解法**：所有 publish 命令统一加 `-p:GeneratePackageOnBuild=false`。

### 7. YAML 被 PowerShell here-string 搞崩

```
yaml.scanner.ScannerError: while scanning a simple key, ... line 158, column 1, could not find expected ':'
```

**原因**：here-string 的结束符 `"@` **必须顶格**，顶格之后 YAML 块标量（`run: |`）的缩进就断了，解析器直接崩。
**解法**：别在 `run: |` 里用 here-string，改成字符串数组：

```powershell
$readme = @(
  '第一行'
  '第二行'
)
Set-Content -LiteralPath "$dist/README.txt" -Value $readme -Encoding UTF8
```

`build.yml` 里生成 `README.txt` 那步就是这么绕开的。

### 8. `Get-ChildItem -Include ... | Remove-Item` 空结果时抛 NullReferenceException

```
Remove-Item : Object reference not set to an instance of an object.
```

**原因**：PS 5.1 的 `Remove-Item` 管道绑定在空集合上的老 bug。
**解法**：先收集成数组再逐个删：

```powershell
$junk = @(Get-ChildItem -LiteralPath $OutDir -Recurse -File |
          Where-Object { $_.Extension -in '.pdb', '.xml' })
foreach ($f in $junk) { Remove-Item -LiteralPath $f.FullName -Force -ErrorAction SilentlyContinue }
```

### 9. 130 MB 的 zip 别用 `Compress-Archive`

**原因**：慢、吃内存，而且路径里有空格/中文时容易出岔子。
**解法**：优先用 `windows-latest` 自带的 7-Zip，没有才回退：

```powershell
$sevenZip = 'C:\Program Files\7-Zip\7z.exe'
if (Test-Path -LiteralPath $sevenZip) {
  & $sevenZip a -tzip -mx=5 "..\$zip" $folder
} else {
  Compress-Archive -Path "$folder/*" -DestinationPath "..\$zip" -Force
}
```

### 10. 测试：3/16 失败是正常的

`dotnet test -f net11.0-windows` 的结果是 **13 通过 / 3 失败**。失败的是：

- `Arsenal.ImageMounter.Tests.Version.ValidateAuthenticodeWinPE64`
- `Arsenal.ImageMounter.Tests.Win32API.PartitionTests`
- `Arsenal.ImageMounter.Tests.Win32API.QueryDefaultReparsePoints`

**原因**：这三个需要**管理员权限 + 已安装虚拟 SCSI 驱动**，GitHub 托管的 runner 上没有。
**解法**：`build-all.yml` 里 tests 是**手动可选**的（`workflow_dispatch` 的 `run_tests` 输入），且 `continue-on-error`，不让它卡住产物。

顺带：`-f net10.0-windows` 跑测试会直接崩，因为 runner 上只有 11.x 运行时：

```
You must install or update .NET to run this application.
Framework: 'Microsoft.NETCore.App', version '10.0.0'
```

所以测试固定用 `-f net11.0-windows`。

### 11. （仅本地）`NuGet.Config` 未授权访问

```
error : 由于未授权访问权限，因此未能读取 NuGet.Config
```

**原因**：`dotnet` 会无条件读 `%APPDATA%\NuGet\NuGet.Config`，该目录不存在时在受限环境下报这个。`--configfile` 和 `-p:RestoreConfigFile` 都绕不过。
**解法**：先建目录再写一份标准配置。CI 里 `setup-dotnet` 已经处理好，**不需要在 workflow 里做这件事**，记在这里只是备查。

### 12. 驱动包内嵌的证据

`DriverSetup/DriverFiles.zip` = 2,977,579 字节。发布后：

| 产物 | 大小 |
|---|---|
| `ArsenalImageMounterCLISetup.dll` | 2,987,008 |
| `ArsenalImageMounterGUISetup.dll` | 2,992,128 |
| `ArsenalImageMounterMountTool.exe`（net48） | 3,038,208 |

都大于 `DriverFiles.zip`，说明驱动确实被嵌进去了。改动 `DriverSetup/DriverFiles.zip` 之后这几个体积应该同步变化，可以用来快速验证内嵌是否生效。

### 13. 单文件发布要连原生库一起自解压

只加 `-p:PublishSingleFile=true` 是不够的：`libewf.dll`、`libqcow.dll` 这些是 P/Invoke 加载的原生库，默认会被排除在单文件之外，运行时找不到就直接崩。必须同时加：

```powershell
-p:PublishSingleFile=true -p:IncludeNativeLibrariesForSelfExtract=true
```

而且 `net48` 不支持单文件发布，只有 `net10.0-windows` 这类 .NET 5+ 目标才行。

---

## 五、想改 / 新增工作流时

### 加一个新的构建目标

1. 先确认项目的 TFM。去 `Managed Source/<项目>/<项目>.csproj|vbproj` 里看有没有 `<TargetFrameworks>` 覆盖：
   - **有覆盖** → 用覆盖里的值（通常带 `-windows`）
   - **没有覆盖** → 继承四件套，用 `net10.0`（**不带 `-windows`**，见坑 4）
2. 确认是不是可执行：`<OutputType>Exe</OutputType>` 或 `WinExe`，且代码里有 `Main`。
3. 往 `build-all.yml` 的 `apps` job 矩阵里加一项：

```yaml
- name: <artifact 名>
  project: Managed Source/<路径>/<项目>.csproj
  tfm: net10.0
  rid: win-x64        # net48 写空字符串
```

4. 本地先验一遍再推：

```powershell
dotnet publish "<项目>" -c Release -f <tfm> -r <rid> --self-contained true `
  -p:GeneratePackageOnBuild=false -o out/<名字>
```

### 换 SDK 版本

改 `env.DOTNET_SDK_VERSION` 即可，但**必须**同时确认 `Directory.Build.props` 的 `TargetFrameworks` 里最高的那个 TFM 有对应 SDK，否则必然 NETSDK1045。

### 验证 workflow 改对了没

推上去之后看 Actions 页面，或者：

```powershell
gh run list --limit 5
gh run view <run-id> --log-failed
```

注意：`build.yml` 有 `paths-ignore`（`**.md`、`Third-Party-Licenses/**`、`libewf/**`、`Unmanaged Source/**`）。**只改这个 README 不会触发构建**，这是故意的。

---

## 六、非托管部分（不在 CI 里）

`Unmanaged Source/` 下的驱动是 vcxproj，编译需要：

- Visual Studio 2022 + **WDK 8.1 或更高**
- 相应的 WDK Visual Studio 扩展

GitHub 托管 runner 上没有 WDK，装起来又慢又容易失败，所以两个工作流都**不构建驱动**。驱动二进制走的是仓库里已签好的 `DriverSetup/DriverFiles.zip`（2.98 MB），直接内嵌进 Setup / MountTool 的 exe。

也就是说：**改驱动源码 → CI 不会重新编译驱动**，需要本地用 WDK 编完、重新打包 `DriverFiles.zip` 再提交。
