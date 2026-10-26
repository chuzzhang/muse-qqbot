# QQ Bot 交接文档（供新 AI 接手使用）

> 版本：v2，2026-10-27。本文件取代 2026-10-25 的 v1 版。
> 本文档只写部署、运行、回复消息的方法，不含任何凭据值、真实 ID、人设内容。
> 文中所有 `<...>` 均为占位符，接手时从项目 `.env` 或队列文件里读真实值，不要把真实值写回本文档。

## 0. 当前状态快照（2026-10-27 核实）

- systemd 服务 `qqbot-ws`：`active` 且 `enabled`，登录与心跳正常。
- 队列：`queue/pending/` 与 `queue/processing/` 均为空，无积压、无超时孤儿。
- 三个项目外通道均在运行（见第 12 节）：主通道钩子 `qqbot-muse-reply` 已启用、轮询 5 秒；看门狗钩子 `qqbot-watchdog` 已启用、轮询 60 秒；兜底定时任务 `qqbot-reply-cron` 已启用、每 1 分钟。
- 已知未解决事项只有一件：群内不 `@` 的全量消息能否投递，取决于 QQ 侧群设置（见第 8 节），代码侧已就绪，截至本文档日期未核实该设置已被修改。
- 已知缺口：项目现存备份均为 2026-10-24 的旧包，早于其后大量改动。接手后如需备份，按第 13 节规则重新打包，并继续排除 `.env`。

与 v1 的主要变化：主通道钩子轮询间隔确认为 5 秒（项目 `README.md` 里写的 20 秒是旧值，以本文档和钩子实际配置为准）；新增「无错误不汇报」的汇报纪律（见第 12.4 节）；备份状态已注明过期。

## 1. 系统概览

这是一个 QQ 开放平台官方机器人项目，只用 WebSocket 网关收消息，不用 Webhook，不需要域名和 HTTPS 证书。

```text
QQ 用户消息
  -> QQ 网关 (WebSocket)
  -> bot.py            只负责收消息、下载附件图片、写队列 JSON，不负责组织回复内容
  -> queue/pending/    待处理队列（每条消息一个 .json 文件）
  -> 回复 Worker       外部 AI / 程序：认领队列文件、生成回复内容、调用发送脚本
  -> send_reply.py     发送文字被动回复
     send_image.py     发送图片被动回复
  -> QQ API (https://api.sgroup.qq.com)  携带原消息 msg_id 发出，被动回复
  -> queue/done/ 或 queue/failed/       发送脚本自动归档
```

关键设计点：

- 收消息和回消息是两个独立环节。`bot.py` 挂了会影响收消息；Worker 挂了只影响回复，消息会积压在队列里。
- 所有正式回复都是**被动回复**：请求里必须带原消息的 `msg_id`，且受被动窗口限制（见第 7.3 节）。
- 回复内容由 Worker 决定，本项目代码不定义回复文案风格。接手 AI 自行决定回复内容，只需按本文档的队列和发送规范操作。

## 2. 项目位置与文件说明

项目目录（下文称 `BASE_DIR`）：`~/workspace/qqbot-ws/`

