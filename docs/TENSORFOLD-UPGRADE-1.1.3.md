# TensorFold 1.1.3 compatibility check

Status checked 2026-10-24. This branch records an upgrade attempt; it does not change the production engine pin.

## Versions checked

- BLACKFOLD recipe: upstream `Blackwellboy/DeepSeek-V4.1-Flash-1017-BLACKFOLD-TP2` at `d905cf0b9d126da32b9f66907454c1625d78a378`.
- Current runnable engine recipe: `bertholomus/TensorFold` commit `6366d2c585b4f05fe33bfe5cbdb33ea15264841a`, which contains its `dsv41` CUDA lane family and TP2 runner.
- Latest TensorFold release: `v1.1.3`, tag `9c3b5ecacf005b70f726e1d3e036e884e53a69df`, resolving to release commit `36415df4f76e4458e82d0fa729ecb28e287ebf38`.
- TensorFold `main` at check time: `1b44f319fb8d605ad905c3eb594e4deea45c3026`.

## Why a direct upgrade is blocked

TensorFold 1.1.3 contains DeepSeek-V4.1 CUDA components, but its own README and `docs/known-issues.md` say release binaries do not serve DeepSeek checkpoints until [PR #827](https://github.com/ashhart/TensorFold/pull/827)'s whole-model gate passes. PR #827 remains open. The PR is a separate lane backend on the 1.1.0 core and explicitly has drafts disabled; it is not the BLACKFOLD two-node CUDA runner.

The recipe's engine is not a small patch on upstream 1.1.3. It relies on the `dsv41` family, TP2 runner and CUDA implementation from Bertholomus's fork. Upstream 1.1.3 has a differently organized `deepseek_v41` family and lane core. Replacing the pin with upstream 1.1.3 would therefore remove the tested serving path, and a release build refuses this checkpoint.

## Patch results

Checked each existing patch with `git apply --check` against upstream tag `v1.1.3`:

| Patch | Result | Reason |
|---|---|---|
| `0001-native-stream-tool-call-whitespace.patch` | Already covered upstream; patch does not apply | The current server has whitespace handling in `zig/src/server/reply_text.zig` and stream tests for Chat, Messages and Responses. |
| `0002-dsv41-fixed-solo-head-overhead.patch` | Does not apply | The patch targets the old `dsv41` family and pre-1.1.1 lane configuration. Upstream 1.1.3 has a different planner/config layout; no equivalent opt-in solo-head cost control was identified in the DeepSeek family. |
| `0003-dsv41-copy-windows-off.patch` | Does not apply | The patch targets the old `dsv41` family and old lane config. The 1.1.3 lane core has different copy controls; equivalence for BLACKFOLD's CUDA draft-head path is unproven. |

Patch 0001 should not be carried forward. Patches 0002 and 0003 must not be mechanically rebased: they need a CUDA backend that exists in the upstream serving path and fresh correctness/performance evidence.

## Decision

Keep the known-working `6366d2c` based build for production. Do not claim TensorFold 1.1.3 is a drop-in upgrade and do not deploy it for BLACKFOLD until the upstream DeepSeek whole-model gate is merged and a two-GB10 TP2 build passes load, inference, resumed-chat, and multi-request checks. This is a compatibility finding, not a performance comparison.

## Reproduction

```bash
git clone --depth 1 --branch v1.1.3 https://github.com/ashhart/TensorFold.git TensorFold-v1.1.3
for patch in patches/000*.patch; do git -C TensorFold-v1.1.3 apply --check "$PWD/$patch"; done
```
