# Agent Guidelines for Safe AI Lab

This document provides high‑level guidance for any **agentic (LLM) workflows** that interact with the *Safe AI Lab* repository. It is meant to help automated agents (such as those powered by Pi, GitHub‑based bots, or custom skill pipelines) produce safe, auditable contributions.

## General Principles
1. **Safety First** – Respect the sandbox policies defined in `openshell-configs/ollama-policy.yaml`. Do not propose changes that would widen permissions unless explicitly approved.
2. **Traceability** – Every change that originates from an agent must be clearly attributed. The attribution is enforced via the `CONTRIBUTING.md` guidelines (see next section).
3. **Reproducibility** – Prefer deterministic outputs (e.g., fixed random seeds) when generating code or configuration files.
4. **Minimal Privilege** – Only request the capabilities the agent truly needs (e.g., GitHub access, Ollama inference).
5. **Documentation** – Update relevant documentation when you modify functionality (README, policy files, etc.).

## Workflow for Agents
1. **Clone the repository** and create a new branch.  
2. **Perform the required operation** (code generation, config update, policy tweak, etc.) using the appropriate skill or tool.
3. **Create a commit** that includes a **Co‑authored‑by** line (see `CONTRIBUTING.md`).
4. **Open a Pull Request** targeting `main`.
5. **Add a comment** on the PR that identifies:
   * the model name and version used (e.g., `glm‑5.3‑flash`),
   * the agent or skill name (e.g., `github` skill), and
   * a link to the public definition of that skill/agent if available.
6. **Run the CI checks** (the repository includes a GitHub Actions workflow that validates the commit metadata).
7. **Merge** once the PR passes review and CI.

## Helpful Links
- **Contribution Guide:** see [`CONTRIBUTING.md`](CONTRIBUTING.md) for the exact commit signature format.
- **Skill Definitions:** the `github` skill is documented at `/sandbox/.agents/skills/github/SKILL.md`.
- **Policy Reference:** `openshell-configs/ollama-policy.yaml`.

---
*These guidelines are intentionally lightweight to keep agents productive while maintaining auditability and security.*
