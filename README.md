# SDV Fleet Monitor: SDD skills

The spec-driven development skills our team uses with Claude Code. They live in `.claude/skills/` and load automatically when Claude Code runs in this folder.

| Step | Skill | Source |
|---|---|---|
| Requirements | `grilling` | [mattpocock/skills](https://github.com/mattpocock/skills) |
| Spec, plan, tasks | `spec-driven-development` | [addyosmani/agent-skills](https://github.com/addyosmani/agent-skills) |
| Build | `tdd` | [mattpocock/skills](https://github.com/mattpocock/skills) |
| Verify | `verification-before-completion` | [obra/superpowers](https://github.com/obra/superpowers) |
| Review | `code-review-and-quality` | [addyosmani/agent-skills](https://github.com/addyosmani/agent-skills) |

All three sources are MIT licensed; each skill folder keeps its LICENSE.

## Use it

1. Read what to build in [`docs/product-brief.md`](docs/product-brief.md), then write your prompt in `prompts/`.
2. Start Claude Code here with a clean profile, so only these skills load:

   ```bash
   CLAUDE_CONFIG_DIR=~/.claude-sdd claude
   ```

3. Paste your prompt. Specs go in `specs/`.
