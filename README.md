# AI Ad trend monitor System (n8n POC)

## 1. Overview

This Proof of Concept (POC) demonstrates a modular, production-oriented automation system built with **n8n**.

The system scrapes advertising data from a target website, analyzes image content using AI services, identifies weekly trends, and generates new AI-based Guardio ads inspired by those trends.

All main and sub-workflows share global context parameters and interact through Google Sheets, creating a traceable, data-driven pipeline.

---

## 2. Installation

1. Import all six workflow JSON files into your n8n instance.
2. Create the required credentials:
   - **Google Sheets** – OAuth2 for read/write access.
   - **Google Drive** – for future integration of generated ads.
   - **Imagga API** – for image analysis (100 free API calls per month).
   - **Hugging Face API** – for AI image generation (free tier).
   - **OpenAI API** – for text prompt generation (optional; can use local Ollama models instead).
   - **SMTP** – for email notifications (Gmail, Outlook, or other providers).
3. Create a Google Spreadsheet for the ads data and extract its ID.
4. Update all `Context params` nodes across workflows with:
   - `SPREADSHEET_ID`
   - `email_from`
   - `email_to`
   - and any additional context parameters listed below.
5. Create a dedicated **Google Drive** folder for generated ads and record its ID in the `DRIVE_FOLDER_ID` context parameter (for reference only - currently not in use, choose folder manually within ‘Upload file’ node in ‘SUB Generate Guardio Ads’ workflow).
6. Replace all hard-coded self-hosted URLs (`http://localhost:5678/`) inside email and error handler nodes with your deployed n8n base URL.

---

## 3. Required Context Parameters

### Daily Scraper

- `BASE_URL` – The base URL of the site to scrape (currently hard-coded for Ynet URL).
- `MAX_PAGES` – Number of pages to scan.
- `AD_HOSTS` – Known ad host domains.
- `AD_SELECTORS` – CSS selectors to locate ads.
- `RATE_LIMIT_SEC` – Delay between page fetches.
- `USER_AGENT` – Browser header for HTTP requests.
- `SPREADSHEET_ID` – Google Sheet for storing results.
- `email_from` / `email_to` – Notification settings.

### Weekly Trends

- `SPREADSHEET_ID`
- `week_start` / `week_end`
- `email_from` / `email_to`

### Generate Guardio Ads

- `DRIVE_FOLDER_ID` – Google Drive folder for output images.
- `RATE_LIMIT_SEC` – Delay between API calls.
- `ai_model` – AI model used to create prompts.

---

## 4. Daily Flow

### 4.1 MAIN – Daily Scraper

**Purpose:**

Collect new ads, analyze their content, and update analytical datasets.

**Flow:**

1. A daily schedule trigger starts the workflow.
2. Context parameters are loaded.
3. Calls `SUB Scrape Ads` to extract new ads from `BASE_URL`.
4. New ads are filtered by hash to avoid duplicates.
5. For each new image, `SUB Analyze Image` is executed.
6. Analysis results are appended to the `image_analysis` sheet.
7. The status of each run is logged in the `execution_logs` sheet:
   - `new` → `scraping` → `analyzing` → `success`.
8. Sends a summary email upon completion.
9. If any error occurs, the `Error Handler` workflow is triggered.

---

### 4.2 SUB – Scrape Ads

**Purpose:**

Extract or simulate ads from a given site.

**Logic:**

- Performs an HTTP GET to `BASE_URL`.
- Uses `AD_SELECTORS` and `AD_HOSTS` to identify ads.
- Follows pagination up to `MAX_PAGES`, respecting `RATE_LIMIT_SEC`.
- Articles extraction currently hard-coded for Ynet pattern: /article/xxxxx.
- Since scraping is currently blocked (e.g., by JavaScript rendering or paywall), the flow inserts three manually defined **stub ads** to simulate data for the POC.
- Calculates a hash for each image to detect duplicates.
- Returns only new ads to the main workflow.

This sub-flow is fully dynamic: changing the `BASE_URL` in context parameters allows it to scrape any website with similar ad structures.

---

### 4.3 SUB – Analyze Image

**Purpose:**

Perform visual analysis using the **Imagga API**.

**Logic:**

- Makes four parallel API requests:
  1. `/colors` – Extracts dominant colors (`overall_count=5`), though the API sometimes returns only two colors. A normalization step should be added to consistently keep the top three for accurate trend analysis.
  2. `/tags` – Detects objects and characters.
  3. `/text` – Recognizes embedded text and CTAs.
  4. `/faces` – Detects human faces.
- n8n executes these nodes top-to-bottom, so they run one after the other.
- In production, a `TIME_RATE` node will be applied to balance concurrency and prevent API throttling.
- Results are merged and stored in the `image_analysis` sheet.

---

## 5. Weekly Flow

### 5.1 MAIN – Weekly Trends

**Purpose:**

Aggregate analytics, detect visual trends, and generate new ads.

**Flow:**

