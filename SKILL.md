---
name: clean-agent
description: Use when the user wants a create-review-repair loop, adversarial generation, AI self-review before human review, reduced human review bandwidth, or separate creator and reviewer SubAgents. This skill defines a general coordination pattern, not just agent-file creation: the main Agent coordinates only, creator SubAgents write files and return paths, reviewer SubAgents read only and return PASS / RETRY / FAILED with short audit notes.
---

# Clean Agent

`clean-agent` defines a general adversarial generation loop: a creator produces an artifact, an independent reviewer checks it against the same specification, and a coordinator decides whether to retry internally or escalate to a human.

It is not limited to creating agent files. Here, an agent means an AI role: creator agent, reviewer agent, or coordinator agent. Use the pattern for any generated artifact that benefits from independent review, such as documentation, specs, plans, reports, prompts, code proposals, UI descriptions, skill files, or agent files.

The goal is to reduce human review bandwidth, not to save tokens. Token use is acceptable when it prevents humans from spending attention on drafts that an independent AI reviewer can already reject.

Core rule:

```text
One shared specification drives creation and review; creators write files and return only paths; reviewers read only and return PASS / RETRY / FAILED with short audit notes; RETRY is repaired inside the AI loop; FAILED escalates to human decision; PASS is the first point where the artifact is worth human review.
```

## Modes

Identify the mode before acting.

| Mode | Use When | Goal | Who May Run It |
| --- | --- | --- | --- |
| Creation mode | An artifact must be generated, edited, organized, or repaired | Write the artifact to files from the shared specification | Creator SubAgent only |
| Review mode | An artifact must be checked against the shared specification | Read-only review and return PASS / RETRY / FAILED | Reviewer SubAgent only |
| Coordination mode | A user wants a create-review-repair loop or wants fewer human reviews | Dispatch creator and reviewer SubAgents, route retries, decide escalation | Main Agent only |

If the user does not name a mode, use these defaults:

- For a new artifact that should be AI-reviewed before the user sees it, the main Agent uses coordination mode.
- For prompts like "review", "check", "does this meet the spec", or "self-review first", the main Agent dispatches a reviewer SubAgent.
- For prompts about reducing manual review, AI gatekeeping, adversarial generation, create-review-repair, or showing humans only reviewed candidates, use coordination mode.

## Role Boundaries

Keep creation, review, and coordination separate because self-review hides errors.

- The main Agent does not create or review artifact content directly when this loop is active.
- Creation mode is run by a creator SubAgent.
- Review mode is run by an independent reviewer SubAgent.
- Coordination mode is run only by the main Agent.
- The main Agent tracks user constraints, shared specification references, artifact paths, review conclusions, short audit notes, retry count, and escalation status.
- The creator writes files and returns only artifact paths.
- The reviewer does not modify files and returns only PASS / RETRY / FAILED with concise evidence and repair guidance.
- Reviewer notes should be small enough for the main Agent to pass directly back to the creator.

## Shared Specification

The creator and reviewer must use the same specification sources. A shared specification is a reference set, not a copied template.

Prefer stable references over long pasted text:

- A skill name, such as `clean-doc`, `skill-creator`, or another domain skill.
- A file path, section heading, line range, issue, ticket, design doc, README, or existing specification.
- Current-turn user constraints when there is no stable file reference.
- A combination of references, such as "Skill: clean-doc + File: docs/style.md + user constraint: keep it under 500 words".

The coordinator should pass enough references for both SubAgents to read the same basis. Do not paste large specifications into the prompt just to satisfy a template. Use a short summary only when the source exists only in the current conversation.

Example:

```text
Shared specification sources:
- Skill: clean-doc
- File: docs/writing-style.md
- User constraints: write for new contributors; avoid implementation history
```

## Creation Mode

Creation mode writes or repairs artifact files from the shared specification.

Workflow:

1. Read the shared specification sources provided by the coordinator.
2. Read only the additional context needed to create the artifact.
3. Write the artifact to the requested path or to the path implied by the specification.
4. Optionally self-check for obvious violations, without treating this as a substitute for independent review.
5. Return only the artifact path list.

Creator output format:

```text
Artifact paths:
- path/to/artifact-a.md
- path/to/artifact-b.html
```

Do not return the full artifact body, a change summary, self-review notes, or residual risks.

## Review Mode

Review mode is read-only. The reviewer reads the shared specification and artifact files, then decides whether the artifact is ready for human review, needs internal repair, or requires human decision.

Review checklist:

- Does the artifact follow the same shared specification sources given to the creator?
- Does it satisfy the goal, audience, inputs, outputs, and acceptance criteria?
- Would showing it now waste human review bandwidth on basic quality issues?
- Are output paths, formats, role boundaries, and required files correct?
- Did the creator write files instead of returning full content to the coordinator?
- Are failures concrete enough for a creator to repair without human judgment?
- Is there any specification conflict, missing input, or unresolved choice that needs human decision?
- Did the reviewer avoid editing or rewriting the artifact?

Use exactly one conclusion:

