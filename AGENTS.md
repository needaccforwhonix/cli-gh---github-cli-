# AGENTS.md

This is the GitHub CLI (`gh`), a command-line tool for interacting with GitHub. The module path is `github.com/cli/cli/v2`.

## Build, Test, and Lint

```bash
make                                       # Build (Unix) — outputs bin/gh
go run script/build.go                     # Build (Windows)
go test ./...                              # All unit tests
go test ./pkg/cmd/issue/list/... -run TestIssueList_nontty  # Single test
go test -tags acceptance ./acceptance      # Acceptance tests
make lint                                  # golangci-lint (same as CI)
```

**Before committing, ensure both tests and linter pass:**
```bash
go test ./...
make lint
```

## Architecture

Entry point: `cmd/gh/main.go` → `internal/ghcmd.Main()` → `pkg/cmd/root.NewCmdRoot()`.

Key packages:
- `pkg/cmd/<command>/<subcommand>/` — CLI command implementations
- `pkg/cmdutil/` — Factory, error types, flag helpers (`NilStringFlag`, `NilBoolFlag`, `StringEnumFlag`)
- `pkg/iostreams/` — I/O abstraction with TTY detection, color, pager
- `pkg/httpmock/` — HTTP mocking for tests
- `api/` — GitHub API client (GraphQL + REST)
- `internal/featuredetection/` — GitHub.com vs GHES capability detection
- `internal/tableprinter/` — Table output for list commands

## Command Structure

A command `gh foo bar` lives in `pkg/cmd/foo/bar/` with `bar.go`, `bar_test.go`, and optionally `http.go`/`http_test.go`.

### Canonical Examples

- **Command + tests**: `pkg/cmd/issue/list/list.go` and `list_test.go`
- **Factory wiring**: `pkg/cmd/factory/default.go`
- **Unit tests**: `internal/agents/detect_test.go`

### The Options + Factory Pattern

Every command follows this structure (see `pkg/cmd/issue/list/list.go`):

1. `Options` struct with `IO`, `HttpClient`, `Config`, `BaseRepo` + flags
2. `NewCmdFoo(f *cmdutil.Factory, runF func(*FooOptions) error)` constructor — `runF` is the test injection point
3. Separate `fooRun(opts)` function with the business logic

Key rules:
- Lazy-init `BaseRepo`, `Remotes`, `Branch` inside `RunE`, not the constructor
- Commands register in `pkg/cmd/root/root.go`; subcommand groups use `cmdutil.AddGroup()`

### Command Examples and Help Text

Use `heredoc.Doc` for examples with `#` comment lines and `$ ` command prefixes:
```go
Example: heredoc.Doc(`
    # Do the thing
    $ gh foo bar --flag value
`),
```

### JSON Output

Add `--json`, `--jq`, `--template` flags via `cmdutil.AddJSONFlags(cmd, &opts.Exporter, fieldNames)`. In the run function: `if opts.Exporter != nil { return opts.Exporter.Write(opts.IO, data) }`. See `pkg/cmd/pr/list/list.go`.

## Testing

### HTTP Mocking

Use `httpmock.Registry` with `defer reg.Verify(t)` to ensure all stubs are called:

```go
reg := &httpmock.Registry{}
defer reg.Verify(t)

reg.Register(
    httpmock.REST("GET", "repos/OWNER/REPO"),
    httpmock.JSONResponse(someData),
)
reg.Register(
    httpmock.GraphQL(`query PullRequestList\b`),
    httpmock.FileResponse("./fixtures/prList.json"),
)
client := &http.Client{Transport: reg}
```

Common: `REST(method, path)`, `GraphQL(pattern)`, `JSONResponse(body)`, `FileResponse(path)`. See `pkg/httpmock/` for all matchers/responders.

### IOStreams in Tests

```go
ios, stdin, stdout, stderr := iostreams.Test()
ios.SetStdoutTTY(true)  // simulate terminal
```

### Assertions

Use `testify`. Always use `require` (not `assert`) for error checks so the test halts immediately:

```go
require.NoError(t, err)
require.Error(t, err)
assert.Equal(t, "expected", actual)
```

### Generated Mocks

Interfaces use `moq`: `//go:generate moq -rm -out prompter_mock.go . Prompter`. Run `go generate ./...` after interface changes.

### Table-Driven Tests

Use table-driven tests for functions with multiple input/output scenarios. See `internal/agents/detect_test.go` or `pkg/cmd/issue/list/list_test.go` for examples:

```go
tests := []struct {
    name      string
    // inputs and expected outputs
}{
    {name: "descriptive case name", ...},
}
for _, tt := range tests {
    t.Run(tt.name, func(t *testing.T) {
        // arrange, act, assert
    })
}
```

## Code Style

- Add godoc comments to all exported functions, types, and constants
- Avoid unnecessary code comments — only comment when the *why* isn't obvious from the code
- Do not comment just to restate what the code does
- Never use em dashes (—) in code, comments, or documentation; use regular dashes (-) or rewrite the sentence instead

## Error Handling

Error types in `pkg/cmdutil/errors.go`:
- `FlagErrorf(...)` — flag validation (prints usage)
- `cmdutil.SilentError` — exit 1, no message
- `cmdutil.CancelError` — user cancelled
- `cmdutil.PendingError` — outcome pending
- `cmdutil.NoResultsError` — empty results

Use `cmdutil.MutuallyExclusive("message", cond1, cond2)` for mutually exclusive flags.

## Feature Detection

