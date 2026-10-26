# Remote Liquid model: robot server + model server

Split the existing closed loop in two, without changing what the
robot side trusts:

```
robot server (VPS / sim machine)              model server (molab / GPU VPS)
  simulator + GRU policy + GaitPlanner          LiquidAI/d1-3B-w8a8 only
  SafetySupervisor + MissionTracker             model_server.py
  decision_schema.validate                        POST /system_one
  make_decide(remote.ask, ...)  ── HTTPS ──►     model.system_one(state, questions)
  (RemoteDecider.ask, stdlib only)              ◄── {"answers": {...}}
```

What stays on the robot server: the simulator, the actor checkpoint,
validation, stale-answer rejection, and safety. What moves: only the
one `system_one` forward pass. The robot server therefore needs no
torch-CUDA Liquid stack, no `transformers` / `torchao`, and no
Hugging Face download for the Liquid model. It still needs its own
sim/policy stack (and the actor checkpoint -- so use the full
`feature/liquid-ai-runtime` branch, **not** `no-weights`, for a real run).

## Files

- `src/contact_rl/runtime/remote_decider.py` -- drop into
  `src/contact_rl/runtime/` on the robot side. `RemoteDecider.ask`
  has exactly the `LiquidDecider.ask` contract: `(state, questions)`
  in, `answers` dict out. Failures raise `RemoteUnavailable`, to be
  treated like a local model failure (keep the last valid command).
- `src/contact_rl/runtime/model_server.py` -- drop into the same
  folder on the model side. Stdlib-only HTTP server (no FastAPI
  dependency): `GET /health`, `POST /system_one`, Bearer auth,
  one-inference-at-a-time lock. The real model is loaded **before**
  it starts listening, so the first request never pays model-load
  time inside the robot's 2-second tick deadline.
- `rollout_remote.patch` -- adds `--decider remote` (+ `--api-url`,
  `--api-timeout` default 1.8 s, `--api-key`) to `rollout.py`.
  Apply from the repo root with `patch -p1 < rollout_remote.patch`.
  Verified: it applies cleanly to `rollout.py` at
  `feature/liquid-ai-runtime` tip `1e62e8f` and the patched file
  compiles.
- `tests/test_remote_e2e.py` -- transport test with the mock model
  (see below). It does **not** test the real d1 model, the GRU
  policy, or the simulator.

## Local test (no GPU, mock model) -- verified 2026-10-26

Terminal 1 -- model server:

```bash
cd .  # repo root
LIQUID_API_KEY=test-only-local-key PYTHONPATH=src \
  python3 -m contact_rl.runtime.model_server --host 127.0.0.1 --port 8001 --mock
```

Terminal 2 -- the same test the author ran (server + client, mock):

```bash
cd .  # repo root
python3 tests/test_remote_e2e.py
# health ok, mission/tick/one-shot answers, wrong key -> 401,
# unauthenticated -> 401, 3 asks in 0.004 s (tick deadline 2.0 s)
```

## Real deployment

Model server (GPU machine, model already in the repo's dependency set):

```bash
cd rl-ai   # feature/liquid-ai-runtime, with the two files above added
export LIQUID_API_KEY='<generate a long random token, never commit it>'
uv run --with "transformers>=5.19" --with "torchao>=0.18" --with pillow \
  python -m contact_rl.runtime.model_server --host 127.0.0.1 --port 8001
# health check from the robot server:
curl http://127.0.0.1:8001/health   # from the model machine, or via the proxy URL
```

Robot server (sim machine, same branch + patch applied):

```bash
cd rl-ai
export LIQUID_API_KEY='<the same token>'
uv run python -m contact_rl.runtime.rollout \
  --decider remote --api-url https://model.example.com \
  --checkpoint best --device cuda:0 \
  --instruction "Move toward the target and turn left." --target 2.0 0.0
```

## Security and reliability -- read before going public

- **No TLS in this server.** It speaks plain HTTP. Put it behind a
  TLS reverse proxy (nginx / Caddy) and bind `127.0.0.1` when the
  proxy is on the same machine. A Bearer token over plain HTTP is
  not a secret.
- **Token handling.** The token lives only in `LIQUID_API_KEY` (or
  `--api-key` at process start). Never put it in a notebook, a
  script, or a commit -- a molab notebook synced from GitHub has its
  code visible. Rotate it if it ever leaks.
- **molab is unproven as the model host.** Its docs describe sharing
  notebooks/apps, not exposing a custom port, and sessions end after
  12 hours / 90 minutes idle. Before relying on molab, run the model
  server there and `curl` it from the robot server. If the port is
  not reachable, run this same file on a GPU VPS instead.
- **Failure behaviour is intentional.** HTTP timeout is 1.8 s by
  default, under the 2.0 s tick deadline. A late or failed answer is
  discarded, the last valid command holds, and after repeated
  failures the runtime's safety logic stops the mission. Do not
  "fix" this by retrying inside a tick or by raising the timeout
  above the deadline.
- **One model, one inference at a time.** The server serialises
  calls with a lock. A second robot queues behind the first; it does
  not get a parallel model.

## Status

Built and transport-tested locally 2026-10-26 (mock model only),
then pushed to branch `remote-liquid-decider` (from
`feature/liquid-ai-runtime` tip `1e62e8f`) at the user's explicit
confirmation. Real-model, policy and sim runs over this transport
are NOT yet tested; see the security and reliability section above.
