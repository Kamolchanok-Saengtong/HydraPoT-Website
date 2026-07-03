# HydraPoT — Architecture & Agent Documentation

> A Multi-Agent Hybrid Platform for Honeypot-Based Threat Intelligence and Interactive Attack Analysis

---

## What HydraPoT Is

HydraPoT is a multi-agent SSH honeypot. Instead of relying on a single static
honeypot or a single AI model, it routes each attacker command to one of
three specialized backend agents based on **what kind of reasoning the
command actually requires** — not based on how dangerous the command is.

```
Attacker connects via SSH
        │
        ▼
   AsyncSSH frontend
        │
        ▼
   HoneyRouter (rule-based classifier)
        │
   ┌────┼────────────┬──────────────┐
   ▼                 ▼              ▼
Cowrie          On-device LLM    Cloud LLM
(known/basic)   (context-aware)  (obfuscated/complex)
```

The goal: give attackers a consistently realistic, interactive shell — one
that's cheap to run for the 90% of commands that are simple, and only pays
the cost of a larger model when a command genuinely needs deeper reasoning
to fake convincingly.

---

## The Three Agents

### 1. Cowrie — known/basic commands
A real, open-source SSH/telnet honeypot (Docker container) that handles
commands with well-understood, deterministic output: `ls`, `whoami`, `cat`,
`ps`, `uname`, package management, networking tools, etc. Cowrie is fast,
free to run, and battle-tested for these cases.

### 2. On-device LLM — context-dependent commands
A locally-hosted LLM (currently a 7B-class model, quantized, served via
llama-cpp on consumer GPU hardware) that handles commands requiring
awareness of *session state* — files the attacker created, permissions
they changed, scripts they're executing, services they queried. Runs
entirely on local hardware: no API cost, no external network calls,
moderate latency.

### 3. Cloud LLM — commands requiring semantic reconstruction
A larger, hosted LLM used only for commands whose real intent is hidden
behind encoding, indirection, or dynamic evaluation — the small fraction
of attacker traffic that needs genuine reasoning to safely and
convincingly simulate. Highest latency and cost, used sparingly by design.

---

## Routing — How HydraPoT Decides Which Agent Handles a Command

**This is the core architectural decision of the project, and it has been
deliberately redesigned to separate two concerns that are often conflated:**

1. **How dangerous is this command?** (Functional Impact / FI score)
2. **Which agent should execute it?** (Routing)

In earlier iterations of HydraPoT these were the same thing — a command's
danger score directly decided where it went. The current architecture
treats them as **independent axes**:

```
FI score  →  used ONLY for memory/context retention (what's worth
              remembering across a session), NOT for routing

Routing   →  decided entirely by a rule-based classifier that asks
              "what KIND of command is this, mechanically?"
```

### Why this separation matters

A command can be highly dangerous but completely transparent in intent
(`rm -rf /`) — Cowrie can simulate that perfectly well without any LLM
reasoning. Conversely, a command can be functionally trivial but have its
real intent completely hidden (`eval $(echo whoami)` evaluates to something
as simple as `whoami`, but the *form* requires decoding before any agent
can know what to simulate). Routing by danger level alone would send the
first to an expensive LLM unnecessarily and might let the second slip
through a pattern-matcher untouched.

```
eval $(echo whoami)                       → low impact, but routes to Cloud
                                             (intent is hidden, must decode)

rm -rf /                                  → high impact, but routes to Cowrie
                                             (intent is explicit, no decoding needed)
```

### The Rule-Based Router

Routing is decided by `classify(cmd)`, which checks three tiers in order:

```
1. Is this command obfuscated / does it require semantic reconstruction?
       → CLOUD

2. Does this command match a known basic-command pattern?
       → COWRIE

3. Does this command match a known context-dependent pattern?
       → ON-DEVICE

4. (fallback)
       → COWRIE
```

**Tier 1 — Obfuscation detection (mechanism-based, not statistical).**
Earlier designs used entropy thresholds and operator-counting heuristics to
guess whether a command was obfuscated. These were found to be unreliable
and were replaced entirely with detection anchored to six concrete,
nameable shell mechanisms — every rule corresponds to an actual technique
an attacker would use, not a statistical proxy for one:

- **Indirect execution** — `eval`, subshell-wrapped `bash -c` with decoding
- **Encoding/decoding** — base64, hex escapes, `xxd -r`
- **Dynamic shell evaluation** — bare subshells, variable-splitting tricks
- **Interpreter-mediated execution** — `python3 -c "os.system(...)"` style
  payloads (only escalated when genuine execution intent is present —
  `python3 -c "print(123)"` is NOT obfuscated)
