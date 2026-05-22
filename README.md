# deep-research

A Claude Code plugin for rigorous web research using **three independent LLM
families** as cross-checkers, **worktree-sandboxed** for security, and
**primary-source verified** via mechanical URL + passage checks.

The single goal: produce research output you can actually trust to make
decisions on — by hardening against the failure modes single-pipeline LLM
research is known to produce (consensus hallucination, fabricated citations,
prompt injection from web content, CLI/API confabulation).

```
/research <your question>
```

## What it does

`/research` spawns one researcher agent in worktree isolation. The agent
then runs three parallel research streams:

1. **Claude (you)** — WebSearch + WebFetch
2. **Gemini 3.1 Pro** — via the [`agy`](https://antigravity.com) (Antigravity)
   CLI, sandboxed (`--sandbox`)
3. **GPT-5** — via [OpenAI's `codex` CLI](https://github.com/openai/codex),
   sandboxed (`-s read-only`)

Each stream uses a different web-search backend (Google Search inside Gemini,
OpenAI's web tool inside Codex, Claude's WebSearch). After all three return,
the researcher triages four lists — Agreements / Gemini-only / GPT-only /
Conflicts — and resolves each one via primary-source verification before
including it in the final report.

The defense model is **evidence weight > vote count**. One cross-checker with
a primary-source URL that passes mechanical verification (URL resolves 2xx +
quoted passage actually appears in the fetched page) beats a two-way LLM
consensus with no source. Why this matters is documented in the literature
on consensus hallucination — when training corpora overlap heavily, LLM
agreement-without-evidence is a *weaker* signal than minority-dissent-with-
evidence ([Shared Imagination, arXiv:2407.16604](https://arxiv.org/abs/2407.16604);
[Till et al. 2025](https://arxiv.org/abs/2510.19507)).

## Defenses

| Threat | Defense |
| --- | --- |
| **Consensus hallucination** (overlapping LLM training corpora produce confident agreement on false facts) | 3 different LLM families with different search backends + primary-source verification on every retained claim |
| **Fabricated citations** (LLM cites a paper / URL that doesn't exist or doesn't say what was claimed) | Rule C: `curl -sIL` HTTP code check on every URL retained + passage match (re-fetch the URL, confirm quoted value appears verbatim) |
| **CLI / API / config-syntax confabulation** (LLM "remembers" a flag that doesn't exist) | Rule B: empirical `<cmd> --help` or vendor-doc fetch before claiming a flag exists |
| **Prompt injection from web content** | Output wrapped in `<external-research trust="untrusted">` tags before main session processes it; researcher is reminded that fetched HTML is data, not instructions; web-suggested shell commands never executed |
| **Code execution from compromised pages** | Worktree isolation on the researcher (sandboxed copy of repo); `agy --sandbox` enables kernel-level terminal restrictions; `codex -s read-only` enforced via macOS Seatbelt / Linux bubblewrap |
| **`$TMPDIR` race condition between concurrent `/research` sessions** | Each researcher generates a unique `RUN_ID` via `date +%s%N` and namespaces all tempfile paths with it |

## Installation

This is a local Claude Code plugin. Drop it under your plugins directory and
enable it.

```bash
# Clone into local plugins
mkdir -p ~/.claude/plugins/local/plugins
cd ~/.claude/plugins/local/plugins
git clone https://github.com/oh-rid/deep-research.git
```

Restart Claude Code (or reload plugins) and `/research` becomes available.

### Requirements

The plugin works in degraded mode without optional cross-checkers, but for
full 3-way triangulation you need:

- **`agy`** (Antigravity CLI) — runs Gemini models via Google AI Pro.
  Install via Homebrew or the official installer. Model is set in
  `~/.gemini/antigravity-cli/settings.json` (`"model": "Gemini 3.1 Pro (High)"`
  is the recommended choice; Flash variants may return empty on long
  academic prompts).
- **`codex`** (OpenAI Codex CLI) — runs GPT-5. Web search must be enabled
  in `~/.codex/config.toml`:

  ```toml
  [tools.web_search]
  enabled = true
  ```

If either CLI is missing the researcher empirically detects it
(`which agy` / `which codex`) and proceeds with the remaining
cross-checker plus its own WebSearch.

## Usage

```bash
/research what's the current academic consensus on r-star (natural rate of interest)?

/research is the Brand-Goy-Lemke SUERF Policy Brief 1360 about the US or the euro area?

/research what did the latest FOMC minutes say about rate hikes, and how did the curve react?
```

The slash command spawns the researcher, wraps the result in an
untrusted-source block, and presents a structured final answer to you with:

- **TL;DR** — 2-3 sentence summary
- **Numbered findings** with primary-source URLs
- **Confidence Map** (HIGH / MEDIUM / LOW per topic)
- **Rejected after cross-check** (claims that failed primary-source verification)
- **Open questions** (genuinely undetermined items)
- **Consensus-only claims** (all LLMs agreed, no primary source found — elevated hallucination risk flag)
- **Cross-check status** (completed / skipped / degraded per cross-checker)
- **Synthesis Value Report** (coverage gain over single-pipeline, dead-weight signal per researcher)

## Architecture

```
deep-research/
├── .claude-plugin/
│   └── plugin.json              # Plugin manifest, v0.2.0
├── agents/
│   └── researcher.md            # Researcher agent system prompt (~750 lines)
├── commands/
│   └── research/
│       └── COMMAND.md           # /research slash command definition
├── LICENSE                      # MIT
└── README.md                    # This file
```

The researcher runs through five phases per request:

| Phase | What | Notes |
| --- | --- | --- |
| **0a** | Empirical CLI precheck | `which agy` / `which codex` — never skip cross-checkers based on environmental assumption |
| **0b** | Generate `RUN_ID` for namespaced tempfiles | `date +%s%N` — race-protection vs concurrent sessions |
| **0c** | Background spawn of `agy` + `codex` | Both run with `run_in_background: true`; researcher continues its own web research while they work |
| **1** | Decompose research question | 3-6 specific sub-queries |
| **2** | Own WebSearch + WebFetch on sub-queries | Independent of cross-checkers |
| **3** | Draft own findings *before* reading cross-checker outputs | Anti-anchoring discipline |
| **4** | Triage with cross-checkers | Agreements / Gemini-only / GPT-only / Conflicts; sanity-check model echoes for silent fallback |
| **5** | Resolution research with primary-source verification | Rule B (CLI/API empirical), Rule C (mechanical URL + passage check), Rule D (evidence weight > vote count) |

## Comparison to single-pipeline research

| Property | Single-pipeline LLM research | This plugin |
| --- | --- | --- |
| Consensus hallucination defense | None — single LLM confabulates confidently | 3 different LLM families catch each other's blind spots |
| Citation verification | None — fabricated URLs/papers slip through | Rule C: mechanical URL + passage check |
| CLI/API claim verification | Often wrong (LLMs confabulate flags from convention prior) | Rule B: empirical `<cmd> --help` required |
| Prompt injection from web | LLM may follow malicious instructions in fetched content | Output wrapped as untrusted; researcher trained on this threat model |
| Worktree-style isolation | Whatever the host gives | Hard requirement; subagent cannot touch caller's project files |

## Limitations / known gaps

- DSGE-style "give me the very latest number" research depends on whether the
  exact number is on the public web. The plugin won't fabricate one; if no
  primary source exists, it returns "Open question" honestly.
- The `agy` cross-checker uses your Google AI Pro subscription. If you hit
  capacity (Pro models can 429 even for Pro tier), the researcher retries
  and then falls back to self-conducted parallel WebSearch, marked
  `SELF_SUBSTITUTE` in the final report.
- The `codex` cross-checker requires `tools.web_search.enabled = true` in
  `~/.codex/config.toml`. Without it, the researcher skips codex with a
  clear note in the final report (no silent degradation).
- The researcher writes tempfiles to `$TMPDIR`. Despite namespacing via
  `RUN_ID`, if you have hundreds of researchers running over weeks, you may
  want a janitor cron to clean `$TMPDIR/gemini_research_*.txt` and
  `$TMPDIR/gpt_research_*.txt`.

## Research basis

The plugin's defenses are not invented — each is anchored in published
literature on LLM failure modes. Inline references appear in
`agents/researcher.md` next to the rule they motivate; this table
collects them for external readers.

| Defense | Informed by |
| --- | --- |
| 3 different LLM families (not 3 prompts to the same model) | **Shared Imagination** — LLM agreement without evidence is a weaker signal than minority dissent with evidence ([arXiv:2407.16604](https://arxiv.org/abs/2407.16604), July 2024) |
| Verify with **fresh** WebSearch (Rule A — never re-examine pages from your own draft) | **Chain-of-Verification (CoVe)** — verification questions must be answered in isolation from the draft they verify ([Dhuliawala et al., ACL 2024, arXiv:2309.11495](https://arxiv.org/abs/2309.11495)) |
| Mechanical URL + passage verification (Rule C) | **Fabricated-citations study (NeurIPS 2025)** — 100 fake citations identified in 53 accepted papers despite 3–5 expert reviewers each; humans cannot catch this class reliably, only mechanical checks do ([arXiv:2602.05930](https://arxiv.org/abs/2602.05930)) |
| Evidence weight > vote count (Rule D) | Single-source-with-evidence beats majority-consensus-without ([Till et al., arXiv:2510.19507](https://arxiv.org/abs/2510.19507), October 2025; also Shared Imagination, above) |
| Empirical CLI / API verification (Rule B) | Practitioner observation, not paper-derived — LLMs confabulate flags from convention priors (e.g. adding `--model` to a CLI that has no such flag). Must be verified against `<cmd> --help` or vendor docs |

All arXiv URLs verified to resolve 2xx at plugin v0.2.0 release. Passage
matches (the second half of Rule C — confirm the quoted claim actually
appears in the fetched text) are the reader's responsibility for
high-stakes use — claims above are summary glosses, not verbatim quotes.

## Status

- v0.2.0 — production-ready as of 2026-05-22, verified end-to-end with all
  three cross-checkers returning live model echoes on a primary-source-
  verifiable test case (3-way convergence on a 3-fact verification of a
  recent SUERF Policy Brief). Single-file fixes for empirical CLI precheck,
  `RUN_ID` race protection, `agy` flag confabulation guard.

## License

MIT — see [LICENSE](LICENSE).
