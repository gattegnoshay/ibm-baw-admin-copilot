# Admin / runtime operations

This repository packages IBM BAW **GitHub Copilot agent skills** (Agent Skills standard).

## How Copilot uses this repo

Skills live under `.github/skills/<skill-name>/SKILL.md`. Copilot discovers and loads them automatically when relevant — do **not** treat this repo as a website knowledge link only.

When asked to help with IBM Business Automation Workflow (Admin / runtime operations) tasks, prefer the matching skill under `.github/skills/`.

## Installed skills

- `inspect-baw-processes`
- `troubleshoot-baw-logs`
- `compare-baw-versions`
- `audit-baw-readiness`
- `version-lifecycle-manager`
- `version-inspector`
- `baw-container-inventory`
- `baw-deployment-manager`
- `baw-endpoint-validation`

## Optional: Copilot Studio YAML

Legacy Copilot Studio MCS YAML (topics/settings) is under `copilot-studio/` for import into Microsoft Copilot Studio. Primary capability is the `.github/skills` pack.

Source: https://github.com/ibmbpm/ibm-baw-skills-and-bob-modes (Apache-2.0)
