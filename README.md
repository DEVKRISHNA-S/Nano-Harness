# Nano Harness

A single-file, ~200-line **code-first agent loop** — built for *reading and learning*, not production use. It shows the whole agent loop (call LLM → parse code → execute → observe → repeat) in one file with no framework.

>  Educational only. it is not use for real work. No real production-grade sandboxing — it's a teaching tool for understanding how agent harnesses work under the hood.

## What it does

Given a `TASK` string, the agent:
1. Calls an LLM with the system prompt + task
2. The model replies with a fenced ` ```python ` code block (no prose)
3. That code is executed in a restricted `exec()` sandbox with a handful of tool functions available
4. stdout / stderr / exceptions are captured and fed back to the model as the next message
5. Repeats until the model calls `final_answer(result)` or `MAX_STEPS` (50) is hit

## Project structure

```
nano-harness/
├── nano_harness.py     # the whole agent — single file
├── .env                 # your real secrets (gitignored)
├── .env.example          # template, safe to commit
└── README.md
```

## Setup

```bash
git clone <your-repo-url>
cd nano-harness
python3 -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install openai python-dotenv
```

Create `.env` in the project root:

```env
# Using Groq (OpenAI-compatible API, has a real free tier)
OPENAI_BASE_URL=https://api.groq.com/openai/v1
HF_TOKEN=your_groq_api_key_here
NANO_MODEL=openai/gpt-oss-20b
```

> Note: despite the var name `HF_TOKEN` (left over from the original Hugging Face router design), this currently holds a **Groq** API key, since HF's own free inference credits ($0.10/month) run out almost immediately. Get a free Groq key at [console.groq.com/keys](https://console.groq.com/keys) — no card required.

## Run

```bash
python nano_harness.py
```

You'll see step-by-step output: the raw model response, the extracted code, and the observation fed back on each iteration, until either:
- `✓ Task complete: <result>` — the model called `final_answer()`, or
- `✗ Max steps reached` — it hit the 50-step cap without finishing

## Configuration (top of file)

| Constant | Meaning |
|---|---|
| `TASK` | The task string given to the agent |
| `MODEL` | Model ID, from `NANO_MODEL` env var |
| `BASE_URL` | API base URL, from `OPENAI_BASE_URL` env var |
| `API_KEY` | From `HF_TOKEN` (or `OPENAI_API_KEY`) env var |
| `WORKSPACE` | Directory tool access is confined to (current working dir) |
| `MAX_STEPS` | Hard cap on loop iterations (50) |
| `TIMEOUT_S` | Timeout for shell commands / web fetches (30s) |
| `MAX_CHARS` | Max chars of any tool output before truncation (8000) |
| `ALLOW_WRITE` | Whether `write_file` is enabled (False by default) |
| `ALLOW_COMMANDS` | Shell command allowlist: `ls cat pwd echo head tail wc rg` |
| `TEMPERATURE` | Sampling temperature for the LLM call (0.2, low for determinism) |

## Tools available to the model

| Tool | Does | Safety |
|---|---|---|
| `list_dir(path='.')` | List directory contents | Path confined to `WORKSPACE` |
| `read_file(path, max_chars=4000)` | Read a file | Path confined, size-limited |
| `write_file(path, content)` | Write/create a file | Path confined, **off by default** |
| `exec_cmd(args)` | Run a shell command | Allowlist only |
| `web_fetch(url, max_bytes=10000)` | Fetch a webpage | Byte-limited, errors caught |
| `hf_search(query, resource_type='models', limit=5)` | Search Hugging Face Hub API | Requires `API_KEY` set |
| `final_answer(value)` | Ends the loop, returns the result | — |

All tools are defined as closures inside `main()`, sharing `ws` (resolved workspace path) and `done`/`final_result` state via `nonlocal`.

## Sandboxing notes

- **Path confinement**: `safe_path()` resolves every path against `WORKSPACE` and rejects anything that escapes it (`Path.relative_to` check).
- **Command allowlist**: `exec_cmd` rejects anything not in `ALLOW_COMMANDS` before it reaches `subprocess.run`.
- **Restricted `exec()`**: code runs with `"__builtins__": {}` — the model's code has *no* access to Python builtins (no `import`, no `open`, etc.) beyond the tool functions explicitly injected into `exec_globals`. This is intentional but strict: e.g. the model cannot `import typing` — it has learned (via the error-feedback loop) to write plain code without imports.
- **Output size limits**: every tool output is passed through `clip()` to stay under `MAX_CHARS`.
- **Errors become observations, not crashes**: `FileNotFoundError`, `PermissionError`, `TimeoutExpired`, and any other exception are caught, converted into a string, and fed back to the model as the next turn — this is how the agent "learns" to correct course mid-task.

## Known limitations / things to watch

- Uses `client.chat.completions.create()` (not the newer Responses API) — some reasoning models (e.g. GPT-OSS family) attempt native tool-calling against `responses.create()` and throw `tool_use_failed` errors; `chat.completions` avoids this.
- `hf_search` requires network access to `huggingface.co` even when the LLM itself is served via Groq.
- No retry/backoff on API errors — a single failed call will raise and stop the whole run.
- `WORKSPACE` defaults to the current working directory — run the script from a dedicated folder, not your home directory, to avoid giving the agent `list_dir`/`read_file` access to unrelated files.

