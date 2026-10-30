# AstrBot 插件技术锚点（写设计与文档前对照）

> 触发条件：涉及插件结构、装饰器、配置 schema、数据目录、发布要求，或对某个 API 的写法不确定时读本文件。
> 权威优先级：本机 AstrBot 源码 / 官方文档 > 官方模板 > 社区插件。三者冲突时以**运行中的实际版本源码**为准，并把差异写进开发文档。

## 1. 官方文档索引

| 主题 | 链接 |
|---|---|
| 插件开发指南（环境、metadata、依赖、开发原则） | <https://docs.astrbot.app/dev/star/plugin-new.html> |
| 最小实例（`main.py` 骨架） | <https://docs.astrbot.app/dev/star/guides/simple.html> |
| 处理消息事件（指令、指令组、过滤器、钩子） | <https://docs.astrbot.app/dev/star/guides/listen-message-event.html> |
| 发送消息 | <https://docs.astrbot.app/dev/star/guides/send-message.html> |
| 插件配置（`_conf_schema.json`、`AstrBotConfig`） | <https://docs.astrbot.app/dev/star/guides/plugin-config.html> |
| 调用 AI | <https://docs.astrbot.app/dev/star/guides/ai.html> |
| 存储 | <https://docs.astrbot.app/dev/star/guides/storage.html> |
| 会话控制器 / 文转图 / 国际化 / 可视化页面 | `/dev/star/guides/session-control`、`html-to-pic`、`plugin-i18n`、`plugin-pages` |
| 发布到插件市场 | <https://docs.astrbot.app/dev/star/plugin-publish.html> |
| 官方插件模板仓库 | <https://github.com/Soulter/helloworld> |

## 2. 目录与固定文件

```text
AstrBot/data/plugins/astrbot_plugin_<name>/
├── main.py            # 必需。插件类所在文件必须是 main.py
├── metadata.yaml      # 必需。AstrBot 靠它识别插件元数据
├── _conf_schema.json  # 有配置时必需
├── requirements.txt   # 有第三方依赖时必需
├── logo.png           # 可选，1:1，建议 256x256
├── README.md          # 使用/配置说明、灵感来源、许可声明
└── skills/            # 可选，随插件提供 Skill（子目录各带 SKILL.md）
```

插件命名：推荐 `astrbot_plugin_` 前缀、全小写、无空格、尽量短。

## 3. 入口骨架

```python
from astrbot.api.event import filter, AstrMessageEvent
from astrbot.api.star import Context, Star
from astrbot.api import AstrBotConfig, logger


class MyPlugin(Star):
    def __init__(self, context: Context, config: AstrBotConfig):
        super().__init__(context)
        self.config = config          # AstrBotConfig 继承 dict，可 self.config.save_config()

    @filter.command("helloworld")
    async def helloworld(self, event: AstrMessageEvent):
        """handler 文档字符串会被解析成插件指令说明，务必写。"""
        yield event.plain_result(f"Hello, {event.get_sender_name()}!")

    async def terminate(self):
        """插件被停用/卸载时调用，用来释放资源。"""
```

要点：handler 前两个参数固定 `self, event`；handler 必须写在插件类内；需要在 `main.py` 外拆服务时，在 handler 里调用外部模块。

## 4. 装饰器速查（全部来自 `astrbot.api.event.filter`）

| 用途 | 写法 | 备注 |
|---|---|---|
| 指令 | `@filter.command("name")` | 指令名不能含空格 |
| 指令别名 | `@filter.command("help", alias={"帮助"})` | v3.4.28+ |
| 带参指令 | `@filter.command("add")` + `(self, event, a: int, b: int)` | AstrBot 自动解析参数 |
| 指令组 | `@filter.command_group("math")` / `@math.group("calc")` / `@calc.command("add")` | 组函数体 `pass`；无子指令时报错并渲染树 |
| 事件类型 | `@filter.event_message_type(filter.EventMessageType.GROUP_MESSAGE)` | 只有 `PRIVATE_MESSAGE` / `GROUP_MESSAGE` / `ALL` |
| 平台过滤 | `@filter.platform_adapter_type(filter.PlatformAdapterType.AIOCQHTTP \| ...)` | 多平台用位运算 |
| 权限 | `@filter.permission_type(filter.PermissionType.ADMIN)` | 管理员指令 |
| 钩子 | `@filter.on_astrbot_loaded()` / `@filter.on_llm_request()` / `@filter.on_waiting_llm_request()` | 钩子不能与上面的过滤器混用；钩子里不能用 `yield`，改消息用 `await event.send(...)` |

