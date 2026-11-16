# Contracts

The interface files of this folder are under the Apache License 2.0 ([LICENSE](LICENSE)), unlike the rest of the repository, which is under AGPLv3: an application that uses them takes no obligation from the AGPL.

## Files

| File | Package | Content |
|---|---|---|
| [core.wit](core.wit) | `os:core@0.1.0` | Shared types, size caps, value encoding, background wake-up, the worlds and their linking rules |
| [store.wit](store.wit) | `os:core` | Storage, kept handles (`keep`), object exchange (`exchange`) |
| [ui.wit](ui.wit) | `os:core` | Surfaces and the open interface tree |
| [input.wit](input.wit) | `os:core` | The text input protocol: fields drawn by applications, input engines |
| [actions.wit](actions.wit) | `os:core` | Actions (application side), tasks (agent side) |
| [agent.wit](agent.wit) | `os:core` | Inference, memory, audio, the agent's entity, reading another surface's accessibility tree |
| [catalog.wit](catalog.wit) | `os:catalog-agent@0.1.0`, `os:catalog-app@0.1.0` | The two catalogue levels (types only) |
| [manifest.wit](manifest.wit) | `os:manifest@0.1.0` | The signed package manifest: actions, schemas, classes, hosts, rights (types only) |
| [shell.wit](shell.wit) | `os:shell@0.1.0` | Window and surface management, as the inputs and outputs of the shell's actions (types only) |
| [mcp.md](mcp.md) | | How a tool of an MCP server maps to actions, and what an adapter component is |
| [deps/](deps) | `wasi:*@0.3.2` | WASI packages the worlds import, vendored |

Unsettled points are marked `OPEN:` in the files, each with its alternatives and what decides it.

## Worlds and what they may import

| World | For | Imports | Exports |
|---|---|---|---|
| `base` | Included by all | `wasi:cli` environment, exit, stdio, terminal; `wasi:clocks` monotonic and system; `wasi:random` | |
| `service` | A component without interface | `base`, `wasi:filesystem` (preopens `/home`, `/tmp`), `wasi:sockets`, `wasi:http` client, `keep`, `background` | `actions`, `start`, `suspend`, `resume`, `wake` |
| `app` | A component that draws with the catalogue | `service`, `exchange`, `text-input`, the two catalogue levels | `show` |
| `agent` | The agent | `base`, `inference`, `memory`, `audio`, `entity`, `accessibility`, the agent catalogue level | `handle` |
| `wake-word` | The wake-word detector | `base` (its output goes nowhere) | `detect` |
| `input-engine` | An input method | `base` | `engine` |

`base` is what the Rust standard library of `wasm32-wasip3` links into an empty component (checked with Rust 1.100), plus randomness. The installer refuses a component with an import outside its world; an import the manifest does not grant is linked to an implementation that refuses every call. Imports match by semver within one WASI series, as Wasmtime's linker does.

## WASI

`os:core` uses WASI 0.3, released on 11 June 2026; the current release is 0.3.2 (13 October 2026), and 0.3.3 is scheduled for 8 December 2026 ([release schedule](https://github.com/WebAssembly/WASI/blob/main/docs/Release.md)). WASI 0.3 rests on the Component Model's async (`async` functions, `stream`, `future`); 0.3.1 adopted `map`, and the `implements` and `external-id` annotations on interface names. Nothing else is adopted ([adopted features](https://github.com/WebAssembly/WASI/blob/main/docs/ComponentModelFeatures.md)), and these files use nothing else: today every world needs only async.

Threads: WASI 0.3 gives concurrency inside one instance (async tasks interleaved at waits) and parallelism only between instances. The Component Model's threads (🧵, `thread.new-indirect` and the other `thread.*` built-ins) are specified but gated and not adopted by any WASI release; without the core proposal shared-everything-threads (phase 1) they are cooperative only, one instance running one thread at a time ([Concurrency.md](https://github.com/WebAssembly/component-model/blob/main/design/mvp/Concurrency.md), [gated features](https://github.com/WebAssembly/component-model/blob/main/design/mvp/Text.md#gated-features), [proposals](https://github.com/WebAssembly/proposals)). Wasmtime implements the threading built-ins behind the flag `component-model-threading`.

`deps/` holds the six packages of the [WASI 0.3.2 release tarball](https://github.com/WebAssembly/WASI/releases/tag/v0.3.2) the worlds need (cli, clocks, filesystem, http, random, sockets), placed by [wit-deps](https://github.com/bytecodealliance/wit-deps) 0.6 from [deps.toml](deps.toml), locked in [deps.lock](deps.lock). To refresh:

    wit-deps -m contracts/deps.toml -l contracts/deps.lock -d contracts/deps lock

`verify` checks that the WIT resolves; that every world encodes as a valid component, validated with only the Component Model features that world uses, as listed in `verify` (a world missing from the list, or a feature WASI has not adopted, fails); that every right `manifest.wit` names is served by an import of a world or marked reserved; and that `deps/` is byte for byte the tarball `deps.toml` pins (it downloads it once into `target/`).
