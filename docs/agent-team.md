# Agent team

To build Mona's Project Pulse dashboard, I'm using a four-agent custom team defined under `.github/agents/`, orchestrated with GitHub Copilot CLI running in a Codespace.

| Agent | Model | Responsibility | Definition file |
|---|---|---|---|
| **Orchestrator** | Claude Opus 4.7 (copilot) | Breaks the request into tasks, delegates to Planner, Coder, and Designer, runs non-overlapping work in parallel, and reports the integrated result. Does not implement work itself. | `.github/agents/orchestrator.agent.md` |
| **Planner** | Claude Opus 4.7 (copilot) | Researches the repository and docs, identifies edge cases/dependencies, and produces an ordered implementation plan with file assignments for the Orchestrator to split into phases. Does not write code. | `.github/agents/planner.agent.md` |
| **Coder** | GPT-5.5 (copilot) | Implements the dashboard's code-oriented tasks within its assigned file scope, including Project Pulse support files like `.vscode/launch.json`, with deterministic, testable behavior. | `.github/agents/coder.agent.md` |
| **Designer** | Gemini 3.1 Pro (copilot) | Owns UI/UX, accessibility, and visual design for Project Pulse, producing a polished dashboard with project cards, status badges, and responsive layout using CSS hooks like `.dashboard` and `.project-card`. | `.github/agents/designer.agent.md` |

All four agents avoid staging, committing, or pushing changes — git operations stay under my control through Copilot CLI prompts.
