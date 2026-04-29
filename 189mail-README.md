# 189mail Skill 实现说明

本目录是 OpenClaw 的 `189mail` skill，用于处理中国电信 189 邮箱：查看邮件、搜索/读取邮件、SMTP 发信，以及通过网页登录免密码操作。

官方入口：<https://webmail30.189.cn/w2/index.html>

---

## 1. 当前实现结构

```text
189mail/
├── SKILL.md                  # skill 元信息、工作流、安全边界
├── 189mail-README.md         # 本实现说明
├── agents/
│   └── openai.yaml
└── scripts/
    ├── imap_mail.py          # IMAP/POP3/SMTP 脚本
    └── webmail_browser.py    # 网页登录 + agent-browser/CDP 操作脚本
```

---

## 2. 安全原则

- 不把邮箱密码、客户端专用密码、短信验证码、二维码、Cookie 写入仓库、记忆或日志。
- 发邮件、删除邮件、移动邮件、改规则、下载附件、公开转发内容前，必须先让用户确认。
- “免密码”模式不保存登录态；由用户在可见浏览器里完成网页登录，OpenClaw 只操作当前已登录页面。
- 邮件正文默认只输出摘要和任务相关的最小必要内容。

---

## 3. IMAP / POP3 / SMTP 脚本

入口：

```bash
python3 /Users/hejunhui/.openclaw/workspace/skills/189mail/scripts/imap_mail.py --help
```

### 凭据环境变量

```bash
export MAIL189_EMAIL="手机号@189.cn"
export MAIL189_PASSWORD="客户端专用密码/授权码"
```

可选覆盖：

```bash
export MAIL189_IMAP_HOST="imap.189.cn"
export MAIL189_IMAP_PORT="993"
export MAIL189_POP3_HOST="pop.189.cn"
export MAIL189_POP3_PORT="995"
export MAIL189_SMTP_HOST="smtp.189.cn"
export MAIL189_SMTP_PORT="465"
export MAIL189_TIMEOUT="20"
export MAIL189_TZ="Asia/Shanghai"
```

### 已实现能力

```bash
# 最近邮件：默认 auto，先 IMAP，失败后自动 POP3 兜底
python3 scripts/imap_mail.py list --limit 10

# 今天邮件
python3 scripts/imap_mail.py list --today --limit 10

# 强制 POP3 查询今天邮件
python3 scripts/imap_mail.py list --protocol pop3 --today --limit 10

# 搜索邮件（IMAP）
python3 scripts/imap_mail.py search --query "会议" --limit 10

# 读取指定 UID（IMAP）
python3 scripts/imap_mail.py read --uid 12345

# SMTP 发信，必须显式确认参数
python3 scripts/imap_mail.py send \
  --to "name@example.com" \
  --subject "主题" \
  --body "正文" \
  --yes-i-understand-send-email
```

### 实测结论

- SMTP 使用客户端专用密码可登录成功。
- POP3 SSL 可登录并读取邮件元信息，但偶尔会出现服务端内部错误，重试可恢复。
- IMAP 在当前账号上连接成功但登录阶段超时，因此 `list` 默认增加了 `--protocol auto`：IMAP 异常时自动兜底 POP3。

---

## 4. 网页登录免密码模式

入口：

```bash
python3 /Users/hejunhui/.openclaw/workspace/skills/189mail/scripts/webmail_browser.py --help
```

### 推荐流程

```bash
# 1. 打开系统 Chrome 专用窗口，并启用 CDP 远程调试端口
python3 scripts/webmail_browser.py open --system-chrome

# 2. 用户在弹出的 Chrome 窗口里自行完成 189 登录、验证码、短信或扫码

# 3. OpenClaw 连接同一个 Chrome 页面，确认邮箱页可见
python3 scripts/webmail_browser.py wait-login --cdp

# 4. 抓取当前页面可交互结构
python3 scripts/webmail_browser.py snapshot --cdp --compact
```

### 为什么使用 `--system-chrome`

最初尝试过 `agent-browser --headed`，但如果后台 daemon 已经以 headless 模式启动，`--headed` 会被忽略，用户看不到可登录页面。

因此改为：

