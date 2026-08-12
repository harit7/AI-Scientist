# llm_runtimes integration notes

Additive integration of the vendored `llm_runtimes/` package (embedded
OpenAI-compatible server) into the AI-Scientist v1 scaffold. No existing model
paths, templates, or scaffolding were modified; only new model names were added.

## Files changed

- `ai_scientist/llm.py`
  - `AVAILABLE_LLMS`: added `claudecli-sonnet`, `claudecli-opus`,
    `claudecli-haiku`, `local-qwen`. All argparse `--model` choices in
    `launch_scientist.py`, `generate_ideas.py`, and `perform_writeup.py` read
    this list, so the new models are selectable everywhere without further edits.
  - `create_client()`: new first branch routes `claudecli-*` / `local-*` to
    `openai.OpenAI(base_url=ensure_server(), api_key="llm-runtimes")`.
  - `get_response_from_llm()`: new first branch sends `claudecli-*` / `local-*`
    through the standard OpenAI chat-completions path. Placed before the
    existing `"claude" in model` check, which would otherwise misroute
    `claudecli-*` to the Anthropic SDK.
  - `get_batch_responses_from_llm()`: new first branch, same routing with
    `n=n_responses`.
- `llm_runtimes/` — vendored runtime package (server, claude CLI backend,
  in-process vLLM backend); newly tracked, see `llm_runtimes/README.md`.
- `RUNTIMES_NOTES.md` — this file.

No other file has its own model dispatch (`perform_review.py` uses the llm.py
helpers; `review_iclr_bench/iclr_analysis.py` is a standalone benchmark script
and was left untouched).

## Usage examples

Usual API backend (unchanged):

```python
from ai_scientist.llm import create_client, get_response_from_llm
client, model = create_client("gpt-4o-2024-05-13")
text, hist = get_response_from_llm("Say hi", client, model, "You are helpful.")
```

Claude CLI backend (subscription auth via local `claude` binary, no API key):

```python
from ai_scientist.llm import create_client, get_response_from_llm
client, model = create_client("claudecli-haiku")   # or -sonnet / -opus
text, hist = get_response_from_llm("Say hi", client, model, "You are helpful.")
```

Local vLLM backend (GPU, no API key):

```python
from ai_scientist.llm import create_client, get_response_from_llm
client, model = create_client("local-qwen")
text, hist = get_response_from_llm("Say hi", client, model, "You are helpful.")
```

Or from the CLI, e.g. `python launch_scientist.py --model claudecli-sonnet ...`.

## Known limitations

- `claudecli-*`: `temperature` is ignored (the `claude -p` CLI does not expose
  sampling controls); usage token counts in responses are zeroed.
- `local-*`: default model is `Qwen/Qwen3-8B-AWQ`, loaded in-process by vLLM on
  the GPU selected by `LLM_RUNTIMES_LOCAL_GPU` (default 0). First request pays
  the engine warm-up cost.
- The embedded server binds `LLM_RUNTIMES_PORT` (default 8399); concurrent
  processes on the same machine share one server instance.
- Aider-driven experiment/writeup stages in `launch_scientist.py` construct
  their own `aider.models.Model(model)`; the new backends cover the
  `create_client`-based stages (idea generation, novelty check, review,
  citation/writeup LLM calls), not Aider's internal model access.
