# Mapping an MCP server to actions

How a tool described by an MCP server maps to the actions of [actions.wit](actions.wit) and the manifest of [manifest.wit](manifest.wit), what cannot be mapped mechanically, and what an adapter component is.

Sources, read on 16 November 2026: the MCP schema of the latest version, [2026-07-28](https://github.com/modelcontextprotocol/modelcontextprotocol/blob/main/schema/2026-07-28/schema.ts) (`Tool`, `ToolAnnotations`, `CallToolResult`), the [stdio transport](https://github.com/modelcontextprotocol/modelcontextprotocol/blob/main/docs/specification/2026-07-28/basic/transports/stdio.mdx) and the [tools page](https://github.com/modelcontextprotocol/modelcontextprotocol/blob/main/docs/specification/2026-07-28/server/tools.mdx); the example is the research draft `research/drafts/artcraft-suite.md` of the documentation repository (ArtCraft's applications, commits listed there).

## What MCP gives

An MCP server lists tools (`tools/list`) and runs them (`tools/call`). Over stdio, the client starts the server as a process and exchanges newline-delimited JSON-RPC 2.0 messages on its stdin and stdout. A tool has:

- `name`: 1 to 128 characters of `A-Z a-z 0-9 _ - .`, case-sensitive (a SHOULD, not a MUST);
- `title`, `description`: text for people and for the model;
- `inputSchema`: a JSON Schema object with `"type": "object"` at the root, any 2020-12 keyword allowed;
- `outputSchema` (optional): the schema of `structuredContent` in the result;
- `annotations`: `readOnlyHint`, `destructiveHint`, `idempotentHint`, `openWorldHint`. The specification calls them hints and says a client must treat them as untrusted unless the server is trusted.

A result has `content` (text, image, audio, resource blocks), optional `structuredContent`, and `isError`. A server can also offer resources and prompts, and ask the client for sampling (a model call) and elicitation (a question to the user).

## Mechanical mapping

| MCP | DodOS | Rule |
|---|---|---|
| tool `name` | action `name` | Lowercased, `_` and `.` become `-`, cut to `[a-z][a-z0-9-]{0,63}`. Two tools that map to one name: the second is refused and named by hand |
| `title` | action `title` | Cut to 64 bytes |
| `description` | action `description` | Cut to 1024 bytes. Third-party text: `action-info` carries the publisher's provenance |
| `inputSchema` | `input-schema` | Kept if it fits the subset of manifest.wit; `additionalProperties: false` added at every object (the confirmation must show everything sent); a schema outside the subset is refused, not rewritten |
| `outputSchema` | `output-schema` | Kept if it fits the subset; no `additionalProperties: false` added here, since an update may add an output property (manifest.wit). Without one: `{"type": "object", "properties": {"text": {"type": "string"}, "objects": {"type": "array", "items": {"type": "string", "os-item": {"kind": "object", "rights": ["read"], "content-types": []}}}}}` |
| result `structuredContent` | `outcome.output` | When there is an output schema |
| result `content` text blocks | `outcome.output` `{"text": ...}` | Joined in order |
| result image, audio and resource blocks | `outcome.objects` | Each written as an object of `home`, which needs a class that allows a write (`home` is read-only when change is `none`), and returned as a binding named by an `"os-item"` property of `output` (`objects` above), so the agent passes it on without reading it. A tool that returns them is declared `reversible` |
| `isError: true` | error `invalid(<text>)` | The text is third-party: the reply's marks reach the task with it (tasks.invoke) |
| progress notifications | the `progress` stream | `note` is third-party text |
| `notifications/cancelled` | dropping the reading ends | Both ways |

## What cannot be mapped mechanically

1. **The effect class.** The annotations are hints written by the server; the class is declared in the signed manifest by whoever packages the adapter, and that publisher answers for it. The hints only seed a proposal: `readOnlyHint` true gives change `none`; false with `destructiveHint` false gives `reversible`; otherwise `irreversible`; `openWorldHint` false gives reach `local`, otherwise `external`. A human checks each one. A wrong class for reach `local` is harmless (the instance is linked to its class and fails), for reach `external` it is not (see the `OPEN` point of `world service` in core.wit).
2. **Paths.** MCP tools take file paths as strings (`doc_open {path}`, `doc_export {path?}`). DodOS has no global name: a path parameter becomes a storage item (`"os-item"`, with its kind, rights and media types), and the adapter maps it to a file of a preopened directory. Which strings are paths is not in the schema; a human marks them.
3. **Sessions.** An MCP server is one long-lived process that keeps state between calls (open documents, `doc_select {index}`). An action runs in a fresh instance that keeps nothing. A sequence "open, change, save" becomes one action that takes the document as an object, applies its operations and writes a new version. The design of these actions is a human's.
4. **Hosts.** An action of reach `external` must list its hosts in the manifest. The server's code does not say which hosts it calls.
5. **Untyped commands.** A tool whose parameters are free text or any JSON (`command_run {id, params?}` over 500 engine commands whose parameters are a free-text description) gives the confirmation nothing to draw from. Not mapped. Alternative for a publisher who insists: class `irreversible`, so every call is confirmed with its raw JSON.
6. **Driving pixels.** Tools that click, drag, type or take screenshots in a window (`ui_pointer`, `ui_click`, `ui_screenshot`) are what the agent must never do: it acts by typed actions. Not mapped.
7. **Sampling, elicitation, prompts.** Sampling would let a third party run the agent's model on the task's budget with its own text; elicitation would let it ask the user questions outside the system's surfaces; a prompt is third-party text offered as instructions. None is mapped. A question to the user becomes an input of the action; a resource becomes an object.
8. **Remote servers.** Over Streamable HTTP with OAuth, the adapter is an action of reach `external` to the server's host; a token kept between calls lives in `home`, since an action instance keeps nothing else, and the whole adapter (its long-lived instance included) can read it, as any application reads its own credentials. A token typed by the user arrives through a secret field. Streamable HTTP sends every request as a `POST` with a body, which an action of change `none` cannot do (core.wit, `world service`): a remote adapter is declared `reversible`, or waits for the owners' answer on the `OPEN` point of reach `external`.

## The adapter component

An adapter is an ordinary component of the `service` world that exports `actions`, packaged and signed by its publisher. For each `invoke`, it builds the `tools/call` message from `request.input`, maps the bound items to files, passes the message to the server, and maps the result back. Its manifest is drafted by a tool from `tools/list` (the mechanical part above), then completed and signed by a person (classes, items, hosts, refused tools).

How the server runs inside it:

- **Linked in.** When the server's core is a function from one JSON-RPC message to one answer, with no socket and no thread, the adapter links it and calls it directly; stdio disappears. The draft finds this shape in ArtCraft's hand-written servers (`handle_line(&str)` in VectorCraft, LightCraft, DesignCraft and PdfCraft) and in the shared `craft-libs` MCP crate, which leaves the transport to the caller. This needs the server to compile to `wasm32-wasip3`, which none targets today.
- **Not as written.** A server on tokio with the multi-threaded runtime and the `net` feature (PhotoCraft, FilmCraft through `rmcp`) does not compile for WASI, and a gateway that starts each application as a child process (`craft-ai-mcp`) has no equivalent: a component starts no process.
- **Native servers** stay outside DodOS: there is no native application ABI.

Example, PdfCraft: 176 tools, each with an input schema in a 2020-12 subset and `read_only` and `destructive` flags, served by `handle_line`. It is the best case for a first adapter. PhotoCraft, tool by tool:

| Tool | Action | Class | Why |
|---|---|---|---|
| `doc_inspect {index?}` | `inspect {document: item object read}` | none, local | Session index replaced by an object |
| `doc_render_preview {index?, max_side?}` | `render-preview {document, max-side}` returning an image object | none, local | |
| `doc_export {path?, format?, quality?, ...}` | `export {document, folder: item collection write, format, quality}` | reversible, local | Writes a new object; the store keeps versions |
| `command_run {id, params?, wait?}` | not mapped | | Untyped parameters (point 5) |
| `ui_pointer {events[]}` | not mapped | | Drives pixels (point 6) |
