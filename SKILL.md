---
name: wechat-space-audit
description: 盘点 Windows 上微信（4.x / xwechat）占用的空间，区分「可安全删的缓存」与「不能碰的数据」（尤其收到的文档），产出带勾选列的清理确认清单，确认后送系统回收站并核验，最后出执行记录。当用户说「看看微信有什么临时文件要删」「微信缓存清理」「微信占了多大空间」「微信清理一下」，或泛指清理某款软件的缓存并要求先出清单时使用。含微信 4.x 真实目录地图、各目录体积量级、分档清理策略、SHFileOperationW 送回收站方案与 $I 元数据验证法、「微信驻留托盘无法程序化优雅退出」的坑、以及「先扫描后确认」的安全流程。
agent_created: true
---

# 微信空间体检（Windows / 微信 4.x）

## 一、铁律：先扫描，后确认，绝不先删

用户提这类需求时，**默认只做只读扫描并出清单**。哪怕他说"清理一下"，第一轮也只给列表。

- 扫描阶段**只读**，不移动、不重命名、不删除任何文件。
- 交付一份带「是否执行」列的清单，让用户勾选。
- 拿到勾选后才动手，**每批不超过 10 项**，逐批核验。

**必须问清或默认保护的东西**：
- 用户如果说"收到的文档不要删" —— 那就是 `msg\file`，一个字节都不能碰。
- 默认保护：`msg\file`、`db_storage`、`config`、`all_users\login`、`resource`、`xplugin\plugins` 运行时本体。

## 二、微信 4.x 目录地图（核心知识）

微信 4.x 与 3.x 目录结构完全不同。**4.x 用的是 `xwechat_files`，不是 `WeChat Files`**。

### 2.1 聊天数据目录
`C:\Users\<用户>\Documents\xwechat_files\`

```
xwechat_files/
├── all_users/            登录态、头像、全局配置（≈300 KB）
├── Backup/               备份（常为空）
├── global_config/        全局配置
└── wxid_<xxx>/           每个微信号一个目录
    ├── msg/
    │   ├── file/         🔴 收到的文档（用户红线，绝对不动）
    │   ├── video/        ⚠️ 聊天视频（内容本体，删了旧记录变「已过期」）
    │   ├── attach/       ⚠️ 按会话哈希存的聊天图片/文件（内容本体）
    │   └── migrate/      迁移配置（通常 0）
    ├── db_storage/       🔴 消息数据库（含 emoticon.db）
    ├── cache/<年-月>/     ✅ Message / Sns / Emoticon / HttpResource / WeAppIcon
    ├── temp/             ✅ RWTemp / ImageTemp / head_image
    ├── resource/         🟡 表情与皮肤资源包（数字命名，删了会重下）
    ├── business/         🟡 业务数据（含空的 emoticon 目录）
    ├── config/           🔴 账号配置
    └── apm_record/       ✅ 性能埋点
