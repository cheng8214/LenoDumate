# 百度搭子 DuMate 每日签到（Leno 实现）

**百度搭子 DuMate（千帆桌面客户端）每日签到**：
读本机客户端登录态 → AES-256-GCM 解密 cookies → 调 console.bce.baidu.com 的签到接口。

## 用法

CLI：

```
leno dumate_sign.leno              # 签到（默认；今天已签到会提示无需重复）
leno dumate_sign.leno status       # 只查状态，不签到（= -s = --status）
leno dumate_sign.leno health       # 登录态健康检查（剩余天数）
leno dumate_sign.leno credits      # 积分套餐与到期时间（按最近到期排序）
leno dumate_sign.leno diag         # 诊断：账号/cookie 数/请求头长度（不打印 cookie 值）
leno dumate_sign.leno json         # 结果输出一行 JSON（便于脚本调用）
```

GUI（窗口版，复用同一份核心 `dumate_core.leno`）：

```
leno dumate_gui.leno               # 正常开窗：打开即自动拉取今日状态/登录态/积分明细；其余动作见下
```

GUI 的自动化与无头验证：

| 环境变量 | 作用 |
| --- | --- |
| `DUMATE_GUI_AUTO=N` | 跑 N 帧后打印摘要退出（无头冒烟；配合 `SDL_VIDEODRIVER=dummy`） |
| `DUMATE_GUI_SELFTEST=1` | 离线逐项自检后退出（不联网、不写真实配置） |
| `DUMATE_GUI_NO_NET=1` | 不联网（动作只报"离线"，验证界面与状态机） |

积分明细用 **Table 控件**（四列：类型 / 积分 / 到期 / 剩余天），点表头可排序，斑马纹，永久有效显示"永久"。

GUI 里**定时签到**是"程序在跑即生效"——到点且今天还没签过才发一次请求（按日期去重，一天只签一次），
配置持久化到脚本同目录的 `dumate_gui_cfg.json`。

**打开即自动拉取**：窗口一打开就自动查今日状态、登录态、积分明细（填进表格），不用挨个点按钮。

**界面不卡**：所有网络请求（打开时自动拉取 / 点按钮 / 定时签到）都在**后台线程**跑，结果经
`Channel` 回传、由主循环每 150ms 排空一次 ⇒ SDL 事件循环从不等 HTTP ⇒ 窗口任何时候都能拖能点
（此前是同步请求，开窗时三个串行请求会把窗口按到"未响应"，点一次签到同样要冻三个请求的时间）。

**托盘**（驻留系统托盘，参照 LenoWorkBuddy 的 `wb_tray.leno` / LenoTrae 的 `trae_tray.leno` 实现）：
- 右键托盘图标 = 弹菜单（显示/隐藏窗口 · 立即签到 · 查状态 · 开机自启动 · 退出）；
- 左键图标 = 直接显示/隐藏窗口；
- **关窗默认收进托盘而非退出**（双击托盘图标或菜单"显示窗口"再开）；
- 开机自启写入的命令行带 `--tray` ⇒ 启动即只留托盘，不打扰用户。

图标：`dumate.ico`（exe/应用图标，由 `resource.toml` 的 `[pack] icon` 引用）与
`images/dumate.png`（托盘图标），二者均由 `tools/gen_dumate_icon.leno` 生成（圆底 + 白色对勾）。

环境变量：

| 变量 | 作用 |
| --- | --- |
| `DUMATE_APP_DIR` | 覆盖登录态目录（默认 `%APPDATA%\qianfan-desktop-app`；测试用） |
| `DUMATE_API_BASE` | 覆盖接口地址（默认 `https://console.bce.baidu.com`；指向本地 stub 可离线回归） |

## 签到流程

1. 读 `%APPDATA%\qianfan-desktop-app\auth.json`，取 `activeProfileId` 对应账号的 `encryptedCookies`；
2. 读同目录 `.cookie-key`（32 字节 AES-256 密钥，**二进制读取**——`files.read` 是文本语义会剥 BOM）；
3. base64 解码后按 `iv(12) + authTag(16) + ciphertext` 拆帧，走 `Crypto.gcm_open_parts`（AES-256-GCM）
   解密出 cookies JSON——**标签不符一律抛错**，绝不交付未认证数据；
4. 按域名匹配（`console.bce.baidu.com`）拼 `Cookie` 请求头，取 `bce-user-info` 的值作 `csrftoken`；
5. `GET /api/dumate/points/loginBonusInfo`：`signInDays` 含今天 ⇒ 今日已签到；
6. 未签且非只读 ⇒ `POST /api/dumate/points/loginBonus`，再查一次 info 二次确认。

## 安全边界（三条硬规矩）

1. 只**读**登录态文件，绝不改写；
2. **永不打印 token / cookie 值**（diag 只打印长度与计数）；
3. 密钥只在本次运行内存里使用，不落盘、不缓存。

## 目录结构

```
dumate_core.leno    核心：登录态解密 / 请求头 / 接口调用 / 签到·健康·积分（CLI 与 GUI 共用）
dumate_sign.leno    CLI 入口（参数解析 + 输出格式）
dumate_gui.leno     GUI 入口（SDL3 窗口；打开即自动拉取状态/健康/积分；自愈式离线自检 + 无头冒烟）
dumate_tray.leno    托盘 + 自绘菜单 + 开机自启（系统集成层，参照 wb_tray.leno / trae_tray.leno）
tools/
  gen_dumate_icon.leno  生成 Dumate 图标（圆底 + 白色对勾），输出 images/dumate.png
images/
  dumate.png       托盘图标（由 gen_dumate_icon.leno 生成）
dumate.ico         exe / 应用图标（由 tools/png_to_ico.leno 从 PNG 转来）
test/
  dumate_stub.leno  本地 stub 服务端（纯 Leno，sockets；场景由 DUMATE_STUB_MODE 控制）
  test_dumate.leno  离线回归：72 项（纯函数单测 + 自制 fixture 加解密往返 + stub 端到端 + CLI 冒烟）
dumate_gui_cfg.json  GUI 自动化配置（自动生成，可删）
dumate_gui_cfg.selftest.json  自检专用配置（DUMATE_GUI_SELFTEST=1 时用 —— 自检不碰真实配置）
```

## 测试

```
leno test\test_dumate.leno        # 72 项全过即通过
```

覆盖：cookie 域名匹配与引号转义、`signInDays` 今日判定、永久积分哨兵、AES-256-GCM 加解密往返
（fixture 现场加密， iv+tag+ct 布局与真实客户端一致）、未签→领取→二次确认、已签、领取被拒、
登录过期、dry 只读（含"只读不偷偷提交"验证）、积分排序、CLI 子进程端到端。

## 两处实现说明

| 点 | 说明 | 本实现 |
| --- | --- | --- |
| `totalPoints` 为 0 | 接口给 0 时不拼"（累计 0 积分）"后缀 | 按"无累计"处理（`total_points()` 把 0 归一成空） |
| 积分 `expireDate` 给到 253370764800（约公元 9998 年的"永久"哨兵） | `times.format` 对超大值返回空串 | 格式化不出来就如实展示"永久"，不扔异常、不编日期 |

## 依赖

- LenoLang ≥ 0.2.2（`leno`，带 `LenoCrypto` 的 AES-256/GCM 与 `LenoWeb` 的 HTTP）
- 本机已登录百度搭子桌面客户端（登录态在 `%APPDATA%\qianfan-desktop-app`）