- `PASS`: The artifact meets the AI review gate and can be shown to a human for review or confirmation.
- `RETRY`: The artifact does not meet the gate, but a creator can repair it from short audit notes. Do not escalate to the human yet.
- `FAILED`: The AI loop cannot reliably close the gap. Human decision, clearer specification, missing input, or a direction change is needed.

Typical `RETRY` cases:

- Wrong output format or path.
- Missing required section, file, or field.
- Role confusion that has an obvious repair.
- Evidence gaps that can be filled by reading specified sources.
- Basic quality problems with a clear repair standard.

Typical `FAILED` cases:

- Shared specification sources conflict.
- The task has multiple valid directions and no selection rule.
- Required facts, permissions, or preferences are missing.
- Repeated retries point to the same unresolved issue.
- The reviewer cannot state what a correct repair would look like.
- Further automatic repair would conceal a root problem behind local patches.

Reviewer output format:

```markdown
Mode: Review mode

**Conclusion**
- PASS / RETRY / FAILED

**Audit Notes**
- [Short notes. For RETRY, list required fixes. For FAILED, list decisions needed. For PASS, list optional suggestions only.]

**Human Escalation**
- Not needed / Ready for human review / Human decision needed

**Next Step**
- Deliver / Repair / Request human decision

**Evidence**
- Specification sources: `[skill names / file paths / sections / user constraints]`
- Reviewed files: `[paths]`
- Key evidence: [line numbers, headings, or short excerpts]
```

## Coordination Mode

Coordination mode is for the main Agent. The main Agent organizes the loop without creating or reviewing artifact content directly.

Workflow:

1. Collect shared specification references, acceptance criteria, output paths, retry criteria, failure criteria, human review cost, and prohibitions.
2. Dispatch a creator SubAgent in creation mode.
3. Dispatch an independent reviewer SubAgent in review mode with the same shared specification sources and the creator's returned paths.
4. If the reviewer returns `PASS`, deliver the artifact paths and review result to the human.
5. If the reviewer returns `RETRY`, do not show the artifact body to the human. Pass the short audit notes back to a creator SubAgent for repair.
6. If the reviewer returns `FAILED`, stop the automatic loop and ask the human for the needed decision or input.
7. Repeat review after repair until PASS, FAILED, retry limit, or user stop.

Prompt template for a creator SubAgent:

```text
You are a clean-agent creator SubAgent. Use clean-agent creation mode.

Shared specification sources:
[List skill names, file paths, sections, existing specs, and user constraints. Use a minimal summary only when no stable reference exists.]

Task: Create or modify the artifact according to the shared specification sources and write it to files.

Constraints:
- Do not return the full artifact body.
- Write results to the requested or specification-defined paths.
- Return only artifact paths when complete.
```

Prompt template for a reviewer SubAgent:

```text
You are a clean-agent reviewer SubAgent. Use clean-agent review mode.

Shared specification sources:
[Use the exact same source references given to the creator.]

Files to review:
[Creator-returned paths]

Task: Read-only review whether the files satisfy the shared specification and decide whether to deliver, repair, or request human decision.

Constraints:
- Do not modify files.
- Do not rewrite the artifact.
- The conclusion must be PASS, RETRY, or FAILED.
- RETRY means repair inside the AI loop before human review.
- FAILED means the AI loop needs human decision or additional input.
- Return concise audit notes and evidence.
```

Prompt template for repair:

```text
You are a clean-agent creator SubAgent. Continue using creation mode.

Shared specification sources:
[Keep the same references used for creation and review.]

Files to repair:
[paths]

Review conclusion: RETRY
Audit notes:
[Short reviewer notes]

Task: Repair only the issues identified by the reviewer while keeping the shared specification unchanged. Write the result to the original files or requested output paths.

Constraints:
- Do not return the full artifact body.
- Return only artifact paths when complete.
```

Coordinator output format:

```markdown
Mode: Coordination mode

**Loop Result**
- Artifacts: `[paths]`
- Review conclusion: PASS / RETRY / FAILED
- Human escalation: Not needed / Ready for human review / Human decision needed
- Next step: Deliver / Repair / Request human decision
- Retry count: [n]

**Audit Notes**
- [Final reviewer notes]

**Residual Risks**
- [Items that still need user confirmation or could not be verified]
```

## Stop Conditions

Stop the loop when:

- The reviewer returns `PASS`.
- The reviewer returns `FAILED`.
- The configured retry limit is reached.
- Multiple retries point to the same unresolved issue.
- A creator or reviewer cannot access required files or context.
- The reviewer identifies a question that needs human judgment rather than AI repair.

## Completion Standard

A clean-agent loop is complete only when:

- Creation and review were performed by independent SubAgents.
- The main Agent did not directly create or review the artifact content.
- Creator and reviewer used the same shared specification sources.
- The creator wrote files and returned only paths.
- The reviewer read only and returned PASS, RETRY, or FAILED.
- RETRY results stayed inside the AI repair loop.
- FAILED results escalated the blocking decision to the human.
- PASS results were the first point where the artifact was presented as ready for human review.
