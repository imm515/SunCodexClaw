# SunCodexClaw Win Mod

`SunCodexClaw Win Mod` 是一个面向飞书工作流的 Codex 机器人改造版，重点在于把“会话续接、批次处理、群聊边界、恢复与完成通知”这几类真实使用中的摩擦点讲清楚并处理稳。

这个公开仓库当前以 `README` 为主，集中说明 mod 的交互方式和行为变化。上游原版说明保存在 `README.upstream-original.md`，这里默认只讨论 mod 本身。

## 2026-04-23 近期行为更新

- 群聊中的唤醒和打断进一步收紧：需要以前置 `@机器人` 的方式进入当前批次，避免普通跟帖误抢正在处理的任务。
- 附件和文本进入暂存后的确认文案已统一，移动端看到的提示更短、更直接。
- 恢复流程补了误判保护，重启后不再轻易把可恢复状态误报成“conversation records 缺失”。
- 进度文档关闭路径已重新对齐，完成、打断、恢复后的收尾行为更一致。
- 当正式回复回执还没落地时，watcher 的完成卡片会顺延一轮，减少“先看到完成提示、后看到正式回复”的错位感。
- `watcher` 与 `feishu_ws_bot.js` 的回执判断现在是配合关系，不再是各发各的通知。
- Win 版的统一脚本管理继续保留，启动、停止、状态、日志查看和常驻管理可以走同一条入口。

## 这个 mod 主要改什么

这版改造的重点不是增加更多指令，而是把飞书里的执行流程收成一条更稳定的工作流：

- 普通文本默认继续当前已绑定的 session，不再把“先选新任务还是旧会话”当成每次对话的前置门槛。
- 新任务和旧会话切换改成显式入口：`/new`、`新任务`、`新线程`、`/resume <短号或完整session id>`。
- 文件、图片、语音和处理中插入的新消息会先进入同一批次，等用户明确发送 `over` 或 `开始处理` 再统一提交。
- 群聊里的控制边界更明确，避免旁观消息、补充说明或晚到附件直接冲掉当前任务。
- 恢复、进度文档和完成通知更强调“最终交付状态一致”，而不是单纯多发几条提示。
- `watcher` 与机器人主流程之间增加了更明确的完成判定衔接，减少完成提示和正式回复错位。
- Win 版保留了统一控制脚本这条管理链路，适合长期挂着跑而不是只靠单进程前台启动。

## 当前交互规则

### 1. 会话规则

- 不带显式切换命令的普通消息，默认继续当前 session。
- 需要新开任务时，用 `/new`、`新任务`、`新线程` 等显式入口。
- 需要恢复旧会话时，用 `/resume <短号或完整session id>`。
- 如果当前批次还有待处理内容，不允许随意切去别的旧会话，避免两批事情串线。

### 2. 批次规则

- 直接发送普通文本时，可以立即开始处理。
- 如果是先发文件、图片、语音，或者在处理中再次 `@机器人` 打断，新内容会先进入暂存批次。
- 暂存批次只有在收到 `over` 或 `开始处理` 后才会正式提交。
- `over` 支持和正文同条发送，例如：`over, 继续按刚才附件里的顺序整理问题`。
- 类似 `Alpha Over` 这种普通英文短语不会被误判成提交口令。

### 3. 群聊规则

- 群聊里继续推进当前批次时，应以前置 `@机器人` 明确唤醒。
- 这样做的目的不是增加操作成本，而是把“正在执行的内容”和“围观消息或下一批补充”分开。
- 当前批次正在运行时，新的补充通常会先进暂存，而不是直接顶掉当前执行。

### 4. 恢复与完成规则

- 恢复时会优先保留可继续的上下文、附件批次和进度状态。
- 完成提示不会只看 watcher 是否先结束，还会结合 `feishu_ws_bot.js` 写出的回复回执尽量等待正式回复路径落地。
- 目标是让用户最终看到的完成状态，更接近真实交付状态，而不是抢先弹出一个“已完成”。

