# `oasys` CLI — reference

One binary, `gcloud`-shaped (group → verb → flags), for people and agents alike.

- Without a TTY every command prints JSON (`--json` forces it). stdout = data, stderr = logs and
  errors. `oasys schema` prints the whole command tree as JSON.
- Exit codes: `0` ok · `1` run failed · `2` usage or parse error · `3` permission denied.
- Errors: `{"error":{"code","message","line?","col?","path?"}}`.
- Not on `PATH`? Inside an OASys checkout: `cargo run -q --bin oasys -- <args>`, or build once
  with `cargo build --release --bin oasys` and use `target/release/oasys`.

## Author and inspect (offline)

| Command | What it does |
|---|---|
| `oasys check <file.os>…` | parse + build; several files = the books of one device |
| `oasys fmt <file.os>… [--write]` | canonical render; without `--write` it prints it |
| `oasys inspect <file.os> [--page P]` | books, pages, lines, cells, cron, hooks |
| `oasys tree <file.os>` | `deviceTree`: pages and lines with cell counts |
| `oasys book read <file.os>` | `readBook`: the rendered source an agent sees |
| `oasys book edit <file.os> --old S --new S` | `editBook`: `S` must occur once; refused if the result does not parse |
| `oasys cell run <file.os> <Cell>` | `runCell`: evaluate one named cell |
| `oasys query <file.os> '<formula>'` | evaluate `$refs`/`#builtins` (fresh store: refs are empty unless a cell ran) |
| `oasys tool list [--kit k]` | built-in tools with kit, default mode and schema |
| `oasys tool run <name> --file <dev.os> --args '<json>'` | call one tool against a device |

Device-scoped commands also take `--scope <Name>` (one declared `scope`), `--gov
all|agents,conf,ignore,imports,run` (governance capabilities, deny by default) and `--dry-run`.
Copy `--old` text from `oasys book read`, not from the file: the render normalizes spacing.

Typical agent loop on a device:

```bash
oasys tree dev.os
oasys book read dev.os
oasys book edit dev.os --old 'RET null' --new 'RET $n * $n'
oasys cell run dev.os got
oasys check dev.os && oasys run dev.os
```

## Run

```bash
oasys run dev.os                          # whole device: pages of `main` start together
oasys run dev.os --entry Chat/Main        # one page or one line
oasys run dev.os --cell Total             # one globally named cell
oasys run dev.os --state score=50 --state 'user={"name":"ana"}'   # JSON when it parses
oasys run dev.os --timeout 120            # watchdog in seconds (0 = off)
echo '{"message":"hi"}' | oasys run chat.os   # answers an ASK from stdin
```

The output is NDJSON `ExecutionEvent`s, the same stream the app receives over SSE. Useful types:
`cell_output`, `cell_status`, `variables` (store snapshot), `error`, `page_done`, `run_done`.
Credentials in the CLI come from `OASYS_CREDENTIAL_<NAME>` or `.env`. Tools in `ask` mode are
denied (no approver), so give a self-editing agent `tools: { editBook: allow }`.

## Account (`oasys cloud`)

The CLI calls the same HTTP routes as the app, with your token. Sign in once:

```bash
oasys cloud login --email you@example.com --api https://<backend>   # default http://localhost:3000
oasys cloud whoami
oasys cloud projects                              # boards (API name: projects)
oasys cloud push a.os dir/ --project "My Board"   # creates the board by name if missing
oasys cloud bench --project "My Board"            # score every device of the board on the server
```

- The session is stored in `~/.config/oasys/session.json` (0600); the password never is.
  `OASYS_PASSWORD` avoids the prompt in scripts.
- Supabase URL and anon key come from `OASYS_SUPABASE_URL` / `SUPABASE_URL` and
  `OASYS_SUPABASE_ANON_KEY` / `VITE_SUPABASE_ANON_KEY`.
- `push` is idempotent: a device is named by its `@benchmark(id)` or the file stem.
- There is no `oasys cloud pull` or remote run yet.

## Bench

