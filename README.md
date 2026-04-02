# Batch Experiment Cost Optimizer

Minimize cloud batch experiment costs by checking for cached results before submitting new tasks. Works with any cloud batch system (GCP Cloud Run Jobs, AWS Batch, Azure Container Instances) that writes results to object storage.

## Quick Start

```
You: "Run permutation tests on the top 48 specs, 30 shuffles each"
Claude: Checking GCS cache... 20 specs already have results. Submitting 28 new specs x 30 shuffles = 840 tasks (~$5-8, vs $14 without cache).
```

## Installation

**Claude Code:**
```bash
git clone https://github.com/wan-huiyan/batch-experiment-cost-optimizer.git ~/.claude/skills/batch-experiment-cost-optimizer
```

**Cursor:**
```bash
mkdir -p .cursor/rules
cp ~/.claude/skills/batch-experiment-cost-optimizer/SKILL.md .cursor/rules/batch-experiment-cost-optimizer.mdc
```

## What You Get

| Without this skill | With this skill |
|---|---|
| Re-run all 1,440 tasks every time (~$14) | Check cache, submit only 840 new tasks (~$5) |
| No cost visibility before submission | Cost table with scenarios before committing |
| All-or-nothing runs | Additive shuffles with seed offset |
| Manual tracking of what's been computed | Automatic cache detection from file naming |

## The 5 Patterns

1. **Cache Check** -- List existing results in object storage, skip already-computed specs
2. **Cost Estimation** -- Present cost scenarios (with cache deduction) before submitting
3. **Additive Runs** -- Add more samples to existing results with shuffle offset for seed determinism
4. **Spec Exclusion** -- Filter contaminated/invalid configs before cost estimation AND submission
5. **Mixed Aggregation** -- Seamlessly combine cached + new results in the final analysis

## How It Works

| Step | What happens | Cloud-agnostic? |
|------|-------------|-----------------|
| 1. List cached results | `gsutil ls` / `aws s3 ls` / `az storage blob list` | Yes |
| 2. Parse spec IDs from filenames | `results_spec_{idx}_shuffle_{n}.json` pattern | Yes |
| 3. Filter selected specs | Remove cached + contaminated specs | Yes |
| 4. Show cost estimate | `new_tasks x $0.01/task` (conservative) | Yes |
| 5. Submit delta only | Cloud Run / AWS Batch / Azure CI | Platform-specific |
| 6. Aggregate all results | Read ALL files (cached + new) | Yes |

## Key Design Decisions

- **File naming convention drives cache detection** -- predictable patterns like `results_spec_{idx}_shuffle_{n}.json` allow parsing without reading file contents
- **Conservative cost estimates** -- $0.01/task assumes 2 min x 1 vCPU; actual Cloud Run costs are typically 30-50% lower
- **Seed determinism via offset** -- `seed = base + spec_id * 1000 + (shuffle_idx + offset)` ensures no collisions between runs

## Limitations

- Requires predictable file naming in object storage (not for unstructured result dumps)
- Cost estimates are rough upper bounds, not precise billing predictions
- Cache check adds ~30-60s of `gsutil ls` latency before submission
- Does not handle partial results within a single spec (all-or-nothing per spec)

## Related Skills

- [`cloud-run-batch-experiment`](https://github.com/wan-huiyan/cloud-run-batch-experiment) -- Set up the initial batch infrastructure (Dockerfile, Cloud Build, gcloud run jobs)
- [`gcp-pipeline-cost-analysis`](https://github.com/wan-huiyan/gcp-pipeline-cost-analysis) -- Broader GCP cost estimation across multiple services

## Version History

- **1.0.0** (2026-04-02) -- Initial release. Extracted from a real session where cache-then-submit saved 42% on a 48-spec permutation test run (~$5 vs ~$14).

## License

MIT