| 路径 | 作用 |
|---|---|
| `bot.py` | 网关接收进程：登录、收消息、入队、`/ping` 直答、脚本级即时回复、botpy 补丁 |
| `qqbot_common.py` | 公共库：`.env` 读取、代理、队列读写、Token 获取、`download_media()` |
| `common.py` | 兼容垫片，转导出 `qqbot_common`，新代码直接用 `qqbot_common` |
| `send_reply.py` | 发送一条文字被动回复，并归档队列文件 |
| `send_image.py` | 发送一张图片被动回复，并归档队列文件 |
| `send_proactive.py` | 发送主动消息（不带 `msg_id`），仅作兜底，见第 9 节限制 |
| `quota_guard.py` | 每周用量限额保护，见第 10 节 |
| `ensure_service.py` | 把模板渲染成 systemd 单元并安装 |
| `templates/qqbot-ws.service.tmpl` | systemd 单元模板，含 `{BASE_DIR}` 和 `{PROXY_ENV_LINES}` 占位 |
| `requirements.txt` | 固定依赖：`qq-botpy==1.2.1`、`aiohttp>=3.9,<4`、`python-dotenv>=1.0,<2`、`PyYAML>=6.0` |
| `.env.example` | 环境变量模板，复制为 `.env` 后填真实值 |
| `.env` | 唯一存放凭据的文件，权限必须 `600`，不进备份、不进版本管理 |
| `queue/` | 队列目录，见第 5 节 |
| `generated/` | 业务侧生成的图片等产物存放处（发送脚本不关心其来源） |
| `backups/` | 项目备份包存放处，备份不含 `.env`（现存包为 2026-10-24 旧包，见第 0 节） |
| `README.md` / `RESTORE.md` | 英文版说明和恢复步骤，可与本文档互相参照；`README.md` 中主通道轮询写的是旧值 20 秒，以第 12 节为准 |

队列子目录：

| 目录 | 含义 |
|---|---|
| `queue/pending/` | 新消息待认领 |
| `queue/processing/` | 已被某个 Worker 认领、正在处理（此目录由 Worker 约定使用，`bot.py` 本身不写这里） |
| `queue/done/` | 已成功回复、归档 |
| `queue/failed/` | 发送失败、归档 |
| `queue/media/` | 收到消息里的图片附件下载到本地后的存放处 |

## 3. 环境变量（`.env`）

从模板复制：

```bash
cd ~/workspace/qqbot-ws
cp .env.example .env
chmod 600 .env
```

| 变量名 | 必填 | 说明 |
|---|---|---|
| `QQ_BOT_APP_ID` | 是 | QQ 开放平台分配的机器人 AppID |
| `QQ_BOT_APP_SECRET` | 是 | QQ 开放平台分配的机器人 AppSecret |
| `BOT_MODE` | 是 | 当前队列模式取值 `muse`；不是这个值时 `bot.py` 收到消息也不入队 |
| `QUOTA_USAGE_URL` | 否 | 限额用量查询地址，留空表示未接入外部用量源 |
| `QUOTA_USAGE_PERCENT` | 否 | 手工指定用量百分比（测试/临时覆盖用），留空表示未指定 |

另有代理变量（不是写在 `.env` 里，而是运行环境 / systemd 单元里的环境变量）：
`http_proxy`、`https_proxy`、`all_proxy`、`no_proxy` 及其大写形式。
本运行环境无直连出口，所有 REST 和 WebSocket 流量都必须走代理，见第 11 节补丁说明。

安全纪律：

- 凭据只出现在 `.env` 里。不要写入 systemd 单元、代码、本文档、备份、日志或聊天记录。
- 渲染后的 `qqbot-ws.service` 文件会含有带口令的代理地址，同样按敏感文件对待，不要复制、引用或外发其内容。
- `.gitignore` 已排除 `.env`、`.venv/`、队列数据、日志和渲染后的 `qqbot-ws.service`，不要移除这些规则。

## 4. 从零部署

前置条件：

1. 已在 QQ 开放平台创建机器人并拿到 AppID / AppSecret（通过带外方式交给接手者，不要经过本文档传递）。
2. 已把运行机器当前的出口 IP 加入 QQ 开放平台的 IP 白名单。出口 IP 变化后机器人会连不上，先查白名单。
3. 机器上有 Python 3、systemd，以及可用的出口代理环境变量（无直连出口的环境必须）。

步骤：

