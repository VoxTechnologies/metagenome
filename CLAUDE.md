# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Status: ARCHIVED (2026-05-10) — design repo, never deployed

**Read this before proposing any change.** This repo is a *design* for a PMDA-compliant MinION metagenomic pathogen-screening pipeline for xenotransplantation donor pigs. It was **never deployed to AWS**: no terraform state, no CloudWatch logs, no production S3 buckets. Numbers in `docs/grants/` labelled "Scenario" (e.g. "47 samples processed") are hypothetical grant figures, not measured data.

- Authoritative successor: `fluxia-samd/apps/donor-screening/` (Vercel + Convex + Modal stack), <https://github.com/masterleopold/fluxia-samd>
- Three files here are cited as **IEC 62304 SOUP sources** and were ported downstream — treat them as frozen provenance, not as code to refactor:
  - `scripts/phase1_basecalling/generate_summary_from_fastq.py:31-176`
  - `scripts/phase5_quantification/absolute_copy_number.py:109-140`
  - `scripts/phase3_host_removal/remove_host.sh`
- Preserved for SOUP traceability, JST grant-trail addressability, and AWS-fallback recovery. Tag: `archive/aws-design-never-deployed`.

Practical consequence: default to **read, explain, and trace** rather than build. Before writing code here, confirm the user does not actually want the change in `fluxia-samd`. Changes that would ship as product belong downstream; changes here should preserve the archived record.

## Environment and commands

There is no `pyproject.toml`, `pytest.ini`, or lockfile — dependencies come from `requirements.txt` and are **not installed in the ambient environment** (verified 2026-07-27: `boto3`, `black`, `flake8` all absent). Create a venv first, or every command below fails on import.

```bash
uv venv && source .venv/bin/activate && uv pip install -r requirements.txt

# Tests — run from repo root; imports resolve via rootdir (`from lib.models...`), no conftest.py
pytest tests/                                        # full suite
pytest tests/integration/test_new_patterns.py -v     # v2.0 patterns (Pydantic + Repository)
pytest tests/test_pmda_compliance.py::TestPMDACompliance::test_action_thresholds -v  # single test
pytest tests/ --cov=scripts --cov=lambda

black scripts/ lambda/ tools/ lib/                   # format
flake8 scripts/ lambda/ lib/                         # lint (no config file; defaults apply)

# Docs portal (Next.js 15 / React 19, separate toolchain)
cd docs-portal && npm install && npm run dev         # :3000; also `npm run type-check`

# Surveillance UIs
cd surveillance/dashboard && streamlit run app.py    # :8501
cd surveillance/api && uvicorn main:app              # :8000
```

**The test suite is red and was left that way.** Last measured: 22 failed / 8 passed (excluding 3 modules that fail collection on missing `boto3`). Failures are schema drift between the tests and `templates/config/pmda_pathogens.json` — e.g. `test_action_thresholds` expects `'quarantine'` where the config now says `'QUARANTINE'`. Do **not** read a failing suite as a regression you introduced; establish the baseline first.

Pipeline CLI entry points (`tools/workflow_cli.py start|status|metrics`, `tools/deployment_script.sh`, `tools/database_setup.sh`) all talk to AWS resources that do not exist. They are executable documentation.

## Architecture

**7-phase containerless pipeline.** Lambda orchestrates EC2 instances built from custom AMIs — deliberately no Docker. Each phase launches, runs a UserData script, and self-terminates.

```
S3 upload → lambda/orchestration/pipeline_orchestrator.py → Step Functions → EC2 per phase
  Phase 0 sample prep routing (t3.small)   → scripts/phase0_sample_prep/
  Phase 1 basecalling (g4dn.xlarge, GPU)   → Dorado duplex, FAST5/POD5 → FASTQ
  Phase 2 QC (t3.large)                    → NanoPlot/PycoQC, RIN for RNA
  Phase 3 host removal (r5.4xlarge, 128GB) → Minimap2 vs Sus scrofa
  Phase 4 pathogen detection (4× parallel) → Kraken2 / RVDB / BLAST / PERV typing
  Phase 5 quantification (t3.large)        → PhiX174 spike-in → copies/mL
  Phase 6 reports (t3.large)               → PMDA checklist, PDF + JSON
```

