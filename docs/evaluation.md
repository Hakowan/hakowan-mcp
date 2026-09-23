# Hakowan LLM evaluation

This suite evaluates natural-language-to-`FigureSpec` systems by observable visualization behavior rather than textual similarity.

## Stages

Every candidate is scored independently for:

1. JSON Schema validity
2. runtime reconstruction and strict semantic validation
3. compilation
4. backend rendering
5. attribute grounding
6. grammar intent
7. camera framing
8. bounded diagnostic repair
9. repair patch minimality

The bundled suite contains 21 requests over five deterministic datasets,
including missing fields, non-positive log input, NaNs, constant fields,
extreme aspect ratios, backend limitations, point clouds, disconnected
components, comparisons, occlusion, and default artifact selection for a
simple visualization request.

## Deterministic run

```sh
python -m hakowan_mcp.eval --provider reference
```

This exercises the evaluator with controlled faulty first attempts and one repair. Reports are written to `llm-eval-report.json` and `llm-eval-report.html`.

Add browser-backed occupancy scoring:

```sh
python -m hakowan_mcp.eval --provider reference --observe
```

## Replay real responses

Store provider output as:

```json
{
  "model": "provider/model-id",
  "responses": {
    "scalar-temperature": [
      {"kind": "spec", "value": {"$schema": "...", "version": "1.1", "root": {}}, "metadata": {"latency_ms": 420, "input_tokens": 1200, "output_tokens": 350}}
    ]
  }
}
```

Then run:

```sh
python -m hakowan_mcp.eval --provider replay --responses responses.json
```

Each case may contain a second response with `kind: "patch"` for bounded repair.

## Live provider plugin

Pass `--provider package.module:callable`. The callable receives JSON-safe `case`, `context`, `attempt`, `candidate`, and `diagnostics` keyword arguments. It returns a complete spec mapping, JSON text, or `{ "kind": "patch", "value": [...] }`.

The context contains `hkw.schema()`, `hkw.inspect()` output for the case dataset, and up to three feature-matched canonical examples loaded from `HAKOWAN_GALLERY` or a sibling `hakowan-gallery` checkout.

Live runs are explicit and never part of ordinary CI.

Provider `raw` responses and arbitrary `metadata` (for example model ID, token
usage, latency, temperature, and cache statistics) are preserved in each case's
attempt trace.

## Strict MCP host benchmark

Run the suite through fresh Oh My Pi sessions with an enforced MCP-only tool
policy:

```sh
python -m hakowan_mcp.eval.mcp_harness \
  --model anthropic/claude-sonnet-4-6 \
  --gallery ../hakowan-gallery \
  --output /tmp/sonnet.responses.json \
  --timeout 300 \
  --max-tool-calls 32 \
  --keep-events
```

The harness creates deterministic datasets and an isolated MCP configuration in
a temporary workspace. Each case runs in a fresh host process. Wall-clock and
tool-call budgets are enforced; timeouts, nonzero exits, missing required tools,
non-MCP calls, provider retries, token usage, and reported cost are retained in
the response metadata.

Output-envelope compliance is reported independently from FigureSpec behavior:
`direct_json` passes the format contract, `normalized_envelope` records JSON
recovered deterministically from Markdown or surrounding prose, and `unusable`
records output with no complete response object. Normalized specifications still
receive ordinary schema, semantic, compile, render, intent, and camera scores.

The `default-visualization-output` case asks only “Visualize the model.” The
strict MCP harness does not name an output format; it records whether the agent
calls `observe_spec` for an image, `render_spec` for HTML, both, or neither.
`run_metadata.artifact_output_cases` contains the expected and actual choice,
called output paths, and pass status. Image-only passes; HTML-only, both, and
missing output fail this focused policy check without masking FigureSpec scores.

## Generated output

Benchmark responses, event streams, JSON reports, HTML reports, screenshots,
and other run artifacts are intentionally not versioned. Write them to a local
or CI artifact directory and publish durable records as release assets when
needed. Keep the model and host identifiers, Hakowan and `hakowan-mcp` commits,
case-set and gallery digests, harness limits, and artifact checksums with each
archived run so it remains reproducible.