## Watcher 与 Win 管理亮点

### 1. watcher 不只是附带通知

- `watcher` 会跟踪 Codex session 事件，但最终完成提示不再只按 watcher 自己的完成时点硬推。
- 机器人主流程会写入回复回执，`watcher` 会据此判断是否应当继续等待正式回复落地。
- 这让“watcher 已完成”和“飞书里用户已看到正式回复”之间的关系更清楚，减少重复通知或过早完成提示。

### 2. Win 版强调统一管理

- 除了 `node tools/feishu_ws_bot.js` 直接启动之外，Win Mod 也保留了统一脚本管理入口。
- 日常启停、批量状态检查、日志跟踪和重启可以走同一套控制命令，而不是分散靠手动进程管理。
- 对长期运行场景来说，这比只展示一个启动命令更重要，因为它把 bot、watcher 和运行状态检查收进了同一条管理链路。

## 典型使用方式

### 新任务

```text
新任务
帮我检查 watcher 为什么没有推送完成通知
```

效果：

- 开启新的 session
- 同条消息里的正文继续作为本轮输入

### 恢复旧会话

```text
/resume abc123
继续刚才的排查，并把控制脚本也一起检查
```

效果：

- `abc123` 可按短号或完整 session id 匹配
- 恢复后，正文会继续发到该 session

### 附件先暂存，再显式开始

```text
@机器人
[连续发送文件]
over, 先按文件里的顺序整理问题
```

效果：

- 附件先进入同一批次
- `over` 把本批内容一次性提交处理

## 现在有什么能力

- 支持飞书 `text`、`post`、`image`、`file`、`audio`
- 支持显式新会话和旧会话恢复
- 支持“先暂存，后提交”的批次处理
- 支持会话短号、业务别名、session 路由
- 支持飞书云文档进度反馈
- 支持本地图片和文件作为飞书原生消息回发
- 支持 watcher 与正式回复回执配合的完成判断
- 支持多账号运行与统一控制脚本
- 支持长期运行场景下的启动、停止、状态、日志查看与常驻管理
- 支持微信实验接入线，用于验证扫码登录和长轮询消息通道

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

常用控制命令：

```bash
bash tools/feishu_bot_ctl.sh status all
bash tools/feishu_bot_ctl.sh start all
bash tools/feishu_bot_ctl.sh restart assistant
bash tools/feishu_bot_ctl.sh logs assistant --follow
bash tools/feishu_bot_ctl.sh stop all
```

## 配置建议

推荐先把重点放在稳定的 session 续接和批次处理上，而不是额外叠很多本地长期记忆：

```yaml
codex:
  history_turns: 0
memory:
  enabled: false
  role_memory: ""
```

适合作为起步值的原因很简单：

- 上下文延续优先交给真实 session
- 避免“本地记忆”和“当前 session”同时充当上下文来源
- 降低恢复、切换和批次处理时的歧义

## 关键文件

- `tools/feishu_ws_bot.js`
  飞书机器人主逻辑，负责消息解析、session 控制、批次处理、恢复、回复发送和回复回执写入。
- `tools/feishu_bot_ctl.sh`
  常用控制脚本，负责启停、状态和日志查看。
- `tools/5/codex_local_watcher.js`
  本地 watcher，负责跟踪 Codex session 事件，并与回复回执配合完成通知判断。
- `tools/weixin_openclaw_bot.js`
  微信实验接入线，用于验证 OpenClaw 风格的扫码登录和消息通道。
- `README.upstream-original.md`
  保留的上游原版说明，便于对照。

## 说明

- `config/secrets/local.yaml` 不应提交到仓库。
- `.runtime/`、会话记录、日志和其他运行态文件应保留在本机。
- 这里的示例以当前 mod 行为为准，重点是说明交互方式和工作流边界。
