# Agentic AgentGeneration Data Synthesis

A lightweight pipeline to build **agentic trajectory training data** with the design you requested:

1. Use GPT (default: `gpt-5.4-nano`) to generate a non-oracle `action_search_prompt` from `query`.
2. Feed that prompt into your controller/backend (Nuva) and collect real `session_history` + `response_result`.
3. Use GPT again to critique quality with two scores:
   - `match_score` (1-6): similarity to `gold_agent`
   - `capability_score` (1-6): standalone problem-solving capability
4. Auto-compute:
   - `keep_for_sft` (`match_score >= 4 and capability_score >= 4`)
   - `quality_tag` (`strong_positive | positive | borderline | negative`)

## File overview

- `synth_pipeline.py`: End-to-end CLI pipeline.

## Input format

Provide one JSON sample with the following shape:

```json
{
  "id": "partiii_000001",
  "query": "An investigator is studying the incidence of the common cold among medical students...",
  "gold_agent": {
    "backbone": "mistralai__Mixtral-8x7B-Instruct-v0.1",
    "tools": ["Stress-induced immunosuppression markers during exam periods"]
  },
  "run": {
    "session_id": "nuva_session_123",
    "session_history": [
      {"step": 1, "action": "SearchLLM", "argument": "...", "observation": "..."},
      {"step": 2, "action": "SearchTool", "argument": "...", "observation": "..."},
      {"step": 3, "action": "BuildContext", "argument": "...", "observation": "..."},
      {"step": 4, "action": "Generate", "argument": "...", "observation": "..."}
    ],
    "response_result": {
      "backbone": "...",
      "tools": ["..."],
      "profile": "...",
      "policy": "..."
    }
  }
}
```

## Usage

### 1) Mock run (no API key)

```bash
python3 synth_pipeline.py --input sample_input.json --output sample_output.json --mock
```

### 2) Real OpenAI run

```bash
export OPENAI_API_KEY=... 
python3 synth_pipeline.py --input sample_input.json --output sample_output.json --model gpt-5.4-nano
```

## Output format

The generated JSON includes:

- Original fields (`id`, `query`, `gold_agent`, `run`)
- Generated search guidance:
  - `action_search_prompt`
  - `capability_keywords`
  - `reasoning_type`
  - `expected_evidence`
- Evaluation:
  - `match_score`, `capability_score`
  - rationales and missing/noisy capability lists
  - `keep_for_sft`, `quality_tag`

## Notes

- `gold_agent` is **not used to generate** `action_search_prompt`; it is only used in critique/evaluation.
- Score definitions and bucketing logic are encoded in the system prompts and post-processing logic.
