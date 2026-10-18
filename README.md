# Auto Hus

自动续期 [hus.co.id](https://hus.co.id) VPS 订阅的工具：通过浏览器自动化完成登录、Cloudflare 验证、续期全流程，并在遇到验证码拦截时自动更换 WARP IP 重试，最后通过 Telegram 推送结果、VPS 信息与截图。

## 工作原理

1. 用 **DrissionPage** 启动 Chromium（Linux 下由 `pyvirtualdisplay`/Xvfb 提供虚拟显示），打开 hus.co.id 面板
2. 通过 Telegram Widget 登录：脚本用 **Telethon** 监听授权对话并自动点击 `Confirm` 完成授权
3. 自动处理 **Cloudflare Turnstile** 挑战
4. 枚举账户下所有 VPS，逐一执行续期
5. 验证码自适应求解：
   - **reCAPTCHA**：DrissionPage 进入音频 iframe，下载音频后用语音识别（speech_recognition）求解
   - **算术验证码**：ddddocr OCR 识别
6. 若验证码被服务端拒绝，自动**重启 WARP 更换出口 IP** 后重新提交（多层重试，见下方调参）
7. 续期结果检测基于结果页 `page.html` 正则：
   - `#response` 内 `alert-success` → 成功
   - `#response` 内 `alert-danger` / `alert-warning` → 失败
   - 整页含 `renewed until` 作为兜底成功判定
8. 通过 **Telegram Bot** 推送续期结果、VPS 信息及截图

## 快速开始

### 环境要求

- Python 3.11+（CI 使用 3.13）
- Linux 系统（需 Xvfb 虚拟显示）
- Chromium 浏览器
- ffmpeg（reCAPTCHA 音频解码需要）

### 安装依赖

```bash
# 系统依赖
sudo apt-get update && sudo apt-get install -y xvfb chromium ffmpeg unzip curl wget jq ca-certificates

# Python 依赖（版本已固定）
pip install -r requirements.txt
```

### 配置

通过 `HUS_BATCH` 环境变量配置账号，多个账号用分号 `;` 或换行分隔。（建议：最好是一个yml一个账号，因为测试下来一个账号极端情况要跑10-20分钟的，主要是运气不好的情况下会ip不停换）

#### HUS_BATCH 格式

```
phone,tg_bot_token,tg_chat_id,tg_api_id,tg_api_hash,tg_session
```

| 字段 | 必填 | 说明 |
|---|---|---|
| `phone` | ✅ | 手机号，格式 `+国家码-号码`，如 `+506-60012545` |
| `tg_bot_token` | ✅ | 用于推送通知的 Telegram Bot Token |
| `tg_chat_id` | ✅ | 接收通知的 Telegram Chat ID |
| `tg_api_id` | ✅ | 该账号对应的 Telegram API ID（https://my.telegram.org） |
| `tg_api_hash` | ✅ | 该账号对应的 Telegram API Hash |
| `tg_session` | ✅ | 该账号的 Telegram StringSession（运行 `setup_tg_session.py` 生成） |

> TG 凭据（后三个字段）由每个账号独立提供，互不影响。

#### 示例

单账号：
```
+506-60012545,7126463574:AAH...,453472000,34604959,96c51d9e7a31...,1AZWar...
```

多账号（分号分隔，建议：最好是一个yml一个账号，因为测试下来一个账号极端情况要跑10-20分钟的，主要是运气不好的情况下会ip不停换）：
```
+506-60012545,7126463574:AAH...,453472000,34604959,96c51d9e7a31...,1AZWar...;+503-72796921,7200000001:AAF...,453472001,34604960,abc123def456...,2BVta...
```

#### 其他环境变量

| 变量 | 必填 | 默认值 | 说明 |
|---|---|---|---|
| `SCREENSHOT_DIR` | ❌ | `output/screenshots` | 截图保存目录（CI 中设为 `screenshots`） |
| `DEBUG_FLAG` | ❌ | `0` | 设为 `1` 输出调试日志 |
| `HUS_HY2_PROXY_URL` | ❌ | - | 代理 URL（Secret，CI 中映射为 `PROXY_CONTENT` 环境变量，由 `gen_singbox_config.py` 转 sing-box 配置） |
| `SOCKS_PORT` | ❌ | `10808` | 本地 SOCKS 代理端口（sing-box 监听） |
| `PROXY_RETRY_COUNT` | ❌ | `3` | 代理启动失败的重试次数 |
| `MAX_LOGIN_RETRIES` | ❌ | `1` | 登录重试次数 |
| `MAX_RENEW_RETRIES` | ❌ | `5` | 外层续期流程重跑次数（实际执行 = 该值 + 1） |
| `MAX_WARP_RETRIES` | ❌ | `20` | 中层：验证码被拒后「换 WARP IP → 重新提交」循环次数 |
| `MAX_WARP_ROTATE_ATTEMPTS` | ❌ | `20` | 内层：单次换 IP 内部 `warp-cli` 重注册轮数 |
| `WARP_CMD_TIMEOUT` | ❌ | `10` | 单条 `warp-cli` 命令超时秒数（超时强杀进程组防止卡死） |

### 获取 Telegram API ID 和 API Hash

1. 访问 https://my.telegram.org 用你的tg手机号登录（会收到 Telegram 验证码）
2. 点击 **API development tools**
3. 填写表单（App title / Short name 随便填，如 `husauto`）
4. 提交后页面显示 **App api_id** 和 **App api_hash**

> ⚠️ api_id 和 api_hash 是固定的，创建一次后不要重复创建，否则旧的会失效。

## Telegram Session 生成

每个 TG 账号需要独立的 session string。通过 `setup_tg_session.py` 生成：

### 用法

单次交互式运行（同一进程内完成发码与登录，**不接收命令行参数**，验证码/密码通过运行时的交互提示输入）：

```bash
HUS_BATCH='phone,tg_bot_token,tg_chat_id,tg_api_id,tg_api_hash,' \
  python3 setup_tg_session.py
```

- 第 6 字段（`tg_session`）留空即可，脚本会自动生成并输出带 session 的完整行
- 运行后脚本会发送验证码，并依次提示：

```
请输入验证码:
请输入两步验证密码（若无直接回车）:
```

- 若 `HUS_BATCH` 第 6 字段已填入旧 session，脚本会先校验其是否有效；有效则直接输出原行，失效才重新登录

### 输出

脚本会输出完整的 HUS_BATCH 行（第 6 字段已填入生成的 session），直接复制替换即可：

```
==================================================
复制以下内容替换 HUS_BATCH 中对应的行（第6字段为 session）:
==================================================
+506-63532545,7126463574:AAH...,453472000,34604959,96c51d9e7a31...,1AZWar...
==================================================
```

## GitHub Actions 自动化

项目内置 GitHub Actions 工作流 `.github/workflows/hus-renew.yml`。

### 调度

```yaml
on:
  schedule:
    - cron: '06 1,13 * * *'   # 每天 UTC 01:06 与 13:06 各执行一次
  workflow_dispatch:          # 支持手动触发
```

还支持 `workflow_dispatch` 手动触发。

### 仓库结构

工作流会先检出**公开仓库**（本仓库），再用 `PRIVATE_REPO_TOKEN` 检出**私有仓库**（含 `hus-renew.py`、`gen_singbox_config.py`、`requirements.txt`）到子目录运行。

| 变量 / Secret | 必填 | 说明 |
|---|---|---|
| `HUS_BATCH` | 是 | 账号列表，格式见上 |
| `HUS_HY2_PROXY_URL` | 否 | 代理 URL（映射为 `PROXY_CONTENT` 环境变量） |
| `SOCKS_PORT` | 否 | SOCKS 端口（默认 10808） |
| `PRIVATE_REPO_TOKEN` | 是 | 用于检出私有仓库的 Personal Access Token |
| `PRIVATE_REPO`（工作流 env） | 否 | 私有仓库坐标，默认 `jyucoeng/auto_hus_private` |
| `PROJECT_DIR`（工作流 env） | 否 | 私有仓库检出子目录，默认 `hus` |

### 工作流行为

- 执行前随机延迟 0–120 秒，避免请求集中
- 启用 **WARP**（`fscarmen/warp-on-actions`）提供出口 IP，并在验证码被拒时轮换
- 配置 `HUS_HY2_PROXY_URL` 时自动下载并启动 **sing-box** 作为浏览器代理
- 截图与 `hus-renew.log` 作为 Artifact 上传，保留 7 天
- 运行结束后清理本地截图目录
- 每次执行后将时间戳写入 `<PROJECT_DIR>/time.txt` 并推送到仓库（带并发重试）
- 自动清理超过 24 小时且已完成的旧工作流运行记录，保留未完成的与最近 24 小时内的

## 代理配置

通过 Secret `HUS_HY2_PROXY_URL` 传入代理 URL，由 `gen_singbox_config.py` 自动转换为 sing-box 配置（脚本读取 `PROXY_CONTENT` 环境变量）。支持协议：

| 协议 | 传输层 | TLS 选项 |
|---|---|---|
| VLESS | TCP / WebSocket / gRPC / HTTPUpgrade / SPLITHTP | Reality / TLS / None |
| VMess | TCP / WebSocket / gRPC / HTTPUpgrade | TLS / None |
| Trojan | TCP / WebSocket / gRPC | TLS |
| Shadowsocks | - | - |
| Hysteria2 | - | TLS |
| TUIC | - | TLS |
| AnyTLS | - | TLS |
| SOCKS5 | - | - |

未配置代理时脚本以 WARP/直连模式运行。

## 项目结构

```
.
├── .github/workflows/
│   └── hus-renew.yml             # 自动续期工作流
├── hus-renew.py                  # 主程序：登录、续期、验证码、WARP 轮换、通知
├── setup_tg_session.py           # TG Session 生成脚本
├── gen_singbox_config.py         # 代理 URL 解析与 sing-box 配置生成
└── README.md
```

## 注意事项

- 脚本在 Linux 下通过 Xvfb 虚拟显示运行 Chromium（DrissionPage），无需图形界面
- 浏览器自动化使用 **DrissionPage**（Chromium 内核），Cloudflare 绕过依赖其 undetected 模式
- 手机号在日志和 TG 通知中会被部分遮蔽（如 `+506-63**2545`，国家代码后前 2 位可见）
- reCAPTCHA 解决依赖音频语音识别（speech_recognition + ffmpeg），部分 IP 可能被 Google 限制
- 算术验证码通过 ddddocr OCR 识别，成功率较高
- Telegram 登录授权通过 Telethon 后台线程自动点击 `Confirm` 完成
- 每个 TG 账号的 API 凭据独立，互不影响
- 续期结果检测基于结果页 DOM（`#response` 的 alert 样式 + `renewed until` 兜底），不依赖 JS 执行返回值
- 多账号时仅发送一条汇总报告；单账号发送各自独立报告
