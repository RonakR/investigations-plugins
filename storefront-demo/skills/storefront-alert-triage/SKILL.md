---
name: storefront-alert-triage
description: >
  Assess any alert reaching this environment and decide whether it is noise or a
  real incident, with the evidence behind the call. This owns alert triage for
  the storefront stack — gateway, checkout-service, catalog-service,
  payments-service, postgres — whose alert rules are tiered so the root-cause
  alert fires first and the symptoms trail it, which means the alert you are
  handed is usually not the cause. Use whenever you are working with an alert in
  any way, including judging whether one is worth waking someone for, and
  including an unfamiliar alert you are checking for storefront relevance. It
  recognises CheckoutServerErrors, GatewayHighErrorRate, GatewayHighLatency,
  CheckoutPaymentsLatency, PaylinityRequestLatency, PostgresBlockedQueries,
  PostgresLongRunningTransaction, CatalogDBPoolSaturation and
  CheckoutDBPoolSaturation, and classifies any alert outside that set as noise.
metadata:
  incident-io-plugin-version: "0.21.0"
---

# Storefront alert triage

## The model, and why it matters

The storefront is five services on one request path:

```
gateway ──▶ catalog-service ──▶ postgres
   │                               ▲
   └──────▶ checkout-service ──────┘
                  │
                  └──▶ payments-service ──▶ paylinity-provider (third party)
```

Every alert on this stack is a Prometheus rule, and **the rules are deliberately
tiered by their `for:` duration so the root-cause alert fires first and the
symptom alerts trail it by seconds**. `GatewayHighErrorRate` waits 90s precisely
so it never leads; `PaylinityRequestLatency` waits 10s precisely so it does.

The consequence is the whole reason this skill exists: **the alert you are handed
is usually a symptom, and the alert that fired *before* it names the cause.** An
answer that treats the alert in hand as the problem will be wrong about two
failures in three.

## Scope

This covers reading a storefront alert and calling it noise or real. It does not
cover fixing anything, changing the alert rules, or operating the demo harness
itself — the harness README owns those.

## The alert set

Nine rules, all from the **Prometheus Alertmanager alerts** source. Thresholds
and `for:` values here are read from `infrastructure/prometheus/rules.yml` in the
`ai-demo-harness` repository — that file is the source of truth, so re-read it if
a value here looks wrong rather than trusting this table.

| Alert | Service | Fires on | `for:` | Role |
|---|---|---|---|---|
| `PaylinityRequestLatency` | payments-service | p95 to the third-party provider > 3s | 10s | **Lead** |
| `PostgresBlockedQueries` | postgres | any query blocked on a lock | 10s | **Lead** |
| `PostgresLongRunningTransaction` | postgres | oldest txn open > 60s | 10s | Corroborates the lock |
| `CheckoutServerErrors` | checkout-service | checkout 5xx > 5% | 15s | **Ambiguous — see below** |
| `GatewayHighLatency` | gateway | gateway p95 > 2s | 25s | Generic, trails everything |
| `CheckoutDBPoolSaturation` | checkout-service | checkout pool > 80% in use | 30s | Supporting evidence |
| `CheckoutPaymentsLatency` | checkout-service | p95 to payments-service > 3s | 45s | Symptom |
| `CatalogDBPoolSaturation` | catalog-service | catalog pool > 80% in use | 90s | Symptom |
| `GatewayHighErrorRate` | gateway | gateway 5xx > 5% | 90s | Customer-impact confirmation |

## The three cascades

Match the set of alerts firing around the same time against these. The **lead**
alert names the cause; everything after it is that cause spreading.

