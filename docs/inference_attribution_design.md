# LiteLLM Inference Attribution Design

Author(s): @simanadler

**Status:** Draft (2026-11-09)

This document designs the LiteLLM proxy changes needed to satisfy the [Inference Attribution Requirements](https://github.com/opencost/opencost/blob/develop/docs/inference-attribution-requirements.md) so OpenCost can attribute inference cost by tenant, workload and user. LiteLLM plays the **gateway and metering** roles from that document's Section 3. OpenCost owns cost. LiteLLM exports identity and token volume, never a monetary value.

Related documents:

- Inference Attribution Requirements (opencost repo, `docs/inference-attribution-requirements.md`). Referenced below as "the requirements".
- Inference Cost Attribution Plan (opencost repo, `docs/inference-cost-attribution-plan.md`). Referenced below as "the plan". Its Section 1.9 shows the OpenCost-side configuration this design must be compatible with.
- [ADR 001](adrs/adr_001_provider_usage_fields_ride_as_usage_extras.md). Provider usage fields ride as extras on the normalized usage object. Cached-token details (fact F12) are read from there and no side channel is introduced.

## 1. Assumptions

- The target deployment is the LiteLLM proxy in front of self-hosted engines (vLLM or any OpenAI-compatible server), which is the "LiteLLM in front of a plain vLLM" candidate from the requirements Section 10.
- Virtual keys backed by the LiteLLM database are the primary identity source. This is what makes tenant and workload trusted. Deployments without a database, or relying on JWT claim mapping (enterprise only), are discussed as issues in Section 6.
- OpenCost consumes aggregate Prometheus series only, per the plan. Per-request records (requirements 7.4) are phase 2 because no consumer needs them yet.
- The design targets this fork of LiteLLM, which has stream usage injection, partial-usage recovery on stream failure, and disconnect billing. Upstreaming is called out as an issue.

## 2. Fact-by-fact assessment of LiteLLM today

| Fact | Tier | LiteLLM today | Verdict |
|---|---|---|---|
| F1 `tenant_id` | MUST | Virtual key resolves to `team_id` and `org_id` through the combined-view join at auth time (`litellm/proxy/utils.py:6405-6450`, `litellm/proxy/auth/user_api_key_auth.py:2239`). Both are trusted, database-derived. No field is named "tenant". | Partial. Needs a mapping, not new machinery. |
| F2 `workload_id` | MUST | A key carries `key_alias`, `agent_id`, `project_id` and free-form `metadata` JSON (`litellm/models/verification_token.py:17-32`), all surfaced on `UserAPIKeyAuth` at auth time. No first-class workload entity. | Partial. Needs a mapping. |
| F3 `user_id` | MAY | `end_user_id` is resolved per request from the `user` param, `x-litellm-customer-id` or configured header mappings (`litellm/proxy/auth/auth_utils.py:1890-1973`). It is client-supplied and unbounded. | Partial. Needs opt-in emission and hashing. |
| F4/F5 prompt and completion tokens | MUST | `StandardLoggingPayload.prompt_tokens` / `completion_tokens` come from the provider `usage` object. The proxy injects `stream_options.include_usage` on tracked streaming routes and strips the extra chunk before the client sees it (`litellm/proxy/common_request_processing.py:666-700`, `litellm/litellm_core_utils/streaming_handler.py:2832-2841`). | Satisfied for chat and text completions. Gap on other surfaces, see Section 6.
| F6 requested model | MUST | `model_group` (post-alias) and `_litellm_client_requested_model` (pre-alias, `litellm/proxy/common_request_processing.py:3422-3427`) are both available. | Satisfied.
| F7 served model | MUST | `metadata["deployment"]` equals `litellm_params.model` and `model_id` identifies the deployment (`litellm/router_attempt.py:152-160`). The string is provider-qualified (`openai/foo`), so it is **not** equal by construction to the engine's `model_name`. | Partial. Needs derivation plus a per-deployment override.
| F8 deployment namespace | MUST | No Kubernetes namespace concept exists. `kubernetes_pod_routing` metadata carries `service_host` and pod IP, not a namespace (`litellm/router_utils/kubernetes_pod_discovery.py:481-516`). | Missing. Must be declared per deployment.
| F9 tokens on failure after prefill | SHOULD | The failure payload carries real tokens recovered from stream chunks via `combined_usage_object` (`streaming_handler.py:2776-2804`, `litellm/litellm_core_utils/litellm_logging.py:4598-4605`). But `PrometheusLogger.async_log_failure_event` increments no token metrics today (`litellm/integrations/prometheus.py:2748-2807`). | Data exists, export missing.
| F10 outcome | SHOULD | `status` plus `StandardLoggingPayloadErrorInformation` with `error_class` and `error_code`, including a `ClientDisconnected` special case (`litellm/litellm_core_utils/litellm_logging.py:7388-7484`). | Satisfied.
| F11 request ID | SHOULD | `litellm_call_id`, echoed to the client as `x-litellm-call-id`. | Satisfied.
| F12 cached prompt tokens | MAY | `usage.prompt_tokens_details.cached_tokens` rides the usage object per ADR 001 and reaches the payload's `metadata["usage_object"]`. | Satisfied.

## 3. Design

### 3.1 Identity resolution

Attribution identity is resolved **once**, inside the trusted auth boundary, and then rides the logging payload. The resolution is a pure function of the `UserAPIKeyAuth` object plus configuration, executed in `add_user_api_key_auth_to_request_metadata` (`litellm/proxy/litellm_pre_call_utils.py:2117-2129`), which is the single choke point where auth fields already reach every logging callback and SpendLogs.

Each dimension has a configurable source. The defaults follow the conventional LiteLLM deployment the requirements describe in Section 10 (keys per application, teams per tenant).

| Dimension | Default source | Other allowed sources |
|---|---|---|
| `tenant_id` | `team_id` | `org_id`, `key_metadata:<json path>`, `team_metadata:<json path>`, `org_metadata:<json path>`, `none` |
| `workload_id` | `key_alias` | `agent_id`, `project_id`, `project_alias`, `key_metadata:<json path>`, `none` |
| `user_id` | `end_user_id` | `user_id` (the key owner), `none` |

Rules that follow from the requirements Section 4:

- Sources are restricted to fields the gateway itself resolved or stored. A client header is never an identity source. The one exception, a verified workload header, is phase 2 and described in Section 3.4.
- A dimension that resolves to an empty or missing value is emitted as an empty label, which OpenCost reports as unattributed. It is never defaulted to a team name, a fallback tenant, or `"other"`.
- `user_id` may be hashed. When `user_id_hash` is `keyed`, the emitted value is HMAC-SHA256 of the raw end-user id under an operator-supplied secret (env var, never written to config files), truncated to 16 hex characters. `plain` emits the raw value. `off` omits the label entirely, matching the requirements Section 4.2 default. A bare or unkeyed hash is not offered.

The resolved tuple is stamped onto the request metadata as an `inference_attribution` block (Section 3.4), so every sink sees the same values and no sink re-derives them.

### 3.2 Prometheus metrics

Two new counters, registered by `PrometheusLogger` and gated by the attribution config (default off, so existing deployments are unaffected):

| Metric | Type | Labels |
|---|---|---|
| `litellm_attributed_input_tokens_total` | Counter | `tenant_id`, `workload_id`, `serving_model`, `namespace` |
| `litellm_attributed_output_tokens_total` | Counter | `tenant_id`, `workload_id`, `serving_model`, `namespace`, `user_id` (only when `user_id` source is not `none`) |

Decisions:

- **New metrics, not reuse of `litellm_input_tokens_metric`.** The existing token counters carry `requested_model`, `hashed_api_key`, `api_provider` and other labels the requirements say to exclude (Section 7.3, `requested_model` inflates cardinality and does not change cost), and their `model` label is the provider-qualified name, which breaks the engine-join. Dedicated counters keep the OpenCost contract exact and let label defaults match the plan's Section 1.9 (`tenant_id`, `workload_id`, `serving_model`, `namespace`, `user_id`) with zero OpenCost configuration.
- **`serving_model` label value.** Derived per deployment as the provider-prefix-stripped `litellm_params.model` (`openai/foo` becomes `foo`), overridable by a `model_info.serving_model` field on the deployment (Section 3.6). For an OpenAI-compatible engine the model string sent to the provider must match the engine's served model name anyway, so the derived default is normally right, and the override covers every other case. The label is emitted from the deployment that actually served the request, so canary and multi-deployment model groups attribute correctly.
- **`namespace` label value.** From the per-deployment `model_info.inference_namespace` field (Section 3.6). Missing value emits an empty label. This is operator-declared because LiteLLM cannot discover it, and the OpenCost plan Section 1.3.2 already documents why the scrape target's namespace is the wrong value for a gateway-hosted producer.
- **Failure increments.** Both counters are incremented in `async_log_success_event` **and** `async_log_failure_event`. On failure the payload's tokens come from the partial-usage recovery path, so F9 (prompt tokens after prefill began) is exported with no estimate. Requests rejected before inference (auth failures, malformed bodies) never reach LLM logging and are simply not counted, which is the required behavior. The proxy-level failure hook (`async_post_call_failure_hook`) is not used for token counting because it has no usage.
- **Counted once.** Success fires once per logical request with the assembled stream response, and request-level failure dispatch is deduplicated per logical request, so in-stack retries and fallbacks produce exactly one increment. This is a property the conformance tests pin down (Section 5).
- **Cardinality.** The counters go through the existing `_LabeledMetric` wrapper and series caps (`prometheus_metrics_max_series_per_metric` and friends), and when the `user_id` label is enabled they additionally use the bounded end-user series tracker with its own limits. `tenant_id` and `workload_id` are bounded by the number of teams and keys, which is the cardinality the requirements Section 4 assumes.
- **Multi-worker.** Counters work under `PROMETHEUS_MULTIPROC_DIR` the same as the existing ones, so the two-worker proxy QA setup from the repo instructions applies unchanged. The `/metrics` endpoint and its auth (`require_auth_for_metrics_endpoint`) are reused as-is, and OpenCost's scraper needs a key for it unless that is disabled.

OpenCost-side configuration this maps onto (plan Section 1.9):

```yaml
INFERENCE_ATTRIBUTION_ENABLED: "true"
INFERENCE_ATTRIBUTION_INPUT_TOKENS_SERIES: "litellm_attributed_input_tokens_total"
INFERENCE_ATTRIBUTION_OUTPUT_TOKENS_SERIES: "litellm_attributed_output_tokens_total"
# label names match the defaults, so no INFERENCE_ATTRIBUTION_*_LABEL overrides are needed
```

### 3.3 Failure and stream semantics

The requirements Section 6 cases map onto LiteLLM paths as follows.

| Requirements case | LiteLLM path | Attributed export |
|---|---|---|
| Success | `async_log_success_event`, tokens from provider `usage` | Input and output incremented, `usage_captured=true` |
| Rejected before inference | Fails in auth or pre-call, never reaches LLM logging | Nothing counted |
| Failure after prefill began | `async_log_failure_event` with `combined_usage_object` recovery | Input incremented from recovered prompt tokens, output zero, `usage_captured=false` |
| Stream truncated or interrupted | Same failure path, chunks already received are assembled by `stream_chunk_builder` | Input incremented. Output from assembled chunks is exact for delta-style providers (each chunk carries its own deltas), not an estimate, so it is exported with `usage_captured=false` meaning "no provider usage object", per the requirements note that a chunk-count value must be flagged |
| Client disconnect mid-stream | Proxy assembles the partial response and dispatches success logging (`_bill_partial_streamed_spend_on_disconnect`, `common_request_processing.py:505-571`), deduplicated against a natural end-of-stream dispatch | Same as success, with `usage_captured` reflecting whether the provider usage chunk arrived |
| Cache-hit short circuit | Normal success path with whatever `usage` the provider sent | As reported |

This requires one new logging-payload field, `usage_captured`. It is true when the provider's own usage object was received (final usage chunk or non-streaming response) and false when tokens came from chunk assembly, recovery, or are absent. The stream wrapper already knows whether a usage-only chunk arrived because it records and strips it (`_record_and_strip_usage_chunk`), so the flag is set where that happens and read in `get_standard_logging_object_payload`. It is a LiteLLM logging field, not a provider usage extra, so it does not conflict with ADR 001.

The requirements' concern about overriding a client's `stream_options.include_usage` does not apply as written, because LiteLLM strips the injected usage chunk before the client sees it. The client-visible response is unchanged.

### 3.4 Per-request records (phase 2)

The requirements Section 7.4 asks for durable per-request records carrying F1 to F11 for billing-grade or per-user attribution where Prometheus cardinality is impractical. OpenCost does not consume these yet, so this is deliberately phase 2, but the payload groundwork lands in phase 1.

Phase 1 adds an `inference_attribution` block to `StandardLoggingMetadata`:

```python
class StandardLoggingInferenceAttribution(LiteLLMPydanticObjectBase):
    tenant_id: str = ""
    workload_id: str = ""
    user_id: str = ""          # hashed or plain per config, empty when unset
    serving_model: str = ""
    namespace: str = ""
    usage_captured: bool = True
```

Stamped once at the pre-call choke point (Section 3.1), it reaches every callback and SpendLogs for free, and the Prometheus counters read it instead of re-deriving identity.

Phase 2 then consists of, in order of preference:

1. A webhook-style callback (following `OpenMeterLogger`, `litellm/integrations/openmeter.py`) that emits one JSON record per request with `litellm_call_id`, the attribution block, requested model (`model_group`), prompt, completion and cached tokens, `status`, `error_class` and timestamps. F6 through F11 are all already on the payload.
2. Optionally a management endpoint to query stored records. If SpendLogs is the store, the attribution block rides `SpendLogsMetadata` (which already carries arbitrary passthrough dicts and `kubernetes_pod_routing`) rather than new columns.

Phase 2 also includes the **verified workload header**. An operator may enable accepting `x-litellm-workload-id` from a client, but only when the key's metadata carries an allowlist of permitted values, and a value outside the allowlist is recorded as unattributed. That keeps the requirements Section 4.1 rule intact, because the gateway verifies the header against data it owns rather than trusting it unchanged.

### 3.5 Response headers (optional)

LiteLLM already returns `x-litellm-model-id`, `x-litellm-model-name` and `x-litellm-model-api-base` (`common_request_processing.py:2429-2436`). When attribution is enabled it additionally emits the requirements Section 7.2 headers, `x-inference-served-model` and `x-inference-namespace`, so a downstream ext-proc metering service can consume them. This is config-gated and off by default because the header names are still provisional in the requirements document. `x-inference-prompt-tokens` is not emitted, because LiteLLM does not tokenize before inference for OpenAI-compatible engines and the requirements only ask for it when the router knows the value.

### 3.6 Configuration

Attribution config rides `litellm_settings` and is validated at proxy startup the same way `prometheus_deployment_and_latency_caller_identity` is (`litellm/proxy/proxy_server.py:6859-6865`).

```yaml
litellm_settings:
  success_callback: ["prometheus"]
  inference_attribution:
    tenant_id: team_id            # org_id | key_metadata:<path> | team_metadata:<path> | org_metadata:<path> | none
    workload_id: key_alias        # agent_id | project_id | project_alias | key_metadata:<path> | none
    user_id: end_user_id          # user_id | none
    user_id_hash: keyed           # keyed | plain | off  (label omitted when off)
    emit_response_headers: false
general_settings:
  # or the master environment variable for the HMAC secret
  # LITELLM_ATTRIBUTION_USER_HASH_SECRET
```

Per-deployment fields in `model_list` (both are plain `model_info` extra fields, which `ModelInfo` allows and which already flow into request metadata at `litellm/router_attempt.py:152-160`):

```yaml
model_list:
  - model_name: llama-3-70b              # what clients ask for (F6, post-alias group)
    litellm_params:
      model: openai/meta-llama/Llama-3-70B-Instruct
      api_base: http://llama.llm-d-prod.svc.cluster.local:8000
    model_info:
      inference_namespace: llm-d-prod    # F8
      serving_model: meta-llama/Llama-3-70B-Instruct   # F7 override, optional
```

Startup validation fails fast on a config that names a metadata path source that can never resolve (for example `key_metadata:attribution.tenant` when no key carries it is legal, but a malformed path is not) and logs a warning listing deployments that lack `inference_namespace`, because a missing namespace silently joins to nothing on the OpenCost side.

### 3.7 Cost exclusion

The requirements Section 1 says the serving stack never reports a monetary value, and the plan's Appendix B flags gateway-computed cost as a source-of-truth conflict. The attribution metrics and records therefore contain no cost fields. `response_cost` and `cost_breakdown` stay on the payload for LiteLLM's own budget features and are simply not exported by the attribution path. For self-hosted engines LiteLLM's own cost is zero unless custom pricing is configured, which is expected and fine, since OpenCost owns the rate.

## 4. File changes

| File | Change | What |
|---|---|---|
| `litellm/types/utils.py` | Extend | `StandardLoggingInferenceAttribution`, `StandardLoggingMetadata.inference_attribution`, `StandardLoggingPayload.usage_captured` |
| `litellm/proxy/litellm_pre_call_utils.py` | Extend | Resolve attribution identity in `add_user_api_key_auth_to_request_metadata`, stamp the block on metadata |
| new `litellm/litellm_core_utils/inference_attribution.py` | Add | Pure identity-resolution and HMAC helpers, config type, source parsing |
| `litellm/litellm_core_utils/litellm_logging.py` | Extend | Copy `usage_captured` into the payload in `get_standard_logging_object_payload` |
| `litellm/litellm_core_utils/streaming_handler.py` | Extend | Set the usage-captured signal when a provider usage chunk arrives |
| `litellm/types/integrations/prometheus.py` | Extend | Two metric names in `DEFINED_PROMETHEUS_METRICS`, label lists in `PrometheusMetricLabels` |
| `litellm/integrations/prometheus.py` | Extend | Construct the two counters behind the config gate, increment in `async_log_success_event` and `async_log_failure_event` |
| `litellm/proxy/common_request_processing.py` | Extend | Optional `x-inference-served-model` and `x-inference-namespace` response headers |
| `litellm/proxy/proxy_server.py` | Extend | Startup validation of `inference_attribution` config and namespace warnings |
| `litellm/__init__.py` | Extend | Module attrs for the config, following the existing prometheus settings pattern |
| phase 2, `litellm/integrations/` | Add | Per-request record callback (webhook sink) and the verified workload header |

The identity-resolution decision (resolved once at auth, rides the standard logging payload, no per-sink re-derivation) is the kind of data-movement decision `docs/adrs/AGENTS.md` asks to record, so the implementing PR adds an ADR for it in the same PR.

## 5. Tests and QA

Conformance tests map one-to-one onto the requirements Section 9 table. All are unit tests under `tests/unit/` mirroring the litellm paths, plus proxy-level tests for the config wiring.

| Requirements test | Test |
|---|---|
| Non-streaming success with identity | `async_log_success_event` increments both counters with resolved labels, one record |
| Streaming success | Usage captured from the assembled stream, counters incremented once |
| Request without identity | Empty label values, no defaulting to a team or `"other"` |
| Model rewritten by the router | `serving_model` is the deployment's model, `model_group` (requested) recorded separately |
| Failure after prefill | Input counter incremented from recovered tokens, output zero, `usage_captured=false` |
| Rejected before inference | No counter increments from the auth-failure path |
| Client disconnect mid-stream | Partial tokens attributed exactly once (the disconnect-billing dispatch is deduplicated against natural end-of-stream) |
| Retries and fallbacks | One increment per logical request across deployment attempts |
| Coverage | Sum of attributed input tokens equals the sum of payload prompt tokens across a mixed success, failure and disconnect scenario |

Regression risk to pin explicitly. The existing `PrometheusLogger` failure path increments no token metrics, and the disconnect path dispatches *success* logging for a partial stream. Both behaviors are load-bearing for this feature and both have subtle dedup semantics, so the disconnect and retry tests must fail if either dispatch fires twice or zero times.

QA follows the repo instructions. Run the proxy with two workers and shared Redis, make real streaming and non-streaming calls against a vLLM deployment (a local OpenAI-compatible mock standing in for vLLM where a real engine is unavailable), and show `curl /metrics` output containing the attributed counters with correct labels before and after the change. The before run shows the metrics absent, the after run shows them present with tokens matching the response `usage`.

## 6. Issues

These are the places where LiteLLM does not meet the requirements as written, or where the design carries a risk worth flagging.

1. **`serving_model` is not equal to the engine's `model_name` by construction (F7).** LiteLLM's deployment model strings are provider-qualified (`openai/foo`) and the public `model_name` group is a client-facing alias that can front several deployments. The derived default (prefix-stripped `litellm_params.model`) is right for OpenAI-compatible engines because the engine rejects unknown model names, but a wrong or missing `serving_model` silently joins to nothing or to the wrong model on the OpenCost side. The `model_info.serving_model` override plus the startup warning are the mitigation, and the conformance test for rewrites pins it.

2. **Deployment namespace has no discoverable source (F8).** LiteLLM cannot know which Kubernetes namespace a deployment's `api_base` points into. It must be declared per deployment, and a missing declaration is a silent empty label. This is the same silent-failure mode the plan flags in its Section 1.3.2. The startup warning helps, but a wrong declaration (typo in the namespace) still joins to nothing. The `llm_dimension_unmatched_keys` metric on the OpenCost side is the backstop that makes it visible.

3. **Trusted identity requires the database, and JWT claim mapping is enterprise-only.** The requirements Section 4.1 trust boundary is satisfied by virtual keys resolving team and org through the combined-view join, which needs the LiteLLM database. This is the "Needs a database" concern from the requirements Section 10 table, confirmed. A database-less deployment can only run with the master key, which makes every request one identity and defeats attribution. Mapping tenants from JWT claims without keys would avoid per-app key management, but JWT auth is enterprise-gated in this tree, so an OSS deployment cannot use that path.

4. **`user_id` (end-user) is client-supplied.** The authenticated identity is the key, team and user behind it. The end-user id arrives in the request body or a header set by the calling application, so a compromised or sloppy app can mislabel users within its own tenant. This only distorts that tenant's own user breakdown, which the requirements accept since `user_id` is MAY and the tenant and workload dimensions stay trusted, but operators should know the boundary. Cardinality is the bigger practical risk, which is why the label is off by default and goes through the bounded series tracker when on.

5. **Streaming usage injection does not cover every API surface.** `stream_usage_tracking_updates` handles `acompletion` and `atext_completion` routes only (`common_request_processing.py:663`). Streaming calls on `/v1/messages` and `/v1/responses` do not get `include_usage` injected. Anthropic-style streams always carry usage in `message_delta`, but OpenAI-style `/v1/responses` streams that omit usage would be exported with `usage_captured=false` and zero output tokens. If those surfaces matter for a deployment, the injection needs extending, and the requirements Section 6 says such records must be marked, which the `usage_captured` flag does.

6. **No `usage_captured` signal exists today.** The payload does not distinguish "provider told me the tokens" from "I assembled the tokens from chunks". The flag is new and must be set at the two places that know (usage chunk receipt, non-streaming response parse), or the marking requirement is unmet. It is small, but it touches the logging payload, which every sink reads, so it needs the full callback test sweep.

7. **Series-cap overflow misattributes rather than undercounts.** The `_LabeledMetric` overflow series preserves totals but collapses label values to `"other"` (or the end-user tracker evicts by LRU). For spend metrics that is the right tradeoff. For attribution it means the OpenCost coverage check still passes (totals match the engine) while the per-tenant split quietly degrades. Mitigation is operational, keep the caps comfortably above the real team and key counts, and the unmatched-keys and coverage signals will not catch it. This is worth documenting in the operator docs for the feature.

8. **Cost overlap is resolved by exclusion, not removal.** LiteLLM keeps computing spend for its own budgets, max-budget enforcement and UI. Nothing in this design removes or disables that. The requirements Section 11 open question about metering ownership is answered here only for the OpenCost feed, which carries no cost. An operator reading both LiteLLM spend dashboards and OpenCost attributed cost for self-hosted models will see two different numbers with two different meanings, and the docs must say so plainly.

9. **This fork has diverged from upstream.** The partial-usage recovery on stream failure, disconnect billing and Kubernetes pod routing that this design relies on exist in this tree. If the implementation is upstreamed, each dependency needs checking against upstream LiteLLM, and the failure-path token export in particular may need the recovery feature upstreamed first.

10. **`requested_model` on existing metrics invites the wrong join.** Operators already scraping `litellm_input_tokens_metric` may point OpenCost at it because it looks similar. Its `model` label is provider-qualified and it carries `requested_model`, so the join it produces is wrong in both directions. The operator docs for the feature should name the two new series explicitly and warn against the existing ones.

## 7. Open questions

- **Header names.** The requirements Section 7.2 names are placeholders pending agreement with serving-stack maintainers. Emitting them is config-gated and off by default so a rename is cheap, but the flag name should not bake in the provisional names either.
- **Per-request record sink.** Webhook callback versus SpendLogs query versus a dedicated table is a phase 2 decision. The payload groundwork (the `inference_attribution` block) is the same either way.
- **End-user budget coupling.** `end_user_id` today also drives end-user budget tracking, which has its own opt-in flags (`enable_end_user_cost_tracking_prometheus_only`). The attribution `user_id` source deliberately reads the resolved id directly rather than going through `get_end_user_id_for_cost_tracking`, so attribution works without enabling end-user budgets. If that independence is unwanted, the two configs should be merged before phase 1 ships.
- **Wildcard and expanded-model rows.** Recent routing work prices expanded wildcard `/model/info` rows as the model calls route to. If a deployment's served model is resolved per call rather than per config row, the `serving_model` label derivation must use the per-call resolution, which is what `metadata["deployment"]` already is. Worth a dedicated test if wildcard model groups are in scope.
