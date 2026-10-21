# astrbot_plugin_burst

**消息防抖聚合**：把用户连着发的几条消息攒成一条，再交给模型。

版本 `0.2.1` · AstrBot `>= 4.28.2` · 平台 `aiocqhttp`（OneBot v11）

---

## 它解决什么

用户连发三条（「今天好累啊」/「刚下课」/「陪我聊会」），模型就被触发三次：
每次看到的上下文都不全，回复也碎，token 还翻倍。

本插件包住 `Platform.commit_event` —— 消息进事件队列的唯一咽喉。没到静默期的消息
**根本不进流水线**，所以这一轮从头到尾只有一个事件，不存在「让位事件落不落历史」的问题。

## 触发规则

谁先满足谁赢：

| 规则 | 条件 | 默认 |
|---|---|---|
| 静默放行 | 距最后一条满 N 秒 | `idle_seconds = 20` |
| 长消息 | 单条去掉空白后 ≥ N 字符（标点、emoji 都算） | `instant_chars = 25` |
| 排队兜底 | 上一轮迟迟不收尾 | `hold_max_seconds = 300` |

**立刻放行、不进缓冲**：

- 空文本通知（"对方正在输入"这类）
- 指令（以 `/` 开头）—— 否则 `/stop`、`/new` 也要等 20 秒
- 结构性消息（语音 / 文件 / 引用 / QQ 表情 / 转发）：先把已挂起的那批出手，再原样放行

**进缓冲**：纯文字 + 图片。图片不单独直通，作为附件挂在聚合后的整条消息上。

## 和 pacer 一起用时：排队，而不是打断

装了 [pacer](https://github.com/Lansheen3096/astrbot_plugin_pacer) 时，新消息到达会先问它
「这一轮在飞吗、已经发出去几段」：

```
在飞 且 已发出 0 段   → 新消息排队，上一轮继续跑（它还没开口，不算被打断）
在飞 且 已发出 ≥1 段  → 正常打断语义（用户确实想掐掉后面那些）
不在飞                → 走普通聚合逻辑
```

上一轮收尾（发完 / 被打断 / 没东西可发）时，pacer 回调 `burst.release_pending()` 放行排队的那批。
只装 burst 时这部分不生效，基础聚合功能不受影响。

合并后的消息形态（每行一条消息 + 到达时刻，末尾带一条时间锚）：

```
[14:03:12] 今天好累啊
[14:03:15] 刚下课
<system_reminder>Current datetime: 2026-10-12 14:03 (CST), Weekday: Monday</system_reminder>
```

时间锚是给记忆插件用的：`astrbot_plugin_memory_digest` 靠它照抄条目日期，推算出来的日期会被丢掉。

## 配置

| 配置项 | 默认 | 说明 |
|---|---|---|
| `idle_seconds` | `20.0` | 静默放行秒数 |
| `instant_chars` | `25` | 长消息立即放行字数（去空白后） |
| `hold_max_seconds` | `300.0` | 排队兜底上限；只在上一轮没正常收尾时触发 |
| `timezone_label` | `CST` | 时间标记里的时区名 |

## 安装

```bash
# AstrBot 插件市场搜索「消息防抖聚合」，或：
cd <AstrBot>/data/plugins
git clone --depth 1 --branch v0.2.1 https://github.com/Lansheen3096/astrbot_plugin_burst
# docker 部署：拷进宿主机挂载的数据目录后重启容器
```

插件目录名要和 `metadata.yaml` 里的 `name` 一致。

## 日志

`[burst]` 前缀，方便 grep。典型输出：攒了几条、为什么放行（静默 / 长消息 / 指令直通）。

## 坑

- **`ComponentType` 枚举的值是大写**（`"Image"` / `"Plain"`），直接跟 `"image"` 比恒为 `False`。
  代码里统一先取 `.value` 再 `lower()`。
- **改了插件目录名要连 pacer 那边一起改**：两者靠插件名查注册表互相对接。

## License

MIT
