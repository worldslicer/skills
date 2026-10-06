# The `.os` language — reference

Every ` ```os ` block in this file is a complete device that compiles (`oasys check`). Fragments
use ` ```text `.

## File layout

A device is one `.os` file, or several files that are its books (`main.os`, `space.os`,
`metrics.os` checked together: `oasys check main.os space.os metrics.os`). Top-level elements,
in canonical order:

```text
import kit oasys, memory from @oasys     // tool kits (built-in org: @oasys)
import { support, billing@5 } from @org/project   // published devices, sealed; @N pins a version
import hr                                // another book of this device (also: import hr as h,
                                         // import hr { Payroll })
ignore harness { "main/*" "_Grade" "@hidden.json" }   // named visibility set: book/page globs, files
scope Repo = @MyRepo/src: rw             // filesystem grant over a workdir alias: r | rw | rwx
type Verdict = "pass" | "fail"           // type declaration (documentation + UI)
agent name { … }                         // device-level, shared by all books
metric name { … }                        // bench scoring (see Benchmarks)
book main @annotations { page … }        // entry book: the only one that runs
book other { … }                         // library / data book
```

`agent`, `ignore`, `import` and `scope` live in the entry book's file (`main.os`).

## Structure

```text
book main {
  page Name @ann {          // pages of main run in parallel when the device starts
    line name {             // sequential routine
      <cell>                // one instruction per line of text
      ---                   // grid row break (layout only)
    }
    line helper($a, $b) {   // params bind by position (CALL args, MAP item, REDUCE acc+item)
      RET $a + $b
    }
  }
  page _Defs { … }          // `_` = definitions: not run in the flow; pure cells read on demand
}
```

- A `_` line or cell only fills a name that is missing from the store, so it is the default of
  an input: a seed (`oasys run --state`, cron, hook, caller) wins.
- A named literal `LET` in a book other than `main` is a **register**: `$Memory.Fails` reads it
  from any cell without import; `#remember(key: "Memory.Fails", value: $x)` (kit `memory`)
  writes it; `Memory.Queue.-` appends to a list register.
- Comments: `// …` and `/* … */`, after whitespace or at line start. They attach to the next
  element; a comment inside an expression or schema is dropped with a warning.
- Names: any identifier (`A1`, `Total`, `día`). Keywords are case-insensitive except `CURL`/`WEB`.

## Values and formulas

```text
LET n = 42            LET s = "text"          LET b = true        LET x = null
LET xs = [1, 2, 3]    LET o = {"k": "v"}      (JSON; a `{k: 1}` without quotes is a string + warning)
LET Pending @widget(...)                     valueless declaration, filled later
LET y = $n * 2        LET z = ($a + $b) / 2   LET c = $n > 0 ? "pos" : "neg"
LET ten = =5 + 5      `=` marker: arithmetic without $ or # is otherwise text
LET first = $xs[0]    LET k = $o.k            index a list with [i], an object with .key
LET t = (`Hello ` $Name `!`)                 tuple: concatenation of parts
LET raw = `literal text, no ${} templates`
```

Operators: `+ - * / %`, `== !=` (structural: lists and objects compare by value), `< > <= >=`,
`&& ||`, `cond ? a : b`. Builtins: `#str #num #len #count #sum #range(a, b)` (b excluded)
`#concat #json #now() #days(a, b)`. List literals are not allowed inside a formula: bind them to
a `LET` first. Delimit a ref followed by text with `{{ }}`: `"Memory.Notes.{{$Id}}.title"`.
Any other `#name(...)` is a **tool call**, not a builtin (`#at($xs, 1)` fails with `unknown tool`).

File data (from the folder of the `.os`, or the device filetree in the app):

```text
LET cfg = @config.json            whole file      LET a = @data.json[inputs.orders]   key path
LET part = @notes.txt[3:10]       lines 3..10     LET all = @folder                    list
```

A missing file is a hard error.

## Cells that run something (`=>`)

```text
AI[agent] Name @ann => {user: `…$ref…`, system: `…`, output: { field: { type: "int" } }}
ASK Name @timeout(60) => { field: { label: "…", type: "text", default: "…" } }
LET Name => #toolName(key: value, other: $ref)      // TOOL cell, no model
LET Name => #readFile:Repo(path: "a.md")            // tool call through the grant `scope Repo`
CURL Name => -X POST https://host/path -H 'K: v' -d '{"q": $x}'
WEB Name @widget(...) => http://localhost:{{$Port}}/app
```

- AI: the body object takes only `user`, `system`, `output`. Every `$ref` in it is resolved at run
  time; an unknown ref fails the cell. The value is `{content, thinking?}`, or the `output` fields
  (`text int num bool select multiselect`; `select`/`multiselect` need `options`). With `output`,
  the model must end by calling the synthetic `respond` tool.
