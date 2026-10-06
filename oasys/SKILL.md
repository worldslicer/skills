---
name: oasys
description: >
  Write, check, run and benchmark OASys devices: programs in the `.os` DSL where the program owns
  the control flow and an LLM is one instruction (an AI cell). Use when the user mentions OASys, a
  `.os` file or device, books/pages/lines/cells, AI cells or `agent {}` blocks, the `oasys` CLI,
  or asks to create, import, push, clone, run, test or benchmark a device (`@benchmark`,
  `oasys bench`).
metadata:
  product: OASys (Operative Agentic System)
  cli: oasys
---

# OASys

**Deterministic agentic programs. The program owns the loop; the model is one instruction.**
A **Device** is a program written in the `.os` DSL. A Rust engine runs it with a program counter,
call frames and a shared VariableStore. An LLM appears only inside an **AI cell**: one bounded call
that returns a value to the store. It never decides what runs next. The visual grid in the app and
the `.os` text are the same model and round-trip without loss.

Hierarchy: `Device → Books → Pages → Lines → Cells`.

| Use | Never say |
|---|---|
| Device | project, workspace, script (a *Book* is a module inside a Device) |
| Page / Line / Cell | thread, worker / row, routine / node, step, block |
| VariableStore | state, memory |
| `.os` source | script, code file |
| Board (UI group of devices) | "project" in user-facing text |

- **Book**: program unit. Only the entry book `main` runs; other books are libraries or data.
- **Page**: unit of parallelism. Pages of `main` start together. A page or line named `_…` is a
  definition: it does not run in the flow, its pure cells are read on demand.
- **Line**: sequential routine, target of `CALL` and `BIF`, may take params `line f($x) {…}`.
- **Cell**: one instruction (`LET`, `AI`, `ASK`, `CALL`, `RET`, `BIF`, `MAP`, `CURL`, a tool…).

## A complete device

```os
// Triage a ticket: typed AI output, a retry, a branch on the answer.
import kit oasys, filesystem from @oasys

agent triager {
  model: "qwen2.5:1.5b"
  provider: ollama
  max_iter: 6
  timeout: 120
  temperature: 0
  seed: 42
  kits: [filesystem]
  tools: { readFile: allow, writeFile: deny }
}

book main {
  page Triage {
    line intake {
      ASK Ticket => { text: { label: "Paste the ticket", type: "text", default: "I was charged twice." } }
      AI[triager] Label @retry(2, wait=1) => {user: $Ticket.text, system: `Classify the ticket.`, output: {
        team: { type: "select", options: ["billing", "tech", "other"] },
        urgent: { type: "bool" }
      }}
      BIF $Label.urgent -> escalate
      LET Route = $Label.team
      RET $Route
    }
    line escalate {
      LET Route = "on-call"
    }
  }
}
```

## Cheatsheet

```text
// comment   /* block */          comments attach to the next element and survive a save
---                                grid row separator between lines / cells (layout only)
LET x = 42                         `=` binds a value: number, "string", [list], {"json": 1}
pub LET x = 1                      scope prefix (priv | pub); canonical form puts it first
LET y = $x * 2 + 1                 formula: $refs, + - * / %, == != < > <= >=, && ||, c ? a : b
LET s = #str($x)                   builtins: #str #num #len #count #sum #range(a,b) #concat
                                   #json(x) #now() #days(a,b)
LET t = (`total: ` $x)             tuple: concatenates parts; backticks = raw text, no ${}
LET d = @data.json[inputs.orders]  file data (next to the .os / device filetree); @f.txt[3:10] lines
AI[agent] R => {user: `...`, system: `...`, output: {f: {type: "int"}}}
                                   `=>` runs something. $R.content, or $R.f with an output schema
ASK A => { f: { label: "Q", type: "text", default: "x" } }    $A.f ; unattended runs use default
CALL[line] / CALL[Page.line($a)] Name / CALL[book.Page.line]  binds the callee's RET to Name
CALL[a, b] Both                    fork-join: Both = {a: …, b: …}
RET $value                         return to caller; at top level it ends the page
BIF $cond -> line                  branch: jump to line (then continue after it, in order)
LET ys = MAP $xs &line             also FILTER $xs &line, REDUCE $xs <init> &line (init: literal or expr like $xs[0]), FOREACH $xs &line
LET a = $xs[0] + $o.key            index lists with [i], objects with .key
LET r => #readFile(path: "a.md")   tool cell: call one tool directly, no model
CURL Resp => -X POST https://api.x/y -H 'K: v' -d '{"q": $x}'    $Resp.status, $Resp.body
LET R @result = …                  $R = {ok: v} | {err: {message, kind, cell, attempts}}
LET R @retry(3, wait=2) = …        re-run a failing cell before failing the run
```

