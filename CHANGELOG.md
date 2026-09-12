# Changelog

## 0.2.2

- fix: `LLM_CONFIGS` 端点级 `enable_thinking` 声明此前被静默丢弃——0.2.1 引入了字段与客户端处理，但 `load_llm_models_from_env()` 构造 `LLMModelConfig` 时漏传该字段，Pydantic 取默认 `None`，于是「端点级显式 > `ANGINEER_CHAT_TEMPLATE_KWARGS` > 隐式 URL/模型名规则」里最高优先级整层失效（直连 vLLM/DGX 端点三层全不命中、发不出任何思考控制）。loader 现补传该字段，并新增宽松布尔解析：`true/false` 与 `"true"/"false"/"1"/"0"/"yes"/"no"/"on"/"off"` 都认，识别不了按默认值处理并打 WARNING（不把原值直接透传给 pydantic，避免 `.env` 写成字符串或拼错时在加载期抛异常把调用方进程打挂）
- fix: `enabled` 同类隐患——原实现 `bool(item.get("enabled", True))` 在 `.env` 写 `"enabled": null` 时得到 `bool(None) == False`，等于把「未声明」当成「显式禁用」（线上 `Qwen3.8-Flash` 即因此被静默禁用、且其 `enable_thinking: false` 声明同时失效）；现走宽松解析，`null`/缺失即启用
- fix: `ANGINEER_CHAT_TEMPLATE_KWARGS` 为空串/纯空白/非法 JSON 时不再抛异常——该行在每次请求的热路径上（`_build_extra_body` 的 4 处调用点），运维把该键清空（键在、值空）即全量请求抛 `JSONDecodeError`；又因该异常是 `ValueError` 子类，消费方容易把它误映射成「请求非法」，按 400 排查会跑偏。现空值按未设置处理、非法 JSON 降级为未设置并打 WARNING，变量未设置时的行为与旧版完全一致
- chore: 补 loader 级回归测试——此前没有任何测试走 `load_llm_models_from_env()`（全是手工构造 `LLMModelConfig`，正好绕过漏参那一行，这是缺陷能活到线上的直接原因）。新增断言覆盖声明 `false/true/省略`、字符串形态、无法识别值、`enabled: null`，以及「线上 `.env` 真实形态 → `extra_body`」的端到端用例；并修掉 `test_enable_thinking_switch.py` 一处假覆盖（原 `_extra_body_for` 把环境变量 patch 成空串后又 `pop`，实际测的是「未设置」，空串路径从未被覆盖）

## 0.2.1

- fix: 流式 reasoning 增量的 delta 事件恒带 text 键——旧实现发无 text 键的 `{type:'delta'}` 事件，聚合环与 `chat_stream`/`achat_stream` 消费方按 `event['text']` 取值即 KeyError('text') 判整次生成失败（会吐思考增量的模型直连端点必踩，评测 53 题全灭事故根因）；生产方补 `text:''`、同步/异步消费方 4 处改 `event.get('text','')` 双保险
- feat: LLM_CONFIGS 端点级 `enable_thinking` 显式开关——可选 true/false/null（null=缺省零变化），显式值按 vLLM 标准形态发 `chat_template_kwargs.enable_thinking`，优先级 端点显式 > `ANGINEER_CHAT_TEMPLATE_KWARGS` > 隐式 URL/模型名规则（直连 vLLM/DGX 思考模型不再依赖不会命中的隐式规则）；chat/achat 四处重复的 extra_body 构建收敛为 `_build_extra_body` 单一出口
- fix: LLM http 客户端默认绕过系统代理（`LLM_HTTP_TRUST_ENV=1` 恢复原行为）
- chore: tests/helpers 适配 https 端点迁移（网关 80 全量 301 后 IP 明文丢 Authorization）

## 0.2.0

- feat: 支持 `ANGINEER_CHAT_TEMPLATE_KWARGS` 环境变量透传 `chat_template_kwargs` 至 extra_body，并适配 DGX reasoning 字段

## 0.1.1

- fix: 判分 JSON 非法转义修复——清洗 LaTeX 转义（如 \L、\d）导致的 JSON 解析失败，避免语义判分按满分误计

## 0.1.0（对外发布基线）

本版本为面向外部消费方（DredgeAI）发布的首个稳定基线，改动均向后兼容：

### 新增
- 异步 API：`achat` / `achat_result` / `achat_stream` / `achat_stream_events` / `achat_result_guarded`（基于 `AsyncOpenAI`）。
- 超时四段完整生效：`connect` / `read` / `write` / `pool` 通过 `httpx.Timeout` 传入（新增 `ANGINEER_TIMEOUT_WRITE` / `ANGINEER_TIMEOUT_POOL`）。
- 统一错误层级：`LLMError` 基类 + `ProviderUnavailableError` / `ProviderAuthError` / `RateLimitedError` / `LLMStreamError` / `AllProvidersFailedError`（`errors.py`）。
- `ChatResult` 元数据：`latency_seconds` / `attempts` / `used_config` / `used_model` / `circuit_breaker_state`。
- 可选用量回调 `usage_callback`（不落库，回调异常不影响请求）。
- 熔断可观测：`success_count` / `total_calls` / `last_error_message` / `last_success_time`。
- `messages` / `tools` 输入校验（Pydantic），非法输入尽早抛 `ValueError`。
- 流式语义固化：首个 delta 前失败 → 换 Provider；已产出后失败 → `stream_failed` 事件 / `LLMStreamError`（携带 partial text）。
- 模块级单例 `get_llm_client()` 线程安全初始化。
- 依赖清理：移除 fastapi / uvicorn / python-multipart；显式声明 httpx。HTTP 运行时依赖移至 `aichat-api` / `docs-api`（各自新增 pyproject，Dockerfile 同步更新）。
- 包内测试由 2 个扩至 66 个；新增 README 与 CHANGELOG。

### 行为变更
- `default_model` 改为"优先排序"而非"钉死"：默认 Provider 失败时会 fallback 到其他可用 Provider；显式 `config_name` / `model` 仍严格过滤。
- 单 Provider 失败时直接抛出精确错误（如 `RateLimitedError`）；多个 Provider 全部失败才抛 `AllProvidersFailedError`。
- `chat_stream_events` 的 `done` 事件增加元数据字段（`used_config` 等）。
- `_prepare_messages` 不再修改调用方传入的 `messages`（内部浅拷贝）。
