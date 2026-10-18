# Contributing to Safe AI Lab

Thank you for contributing! This repository supports **agent‑generated contributions** (LLM‑assisted PRs and commits). To keep a clear audit trail, we enforce the following rules:

## 1. Commit Attribution
All commits that are created or amended by an LLM **must** include a `Co‑authored‑by:` trailer in the commit message, specifying the model used:

```
Co‑authored‑by: <Model Name> <Model Version> <model@example.com>
```

*Example:*
```
Co‑authored‑by: glm‑5.3‑flash v1.0 <glm-5.3-flash@example.com>
```

The email address is not required to be real; it is only a placeholder to keep the Git trailer format.

## 2. PR Comment Attribution
When opening a Pull Request that contains LLM‑generated changes, add a comment that:
1. States the **model name and version** used.
2. Mentions the **agent or skill** that performed the generation (e.g., `github` skill).
3. Provides a **link** to the public definition of that skill/agent if it exists.

*Example PR comment:*
> Generated with **glm‑5.3‑flash (v1.0)** using the **`github` skill**. Definition: <https://github.com/yourorg/safe-ai-lab/blob/main/openshell-configs/github‑provider.yaml>

## 3. Review Process
* Maintainers will verify that the `Co‑authored‑by:` line and PR comment are present.
* Automated CI checks will reject PRs missing the required attribution.

## 4. General Guidelines
* Follow the repository’s coding style (2‑space indentation for JSON/YAML, `shellcheck`‑clean Bash scripts).
* Keep changes as small and focused as possible.
* Update any impacted documentation.

---
*These rules help maintain transparency for all AI‑assisted contributions while preserving the integrity of the safe execution environment.*
