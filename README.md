# MinION Pathogen Screening Pipeline

> **ARCHIVED 2026-05-10 — repositioned as Fluxia Donor Screening on a Vercel + Convex + Modal stack.**
>
> This codebase was **never deployed to AWS**. No terraform state, no Lambda CloudWatch logs, no S3 production buckets — see Stage 1 PoC §13.5 of the Fluxia v3.0 implementation record for the full audit. The "47 samples processed" and similar figures in `docs/grants/` are explicitly labelled "Scenario" — hypothetical metrics for grant submissions, not measured operational data.
>
> **The authoritative donor-screening implementation now lives at:**
> [`fluxia-samd/apps/donor-screening/`](https://github.com/masterleopold/fluxia-samd/tree/main/apps/donor-screening) (the Fluxia SaMD monorepo).
>
> Pointers from v3.0 documentation reference specific functions in this repo as **SOUP sources**:
> - `scripts/phase1_basecalling/generate_summary_from_fastq.py:31-176` → ported to `fluxia-samd/apps/qc/qc/fastq_summary.py:compute_qc_metrics`
> - `scripts/phase5_quantification/absolute_copy_number.py:109-140` → ported to `fluxia-samd/apps/donor-screening/quantification/copy_number.py:compute_copy_numbers`
> - `scripts/phase3_host_removal/remove_host.sh` → ported to `fluxia-samd/apps/donor-screening/compute/run_host_removal.py:remove_host`
>
> This repo is preserved (rather than deleted) for IEC 62304 SOUP traceability + JST grant trail addressability + AWS-fallback recovery posture. Tag: `archive/aws-design-never-deployed`. See the Fluxia v3.0 closure docs (Implementation/06d) for the SDLC closeout statement.

----

PMDA-compliant metagenomic analysis pipeline design for xenotransplantation donor pig screening using Oxford Nanopore MinION Mk1D. *(See archive banner above — this design was never built.)*

## Overview

This pipeline provides comprehensive pathogen detection for 91 PMDA-designated pathogens in donor pigs intended for xenotransplantation. The system uses Oxford Nanopore long-read sequencing technology combined with AWS cloud infrastructure for scalable, cost-effective analysis.

**Latest Updates**:
- **v2.3.0 (2025-01-17)**: J-STAGE Terms of Service compliance - 24h data retention limit, DynamoDB TTL, S3 lifecycle rules, aggregated statistics only
- **v2.0.0 (2025-01-15)**: Major code quality improvements:
  - Type-safe Pydantic models (18 models with auto-validation)
  - Repository pattern (RDS + SQLite for testing)
  - Unified logging with AWS Lambda Powertools
  - CloudWatch audit queries (12 pre-built queries)
  - 10x faster tests, 60x faster PMDA audit reports
- **v2.2.0 (2025-11-15)**: Added Slack notification integration to 4-Virus Surveillance System with real-time alerts
- **v2.1.0 (2025-11-14)**: Protocol 12 now includes circular and single-stranded DNA virus detection, achieving TRUE 100% pathogen coverage

See [docs/RECENT_UPDATES.md](docs/RECENT_UPDATES.md) for complete changelog.

## Key Features

- **PMDA Compliance**: Full coverage of 91 designated pathogens
- **PERV Detection**: Critical detection of Porcine Endogenous Retroviruses (PERV-A, B, C)
- **4-Virus Surveillance**: Real-time monitoring of Hantavirus, Polyomavirus, Spumavirus, and EEEV with external data integration (MAFF, E-Stat, PubMed, J-STAGE compliant) and Slack notifications
- **Real-time Analysis**: Streaming analysis capability with MinION
- **Cloud-Native**: Serverless architecture on AWS (Lambda + EC2 on-demand)
- **Automated Workflow**: End-to-end automation from basecalling to reporting
- **Quality Assured**: Q30+ accuracy with duplex basecalling
- **Cost Optimized**: Spot instances and on-demand scaling

## System Architecture

**Implementation**: Containerless serverless architecture using Lambda functions to orchestrate EC2 instances with custom AMIs.

```
MinION Sequencer → S3 Upload → Lambda Orchestrator → Step Functions
                                                            ↓
                                      Lambda triggers EC2 instances per phase
                                                            ↓
                    [Phase 0: Sample Prep Routing (t3.small) →
                     Phase 1: GPU EC2 Basecalling (g4dn.xlarge) →
                     Phase 2: QC (t3.large) →
                     Phase 3: Host Removal (r5.4xlarge) →
                     Phase 4: Pathogen Detection (4x parallel EC2) →
                     Phase 5: Quantification (t3.large) →
                     Phase 6: Report Generation (t3.large)]
                                                            ↓
                                      EC2 auto-terminates after completion
                                                            ↓
                                            Reports stored in S3
```

**Key Features**:
- No Docker containers - uses custom AMIs with pre-installed analysis tools
- Lambda functions orchestrate EC2 lifecycle (launch, monitor, terminate)
- EFS for shared reference databases (Kraken2, BLAST, PERV DB)
- Spot Instances for 70% cost savings
- Each EC2 instance runs UserData scripts and auto-terminates

## Requirements

### Hardware
- Oxford Nanopore MinION Mk1D
- GPU-enabled EC2 instances (g4dn.xlarge for basecalling)
- High-memory EC2 instances (r5.4xlarge for pathogen detection)

### Software
- Python 3.11+
- Terraform 1.0+
- AWS CLI configured
- **Note**: No Docker required - uses custom AMIs with pre-installed tools
- **New Dependencies** (v2.0):
  - `pydantic>=2.5.0` - Type-safe data models
  - `pydantic-settings>=2.1.0` - Configuration management
  - `aws-lambda-powertools>=2.0.0` - Structured logging (optional, for Lambda)

### AWS Services
- S3 for data storage
- Lambda for orchestration
- EC2 for compute
- Step Functions for workflow
- RDS Aurora Serverless for metadata
- EFS for reference databases
- SNS for notifications

## Documentation

- **[Quick Start Guide](docs/QUICK_REFERENCE.md)** - Commands and usage
- **[Architecture Overview](docs/ARCHITECTURE.md)** - System design and components
- **[Code Patterns](docs/PATTERNS.md)** - Development patterns and conventions
- **[v2.0 Patterns Guide](docs/NEW_PATTERNS_GUIDE.md)** - Type safety, repository pattern, logging
- **[API Reference](docs/API_REFERENCE_V2.md)** - v2.0 API documentation
- **[Recent Updates](docs/RECENT_UPDATES.md)** - Changelog and version history
- **[Full Technical Reference](CLAUDE_REFERENCE.md)** - Detailed implementation notes
- **[NVIDIA Grant Materials](docs/grants/)** - Academic grant application (DGX Spark ARM)

## Installation

### 1. Clone Repository

```bash
git clone https://github.com/your-org/minion-pipeline.git
cd minion-pipeline
```

### 2. Install Dependencies

```bash
pip install -r requirements.txt
```

### 3. Configure AWS

```bash
aws configure
export AWS_REGION=ap-northeast-1
export ENVIRONMENT=production
```

### 4. Deploy Infrastructure

```bash
cd infrastructure/terraform
terraform init
terraform plan
terraform apply
```

### 5. Setup Databases

```bash
./tools/database_setup.sh --all
```

### 6. Deploy Pipeline

```bash
./tools/deployment_script.sh deploy
```

## Usage

### Starting a Workflow

```bash
# Using CLI
./tools/workflow_cli.py start \
  --run-id RUN-2024-001 \
  --bucket minion-data \
  --input-prefix runs/RUN-2024-001/fast5/

# Using API
curl -X POST https://api.your-domain.com/workflows \
  -H "x-api-key: YOUR_API_KEY" \
  -d '{
    "run_id": "RUN-2024-001",
    "bucket": "minion-data",
    "input_prefix": "runs/RUN-2024-001/fast5/"
  }'
```

### Monitoring Progress

```bash
# Check status
./tools/workflow_cli.py status --run-id RUN-2024-001 --watch

# View metrics
./tools/workflow_cli.py metrics --run-id RUN-2024-001

# Launch dashboard
streamlit run tools/monitoring_dashboard.py
```

### Accessing Reports

Reports are automatically generated in multiple formats:
- **PDF**: Comprehensive report with visualizations
- **JSON**: Machine-readable PMDA checklist
- **HTML**: Interactive web report

## Sample Preparation Protocols

### Protocol 12 v2.1 (Recommended)
- **Universal workflow** for 100% pathogen coverage
- **Time**: 15.5 hours hands-on
- **Cost**: ¥162,000/sample
- **Key feature**: Includes circular/ssDNA virus detection (PCV2, PCV3, TTV, PPV)

### Protocol 11 (Optional)
- Use for ultra-high sensitivity (<50 copies/mL)
- Target: Polyomavirus, Hantavirus, EEEV, Spumavirus

### Protocol 13 (Conditional)
- Triggered by retrovirus pol signatures
- Spumavirus-specific screening

For detailed protocols, see [docs/PROTOCOLS_GUIDE.md](docs/PROTOCOLS_GUIDE.md).

## Pipeline Phases

### 0. Sample Preparation Routing
- Determines DNA vs RNA extraction workflow
- Selects appropriate protocol based on sample type

### 1. Basecalling
- Converts raw signal (FAST5/POD5) to sequences (FASTQ)
- Uses Dorado with duplex mode for Q30 accuracy
- GPU-accelerated on g4dn instances

### 2. Quality Control
- Assesses read quality metrics
- Filters low-quality reads
- Generates QC reports with NanoPlot

### 3. Host Genome Removal
- Aligns reads to Sus scrofa reference genome
- Removes host DNA contamination
- Retains unmapped (potentially pathogenic) reads

### 4. Pathogen Detection
- **Kraken2**: Rapid taxonomic classification
- **RVDB**: Viral database search
- **BLAST**: PMDA custom database alignment
- **PERV-specific**: Targeted PERV detection and typing

### 5. Quantification
- Spike-in normalization (PhiX174)
- Absolute copy number calculation (copies/mL)
- Confidence interval estimation

### 6. Report Generation
- PMDA compliance checklist (91 pathogens)
- Detection summary with risk assessment
- Quality metrics and validation

## Configuration

### Default Configuration

See `templates/config/default_pipeline.yaml` for full configuration options.

```yaml
phases:
  basecalling:
    skip_duplex: false  # Use duplex for Q30
    min_quality: 9

  pathogen_detection:
    databases:
      - kraken2
      - rvdb
      - pmda
    confidence_threshold: 0.1

  reporting:
    formats:
      - pdf
      - json
    pmda_checklist: true
```

### Custom Configuration

```bash
# Create custom config
./tools/workflow_cli.py config --create-default

# Edit config
vi /etc/minion-pipeline/custom.yaml

# Validate
./tools/workflow_cli.py config --validate custom.yaml
```

## PMDA Compliance

### 91 Pathogen Coverage

The pipeline is designed to screen all 91 PMDA-designated pathogens:
- **Viruses**: 41 pathogens including circular/ssDNA viruses
- **Bacteria**: 27 pathogens
- **Parasites**: 19 pathogens
- **Fungi**: 2 pathogens
- **Special Management**: 5 pathogens (PCV2, PCV3, PERV-A/B/C)

> **Count discrepancy (found 2026-07-27, unresolved).** These figures do not reconcile with `templates/config/pmda_pathogens.json`, which declares `total_pathogens: 91` while `categories.viruses.count` says 41 against an array of 48 entries. Across all categories the config enumerates 101 entries / 98 unique names (3 names appear in both `viruses` and `special_management`). Treat `md/厚労省異種移植指針_91病原体リスト.md` as the authority and re-derive the counts before citing them. The PMDA compliance tests are currently red against this config.

### Critical Pathogens

Immediate alerts are triggered for:
- PERV-A, PERV-B, PERV-C (xenotransplantation critical)
- ASFV, CSFV, FMDV (disease outbreak risks)
- Prion detection

### Reporting Requirements

All reports include:
- Complete 91 pathogen checklist
- PERV-specific analysis section
- Quantitative results (copies/mL)
- Detection confidence levels
- Quality control metrics

## Testing

```bash
# Run unit tests
python -m pytest tests/

# Run integration tests
python tests/test_pipeline_integration.py

# Run PMDA compliance tests
python tests/test_pmda_compliance.py
```

## Monitoring

### CloudWatch Dashboard

Access the dashboard at:
```
https://console.aws.amazon.com/cloudwatch/home?region=ap-northeast-1#dashboards:name=minion-pipeline-production
```

### Metrics

Key metrics tracked:
- Workflow execution time
- Phase completion rates
- Pathogen detection counts
- Resource utilization
- Cost per analysis

### Alerts

Configured alerts:
- PERV detection (CRITICAL)
- High error rate
- Long-running workflows
- Cost threshold exceeded

## Cost Optimization

### Spot Instances
- Basecalling uses GPU spot instances (70% cost reduction)
- Automatic fallback to on-demand if spot unavailable

### Auto-scaling
- EC2 instances auto-terminate after phase completion
- Lambda functions scale automatically

### Data Lifecycle
- Raw data: 30-day retention
- Processed data: 90-day retention
- Reports: 365-day retention

## Troubleshooting

### Common Issues

1. **Basecalling Timeout**
   - Check GPU availability
   - Verify CUDA drivers
   - Increase instance size

2. **High Host Contamination**
   - Review extraction protocol
   - Check depletion efficiency
   - Adjust alignment parameters

3. **Low Pathogen Detection**
   - Verify database integrity
   - Check confidence thresholds
   - Review sample quality

### Logs

Access logs via:
```bash
# CloudWatch logs
aws logs tail /aws/lambda/minion-pipeline-production --follow

# EC2 instance logs
aws ssm start-session --target INSTANCE_ID
```

## Documentation

### 📚 Documentation Portal

**Interactive Documentation Site**: [docs-portal](./docs-portal)

Start the documentation portal:
```bash
cd docs-portal
npm install
npm run dev
# Access: http://localhost:3000
```

Key Pages:
- **Getting Started** - Setup and first workflow
- **Architecture** - System design and AWS infrastructure
- **4-Virus Surveillance** - Real-time monitoring system for Hantavirus, Polyomavirus, Spumavirus, and EEEV
- **PMDA Compliance** - 91 pathogen coverage details
- **API Reference** - Complete API documentation

### Quick References
- **[CLAUDE.md](CLAUDE.md)** - Essential guide for Claude Code development (optimized)
- **[Development Guide](docs/DEVELOPMENT_GUIDE.md)** - Commands, conventions, and patterns
- **[Architecture](docs/ARCHITECTURE.md)** - Pipeline architecture and AWS infrastructure
- **[Protocols Guide](docs/PROTOCOLS_GUIDE.md)** - Sample preparation protocols (11, 12, 13)
- **[4-Virus Surveillance](surveillance/README.md)** - External + internal virus monitoring system
- **[Recent Updates](docs/RECENT_UPDATES.md)** - Latest changes and version history
- **[Technical Details](docs/TECHNICAL_DETAILS.md)** - In-depth technical documentation
- **[Troubleshooting](docs/TROUBLESHOOTING.md)** - Common issues and AWS debugging

### Detailed Documentation
- **[Audit Reports](docs/audits/README.md)** - Code quality audits (9 audits, 37 bugs fixed, zero-bug certification)
- **[Bug Fixes](docs/bug-fixes/README.md)** - Detailed bug fix documentation and analysis
- **[Sprint Reports](docs/sprints/README.md)** - Development sprint tracking and metrics
- **[Development Guide](docs/development/)** - Development workflow, coding standards, and best practices
- **[API Documentation](docs/API_DOCUMENTATION.md)** - REST endpoints and authentication
- **[Deployment Guide](docs/DEPLOYMENT_GUIDE.md)** - AWS infrastructure setup instructions
- **[Session History](docs/claude-sessions/README.md)** - Development session logs
- **[Technical Q&A](docs/technical-qa/)** - Technical question and answer documentation
- **[Grant Application](docs/grants/)** - NVIDIA Academic Grant Program application materials (DGX Spark deployment)

## Funding

**NVIDIA Academic Grant Program Application (2025)**
- Requesting 2× NVIDIA DGX Spark systems for on-premises PMDA-compliant deployment
- Requesting 2,500 A100 GPU hours for ARM vs x86 benchmarking
- See [docs/grants/](docs/grants/) for complete application materials including:
  - 50-sample benchmark results (100% accuracy agreement ARM vs x86)
  - PMDA compliance justification
  - Cost analysis (96.4% operational cost reduction)
  - Educational integration plan (Meiji University)

## Support

- **Documentation**: See `/docs` directory
- **Issues**: GitHub Issues
- **Contact**: support@your-org.com

## License

Proprietary - All rights reserved

## Citations

If using this pipeline, please cite:
- Oxford Nanopore Technologies
- Kraken2 (Wood et al., 2019)
- RVDB (Goodacre et al., 2018)
- PMDA Xenotransplantation Guidelines (2024)

## Version

Current Version: 1.0.0

## Changelog

See [CHANGELOG.md](CHANGELOG.md) for version history.