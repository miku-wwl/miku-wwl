# 《轩辕剑外传：苍之涛》花屏与字体修复复盘

整理日期：2026-10-21，Pacific/Auckland。对象：`D:\SWDMD`。本文供后续 agent 复用定位思路；版本、文件偏移和显卡参数须按实际程序重新确认。

## 1. 已确认的结果与当前入口

用户已于 2026-10-21 明确确认：`SWDMD-Fixed.exe` 已经解决问题。

| 项目 | 证据与结论 |
| --- | --- |
| 花屏 | 安装图形兼容层后，工具实测主菜单黑绿碎块恢复为正常飘叶；进程实际加载游戏目录的兼容 DLL 和系统 Direct3D 11。 |
| 对话字体 | 原始字体请求与英文别名请求的 GDI 对照测试复现并消除了缺字、字高异常；最终游戏内效果由用户确认。 |
| 官方字库 | `mingliu.ttc` 现已存在并注册；2026-10-21 的全新进程可直接选中 MingLiU，无需私有加载字体。 |
| 原程序 | `SWDMD.EXE` 保持原始 SHA-256；另建修复副本。 |
| 长时间运行、全流程战斗、声音、存读档回归 | 未完成系统性测试，不能由菜单和字体修复推断全部通过。 |

**当前已获用户确认的入口是 `D:\SWDMD\SWDMD-Fixed.exe`。** 本次总结时只做读取与文档整理：`Start-Fixed.cmd` 仍指向原版 `SWDMD.EXE`，不能因为文件名带 Fixed 就认定它会启动字体修复版。需要调整入口时应另行在明确任务范围内进行，并保留原启动器。

## 2. 先保全现场，再判断是哪一层出错

本机当时为 Windows 11，构建 26200，AMD Radeon 780M，桌面 3840×2160；游戏为 32 位旧程序。操作前备份了：

- `Save`、`User.cfg`、`debug.txt`。
- 当前用户的**该游戏路径**兼容性注册表值。
- 关键 EXE/DLL 的 SHA-256，供之后确认原文件未变。

备份目录：

```text
D:\SWDMD\_repair_backup\20261020-222727
D:\SWDMD\_repair_backup\20261020-223703-font
```

后续 agent 应先读取已有记录和当前配置，避免对已修复的目录重复打补丁。用户已经在运行游戏时，先确认当前进度与进程状态，避免关闭未保存的会话。

## 3. 第一次修复：旧图形接口兼容性

### 定位方法

1. 读取游戏目录、初始化日志、显示适配器和每个 EXE 的兼容性设置。
2. 检查 PE 导入表，确认实际使用的接口。该程序导入 `DDRAW.dll`，调用 `DirectDrawCreate`、`DirectDrawEnumerateA`；使用 DirectDraw/旧 Direct3D 路径。说明书要求 DirectX 8.1，并不等于程序一定导入 `D3D8.dll`。
3. 查看真实画面：修复前主菜单的飘叶表现为黑绿碎块。不要仅凭“能启动”判断渲染正常。

原兼容性值混有 XP SP3、256 色、16 位色等多项标记：

```text
~ DWM8And16BitMitigation DISABLEDXMAXIMIZEDWINDOWEDMODE ~ WINXPSP3 ~ 256COLOR ~ 16BITCOLOR HIGHDPIAWARE
```

这些设置与图形兼容层一并调整后问题改善；没有单独逐项做 A/B 测试，因此不能把某个颜色标记断言为唯一根因。

### 实施方法

使用作者官方发布的 **dgVoodoo2 2.87.5**，核对下载压缩包 SHA-256：

```text
5FFDE6927F7355CA3FDD5D785B581256A8E6539FA13E395A891ADE6BA1040850
```

只把同一发行包中 `MS\x86\DDraw.dll`、`MS\x86\D3DImm.dll` 放到游戏 EXE 同目录，并使用该版本配套的完整 `dgVoodoo.conf`。本例不需要额外加入 D3D8/D3D9 DLL，也没有替换 Windows 系统目录的 DirectX DLL。

本例关键配置如下。这是**完整配置中的相关项摘录**，不是让其他 agent 用它覆盖整个配置文件：

