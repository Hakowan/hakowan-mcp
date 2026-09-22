# Agent integration with MCP

Hakowan exposes its deterministic visualization workflow through the Model
Context Protocol (MCP). The external host—such as a coding agent or IDE—owns
language understanding, model selection, credentials, and conversation state.
Hakowan owns data inspection, schema validation, compilation, rendering,
observation, and safe patches. The server never calls an LLM provider.

## Install

```sh
pip install "hakowan-mcp"
```

The MCP integration uses the official Python MCP SDK v2.

## Start a local server

For local agent hosts, use the default stdio transport:

```sh
hakowan-mcp --root /path/to/project
```

The equivalent module command is:

```sh
python -m hakowan_mcp --root /path/to/project
```

`--root` is the security boundary for all agent-controlled reads and writes.
Relative paths resolve under it. Absolute paths, `..` traversal, and symlinks
that resolve outside it are rejected.

For a local Streamable HTTP endpoint:

```sh
hakowan-mcp \
  --transport streamable-http \
  --host 127.0.0.1 \
  --port 8000 \
  --root /path/to/project
```

Clients connect to `http://127.0.0.1:8000/mcp`. Keep the default loopback host
unless authentication and transport security are configured outside Hakowan.

## Host configuration

A generic stdio MCP configuration is:

```json
{
  "servers": {
    "hakowan": {
      "type": "stdio",
      "command": "hakowan-mcp",
      "args": ["--root", "/path/to/project"]
    }
  }
}
```

GitHub Copilot in VS Code can place this server entry in `.vscode/mcp.json`.
Other hosts use the same command and arguments under their MCP-server
configuration format. For repository-aware harnesses such as Oh My Pi, register
the command as a project MCP server and set the project directory as `--root`.

## Tools

| Tool | Purpose |
|---|---|
| `get_schema` | Return a compact schema catalog by default, or an explicit fragment/full schema. |
| `get_spec_template` | List or return minimal validated FigureSpec templates; prefer this before schema retrieval. |
| `get_spec` | Resolve a session-local content-addressed specification handle. |
| `inspect_data` | Inspect geometry, topology, attributes, ranges, and non-finite values. |
| `search_gallery` | Optionally retrieve feature-matched canonical gallery recipes. |
| `validate_spec` | Validate schema, resources, semantics, backend support, and compilation. |
| `compile_spec` | Return resolved views, bounds, counts, legends, and annotations. |
| `fit_camera` | Resolve scene geometry to a concrete fitted camera and minimal patch. |
| `render_spec` | Validate and render a specification inside the workspace. |
| `observe_spec` | Capture deterministic views, passes, visibility, and occlusion evidence. |
| `evaluate_visual_patch` | Compare before/after evidence and roll back non-improving patches. |
| `get_backends` | Return declared backend features, passes, and limitations. |
| `apply_patch` | Atomically apply JSON Pointer patches and revalidate. |

The server also exposes:

- `hakowan://schema` — canonical JSON Schema resource;
- `hakowan://agent-instructions` — the recommended safe workflow;
- `author_figure` — a reusable prompt containing that workflow.

## Recommended agent loop

```text
inspect_data
    ↓
get_spec_template for the closest intent
    ↓
get_schema(fragment=...) only when a field remains unclear
    ↓
author canonical FigureSpec JSON
    ↓
validate_spec(strict=true) → retain spec_id
    ↓
apply_patch(spec_id, ...) for repairs → retain the new spec_id
    ↓
fit_camera(spec_id, ...) when framing matters
    ↓
render_spec(spec_id, ...)
    ↓
observe_spec(spec_id, ...) only when visual evidence matters
    ↓
evaluate_visual_patch for bounded visual repairs
```

This order keeps normal tasks on compact templates and content-addressed handles.
Do not request the complete schema for a routine visualization.

The agent should never invent attributes. Schema failures include stable codes,
JSON Pointer paths, and safe removal patches when unambiguous. `apply_patch`
operates atomically on raw JSON, so it can repair a document that does not yet
pass the `FigureSpec` schema. Semantic validation failures retain stable codes,
paths, messages, and hints.

## Visual evidence and bounded repair

`observe_spec` uses a declared Figure camera by default. If the specification
has no camera, it captures the front, right, top, and isometric presets. Its
default response contains compact `evidence`, `visual_diagnostics`, and a
workspace-relative `manifest_path`. Set `include_manifest=true` only when the
complete artifact manifest must be returned inline; it is always written to the
output directory. Evidence covers occupancy, projected and visible bounds,
vertex clipping, per-layer screen and element visibility, estimated occlusion,
depth order, visible attribute ranges, legend presence, and contrast.

