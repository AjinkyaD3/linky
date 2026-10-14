# CI/CD pipeline

Two repos, each with its own pipeline. Nothing reaches production without passing the same
end-to-end smoke suite that gates pull requests.

```
                         BACKEND (linky-backend-springboot)
 PR / push  ->  test .......... 82 unit + MockMvc tests (H2, Redis mocked)
            ->  integration ... boots the real jar on Postgres 16 + Redis 7, runs scripts/smoke-test.sh
                                (also proves Flyway migrations match the entities: ddl-auto=validate)
            ->  docker ........ build image, Trivy scan (fails on CRITICAL), publish to GHCR from main
            ->  codeql ........ static security analysis (also weekly)
 main       ->  deploy ........ [approval gate] -> Render deploy hook -> wait for /actuator/info
                                to report the new commit -> smoke-test production

                         FRONTEND (linky-frontend-nextjs)
 PR / push  ->  build ......... eslint, tsc, next build, then serve the build and run scripts/smoke.sh
            ->  audit ......... npm audit (fails on high/critical runtime vulnerabilities)
            ->  codeql ........ static security analysis (also weekly)
 deploy     ->  Vercel Git integration deploys (preview per PR, production on main);
                post-deploy.yml smoke-tests every successful deployment URL
```

## What each guard catches

| Guard | Catches |
|---|---|
| Real-services integration job | Postgres-only SQL problems, Redis stream/Lua/cache bugs, schema drift between Flyway and entities (none of which H2 + mocks can see) |
| `scripts/smoke-test.sh` | Auth, redirects, one-time-link race (10 parallel clicks), password links, input validation, and the full click pipeline end to end |
| Trivy | Critical CVEs in the base image and packaged dependencies |
| `npm audit` | Known vulnerable runtime dependencies (it caught a critical Next.js advisory, fixed by upgrading 16.3.5 -> 16.4.0) |
| CodeQL | Injection, unsafe deserialization, auth mistakes, and similar patterns |
| Deploy verification | "Deployed" is not "live": the pipeline polls `/actuator/info` until the new commit serves traffic, then smoke-tests production |
| Dependabot | Weekly PRs for Maven, npm, Docker and GitHub Actions updates |

## One-time setup (cannot be done from code)

**Backend repo** (Settings > Secrets and variables > Actions)

| Name | Kind | Value |
|---|---|---|
| `RENDER_DEPLOY_HOOK_URL` | secret | Render service > Settings > Deploy Hook |
| `BACKEND_URL` | variable | e.g. `https://your-app.onrender.com` |
| `BACKEND_HEALTH_URL` | variable | `<BACKEND_URL>/actuator/health` (for the keep-alive cron) |

Also:
1. Render: set **Auto-Deploy to Off** so only the pipeline deploys.
2. GitHub > Settings > Environments > create **production**, add yourself as a required reviewer.
3. GitHub > Settings > Branches: protect `main`, require the `test`, `integration` and `docker` checks.
4. Render environment variables: `TRUSTED_PROXY_HOPS=1`, `IP_HASH_SALT`, `METRICS_TOKEN`, and
   `REDIS_KEY_PREFIX=prod:` if any other environment shares the same Redis.

**Frontend repo**
1. Connect the repo to Vercel (Git integration). Production branch = `main`.
2. Protect `main`, require the `build` and `audit` checks.

## Rollback

- Backend: Actions > Deploy > Run workflow > enter the SHA of the last good release. The pipeline
  redeploys that commit and smoke-tests it. Images for every release are in GHCR (`:<sha>`).
- Frontend: Vercel > Deployments > previous deployment > Promote to Production (instant).

## What was and was not verified

Verified locally: both smoke scripts pass against the running apps; the integration job's steps
were simulated end to end on an empty Postgres database; all workflow files pass
`action-validator` schema validation.

Not verified (needs GitHub, Render and Vercel accounts): the workflows have not run on GitHub yet,
so expect to fix small environment details on the first run. The Render `ref=` deploy-hook
parameter used for rollback is from Render's documentation and untested here.
