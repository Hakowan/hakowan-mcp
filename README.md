# Hakowan MCP

Provider-neutral agent integration for [Hakowan](https://github.com/Hakowan/hakowan).

`hakowan-mcp` exposes Hakowan's deterministic inspection, schema, validation,
compilation, rendering, observation, camera-fitting, and patch operations over
MCP. Model selection, credentials, conversation state, and reasoning remain the
responsibility of the MCP host.

## Install

Requires Python 3.11+ and Hakowan 0.6.x.

```sh
pip install hakowan-mcp
```

Optional observation support:

```sh
pip install 'hakowan-mcp[observe]'
playwright install chromium
```

## Run

```sh
hakowan-mcp --root /path/to/project
```

Streamable HTTP:

```sh
hakowan-mcp --root /path/to/project \
  --transport streamable-http --host 127.0.0.1 --port 8000
```

Connect clients to `http://127.0.0.1:8000/mcp`. The server has no built-in
authentication, so keep it on loopback unless an authenticated proxy protects
it.

See [`docs/mcp.md`](docs/mcp.md) for tools, resources, prompts, host
configuration, path confinement, and transport details.

## Add to AI harnesses

Use the stdio server with an absolute project root. The root is the security
boundary for every file Hakowan reads or writes.

Claude Code:

```sh
claude mcp add --scope project --transport stdio hakowan -- \
  hakowan-mcp --root /absolute/path/to/project
```

Codex:

```sh
codex mcp add hakowan -- hakowan-mcp --root /absolute/path/to/project
```

Oh My Pi reads the following project-level `.mcp.json` directly. Pi reads it
after `pi install npm:pi-mcp-adapter`; Cursor uses the same content at
`.cursor/mcp.json`:

```json
{
  "mcpServers": {
    "hakowan": {
      "type": "stdio",
      "command": "hakowan-mcp",
      "args": ["--root", "/absolute/path/to/project"]
    }
  }
}
```

For GitHub Copilot in VS Code, place the same server entry under `servers`
instead of `mcpServers` in `.vscode/mcp.json`. In any harness, use its `/mcp`
command or server list to confirm the connection. See [`docs/mcp.md`](docs/mcp.md)
for transports, tools, path confinement, and the recommended agent workflow.

### Token-efficient by design

Hakowan MCP avoids repeatedly sending full schemas, specifications, and
observation manifests through the model context. Agents begin with compact
schema catalogs and focused templates, then chain calls through
content-addressed `spec_id` handles and request detailed payloads only when
needed. This keeps routine visualization workflows substantially smaller than
passing complete `FigureSpec` documents between every tool call.

The optional evaluator for canonical `FigureSpec` JSON is documented in
[`docs/evaluation.md`](docs/evaluation.md).

## Responsibility boundary

Hakowan owns the schema and deterministic visualization behavior. This package
owns MCP transport, agent instructions, gallery grounding, host execution, and
LLM evaluation. It imports Hakowan's public APIs and does not fork its schema or
validator.

The initial implementation was extracted from Hakowan commit `1012b00`. This
release targets the Hakowan 0.6 series and CI verifies compatibility with v0.6.0.

## LLM benchmark

Verify the evaluator with its deterministic reference provider:

```sh
hakowan-mcp-eval --provider reference
```

Expected: `21/21 passed (100.0%)`, plus `llm-eval-report.json` and
`llm-eval-report.html`.

Benchmark a real model through isolated, MCP-only Oh My Pi sessions, then score
the captured responses:

```sh
python -m hakowan_mcp.eval.mcp_harness \
  --model github-copilot/gpt-5-mini \
  --gallery ../hakowan-gallery \
  --output /tmp/hakowan.responses.json \
  --keep-events

hakowan-mcp-eval --provider replay \
  --responses /tmp/hakowan.responses.json \
  --json /tmp/hakowan.report.json \
  --html /tmp/hakowan.report.html
```

The harness prints per-case progress and stores replay data plus optional event
streams. The scorer prints `N/21 passed (P.P%)`; JSON and HTML reports contain
stage and intent scores. Add `--observe` for browser-backed occupancy checks.
See [`docs/evaluation.md`](docs/evaluation.md) for all options.
