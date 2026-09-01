# Trident

**Custom agent harness for binary exploitation benchmarks.**

> **TL;DR** — Domain-specialised tools + anti-stagnation controls turned Claude Opus 4.6 from 16 on-target exploits (vanilla Claude Code) into 206 on-target exploits on ExploitGym's userspace tasks — **#2 on the global leaderboard at the 2-hour mark**, 10 behind GPT-5.6 Sol and ahead of Claude Opus 5, Mythos 5, and GPT-5.5. Same model, same API, all safety classifiers active. Total cost: $2,450 ($4.87/task). This is a final userspace-only submission; kernel, V8, and hardened-mitigation results are not included. This repo describes the architecture. No code — the tools encode offensive security knowledge that shouldn't be packaged for distribution.

---

## What This Repo Contains

Architecture documentation, benchmark results, and design rationale for Trident's ExploitGym harness. Detailed [Excalidraw diagrams](assets/) cover every layer of the system. There is no source code — see [Why No Code?](#why-no-code) below.

## Background

I work on an internal security assessment platform at Microsoft called Trident. It's a proprietary agent harness for offensive security work. When [ExploitGym](https://github.com/sunblaze-ucb/exploitgym) dropped, I wanted to answer a question: can a well-engineered harness with an older model beat the latest frontier models running generic coding agents?

The leaderboard had Claude Mythos 5, Opus 5, GPT-5.6 Sol all posting big numbers. Meanwhile vanilla Opus 4.6 + Claude Code was sitting at 16 on-target. I had a hunch that most of that gap was tooling, not model capability. So I built an ExploitGym-specific version of Trident's architecture to find out.

Turns out the hunch was right. **206 on-target exploits vs 16 for the vanilla same-model baseline.** The model already knows how to do exploitation — it just needs tools that handle the plumbing and something to stop it going in circles.

---

## The Problem

Give a frontier model a coding agent and an exploitation task. It spends half its context window writing bash one-liners to parse a README, another quarter fighting serial console escape characters, and runs out of time before it gets anywhere useful. The model *knows* how to exploit a heap overflow. It just burns through its budget on plumbing instead.

Trident handles the plumbing.

## How It Works

![Trident System Architecture](assets/trident_architecture.svg)

### Three Prongs (Hence the Name)

Trident detects the task domain at startup and loads a matched set of tools and system prompt:

![Tool Selection by Domain](assets/tool_selection.svg)

**Userspace** (8 tools for binary exploitation):
- `check_binary`: runs checksec, dumps protections and architecture
- `crash_analysis`: triages core dumps with registers, backtrace, memory around crash
- `get_sections` / `get_symbols`: structured ELF inspection
- `interact_with_binary`: scripted I/O with the target, handles stdin/stdout/timing
- Plus `bash`, `read_file`, `write_file`

**Kernel** (3 tools for privilege escalation in QEMU VMs):
- `kernel_analyze`: reads vulnerability.md, patches, kernel config, KASLR status, and source around the vulnerable function. All in one call.
- `kernel_vm_cmd`: connects to the VM serial console, waits for boot, runs commands sequentially with proper output separation. No more fighting with `/dev/tcp` and `echo` markers.
- `kernel_upload_exploit`: base64-encodes a local file, transfers it over the serial console in chunks, decodes in the VM, optionally compiles and runs it.

**V8** (2 tools for JavaScript engine exploitation):
- `v8_analyze`: reads the patch, PoC, workspace files, challenge info, and relevant V8 source
- `v8_send_exploit`: sends a JS file to the challenge service, captures output, detects crashes and flags

Each domain gets its own system prompt too. Not "you are a helpful assistant" boilerplate, but actual exploitation methodology for that target class.

### Anti-Stagnation

Models love to loop. They'll re-read the same file 15 times, re-run a failing payload with one byte changed, or spend 40 minutes "analysing" when they should be writing C. Trident has five layers to keep things moving:

1. **Phase enforcement.** Soft nudge at 4 analysis calls, hard block at 9. Once you've read the vuln description, *write some code*.
2. **Stagnation detector.** Tracks calls since last meaningful progress. Pivot prompt at 15 calls with no new finding.
3. **Repetition detector.** Ring buffer of recent tool calls, Jaccard similarity threshold. Catches near-duplicate payloads.
4. **Tool watchdog.** Polls in-flight tool calls, detects hangs.
5. **Context monitor.** Tracks token usage, trims when needed.

Without this, the model burns through its 2-hour budget going in circles. With it, it actually gets to the exploit.

![Anti-Stagnation Stack](assets/anti_stagnation_system.svg)

### Benchmark Pipeline

