# system-design skill

A Claude Code / agent skill for staff+ system design: designing systems, studying, and practicing interviews.

Distilled from Hello Interview's *System Design* course and restated in original words, then extended for staff+ depth. Premium-gated sections were rebuilt from their outlines plus independent engineering knowledge; each case study notes its source coverage.

## Commands

| Command | What it does |
|---|---|
| `/system-design mock [question] [level]` | Claude plays a strict interviewer; you design; feedback at the end |
| `/system-design answer <question>` | Staff-level model answer, annotated with the signals each move earns |
| `/system-design grade` + design/transcript | Scores against the staff rubric with evidence and a practice plan |
| `/system-design learn <topic>` | Teaches one concept, pattern, or technology |
| `/system-design drill` | Picks practice questions for your weak spots |
| (no command) | Any design question; Claude answers as a staff engineer |

## Layout

```
SKILL.md                    entry point, modes, routing
references/                 delivery framework, core concepts, numbers, patterns,
                            technologies, staff+ lens
references/interview/       rubric, question bank, probe bank, feedback template
case-studies/               32 problems (Bitly → ChatGPT)
```

## Install

```bash
git clone <this repo> ~/.agents/skills/system-design
ln -s ../../.agents/skills/system-design ~/.claude/skills/system-design
```

## Test results (2026-10-18)

Each test ran once per version (with vs without the skill):

- Design answers, blind-graded with the staff rubric: **Hire** with the skill vs **Lean hire** without (rate limiter, hotel booking).
- Mock interviewer behaviour: 6/6 vs 2.5/6. Without the skill the interviewer leaked requirements and hinted at flaws.
- Transcript grading: about equal; the skill adds a crux check and a practice plan.
