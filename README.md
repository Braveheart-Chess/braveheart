<p align="center">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="https://raw.githubusercontent.com/Braveheart-Chess/braveheart-branding/main/banner-dark.png">
    <img src="https://raw.githubusercontent.com/Braveheart-Chess/braveheart-branding/main/banner-light.png" alt="Braveheart" width="640">
  </picture><br>
  <sub>Logo © the artist, used under licence. All rights reserved; not covered by this repository's licence (<a href="https://github.com/Braveheart-Chess/braveheart-branding#licence">details</a>).</sub>
</p>

# Braveheart

A single-threaded UCI chess engine in Rust whose classical search is
measurably strong before any NNUE work starts. Every change is justified by a
number: perft for correctness, bench for speed, stats for search efficiency,
and SPRT for Elo.

**The name.** *Braveheart* is the film about William Wallace, who shares the
author's surname, and Scotland has its own chess opening: the Scotch
(1.e4 e5 2.Nf3 Nc6 3.d4), an open, aggressive start. The mascot follows suit:
a snarling panther with a woad-blue face, a braid and a tartan scarf.

**Self-play and training have their own repositories.** This one holds the
engine and what measures it: perft, bench, the performance gate, SPRT, matches,
the rating ladder and the embedded net. Self-play data generation, relabelling,
their audits and the dataset ledger moved to
[braveheart-selfplay](https://github.com/Braveheart-Chess/braveheart-selfplay); the bullet trainer, the
`Train` and `GPU smoke` workflows, `nnue-eval` and the net ledger moved to
[braveheart-trainer](https://github.com/Braveheart-Chess/braveheart-trainer). Both were split out
with their history in November 2026 and build against a pinned engine release.
A new net comes back here as an ordinary pull request, judged by its SPRT.

Braveheart was called **rusty-chess-v2** through v0.52.0. The research record
([docs/research-log.md](docs/research-log.md), braveheart-trainer's
[nets.md](https://github.com/Braveheart-Chess/braveheart-trainer/blob/main/docs/nets.md),
braveheart-selfplay's [datasets.md](https://github.com/Braveheart-Chess/braveheart-selfplay/blob/main/docs/datasets.md),
[CHANGELOG.md](CHANGELOG.md)) keeps the old name where it describes what ran,
and stored data schemas such as `rusty-chess-selfplay/1` keep it too, so
existing data stays readable. The data has changed buckets three times, never
its keys: a path recorded under `s3://jwall-engineering-rusty-chess-nnue/`,
`s3://jwall-engineering-braveheart-nnue/` or the management account's
`jwall-engineering-braveheart-*` buckets names the same key in the production
account's `jaws-braveheart-selfplay-positions` (`data/selfplay/`) or
`jaws-braveheart-trainer-nets` (`data/nets/`). The old buckets were deleted on
2026-11-21, after a byte-for-byte check against the production copies.

The full plan is in [docs/roadmap.md](docs/roadmap.md), and the post-mortem of the
first attempt that shaped it is [docs/lessons-from-rusty-chess-v1.md](docs/lessons-from-rusty-chess-v1.md).
The repository rules
are in the [repository conventions](https://github.com/jwallace145/repository-template/blob/main/repository-conventions.md), restated for
contributors in [CONTRIBUTING.md](CONTRIBUTING.md) and for coding agents in
[AGENTS.md](AGENTS.md).

## Layout

| Path | Contents |
| --- | --- |
| `src/` | The `braveheart` binary (root package): UCI protocol, `bench` and `perft` subcommands. |
| `crates/core` | Board, bitboards, magics, move generation, Zobrist hashing, make/unmake. No dependencies. |
| `crates/search` | Search, transposition table, move picker, evaluation (the embedded NNUE net, and PeSTO), `SearchStats` counters and the JSON lines telemetry writer. |
| `crates/tools` | Perft suite runner, the performance gate, bench, speed and stats reports, the openings generator, SPRT, match and rating reports, and `nnue-scale`, which measures what a net values each piece at ([docs/nnue.md](docs/nnue.md#the-evaluations-units)). |
| `ci/` | The work the dedicated instances run: `strength-task.sh` (SPRT) and `match-task.sh` (Match). |
| `tests/` | UCI smoke tests, the perft suite `perft.epd`, mate-in-N positions. |
| `perf-targets.toml` | The performance bars `perf-gate` enforces; they only go up. |
| `benches/` | Criterion microbenchmarks for the binary; crates keep their own under `crates/*/benches`. |

## Building and running

```sh
cargo build --release
./target/release/braveheart                 # UCI: position, go depth|movetime|wtime.., stop, quit
./target/release/braveheart bench           # signature and NPS over 50 fixed positions
./target/release/braveheart bench --depth 10 --hash 16 --out bench.jsonl
./target/release/braveheart perft --depth 5 --fen "<fen>"   # split by root move
./target/release/braveheart perft --suite tests/perft.epd --out perft.jsonl
```

Local builds target the host CPU (`.cargo/config.toml`); release CI builds a
portable baseline instead.

## Measuring

Three layers are measured separately because each one fails differently.

| Layer | Command | Output |
| --- | --- | --- |
| Move generation speed | `braveheart perft --suite tests/perft.epd` | every count in the suite, and perft NPS (bulk counting) |
| Search speed | `braveheart bench` | the node **signature** and NPS |
| Search efficiency | `cargo build --release --features stats`, then `bench` | fail-high-first rate, EBF, TT hit rate, qsearch share, cutoffs by move-picker stage |
| Strength | the `sprt` label on a PR | Elo against the PR's base, timed and at fixed nodes per move (both gate), on standardized runners |
| Head-to-head | the `Match` workflow, by hand | Elo between any two refs at any time control or node budget, a fixed number of games |

The `stats` feature compiles the counters in; without it every `stat!` in
the search expands to nothing. `bench --out FILE` writes JSON lines: one
`summary` record per position and one `bench` record at the end, each with a
schema version `v`. CI runs bench on every PR and uploads `bench.jsonl` and
`stats.jsonl` as artifacts; if the PR description carries a `Bench: <nodes>`
line, CI checks that it matches.

Each layer also has a coverage suite that only grows (`tests/perft.epd`
for move generation, the bench positions for search) and a performance bar
in `perf-targets.toml` that only rises. `perf-gate` compares a run's JSON
lines to the bars; CI reports speed bars on every PR, enforces them on
release PRs, and always enforces the efficiency bars. Every PR gets a
bench comment, refreshed on each push, with the deltas against the latest
release, whose telemetry is attached to the GitHub release as assets. See
[CONTRIBUTING.md](CONTRIBUTING.md).

Every release's numbers, with the full history, are in [docs/benchmarks.md](docs/benchmarks.md).
Every NNUE net, with its data, parameters and results, is in
braveheart-trainer's [nets.md](https://github.com/Braveheart-Chess/braveheart-trainer/blob/main/docs/nets.md),
with a diagram of every net's architecture in its
[nnue-architectures/](https://github.com/Braveheart-Chess/braveheart-trainer/blob/main/docs/nnue-architectures/README.md);
every self-play dataset is in braveheart-selfplay's
[datasets.md](https://github.com/Braveheart-Chess/braveheart-selfplay/blob/main/docs/datasets.md).
To watch the engine play other engines locally, see
[tools/engine-battle/README.md](tools/engine-battle/README.md): EngineBattle,
with Stockfish and Viridithas as opponents, for viewing rather than measuring.

To play it or watch it on Lichess, see
[braveheart-lichess-bot](https://github.com/Braveheart-Chess/braveheart-lichess-bot),
the bot's own repository. The logo, the avatar, the favicons and the social
preview live in [braveheart-branding](https://github.com/Braveheart-Chess/braveheart-branding).

## Phases

| Phase | Gate |
| --- | --- |
| 0. Scaffolding and telemetry skeleton | CI uploads `bench.jsonl` on every PR; Release Please cuts `v0.1.0` |
| 1. Board and move generation | Full perft suite passes; bulk perft ≥ 100M NPS |
| 2. Baseline search | Legal games under fastchess, stable bench signature, stats on every search |
| 3. Move ordering and transposition table | Fail-high-first ≥ 90%, EBF ≤ 3 |
| 4. Selectivity | EBF ≈ 2, each technique gains Elo under SPRT |
| 5. First release | `brew install` works; publicly rated |
| 6. NNUE | NNUE build beats PeSTO build by 100+ Elo |
| 7. Extensions | Lazy SMP, HTTP API |

## License

The code is MIT licensed; see [LICENSE](LICENSE).

The logo is not part of this repository and is **not** covered by the MIT
licence. The artwork is copyright © the artist, used by this project under
licence, with all rights reserved; it lives in
[braveheart-branding](https://github.com/Braveheart-Chess/braveheart-branding),
whose [LICENSE](https://github.com/Braveheart-Chess/braveheart-branding/blob/main/LICENSE)
has the terms. Forks and redistributions must not use it.
