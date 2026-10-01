# Storefront demo plugin

Skills for operating the storefront demo stack, synced into incident.io so the
agents investigating its alerts follow the same procedure a responder would.

The stack itself — the services, the Prometheus rules, the failure scenarios —
lives in a separate repository,
[incident-io/ai-demo-harness](https://github.com/incident-io/ai-demo-harness).
This repository holds only the agent-facing content. The alert thresholds and
cascade orderings the triage skill states are read from that repository's
`infrastructure/prometheus/rules.yml`, which remains the source of truth — if
the rules change there, the skill's alert table is what goes stale.

## Skills

| Skill | What it does |
|---|---|
| [`storefront-alert-triage`](skills/storefront-alert-triage/SKILL.md) | Triage a storefront alert down to the root cause of its cascade and call it noise or real, with the impact evidence behind the call. |

## Connections

What these skills call, and where each comes from per environment.

| Connection | Used for | incident.io | Coding agents |
|---|---|---|---|
| incident.io | `alert_show`, `alert_list` — reading the alert and what else fired around it | Built in | The incident.io MCP server |
| **production metrics** (Prometheus) | Is the alert's metric still breaching | Telemetry datasource `Prometheus`, reached over the harness connector tunnel | Same datasource via the incident.io MCP server's telemetry tools |
| **application logs** (Loki) | Whether real requests failed | Telemetry datasource `Loki`, same tunnel | As above |
| **order history** (PostgreSQL) | Whether the order rate dropped (a failed checkout never reaches the `orders` table, so there is no success ratio to read) | Telemetry datasource `postgres`, same tunnel | As above |

All three telemetry datasources reach a docker-compose stack running on a
laptop, through the harness `connector-proxy`. When the stack is down or pointed
at a different incident.io workspace, queries fail with a connection error
rather than returning empty — the triage skill treats those as "could not
check", never as "no impact".
