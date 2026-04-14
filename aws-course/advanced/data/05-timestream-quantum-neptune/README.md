# 05 — Specialized Databases

## Concept
Timestream = time-series. QLDB = immutable ledger (being deprecated; migrate to Aurora Postgres). Neptune = graph (property + RDF).

## Exercises
1. **Timestream**: ingest 10k IoT sensor readings. Query: downsample to 1-min avg over last 24h.
2. **Memory vs magnetic store**: configure retention (memory 24h, magnetic 365d). Time query across both.
3. **Neptune**: property graph of social network. Load sample. Gremlin query "friends of friends not already friends".
4. **Neptune ML**: run a node classification example (skim).
5. **QLDB** (if still available in your region): ledger of financial transactions. Verify tamper-proof with PartiQL history query.
6. **Graph vs relational**: write `decision.md` — when graph wins.

## Verification
- Timestream query returns downsampled rows.
- Gremlin friends-of-friends returns expected set.

## Gotchas
- Timestream query language is SQL-ish but quirky.
- QLDB deprecation — new projects should avoid.
- Neptune cluster min = 1 instance db.t3.medium.

## Cleanup
```bash
cdk destroy
```
