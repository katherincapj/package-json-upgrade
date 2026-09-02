# frontend-project-upgrade Skill

## Purpose

Identifies and validates frontend projects during the dependency upgrade process.
Provides framework-specific validation logic, build commands, and scripts for
Vue, React, Svelte, Angular, or other frontend frameworks.

## Frontend Project Classification

A project is classified as **frontend** if it contains one or more of these indicators:

1. **Framework indicators in package.json**:
    - `vue` or `vue-next` (Vue 2/3)
    - `react`, `preact`, `solid-js`
    - `svelte` or `sveltekit`
    - `next`, `nuxt`, `remix`, `astro`
    - `@angular/core`

2. **Build tools specific to frontend**:
    - `webpack`, `vite`, `parcel`, `rollup`
    - `@vitejs/plugin-*` (vite plugins)
    - `@vue/cli-service`, `vue-cli`
    - `create-react-app` configuration

3. **Frontend-only dev dependencies**:
    - `vue-loader`, `ts-loader`
    - `babel-loader`, `style-loader`, `sass-loader`
    - `postcss`, `tailwindcss`, `less`, `sass`
    - `storybook`, `chromatic`

4. **Frontend-specific scripts in package.json**:
    - `serve`, `dev`, `start` (web development server)
    - `build` (with no compile step, just bundling)
    - `preview` (vite-style)

## Validation Steps for Frontend Projects

When upgrading frontend dependencies, run these validation scripts in order
(skip if not present in package.json):

```
1. typecheck    (via tsc, vue-tsc, or similar)
2. lint         (via eslint, stylelint, or similar)
3. test         (unit tests — jest, vitest, mocha, etc.)
4. build        (webpack, vite, or bundler build)
```

## Common Frontend Commands by Framework

### Vue 2 (@vue/cli)

- Dev: `npm run serve`
- Build: `npm run build`
- Test: `npx vue-cli-service test:unit`
- Lint: `npm run lint`

### Vue 3 (Vite)

- Dev: `npm run dev`
- Build: `npm run build`
- Test: `npm run test`
- Lint: `npm run lint`

### React (Create React App)

- Dev: `npm start`
- Build: `npm run build`
- Test: `npm test -- --passWithNoTests`
- Lint: `npm run lint` or ESLint via package scripts

### React (Vite)

- Dev: `npm run dev`
- Build: `npm run build`
- Test: `npm run test`
- Lint: `npm run lint`

### Next.js

- Dev: `npm run dev`
- Build: `npm run build`
- Test: `npm run test` or `npm run test:e2e`
- Lint: `npm run lint` or `next lint`

## Common Failure Modes

**Peer dependency conflicts** — frontend frameworks often have strict peer requirements:

- React version mismatches with react-dom, @types/react
- Vue version with vue-loader or @vitejs/plugin-vue
- Angular version with @angular/\* packages
- Vite version with @vitejs/plugin-\* packages

**TypeScript incompatibility** — many frontend packages rely on specific TypeScript versions:

- React type definitions (@types/react)
- Vue type definitions
- Vite TypeScript plugin versions

**Lint rule changes** — ESLint, stylelint, or Prettier upgrades may introduce new rules:

- Breaking rule behavior (off → error)
- New mandatory rule configurations
- Syntax changes in rule options

## Rollback Guidance

For frontend projects:

1. Never rollback a build-tool package (webpack, vite) without rolling back plugins together
2. Check coupled-package groups in upgrade-policy.yaml — they must move as a unit
3. If peer conflict occurs, classify as `peer-conflict` and skip (policy will decide fallback)
4. Pre-existing test failures are rare in frontend; usually indicates a breaking change

---

## Usage

When the upgrade-agent classifies a repo as **frontend**, it should:

1. Reference this skill for validation order and common commands
2. Extract framework-specific build/test scripts from package.json
3. Use policy.validation.order but adjust for frontend-specific available scripts
4. Classify failures as detailed in "Common Failure Modes"
