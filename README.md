# SunCodexClaw Win 办公工作流改版

这个仓库基于 `Sunbelife/SunCodexClaw`，重点不是再包一层聊天壳，而是把飞书消息、本地工作区、附件处理、进度反馈和 Windows 多账号运维串成一条稳定可跑的执行链路。

`README.原版.md` 保留为作者原始 README，当前文件只说明这个仓库现在的行为和改动重点。

## 2026-04-22 最近更新

现在这个公开仓库默认按 `README-first / docs-first mod fork` 维护。

也就是：

- 默认首页继续保留我自己的 mod 版本 README
- 作者原版 README 单独保存在 `README.原版.md` / `README.upstream-original.md`
- 公开侧后续优先维护 README、plan、examples、logs 和 rationale
- 私库仍然是唯一真实运行和验证的代码来源

如果作者认同这里写清楚的问题、目的和例子，再由作者按更专业的方式补代码实现即可。

## 项目定位

- 保留飞书 + Codex 机器人主链路
- 更强调 Windows 办公场景下的稳定执行
- 重点放在附件暂存、打断恢复、富文本兼容、长任务反馈和控制脚本

## 关键更新

### 1. 附件先暂存，再显式激活

- 收到附件后不会立刻把文件塞进执行链
- 只有出现明确处理指令时才会激活当前批次

用户可见模板：

```text
@机器人
[上传文件/图片]
开始处理
```

实际处理：
文件或图片先进入暂存批次；出现“开始处理”这类激活文本后，当前批次里的附件信息和附带文本才会被组装进同一次 `codex exec` 输入。

### 2. 打断内容不会直接覆盖旧任务

- 新消息进来时，旧执行链不会被粗暴覆盖
- 被打断内容会先进入待处理批次，等你明确切换后再恢复

用户可见模板：

```text
开始处理
```

实际处理：
如果当前线程已有运行中的任务，新进入的内容不会直接改写当前执行上下文，而是先写入待处理批次；再次出现显式处理指令时，系统才切换到这批输入继续执行。

### 3. 富文本消息按内容解析，不只吃原始结构

- 支持 `post.zh_cn`、`post.en_us` 和顶层 `content`
- 会尽量提取正文、引用内容、图片和文件信息，继续走同一套任务入口

用户可见模板：

```text
看下这个报错
[图片]
[文件]
```

实际处理：
收到飞书 `post` 消息后，系统会优先提取正文文本，再补入引用消息内容、图片 key 和文件 token；传给模型的是整理后的任务输入，而不是原始富文本 JSON。

### 4. 线程和批次隔离更稳

- 增加本地线程隔离，减少不同轮次附件串组
- 群聊场景下更适合连续交互

用户可见模板：

```text
第 1 批：文本 + 附件
第 2 批：文本 + 附件
```

实际处理：
附件归组不只看聊天范围，还会结合本地线程和待处理批次状态分开存放；前一批未处理完时，后一批附件不会自动并入前一批输入。

### 5. 归档可以分模式管理

- 支持 `conversation_records.mode`
- 便于把不同运行形态分开留档

用户可见模板：

```json
{
  "conversation_records": {
    "mode": "off"
  }
}
```

```json
{
  "conversation_records": {
    "mode": "long_input_only"
  }
}
```

```json
{
  "conversation_records": {
    "mode": "full"
  }
}
```

实际处理：
`conversation_records.mode` 会按配置值决定记录范围：`off` 不归档，`long_input_only` 只记录长输入相关内容，`full` 记录完整过程。

### 6. Windows 运维入口更完整

- 提供多账号启停、状态、日志、自启动脚本
- 交互菜单和危险确认改成数字输入

用户可见模板：

```powershell
powershell -ExecutionPolicy Bypass -File tools/5/feishu_bot_ctl.ps1 list
powershell -ExecutionPolicy Bypass -File tools/5/feishu_bot_ctl.ps1 status assistant
powershell -ExecutionPolicy Bypass -File tools/5/feishu_bot_ctl.ps1 logs assistant
powershell -ExecutionPolicy Bypass -File tools/5/feishu_bot_ctl.ps1 profiles
powershell -ExecutionPolicy Bypass -File tools/5/feishu_bot_ctl.ps1 profile -Profile coding
```

实际处理：
`tools/5/feishu_bot_ctl.ps1` 支持 `list`、`start`、`stop`、`restart`、`status`、`logs`、`profiles`、`profile`；交互菜单和确认步骤都改成数字输入，默认确认分支是取消。