- **Pipe-to-interpreter** — remote payload fetched and piped directly into
  a shell or interpreter
- **Reverse-shell primitives** — `/dev/tcp/`, `mkfifo`, `nc -e`

**Tier 2 / Tier 3 — Pattern-based classification.** Everything not caught
by Tier 1 is matched against two curated pattern sets: a "basic" set
(file operations, process inspection, networking utilities, package
management — the bulk of real attacker traffic) and an "on-device" set
(service management, script execution, anything requiring awareness of
what previously happened in the session).

---

## Functional Impact (FI) — What It Actually Does

FI is a 0–4 severity score used **exclusively for memory management** — it
has no bearing on which agent handles a command.

| FI | Meaning | Examples |
|----|---------|----------|
| 0 | Read/Display | `ls`, `whoami`, `cat`, `uname -a` |
| 1 | Create/Install | `touch`, `mkdir`, `apt install` |
| 2 | Modify/Navigate | `chmod`, `rm`, `cd`, `echo > file` |
| 3 | Service/Download/Elevate | `sudo`, `systemctl`, `curl\|bash` |
| 4 | Impact/Delete/Passwd | `passwd`, `rm -rf`, `useradd`, `chmod 777` |

FI feeds a retention-scoring mechanism that decides which session events
are worth keeping in the LLM's working memory as a session grows long:

```
Retention = (0.7 × FI_weight) + (0.3 × Recency_weight)
```

Higher-impact, more-recent commands are retained longer; low-impact,
older commands are pruned first once the buffer fills. This keeps the
LLM's context window focused on what an attacker would actually remember
mattering, rather than growing unboundedly.

---

## SRi — System State Register (Shared Memory Across Agents)

The hardest problem in a multi-agent honeypot is **consistency**: if an
attacker creates a file via one agent and checks it via another, the
honeypot has to behave as if it's one coherent machine, not three
disconnected simulators.

HydraPoT solves this with a per-session shared state object (`SYSTEM_STATE`)
containing tracked files, installed packages, users, and version info. Every
command that changes state updates this object; every LLM-backed agent
reads from it before responding, so an attacker's file permissions, package
installs, and session history persist correctly regardless of which agent
they happen to be talking to in that moment.

```
attacker: echo 'secret' > /tmp/f.txt    → handled by Cowrie
                                           SYSTEM_STATE updated: /tmp/f.txt exists

attacker: cat /tmp/f.txt                → routed to On-device LLM
                                           (file content needed, Cowrie can't track it)
                                           On-device reads SYSTEM_STATE, sees the file,
                                           responds correctly: "secret"
```

One architectural limitation, by design: Cowrie reads from its own real
Docker filesystem, not from `SYSTEM_STATE` — so state created by an LLM
agent is never expected to be visible to a subsequent Cowrie-routed command
on the same path. The router includes a safety check that detects when a
command references a file that only exists in shared state and redirects
it away from Cowrie accordingly, so this gap doesn't surface to the
attacker.

---

## Evaluation Framework

HydraPoT includes a structured evaluation suite (referred to internally as
the NSC evaluation framework) built to measure whether the architecture's
added complexity is actually worth it:

**Architecture overhead** — comparing the full multi-agent system against
three single-agent baselines (Cowrie-only, on-device-only, cloud-only) on
identical workloads, measuring latency, CPU, RAM, VRAM, and throughput.

**Context preservation** — testing whether state set by one agent is
correctly readable by a different agent later in the same session, across
all architecturally valid agent-pair combinations, scored against ground
truth captured from real attacker command sequences replayed in a clean
Docker environment.

**SRi overhead** — isolating the specific latency cost of injecting shared
session state into LLM prompts, separated from all other sources of
latency variance.

---

## Summary for the Website

> HydraPoT is a multi-agent SSH honeypot that routes each attacker command
> to the cheapest backend capable of handling it convincingly — a real
> honeypot engine for known commands, a local LLM for commands requiring
> session awareness, and a cloud LLM reserved for the small fraction of
> traffic that's deliberately obfuscated. Routing is decided by a
> rule-based classifier built on concrete shell-execution mechanisms,
> not statistical heuristics. A shared state layer keeps the simulated
> machine consistent no matter which backend an attacker happens to be
> talking to, and a structured evaluation framework measures whether the
> added architectural complexity earns its cost in realism and efficiency.