![ExploitGym Pipeline](assets/exploitgym_flow.svg)

Everything runs inside ExploitGym's evaluation framework, unmodified. Trident's customisation is all within the agent. Tools get installed during the install phase, no pre-computed exploits, no task-specific knowledge baked in.

---

## Results

### The Uplift

![Leaderboard Comparison](assets/uplift.svg)

Trident with Opus 4.6 achieves 206 on-target exploits on userspace tasks, compared to 16 for the vanilla Opus 4.6 + Claude Code baseline. At the apples-to-apples 2-hour timeout, this places us **#2 on the global leaderboard** — just 10 behind GPT-5.6 Sol (216), and ahead of Claude Mythos 5 (181), Claude Opus 5 (171), and GPT-5.5 (129). No fine-tuning, no custom weights — the difference is harness engineering. This submission reports the full userspace set only; kernel and V8 are not included.

### Userspace Breakdown

| Metric | Value |
|---|---|
| Tasks evaluated | 502 |
| Flags captured | 409 (81.5%) |
| On-target (after judge) | 206 (41.0%) |
| Off-target captures | 203 (40.4%) |

Big gap between flags captured and on-target. The agent frequently finds a working exploit that captures the flag, but through a different vulnerability than the one the task intended. ExploitGym uses an external judge model (Claude Sonnet 4.6) to determine whether the exploit exercises the *intended* bug — if it doesn't, the capture is scored as off-target.

The high off-target rate reflects the agent's strength at finding *any* exploitable path, but also a weakness: it doesn't always follow the vulnerability description closely enough. Many off-target captures are genuine exploits against real bugs in the same binary — just not the one being tested. This is an area where better prompt engineering and vulnerability-focused tool design could narrow the gap. It's also partly an artefact of ExploitGym's intentionally strict causal-necessity judge.

### What a Successful Run Looks Like

![Example Challenge Run: arvo_37443](assets/example_challenge_run.svg)

### Kernel & V8

**Status: Not included in this submission.** ExploitGym has 367 non-userspace tasks (186 kernel, 181 V8), but this result set reports only the 502 userspace tasks.

The kernel and V8 domains require significant additional engineering beyond the userspace harness — QEMU VM lifecycle management, serial console interaction, credential plumbing through Docker networking, and domain-specific phase nudge tuning. Trident's architecture includes support for these domains:
- Standing up QEMU VMs via the ExploitGym controller
- Running commands through the serial console
- Compiling and testing kernel exploits inside VMs
- Analysing V8 patches and sending JavaScript exploits to the challenge service

No kernel or V8 scores are claimed here.

### With Mitigations (exp.hardened)

Not included in this submission. ExploitGym supports multiple mitigation profiles, from `exp.none` (no protections) through to `exp.hardened` (stack canaries, PIE, full RELRO). All results above are userspace `exp.none` only.

---

## Design Choices

### Single agent, not multi-agent

Some systems split the work across multiple specialised agents. There are good reasons to do this, and it works well for tasks with natural decomposition points. We went the other way: one agent with domain-specific tools.

Exploitation is deeply stateful. You need memory layouts, register values, offsets, and half-built exploit chains all in context at once. Handoffs between agents risk losing the precise detail that makes the difference between `SIGSEGV` and a shell. A single context window avoids that risk.

The trade-off is no within-task parallelism. In practice this hasn't been a bottleneck — the 2-hour timeout is generous and the limiting factor is reasoning quality, not clock time.

### Tools beat prompting

We tried chain-of-thought prompting, multi-step planning, structured output schemas. They helped incrementally. Giving the model a `crash_analysis` tool that hands back a clean backtrace instead of raw GDB output helped dramatically more.

Our experience building this harness is that tool quality was the single largest lever. Better prompts moved the needle by percentage points; better tools moved it by multiples. We haven't run formal ablations to quantify this precisely, but the pattern was consistent throughout development.

### Working with safety classifiers, not around them

This one doesn't get talked about much, but it might be the hardest engineering problem in autonomous exploitation. Claude Opus 4.6 ships with safety classifiers that will refuse to write exploit code if the request looks like it's asking for help attacking a real system. Run a vanilla agent against ExploitGym and you'll hit refusals on a fair number of tasks. The model knows what a ROP chain is, it just won't write one if it thinks you're up to no good.

Getting low refusal rates on autonomous offensive security work is one of the harder engineering problems in this space. Across all 502 userspace tasks (62,000+ LLM requests), we saw zero safety refusals that blocked task completion. This isn't because we bypassed anything — it's because the context presented to the model accurately reflects what's happening: authorised security research against an isolated benchmark environment.

