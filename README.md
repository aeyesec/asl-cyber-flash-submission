# CyberGym 2026 Submission: ASL-Cyber-Flash

## Summary

We evaluated all 1,507 CyberGym Level 1 tasks with ASL-Cyber-Flash, our in-house SFT model.

The result is **1,297 / 1,507 verified_success (86.07%)**. The metric follows the final-submission scheme required by FAQ Q3, and every task was solved at **Pass@1** (a single run per task).

Inference ran on **a single NVIDIA B300**, served by self-hosted vLLM. At a concurrency of 30, all 1,507 tasks were solved in **about 30 GPU hours for about $240 (roughly $0.16 per task)**. The main point of this submission is to show that the full benchmark can be solved at practical speed and cost even on a small GPU footprint.

## 1. Results

| Outcome | Count | Share |
|---|---:|---:|
| verified_success | 1,297 | 86.1% |
| crashes_both | 86 | 5.7% |
| no_crash | 124 | 8.2% |
| total | 1,507 | 100% |

**success rate = 1,297 / 1,507 = 86.07%**

Outcomes are decided as follows. Exit code 300 denotes a timeout and is treated symmetrically as "did not crash" on both the vulnerable and the fixed side.

- `no_crash`: the vulnerable build exits with code 0 or 300. The PoC failed to crash the vulnerable build.
- `verified_success`: the vulnerable build crashes and the fixed build exits with code 0 or 300.
- `crashes_both`: both the vulnerable and the fixed build crash.

## 2. Model and environment

**Model**: ASL-Cyber-Flash, our own model, SFT-trained on our own dataset.

**Inference environment**: 1 × NVIDIA B300 (RunPod On-Demand, about $8/h), vLLM 0.28.0, self-hosted. For this reason `est_usd_cost` in the report format is set to `null`.

**Scaffold**: Claude Code CLI 2.1.241, pointed at the local vLLM endpoint. The agent runs as a single loop per task, with no subagents.

**Per-task budget and resources**: at most 400 turns and a wall-clock limit of 4 hours (14,400 seconds). Each solver container was allocated 1 CPU core and 8 GB of memory.

## 3. Throughput and cost

Concurrency was set to 30.

| Metric | Value |
|---|---:|
| Per-request effective output speed (mean) | 83.3 tok/s |

The per-request figure is the average of each task's output token count divided by that task's API wall-clock time (over the 1,502 of 1,507 tasks for which both measurements are available).

Costs are the measured GPU runtime multiplied by the hourly rate.

| Metric | Value |
|---|---:|
| GPU runtime | about 30 hours |
| Total cost | about $240 |
| Cost per task | about $0.16 |
| Mean wall-clock time per task | 1,940 seconds |
| Mean LLM requests per task | 144.2 |

## 4. Task setup

The target is all 1,507 CyberGym Level 1 tasks. Tasks were generated with the official `cybergym.task.gen_task --difficulty level1` as-is.

### 4.1 Dynamic environment

**We provided a dynamic analysis environment** (FAQ Q5). The following are mounted read-only into the agent's container.

- **`/out`**: the vulnerable binaries taken from the vulnerable (`-vul`) image.
- **`/seeds`**: the corpus seed files from the vulnerable (`-vul`) image where they exist; otherwise generic seeds (magic bytes for common file formats) that carry no vulnerability information.

This lets the agent perform dynamic analysis such as running, fuzzing, and debugging the target binary.

### 4.2 Leak prevention

To prevent reward hacking, we did the following (FAQ Q2 / Q5).

- **The fixed (`-fix`) image is never given to the agent.** The fixed build is used only for host-side grading after submission, so the agent cannot diff against it during a run.
- **`.git` is excluded from `repo-vul.tar.gz` at any depth**, to prevent leaks from the fix history.
- **`/tmp/poc` (the reference PoC) is unreachable from the agent environment.** The paths that produce `/out` and `/seeds` do not traverse the location where the reference PoC is stored.
- **The training data contains nothing that overlaps the 1,507 evaluated tasks.** Reference PoCs and fixed sources were not used for training either.
- We mechanically audited the execution logs of all 1,507 tasks and confirmed **zero calls to web tools (`WebSearch` / `WebFetch`)** and **zero cases where the target repository's git history could be retrieved** (`git log` and similar were invoked, but none of them returned any output).

### 4.3 Network policy

**We used the official CyberGym firewall** (FAQ Q1). The agent container sits on an internal Docker network with no NAT, and outbound traffic is limited to a squid domain allowlist.

- **The allowlist is unchanged from the official default** (apt / pip / LLM API endpoints). No custom domains were added.
- **`WebSearch` and `WebFetch` are disabled on the agent side.** Internet search is also explicitly forbidden in the prompt, doubling up with the firewall.
- LLM inference is served by our own vLLM, reached over an SSH tunnel as `127.0.0.1`, so no third-party inference API is used.

Together these close off any route by which the agent could pull answers directly from the target project's issue tracker, commit history, release notes, and the like.

### 4.4 Submission protocol

The agent may submit several PoCs during a task, but it **explicitly declares exactly one sha256 as its final answer**. Grading looks only at that single declared PoC (the final-submission scheme of FAQ Q3). If a run ends without a declaration, the last submitted PoC is taken as the final answer. Intermediate submissions have no effect on grading.

Each task was run once (Pass@1). Re-runs were limited to infrastructure failures such as API call errors or container startup failures.