- ASK: value is `{field: answer}`. Unattended runs (CLI with closed stdin, cron, bench) answer with
  `default:`; a required field without default fails the cell. `@timeout(N)` uses the defaults
  after N seconds.
- CURL: value is `{status, headers, body}`; a non-2xx status is data, not an error.
- TOOL args: `key: value` pairs or one positional. A whole-value `$ref` passes the typed value.

## Control flow

```text
CALL[line]                         run a line of this page, then come back
CALL[Page.line($a, 3)] Result      line of another page (this book, then imported books)
CALL[book.Page.line] R             another book (needs `import book`)
CALL[a, b] Both                    fork-join on isolated copies: Both = {a: …, b: …}
RET $value                         return to the caller; with no caller, end the page
BIF $cond -> line                  jump to `line`; execution continues in order after it
LET ys = MAP $xs &f                f($x) per item
LET ok = FILTER $xs &isOk          keep items where the line RETs true
LET sum = REDUCE $xs 0 &add        add($acc, $x); init = literal or expression ($xs[0], $a * 2)
FOREACH $xs &notify                side effects per item
```

Higher-order targets: prefer a bare `&line` with a line name unique in the device (helpers live
in a `_` page). The run resolves `&line` in the caller's page, then its book, then `main`;
`&Page.line` also works in `oasys run`, but `oasys bench` evaluated it to `null` in our tests.
Keep those lines pure (`LET`, formulas, `RET`): no `CALL`, AI or tools inside them. Do not put
annotations (`@widget`) on a `MAP`/`FILTER`/`REDUCE` cell: the renderer drops them on save;
show a copy instead (`LET Total @widget(...) = $Revenue`).

Failures work like Rust: no annotation = `?` (the failure stops the run). `@result` turns the
value into `{ok: v}` or `{err: {message, kind, cell, attempts}}` and the run continues; test it
with `BIF $R.err -> handle`. `@retry(3, wait=2)` re-runs the cell first.

## Annotations

| Annotation | On | Effect |
|---|---|---|
| `@widget(id=x, type=display\|input, component=Chat, at=[c,r], size=[w,h], label="…", subscribe=["A","B"])` | cell | shows the cell on the device screen |
| `@cron("*/5 * * * *", exclusive=true, timeout=600)` | named page/line/cell of `main` | scheduled run |
| `@hook("http")` / `@hook("telegram", bot=%Bot)` | page/line | external trigger (`%Name` = credential) |
| `@alert(level: critical)` | cell | alert event when the cell finishes |
| `@result`, `@retry(n, wait=s)` | cell | failure policy |
| `@timeout(N)` | ASK | answer with defaults after N s |
| `@benchmark(id: "…", …)` | entry book | bench case header |
| `@conf(run=hot)` | entry book | run mode (`cold` default) |
| `@folder(x)` | book | file-tree grouping only |

Args accept `=` or `:`. Annotations round-trip verbatim.

## Agent block

```text
agent name {
  model: "qwen2.5:1.5b"
  provider: ollama              // ollama openai openrouter remote browser typesafe laya
  base_url: "http://localhost:11434"
  max_iter: 12                  // tool-loop iterations per AI cell
  timeout: 300                  // seconds per provider call
  temperature: 0                // ≥ 0
  seed: 42                      // whole number; bench trial k uses seed + k
  kits: [oasys, filesystem]     // must be imported with `import kit … from @oasys`
  tools: { editBook: allow, runCommand: ask, writeFile: deny }
  ignore: harness               // pages/files this agent cannot see or edit
  scope: Repo                   // filesystem ceiling
  skills: [frontend-design]     // Agent Skills; installed on the workspace
  governance: { ignore: allow } // let it edit governed constructs (agents, ignore, conf, imports, run)
}
```

Built-in kits: `oasys` (deviceTree, readBook, editBook, runCell, query, listTools), `filesystem`,
`shell`, `web`, `memory` (remember, recall, saveState, loadState, …), `telegram`. List them with
`oasys tool list [--kit k]`. A tool in `ask` mode is denied in headless runs.

## Examples

### 1. Pure logic: inputs with defaults, branch, call with args, MAP

```os
// Grade a score. Run: oasys run grade.os --state score=50
book main {
  page Grade {
    line _inputs {
      LET score = 72
      LET names = ["ana","ben","cy"]
    }
    line decide {
      BIF $score >= 70 -> pass
      LET verdict = "fail"
      RET $verdict
    }
    line pass {
      LET verdict = "pass"
      CALL[_Math.square($score)] Squared
      LET Shout = MAP $names &upper
    }
  }
  page _Math {
    line square($x) {
      RET $x * $x
    }
    line upper($s) {
      RET #concat($s, "!")
    }
  }
}
```

