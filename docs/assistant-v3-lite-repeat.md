# Run Assistant V3 against Splunk with SREGym-Lite

This fork's wrapper runs one or all 21 Lite cases sequentially: inject a fault,
send metrics/traces/logs/events to your Splunk org, check representative causal
telemetry before launching Assistant V3, collect its diagnosis and ATIF trace,
grade it with the unchanged benchmark judge, and package the results by case.
It uses the `svelte` deployment profile and a time-window/symptom prompt, so
the scores are a **local pilot, not leaderboard-comparable**. The wrapper does
not provision Docker, Kind, the Splunk org, or the separate Assistant server.

## 1. Prepare the benchmark checkout

Use `khubis/SREGym` (the fork with this Splunk adapter), not
`SREGym/SREGym` upstream (which has no Splunk adapter). For a fresh
SSH-authenticated clone after this documentation is merged:

```bash
git clone --branch main --recurse-submodules git@github.com:khubis/SREGym.git
cd SREGym
```

From that checkout (or an existing checkout on the fork's `main` branch):

```bash
git submodule update --init --recursive
uv sync --group dev
docker info >/dev/null
kubectl get nodes                 # all nodes must be Ready
bash docker/agents/build.sh      # builds sregym-agent-base:latest from this checkout
```

If there is no Kind cluster yet, follow [Kind setup](../kind/README.md) first.
The image must be rebuilt after changing containerized code. Do not use the
published SREGym agent image for this fork's Assistant adapter.

## 2. Configure the Splunk target and judge

Copy the [fill-in template](../.env.splunk-lite.example) to the gitignored
`.env` in this checkout, fill the required blanks (except the JWT generated in
step 3), and load it in the **same shell**
that will run the wrapper. Do not overwrite an existing `.env` or commit secrets.

```bash
cp -n .env.splunk-lite.example .env
# Edit .env; replace the Splunk and judge blanks. The JWT comes from step 3.
chmod 600 .env
set -a
source .env
set +a
```

| Variables | Where to get them / purpose |
|---|---|
| `SYNTHETIC_SF_TOKEN`, `SYNTHETIC_REALM`, `SYNTHETIC_ORG_ID`, `SYNTHETIC_USER_ID` | Query token and identity for the **same** synthetic Splunk Observability org. |
| `SYNTHETIC_SPLUNK_ACCESS_TOKEN` | Org access token with `INGEST` scope for the collector; this is not the SF query token. |
| `SPLUNK_HOST`, `SPLUNK_HEC_PORT`, `SPLUNK_HEC_TOKEN`, `SPLUNK_LOGS_CONNECTION_ID` | Logs HEC destination and the Logs Observer connection pointing to it. The host is a hostname only. |
| `ASSISTANT_V3_URL`, `ASSISTANT_V3_AUTH_TOKEN` | Running v3 server URL and its raw bearer JWT (generate below; do not include `Bearer ` in the value). |
| `JUDGE_API_BASE`, `JUDGE_API_KEY` | Authorized Azure endpoint and key for the pinned `azure/gpt-5.6-luna` judge; these can match the Assistant server's `OPENAI_ENDPOINT` and `OPENAI_API_KEY`. |

`SPLUNK_LOGS_CONNECTION_ID` is the **ID of the Logs Observer connection in
this org**, not the HEC token. That connection must point to the same HEC
destination that receives this run's logs and Kubernetes events.

The wrapper does **not** source `.env` automatically. With `--credentials
synthetic`, it maps the five `SYNTHETIC_*` values to canonical `SF_TOKEN`,
`SPLUNK_O11Y_INGEST_TOKEN`, `SFX_REALM`, `ORG_ID`, and `USER_ID` only for the
benchmark child. If your shell also has one of those canonical names set to a
different value, the wrapper fails rather than risk using two orgs; unset the
stale canonical value. `SPLUNK_HEC_INDEX` is optional (`main` by default) and
must be permitted by the HEC token. Never print or commit token values.

## 3. Start Assistant V3 separately

This is the **LangChain Deep Agents V3** server from a sibling `assistant`
checkout, not another Assistant implementation. Follow its
`README.md` / `.env.v3.sample` to configure and start it. For a local server,
make a private Assistant env file from that sample; set its `SF_TOKEN`,
`SFX_REALM`, `ORG_ID`, and `USER_ID` to the **same values** as the synthetic
variables above, and configure its Luna model endpoint. In a separate terminal:

```bash
cd ../assistant
cp -n .env.v3.sample .env.sregym
# Edit .env.sregym to use the same synthetic org and your Assistant model settings.
set -a
source .env.sregym
set +a
python3 scripts/local_auth_jwt.py >/dev/null  # auth preflight; refresh SF_TOKEN if HTTP 401
ASSISTANT_V3_ENV_FILE=.env.sregym make run-local-server-v3
```

The command above assumes you created `assistant/.env.sregym`; the Assistant
repo ignores `.env*`. It listens on `http://127.0.0.1:8903` by default.
Specifying `ASSISTANT_V3_ENV_FILE` prevents an unrelated `assistant/.env` from
silently replacing the synthetic org. Use a dedicated local eval database if
your Assistant development database has an incompatible schema; see the
[full guide](assistant-v3-evaluations.md).

Back in the SREGym shell, mint the JWT from the **same synthetic identity**.
The Assistant helper refreshes its private `assistant/.local/local_auth.jwt`
cache when needed. Command substitution keeps the token out of terminal output:

```bash
export ASSISTANT_V3_AUTH_TOKEN="$(
  cd ../assistant &&
  SF_TOKEN="$SYNTHETIC_SF_TOKEN" SFX_REALM="$SYNTHETIC_REALM" \
    ORG_ID="$SYNTHETIC_ORG_ID" USER_ID="$SYNTHETIC_USER_ID" \
    python3 scripts/local_auth_jwt.py
)"
```

If either JWT command returns HTTP 401, refresh the synthetic **SF query token**
in both private env files before proceeding. Replacing the HEC token or the
collector's INGEST token will not fix this authentication failure.

The runner checks that it can authenticate to Assistant before injecting a
fault, but cannot inspect which org an already-running Assistant server uses.
Confirm its startup configuration matches this file. Refresh the JWT if it
expires during a long suite.

## 4. Smoke-test one case, then run the suite

This is one command **after** the setup above, not a provisioning command:

```bash
uv run --no-sync python -m sregym.results.assistant_v3_lite_repeat run \
  --credentials synthetic \
  --problem cronjob_sidecar_blocks_completion_hotel_reservation \
  --agent-image sregym-agent-base:latest \
  --output results/reproductions/cronjob-smoke
```

After reviewing `results/reproductions/cronjob-smoke/summary.md`, run the suite:

```bash
uv run --no-sync python -m sregym.results.assistant_v3_lite_repeat run \
  --credentials synthetic \
  --agent-image sregym-agent-base:latest \
  --output results/reproductions/lite-21
```

The command runs cases sequentially in separate benchmark child processes,
one attempt each, using the svelte
profile, Assistant V3, GPT-5.6 Luna/medium, the matching API judge, Splunk,
and the reviewed symptom-guided prompt. Assistant execution is limited to
1,200 seconds per case; grading and cleanup can take additional time. A timeout
is an incomplete attempt, never a zero diagnosis score. It reuses the local image rather than
rebuilding it. Rebuild that image with the full-guide command after changing
containerized code; check its image ID before reusing it. The wrapper refuses
to start if the image is absent, another wrapper run holds the lock, fewer than
6 GiB of host memory or 10 GiB of disk are free, or Docker has under 8 GiB
allocated. It also requires Ready nodes in the selected `kubectl` context;
the benchmark runner subsequently checks Assistant authentication before injection.
These gates are rechecked before **every** case. They are *start gates*, not a
guarantee against later pressure; watch Docker/host memory during the first
case. It never prunes resources. On this constrained laptop, the previously
live-tested 4 GiB floor can be explicitly selected for the sequential suite
with `--min-available-gib 4`. Never lower it further. Stop if available RAM
approaches 2 GiB; this is a local override, not the recommended default.
If memory is temporarily below the chosen floor between cases, the wrapper
waits and rechecks for up to five minutes; it does not wait through disk,
Docker, image, or cluster failures and never lowers the floor automatically.

## Recover or review a partial batch

Every attempt is checkpointed in its raw timestamped batch. The wrapper also
writes `campaign.json` in the output folder before launch, records each new
batch there, and updates `summary.md` after each case. Keep raw batches until
the package is reviewed. If a case fails, inspect its saved artifacts, address
the cause, then run the **same command with the same `--output`**. It skips
previously valid cases, retains invalid attempts, and retries only unfinished
cases on that later invocation. It refuses a changed org, realm, Logs
connection, Assistant URL, image tag, case selection, or model configuration;
use a new output folder for a new campaign. If a crash happens before any
case checkpoint, inspect the raw batch and Kubernetes cleanup before retrying.

To rebuild a report from known raw batches without launching a simulation:

```bash
uv run --no-sync python -m sregym.results.assistant_v3_lite_repeat finalize \
  --credentials synthetic \
  --batch results/MMDD_HHMM \
  --output results/reproductions/lite-21
```

To resume the same campaign, repeat the original `run` command:

```bash
uv run --no-sync python -m sregym.results.assistant_v3_lite_repeat run \
  --credentials synthetic \
  --agent-image sregym-agent-base:latest \
  --output results/reproductions/lite-21
```

`run` does not accept manual `--batch` or `--resume-csv` arguments; those are
legacy lower-level runner controls. The wrapper rejects duplicate valid attempts; it
does not silently pick a favorable result. It also refuses a post-run Splunk
query error. A *missing* signal is kept as missing in the saved proof, not
converted into success. Open `summary.md` first: it reports valid benchmark
scores separately from classified invalid or missing attempts, with a mean
whose denominator is only valid attempts. Even an all-invalid batch gets a
summary. `selection.md` is the raw selection record; `by-case/<case-id>/`
contains the prompt, benchmark oracle, saved
pre-agent causal proof, representative post-run metrics/traces/logs/object
queries, final answer, native and ATIF traces, raw judge result, metrics, and
SHA-256 provenance manifest. `verification.md` distinguishes confirmed
telemetry from candidate clues. This is a representative presence and causal
gate, **not** exhaustive ingestion parity. Splunk-visible numeric grading and
independent post-grade oracle audit remain separate review work; the wrapper
does not claim them as complete.

The campaign record contains only a fingerprint of non-secret target and run
settings, not tokens. Saved scored attempts are checked for the selected Logs
connection, `svelte` profile, model, reasoning effort, and judge before reuse.
The authenticated Assistant preflight proves connectivity, **not** the org
configured inside an independently started V3 server. The operator must still
confirm its startup environment uses the intended synthetic org. This is the
remaining identity limitation; the wrapper does not claim automatic server-org
attestation.

The collector delivery audit also requires run-scoped export counters. If a
partial counter sample is missing at close, it waits at most the two-minute
SignalFlow ingest guard and rechecks that same historical interval. A total
query failure, unresolved counter, counter decrease, send failure, or undrained
queue remains invalid. The raw delivery artifact records initial and settled
counter samples so an excluded score can be diagnosed without assuming that
missing accounting means missing logs.

A read token can also be revoked or rotated while a long case runs. An opening
evidence check does not prove that closing queries will still be authorized.
If `delivery_audit.json` has `valid: false`, closing `4xx` signals, and unknown
counter deltas, check the current `SF_TOKEN` (or `SYNTHETIC_SF_TOKEN`) against
SignalFlow and the configured Logs connection before resuming. Keep the score
excluded, refresh the token, and rerun the same command/output after confirming
the Assistant server uses the same org. Do not change the prompt or bypass the
delivery gate to salvage a provisional judge score.

Each valid summary row links directly to its exact answer, benchmark oracle,
unchanged judge CSV, pre-agent verification, and native/ATIF traces. Its
"pre-agent causal check" column is bounded evidence, not proof that every
emitted telemetry item reached Splunk. Invalid rows link to raw attempt
metadata; they have no manufactured score or dossier. Retain the raw batch
until the package is inspected, and use the resume commands above after an
interruption.

This release supports the 21 Lite cases with the `svelte` deployment profile
only. The larger case catalog and standard-profile execution require separate
qualification; neither is implied by this pilot command.

The 2026-10-01 local qualification package is at
`results/reproductions/lite-21-hardened-20260930-v2/summary.md`: 21 valid
cases, none missing, 43.05/100 mean. Case 21's selected attempt has all four
closing signal checks ready, drained collector queues, and zero send/enqueue
failures. Earlier invalid attempts remain in timestamped raw batches, not in
the selected per-case score. This is a local pilot result, not a leaderboard
comparison or proof of exhaustive MELT completeness.

Check the wrapper itself before a campaign:

```bash
uv run --no-sync pytest -q tests/results/test_assistant_v3_lite_repeat.py tests/results/test_lite_case_dossiers.py tests/problems/test_search_rate_retry_collapse.py
uv run --no-sync python -m sregym.results.assistant_v3_lite_repeat --help
```

Do not run the unfiltered `tests/problems` tree against an active Kind cluster:
it includes live integration tests that deploy applications. The focused
wrapper, dossier, and `test_search_rate_retry_collapse.py` tests above are
non-live. If a case reports `cleanup_timeout_after_agent_exit`, its provisional
judge score is excluded; inspect and reconcile only that case's task-owned
cluster resources, then rerun the same campaign command after the underlying
cleanup fault is fixed. The original incomplete attempt remains in its raw batch.