```ini
[General]
OutputAPI = d3d11_fl11_0
FullScreenMode = true
ScalingMode = stretched_ar

[GeneralExt]
FullscreenAttributes = fake
FPSLimit = 60

[DirectX]
VideoCard = geforce_ti_4800
VRAM = 256
Resolution = unforced
Antialiasing = appdriven
ForceVerticalSync = true
dgVoodooWatermark = false
```

这些参数在本机得到验证，不是所有 AMD 显卡、所有游戏的通用最优设置。修复时先保证原分辨率和正常渲染，再考虑提高清晰度。

**按节编辑 INI。** 配置里 Glide 与 DirectX 有同名的 `VideoCard`、`Resolution` 等键；不要全文件正则替换同名键。本次早期编辑误把一个 DirectX 显卡名称也写入未使用的 Glide 节，虽未影响本游戏，但应避免效仿这个编辑失误。

兼容性注册表调整仅作用于该游戏 EXE，保留高 DPI 标记，移除前述堆叠的旧兼容标记。不要清空其他程序的设置。正常权限启动时，使用游戏目录作为工作目录；本例启动器使用 `RunAsInvoker HIGHDPIAWARE`。`RunAsInvoker` 只是请求继承当前普通权限，不会赋予管理员权限，不能作为绕过系统授权的方式。

### 验证方法

- 确认进程加载的是 `D:\SWDMD\DDraw.dll`、`D:\SWDMD\D3DImm.dll`，同时加载系统 `D3D11.dll`；只看到补丁文件存在不算生效。
- 再检查修复前同一菜单中的透明贴图、颜色、飘叶；有条件再验证场景、过场和菜单。
- 水印只用于辅助确认，确认实际模块后可关闭。窗口/假全屏配置并不能证明 Alt+Tab 或失焦崩溃已经修好。

## 4. 第二次修复：字体缺失与字体名称编码

### 不能把所有“乱码”都当作文本文件编码错误

用户照片里的问题是对话字形缺损、裁切、宽高不正常。资源中的 Big5 文本可以正确解码为 `車芸：原來是夢啊……`，因此没有批量转换 GLL 文件。

进一步检查发现：

1. 系统 ANSI 代码页为 936，游戏资源为繁体 Big5。
2. 系统最初只有 `mingliub.ttc` 扩展字库，缺少基础 `mingliu.ttc`。名字带 `MingLiU-ExtB` 不等于基础細明體已安装。
3. EXE 使用 `CreateFontIndirectA`，写死 Big5 编码的字体名“細明體”；字体请求中的 `lfCharSet = 136`（Big5 字符集）、`lfHeight = -20`、`lfWeight = 400`。
4. ANSI 字体 API 的**字体名称解码**和文本绘制的**字符集选择**是两个检查点；设置 Big5 字符集并不能保证 Big5 字节形式的字体名称在代码页 936 下仍被正确识别。

### 用真实 GDI 请求验证，不只猜字体

附件 `Probe-GameFont.ps1` 在内存 DC 中复现该游戏的字体请求：用 `GetTextFaceW` 读取实际选中的字体，用 `GetGlyphIndicesW` 检查测试字符串缺字，用 `GetTextExtentPoint32A` 测量文字。它不创建窗口、不向游戏发送输入，不注入游戏进程。

2026-10-21 在**官方基础字库已安装后**重新测试：

| 请求的字体名称 | 实际选中字体 | 测试字符串缺字数 | 实测字高 |
| --- | --- | ---: | ---: |
| 原始 Big5 字节“細明體” | Ebrima | 8 | 28 像素 |
| ASCII 英文别名 `MingLiU` | MingLiU | 0 | 20 像素 |

因此本机仅补装字库还不够，还需要修正字体名称。28 像素的字体度量超过游戏为此请求准备的 20 像素字格，可以解释裁切现象；最终修复仍要以游戏实际画面确认。

运行示例（在脚本所在目录，用 PowerShell 7；脚本也保存为带 BOM 的 UTF-8）：

```powershell
pwsh -NoProfile -File .\Probe-GameFont.ps1
pwsh -NoProfile -File .\Probe-GameFont.ps1 -AsciiFontName MingLiU
```

脚本默认参数是本游戏的专用复现模板。其他游戏应重新确认 LOGFONT 参数、样本文本和 API。可选 `-PrivateFontPath` 只在诊断进程内暂时加载字体，用于隔离验证；这不是系统安装，也不会让别的游戏进程自动获得该字体。