### 7. 配置写回更稳

- YAML 写回更稳定
- 更适合长期维护本地运行配置

用户可见模板：

```json
{
  "progress": {
    "enabled": true,
    "message": "已接收，正在执行。",
    "mode": "message"
  },
  "codex": {
    "model": "",
    "reasoning_effort": "",
    "profile": ""
  }
}
```

实际处理：
本地配置在补齐运行字段或更新持久化状态时，会按原有 YAML/JSON 结构回写对应节点，尽量避免把整个文件重排成难以维护的单段内容。

## 主要能力

- 飞书 WebSocket 长连接接收消息，并按账号路由到对应 `codex.cwd`
- 文本、图片、语音、文件统一进入本地执行链
- 长任务可回消息，也可持续写入飞书云文档
- 机器人支持本地长期记忆与线程级短期上下文并存
- 多账号可用统一控制面板管理启动、停止、重启、日志和状态
- Windows 启动脚本支持模型档位切换与安全确认

## 交互模板

### 1. 线程控制

用户可见模板：

```text
/threads
/thread list
/thread current
/thread new 排查支付回调
/thread switch <线程ID或名称>
/reset
```

实际处理：
系统支持列出线程、查看当前线程、新建线程、切换线程，以及清空当前线程上下文；`/reset` 只作用于当前线程。

### 2. 进度反馈模式

用户可见模板：

```json
{
  "progress": {
    "enabled": true,
    "message": "已接收，正在执行。",
    "mode": "message"
  }
}
```

```json
{
  "progress": {
    "enabled": true,
    "message": "已接收，正在执行。",
    "mode": "doc",
    "doc": {
      "title_prefix": "Codex 任务进度",
      "share_to_chat": true,
      "link_scope": "same_tenant",
      "include_user_message": true,
      "write_final_reply": true
    }
  }
}
```

实际处理：
`progress.mode = "message"` 时直接在聊天里回进度；`progress.mode = "doc"` 时会持续写入飞书云文档，并可把文档链接回发到当前会话。

### 3. 本地长期记忆

用户可见模板：

```json
{
  "memory": {
    "enabled": true,
    "role_memory": "默认使用简体中文，先执行再解释。"
  }
}
```

实际处理：
每个账号可启用一份本地长期记忆，线程内短期上下文和账号级长期记忆同时存在；长期记忆用于保留稳定偏好、角色约束和最近摘要。

### 4. 显式发送图片和文件

用户可见模板：

```text
[[FEISHU_SEND_IMAGE:output/result.png]]
[[FEISHU_SEND_FILE:output/report.xlsx]]
```

实际处理：
当回复中单独输出这类指令时，系统会把对应本地路径解析成飞书原生图片或文件消息，而不是只回一个本地路径字符串。

## 控制脚本

`tools/5/feishu_bot_ctl.ps1` 是当前仓库里最重要的 Windows 运维入口。

当前支持：

- `list`、`start`、`stop`、`restart`、`status`、`logs`
- `profiles`、`profile`
- 交互式控制面板
- 多账号统一启停
- 能力预设切换

## 目录重点

- [tools/feishu_ws_bot.js](tools/feishu_ws_bot.js)：飞书机器人主逻辑
- [tools/lib/local_secret_store.js](tools/lib/local_secret_store.js)：本地 secrets 读写
- [tools/5/feishu_bot_ctl.ps1](tools/5/feishu_bot_ctl.ps1)：Windows 控制面板
- [config/secrets/local.example.yaml](config/secrets/local.example.yaml)：本地配置模板
- [config/feishu/default.example.json](config/feishu/default.example.json)：账号默认配置模板
- [README.原版.md](README.原版.md)：作者原始 README

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

前台启动单账号：

```bash
node tools/feishu_ws_bot.js --account assistant
```

Windows 下可直接用：

```powershell
powershell -ExecutionPolicy Bypass -File tools/5/feishu_bot_ctl.ps1
```

## 使用前要补齐

- 飞书应用密钥
- 机器人账号配置
- Codex / OpenAI 凭据
- 本机工作目录
- 额外工作区
- 账号级 `system_prompt` 和 `role_memory`

## 说明

这个仓库提供的是可复用的工作流骨架，不包含你的实际运行环境。`config/secrets/local.yaml`、运行日志、`.runtime/` 和其他本机状态文件应保留在你自己的环境中。