1. 用系统 Google Chrome 打开一个可见窗口；
2. 启动参数加入 `--remote-debugging-port=9222`；
3. `agent-browser --cdp 9222` 接入同一个窗口；
4. 用户在这个窗口里登录后，OpenClaw 能读取并操作同一页面。

该方式不需要用户向 OpenClaw 提供密码。

### 常用命令

```bash
# 打开可见网页登录窗口
python3 scripts/webmail_browser.py open --system-chrome

# 连接可见窗口并等待登录后邮箱页面
python3 scripts/webmail_browser.py wait-login --cdp

# 查看页面结构
python3 scripts/webmail_browser.py snapshot --cdp --compact --depth 10

# 列出当前页面可见邮件，支持简单过滤
python3 scripts/webmail_browser.py list --cdp --today
python3 scripts/webmail_browser.py list --cdp --days 7 --unread
python3 scripts/webmail_browser.py list --cdp --days 14 --query ollama

# 直接用 agent-browser 操作已登录页面
agent-browser --session 189mail --cdp 9222 get text body --json
agent-browser --session 189mail --cdp 9222 snapshot -i -c -d 10 --json
```

---

## 5. 页面操作经验

登录成功后，页面地址一般为：

```text
https://webmail30.189.cn/w2/template/mailbox.html
```

进入收件箱后，正文文本可通过：

```bash
agent-browser --session 189mail --cdp 9222 get text body --json
```

读取到的页面通常包含：

- 今天/昨天/上周等分组；
- 发件人；
- 主题和摘要；
- 时间；
- 未读状态通常体现在 DOM class：`ml-unread`。

示例：统计当前列表内未读邮件，可用页面 JS：

```bash
agent-browser --session 189mail --cdp 9222 eval '
(() => [...document.querySelectorAll(".mail-list-bd-item")].map(el => ({
  unread: el.className.includes("ml-unread"),
  from: el.querySelector(".j-sender")?.innerText?.trim(),
  title: el.querySelector(".ml-con-title")?.innerText?.trim(),
  date: el.querySelector(".ml-time")?.innerText?.trim()
})))()
' --json
```

---

## 6. 已验证场景

### 查看今天邮件

网页登录后进入收件箱，通过页面文本确认今天邮件：

- 今天 1 封；
- 发件人：QNAP Systems, Inc.；
- 主题：Defend Your Data Against Ransomware Threats with Three Lines of Defense。

### 统计最近一周未读邮件

通过 `.mail-list-bd-item.ml-unread` 统计当前收件箱第一页的最近邮件，按 2026-04-23 至 2026-04-29 口径统计：

- 4/29：1 封
- 4/28：6 封
- 4/26：3 封
- 4/25：2 封
- 4/24：2 封
- 合计：14 封

主要来源：QNAP、GitHub/Peter Steinberger、Ollama、山姆/沃尔玛发票、Google、Strava、招商银行、Garmin、Myfxbook。

---

## 7. 本轮优化记录

已把网页登录页面解析逻辑封装成：

```bash
python3 scripts/webmail_browser.py list --cdp [--today] [--days N] [--unread] [--query 关键词] [--limit N]
```

说明：

- `--today`：只返回今天邮件。
- `--days N`：按当前日期向前 N 天过滤，例如 `--days 7`。
- `--unread`：只返回 DOM class 含 `ml-unread` 的未读邮件。
- `--query`：在发件人、主题、摘要中做大小写不敏感匹配。
- 默认只解析当前已加载/可见邮件列表，不自动翻页，避免频繁请求触发风控。
- 返回 JSON：`count` + `items`，每项包含 `from/title/abstract/dateText/date/unread/group`。

验证命令：

```bash
python3 scripts/webmail_browser.py list --cdp --days 7 --unread --limit 5
```

已验证可正常输出最近 7 天未读邮件列表。

## 8. 后续改进方向

- 支持自动分页扫描，但要控制频率，避免触发 189 风控。
- 支持打开单封邮件并只读取摘要，不默认展示完整正文。
- 支持可恢复删除：按关键词选择邮件并移入垃圾箱，执行前输出匹配清单。
- 如需持久化登录态，应先和用户确认；默认仍不保存 Cookie。
