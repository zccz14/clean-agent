# clean-agent

`clean-agent` is a reusable skill for running an adversarial create-review-repair loop before human review.

It is not a skill for creating agent files specifically. In this skill, an agent means an AI role: a creator, a reviewer, or a coordinator. The pattern applies to documents, specs, plans, prompts, reports, code proposals, UI descriptions, agent files, and other AI-generated artifacts.

The goal is to reduce human review bandwidth, not to minimize token use. Human attention is the expensive resource. Drafts that cannot pass an independent AI review should be repaired inside the AI loop before they are shown to a person.

## When To Use

Use `clean-agent` when you want:

- A creator SubAgent to write an artifact to disk.
- An independent reviewer SubAgent to read-only review it.
- A main Agent to coordinate retries without showing low-quality drafts to the user.
- A strict PASS / RETRY / FAILED gate before human review.

## Core Pattern

```text
One shared specification drives both creation and review.
The creator writes files and returns only paths.
The reviewer only reads and returns PASS / RETRY / FAILED with short audit notes.
RETRY stays inside the AI repair loop.
FAILED escalates to human decision.
PASS is the first point where the artifact is worth human review.
```

## Installation

This repository is a Markdown skill package. It does not require npm, Python, or a build step.

Recommended:

```bash
npx skills add zccz14/clean-agent
```

To inspect the package before installing:

```bash
npx skills add zccz14/clean-agent --list
```

Manual options:

1. Copy this entire repository directory into a compatible skills directory, such as `~/.config/opencode/skills/clean-agent/`, `~/.agents/skills/clean-agent/`, or `~/.claude/skills/clean-agent/`.
2. Or configure your tool's skills path to include this repository or its parent directory, so it can discover the root `SKILL.md`.

`SKILL.md` lives at the root of this repository.

For opencode, a config example is:

```json
{
  "skills": {
    "paths": ["/path/to/clean-agent"]
  }
}
```

Restart the host agent application after changing skill paths or adding skill files.

## Repository Structure

```text
clean-agent/
├── .gitignore
├── LICENSE
├── README.md
└── SKILL.md
```

## Usage

Ask the main Agent to coordinate a clean-agent loop, or assign a SubAgent one of the explicit modes from the skill:

- `creation mode`: generate or repair an artifact, write it to files, return only paths.
- `review mode`: read the shared specification and artifact files, then return PASS / RETRY / FAILED.
- `coordination mode`: main Agent only; dispatch creator and reviewer SubAgents, route retries, and escalate only PASS or FAILED states.

The most important habit is to pass shared specification references, not long pasted specifications, to both creator and reviewer. Examples include a skill name, file path, section heading, issue, design doc, or concise user constraints from the current request.
