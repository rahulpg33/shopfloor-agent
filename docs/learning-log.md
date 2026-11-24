# shopfloor-agent: Learning Log

A running record of what I built, why, what went wrong and what I measured.
I add a section after every milestone. Everything here comes from runs I actually did.

**Author:** Rahul Girijan · **Started:** 9 Nov 2026 · **Last updated:** 22 Nov 2026

---

## Contents

1. [The project in one page](#1-the-project-in-one-page)
2. [My machine](#2-my-machine)
3. [M0: Setting up the workshop](#3-m0-setting-up-the-workshop)
   - [0.0 Preparation: freeing disk space](#00-preparation-freeing-disk-space)
   - [0.1 WSL2](#01-wsl2-linux-inside-windows)
   - [0.2 Docker](#02-docker)
   - [0.3 Limiting RAM with .wslconfig](#03-limiting-ram-with-wslconfig)
   - [0.4 Ollama: a local AI model](#04-ollama-a-local-ai-model)
   - [0.5 Python with uv](#05-python-with-uv)
   - [0.6 Git](#06-git)
   - [0.7 GitHub repo and first commit](#07-github-repo-and-first-commit)
4. [Measurements](#4-measurements)
5. [Problems I hit and how I fixed them](#5-problems-i-hit-and-how-i-fixed-them)
6. [Command cheat sheet](#6-command-cheat-sheet)
7. [Glossary](#7-glossary)
8. [Interview talking points](#8-interview-talking-points)
9. [Roadmap and status](#9-roadmap-and-status)

---

## 1. The project in one page

**What:** a maintenance assistant for a production line that runs entirely on my laptop.
An operator or technician asks something like *"Is machine CNC-03 OK? If not, open a ticket."*
The agent checks real data through tools, respects the user's role, and every request is traced and scored.

**Why this project:** my previous work (an LLM app at 3Kubik GmbH) already covers RAG and Azure deployment.
This project fills the gaps I see in junior AI Engineer job ads in Germany:
a real agent framework, evals with scores, tracing, a self-hosted LLM, FastAPI, PostgreSQL and access control.

**Architecture (planned):**

```
 User (Swagger UI / curl)
   │ 1. log in → token with role: operator | technician
   ▼
 FastAPI  (/login  /chat  /health  /metrics)
   │ 2. checks the token, keeps only the tools this role may use
   ▼
 LangGraph agent ──── thinks with ───► Ollama (Qwen 3.5 4B, on my CPU)
   │ 3. calls tools over MCP
   ▼
 My MCP server: get_machine_status · list_open_tickets · check_spare_part · create_ticket*
   │ 4. reads and writes                               (* technician only)
   ▼
 PostgreSQL (machines · tickets · spare_parts · audit_log)

 Every request → trace in Langfuse · Every push → GitHub Actions tests · 25 eval cases → score table
```

**Core concepts**

| Concept | Plain meaning | In this project |
|---|---|---|
| LLM | A model that reads text and writes text | Qwen 3.5 4B in Ollama |
| Tool | A normal function the model may ask to run | `get_machine_status("CNC-03")` |
| Agent | A loop: think → call a tool → read the result → think again → answer | LangGraph |
| MCP server | A standard way to offer tools to any AI app | My own server (MCP Python SDK v2) |
| Eval | An automatic test for AI behaviour that gives a score | 25 cases, before/after table |
| Trace | A recording of every step of one request | Langfuse Cloud (free tier) |
| RBAC | Role-based access control | Operator reads, technician may create tickets |

**Rule I follow:** never invent results. Every number in this repo comes from a run I did.

---

## 2. My machine

The hardware decides the design: no GPU means CPU inference, a small model and short answers.

| Part | Value | Consequence |
|---|---|---|
| Laptop | Dell Inspiron 14 5410, Windows 11 | Everything local, cost 0 EUR |
| CPU | Intel Core i5-11320H | Runs a 4B quantized model at ~9.5 tokens/s |
| RAM | 16 GB | WSL2/Docker capped at 5 GB; model uses ~3.1 GB |
| GPU | Intel Iris Xe (integrated) | No CUDA → Ollama on CPU, not vLLM |
| Disk C: | 252 GB partition | Keep 20–30 GB free for images and models |

**RAM budget while working:** Windows + VS Code + apps ~6 GB · model ~3–4 GB · WSL2 + Docker max 5 GB.

---

## 3. M0: Setting up the workshop

**Goal of M0:** install and verify every tool the project depends on, before writing project code.
**Status:** done (22 Nov 2026).

### 0.0 Preparation: freeing disk space

- **Problem:** C: had only **10.8 GB free** of 252 GB. Docker images and AI models need far more.
- **How I found the space:** Settings → System → Storage. The "Temporary files" and "Other" categories
  showed the big items: old installers and videos in Downloads, a full Recycle Bin, unused apps, old Python environments.
- **Before deleting:** I checked that my thesis files were backed up. Trained YOLOv8 weights and the final report
  are in `F:\Thesis`; I compared files by exact size to confirm the copies were identical.
- **Removed:** Recycle Bin contents, Siemens NX + its installer, Microsoft Clipchamp, Anaconda3, an old
  `yolov5_env`, and old videos and installers.
- **Lessons:** always verify backups before deleting; `Shift + Delete` skips the Recycle Bin;
  uninstallers need an admin "Yes" that only the user can give.

### 0.1 WSL2 (Linux inside Windows)

**What:** Windows Subsystem for Linux 2 runs a **real Linux kernel** in a lightweight virtual machine inside Windows.
A **distribution (distro)** is the Linux flavour on top; mine is **Ubuntu**.

**Why:** Docker containers are a Linux technology, and most servers and AI tools run on Linux.
WSL**1** only translated Linux calls and could not run Docker; WSL**2** has a real kernel, so it can.

**Background:** the **kernel** is the core of an operating system that talks to the hardware.
Windows and Linux have different kernels, so Linux programs cannot run directly on Windows.

**What I did**

| Step | Command | Why |
|---|---|---|
| 1 | PowerShell **as administrator** | Installing WSL changes Windows features |
| 2 | `wsl --status` | Check first: it was not installed |
| 3 | `wsl --install` | Installed WSL 3.0.5 and enabled *Virtual Machine Platform* |
| 4 | Restart | Windows only switches features on at boot |
| 5 | `wsl --install -d Ubuntu` | Ubuntu didn't install automatically; `-d` = distribution |
| 6 | Created Linux user + password | Linux has its own accounts; the password is invisible while typing |
| 7 | `uname -a` | Proof: `Linux … 6.18.58.1-microsoft-standard-WSL2 … x86_64` |
| 8 | `wsl -l -v` | `Ubuntu  Running  2` → version 2 confirmed |

**Reading the Linux prompt:** `rahul@ValkNut:/mnt/c/WINDOWS/system32$` = user @ computer : current folder, `$` = ready.
`/mnt/c` is the Windows C: drive seen from Linux; my Linux home is `~` = `/home/rahul`.

**Summary:** WSL2 = a real Linux kernel in a light VM inside Windows. I need it because Docker and most AI/server tools are built for Linux.

### 0.2 Docker

**Problem it solves:** "it works on my machine" — code fails elsewhere because the environment differs.

**What:** Docker packs a program with everything it needs into a **container** that runs the same everywhere.

| Word | Meaning | Analogy |
|---|---|---|
| Image | Read-only template: program + dependencies | Recipe / a class |
| Container | A running copy made from an image | A cooked dish / an object |
| Docker Hub | Public registry of images | App store |
| Pull | Download an image | — |
| Dockerfile | Instructions to build your own image | Writing a recipe |
| Docker Compose | One file that starts several containers together | A full menu |
| Port mapping (`5432:5432`) | Opens a door from my laptop into a container | — |
| Volume | Storage outside the container that survives restarts | — |
| Tag (`:latest`, `:17`) | An image's version label; pin exact versions in projects | — |

**Container vs. virtual machine:** a VM has its own full OS and kernel (GBs, minutes to start);
a container shares the host's Linux kernel (MBs, seconds). That is why Docker on Windows needs WSL2.

**How it runs on Windows:** Docker Desktop (the app) → Docker Engine / **daemon** (does the work) inside WSL2.
The `docker` command is the **client** that sends requests to the daemon.

**What I did:** installed Docker Desktop 4.99.0 with the WSL 2 backend, skipped sign-in (free for personal use),
enabled *Settings → Resources → WSL integration → Ubuntu* so `docker` works in Ubuntu too.

**Verified (from Ubuntu):** `docker --version` → 29.9.3 · `docker run hello-world` → "Hello from Docker!"
· `docker images` → 1 image · `docker ps -a` → 2 containers from that 1 image (status `Exited (0)` = success)
· `docker container prune` removed the stopped containers.

**Summary:** an image is the recipe, a container is the running copy, Compose starts several containers together.

### 0.3 Limiting RAM with .wslconfig

**Problem:** by default WSL2 may use up to half the RAM (8 GB). With the model (~3–4 GB, outside WSL) and
Windows (~6 GB), that adds up to more than 16 GB.

**Solution:** a config file `C:\Users\rahul\.wslconfig`:

```ini
[wsl2]
memory=5GB
swap=4GB
```

- `memory` = hard limit for WSL2 + Docker. `swap` = SSD space used as slow overflow instead of crashing.
- Changes only apply after `wsl --shutdown`.

**Verified:** `free -h` in Ubuntu: before **7.6Gi** RAM / 2.0Gi swap → after **4.8Gi** / **4.0Gi**.

**Debugging story:** the first attempt had no effect. `ls -la /mnt/c/Users/rahul/ | grep -i wslconfig`
showed the file size was **0 bytes** — Notepad had created it but nothing was saved.
I wrote it from Linux instead (`cat > file << 'EOF' … EOF`), confirmed 27 bytes with `cat`, restarted WSL, and it worked.

**Summary:** `.wslconfig` limits WSL2/Docker RAM. Check a file's size, not just its name.

### 0.4 Ollama: a local AI model

**Self-hosting:** the model runs on my own machine — no internet, no API key, no per-request cost,
and no data leaves the computer. This matters for companies with GDPR/DSGVO or confidentiality needs.

**What a model is physically:** one big file of learned numbers (parameters). Running it = loading it into RAM
and doing a lot of maths per token.

**What Ollama does:** downloads models (`ollama pull`), runs them on CPU/GPU, and serves an API at
`http://localhost:11434`. My agent will call that address the way an app calls a cloud API.
Inside, Ollama uses **llama.cpp** as its runner.

**Model:** `qwen3.5:4b` (3.3 GB on disk). Chosen because it fits my CPU/RAM and supports **tool calling**.

| Term | Meaning |
|---|---|
| Parameters (4b) | 4 billion learned numbers; more = usually smarter but slower and heavier |
| Quantization (Q4_K_M) | Numbers squeezed from 16 to ~4 bits → ~4× smaller/faster, small quality loss |
| Token | A piece of text, ~¾ of a word |
| Tool calling | The model can request a function call instead of plain text — the basis of an agent |
| Context window (4096) | How many tokens the model can see at once (prompt + history + answer) |
| Cold / warm start | First load from disk into RAM vs. model already in RAM |
| Thinking mode | The model writes its reasoning before answering |

**Key insight:** response time ≈ tokens written ÷ tokens per second. On a CPU, speed per token is fixed,
so the way to go faster is to make the model write less. Thinking mode made a 3-sentence answer take
2.5 minutes because ~90% of the tokens were reasoning. Turning it off cut that to ~10 seconds,
but the short answer was less accurate — a trade-off the evals in M7 will measure.

(Full numbers in [Measurements](#4-measurements).)

### 0.5 Python with uv

**Problem:** "dependency hell" — projects need different package versions and break each other in one shared Python.
I had seen this with several conda environments (~11 GB) during my thesis.

**Solution:** one **virtual environment** per project, managed by **uv**:

| Old way | uv |
|---|---|
| Download Python from python.org | `uv python install 3.12` |
| `python -m venv .venv` + activate | Automatic; `uv run …` uses it |
| `pip install x` | `uv add x` |
| `requirements.txt` | `pyproject.toml` (what I asked for) + `uv.lock` (exact pinned versions) |

- `pyproject.toml` = the project's ID card; I change it through `uv add`.
- `uv.lock` = every package pinned to one version, written by uv, committed to Git → reproducible setup.
- `.python-version` = tells uv which Python this project uses.

**Installed:** uv 0.13.9, Python 3.12.16. The PC also has old Python 3.11.1 and 3.10.8, so I always use `uv run`
instead of plain `python` to get the right interpreter and packages.

**Installer command explained:** `irm` downloads a script, `| iex` runs it, `-ExecutionPolicy ByPass` allows it once.
Only do this with scripts from official sources. After installing, open a **new** terminal so the updated **PATH** is read.

**Decision:** I write the code on Windows (PowerShell + VS Code) because Ollama runs there; Linux is used through Docker.

### 0.6 Git

**What:** version control — snapshots (commits) of the whole project over time, each with a message.
**Git** is the program on my PC; **GitHub** is the website that hosts repos online.

| Word | Meaning |
|---|---|
| Repository | A folder Git tracks; history lives in the hidden `.git` folder |
| Commit | One snapshot + message; identified by a hash like `c43663a` |
| Staging (`git add`) | Choosing which changes go into the next commit |
| Branch | A separate line of work; `main` is the main one |
| Remote / `origin` | The copy on GitHub |
| Push / pull | Upload / download commits |
| `HEAD -> main` | "I am here" |
| `.gitignore` | Files Git must never track (`.venv`, `.env`, caches) |

**Daily workflow:** edit → `git status` → `git add` → `git commit -m "…"` → `git push`

**Rules:** clear commit messages (recruiters read them); **never commit secrets** — they stay in the history even after deletion.

**Setup:** updated Git 2.39.0 (Dec 2022) → **2.56.1** with `winget upgrade --id Git.Git -e`, then:

```powershell
git config --global user.name "Rahul Girijan"
git config --global init.defaultBranch main
git config --global core.autocrlf true   # Windows/Linux line endings
```

### 0.7 GitHub repo and first commit

- Created `rahulpg33/shopfloor-agent` (public) on GitHub with a README, **MIT license** and the **Python .gitignore**.
- Cloned it to `C:\Users\rahul\projects\shopfloor-agent` (not Desktop/Documents, which OneDrive may sync).
- `uv init --python 3.12` created a **src-layout package**: code lives in `src/shopfloor_agent/`, and
  `[project.scripts]` maps the command `shopfloor-agent` to `shopfloor_agent:main`. Run it with `uv run shopfloor-agent`.
  (Project name uses a hyphen, the Python package an underscore.)
- Added `.env.example` (safe example settings; the real `.env` is never committed).
- `git status` did **not** list `.venv` → `.gitignore` works.
- First commit `c43663a` "Set up Python 3.12 project with uv and .env.example", pushed: `20c5078..c43663a main -> main`.

---

## 4. Measurements

All measured on my laptop (Intel i5-11320H, CPU only, 16 GB RAM), 20 Nov 2026,
Ollama with `qwen3.5:4b` (Q4_K_M), prompt: *"Explain what a PLC is in 3 sentences."*

| Metric | Thinking ON | Thinking OFF (`--think=false`) |
|---|---|---|
| Total duration | 151.3 s | **10.3 s** |
| Tokens generated (eval count) | 1,300 | 95 |
| Generation speed (eval rate) | 9.35 tokens/s | 9.65 tokens/s |
| Load duration | 11.6 s (cold start) | 0.008 s (warm start) |

- RAM used by the loaded model (`ollama ps`): **3.1 GB**, `100% CPU`, context 4096.
- Observation: the answer without thinking contained an inaccurate phrase ("self-monitoring tasks"); with thinking it was clearer.

**README-ready line:** *Local inference on an Intel i5-11320H (CPU only): Qwen 3.5 4B generates ~9.5 tokens/s using 3.1 GB RAM.
Disabling thinking mode cut a 3-sentence answer from 151 s to 10.3 s (1,300 → 95 generated tokens).*

---

## 5. Problems I hit and how I fixed them

| Problem | Cause | Fix | Lesson |
|---|---|---|---|
| C: nearly full (10.8 GB free) | Old installers, videos, Recycle Bin, unused apps | Storage settings, verified backups, removed ~50 GB | Check backups before deleting |
| `wsl -l -v`: "no installed distributions" | Ubuntu didn't install automatically | `wsl --install -d Ubuntu` | Read what the tool reports |
| `^V: command not found` | Ctrl+V doesn't paste in a Linux terminal | Right-click or Ctrl+Shift+V | — |
| `uname -e` error | Wrong option | `uname -a`; use `--help` | Error messages usually name the fix |
| `free -h` still showed 7.6Gi | `.wslconfig` was 0 bytes (never saved) | Wrote it with `cat > … << 'EOF'`, `wsl --shutdown` | Check file size; config needs a restart |
| `free -h` didn't run after `wsl` | Two commands pasted at once; the second was lost | Run commands one at a time | One command → read output → next |
| `ollama ps` empty | Ollama unloads models ~5 min after last use | Run it right after a request | Models load on demand |
| Thinking answer took 2.5 min | 90% of tokens were reasoning | `--think=false` → 10.3 s | Fewer tokens = faster on CPU |
| `uv: not recognized` risk | Old terminal has the old PATH | Open a new terminal | PATH is read when a terminal starts |
| `git clone`: "Repository not found" | Repo not created on GitHub yet | Created it, then cloned | GitHub also says this for private repos |
| "not a git repository" | Ran git commands in the parent folder | `cd shopfloor-agent` | Check the prompt's folder |
| `uv run main.py`: program not found | uv 0.13.9 creates a src-layout package, not `main.py` | `uv run shopfloor-agent` | Read what the tool actually created |
| "LF will be replaced by CRLF" | `core.autocrlf true` | Nothing; it's informational | Warnings ≠ errors |

---

## 6. Command cheat sheet

| Command | Where | What it does |
|---|---|---|
| `wsl --status` / `wsl -l -v` | PowerShell | WSL status / list distros + version |
| `wsl` / `exit` | PowerShell / Ubuntu | Enter Linux / leave it |
| `wsl --shutdown` | PowerShell | Stop all Linux systems (applies `.wslconfig`) |
| `uname -a` · `free -h` | Ubuntu | Kernel info · memory in GB |
| `ls -la` · `cat file` · `cmd \| grep text` | Ubuntu | List all files · print a file · filter output |
| `cat > file << 'EOF' … EOF` | Ubuntu | Write text into a file |
| `docker run hello-world` | Either | Pull and run a test container |
| `docker images` · `docker ps -a` | Either | Stored images · all containers |
| `docker container prune` | Either | Remove stopped containers |
| `ollama list` · `ollama ps` | PowerShell | Models on disk · loaded models + RAM |
| `ollama pull <m>` · `ollama rm <m>` | PowerShell | Download · delete a model |
| `ollama run <m> --verbose [--think=false]` | PowerShell | Chat + speed statistics |
| `uv python install 3.12` · `uv python list` | PowerShell | Install / list Pythons |
| `uv init` · `uv add <pkg>` · `uv run <cmd>` | Project folder | Create project · add package · run in the project's env |
| `git status` · `git add .` · `git commit -m "…"` · `git push` | Project folder | Daily Git workflow |
| `git log --oneline` · `git remote -v` | Project folder | History · where origin points |
| `dir -Force` · `type file` · `code .` | PowerShell | List incl. hidden · print a file · open folder in VS Code |

---

## 7. Glossary

- **Admin / elevated:** running with rights to change system settings (the blue "Yes" prompt).
- **Client / daemon:** the command I type vs. the background service that does the work.
- **Context window:** how many tokens a model can see at once.
- **Distro:** a complete Linux package (kernel + tools), e.g. Ubuntu.
- **Environment variable:** a named value the shell knows, e.g. `$env:USERPROFILE`.
- **Exit code 0:** the program finished successfully; other numbers mean errors.
- **Flag:** an option after a command, e.g. `--verbose`, `-d`.
- **Hash (commit ID):** a unique fingerprint of a commit, shown short like `c43663a`.
- **Kernel:** the core of an OS that talks to the hardware.
- **Localhost / port:** my own computer / a numbered door on it (Ollama: 11434, PostgreSQL: 5432).
- **PATH:** the list of folders where the system looks for programs.
- **Pipe (`|`):** sends one command's output into the next command.
- **Quantization:** storing model numbers with fewer bits to save RAM and time.
- **Redirect (`>`):** sends output into a file instead of the screen.
- **Registry:** a store to download images/packages from (Docker Hub, PyPI).
- **Swap:** disk space used as slow overflow memory.
- **Token:** a piece of text, ~¾ of a word.
- **Virtual environment:** a private set of packages for one project.
- **Virtual machine:** a software computer with its own OS running inside a real one.

---

## 8. Interview talking points

- **Why self-host an LLM?** Many German companies cannot send data to cloud APIs (GDPR/DSGVO, trade secrets).
  Running the model locally keeps all data on the machine.
- **Why a 4B model?** It fits a 16 GB CPU-only laptop (3.1 GB RAM, ~9.5 tokens/s). I accept weaker answers
  and measure the mistakes with evals instead of hiding them.
- **How did you make it faster?** I measured first: 1,300 of the tokens were "thinking". Response time is roughly
  tokens ÷ speed, so I disabled thinking: 151 s → 10.3 s. Then I noted the quality cost to test properly in evals.
- **Container vs. VM?** A VM carries its own OS and kernel; a container shares the host's Linux kernel,
  so it's smaller and starts in seconds. That's also why Docker on Windows runs inside WSL2.
- **Why uv and a lock file?** One isolated environment per project, and `uv.lock` pins every version,
  so CI and other developers get exactly my setup.
- **How do you keep secrets out of Git?** `.env` is in `.gitignore`; `.env.example` documents the settings.
  A committed secret stays in the history even after deletion, so prevention matters.
- **A debugging example:** my WSL memory limit didn't apply. I checked the result (`free -h`), then the input
  (the file was 0 bytes), fixed it, and verified again.

---

## 9. Roadmap and status

| Milestone | Content | Status |
|---|---|---|
| **M0** | WSL2, Docker, RAM limit, Ollama + speed test, uv + Python, Git, GitHub repo | ✅ Done (22 Nov 2026) |
| M1 | PostgreSQL in Docker Compose + fictional factory data | Next |
| M2 | My MCP server with 4 tools, tested with MCP Inspector | |
| M3 | First LangGraph agent using the MCP tools | |
| M4 | FastAPI + Swagger, pytest, GitHub Actions CI | |
| M5 | Login with roles (JWT), tool filtering, audit log | |
| M6 | Langfuse tracing | |
| M7 | 25 eval cases (rule checks + LLM-as-judge) → baseline score | |
| M8 | Improve, re-run evals, before/after table, README, demo GIF | |

**Open checks from M0:** confirm `uv run shopfloor-agent` prints the greeting, and `type .env.example` shows the 3 lines.
