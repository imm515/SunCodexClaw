# SunCodexClaw Win Mod 改安版本

`SunCodexClaw` 的这个版本是一个面向飞书的 Codex 执行机器人，定位为 `Win Mod 改安版本`。它把飞书消息、本地工作区、附件处理、会话恢复、进度反馈和 Windows 控制脚本连成一条可运行的执行链。


## 2026-04-22 最近更新

这次公开库不再继续维护一套并行代码改版，而是回到 `README-first / docs-first mod fork` 的思路。

现在这份公开版主要承担的是：

- 记录我自己这条 mod 路线想解决什么问题
- 记录私库里最近真正验证过的方向、日志、目的和意义
- 给作者和后来者一个足够清楚的思路入口
- 如果作者认同，再由作者按更专业的方式补实现代码

一句话说，公开库以后默认更像“我的 mod 版本说明书”，而不是“我要长期自己维护的第二套代码库”。

## 2026-04-22 这次补充什么

1. 公开维护方式改了
   - 默认不再维护和私库并行的一整套公开代码改版。
   - 以后公开侧优先维护 README、plan、examples、logs 和 rationale。
   - 私库仍然是唯一真实运行和验证的代码来源。

2. 这份 README 恢复回我原来自己写的 mod 版本结构
   - 不再只是跟着作者 README 做小修小补。
   - 重新保留我自己对这个项目的使用视角、交互视角和改造重点。

3. 最近私库这波已经验证过的重点，公开侧只做说明，不强行复制代码
   - 会话/线程这条链路更强调当前线程延续，而不是把公开库继续做成第二套长期维护实现。
   - 公开侧后续更适合记录“为什么这样改、实际有什么帮助、典型日志和例子是什么”。

## 现在有什么

- 支持飞书 `text`、`post`、`image`、`file`、`audio`
- 支持显式新会话和旧会话恢复：`/new`、`新任务`、`/resume <session id>`
- 支持“先暂存，后启动”的批次处理
- 支持会话短号、业务别名、session 路由
- 支持云文档进度和本地 watcher 通知
- 支持 Windows 下多账号统一启停、状态、日志、自动启动

## 代码里实际支持的交互

### 1. 新任务

```text
新任务
帮我检查 watcher 为什么没有推送完成通知
```

代码处理结果：

- 首行会被识别为新会话命令
- 后面的正文会继续作为同一轮 prompt
- 当前线程会清空旧 history，并等待新的 Codex session 绑定

### 2. 恢复旧会话

```text
/resume abc123
继续刚才的排查，并把控制脚本也一起检查
```

代码处理结果：

- `abc123` 会先按短号或完整 `session id` 匹配
- 匹配到旧 session 后，正文会继续发到该 session
- 回复文本里会补上 session 短号和业务别名，方便后续继续追踪

### 3. 附件先暂存，再显式开始

```text
@机器人
[连续发送文件]
over, 先按文件里的顺序整理问题
```

代码处理结果：

- 文件不会在收到时立刻执行
- 当前批次会先写入待处理清单
- `over, ...` 会触发批次激活，并把后面的文字一起并入本轮输入

### 4. 打断时不直接覆盖正在执行的任务

```text
@机器人
这个问题先别停，另外把这两张图也记进去
```

代码处理结果：

- 如果当前任务还在执行，新输入不会直接冲掉旧任务
- 文本和附件会先进暂存批次
- 等你显式发送 `开始处理` 或 `over` 后才切换进去

## 最近这几天已经补进来的重点

- 明确要求先 `/new` 或 `/resume`，不再默认偷偷开新 session
- `新任务`、`新线程`、`下一个任务` 等自然语言别名支持多行正文
- `/resume <id>` 支持多行正文和短号匹配
- 模糊命令会按普通正文处理，避免误切会话
- watcher 已并入 `tools/5` 控制链路，可统一 `start/stop/status/logs`
- stage window 会落盘恢复，并按超时自动失效

## 推荐配置思路

当前更推荐把 Feishu 机器人自带的本地长期记忆关掉，把上下文延续尽量交给同一条 Codex session：

- `codex.history_turns: 0`
- `memory.enabled: false`
- 需要继续时显式发送 `/resume <短号或完整session id>`

原因很直接：

- 本地记忆会和真实 session 上下文形成两套来源，容易互相打架
- 这次修复的核心已经转到 session 绑定、恢复、短号追踪、stage window 和显式会话切换
- 对这条链路来说，优先保证“继续同一 session”通常比“再叠一层机器人记忆”更稳定

## 关键文件

- `tools/feishu_ws_bot.js`
  飞书机器人主逻辑，包含消息解析、会话控制、附件暂存、恢复与回复发送
- `tools/5/feishu_bot_ctl.ps1`
  Windows 控制入口，负责账号和 watcher 的启停、状态、日志、配置档位
- `tools/5/codex_local_watcher.js`
  本地 session watcher，负责读取 Codex session 文件并推送 webhook
- `tests/feishu_ws_bot_command.test.js`
  会话命令和最近控制流修复的回归测试

## 快速开始

```bash
git clone https://github.com/imm515/SunCodexClaw.git
cd SunCodexClaw
npm install
```

复制模板：

```bash
cp config/secrets/local.example.yaml config/secrets/local.yaml
cp config/feishu/default.example.json config/feishu/default.json
```

前台启动：

```bash
node tools/feishu_ws_bot.js --account default
```

Windows 控制脚本：

```powershell
powershell -ExecutionPolicy Bypass -File tools/5/feishu_bot_ctl.ps1 -Command info
powershell -ExecutionPolicy Bypass -File tools/5/feishu_bot_ctl.ps1 -Command status -Account watcher
```

运行测试：

```bash
node --test tests/feishu_ws_bot_command.test.js
```

## 配置说明

- `config/secrets/local.example.yaml`
  放飞书密钥、Codex 参数、watcher webhook、语音转写和本地路径模板
- `config/feishu/default.example.json`
  放账号级默认行为，例如回复模式、进度模式、群聊触发规则

推荐起步值：

```yaml
codex:
  history_turns: 0
memory:
  enabled: false
  role_memory: ""
```

## 说明

- 本仓库默认不提交 `config/secrets/local.yaml`
- `.runtime/`、`conversation_records/`、日志和运行态文件都应留在你自己的机器上
- README 里的示例均来自当前代码支持的实际处理路径，不是虚构指令
