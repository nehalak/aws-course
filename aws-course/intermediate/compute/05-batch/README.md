# 05 — AWS Batch

## Concept
Managed batch computing. Submit jobs → Batch provisions compute (Fargate/EC2/Spot) → runs → scales down.

## Exercises
1. **Fargate compute environment + job queue + job definition**. Job runs `python -c "print('hello batch')"`.
2. **Submit 100 jobs**; watch queue drain in parallel across Fargate tasks.
3. **EC2 CE with Spot**: 70% cheaper. Submit 1000 CPU-bound jobs (prime sieve). Observe cost.
4. **Array job**: single submission with `--array-properties size=50` runs 50 parallel children.
5. **Job dependencies**: job B depends on job A success.
6. **Multi-node parallel**: MPI-style job for distributed workload (skip if no MPI need).

## Verification
- All 100 jobs succeed.
- Spot CE shows interruptions handled via retry.

## Gotchas
- Cold start for EC2 CE = 5+ min first job.
- Fargate jobs: max 4 vCPU 30GB.
- Job def revisions pile up — clean old.

## Cleanup
```bash
cdk destroy
```
