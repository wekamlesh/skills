# Plan Patterns

Use these compact templates when drafting a learning plan. Adapt wording and depth to the learner, tool, and timeline.

## Intake Summary

```markdown
# <Tool> Learning Plan

## Learner Profile

- Tool: <tool/version if known>
- Timeline: <N days/weeks>
- Time budget: <hours per day/week>
- Level: <beginner/intermediate/advanced>
- Environment: <OS/local/cloud/project path>
- Goal: <job prep/project/production/certification/etc.>

## Assumptions

- <version-sensitive assumption>
- <setup assumption>
- <safety assumption>
```

## Daily Module Template

```markdown
## Day <N>: <Theme>

### What You Will Learn

Explain the day's core idea in 2-5 practical paragraphs.

### Concepts

- <concept 1>
- <concept 2>
- <concept 3>

### Hands-On Lab: <Lab Name>

Goal: <visible result the learner will produce>

Tasks:

1. <task>
2. <task>
3. <task>

Success criteria:

- <how the learner knows it worked>
- <command/output/UI state to verify>

Troubleshooting:

- Symptom: <common problem>
  Fix: <diagnostic action and likely fix>

### Checklist

- [ ] <complete setup/task>
- [ ] <verify outcome>
- [ ] <write down one lesson learned>

### Self-Check

1. <concept question>
2. <practical debugging question>
3. <when would you use this in real work?>
```

## Module Roadmap Template

```markdown
## Module <N>: <Capability>

Outcome: <what the learner can do after this module>

Topics:

- <topic>
- <topic>

Practice:

- Lab: <lab>
- Mini task: <task>
- Stretch task: <optional challenge>

Job/production connection:

- <why this matters in real systems>
```

## Capstone Template

```markdown
## Capstone Project: <Project Name>

Build: <one concise description>

Requirements:

- <requirement>
- <requirement>
- <requirement>

Milestones:

1. <setup>
2. <core workflow>
3. <observability/troubleshooting>
4. <security/backup/production hardening>

Demo checklist:

- [ ] <evidence>
- [ ] <evidence>
- [ ] <evidence>
```

## Suggested Learning Arc

For most software tools, use this progression:

1. Orientation: what the tool is for, core mental model, where it fits.
2. Setup: installation, version check, local environment, first successful run.
3. Core operations: the 5-10 commands/workflows used most often.
4. Data/configuration: files, schemas, config, state, networking, or storage.
5. Integration: connect it with another tool, app, service, or workflow.
6. Debugging: logs, status commands, errors, health checks, reset strategies.
7. Automation: scripts, CI/CD, IaC, scheduled jobs, or repeatable workflows.
8. Security: users, permissions, secrets, TLS, network exposure, least privilege.
9. Production: backup/restore, upgrades, monitoring, performance, scaling.
10. Capstone: small real-world project that combines the above.

## Resource Section Template

```markdown
## Resources

Official docs:

- [<title>](<url>) - <why it matters>

Blogs/articles:

- [<title>](<url>) - <what to use it for>

Videos/courses:

- [<title>](<url>) - <target level or use>

Practice:

- <exercise/project idea>
```
