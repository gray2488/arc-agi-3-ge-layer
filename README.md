# GoA ARC-AGI-3 GE Layer

Run-7 is the 46-cell Kaggle notebook containing the same 36-cell agent core as Run-21_3, plus the GE1–GE4 grid text-encoding and integration layer in cells 20–29.

## What the agent does

The notebook executes top to bottom:

1. **Preflight:** checks paths, GPU and host resources, parquet inputs, and disk capacity before expensive work.
2. **Offline setup:** installs the pinned wheel set from the offline bundle and prepares the games cache.
3. **Model serving:** launches two local vLLM endpoints on one GPU and waits for their health checks.
4. **Core module assembly:** writes the same core agent package used by Run-21_3.
5. **GE layer assembly:** writes the GE1 wire encoder/decoder and validation layer, GE2 frame/state adapters, GE3 policy and candidate selector, GE4 integration and gate framework, the GE4 test module, and the integration cell that wires the layers together.
6. **Certification and conformance:** performs the core pinned-file, build, template, preflight, coder-solver, and framework-loading checks.
7. **Game loop:** opens the offline Arcade environment, checks the expected game count, performs a smoke initialization, then runs the configured game loop and records run telemetry.

At each agent step, the shared core settles the frame and computes its anchor/checksum; extracts objects; supplies structured state to the context manager; checks for loops; asks the reasoner for a plan; optionally invokes the coder-solver; and dispatches the selected action.

## Runtime

Two separate vLLM servers are configured on one GPU:

| Role | Model | Local endpoint | Context length | GPU memory utilization |
|---|---|---|---:|---:|
| Reasoner | `qwen3.8-27b` (FP8) | `http://127.0.0.1:8000` | 16,384 | 0.40 |
| Coder | `qwen3-coder-30b-a3b-instruct` | `http://127.0.0.1:8001` | 16,384 | 0.42 |

Both endpoints use the OpenAI-compatible chat-completions interface. The notebook transport routes requests by role.

## CEGIS loop and sandbox

When the reasoner decides that a task needs a program, the agent builds a typed task specification and sends it to the coder model. The returned Python candidate is passed to `sandbox_v6.run_candidate()`. If the candidate produces the required board change, it can be admitted. Otherwise, the replay verifier builds a counterexample, the CEGIS taxonomy classifies the failure, the router selects a repair budget, and the solver may request a repair or retire the candidate. The configured repair-attempt cap is 6 per game.

The sandbox parses candidate code with Python's AST, checks imports against an allowlist (including a second module check in the child wrapper), constructs restricted builtins, and executes the candidate in a separate subprocess with resource limits. Its entry point returns a structured result rather than raising candidate errors into the caller.

## GE1–GE4 layer

- **GE1 — wire:** RLE encoding/decoding, frame registry, evidence logging, and validation.
- **GE2 — adapters:** frame/state adaptation, census validation, and SDK vocabulary binding.
- **GE3 — selector:** policy, cost bundles, candidate selection, and dispatch selection.
- **GE4 — integration:** runtime configuration, upstream loading, cap checks, telemetry, and trust verification.

The GE4 integration source includes `_ed25519_verify`, `_verify_trust_record`, and `_verify_owner_attestation`. These routines verify Ed25519 signatures and trust/owner-attestation records; this is cryptographic trust machinery, not merely a grid formatter.

**Trust-root configuration caveat:** the notebook source sets `TRUST_ROOT_PUBKEY_HEX` to a zero-filled placeholder and defines `/etc/ge4/trust_root.pub` as the trust-root file path. A deployment must provision its real trust root or rebuild with its own pinned release root. The GE4 gate-test module calls `Ed25519PrivateKey.generate()` to create temporary signing keys in memory for test fixtures; these are generated at test runtime, not embedded key material.

The GE test module exposes `_run_all()`, which exits non-zero on failure. Its presence does not, by itself, establish that the tests have been executed successfully.

## Gate chain

The shared core gate chain runs before the game loop:

1. **Certified file repair:** compares module bytes with their MD5 pins, repairs trailing-newline drift where permitted, and refuses on a mismatch.
2. **R-13 build check:** builds the single-file artifact and checks the embedded certified source set.
3. **Template conformance:** sets the conformance flag only after the checks run; a skipped check is not treated as a pass.
4. **Frozen preflight runbook:** runs eight CS checks covering schema integrity, sandbox restrictions, benign execution, inference call logging and caps, attempt arithmetic, solver-to-sandbox wiring, and coder-parameter wiring.
5. **Frame probe:** checks that the framework Agent can be loaded directly and that the MyAgent main loop is available.
6. **MD5 register:** provides two views of the pinned file set.

The GE layer is assembled before this chain. Its integration cell wires GE1–GE4 together, and the GE4 test module's `_run_all()` is the explicit test entry point.

## Budget constants

| Constant | Value |
|---|---:|
| `EXPECTED_GAMES` | 25 |
| `MAX_ACTIONS` | 80 |
| `MAX_MINUTES_PER_GAME` | 17.0 |
| `GLOBAL_TIME_BUDGET_H` | 7.5 |
| `MAX_MODEL_LEN` | 16,384 |
| `REASONER_ANSWER_CAP` | 200 |
| `REASONER_THINKING_BUDGET` | 1,000 |
| `REASONER_DEEP_THINKING_BUDGET` | 2,000 |
| `REASONER_DEEP_MAX_TOKENS` | 6,144 |
| `CODER_CODE_CAP` | 2,048 |
| `CODER_ATTEMPTS_PER_GAME_CAP` | 6 |
| `MAX_TEXT_CHARS` | 4,000 |
| `MAX_REGION_CELLS` | 576 |
| `MAX_DELTA_PACKET_CHARS` | 4,000 |
| `_MAX_OBS_OBJECTS` | 24 |
| `_MAX_DEAD_SIGS_PER_LEVEL` | 200 |
| `GE_CAP` | 2,500 (research model-test cap; not the production H limit) |

## What this notebook is not

Neither Run-21_3 nor Run-7 is the notebook that cleared levels. The scoring runs used agent version **1.0.23 or later**, and those scoring-run notebooks were never archived. Neither notebook contains a delegation brief or a coordination protocol between the two models. These artifacts must not be represented as production-ready or as competition winners.

## Scope

This repository contains Run-7 with its additional GE1–GE4 layer. It makes no performance claim beyond what is recorded in the notebook itself.