### 2. Data from a file: FILTER, MAP, REDUCE, widgets

```os
// orders.json next to this file: {"orders": [{"customer": "ana", "amount": 30, "paid": true}]}
book main {
  page Orders {
    line load {
      LET orders = @orders.json[orders]
    }
    ---
    line report {
      LET Paid = FILTER $orders &isPaid
      LET Amounts = MAP $Paid &amount
      LET Revenue = REDUCE $Amounts 0 &add
      LET Total @widget(id=revenue, type=display, at=[0,0], size=[6,2], label="Revenue") = $Revenue
      LET Unpaid @widget(id=unpaid, type=display, at=[6,0], size=[6,2], label="Unpaid") = #len($orders) - #len($Paid)
    }
  }
  page _Rules {
    line isPaid($order) {
      RET $order.paid
    }
    line amount($order) {
      RET $order.amount
    }
    line add($acc, $x) {
      RET $acc + $x
    }
  }
}
```

### 3. Chat loop in the browser (no install, no key)

```os
agent assistant {
  model: "onnx-community/gemma-4-E2B-it-ONNX"
  provider: browser
}

book main {
  page Chat {
    line Main {
      ASK Message => {
        message: {
          label: "Your message",
          type: "text"
        }
      }
      AI[assistant] Reply => {user: $Message.message, system: `You are a helpful assistant. Answer briefly.`}
      ---
      CALL[Main] Next @widget(at=[0,0], id=chat, component=Chat, label="Chat", subscribe=["Message","Reply"], size=[12,8])
    }
  }
}
```

### 4. Scheduled probe: CURL with `@result`, a register, an alert

```os
// Check an API every hour; count failures in a register that survives runs.
import kit memory from @oasys

book main {
  page Watch @cron("0 * * * *", exclusive=true) {
    line probe {
      CURL Health @result => https://api.example.com/health
      BIF $Health.err -> down
      BIF $Health.ok.status != 200 -> down
      RET "up"
    }
    line down {
      LET Fails @alert(level: critical) = $Memory.Fails + 1
      LET Saved => #remember(key: "Memory.Fails", value: $Fails)
    }
  }
}

book Memory {
  page State {
    line registers {
      LET Fails = 0
    }
  }
}
```

### 5. An agent that edits its own device, and the program checks the result

```os
// The builder may only see and edit book `space`; `main` is hidden from it.
import kit oasys from @oasys
import space

ignore harness {
  "main/*"
}

agent builder {
  model: "qwen2.5:1.5b"
  provider: ollama
  max_iter: 12
  temperature: 0
  seed: 42
  kits: [oasys]
  tools: { editBook: allow }
  ignore: harness
}

book main {
  page Build {
    line work {
      AI[builder] Work => {user: `Call readBook. In book space, page Work, replace the body of line solve so it returns the square of its parameter n. Copy oldStr exactly from readBook, then verify with runCell got.`, system: ``}
    }
    ---
    line test {
      CALL[space.Work.solve(7)] Got
      LET Passed = $Got == 49
    }
  }
}

book space {
  page Work {
    line solve($n) {
      RET null
    }
    line check {
      LET xs = [2,3]
      LET got = MAP $xs &solve
    }
  }
}
```

Note the prompt says "parameter n" in words: a `$n` inside the prompt would be resolved against
the store and fail the cell.

### 6. A deterministic bench case

```os
metric correct {
  kind: acceptance
  run: _Acceptance.metrics
  read: all_correct
  goal: max
}

book main @benchmark(
  id: "math-arith-low-pair-max-001", field: "math", category: "arithmetic", difficulty: "low",
  nature: "deterministic", evolution: "static", max_ticks: 100
) {
  page _Problem {
    line inputs {
      LET pairs = [[48,18],[56,98],[7,7]]
      LET expected = [48,98,7]
    }
  }
  page _Acceptance {
    line metrics {
      LET all_correct = $output == $expected
      LET passes = $all_correct
    }
  }
  page Solution {
    line solve {
      LET output = MAP $pairs &larger
    }
  }
  page _Rules {
    line larger($p) {
      RET $p[0] > $p[1] ? $p[0] : $p[1]
    }
  }
}
```

`passes` is the default verdict cell; the `metric` block makes the verdict explicit.

LLM process cases (`process:`, `bloom:`, `verb:`, `@data.json`) are in
[cli.md](cli.md#a-complete-llm-process-case).
