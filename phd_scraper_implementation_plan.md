# PhD Student Scraping System - Implementation Plan

## Overview

A comprehensive, two-phase scraping system for collecting PhD students from sociology, management, and political science departments at US/Canadian institutions.

## Target Data Fields
- name, email, cohort/year matriculated, research interests
- CV (PDF/Word validated), personal website, Google Scholar, LinkedIn

## Two-Phase Architecture

### Phase 1: Fast Async Scraping (aiohttp)
- Parallel HTTP requests (10 concurrent by default)
- Handles ~60-70% of departments (static HTML sites)
- Marks JS-heavy sites for Phase 2
- Output: `phd_students_phase1.csv` + `needs_chrome_scrape.json`

### Phase 2: Parallel Chrome Scraping
- Multiple Chrome instances (3 workers by default)
- Handles JS-heavy sites (Yale, Columbia, Michigan, etc.)
- Profile enrichment visits for CV/social links
- Output: `phd_students_phase2.csv`

## Script Architecture

```
scripts/
├── phd_scraper/                    # Main package
│   ├── __init__.py
│   ├── config.py                   # Configuration and YAML loading
│   ├── models.py                   # PhDStudent dataclass
│   ├── validators.py               # Name, email, CV validation (strict)
│   ├── extractors.py               # All extraction functions (consolidated)
│   ├── async_scraper.py            # Phase 1: aiohttp-based
│   ├── chrome_scraper.py           # Phase 2: parallel Chrome
│   ├── pipeline.py                 # Main orchestration
│   ├── progress.py                 # Progress tracking/resume
│   └── deduplication.py            # Cross-phase deduplication
├── run_phd_scraper.py              # CLI entry point
└── phd_scraper_config.yaml         # Configuration file
```

## Key Improvements Over Previous Scripts

### 1. Parallelization
- **Phase 1**: asyncio + aiohttp for 10x concurrent requests
- **Phase 2**: ThreadPoolExecutor with 3 Chrome instances

### 2. Stricter CV Validation
```python
def is_valid_cv_url(url: str) -> bool:
    """CV must be PDF/Word or have explicit /cv/ or /resume/ path"""
    url_lower = url.lower()
    is_document = url_lower.endswith('.pdf') or url_lower.endswith('.doc') or url_lower.endswith('.docx')
    has_cv_path = any(p in url_lower for p in ['/cv/', '_cv.', '-cv.', 'vita.pdf', '/resume/', '_resume.'])
    # Reject curriculum/courses pages
    is_curriculum = any(p in url_lower for p in ['/curriculum/', '/courses/', '/academics/'])
    return (is_document or has_cv_path) and not is_curriculum
```

### 3. Better Name Validation
- Expanded garbage patterns (rejects "Academic Experience", "Political Economics", etc.)
- Exact match rejection for common false positives

### 4. Deduplication
- Primary key: (normalized_name, institution)
- Completeness scoring to keep best record
- Merge fields from multiple scrapes

### 5. Progress Tracking
- JSON checkpoint after each department
- `--resume` flag to continue from last checkpoint
- Detailed scrape log

## CLI Usage

```bash
# Full pipeline (phase1 + phase2 + enrichment)
python run_phd_scraper.py --all

# Phase 1 only (fast async)
python run_phd_scraper.py --phase1

# Phase 2 only (Chrome for failed depts)
python run_phd_scraper.py --phase2

# Single discipline
python run_phd_scraper.py --discipline sociology

# Test mode (5 departments)
python run_phd_scraper.py --test --limit 5

# Resume from checkpoint
python run_phd_scraper.py --resume

# Adjust parallelism
python run_phd_scraper.py --all --workers 5 --concurrent 15
```

## Output Files

```
data/processed/
├── phd_students/
│   ├── phd_students_phase1.csv      # Phase 1 results
│   ├── phd_students_phase2.csv      # Phase 2 results
│   ├── phd_students_combined.csv    # Merged + deduplicated
│   ├── needs_chrome_scrape.json     # Departments needing Phase 2
│   └── scrape_progress.json         # Checkpoint file
├── phd_students_sociology.csv       # Final by discipline
├── phd_students_management.csv
└── phd_students_polisci.csv
```

## Configuration (phd_scraper_config.yaml)

```yaml
disciplines:
  - sociology
  - management
  - political_science

input_files:
  sociology: data/input/phd_granting_sociology_depts_v2.csv
  management: data/input/management_phd_programs_v4.csv
  political_science: data/raw/departments/political_science_departments.csv

phase1:
  max_concurrent: 10
  timeout_seconds: 30
  min_students_threshold: 3

phase2:
  num_workers: 3
  headless: true
  page_timeout: 60

validation:
  strict_cv_validation: true
  validate_emails: true
```

## Department Input Files

| Discipline | Programs | File | Status |
|------------|----------|------|--------|
| Sociology | 73 | `phd_granting_sociology_depts_v2.csv` | Ready |
| Management | 65 | `management_phd_programs_v4.csv` | Ready |
| Political Science | 103 | `political_science_departments.csv` | Ready |
| **Total** | **241** | | |

## Quality Targets

| Metric | Target |
|--------|--------|
| Sociology students | 800+ |
| Management students | 200+ |
| Political Science students | 600+ |
| Email coverage | >60% |
| Valid CV links | >15% |
| Year/cohort info | >40% |
| Duplicates | <2% |

## Verification

Run `python scripts/run_phd_scraper.py --verify` to check:
- Total counts per discipline
- Coverage rates for each field
- Duplicate detection
- Sample validation
