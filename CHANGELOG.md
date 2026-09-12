# Changelog

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
