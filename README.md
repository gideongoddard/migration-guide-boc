# Brixworth Osteopathic Clinic website

This README is for whoever is maintaining the Brixworth Osteopathic Clinic website — it explains what the site is built with, how to work on it locally, how it gets deployed, and what's due for modernisation.

If you've just taken over ownership of this site, start with the handover guide (separate document) for the one-time ownership transfer steps. This README is your ongoing reference for everything after that.

---

## Table of contents

- [Overview](#overview)
- [Tech stack](#tech-stack)
- [How the pieces fit together](#how-the-pieces-fit-together)
- [Running the site locally](#running-the-site-locally)
- [Project structure](#project-structure)
- [Content editing (for the client)](#content-editing-for-the-client)
- [Build and deploy](#build-and-deploy)
- [Contact form submissions](#contact-form-submissions)
- [Managing CMS users](#managing-cms-users)
- [Common tasks](#common-tasks)
- [Deferred modernisation work](#deferred-modernisation-work)
- [Troubleshooting](#troubleshooting)
- [Useful links](#useful-links)

---

## Overview

The site is a **static website** — every page is pre-built as an HTML file during deploy, then served from a CDN. There's no database and no server running at request time. The client can edit blog articles through a web-based admin panel, and each edit triggers a full rebuild and redeploy of the site.

That architecture makes the site fast, cheap to host, and resistant to most categories of security issue. The tradeoff is that any content change takes a minute or two to go live (the time a full build takes), and anything genuinely dynamic (user accounts, real-time updates, etc.) would require adding a separate service.

---

## Tech stack

| Piece | What it is | Why it's here |
|---|---|---|
| **Gatsby 4** | React-based static site generator | Builds the site from source files into static HTML/CSS/JS |
| **React 17** | UI library | Powers the page components |
| **Netlify** | Hosting platform | Builds the site on every commit, serves it over a CDN, handles DNS, forms, and CMS authentication |
| **Netlify CMS** | Git-based content management | Gives the client a web UI to edit articles without touching code |
| **Netlify Identity** | Authentication service | Controls who can log in to the CMS |
| **Git Gateway** | CMS-to-Git bridge | Lets the CMS commit article changes to GitHub without needing editors to have GitHub accounts |
| **Netlify Forms** | Form submission handling | Captures contact form submissions and emails them to the client |
| **GitHub** | Code hosting | Stores the source code; pushes trigger Netlify builds |

A few of these have newer equivalents that this site will eventually move to — see [deferred modernisation work](#deferred-modernisation-work).

---

## How the pieces fit together

It helps to understand the flow of a typical content change. When the client logs in at `/admin` on the live site and publishes a new article:

1. The CMS authenticates them through Netlify Identity (they log in with an email/password, or via GitHub if they've set that up).
2. Git Gateway, using Netlify Identity's auth token, commits their change to the GitHub repository on their behalf — they never touch GitHub directly.
3. GitHub sends a webhook to Netlify saying "a new commit arrived on `main`".
4. Netlify kicks off a build: it pulls the latest code, runs `npm run build`, and produces a fresh `public/` directory of static files.
5. Netlify publishes the new `public/` contents to its CDN, replacing the previous version.
6. A minute or two after the client clicked "Publish", the new article is live.

Developer changes (code, styling, new features) follow the same flow — you just push commits directly to GitHub rather than going through the CMS.

---

## Running the site locally

### Prerequisites

- **Node.js 20** (or 18 — anything from 18 upwards is fine). The repository includes an `.nvmrc` file specifying the version; if you use [nvm](https://github.com/nvm-sh/nvm), `nvm use` in the project directory will switch to the right version automatically.
- **Git**
- A code editor (VS Code recommended, but anything works)

### First-time setup

```bash
git clone https://github.com/<your-username>/brixworth-osteo.git
cd brixworth-osteo
npm install
```

The install step takes a couple of minutes on first run.

### Start the dev server

```bash
npm run develop
```

This starts Gatsby's development server. You'll see it come up at [http://localhost:8000](http://localhost:8000). Any changes you make to source files are reflected in the browser automatically — usually within a second or two.

Gatsby also exposes a GraphQL explorer at [http://localhost:8000/___graphql](http://localhost:8000/___graphql) if you want to poke around the data layer while working on a component.

### Other useful commands

| Command | What it does |
|---|---|
| `npm run develop` | Start the dev server (with hot reload) |
| `npm run build` | Build the production site into `public/` |
| `npm run serve` | Serve the built site locally on port 9000 (useful for checking production builds) |
| `npm run clean` | Delete Gatsby's cache and `public/` directory. First thing to try when builds misbehave. |

---

## Project structure

```
brixworth-osteo/
├── articles/                 # Blog post Markdown files (managed by the CMS)
├── src/
│   ├── components/           # Reusable React components (header, footer, etc.)
│   ├── pages/                # Page components — each file here becomes a route
│   └── templates/            # Page templates used for generated pages (e.g. article template)
├── static/
│   ├── admin/
│   │   ├── config.yml        # CMS configuration — defines content types and fields
│   │   └── index.html        # CMS admin interface entry point
│   └── assets/               # Images uploaded via the CMS land here
├── gatsby-config.js          # Gatsby plugin configuration and site metadata
├── gatsby-node.js            # Build-time logic (e.g. generating article pages from Markdown)
├── package.json              # Dependencies and npm scripts
└── .nvmrc                    # Pins the Node.js version
```

A few things worth noting:

- Everything in `static/` is copied verbatim into the final site — no processing, no bundling. That's why the CMS admin lives there.
- Files in `src/pages/` automatically become routes. `src/pages/about.js` becomes `/about/`.
- Article pages are generated at build time in `gatsby-node.js` — for each file in `articles/`, a page is created using the template in `src/templates/`.
- **`static/admin/config.yml` references the repository path in its `backend` section.** If the repository is ever renamed or moved to a different GitHub account, this file needs updating to match. Search for `repo:` in that file.

---

## Content editing (for the client)

The client edits articles through the CMS at `https://<the-site>/admin`. They log in with their Netlify Identity credentials (email/password or GitHub), then get a visual editor for creating and editing articles.

You shouldn't need to touch article content yourself, but it's worth understanding what the CMS does:

- Each article is a Markdown file in the `articles/` directory of the repo, with frontmatter (metadata) at the top — title, date, featured image, etc.
- When the client saves an article, Git Gateway commits the change to `main` on their behalf.
- Images uploaded through the CMS go into `static/assets/` and get committed alongside.
- Every save or publish triggers a new build and deploy.

The CMS configuration lives at `static/admin/config.yml`. If the client needs new fields (e.g. an "author" field on articles), or a new content type (e.g. "testimonials"), you edit this file. The [Netlify CMS docs on widgets](https://www.netlifycms.org/docs/widgets/) are the reference for what's available.

---

## Build and deploy

Netlify builds the site automatically on every push to `main`. You don't need to do anything manually — commit, push, and watch the Deploys tab in Netlify.

### Build settings (for reference)

These are configured in the Netlify dashboard under **Site configuration → Build & deploy**:

- **Build command:** `npm run build`
- **Publish directory:** `public`
- **Base directory:** `/`
- **Node version:** set via `.nvmrc` in the repo

### If a build fails

- Check the deploy log in Netlify — it's usually obvious what went wrong.
- Most common cause: a dependency issue. `npm run clean` locally, then `rm -rf node_modules package-lock.json && npm install` to regenerate things.
- If the failure is in a fresh Netlify build but works locally, suspect a Node version mismatch first. Confirm `.nvmrc` matches what you're building with locally.

### Rolling back

Netlify keeps the last 30 days of deploys on the Starter plan. If a deploy breaks the site, go to **Deploys**, find a previous working deploy, and click **Publish deploy**. The live site reverts immediately while you figure out what went wrong.

---

## Contact form submissions

The site has a contact form that uses Netlify Forms. Submissions:

1. Arrive in the Netlify dashboard under the site's **Forms** tab.
2. Trigger an email notification to the client's email address (configured under **Forms → Form notifications**).

If the client wants to change the notification email, or add CC recipients, that's done in the Netlify dashboard — no code change needed.

The form itself is a standard HTML form with a `data-netlify="true"` attribute. Netlify auto-detects the form on build. If you add new fields, they'll start appearing in submissions automatically.

Spam is handled by Netlify's built-in honeypot and Akismet filter. Submissions flagged as spam still appear in the dashboard but under a separate tab.

---

## Managing CMS users

CMS users are managed through Netlify Identity. In the Netlify site dashboard:

- **Integrations → Identity → Users** shows the list of people who can log in to the CMS.
- To add a user: click **Invite users**, enter their email. They receive an invitation email with a link to set a password.
- To remove a user: click their name, then **Delete user**.

There are currently two users: the client and their business assistant.

### GitHub as an external provider

The CMS login screen offers a "Log in with GitHub" option in addition to email/password. This is configured under **Integrations → Identity → Settings and usage → External providers**, and backed by a GitHub OAuth app.

If the "Log in with GitHub" option ever stops working, check:

1. The OAuth app still exists and its Client ID/Secret match what Netlify has.
2. The OAuth app's callback URL is still `https://api.netlify.com/auth/done`.

---

## Common tasks

### Add a new page

1. Create a new file in `src/pages/`, e.g. `src/pages/services.js`.
2. Export a default React component — this is the page.
3. Commit and push. The new page is live at `/services/` after the build completes.

### Add a new component

Components live in `src/components/`. There's no magic — create a file, export a component, import it where needed.

### Update dependencies

Be cautious here. Given the dated stack, naive `npm update` runs can pull in versions that aren't compatible with Gatsby 4. Safer approach:

1. Read the Gatsby 4 migration notes for the dependency you want to update.
2. Update one package at a time.
3. Run `npm run build` locally after each update. If it fails, roll back and investigate before continuing.
4. Dependabot is useful for visibility but don't merge its PRs blindly.

### Check what's causing a slow build

Gatsby builds include timing info in the output. The usual culprits are `gatsby-plugin-sharp` (image processing) on content-heavy sites, or a large number of articles causing long GraphQL query times. Neither is a problem on this site at its current size.

---

## Deferred modernisation work

The stack is functional but includes several pieces due for upgrade. None of these are urgent but they'll become more pressing over time. Listed roughly in order of ease / value.

### Netlify CMS → Decap CMS

**Effort:** small (a few hours). **Urgency:** low, but growing.

Netlify CMS was renamed to Decap CMS in 2023. The code is the same; maintenance moved to a new team after Netlify stopped actively developing it. The migration is essentially: swap `netlify-cms-app` for `decap-cms-app` in `package.json`, swap `gatsby-plugin-netlify-cms` for its Decap equivalent (or use a manual include), and update the `<script>` reference in `static/admin/index.html`. Config stays the same.

[Decap CMS docs](https://decapcms.org/) and [Decap's migration guidance](https://decapcms.org/docs/faq/#i-had-netlify-cms-installed-what-should-i-do) are the references.

### React 17 → 18 and `react-helmet` → `react-helmet-async`

**Effort:** medium. **Urgency:** low.

React 18 is the current stable. The upgrade is usually straightforward but touches every component, and `react-helmet` (used for managing `<head>` tags) isn't compatible with React 18's concurrent rendering — it needs replacing with `react-helmet-async`. Do this as a single coordinated change, not two separate ones.

### Gatsby 4 → 5

**Effort:** medium to large. **Urgency:** medium over the next year or two.

Gatsby 5 has breaking changes in configuration and APIs. The [Gatsby 5 migration guide](https://www.gatsbyjs.com/docs/reference/release-notes/migrating-from-v4-to-v5/) walks through them. Most of the work is mechanical but expect some debugging. Best done after the React 18 upgrade, as Gatsby 5 prefers it.

### Netlify Identity migration

**Effort:** medium. **Urgency:** monitor — could become urgent suddenly.

Netlify Identity is deprecated (Feb 2025). Existing instances still work but no new ones can be created, and Netlify won't fix bugs or provide support. They've said the service will continue, but that's a softer commitment than "we'll keep developing it."

Two realistic migration paths when the time comes:

1. **Switch the CMS backend from `git-gateway` to `github`.** Simpler architecturally, but every CMS editor would need a GitHub account with push access to the repository. Not ideal for non-technical editors.
2. **Use [DecapBridge](https://decapbridge.com/).** A community-built service designed for exactly this situation. Keeps the no-GitHub-account-needed editor experience.

Don't do this pre-emptively — Identity still works fine. But watch the Netlify changelog and have a migration plan sketched out so you can act quickly if Identity's timeline changes.

---

## Troubleshooting

### "The site won't build on Netlify but works locally"

Usually a Node version mismatch. Confirm `.nvmrc` is committed and contains a current version. Also try a fresh build by deleting the Netlify build cache: **Site configuration → Build & deploy → Post processing → Clear cache and retry deploy**.

### "The CMS won't load at /admin"

Open the browser console. Common causes:

- **404 on admin scripts:** `static/admin/` isn't being copied to the build output — check nothing has changed in the Gatsby config that would affect static file handling.
- **Auth redirect loop:** Netlify Identity is misconfigured or disabled. Check **Integrations → Identity** is still enabled.
- **"Git Gateway error":** Git Gateway's link to GitHub is broken. In Netlify, go to **Integrations → Identity → Services → Git Gateway → Edit settings** and reauthorize.

### "A client published an article but it isn't showing up"

Check **Deploys** in Netlify. A publish from the CMS should trigger a build within seconds. If no build ran, Git Gateway may be failing silently — check the CMS browser console at the moment of publish, and verify Git Gateway is still authorized.

### "Contact form submissions aren't arriving"

Check the **Forms** tab in Netlify — are submissions being captured but not sent, or not captured at all?

- **Not captured:** the form HTML may have lost its `data-netlify` attribute during an edit, or Netlify's form parser may not be detecting the form. Rebuilding the site usually fixes detection issues.
- **Captured but not emailed:** check **Forms → Notifications**. The notification email address may have changed or the notification may have been disabled.

---

## Useful links

- [Gatsby 4 documentation](https://v4.gatsbyjs.com/docs/) — note: the main Gatsby docs site shows v5 by default
- [Netlify CMS docs](https://www.netlifycms.org/docs/)
- [Decap CMS docs](https://decapcms.org/docs/) — the forward-looking successor
- [Netlify docs](https://docs.netlify.com/)
- [Netlify Identity docs](https://docs.netlify.com/manage/security/secure-access-to-sites/identity/overview/)
- [Netlify Forms docs](https://docs.netlify.com/forms/setup/)
- [React 17 docs](https://17.reactjs.org/)
- [MDN (general web reference)](https://developer.mozilla.org/)

---

*This README reflects the state of the site at the point of handover. Update it as the site evolves — future-you (or your successor) will thank you.*
