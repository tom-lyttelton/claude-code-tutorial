# Academic Web Scraping Architecture

A generalizable two-phase scraping system for collecting data from academic department websites (faculty, PhD students, staff, etc.).

## Overview

This architecture handles the diversity of academic websites through a two-phase approach:
1. **Phase 1 (Fast)**: Async HTTP requests for static HTML sites (~60-70% of sites)
2. **Phase 2 (Thorough)**: Chrome browser for JS-heavy sites + profile enrichment

## When to Use This Architecture

- Scraping people directories from university websites
- Collecting structured data (names, emails, CVs, research interests)
- Handling a mix of static HTML and JavaScript-rendered sites
- Need for resume capability for long-running scrapes

## Package Structure

```
scripts/
├── {entity}_scraper/              # e.g., phd_scraper/, faculty_scraper/
│   ├── __init__.py                # Package exports
│   ├── config.py                  # Configuration and YAML loading
│   ├── models.py                  # Data models (Person dataclass)
│   ├── validators.py              # Name, email, CV/document validation
│   ├── extractors.py              # HTML parsing and data extraction
│   ├── async_scraper.py           # Phase 1: aiohttp-based
│   ├── chrome_scraper.py          # Phase 2: parallel Chrome workers
│   ├── pipeline.py                # Main orchestration
│   ├── progress.py                # Checkpointing and resume
│   └── deduplication.py           # Cross-phase deduplication
├── run_{entity}_scraper.py        # CLI entry point
└── {entity}_scraper_config.yaml   # Configuration file
```

## Core Components

### 1. Data Model (`models.py`)

Define a dataclass for the entity being scraped:

```python
@dataclass
class Person:
    # Required fields
    name: str
    institution: str
    department: str

    # Contact
    email: Optional[str] = None
    profile_url: Optional[str] = None

    # Documents/Links
    cv_url: Optional[str] = None
    personal_website: Optional[str] = None
    google_scholar_url: Optional[str] = None
    linkedin_url: Optional[str] = None

    # Role-specific (customize per entity)
    title: Optional[str] = None           # Faculty: "Associate Professor"
    year_entered: Optional[int] = None    # Students: cohort year
    research_interests: Optional[str] = None

    # Metadata
    country: str = "US"
    source_url: Optional[str] = None
    scraped_at: Optional[str] = None
    scrape_phase: int = 1

    def completeness_score(self) -> int:
        """For deduplication - higher = more complete record"""
        score = 0
        if self.email: score += 10
        if self.cv_url: score += 8
        # ... etc
        return score
```

### 2. Validators (`validators.py`)

Key validation functions:

```python
def validate_name(name: str) -> Tuple[bool, str]:
    """Reject navigation elements, titles, field names"""
    # Length check (4-60 chars)
    # Letter ratio check (>85%)
    # Garbage pattern rejection
    # Must have uppercase, 2-5 words

def validate_email(email: str) -> bool:
    """Reject generic/department emails"""
    # Format validation
    # Reject: info@, admin@, department@, etc.

def is_valid_document_url(url: str) -> bool:
    """Validate CV/resume URLs"""
    # Accept: .pdf, .doc, .docx
    # Accept: /cv/, /resume/, /vita/ paths
    # Reject: /curriculum/, /courses/, /academics/
```

### 3. Extractors (`extractors.py`)

HTML parsing functions:

```python
def extract_students_from_html(html, base_url, department, ...) -> List[Person]:
    """Main extraction function"""
    # 1. Find main content, remove nav/footer
    # 2. Build page-wide email map
    # 3. Try priority selectors (specific to general)
    # 4. Fallback to tables, lists, headings
    # 5. Last resort: profile-like links

# Priority selectors (ordered specific to general):
PRIORITY_SELECTORS = [
    '.person-card', '.profile-card', '.student-card',
    '.directory-item', '.people-item',
    'article.person', '.views-row .person',
    # ... etc
]
```

### 4. Phase 1: Async Scraper (`async_scraper.py`)

```python
class AsyncScraper:
    async def fetch_page(self, session, url) -> Tuple[html, error, time]:
        """Fetch with retries and timeout"""

    def detect_js_heavy(self, html, student_count, url) -> bool:
        """Identify sites needing Chrome"""
        # Known JS-heavy sites list
        # JS framework markers (react, angular, vue)
        # Low student count + JS markers

    async def scrape_all(self, departments) -> Tuple[students, needs_chrome, stats]:
        """Parallel scraping with semaphore for concurrency"""
```

