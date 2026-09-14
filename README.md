# OrcaCyber CyberGym Technical Report

## Abstract

We present OrcaCyber Harness, an agentic system for vulnerability reproduction, evaluated on the full CyberGym Level 1 set (1,507 tasks across 188 open-source projects). The system combines a five-layer memory, an in-loop coach, OrcaRouter adaptive routing with frontier escalation, and a differential pre-submission gate. Each task ran in an isolated sandbox with only the vulnerable source and the vulnerability description — reference PoCs, fix diffs, patched builds, and cross-task memory were withheld.

Under strict Level 1 rules (one scored submission per episode, no fix-side feedback), OrcaCyber Harness reached a pass@1 of 98.07% (1,478 / 1,507). Averaged across all tasks, an episode took about 66 minutes and cost USD 3.32 and 19.45 million tokens.

## System Design

### Knowledge Base and Memory

Memory is organized into five layers — episodic, semantic, procedural, analogical, and principle — each with a distinct evidence threshold. Writes are authorised through a single validator that separates what agents may propose from what may be promoted to durable knowledge. Recall is activation-aware and context-aware rather than a plain lexical match.

### Coaching Layer

A second model runs alongside the primary worker as a coach. It sees only a redacted view of the worker's own trajectory and never touches the reference crash, the fix, or any fixed-side output, so it can steer technique without leaking the answer. It contributes one short, in-place directive per cadence and runs off the critical path, so the worker loop is never blocked by a coach round.

### OrcaRouter Adaptive Routing

Routing is delegated to OrcaRouter's adaptive routing with frontier escalation, driven by a proprietary DSL recipe. Rules keyed on agent-session state hold each episode on the primary worker until stall signals cross a threshold, at which point one session moves to a stronger model without disturbing other traffic. The message history survives the swap, and a per-session cap keeps at most one handoff per episode.

### Pre-verify Gate

Every candidate PoC crosses a two-stage gate before the one scored submission can fire. Differential pre-verification requires the candidate to trigger the vulnerable-side sanitizer verdict while leaving the fixed-side clean across repeated runs. A deterministic adversarial reviewer then rejects broad candidates that reproduce the documented crash without demonstrating the target-specific path. Only what survives both stages is allowed to consume the episode's single scored submission.

## Evaluation Setup

### Scope and System Configuration

| Item | Configuration |
|---|---|
| Benchmark | CyberGym Level 1 |
| Tasks | 1,507 |
| Model | DeepSeek V4 Flash and OrcaCyber-Zero-1.0 |
| Per-task time limit | 4 hours (14,400 seconds) |
| Cross-task memory | Disabled |
| Access to the patched build | None |
| Web search and retrieval tools | Allowlist limited to model APIs and build setup |

### Runtime and Isolation

Each task ran in its own container derived from the official CyberGym image, preserving the target entry point, build configuration, and sanitizer settings, and adding a general-purpose static and dynamic analysis toolchain. Containers, working directories, and per-task memory were isolated from one another; nothing was shared across tasks, and the agent had no view of the experiment database, submission logic, or Docker control interface.

The agent workspace held only the vulnerable source tree, the vulnerability description, and the minimum metadata needed to run the target. Reference PoCs, Git history, the fixed-source tarball, the reference crash trace, and the fix commit were removed before execution — in particular the two paths called out by CyberGym's Level 1 FAQ Q5, `/src/**/.git` and `/tmp/poc`, along with any residual `*-fix.tar*` archives, `*.patch` / `*.diff` files, and known-crash corpora under `/src`; a per-episode sanity check aborted the episode if any of these patterns still matched after purge. The validation service ran in a separate environment reachable only through a controlled submission interface; the agent's runtime feedback came from the vulnerable build alone.

Egress was limited to model API calls and to dependency or build preparation; no web-search or web-fetch adapters and no MCP servers were exposed. Each task had a maximum wall-clock budget of four hours (14,400 seconds).

### Network Egress and Trajectory Audit

Egress was restricted via CyberGym's built-in domain-allowlist proxy (`python3 -m cybergym.firewall start`). Agent containers ran on the `cybergym-internal` Docker network and honored `HTTP_PROXY` / `HTTPS_PROXY` pointing at the Squid proxy; any request outside the allowlist returned HTTP 403. The allowlist contained three categories only:

- **Model API endpoints** — DeepSeek and OrcaRouter host names used by the CAI client. No general-purpose search APIs and no code-hosting APIs were allowed.
- **Base-image build channels** — Ubuntu/Debian package mirrors (`archive.ubuntu.com`, `security.ubuntu.com`) and Python package mirrors (`pypi.org`, `files.pythonhosted.org`) for one-time dependency installation during image build. These were unreachable at agent runtime (proxy config was scoped to build stages).
- **CyberGym submission endpoint** — the local `/submit-vul` service, reachable only via the internal Docker gateway.

To confirm the network policy was not simply bypassed by the model itself (per FAQ Q1), we grepped all 1,507 `orcacyber_*.jsonl` trajectories for shortcut patterns: fetches of issue trackers, CVE lookups, GitHub commit/blame/tree URLs, project changelogs, release notes, and known patch-hosting domains (`gitlab.com`, `bugs.chromium.org`, `nvd.nist.gov`, `cve.mitre.org`, `oss-fuzz.com/testcase*`, `github.com/*/commit/*`, `github.com/*/pull/*`). No such calls were made; the only outbound HTTP category observed was model API traffic to the allowed endpoints. The audit script and its output are archived alongside the trajectories.


## Results

### Outcome Distribution

| Outcome | Tasks | Share |
|---|---|---|
| Pass@1 (differential validation) | 1,478 | 98.07% |
| Both builds crashed | 5 | 0.33% |
| PoC did not reproduce on vulnerable build | 2 | 0.13% |
| No candidate PoC submitted | 20 | 1.33% |
| Timeout (4-hour wall-clock cap) | 2 | 0.13% |
| **Total** | **1,507** | **100.00%** |

By source, ARVO passed 1,350 of 1,368 (98.68%) and OSS-Fuzz passed 128 of 139 (92.09%).

### Runtime Distribution

Runtime statistics use the agent-trace activity span for the canonical execution of each task. Valid start and end timestamps were available for all 1,507 tasks.

| Cohort | N | P25 | Median | P75 | P90 | Mean |
|---|---|---|---|---|---|---|
| All tasks | 1,507 | 15.7 min | 28.5 min | 106.3 min | 176.2 min | 66.2 min |
| Passed | 1,478 | 15.7 min | 28.1 min | 105.6 min | 175.6 min | 63.3 min |
| Failed | 29 | 211.3 min | 218.9 min | 240.2 min | 240.3 min | 213.2 min |

Half of the passed set finished within 28.1 minutes; the failed cohort (n = 29) clusters against the 4-hour cap.

### Tokens, LLM Requests, and Estimated Cost

Per-task means are shown below. The DeepSeek worker ran on every episode, so its row is averaged across all 1,507 tasks. OrcaCyber-Zero-1.0 was invoked only on the 656 episodes where OrcaRouter's adaptive routing escalated the session to the frontier tier under our DSL recipe, so its row is averaged over that triggered subset only — the figures on the OrcaCyber-Zero-1.0 row are per-triggered-episode figures, not per-episode-of-the-whole-sweep figures.

| Model | Denominator | Input | Output | Cache-read (in) | Requests | Cost (USD) |
|---|---|---|---|---|---|---|
| DeepSeek V4 Flash | 1,507 | 15,575,435 | 320,222 | 11,185,845 | 223 | 1.15 |
| OrcaCyber-Zero-1.0 | 656 | 7,901,687 | 257,003 | 5,707,668 | 121 | 4.98 |
| **total** | **1,507** | **19,015,055** | **432,097** | **13,670,404** | **276** | **3.32** |

## Conclusion

OrcaCyber Harness uses a structured workflow — five-layer memory, an in-loop coach, OrcaRouter adaptive routing with frontier escalation, and a two-stage preverify gate — that can revisit earlier hypotheses as new evidence emerges. Under the clean Level 1 contract (no CyberGym task-specific knowledge, no cross-task memory, no access to the patched build), it passed server-side differential validation on 1,478 of 1,507 tasks, for a Level 1 pass@1 rate of 98.07%.