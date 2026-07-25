# SpecRoster Actions 🐓

GitHub Actions for [SpecRoster](https://github.com/SpecRoster) — the
orchestration control plane for test execution. These are thin shims that
run inside **your** CI: they authenticate to SpecRoster via GitHub Actions
OIDC (no API keys to manage), fetch the **Blast Radius** selection manifest,
run the selected tests, and report results back. All product logic lives
server-side.

Supported runners: **pytest** (Python) and **gotest** (Go). More coming —
.NET is next.

## `select` — run only the tests your change can break

Add to your existing `pull_request` workflow:

```yaml
permissions:
  contents: read
  id-token: write          # OIDC exchange with SpecRoster

steps:
  - uses: actions/checkout@v4
    with: { fetch-depth: 0 }    # the diff needs the PR base
  - uses: SpecRoster/actions/select@v1
    with:
      api-url: https://your-specroster-host
      src-dir: src/mypkg        # pytest only
      # runner: gotest          # for Go projects
```

In **shadow mode** (the default for new installs) the full suite still runs
— SpecRoster reports what it *would* have selected and its measured
would-be miss-rate before anything is ever skipped.

Inputs: `api-url` (required) · `runner` (`pytest`|`gotest`) · `src-dir` ·
`test-dir` · `trigger` (`pr`|`nightly`) · `budget-ms` · `core-tests` ·
`pytest-args` · `junit-path`.

## `coverage` — the nightly snapshot that powers selection

```yaml
on:
  schedule: [{ cron: '0 7 * * *' }]
  workflow_dispatch:

steps:
  - uses: actions/checkout@v4
  - uses: SpecRoster/actions/coverage@v1
    with:
      api-url: https://your-specroster-host
      # runner: gotest
```

Runs the full suite with per-test coverage and uploads the snapshot the
reverse index is built from. This is also the full-suite safety net that
makes selection safe: a selection miss costs hours of latency, never a
shipped bug.

## Reliability — SpecRoster can never fail your build

**`select` fails open.** If SpecRoster is unreachable, errors, or rejects the
request, the action does **not** fail your job. It prints a warning saying what
happened, **runs your full test suite**, and skips reporting that run. Your
build then passes or fails purely on your own tests — exactly as it would if
you had never installed us.

That's a deliberate design property, not a fallback we hope you never hit.
Test selection is *advisory*: it only ever narrows what runs, so the safe
answer to "we can't work out what to select" is always "run everything." We are
not a gate, and an outage on our side should cost you a few minutes of CI time,
never a red X on your pull request.

The warning distinguishes the two cases, because they need different responses:

| What you see | Meaning |
|---|---|
| `HTTP 401` / `403` | The SpecRoster App isn't installed on this repo, or is suspended — **your setup**, worth fixing |
| `could not obtain a GitHub OIDC token` | The job is missing `permissions: id-token: write` — **your setup** |
| `HTTP 000` or `5xx` | SpecRoster was unreachable or erroring — **ours**, usually transient |

Before failing open, calls **retry automatically** on HTTP 429 (our per-IP rate
limit) and transient 5xx, honoring `Retry-After` — so a wide sharded matrix
egressing through a single runner IP rides out the limit rather than tripping
over it. You normally never see any of this.

Your `.specroster.yml` config is capped at **64 KiB** (a real config is a few
KB), and the whole request body — which carries your change set — at **1 MiB**.

---

Source of truth for these actions lives in the main SpecRoster repository;
this repo is a published mirror. Issues → the SpecRoster org.

## Installing the coverage collectors

The non-pytest `coverage` runners need a `specroster-*cover` collector
binary on the runner's PATH (pytest's collection is plain coverage.py and
needs nothing extra). Grab the latest from this repo's Releases:

```yaml
- name: Install SpecRoster collector
  run: |
    curl -fsSL -o /usr/local/bin/specroster-dotnetcover \
      https://github.com/SpecRoster/actions/releases/latest/download/specroster-dotnetcover_linux_amd64
    chmod +x /usr/local/bin/specroster-dotnetcover
```

Available: `specroster-gocover` (Go projects can also `go run` it),
`specroster-dotnetcover`, `specroster-jestcover`, `specroster-jvmcover`,
`specroster-rbcover`, `specroster-phpcover` — each for
`linux`/`darwin` × `amd64`/`arm64`, with `SHA256SUMS` alongside.
