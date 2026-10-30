# Build a Kiro skill: rust-cucumber-test-case-generator (no scripts)

## Goal

Build an agent skill that turns a Rust codebase into a reviewed, runnable functional test suite using the [`cucumber`](https://crates.io/crates/cucumber) crate (cucumber-rs): Gherkin `.feature` files, Rust step definitions, a coverage figure based on real test runs, and a report.

The skill must contain **instructions only**. The agent running it does the work with its built-in file tools and the target project's existing `cargo`.

## Hard constraints

- The skill folder contains no scripts or executable files: no `.py`, `.sh`, `.js`, `.rs`, Dockerfile or lockfile. Rust examples appear only as code blocks in Markdown.
- The skill must not use Docker, Python, tree-sitter, `syn`-based tools, `cargo install` or any other helper tool.
- The skill must explicitly forbid the agent from writing throwaway helper scripts at run time, including one-line Python or Node commands. The generated Rust test code is the deliverable and is the only code it writes.
- Allowed commands: read-only search (`grep`, `find`) if no built-in tool does the job, and `cargo add --dev`, `cargo check`, `cargo test` and `cargo metadata`.
- If `cargo` or `Cargo.toml` is missing, stop and tell the user. Don't install a toolchain.
- Never edit the target's `src/` files, including to change visibility. If something can't be reached from a test, report it.
- Files may be written only inside `functional-tests/` (or a path the user gives). `Cargo.toml` and `Cargo.lock` may change only after the user approves the exact change.

## Files to create

Check Kiro's current documentation for the correct skill location and frontmatter format. I expect `.kiro/skills/rust-cucumber-test-case-generator/`. If this Kiro version doesn't support skills, stop and tell me before substituting steering files or anything else.

```
rust-cucumber-test-case-generator/
├── SKILL.md                          # frontmatter + the full workflow
├── references/
│   ├── surface-mapping.md            # how to build index.json by hand from Rust source
│   ├── capability-extraction.md      # how to find good capabilities
│   ├── gherkin-authoring.md          # scenario quality + file conventions
│   ├── cucumber-rs.md                # project setup, World, steps, running, pitfalls
│   ├── coverage-calculation.md       # exact coverage procedure
│   └── report-format.md              # exact report layout
├── assets/
│   └── capabilities.example.yaml
└── evals/
    └── evals.json
```

The frontmatter description must trigger when a user asks for functional tests, acceptance tests, BDD or Gherkin scenarios, Cucumber tests, `.feature` files, or "test cases" for a Rust crate, workspace, service or CLI, even if they don't say "Gherkin" or "cucumber".

Keep `SKILL.md` focused on the workflow, and tell the agent exactly when to read each reference file.

## Workflow the skill must define

```
Map surface → Propose capability → User approves? → Write to YAML → (repeat until user is done)
                                                                            │
                                                     Set up cucumber (with approval)
                                                                            │
                     ┌──► Write scenarios + step definitions ◄──────────────┘
                     │                 │
 coverage < threshold│                 ▼
 (deepen EXISTING    │          Compile and run
  capabilities only) │                 │
                     │                 ▼
                     └───────── Check coverage
                                       │ coverage ≥ threshold (default 80%)
                                       ▼
                                 Write report
```

All artifacts live in `functional-tests/`:

| Artifact | Purpose |
|---|---|
| `index.json` | public API surface; the coverage denominator |
| `capabilities.yaml` | approved capabilities, their entities, any exclusions |
| `features/*.feature` | the Gherkin scenarios |
| `cucumber.rs`, `steps/*.rs` | test runner and step definitions |
| `last-run.txt` | captured output of the latest test run |
| `coverage.json` | coverage result and uncovered entities |
| `functional-test-report.md` | final human-readable report |
| `report/*.csv` | four tables that open in Excel |

### Step 1: Map the public surface

Run `cargo metadata --no-deps --format-version 1` to find the packages and their targets, then follow `references/surface-mapping.md` and write `index.json`:

```json
{
  "root": "/abs/path",
  "packages": [{"name": "shop-api", "path": "crates/api", "targets": ["lib", "bin"]}],
  "files_scanned": [{"file": "crates/api/src/auth.rs", "entities": 3}],
  "entities": [
    {
      "id": "crates/api/src/auth.rs:AuthService::create_session",
      "name": "AuthService::create_session",
      "kind": "method",
      "package": "shop-api",
      "file": "crates/api/src/auth.rs",
      "lineno": 42,
      "signature": "pub async fn create_session(&self, user: &User) -> Result<Token, AuthError>",
      "docstring": "Issue a session token...",
      "route": "POST /login",
      "cfg": "feature = \"admin\""
    }
  ],
  "totals": {"public_entities": 37}
}
```

`kind` is one of `function`, `method`, `macro`, `route` or `command`. `route` and `cfg` appear only when they apply.

**Without a parser, the mapping must be auditable:**

1. List every in-scope file first, then process them one at a time. A file with no public entities is still recorded in `files_scanned` with `0`.
2. Read each file. Don't infer entities from search results alone.
3. On large codebases, write `index.json` in batches so progress isn't lost.
4. Run the verification pass in `references/surface-mapping.md`, compare the results with `index.json` and resolve every difference.
5. Report the entity count and any files that couldn't be read.
6. Once capability work begins, `index.json` is frozen. Changes need the user's approval, so the denominator can't drift to improve the score.

### Step 2: Extract capabilities, one at a time

A capability is a user-meaningful behaviour ("Place an order"), not a function. It usually groups several entities.

For each candidate, propose a name, a one-line description and the exact entity IDs, then ask the user to approve, edit or reject it.

- Approve: append to `capabilities.yaml`.
- Edit: apply the changes (rename, add or remove entities, merge, split), then append.
- Reject: drop it and propose the next one.

Continue until the user says they're done. Skip per-capability approval only if the user explicitly asked for that, and tell them it was skipped.

Capabilities can only be created here. Before leaving this step, tell the user about any area of the surface with no capability, because the coverage loop can't add one later.

```yaml
capabilities:
  - id: cap-001
    name: User login
    description: Authenticate a user with email/password and issue a session token.
    entities:
      - src/auth.rs:login_user
      - src/auth.rs:AuthService::create_session
    feature_file: features/user_login.feature
    scenarios: 0
    status: approved   # approved | rejected | proposed
excluded_entities:
  - id: src/legacy.rs:old_handler
    reason: Dead code, not reachable from any route
    approved_by_user: true
```

### Step 3: Set up cucumber

Follow `references/cucumber-rs.md`. Show the user the exact `Cargo.toml` change and every new dependency before making it, and don't continue without approval.

### Step 4: Write scenarios and step definitions

Write one `.feature` file per capability, named as a slug of the capability name (`User login` → `user_login.feature`), with one `Feature:` named after the capability and its description as the body. Write matching steps in `steps/<same_slug>.rs`.

Requirements for `references/gherkin-authoring.md`:

- Describe observable behaviour in the user's or API client's language, never function calls.
- One behaviour per scenario, with concrete Given/When/Then and realistic values.
- Cover the success case, `Err` and validation paths, authorization failures, boundary and empty values, and error handling the code really implements. Never invent behaviour the code doesn't have.
- Use `Scenario Outline` with `Examples` for the same behaviour across several inputs.
- Quality matters more than count.
- **Every scenario must carry `@covers` tags** naming the entities it really exercises:

```gherkin
@covers:src/auth.rs:login_user @covers:src/auth.rs:AuthService::create_session
Scenario: Successful login with valid credentials
  Given a registered user "ada@example.com" with password "correct-horse"
  When the user submits the login form with those credentials
  Then a session token is returned
  And the response status is 200
```

Requirements for step definitions:

- Steps must drive the real entities named in the tags. Never mock the entity under test.
- Every `then` step must contain a real assertion.
- No `todo!()`, `unimplemented!()`, empty step bodies or `#[ignore]`. Search for these before running.
- Reuse existing steps instead of defining near-duplicates, which cause ambiguous-step failures.

### Step 5: Compile and run

Run `cargo check --tests`, fix compile errors, then run the suite and capture the output to `functional-tests/last-run.txt`. The exact commands go in `references/cucumber-rs.md`.

When a scenario fails, read the source and decide which side is wrong:

- **The test is wrong:** fix the scenario or step.
- **The code looks wrong:** leave the scenario failing and report it to the user with the evidence. Never weaken an assertion to match suspected buggy behaviour.
- **It needs something unavailable** (database, external API, credentials): ask the user how to provide it. Never connect to a real or production service. Until then, the scenario is reported as not run.

### Step 6: Check coverage

Follow `references/coverage-calculation.md`. Recalculate from the files every time and never reuse a remembered number.

1. `public` = all IDs in `index.json`, minus `excluded_entities` with `approved_by_user: true`.
2. `tagged` = all `@covers:` values found by searching each feature file.
3. `passed` = scenarios shown as passing in `last-run.txt`. A `Scenario Outline` passes only if all its examples pass.
4. An entity is **covered** only if it is listed under an approved capability **and** tagged on a passing scenario in that capability's feature file.
5. `coverage = covered / public`.
6. Set each capability's `scenarios:` to the number of `Scenario:` and `Scenario Outline:` lines found by search in its file. An outline counts as 1. The agent must never type this number from memory.
7. Report as errors:
   - IDs in `capabilities.yaml` or in tags that aren't in `index.json`, for **every** capability
   - entities listed under a capability but not tagged
   - entities tagged but not listed under that capability

Write `coverage.json` with `total_public_entities`, `covered_entities`, `coverage_pct`, `threshold`, `meets_threshold`, `capabilities_total`, `capabilities_with_scenarios`, `scenarios_total`, `scenarios_passed`, `scenarios_failed`, `scenarios_not_run`, `uncovered_entities` (id, kind, file, signature, route), `excluded_entities` and `errors`.

Show the counts so the user can check the arithmetic. The default threshold is 80%; honour a different one if the user asks.

This measures **entity coverage**, not line or branch coverage, and the report must say so.

### Step 7: Loop or report

**Below threshold: never create a capability.**

1. Read each uncovered entity's source and find the existing capability it naturally belongs to.
2. Add its ID to that capability's `entities`.
3. Add scenarios and steps that really exercise it, with `@covers` tags.
4. Repeat Steps 5 and 6.

If coverage stalls, or an entity is dead code, trait plumbing or fits no capability, ask the user to exclude it or reconsider the capability set. If failing scenarios caused by suspected bugs are holding coverage down, tell the user. Never write a hollow scenario or a throwaway capability. An honest 85% beats a padded 100%.

**At or above threshold:** write the report using `references/report-format.md`, then tell the user where it is and summarize in one paragraph what was and wasn't covered, including failures and suspected bugs.

If the user asks to stop below threshold, write the report anyway and mark it `BELOW THRESHOLD`.

## Rules for references/surface-mapping.md

**File scope**

- Include `.rs` files under each package's `src/`. Use `cargo metadata` for targets with custom paths.
- Skip `target/`, `tests/`, `benches/`, `examples/`, `functional-tests/`, `vendor/`, `build.rs`, `*.pb.rs`, `*_generated.rs` and anything under a `generated/` directory.
- Skip anything inside `#[cfg(test)]`, and any `#[test]` or `#[tokio::test]` function.

**Entity IDs**

- File paths are relative to the workspace or package root, using `/` separators.
- Free function: `src/auth.rs:login_user`
- Inherent method: `src/auth.rs:AuthService::create_session`
- Trait method declaration: `src/store.rs:Store::save`
- Foreign trait implementation method: `src/svc.rs:MyService[Handler]::call`
- Item in an inline module: `src/auth.rs:tokens::issue`
- Drop generics and lifetimes from IDs (`impl<T> Repo<T>` → `Repo`). IDs must contain no spaces, because they're also used in Gherkin tags.
- IDs must be unique. If the same name appears twice in a file, such as under different `#[cfg]` attributes, add a numeric suffix (`::save#2`).

**What counts as public**

- Only plain `pub` counts. `pub(crate)`, `pub(super)`, `pub(in ...)` and unmarked items are private.
- In a library target, an item must also be reachable from `lib.rs` through `pub mod` or `pub use`. A `pub fn` inside a private module with no re-export is private. A re-exported item keeps the ID of the file that defines it.
- In a binary-only package, `pub` items in its modules count, because no outside crate can import them to decide otherwise. `main` is excluded.
- A method is public only if its type is public.

**What counts as an entity**

- `kind: function`: public free functions, including `async`, `const` and `unsafe`.
- `kind: method`: public inherent methods and the methods declared in public traits.
- `kind: macro`: `#[macro_export]` macros and public procedural macros.
- Structs, enums, traits and type aliases aren't entities themselves, because the behaviour lives in their functions.
- Implementations of a local trait aren't separate entities. The trait's method declaration represents them.
- Exclude methods from `impl` blocks of these standard or plumbing traits: `Clone`, `Copy`, `Debug`, `Default`, `Display`, `Drop`, `Eq`, `PartialEq`, `Ord`, `PartialOrd`, `Hash`, `From`, `Into`, `TryFrom`, `TryInto`, `AsRef`, `AsMut`, `Deref`, `DerefMut`, `Iterator`, `Serialize`, `Deserialize`. Anything from `#[derive(...)]` is also excluded.
- Exclude trivial accessors whose whole body only returns or sets a field.
- Record `#[cfg(...)]` conditions on an entity in `cfg`. Entities behind disabled features stay in the denominator unless the user excludes them or chooses to enable those features.
- Code generated by macros isn't visible in the source. If a macro appears to generate public items or routes, tell the user rather than guessing.

**Routes and entry points**

These are always included, whatever their visibility. `route` is `VERB /path`, including prefixes from any enclosing scope. The ID uses the handler's name. If the handler is an inline closure, use `route_<verb>_<path>`, lowercased, with runs of non-word characters replaced by `_` (`GET /health` → `route_get_health`).

| Framework | What to look for |
|---|---|
| Axum | `.route("/x", get(handler).post(other))`, with prefixes from `.nest("/api", ...)`. Each verb is one entity. |
| Actix-web | `#[get("/x")]` style attributes, `#[route("/x", method = "GET")]`, `web::resource("/x").route(web::get().to(handler))`, with prefixes from `web::scope("/api")` |
| Rocket | `#[get("/x")]` style attributes, with prefixes from `.mount("/api", routes![...])` |
| Warp | filter chains using `warp::path!`, `warp::get()` and `.and_then(handler)` |
| Tonic | methods in a `#[tonic::async_trait] impl Greeter for MyGreeter` block, with `route` as `GRPC /package.Service/Method` |
| Clap | `kind: command`. Each subcommand variant of a `#[derive(Subcommand)]` enum, or the root command if there are none. `route` is `CMD <name>`; ID like `src/cli.rs:Commands::Build`. |

For any other framework, read how routes are registered and apply the same rules.

**Verification pass**

Search with these patterns and account for every hit, either as an entity or as a deliberate exclusion:

- `^\s*pub\s+(async\s+|const\s+|unsafe\s+|extern\s+"[^"]*"\s+)*fn\s`
- `^\s*pub\s+trait\s`
- `#\[(get|post|put|delete|patch|head|options|route)\(`
- `\.route\(`, `web::resource\(`, `warp::path`, `tonic::async_trait`
- `Subcommand`, `macro_export`, `proc_macro`

**Always**

- If these rules mislabel something, the source is the truth. Correct it and mention it to the user.
- Build no call graph. Understanding comes from reading the source.

## Rules for references/capability-extraction.md

- Start from routes, commands and entry points, then open each handler and follow its `use` statements to the services, validators and models behind it.
- Use the module tree and naming for grouping. For libraries with no routes, start from what `lib.rs` exports.
- Treat `index.json` as a pointer, and open the file when anything matters.
- A capability should be describable in one sentence and testable in a handful of scenarios. If it needs about 20 scenarios, it's probably two.
- Group CRUD into "Manage X" only if that reads naturally.
- Copy entity IDs exactly. List every public entity the capability really exercises, and never pad the list.

## Rules for references/cucumber-rs.md

Read the docs for the `cucumber` version that gets installed, and write this reference to match its real API. Use the following as the starting point and correct anything that has changed.

**Single-package project.** After approval, run `cargo add --dev cucumber` plus any other dev-dependencies needed (such as `tokio` with `macros` and `rt-multi-thread`), and add:

```toml
[[test]]
name = "cucumber"
path = "functional-tests/cucumber.rs"
harness = false
```

**Virtual workspace (no root package).** `[[test]]` can't go in the root manifest. After approval, create `functional-tests/Cargo.toml` as a new package that depends on the member crates under test, and add it to `[workspace] members`. If a member has `publish` or version rules that conflict, ask the user.

**Runner:**

```rust
use cucumber::World;

mod steps;

#[tokio::main]
async fn main() {
    let features = concat!(env!("CARGO_MANIFEST_DIR"), "/functional-tests/features");
    steps::AppWorld::cucumber()
        .fail_on_skipped()
        .run_and_exit(features)
        .await;
}
```

Adjust the features path for the workspace layout, where the manifest is already inside `functional-tests/`.

**World and steps:**

```rust
use cucumber::{given, then, when, World};

#[derive(Debug, Default, World)]
pub struct AppWorld {
    response_status: Option<u16>,
}

#[given(expr = "a registered user {string} with password {string}")]
async fn registered_user(world: &mut AppWorld, email: String, password: String) {
    // Set up real state here
}

#[then(expr = "the response status is {int}")]
async fn response_status(world: &mut AppWorld, expected: u16) {
    assert_eq!(world.response_status, Some(expected));
}
```

**How tests reach the code**

- Library API: import it from the crate. Integration tests can only see `pub` items.
- HTTP service with a public router or app builder: call it in-process if the project exposes that.
- Binary-only service or CLI: spawn it using `env!("CARGO_BIN_EXE_<name>")`. For a service, use a free port and wait until it's ready.
- If an entity can't be reached any of these ways, don't change its visibility. Report it as a candidate for exclusion.

**Running:**

```bash
cargo check --tests
cargo test --test cucumber -- --color never > functional-tests/last-run.txt 2>&1
cargo test --test cucumber -- --tags '@some-tag'
```

**Pitfalls the reference must cover**

- Scenarios run **concurrently** by default. Shared ports, databases, files or environment variables need isolating. Tag scenarios `@serial`, or set a concurrency limit in the runner.
- A new `World` is created per scenario, so state doesn't carry across scenarios.
- `World` must implement `Debug`.
- Undefined steps are skipped rather than failed by default, which can make a suite look green. `fail_on_skipped()` must stay on.
- Plain `.run()` may not produce a failing exit code. Use `run_and_exit`.
- Two steps matching the same text cause an ambiguous-step failure.
- Don't use retries to hide flaky scenarios.

## Rules for references/report-format.md

Creating a real `.xlsx` needs code, so the report is plain text:

**`functional-test-report.md`:** entity coverage and threshold, PASS or BELOW THRESHOLD, pass/fail/not-run totals, a capabilities table, uncovered entities, exclusions with reasons, suspected bugs with evidence, any errors from Step 6, and the command to rerun the suite.

**`report/*.csv`:** RFC 4180 with UTF-8 encoding. Quote any field containing a comma, quote or line break, and double any embedded quotes.

| File | Columns |
|---|---|
| `summary.csv` | Metric, Value (codebase, packages, public entities, covered entities, coverage, threshold, result, capabilities, capabilities with scenarios, total scenarios, passed, failed, not run, feature files) |
| `capabilities.csv` | ID, Capability, Description, # Entities, # Scenarios, Feature File, Status, Entities |
| `coverage.csv` | Entity ID, Kind, Package, File, Line, Route, Signature, Covered (YES/NO/EXCLUDED), Capability |
| `test-cases.csv` | #, Feature, Scenario, Tags, Steps, Result (PASSED/FAILED/NOT RUN) |

`coverage.csv` lists every entity in `index.json`, one row each. `test-cases.csv` lists every scenario, with steps separated by line breaks.

The agent must read the files back and confirm that the row counts match `index.json` and the scenario total.

## evals/evals.json

Use this format: `{"skill_name": "...", "evals": [{"id", "prompt", "expected_output", "files"}]}`. Include these prompts:

1. "I have a small Axum service in this repo. Generate cucumber tests for it and a coverage report. Auto-approve the capabilities you find so I don't have to sit through each one."
2. "Write acceptance tests for this Actix-web API. Every route handler needs at least a happy-path and an error scenario, and I want to see what's still uncovered."
3. "Generate BDD test cases for this Rust library crate. Walk me through the capabilities one at a time before writing anything, and stop at 80% coverage. Final deliverable should be a spreadsheet."
4. "Add cucumber tests for this clap CLI in a Cargo workspace."

In every eval, `expected_output` must state that no helper scripts were written, `src/` was unchanged, `Cargo.toml` changed only after approval, and the suite really ran.

## Verify before finishing

Create fixtures in a temporary directory **outside the skill**: a small Axum service with a library target and a binary, a small Actix-web service, a clap CLI, and a virtual workspace with two crates.

Work out the expected entities by hand first, including traps:

- a `pub(crate)` function
- a `#[cfg(test)]` module
- a `pub fn` in a private module, with and without a `pub use` re-export
- a derive-only struct
- an `impl Display`
- a trivial accessor
- an inline closure route
- a route nested under a prefix
- a private route handler

Then follow the finished skill against each fixture and confirm that:

- [ ] `index.json` matches the expected entities exactly
- [ ] approvals happen one capability at a time unless auto-approve was requested
- [ ] `Cargo.toml` changes wait for approval, and the workspace fixture gets its own test package
- [ ] the `cucumber` version in use accepts `@covers` tags containing `:`, `::`, `[` and `]`. If it doesn't, change the ID format everywhere in the skill and retest
- [ ] an undefined step makes the run fail
- [ ] a deliberately broken assertion fails the run, and its entity is no longer covered
- [ ] a scenario missing its `@covers` tag lowers coverage
- [ ] a mistyped entity ID is reported, including on a capability with `scenarios: 0`
- [ ] concurrent scenarios don't interfere with each other on the service fixture
- [ ] the loop adds scenarios and never adds a capability
- [ ] the report's numbers match `coverage.json` and `last-run.txt`, and the CSVs open with correct columns
- [ ] `src/` is unchanged in every fixture
- [ ] the skill folder contains only `.md`, `.yaml` and `.json` files

Fix the skill text whenever a run exposes a gap, then rerun. Delete the fixtures when finished.

## When you finish, report

- the files you created
- the skill location you used and how you confirmed it
- the `cucumber` version you tested against
- the results of each checklist item, including anything that failed
- any rule you couldn't follow exactly, and why
