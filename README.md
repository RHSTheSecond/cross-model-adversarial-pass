# CAP — Cross-model Adversarial Pass

**Runs OpenAI's Codex CLI and a fresh Claude subagent as parallel adversarial reviewers, then adjudicates both.**

CAP is a [Claude Code](https://claude.com/claude-code) skill for the moment before you
ship: a design you're about to build, a PR you're about to merge, a job you're about to
put in production. It spawns two reviewers from **different model families**, each
ordered to refute and attack rather than confirm, and then makes you reconcile what they
found.

An LLM asked *"is this good?"* will rationalize a yes. One ordered to produce the
concrete failure scenario — inputs and state → what breaks — has to engage with the
evidence.

## Why two reviewers

Different model families catch different failure modes, and cross-reviewer agreement is
a real confidence signal. CAP sorts every finding into three buckets and surfaces all
three:

| Bucket | How to weigh it |
|---|---|
| **Both flagged independently** | Highest confidence. Near-automatic accept, *raised* severity. |
| **Codex only** | The independent anchor — different lab, zero shared context. Adjudicate on merits; lean accept when it's `[confirmed]`-tagged. |
| **Claude subagent only** | Same family as your main agent, so it's weaker corroboration — but it has none of your session's context, so it catches what that context blinds the main agent to. |

That last row is the part people underrate. Most of what a main agent misses, it misses
because it spent forty turns convincing itself the design was sound. A fresh subagent
never sat through that.

## Three modes

| Mode | Use it for |
|---|---|
| `DESIGN` | Review a proposed architecture **before** building it |
| `PR` | Evaluate a pull request or diff before merging |
| `PROD` | Go/no-go gate before shipping — demands an explicit **GO / GO-WITH-CONDITIONS / NO-GO**, with automatic NO-GO conditions that aren't judgment calls |

Every mode ends with a mandatory **SIMPLER PATH** section: is there a materially simpler
or more effective way to accomplish the underlying goal? Buy instead of build, use an
existing tool, delete the requirement, take the smaller scope that captures 80% of the
value, pick the boring pattern over the clever one. This section is never allowed to be
dropped in summarization — it's the finding most likely to save you the whole project.

## Install

**Requires:** Claude Code, plus the OpenAI Codex CLI installed and signed in.

```bash
npm i -g @openai/codex              # cross-platform
# macOS alternative: brew install --cask codex   (it's a cask, not a formula —
#                                                 plain `brew install codex` fails on Linux)
codex login
```

**Clone the repo — don't just grab `SKILL.md`.** The skill invokes `bin/cap-bundle`,
which is where every safety guarantee actually lives. A skill file on its own is a
loaded gun with the safety catch left in the box.

```bash
git clone https://github.com/RHSTheSecond/cross-model-adversarial-pass.git
cd cross-model-adversarial-pass
mkdir -p ~/.claude/skills
ln -sn "$PWD/skills/cap" ~/.claude/skills/cap    # symlink so `git pull` keeps you current
head -2 ~/.claude/skills/cap/SKILL.md            # verify: should print `---` then `name: cap`
./bin/cap-bundle --help                          # verify the script runs
tests/run.sh                                     # optional: run the guard regression suite
```

Or install it as a plugin:

```
/plugin marketplace add RHSTheSecond/cross-model-adversarial-pass
/plugin install cross-model-adversarial-pass
```

## Use

In Claude Code, say any of:

```
cap this design
cap PR #12
cap this for prod
/cap
```

The skill picks the mode from context, assembles the input packet, builds a sanitized
bundle, spawns both reviewers in parallel, validates their output, and delivers a
verdict-first report.

## ⚠️ Read this before your first run

**Whatever CAP reviews leaves your machine.** Codex is a cloud service, and `codex exec`
is *agentic* — it reads **every file in its working directory**, not just the ones you
name in the prompt.

`bin/cap-bundle` is the mitigation, and it is deliberately a script rather than
instructions, because instructions failed. It flattens the approved files into one
document and gives Codex an empty working directory, so it isn't handed your repo:

```bash
bin/cap-bundle --dry-run --prompt p.txt --out o.txt -- src/    # see exactly what would go
```

**The payload is one text document, and Codex runs with an empty working directory.**
Your repo is never handed to the agentic reviewer as a directory, so it can't
*accidentally* traverse a symlink, read a `.git` packfile, auto-load an in-tree
`AGENTS.md` as instructions, or enumerate control-character filenames — the exposures
four rounds of review kept finding. The document *file* is byte-for-byte what's sent
(read it, or its sha, before approving — the printed inventory is a filename summary,
not the payload).

**It is not a read jail.** `codex exec -s read-only` blocks writes, not reads — a
reviewer *instructed* to (e.g. by injected text in a reviewed PR) can still read
absolute paths elsewhere and echo them into its review. This is
[demonstrated, not theoretical](tests/probe-read-isolation.sh). The empty cwd removes
the accidental and auto-load exposures; deliberate read-exfil is mitigated only by the
prompt telling the reviewer the material is data. **In PR mode, treat the material as
untrusted** — see [SECURITY.md](SECURITY.md).

```bash
cap-bundle build --doc cap_material.txt -- src/ docs/   # build + gate + write the document
# read the inventory it prints, then:
cap-bundle send --doc cap_material.txt --prompt p.txt --out o.txt   # refuses if the doc changed
```

`build` needs no Codex install and sends nothing; `send` verifies the document's sha and
aborts on any drift — and because the document is one regular text file, its hash
captures 100% of the review *material* (no directory blind spots; the operator's own
prompt is concatenated at send and isn't part of the material hash). A one-shot
`cap-bundle --prompt … --out … -- <paths>` does build-gate-send in a single race-free
call.

What the script guarantees — pinned by [`tests/run.sh`](tests/run.sh) where a test can
reach it. Claims about *Codex's own behavior* were checked by hand against
`codex-cli 0.142.5`: AGENTS.md auto-injection suppression was confirmed on the wire with
canaries, and read isolation was **dis**confirmed — `-s read-only` reads absolute paths
just fine ([`tests/probe-read-isolation.sh`](tests/probe-read-isolation.sh)), which is
why the empty-cwd claim below is scoped to *accidental* traversal, not a read jail:

- **The empty working directory** (`-C <empty-dir> -s read-only`, material inline on
  stdin) means a symlink, `.git` packfile, in-tree `AGENTS.md`, or control-character
  filename in the *source* can't become something Codex *accidentally* reads — those
  artifacts aren't on its cwd. It does **not** stop a reviewer that's instructed to read
  an absolute path elsewhere (read-only permits reads); see the "not a read jail" note
  above and the [probe](tests/probe-read-isolation.sh).
- **Build-time enumeration is still gated**, because the source tree is real before it
  is flattened into the document. `git archive HEAD` emits only tracked, committed
  entries (a gitignored `.env` cannot get in); non-regular entries (symlinks/fifos),
  symlinked `--files` path components, control-character filenames, and `.git`/`.hg`/
  `.svn`/`.bzr` internals all abort or are stripped before inlining. Only text files
  are inlined; binary *contents* are not transmitted — but their **paths and byte sizes
  are** (visible in the inventory), so a revealing filename like `acme-layoff-plan.xlsx`
  still leaves the machine. Exclude such paths if the name itself is sensitive.
- **A scratch `CODEX_HOME` (your auth token and nothing else) plus
  `-c project_doc_max_bytes=0`** stops your `~/.codex/AGENTS.md` — which Codex
  otherwise uploads on every run — from being auto-injected. Verified on the wire via
  Codex's own session transcripts: a canary planted in that file appears in the payload
  without this configuration and is absent with it. (The `--ignore-user-config` and
  `--ignore-rules` flags do **not** suppress AGENTS.md — they suppress `config.toml`
  and execpolicy `.rules`. An earlier draft claimed otherwise; a CAP run on this repo
  caught it.)
- **`-s read-only --ephemeral`**, because a reviewer has no reason to write anything.
- **The script refuses its own artifacts** — if `--prompt`/`--out`/`--doc` would land
  inside the reviewed paths, it stops before writing anything. The check resolves each
  path to its nearest existing ancestor, so it holds even when the file (or its parent
  dir) doesn't exist yet and even through a symlinked parent; a symlink *as* the
  artifact path is rejected outright.
- **The guard runs before any write, and `$OUT` is only touched after it passes** — so
  `--out` pointing at a reviewed source file refuses without truncating or otherwise
  modifying that file (an earlier ordering could have emptied it; it can't now). A
  refused run also never leaves a prior run's review behind to be misread as a pass.
- **The secret scan fails closed** — if the scanner itself errors (a shadowed `grep`, an
  unreadable file), the run aborts rather than reporting "nothing matched."
- **`build`/`send` bind approval to transmission.** `send` verifies the document's
  sha256 and refuses if a byte changed since `build`. The artifact is one regular text
  file, so its hash captures the whole review material — no directory blind spots.

The secret gate is a **blocking smoke test, not a guarantee**. It catches known
credential *formats* (`BEGIN … PRIVATE KEY`, `AKIA…`, `AIza…`, `sk-proj-…`/`sk-ant-…`,
`sk_live_`/`sk_test_`, `whsec_`, `ghp_`, `xox…`, Slack webhook URLs, `user:pass@host`
DSNs) and sensitive *filenames* (`.env`, `.pem`, `id_rsa`, `.npmrc`). It will not
recognize a customer CSV or a credential that doesn't look like one, and its pass
message says exactly that. For files that legitimately contain credential-shaped text
(security docs, test fixtures) there's `--allow <path>` — exact single-path waivers,
printed loudly, only for files you've read. **You are still the reviewer of what goes
in the path list.**

If the material is sensitive enough that even a tracked-files bundle is too much, skip
the Codex half and label the result `single-reviewer CAP — lower confidence`. A degraded
pass that says so is fine. One that pretends to be a full pass is the hazard.

## Why not just `/review` or `codex exec review`?

Fair question — both exist and both are cheaper. The difference is what they're
instructed to do. A reviewer asked to assess quality confirms; CAP's reviewers are
ordered to **refute**, must supply a concrete failure scenario for every finding, must
state what evidence would **disprove** their own Critical claims, and must file a
coverage ledger so silence can't pass for approval. Then two model families have to
disagree in public before anything is called confirmed. And every run ends by asking
whether the thing should exist at all — which no linter will ever ask you.

## Design notes

A few decisions that are load-bearing, in case you fork this:

- **Evidence discipline.** Every finding must be tagged `[confirmed]` / `[inferred]` /
  `[verify]`, and every Critical/High finding must state what evidence would *disprove*
  it. This is what stops an adversarial prompt from producing confident fiction.
- **Fail closed.** If a reviewer's output is missing, truncated, or lacks its SIMPLER
  PATH section, that reviewer is **unavailable** — never a summarized partial, never an
  implied pass.
- **Corroboration is never erased.** If a reviewer independently flags something you
  already suspected, that *raises* severity. It doesn't get dropped as "we knew that."
- **The reviewers are not oracles.** Step 4 requires the main agent to give its own
  agree/disagree-and-why on each finding. Refuting the reviewers is fair game.

- **Safety logic is code, not prose.** Everything a mistake would make dangerous lives in
  `bin/cap-bundle`. Everything requiring judgment — mode choice, adjudication, the
  simplicity challenge — stays in the skill. That line is the main design decision here.
- **The automatic NO-GO conditions must not be overridden by the agent** that produced
  the work — a skill rule, not a code-enforced one (CAP is instructions). It may
  disagree, but the skill directs it to escalate to a human rather than resolve it.

### Provenance

CAP has been run against itself twice. v2's self-review caught the agentic-cwd exposure,
missing fail-closed validation, corroboration-erasing report structure, and adversarial
prompting with no evidence discipline.

The v3 self-review — the first with two reviewers instead of one — returned **NO-GO on
this repo** and found four Criticals that v2's single-reviewer pass had missed, including
one proven by running the command: `cd "$UNSET_VAR"` exits 0 and silently leaves Codex in
your repo root, so the documented "sanitized bundle" wasn't protecting anyone. That
finding is why `bin/cap-bundle` exists at all.

The rewrite was then CAP'd again, and **failed again** — this time because the fix
itself made claims that weren't true (the `--ignore-*` flags don't suppress `AGENTS.md`;
one reviewer proved it by capturing the actual upload) and trusted filesystem semantics
it didn't control (a tracked symlink walked straight through every scan). Round two's
findings are what the structural gate, the scratch `CODEX_HOME`, and the test suite
exist to close. See [`examples/sample-report.md`](examples/sample-report.md) for one of
the actual reports.

Treat this as provenance, not proof. A tool grading its own homework has caught real
bugs every time — and each pass found things the previous one was structurally blind to.
Which is the argument for the second reviewer, not for trusting either one.

## Honest limits

- **Cost and latency are real.** A Codex pass is multi-turn agentic and runs 2–10 minutes
  against a small bundle — longer and pricier against a large one. This is a gate for
  things that matter, not a linter you run on save.
- **The model follows your Codex config.** `cap-bundle` runs Codex under a scratch
  `CODEX_HOME`, so it reads the top-level `model` and `model_reasoning_effort` from your
  real `~/.codex/config.toml` and passes them explicitly (fallbacks: `gpt-5.5`, `high`).
  `CAP_MODEL` / `CAP_EFFORT` override. If Codex still rejects the model, CAP degrades to
  single-reviewer — check that the label says so rather than assuming you got both.
- **Same-family corroboration is weaker than it looks.** The Claude subagent has none of
  your session's *conversation*, but it inherits the same `CLAUDE.md` and project memory —
  which is often exactly where the reasoning that biased the main agent lives.
- **It reviews what you hand it.** `git archive HEAD` means uncommitted work is invisible
  to the review — commit first, or pass `--files`. Submodule contents and
  `.gitattributes export-ignore`d files are excluded too; the script warns when either
  applies, because a confident review of *less code than you think* is its own failure
  mode.
- **Retention is between you and your vendor.** Whether OpenAI retains, logs, or trains on
  what you send depends on your account type and data-control settings. Check them before
  your first run; this repo makes no representation about it.
- **Prompt injection is a live risk in PR mode**, which reviews untrusted diffs by
  definition. The prompt tells reviewers to treat the material as data and to report any
  text addressing them — that is mitigation, not immunity.
- **Tested by one person, on macOS.** `codex-cli 0.142.5`, 2026-07-30. The bundle
  script's guards are pinned by [`tests/run.sh`](tests/run.sh) — one test per bug
  found in this repo's own adversarial reviews, all local, no network, including a
  fake-`codex` capture of the send-stage invocation. The end-to-end review path has
  been exercised by the author and nobody else yet, and Linux/GNU-userland behavior
  is untested beyond CI.

## Name

CAP originally stood for *Codex Adversarial Pass* — v1 was a single Codex reviewer. v3
added the parallel Claude subagent and the cross-model adjudication layer, so the
expansion became **Cross-model Adversarial Pass**. Same acronym, same trigger word,
accurate description.

## License & support

MIT — see [LICENSE](LICENSE).

Shared as-is because people kept asking for it. Issues and PRs are welcome; no support
or response time is promised. If you build something better on top of it, I'd genuinely
like to see it.

Tested against: Codex CLI with `gpt-5.5`, Claude Code, 2026-07-30.

— [Richard Schmidt](https://richardschmidt.ai)
