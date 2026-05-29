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
│   ├── layouts/         # shared page shells
│   │   └── Layout.astro
│   └── pages/           # each .astro file = a route
│       └── index.astro
├── astro.config.mjs
├── tsconfig.json
└── package.json
```

## Deployment

Designed for **Cloudflare Pages** (free, auto-deploy from GitHub):

- Build command: `npm run build`
- Output directory: `dist`

## License

Private / personal project.
