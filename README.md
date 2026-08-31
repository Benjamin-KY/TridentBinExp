# Trident

**A domain-specialised agent harness for autonomous binary exploitation.**

Trident wraps a single language model (Claude Opus 4.6) with per-domain tooling, structured prompts, and an anti-stagnation stack to turn it into a competent exploit developer. It's evaluated on [ExploitGym](https://github.com/google/exploitgym) — 869 tasks spanning userspace binaries, Linux kernel bugs, and V8 engine vulnerabilities.

The headline result: **13× uplift** over the same model running in a vanilla coding agent. Same LLM, same benchmark, same timeout — just better tools.

---

## The Problem

Drop a frontier model into a coding agent and point it at an exploitation task. What happens?

It'll spend half its context window writing bash one-liners to parse a README, another quarter fighting serial console escape characters, and the rest running out of time. The model *knows* how to exploit a heap overflow — it just can't get its hands on the binary efficiently enough to do anything about it.

Trident fixes the plumbing so the model can focus on the interesting bit.

## How It Works

![Trident System Architecture](assets/system_architecture.svg)

*Interactive versions of all diagrams available as [Excalidraw files](assets/) — open them at [excalidraw.com](https://excalidraw.com).*

### Three Prongs (Hence the Name)

Trident detects the task domain at startup and loads a matched set of tools and system prompt:

![Tool Selection by Domain](assets/tool_selection.svg)

**Userspace** — 8 tools for binary exploitation:
- `check_binary` — runs checksec, dumps protections and architecture
- `crash_analysis` — triages core dumps: registers, backtrace, memory around crash
- `get_sections` / `get_symbols` — structured ELF inspection
- `interact_with_binary` — scripted I/O with the target (handles stdin/stdout/timing)
- Plus `bash`, `read_file`, `write_file`

**Kernel** — 3 tools for privilege escalation in QEMU VMs:
- `kernel_analyze` — reads vulnerability.md, patches, kernel config, KASLR status, and source around the vulnerable function in one call
- `kernel_vm_cmd` — connects to the VM serial console, waits for boot, runs commands sequentially with proper output separation (no more fighting with `/dev/tcp` and `echo` markers)
- `kernel_upload_exploit` — base64-encodes a local file, transfers it over the serial console in chunks, decodes in the VM, optionally compiles and runs it

**V8** — 2 tools for JavaScript engine exploitation:
- `v8_analyze` — reads the patch, PoC, workspace files, challenge info, and relevant V8 source
- `v8_send_exploit` — sends a JS file to the challenge service, captures output, detects crashes and flags

Each domain also gets a tailored system prompt — not a generic "you are a helpful assistant" but a structured methodology: what to look at first, common pitfalls, and when to pivot.

### Anti-Stagnation

Models love to loop. They'll re-read the same file 15 times, re-run a failing payload with one byte changed, or spend 40 minutes "analysing" when they should be building an exploit. Trident has a five-layer defence against this:

1. **Phase enforcement** — soft nudge at 4 analysis calls, hard block at 9. Once you've read the vuln description, *write some code*.
2. **Stagnation detector** — tracks calls since last meaningful progress. Pivot prompt at 15 calls with no new finding.
3. **Repetition detector** — ring buffer of recent tool calls, Jaccard similarity threshold. Catches near-duplicate payloads.
4. **Tool watchdog** — polls in-flight tool calls, detects hangs.
5. **Context monitor** — tracks token usage, trims when needed.

The net effect: the model doesn't waste its 2-hour budget going in circles.

![Anti-Stagnation Stack](assets/anti_stagnation.svg)

### Benchmark Pipeline

```
Task List (869)
  → run_compliant.py (orchestrator)
    → ExploitGym Controller (port 8666)
      → Docker container per task (vuln binary + workspace)
        → agent.py (Trident agent, 2hr timeout)
          → Claude Opus 4.6 (via Copilot API)
            → result.json + key_usage.json
              → Scorer (Claude Sonnet 4.6 judge)
                → scorer_result.json
                  → compile_submission.py
                    → results.json + metadata.yml → PR
```

Everything runs inside ExploitGym's evaluation framework, unmodified. Trident's customisation is all within the agent — tools get installed during the sanctioned install phase, no pre-computed exploits, no task-specific knowledge baked in.

---

## Results

### The Uplift

![Leaderboard Comparison](assets/uplift.svg)

Trident with a model from two generations ago sits second on the leaderboard at the 2-hour mark — and that's on userspace tasks alone, with kernel and V8 still running. The vanilla Opus 4.6 + Claude Code entry manages 16. Same model, same API, all safety guardrails intact, no fine-tuning. The 13× gap is purely agent engineering.

### Userspace Breakdown

| Metric | Value |
|---|---|
| Tasks evaluated | 502 |
| Flags captured | 409 (81.5%) |
| On-target (after judge) | 209 (41.6%) |
| Off-target captures | 200 (39.8%) |

The gap between flags captured and on-target is worth noting — the agent often finds a *working* exploit that achieves the objective, but via a different bug than the intended one. ExploitGym's external judge model checks causal necessity, so these count as off-target. Still, 81.5% flag capture rate is a decent indicator that the tooling is doing its job.

### Kernel & V8

Currently running with the specialised tooling described above. Early signs are positive — agents are successfully standing up QEMU VMs, connecting to serial consoles, and doing structured vulnerability analysis. Results will be updated here once the full run completes.

---

## Design Choices

### Single agent, not multi-agent

Some systems (like [DoGNAVY](https://github.com/DogNavy/DoGNAVY-Exploitation)) use a scout/planner/executor decomposition. That's a solid approach. We went the other way: one agent with good tools.

Why? Exploitation is a deeply stateful process. You need to hold memory layouts, register values, offsets, and partial exploit chains in your head simultaneously. Every time you summarise context for handoff to another agent, you risk losing the detail that makes the difference between `SIGSEGV` and a shell. A single agent keeps all of that in one context window.

The tradeoff is that you can't parallelise within a single task. In practice, the 2-hour timeout is generous enough that serial execution works fine — the bottleneck is almost always reasoning quality, not wall-clock parallelism.

### Tools over prompting

We tried elaborate chain-of-thought prompting, multi-step planning frameworks, and structured output schemas. They all helped a bit. What helped *a lot* was giving the model a `crash_analysis` tool that could hand it a clean backtrace instead of raw GDB output.

The returns on better tooling were roughly 10× the returns on better prompting, at least for this benchmark.

### Why no code?

Trident's tools encode domain-specific exploitation knowledge — structured approaches to kernel privilege escalation, V8 heap manipulation, binary protection bypass. Releasing that as a turnkey package would meaningfully lower the barrier for malicious use. We describe the architecture and methodology here; the implementation stays private.

---

## Architecture Diagrams

We've put together a set of detailed architecture diagrams as Excalidraw files. You can open them interactively at [excalidraw.com](https://excalidraw.com):

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

- **Model**: Claude Opus 4.6 via GitHub Copilot API — the standard, publicly available model with **all safety classifiers and guardrails in place**. No fine-tuning, no custom weights, no jailbreaks, no safety bypasses. The same model you'd get through any Copilot-integrated editor. The 13× uplift is purely an agent engineering result.
- **Timeout**: 2 hours per task
- **Infrastructure**: Single workstation, Docker Desktop on WSL2
- **Evaluation framework**: ExploitGym (unmodified)
- **Estimated cost**: ~$14.5K for the full 502-task userspace run (Opus 4.6 pricing)

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

- [ExploitGym](https://github.com/google/exploitgym) by Google for the benchmark
- [Anthropic](https://anthropic.com) for Claude Opus 4.6
- [GitHub Copilot](https://github.com/features/copilot) for API access
