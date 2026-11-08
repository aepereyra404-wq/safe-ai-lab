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
   openshell provider profile import -f openshell-configs/profiles/OllamaProvider.yaml

   # GitHub provider – added for source‑control support
   openshell provider profile import -f openshell-configs/profiles/GithubProvider.yaml --global
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
       --policy ./openshell-configs/baseline_policy.yaml \
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
- **Add more file‑system permissions** – Edit `openshell-configs/baseline_policy.yaml` under `read_write:` (e.g., add `/home/pi/data`).
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

### Troubleshooting OpenShell GitHub Authentication

This section describes how to verify GitHub authentication when using Pi inside an OpenShell sandbox to work with personal GitHub repositories.

The setup uses:

- An OpenShell GitHub provider containing a fine‑grained GitHub Personal Access Token (PAT).
- The built‑in `github` provider profile.
- Git HTTPS authentication through an OpenShell‑managed credential placeholder.
- Repository‑scoped permissions and network policies.

#### 1. Verify the GitHub PAT

Before troubleshooting OpenShell, verify that the PAT itself is valid.

On the host machine, export the token:

```bash
export GITHUB_TOKEN="github_pat_..."
```

Test authentication against GitHub:

```bash
curl -sS -o /dev/null \
  -w 'HTTP %{http_code}\n' \
  -H "Authorization: Bearer $GITHUB_TOKEN" \
  -H "Accept: application/vnd.github+json" \
  https://api.github.com/user
```

**Expected result:**

```
HTTP 200
```

A `401` response indicates an invalid, expired, or otherwise rejected token.

Verify repository access:

```bash
curl -sS \
  -H "Authorization: Bearer $GITHUB_TOKEN" \
  -H "Accept: application/vnd.github+json" \
  https://api.github.com/repos/OWNER/REPO
```

For Git operations, the fine‑grained PAT should have:

- Repository access restricted to the intended repository.
- **Contents:** Read and write.
- **Metadata:** Read‑only (automatically granted).

If the repository belongs to an organization, check whether the token requires organization approval.

#### 2. Verify the OpenShell provider

Check that the GitHub provider exists:

```bash
openshell provider get my-github
```

**Expected configuration includes:**

```
Name: my-github
Type: github
Credential keys: GITHUB_TOKEN
```

Verify that the provider is attached to the sandbox:

```bash
openshell sandbox provider list pi-container
```

The GitHub provider should appear in the list.

#### 3. Verify the GitHub provider profile

Inspect the built‑in GitHub profile:

```bash
openshell profile export github -o yaml
```

**Relevant configuration:**

```yaml
credentials:
  - name: api_token
    env_vars:
      - GITHUB_TOKEN
      - GH_TOKEN
    required: true
    auth_style: bearer
    header_name: authorization

endpoints:
  - host: api.github.com
    port: 443
    protocol: rest
    access: read-write
    enforcement: enforce

  - host: github.com
    port: 443
    protocol: rest
    enforcement: enforce
    rules:
      - allow:
          method: GET
          path: "**"
      - allow:
          method: HEAD
          path: "**"
      - allow:
          method: OPTIONS
          path: "**"
      - allow:
          method: POST
          path: "/**/git-upload-pack"
      - allow:
          method: POST
          path: "/**/git-receive-pack"
```

The profile should allow both Git fetch and push operations.

#### 4. Verify credential injection inside the sandbox

Connect to the sandbox:

```bash
openshell sandbox connect pi-container
```

Check whether the credential is present without exposing its contents:

```bash
python3 - <<'PY'
import os
v = os.environ.get("GITHUB_TOKEN", "")
print("Present:", bool(v))
print("Length:", len(v))
print("Starts with openshell:", v.startswith("openshell:"))
print("Looks like GitHub PAT:", v.startswith("github_pat_"))
PY
```

**Expected:**

```
Present: True
Starts with openshell: True
Looks like GitHub PAT: False
```

OpenShell should expose an opaque credential placeholder rather than the actual PAT.

#### 5. Verify Git credential helper configuration

Inspect the configured Git credential helper:

```bash
git config --show-origin --get-all credential.helper
```

The expected helper is:

```bash
!f() { echo "username=git"; echo "password=$GITHUB_TOKEN"; }; f
```

Verify that Git actually receives the placeholder:

```bash
printf 'protocol=https\nhost=github.com\n\n' |
  git credential fill |
  python3 -c '
import sys, os

d = dict(line.rstrip("\n").split("=", 1)
         for line in sys.stdin if "=" in line)

p = d.get("password", "")
 e = os.environ.get("GITHUB_TOKEN", "")

print("Username:", d.get("username"))
print("Password present:", bool(p))
print("Password equals GITHUB_TOKEN:", p == e)
print("Password is OpenShell placeholder:", p.startswith("openshell:"))
'
```

