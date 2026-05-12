# sumologic-agent — RepoDocs
_Generated on 2026-05-11_

## Summary

### Overview
This repo is an Ansible role that installs and configures the Sumo Logic Installed Collector agent on RHEL-based Linux hosts. It downloads a pod-specific Sumo Logic RPM, lays down `user.properties` and `sources.json` from Jinja2 templates, and starts the `collector` service so the host ships local OS logs (and any caller-supplied logs) into Sumo Logic. It is org infrastructure tooling — a reusable role consumed by Ansible playbooks that provision Linux servers in this org.

### Tech Stack
| Category | Technology | Version |
|----------|-----------|---------|
| Language | YAML (Ansible) / Jinja2 | — |
| Framework | Ansible role | min_ansible_version 2.1 (per `meta/main.yml`) |
| Database | _Not applicable._ | — |
| Build Tool | _Not applicable (Ansible role, no build step)._ | — |
| CI/CD | GitHub Actions (`.github/workflows/secrets-scan.yml`) | — |
| Cloud/Infra | RHEL/EL 5–6 target platform; Sumo Logic Installed Collector | EL 5, 6 (declared in `meta/main.yml`) |

### Consumers
| Consumer | Type | How They Use It |
|----------|------|----------------|
| Ansible playbooks targeting RHEL Linux hosts | Internal (org infra) | Include this role (e.g. `roles: [ chrisdodds.sumologic-agent ]`) to install the Sumo Logic collector on provisioned servers |
| GitHub Actions (`edplato/trufflehog-actions-scan`) | CI | Scheduled secrets scan weekdays at 14:00 UTC on this repo |
| Slack (`rtCamp/action-slack-notify`) | CI notification | Posts to `#github-token-scan` channel when the secrets scan fails |

### Dependencies on Org Repos
_None — this role declares `dependencies: []` in `meta/main.yml` and has no imports/includes of other org repos._

### External Integrations
| Service | Purpose | Integration Type |
|---------|---------|-----------------|
| Sumo Logic (Installed Collector) | Log shipping agent; downloads RPM from `sumologic_rpm_url`, authenticates with `accessid` / `accesskey` in `user.properties` | SDK (RPM/agent) |
| Slack | CI failure notification via `rtCamp/action-slack-notify` webhook | Webhook (outbound) |

### Async & Scheduled Work
| Channel / Job | Type | Direction | Purpose |
|--------------|------|-----------|---------|
| `secret-scan` GitHub Actions workflow | Cron `0 14 * * 1-5` | N/A (job) | Weekday TruffleHog secrets scan of the repo |
| Sumo Logic `collector` service | systemd service started by role | Produces | Tails `/var/log/secure*`, `/var/log/messages*`, `/var/log/yum.log` (plus any `sumologic_tracked_logs` entries) and forwards to Sumo Logic |

### Upgrade Alerts
| Dependency | Current Version | Issue | Severity |
|-----------|----------------|-------|----------|
| Ansible `min_ansible_version` | 2.1 | Ansible 2.1 is EOL (2.9 is last of the 2.x line and itself EOL); declared minimum is decades behind current Ansible | Severe |
| Target platform `EL` versions | 5, 6 | RHEL/CentOS 5 and 6 are EOL (CentOS 6 ended 2020-11-30); role advertises support only for EOL OS versions | Severe |
| `actions/checkout@master` (workflow) | `@master` (floating, ≤ v2 era) | Uses deprecated `@master` ref and pre-v3 checkout; older Node runtime, no pinned SHA | Severe |
| `edplato/trufflehog-actions-scan@master` | `@master` | Third-party action pinned to floating `master`; project is unmaintained / archived community fork | Severe |

## API Reference

This repo is an Ansible role, not a service — its "API" is its variable contract and the tasks it executes.

**Required role variables** (caller must supply):
- `sumologic_rpm_url` (string) — URL to the pod-specific Sumo Logic Installed Collector RPM (e.g. `https://collectors.us2.sumologic.com/rest/download/rpm/64`).
- `sumologic_access_id` (string) — Sumo Logic access ID; written to `accessid` in `/opt/SumoCollector/config/user.properties`.
- `sumologic_access_key` (string) — Sumo Logic access key; written to `accesskey` in `/opt/SumoCollector/config/user.properties`.

**Optional role variables** (with defaults from `defaults/main.yml`):
- `sumologic_ephemeral_agent` (bool, default `true`) — written as `ephemeral` in `user.properties`; controls automatic cleanup of unused collectors in the Sumo console.
- `env_timezone` (string, default `'Etc/UTC'`) — TZ-format timezone applied to every `sources.json` entry via `forceTimeZone: true`.
- `sumologic_tracked_logs` (list, default `[]`) — additional local-file log sources. Each entry is an object with fields:
  - `name` (string)
  - `description` (string)
  - `category` (string)
  - `path` (string) — `pathExpression` for the source
  - `filters` (string) — raw JSON-block contents inserted into the `filters` array

**Tasks executed** (in order, from `tasks/main.yml`):
1. `Download SumoCollector` — `get_url` of `sumologic_rpm_url` → `/tmp/sumo_collector.rpm`.
2. `Install SumoCollector redhat` — `yum` install of the downloaded RPM.
3. `Put config file in place` — template `templates/user.properties.j2` → `/opt/SumoCollector/config/user.properties`, owner `root`, group `sumologic_collector`.
4. `Put sources file in place` — template `templates/sources.json.j2` → `/opt/SumoCollector/config/sources.json`, owner `root`, group `sumologic_collector`.
5. `Start service` — `service: name=collector state=started enabled=yes`.

