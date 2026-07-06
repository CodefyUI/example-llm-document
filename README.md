# example-llm-document

A working LLM microservice in four nodes:

    Start --(trigger)--> GraphInput "document" --> LLMChat (Ollama, llama3.2) --> GraphOutput "result"

POST `{"inputs": {"document": "<text>"}}` and get `{"outputs": {"result": "<3-bullet summary>"}}`
back. It runs against a local Ollama server by default, so the graph contains no API
keys and works fully on your own machine, air-gapped after the one-time model pull.
The instruction lives in the `LLMChat` node's `system_prompt` param ("You are a
document processing service. Summarize the provided document in 3 concise bullet
points."), so the user message is purely your document -- a clean instruction/data
split you can retarget by editing that one param (translate, extract keywords,
classify, ...). Because the contract is declared with `GraphInput`/`GraphOutput`
nodes, this graph can be published as a versioned HTTP API with durable API keys
and per-run records -- that is the point of this example.

This is a [CodefyUI](https://docs.codefyui.com) **project directory**: a
self-contained git repo where the graph files ARE the storage (see
[Project Directories](https://docs.codefyui.com/usage/project-directories)).

## Prerequisites

Requires CodefyUI (`cdui`) installed -- see the
[installation guide](https://docs.codefyui.com/getting-started/installation).

Then pick ONE of the two provider setups. The graph ships pointing at Ollama.

### Option A (default): local Ollama, no API key at all

1. Install Ollama from https://ollama.com/download and make sure it is running.
2. Pull the model the graph uses (about 2 GB):

   ```bash
   ollama pull llama3.2
   ```

That is everything. The `LLMChat` node talks to Ollama's OpenAI-compatible endpoint
at `http://127.0.0.1:11434/v1` (the node's `ollama_base_url` param).

### Option B: hosted provider (ChatGPT API or Claude API)

1. Select the `LLMChat` node on the canvas and switch its `provider` dropdown to
   `ChatGPT API` or `Claude API`, and set `model` accordingly.
2. Give the CodefyUI **server process** the provider key via an environment
   variable, then start the server. Exact names the node reads:
   - ChatGPT API: `CODEFYUI_OPENAI_API_KEY` or `OPENAI_API_KEY`
   - Claude API: `CODEFYUI_ANTHROPIC_API_KEY` or `ANTHROPIC_API_KEY`

   PowerShell:

   ```powershell
   $env:CODEFYUI_OPENAI_API_KEY = "sk-..."
   cdui start --project .
   ```

   bash:

   ```bash
   export CODEFYUI_OPENAI_API_KEY="sk-..."
   cdui start --project .
   ```

Leave the node's `openai_api_key` / `anthropic_api_key` params empty. **The
security story:** the provider key lives only in the server's environment -- it
is never written into the saved graph, never leaves the server, and never
reaches your users. API callers only ever hold the `cdui_` app key you mint
below, which you can revoke per caller at any time while your provider key
stays private.

## Quick start

```bash
git clone https://github.com/CodefyUI/example-llm-document
cd example-llm-document
cdui start --project .
```

Open the printed editor URL, load the `llm-document` graph, and press Run. The
`document` input has a built-in default text, so the canvas run works without
typing anything; the summary appears on the LLMChat/GraphOutput nodes. First
run note: if Ollama has to load the model from disk, the first call can take
tens of seconds on CPU (about 35 s measured on a cold start); warm calls take a
few seconds for short documents.

## Publish it as an HTTP API

Commit any changes first (an uncommitted manifest is exactly the kind of dirty
tree `cdui project publish` warns about), then:

```bash
cdui project publish . --graph llm-document --slug llm-document
```

Management calls (publish, mint keys) use the editor session token from
`<user_data_dir>/codefyui/session.token` (Windows:
`%LOCALAPPDATA%\codefyui\session.token`; Linux:
`~/.local/share/codefyui/session.token`; macOS: `~/Library/Application
Support/codefyui/session.token`). It rotates on every server restart, so
scripts should re-read the file. Windows note: in PowerShell, `curl` is an
alias for `Invoke-WebRequest`; always type `curl.exe`.

### Mint an API key (shown once -- copy it now)

PowerShell:

```powershell
$token = Get-Content "$env:LOCALAPPDATA\codefyui\session.token"
'{"name": "my-first-key"}' | Set-Content -Encoding ascii key.json
curl.exe -s -X POST "http://127.0.0.1:8000/api/keys" `
  -H "X-CodefyUI-Token: $token" -H "Content-Type: application/json" `
  --data "@key.json"
```

bash:

```bash
token=$(cat ~/.local/share/codefyui/session.token)
printf '%s' '{"name": "my-first-key"}' > key.json
curl -s -X POST "http://127.0.0.1:8000/api/keys" \
  -H "X-CodefyUI-Token: $token" -H "Content-Type: application/json" \
  --data "@key.json"
```

The response contains the full key exactly once, in `token`:
`{"id": 1, "name": "my-first-key", "prefix": "cdui_XXXXXXX", "token": "cdui_XXXXXXX_..."}`.

### Invoke it

Write `payload.json`:

```json
{
  "inputs": {
    "document": "CodefyUI is a visual programming environment for building and understanding AI systems. Users assemble graphs from nodes that represent data sources, tensor operations, model layers, and training loops, then execute them directly on the canvas."
  },
  "timeout_s": 120
}
```

PowerShell:

```powershell
curl.exe -s -X POST "http://127.0.0.1:8000/api/apps/llm-document/invoke" `
  -H "Authorization: Bearer cdui_YOUR_KEY" -H "Content-Type: application/json" `
  --data "@payload.json"
```

bash:

```bash
curl -s -X POST "http://127.0.0.1:8000/api/apps/llm-document/invoke" \
  -H "Authorization: Bearer cdui_YOUR_KEY" -H "Content-Type: application/json" \
  --data "@payload.json"
```

Verified response (a real run against the graph before this transplant; summary
trimmed):

```json
{
  "status": "ok",
  "run_id": "8df8d6284cde4f7aa7eb2b1da9b63082",
  "graph": "llm-document",
  "app": "llm-document",
  "version": 1,
  "device": "cpu",
  "outputs": {
    "result": "Here are 3 concise bullet points summarizing the provided document: ..."
  },
  "error": null,
  "timing": { "total_s": 34.718 }
}
```

Re-publishing after canvas edits appends version 2, 3, ... to the same slug;
canvas edits never change the published app until you re-publish. Every invoke
is recorded: `GET /api/apps/llm-document/runs` (list) and
`GET /api/apps/llm-document/runs/{run_id}` (full inputs/outputs/timings).
`GET /api/apps/llm-document/openapi.json` serves the machine-readable contract.

## Troubleshooting

**Ollama is not running.** Invoke still returns the standard envelope, with the
failure attributed to the LLM node (`status` stays a top-level field so
callers can branch on it):

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

Fix: start Ollama (launch the app, or run `ollama serve`) and confirm
`curl http://127.0.0.1:11434/v1/models` lists your model; `ollama pull llama3.2`
if it is missing.

**Missing `document` input.** The contract is enforced before anything runs --
HTTP 422:

```json
{"status": "error", "error": {"code": "invalid_input", "message": "invalid inputs", "details": [{"input": "document", "reason": "missing required input"}]}}
```

**Changing the model.** Select the LLMChat node and edit the `model` param to
any tag from `ollama list` (for example `llama3.2:1b` for a smaller/faster
model, or `qwen3`). The `ollama_base_url` param accepts any OpenAI-compatible
endpoint, so a remote Ollama box or another local OpenAI-compatible server
works too. After editing, File > Save and re-publish to ship the change as the
next version.

## Continuous integration

```bash
cdui project validate .
```

runs the full publish pre-flight (secret check, contract, entry points,
wiring, node validity) on every graph in `graphs/`. See
[Project Directories](https://docs.codefyui.com/usage/project-directories) for
the complete model (the logic/layout split, assets, `.env` handling, and
`cdui project restore`/`freeze` for plugin pins).
