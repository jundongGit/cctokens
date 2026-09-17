# cctokens

查看 Claude Code 的 token 消耗按项目分布。

纯本地工具：只读 `~/.claude/projects` 下的会话日志，**不联网、不调 API、
不需要 Claude Code 能运行**。额度用满、Claude Code 已经跑不动的机器上照样可用。

只依赖 Python 3.8+（macOS 与主流 Linux 自带），无第三方包。

## 安装

方式一，从仓库取：

```bash
mkdir -p ~/bin && curl -fsSL https://raw.githubusercontent.com/jundongGit/cctokens/main/cctokens -o ~/bin/cctokens && chmod +x ~/bin/cctokens
```

方式二，无网络环境：把 `install-oneliner.txt` 的内容整行粘贴到目标机器终端执行，
脚本内嵌在命令里，不需要访问任何服务器。

`~/bin` 不在 PATH 时用全路径 `~/bin/cctokens` 调用，或在 `~/.zshrc` 追加
`export PATH="$HOME/bin:$PATH"`。

## 用法

```bash
cctokens                    # 最近 7 天
cctokens --days 30          # 最近 30 天
cctokens --month            # 本月
cctokens --today            # 今天
cctokens --since 2026-09-01 # 指定起始日期
cctokens --all              # 全部历史（数据量大时较慢）
cctokens --sessions         # 额外列出最烧 token 的单次会话
cctokens --auto             # 识别疑似定时 / 无人值守任务
cctokens --json             # 输出 JSON，供脚本消费
cctokens --top 30           # 项目排名显示条数，默认 20
cctokens --dir ~/.claude    # 指定 Claude 配置目录
```

## 输出

```
  Claude Code Token 用量 · 最近 7 天
  2026-09-11 ~ 2026-09-17   来源 /Users/you/.claude/projects
──────────────────────────────────────────────────────────────────
  总计       5.5B  tokens        13,110 次请求        25 个项目
  输入      51.1K   输出     10.0M   缓存写    103.1M   缓存读      5.4B
──────────────────────────────────────────────────────────────────

  按项目

   1. ~/Dev/project-a                     1.2B  22.1% ██████
   2. ~/Dev/project-b                   978.5M  17.7% █████
   ...

  按天 / 按模型 / 最烧 token 的会话
```

## 定位定时任务的消耗

`--auto` 用来回答「额度是不是被后台定时任务悄悄吃掉了」。

```bash
cctokens --days 60 --auto
```

判据有四条，命中任意一条即列出：

1. **非交互入口** —— `entrypoint` 含 sdk / headless / cron / api。人坐在前面用的
   `cli`、`claude-desktop`、`vscode` 不算
2. **同一 prompt 跨天重复** —— cron 每次都发同样的 prompt
3. **触发时刻规律** —— 多次触发落在同一分钟窗口（±5 分钟）内
4. **无后续人类回合** —— 全程只有最初那一条输入

输出示例：

```
  疑似定时 / 无人值守任务

      5.3M   0.0%   入口 sdk-cli / 跨 5 天重复 5 次 / 固定时刻 10:17 前后
           prompt: 运行 linkedin-daily-post skill：...
           触发: 08-28 10:17:07  09-02 10:17:05  09-03 10:17:05 ...
           项目: ~/Claude_Projects/USA-trip
```

没命中时会打印在目标机器上排查定时任务的命令清单。

**注意**：`crontab -l` 查不全。macOS 上定时任务常配在 launchd
（`~/Library/LaunchAgents/*.plist`），Linux 上可能在 systemd timer。
`--auto` 是从用量日志反向发现，不依赖调度器配置在哪，这是它的价值。

交叉验证：

```bash
crontab -l; sudo crontab -l              # cron
ls ~/Library/LaunchAgents                # macOS 用户级
launchctl list | grep -v com.apple       # macOS 运行中（第 2 列是退出码，非 0 即失败）
systemctl --user list-timers             # Linux
```

## 统计口径

- 项目路径取自日志里的 `cwd` 字段，不是目录名回推，带 `&`、空格的路径也准确
- 按 `requestId` 去重，同一请求写多行不会重复计数
- 日期按**本地时区**归属（日志里存的是 UTC）
- token 数 = 输入 + 输出 + 缓存写 + 缓存读，与 `ccusage` 的 Total 列逐字段一致

**与订阅额度的关系**：这里统计的是日志记录的 token 数，与订阅额度的扣减口径不同，
不能用它推算还剩多少额度。查额度剩余在能运行的机器上敲 `/usage`。

## 与 ccusage 的区别

[ccusage](https://github.com/ccusage/ccusage) 是更完整的工具，会把用量折算成
美元等价成本，并支持 Codex / Qwen 等其它 CLI。

cctokens 只做一件事：**按项目看 token 花在哪**。差别：

- 无需 npm / node，Python 单文件，可粘贴安装
- 按项目路径分组（ccusage 按 session id 分组，看不出是哪个项目）
- 不折算成本——成本折算需要跟着各模型价目表更新，那件事交给 ccusage

## License

MIT
