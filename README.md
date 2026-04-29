# 189mail-skill
使用中国电信 189 邮箱处理邮件任务；当用户要求查看 189 邮箱、搜索/阅读邮件、整理未读邮件、检查邮箱设置、通过 189 邮箱拟稿或发送邮件时触发。优先使用官方 IMAP/SMTP；需要网页操作时使用官方 Webmail 入口并让用户完成人机验证。
本 skill 在OpenClaw 中使用方法：
- 将 skill 文件（解压后） 放置在openclaw相应 skill目录。如 workspace/skills 中。
- 刷新 OpenClaw  skill 后，在 openclaw tui 或 dashboard 中 的对话框中输入：/189 mail “相应对邮件的处理 要求”，即可调用技能，访问 189 邮箱。 
- 初次使用，IMAP / POP 3 / SMTP 方式需设置环境变量，或网页登录免密码模式。
