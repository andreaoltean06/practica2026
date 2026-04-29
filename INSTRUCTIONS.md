# INSTRUCTIONS.md - Web Scraper Instructions

## General Guidelines for Building a Job Web Scraper

This document provides instructions for creating a web scraper for job listings using **Node.js** and **JavaScript**, conforming to the [Job Model](MODEL.md#job-model-schema) and [Company Model](MODEL.md#company-model-schema).

---

## 1. Project Structure

```
scraper/
├── package.json
├── index.js            # Main entry point
├── config.js           # Configuration (URLs, selectors, etc.)
├── scrapers/
│   └── [company].js    # Company-specific scraper
├── parsers/
│   └── jobParser.js    # Job data parser/validator
├── utils/
│   └── helpers.js      # Utility functions
└── output/
    └── jobs.json       # Scraped jobs output
```

---

## 2. Dependencies

Recommended packages:

```json
{
  "dependencies": {
    "axios": "^1.6.0",
    "cheerio": "^1.0.0-rc.12",
    "puppeteer": "^22.0.0"
  }
}
```

- **axios** + **cheerio**: For static HTML pages
- **puppeteer**: For dynamic/JavaScript-rendered pages

---

## 3. Scraper Implementation

### 3.1. Main Entry Point (`index.js`)

```javascript
const { scrapeJobs } = require('./scrapers/[company]');
const { validateJob } = require('./parsers/jobParser');

async function main() {
  const jobs = await scrapeJobs();
  
  const validatedJobs = jobs
    .map(validateJob)
    .filter(job => job !== null);
  
  console.log(JSON.stringify(validatedJobs, null, 2));
}

main().catch(console.error);
```

### 3.2. Scraper Module

```javascript
const axios = require('axios');
const cheerio = require('cheerio');

async function scrapeJobs() {
  const { data: html } = await axios.get('https://company-careers.com/jobs');
  const $ = cheerio.load(html);
  
  const jobs = [];
  
  $('.job-listing').each((_, el) => {
    const url = $(el).find('a').attr('href');
    const title = $(el).find('.job-title').text().trim();
    const location = $(el).find('.location').text().trim();
    
    jobs.push({ url, title, location });
  });
  
  return jobs;
}

module.exports = { scrapeJobs };
```

### 3.3. Parser/Validator (`parsers/jobParser.js`)

```javascript
function validateJob(job) {
  // Required fields
  if (!job.url || !job.title) {
    return null;
  }
  
  // URL validation
  try {
    new URL(job.url);
  } catch {
    return null;
  }
  
  // Title max length
  if (job.title.length > 200) {
    return null;
  }
  
  return {
    url: job.url.trim(),
    title: job.title.trim(),
    company: job.company ? job.company.toUpperCase() : undefined,
    cif: job.cif,
    location: Array.isArray(job.location) ? job.location : [job.location].filter(Boolean),
    tags: job.tags ? job.tags.map(t => t.toLowerCase()).slice(0, 20) : [],
    workmode: ['remote', 'on-site', 'hybrid'].includes(job.workmode) ? job.workmode : undefined,
    date: new Date().toISOString(),
    status: 'scraped',
    salary: job.salary
  };
}

module.exports = { validateJob };
```

---

## 4. Job Model Fields

| Field          | Type     | Required | Rules |
|----------------|----------|----------|-------|
| url            | string   | Yes      | Valid HTTP/HTTPS URL, canonical, unique |
| title          | string   | Yes      | Max 200 chars, no HTML, trimmed, **DIACRITICS ACCEPTED** |
| company        | string   | No       | Full legal name, UPPERCASE, **DIACRITICS ACCEPTED** |
| cif            | string   | No       | 8-digit CIF/CUI (no RO prefix) |
| location       | string[] | No       | Romanian cities, **DIACRITICS ACCEPTED** |
| tags           | string[] | No       | Lowercase, max 20, **NO DIACRITICS** |
| workmode       | string   | No       | Only: "remote", "on-site", "hybrid" |
| date           | date     | No       | UTC ISO8601 (e.g. "2026-01-18T10:00:00Z") |
| status         | string   | No       | Default: "scraped" |
| vdate          | date     | No       | Set only when verified |
| expirationdate | date     | No       | vdate + 30 days max |
| salary         | string   | No       | Format: "MIN-MAX CURRENCY" (e.g. "5000-8000 RON") |

---

## 5. Company Model Fields

| Field       | Type     | Required | Rules |
|-------------|----------|----------|-------|
| id          | string   | Yes      | Exact CIF/CUI, 8 digits |
| company     | string   | Yes      | Legal name, UPPERCASE, **DIACRITICS REQUIRED** |
| brand       | string   | No       | Commercial name for display |
| group       | string   | No       | Parent company group |
| status      | string   | No       | "activ", "suspendat", "inactiv", "radiat" |
| location    | string[] | No       | Cities/addresses, **DIACRITICS ACCEPTED** |
| website     | string[] | No       | Valid URL, no trailing slash |
| career      | string[] | No       | Careers page URL, no trailing slash |
| lastScraped | string   | No       | ISO8601 date |
| scraperFile | string   | No       | Scraper filename reference |

---

## 6. Best Practices

1. **Respect robots.txt** - Check before scraping
2. **Rate limiting** - Add delays between requests (1-2 seconds)
3. **Error handling** - Gracefully handle network errors, CAPTCHAs, page changes
4. **User-Agent** - Use a valid User-Agent header
5. **Data validation** - Always validate output against the model schema
6. **Diacritics** - Preserve diacritics where specified (title, company, location), remove where specified (tags)
7. **Status flow** - New jobs start with status `"scraped"`

---

## 7. Output Format

The scraper should output JSON:

```json
[
  {
    "url": "https://company.com/jobs/123",
    "title": "Developer Software",
    "company": "COMPANIA EXEMPLU SRL",
    "cif": "12345678",
    "location": ["București", "Cluj-Napoca"],
    "tags": ["javascript", "nodejs", "react"],
    "workmode": "hybrid",
    "date": "2026-04-29T10:00:00Z",
    "status": "scraped",
    "salary": "5000-8000 RON"
  }
]
```

---

## 8. Running the Scraper

```bash
npm install
node index.js > output/jobs.json
```