Full grammar, annotations (`@widget`, `@cron`, `@hook`, `@alert`), `ignore`, `scope`, `metric`,
multi-book devices and more complete examples: [references/dsl.md](references/dsl.md).

## Agents, kits and tools

```text
import kit oasys, filesystem from @oasys   // EVERY kit an agent lists must be imported here

agent name {
  provider: ollama | openai | openrouter | remote | browser | typesafe | laya
  model: "qwen2.5:1.5b"  base_url: "http://localhost:11434"   // optional; keys never go in .os
  temperature: 0  seed: 42  max_iter: 12  timeout: 300
  kits: [oasys, filesystem]                  // built-in kits: oasys filesystem shell web memory telegram
  tools: { editBook: allow, runCommand: ask, writeFile: deny }
  ignore: harness                            // hide pages/files from this agent
  skills: [frontend-design]                  // Agent Skills (skills.sh names)
}
```

- An agent has **no prompt**. The prompt lives in the AI cell (`user`, `system`).
- `AI[x]` with an undeclared `x` runs on the default agent (compile warning, not an error).
- An agent with no `kits`/`tools` gets the `oasys` kit implicitly.

Tools of the `oasys` kit, which an agent uses to read and change its own device:

| Tool | Args | Use |
|---|---|---|
| `deviceTree` | — | books → pages → lines with cell counts; call once |
| `readBook` | — | full **rendered** `.os` the agent can see |
| `editBook` | `{oldStr, newStr}` | replace `oldStr` (must occur exactly once) with `newStr`; rejected and nothing written if the result does not parse |
| `runCell` | `{cell}` | force-evaluate one named **cell** (not a line) and return its value |
| `query` | `{formula}` | evaluate `$refs`/`#builtins` read-only |
| `listTools` | — | what this agent may call |

`editBook` rules: copy `oldStr` **verbatim from `readBook`**, never from the file on disk, because
`readBook` shows the canonical render (`line solve($n) {`, lists without spaces, `pub LET`).
Keep `oldStr` small but unique, include the line header when replacing a body, then verify with
`runCell`. `respond` is not a kit tool: the engine adds it when an AI cell has an `output` schema,
and the model must finish by calling it with the fields.

## Create a device

- **App**: open a Board → *New Device* (name, description, capabilities, keywords). Edit it in the
  Textual view (paste `.os`) or the Visual grid, then *Run*. Onboarding starters (chat, summarize,
  automate, data, api) are good templates.
- **Terminal**: write `name.os`, then loop until clean:

```bash
oasys check name.os          # parse + build; exit 2 with {line,col} on error
oasys fmt name.os --write    # canonical form (what readBook and the app show)
oasys run name.os            # execute; NDJSON events when piped
```

If `oasys` is not on `PATH`, inside an OASys checkout use `cargo run -q --bin oasys -- <args>`.

## Run

`oasys run dev.os [--entry Page | Page/line] [--cell Name] [--state key=value]… [--timeout N]`

- Without a TTY the output is NDJSON `ExecutionEvent`s; the last is `run_done`. Look for
  `"type":"error"` and the final `"type":"variables"` snapshot.
- `ASK` reads answers on stdin; with stdin closed it uses each field's `default:`.
- `--state k=v` seeds the store, but a plain `LET k = …` overwrites the seed. Put defaults in a
  `_` line (`line _inputs { LET k = 1 }`): it only fills names that are missing.
- Exit codes: `0` ok · `1` run failed · `2` usage/parse error · `3` permission denied.
- Inspect without running: `oasys tree`, `oasys book read`, `oasys cell run dev.os Name`,
  `oasys query dev.os '<formula>'`, `oasys inspect dev.os`.

## Push, import, clone

```bash
oasys cloud login --email you@example.com --api https://<backend>   # password asked, not stored
oasys cloud projects                                                 # boards of your workspace
oasys cloud push devices/ --project "My Board"                       # one device per .os, idempotent
```

- `push` names a device by its `@benchmark(id)` or the file stem; re-pushing updates it.
- **Import a published device** as a sealed component; agents get a `device:<alias>` tool:
  `import { support, billing@5 } from @org/project` (latest unless `@N`).
