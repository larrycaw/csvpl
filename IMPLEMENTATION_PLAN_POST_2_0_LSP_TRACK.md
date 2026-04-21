# CSVPL Post-2.0 — Language Server (LSP) Track

This document is a follow-on to the Option C platform refresh. It proposes a
new track — shipped *after* `2.0.0` GA — that gives `.csvpl` scripts a
Language Server Protocol implementation so editors (VS Code, IntelliJ
2024.2+, Neovim, Zed, Helix) offer first-class editing.

The track is shaped by IntelliJ's LSP client floor, because IntelliJ gates
rich features on specific platform versions:

| IntelliJ version | LSP capability unlocked |
|---|---|
| 2024.2+ | `rename`, `codeAction` |
| 2024.3+ | `inlayHint` |
| 2025.1+ | `semanticTokens` |

The server must advertise each capability correctly in `initialize`, and the
IntelliJ plugin bundle must declare a `since-build` of `242` (2024.2) with
no hard `until-build` cap so users on 2025.1+ receive the full set.

## Prerequisites from the 2.0 program

- **PR 7 — Explicit extension registry.** The LSP server introspects
  commands and processing elements from `CommandRegistry` /
  `ProcessingElementRegistry` rather than reflection. Without this PR, the
  server cannot enumerate built-ins cleanly.
- **PR 8 — CLI/API normalization & error model.** Parse/validation errors
  must carry `file:line:col` spans; the LSP server turns those directly
  into `Diagnostic`s. Required for PR L3 below.
- **PR 9 — Streaming pipeline.** Not a hard prerequisite, but hover and
  inlay-hint content (row-count estimates, inferred column types) is much
  better when the pipeline is introspectable.

## Architecture

- New Maven module `csvpl-lsp` inside the reactor, depending on
  `csvpl-api` and `csvpl-core`. The runtime artifact is a self-contained
  executable jar plus a thin wrapper script.