### 5. Phase 2: Chrome Scraper (`chrome_scraper.py`)

```python
class ChromeWorker:
    """Single browser instance"""
    def fetch_page(self, url) -> Optional[str]
    def scrape_department(self, dept) -> ScrapeResult
    def scrape_profile(self, person) -> Person  # Enrichment

class ChromeScraper:
    """Pool of workers with ThreadPoolExecutor"""
    def scrape_departments(self, depts) -> Tuple[students, stats]
    def enrich_profiles(self, students) -> List[Person]
```

### 6. Progress Tracking (`progress.py`)

```python
class ProgressTracker:
    def save_checkpoint(self, students, departments, phase)
    def load_checkpoint(self) -> Optional[Dict]
    def get_remaining_departments(self, all_depts, phase) -> List
```

### 7. Deduplication (`deduplication.py`)

```python
def deduplicate(students, merge_records=True) -> Tuple[deduped, num_removed]:
    """
    Key: (normalized_name, institution)
    Keep record with highest completeness_score
    Optionally merge fields from duplicates
    """
```

## Configuration

```yaml
# {entity}_scraper_config.yaml

disciplines:
  - sociology
  - management

input_files:
  sociology: data/input/sociology_depts.csv
  management: data/input/management_programs.csv

phase1:
  max_concurrent: 10
  timeout_seconds: 30
  min_threshold: 3        # Min results to consider success
  retry_count: 2

phase2:
  num_workers: 3
  headless: true
  page_timeout: 60
  scrape_profiles: true   # Visit individual profile pages

validation:
  strict_document_validation: true
  validate_emails: true

chrome_required_sites:    # Always use Phase 2 for these
  - yale.edu
  - columbia.edu
  - umich.edu
```

## Input File Format

CSV with required columns:
- `institution` or `university`
- `department` or `department_name`
- `grad_students_url` or `faculty_url` (the page to scrape)
- `country` (optional, defaults to "US")

## CLI Interface

```bash
# Full pipeline
python run_{entity}_scraper.py --all

# Phase 1 only (fast)
python run_{entity}_scraper.py --phase1

# Phase 2 only (requires Phase 1 first)
python run_{entity}_scraper.py --phase2

# Single discipline
python run_{entity}_scraper.py --discipline sociology

# Test mode
python run_{entity}_scraper.py --test --limit 5

# Resume interrupted run
python run_{entity}_scraper.py --resume

# Verify results
python run_{entity}_scraper.py --verify

# Adjust parallelism
python run_{entity}_scraper.py --concurrent 15 --workers 5
```

## Output Structure

```
data/processed/{entity}/
├── {entity}_phase1.csv         # Phase 1 results
├── {entity}_phase2.csv         # Phase 2 results
├── {entity}_combined.csv       # Merged + deduplicated
├── needs_chrome_scrape.json    # Departments for Phase 2
└── scrape_progress.json        # Checkpoint file

data/processed/
├── {entity}_sociology.csv      # By discipline
├── {entity}_management.csv
└── {entity}_polisci.csv
```

## Adapting for Different Entities

### For Faculty Scraping

1. **Model changes**: Add `title`, `rank`, `tenure_status`, remove `year_entered`, `cohort`
2. **Selector changes**: Prioritize `.faculty-card`, `.professor-profile`
3. **Validation**: May want to allow "Professor" in names (as part of full name)
4. **URL patterns**: `/people/faculty`, `/faculty/`, `/our-faculty/`

### For Staff Scraping

1. **Model changes**: Add `position`, `department_role`
2. **Simpler extraction**: Usually just name, title, email, phone
3. **Different page structure**: Often in tables or simple lists

## Dependencies

```bash
# Required (Phase 1)
pip install aiohttp beautifulsoup4 lxml pyyaml tqdm

# Optional (Phase 2 - Chrome scraping)
pip install undetected-chromedriver selenium
```

## Best Practices

1. **Start with Phase 1 only** to identify problematic sites
2. **Use test mode** (`--limit 5`) when developing
3. **Check `needs_chrome_scrape.json`** to tune JS-heavy site detection
4. **Monitor coverage rates** with `--verify`
5. **Save checkpoints frequently** - scraping can be interrupted
6. **Add sites to `chrome_required_sites`** as you discover them
