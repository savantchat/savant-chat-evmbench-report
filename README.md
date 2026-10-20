# Savant Chat on EVMBench — public high-severity track

This repository publishes the audit reports **Savant Chat** produced for the public EVMBench
high-severity set, and EVMBench's own verdict on each of them.

The set is 117 issues: the published high-severity issues minus three excluded as low quality
(`2026-01-tempo-mpp-streams/H-01`, `2026-01-tempo-mpp-streams/H-02`,
`2026-01-tempo-stablecoin-dex/H-03`).

## Result

**112 / 117 = 95.7 %**

Scored the way EVMBench scores: one audit report per issue, judged as a whole against that issue's
ground-truth finding by **EVMBench's own `gpt-5` critic** (`reasoning_effort: high`, structured
verdict). One verdict per report — no voting, no retries, no per-hypothesis credit.

A report counts as a hit when the critic finds that it describes the issue's vulnerability: the same
mechanism, the same code path and the same fix. "Same area, similar impact" is not a hit, and the
critic's written reasoning for every verdict is published so you can check that for yourself.

## Files

| File | Contents |
|---|---|
| `reports/<audit>__<issue>.md` | the audit report for that issue, as delivered |
| `reports.jsonl` | one row per issue: the same reports with their findings as structured fields |
| `evmbench_judgement.jsonl` | one row per issue: EVMBench's `gpt-5` critic verdict and its full reasoning |
| `issues.jsonl` | one row per issue: EVMBench's ground-truth description and root-cause files |
| `per_report_results.md` | the hit/miss table, one row per report |

### `reports.jsonl`

```jsonc
{
  "issue_id": "2024-01-renft/H-02",
  "audit": "2024-01-renft",
  "report_file": "reports/2024-01-renft__H-02.md",
  "findings_count": 14,
  "findings": [
    {
      "title": "...",
      "synopsis": "...",
      "technical_details": "...",
      "assumptions_and_preconditions": "...",
      "proof_of_concept": "...",
      "impact_assessment": "...",
      "remediation": "..."
    }
  ]
}
```

### `evmbench_judgement.jsonl`

```jsonc
{
  "issue_id": "2024-01-renft/H-02",
  "report_file": "reports/2024-01-renft__H-02.md",
  "evmbench_critic_detected": true,       // did EVMBench's gpt-5 critic match this report to the issue?
  "evmbench_critic_reasoning": "...",     // the critic's own explanation of the verdict
  "judge_model": "gpt-5",
  "judge_reasoning_effort": "high"
}
```

## Reproducing the verdicts

Every verdict here is one call to EVMBench's own grader, so it reproduces without Savant Chat:

1. take `reports/<audit>__<issue>.md` as the audit content;
2. take the issue's ground-truth finding — `ground_truth_description` in `issues.jsonl`, which is
   EVMBench's own `audits/<audit>/findings/<issue>.md`;
3. run EVMBench's `DetectGrader` (`evmbench/nano/grade/detect.py`) on the pair with `gpt-5` at
   `reasoning_effort: high` and its structured verdict schema, unmodified.

The grader is an LLM, so verdicts are not bit-exact. On identical input it was stable here —
re-judging 23 of these report/issue pairs a second time returned the same verdict 23 out of 23
times, in both directions — but a re-run is not guaranteed to agree on every issue.

## Note on the reports

The reports are published as delivered, with one mechanical edit: internal scanner scaffolding was
removed from the text — the line markers used to build a code fragment, and a small number of lines
in which the model quoted or discussed its own instructions instead of describing the contract. No
finding and no section was removed, and nothing was added or rewritten.
