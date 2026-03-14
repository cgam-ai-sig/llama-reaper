# tuku

GPU cleanup tools for shared servers running llama.cpp. Automatically detects and kills idle `llama-server` / `llama-cli` processes that are blocking the GPU.

Solves the "someone forgot to stop their llama-server and now nobody else can use the GPU" problem. Ollama handles this with automatic model unloading — llama.cpp doesn't, so we built this.

## What's included

| Tool | Purpose |
|------|---------|
| `tuku` | Watchdog that scores idle llama.cpp processes and kills them on a schedule |
| `tuku-status` | GPU dashboard showing everything running on the GPU (llama.cpp + Ollama) |
Ollama processes are **never** targeted. Only `llama-server`, `llama-cli`, and `llama-cpp` processes are eligible for reaping.

## Quick start

```bash
git clone https://github.com/cgam-ai-sig/tuku.git
cd tuku
sudo ./install.sh --with-reaper
```

This installs the tools to `/usr/local/bin/` and sets up the system-wide reaper (checks every 5 min, ignores processes younger than 1 hour, kills after 30 min of continuous idleness).

If you just want the tools without the automated reaper:

```bash
sudo ./install.sh
```

## How the reaper works

Each run, the reaper scores every llama.cpp process on several idle signals:

| Signal | Points | What it means |
|--------|--------|---------------|
| No GPU memory allocated | +3 | Process isn't on the GPU at all |
| GPU memory held, 0% utilization | +3 | Sitting on VRAM but not computing |
| No CPU time change since last check | +3 | Process hasn't done any work |
| No network connections (for servers) | +2 | Nobody is connected to this server |
| Running > 2 hours | +1 | Age bonus |
| Running > 8 hours | +2 | Larger age bonus (replaces the +1) |

**Score >= 6** → process is considered idle. The reaper records a `.alive` timestamp tracking the last time the process showed activity.

**Score >= 8** → KILL candidate. The reaper checks how long the process has been continuously idle (based on the `.alive` timestamp). If it's been idle for longer than `--max-idle` minutes, the process is killed (SIGTERM, then SIGKILL after 10s if needed).

If the score drops below 6 between runs (e.g., someone connects), the `.alive` timestamp is reset. The idle clock starts over.

### The `--min-age` age gate

`--min-age` (default: 60 minutes) is an **age gate**. Processes younger than `--min-age` minutes are completely ignored, regardless of their idle score. This prevents the reaper from killing a server that was just started and hasn't received its first request yet.

### The `--max-idle` idle duration

`--max-idle` (default: 30 minutes) is the **continuous idle duration** before a process is killed. The reaper tracks each process's last activity via `.alive` timestamp files. Once a process has been continuously idle (score >= 8) for `--max-idle` minutes, it gets killed.

### Timeline example

With the default `--interval 5 --min-age 60 --max-idle 30`:

1. **t=0**: User starts `llama-server`, walks away.
2. **t=5–55**: Reaper runs every 5 min. Process is younger than `--min-age` (60m). **Skipped.**
3. **t=60**: Reaper runs. Process is 60 minutes old — now eligible. Scores high. `.alive` timestamp set to now (first idle detection).
4. **t=65–85**: Reaper keeps running. Score stays high. Idle duration accumulates.
5. **t=90**: Reaper runs. Process has been continuously idle for 30 min (>= `--max-idle`). **KILLED.**

Worst case: an idle process survives ~90 minutes (60m age gate + 30m idle duration).

## Configuration

### Install the reaper cron

```bash
# System-wide (all users, needs root)
sudo tuku install --system --interval 5 --min-age 60 --max-idle 30

# User-only (just your processes)
tuku install --interval 5 --min-age 60 --max-idle 30
```

**System mode** creates `/etc/cron.d/tuku` and runs as root, managing all users' processes.

**User mode** adds an entry to your personal crontab, only managing your own processes.

### Remove the reaper cron

```bash
tuku uninstall           # user cron
tuku uninstall --system  # system cron
```

### One-off checks

```bash
tuku --dry-run --verbose  # see what would happen, don't kill anything
tuku list                 # show all llama.cpp processes with scores
tuku --force              # skip --max-idle duration check, kill immediately if score >= 8
```

## tuku-status

GPU dashboard with four output modes:

```bash
tuku-status              # full color dashboard
tuku-status --compact    # single-line summary
tuku-status --json       # machine-readable JSON
tuku-status --no-color   # no ANSI codes (for piping/logging)
```

Example output (`--no-color`):

```
━━━ GPU ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  GPU      NVIDIA GeForce RTX 4090
  VRAM     14227 / 24564 MiB  ━━━━━━━━━───── 57%
  Temp     52°C    Power   138W / 450W    Util  12%

━━━ Ollama ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Status   ● running (1 models loaded)
  Active   llama3.1:8b (5 GB, until 4m)

━━━ llama.cpp ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  PID      USER     VRAM        MODEL                          PORT    UPTIME
  185334   alice    8413 MiB    Qwen2.5-Coder-32B-Q4_K_M.gguf 8080    2h 14m
  191207   bob      5120 MiB    mistral-7b-v0.3.Q5_K_M.gguf   9090    45m

━━━ Reaper ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Status   ● active (every 5m, min-age 60m, max-idle 30m)
```

Shows GPU stats, Ollama status with loaded models and TTL, all llama.cpp processes with user/PID/VRAM/model/port/uptime, and whether the reaper is installed.

## Uninstall

```bash
sudo ./install.sh uninstall          # remove tools from /usr/local/bin/
sudo ./install.sh uninstall --purge  # also remove /var/lib/tuku/ state files
```

## Requirements

- Linux (tested on Ubuntu 24.04)
- bash 4+
- nvidia-smi (NVIDIA GPU drivers)
- coreutils, procps, iproute2 (`ss`), curl
- No root needed for day-to-day use (only for install and system-wide reaper setup)

## License

MIT
