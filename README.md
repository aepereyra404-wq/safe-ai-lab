# Safe AI Lab

A minimal starter kit to run **Pi** agents safely inside an **openshell** sandbox, leveraging **Ollama Cloud** for LLM inference and **GitHub** for source‑control capabilities.

> **Goal** – Provide a reproducible, low‑trust environment where agents can execute code, perform network calls, and interact with version‑control systems while being confined by a strict sandbox policy.

---

## Table of Contents
1. [Architecture Overview](#architecture-overview)
2. [Prerequisites](#prerequisites)
3. [Installation & Setup](#installation--setup)
4. [Running the Lab](#running-the-lab)
5. [Customization](#customization)
6. [Verification & Testing](#verification--testing)
7. [Teardown & Cleanup](#teardown--cleanup)
8. [Troubleshooting](#troubleshooting)
9. [Repository Layout](#repository-layout)
10. [Contributing & License](#contributing--license)

---

### Architecture Overview
```
+-----------------+      +----------------------+      +-----------------+
|   Pi CLI        | →    | openshell sandbox    | →    | Ollama Cloud    |
| (agent code)   |      | (Docker container)  |      | (LLM service)  |
+-----------------+      +----------------------+      +-----------------+
                               |
                               └─► GitHub API (source control)
```
*Pi reads its configuration from `~/.pi/agent/` inside the sandbox. The sandbox policy is defined in `openshell-configs/`.*

---

### Prerequisites
| Tool | Minimum version | Install command |
|------|----------------|-----------------|
| `openshell` CLI | v0.10.0 | `curl -sSfL https://openshell.dev/install.sh \| sh` |
| `pi` CLI | v1.2.0 | `npm i -g @earendil-works/pi` |
| Docker/Podman | 20.10+ | (see Docker docs) |
| `Ollama` API key | – | Export `export OLLAMA_API_KEY=your_key` |
| `GitHub` token | – | Export `export GITHUB_TOKEN=your_token` |

> **Tip:** Store secrets in a `.env` file (not committed) or use a secret manager.

---

### Installation & Setup
1. **Clone the repository**
   ```bash
   git clone https://github.com/yourorg/safe-ai-lab.git
   cd safe-ai-lab
   ```
2. **Export required environment variables**
   ```bash
   : "${OLLAMA_API_KEY:?Set OLLAMA_API_KEY}" 
   : "${GITHUB_TOKEN:?Set GITHUB_TOKEN}"   # needed for the GitHub provider
   ```
3. **Import the provider profiles**
   ```bash
   # Ollama Cloud provider (already in the repo)
   openshell provider profile import -f openshell-configs/ollama-provider.yaml

   # GitHub provider – added for source‑control support
   openshell provider profile import -f openshell-configs/github-provider.yaml --global
   ```
4. **Create provider instances**
   ```bash
   openshell provider create --name pi-ollama    --type pi-ollama    --from-existing
   openshell provider create --name github      --type github       --from-existing
   ```
5. **Create the sandbox**
   ```bash
   openshell sandbox create \
       --name pi-container \
       --from ghcr.io/nvidia/openshell-community/sandboxes/pi:latest \
       --policy ./openshell-configs/ollama-policy.yaml \
       --provider pi-ollama \
       -- pi
   ```
6. **Upload Pi configuration files into the sandbox**
   ```bash
   openshell sandbox upload pi-container ./pi-configs/models.json   /sandbox/models.json
   openshell sandbox upload pi-container ./pi-configs/settings.json /sandbox/settings.json
   ```
7. **Enter the sandbox and place the config where Pi expects it**
   ```bash
   openshell sandbox connect pi-container
   # inside the sandbox shell:
   mkdir -p ~/.pi/agent
   cp /sandbox/models.json   ~/.pi/agent/models.json
   cp /sandbox/settings.json ~/.pi/agent/settings.json
   ```
8. **Test the agent**
   ```bash
   pi chat "Hello, world!"
   ```
   You should see a response from the selected Ollama model.

---

### Customization
- **Change the default model** – Edit `pi-configs/models.json` (change `defaultModel` in `settings.json` or adjust the `providers` block).
- **Add more file‑system permissions** – Edit `openshell-configs/ollama-policy.yaml` under `read_write:` (e.g., add `/home/pi/data`).
- **Extend network policy** – Add additional allowed endpoints under `network_policies:` if you need services beyond Ollama.
- **Enable Git operations** – The GitHub provider profile is already imported; you can now run `git clone https://github.com/<repo>.git` inside the sandbox (read‑only by default).

---

### Verification & Testing
```bash
# Simple sanity check – Pi can evaluate code
pi eval '2 + 2'           # should output 4

# Verify filesystem isolation
touch /tmp/test && ls /tmp/test   # succeeds (read_write)
if touch /etc/forbidden 2>/dev/null; then echo "ERROR: /etc is writable"; else echo "/etc correctly read‑only"; fi

# Verify GitHub access (requires the token above)
git clone https://github.com/octocat/Hello-World.git /tmp/hw && echo "Git clone succeeded"
```

---

### Teardown & Cleanup
```bash
openshell sandbox destroy pi-container
openshell provider destroy pi-ollama
openshell provider destroy github
```

---

### Troubleshooting
| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| `OLLAMA_API_KEY not set` | Missing env var | `export OLLAMA_API_KEY=…` |
| `Network policy denied` | Endpoint not in policy | Add it to `network_policies` in `ollama-policy.yaml` |
| `Git clone fails (authentication)` | GitHub token not exported or wrong scope | Export `GITHUB_TOKEN` with `repo` read permissions |
| `/etc` write succeeds | Policy mis‑configuration | Ensure `/etc` is only in `read_only` list |

---

### Repository Layout
```
.
├─ README.md                ← this guide
├─ openshell-configs/       ← OpenShell sandbox policy & provider profiles
│   ├─ ollama-policy.yaml   ← Filesystem & network policy
│   ├─ ollama-provider.yaml← Ollama Cloud provider profile
│   └─ github-provider.yaml ← GitHub API & Git operations profile
└─ pi-configs/             ← Pi runtime configuration
    ├─ models.json          ← Provider / model mapping
    └─ settings.json        ← Default model & provider
```

---

### Contributing & License
- **License:** MIT (see `LICENSE`).
- **How to contribute:** Fork → create a feature branch (`git checkout -b feat/…`) → run `./scripts/setup.sh` to verify → open a PR.
- **Security reporting:** Please email security@yourorg.com with details of any vulnerability.

