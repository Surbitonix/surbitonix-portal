# surbitonix-portal

Personal web portal for **surbitonix.com**, built with [Astro](https://astro.build/).
A modern, fast, mostly-static site designed for easy hosting on Cloudflare Pages.

## Requirements

- **Node 22** (pinned in `.nvmrc`). With [nvm](https://github.com/nvm-sh/nvm):

  ```bash
  nvm install   # installs the version from .nvmrc (22)
  nvm use       # switches to it
  ```

## Getting started

```bash
npm install      # install dependencies
npm run dev      # start the dev server at http://localhost:4321
```

## Commands

| Command           | Action                                       |
| ----------------- | -------------------------------------------- |
| `npm run dev`     | Start local dev server at `localhost:4321`   |
| `npm run build`   | Build the production site to `./dist/`       |
| `npm run preview` | Preview the production build locally         |

## Project structure

```text
surbitonix-portal/
├── public/              # static assets (favicon, images)
├── src/
│   ├── layouts/         # shared page shells + global CSS variables
│   │   └── Layout.astro
│   ├── components/      # reusable building blocks
│   │   ├── Intro.astro      # cinematic entry animation
│   │   └── Console.astro    # interactive mini-terminal
│   └── pages/           # each .astro file = a route
│       └── index.astro      # the landing page
├── docs/
│   └── how-it-works.md  # deep dive: Astro intro + how the animations work
├── astro.config.mjs
├── tsconfig.json
└── package.json
```

## How it works / learning the codebase

New to Astro, or want to understand the entry animation, the typewriter brand,
and the interactive terminal? See **[docs/how-it-works.md](docs/how-it-works.md)** —
a walkthrough written to learn from, with pointers to the real code.

## Deployment

Designed for **Cloudflare Pages** (free, auto-deploy from GitHub):

- Build command: `npm run build`
- Output directory: `dist`

## License

Private / personal project.