### 补齐官方基础字库

安装 Windows 官方能力：

```text
Language.Fonts.Hant~~~und-HANT~0.0.1.0
```

可使用 Windows 设置中的“中文（繁体）补充字体”，或在已获授权、由用户完成管理员确认的情况下运行系统 DISM：

```text
DISM.exe /Online /Add-Capability /CapabilityName:Language.Fonts.Hant~~~und-HANT~0.0.1.0 /NoRestart
```

本次已得到用户允许后才安装。系统管理员确认由用户处理；不自动操作安全确认窗口，不擅自重启电脑。

本次处理约 25 分钟，过程中有长时间的 49% 和 100% 等待。必须区分“下载完成”和“安装完成”：最终 DISM 日志于 2026-10-20 23:10:45 结束会话，记录 `Reboot required=no`；复核还确认 `C:\Windows\Fonts\mingliu.ttc` 存在、字体注册项存在、全新 GDI 进程可选中 MingLiU。没有保留到最终进程退出码，不虚构退出码为 0。

系统范围的既有“待重启”标记，不等于本次字体安装一定要求重启；应以该操作的最终结果判断。

### 在副本里把字体名换成 ASCII 别名

只适用于以下精确原版文件：

```text
文件：SWDMD.EXE
长度：983040 字节
SHA-256：756A8FDA20AB7D55B6733954C0F44ACEC74C1D9C7F5D35391AA0A5E70522CA13
文件偏移：930320（0xE3210）
原 8 字节：B2 D3 A9 FA C5 E9 00 00
新 8 字节：4D 69 6E 67 4C 69 55 00
```

原空间为 6 字节 Big5 字体名加两个零字节；英文 `MingLiU` 为 7 个 ASCII 字节加一个终止零，仍占 8 字节，不改变文件长度。生成 `SWDMD-Fixed.exe`，原版不覆盖。

可复现本例副本的 PowerShell 代码如下。它会拒绝未知版本和已存在的目标文件；**已经修复的目录无需再次运行**：