```bash
cd ~/workspace/qqbot-ws

# 1) 创建虚拟环境并安装固定依赖
python3 -m venv .venv
.venv/bin/pip install -r requirements.txt

# 2) 准备凭据
cp .env.example .env
# 编辑 .env 填入真实 QQ_BOT_APP_ID / QQ_BOT_APP_SECRET，确认 BOT_MODE=muse
chmod 600 .env

# 3) 无登录自检（只验补丁、配置是否存在、代理是否设置，不真正登录）
.venv/bin/python bot.py --dry-run

# 4) 渲染并安装 systemd 单元（需要 root；会把当前环境的代理变量写进单元）
.venv/bin/python ensure_service.py --install
# 说明：ensure_service.py 不带参数时只在项目目录生成 qqbot-ws.service 供检查；
# --install 才复制到 /etc/systemd/system/qqbot-ws.service 并执行 daemon-reload；
# --enable 还会 enable --now，但没有 .env 时脚本会拒绝 --enable。

# 5) 启动
systemctl enable --now qqbot-ws
systemctl status qqbot-ws
journalctl -u qqbot-ws -f
```

前台调试运行（不用 systemd 时）：

```bash
cd ~/workspace/qqbot-ws
.venv/bin/python bot.py
```

从备份恢复：按 `RESTORE.md` 执行——解包备份、（如包内有校验文件）`sha256sum -c SHA256SUMS`、重建 `.venv`、带外恢复 `.env`、跑 `--dry-run`、跑 `ensure_service.py --install`、启动服务、最后在 QQ 里发 `/ping` 验证。备份有意不含 `.venv`、日志、缓存和已处理的队列数据，队列目录会重建为空。

部署完成验证清单：

- [ ] `bot.py --dry-run` 输出正常，四项补丁均已应用
- [ ] `systemctl is-active qqbot-ws` 为 `active`，`systemctl is-enabled qqbot-ws` 为 `enabled`
- [ ] 日志里能看到登录成功和心跳启动，无反复断线报错
- [ ] 在 QQ 私聊机器人发 `/ping`，立即收到 `pong`
- [ ] 私聊发一条普通文字，`queue/pending/` 出现一个新 `.json`
- [ ] 用第 6 节方法回复该文件，QQ 里收到回复且文件被归档到 `queue/done/`

## 5. 队列文件格式与认领规范

每条入队消息是一个 JSON 文件，文件名形如 `<毫秒时间戳>-<短id>.json`。字段如下（值均为示意，不写真实值）：

```json
{
  "type": "c2c",
  "msg_id": "<原消息ID，回复时必须原样带回>",
  "event_id": "<事件ID，可为空>",
  "content": "<消息文本内容>",
  "author_id": "<发送者ID，按 type 解释>",
  "group_id": "<群ID，type=group 时有值，其余为 null>",
  "channel_id": "<子频道ID，type=channel 时有值>",
  "guild_id": "<频道/私信相关ID，type=dm 时有值>",
  "timestamp": "<消息时间，ISO 字符串>",
  "timestamp_epoch": 0.0,
  "attachments": [{"content_type": "...", "filename": "...", "size": 0, "url": "<附件地址>"}],
  "media_files": ["<附件图片下载到本地后的路径>"],
  "enqueued_at": 0.0,
  "id": "<队列内部ID>"
}
```

`type` 取值与含义：

| `type` | 场景 | 回复目标字段 |
|---|---|---|
| `c2c` | 私聊 | `author_id` |
| `group` | 群聊（`@` 消息与全量群消息都归为此类） | `group_id` |
| `channel` | 频道文字消息 | `channel_id` |
| `dm` | 频道私信 | `guild_id` |

Worker 处理规范（多 Worker 并发时必须遵守，否则会重复回复或抢文件）：

1. 同时看 `queue/pending/` 和 `queue/processing/` 两个目录，不要只看一个。
2. 认领动作是把文件从 `pending/` 原子移动到 `processing/` 再处理（`mv queue/pending/<file> queue/processing/<file>`）。先移动成功者拥有这条消息，其他 Worker 不要再碰。
3. `processing/` 里停留**不到 10 分钟**（按文件修改时间）的文件，视为别的 Worker 正在处理，不要碰、不要重复回复。
4. `processing/` 里停留**超过 10 分钟**的文件，视为处理者中断遗留的孤儿，需要回收处理：即使看起来已过被动窗口也要尝试发送一次，让发送脚本把它归档到 `done/` 或 `failed/`，不要让它永久留在 `processing/`。
5. 处理顺序：先孤儿、后新认领；同类按时间从旧到新（窗口最紧的先处理）。
6. 每处理完一条，确认该文件已离开 `processing/`。发送脚本正常会自动归档；若脚本没有归档，手工移动：成功移 `queue/done/`，失败移 `queue/failed/`。