```

**实操提示**：量级上，主号常在 **300–400 MB**，小号十几 MB。`msg/file` + `msg/video` + `msg/attach` 三者合计通常占聊天目录的 85% 以上。

### 2.2 程序数据目录（**真正的大头**）
`C:\Users\<用户>\AppData\Roaming\Tencent\xwechat\`

```
xwechat/
├── xplugin/plugins/      🟡 运行插件，常 700 MB+
│   ├── RadiumWMPF/       小程序运行时（最大，单版本 400 MB+）
│   ├── XPlayer/ WeChatPlayer/     播放器
│   ├── WeChatOcr/        OCR
│   ├── XFilesOfficeReader/  Office 预览
│   ├── XEditor/ WeChatUtility/ WMPFDrm/ WaveAudioModel/
├── radium/               🟡 内置浏览器与小程序引擎，常 500 MB+
│   ├── users/<uid>/      xworker/（mpxworker/liteapp/search）+ applet/（publicLib/codecache/data/packages）
│   └── web/profiles/     ← 里面每个 profile 都带 Cache/，**最大单块可清空间**
│       ├── multitab_<uid>/Cache/    常 150–200 MB  ✅ 最值得清
│       ├── webview_<uid>/Cache/
│       └── game*/ game_<uid>/ web_shell/
└── log/                  ✅ mm_YYYYMMDD.xlog 按天一个，常 80–90 MB
```

**这条最关键**：用户以为"微信占空间"是聊天记录，其实**程序数据目录常常是聊天目录的 3–4 倍**。示例：聊天 349 MB vs 程序 1.35 GB。

### 2.3 表情 / emoticon 在哪（4.x 与 3.x 不同）

3.x 有 `FileStorage\CustomEmotion`。**4.x 没有这个顶层目录**，表情分散在三处：
- `cache\<年-月>\Emoticon\`（缓存，可删）
- `db_storage\emoticon\emoticon.db`（表情库索引，不能删）
- `business\emoticon\`（常为空）

搜索时用 `find <root> -iname "*emoticon*" -o -iname "*emoji*"` 才能找全。

## 三、分档清理策略

清单一律按四档组织，用户一眼能看懂风险：

| 档位 | 内容 | 删除后果 | 量级 |
|---|---|---|---|
| **1-安全** | `log/*.xlog`、`radium/web/profiles/*/Cache`、`cache/`、`temp/`、`xworker/search` | 无感，微信自动重建 | 300 MB+ |
| **2-较安全** | `xworker/mpxworker`、`xworker/liteapp`、`applet/publicLib`、`applet/codecache` | 不丢数据，小程序首次打开变慢 | 200 MB+ |
| **3-会丢内容** | `msg/video`、`msg/attach` | ⚠️ 旧聊天里视频变「已过期」、图片可能永久丢失 | 视用户 |
| **0-保护** | `msg/file`、`db_storage`、`config`、`all_users/login`、`resource`、`xplugin/plugins` | 丢内容 / 功能损坏 / 需重登 | 最大 |

**第 2 档里的例外**：`applet/data/` 是小程序可能写入的本地数据，**标"建议保留"**，不要跟 `publicLib`/`codecache` 一起清。

**必须主动告知用户官方替代方案**：微信自带「设置 → 通用 → 存储空间」一键清缓存，效果与第 1 档基本重合，且是官方路径，更稳妥。这份清单的价值在于**透明和可控**，不是否定官方功能。

## 四、操作约束（容易踩的坑）

1. **必须先完全退出微信**。这是硬约束。
   检测方式：`Get-Process | Where-Object { $_.ProcessName -match 'wechat|weixin|wxwork' }`
   注意微信 4.x 正常运行时会有**十几个进程**（实测 5 个 `Weixin` + 9 个 `WeChatAppEx`），全退了才能清。
   运行中删除会失败（文件被占用）或导致微信异常。
2. **⚠️ 微信驻留托盘时，「优雅退出」在程序层面根本不存在**（实测踩过，浪费了两轮）。
   - Windows 微信「关闭窗口」= 最小化到托盘，不是退出。
   - 托盘态下**所有 `Weixin` 进程 `MainWindowHandle = 0`**（先探测再断言），`CloseMainWindow()` 静默失败，
     6 秒 / 12 秒后进程数纹丝不动。
   - 微信没有 `--quit` 之类命令行退出参数，也没有可用的 IPC 退出通道。
   - 只剩两条路：① **请用户从托盘右键退出**（最稳、零损失）；② 经用户**明确授权**后
     `taskkill /PID <主进程PID> /T /F`。影响面：正在输入未发送的消息、未完成的文件传输会中断；
     聊天记录（实时写本地库）、登录态、已收到的文件不受影响。
   - **绝不可未经确认就强杀**——这属于破坏性操作，必须先问。
3. **删除一律走系统回收站，且判成功只看「原路径是否消失」**（见第五节）。
4. **9 月的 profile 目录来源不明时要问**。本机曾出现用户不认识的 `ttprofile`、`wxprofile`。
5. **不要用 `du -sh <子目录>/*/` 后直接接 `awk/sed` 裁剪**——容易把体积数字吃掉只剩路径。
   稳妥写法：`for d in */; do printf "%-8s %s\n" "$(du -sh "$d" | cut -f1)" "$d"; done`
6. **路径拼接别用 `r"...\users\\"`**。raw string 末尾双反斜杠会输出成两个 `\`。
   正确写法：`r"\radium\users" + "\\" + uid`。
7. PowerShell 工具在本机不回显 stdout —— 要拿结果就写进 `$env:TEMP\xxx.txt` 再用 Bash `cat` 读。
   且 Bash 里**不能调 powershell/cmd**（会被安全策略拦）；但 `tasklist | grep -i weixin` 这类直接调用是可用的。

## 五、执行阶段：安全删除到回收站

用户确认范围后才进这一节。核心原则：**走系统回收站（可还原），绝不 `rm` / 永久删除**。

### 5.1 用 `SHFileOperationW` + `FOF_ALLOWUNDO`

本机 **PowerShell 的 COM 被安全策略拦截**（`Shell.Application` 用不了），`send2trash` 等库又要额外装包。
可靠做法是 Python + ctypes 直接调 shell32：

```python
class SHFILEOPSTRUCTW(ctypes.Structure):
    _fields_ = [("hwnd", wt.HWND), ("wFunc", wt.UINT),
                ("pFrom", wt.LPCWSTR), ("pTo", wt.LPCWSTR),
                ("fFlags", ctypes.c_uint16), ("fAnyOperationsAborted", wt.BOOL),
                ("hNameMappings", ctypes.c_void_p), ("lpszProgressTitle", wt.LPCWSTR)]