```powershell
$ErrorActionPreference = 'Stop'
$src = 'D:\SWDMD\SWDMD.EXE'
$dst = 'D:\SWDMD\SWDMD-Fixed.exe'
$originalHash = '756A8FDA20AB7D55B6733954C0F44ACEC74C1D9C7F5D35391AA0A5E70522CA13'
$fixedHash = 'E9C05F694528B6E0E8C3C24F28A274E204DB75F44B09DEF5B2B4701B04ED84C5'
if (Test-Path -LiteralPath $dst) { throw '目标已存在，请先核验，禁止覆盖。' }
if ((Get-FileHash -LiteralPath $src -Algorithm SHA256).Hash -ne $originalHash) {
    throw '不是本例精确版本，必须重新定位，不能套用偏移。'
}
$bytes = [IO.File]::ReadAllBytes($src)
if ($bytes.Length -ne 983040) { throw '长度不符' }
$offset = 930320
$old = [byte[]](0xB2,0xD3,0xA9,0xFA,0xC5,0xE9,0,0)
for ($i=0; $i -lt 8; $i++) {
    if ($bytes[$offset+$i] -ne $old[$i]) { throw '原字体名或填充空间不符' }
}
$new = [Text.Encoding]::ASCII.GetBytes("MingLiU`0")
[Array]::Copy($new,0,$bytes,$offset,8)
$sha = [Security.Cryptography.SHA256]::Create()
try { $computed = [BitConverter]::ToString($sha.ComputeHash($bytes)).Replace('-','') }
finally { $sha.Dispose() }
if ($computed -ne $fixedHash) { throw '输出内容不符，未写入' }
[IO.File]::WriteAllBytes($dst,$bytes)
if ((Get-FileHash -LiteralPath $dst -Algorithm SHA256).Hash -ne $fixedHash) {
    throw '写入后校验失败，不要运行该副本。'
}
Get-FileHash -LiteralPath $src,$dst -Algorithm SHA256
```

2026-10-21 已重新逐字节比较：两个文件同为 983040 字节；差异只在 930320～930326 共 7 个字节；930327 的终止零与其他所有内容未变。没有修改指令逻辑、资源文本或存档。

**其他 EXE 不能照抄该偏移。** 应重新找字体 API、字体字节、别名、缓冲区空间和引用关系。新名称超过可用空间时，不可越界覆盖。没有相同哈希、预期字节和足够空间，就应停止这条固定偏移方案。

## 5. 本次走过的弯路，后续 agent 应跳过

- 区域模拟器没有成为成功方案：便携启动尝试缺少共享程序集，之后受到自动审批拒绝；没有部署为最终启动入口。不要写成“Locale Emulator 修好了”。相关临时文件已归档到字体备份中的 `unused-CompatLocale`。
- 官方字体查看器的备用安装尝试没有得到成功安装的验证；最终基础字库可用的证据来自 Windows 官方组件安装及之后的注册/GDI 复核。
- 多次在自动控制、切换焦点或输入阶段发生崩溃；WER 曾出现图形 DLL、游戏自身异常和 UI Automation 模块。模块出现在日志里不等于已经证明它是根因。不能把此次字体修复扩写成“所有崩溃已解决”。
- 对这种自动操作不稳定的旧游戏，先读当前 computer-use 技能；控制失败后按工具的恢复次数处理。需要时让用户手动进入对话，再读取画面，避免反复触发崩溃。用户按 Esc 停止控制时，当轮立即停止。
- 不能通过换工具、换命令包装来绕过自动审批拒绝，也不能声称看见了未实际看到的游戏画面。

## 6. 其他旧游戏可复用的判断顺序

1. **备份与识别版本**：保存存档、配置、逐 EXE 兼容设置与关键哈希。
2. **判别图形接口**：用导入表/实际调用决定兼容 DLL 和位数，而不是只看游戏年代或安装说明。
3. **修复渲染并确认加载**：官方来源、游戏目录内的最小兼容层；先验证原分辨率与同一异常画面。
4. **将字体与文本分开检查**：资源文本是否正确、基础字体是否存在、字体名在当前代码页如何解释、实际选中了什么字体、字形和度量是否适配游戏字格。
5. **选择可回退的最小变更**：本例是官方字库加副本中的 ASCII 字体别名；其他游戏需按证据选择，不能泛化为所有程序都应补 DLL 或改 EXE。
6. **分层记录验证**：文件/哈希检查、独立 API 复现、进程模块确认、实际游戏画面、用户确认各自说明；没有做的长时或全流程测试明确记为未运行。

## 7. 现有文件指纹与回退线索

| 文件 | SHA-256 |
| --- | --- |
| 原版 `SWDMD.EXE` | `756A8FDA20AB7D55B6733954C0F44ACEC74C1D9C7F5D35391AA0A5E70522CA13` |
| 修复版 `SWDMD-Fixed.exe` | `E9C05F694528B6E0E8C3C24F28A274E204DB75F44B09DEF5B2B4701B04ED84C5` |
| `DDraw.dll`（本次发行包） | `612A24408A090A3C6F3886557FA18034EE742E94AD0A40EBDF854D2816176C2E` |
| `D3DImm.dll`（本次发行包） | `93C534F2D17419EA78F15551F7E0AAC78B3C503733A840914FA063708A5AFE8E` |

副本与图形兼容层是两个独立修改层：启动原版只绕开字体别名补丁，仍会加载目录中的图形兼容层；需要完全回退时，应先关闭游戏，按备份记录恢复配置和该 EXE 的兼容值，并把本次新增兼容文件移到备份目录，而非覆盖或批量删除。无需回退存档或卸载系统字体来试图还原 EXE。

本复盘没有改写启动器、游戏 EXE、图形配置、系统区域或存档，只增加交接文档与诊断脚本。

## 官方资料

- [dgVoodoo2 2.87.5 作者发行页](https://github.com/dege-diosg/dgVoodoo2/releases/tag/v2.87.5)
- [dgVoodoo 作者说明](https://dege.freeweb.hu/dgVoodoo2/ReadmeGeneral/)
- [微软：补齐缺失字体与繁体中文补充字体](https://learn.microsoft.com/en-us/windows/deployment/windows-missing-fonts)
- [微软：语言和区域按需功能及能力名称](https://learn.microsoft.com/zh-cn/windows-hardware/manufacture/desktop/features-on-demand-language-fod?view=windows-11)
