---
name: batch-experiment-cost-optimizer
description: |
  Minimize cloud batch experiment costs by checking for cached results before submitting
  new tasks. Use when: (1) running iterative experiments where some specs/configs were
  already computed in prior runs, (2) adding more samples/shuffles to an existing batch,
  (3) the user asks "how much will this cost?" before submitting, (4) scaling up from a
  pilot run to full coverage. Applies to any cloud batch system (GCP Cloud Run Jobs,
  AWS Batch, Azure Container Instances, Databricks Jobs) with object storage results
  (GCS, S3, Azure Blob). Trigger on phrases like "skip cached", "incremental run",
  "how much will this cost", "budget for batch", "don't re-run existing", "additive
  shuffles", "top-up experiment", or when the user expresses cost concern about a
  large batch submission.
author: Claude Code
version: 1.0.0
date: 2026-04-02
---

# Batch Experiment Cost Optimizer

## Problem

When running large parametric experiments on cloud batch infrastructure (e.g., 448 specs x
30 shuffles = 13,440 tasks), re-running everything from scratch is wasteful if prior runs
already computed some results. Users need a way to:

1. **Check what's already cached** in object storage before submitting
2. **Submit only the delta** (new specs/configs not yet computed)
3. **Estimate cost** before committing to a run
4. **Aggregate results** from both cached and new runs seamlessly

## Context / Trigger Conditions

- User says "run permutation/experiment on N specs" and some were already run previously
- User asks "how much will this cost?" or sets a budget ("under $10")
- Script has a `--no-clean` flag suggesting additive runs were considered
- Prior results exist in GCS/S3 with a predictable naming pattern (e.g., `results_spec_{idx}_shuffle_{n}.json`)
- User wants to "top up" from 30 shuffles to 80 on a specific config

## Solution

### Pattern 1: Cache Check Before Submission

```python
# 1. List existing results in object storage
ls_result = subprocess.run(
    ["gsutil", "ls", f"gs://{bucket}/{prefix}/results_spec_*.json"],
    capture_output=True, text=True, timeout=30)

# 2. Parse cached spec indices from filenames
cached_specs = set()
for f in ls_result.stdout.strip().split("\n"):
    if "results_spec_" in f:
        si = int(f.split("results_spec_")[1].split("_shuffle_")[0])
        cached_specs.add(si)

# 3. Filter out already-computed specs
new_specs = [s for s in selected_specs if s["spec_index"] not in cached_specs]
print(f"GCS cache: {len(selected_specs) - len(new_specs)} already tested, {len(new_specs)} new")
```

For AWS S3: replace `gsutil ls` with `aws s3 ls` or `boto3.list_objects_v2()`.
For Azure: replace with `az storage blob list` or `BlobServiceClient.list_blobs()`.

### Pattern 2: Cost Estimation Before Submission

Present a cost table to the user BEFORE submitting:

```python
# Conservative estimate: ~$0.01 per task (2 min x 1 vCPU at Cloud Run pricing)
cost_per_task = 0.01
scenarios = {
    "Top 12 per mode (48 specs)": 48 * 30,
    "Top 100 globally": 100 * 30,
    "All 440 clean specs": 440 * 30,
}
for name, tasks in scenarios.items():
    cached = len(cached_specs & set_of_specs_for_scenario)
    new_tasks = tasks - (cached * shuffles_per_spec)
    print(f"{name}: {new_tasks} new tasks, ~${new_tasks * cost_per_task:.2f}")
```

Key: always show **new tasks only** (after cache deduction), not total.

### Pattern 3: Additive Runs (Shuffle Offset)

When adding more samples to existing results without re-running:

```python
# Config includes shuffle_offset so new tasks get different random seeds
config = {
    "n_shuffles": 50,        # New batch size
    "shuffle_offset": 30,     # Continue from where prior run ended
}

# In the task runner, offset the shuffle index for seed generation:
shuffle_idx = (task_index % n_shuffles) + shuffle_offset
rng = random.Random(42 + spec_index * 1000 + shuffle_idx)
```

### Pattern 4: Exclude Contaminated/Invalid Specs

Filter before cost estimation AND before submission:

```python
parser.add_argument("--exclude-bundles", type=str, default="",
                    help="Comma-separated bundle IDs to exclude (e.g. S54)")
exclude = set(b.strip() for b in args.exclude_bundles.split(",") if b.strip())
ok_results = [r for r in ok_results
              if specs_by_idx.get(r["spec_index"], {}).get("bundle_id") not in exclude]
```

### Pattern 5: Seamless Aggregation of Mixed Sources

The aggregation step must read ALL results from storage (cached + new), not just the
newly submitted batch:

```python
# Read ALL result files, regardless of which run produced them
all_files = subprocess.run(["gsutil", "ls", pattern], ...)
# Parse all, group by spec_index, compute statistics
for spec_idx, shuffles in grouped_results.items():
    real_effect = get_real_effect(spec_idx)
    perm_effects = [abs(s["abs_eff"]) for s in shuffles]
    n_extreme = sum(1 for pe in perm_effects if pe >= real_effect)
    perm_p = (n_extreme + 1) / (len(shuffles) + 1)
```

## Verification

After implementing, verify:
1. `--skip-cached` correctly identifies N cached specs (matches `gsutil ls` count)
2. Cost estimate shows $0 when all specs are cached
3. New submission has fewer tasks than full run
4. Aggregation produces results for ALL selected specs (cached + new), not just new ones
5. Results from mixed runs (old + new) are statistically compatible

## Example

Real-world session where this saved ~60% cost:

```
$ python submit_permutation.py --per-mode 12 --n-shuffles 30 --exclude-bundles S54 --skip-cached

=== SCA Permutation Submission ===
1. Loading SCA results from GCS...
   Loaded 448 SCA results
   Excluded 8 specs from bundles {'S54'}
   48 specs selected (12 per mode)
   GCS cache: 20 specs already tested, 28 new    # <-- 42% savings
   
2. Uploading config (28 specs x 30 shuffles = 840 tasks)...
   Est. cost: ~$5-8 (vs ~$14 without cache)
   
3. Submitting 840 tasks to Cloud Run...
   Succeeded: 840/840

5. Aggregating all 48 specs (20 cached + 28 new)...
   Significant (perm p < 0.05): 7/43
```

## Notes

- **File naming convention matters**: Use predictable patterns like `results_spec_{idx}_shuffle_{n}.json`
  so cache checks can parse spec/shuffle IDs from filenames without reading file contents.
- **Don't delete cached results**: Use `--no-clean` flag. The default should preserve prior results.
- **Seed determinism**: When using shuffle_offset, ensure seeds don't collide with prior runs.
  Pattern: `seed = base + spec_id * 1000 + shuffle_idx` where shuffle_idx includes offset.
- **Cost estimate is conservative**: Cloud Run bills per-second, so 2-min tasks cost less than
  $0.01 each. The $0.01/task estimate provides a safe upper bound.
- **Cross-platform**: The cache-check-then-submit pattern works identically on AWS Batch (S3),
  Azure Container Instances (Blob Storage), or any system with object storage results.

## See Also

- `cloud-run-batch-experiment`: How to set up the initial batch infrastructure
- `gcs-async-cache-gap`: Fixing missing cache writes in webapp async paths
- `gcp-pipeline-cost-analysis`: Broader GCP cost estimation across services