| Cascade | Order | What actually broke |
|---|---|---|
| Payment outage | `PaylinityRequestLatency` → `CheckoutServerErrors` → `CheckoutPaymentsLatency` → `GatewayHighErrorRate` | The third-party payment provider is slow. Nothing in our code is wrong. |
| Analytics query | `PostgresBlockedQueries` → `CheckoutServerErrors` → `PostgresLongRunningTransaction` → `CatalogDBPoolSaturation` | A long transaction holds a lock on `products`; catalog starves for connections. |
| Promo checkout | `CheckoutServerErrors` → `GatewayHighErrorRate` | A latent bug in checkout-service panics on promo-code orders. |

### The CheckoutServerErrors rule

`CheckoutServerErrors` appears in **all three** cascades — checkout returns 5xx
whether it panicked itself, got a 502 from payments, or got a 502 from a starved
catalog. On its own it identifies nothing. Disambiguate by what else fired:

- `PaylinityRequestLatency` also firing → **payment outage**, not a checkout bug.
- `PostgresBlockedQueries` also firing → **analytics query lock**, not a checkout bug.
- Neither → **promo-checkout bug**, genuinely inside checkout-service.

Check for the other two before concluding anything about checkout. This is the
single most common way to get a storefront triage wrong.

## Tool surface

| What you want | Where it comes from |
|---|---|
| The alert's title, labels, status, timings | The incident.io connection's `alert_show` |
| What else was firing in the same window | The incident.io connection's `alert_list`, bounded to ±5 minutes around the alert |
| Whether the metric is still breaching | The **production metrics** datasource (Prometheus) |
| Whether real requests are failing | The **application logs** datasource (Loki) |
| Whether customers actually lost orders (order *rate*, not a success ratio) | The **order history** datasource (PostgreSQL) |

Where the session has a telemetry tool, ask these as questions and let it plan
the query. Where it does not, say which check you could not run and mark that
line of the output `unchecked` — never infer the answer from the alert alone.

## Reading results

- **An empty result and an unreachable source are different findings.** The
  storefront telemetry reaches incident.io through a tunnel from a local
  docker-compose stack, and that tunnel is frequently down — a failed query
  returns a connection error, not zero rows. If a query errors, the finding is
  "could not check", never "no impact found". Reporting a dead tunnel as a clean
  bill of health is the worst outcome this skill can produce.
- **A resolved alert is not automatically noise.** These alerts resolve as soon
  as the metric drops back under threshold, which happens the moment a scenario
  is switched off. Short-lived and harmless are different things — check impact
  before calling it.
- **A failed checkout never appears in the orders table.** Every row in
  `orders` carries `status = 'completed'` — there is no failed or abandoned
  state to count. Checkout failure therefore shows up only as a **drop in order
  rate**, never as a success ratio. Don't go looking for one.
- **The orders table holds a short window.** It is written continuously by the
  traffic generator and wiped when the stack is torn down, so there may be well
  under an hour of history. Compare against the minutes immediately before the
  alert, and where the baseline doesn't reach back far enough, say so rather
  than reporting no change.
- **`status_code` is a parsed field in Loki, not a stream label.** The stream
  labels are `container`, `env`, `level`, `service` and `service_name`.
  Filtering on `status_code` without the `| json` stage matches nothing, which
  reads exactly like "no failures".
- **No series returned is not a failed query.** A 5xx query with nothing over
  threshold comes back with no series; a broken tunnel comes back with a
  connection error. The first is a finding, the second is `unchecked`.
- **Absence of a lead alert is evidence.** If `CheckoutServerErrors` fired and
  neither Paylinity nor Postgres did, that absence is what identifies the promo
  bug. Say so explicitly rather than reporting it as nothing found.

## Procedure

1. **Read the alert you were given.** Title, service label, fired and resolved
   times. *Done when* you can name the service and the window.
2. **Find what else fired in that window.** List alerts from ±5 minutes around
   it and order them by fire time. *Done when* you have the ordered set, or have
   confirmed this alert fired alone.
3. **Identify the lead.** Match the ordered set against the three cascades above.
   The earliest alert in a known cascade is the cause; apply the
   CheckoutServerErrors rule if it is in the set. *Done when* you can name the
   cause, or state that the set matches no known cascade.
