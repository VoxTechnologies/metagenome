# CLAUDE_REFERENCE.md

Detailed reference documentation moved from CLAUDE.md for performance optimization. Start at [CLAUDE.md](CLAUDE.md); come here only for the full directory tree, the PERV marker table, and the as-designed AWS detail.

> **Everything below describes the design as specified, not a system that ran.** This repo was archived 2026-05-10 and never deployed to AWS — see the status section of [CLAUDE.md](CLAUDE.md). Instance types, cost percentages, scaling strategy, and the version history in "Recent Implementation History" are design intent and changelog entries, never measured behaviour. The debug commands at the bottom target AWS resources that do not exist.

## Architecture Overview

**7-Phase Pipeline**: phase0_sample_prep → phase1_basecalling → phase2_qc → phase3_host_removal → phase4_pathogen → phase5_quantification → phase6_reports

**AWS Pattern (Containerless)**:
- S3 upload → Lambda orchestrator → Step Functions → EC2 per phase (custom AMIs, auto-terminate)
- No Docker containers - uses custom AMIs with pre-installed tools
- Phase 1: g4dn.xlarge (GPU for Dorado basecalling)
- Phase 3: r5.4xlarge (128GB RAM for host removal)
- Phase 4: 4x parallel EC2 (Kraken2/BLAST pathogen detection)
- Spot Instances: 70% cost savings
- RDS: Aurora Serverless v2 for pipeline metadata

## Complete Directory Structure

```
scripts/
├── phase0_sample_prep/        # Sample routing & workflow determination
├── phase1_basecalling/        # FAST5→FASTQ (Dorado duplex)
├── phase2_qc/                 # Quality metrics (NanoPlot/PycoQC)
│                              # RNA integrity (RIN) for RNA viruses
├── phase3_host_removal/       # Host depletion (Minimap2)
│                              # CpG methylation (DNA), rRNA depletion (RNA)
├── phase4_pathogen/           # Multi-DB screening (Kraken2/BLAST)
│   ├── perv_typing.py         # CRITICAL: PERV-A/B/C subtype detection
│   ├── detect_4viruses.py     # Hantavirus/Polyomavirus/Spumavirus/EEEV
│   ├── detect_pmda_4viruses.py # PMDA-specific 4-virus detection
│   └── detect_pmda_all_91_pathogens.py  # Full 91 pathogen coverage
├── phase5_quantification/     # Abundance calculation (copies/mL)
└── phase6_reports/            # PMDA-compliant report generation

lambda/
├── orchestration/             # Pipeline coordination (S3 trigger)
├── monitoring/                # EC2 lifecycle management
├── phases/                    # Phase-specific handlers
└── shared/                    # Common validation logic

surveillance/                  # 4-Virus Surveillance System (v2.2.0)
├── external/                  # External information collectors
│   ├── estat_client.py        # E-Stat API (APP_ID via E_STAT_APP_ID env var)
│   ├── maff_scraper.py        # MAFF surveillance reports
│   └── academic_monitor.py    # PubMed + J-STAGE web scraping
├── internal/                  # Internal pipeline integration
│   └── pipeline_listener.py   # Phase 4 result monitoring
├── alerting/                  # Notification system
│   ├── severity_engine.py     # 4-level classification (CRITICAL/HIGH/MEDIUM/LOW)
│   ├── notification_router.py # Multi-channel alerts
│   └── slack_client.py        # Slack Bot API + Webhooks (NEW v2.2.0)
├── dashboard/                 # Streamlit UI (port 8501)
│   └── app.py                # Real-time monitoring, 30s auto-refresh
├── api/                       # FastAPI REST API (port 8000)
│   └── main.py               # /api/v1/detections, /api/v1/alerts
├── lambda/                    # Lambda functions
│   ├── external_collector/    # Daily @ 11:00 JST (cron: 0 2 * * ? *)
│   └── pipeline_listener/     # Real-time S3 events
├── config/
│   ├── severity_rules.yaml   # Severity classification rules
│   └── config.yaml           # System configuration, Slack settings
├── tests/
│   └── test_slack_integration.py  # Slack notification tests (NEW v2.2.0)
├── scripts/
│   └── setup_lambda_env.sh   # Lambda deployment automation (NEW v2.2.0)
├── docs/
│   └── SLACK_SETUP.md        # Slack integration guide (NEW v2.2.0)
└── .env.template              # Environment variables template (NEW v2.2.0)

tools/
├── workflow_cli.py            # Main CLI (start/status/metrics)
├── database_setup.sh          # Reference database initialization
└── monitoring_dashboard.py    # Streamlit dashboard

infrastructure/
├── terraform/                 # IaC for AWS resources (never applied; no state file)
├── database/                  # schema.sql, seed_data.sql
└── surveillance/              # DynamoDB schemas + surveillance terraform

ec2_setup/                     # AMI build + tool install scripts (Dorado, Kraken2, BLAST, ...)
md/                            # Original Japanese design specs (91-pathogen list, PMDA assessment)
docs-portal/                   # Next.js 15 / React 19 documentation site (separate toolchain)
```

