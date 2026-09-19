# Latency profiling

Tooling to measure, attribute and reduce the time between a user finishing a
sentence and the agent starting to speak.

| file | what it is |
| --- | --- |
| [`design.md`](design.md) | why the profiler is built this way — the clock model, why no clock synchronisation is needed, what it deliberately does not measure |
| [`lever-analysis.md`](lever-analysis.md) | **start here.** Every latency lever, measured: what moved, what did not, and what turned out to be actively harmful |
| [`baseline-2026-09-17-turn-taking.md`](baseline-2026-09-17-turn-taking.md) | baseline across six turn shapes, n=36 |
| [`baseline-2026-09-17-smoke.md`](baseline-2026-09-17-smoke.md) | first end-to-end run, single fixture |
| `fixtures/` | benchmark manifests (the audio lives outside the repo) |

## The headline

End-to-end p50 is **~1550 ms**, and **gradbot's own orchestration accounts for
almost none of it**. Every internal reaction gap measures 0.0–1.2 ms:

| gap | p50 |
| --- | --- |
| `vad_eot` → `flush.begin` | 0.1 ms |
| `flush.end` → `llm.push.begin` | 0.0 ms |
| `tts.first_audio` → `out.first_audio` | 1.2 ms |

The one exception was the first LLM token waiting in a buffer for a word
boundary before TTS could start — 45 ms at p50, 297 ms at p90, affecting 48% of
turns. That is now fixed; the gap measures 0.0 ms.

What remains is STT end-of-turn detection (~610 ms) and TTS synthesis (~380 ms),
both server-side. See `lever-analysis.md` for the levers that were tried and
closed, including two that looked promising and were not.

## How it works

Two halves that never share a clock:

- **Server side** — `gradbot_lib/src/trace.rs` records spans against one
  monotonic `Instant` per session, written as JSONL to a side channel (never
  over the client socket, which would perturb the thing being measured).
  Enabled by setting `trace_dir` in the server config; off by default and free
  when off.
- **Client side** — `examples/gradbot-bench.rs` plays fixture audio at real-time
  pace and records when it sent the user's last speech sample and when the first
  agent audio came back.

They join on **cumulative input sample index**, which both sides count
independently. No clock synchronisation, so no clock-skew error. See
`design.md`.

## Running it

```bash
# 1. an OpenAI-compatible LLM endpoint. Note the /v1 — gradbot appends
#    /chat/completions directly, and omitting it yields a 404 at the first turn.
export LLM_BASE_URL="http://<host>:<port>/v1"
export GRADIUM_API_KEY=...

# 2. a server with tracing on
#    (set trace_dir in your config; see configs/gradbot.toml)
cargo run --release --bin gradbot_bin -- --config <your.toml>

# 3. the benchmark
cargo run --release --example gradbot-bench -- \
  --url ws://127.0.0.1:8123/v1/realtime \
  --manifest docs/latency/fixtures/turn_taking.json \
  --repetitions 3 \
  --trace-dir <the same trace_dir> \
  --out-marks /tmp/marks.json \
  --out-report /tmp/report.md
```

The report gives end-to-end percentiles per fixture category, a per-turn
waterfall, the `detection_lag` / `server_internal` / `network` split, and the
turn-taking quality metrics.

## Reading the numbers honestly

The report is built to make misleading output hard, because the failure mode
here is not a crash — it is a plausible wrong number.

- **Every aggregate states its `n`**, and turns that could not be measured are
  listed explicitly rather than dropped.
- **A missing measurement is never a zero.** `detection_lag` is `Option<f64>`;
  a missing anchor is an error naming the span, not a silent 0.
- **`premature_cut_rate` is not optional reading.** Endpointing latency trades
  directly against interrupting the user. A latency number without it is half a
  result.
- **`premature_cut_rate` cannot see everything.** Shortening the flush truncates
  the transcript reaching the LLM — the agent answers a half-heard sentence
  while still speaking only after the user stopped. Compare transcript
  characters per turn across runs (same audio in) to catch it.
- **The LLM row is labelled `measured-local`** when the LLM is co-located. Its
  network cost is then ~0, unlike production.

## Gotchas

- **Fixture boundaries must be ground truth, not inferred.** The fixtures are
  single synthesized turns with trailing silence trimmed, so `speech_end_sample`
  is the file length by construction. Running a VAD to find turn boundaries
  would mean using a VAD to define the reference for measuring VAD latency.
- **Synthetic fixtures flatter the VAD.** TTS speech is cleaner than real
  speech, so endpointing numbers here are a floor.
- **One benchmark run is not one session.** gradbot restarts the STT stream on
  silence timeouts and `reset_asr`, and opens a fresh TTS stream per turn, so a
  3-repetition run over 12 turns is roughly 3 STT plus ~39 TTS streams. Against
  a shared session cap, prefer 2 repetitions.
- **Cold builds** need `CMAKE_POLICY_VERSION_MINIMUM=3.5` (audiopus_sys vs
  CMake 4) and `LIBRARY_PATH=/usr/lib/python3.12/config-3.12-x86_64-linux-gnu`
  (gradbot_py linking). A warm `target/` hides both.
