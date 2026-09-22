# Hakowan MCP

Provider-neutral agent integration for [Hakowan](https://github.com/Hakowan/hakowan).

`hakowan-mcp` exposes Hakowan's deterministic inspection, schema, validation,
compilation, rendering, observation, camera-fitting, and patch operations over
MCP. Model selection, credentials, conversation state, and reasoning remain the
responsibility of the MCP host.

## Install

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

See [`docs/mcp.md`](docs/mcp.md) for tools, resources, prompts, host
configuration, path confinement, and transport details.

## Token-efficient workflow

The server returns compact schema catalogs, spec handles, validation results,
and visual evidence by default. Agents should start with `inspect_data` and
`get_spec_template`, chain later calls with `spec_id`, and request full schemas,
specifications, or manifests only when needed.

## Evaluation

The installed evaluator targets canonical Hakowan `FigureSpec` JSON:

```sh
hakowan-mcp-eval --provider reference
python -m hakowan_mcp.eval.mcp_harness \
  --model github-copilot/gpt-5-mini \
  --gallery ../hakowan-gallery \
  --output /tmp/hakowan.responses.json
```

See [`docs/evaluation.md`](docs/evaluation.md). Generated responses and reports
are intentionally excluded from Git; CI or release assets should retain them.

## Responsibility boundary

Hakowan owns the schema and deterministic visualization behavior. This package
owns MCP transport, agent instructions, gallery grounding, host execution, and
LLM evaluation. It imports Hakowan's public APIs and does not fork its schema or
validator.

The initial implementation was extracted from Hakowan commit `1012b00`; the
repositories now evolve independently through the version range above.