## 6. 回复文字消息

工作目录固定为 `~/workspace/qqbot-ws`，队列文件路径用认领后的 `processing/` 路径（或尚未认领时用 `pending/` 路径，二者脚本都接受，关键是同一个文件只处理一次）。

```bash
cd ~/workspace/qqbot-ws

# 直接传回复文本
.venv/bin/python send_reply.py queue/processing/<file>.json "回复文本"

# 回复较长时从文本文件读，避免命令行转义问题
.venv/bin/python send_reply.py queue/processing/<file>.json --text-file reply.txt
```

行为说明：

- 脚本读取队列 JSON，按 `type` 拼目标地址：`c2c` 走 `/v2/users/<author_id>/messages`，`group` 走 `/v2/groups/<group_id>/messages`，`channel` 走 `/channels/<channel_id>/messages`，`dm` 走 `/dms/<guild_id>/messages`。请求体带 `content` 和原 `msg_id`；`c2c`/`group` 还带 `msg_type=0` 和一个随机 `msg_seq`。
- `msg_seq` 必须每次发送随机生成：同一 `msg_id` 的第二次被动回复如果不带不同的 `msg_seq`，会被 QQ 去重，返回错误码 `40054005`。脚本已内置此逻辑，不要删。
- 成功：打印 `{"sent": true, "response": {...}}`，队列文件自动移入 `queue/done/`，退出码 `0`。
- 失败：打印失败原因，队列文件自动移入 `queue/failed/`（带 `--no-archive` 时除外），退出码 `1`。参数缺失（没给回复文本）退出码 `2`；限额已满拒绝发送退出码 `3`。
- 已过被动窗口时脚本只打印警告、仍尝试发送，不要因为看到警告就放弃。

常用选项：

| 选项 | 用途 |
|---|---|
| `--no-archive` | 发送后不归档队列文件。典型场景：同一条消息要发两条回复（如先发一条、再发图片），第一条用此选项保留文件给第二条发送用，第二条不加此选项完成归档 |
| `--ignore-quota` | 忽略限额检查强制发送。仅用于发送限额暂停通知本身等特殊场景，普通回复不要加 |
| `--text-file <path>` | 从文件读取回复文本，与位置参数二选一 |

## 7. 回复图片、接收图片

### 7.1 发送图片（被动回复）

```bash
cd ~/workspace/qqbot-ws

# 本地图片 + 配文（配文可省略）
.venv/bin/python send_image.py queue/processing/<file>.json <本地图片路径> "配文"

# 图片来源也可以是 http(s) 地址，脚本会自行下载后再发送
.venv/bin/python send_image.py queue/processing/<file>.json https://<图片地址> "配文"

# 若这张图不是该消息的最后一次发送，再加 --no-archive 保留队列文件
.venv/bin/python send_image.py queue/processing/<file>.json <本地图片路径> "配文" --no-archive
```

规则：

