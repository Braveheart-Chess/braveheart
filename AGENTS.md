# Guidance for coding agents

This repository is a single-threaded UCI chess engine in Rust: the engine, perft, bench, the performance gate, SPRT, matches, the rating ladder and the embedded net. Self-play data generation and relabelling live in [braveheart-selfplay](https://github.com/Braveheart-Chess/braveheart-selfplay), and NNUE training and the trainer's engine check (`nnue-eval`) in [braveheart-trainer](https://github.com/Braveheart-Chess/braveheart-trainer); each has its own `AGENTS.md`. Read [README.md](README.md) for the layout, [docs/roadmap.md](docs/roadmap.md) for the phased plan and its exit gates, [docs/lessons-from-rusty-chess-v1.md](docs/lessons-from-rusty-chess-v1.md) for the bugs that sank the first attempt and the guards that catch them, and [CONTRIBUTING.md](CONTRIBUTING.md) for the full rules. The rules below are the ones you MUST follow.

## Rules

- **Never push to `main`.** Every change goes through a pull request on a branch. There are no exceptions after the initial scaffold.
- **Commits are a single line**: `type(scope): subject`, at most 100 characters, with a lowercase imperative subject and no trailing period. Types: `feat`, `fix`, `perf`, `revert`, `docs`, `refactor`, `test`, `build`, `ci`, `style`, `chore`; never invent another. Scope: exactly one of `SCOPES` in `commitlint.config.mjs` (the Scopes table in CONTRIBUTING.md); never invent one, and if none fits, ask. The only thing allowed after the header is a blank line and `Co-authored-by: Name <email>` trailers. No body text. Put details in the PR description.
- **PR titles are valid commit headers**, scope included; squash merges use the title as the commit message.
- **PR bodies use the template's sections, in order, every one filled in**; write "n/a" rather than deleting one:

  ```sh
  gh pr create --title "type(scope): subject" --body-file <file>
  ```
- **Run the repo's checks locally before opening the PR** (see below).
- **Never change search or eval without a bench run.** Put the `Bench: <nodes>` signature before and after in the PR description. A changed signature on a `perf` PR is a bug; a `perf` PR must leave the signature unchanged. Use `feat` for anything that changes the signature and `fix` for correctness (perft, TT, mate scores).
- **Every measured layer ships with a coverage suite and a performance bar, and both only grow.** Move generation is covered by `tests/perft.epd` (append positions, never remove or change a line), search by the bench positions, efficiency by the `stats` counters. The bars live in `perf-targets.toml`; raise one when the engine clears it, never lower one. CI's `Bench` job runs `perf-gate`: speed bars are reported on every PR and enforced on release PRs, efficiency bars are always enforced, and a PR that loosens a bar fails. The same job posts the bench comment with deltas against the latest release; read it before claiming a speed or efficiency change, and quote its numbers in the PR description.
- **Never claim an Elo gain without an SPRT result in the PR.** A `feat` that touches search or eval gets the `sprt` label; the `SPRT` workflow plays the PR against its base twice at once, timed at the `[sprt] tc` control and at fixed nodes per move, and posts both in one comment. **Both results gate** (the owner's rule since 2026-11-02): the timed one is what the engine does at a clock, the fixed-node one whether the search got smarter per node (fixed nodes alone reward a change that makes nodes dearer, and once overstated five releases' gains about sevenfold, so neither is enough alone). Quote both, and the comment's reading of the pair, in the PR description. Only an accepted H1 in **both** passes the check: H0 or an inconclusive result in either fails it, and a feature that does not gain Elo is reverted even if it looks right. A speed gain that is no smarter per node fails too. The owner may merge over a failed check for a reason stated in the PR (a speed change, an inconclusive result at high LOS, a correctness fix); an agent never does. Never run SPRT on a laptop and paste the numbers; the standardized runner is the record.
- **Every experiment updates `docs/`.** SPRT, matches, ratings and one-off measurements all get recorded, including smoke tests and failed, cancelled, interrupted and dropped runs; data generation, relabelling and training are recorded in their own repositories. See [Experiments and the research record](#experiments-and-the-research-record).

## Experiments and the research record

This project is research: a result counts only when someone else can reproduce it and check it from the record.

| When you run | Record it in |
|:--|:--|
| Datagen, a Relabel run, a generation's or label set's audit | braveheart-selfplay's [docs/datasets.md](https://github.com/Braveheart-Chess/braveheart-selfplay/blob/main/docs/datasets.md) |
| Training, a net's SPRT, a release's rating of a net | braveheart-trainer's [docs/nets.md](https://github.com/Braveheart-Chess/braveheart-trainer/blob/main/docs/nets.md), and its "What we've learned" and "Next experiments" when the result changes what to try next |
| A design measurement (search, evaluation, king buckets) | That topic's page in `docs/`; relabelling and datagen settings are braveheart-selfplay's |
| Anything that completes or changes a phase item | [docs/roadmap.md](docs/roadmap.md) |
| A research session | [docs/research-log.md](docs/research-log.md) is frozen at the split (2026-11-17): new entries go to the repository whose experiment it is |

- **Record it promptly.** Use the same PR when the experiment has one. Otherwise open a docs PR when the run ends, before another experiment builds on the result.
- **Decide before you run.** Write down the question, the one thing that changes, what stays the same and what result would keep or drop it. Don't move the criterion after seeing the result.
- **Make it reproducible.** Record the commit or tag, the workflow inputs or exact command, the run id, seeds, the generations and label sets, each net's id and SHA-256, the trainer revision, and the hardware and market.
- **Use primary sources.** Take numbers from shard manifests, `task.json`, `datagen-audit`, `training.json`, a PR's SPRT and bench comments, or release notes, never from progress lines, memory or another document. Link the source. Label estimates, such as costs, and local measurements as what they are.
- **Audit before you build on it.** A generation needs a full audit with a GO, and a label set its recorded audit, before anything trains on it. Name a net's data by its generations and each one's label set, and compare losses only on identical held-out positions and labels. Where a check could pass without testing anything, add a negative control, as the determinism step does.
- **Say what wasn't verified.** Name any check you skipped or couldn't run, and why. When you check a ledger against its sources, add the result to its audit log, gaps included.
- **Keep every result and correct in the open.** Never delete or reword a failed or disappointing result. Record every attempt, including reclaimed legs and re-runs, and the gate in force when the decision was made. A later rule doesn't rewrite an earlier decision; note how the two differ.

## Engine rules

- Correct before fast: perft stays in CI permanently and must pass before any search change merges.
- The root is an ordinary negamax node: children get `-beta, -alpha` and alpha rises on improvement. Never search root moves with a full window.
- An aborted search is discarded entirely: once the stop flag is set, no transposition table store happens, no score is compared, and the result is the last *completed* iteration. The bench signature must be identical run to run; any variance means abort handling is leaking.
- A transposition table bound is derived from the final score against the original alpha and beta at every node, quiescence included. Never store a bounded score as exact.
- A move-picker stage with zero cutoffs over a bench run is a bug until proven otherwise; read the per-stage counters.
- Mate scores are offset by ply from the root, never by remaining depth.
- No heap allocation in the search path. Move lists are stack-allocated.
- Telemetry (`--features stats`) compiles to nothing in release builds; every counter is a macro.
- One feature per commit so the bench signature can be bisected.
- Eval is frozen at PeSTO until the Phase 4 exit gate holds.
- Cargo workspace members are listed explicitly in `Cargo.toml` and every crate carries its own `version`; Release Please bumps them together. Never edit versions or `CHANGELOG.md` by hand.

## Before opening a PR

```sh
cargo fmt --all --check
cargo clippy --workspace --all-targets --all-features -- -D warnings
cargo test --workspace --locked
cargo test --workspace --locked --features stats
cargo run --release -- bench   # note the signature
npm ci && npm test && npm run test:commitlint
```

## Things that need the owner

Repository secrets (`RELEASE_BOT_CLIENT_ID`, `RELEASE_BOT_PRIVATE_KEY`), the release bot app installation, repository settings, and the Homebrew tap. List them in the PR instead of attempting them.