FO_DELETE = 3
FLAGS = 0x0040 | 0x0010 | 0x0004 | 0x0400 | 0x0200
#        ALLOWUNDO NOCONFIRMATION SILENT NOERRORUI NOCONFIRMMKDIR

def to_recycle_bin(paths):
    ctypes.windll.ole32.CoInitialize(None)      # 必须先初始化 COM（STA）
    src = "\0".join(paths) + "\0\0"             # pFrom 必须双 null 结尾
    buf = ctypes.create_unicode_buffer(src)
    op = SHFILEOPSTRUCTW()
    op.wFunc = FO_DELETE
    op.pFrom = ctypes.cast(buf, wt.LPCWSTR)
    op.fFlags = FLAGS
    return int(ctypes.windll.shell32.SHFileOperationW(ctypes.byref(op)))
```

脚本设计成 `dry`（预览清单，不动文件）/ `run`（执行）两种模式，先跑 dry 给用户核对，再 run。

### 5.2 ⚠️ 两个必须知道的坑

**① 返回值不可靠 —— 千万别用它判定成败。**
实测：10 个文件 + 11 个目录确实全部进了回收站，但其中 14 次调用返回码是 `2`（ERROR_FILE_NOT_FOUND）。
**唯一可信的判定是 `os.path.exists(原路径)` 是否变成 False**，返回码只当参考信息打日志。
（这个坑不修，会得出「14 项失败」的错误结论。）

**② 验证「真的进了回收站」只能解析 `$I` 元数据。**
- **不要**用 `find <回收站> -mmin -60` 过滤 —— **回收站保留原文件的 mtime**，
  刚删掉的目录 mtime 可能是几天前，会全部漏掉，让人误判成「没进回收站」。
- 正确做法：读 `C:\$Recycle.Bin\<SID>\$I*`，里面是 **UTF-16LE 编码的原始完整路径**。
  抽可打印片段后搜 `xwechat` / `radium` 等关键字即可逐条对上。
- 回收站里 `$I<id>` 是元数据、`$R<id>` 是实际内容（**保留原扩展名**，如 `$R5PL55G.xlog`），
  两者配对存在 = 可还原。目录类条目的 `$I` 记录的是目录自身大小（偏小），属正常。
  递归删除的目录，其**内部文件也会各自生成 `$I` 条目**，条目数远多于删除项数。
- 交叉验证技巧：`du -sh C:\$Recycle.Bin` 删除前后差值 ≈ 本次删除体积（实测 5.4G→5.6G 对应 203 MB）。

### 5.3 目录粒度的讲究（错了会伤微信）

- `log/` **只删 `*.xlog`，必须保留 `mm.mmap3` / `mm1.mmap3` / `update.mmap3`**
  —— 这三个是内存映射索引文件，不是日志。
- `web/profiles/<profile>/Cache` —— 删 `Cache` 子目录本身，**父级 profile 目录必须保留**。
- `xworker/`、`applet/` 母目录必须保留，只删其中的 `search`/`mpxworker`/`liteapp`/`publicLib`/`codecache`；
  删完必须 `ls` 母目录确认结构完整。
- **分批执行**（每批 ≤10 项，天然按档位切：第 1 档一批、第 2 档一批），每批删完立刻核验：
  ① 目标原路径已消失 ② 母目录还在 ③ 保护项完好。
- 删除前后各测一次**全量体积**（程序目录 + 聊天目录），用差值证明效果。
- 核验保护项时**注意路径别写错**：用 `$R/../../all_users` 这种相对跳级会指到错误位置，
  制造出「登录态不见了」的假警报。直接用绝对路径。

### 5.4 交付执行记录（不只是口头汇报）

删完必须给一份**执行记录** Excel，至少四块：
① 执行摘要（项数 / 回收体积 / 成功失败数 / 删除方式 / 进程处理方式）
② 逐项执行明细（删除前**实测**体积 + 文件数 + 结果）
③ 保留项核验（删除后实测，证明零误伤）
④ **回收站凭据**（`$I` 解析抽样，逐条列出原始路径）

并且必须明确说清一件事：**空间还没真正释放** —— 数据仍占着回收站，
确认微信使用无异常后清空回收站才会真正腾出空间。这是用户最容易误解的点。

## 六、交付物

用户偏好：表格类物料用 **Excel**（不是 Word），带勾选列。

Excel 三个 sheet：
1. **清理确认清单**：`是否执行 | 序号 | 档位 | 项目 | 完整路径 | 体积 | 类型 | 是什么 | 删除后果 | 建议`
   - 档位列用色块：安全=绿 `C6EFCE`、较安全=黄 `FFEB9C`、会丢内容=红 `FFC7CE`、保护=灰 `D9D9D9`
   - 「是否执行」列留空供用户填「删」
   - 开 `freeze_panes` 和 `auto_filter`
2. **空间汇总**：扫描范围 / 可清理统计 / 结论三段，都写实测值
3. **运行状态与须知**：进程清单 + 操作须知（先退微信、分档说明、官方替代方案）

数据集里的**所有体积必须是 `du` 实测值**，不允许估算或写"约"以外的模糊表述。

## 七、收尾清单

**扫描阶段**
- [ ] 确认**没有删任何文件**（除非用户已明确勾选确认）
- [ ] Excel 能打开、路径无双反斜杠、保护项确实没进可删列表
- [ ] README.md 写清「尚未删除，等待确认」的状态
- [ ] 弹资源管理器文件夹 + 给交付卡片
- [ ] 工作区日志记下实测体积与目录地图

**执行阶段**
- [ ] 执行前已展示完整路径清单 + 说明回收站方案 + 用户明确确认范围
- [ ] 微信进程已归零（或已获授权强杀）
- [ ] 每批 ≤10 项，按档位切批，逐批核验
- [ ] **逐项核验目标原路径已消失**（不看返回码）
- [ ] **逐个核验保护项完好**，尤其 `msg/file` 的文件数要能对上（实测 17 个）
- [ ] 母目录结构完整（`xworker/`、`applet/`、`web/profiles/` 都还在）
- [ ] 回收站 `$I` 解析抽样，证明可还原
- [ ] 交付执行记录 Excel，并把 README 状态从「待确认」改成「已执行 + 结果」
- [ ] **口径说清**：空间未真正释放，确认无异常后清空回收站才生效
- [ ] 弹资源管理器文件夹 + 给交付卡片