- Uses [LSP4J](https://github.com/eclipse-lsp4j/lsp4j) — mature, widely
  used, matches what IntelliJ's LSP client expects on the wire.
- **Reuses the core parser.** PR L1 refactors the existing line-oriented
  dispatcher into a lossless AST (preserves spans, whitespace, comments).
  No second grammar.
- Stdio transport by default; optional TCP for debugging.
- Distributed via Maven Central (`ai.preferred:csvpl-lsp`) and as a
  GitHub release asset. IntelliJ and VS Code extensions bundle a pinned
  version.

## PR sequence

### PR L1 — Lossless parser + AST in core
Refactor the script parser to emit a full AST with source spans and trivia
(whitespace, comments) retained. The current line-by-line dispatch is
insufficient for rename and formatting. No behavior change for the CLI —
the existing executor consumes the new AST.
- **Depends on:** 2.0 PR 7, PR 8.
- **Risk:** Medium — touches core parsing. Mitigated by the golden-output
  suite from 2.0 PR 10.

### PR L2 — `csvpl-lsp` module skeleton
Add the Maven module, wire LSP4J, implement the `initialize` /
`initialized` / `shutdown` handshake. Advertise only document-sync
capabilities at first. Stdio transport. Logs go to stderr; protocol
traffic to stdout.
- **Depends on:** L1.
- **Acceptance:** `csvpl-lsp` launches, a minimal LSP4J test client
  completes the handshake.

### PR L3 — Diagnostics & completion
- `textDocument/publishDiagnostics` driven by the parser + registry
  (unknown command, wrong arity, unknown option, unknown column
  reference).
- `textDocument/completion` for command names, PE names, option keys,
  and column names (inferred from CSV headers when a `ReadCsv`-style
  command resolves to a concrete file).
- **Depends on:** L2.

### PR L4 — Hover, go-to-definition, symbols
- `textDocument/hover`: pull command/PE help text from `@Help`
  annotations introduced in 2.0 PR 8.
- `textDocument/definition`: variables and file-path references.
- `textDocument/documentSymbol`: outline (commands as top-level, PEs as
  nested).
- **Depends on:** L3.

### PR L5 — Rename + code actions  *(unlocks IntelliJ 2024.2+)*
- `textDocument/prepareRename` + `textDocument/rename` for user-defined
  names (variables, sub-script labels). Built-in command/PE names are
  not rename targets.
- `textDocument/codeAction`:
  - Quick-fixes: "Unknown column 'foo' — did you mean 'Foo'?", "Add
    missing required argument", "Replace deprecated PE name with its
    successor".
  - Refactors: extract-to-subscript, inline variable.
- **Depends on:** L1 (spans), L4 (symbol table).

### PR L6 — Inlay hints  *(unlocks IntelliJ 2024.3+)*
- `textDocument/inlayHint`: inferred column types next to expressions,
  resolved file paths at `ReadCsv`, row-count estimates when cheap to
  compute.
- Per-category opt-in via `workspace/configuration`
  (`csvpl.inlayHints.types`, `csvpl.inlayHints.rowCounts`, ...).
- **Depends on:** L4 (type inference scaffolding).

### PR L7 — Semantic tokens  *(unlocks IntelliJ 2025.1+)*
- `textDocument/semanticTokens/full`, `/range`, `/full/delta`.
- Token types: `keyword` (commands), `function` (PEs), `variable`,
  `parameter`, `string`, `number`, `comment`, `property` (option keys),
  `class` (column identifiers).
- Modifiers: `deprecated` (for registry entries marked so),
  `defaultLibrary` (built-ins), `readonly`.
- **Depends on:** L1 (AST tokens with spans).
- **Perf note:** cache token arrays per document version; serve `/delta`
  to avoid re-tokenizing on every keystroke.

### PR L8 — Formatting, folding, signature help
- `textDocument/formatting` + `rangeFormatting` (config: indent width,
  option alignment).
- `textDocument/foldingRange` for scripts and nested blocks.
- `textDocument/signatureHelp` for PE arguments.
- **Depends on:** L1.

### PR L9 — IntelliJ plugin
- Thin plugin declaring a `.csvpl` file type and wiring
  `LspServerSupportProvider` to launch `csvpl-lsp`.
- `plugin.xml`: `since-build=242`, no `until-build` cap.
- Ships with a bundled server jar; also supports a "use local `csvpl-lsp`"
  setting for dogfooding.
- Publish to JetBrains Marketplace.
- **Depends on:** L7 (so launch-day users get the full capability set on
  2025.1).

### PR L10 — VS Code & Neovim clients
- VS Code extension that spawns `csvpl-lsp` over stdio. TextMate grammar
  as a client-side fallback for when the server is starting.
- Neovim snippet for `nvim-lspconfig` in `docs/lsp.md`.
- Zed extension (optional — Zed's extension API is LSP-first so the cost
  is low).
- **Depends on:** L7.

### PR L11 — Test harness
- Golden-output tests per LSP method against fixture scripts under
  `src/test/resources/lsp/`.
- Integration test that spawns `csvpl-lsp` as a subprocess and drives it
  via an LSP4J client.
- IntelliJ plugin smoke test via `runIdeForUiTests`.
- **Depends on:** L2 onward (added incrementally but formalized here).

### PR L12 — Docs & release
- `docs/lsp.md` with a capability matrix, per-editor setup, and
  screenshots.
- Marketplace listings (JetBrains, VS Code).
- `csvpl-lsp` published to Maven Central alongside the rest of 2.x;
  version pinned inside the editor extensions.
- **Depends on:** L9, L10.

## Advertised `ServerCapabilities` matrix

| Capability | Delivered in | Surfaces in IntelliJ on |
|---|---|---|
| `textDocumentSync` | L2 | any |
| `diagnosticProvider` | L3 | any |
| `completionProvider` | L3 | any |
| `hoverProvider` | L4 | any |
| `definitionProvider` | L4 | any |
| `documentSymbolProvider` | L4 | any |
| `renameProvider` (+ `prepareProvider`) | L5 | 2024.2+ |
| `codeActionProvider` | L5 | 2024.2+ |
| `inlayHintProvider` | L6 | 2024.3+ |
| `semanticTokensProvider` (full/range/delta) | L7 | 2025.1+ |
| `documentFormattingProvider` | L8 | any |
| `foldingRangeProvider` | L8 | any |
| `signatureHelpProvider` | L8 | any |

## Non-goals

- No debug-adapter (DAP) implementation.
- No live CSV preview over LSP — that belongs in editor-specific surfaces
  (VS Code webview, IntelliJ tool window) and is out of scope here.
- No new parser. PR L1 is a refactor of the existing one, not a rewrite.
- Backwards compatibility with pre-2.0 CSVPL is not a goal; the LSP
  targets 2.x scripts only.

## Risks & mitigations

- **IntelliJ LSP API gaps.** The IntelliJ LSP client does not cover the
  full LSP spec (e.g., server-initiated commands from code actions have
  caveats). *Mitigation:* document gaps in `docs/lsp.md`; fall back to
  text-edit-only code actions where commands are not surfaced.
- **Semantic-tokens perf on large scripts.** *Mitigation:* cache per
  document version; serve `semanticTokens/full/delta` after the first
  request; measure in PR L11.
- **Version skew between server and editor extensions.** *Mitigation:*
  extensions pin a specific `csvpl-lsp` version and declare the
  minimum `ServerCapabilities` they require; the server refuses
  mismatched clients with a clear error.
- **Parser refactor regressions.** *Mitigation:* PR L1 gates on the
  golden-output suite added in 2.0 PR 10; ship behind a
  `csvpl.parser=legacy|ast` flag for one release if needed.

## Release plan

- `2.1.0` — PR L1 lands (parser refactor only, no LSP exposure).
- `2.2.0` — PR L2 + L3 + L4 (usable LSP with diagnostics/completion/hover).
- `2.3.0` — PR L5 + L6 (IntelliJ 2024.2 & 2024.3 capabilities live).
- `2.4.0` — PR L7 + L8 (IntelliJ 2025.1 capability live; formatting).
- `2.5.0` — PR L9 + L10 + L11 + L12 (editor extensions + docs +
  marketplace).