Verified 2026-07-27: `infrastructure/cloudformation/` and `infrastructure/custom_amis/` do not exist — AMI definitions live in `ec2_setup/`, and there are no CloudFormation templates.

## Extended Code Patterns

### Complete File Validation Pattern
```python
from pathlib import Path
import logging

logger = logging.getLogger(__name__)

def process_file(file_path: str) -> Path:
    """Validate file exists and is not empty."""
    file = Path(file_path)

    if not file.exists():
        logger.error(f"File not found: {file}")
        raise FileNotFoundError(f"File not found: {file}")

    if file.stat().st_size == 0:
        logger.error(f"Empty file: {file}")
        raise ValueError(f"Empty file: {file}")

    logger.info(f"Processing file: {file} ({file.stat().st_size} bytes)")
    return file
```

### PERV Detection Pattern (Extended)
```python
# From perv_typing.py:14-31
PERV_MARKERS = {
    "PERV-A": {
        "env_motif": "KETQKERQKR",
        "gag_region": "MGNLTVD",
        "pol_signature": "YIDDILLI"
    },
    "PERV-B": {
        "env_motif": "KEKMKERQKR",
        "gag_region": "MGNLTVD",
        "pol_signature": "YIDDILLI"
    },
    "PERV-C": {
        "env_motif": "KGKGKERQKR",
        "gag_region": "MGNLTVE",
        "pol_signature": "YIDDILLI"
    }
}

# Any detection triggers immediate alert
if any(perv_detected.values()):
    send_sns_alert(perv_subtype, copies_per_ml)
```

### Test Pattern with Moto
```python
import pytest
from moto import mock_s3, mock_lambda, mock_dynamodb

@mock_s3
@mock_dynamodb
def test_pipeline_integration():
    # AWS services are mocked
    pass
```

## Configuration Details

### Key Configuration Files
- `templates/config/pmda_pathogens.json` - 91 pathogen definitions, Protocol 12 v2.1 specs
- `lambda/orchestration/pipeline_orchestrator.py` - Main orchestration logic
- `scripts/phase4_pathogen/perv_typing.py` - CRITICAL PERV detection
- `surveillance/config/severity_rules.yaml` - 4-level severity classification
- `surveillance/config/config.yaml` - System configuration, Slack integration
- `infrastructure/terraform/*.tf` - AWS infrastructure as code

### Database Locations (Complete)
```
/mnt/efs/databases/
├── kraken2/
│   └── pmda_2024/           # PMDA pathogen database
├── blast/
│   └── nt_2024/            # NCBI nucleotide database
├── perv/
│   └── references/          # PERV-specific references
└── pmda/
    └── 2024.1/             # Custom PMDA database
```

## Implementation Notes

### Error Handling Requirements
- Always validate file existence/size before processing
- Use proper logging with appropriate levels
- Implement retry logic for AWS operations
- Handle rate limiting for external APIs

