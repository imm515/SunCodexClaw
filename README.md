# SunCodexClaw Win Mod 版本

`SunCodexClaw` 的这个版本是一个面向飞书的 Codex 执行机器人，定位为 `Win Mod 版本`。它把飞书消息、本地工作区、附件处理、会话恢复、进度反馈和 Windows 控制脚本连成一条可运行的执行链。


## 2026-04-22 最近更新

现在这个公开仓库默认按 `README-first / docs-first mod fork` 维护。

也就是：

- 默认首页继续保留我自己的 mod 版本 README
- 作者原版 README 单独保存在 `README.原版.md` / `README.upstream-original.md`
- 公开侧后续优先维护 README、plan、examples、logs 和 rationale
- 私库仍然是唯一真实运行和验证的代码来源

如果作者认同这里写清楚的问题、目的和例子，再由作者按更专业的方式补代码实现即可。

这次一起补进 README 的最近行为变化：

- 默认直接续接当前线程，不再把“先选新任务还是旧会话”当成第一道门槛
- `over` / `开始处理` 继续作为暂存批次的显式提交口令，但会拒绝英文前缀混写导致的误判
- 飞书端暂存、恢复、下一批提示改成更短的移动端文案，减少长回复刷屏
- watcher 完成卡片会优先让位给飞书正式回复，避免过早弹出“已完成”造成误解

## 现在有什么

- 支持飞书 `text`、`post`、`image`、`file`、`audio`
- 支持显式新会话和旧会话恢复：`/new`、`新任务`、`/resume <session id>`
- 支持“先暂存，后启动”的批次处理
- 支持会话短号、业务别名、session 路由
- 支持云文档进度和本地 watcher 通知
- 支持 Windows 下多账号统一启停、状态、日志、自动启动

## 这个 mod 主要改什么

这版公开仓库不再强调“又维护了一套公开代码”，而是把我在私库里长期实际跑出来的工作方式讲清楚。

核心不是炫技，而是把下面几件事做稳：

- 飞书里一条线程要能长期续接，不要每次都重新选会话
- 文件、图片、语音、打断消息要先进入同一批次，等我明确说 `over` / `开始处理` 再提交
- 群聊里谁在控制当前批次、什么时候必须重新 `@机器人`，要有明确边界
- watcher、bot、控制脚本要走一条统一链路，不要分散成几套互相打架的状态
- 机器人真正长期跑起来时，完成通知、恢复通知、补发逻辑不能乱

所以这个 `mod` 更像一个“飞书工作流执行版”，重点是把会话、批次、恢复、控制和通知这些真实使用中的摩擦点压下去。

## 群聊和批次处理规则

这版和普通聊天 bot 最大的区别之一，是它把群聊里的消息处理改成了更像“工作流批次”：

- 群聊默认按一个共享会话范围处理，不是每个人各开一条看不见的隐形线程
- 群里继续推进当前批次时，通常需要明确 `@机器人`，避免旁人顺手一句话把正在处理的内容冲掉
- 如果当前批次还在运行，非打断型的新消息不会直接插进去，而是自动延后到下一批
- 当前批次还有待处理内容时，不允许随意 `resume` 混进旧会话，避免两批事情串线
- stage window 和群聊 dialog carry 会保留一段恢复窗口，超过时间再失效，不要求每次都从零开始

这套规则的目的很简单：

- 让“正在跑的事”和“下一批准备排队的事”分开
- 让群里协作时更像排队进单，而不是谁手快谁覆盖
- 让文件、补充文字、后续说明都能并入同一批，不因为消息顺序稍乱就丢上下文

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

- 默认直接接着当前线程；只有明确发 `new` / `新任务` 或 `/resume <id>` 时才切换
- `新任务`、`新线程`、`下一个任务` 等自然语言别名继续支持多行正文
- `/resume <id>` 继续支持多行正文和短号匹配
- `over` / `开始处理` 仍是显式提交口令，但像 `Alpha Over` 这类英文前缀混写会被当普通正文，避免误触发
- 飞书端暂存窗口、恢复提示、下一批提示都压缩成更短的移动端文案
- watcher 完成通知会先等飞书助手正式回复，等不到时再顺延到下一批，减少“完成提示早于正式回复”的错觉
- watcher 已并入 `tools/5` 控制链路，可统一 `start/stop/status/logs`
- stage window 会落盘恢复，并按超时自动失效

## watcher 和 tools/5 控制链路

我后面把这条链路越收越紧，不再把 watcher、bot、恢复通知、日志检查拆成多套各管一截。

现在这版的思路是：

- `tools/5/feishu_bot_ctl.ps1` 作为统一入口
- bot 和 watcher 都走同一条 Windows 控制链路
- watcher 有自己独立的控制入口和状态查看，不再混在 bot 批量操作里含糊带过
- `mini` 启动时可以顺带把 watcher 拉起来，减少“bot 起了但 watcher 没跟上”的断层
- ctl 通知和 watcher webhook 会尽量分流，避免把控制面噪音混进用户看的结果通知

公开角度真正有意义的不是“多了几个脚本命令”，而是：

- 日常看状态、启停、查日志都更像一个完整系统
- watcher 不再只是“顺带有个通知器”，而是正式进入运行链路
- 后续如果作者要按更专业方式重写，也能很清楚知道这套 mod 到底在补什么空白

## watcher 恢复和补发逻辑

这版另一个长期折腾出来的点，是“完成了”不等于“用户已经看到了”。

所以私库里后来加重的不是单纯再发几条通知，而是恢复与补发逻辑：

- watcher 会记录恢复检查点，重启后能继续回放必要的进度
- 旧的 progress 文档和 backlog 会做恢复检查，不是重启一次就当没发生过
- 正式完成通知会尽量等飞书助手正文落地，再决定卡片怎么发
- 如果完成时机和飞书回执错开，通知可以顺延，而不是提前给出一个误导性的“已完成”
- 恢复通知会尽量简短，健康状态下不靠大段 chatter 刷屏

这部分对 README 来说很重要，因为它解释了这个 mod 的真实目标：

- 不是“多发通知”
- 而是“尽量让最终看到的完成状态和真实交付状态一致”

## 长期运行配置思路

这个 mod 也不是把默认值写死就结束，而是按“长期挂着跑”来整理：

- 能力覆盖、默认模型、账号级 preset 用持久配置控制，而不是每次手工改
- `mini` 自启动和 watcher 自启动是为了让日常运行链路更稳定
- 本地长期记忆尽量关掉，把上下文延续交给真实 Codex session
- 模糊命令、歧义关键词、看似像控制词的普通正文，优先按“不要误触发”来处理

对我来说，这样的取舍比“功能看起来更多”更重要：

- 会话别乱切
- 批次别乱混
- 完成通知别抢跑
- 控制入口别分裂
- 默认值要能长期稳定地复用

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
