# How surbitonix.com works

A learning-oriented walkthrough of this site: a quick intro to **Astro**, then
how each piece (the entry animation, the typewriter brand, and the interactive
terminal) actually works — with pointers to the real files so you can poke at
them.

---

## 1. Astro in 90 seconds

[Astro](https://astro.build/) is a framework for building **fast, mostly-static
websites**. The big ideas you're using here:

- **Static-first.** `npm run build` renders your pages to plain HTML/CSS in
  `dist/`. There's no server doing work per request — a CDN (Cloudflare Pages)
  just serves files. That's why it's cheap and fast.
- **`.astro` components.** A reusable building block. Every `.astro` file has
  the same three-part anatomy:

  ```astro
  ---
  // 1. FRONTMATTER (runs at build time, on the server / your machine).
  //    Plain JavaScript/TypeScript: imports, data, props.
  const greeting = 'hello';
  ---

  <!-- 2. TEMPLATE (the HTML this component outputs) -->
  <h1>{greeting}</h1>

  <!-- 3. STYLES + SCRIPTS -->
  <style>
    /* scoped to THIS component by default — no global leakage */
    h1 { color: cyan; }
  </style>
  <script>
    // ships to the browser; runs on the visitor's device
    console.log('client side');
  </script>
  ```

- **File-based routing.** A file at `src/pages/index.astro` becomes the `/`
  route. `src/pages/about.astro` would become `/about`. No router config.
- **Components are imported, not registered.** In the frontmatter you
  `import Console from '../components/Console.astro'` and then use it as
  `<Console />` in the template.
- **Two execution times — keep them straight:**
  - _Frontmatter_ (between the `---` fences) runs **at build time**.
  - `<script>` blocks run **in the browser** at runtime. All the animation
    logic lives in `<script>` blocks because it needs to react to the user.

That's genuinely most of what you need to read this codebase.

---

## 2. Project map

```text
surbitonix-portal/
├── public/                 # static assets served as-is (favicon, images)
├── src/
│   ├── layouts/
│   │   └── Layout.astro     # the HTML <head>/<body> shell + global CSS vars
│   ├── components/
│   │   ├── Intro.astro      # the cinematic entry animation (stage B → A)
│   │   └── Console.astro     # the interactive mini-terminal easter egg
│   └── pages/
│       └── index.astro       # the landing page (uses Layout + Intro + Console)
├── docs/
│   └── how-it-works.md       # you are here
├── astro.config.mjs
└── package.json
```

`Layout.astro` defines the shared shell and the **CSS custom properties** (the
color palette) in `:root` — `--bg`, `--fg`, `--accent`, `--accent-2`,
`--muted`, `--border`. Every component references those variables, so the theme
is consistent and changeable in one place.

---

## 3. The landing page (`src/pages/index.astro`)

Two structural pieces:

1. **The hero** — the badge, the big `surbitonix_` heading, and the tagline.
2. **The workspace** — a responsive 2-column grid:
   - left: the project + connect **tiles**
   - right: the **terminal**

   On screens narrower than `880px` it collapses to a single column (tiles
   first, terminal last) via a `@media (min-width: 880px)` rule. That's the
   mobile-friendly behavior.

The tile contents are plain data arrays in the frontmatter (`projects` and
`links`), looped over with `.map(...)` in the template. To add a project or
change a tagline, you edit the array — no markup surgery. Each project has a
`status: 'soon' | 'live'` that drives the little pill and (later) whether
`cd <project>` in the terminal actually navigates.

---

## 4. The entry animation (`src/components/Intro.astro`)

A full-screen overlay (`#intro`, `z-index: 9999`) that plays on load, then
fades away to reveal the page. It has **two stages**:

- **Stage B — the cityscape.** A background image with a slow zoom (the
  "Ken Burns" effect), done purely in CSS:

  ```css
  .kenburns {
    animation: kenburns 6s ease-out forwards;
  }
  @keyframes kenburns {
    from { transform: scale(1.05); }
    to   { transform: scale(1.18) translateY(-1.5%); }
  }
  ```

- **Stage A — the terminal boot.** A `<pre>` that gets text typed into it by
  JavaScript, ending on the `Press Enter to continue....` prompt.

### How the sequence is driven (the `<script>`)

The logic is an `async` function using a tiny `wait(ms)` helper that returns a
promise wrapping `setTimeout`. That lets it read top-to-bottom like a script:

```ts
await wait(3500);              // hold the cityscape
imageStage.classList.remove('is-active');
termStage.classList.add('is-active');   // cross-fade to the terminal
await wait(700);
for (const line of lines) await typeLine(line);  // type the boot lines
await typeLine(hint);          // "Press Enter to continue...."
await wait(AUTO_ADVANCE_MS);   // wait, then auto-advance
finish();
```

`typeLine` appends one character at a time with `await wait(55)` between each —
that's the typing effect.

### Ending it: `finish()`

`finish()` is the single exit. It can be called three ways: the boot sequence
finishes and auto-advances, the user presses **Enter/Esc**, or clicks **Skip**.
It:

1. adds `fade-out` (CSS fades the overlay out over 0.8s),
2. **dispatches a `intro:reveal` event** (important — see §5),
3. after 800ms adds the `intro-done` class to `<html>`, which `display: none`s
   the overlay and unlocks page scroll.

### Accessibility

An inline script at the top checks
`matchMedia('(prefers-reduced-motion: reduce)')`. If the visitor prefers
reduced motion, it adds `intro-done` **before paint** so the animation never
plays — they go straight to the page. This pattern (respect reduced motion)
shows up again in the typewriter.

---

## 5. The typewriter brand (`surbitonix_`)

This is the effect where `surbitonix` types itself out on the landing page,
leaving the `_` blinking. It lives in a `<script>` at the bottom of
`index.astro` (`setupBrandTypewriter`).

The interesting part is **timing**, and it's worth understanding because it was
a real bug we fixed:

- The word must type out **only after the intro reveals the page** — otherwise
  it types while hidden behind the overlay and you miss it.
- The first attempt used a fixed "type after N seconds" timer as a fallback.
  But the intro can run ~18 seconds (if you let it auto-advance), and the timer
  was set to 12s — so it fired *during* the intro, behind the overlay. The
  animation "worked" but was invisible. Classic **race condition**.

The fix is **event coordination** instead of guessing with timers:

1. `Intro.astro`'s `finish()` dispatches `document.dispatchEvent(new CustomEvent('intro:reveal'))`
   exactly when the overlay starts fading.
2. The typewriter listens for that event and starts typing 200ms later, so the
   letters appear as the page fades in:

   ```ts
   document.addEventListener('intro:reveal', () => setTimeout(type, 200), { once: true });
   ```

3. There's still a long (30s) safety net and a check for "no intro present", so
   it's robust if the intro ever changes or is removed.

The typing itself is dead simple — slice the word one more character each tick:

```ts
const step = () => {
  brand.textContent = word.slice(0, i);
  if (i < word.length) { i++; setTimeout(step, 150); }  // 150ms = speed knob
};
```

The blinking `_` is a separate `<span class="cursor">` with a CSS `blink`
animation — it's always there; the typed word just grows in front of it.

**Lesson worth keeping:** when two animations depend on each other, make one
*signal* the other (events/callbacks) rather than assuming a fixed duration.

---

## 6. The interactive terminal (`src/components/Console.astro`)

A fake-but-real shell. The structure:

- **Markup:** a title bar (the three traffic-light dots), a scrolling output
  area (`#console-out`), and an `<input>` for typing commands.
- **A `commands` object** maps command names to functions:
  `help`, `ls`, `about`, `projects`, `cd`, `contact`, `neofetch`, `whoami`,
  `date`, `clear`. Looking a command up is just `commands[name]`.
- **`run(raw)`** parses the typed line into `command + args`, handles a few
  special cases (`echo`, `sudo hire-me`, `cat`, `rm -rf` easter egg), then
  dispatches to the `commands` map, or prints `command not found`.
- **History:** an array of past commands; **↑/↓** walk through it.
- **Output is built with small `print(html)` calls** that append a `<div>` to
  the output area, then auto-scroll to the bottom.

`cd fxorbit` / `cd jyra` check a `projects` map's `live` flag: while `false`
they print `[soon]`; once you flip them to `true`, they'll `window.location`
to the subdomain. (Keep that flag in sync with the tile `status` in
`index.astro`.)

User input is passed through an `escapeHtml` helper before being echoed, so
typing `<script>` can't inject anything — a small but real security habit.

---

## 7. Running & deploying

```bash
npm install      # once
npm run dev      # local dev server (localhost:4321, or :4322 if busy)
npm run build    # production build into dist/
npm run preview  # serve the built dist/ locally to sanity-check
```

**Deployment is automatic.** The GitHub repo is connected to **Cloudflare
Pages**: every push to the main branch triggers a build (`npm run build`) and
publishes `dist/`. The custom domain `surbitonix.com` points at it via a
`CNAME` record in Cloudflare DNS. So the workflow is just: commit → push →
it's live.

---

## 8. Tweak cheat-sheet

| I want to change…                  | File · what to edit                                            |
| ---------------------------------- | -------------------------------------------------------------- |
| Brand typing **speed**             | `index.astro` → `setTimeout(step, 150)` (ms per char)          |
| Brand typing **start delay**       | `index.astro` → `setTimeout(type, 200)` after `intro:reveal`   |
| Intro **cityscape hold**           | `Intro.astro` → `await wait(3500)`                             |
| Intro **boot-line text/speed**     | `Intro.astro` → `lines[]` and `await wait(55)` in `typeLine`   |
| Intro **auto-advance wait**        | `Intro.astro` → `AUTO_ADVANCE_MS`                              |
| Add / edit a **project tile**      | `index.astro` → `projects[]` array                             |
| Add / edit a **connect tile**      | `index.astro` → `links[]` array                                |
| Mark a project **live**            | `index.astro` `status: 'live'` **and** `Console.astro` `live: true` |
| Add a **terminal command**         | `Console.astro` → add to the `commands` object                 |
| **Colors / theme**                 | `Layout.astro` → `:root` CSS variables                         |
| **Two-column breakpoint** (mobile) | `index.astro` → `@media (min-width: 880px)`                    |