```bash
oasys bench [corpus|case.os] [--filter id-substring] [--baseline file] [--no-write]
            [--stochastic] [--model [agent=]provider:model]… [--temperature T] [--trials N]
            [--jobs N] [--out DIR] [--persist --project BOARD] [--promote]
```

| Flag | Effect |
|---|---|
| (none) | runs deterministic cases, compares with `target/bench/latest.json`, overwrites it |
| `--no-write` | compare without overwriting the baseline |
| `--stochastic` | also run cases that call a model (`process:` or `epochs:` in the header) |
| `--model` | swap models without editing `.os`; the named-agent form wins over the global one |
| `--temperature T` | 0–2 for every agent; seeds stay, trial k uses `seed + k` |
| `--trials N` | 1–50 attempts per input instead of the header's `trials` |
| `--jobs N` | 1–64 units at once (a case, or one attempt); output order is unchanged |
| `--out DIR` | one JSONL row per (case, input, trial): model digest, seed, temperature, passed, effort |
| `--persist --project B` | store each case as a run of the board device named by its id (push first) |
| `--promote` | write solved dynamic cases as deterministic siblings |

`--model`, `--temperature` and `--trials` without `--stochastic` are usage errors (exit 2). With
local Ollama, `--jobs` above `OLLAMA_NUM_PARALLEL` gains nothing.

### Writing a case

- Header `@benchmark(...)` on `book main` (keys in [SKILL.md](../SKILL.md#benchmark)).
- Task data in `data.json` next to the `.os`: `{inputs, expected}`, or many inputs as
  `{"cases": [{"id", "split": "design"|"hidden", "inputs", "expected"}]}`. Every variant `pN.os` of
  a task must see the same data.
- `_Task` reads `@data.json[inputs.x]`; `_Accept` reads `@data.json[expected]`.
- `metric` kinds: `exact` (`read` vs `against`), `predicate` (`threshold`, `goal`), `judge` (an LLM
  verdict; auxiliary only), `acceptance` (a boolean cell), `cases` (dynamic: calls `call:` on
  hidden cases).
- `bloom` columns and their verbs: perceive-retrieve (extract, retrieve, search, …),
  understand-reason (classify, summarize, analyze, …), plan-decide (plan, schedule, …),
  execute-operate (compute, transform, apply, …), verify-correct (debug, verify, evaluate, …),
  create-orchestrate (compose, implement, build, …). An unknown verb is exit 2.
- Topologies: `single` (one call), `chain` (stages), `colony` (roles, at least one verifier),
  `verification` (a deterministic check on the input and the answer, one retry with its message).

### A complete LLM process case

`p2.os` next to a `data.json` such as
`{"inputs": {"orders": [{"customer": "ana", "amount": 30}]}, "expected": {"ana": 30}}`:

```os
metric correct {
  kind: exact
  run: _Accept.check
  read: got
  goal: max
  against: want
}

agent solver {
  model: "qwen2.5:1.5b"
  provider: ollama
  temperature: 0
  seed: 42
}

book main @benchmark(
  id: "llm-execute-operate-order-totals-p2", field: "llm", category: "bloom", difficulty: "low",
  nature: "stochastic", evolution: "static", max_ms: 60000,
  bloom: "execute-operate", verb: "compute", task: "order-totals", process: 2, topology: "chain", trials: 3
) {
  page _Task {
    line inputs {
      LET orders = @data.json[inputs.orders]
    }
  }
  page _Accept {
    line check {
      LET want = @data.json[expected]
      LET got = $Answer
    }
  }
  page Solve {
    line run {
      AI[solver] Notes => {user: (`Group these orders by customer and add the amounts. Do not answer yet. ` $orders), system: ``}
      AI[solver] Answer => {user: (`Orders: ` $orders ` Notes: ` $Notes.content ` Give the total per customer.`), system: ``, output: {
        ana: {
          type: "int"
        }
      }}
    }
  }
}
```

```bash
oasys bench tasks/order-totals --stochastic --no-write --trials 3 --out target/bench/runs
```
