# gitultra-metrics-suite

A self-hosted engineering metrics pipeline: import commit and PR/MR stats
from your GitHub, GitLab, Bitbucket, or Azure DevOps org into Postgres,
and generate PDF reports (team/author activity, PR review health, cycle
time, DORA metrics, repo trends) from it.

**Free to run.** A license key unlocks the Team and Enterprise plans in
the same install: no second repo, no reinstall. Add the key to `.env`,
run `docker compose up -d`, done.

This repo is just the `docker-compose.yml` that runs it. The application
images are published pre-built at
[ghcr.io/gitultrahq](https://github.com/GitUltraHQ?tab=packages).

**Full docs: [docs.gitultra.com](https://docs.gitultra.com/)** ·
New to the paid plans? See [QUICKSTART.md](QUICKSTART.md).

## Quickstart

```
cp .env.example .env
# edit .env: set a POSTGRES_PASSWORD, and the token(s) for whichever
# provider(s) you're importing from (GitHub/GitLab/Bitbucket/Azure DevOps).
# On a paid plan, also fill in the "Paid plans" section.

sudo mkdir -p /var/lib/eng-metrics-suite
sudo chown "$(id -u):$(id -g)" /var/lib/eng-metrics-suite
mkdir -p reports

docker compose up -d
```

See [Getting Started](https://docs.gitultra.com/getting-started/)
for why those two `mkdir`s matter, then
[Discovering Repos](https://docs.gitultra.com/discovering-repos/)
to actually queue something up. `docker-compose.yml` also declares a
`git_mirrors` volume for `git-processor`'s persistent per-repo clone
cache — Docker manages that one itself, no `mkdir`/`chown` needed.

## What's included

Free plan:

- **`git-processor`** and **`pr-processor`**: import commits and PRs/MRs.
- **`eng-reports`**: the PDF reports, including `allocation_report.py`
  for investment allocation.
- **`issue-processor`** (optional): Jira, for change failure rate/MTTR
  and/or investment allocation. See below.
- **`postgres`** and **`migrate`**: the database and its schema.

Paid plans (need `GITULTRA_LICENSE_KEY`; without one, each of these logs
one "no license key" line and idles, so they're safe to leave in the
stack):

- **`eng-api`**: REST/JSON API for dashboard integrations, on
  `ENG_API_PORT` (default 8000). See
  [eng-api](https://github.com/GitUltraHQ/eng-api)'s README for the
  endpoints, auth and config.
- **`gitultra-mcp`**: MCP server so an AI agent (Claude Desktop, Claude
  Code, etc.) can query your metrics conversationally, on
  `GITULTRA_MCP_PORT` (default 8100). See
  [gitultra-mcp](https://github.com/GitUltraHQ/gitultra-mcp)'s README.
- **Paid reports** (Executive, QBR, Onboarding), run from the same
  `eng-reports` container as the free ones. See [QUICKSTART.md](QUICKSTART.md).

**Not included in the compose file:** `rewrite-ratio`. It's a one-shot
diagnostic CLI, not a long-running service, so it doesn't belong in a
`docker compose up` stack -- run it directly per [QUICKSTART.md](QUICKSTART.md).

### Optional: Change Failure Rate / MTTR via Jira

`issue-processor` just sits idle (one "idle" log line, no restarts) if
nothing Jira-related is configured at all, so it's safe to leave running
even if you don't use this. To enable it: set the `JIRA_*` variables in `.env`, then drop a
`jira_project_map.yaml` (see
[issue-processor](https://github.com/GitUltraHQ/issue-processor)'s
`jira_project_map.example.yaml`) into `/var/lib/eng-metrics-suite/` and
run its seed script once:

```
docker compose run --rm --entrypoint python3 issue-processor discover_jira_projects.py \
    --config /var/lib/eng-metrics-suite/jira_project_map.yaml
```

This is a hand-maintained mapping, not auto-discovered — there's no Jira
API that can derive which project's Fix Versions apply to which repo.

### Optional: Investment Allocation Report via Jira

A separate `issue-processor` flow (independent of the CFR/MTTR one
above — enable one, the other, or both) that categorizes and tracks Jira
work items for `eng-reports`' second report script. Set
`JIRA_ALLOCATION_ISSUE_TYPES` (and optionally `JIRA_CATEGORY_FIELD`/
`JIRA_STORY_POINTS_FIELD`) in `.env`, then drop an
`allocation_projects.yaml` (see
[issue-processor](https://github.com/GitUltraHQ/issue-processor)'s
`allocation_projects.example.yaml` — just a list of project keys, no repo
mapping needed) into `/var/lib/eng-metrics-suite/` and run its seed script
once:

```
docker compose run --rm --entrypoint python3 issue-processor discover_allocation_projects.py \
    --config /var/lib/eng-metrics-suite/allocation_projects.yaml
```

Then generate the report:

```
docker compose run --rm eng-reports allocation_report.py --output /out/allocation.pdf
```

## Moving to a paid plan

1. Put your license key in `.env` as `GITULTRA_LICENSE_KEY`, and set
   `ENG_API_KEY` to a secret you choose (`openssl rand -hex 32`).
2. `docker compose up -d`

Your existing data stays where it is; nothing is re-imported. If the key
is invalid or expired, `eng-api` and `gitultra-mcp` stop with the reason
in `docker compose logs eng-api`.

## Requirements

- Docker + Docker Compose (v2)
- For the paid plans: a license key, from support@gitultra.com. No
  `docker login` or GitHub account needed for any plan: every image is
  public on GHCR, and the license key is what unlocks the paid features.

## License

See [LICENSE](LICENSE).