- 只发一张图。大小上限 **10 MB**，格式只认 PNG / JPEG / GIF / WEBP（按文件头魔数校验，不是看扩展名），不合规直接报错失败。
- `c2c` / `group`：先调 v2 文件接口上传——`POST /v2/users/<id>/files` 或 `/v2/groups/<id>/files`，body 为 `{"file_type": 1, "file_data": "<图片的 base64>", "srv_send_msg": false}`，拿到 `file_info`；再发 `msg_type=7` 的消息，body 带 `msg_id`、随机 `msg_seq`、`media: {"file_info": "<file_info>"}`，配文放在 `content`。`file_info` 有效期约 86400 秒。
- `channel` / `dm`：不走上面两步，直接用 multipart form-data 把图片文件（字段名 `file_image`）连同 `msg_id`、可选 `content` 一起 POST 到对应的频道/私信消息接口。
- 归档与退出码规则与 `send_reply.py` 相同：成功归档到 `done/`，失败归档到 `failed/`（`--no-archive` 时不归档）。
- 图片来源是 URL 时，下载走与收图相同的直连失败转代理抓取逻辑（见 7.2），下载上限同样受发送上限约束，最终仍按 10 MB 校验。
- 图片本身如何生成不在本项目范围内：Worker 用自己可用的绘图能力准备好本地图片文件后，再调用本脚本发送即可。

典型发图流程（Worker 侧）：

```text
1. 认领队列文件到 processing/
2. 按业务需求生成 / 准备好图片文件，确认格式和大小合规
3. 执行 send_image.py（配文即该条回复的文字部分）
4. 检查退出码和输出的 sent 结果，确认队列文件已离开 processing/
```

### 7.2 接收消息中的图片

这部分由 `bot.py` + `qqbot_common.download_media()` 自动完成，Worker 不需要写下载代码：

- 入队时遍历消息附件，只处理图片类附件。下载上限 **15 MB**，超时和失败不阻断入队，失败信息记在该附件的 `download_error` 字段。
- 下载策略：先直连 QQ CDN 地址（10 秒超时），失败再经 `https://images.weserv.nl` 转抓（会对图片重编码，画面一致但字节不同）。
- 保存前校验 PNG / JPEG / GIF / WEBP 文件头，通过后存到 `queue/media/`，本地路径追加进队列 JSON 的 `media_files` 列表。
- Worker 读队列文件时，直接用 `media_files` 里的本地路径查看或处理用户发来的图片，不要重新去下载 `attachments[].url`。

### 7.3 被动回复窗口（重要）

| 场景 | 大约窗口 | 说明 |
|---|---|---|
| `c2c` 私聊 | 约 60 分钟 | 时间相对宽裕 |
| `group` 群聊 | 约 5 分钟 | 最紧，群消息要优先处理 |
| `channel` / `dm` | 约 5 分钟 | 同上 |

- 窗口从原消息时间算起。超窗后发送会失败，典型错误码 `40034031`（`msgid` 已过期，不能回复），这条消息无法再用被动回复补发。
- 代码里窗口只是警告判断（`passive_window_expired()`），不是硬拒绝：没超窗判断错了也要发，判断已超窗也尝试发一次，以 QQ 实际返回为准。

## 8. bot.py 自身的直答与入队规则

- `/ping`：文本恰好等于 `/ping` 时由 `bot.py` 直接回复 `pong`，不入队。用于存活检查。
- `bot.py` 内还有少量脚本级即时回复逻辑（在 `bot.py` 的 `_fast_answer` / `_GREETINGS` / `_IDENTITY_KEYS` / `_IMAGE_KEYS` 相关代码里），以及群/频道普通文字消息入队前的一条即时确认逻辑。这部分是代码写死的行为，接手 AI 如需增删关键词或改行为，直接改 `bot.py` 对应代码并重启服务；本文档不展开其具体文案。
- 限额已满时不入队，只回复一条固定的暂停通知（见第 10 节）。
- 其余消息一律入队，等 Worker 处理。

事件与 Intents：

- 代码启用的 Intents 只有 `public_messages + public_guild_messages + direct_message`。**不要加 `guild_messages`**：那是私域机器人专属，公域机器人请求它会被网关以错误码 `4014`（`disallowed intents`）断开。
- 已处理的消息事件：私聊 `C2C_MESSAGE_CREATE`、群 `@` 消息 `GROUP_AT_MESSAGE_CREATE`、群全量消息 `GROUP_MESSAGE_CREATE`、频道 `@`/普通消息、频道私信。群全量消息依赖第 11 节的解析器补丁。
- 群里不 `@` 的消息机器人能否收到，还取决于 QQ 侧群设置：需要在手机 QQ 该群的机器人设置里把「机器人可获取的群聊消息范围」设为「获取群内全部消息」。代码侧就绪不等于 QQ 侧会投递，没收到全量消息时先查这个设置，不要先改代码。

