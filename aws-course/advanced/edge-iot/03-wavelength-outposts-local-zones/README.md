# 03 — Wavelength, Outposts, Local Zones

## Concept
Wavelength = 5G edge (carrier networks). Outposts = AWS hardware on your premises. Local Zones = metro-level extensions.

## Exercises
1. **Local Zones**: enable in account. Launch EC2 in e.g. `us-east-1-bos-1a`. `ping` from local ISP vs main region — observe latency diff.
2. **Architecture (paper)**: design ultra-low-latency gaming service using LZ + Global Accelerator. Submit `design.md`.
3. **Wavelength** (paper, since it needs carrier): design a 5G AR app architecture.
4. **Outposts** (paper): design a HIPAA-compliant deployment with data never leaving premises. Outline network + KMS + services.
5. **Compare**: write `edge-options.md` — LZ vs WL vs OP vs CF Functions.

## Verification
- Local Zone EC2 accessible; latency measurably lower to you.

## Gotchas
- Not all services in LZ / WL — check service list.
- Outposts minimum order significant $$$.

## Cleanup
Terminate LZ EC2. Disable LZ (it's free to have enabled).
