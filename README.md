# Skills

Project-starter skills for Claude Code. Type one command, answer a few questions, get a ready project.

## Install

**Claude Code plugin** (recommended). Run `claude`, then:

```
/plugin marketplace add Taufiqul7756/skills
/plugin install taufiqul7756-skills@taufiqul7756
/reload-plugins
```

Update later:

```
/plugin marketplace update taufiqul7756
```

**Any agent, as editable files** (via skills.sh):

```
npx skills@latest add Taufiqul7756/skills
```

**Manual**: copy `skills/nextjs/` to `~/.claude/skills/nextjs/` (all projects) or `.claude/skills/nextjs/` (one project).

## Skills

### `/nextjs`

Two ways to start:

| You do                                                            | What happens                                     |
| ----------------------------------------------------------------- | ------------------------------------------------ |
| Open `claude` in `D:\` and type `/nextjs my-app`                  | Creates `D:\my-app` and builds the app inside it |
| Make the folder yourself, open `claude` inside it, type `/nextjs` | Builds the app in that folder (no new folder)    |

If `/nextjs` doesn't show, use `/taufiqul7756-skills:nextjs`.

It installs the default stack first — Next.js 16 (Turbopack), TypeScript, Tailwind v4, ESLint + Prettier, Husky + lint-staged + commitlint, Zod, Axios, Framer Motion, react-icons, lodash, env files, Dockerfile, GitHub Actions CI, `CLAUDE.md`, `CONTEXT.md`, and `docs/{prd,tasks,designs}` — then asks 8 quick questions:

| Question             | Default                                                      |
| -------------------- | ------------------------------------------------------------ |
| State management     | Zustand + TanStack Query (Redux → Redux Toolkit + RTK Query) |
| Forms                | React Hook Form + Zod                                        |
| Auth                 | None (or your own backend with cookie refresh)               |
| Testing              | Vitest + Testing Library                                     |
| UI                   | Own components on design tokens (or shadcn/ui)               |
| Extras               | Dark mode, toasts, React Compiler                            |
| Brand color          | Indigo                                                       |
| What you're building | fills `CONTEXT.md` and the README                            |

Finally it runs lint, typecheck, tests and build, and makes the first commit.

## License

MIT