The three-layer split matters more than the file tree: **`scripts/phase*/`** is what runs *on* EC2 (bioinformatics, mostly standalone Python/shell), **`lambda/`** is what launches and watches it (orchestration, phase triggers, EC2 lifecycle, API), **`lib/`** is shared library code imported by both. Reference DBs live on EFS (`/mnt/efs/databases/{kraken2,blast,perv}`), so scripts assume those paths exist.

**v2.0 patterns** (`lib/`, added 2025-01):
- `lib/models/` — Pydantic v2 models. `pathogen.py` carries the PERV/91-pathogen types; `PERVDetectionResult.requires_alert` is True for confidence HIGH **or MEDIUM**, and that property is what gates the SNS alert.
- `lib/repositories/` — `interfaces.py` defines `typing.Protocol` contracts; `rds_repository.py` (RDS Data API/PostgreSQL) for production, `sqlite_repository.py` for tests, `dynamodb_repository.py` for surveillance state. Swap the implementation, never the call sites.
- `lib/logging/` — AWS Lambda Powertools Logger/Tracer/Metrics; `lib/audit/cloudwatch_queries.py` holds 12 prebuilt PMDA audit queries.

**Surveillance system** (`surveillance/`, v2.3.0) is a semi-independent subsystem with its own `requirements.txt` and terraform: 4-virus monitoring (Hantavirus, Polyomavirus, Spumavirus, EEEV) combining `external/` scrapers (MAFF, E-Stat, PubMed, J-STAGE; daily Lambda at 11:00 JST) with `internal/` Phase 4 result listening, feeding a 4-level severity engine in `alerting/` that posts to Slack.

`md/` holds the original Japanese design specifications (91-pathogen list, PMDA suitability assessment, NGS platform comparisons). When a requirement's origin is unclear, it is usually there rather than in `docs/`.

## Domain constraints

- **PERV**: any PERV-A/B/C detection triggers an immediate SNS alert (`scripts/phase4_pathogen/perv_typing.py`). Do not weaken the threshold path.
- **PMDA**: 91 pathogens, PPA >95%, NPA >98%. Caveat found 2026-07-27: `templates/config/pmda_pathogens.json` self-reports `total_pathogens: 91`, but `categories.viruses.count` says 41 while the array holds 48 entries, and the categories enumerate 101 entries / 98 unique names (3 names appear in both `viruses` and `special_management`). The counts do not reconcile. Verify against `md/厚労省異種移植指針_91病原体リスト.md` before relying on any figure.
- **J-STAGE ToS**: 24h max retention, aggregated statistics only (`surveillance/docs/JSTAGE_COMPLIANCE.md`).
- **AWS region**: always `ap-northeast-1` (Tokyo), hardcoded for regulatory reasons — pass `region_name='ap-northeast-1'` explicitly on every boto3 client.
- **No patient data in git.** Encrypted S3 only.

## Conventions

File validation (`Path.exists()` plus a zero-byte check) and BAM auto-indexing (`pysam.index` when `.bai` is missing) are applied at the top of every analysis script — match that when touching `scripts/`.

| File | Why it is load-bearing |
|------|------------------------|
| `scripts/phase4_pathogen/perv_typing.py` | PERV detection logic; changing it invalidates PMDA validation |
| `templates/config/pmda_pathogens.json` | 91-pathogen definitions; regulatory review artifact |
| `lambda/orchestration/pipeline_orchestrator.py` | Entry point for the whole workflow |
| `lib/models/pathogen.py` | Alert-gating types shared by scripts, Lambda, and surveillance |
| `lib/repositories/interfaces.py` | Contract all three repository backends must satisfy |

## Further reading

`CLAUDE_REFERENCE.md` (full directory tree, detailed AWS pattern) · `docs/ARCHITECTURE.md` · `docs/NEW_PATTERNS_GUIDE.md` (v2.0 patterns) · `docs/API_REFERENCE_V2.md` · `docs/QUICK_REFERENCE.md` · `docs/RECENT_UPDATES.md` · `docs/grants/` (NVIDIA DGX Spark ARM application — scenario data)