`visual_criteria` may override `min_occupancy`, `max_occupancy`,
`max_clipped_fraction`, and `min_contrast`. Defaults are 0.02, 0.95, 0.05, and
0.08 respectively.

`evaluate_visual_patch` applies at most three operations by default, validates
the candidate, captures before/after evidence under identical settings, and
accepts only a measurable change with no metric regression. A rejected
candidate returns `rolled_back=true` and the original canonical spec. Both
observation bundles remain available for inspection.

## Specification terminology and composition

`LayerPropertiesSpec.data` is always the geometry source: either a workspace
mesh file or an external ID resolved through `data_bindings`. A `data` field
inside a channel, texture, or transform is an attribute reference within that
geometry source. The generated schema exposes this distinction through the
`x-hakowan-role` annotation, and focused schema fragments preserve it.

Each key under `channels` is one semantic visual slot. Fluent `.channel()` calls
are immutable by default and wrap the previous layer. Compilation visits nodes
from root to leaf, so the outermost value for a slot wins while unrelated slots
compose. With `in_place=true`, channels append to the same node in argument
order and the first value for a duplicate slot remains effective.

`Figure` contains portable visualization intent: layer composition, camera,
lighting, environment, and semantic output settings. `Config` is invocation
policy for a concrete render. Passing an explicit `Config` replaces the complete
Figure-derived renderer configuration rather than partially merging with it.

## Specification handles and compact responses

Successful validation returns a session-local, content-addressed `spec_id` such
as `sha256:...`. The `spec` argument accepted by validation, patch, camera,
compile, render, observation, and visual-patch tools may be either full JSON or
a `spec_id`. Chain calls with the handle rather than resending canonical JSON.

`validate_spec`, `fit_camera`, and `apply_patch` omit full canonical JSON by
default. Set `include_spec=true` only when a caller genuinely needs it inline;
`get_spec(spec_id)` remains available explicitly. Handles are deduplicated by
canonical content and retained in a 128-entry session-local LRU store.

## Focused schemas and templates

Call `get_schema()` without arguments for a compact catalog. Pass a fragment
such as `transform.clip`, `texture.scalar_field`, `channel.vector_field`, or
`scene.camera.orthographic` for one definition. Compact fragments list unresolved
definition names instead of embedding them; set `include_dependencies=true`
only when a self-contained JSON Schema fragment is required. Use
`get_schema(fragment="full")` or the `hakowan://schema` resource only as a last
resort.

`get_spec_template()` lists minimal canonical patterns for surface and point
fields, vector glyphs, wireframe overlays, clipping planes, and layouts.

## Data references

Mesh-file specifications may use paths relative to `--root`. In-memory-style
external IDs use explicit workspace bindings:

```json
{
  "spec": {
    "$schema": "https://hakowan.github.io/hakowan/schema/v1.json",
    "version": "1.0",
    "root": {
      "kind": "layer",
      "spec": {
        "data": {"kind": "external", "id": "input"}
      }
    }
  },
  "data_bindings": {
    "input": "data/simulation.ply"
  }
}
```

Arbitrary Python function references are intentionally unsupported through MCP.
Use Hakowan's restricted expression specification for serializable filters and
custom scales.

## Optional gallery discovery

The gallery is not required. Data inspection, schema retrieval, validation,
compilation, rendering, observation, and patching work without it.
`search_gallery` prefers recipes from a local checkout, then falls back to the
published static corpus at
`https://hakowan.github.io/hakowan-gallery/agent/v1/index.json`. If neither is
available, it returns `ok=true`, `available=false`, and an empty `matches` list
so agents can continue normally.

Local checkouts are discovered in this order:

1. the optional `--gallery` argument;
2. `HAKOWAN_GALLERY`;
3. a sibling `hakowan-gallery` checkout beside the workspace.

Set `HAKOWAN_GALLERY_URL` to use a different static corpus URL, or to an empty
string to disable remote retrieval. The server caches remote JSON in memory,
revalidates the index with `ETag` or `Last-Modified`, and verifies recipe
artifacts against the SHA-256 hashes in the index. Network and corpus errors are
non-fatal. Private data inspection, compilation, rendering, and observation
always remain local.

Set `include_spec=true` only when the full canonical example is needed; compact
metadata is cheaper for discovery.

## Development and inspection

Run with the official MCP Inspector:

```sh
mcp dev src/hakowan_mcp/app.py --with-editable .
```

Or exercise the installed server from any MCP v2 client using the
`hakowan-mcp` command. The default stdio transport reserves stdout for protocol
messages; Hakowan logs are emitted through Python logging rather than prints.
