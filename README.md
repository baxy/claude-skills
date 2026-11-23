# Claude Skills

Claude Code skills for NestJS development, code review and legacy codebase analysis.

| Skill                   | Use it to                                                                                                                                   |
|-------------------------|---------------------------------------------------------------------------------------------------------------------------------------------|
| [`nestjs-developer`](nestjs-developer/SKILL.md)           | Write, generate or refactor NestJS code: modules, controllers, services, DTOs, guards, interceptors, entities, migrations, tests, `main.ts` |
| [`nestjs-reviewer`](nestjs-reviewer/SKILL.md)             | Review a NestJS branch, pull request or set of files and report findings with evidence and fixes                                            |
| [`legacy-to-clean-specs`](legacy-to-clean-specs/SKILL.md) | Reverse-engineer a legacy codebase and produce specifications for a clean rewrite or modular replacement                                    |

Version 1.0.0.

## Installation

Clone the repository and symlink the skills, so a `git pull` updates them:

```bash
git clone https://github.com/baxy/claude-skills.git ~/workspace/claude-skills
ln -s ~/workspace/claude-skills/nestjs-developer      ~/.claude/skills/nestjs-developer
ln -s ~/workspace/claude-skills/nestjs-reviewer       ~/.claude/skills/nestjs-reviewer
ln -s ~/workspace/claude-skills/legacy-to-clean-specs ~/.claude/skills/legacy-to-clean-specs
```

To share them with everyone on one project, copy the skill folders into that project's `.claude/skills/` instead.

The reviewer reads the developer's rules through `../nestjs-developer/rules/`, so **keep the two folders side by side**.
Without `nestjs-developer`, the reviewer falls back to its own checklist and says so.

Start a new Claude Code session after installing.

## Usage

The skills load on their own when a task matches them. You can also name them:

```
Add a cancel action to orders: only pending or paid orders can be canceled, with an optional reason.

Use nestjs-reviewer to review this branch against main.

Analyze this legacy codebase and write specs for a clean modular rewrite.
```

## nestjs-developer

`SKILL.md` maps each task (new endpoint, entity change, authentication, tests...) to the rules to read before writing
code. Each rule in `rules/` has the reasoning, an incorrect example and a correct example. `references/` covers project
setup (ESM, NestJS 12, Prisma 7) and the Vitest and Jest test runners.

**Assumed stack:** NestJS 12 (11 supported) on Fastify (a project already on Express or `@nestjs/platform-node` keeps
it, and you can ask for another adapter), TypeORM or Prisma 7 on PostgreSQL, `class-validator`, `@nestjs/swagger`,
Vitest or Jest, `nestjs-pino`. The skill reads `package.json` and keeps what the project already uses; with a different
library (Mongoose, Drizzle, Zod) it applies the intent of the rule.

| Section              | Prefix      | Rules |
|----------------------|-------------|-------|
| Architecture         | `arch-`     | 11    |
| Dependency Injection | `di-`       | 6     |
| Error Handling       | `error-`    | 4     |
| Security             | `security-` | 6     |
| Performance          | `perf-`     | 4     |
| Testing              | `test-`     | 6     |
| Database & ORM       | `db-`       | 12    |
| DevOps & Deployment  | `devops-`   | 3     |
| API Design           | `api-`      | 12    |
| Microservices        | `micro-`    | 4     |
| Code Style           | `style-`    | 4     |
| Documentation        | `docs-`     | 2     |

## nestjs-reviewer

Reviews the diff against the base branch (plus what the changed files depend on) using
[`checklist.md`](nestjs-reviewer/references/checklist.md), confirms every finding against its rule, and reports:

1. Summary
2. Critical issues (must fix)
3. Warnings (should fix)
4. Suggestions (consider)
5. Positive observations
6. Recommendations

Every Critical issue and Warning comes with a `file:line`, the snippet, a fix and the rule ID, or, for a defect no rule
covers, the input that makes the code fail. A Suggestion says what it gains. The reviewer does not change code unless
asked.

## legacy-to-clean-specs

Explores a codebase with evidence for every claim, writes its current behavior as EARS requirements, catalogs the
technical debt, and proposes a target architecture. The output is one folder, `docs/specs/{yyyy-mm-dd}-{name}/`: an
index and one numbered file per section.

### Complementary skills

These skills from [Jeffallan/claude-skills](https://github.com/Jeffallan/claude-skills) pick up where
`legacy-to-clean-specs` stops, and it suggests them when they fit. They are not included in this repository, so they
stay current with their source.

| Skill                     | Use it when                                                                              |
|---------------------------|------------------------------------------------------------------------------------------|
| **architecture-designer** | The target architecture needs deeper ADRs, pattern selection or non-functional analysis |
| **feature-forge**         | New features go on top of the extracted specs, or requirements need a workshop          |
| **fullstack-guardian**    | The specs are approved and the new project's implementation begins                      |

Install them with `npx skills add`:

```bash
npx skills add jeffallan/claude-skills --skill architecture-designer
npx skills add jeffallan/claude-skills --skill feature-forge
npx skills add jeffallan/claude-skills --skill fullstack-guardian
```

Or clone the repository once and symlink them, then `git pull` inside it to update:

```bash
git clone https://github.com/Jeffallan/claude-skills.git ~/workspace/jeffallan-claude-skills
ln -s ~/workspace/jeffallan-claude-skills/skills/architecture-designer ~/.claude/skills/architecture-designer
ln -s ~/workspace/jeffallan-claude-skills/skills/feature-forge         ~/.claude/skills/feature-forge
ln -s ~/workspace/jeffallan-claude-skills/skills/fullstack-guardian    ~/.claude/skills/fullstack-guardian
```

## License

[MIT](LICENSE)
