# Ghost Witness Template System Documentation

**Version:** 2.0
**Last Updated:** 2026-01-07

---

## Table of Contents

1. [Overview](#overview)
2. [What Are Templates?](#what-are-templates)
3. [Template Registry](#template-registry)
4. [Available Templates](#available-templates)
5. [Template Structure](#template-structure)
6. [Processor Types](#processor-types)
7. [Extractor Types](#extractor-types)
8. [Creating Templates](#creating-templates)
9. [Testing Templates](#testing-templates)
10. [Best Practices](#best-practices)

---

## Overview

The Ghost Witness template system separates **business logic** (extraction rules) from **execution logic** (TEE server). This architecture enables:

- **Rapid Deployment**: New portals = new templates (no code changes)
- **Community Contribution**: Anyone can create templates
- **Version Control**: Templates are versioned and hashed
- **Reliability**: Templates tested independently of server code

### Design Philosophy

> **"Business Logic Lives in Templates, Not Code"**

When a new government portal needs support, you create a JSON template file - not modify server code. This keeps the TEE server simple, generic, and stable.

---

## What Are Templates?

A template is a **JSON configuration file** that defines:

1. **What endpoints to call** (AJAX APIs, HTML pages, file URLs)
2. **What parameters are needed** (application numbers, dates, etc.)
3. **How to extract data** (processor pipeline with extractors)
4. **What to include in proof** (proof_data fields)

### Execution Flow

```
┌─────────────────────────────────────────────────────────┐
│ 1. Client reads template                                │
│    - Understands what endpoints will be called          │
│    - Determines what HTTP context is needed             │
│    - Obtains session cookies, auth headers, etc.        │
└────────────────┬────────────────────────────────────────┘
                 │
                 ↓
┌─────────────────────────────────────────────────────────┐
│ 2. Client sends request to TEE                          │
│    POST /api/attest                                     │
│    {                                                     │
│      "template_id": "...",                              │
│      "template_hash": "...",                            │
│      "parameters": { ... },                             │
│      "processor_contexts": [                            │
│        {                                                 │
│          "processor_id": "...",                         │
│          "session_cookies": [...],                      │
│          "headers": {...}                               │
│        }                                                 │
│      ]                                                   │
│    }                                                     │
└────────────────┬────────────────────────────────────────┘
                 │
                 ↓
┌─────────────────────────────────────────────────────────┐
│ 3. TEE loads and validates template                     │
│    - Verify template hash                               │
│    - Check template is enabled in registry              │
│    - Load processor pipeline                            │
└────────────────┬────────────────────────────────────────┘
                 │
                 ↓
┌─────────────────────────────────────────────────────────┐
│ 4. TEE executes processors sequentially                 │
│    For each processor:                                  │
│    - Make HTTP request to endpoint                      │
│    - Use provided cookies/headers from context          │
│    - Apply extractors to response                       │
│    - Check conditions for next processor                │
└────────────────┬────────────────────────────────────────┘
                 │
                 ↓
┌─────────────────────────────────────────────────────────┐
│ 5. TEE generates cryptographic proof                    │
│    - Build proof_data from extracted fields             │
│    - Sign with TEE wallet                               │
│    - Generate TEE quote                                 │
│    - Package all_data.zip with downloaded files         │
└────────────────┬────────────────────────────────────────┘
                 │
                 ↓
┌─────────────────────────────────────────────────────────┐
│ 6. TEE returns proof package                            │
│    {                                                     │
│      "proof_package": { ... },                          │
│      "all_data_zip": "..."                              │
│    }                                                     │
└─────────────────────────────────────────────────────────┘
```

### Example

**Template (`tanzania_tcra_witness_V4`):**
```json
{
  "id": "tanzania_tcra_witness_V4",
  "parameters": [
    { "name": "number", "type": "string", "required": true }
  ],
  "processors": [
    {
      "processor_id": "fetch_application",
      "processor_action": {
        "action_type": "ajax_capture",
        "endpoints": [
          "https://tanzanite.tcra.go.tz/v1/get-all-application-list-from-lnc-by-ajax"
        ]
      }
    }
  ]
}
```

**Client provides:**
- `parameters`: `{ "number": "TCRA/APP/TECESRD/0252/2025" }`
- `processor_contexts`: Session cookies for authenticated access

**TEE executes:**
- Calls endpoint with provided cookies
- Extracts application data
- Generates cryptographic proof

---

## Template Registry

**Location:** `/templates/registry-simple.json`

The registry is the **index of all available templates**. It controls:

- Which templates are available
- Which templates are enabled/disabled
- Template metadata (name, URL, tags)

### Registry Structure

```json
{
  "version": "2.0",
  "updated": "2026-01-07",
  "templates": [
    {
      "id": "tanzania_tcra_witness_V4",
      "name": "TCRA Full with PDF (CSS Selectors)",
      "url": "https://tanzanite.tcra.go.tz",
      "tags": ["tanzania", "tcra"],
      "enabled": true
    }
  ]
}
```

### Registry Fields

| Field | Required | Description |
|-------|----------|-------------|
| `id` | Yes | Must match template filename in `/web-templates/` |
| `name` | Yes | Human-readable display name |
| `url` | Yes | Portal URL where template applies |
| `tags` | No | Categorization tags (country, authority) |
| `enabled` | Yes | `true` to allow usage, `false` to disable |

### Enabling/Disabling Templates

To **disable** a template without deleting it:

```json
{
  "id": "old_template_v1",
  "enabled": false
}
```

Disabled templates return `403 Forbidden` when requested.

---

## Available Templates

### Tanzania TCRA Templates

#### tanzania_tcra_witness_V4
**Status:** ✅ **RECOMMENDED**
**Processors:** 3 (AJAX → HTML → PDF download)

**What it extracts:**
- Application number, status, dates
- Model name, license number
- Manufacturer name
- License PDF hash (SHA-256)

---

#### tanzania_tcra_witness_V4_MOCK
**Status:** 🧪 Testing Only
**Purpose:** Same as V4 but calls localhost:4000 mock server

---

#### tanzania_tcra_witness_V2
**Status:** ⚠️ Active but V4 recommended
**Processors:** 1 (AJAX only)

---

### South Africa ICASA Templates

#### icasa_witness_v2
**Status:** ✅ Active
**Processors:** 1 (HTML fetch)

**What it extracts:**
- Certificate number, type
- License holder, description
- Status, dates

---

### Template Versions Summary

| Template | Version | Status | Processors | Use |
|----------|---------|--------|------------|-----|
| **tanzania_tcra_witness_V4** | 4.0 | ✅ Production | 3 | Full attestation with PDF |
| **tanzania_tcra_witness_V4_MOCK** | 4.0 | 🧪 Test | 3 | Local testing |
| **icasa_witness_v2** | 2.0 | ✅ Production | 1 | ICASA certificates |
| **tanzania_tcra_witness_V2** | 2.0 | ⚠️ Legacy | 1 | Basic TCRA data |

**See [TEMPLATE_CATALOG.md](./TEMPLATE_CATALOG.md) for detailed documentation of each template.**

---

## Template Structure

### Minimal Template

```json
{
  "id": "unique_template_id",
  "name": "Human-readable Name",
  "version": "1.0",
  "protocol_version": "2.0",
  "type": "pipeline",

  "url_pattern": {
    "active_on": "https://portal.example.com"
  },

  "metadata": {
    "authority": "Authority Name",
    "country": "Country",
    "domain": "portal.example.com"
  },

  "parameters": [
    {
      "name": "application_number",
      "type": "string",
      "required": true,
      "prompt": "Application Number"
    }
  ],

  "processors": [
    {
      "processor_id": "fetch_data",
      "processor_name": "Fetch Application Data",
      "processor_action": {
        "action_type": "ajax_capture",
        "timeout": 15000,
        "endpoints": ["https://portal.example.com/api/applications"]
      },
      "extractors": [...]
    }
  ],

  "proof_data": {
    "output_format": "json",
    "parts": [...]
  }
}
```

### Top-Level Fields

| Field | Required | Type | Description |
|-------|----------|------|-------------|
| `id` | Yes | string | Unique identifier (matches filename) |
| `name` | Yes | string | Display name |
| `version` | Yes | string | Template version (semver recommended) |
| `protocol_version` | Yes | string | Ghost Witness protocol version (2.0) |
| `type` | Yes | string | Always "pipeline" |
| `url_pattern` | Yes | object | Portal URL configuration |
| `metadata` | Yes | object | Authority, country, domain info |
| `parameters` | Yes | array | User inputs required |
| `processors` | Yes | array | Extraction pipeline steps |
| `proof_data` | Yes | object | Fields included in cryptographic proof |
| `more_data` | No | object | Additional fields (not in signature) |

---

## Processor Types

Processors are the **steps in your extraction pipeline**. They execute sequentially, with each step potentially depending on previous steps.

**How processors work:**
1. Template defines endpoint(s) to call
2. Client provides HTTP context (cookies, headers) for each processor
3. TEE makes HTTP request using provided context
4. TEE applies extractors to response
5. TEE outputs extracted data

### 1. ajax_capture

**Purpose:** Call AJAX/API endpoint and extract JSON data

**How it works:**
1. TEE makes HTTP request to endpoint
2. TEE receives JSON response
3. TEE applies json_filter extractor
4. TEE outputs filtered data

**Example:**

```json
{
  "processor_id": "fetch_applications",
  "processor_name": "Fetch Application List",
  "processor_action": {
    "action_type": "ajax_capture",
    "timeout": 15000,
    "endpoints": [
      "https://tanzanite.tcra.go.tz/v1/get-all-application-list-from-lnc-by-ajax"
    ]
  },
  "extractors": [
    {
      "id": "filter_by_number",
      "type": "json_filter",
      "filter": {
        "field": "applicationNumber",
        "operator": "equals",
        "value": "{number}"
      },
      "output_fields": ["id", "status", "licenceNumber"]
    }
  ]
}
```

**Use Cases:**
- Filter large JSON arrays
- Extract specific records from API responses
- Get data from AJAX endpoints

---

### 2. html_fetch

**Purpose:** Fetch HTML page and extract data using CSS selectors or regex

**How it works:**
1. TEE makes HTTP GET request to endpoint
2. TEE receives HTML response
3. TEE applies CSS selector or regex extractors
4. TEE returns extracted fields

**Example:**

```json
{
  "processor_id": "fetch_details",
  "processor_name": "Fetch Application Details Page",
  "processor_action": {
    "action_type": "html_fetch",
    "timeout": 15000,
    "endpoints": [
      "https://tanzanite.tcra.go.tz/type-approval-application-details.htm/{{fetch_applications.id}}/1"
    ]
  },
  "extractors": [
    {
      "id": "extract_manufacturer",
      "type": "css_selector",
      "fields": [
        {
          "name": "manufacturer_name",
          "selector": "td:has(span:contains('Manufacturer Name')) + td",
          "attribute": "text"
        }
      ]
    }
  ]
}
```

**Template Variables:**
- Use `{{processor_id.field}}` to reference previous processor outputs
- Example: `{{fetch_applications.id}}` uses ID from previous step

**Use Cases:**
- Scrape details pages
- Extract data from HTML pages
- Get links to download files

---

### 3. file_download

**Purpose:** Download files (PDF, ZIP, etc.) and calculate hash

**How it works:**
1. TEE makes HTTP GET request to file URL
2. TEE downloads file as binary buffer
3. TEE calculates SHA-256 hash
4. TEE stores file temporarily for packaging
5. TEE returns file metadata

**Example:**

```json
{
  "processor_id": "download_certificate",
  "processor_name": "Download License PDF",
  "processor_action": {
    "action_type": "file_download",
    "timeout": 30000,
    "endpoints": [
      "{{fetch_details.licence_pdf_url}}"
    ]
  },
  "extractors": []
}
```

**Output Fields (automatic):**
- `file_hash`: SHA-256 hash (hex)
- `filename`: Downloaded filename
- `file_size`: Size in bytes
- `content_type`: MIME type

**Use Cases:**
- Download certificates/licenses
- Verify file integrity with hash
- Include files in attestation package

---

### Conditional Processors

Processors can execute **conditionally** based on previous outputs:

```json
{
  "processor_id": "download_pdf",
  "condition": {
    "field": "fetch_details.licence_pdf_url",
    "operator": "exists",
    "value": ""
  },
  "processor_action": { ... }
}
```

**Operators:**
- `exists`: Field has a non-empty value
- `equals`: Field equals specific value
- `contains`: Field contains substring

**Example Flow:**

```
1. ajax_capture → Extract application (may have ID or not)
2. html_fetch (if ID exists) → Get PDF URL
3. file_download (if PDF URL exists) → Download PDF
```

---

## Extractor Types

Extractors define **how to get data** from processor responses.

### 1. json_filter

**Purpose:** Filter JSON arrays by field value

**Syntax:**

```json
{
  "id": "filter_application",
  "type": "json_filter",
  "filter": {
    "field": "applicationNumber",
    "operator": "equals",
    "value": "{number}"
  },
  "output_fields": ["id", "status", "licenceNumber"]
}
```

**Filter Operators:**
- `equals`: Exact match
- `contains`: Substring match
- `starts_with`: Prefix match
- `ends_with`: Suffix match

**Output:**
First matching object with specified fields only.

---

### 2. css_selector

**Purpose:** Extract data from HTML using CSS selectors

**Syntax:**

```json
{
  "id": "extract_details",
  "type": "css_selector",
  "repeat": false,
  "fields": [
    {
      "name": "manufacturer_name",
      "selector": "td:has(span:contains('Manufacturer Name')) + td",
      "attribute": "text"
    },
    {
      "name": "pdf_link",
      "selector": "a.btn-success[href*='lims-ndoo']",
      "attribute": "href"
    }
  ]
}
```

**Attributes:**
- `text`: Inner text content
- `href`: Link URL
- `src`: Image/script source
- `value`: Input value
- `data-*`: Data attributes

**CSS Selectors:**
- Standard CSS3 selectors
- jQuery-style extensions (`:has`, `:contains`)
- Attribute selectors (`[href*='pdf']`)

---

### 3. regex

**Purpose:** Extract data using regular expressions

**Syntax:**

```json
{
  "id": "extract_row",
  "type": "regex",
  "repeat": false,
  "parts": [
    {
      "is_public": "no",
      "regex": "<table.*?<tr>"
    },
    {
      "is_public": "yes",
      "field": "status",
      "regex": "([A-Z]+)"
    },
    {
      "is_public": "no",
      "regex": "</tr>"
    }
  ]
}
```

**Parts:**
- `is_public: "no"`: Non-capturing (structure/whitespace)
- `is_public: "yes"`: Capturing (actual data field)
- `field`: Output field name
- `regex`: Regular expression pattern

---

## Creating Templates

### Step 1: Understand the Portal

1. **Navigate** to the government portal
2. **Inspect** network traffic (Chrome DevTools → Network tab)
3. **Identify** endpoints:
   - AJAX/API endpoints
   - HTML detail pages
   - Download links

### Step 2: Design Processor Pipeline

**Ask:**
- What data do I need to extract?
- In what order should processors run?
- Which steps depend on previous steps?
- What endpoints need to be called?

### Step 3: Define Template Structure

1. Create template file in `/templates/web-templates/`
2. Define metadata and parameters
3. Build processor pipeline with endpoints
4. Define extractors for each processor
5. Specify proof_data fields
6. Add to registry

**See [TEMPLATE_CREATION_GUIDE.md](./TEMPLATE_CREATION_GUIDE.md) for detailed tutorial.**

---

## Testing Templates

### Local Testing

1. **Start TEE Server:**
   ```bash
   cd phala-cloud-node-starter
   npm run dev
   ```

2. **Calculate Template Hash:**
   ```bash
   sha256sum templates/web-templates/your_template_id
   ```

3. **Test Attestation:**
   ```bash
   curl -X POST http://localhost:3000/api/attest \
     -H "Content-Type: application/json" \
     -d '{
       "template_id": "your_template_id",
       "template_hash": "calculated_hash",
       "parameters": { "number": "TEST-001" },
       "processor_contexts": [
         {
           "processor_id": "step_1",
           "session_cookies": [...],
           "headers": {...}
         }
       ]
     }'
   ```

**See [TEMPLATE_TESTING_GUIDE.md](./TEMPLATE_TESTING_GUIDE.md) for comprehensive testing procedures.**

---

## Best Practices

### Naming Conventions

**Template IDs:**
- Format: `{authority}_{type}_{version}`
- Example: `tanzania_tcra_witness_V4`
- Use lowercase with underscores

**Processor IDs:**
- Format: `{verb}_{noun}`
- Example: `fetch_application`, `download_pdf`

**Field Names:**
- Use snake_case
- Be explicit: `application_number` not `number`

### Version Control

**When to increment version:**
- Breaking changes to processor pipeline
- Different proof_data fields
- Major extraction logic changes

**Version numbering:**
- V1, V2, V3... - Major iterations
- Use semantic versioning in `version` field: "4.0"

### Performance

**Timeouts:**
- AJAX/API calls: 10-15 seconds
- HTML fetch: 10-20 seconds
- File download: 20-30 seconds (depends on file size)

**Minimize Processors:**
- Combine extractions when possible
- Use conditional processors to avoid unnecessary calls

### Security

**Never include in templates:**
- Hardcoded credentials
- API keys
- Session tokens
- Personal data

---

## Additional Resources

- **Template Specification:** [TEMPLATE_SPEC.md](./TEMPLATE_SPEC.md)
- **Template Catalog:** [TEMPLATE_CATALOG.md](./TEMPLATE_CATALOG.md)
- **Creation Guide:** [TEMPLATE_CREATION_GUIDE.md](./TEMPLATE_CREATION_GUIDE.md)
- **Testing Guide:** [TEMPLATE_TESTING_GUIDE.md](./TEMPLATE_TESTING_GUIDE.md)
- **Implementation Guide:** [IMPLEMENTATION_GUIDE.md](./IMPLEMENTATION_GUIDE.md)

---

## Support & Contribution

### Report Issues

If a template stops working:
1. Check if portal structure changed
2. Test locally
3. Submit issue with template ID and error details

### Contribute Templates

To contribute a new template:
1. Create template in `/templates/web-templates/`
2. Add to registry with `enabled: false`
3. Test thoroughly
4. Submit PR with template file, registry entry, and test results

---

**Last Updated:** 2026-01-07
**Template Protocol Version:** 2.0
**Maintainer:** Ghost Witness Team
