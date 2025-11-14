## Retention KPI Calculation

### Objective
Measure how well recently activated Tinybots-as-a-Service (TaaS) subscriptions remain active after a retention threshold. The metric is delivered as a daily time series where each point contains:
- `date` — UTC day being evaluated.
- `retentionKPI` — fraction (0..1) of the relevant population that is retained.
- `population` — number of subscriptions considered on that date.

### Business Formula
We answer the question: *Among subscriptions that started recently enough to matter on day `x`, what percentage have stayed active for at least `R` days?*

**Key definitions**
- `P` — population window in days. On day `x` we look at subscriptions that started in `[x - (P - 1), x]`, inclusive. This approximates “recently won deals”.
- `R` — retention threshold in days. A subscription counts as retained on `x` only after it has been active for at least `R` full days.
- `lengthDaysAtX(s)` — number of whole days a subscription has been active by day `x`. It equals the day difference between the start day `s.start_at` and the earlier of the subscription’s `end_at` or `x`. Subscriptions still running past `x` are treated as active through `x`.

**Formula**
```
population(x) = count(
  subscription.start_at ∈ [x - (P - 1), x]
)

retained(x) = count(
  subscription.start_at ∈ [x - (P - 1), x]
  AND lengthDaysAtX(subscription) ≥ R
)

RetentionKPI(x) = retained(x) / population(x)           (population(x) > 0)
RetentionKPI(x) = 0                                     (population(x) = 0)
```

**Illustrative example**

Suppose `P = 5`, `R = 3`, and we evaluate day `x = 2024-05-10` (UTC). Four subscriptions exist:

| ID | start_at   | end_at       | Notes on 2024-05-10 |
|----|------------|--------------|---------------------|
| A  | 2024-05-06 | null         | Active for 4 days → retained |
| B  | 2024-05-08 | 2024-05-09   | Ended before day `x`; active for 1 day → not retained |
| C  | 2024-05-10 | null         | Same-day start, length 0 → not retained yet |
| D  | 2024-05-03 | null         | Started 7 days ago → outside the population window |

- Population window is `[2024-05-06, 2024-05-10]`, so subscriptions A, B, and C are in scope (3 total).
- Retained subscriptions have `lengthDaysAtX ≥ 3`; only A qualifies.
- The KPI for 2024-05-10 is `1 retained / 3 population = 0.3333`.

### Calculation Flow in the Codebase
1. **GraphQL interface** `OperationKPI.retentionKpi` requires `P` and `R`, accepts optional `startDate`, `endDate`, and a `policy` argument (`RESPECT_RETENTION_PERIOD` default, or `IGNORE_RETENTION_PERIOD`).
2. **Argument validation** `RetentionKpiArgsDto` (`src/graphql/schema/kpi/retentionKpi.ts`) ensures positive integers, ISO-8601 dates, and a recognised policy before continuing.
3. **Date normalisation** All dates are normalised to UTC midnight. When no custom range is supplied:
   - Determine the earliest `taas_subscription.start_at`.
   - Set `xStart = earliest + P`.
   - Choose `xEnd = today - R` for RESPECT policy, or `xEnd = today` when IGNORE policy is requested.
   - Prefetch subscriptions from `earliest` through `xEnd` (expanded by `P` days backwards if the caller supplied an explicit range).
4. **Data fetch** `RetentionKpiService` pulls the relevant subscriptions through Prisma in a single query, adding `start_at < today - R` for RESPECT policy to avoid subscriptions that cannot yet satisfy the threshold.
5. **Sliding window loop** Daily iteration from `xStart` to `xEnd` maintains a queue of subscriptions whose `start_at` falls within `[x - (P - 1), x]`. This produces the daily `population`.
6. **Retention evaluation** Each subscription in the queue is given an `effectiveEnd = min(end_at ?? x, x)` so that ongoing subscriptions are evaluated up to the current day. If `effectiveEnd.diff(start_at, 'days') ≥ R`, the subscription is retained for that day.
7. **Output** The service returns `{ date: 'YYYY-MM-DD', retentionKPI, population }` for each day. When `population = 0`, the KPI is forced to 0 to avoid undefined results.

### Date-Range Policies
- **RESPECT_RETENTION_PERIOD (default)** Omits the most recent `R` days when no explicit range is provided, guaranteeing that each reported day represents subscriptions that have had at least `R` days to prove retention.
- **IGNORE_RETENTION_PERIOD** Extends results through the current day, which is useful when the business wants to monitor “early performance” even though the newest subscriptions may still be inside their retention window.

### Reference Files
- `src/graphql/schema/kpi/operationKpi.ts` — GraphQL schema definition and argument wiring.
- `src/graphql/schema/kpi/retentionKpi.ts` — Resolver that validates input, determines date windows, and invokes the service.
- `src/graphql/schema/kpi/retentionKpiService.ts` — Core algorithm: Prisma fetch, sliding window population, retention calculation.
- `src/graphql/utils/datetime.ts` — UTC helpers reused across the KPI logic.
- `test/graphqlIT/RetentionKpiIT.ts` — Integration tests covering example scenarios and both date-range policies.