## 9. 主动消息（兜底，了解即可）

```bash
cd ~/workspace/qqbot-ws
# 主动发图片（不带 msg_id）
.venv/bin/python send_proactive.py <队列文件.json> <图片路径> "配文"
# 主动发纯文字
.venv/bin/python send_proactive.py <队列文件.json> --text-only "文字内容"
```

- 仅支持 `group` / `c2c`，原理是直接向 `/v2/groups/<group_id>/messages` 或 `/v2/users/<author_id>/messages` 发 `msg_type=0`（文字）或走文件接口后发 `msg_type=7`（图片），不带 `msg_id`。
- **限制：机器人默认没有群主动消息权限**，实测会返回错误码 `40034105`（主动消息失败，无权限）。所以主动消息不能当作可靠的补发手段，过期消息不要指望它救回来。正确做法是让用户在群里重新发一条新消息，在新消息的被动窗口内用被动回复发送。

## 10. 限额保护 quota_guard.py

- 用途：每周用量达到 100% 时，`bot.py` 停止入队、`send_reply.py` 停止发送，只允许发一条固定的暂停通知，额度重置后自动恢复。
- 状态缓存文件 `.quota_cache.json`，最多每 5 分钟刷新一次。直接运行 `.venv/bin/python quota_guard.py` 可查看当前状态和是否超限。
- 数据来源（`fetch_usage_percent()`，按优先级）：环境变量 `QUOTA_USAGE_PERCENT` 的静态值 → 请求 `QUOTA_USAGE_URL` 返回的 JSON（支持 `{"percent": n}`、`{"usage_percent": n}` 或 `{"used": n, "limit": n}`）→ 两者都没有则返回未知，视为**未超限**（默认状态，机器人正常工作）。
- 接入真实用量源时，只改 `quota_guard.py` 的 `fetch_usage_percent()` 和对应环境变量，不要改调用方的判断逻辑。

## 11. botpy 1.2.1 适配补丁（升级前必读）

`bot.py` 的 `apply_botpy_patches()` 对固定版本 `qq-botpy==1.2.1` 打了四个补丁，删掉任何一个都会在当前无直连出口的环境里出问题：

| # | 补丁 | 不打的后果 |
|---|---|---|
| 1 | 给 `ConnectionState` 补 `parse_group_message_create` 解析器（1.2.1 原生只有 `parse_group_at_message_create`） | 网关报 unknown event 并丢弃群全量消息，只有 `@` 消息能收到 |
| 2 | 替换 `BotWebSocket.ws_connect`，显式传 `proxy=get_proxy()` 并用 `trust_env=True` 的会话 | WebSocket 连不上代理，网关连接失败 |
| 3 | 替换 `BotHttp.check_session`，创建 `aiohttp` 会话时带 `trust_env=True` | botpy 自身的 REST 调用不走代理而失败 |
| 4 | 替换 `Token.update_access_token`，取 Token 的请求用 `trust_env=True` 的会话（原实现是裸会话） | 首次带凭据启动时报 SSL `WRONG_VERSION_NUMBER`，拿不到 Token |

本项目自有代码的 REST 请求统一用 `qqbot_common.make_aiohttp_session()`（默认 `trust_env=True`），Token 接口为 `https://bots.qq.com/app/getAppAccessToken`，业务 API 基础地址为 `https://api.sgroup.qq.com`，请求头为 `Authorization: QQBot <access_token>` 加 `X-Union-Appid: <AppID>`，Token 在内存里缓存到临近过期。

**升级 botpy 时必须逐项复核这四个补丁是否仍需要、内部函数签名是否变化，不要直接升级依赖就上线。**