- **Import a book** of the same device: `import hr`, then `CALL[hr.Payroll.compute]`.
- **Clone**: copy a device's `.os` from the Textual view into a new device. `oasys cloud pull` does
  not exist yet; never invent it.

## Benchmark

A device becomes a bench case when its entry book carries `@benchmark(...)`:

| Key | Meaning |
|---|---|
| `id` (required) | stable case id; runs are compared by it |
| `field`, `category`, `difficulty` | taxonomy (`difficulty`: low, medium, high) |
| `nature`, `evolution` | `deterministic`/`stochastic`, `static`/`dynamic` |
| `max_ticks`, `max_ms` | budget; over budget = no verdict |
| `verdict` | cell read as pass/fail when there is no `metric` (default `passes`) |
| `process`, `bloom`, `verb`, `task` | LLM process case `pN`; `bloom` ∈ perceive-retrieve, understand-reason, plan-decide, execute-operate, verify-correct, create-orchestrate; `verb` must belong to that column |
| `topology`, `trials` | `single` / `chain` / `colony`; attempts per input (1–50) |
| `epochs` | dynamic case: rounds the agent gets, with `$feedback` between them |

Top-level `metric name { kind, run, read, … }` blocks score the case (`kind`: exact, predicate,
judge, acceptance, cases). Grader pages are `_` pages; hide them from the solver with `ignore`.

```bash
oasys bench backend/benchmarks                      # deterministic cases, diff vs baseline
oasys bench path/ --stochastic --filter json --no-write \
  --model ollama:qwen2.5:1.5b --temperature 0.7 --trials 5 --jobs 4 --out target/bench/runs
oasys bench path/ --stochastic --jobs 4 --persist --project "My Board"   # store runs in the app
oasys cloud bench --project "My Board"              # score on the server instead
```

`--model` (repeatable, `agent=provider:model` or `provider:model`), `--temperature`, `--trials`
need `--stochastic`. `--persist` needs the devices pushed first (`oasys cloud push`). Details and a
full process case: [references/cli.md](references/cli.md).

## Pitfalls

- **Unimported kit = silently missing tools.** `kits: [filesystem]` without
  `import kit filesystem from @oasys` compiles and runs, but the agent never sees those tools.
- **`editBook` is `ask` by default.** Headless runs (CLI, bench, cron) deny it. A self-editing agent
  needs `tools: { editBook: allow }`.
- **`$` in prompts is live.** Every `$name` inside an AI prompt is resolved; an unknown one fails
  the cell (`unresolved ref "$n"`). Describe code in words, or keep it in a `LET` string.
- **`=` vs `=>`.** Pure values use `=`; AI, ASK, CURL and tool cells use `=>`.
  `LET x => 1` is a parse error.
- **`CALL` needs brackets**: `CALL[target]`, never `CALL target`.
- **`runCell` takes a cell name**, not a line name.
- **Index lists with `$xs[0]`**, not `$xs.0` (that is `null`); `.key` is for objects. Any
  `#name(...)` that is not a builtin is a tool call.
- **MAP/FILTER/REDUCE**: use a bare `&line` (unique name, helpers in a `_` page), keep that line
  pure, and never put `@widget` on the MAP cell itself (dropped on save). `RET MAP $xs &f` /
  `RET REDUCE $xs $xs[0] &f` return the result; inside a formula (`#sum(MAP $xs &f)`) they are a
  parse error: bind first (`LET ys = MAP $xs &f`, then `RET #sum($ys)`).
- **Lines are not functions in formulas**: `RET $n * solve($n-1)` is a parse error; run a line with
  `CALL[solve($n - 1)] r` and read `$r`.
- **No list/object literals inside formulas** (`#len([1,2])` fails). Bind the literal to a
  `LET` first and use `$ref`.
- **AI cell body is an object** with only `user`, `system`, `output` keys.
- **BIF falls through.** After the target line ends, the next lines run in order; end a branch
  with `RET`.
- **A `LET` overwrites seeds**; inputs with defaults go in `_` lines.
- **Data files are hard errors when missing**; list harness files in an `ignore` set
  (`"@hidden.json"`) so the solver cannot read them.
- **Keys never go in `.os`**: providers read them from workspace credentials or the environment.
- Use `pnpx` (not `npx`) for Node tooling, e.g. `pnpx skills add worldslicer/skills -s oasys`.