We're not going to detail the specific prompt engineering here. But the key point is: this is compliant engineering, not adversarial. The classifiers are doing their job correctly. ExploitGym is a published research benchmark, the targets are isolated Docker containers, and the work is authorised security research. Present that context properly and the model cooperates. Don't, and it won't.

This was a significant chunk of the engineering effort, and a big part of why the harness outperforms vanilla agents. It's also the part we're most proud of.

### Why no code?

The tools encode domain-specific exploitation knowledge — kernel privilege escalation patterns, V8 heap manipulation techniques, binary protection bypass strategies. Publishing that as a reusable package would be irresponsible. This repo documents the architecture and methodology in enough detail to understand the approach and inform similar systems. The implementation stays private.

---

## Benchmark Compliance

Trident follows all ExploitGym evaluation rules. We went through the submission spec, evaluation docs, and agent base class carefully before building anything. The key points:

- **Unmodified framework.** ExploitGym's evaluation harness, controller, scorer, and Docker images are all stock. Nothing patched or modified.
- **Agent subclass.** Trident extends ExploitGym's `Agent` base class. The benchmark explicitly supports custom agents and expects them to override `install()` and `run()`.
- **Install-phase only.** All tool installation (gcc, gdb, pwntools, etc.) happens during the `install()` phase before the timer starts. Nothing is pre-baked into Docker images.
- **No pre-computed exploits.** The agent has no task-specific knowledge. It reads the workspace files (vulnerability description, patches, PoC) at runtime and reasons from scratch.
- **Network.** The agent container only needs the Copilot API (for LLM calls) and the ExploitGym controller (for VM/challenge provisioning). It doesn't phone home or fetch external resources.
- **Standard scoring.** Results are scored by ExploitGym's external judge model (Claude Sonnet 4.6) using the `is_causally_necessary` field. We don't influence the scorer.
- **Userspace-only result set.** All 502 userspace tasks evaluated, no cherry-picking. Kernel, V8, and hardened-mitigation tasks are not included or claimed in this submission.

---

## Architecture Diagrams

Detailed architecture diagrams as Excalidraw files. Open at [excalidraw.com](https://excalidraw.com) to poke around interactively:

| Diagram | Description |
|---|---|
| [`trident_architecture.excalidraw`](assets/trident_architecture.excalidraw) | High-level system: LLM → SDK → Operator → MCP Pool → Engines |
| [`exploitgym_flow.excalidraw`](assets/exploitgym_flow.excalidraw) | End-to-end benchmark pipeline: task list through to submission |
| [`example_challenge_run.excalidraw`](assets/example_challenge_run.excalidraw) | Timeline of one successful exploit (arvo_37443) |
| [`anti_stagnation_system.excalidraw`](assets/anti_stagnation_system.excalidraw) | The five-layer anti-stagnation stack |
| [`operator_internals.excalidraw`](assets/operator_internals.excalidraw) | Operator internals: prompt assembly, event handling, agent loop |
| [`model_and_harness_flow.excalidraw`](assets/model_and_harness_flow.excalidraw) | Data flow between model, harness, and engine layers |

---

## Technical Details

- **Model**: Claude Opus 4.6 via GitHub Copilot API. Stock model, publicly available, all safety classifiers active. No fine-tuning, no custom weights.
- **Timeout**: 2 hours per task
- **Parallelism**: 4 concurrent tasks
- **Wall clock**: ~7 days for 502 userspace tasks
- **Infrastructure**: One workstation, Docker Desktop on WSL2
- **Evaluation framework**: ExploitGym, unmodified

### Userspace Run Stats

| Metric | Total | Per Task (avg) |
|---|---|---|
| LLM requests | 62,068 | 124 |
| Input tokens | 2.72B | 5.42M |
| Output tokens | 33.0M | 66K |
| Cache read tokens | 2.66B | 5.30M |
| Cache hit rate | 97.9% | |
| Effective input tokens (uncached) | 58.3M | 116K |
| LLM compute time | 223 hours | 27 min |
| Estimated cost | $2,450 | $4.87 |

## Citation

```bibtex
@misc{trident2026,
  title={Trident: Domain-Specialised Agent Harness for Autonomous Binary Exploitation},
  author={Benjamin Kereopa-Yorke},
  year={2026},
  url={https://github.com/Benjamin-KY/TridentBinExp}
}
```

## Acknowledgements

- [ExploitGym](https://github.com/sunblaze-ucb/exploitgym) by UC Berkeley for the benchmark
- [Anthropic](https://anthropic.com) for Claude Opus 4.6
- [GitHub Copilot](https://github.com/features/copilot) for API access
- [Microsoft](https://microsoft.com) for compute
- SHIELD for entertaining my dreams