### Performance Considerations
- EC2 instances auto-terminate after completion
- Use Spot Instances for 70% cost savings
- Parallel processing in Phase 4 (4x EC2)
- 30-second auto-refresh for dashboards

### Security Requirements
- No patient data in git repositories
- All data encrypted in S3
- Use AWS Secrets Manager for credentials
- Environment variables for Slack tokens
- Least privilege IAM roles

## Recent Implementation History

### v2.2.0 - Slack Integration (2025-11-15)
- Implemented `surveillance/alerting/slack_client.py`
- Dual delivery: Bot API + Webhooks
- Rich Block Kit formatting
- Channel routing: #critical-alerts, #pathogen-alerts, #pathogen-monitoring
- Interactive action buttons for critical alerts
- Daily summary notifications
- App ID: A09TVLTGDSL
- Required scopes: `chat:write`, `chat:write.public`, `channels:read`

### v2.1.0 - 4-Virus Surveillance (2025-11-14)
- Target viruses: Hantavirus, Polyomavirus, Spumavirus, EEEV
- External collectors: MAFF, E-Stat, PubMed, J-STAGE
- Internal monitoring: Real-time Phase 4 results
- Cost: ~$7.40/month (Lambda + DynamoDB + S3)
- 3 DynamoDB tables, EventBridge scheduling

### v2.0.0 - Protocol 12 v2.1 (2025-11-13)
- Added Step 2.5 for circular/ssDNA viruses
- Support for PCV2, PCV3, TTV, PPV
- Achieved TRUE 91/91 pathogen coverage

## AWS Infrastructure Summary

### Core Services
- **Compute**: Lambda (orchestration), EC2 (processing), Step Functions (workflow)
- **Storage**: S3 (data), EFS (databases), RDS Aurora (metadata)
- **Messaging**: SNS (alerts), SES (emails), EventBridge (scheduling)
- **Monitoring**: CloudWatch (logs/metrics), X-Ray (tracing)

### Cost Optimization
- Spot Instances: 70% EC2 cost reduction
- Aurora Serverless: Pay-per-use database
- S3 Intelligent Tiering: Automatic archival
- Lambda: Pay only for execution time

### Scaling Strategy
- Horizontal scaling via parallel EC2 instances
- Auto-scaling Lambda concurrency
- DynamoDB on-demand billing
- EFS burst credits for database reads

## Testing Strategy

### Test Coverage Requirements (as specified, not achieved)
- Unit tests: >80% coverage
- Integration tests: All phase transitions
- E2E tests: Complete pipeline runs
- Performance tests: Throughput validation

Actual state measured 2026-07-27: 22 failed / 8 passed, plus 3 modules failing collection on missing `boto3`. `tests/performance/` does not exist, so no benchmark suite was ever written.

### Test Execution
```bash
# Local testing
pytest tests/ --cov=scripts --cov=lambda

# Slack integration testing
python surveillance/tests/test_slack_integration.py --test-all
```

## Support and Troubleshooting

### Common Issues
1. **PERV False Positives**: Check `perv_typing.py` motif thresholds
2. **Slack Not Sending**: Verify SLACK_BOT_TOKEN environment variable
3. **Pipeline Timeout**: Check EC2 instance status in CloudWatch
4. **Database Not Found**: Verify EFS mount point accessibility

### Debug Commands
```bash
# Check Lambda logs
aws logs tail /aws/lambda/pipeline-orchestrator --follow

# Monitor EC2 instances
aws ec2 describe-instances --filters "Name=tag:Pipeline,Values=MinION"

# Verify S3 uploads
aws s3 ls s3://minion-data/phase0/ --recursive

# Test Slack connection
python surveillance/tests/test_slack_integration.py --test-conn
```

### Contact Information
- Technical issues: See docs/TROUBLESHOOTING.md
- Slack setup: surveillance/docs/SLACK_SETUP.md
- Architecture questions: docs/ARCHITECTURE.md
- Protocol questions: docs/PROTOCOLS_GUIDE.md

---

*This reference document contains detailed information moved from CLAUDE.md for performance optimization. For quick access to essential information, see CLAUDE.md.*