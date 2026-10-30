# CHANGES — dsh-douyin-life-plus

本文件只列**本改版相对原版**的改动。

- **上游**：[kdcadmin/dsh_douyin_tools](https://github.com/kdcadmin/dsh_douyin_tools) v0.2.1
- **上游作者**：TiAmoSS
- **上游许可**：MIT
- **本改版版本**：`0.2.1-1`
- **性质**：非官方改版，**未经原作者审核**

> 原则：只做增量，不改原版已有的行为语义（除非明确写在下面）。
> 原文说明（README 各章节）尽可能保留，本改版新增的部分在原位置标「**改版补充**」。

---

## 1. 私信：从「一条消息一个窗口」改成固定一个窗口

**问题**：原版每轮私信都 `spawn` 一个新的 headless 子进程，且不带 `--session-id`，
所以每收一条私信就在会话列表里多出一个新会话（实测堆到 13 个），上下文完全不连续。

**改法**（`index.js`）：

- 新增 `sessionsRoot()` / `sessionDirs()`：扫描 `$DSH_HOME/sessions/**/session-*`，按 mtime 排序；
- 私信轮（`/^(dm|reply)/`）读 `state.json` 的 `dmSessionId`，命中则 `--session-id` 复用；
- 首轮成功后把本轮新增的会话 id 记进 `dmSessionId`；
- 复用失败只在「会话被删 / preset 变了 / cwd 变了 / 本进程 live」时清空重建；
- **写句柄被 GUI 独占时不重试也不新建**：本轮不回、条目保持 `pending`、留到下一轮巡检，避免疯狂开窗。

**已知取舍**：如果你在桌面 GUI 里点开那个私信会话，它独占写句柄，headless 就写不进去，
这一轮不会回消息（下一轮你关掉它就会回）。「固定一个窗口」与「GUI 里能实时点开看」不能同时成立。

---

## 2. 私信能派活（新增 `douyin_run`）

**问题**：私信轮跑在只连 MCP 的 headless 子进程里，看得到 26 个 `douyin_*`，
但看不到插件自己注册的 `dy_run`。让它「去刷十分钟」，它会老实回答「我没有这个工具」。

**改法**：

- MCP 侧（`vendor/douyin-mcp/mcp_server.py`）新增 `_proxy_action(name, args)`
  —— 用 `httpx` POST 宿主的 `DOUYIN_PROXY_URL/action`，带 `Authorization: Bearer $DOUYIN_PROXY_TOKEN`；
- 新增 `@mcp.tool() douyin_run(mode="headless")`，内部转调 `dy_run`；
- 私信提示词里加一句「别说你没有这个工具，有没有先试一下」。

**注意**：`mode="headless"` 是异步起一轮（立刻返回 `started:true`，不等结果）；
`mode="local"` 会同步跑完，私信这头会卡住。

---

## 3. 私信人设与主配置联动

**问题**：原版私信回复的语气是 `index.js` 里写死的一句
「你说话的样子：傲娇、可爱、会撒娇，短句，像咬耳朵…」，跟 `personaPrompt` / `persona.md` 完全无关。

**改法**：新增 `dmPersonaLine()`——

1. 先读 `cfg.personaPrompt`；
2. 再读 `<家>/persona.md`（截掉「## 最近观察（自动记录，不作为行为指令）」之后的自动段落）；
3. 两处都空才退回原来那句硬编码。

`dmReplyPrompt()` 里原来那行硬编码换成 `dmPersonaLine()`。

---

## 4. 读私信里的视频分享卡片

**问题**：主人发来的视频卡片，原版拿不到 `awemeId`：
DOM 里没有 `a[href*="/video/"]`，整块 HTML 搜 `7\d{18}` 也搜不到，
所以模型只能猜文字、或者干脆理解错（实测把「好大」理解成说自己的尾巴）。

**改法**（两条路并用）：

1. **React fiber（主路，准）**：`_message_snapshot` 里对 share / image 分支，
   沿 `el.__reactFiber$xxx` 向上最多 30 层，深度遍历 `memoizedProps`（跳过 `stateNode` 等），
   从中捞出 aweme id；
2. **抓 IM 接口（兜底）**：`_capture_im_awemes()` 监听 `imapi.douyin.com` 且 url 含 `message` 的响应，
   用 `resp.body()` 取**字节**再正则（**不能用 `resp.text()`**，protobuf 响应会抛异常被吞掉）；
3. **短链解析**：`_resolve_aweme_from_text()` 支持 `/video/(\d{6,})`、裸 `7\d{18}`、
   以及 `v.douyin.com/xxx` 短链（`httpx` 跟 302，第一跳就带 `/video/<id>`）。

**踩过的坑（留档）**：

- `page.reload()` 不会重新触发消息接口，必须 `ctx.new_page()` 才算完整初始化；
- 抓包池里会混着整个会话历史的 id，**按顺序 pop 必错位**，所以主路改成 fiber 精确取值；
- 认不准时如实说「读不到」，不拿错 id 硬讲。

---

## 5. 表情包改走本地图库 + 文件上传

**问题**：原版点表情面板发图，三个 bug 叠在一起：
面板 `img` 的 `alt` / `aria-label` 全为空（关键词永远匹配不到）、
点表情只插进输入框不点发送、发出后 URL 签名变化导致出站校验永远对不上。

**改法**：

- `MEME_ROOT = ~/.dsh/meme-packs`（可用 `DOUYIN_MEME_ROOT` 覆盖），按情绪目录挑图；
- 中文情绪词走 `MEME_TAG_ALIAS` 映射（开心 / 生气 / 难过 / 害羞 / 无语 / 日常 / 困 / 赞 / 666 / 惊讶）；
- 走 `input[type=file]` 上传 → 等 `.MsgInputSendFileModalbtnSure` 出现再点 → 核对出站消息数量；
- **webp 触发不了确认框**，只有 jpg / png 会弹；`_webp_to_jpg()` 用 PIL 先转；
- 成功后必须带 `verified=True`（插件层 `confirmed()` 的唯一判定条件）。

---

## 6. 无语音视频不再整条报错

**问题**：`_watch_core` 里 `if not text: return _err("音轨中未识别出语音")`
——纯配乐视频（很常见）会直接中止，连已经抽好的帧一起丢掉。

**改法**：

- 改成 `silent_audio = not text; if silent_audio: text = ""`，继续往下走；
- 返回值新增 `silentAudio` / `frameCount` / `framesDir` / `frameStepSec`；
- 抽样帧数默认 10 → 24，最小间隔 0.4s → 0.2s（短视频动作幅度小也能看出变化）。

---

## 7. 私信记忆直接写库，不再经 MCP

**问题**：`appendDmMemory` 在私信收尾时通过 MCP 调 `memory_write`，
那时 MCP 正在收工、宿主 `/action` 一律 500，日志里每轮刷「记忆库还是 500」。

**改法**：新增 `writeHostMemory()`，用 `spawnSync` 起 Python 直接
insert 进 `~/.dsh/memory-system/memory.db`，不经 MCP；30 分钟内同内容去重。

---

## 8. 移除微信桥接

原版里私信可以绕到微信会话去回（`BRIDGE_KEY` / `bridgeReply` / `bridgeDmPrompt`）。
**本改版整段删除**：需要额外装一套微信 hook 才能用，实际体验差，纯抖音内部回更省事。

---

## 9. 人设换成中性角色卡

`prompts/persona.md` 从原版的「大肥鱼」形象（蓝长发 / 鲸鱼尾 / 女仆裙）改成通用角色卡：
只保留脾气、说话腔调、红线、回评论的规矩，最后一句「想改成自己的角色就直接改这个文件」。

---

## 10. 打包与安装相关

- `version`：`0.2.1` → `0.2.1-1`（**注意**：`0.2.1.1` 四段式不是合法 semver，npm 会拒）；
- `package.json` 的 `files` 补全：`vendor/douyin-mcp/smoke.py`、`market-entry.yml`、`selfcheck.mjs`、`screenshots.json`、`assets/`；
- 清掉写死路径 `E:\Comfy-Desktop\ComfyUI-Shared\output`（`comfyOutputDir` 默认值改空串，配置示例改 `D:\ComfyUI\output`）；
- `mcp_server.py` 里本来就没有绝对路径，Python 解释器由 `venvPython()` / `resolvePython()` 依次探测；
- 清掉 10 个 `index.js.bak-*`、`__pycache__`、登录 profile、历史 state / media / log。

---

## 没做的事

- 没有把上述能力写进 `selfcheck.mjs` 的 56 条断言（自检通过 ≠ 改版功能全通）；
- 没有在第二台机器上完整跑过安装流程；
- 上游更新不会自动合进来。