**Default log sources** (always emitted into `sources.json`, sourceType `LocalFile`, category `OS/Linux`):
- `Linux Secure Log` — `/var/log/secure*`
- `Linux Message Log` — `/var/log/messages*`
- `Linux Yum Log` — `/var/log/yum.log`

## Architecture

```
                +------------------------------------------+
                | Ansible Control Node                     |
                |   playbook includes sumologic-agent role |
                +-------------------+----------------------+
                                    | SSH / become
                                    v
+------------------------------------------------------------------+
| Target RHEL host                                                 |
|                                                                  |
|  /tmp/sumo_collector.rpm  <-- get_url  https://collectors.<pod>. |
|                                          sumologic.com/.../rpm/64|
|                                                                  |
|  yum install -> /opt/SumoCollector/                              |
|     config/user.properties  (templated; accessid/accesskey)      |
|     config/sources.json     (templated; OS logs + custom logs)   |
|                                                                  |
|  service: collector (started, enabled)                           |
|     tails /var/log/secure*, /var/log/messages*, /var/log/yum.log |
|     plus any sumologic_tracked_logs entries                      |
|                                                                  |
+----------------------+-------------------------------------------+
                       | HTTPS (Sumo Logic Installed Collector)
                       v
              +-------------------------+
              | Sumo Logic (cloud, pod- |
              | specific endpoint)      |
              +-------------------------+
```

**Key components**
- `tasks/main.yml` — the role entrypoint; download → yum install → templates → service start.
- `templates/user.properties.j2` — Sumo Logic collector credentials and runtime knobs (`targetCPU = 20`, `syncSources` pointing at `sources.json`).
- `templates/sources.json.j2` — declarative source list; three hard-coded OS sources plus a `{% for log in sumologic_tracked_logs %}` loop for caller-supplied sources.
- `defaults/main.yml` — defaults for `env_timezone`, `sumologic_tracked_logs`, `sumologic_ephemeral_agent`.
- `meta/main.yml` — Ansible Galaxy metadata; author `Chris Dodds`, MIT license, EL 5/6, `dependencies: []`.
- `handlers/main.yml`, `vars/main.yml` — present but empty (scaffolding only).
- `tests/test.yml` + `tests/inventory` — minimal localhost smoke playbook applying the role.

**Data flow**
Caller playbook provides Sumo Logic credentials and (optionally) extra log definitions as Ansible vars → role renders templates and drops them into `/opt/SumoCollector/config/` → the Sumo Logic collector daemon reads the configs on start, registers with Sumo Logic using the access ID/key, and continuously tails the configured local log files, shipping them to the pod referenced by `sumologic_rpm_url`.

**CI/CD tooling**
GitHub Actions (detected via `.github/workflows/secrets-scan.yml`). The single workflow `scan for secrets` runs on a cron `0 14 * * 1-5` (weekdays 14:00 UTC): checkout → `edplato/trufflehog-actions-scan@master` with `--regex --entropy=False --max_depth=1` → on failure, post to Slack `#github-token-scan` via `rtCamp/action-slack-notify@v2.0.2`. There is no build/test/deploy pipeline — the role is consumed by other Ansible playbooks, not packaged here.

**Test architecture**
`tests/test.yml` is a one-liner playbook that applies the role against `localhost`; `tests/inventory` lists `localhost`. There is no Molecule scenario, no assertions, and no CI hook that runs the tests.

**Data model / database schema**
_Not applicable — no datastore. The role emits two configuration files (`user.properties`, `sources.json`) consumed by the Sumo Logic collector._

**Auth & trust boundaries**
Inbound: none — this is an Ansible role applied via SSH/become by the Ansible control node; trust is whatever the control node already has on the target host. Outbound auth to Sumo Logic uses an access-ID / access-key pair written into `user.properties` (and supplied via the `sumologic_access_id` / `sumologic_access_key` role vars — callers are expected to source these from a vault). The collector then authenticates to Sumo Logic over HTTPS using those credentials. Authorization model: _Not applicable._

**Data ownership**
_Not applicable — no datastore is owned or accessed. The role does write configuration files into `/opt/SumoCollector/config/` on each target host, owned by `root:sumologic_collector`._

**Deployment topology**
_Deployment topology not in this repo._ This is a role library; the playbooks that consume it (and decide which hosts/environments it lands on) live elsewhere. The only inferable runtime contract is "RHEL-based Linux host" (EL 5/6 per `meta/main.yml`) with the Sumo Logic collector running as the `collector` systemd service.

## Repo Activity
Derived from git history; current HEAD is `eb422bf`.
- **Created**: 2017-08-13 (`eb149c3` initial commit).
- **Last meaningful change**: 2020-07-07 (`46bb34e` "Create secrets-scan.yml" — added the GitHub Actions secrets-scan workflow). The 2020-07-08 follow-up (`eb422bf`) was a tweak to that same file. The last functional change to the role itself was 2017-09-27 (`49a012f` "Updates to readme, repo download, and adding syncSources to user.properties").
- **Activity level**: 0 commits in the last 90 days (most recent commit is from 2020-07-08, well outside any 90-day window relative to 2026-05-12).
- **Hot spots**: Only six commits exist in the entire history, so 6-month churn is zero. Lifetime-touched files (for context): `README.md` (3 commits), `templates/user.properties.j2` (2 commits), `meta/main.yml` (2 commits).
- **Recent major changes**: _No major changes in the last 6 months._