Commands using feature detection must include a `// TODO <cleanupIdentifier>` comment directly above the if-statement for linter compliance:

```go
// TODO someFeatureCleanup
if features.SomeCapability {
    // use new API
} else {
    // fallback for older GHES
}
```

## API Patterns

```go
client := api.NewClientFromHTTP(httpClient)
client.GraphQL(hostname, query, variables, &data)
client.REST(hostname, "GET", "repos/owner/repo", nil, &data)
```

For host resolution, use `cfg.Authentication().DefaultHost()` — not `ghinstance.Default()` which always returns `github.com`.


## 🛡️ Strict Embedding Separation, Zero-Fallback Law & 24/7 Dual-GPU Invariant
- **Reference**: `/home/m1st/.agents/rules/RULE_Strict_Embedding_Separation_And_Dual_Pipeline.md`

### 1. Das Absolute Fallback-Verbot (Zero-Fallback Law)
Unter keinen Umständen, zu keinem Zeitpunkt und aus keinem Grund darf ein Fallback zwischen verschiedenen Embedding-Modellen stattfinden.
* **Geltende Aktion:** Fällt ein Embedding-Modell aus oder ist überlastet, MUSS die Operation sofort hart fehlschlagen (`Fail-Fast`) oder die Payload transaktional in einer Queue (NATS/SQLite) verharren, bis das exakte Modell bereit ist.
* **Verboten:** Kein stiller oder dynamischer Modellwechsel (weder Jina -> Gemma noch umgekehrt).

### 2. Warum ein Embedding-Fallback mathematisch & informationstheoretisch unmöglich ist
* **Topologische Inkompatibilität heterogener Vektorräume (Non-Isomorphism):**
  Jedes Modell $f_\theta: \mathcal{X} \to \mathbb{R}^D$ projiziert Text in eine spezifische, gelernte Riemannsche Mannigfaltigkeit. Jina v5 ($D=256$) und EmbeddingGemma ($D=768$) spannen zwei völlig inkompatible geometrische Räume auf. Die Basisvektoren der semantischen Achsen sind ohne explizite Procrustes-Transformation nicht ausgerichtet.
* **Kollaps der Kosinus-Ähnlichkeit ($	ext{sim} \approx 0$):**
  Wird eine Suchanfrage mit Modell $B$ berechnet ($v_q = f_B(q)$), während der Dokumentenkorpus mit Modell $A$ indiziert wurde ($v_d = f_A(d)$), verhält sich das Skalarprodukt mathematisch wie das zweier rein zufälliger Vektoren auf einer hochdimensionalen Einheitssphäre:
  $$\mathbb{E}[\text{sim}(u, v)] = 0 \quad \text{mit Varianz} \quad \sigma^2 = \frac{1}{D}$$
  Der Nearest-Neighbor-Algorithmus (HNSW/k-NN) liefert stochastisches Rauschen. Das RAG-System erhält völlig falsche oder irrelevante Kontexte.
* **Irreversible Index-Vergiftung (Index Poisoning):**
  Wird auch nur ein einziger Vektor von Modell $B$ als "Fallback" in den Index von Modell $A$ geschrieben, verunreinigt er die Distanzgraphen und Clusterzentren dauerhaft.
* **Das Gesetz des Fail-Fast:**
  Ein Ausfall muss hart abbrechen (`HTTP 503 Service Unavailable / IngestionQueueBlocked`).

### 3. Duale 24/7 Erfassungspflicht (GPU-Only)
* **GPU-Only Mandat:** Es läuft absolut nichts auf der CPU — GPU ONLY (NVIDIA GB10 CUDA) für ausnahmslos jedes Embedding-Modell.
* **24/7 Parallelität:** Sowohl `jina-embeddings-v5-omni-nano-classification` (256D, ~4,1 GB VRAM) als auch `google/embeddinggemma-300m` (768D, ~1,2 GB VRAM) laufen dauerhaft 24/7 im VRAM (Summe ~5,3 GB VRAM).
* **Duale Erfassung:** Jeder zu indizierende Text/Chunk wird immer von beiden Modellen parallel eingebettet und getrennt persistiert.

### 4. Idioten- & Failsafe-Sicherung auf Datenbankebene
* **SQLite Schema CHECK-Constraints:**
  `model_signature TEXT NOT NULL CHECK(model_signature = '...')` und `dimension INTEGER NOT NULL CHECK(dimension = ...)` erzwingen atomare Abbrüche auf Engine-Ebene bei Modell-Mismatches.
* **Qdrant Collection Constraints:**
  Strikte Trennung in separate Collections (`dgx_text_embeddings_jina_256` vs `dgx_text_embeddings_gemma_768`) mit fixierter Vektordimension.


## ⚡ High-Quality Systems Programming Languages Priority (No-Python Policy)
- **Reference**: `/home/m1st/.agents/rules/RULE_High_Quality_Systems_Programming_Languages.md`
- **Rule**:
  1. **Bevorzugte Sprachen:** High Quality **Golang (Go), Rust, C++, Zig, PowerShell, C** sind IMMER und AUSNAHMSLOS die bevorzugten Programmiersprachen.
  2. **Kein Python:** Python ist für neue Daemons, Watcher, Automatisierungen, APIs, CLI-Tools und Dienste strikt untersagt (GIL-Bottlenecks, Dependency-Drift, Speicherineffizienz).
  3. **Natives Systems-Engineering:** Alle Hintergrunddienste, Caching-Ebenen und Task-Runner müssen als native, speichersichere und nebenläufige Binaries kompiliert werden.
