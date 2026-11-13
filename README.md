# Spec-Driven WordPress Theme

A WordPress starter theme built from scratch from a written specification. Performance, technical SEO, accessibility and security are part of the base, and every rule is documented so that AI coding agents (Claude Code, Codex) build new projects the same way a careful developer would.

> **Source is private.** This repository explains the approach, the architecture and the technical decisions behind the theme. The code is available for technical review on request.

**Results:** 90+ on PageSpeed Insights (mobile and desktop) on sites built with it. An institutional site takes a few days, and a landing page up to one day, depending on scope.

<img width="1900" height="909" alt="image" src="https://github.com/user-attachments/assets/a1f9c6c8-37e8-4693-b18b-4db371743c8b" />
<img width="1901" height="905" alt="image" src="https://github.com/user-attachments/assets/a4a56fd5-e2b5-457d-af26-7b62c369e832" />

## The problem

Most WordPress sites I inherited were built on heavy builder themes: slow pages, markup nobody controls, and a different structure on every project. Adding AI agents to that workflow made it faster, but not more consistent. An agent with no rules invents its own conventions on every task.

The goal was a base where **the rules come before the code**, and where breaking a rule fails a check instead of depending on someone noticing it in review.

## How it works: spec → build → verify

```
Specification (docs/)        Build                      Verify
─────────────────────        ─────                      ──────
conventions, naming,   --->  AI agent + developer  ---> automated gates
performance budget,          follow skills that         (lint, conventions,
security model,              read only the docs         security, a11y,
SEO and a11y rules           the task needs             Lighthouse, fidelity)
                                                        + human review of every diff
```

- **The specification is the source of truth.** Eighteen documents cover code style and naming, design implementation, performance, SEO, accessibility, security, forms, content types and QA. An `AGENTS.md` at the root tells any agent what to read and in what order.
- **Context is loaded on demand.** A small core is always read. The other documents have objective triggers: the performance doc is read only when the section has an LCP image or a new JS library, the security doc only when code handles input or a deploy is being prepared. This keeps the agent's context lean and its output predictable.
- **Repeatable tasks are written as skills**: tool-neutral playbooks for building a section, building a page, turning approved layout into editable fields, SEO, QA and a pre-deploy security pass.
- **A rule that can be checked becomes a script, not just a sentence in a doc.** Hardcoded colors, images without dimensions, third-party tags in templates and static JS imports are all caught by `npm run lint`, not left to review.
- **The theme learns from projects.** Non-obvious bugs go to a gotchas log, and structural gaps found during a real build go to an evolution log. That way the next project starts from a better base instead of repeating the same diagnosis.

## Architecture

- **PHP templates + structured fields.** Pages are composed from section partials (`template-parts/sections/<page>/`), reusable components, and layout parts. Template hierarchy rules are explicit: the WordPress hierarchy files live at the root, and admin-selectable page templates live in `templates/`.
- **Editable content without ACF Pro.** Field definitions are versioned JSON blueprints that the theme registers at runtime, with deterministic keys, so renaming a group never orphans saved values. Repeaters, galleries and global options are implemented natively. Content types and taxonomies are declared in JSON too.
- **Design tokens in three levels**: raw palette → semantic theme tokens (Tailwind CSS v4 `@theme`) → component aliases. Only the raw palette changes per project. `theme.json` mirrors the tokens, and a check fails if they diverge.
- **CSS:** Tailwind utilities in the markup. Custom CSS is allowed only in three documented cases, and it lives in a partial that mirrors the PHP file path, with a matching class prefix that a script enforces.
- **JS:** a single ES module entry. Section and component JS is loaded on demand with dynamic `import()`, bundled with esbuild.

## Decisions and trade-offs

**Performance**
- **The LCP image is preloaded with `imagesrcset`/`imagesizes`**, using the same size and `sizes` as the `<img>`. A preload with a plain URL can download a second, different file on the critical path.
- **Third-party scripts (GTM, pixels, chat) wait for the first user interaction**, with an optional timeout. Trade-off: analytics start a moment later, in exchange for a clean main thread during load.
- **Video and map embeds use facades**: YouTube, Vimeo and maps load on click. A background video uses a poster as the LCP element, and the video comes in afterwards.
- **Self-hosted fonts, block CSS only where blocks exist, form plugin assets only on pages with a form.**
- **A Lighthouse gate before every deploy**: median of 3 runs on the mobile profile, failing below 90 or when LCP/CLS/TBT exceed their budgets, and reporting what pulled the score down. It measures staging with page cache on. Localhost doesn't represent the real server.

**Security**
- **Hardened by default:** security headers on the front end, XML-RPC off, REST closed to anonymous users (with explicit exceptions), file editor disabled, generic login errors and strict upload sanitization. Blocking PHP execution in `uploads/` is a server rule, since no theme code can prevent a file write, so the docs ship the server configuration for it.
- **Login throttling counts attempts by `REMOTE_ADDR`, not `X-Forwarded-For`.** Proxy headers are client-controlled text. Trusting them by default would let an attacker bypass the limit by changing a header on every attempt. Behind a trusted proxy, this is opt-in.
- **No Content-Security-Policy by default, and that is a documented decision.** Real WordPress sites rely on inline scripts (core, form plugins, tag managers). A CSP loose enough not to break them needs `'unsafe-inline'`, and then it no longer protects against XSS. When a project needs CSP, it goes on the server, in report-only mode first.
- **A static security scan in the lint step**: forbidden artifacts (backups, dumps, keys), plaintext secrets, and obfuscation patterns typical of injected backdoors in PHP and in compiled JS. A suppression without a written reason doesn't count.
- **An integrity manifest**: SHA-256 hashes of every delivered file, versioned with the code. On a live server there is no Git, so checking the manifest answers "are these exactly the files that left the repository?" in seconds. This is the first question when a WordPress site may be compromised.

**SEO and accessibility**
- **Native technical SEO**: titles, meta description, canonical, Open Graph, a clean sitemap and JSON-LD (Organization or LocalBusiness, WebSite, Article, BreadcrumbList). **It steps aside automatically** when a major SEO plugin is active, to avoid duplicate tags.
- **WCAG 2.2 AA baseline**: skip link, focus that is always visible and never hidden under a fixed header, `prefers-reduced-motion`, sections with associated headings, and an automated axe audit that fails on serious or critical violations.

**Design fidelity**
- **No approximation:** an exact value from the reference becomes a token. It is never rounded to the nearest default. Each section is measured against the reference on desktop and mobile before the next one starts.

## Limitations

- **Classic PHP templates, not a block theme.** That gives predictable markup and performance, and editors can't break the layout. The trade-off is less editorial freedom than a block-based site.
- **Validations run locally** (`npm run lint`, `build`, `audit`), with no CI pipeline in the theme itself.
- **Login throttling is per IP.** It doesn't stop a distributed botnet. 2FA and edge rules (Cloudflare, fail2ban) are still recommended.
- **Requires PHP 8.2+** and WordPress 6.4+.

## Stack

WordPress · PHP 8.2 · ACF (free) · Tailwind CSS v4 · esbuild · ESLint · Stylelint · Lighthouse · axe · WP-CLI · wp-env (Docker)