1. Triggered weekly (e.g., every Sunday at 10:00 AM).
2. Loads the date range from `week_start` to `week_end`.
3. Reads image data from `image_analysis`.
4. Identifies top colors, objects, and characters of the week.
5. Updates the `weekly_trends` sheet.
6. If new trends exist, calls `SUB Generate Guardio Ads`.
7. Logs execution status to `execution_logs`:
   - `new` → `in_progress` → `generating_ads` → `success`
   - or `business_error` if no new data was found.
8. Sends a weekly summary email, including preview links of generated ads.
9. On any failure, triggers the `Error Handler` workflow.

---

### 5.2 SUB – Generate Guardio Ads

**Purpose:**

Create new ad creatives using AI.

**Logic:**

- Builds ad prompts based on aggregated weekly data.
- Generates the textual prompt using **OpenAI**, chosen for its high prompt quality (and availability of remaining tokens).
- Sends the prompt to **Hugging Face (FLUX.1-schnell)** to generate ad images.
- Uploads generated ads to Google Drive (future integration via AI Agent + MCP Google Drive tool).
- Saves metadata to the `generated_ads` sheet:
  - Model used, colors, timestamp, file ID, and Drive link.

**Note:**

AI Agent + MCP Google Drive integration was tested but excluded due to instability and excessive token consumption. It remains a planned future enhancement.

---

## 6. Error Handler

**Purpose:**

Centralized failure management.

**Logic:**

- Triggered automatically on any workflow error.
- Sends an HTML email containing:
  - Workflow name
  - Run ID
  - Timestamp
  - Error message
  - Direct execution link (base N8N URL currently hard-coded).
- Updates the `execution_logs` sheet with a failure record.
- Uses a red HTML theme for immediate visibility.

---

## 7. System Logic and Design Rationale

- **Unified Context Parameters:**
  The project uses shared context parameters across all workflows because the self-hosted free version of n8n does not support environment variables.
  This ensures consistent configuration and simplifies future migration to `.env`-based setups.
- **Modular Sub-Workflows:**
  Each main workflow delegates tasks to smaller sub-workflows. This approach provides reusability, simpler debugging, and scalability.
  For example, `SUB Analyze Image` could later serve an external API endpoint without modification.
- **Sequential-Parallel API Calls:**
  Imagga endpoints are logically parallel, allowing faster analysis while respecting rate limits.
- **POC Simulations:**
  Because some ad sites block scraping, stub ads (images) were manually inserted to simulate realistic runs.
- **Execution Logging:**
  The `execution_logs` sheet records all main workflow runs.
  The sheet includes columns such as `run_id`, `workflow_name`, `status`, `timestamp`, and `notes`, allowing easy tracking of daily and weekly executions, and updated throughout the flow.
  Future versions will **separate daily and weekly** logs for better readability and reduced redundancy.

---

## 8. Improvements and Future Work

- Refactor the "Update Status" node to avoid multiple executions when several new ads are found (currently runs X times, wile X is the amount of new ads found - doesn’t affect usability, but inefficient).
- Separate the `execution_logs` for daily and weekly runs.
- Add normalization for Imagga color analysis to ensure consistent top-3 color extraction across all images.
- Replace hard-coded localhost URLs with dynamic environment variables.
- Implement AI Agent → MCP Drive integration for direct upload of generated ads.
- Consider using trending ads text to generate text for generated ads.
- Dynamic article extraction (current extraction currently hard-coded for Ynet pattern: /article/xxxxx).
- Introduce a `TIME_RATE` limiter for production stability to Imagga APIs.
- Add caching and hash comparison logic to reduce redundant analyses.
- Integrate a dashboard (Grafana or Metabase) for visualization of trends and stats.
- Replace Google Sheets with PostgreSQL (or other DB) for production.
- Match email HTML templates workflow name at the bottom (some have background while other doesn’t).

---

## 9. Included Files in REPO

- **6 JSON Workflows**
  - `MAIN Daily Scraper.json`
  - `SUB Scrape Ads.json`
  - `SUB Analyze Image.json`
  - `MAIN Weekly Trends.json`
  - `SUB Generate Guardio Ads.json`
  - `Error Handler.json`
- **4 Email Examples**
  - `Daily Workflow Completed – Success.eml`
  - `Weekly Trends Completed – 3 Ads Generated.eml`
  - `Weekly Trends – No Ads Found (Business Error).eml`
  - `MAIN Weekly Trends Workflow Failed – Run 733.eml`
- **1 Spreadsheet Example**
  - `Ads Intelligence DB – spreadsheet example.xlsx`
    - Includes sample daily and weekly run data.
    - Some rows contain partial or pinned data due to interrupted test runs (left intentionally to demonstrate different statuses and intermediate workflow states).
    - Provided for structural demonstration only.
- **3 Generated Ad Samples**
  - `guardio-ad-757-v1.png`
  - `guardio-ad-757-v2.png`
  - `guardio-ad-757-v3.png`
- **1 Architecture Diagram**
  - `Flowchart.png`
- **1 Documentation File**
  - `Ads Intelligence System – README.md`

---