多个装饰器可叠加，逻辑为 **AND**。

## 5. 事件对象关键字段

`AstrMessageEvent`：`get_sender_name()`、`message_str`（纯文本）、`message_obj`（`AstrBotMessage`）。
`AstrBotMessage`：`type`、`self_id`、`session_id`、`message_id`、`group_id`（私聊为空）、`sender: MessageMember`、`message: List[BaseMessageComponent]`（消息链）、`raw_message`、`timestamp`。
消息段类型：`Plain`、`At`、`Image`、`Record`、`Video`、`File`；OneBot v11 另有 `Face`、`Node`、`Nodes`、`Poke`。

## 6. 配置 schema 要点（`_conf_schema.json`）

- `type` 必填：`string` / `text` / `int` / `float` / `bool` / `object` / `list` / `dict` / `template_list` / `file`（v4.13+）。
- 可选键：`description`、`hint`、`obvious_hint`、`default`、`items`（`object` 用）、`invisible`、`secret`、`options`、`labels`、`subtitles`、`slider{min,max,step}`、`editor_mode`、`editor_language`、`editor_theme`、`_special`。
- `secret: true` 只遮罩 WebUI 显示，**不加密**文件里的值，所以代码里不许打印或回显。
- `_special` 常用值：`select_provider`、`select_provider_tts`、`select_provider_stt`、`select_persona`、`select_knowledgebase`（后者返回 list）。其余 `select_providers` / `provider_pool` 等属内部实现，禁止使用。
- 运行时配置落 `data/config/<plugin_name>_config.json`，通过 `__init__` 注入；schema 变更由 AstrBot 递归补齐/清理。

## 7. LLM 提示词相关坑（`on_llm_request`）

- `req.system_prompt += ...` 只适合**稳定**的角色设定；每轮都变的动态内容（时间、状态、记忆摘要）会破坏服务端提示词缓存、显著抬高成本与首 token 延迟。
- 动态内容走 `req.extra_user_content_parts.append(TextPart(...))`（`from astrbot.core.agent.message import TextPart`）。

## 8. 开发原则（官方，写进文档与审查项）

- 功能要经过测试；关键逻辑加注释；错误要有处理，不许因单条消息崩溃。
- 持久化数据写 `data` 目录，**不写插件目录**（更新/重装会覆盖）。
- 网络请求用 `aiohttp` / `httpx` 等异步库，**禁用 `requests`**。
- 提交前 `ruff` 格式化。
- 想给已有插件加功能，优先给原仓库提 PR，而不是另写一个（除非原仓库已停止维护）。
- 借鉴设计与思路要在 README 写明来源链接；使用/移植代码或资源须遵守原许可并保留版权声明。

## 9. 发布要求

- `metadata.yaml` 字段：`name`（英文唯一标识）、`display_name`、`short_desc`（可选）、`desc`（支持 Markdown 多行）、`version`（语义化）、`author`、`repo`、可选 `astrbot_version`（PEP 440 区间，如 `">=4.17.0"`）、`support_platforms`（list）、`tags`（list）、`social_link`。
- `support_platforms` 取值须是 `ADAPTER_NAME_2_TYPE` 的 key：`aiocqhttp`、`qq_official`、`qq_official_webhook`、`telegram`、`wecom`、`wecom_ai_bot`、`lark`、`dingtalk`、`discord`、`slack`、`kook`、`vocechat`、`weixin_official_account`、`weixin_oc`、`satori`、`misskey`、`line`、`matrix`、`mattermost`。
- 插件市场 zip **≤ 16MB**；发布前清掉 `.git`、`__pycache__`、`node_modules` 与开发配置，压缩图片资源，配 `.gitignore`。

## 10. 调试与验证

- AstrBot 采用运行时注入，调试必须起 AstrBot 本体。
- 改完代码去 WebUI `插件` 页点插件的「重载插件」；加载失败的在「加载失败插件」里重载看报错。
- 验证要给出：启动命令、要发的指令、预期回复、该看的日志关键字。

## 11. 不确定时的处理

1. 找到本机 AstrBot 安装目录，读 `astrbot/api/` 下对应模块的真实签名。
2. 读仓库 `docs/zh/dev/` 对应页面，确认版本是否匹配目标 `astrbot_version`。
3. 两边都查不到 → 在开发文档里标为"待验证假设"，并给出一个最小验证脚本/指令；不要凭记忆写进代码。
