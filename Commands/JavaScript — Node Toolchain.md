## What & When

This note covers the **Node.js runtime** and everyday **CLI package tooling** — **nvm** (version manager), **npm** and **yarn** (dependencies), and how they fit before [[Commands/JavaScript — Vite]] or [[Commands/JavaScript — Next.js]] projects.

Hub: [[JavaScript Development]]

---

## Node.js

**Node.js** runs JavaScript outside the browser — dev servers, build tools, [[Codes/JavaScript — Puppeteer]], and [[Load Testing — k6]] scripts.

```bash
node --version    # v20+ or v22 LTS recommended
node script.js
node --watch app.js
```

Download: [nodejs.org](https://nodejs.org/) or install via **nvm** (preferred for multiple versions).

---

## nvm — Node Version Manager

Install multiple Node versions per project (`.nvmrc`).

```bash
# install nvm — see https://github.com/nvm-sh/nvm#installing-and-updating
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash

nvm install 22
nvm use 22
nvm alias default 22

# project pin
echo "22" > .nvmrc
nvm use
```

| Command | Role |
| --- | --- |
| `nvm install 20` | Install Node 20.x |
| `nvm use 20` | Switch shell to Node 20 |
| `nvm ls` | List installed versions |
| `nvm current` | Show active version |

---

## npm — Default Package Manager

Ships with Node. Manages `package.json` dependencies and scripts.

```bash
npm init -y
npm install react
npm install -D typescript vite
npm run dev
npm ci              # clean install from lockfile (CI)
```

| File | Role |
| --- | --- |
| `package.json` | metadata, scripts, dependency ranges |
| `package-lock.json` | exact resolved tree (commit to git) |
| `node_modules/` | installed packages (do not commit) |

Common scripts:

```json
{
  "scripts": {
    "dev": "vite",
    "build": "tsc -b && vite build",
    "preview": "vite preview",
    "lint": "eslint ."
  }
}
```

---

## yarn — Alternative Package Manager

```bash
corepack enable          # Node 16+ — enables yarn/pnpm via Corepack
yarn init -y
yarn add react
yarn add -D typescript
yarn dev
yarn install --frozen-lockfile   # CI equivalent of npm ci
```

| | npm | yarn |
| --- | --- | --- |
| Lockfile | `package-lock.json` | `yarn.lock` |
| Install all | `npm install` | `yarn` |
| Add dep | `npm install pkg` | `yarn add pkg` |
| Run script | `npm run dev` | `yarn dev` |

Pick **one** manager per repo; do not mix lockfiles.

---

## Project Bootstrap

```bash
nvm use
npm create vite@latest my-app -- --template react-ts
cd my-app
npm install
npm run dev
```

Or Next.js: [[Commands/JavaScript — Next.js]]

---

## Environment Variables

```bash
# shell
export VITE_API_URL=http://localhost:8000

# .env.local (Vite — prefix public vars with VITE_)
VITE_API_URL=http://localhost:8000
```

Next.js uses `NEXT_PUBLIC_` prefix — see [[Commands/JavaScript — Next.js]].

---

## npx / yarn dlx

Run packages without global install:

```bash
npx create-vite@latest
npx tsc --init
yarn dlx create-next-app@latest
```

---

## Node vs Python Tooling

| Task | Node | Python |
| --- | --- | --- |
| Version manager | **nvm** | pyenv, uv |
| Package install | npm / yarn | pip, uv |
| API server | Next.js, Express | [[API - FastAPI]] |
| Unit tests | Vitest, Jest | [[Unit Testing - pytest]] |

---

## Quick Reference

| Task | Command |
| --- | --- |
| Node version | `node -v` |
| Switch Node | `nvm use` |
| Init project | `npm init -y` |
| Install deps | `npm install` |
| CI install | `npm ci` |
| Run script | `npm run <name>` |
| Global tool (avoid) | prefer `npx` |

---

## Related Notes

- [[Commands/JavaScript — Vite]]
- [[Commands/JavaScript — Next.js]]
- [[Codes/JavaScript — Basics]]
- [[JavaScript Development]]
- [[Load Testing — k6]]

---

## Tags

#nodejs #npm #yarn #nvm #cli #javascript #toolchain #frontend
