# Agentic AgentGeneration Data Synthesis

这个项目实现你要的流程：

1. 用 GPT（默认 `gpt-5.4-nano`）根据 `query` 生成 **non-oracle** 的 `action_search_prompt`。
2. 用 `query + action_search_prompt` 去调用 Agno-driven NuvaAgent run API（Python 版，等价于你给的 curl）。
3. 再用 GPT 对 run 结果进行 critique，输出：
   - `match_score`（1-6，和 gold 的匹配）
   - `capability_score`（1-6，本身能力）
4. 自动计算：
   - `keep_for_sft` (`match_score >= 4 and capability_score >= 4`)
   - `quality_tag` (`strong_positive | positive | borderline | negative`)

## 关键文件

- `synth_pipeline.py`: 端到端 CLI。
- `sample_input.json`: 本地 mock 示例（已经包含 run）。
- `sample_input_nuva.json`: 真实调用 Nuva API 的输入示例（不含 run）。

## Nuva API 调用（按你给的 curl）

Python 内部会发：

- `POST {nuva_api_base}/agents/{nuva_agent_id}/runs`
- Header: `Authorization: Bearer <token>`
- `Content-Type: multipart/form-data`
- Form fields:
  - `message`
  - `stream=true`
  - `session_id`
  - `user_id`
  - `files`
  - `version`
  - `background=false`

默认参数：

- `--nuva-agent-id Nuva_v1`
- `--nuva-api-base https://api.example.com`（请换成你真实 API 域名）

Web 参考地址：

- `https://os.agno.com/chat?type=agent&id=Nuva_v1`

## 输入格式

### A. 已有 run（跳过 Nuva 调用）

```json
{
  "id": "partiii_000001",
  "query": "...",
  "gold_agent": {
    "backbone": "...",
    "tools": ["..."]
  },
  "run": {
    "session_id": "...",
    "session_history": [],
    "response_result": {}
  }
}
```

### B. 无 run（自动调用 Nuva）

```json
{
  "id": "partiii_000002",
  "query": "...",
  "gold_agent": {
    "backbone": "...",
    "tools": ["..."]
  }
}
```

## 使用方式

### 1) 本地 mock（不需要 key）

```bash
python3 synth_pipeline.py --input sample_input.json --output sample_output.json --mock
```

### 2) 真实 OpenAI + 真实 Nuva

```bash
export OPENAI_API_KEY=...
export NUVA_API_TOKEN=...
python3 synth_pipeline.py \
  --input sample_input_nuva.json \
  --output sample_output_nuva.json \
  --model gpt-5.4-nano \
  --nuva-api-base https://<your-real-api-domain> \
  --nuva-agent-id Nuva_v1
```

## 输出

输出 JSON 包含：

- 原始字段：`id/query/gold_agent`
- 生成字段：`action_search_prompt/capability_keywords/reasoning_type/expected_evidence`
- run 字段：`session_id/session_history/response_result/raw_backend_response`
- 评估字段：`match_score/capability_score/rationale/missing/noisy/keep_for_sft/quality_tag`

## 说明

- `gold_agent` 不参与搜索 prompt 生成，仅用于 critique。
- 支持两种后端返回格式：
  - 顶层包含 `session_history` / `response_result`
  - 或嵌套在 `run.session_history` / `run.response_result`
