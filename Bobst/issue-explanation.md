# OEE waste-only production issue

## Context

Reproduce the issue
    - Machine 
      - BOBST SA/ Manchester 1 / K5E 1350 Rambert/ 
    - https://ra-develop.iotlab.connect.bobst.com/performance/historical?view=detailed&tab=job&start=2026-05-06&end=2026-05-16

The issue is visible in the per-job OEE view for machine `GENV21G018`, job `115 GSM ONEBARRIER ALUBOND 2026 (260514-0745)`.

For this job, the OEE response shows:

- `availability`: `0.044571323326273136`
- `performance`: `0.0`
- `quality`: `0.0`
- `qualityDetails.goodMaterialProduced`: `0.0`
- `qualityDetails.waste`: `0.0`
- `totalProduced`: `"0"`
- `targetProduced`: `"1613.3333333333335"`
- `jobAverageSpeed`: `0.0`

However, telemetry for the same period shows that the machine produced waste:

- `deltaOut`: `0`
- `deltaBad`: `9,141`

So the machine did not produce good material, but it did produce material that was rejected as bad output.

## Actual problem

The current OEE calculation treats `TotalOutput = 0` as no production. That is correct only when both good output and bad output are zero.

In this case, `TotalOutput = 0` and `TotalBadOutput = 9,141`, so production did happen. It was just 100% waste.

Because the calculation currently treats this as no production, it incorrectly returns:

- `performance = 0`, instead of calculating performance from total produced material.
- `qualityDetails.waste = 0`, even though telemetry reports bad output.
- `totalProduced = "0"`, even though total produced material is `good + bad`.
- `jobAverageSpeed = 0`, even though material moved through the machine.

This hides waste-only production from the OEE details and makes the job look like it produced nothing.

## Expected behavior

For OEE performance, the production volume should be:

```text
total produced = good output + bad output
```

For the reported job:

```text
good output = 0
bad output = 9,141
total produced = 9,141
```

Performance should compare `9,141` against the target output `1,613.3333333333335`, using the same existing validation rules as today, including the current maximum performance limit.

Quality should be zero because no good material was produced:

```text
quality = good output / total produced
quality = 0 / 9,141
quality = 0
```

But quality details should still expose the counters:

- `goodMaterialProduced = 0`
- `waste = 9,141`

The job should not be treated as a no-production job.

## Likely root cause in code

The issue appears to come from shared OEE domain rules, not only from the job endpoint response mapping.

The current logic uses good output as the only signal for production:

- `Domain.Oee.OeeCounters.HasProduction` checks only `TotalOutput`.
- `PerformanceDetailsCalculation.GetPerformance()` returns `0` when `HasProduction` is false.
- `QualityCalculation.GetWaste()` returns no waste when good output is zero, because it checks only produced good material.
- OEE details such as `TotalProduced` and `JobAverageSpeed` are populated from `TotalOutput`, not from `TotalOutput + TotalBadOutput`.

This means waste-only production is classified as no production.

## Business impact

When a machine produces only waste, the OEE job view currently under-reports production activity and hides the rejected material amount.

That affects operators and reporting because:

- performance is not calculated from material that actually passed through the machine;
- quality details do not show the waste quantity;
- job-level production totals are misleading;
- waste-only jobs can look like inactive jobs.

## Fix direction

The fix should introduce a consistent concept of effective produced material for OEE:

```text
effective produced = good output + automatic bad output
```

Use this effective produced value for:

- performance numerator;
- `totalProduced` in OEE details;
- job average speed;
- no-production checks.

Keep good material separate:

- `goodMaterialProduced` should remain the good output amount after manual rejection rules.
- `quality` should remain `0` when good output is zero.
- `waste` should still show bad output when bad output exists.

The no-production branch should apply only when both good output and bad output are zero.

## Acceptance criteria

A waste-only job where `TotalOutput = 0` and `TotalBadOutput > 0` should return:

- performance calculated from `TotalOutput + TotalBadOutput`, subject to existing validation limits;
- quality equal to `0`;
- `qualityDetails.goodMaterialProduced = 0`;
- `qualityDetails.waste = TotalBadOutput`;
- `totalProduced = TotalOutput + TotalBadOutput`;
- job average speed based on `TotalOutput + TotalBadOutput`;
- no false `TotalOutputNotSet` error when counters are present and valid.

A true no-production job where both `TotalOutput = 0` and `TotalBadOutput = 0` should continue returning zero production values.

