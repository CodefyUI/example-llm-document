# example-llm-document

A small CodefyUI service that summarizes an arbitrary document into three bullet points using a local LLM served by Ollama. It is a self-contained CodefyUI **project directory** -- a git repo that holds the graph, its canvas layout, project-scoped assets, and an env template together -- so you clone it and run it entirely on your own machine with `cdui start --project .`; with Ollama running, the pipeline needs no API key at all.

## How it works

| Node | Role in this service |
| --- | --- |
| `start-1` (`Start`) | Entry point of the graph. Fires the trigger that starts a run; has no params. |
| `input-document` (`GraphInput`) | Declares the API input `document` (required string) and supplies its value to the rest of the graph. Ships with a long built-in default so a canvas Run works without typing anything. |
| `llm-1` (`LLMChat`) | Sends the document text to a chat model and returns its reply. Configured for local Ollama, model `llama3.2`, at `http://127.0.0.1:11434/v1`, with a fixed `system_prompt` instructing a 3-bullet summary (`temperature` 0.2, `max_tokens` 512). |
| `output-result` (`GraphOutput`) | Declares the API output `result` and exposes the model's reply text. |

Start's trigger edge fires `input-document` first. A data edge then carries the `document` value into `llm-1`'s `text` input (the node's own `prompt` param is left blank -- the live text arrives over that wired input instead). `llm-1` calls Ollama with the fixed `system_prompt` as instruction and your document as the user message; its `text` output flows into `output-result`'s `value` input, exposed by the API as `result`.

## Quick start

```bash
git clone https://github.com/CodefyUI/example-llm-document
cd example-llm-document
```

Before running, make sure a local Ollama server is running with `llama3.2` pulled -- see "Requirements and troubleshooting" below for the exact steps. This graph has no offline fallback: the `LLMChat` node always calls Ollama.

```bash
cdui start --project .
```

This prints the project path, a short git SHA, and a URL to open. Open that URL, load the `llm-document` graph from the toolbar, and press **Run**. The `document` input carries a built-in default value, so the run works with no typing at all; the generated 3-bullet summary appears on the `LLMChat` and `GraphOutput` nodes once Ollama responds. On a cold start, Ollama loading `llama3.2` from disk can take tens of seconds (about 35s was measured while verifying this example); once the model is warm, short documents finish in a few seconds.

## API

### Inputs

| Name | Type | Required | Default | Description |
| --- | --- | --- | --- | --- |
| `document` | string | Yes | `"CodefyUI is a visual environment for building AI systems as node graphs. Saved graphs can be published as versioned HTTP APIs: GraphInput and GraphOutput nodes define the contract, API keys control access, and every invocation is recorded for inspection. This default text exists so you can press Run on the canvas without typing anything."` | The document text to process. API callers must send this as a string. |

### Outputs

| Name | Description |
| --- | --- |
| `result` | The generated summary. |

### Example

Request body:

```json
{
  "inputs": {
    "document": "CodefyUI is a visual programming environment for building and understanding AI systems. Users assemble graphs from nodes that represent data sources, tensor operations, model layers, and training loops, then execute them directly on the canvas. The 1.2 release introduced a publishing workflow that turns any saved graph into a versioned HTTP API. A graph declares its contract through GraphInput and GraphOutput nodes; once published, external clients can invoke it with an API key and receive structured JSON results, while every run is recorded in a local SQLite database for inspection. The team also shipped a plugin system with six curriculum packs that mirror the chapters of an accompanying textbook, covering topics from classical machine learning to transformer-based language models. Plugins are installed from Git repositories with a single command and can contribute custom nodes, example graphs, and documentation. Upcoming work focuses on an inference-first application builder: the goal is that a student who has sketched a working pipeline on the canvas can hand a stable, documented endpoint to a teammate the same afternoon, without writing any server code. Local-first execution remains the default, with GPU acceleration handled automatically when available."
  },
  "timeout_s": 120
}
```

Response (a real recorded run against this graph, reproduced from this example's verified walkthrough):

```json
{
  "status": "ok",
  "run_id": "8df8d6284cde4f7aa7eb2b1da9b63082",
  "graph": "llm-document",
  "app": "llm-document",
  "version": 1,
  "device": "cpu",
  "outputs": {
    "result": "Here are 3 concise bullet points summarizing the provided document:\n\n\u2022 CodefyUI is a visual programming environment for building and understanding AI systems...\n\u2022 The platform has introduced a publishing workflow that enables external clients to invoke published graphs with an API key...\n\u2022 Future work focuses on developing an inference-first application builder..."
  },
  "error": null,
  "timing": { "total_s": 34.718 }
}
```

The envelope shape and the `run_id` / `version` / `device` / `timing` fields are real; `outputs.result` wording is not deterministic -- Ollama can phrase the same 3-bullet summary differently between runs even at `temperature` 0.2, so treat the exact text above as illustrative, not a byte-for-byte guarantee.

## Publish it as a real API

```bash
cdui project validate .
```

runs the full publish pre-flight against every graph in the project: the secret-in-graph check, the input/output contract, entry points, wiring, and node/preset validity; it also fails if `.env` is tracked by git. Run this as your CI gate.

Commit your changes -- `publish` only warns on a dirty tree, but the commit it records should match what you actually tested -- then publish:

```bash
cdui project publish . --graph llm-document --slug llm-document
```

`codefyui.project.toml` sets no default `[publish]` target in this repo, so pass `--graph`/`--slug` explicitly every time. `publish` confirms this project is the one the running server has open, then records `git rev-parse HEAD` and `git status --porcelain` (`git_commit`, `git_dirty`) on the new version -- warning loudly if the tree is dirty.

### Mint a key and invoke it

Read the editor's session token from disk first (it rotates on every server restart). The full API key is shown once, in the mint response's `token` field.

Mint a key -- PowerShell:
```powershell
# payload.json: {"name": "demo"}
$token = Get-Content "$env:LOCALAPPDATA\codefyui\session.token"
curl.exe -s -X POST "http://127.0.0.1:8000/api/keys" `
  -H "X-CodefyUI-Token: $token" -H "Content-Type: application/json" `
  --data "@payload.json"
```

bash:
```bash
TOKEN=$(cat ~/.local/share/codefyui/session.token)   # macOS: ~/Library/Application Support/codefyui/session.token
curl -s -X POST http://127.0.0.1:8000/api/keys \
  -H "X-CodefyUI-Token: $TOKEN" -H "Content-Type: application/json" \
  --data '{"name": "demo"}'
```

`# -> {"id": 1, "name": "demo", "prefix": "cdui_xxxxxxxx", "token": "cdui_..."}`

Invoke it -- LLM calls are slower than most node types, so set `timeout_s` generously (120s below). PowerShell:
```powershell
# payload.json: {"inputs": {"document": "..."}, "timeout_s": 120}
curl.exe -s -X POST "http://127.0.0.1:8000/api/apps/llm-document/invoke" `
  -H "Authorization: Bearer cdui_YOUR_KEY" -H "Content-Type: application/json" `
  --data "@payload.json"
```

bash:
```bash
curl -s -X POST http://127.0.0.1:8000/api/apps/llm-document/invoke \
  -H "Authorization: Bearer cdui_YOUR_KEY" -H "Content-Type: application/json" \
  --data '{"inputs": {"document": "..."}, "timeout_s": 120}'
```

Every resolved invoke writes one row to SQLite: `GET /api/apps/llm-document/runs` lists them newest-first (metadata only), and `GET /api/apps/llm-document/runs/{run_id}` returns the full row, including inputs, outputs, and per-node timings. Both accept either the API key or the session token.

`GET /api/apps/llm-document/versions` lists every published version with `git_commit`, `git_dirty`, and `active`:

```powershell
curl.exe -s "http://127.0.0.1:8000/api/apps/llm-document/versions" -H "X-CodefyUI-Token: $token"
```

```bash
curl -s http://127.0.0.1:8000/api/apps/llm-document/versions \
  -H "X-CodefyUI-Token: $TOKEN"
```

The active version's `GET /api/apps/llm-document/openapi.json` carries the same commit in its `info` block, as `x-codefyui-git-commit` / `x-codefyui-git-dirty` -- so both the versions list and the machine-readable contract trace back to the exact commit you published from.

## Project anatomy

- `graphs/llm-document.graph.json` -- the logic: nodes, edges, params, presets. Positionless, so a parameter edit produces a small, reviewable diff here.
- `layout/llm-document.layout.json` -- canvas positions only, written by the editor; not something you hand-edit.
- `assets/` -- this project's confined data root (`images/`, `models/`, `data/`, and `output/` on demand). Empty here; this graph reads and writes nothing on disk.
- `.env.example` -- committed template of the runtime secret keys this project's nodes might read. `.env` itself is gitignored and never committed.
- `codefyui.project.toml` -- the project manifest: name, plugin pins, and an optional default publish target.

Full model: https://docs.codefyui.com/usage/project-directories

## Make it yours

To use a different model, select the `LLMChat` node on the canvas and edit its `model` param (currently `llama3.2`) to any tag you already have -- check with `ollama list`, or pull a new one first:

```bash
ollama pull <model>
```

Two more params on the same node are just as easy to retarget: `system_prompt` currently pins the task to "Summarize the provided document in 3 concise bullet points" -- rewrite it to translate, extract keywords, classify, or anything else, and the `document` input / `result` output do not need to change. `temperature` (currently `0.2`) trades determinism for variety, and `max_tokens` (currently `512`) caps how long a reply can get.

After editing, save the graph, then re-validate and publish -- canvas edits never affect a published app until you publish again:

```bash
cdui project validate .
cdui project publish . --graph llm-document --slug llm-document
```

This appends version 2 (then 3, ...) to the same `llm-document` slug; the previously published version keeps serving its old snapshot until callers move to the new one.

## Requirements and troubleshooting

Unlike CodefyUI examples that run fully offline out of the box, this one needs a local LLM server reachable before you press Run or invoke it:

1. Install Ollama (https://ollama.com/download) and make sure it is running (`ollama serve`, or just launch the app).
2. Pull the exact model this graph is configured for:
   ```bash
   ollama pull llama3.2
   ```
3. Confirm it answers at the endpoint the `LLMChat` node is configured to call -- `http://127.0.0.1:11434/v1`, its `ollama_base_url` param:
   ```bash
   curl http://127.0.0.1:11434/v1/models
   ```
   The model you pulled should be listed.

No API key is required by default: the `LLMChat` node's `provider` param is `Ollama`, and this graph carries no API-key params at all. (This graph's own description notes that switching `provider` to a hosted option is possible; if you ever do that, `.env.example` reserves `OPENAI_API_KEY` / `CODEFYUI_OPENAI_API_KEY` and `ANTHROPIC_API_KEY` / `CODEFYUI_ANTHROPIC_API_KEY` for LLM keys read at node execute time -- never put a live key in the graph.)

LLM calls are slower than most node types: loading `llama3.2` from disk on a cold start measured about 35 seconds while verifying this example, and warm calls on short documents took a few seconds. Budget for this with a generous `timeout_s` in your invoke payload (120 is a reasonable starting point).

Common failures:

**Ollama is not running, or the model is not pulled.** Invoke still returns the standard envelope, with the failure attributed to the LLM node:

```json
{
  "status": "error",
  "graph": "llm-document",
  "app": "llm-document",
  "outputs": null,
  "error": {
    "code": "execution_error",
    "message": "upstream request failed: All connection attempts failed",
    "node_id": "llm-1",
    "details": null
  }
}
```

Fix: start Ollama and confirm `curl http://127.0.0.1:11434/v1/models` lists `llama3.2`; run `ollama pull llama3.2` if it does not.

**Missing `document` input.** The contract is enforced before anything runs -- HTTP 422:

```json
{"status": "error", "error": {"code": "invalid_input", "message": "invalid inputs", "details": [{"input": "document", "reason": "missing required input"}]}}
```
