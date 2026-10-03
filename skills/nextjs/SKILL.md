---
name: nextjs
description: Scaffold a production-ready Next.js 16 app in the current folder. Installs the default stack first, then grills the user with a few setup questions (state management, forms, auth, testing, UI kit, extras, brand color, what the app is) and wires up every answer.
argument-hint: "[App Name]"
disable-model-invocation: true
allowed-tools: Bash(npx *) Bash(npm *) Bash(node *) Bash(git *) Bash(ls *) Read Write Edit Glob AskUserQuestion
---

# /nextjs — full Next.js project from one command

You are setting up a brand-new Next.js app **in the current working directory**.
Work through the four phases in order. Do not stop between phases except to ask the Phase 2 questions.

## Ground rules

- **Phase 0 and 1 ask nothing.** Install the defaults first, then ask (Phase 2).
- Create and overwrite files with the **Write** tool (it creates folders too). Use Bash only for `npx`, `npm`, `node`, `git`, `ls`. No `mkdir`, `cp`, `rm`, `sed` — use the `node -e` one-liners given here, so the skill works on macOS, Linux and Windows.
- Run every command from the project root. Never `cd` into another folder.
- Replace placeholders everywhere: `{{APP_NAME}}` (display name), `{{PACKAGE_NAME}}` (npm name), `{{DESCRIPTION}}` (one line; use `A Next.js app.` until Phase 2 gives a better one). GitHub Actions expressions like `${{ github.ref }}` are not placeholders — keep them as written.
- Keep files the user already had (`.git`, `.claude/`, `LICENSE`, …). Only README.md is replaced.
- Version drift: this skill targets Next.js 16 / Tailwind 4 / ESLint 9. If something here fails against the installed versions, read `node_modules/next/dist/docs/` (or the package's README), adapt, and keep going. Mention the adaptation in the final summary.
- Never print the full contents of files back to the user. Short progress lines only.

---

## Phase 0 — Preflight (silent)

1. `node --version` → must be ≥ 20.9 (recommend 24 LTS). If older, stop and tell the user to upgrade.
2. `git --version` → must exist.
3. `ls -A` → if a `package.json` already exists, stop and ask whether to continue (this skill is for new projects).
4. Names:
   - `{{PACKAGE_NAME}}` = current folder name, lowercased, spaces/underscores → `-`, only `a-z 0-9 - .`
   - `{{APP_NAME}}` = the text typed after `/nextjs` (here: "$ARGUMENTS"). If that is empty, use the folder name in Title Case.

---

## Phase 1 — Install the defaults (no questions)

Tell the user in one line: "Installing Next.js 16 + default stack — questions come right after."

### 1.1 Scaffold

`create-next-app` refuses folders that contain files like README.md, so scaffold into a temp folder and merge it in without overwriting existing files:

```bash
npx -y create-next-app@latest nextjs-scaffold-tmp --ts --tailwind --eslint --app --src-dir --import-alias "@/*" --use-npm --yes --disable-git --skip-install
```

```bash
node -e "const fs=require('fs');fs.cpSync('nextjs-scaffold-tmp','.',{recursive:true,force:false});fs.rmSync('nextjs-scaffold-tmp',{recursive:true,force:true});for(const f of ['next.svg','vercel.svg','file.svg','globe.svg','window.svg'])fs.rmSync('public/'+f,{force:true})"
```

```bash
npm pkg set name={{PACKAGE_NAME}}
```

### 1.2 Install the default stack

```bash
npm install axios zod framer-motion lodash react-icons clsx tailwind-merge geist
```

```bash
npm install -D @types/node@^24 @types/lodash prettier prettier-plugin-tailwindcss eslint-config-prettier husky lint-staged @commitlint/cli @commitlint/config-conventional
```

(`@types/node@^24` replaces create-next-app's `^20`; newer tools such as Vitest need it.)

### 1.3 Scripts, env files, .gitignore

```bash
npm pkg set scripts.typecheck="next typegen && tsc --noEmit" scripts.format="prettier --write ." scripts.format:check="prettier --check ." scripts.prepare="husky"
```

(`next typegen` creates the route types such as `LayoutProps`; plain `tsc` fails without it on a fresh clone or in CI.)

Write `.env.example` (below), then:

```bash
node -e "const fs=require('fs');for(const f of ['.env','.env.local'])if(!fs.existsSync(f))fs.copyFileSync('.env.example',f);fs.appendFileSync('.gitignore','\n# keep the env template in git\n!.env.example\n\n# personal Claude Code settings\n.claude/settings.local.json\n')"
```

### 1.4 Write these files exactly

Overwrite the create-next-app versions where they exist (Read them first so Write is allowed).

#### `.env.example`

```env
# Copy to .env.local and fill in real values. This file is committed — never put secrets here.
# NEXT_PUBLIC_* values are inlined into the browser bundle at build time.

NEXT_PUBLIC_APP_NAME="{{APP_NAME}}"
NEXT_PUBLIC_SITE_URL=http://localhost:3000
NEXT_PUBLIC_API_URL=http://localhost:8000/api

# Server-only secrets go below (no NEXT_PUBLIC_ prefix), e.g.
# DATABASE_URL=
```

#### `public/robots.txt`

Keeps `public/` in git (the Dockerfile copies it).

```
User-agent: *
Allow: /
```

#### `.nvmrc`

```
24
```

#### `.prettierrc.json`

```json
{
  "semi": true,
  "singleQuote": false,
  "trailingComma": "all",
  "printWidth": 100,
  "tabWidth": 2,
  "arrowParens": "always",
  "endOfLine": "lf",
  "plugins": ["prettier-plugin-tailwindcss"],
  "tailwindStylesheet": "./src/app/globals.css",
  "tailwindFunctions": ["cn", "clsx"]
}
```

#### `.prettierignore`

```
node_modules
.next
out
build
coverage
public
package-lock.json
next-env.d.ts
*.md
```

#### `eslint.config.mjs`

```js
import { defineConfig, globalIgnores } from "eslint/config";
import nextVitals from "eslint-config-next/core-web-vitals";
import nextTs from "eslint-config-next/typescript";
import prettier from "eslint-config-prettier/flat";

const eslintConfig = defineConfig([
  ...nextVitals,
  ...nextTs,
  prettier, // turns off rules that fight Prettier — keep it after the Next configs
  {
    rules: {
      "@typescript-eslint/no-explicit-any": "error",
      "@typescript-eslint/no-unused-vars": [
        "error",
        { argsIgnorePattern: "^_", varsIgnorePattern: "^_" },
      ],
      "@typescript-eslint/consistent-type-imports": "warn",
      "no-console": ["warn", { allow: ["warn", "error"] }],
      "prefer-const": "error",
    },
  },
  globalIgnores([
    ".next/**",
    "out/**",
    "build/**",
    "coverage/**",
    "playwright-report/**",
    "next-env.d.ts",
  ]),
]);

export default eslintConfig;
```

#### `.lintstagedrc.json`

```json
{
  "*.{ts,tsx,js,jsx,mjs,cjs}": ["eslint --fix", "prettier --write"],
  "*.{json,css,yml,yaml}": ["prettier --write"]
}
```

#### `commitlint.config.mjs`

```js
/** Conventional Commits: feat|fix|docs|style|refactor|perf|test|build|ci|chore|revert */
const config = {
  extends: ["@commitlint/config-conventional"],
};

export default config;
```

#### `next.config.ts`

```ts
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  // Turbopack is the default bundler in Next.js 16 for both `next dev` and `next build`.
  output: "standalone", // small self-contained server for the Dockerfile
  poweredByHeader: false,
  images: {
    remotePatterns: [
      // { protocol: "https", hostname: "cdn.example.com" },
    ],
  },
};

export default nextConfig;
```

#### `.vscode/settings.json`

```json
{
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": "explicit"
  },
  "typescript.tsdk": "node_modules/typescript/lib",
  "files.associations": {
    "*.css": "tailwindcss"
  }
}
```

#### `.vscode/extensions.json`

```json
{
  "recommendations": [
    "esbenp.prettier-vscode",
    "dbaeumer.vscode-eslint",
    "bradlc.vscode-tailwindcss"
  ]
}
```

#### `src/config/env.ts`

```ts
import { z } from "zod";

/**
 * Public (browser-safe) environment variables, validated once at import time.
 * A missing or malformed value fails `next build` instead of breaking at runtime.
 *
 * Rules:
 * - Read NEXT_PUBLIC_* with static `process.env.NAME` access (Next.js inlines them at build time).
 * - Never read `process.env` anywhere else — import { env } from "@/config/env".
 * - Add every new variable to .env.example too.
 */
const publicEnvSchema = z.object({
  NEXT_PUBLIC_APP_NAME: z.string().min(1),
  NEXT_PUBLIC_SITE_URL: z.url(),
  NEXT_PUBLIC_API_URL: z.url(),
});

const parsed = publicEnvSchema.safeParse({
  NEXT_PUBLIC_APP_NAME: process.env.NEXT_PUBLIC_APP_NAME,
  NEXT_PUBLIC_SITE_URL: process.env.NEXT_PUBLIC_SITE_URL,
  NEXT_PUBLIC_API_URL: process.env.NEXT_PUBLIC_API_URL,
});

if (!parsed.success) {
  throw new Error(
    `Invalid environment variables. Compare .env.local with .env.example:\n${z.prettifyError(parsed.error)}`,
  );
}

export const env = parsed.data;
```

#### `src/config/site.ts`

```ts
import { env } from "@/config/env";

export const siteConfig = {
  name: env.NEXT_PUBLIC_APP_NAME,
  description: "{{DESCRIPTION}}",
  url: env.NEXT_PUBLIC_SITE_URL,
} as const;
```

#### `src/lib/utils.ts`

```ts
import { clsx, type ClassValue } from "clsx";
import { twMerge } from "tailwind-merge";

/** Merge Tailwind classes safely: cn("px-2", isActive && "bg-primary", className) */
export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs));
}
```

#### `src/lib/api.ts`

```ts
import axios, { AxiosError } from "axios";

import { env } from "@/config/env";

/**
 * The one axios instance for the app. Call it from src/features/<feature>/service.ts,
 * never from components directly.
 */
export const api = axios.create({
  baseURL: env.NEXT_PUBLIC_API_URL,
  timeout: 15_000,
  headers: { "Content-Type": "application/json" },
});

/** Every failed request rejects with an ApiError, so callers handle one shape. */
export class ApiError extends Error {
  status?: number;
  data?: unknown;

  constructor(message: string, status?: number, data?: unknown) {
    super(message);
    this.name = "ApiError";
    this.status = status;
    this.data = data;
  }
}

api.interceptors.response.use(
  (response) => response,
  (error: unknown) => {
    if (error instanceof AxiosError) {
      const body = error.response?.data as { message?: string } | undefined;
      return Promise.reject(
        new ApiError(body?.message ?? error.message, error.response?.status, error.response?.data),
      );
    }
    return Promise.reject(error);
  },
);

/** Readable message for toasts and form errors. */
export function getErrorMessage(error: unknown): string {
  if (error instanceof Error) return error.message;
  if (typeof error === "string") return error;
  return "Something went wrong";
}
```

#### `src/providers/AppProviders.tsx`

```tsx
"use client";

import type { ReactNode } from "react";

/** Every client-side provider is composed here and mounted once in src/app/layout.tsx. */
export default function AppProviders({ children }: { children: ReactNode }) {
  return <>{children}</>;
}
```

#### `src/app/globals.css`

```css
@import "tailwindcss";

/* Class-based dark mode: the `dark:` variant applies under <html class="dark">. */
@custom-variant dark (&:where(.dark, .dark *));

/*
 * Design tokens. Source of truth: docs/designs/design-system.md — keep both in sync.
 * Change --brand to re-theme the app; primary hover/subtle shades derive from it.
 */
:root {
  --brand: #4f46e5;
  --brand-foreground: #ffffff;

  --background: #ffffff;
  --foreground: #0a0a0a;
  --surface: #fafafa;
  --muted: #f4f4f5;
  --muted-foreground: #71717a;
  --border: #e4e4e7;

  --success: #16a34a;
  --warning: #d97706;
  --danger: #dc2626;
  --info: #0891b2;

  --radius: 0.5rem;
}

.dark {
  --background: #09090b;
  --foreground: #fafafa;
  --surface: #18181b;
  --muted: #27272a;
  --muted-foreground: #a1a1aa;
  --border: #27272a;

  --success: #22c55e;
  --warning: #f59e0b;
  --danger: #ef4444;
  --info: #22d3ee;
}

/* Expose tokens as Tailwind utilities: bg-background, text-muted-foreground, bg-primary, ... */
@theme inline {
  --color-background: var(--background);
  --color-foreground: var(--foreground);
  --color-surface: var(--surface);
  --color-muted: var(--muted);
  --color-muted-foreground: var(--muted-foreground);
  --color-border: var(--border);
  --color-primary: var(--brand);
  --color-primary-foreground: var(--brand-foreground);
  --color-primary-hover: color-mix(in oklab, var(--brand) 85%, black);
  --color-primary-subtle: color-mix(in oklab, var(--brand) 12%, var(--background));
  --color-ring: var(--brand);
  --color-success: var(--success);
  --color-warning: var(--warning);
  --color-danger: var(--danger);
  --color-info: var(--info);

  --font-sans: var(--font-geist-sans), ui-sans-serif, system-ui, sans-serif;
  --font-mono: var(--font-geist-mono), ui-monospace, monospace;

  --radius-sm: calc(var(--radius) - 4px);
  --radius-md: var(--radius);
  --radius-lg: calc(var(--radius) + 4px);
}

@layer base {
  * {
    @apply border-border;
  }

  body {
    @apply bg-background font-sans text-foreground antialiased;
  }
}
```

#### `src/app/layout.tsx`

```tsx
import type { Metadata } from "next";
// Self-hosted Geist (npm `geist`) — no Google Fonts request at build time, so offline/Docker builds work.
import { GeistMono } from "geist/font/mono";
import { GeistSans } from "geist/font/sans";

import { siteConfig } from "@/config/site";
import AppProviders from "@/providers/AppProviders";

import "./globals.css";

export const metadata: Metadata = {
  metadataBase: new URL(siteConfig.url),
  title: { default: siteConfig.name, template: `%s · ${siteConfig.name}` },
  description: siteConfig.description,
};

export default function RootLayout({ children }: LayoutProps<"/">) {
  return (
    <html
      lang="en"
      suppressHydrationWarning
      className={`${GeistSans.variable} ${GeistMono.variable} h-full`}
    >
      <body className="flex min-h-full flex-col">
        <AppProviders>{children}</AppProviders>
      </body>
    </html>
  );
}
```

#### `src/app/page.tsx`

```tsx
import { siteConfig } from "@/config/site";

export default function HomePage() {
  return (
    <main className="mx-auto flex w-full max-w-3xl flex-1 flex-col justify-center gap-6 px-6 py-24">
      <span className="w-fit rounded-full bg-primary-subtle px-3 py-1 font-mono text-xs text-primary">
        Next.js 16 · ready to build
      </span>
      <h1 className="text-4xl font-semibold tracking-tight">{siteConfig.name}</h1>
      <p className="text-lg text-muted-foreground">{siteConfig.description}</p>
      <p className="text-sm text-muted-foreground">
        Next step: write the first PRD in <code className="font-mono">docs/prd/</code>, then break
        it into tasks in <code className="font-mono">docs/tasks/</code>.
      </p>
    </main>
  );
}
```

#### `src/app/not-found.tsx`

```tsx
import Link from "next/link";

export default function NotFound() {
  return (
    <main className="flex flex-1 flex-col items-center justify-center gap-4 px-6 text-center">
      <p className="font-mono text-sm text-muted-foreground">404</p>
      <h1 className="text-2xl font-semibold">Page not found</h1>
      <Link
        href="/"
        className="rounded-md bg-primary px-4 py-2 text-sm font-medium text-primary-foreground hover:bg-primary-hover"
      >
        Go home
      </Link>
    </main>
  );
}
```

#### `src/app/error.tsx`

```tsx
"use client";

import { useEffect } from "react";

export default function Error({
  error,
  retry,
}: {
  error: Error & { digest?: string };
  retry: () => void;
}) {
  useEffect(() => {
    console.error(error);
  }, [error]);

  return (
    <main className="flex flex-1 flex-col items-center justify-center gap-4 px-6 text-center">
      <h1 className="text-2xl font-semibold">Something went wrong</h1>
      <p className="text-muted-foreground">Please try again.</p>
      <button
        type="button"
        onClick={() => retry()}
        className="rounded-md bg-primary px-4 py-2 text-sm font-medium text-primary-foreground hover:bg-primary-hover"
      >
        Try again
      </button>
    </main>
  );
}
```

#### `.github/workflows/ci.yml`

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

jobs:
  verify:
    runs-on: ubuntu-latest
    env:
      HUSKY: 0
    steps:
      - uses: actions/checkout@v5
      - uses: actions/setup-node@v5
        with:
          node-version-file: .nvmrc
          cache: npm
      - run: npm ci
      - name: Use example env for CI
        run: cp .env.example .env
      - run: npm run lint
      - run: npm run format:check
      - run: npm run typecheck
      - run: npm run build
```

#### `.github/pull_request_template.md`

```md
## What

<!-- One or two sentences. Link the PRD / task: docs/prd/..., docs/tasks/... -->

## Why

## How to test

- [ ] `npm run lint && npm run typecheck && npm run build` pass
- [ ] Checked in light and dark mode (if UI)
- [ ] Screenshots attached (if UI)
```

#### `.github/dependabot.yml`

```yaml
version: 2
updates:
  - package-ecosystem: npm
    directory: /
    schedule:
      interval: weekly
    open-pull-requests-limit: 5
    groups:
      minor-and-patch:
        update-types: [minor, patch]
  - package-ecosystem: github-actions
    directory: /
    schedule:
      interval: monthly
```

#### `Dockerfile`

```dockerfile
# syntax=docker/dockerfile:1
# Build:  docker build -t {{PACKAGE_NAME}} --build-arg NEXT_PUBLIC_API_URL=https://api.example.com .
# Run:    docker run -p 3000:3000 {{PACKAGE_NAME}}
ARG NODE_IMAGE=node:24-alpine

FROM ${NODE_IMAGE} AS deps
WORKDIR /app
ENV HUSKY=0
COPY package.json package-lock.json ./
RUN npm ci

FROM ${NODE_IMAGE} AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
# NEXT_PUBLIC_* values are inlined at build time — pass real ones with --build-arg.
ARG NEXT_PUBLIC_APP_NAME="{{APP_NAME}}"
ARG NEXT_PUBLIC_SITE_URL=http://localhost:3000
ARG NEXT_PUBLIC_API_URL=http://localhost:8000/api
ENV NEXT_PUBLIC_APP_NAME=$NEXT_PUBLIC_APP_NAME \
    NEXT_PUBLIC_SITE_URL=$NEXT_PUBLIC_SITE_URL \
    NEXT_PUBLIC_API_URL=$NEXT_PUBLIC_API_URL \
    NEXT_TELEMETRY_DISABLED=1
RUN npm run build

FROM ${NODE_IMAGE} AS runner
WORKDIR /app
ENV NODE_ENV=production \
    NEXT_TELEMETRY_DISABLED=1 \
    PORT=3000 \
    HOSTNAME=0.0.0.0
RUN addgroup -S -g 1001 nodejs && adduser -S -u 1001 -G nodejs nextjs
COPY --from=builder /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static
USER nextjs
EXPOSE 3000
CMD ["node", "server.js"]
```

#### `.dockerignore`

```
.git
.github
.husky
.vscode
.claude
.next
node_modules
coverage
playwright-report
docs
npm-debug.log*
.env*
Dockerfile
.dockerignore
```

#### `docs/prd/README.md`

````md
# PRDs

One file per feature: `NNNN-short-name.md` (e.g. `0001-user-login.md`).
Write the PRD before code. Then break it into tasks in `docs/tasks/`.

## Template

```md
# NNNN — <Feature name>

Status: draft | approved | done
Owner: <name>

## Problem
Who has the problem, and what is painful today? (2–4 sentences)

## Goal
What changes for the user when this ships?

## User stories
- As a <user>, I want <action> so that <outcome>.

## Scope
In:
- ...
Out (not now):
- ...

## UX
Pages/routes, key states (empty, loading, error), link screenshots in docs/designs/screenshots/.

## Data & API
Entities, endpoints, validation rules (zod schemas).

## Acceptance criteria
- [ ] ...

## Open questions
- ...
```
````

#### `docs/tasks/README.md`

````md
# Tasks

Small, shippable slices of a PRD. One file per task: `T-NNNN-short-name.md`.
Each task should be doable in one focused session and leave the app working (a thin vertical slice: UI → state → API).

## Template

```md
# T-NNNN — <Task title>

PRD: ../prd/NNNN-<name>.md
Status: todo | doing | done
Blocked by: T-NNNN (or "none")

## What
One paragraph: what the user can do after this task.

## Acceptance criteria
- [ ] ...
- [ ] `npm run lint && npm run typecheck && npm run build` pass

## Notes
Files likely touched, edge cases, links.
```

## Workflow with Claude

1. "Read docs/prd/0001-x.md and split it into tasks in docs/tasks/."
2. "Do T-0001." → Claude implements, runs checks, marks the task `done`.
````

#### `docs/designs/design-system.md`

```md
# Design System — {{APP_NAME}}

Design tokens and component rules. **Claude reads this before building any UI.**
Implementation lives in `src/app/globals.css` — change both together.

---

## Brand

| Token | Value | Use |
| --- | --- | --- |
| `--brand` | `#4f46e5` | Primary buttons, links, focus rings, active states |
| `--brand-foreground` | `#ffffff` | Text/icons on top of brand color |

Hover and subtle shades are derived with `color-mix()` — never hard-code them.

---

## Color tokens → Tailwind utilities

| Utility | Light | Dark | Use |
| --- | --- | --- | --- |
| `bg-background` | `#ffffff` | `#09090b` | Page background |
| `text-foreground` | `#0a0a0a` | `#fafafa` | Headings, body text |
| `bg-surface` | `#fafafa` | `#18181b` | Cards, panels, modals |
| `bg-muted` | `#f4f4f5` | `#27272a` | Hover rows, inputs, chips |
| `text-muted-foreground` | `#71717a` | `#a1a1aa` | Secondary text, placeholders |
| `border-border` | `#e4e4e7` | `#27272a` | All borders (default) |
| `bg-primary` / `text-primary` | `--brand` | `--brand` | Primary actions |
| `text-primary-foreground` | `--brand-foreground` | same | Text on primary |
| `hover:bg-primary-hover` | brand 85% + black | same | Primary hover |
| `bg-primary-subtle` | brand 12% on background | same | Selected/active backgrounds, badges |
| `ring-ring` | `--brand` | `--brand` | Focus rings |
| `text-success` / `bg-success` | `#16a34a` | `#22c55e` | Approved, done |
| `text-warning` / `bg-warning` | `#d97706` | `#f59e0b` | Pending, attention |
| `text-danger` / `bg-danger` | `#dc2626` | `#ef4444` | Errors, destructive |
| `text-info` / `bg-info` | `#0891b2` | `#22d3ee` | Neutral notices |

**Rules**
- Use only these utilities for color. No raw hex, no arbitrary values like `bg-[#123456]`.
- Every screen must work in light and dark mode.

---

## Typography

- Sans: Geist (`font-sans`) — body, labels, inputs, navigation
- Mono: Geist Mono (`font-mono`) — IDs, codes, badges, numbers in tables

| Role | Class |
| --- | --- |
| Page title | `text-2xl font-semibold tracking-tight` |
| Section heading | `text-lg font-semibold` |
| Card title / label | `text-base font-medium` |
| Body | `text-sm` (dense UI) or `text-base` (content pages) |
| Meta / captions | `text-xs text-muted-foreground` |

---

## Spacing & layout

- Use the Tailwind spacing scale (4px steps). Common: `gap-2`, `gap-4`, `p-4`, `p-6`, `px-6 py-8`.
- Page container: `mx-auto w-full max-w-6xl px-4 sm:px-6`
- Card padding: `p-6`
- Mobile first: `sm:` → `md:` → `lg:` → `xl:`

## Radius & shadow

| Utility | Value | Use |
| --- | --- | --- |
| `rounded-sm` | 4px | Inputs, tags |
| `rounded-md` | 8px | Buttons, cards, dropdowns |
| `rounded-lg` | 12px | Large cards, modals |
| `rounded-full` | pill | Badges, avatars |
| `shadow-sm` | — | Cards |
| `shadow-lg` | — | Dropdowns, modals |

---

## Components

### Button
- Variants: `primary` (bg-primary, text-primary-foreground, hover:bg-primary-hover), `secondary` (bg-primary-subtle, text-primary), `outline` (border, transparent), `ghost` (transparent, hover:bg-muted), `danger` (bg-danger, white text)
- Sizes: `sm` (h-8 px-3 text-xs), `md` (h-9 px-4 text-sm), `lg` (h-10 px-6 text-base)
- Always: `focus-visible:ring-2 ring-ring ring-offset-2 ring-offset-background`, disabled = `opacity-50 pointer-events-none`

### Input / FormField
- Label above input (`text-sm font-medium`), required marker `*` in `text-danger`
- Input: `h-9 rounded-sm border bg-background px-3 text-sm`, focus ring `ring-ring`
- Error: border `border-danger`, message below in `text-xs text-danger`

### Badge / status chip
- `rounded-full px-2 py-0.5 font-mono text-xs`
- pending → warning, approved/done → success, rejected/failed → danger

### Card
- `rounded-md border bg-surface p-6 shadow-sm`

### Modal
- Backdrop `bg-black/50`, container `rounded-lg bg-surface shadow-lg`
- Header: title + close button; footer: actions right-aligned

### Table
- Header: `text-xs font-semibold uppercase text-muted-foreground`
- Row: `border-b hover:bg-muted`; actions column right-aligned

---

## Screenshots

Put reference screenshots in `docs/designs/screenshots/` and mention them in the PRD or task.
```

#### `CONTEXT.md`

```md
# CONTEXT — {{APP_NAME}}

The shared language of this project. Agents and humans use these exact terms in code,
docs and conversation. Keep it short; update it when a new term appears.

## What this app is

{{DESCRIPTION}}

## Users

- **User** — (who uses the app and why)

## Glossary

| Term | Meaning | Not to be confused with |
| --- | --- | --- |
| | | |

## Key decisions

- (date) — decision — why
```

#### `CLAUDE.md`

Keep `@AGENTS.md` as the first line (create-next-app writes AGENTS.md and `next dev` keeps it current).

````md
@AGENTS.md
@CONTEXT.md

# {{APP_NAME}} — agent guide

## Stack

Next.js 16 (App Router, Turbopack, React 19) · TypeScript strict · Tailwind CSS v4 · Zod · Axios ·
Framer Motion · react-icons · lodash · ESLint 9 (flat) + Prettier · Husky + lint-staged + commitlint

## Commands

- `npm run dev` — dev server (Turbopack) on http://localhost:3000
- `npm run lint` · `npm run typecheck` · `npm run format` · `npm run build`
- Before saying a task is done: `npm run lint && npm run typecheck && npm run build` must pass.

## Workflow

1. New feature → PRD in `docs/prd/` → tasks in `docs/tasks/` → implement one task at a time.
2. Before any UI work, read `docs/designs/design-system.md`. Use token utilities
   (`bg-background`, `text-muted-foreground`, `bg-primary`, …) — never raw hex or arbitrary values.
3. New domain words go into `CONTEXT.md`. Use those words in code names.
4. Commits follow Conventional Commits (`feat: …`, `fix: …`) — commitlint enforces it.

## Structure

```
src/
├── app/                 # routes only: page.tsx, layout.tsx, loading/error/not-found
├── components/
│   ├── ui/              # generic primitives (Button, Input, Modal, Badge)
│   └── layout/          # Header, Sidebar, Footer
├── features/<feature>/  # domain code: components/, hooks/, schemas.ts, service.ts
├── hooks/               # shared hooks (useDebounce, useMediaQuery)
├── lib/                 # api.ts (axios), utils.ts (cn), other singletons
├── config/              # env.ts (zod-validated env), site.ts
├── providers/           # AppProviders.tsx — every client provider
└── types/               # shared types (ApiResponse<T>, Paginated<T>)
```

## Conventions

- Server Components by default. Add `"use client"` only for state, effects, event handlers or browser APIs.
- Env: `import { env } from "@/config/env"` — never `process.env` in app code. Add new vars to `.env.example`.
- HTTP: only through `api` from `@/lib/api`, wrapped in a feature `service.ts`. Errors arrive as `ApiError`.
- Validation: zod schemas in `features/<feature>/schemas.ts`; infer types with `z.infer`.
- Classes: `cn()` from `@/lib/utils` to merge Tailwind classes.
- Animation: `import { motion } from "framer-motion"`; define variants as consts outside components.
- Icons: `react-icons` (one family per project, e.g. `react-icons/lu`).
- lodash: import per function — `import debounce from "lodash/debounce"`.
- No `any` (use `unknown` and narrow). Props interfaces live above the component.
- Naming: components `PascalCase.tsx`, hooks `useThing.ts`, others `camelCase.ts`, route folders `kebab-case`.

## Next.js 16 notes

- Request interception lives in `src/proxy.ts` (the old `middleware.ts` name is deprecated).
- Turbopack is the default for dev and build — no flag needed.
- `error.tsx` receives `retry()` to re-fetch and re-render.
- When unsure about an API, read `node_modules/next/dist/docs/` (see AGENTS.md).

## Data & state

<!-- Filled in by /nextjs after the setup questions. -->
````

#### `README.md` (replace whatever is there)

````md
# {{APP_NAME}}

{{DESCRIPTION}}

## Getting started

```bash
npm install
cp .env.example .env.local   # then fill in real values
npm run dev                  # http://localhost:3000
```

## Scripts

| Script | What it does |
| --- | --- |
| `npm run dev` | Dev server (Turbopack) |
| `npm run build` | Production build (standalone output) |
| `npm run start` | Serve the production build |
| `npm run lint` | ESLint |
| `npm run typecheck` | TypeScript, no emit |
| `npm run format` | Prettier write |

## Docker

```bash
docker build -t {{PACKAGE_NAME}} --build-arg NEXT_PUBLIC_API_URL=https://api.example.com .
docker run -p 3000:3000 {{PACKAGE_NAME}}
```

## Project docs

- `CONTEXT.md` — shared vocabulary for the project
- `CLAUDE.md` — rules for AI coding agents
- `docs/prd/` — product requirement docs, one per feature
- `docs/tasks/` — tasks split from PRDs
- `docs/designs/design-system.md` — design tokens and component rules
````

### 1.5 Git + Husky

If `.git` does not exist: `git init -b main`. Then:

```bash
npx husky init
```

Then overwrite the hooks (Write tool):

- `.husky/pre-commit` → `npx lint-staged`
- `.husky/commit-msg` → `npx --no -- commitlint --edit "\$1"` (write a literal dollar sign followed by 1)
- `.husky/pre-push` → `npm run typecheck`

### 1.6 Quick check

```bash
npm run format
```

```bash
npm run lint
```

```bash
npm run typecheck
```

Fix anything red before moving on. Then say: "Defaults are in ✓ — a few quick questions."

---

## Phase 2 — Grilling

Use the **AskUserQuestion** tool (two calls, four questions each). Recommended option first, marked "(Recommended)". The user can always type their own answer via "Other" — honour it.

**Round 1**

1. *State management?* (header `State`)
   - Zustand + TanStack Query (Recommended) — Zustand for client UI state, Query for server data
   - Redux Toolkit + RTK Query
   - TanStack Query only
   - None for now
2. *Forms?* (header `Forms`)
   - React Hook Form + Zod (Recommended)
   - Zod only, no form library
3. *Auth?* (header `Auth`)
   - None for now (Recommended)
   - My own backend API (httpOnly cookies + refresh)
4. *Testing?* (header `Tests`)
   - Vitest + Testing Library (Recommended)
   - Vitest + Playwright e2e
   - None for now

**Round 2**

5. *UI components?* (header `UI`)
   - Own components on the design tokens (Recommended)
   - shadcn/ui
6. *Extras?* (header `Extras`, multiSelect)
   - Dark mode toggle (next-themes)
   - Toast notifications (sonner)
   - React Compiler (automatic memoization)
7. *Brand color?* (header `Brand`) — "Other" = any hex, or "later"
   - Indigo #4f46e5 (Recommended)
   - Teal #00d4c8
   - Emerald #059669
   - Neutral #18181b
8. *What are you building?* (header `Project`) — "Pick the closest, or type one line in Other."
   - Admin dashboard / internal tool
   - SaaS web app
   - Marketing / content site
   - E-commerce store

If an answer is unclear or contradicts another (e.g. "Redux" and "TanStack Query only"), ask one short follow-up with your recommended answer. Otherwise go straight to Phase 3.

---

## Phase 3 — Apply the answers

Do only the sections that match. Batch installs: one `npm install` for runtime packages and one `npm install -D` for dev packages covering every chosen option.

### State: Zustand + TanStack Query

Install: `zustand @tanstack/react-query` · dev: `@tanstack/react-query-devtools`

`src/lib/query-client.ts`

```ts
import { isServer, QueryClient } from "@tanstack/react-query";

function makeQueryClient() {
  return new QueryClient({
    defaultOptions: {
      queries: {
        staleTime: 60 * 1000, // don't refetch immediately after server render
        retry: 1,
        refetchOnWindowFocus: false,
      },
    },
  });
}

let browserQueryClient: QueryClient | undefined;

/** A fresh client per request on the server; one shared client in the browser. */
export function getQueryClient() {
  if (isServer) return makeQueryClient();
  browserQueryClient ??= makeQueryClient();
  return browserQueryClient;
}
```

`src/stores/ui-store.ts`

```ts
import { create } from "zustand";

/** Client-only UI state. Server data belongs in TanStack Query, never here. */
interface UiState {
  sidebarOpen: boolean;
  toggleSidebar: () => void;
  setSidebarOpen: (open: boolean) => void;
}

export const useUiStore = create<UiState>()((set) => ({
  sidebarOpen: false,
  toggleSidebar: () => set((state) => ({ sidebarOpen: !state.sidebarOpen })),
  setSidebarOpen: (open) => set({ sidebarOpen: open }),
}));
```

CLAUDE.md → Data & state:

```md
- Server data → TanStack Query. One hook per query in `features/<feature>/hooks/`, keys as arrays
  (`["products", id]`). Mutations invalidate the keys they change.
- Client UI state → Zustand stores in `src/stores/` (`useUiStore`). Never copy server data into Zustand.
```

### State: TanStack Query only

Same as above without `zustand` and without `src/stores/`.

### State: Redux Toolkit + RTK Query

Install: `@reduxjs/toolkit react-redux` (no TanStack Query).

`src/store/slices/ui-slice.ts`

```ts
import { createSlice, type PayloadAction } from "@reduxjs/toolkit";

/** Client-only UI state. Server data comes from RTK Query (src/store/base-api.ts). */
interface UiState {
  sidebarOpen: boolean;
}

const initialState: UiState = { sidebarOpen: false };

export const uiSlice = createSlice({
  name: "ui",
  initialState,
  reducers: {
    toggleSidebar: (state) => {
      state.sidebarOpen = !state.sidebarOpen;
    },
    setSidebarOpen: (state, action: PayloadAction<boolean>) => {
      state.sidebarOpen = action.payload;
    },
  },
});

export const { toggleSidebar, setSidebarOpen } = uiSlice.actions;
```

`src/store/base-api.ts`

```ts
import type { BaseQueryFn } from "@reduxjs/toolkit/query";
import { createApi } from "@reduxjs/toolkit/query/react";
import type { AxiosRequestConfig } from "axios";

import { api, ApiError } from "@/lib/api";

interface AxiosArgs {
  url: string;
  method?: AxiosRequestConfig["method"];
  data?: unknown;
  params?: unknown;
}

interface QueryError {
  status?: number;
  message: string;
}

/** RTK Query on top of the shared axios instance, so interceptors and ApiError still apply. */
const axiosBaseQuery: BaseQueryFn<AxiosArgs, unknown, QueryError> = async ({
  url,
  method = "GET",
  data,
  params,
}) => {
  try {
    const result = await api.request({ url, method, data, params });
    return { data: result.data };
  } catch (error) {
    const apiError = error instanceof ApiError ? error : new ApiError("Request failed");
    return { error: { status: apiError.status, message: apiError.message } };
  }
};

/** Root API. Each feature adds endpoints with baseApi.injectEndpoints() in features/<feature>/api.ts. */
export const baseApi = createApi({
  reducerPath: "api",
  baseQuery: axiosBaseQuery,
  tagTypes: [],
  endpoints: () => ({}),
});
```

`src/store/store.ts`

```ts
import { configureStore } from "@reduxjs/toolkit";

import { baseApi } from "@/store/base-api";
import { uiSlice } from "@/store/slices/ui-slice";

/** A factory, not a singleton: each request (server) and each tab (browser) gets its own store. */
export function makeStore() {
  return configureStore({
    reducer: {
      ui: uiSlice.reducer,
      [baseApi.reducerPath]: baseApi.reducer,
    },
    middleware: (getDefaultMiddleware) => getDefaultMiddleware().concat(baseApi.middleware),
  });
}

export type AppStore = ReturnType<typeof makeStore>;
export type RootState = ReturnType<AppStore["getState"]>;
export type AppDispatch = AppStore["dispatch"];
```

`src/store/hooks.ts`

```ts
import { useDispatch, useSelector, useStore } from "react-redux";

import type { AppDispatch, AppStore, RootState } from "@/store/store";

/** Always use these typed hooks instead of plain useDispatch/useSelector. */
export const useAppDispatch = useDispatch.withTypes<AppDispatch>();
export const useAppSelector = useSelector.withTypes<RootState>();
export const useAppStore = useStore.withTypes<AppStore>();
```

`src/providers/StoreProvider.tsx`

```tsx
"use client";

import { useState, type ReactNode } from "react";
import { Provider } from "react-redux";

import { makeStore } from "@/store/store";

/** Creates the store once per mount — never a module-level singleton (it would leak between requests). */
export default function StoreProvider({ children }: { children: ReactNode }) {
  const [store] = useState(makeStore);
  return <Provider store={store}>{children}</Provider>;
}
```

CLAUDE.md → Data & state:

```md
- Server data → RTK Query. Each feature injects endpoints into `baseApi` in `features/<feature>/api.ts`;
  use tags for cache invalidation.
- Client UI state → slices in `src/store/slices/`. Use `useAppSelector` / `useAppDispatch` only.
```

### State: None

CLAUDE.md → Data & state: `- Fetch in Server Components with the services; add a state library when a real need appears.`

### Forms: React Hook Form + Zod

Install: `react-hook-form @hookform/resolvers`. CLAUDE.md → Data & state:
`- Forms → React Hook Form + zodResolver(schema); schema in features/<feature>/schemas.ts, show field errors under inputs.`

Zod only → CLAUDE.md: `- Forms → controlled inputs or Server Actions; validate with the feature's zod schema.`

### Auth: own backend API

1. Replace `src/lib/api.ts` with this version (adds cookies + one shared refresh on 401):

```ts
import axios, { AxiosError, type InternalAxiosRequestConfig } from "axios";

import { env } from "@/config/env";

/**
 * The one axios instance for the app. Call it from src/features/<feature>/service.ts,
 * never from components directly.
 *
 * Auth: the backend sets httpOnly cookies. On a 401 we call POST /auth/refresh once
 * (shared by all requests that failed at the same time) and replay the request.
 */
export const api = axios.create({
  baseURL: env.NEXT_PUBLIC_API_URL,
  timeout: 15_000,
  withCredentials: true,
  headers: { "Content-Type": "application/json" },
});

/** Every failed request rejects with an ApiError, so callers handle one shape. */
export class ApiError extends Error {
  status?: number;
  data?: unknown;

  constructor(message: string, status?: number, data?: unknown) {
    super(message);
    this.name = "ApiError";
    this.status = status;
    this.data = data;
  }
}

type RetryableConfig = InternalAxiosRequestConfig & { _retry?: boolean };

const REFRESH_PATH = "/auth/refresh";
let refreshPromise: Promise<void> | null = null;

function toApiError(error: AxiosError): ApiError {
  const body = error.response?.data as { message?: string } | undefined;
  return new ApiError(body?.message ?? error.message, error.response?.status, error.response?.data);
}

api.interceptors.response.use(
  (response) => response,
  async (error: unknown) => {
    if (!(error instanceof AxiosError)) return Promise.reject(error);

    const original = error.config as RetryableConfig | undefined;
    const isRefreshCall = original?.url?.includes(REFRESH_PATH);

    if (error.response?.status === 401 && original && !original._retry && !isRefreshCall) {
      original._retry = true;
      try {
        refreshPromise ??= api
          .post(REFRESH_PATH)
          .then(() => undefined)
          .finally(() => {
            refreshPromise = null;
          });
        await refreshPromise;
        return api(original);
      } catch {
        // Refresh failed: fall through with the original 401. UI code (e.g. a global
        // QueryCache onError) should router.push("/login") when it sees status 401.
      }
    }

    return Promise.reject(toApiError(error));
  },
);

/** Readable message for toasts and form errors. */
export function getErrorMessage(error: unknown): string {
  if (error instanceof Error) return error.message;
  if (typeof error === "string") return error;
  return "Something went wrong";
}
```

2. `src/proxy.ts`

```ts
import { NextResponse, type NextRequest } from "next/server";

/**
 * Route guard (Next.js 16 "proxy", formerly middleware).
 * Only checks that a session cookie exists — the backend still validates every request.
 * The cookie must be readable on this app's domain (same domain, or Domain=.example.com).
 */
const SESSION_COOKIE = "access_token"; // must match the cookie name your backend sets
const PUBLIC_PATHS = ["/login"];

export function proxy(request: NextRequest) {
  const { pathname } = request.nextUrl;
  const isPublic = PUBLIC_PATHS.some(
    (path) => pathname === path || pathname.startsWith(`${path}/`),
  );
  const hasSession = request.cookies.has(SESSION_COOKIE);

  if (!hasSession && !isPublic) {
    const loginUrl = new URL("/login", request.url);
    loginUrl.searchParams.set("next", pathname);
    return NextResponse.redirect(loginUrl);
  }

  if (hasSession && isPublic) {
    return NextResponse.redirect(new URL("/", request.url));
  }

  return NextResponse.next();
}

export const config = {
  // Skip Next internals, API routes and static files.
  matcher: [
    "/((?!api|_next/static|_next/image|favicon.ico|.*\\.(?:png|jpg|jpeg|gif|svg|webp|ico)$).*)",
  ],
};
```

3. `src/app/(auth)/login/page.tsx`

```tsx
import type { Metadata } from "next";

export const metadata: Metadata = { title: "Sign in" };

/** Placeholder — build the real form in the first auth task (docs/tasks/). */
export default function LoginPage() {
  return (
    <main className="flex flex-1 items-center justify-center px-6">
      <div className="w-full max-w-sm rounded-lg border bg-surface p-6 shadow-sm">
        <h1 className="text-xl font-semibold">Sign in</h1>
        <p className="mt-2 text-sm text-muted-foreground">
          Login form goes here: React Hook Form + zod → POST /auth/login → backend sets the cookie.
        </p>
      </div>
    </main>
  );
}
```

4. CLAUDE.md → Data & state: `- Auth → backend sets httpOnly cookie \`access_token\`; \`src/proxy.ts\` redirects to /login without it; \`api\` retries once after POST /auth/refresh on 401. Ask before changing cookie names or endpoints.`

If the user typed another auth option (Clerk, Better Auth, Auth.js, Supabase, …), set it up following that library's current Next.js App Router docs instead, and note it in CLAUDE.md.

### Tests: Vitest + Testing Library

Dev install: `vitest @vitejs/plugin-react jsdom @testing-library/react @testing-library/dom @testing-library/jest-dom @testing-library/user-event vite-tsconfig-paths`

`vitest.config.mts`

```ts
import react from "@vitejs/plugin-react";
import tsconfigPaths from "vite-tsconfig-paths";
import { loadEnv } from "vite";
import { defineConfig } from "vitest/config";

export default defineConfig(({ mode }) => ({
  plugins: [tsconfigPaths(), react()],
  test: {
    environment: "jsdom",
    setupFiles: ["./vitest.setup.ts"],
    include: ["src/**/*.test.{ts,tsx}"],
    // Make NEXT_PUBLIC_* from .env / .env.local available to src/config/env.ts in tests.
    env: loadEnv(mode, process.cwd(), "NEXT_PUBLIC_"),
  },
}));
```

`vitest.setup.ts` → `import "@testing-library/jest-dom/vitest";`

`src/lib/utils.test.ts`

```ts
import { describe, expect, it } from "vitest";

import { cn } from "@/lib/utils";

describe("cn", () => {
  it("merges conflicting Tailwind classes, last one wins", () => {
    expect(cn("px-2 text-sm", "px-4")).toBe("text-sm px-4");
  });

  it("drops falsy values", () => {
    expect(cn("block", false && "hidden", undefined)).toBe("block");
  });
});
```

Then: `npm pkg set scripts.test="vitest run" scripts.test:watch="vitest"`, add `- run: npm test` before the build step in `.github/workflows/ci.yml`, add `npm test` to the "done" checks in CLAUDE.md, and add to Data & state: `- Tests → Vitest + Testing Library, next to the code as \`*.test.ts(x)\`.`

### Tests: + Playwright e2e

Everything from Vitest, plus dev install `@playwright/test`, then `npx playwright install chromium`. Write `playwright.config.ts` (testDir `e2e`, baseURL `http://localhost:3000`, `webServer: { command: "npm run build && npm run start", url: "http://localhost:3000", reuseExistingServer: !process.env.CI }`, chromium project only) and `e2e/home.spec.ts` that opens `/` and expects the app name heading. Add `scripts.test:e2e="playwright test"`, add `test-results` and `playwright-report` to `.gitignore`, and set Vitest `include` so it ignores `e2e/`.

### UI: shadcn/ui

```bash
npx shadcn@latest init -d -y
```

Then reconcile `globals.css`: keep the variables shadcn wrote, set its `--primary` (light and dark) to the brand color and `--primary-foreground` to the brand foreground, and add our extra tokens (`--surface`, `--success`, `--warning`, `--danger`, `--info` and their `--color-*` entries) so nothing in this skill's pages breaks. Keep one `@custom-variant dark` line. Update `docs/designs/design-system.md` to list shadcn's token names, and add to CLAUDE.md: `- UI primitives → shadcn/ui in src/components/ui (add with \`npx shadcn@latest add <name>\`); restyle via tokens, not by editing every class.`

### Extras

**Dark mode (next-themes)** — install `next-themes`. Write `src/components/ThemeToggle.tsx`:

```tsx
"use client";

import { useTheme } from "next-themes";
import { LuMoon, LuSun } from "react-icons/lu";

/** Icons swap with the `dark:` variant, so there is no hydration flicker. */
export default function ThemeToggle() {
  const { resolvedTheme, setTheme } = useTheme();

  return (
    <button
      type="button"
      aria-label="Toggle dark mode"
      onClick={() => setTheme(resolvedTheme === "dark" ? "light" : "dark")}
      className="inline-flex size-9 items-center justify-center rounded-md border text-foreground hover:bg-muted focus-visible:ring-2 focus-visible:ring-ring focus-visible:outline-none"
    >
      <LuSun className="size-4 dark:hidden" />
      <LuMoon className="hidden size-4 dark:block" />
    </button>
  );
}
```

Put it on the home page next to the badge: wrap the badge in `<div className="flex items-center justify-between">…<ThemeToggle /></div>`.

**Toasts (sonner)** — install `sonner`; mount `<Toaster richColors position="top-right" />` in AppProviders. CLAUDE.md: `- Toasts → \`toast.success()\` / \`toast.error(getErrorMessage(e))\` from sonner.`

**React Compiler** — dev install `babel-plugin-react-compiler`; add `reactCompiler: true,` to `next.config.ts` under `poweredByHeader`. CLAUDE.md: `- React Compiler is on — don't add useMemo/useCallback by default.`

### Compose `src/providers/AppProviders.tsx`

Write the final file with only the chosen providers, nested in this order:
ThemeProvider → (StoreProvider | QueryClientProvider) → children + Toaster + ReactQueryDevtools.
Full version (Zustand/Query + dark mode + toasts) — delete what wasn't chosen:

```tsx
"use client";

import { QueryClientProvider } from "@tanstack/react-query";
import { ReactQueryDevtools } from "@tanstack/react-query-devtools";
import { ThemeProvider } from "next-themes";
import type { ReactNode } from "react";
import { Toaster } from "sonner";

import { getQueryClient } from "@/lib/query-client";

/** Every client-side provider is composed here and mounted once in src/app/layout.tsx. */
export default function AppProviders({ children }: { children: ReactNode }) {
  const queryClient = getQueryClient();

  return (
    <ThemeProvider attribute="class" defaultTheme="system" enableSystem disableTransitionOnChange>
      <QueryClientProvider client={queryClient}>
        {children}
        <Toaster richColors position="top-right" />
        <ReactQueryDevtools initialIsOpen={false} />
      </QueryClientProvider>
    </ThemeProvider>
  );
}
```

For Redux, swap `QueryClientProvider` (and devtools, `getQueryClient`) for `<StoreProvider>` from `@/providers/StoreProvider`.

### Brand color

Set `--brand` in `src/app/globals.css` and the Brand table in `docs/designs/design-system.md`.
`--brand-foreground`: `#0a0a0a` if the brand is light (relative luminance > 0.4, e.g. teal `#00d4c8`), else `#ffffff`.
If the user said "later" or will paste their own design system, keep indigo and say so in the summary.

### Project description

Turn the answer to question 8 into one clear sentence and replace `{{DESCRIPTION}}` / `A Next.js app.` in `src/config/site.ts`, `README.md` and `CONTEXT.md`. Fill **Users** in CONTEXT.md with the obvious user types for that kind of app (keep it to 1–3, marked as a guess).

---

## Phase 4 — Verify, commit, report

1. Run, fixing anything red before the next one:
   `npm run format` → `npm run lint` → `npm run typecheck` → `npm test` (if Vitest) → `npm run build`
2. Commit (Husky + commitlint run here, which also proves the hooks work):

```bash
git add -A
```

```bash
git commit -m "chore: scaffold next.js app with /nextjs"
```

3. Final message — short, no file dumps:
   - One line: what was created and that lint/typecheck/build pass.
   - The choices, as a compact list (state, forms, auth, tests, UI, extras, brand).
   - Anything adapted for version drift.
   - Next steps: `npm run dev`, fill `.env.local`, then offer: "Want me to grill you about the first feature and write `docs/prd/0001-….md`?"
