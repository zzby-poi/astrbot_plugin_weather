# 天气主动推送

基于 [UAPI 天气接口](https://uapis.cn/docs/api-reference/get-misc-weather) 的 AstrBot 天气插件，支持自然语言查询、按会话记录所在地、每日定时推送、天气波动监测提醒，以及由 LLM 按机器人当前人格生成自然回复。

## 功能特性

- 实时天气查询：用户可以直接询问“今天的天气怎么样”“明天会下雨吗”等。
- 会话级所在地记录：每个会话单独保存所在地，避免不同 webchat 会话之间串城市。
- 自然语言引导：当前会话未设置城市时，会引导用户提供城市或区县。
- 定时主动推送：可配置每天多个时间点推送天气。
- 天气波动提醒：可定时查询天气，发现天气、温度、湿度等明显变化时主动提醒。
- 白名单控制：只有配置了会话 UMO 或用户 ID 的目标才会收到主动推送，也可限制即时查询权限。
- 扩展天气信息：支持扩展气象字段、多天预报、逐小时预报、分钟级降水、生活指数。
- 人格化回复：天气查询、推送、错误提醒、城市设置引导和确认都可通过提示词配置，并保留 AstrBot 当前人格。
- 上下文注入：可选择注入 AstrBot 对话历史、平台消息流水或混合上下文。
- 分段回复：可配置短回复分段发送，兼容 webchat 页面显示逻辑。

## 接口来源

本插件使用 UAPI 天气接口：

- 文档地址：https://uapis.cn/docs/api-reference/get-misc-weather
- 默认接口地址：https://uapis.cn/api/v1/misc/weather

插件支持免费接口模式。若你拥有 UAPI 的 ApiKey，可以在后台配置 `api_settings.api_key`，插件会通过 `Authorization: Bearer <ApiKey>` 请求接口。

## 安装

将本仓库放入 AstrBot 插件目录：

```text
AstrBot/data/plugins/astrbot_plugin_weather
```

安装依赖：

```bash
pip install -r requirements.txt
```

依赖包括：

- `aiohttp`
- `apscheduler`

安装后在 AstrBot 后台启用插件，或重启 AstrBot。

## 基础配置

进入 AstrBot 后台插件配置，找到“天气主动推送”。

### 1. 接口设置

`api_settings`

- `api_base_url`：天气接口地址，默认使用 UAPI。
- `api_key`：ApiKey，留空时使用免费接口。
- `lang`：返回语言，默认 `zh`。
- `timeout_seconds`：接口请求超时时间。

### 2. 查询权限与白名单

主动推送只对白名单生效。

`push_settings.private_session_whitelist`

填写完整会话 UMO。可通过 AstrBot 的 `/sid` 查看，例如：

```text
webchat:FriendMessage:webchat!zzby!xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

`push_settings.user_id_whitelist`

填写用户 ID，例如：

```text
zzby
```

建议：

- webchat 临时会话推荐填写完整 UMO，适合只允许某个会话。
- 如果希望同一用户的多个会话都允许查询或推送，可以填写用户 ID。
- 所在地数据仍按原始会话 UMO 独立保存，不会因为用户 ID 相同而串城市。

`query_settings.allow_non_whitelist_query`

- `true`：非白名单用户也可以即时查询天气。
- `false`：只有白名单会话或用户可以即时查询。

### 3. 记录所在地

用户可以通过自然语言设置所在地：

```text
我的城市是武汉武昌区
```

如果当前会话没有记录城市，插件会引导用户输入所在地：

```text
在武汉武昌区
```

也可以使用命令：

```text
天气 设置城市 武汉武昌区
天气 我的城市
```

后台 `query_settings.location_setting_keywords` 可以配置自然语言设置城市触发词，例如“设置城市”“我的城市是”。后台 `query_settings.weather_query_keywords` 用于配置查询天气触发词，例如“天气”“下雨”“温度”。

默认开启 `query_settings.enable_ai_intent_detection`。命中触发词后，插件会先调用 `llm_generate()` 做轻量意图分类，只有模型判断为 `query_weather` 或 `set_location` 时才接管消息；如果判断为 `none`，会放行给 AstrBot 正常对话。
`query_settings.intent_provider_id` 支持在后台通过“选择提供商”按钮从已配置模型列表中选择；留空时使用当前会话模型。

插件会尽量记录最小区域：

- “湖北武汉”会记录为“武汉”
- “武汉武昌区”会记录为“武昌”

## 主动推送配置

`push_settings`

- `enable_daily_push`：是否启用每日定时推送。
- `daily_push_times`：每日推送时间列表，格式为 `HH:MM`，例如 `["07:30", "12:30", "21:00"]`。
- `enable_llm_message`：是否使用 AI 生成人格化天气文案。
- `daily_extensions`：每日推送是否开启扩展字段、多天预报、生活指数等。

注意：主动推送只会发送给白名单中的会话或用户。

## 天气波动监测

`monitor_settings`

- `enable_monitor`：是否启用天气波动监测。
- `monitor_interval_minutes`：监测查询间隔，默认 30 分钟。
- `alert_cooldown_minutes`：同一会话提醒冷却时间。
- `enable_quiet_hours`：是否启用免打扰时间段。
- `quiet_hours`：免打扰时间段列表，格式为 `HH:MM-HH:MM`，支持跨天，例如 `23:00-07:30`。
- `temperature_delta_threshold`：温度变化阈值。
- `humidity_delta_threshold`：湿度变化阈值。
- `monitor_extensions`：监测查询时使用的扩展信息。

当天气现象改变，或温度、湿度变化超过阈值时，插件会向白名单用户发送提醒。
免打扰时间段内不会执行天气波动监测，也不会更新上一次天气快照。
波动提醒文案使用 `prompt_settings.weather_alert_prompt`，可与每日主动推送提示词分开配置。

## 扩展天气字段

插件支持 UAPI 的可选天气能力：

- `extended`：扩展气象字段
- `forecast`：多天预报
- `hourly`：逐小时预报
- `minutely`：分钟级降水
- `indices`：18 项生活指数

在用户即时查询时，插件可以根据用户问题自动判断是否开启扩展字段：

- “最近天气怎么样”：倾向开启多天预报。
- “雨什么时候停”：倾向开启逐小时预报和分钟级降水。
- “今天适合穿什么”：倾向开启生活指数。

可在 `query_settings.enable_llm_extension_decision` 中关闭 AI 判断，关闭后会使用规则判断。

## 提示词配置

`prompt_settings`

可配置以下提示词：

- `weather_push_prompt`：主动天气推送文案。
- `weather_alert_prompt`：天气波动监测触发后的提醒文案。
- `weather_query_prompt`：用户查询天气时的回复。
- `error_prompt`：天气接口错误、服务异常等提醒。
- `location_guide_prompt`：当前会话未设置城市时，引导用户提供所在地。
- `location_saved_prompt`：用户提供所在地后，确认记录成功。
- `intent_detection_prompt`：命中触发词后的插件意图分类提示词。

这些提示词都会尽量保留 AstrBot 当前人格、称呼习惯、语气和口癖。

常用占位符：

- `{{current_time}}`
- `{{location}}`
- `{{query_city}}`
- `{{weather_json}}`
- `{{reason}}`
- `{{user_message}}`
- `{{push_enabled}}`
- `{{push_hint}}`

## 上下文注入

`context_settings`

- `enable_context`：是否启用上下文注入。
- `source_mode`：上下文来源。
  - `conversation_history`：使用 AstrBot 当前 LLM 对话历史。
  - `platform_message_history`：使用平台最近真实聊天流水。
  - `hybrid`：同时使用两者。
- `conversation_history_count`：注入对话历史条数。
- `platform_history_count`：注入平台流水条数。
- `platform_history_prompt`：平台流水注入提示词。
- `include_bot_messages`：是否包含 Bot 消息。
- `bot_identifiers`：Bot 标识，多个标识用英文逗号分隔。
- `platform_context_max_chars`：平台流水最大注入字数。

如果你希望回复更贴近日常对话，可以尝试开启 `hybrid`。

## 分段回复

`segmented_reply_settings`

- `enable`：是否启用分段回复。
- `allow_external_splitter`：插件自身分段关闭时，非 webchat 平台的普通天气查询可交给 `astrbot_plugin_splitter` 等外部分段插件处理；webchat 会自动整段输出，避免页面截断。
- `segment_push_messages`：是否对每日推送和天气波动提醒启用本插件分段。主动推送没有普通消息结果事件，无法交给 `astrbot_plugin_splitter` 处理。
- `push_words_count_threshold`：主动推送的“不分段字数阈值”，默认较高，避免天气波动提醒因为文本较长而整段发送；填 `0` 表示不限制。
- `words_count_threshold`：文本长度小于等于该值时才分段；更长文本整段发送。
- `split_mode`：分段方式，支持 `regex` 和 `words`。
- `regex`：正则分段规则。
- `split_words`：分段词列表。
- `enable_content_cleanup`：是否启用分段后清理。
- `content_cleanup_rule`：内容清理正则。
- `interval_method`：段间间隔方式，支持 `random` 和 `log`。
- `interval`：随机间隔秒数，例如 `1.0, 2.5`。
- `log_base`：对数间隔基数。

webchat 页面有特殊显示逻辑。普通查询在 webchat 中会绕过外部分段器，避免最后一段覆盖前面的内容；其他平台可按配置交给外部分段器。每日推送和天气波动提醒属于主动发送，由本插件按 `segment_push_messages` 和 `push_words_count_threshold` 控制分段。

## 常用命令

```text
天气 设置城市 武汉武昌区
天气 查询
天气 查询 明天会下雨吗
天气 我的城市
天气 测试推送
```

自然语言触发示例：

```text
今天的天气怎么样
明天会下雨吗
雨什么时候停
最近天气怎么样
今天适合穿什么
```

## 数据存储

用户所在地和最近天气快照存储在：

```text
AstrBot/data/plugin_data/astrbot_plugin_weather/weather_users.json
```

所在地按会话 UMO 独立保存。webchat 新开会话后，如果没有设置城市，插件会重新引导用户设置，避免不同会话之间串城市。

## 注意事项

- 如果后台修改了配置，请重载插件或重启 AstrBot。
- 主动推送依赖白名单，请先配置 `private_session_whitelist` 或 `user_id_whitelist`。
- 如果设置了 `allow_non_whitelist_query=false`，非白名单会话无法即时查询天气。
- ApiKey 留空时使用免费接口，具体额度和限制请以 UAPI 官方说明为准。
- 若天气接口返回城市未找到，请尝试使用更标准的城市或区县名，例如“武汉武昌区”。
