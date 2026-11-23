# Deployment

Everything that is not needed to read or run the project locally. The
[README](../README.md) covers install and the design decisions.

---

## Docker

**Development and production use different images, on purpose.**

The dev container (`.devcontainer/`) runs a pre-built image from Microsoft — the
recommended approach for development: it ships the toolchain, keeps sources mounted, and
rebuilds on change. `Dockerfile` and `Dockerfile.bun` do the opposite: they produce a
minimal, immutable production artifact — multi-stage build, standalone output, non-root
user, no dev dependencies. They are optimized for shipping, not for coding, and are
unsuitable for day-to-day development.

Keeping the two apart is what lets the production image stay small and reproducible while
the development environment stays comfortable.

### Build and run

> If you modify `.env`, re-run the `export` line to refresh the variables in your shell.

```bash
export $(grep -v '^#' .env | xargs)

docker build --progress=plain $(grep -v '^#' .env | xargs -I{} echo --build-arg {}) -t nextjs-production-starter -f Dockerfile .

# or, using Bun
docker build $(grep -v '^#' .env | xargs -I{} echo --build-arg {}) -t nextjs-production-starter -f Dockerfile.bun .

docker run -p 3000:3000 nextjs-production-starter
```

### Cleanup

```bash
# find the container id
docker ps -a --filter "ancestor=nextjs-production-starter"

# stop and delete the container, then the image
docker container rm <container_id>
docker image rm nextjs-production-starter
```

### Who this single-image architecture (Next.js + Hono) suits

A smart monolith: one image to build, deploy and monitor. It fits solo developers and
small teams, proofs of concept, and edge-ready projects that still need cookies, sessions
or server-side auth. Hono handles a lightweight API, Next handles the frontend, and there
is one service to orchestrate instead of two.

Once built, the image deploys to any VPS, VM or container service.

---

## CI/CD

Two workflows, with **different trigger modes** — the distinction matters.

### `quality.yml`

Runs on **every pull request and every push to `main`**: install, lint, typecheck, tests,
build. This is the workflow that gates changes, and it needs no configuration.

It is triggered by `pull_request` — never `pull_request_target`, which would expose
repository secrets and a write token to code coming from a fork. See the README for why.

### `docker-publish.yml`

Runs on **every push to `main`** (and on demand from the Actions tab). It builds the image
from `Dockerfile.bun` and publishes it to GitHub Container Registry with two tags:

```
ghcr.io/<owner>/<repo>:latest
ghcr.io/<owner>/<repo>:<sha>
```

**Nothing to configure.** `ghcr.io` authenticates with `GITHUB_TOKEN`, which GitHub
Actions mints at the start of every job — ephemeral, scoped to this repository, never
stored in your secrets. That is why this workflow can run on push: no credential can be
missing.

Publishing requires elevating that token, declared **in the workflow file itself**, at job
level:

```yaml
permissions:
  contents: read # re-declared: `permissions` replaces the default set
  packages: write # lets this job — and only this job — publish
```

The repository setting stays on `read`; no other workflow is affected. The published image
appears under the repository's **Packages** tab after the first successful run, and the
registry matches the `[registry]` block in `fly.toml`.

This workflow **does not deploy**: the `flyctl deploy` step is commented out and the demo
is served by Vercel. It publishes an image, nothing more.

### Repository variables

No secret is needed. Only these build-time variables, both optional — the build falls back
to empty values.

**Settings → Secrets and variables → Actions → Variables → Repository variables**

- `NEXT_PUBLIC_APP_URL` — the public URL of your app
- `NEXT_PUBLIC_GOOGLE_ANALYTICS_ID` — your Google Analytics id (`G-XXXXXXX`)

Add `FLY_API_TOKEN` as a repository **secret** only if you uncomment the Fly.io deploy
step.

---

## Vercel

Production deploys run through GitHub Actions (`deploy.yml`) — **not** through Vercel's
own Git integration. That integration is intentionally disconnected: this repo receives
commits from agents, and `deploy.yml` is meant to stay the only path to production.

### How it works

- Trigger: every push to `main` — after `quality.yml` passes on that same commit, called
  via `workflow_call` — or a manual run from the Actions tab. Both are restricted to
  `main` by the `production` environment's branch rule.
- The job that holds `VERCEL_TOKEN` runs no code from the repository: no install, no
  local build, no `vercel pull`. The build happens on Vercel's side; the job only runs
  `vercel deploy --prod --yes`, and the token is scoped to that one step (absent while
  the Vercel CLI itself is installed).
- `VERCEL_TOKEN` is a secret on the `production` **environment**, not a repository
  secret — project-scoped and time-limited, so only commits on `main` can use it.
- Deploys queue instead of cancelling (`concurrency: group: vercel-prod,
  cancel-in-progress: false`): a runner build that got cancelled would keep running on
  Vercel's side and could still grab the production alias after a newer deploy.
- No `pull_request`, `pull_request_target` or `workflow_run` trigger — this is a public
  repo, so production secrets never run against code from a fork.

### Setup

1. Create a Vercel project (`vercel.com` → New Project, import the repo once to obtain
   `VERCEL_ORG_ID` / `VERCEL_PROJECT_ID` — already set in `deploy.yml`'s `env:` for this
   repo).
2. In that Vercel project, disconnect the Git integration (Settings → Git): this repo
   deploys only through `deploy.yml`.
3. Create a GitHub **environment** named `production`, restrict it to the `main` branch,
   and add `VERCEL_TOKEN` there as an environment secret (not a repository secret).
4. Push to `main`: `quality.yml` runs first, then `deploy.yml` deploys.

Owner: agent-os. Remove this workflow only once another deployment path replaces it.

## Fly.io

```bash
fly auth login
fly apps create nextjs-production-starter   # non-blocking error if it already exists
fly apps list
```

Pushing to `main` builds and publishes the image to `ghcr.io` automatically, and
`docker-publish.yml` can also be triggered by hand from the Actions tab. Deploying that
image to Fly.io is a **separate step**: the `flyctl deploy` block in the workflow is
commented out, so nothing reaches Fly.io on its own. Uncomment it — and set
`FLY_API_TOKEN` — to have the workflow deploy as well.

## Managed platform or paid VM

A managed platform gives you a public domain, preview URLs, SSL, CDN, rollbacks and logs
without operating any of it, and stays free for personal projects, portfolios, MVPs and
side projects.

A paid VM means running domain, SSL, monitoring, backups, security and scaling yourself,
and paying monthly even with no traffic. It earns its place when you have a specific need:
custom infrastructure, a heavy backend, root access.

For most projects, start managed. Move to a VM when you hit something the platform cannot
do, or when you want full control of the infrastructure.