4. **Confirm impact.** These three are independent — issue them together
   rather than in sequence, so the answer is not delayed three round-trips.

   **Is the alert's own metric still breaching?** Re-run the alert's `expr` from
   `rules.yml`, or ask:

   > **production metrics** (Prometheus): what was the peak of this alert's
   > metric during the window, and is it over its threshold right now?

   **Are real requests failing?** Logs are structured JSON, so `status_code` is
   a parsed field and not a stream label — the `| json | __error__=""` stages
   are required before any comparison on it:

   ```logql
   # Datasource: application logs (Loki)
   sum by (service, status_code) (
     count_over_time(
       {env="production", service=~"gateway|checkout-service|payments-service"}
       | json | __error__="" | status_code >= 500 [1m]
     )
   )
   ```

   **Did customers lose orders?** Compare the order rate during the window
   against the minutes immediately before it:

   ```sql
   -- Datasource: order history (PostgreSQL)
   SELECT date_trunc('minute', created_at) AS minute, count(*) AS orders
   FROM orders
   WHERE created_at > now() - interval '30 minutes'
   GROUP BY 1 ORDER BY 1 DESC
   ```

   *Done when* each of the three has an answer or is explicitly `unchecked`.

5. **Call it.** Noise or real, per the rule below. *Done when* the output
   contract is filled.

## Noise or real

**The verdict is always exactly one of two words: `noise` or `real`.** Never
"unverified", "unknown" or "potential incident". An impact check you could not
run makes that *line* `unchecked` — it never makes the verdict a third thing.

Work down this list and stop at the first rule that matches.

1. **The alert is not one of the nine rules → `noise`.** Decided on rule
   membership alone. Do not run the impact checks to justify it, and do not read
   the rest of this list. Say what the alert is and that it matches no
   storefront rule, leave the Impact lines `unchecked`, and do not invent a
   storefront explanation for it.
2. **It is the lead of a recognised cascade → `real`.** `PaylinityRequestLatency`
   or `PostgresBlockedQueries` firing, or `CheckoutServerErrors` with neither of
   them in the window. The cascade is itself the impact, so this holds **even
   when every impact check is unchecked** — mark them unchecked and still call
   it real.
3. **There is impact evidence → `real`.** The order rate dropped,
   logs show genuine failed requests on a customer-facing path, or the metric is
   still over its threshold now.
4. **All three impact checks returned, and all are clean → `noise`.** A
   recognised alert that fired as a trailing symptom with no lead, the metric
   back under threshold, logs clean and the order rate unchanged.
5. **Anything left → `real`.** You could not complete the checks and no rule
   above settled it. Call it real and name exactly what you could not check.
   Under-calling a real incident costs more than over-calling a quiet one.

Severity shapes how urgently you say it, never which word you pick.

## Output contract

Answer in these headings, every time. A section you could not fill reads
`unchecked — tried <what>`, never silence — and an unchecked section never
changes the Verdict line, which is always `noise` or `real`.

```markdown
**Verdict:** noise | real

**What fired:** <alerts in the window, in fire order, with times>

**Cause:** <the lead alert and what it means — or "no known cascade matched">

**Impact:**
- Metrics: <still breaching? peak?>
- Logs: <real failures? what they say>
- Orders: <order rate vs the minutes before — or "unchecked: <why>">

**Recommended action:** <what a responder should do next>
```

Every number carries where it came from — the query or the tool that produced
it. A figure the reader cannot re-run is an opinion.

## What this skill does not do

- **Fix anything.** It reads and concludes; it never changes a rule, silences an
  alert, or restarts a service.
- **Explain how the harness works.** That is the `ai-demo-harness` README.
- **Handle alerts from other sources.** Anything not in the nine-rule table gets
  the "not a storefront signal" answer above.

A check this skill's tools cannot make is not proof the thing cannot be checked
— say what you wanted and could not reach, and let the reader take it further.
