# CV Extraction from Personal Websites

## Overview

This document describes the methodology for extracting CVs from academic personal websites. The system uses a two-phase approach with intelligent link following.

## Phase 1: Direct CV URLs

**Script:** `scripts/download_cvs_phase1.py`

For students who already have validated CV URLs in the dataset:
1. Load students with `cv_url` field populated
2. Transform cloud storage URLs for direct download:
   - Google Drive: Convert `/view` to `/uc?export=download`
   - Dropbox: Convert `?dl=0` to `?dl=1`
3. Download with async HTTP (5 concurrent requests)
4. Validate PDF header (`%PDF`)
5. Save with consistent naming: `{lastname}_{firstname}_{institution}.pdf`

**Success rate:** ~90% (110/124 in our run)

## Phase 2: Personal Website Extraction

**Script:** `scripts/download_cvs_visible_chrome.py`

For students with personal websites but no direct CV URL:

### Step 1: Load Main Page
```
driver.get(personal_website)
time.sleep(2)  # Wait for JavaScript
```

### Step 2: Search for CV Link on Main Page

Look for links matching these patterns:

**Link Text Patterns (highest confidence):**
- "cv", "c.v.", "curriculum vitae"
- "vita", "vitae"
- "resume", "résumé"
- "download cv", "view cv"

**URL Patterns:**
- Ends with `.pdf` and contains "cv", "vita", "resume"
- Cloud storage URLs (Dropbox, Google Drive, Box)
- Institutional CMS patterns (`/content/dam/`, `/wp-content/uploads/`)

### Step 3: Follow CV-Related Links

If no CV found on main page, scan for internal links that might lead to CV:

```python
CV_PAGE_KEYWORDS = ['cv', 'vita', 'resume', 'about', 'bio', 'research', 'publication']
```

**Link Selection Criteria:**
1. Internal links only (same domain or common hosting like github.io)
2. Link text OR URL contains CV-related keywords
3. Skip social media links (LinkedIn, Twitter, etc.)
4. Visit up to 5 candidate pages

### Step 4: Download and Validate

1. Transform cloud URLs for direct download
2. HTTP GET with browser-like headers
3. Validate response is PDF (`%PDF` header)
4. Save to `data/cv_archive/cvs/`

## File Naming Convention

```
{lastname}_{firstname}_{institution_abbrev}.pdf
```

Examples:
- `smith_john_harvard.pdf`
- `garcia-lopez_maria_stanford.pdf`

Institution abbreviations are standardized (harvard, stanford, mit, etc.)

## Directory Structure

```
data/cv_archive/
├── cvs/                           # Downloaded PDF files
├── cv_download_log.csv            # Phase 1 tracking
├── cv_download_log_chrome.csv     # Phase 2 tracking
└── cv_extraction_failures.csv     # Failed extractions
```

## Log File Schema

```csv
name,institution,personal_website,cv_url,local_path,download_status,download_time,file_size_bytes,error
```

**Status values:**
- `success` - Downloaded and validated as PDF
- `already_exists` - File already downloaded
- `no_cv_found` - No CV link found on page(s)
- `failed` - HTTP error or page load failure
- `invalid_type` - Link found but not a valid PDF

## Detection Strategies (Priority Order)

1. **Link text match (100% confidence)**
   - Text contains "cv", "curriculum vitae", etc.

2. **Cloud storage with CV filename (90%)**
   - Dropbox/Drive/Box URL with "cv" in filename

3. **PDF with CV in path (85%)**
   - URL ends in `.pdf` and path contains "cv", "vita"

4. **Institutional CMS + PDF (75%)**
   - Pattern like `/wp-content/uploads/` + `.pdf`

5. **PDF with faculty name (60%)**
   - PDF filename contains student's last name

## Common Failure Patterns

1. **JavaScript-only sites** - Content loaded dynamically
   - Solution: Use visible Chrome with wait times

2. **CV in non-PDF format** - Google Docs, Word files
   - Currently skipped (could extend to handle)

3. **CV behind authentication** - Faculty portals
   - Cannot access without credentials

4. **No CV posted** - Student hasn't uploaded one
   - Log as `no_cv_found`

## Usage

```bash
# Phase 1: Direct URLs (fast, async)
python scripts/download_cvs_phase1.py

# Phase 2: Personal websites (visible Chrome)
python scripts/download_cvs_visible_chrome.py

# Test mode
python scripts/download_cvs_visible_chrome.py --test

# Limit for testing
python scripts/download_cvs_visible_chrome.py --limit 20
```

## Performance

- **Phase 1:** ~2 it/s (async HTTP)
- **Phase 2:** ~0.25 it/s (Chrome with link following)

Expected total time for 300 students: ~20-25 minutes

## Results Summary

| Phase | Input | Success | No CV | Failed | Invalid |
|-------|-------|---------|-------|--------|---------|
| 1 (Direct URLs) | 124 | 110 | - | 3 | 6 |
| 2 (Websites) | 230+ | TBD | TBD | TBD | TBD |