## 12. 回复 Worker 的通道（项目外组件）

发送脚本只是工具，真正在跑的回复链路由项目外的调度组件驱动。接手时要单独确认这些组件是否存在、是否启用——它们不在项目备份里。

### 12.1 三个通道

| 通道 | 作用 | 当前配置（2026-10-27 核实） |
|---|---|---|
| 主通道：队列轮询钩子 `qqbot-muse-reply` | 高频检查 `queue/pending/`，有新消息就唤醒一个 Worker 回复 | 已启用，轮询间隔 **5 秒**；脚本 `~/hooks/scripts/qqbot-muse-reply.sh` |
| 兜底通道：定时任务 `qqbot-reply-cron` | 主通道失效时兜底扫队列 | 已启用，每 **1 分钟** 一次 |
| 看门狗钩子 `qqbot-watchdog` | 检查服务存活并自愈 | 已启用，轮询 **60 秒**；脚本 `~/hooks/scripts/qqbot-watchdog.sh` |

主通道脚本的防抢逻辑（接手时不要改弱）：

- `pending/` 没有 `.json` 时静默结束，不唤醒 Worker。
- `processing/` 有停留不到 10 分钟的文件时静默（说明已有 Worker 在处理），防重复唤醒抢同一条消息。
- 唤醒后有约 90 秒冷却，冷却内不再重复唤醒。

兜底通道规则与第 5 节相同：同时查 `pending/` 和 `processing/`，只回收超过 10 分钟的 processing 孤儿，不碰未超时的在途文件。

看门狗逻辑：单元文件 `/etc/systemd/system/qqbot-ws.service` 缺失、或服务不是 active 时唤醒处理，执行 `ensure_service.py --install` 重装单元并重新 `enable --now`。该故障在本环境反复出现（环境刷新会清掉 `/etc` 下的单元），自愈链路已多次验证有效。

### 12.2 Worker 处理最小流程

```text
扫描 pending/ + processing/
  -> 无新文件且无超时孤儿：结束，不做任何事
  -> 有文件：mv 认领到 processing/，读取 JSON
  -> 按 content 决定回复内容（文字或图片，内容规则不在本文档范围）
  -> 文字：send_reply.py <file> "回复文本"
     图片：准备好图片文件后 send_image.py <file> <图片路径> "配文"
  -> 确认文件已归档到 done/ 或 failed/，失败的记录原因供排查
```

预期时效参考：消息入队后主通道 5 秒内发现，文字正式回复通常在约 30–60 秒内发出；需要生成图片的回复瓶颈在生成本身，通常更久，群聊 5 分钟窗口内要抓紧。

### 12.3 汇报纪律（运营约定）

- **没有错误就不要汇报。** 例行检查正常、队列为空、无积压、自愈成功，都属于正常结果，保持静默，不要逐次向负责人发状态消息。
- 只在以下情况才汇报：恢复/发送真实失败、服务超过约 10 分钟仍未恢复、队列出现积压、或有需要负责人决定的事项。汇报时说明具体故障、已尝试的处理和当前状态。

## 13. 日常运维命令

```bash
systemctl status qqbot-ws          # 服务状态
systemctl restart qqbot-ws         # 重启（改 bot.py / .env / 模板后需要）
systemctl enable --now qqbot-ws    # 启用并启动
journalctl -u qqbot-ws -f          # 实时看 systemd 日志
journalctl -u qqbot-ws --since "10 minutes ago"   # 看最近日志

ls ~/workspace/qqbot-ws/queue/pending/      # 看积压
ls ~/workspace/qqbot-ws/queue/processing/   # 看在途/是否有孤儿
cd ~/workspace/qqbot-ws && .venv/bin/python quota_guard.py   # 看限额状态
```