**Expected:**

```
Username: git
Password present: True
Password equals GITHUB_TOKEN: True
Password is OpenShell placeholder: True
```

Also check whether another authentication mechanism is configured:

```bash
echo "${GIT_ASKPASS:-not configured}"
 git config --show-origin --get-all credential.helper
```

The Git remote should use HTTPS:

```bash
git remote -v
```

Example:

```
https://github.com/OWNER/REPO.git
```

OpenShell's HTTP credential substitution does not apply to SSH remotes.

#### 6. Test authentication through GitHub CLI

This is an important diagnostic because it tests the provider's Bearer authentication independently of Git.

Inside the sandbox, run:

```bash
gh api user --jq .login
```

**Expected result:** your GitHub username.

If you receive:

```
{"message": "Bad credentials", "status": "401"}
```

GitHub is rejecting the credential. Check the PAT and the credential stored in the OpenShell provider before troubleshooting Git push permissions.

#### 7. Update an incorrect or outdated PAT

If the PAT is incorrect, generate a new fine‑grained token on GitHub and verify it on the host first.

Update the existing OpenShell provider:

```bash
export GITHUB_TOKEN="github_pat_..."
openshell provider update my-github \
  --credential GITHUB_TOKEN \
  --wait \
  --timeout 30
```

The command reads the token from the host environment.

After updating the provider, start a fresh shell/process inside the sandbox and retest:

```bash
gh api user --jq .login
```

A successful response confirms that GitHub API authentication works through the OpenShell provider.

#### 8. Test Git fetch and push

Once GitHub CLI authentication works, test Git operations.

First, verify remote read access:

```bash
GIT_TERMINAL_PROMPT=0 git ls-remote origin
```

Then:

```bash
git fetch origin
```

Finally, test push using a temporary branch:

```bash
git checkout -b openshell-auth-test

echo "OpenShell authentication test" > openshell-auth-test.txt

git add openshell-auth-test.txt

git commit -m "Test OpenShell GitHub authentication"

git push -u origin openshell-auth-test
```

After verification, delete the temporary branch.

#### 9. Inspect OpenShell network logs

If authentication or Git operations fail, inspect the sandbox logs:

```bash
openshell logs pi-container --since 5m
```

**Relevant Git Smart HTTP operations include:**

- `GET /OWNER/REPO.git/info/refs` (remote discovery)
- `POST /OWNER/REPO.git/git-upload-pack` (fetch)
- `POST /OWNER/REPO.git/git-receive-pack` (push)

Interpret the logs carefully:

- `HTTP:GET ALLOWED` – request passed OpenShell's network policy (does not guarantee GitHub acceptance).
- `HTTP:POST ALLOWED` – Git operation permitted by policy.
- `HTTP 401` – Authentication failed.
- `HTTP 403` – Authorization or policy restrictions.
- No `git-receive-pack` request – Git failed before push negotiation; investigate authentication first.

#### Troubleshooting checklist

| Check | Expected result |
|-------|-----------------|
| Host‑side PAT test | HTTP 200 |
| Provider exists | `my-github` |
| Provider attached | `pi-container` |
| Sandbox environment | OpenShell placeholder present |
| Git credential helper | Returns the placeholder |
| `gh api user` | Returns GitHub username |
| `git ls-remote origin` | Succeeds |
| `git fetch origin` | Succeeds |
| `git push` | Succeeds |
| OpenShell logs | Git requests allowed |

**Important:** A valid provider configuration and successful credential injection do not guarantee that the PAT itself is valid. Always test the PAT independently against GitHub before investigating OpenShell network policies or Git authentication behavior.

Never print, commit, or otherwise expose the actual PAT in sandbox logs, terminal output, or repository files.

---


---

### Repository Layout
```
.
├─ README.md                ← this guide
├─ openshell-configs/       ← OpenShell sandbox policy & provider profiles
│   ├─ baseline_policy.yaml   ← Filesystem & network policy
│   └─ profiles/
│       ├─ OllamaProvider.yaml← Ollama Cloud provider profile
│       └─ GithubProvider.yaml ← GitHub API & Git operations profile
└─ pi-configs/             ← Pi runtime configuration
    ├─ models.json          ← Provider / model mapping
    └─ settings.json        ← Default model & provider
```

---

### Contributing & License
- **License:** MIT (see `LICENSE`).
- **How to contribute:** Fork → create a feature branch (`git checkout -b feat/…`) → run `./scripts/setup.sh` to verify → open a PR.
- **Security reporting:** Please email security@yourorg.com with details of any vulnerability.

