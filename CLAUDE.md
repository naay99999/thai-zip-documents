# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

The documentation website for both Thai address packages, deployed on Vercel at https://thai-zip.vercel.app:

- `thaizip` (core search library, repo `naay99999/thai-zip`) — `content/docs/(core)/`, served at `/docs/*`
- `react-thaizip` (CLI that scaffolds the React components, repo `naay99999/react-thai-zip`) — `content/docs/react/`, served at `/docs/react/*`

It consumes both packages **from npm**, exactly like a user project: nothing here imports another repo's source. Bump `thaizip` / `react-thaizip` in `package.json` to document a new release; the live previews, the "View Code" sources, and the manual install steps all follow the installed version.

## Commands

```bash
npm run dev        # generate .generated/ then next dev
npm run build      # generate .generated/ then next build --webpack (typechecks too)
npm test           # vitest: structure guards + Markdown/install-step output
npm run typecheck  # generate .generated/ then tsc --noEmit
```

Use Node 22+ (CI pins Node 24).

## How the components get in

`scripts/generate-styles.mjs` runs as `predev`/`prebuild` and writes the gitignored `.generated/`:

- `.generated/templates/` — `react-thaizip`'s `templates/react/ts/` (minus `shadcn/`) copied out of `node_modules`. Next.js won't compile TypeScript inside `node_modules`, hence the copy. The `@/*` and `@/components/*` tsconfig aliases point here, so templates resolve their own `@/lib/utils` / `@/hooks/*` imports.
- `.generated/styles/<base>/` — each shadcn/ui variant (`base`, `radix`, `aria`) next to the shadcn primitives it composes, with `@/components/ui/*` imports rewritten to relative ones (all three bases use the same specifiers but need different primitives, which one alias can't express).

The shadcn primitives themselves are **vendored** in `vendor/shadcn/<base>/` — `react-thaizip` doesn't publish them (a real project gets them from the shadcn CLI). They're excluded from `tsconfig.json` because their raw `@/components/ui/*` imports only resolve once copied into `.generated/`. When a new `react-thaizip` release makes a template use a primitive that isn't vendored yet, copy it from `react-thai-zip/templates/react/ts/shadcn/__fixtures__/<base>/components/ui/` (or run the pinned `shadcn add` recipe in that repo's CLAUDE.md) into all three bases. Primitive npm dependencies (`radix-ui`, `react-aria-components`, `cn`, `lucide-react`, `class-variance-authority`) are regular dependencies here for the same reason.

`styles/component-demo.css` has `@source "../.generated"` so Tailwind picks up the templates' classes.

## Live examples and the style picker

Each example is a standalone file in `examples/<name>.tsx` (importing `@/components/thai-address-*`), registered in `examples/registry.ts`, and rendered by `<ComponentPreview name="..." size="sm|md|lg" />` with that same file's source collapsed underneath — so the shown code can't drift from the demo. Component pages have a page-level style picker (Vanilla / Base UI / Radix UI / React Aria, `components/docs-style.tsx`, remembered in localStorage): the four stateful components' `@/components/thai-address-*` imports are aliased in `tsconfig.json` to `components/style-switch/*`, which render the vanilla template or the shadcn variant for the chosen style. `address-display` has no shadcn variant, so it maps straight to `.generated/templates/` and gets no picker.

`<ComponentInstall name="..." />` renders the CLI command plus manual steps for the picked style. `lib/component-install.ts` derives them from `react-thaizip`'s published `templates/registry.json` (the CLI's registry as data) with small local copies of the CLI's `resolveWithDependencies`/`selectVariant`; source links point at the `react-thaizip-v<installed version>` tag. Its commands, and ```` ```npm ```` blocks in MDX (remarkNpm), render as npm/pnpm/yarn/bun tabs sharing one persisted `package-manager` group.

## Markdown for AI

`ComponentPreview` and `ComponentInstall` are `mdxAsPlaceholder` in `lib/source.ts` and expanded by `lib/docs-markdown.ts`, so every page's Markdown (`/docs.md`, `/docs/<slug>.md`, `/llms.txt`, `/llms-full.txt`, all statically generated, absolute URLs) carries the example source and install steps. `lib/` uses extensionless relative imports (Next needs them) and keeps fumadocs out of `docs-markdown.ts`, so vitest can import it directly.

## Site structure

Fumadocs on Next.js. The UI package is `@fumadocs/base-ui` (the Base UI build of `fumadocs-ui`, same subpaths) rather than `fumadocs-ui`, because only it ships `layouts/spacious` — the layout fumadocs.dev itself uses; `app/global.css` carries fumadocs.dev's palette (amber `--core-color`, blue `--react-color` for pages under `/docs/react`, switched by the `section-react` class the docs page sets). Build is `next build --webpack` for Fumadocs MDX compatibility. Content is bilingual: Thai at the root locale, English under `content/docs/en/` — `tests/docs-site.test.ts` fails if a required page exists in only one language. `(core)/` and `react/` are each a root folder rendered as a sidebar tab. URLs from before the two packages' docs were merged are kept alive by `redirects()` in `next.config.mjs`.

## CI and deploy

`.github/workflows/ci.yml` runs `npm ci --ignore-scripts`, `npm test`, and `npm run build` on pushes to `main` and all PRs. Vercel project `thai-zip` (team naays99999-projects) is connected to this repo: pushes to `main` deploy to production at https://thai-zip.vercel.app, other branches get preview deployments. `vercel.json` pins the Next.js framework preset (a CLI-created project otherwise defaults to "Other"). This repo isn't published to npm and has no release process; land changes through a PR.