- 服务单元被环境刷新清掉是已知的反复出现的问题：现象是 `/etc/systemd/system/qqbot-ws.service` 不存在、服务查不到。处理就是重跑 `.venv/bin/python ensure_service.py --install` 再 `systemctl enable --now qqbot-ws`，看门狗钩子正常时会自动完成这一步。
- 服务配置了 `Restart=always`、`RestartSec=5`，进程异常退出会自动拉起；网关连接以关闭码 `1006` 断开后自动重连也是常见现象，偶发一次且能自恢复不用处理，持续反复断线才需要查代理和白名单。
- 改了代理环境后必须重新渲染单元（重跑 `ensure_service.py --install`），只改 shell 环境对已运行的 systemd 服务无效。
- 重新备份时：打包项目代码、模板、文档和队列目录结构，排除 `.env`、`.venv/`、日志、缓存、渲染后的 `qqbot-ws.service` 和已处理的队列数据；打包后生成 `SHA256SUMS` 并校验通过再交付。

## 14. 常见故障对照表

| 现象 / 错误码 | 原因 | 处理 |
|---|---|---|
| `4014 disallowed intents` | Intents 里带了私域专属项 | 只保留 `public_messages + public_guild_messages + direct_message`，改后重启 |
| SSL `WRONG_VERSION_NUMBER`（取 Token 时） | Token 请求没走代理（补丁 4 失效） | 检查 `apply_botpy_patches()` 是否完整执行，勿删补丁 |
| 网关连不上 / 反复超时 | 出口 IP 不在白名单，或代理未生效 | 查 QQ 开放平台 IP 白名单与当前出口 IP；确认单元里已渲染代理变量 |
| `40054005` | 同一 `msg_id` 的多次被动回复被去重 | 确认发送脚本每次都带随机 `msg_seq`（内置逻辑，勿改成固定值） |
| `40034031` msgid 已过期 | 已过被动窗口，见第 7.3 节 | 无法被动补发；让用户重新发一条新消息，在新窗口内回复 |
| `40034105` 主动消息无权限 | 机器人没有群主动消息权限 | 预期行为，不要把主动消息当补发方案，见第 9 节 |
| 群全量消息收不到，只有 `@` 消息 | QQ 侧群消息范围设置未开，或补丁 1 失效 | 先查群机器人设置改为获取全部消息，再查补丁 |
| 队列有积压没人回复 | Worker 通道停了，不是 `bot.py` 的问题 | 分别查主通道钩子、兜底定时任务是否启用；临时可人工按第 5、6、7 节手动处理队列文件 |
| `processing/` 有文件长期不动 | Worker 中断遗留孤儿 | 超过 10 分钟按孤儿回收处理，见第 5 节第 4 条 |
| systemd 单元不存在 | 环境刷新清掉 `/etc` 下单元 | 重跑 `ensure_service.py --install` 并重新 enable，见第 13 节 |
| 回复时提示限额已满（退出码 `3`） | `quota_guard` 判定用量 100% | 查 `.quota_cache.json` 和用量源，属预期保护，等重置或核实用量源配置 |

## 15. 交接注意事项清单

- [ ] 凭据通过带外方式交接，`.env` 权限保持 `600`，交接文档和备份里都没有凭据值
- [ ] 确认 QQ 开放平台 IP 白名单包含当前机器出口 IP
- [ ] 确认三个项目外组件（主通道钩子 5 秒、兜底定时任务 1 分钟、看门狗钩子 60 秒）的存在和启用状态，项目备份不包含它们
- [ ] 不要删除 `bot.py` 的四个 botpy 补丁；升级 `qq-botpy` 前逐项复核
- [ ] 不要改发送脚本的随机 `msg_seq` 逻辑；同一条消息发第二条回复时记得处理好归档（第一条带 `--no-archive`）
- [ ] 群消息优先处理：5 分钟窗口比私聊的 60 分钟紧得多
- [ ] 遵守第 12.3 节汇报纪律：无错误不汇报，只报真实故障、积压和需决定的事项
- [ ] 现存备份是 2026-10-24 旧包；如需可靠备份，按第 13 节规则重新打包（继续排除 `.env`），并在恢复后重新验证 `/ping` 和一条真实回复
