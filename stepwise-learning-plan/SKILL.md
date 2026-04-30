---
name: stepwise-learning-plan
description: Build tailored day-wise or step-by-step learning plans for software tools, platforms, databases, DevOps systems, and technical repositories. Use when a user wants to learn tools such as Docker, GitLab, InfluxDB, MongoDB, PostgreSQL, Kubernetes, Redis, Linux, CI/CD, observability stacks, or similar software through a structured plan with explanations, setup guidance, hands-on labs, troubleshooting, interview preparation, best practices, security, and production-readiness topics. Also use when a user asks for a crash course, roadmap, study plan, hands-on curriculum, practical labs, or beginner/intermediate/advanced learning path.
---

# Stepwise Learning Plan

## Goal

Create practical learning paths that feel like a tutor guiding the learner through small wins. Balance concepts, hands-on tasks, troubleshooting, and production/security habits so the learner can build confidence while doing real work.

## Intake First

Ask clarifying questions before drafting the plan unless the user already provided the details. Keep the first round focused:

1. What software/tool do you want to learn?
2. How many days or weeks do you want the plan to cover, and how much time can you spend per day?
3. What is your current level: beginner, intermediate, or advanced?
4. What operating system or environment will you use: macOS, Linux, Windows, Docker Desktop, VM, cloud instance, or an existing project/server?
5. What is your goal: job/interview prep, production usage, project delivery, certification, personal learning, or migration/support work?

If the user wants the plan saved as a file, create it in the selected project or the directory where the relevant software/project is installed. If no destination is clear, ask where to place it. Otherwise, provide the plan inline as Markdown.

## Research Rules

Use current sources when creating a learning plan for real software:

- Browse for the latest official documentation, installation guides, quickstarts, release notes, and best-practice pages before finalizing the plan.
- Prefer official docs first. Add high-quality blogs, videos, courses, or tutorials only when they improve the learner's practice path.
- Cite links in the output when external resources shaped the plan.
- Note version-sensitive assumptions, such as major version, deployment model, package manager, or OS-specific setup.
- If browsing is unavailable, state that the plan is based on available knowledge and mark docs/resources as "verify latest official docs."

## Workflow

1. Confirm scope from the intake answers.
2. Research the latest official docs and useful supplemental resources.
3. Choose the structure:
   - Day-wise plan for fixed timelines.
   - Step-by-step/module plan for flexible timelines.
   - Both when the user asks for a roadmap plus daily execution.
4. Start with fundamentals and local setup.
5. Add hands-on labs that produce visible progress and small completed outcomes.
6. Progress toward real workflows, troubleshooting, production practices, security, backup/restore, monitoring, automation, and best practices.
7. Include interview/job-prep questions when relevant or when the user requested complete preparation.
8. End with a capstone project, review checklist, and next-step recommendations.

## Output Requirements

Use Markdown by default. Write like a tutor first, then include checklists for execution.

Every complete plan should include:

- Title and learner profile summary.
- Assumptions and required environment.
- Installation/setup instructions or links to current official setup docs.
- Day-wise schedule and/or module-wise roadmap.
- For each day/module: concepts, explanations, commands or tasks, hands-on lab, expected outcome, troubleshooting prompts, self-check questions, and small deliverable.
- Resource list grouped by official docs, blogs/articles, videos/courses, and practice projects.
- Production, security, performance, and best-practice topics after the basics.
- Interview questions or job-prep prompts where useful.
- Capstone project and completion checklist.

For reusable section templates and lab patterns, read `references/plan-patterns.md`.

## Hands-On Lab Design

Make labs small, satisfying, and cumulative:

- Each lab should have a clear objective, setup, tasks, success criteria, and cleanup/reset step.
- Prefer tasks the learner can complete in 20-90 minutes.
- Use real commands/configuration where safe and appropriate.
- Explain what each command or configuration is teaching.
- Include common errors and how to debug them.
- Increase difficulty gradually: inspect, modify, connect, automate, secure, monitor, recover.

## Tone And Teaching Style

Use clear tutor-style explanations. Do not dump only a checklist unless the user asks for brevity. Explain why each topic matters, then give a concise action checklist.

Keep the learner moving:

- Use encouraging but professional wording.
- Make progress visible through small deliverables.
- Avoid overwhelming the learner with every advanced topic on day one.
- Prefer practical examples over abstract definitions.

## File Creation

When asked to create a learning-plan artifact:

- Save Markdown as `<tool-name>-learning-plan.md` unless the user requests another name.
- Place it in the selected project, repo, or software installation directory when provided.
- Include links and citations in the file.
- Keep commands OS-specific based on the user's chosen environment.
