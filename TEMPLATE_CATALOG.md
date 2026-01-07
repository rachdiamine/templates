# Ghost Witness Template Catalog

**Version:** 2.0
**Last Updated:** 2026-01-07

This document provides detailed information about each available template, including use cases, parameters, workflow, and example usage.

---

## Table of Contents

1. [Tanzania TCRA Templates](#tanzania-tcra-templates)
   - [tanzania_tcra_witness_V4](#tanzania_tcra_witness_v4) ⭐ **RECOMMENDED**
   - [tanzania_tcra_witness_V4_MOCK](#tanzania_tcra_witness_v4_mock)
   - [tanzania_tcra_witness_V3](#tanzania_tcra_witness_v3)
   - [tanzania_tcra_witness_V2](#tanzania_tcra_witness_v2)
   - [tanzania_tcra_ajax_witness](#tanzania_tcra_ajax_witness)
   - [tanzania_tcra_witness](#tanzania_tcra_witness)
   - [tanzania_tcra_invoice_V1](#tanzania_tcra_invoice_v1)

2. [South Africa ICASA Templates](#south-africa-icasa-templates)
   - [icasa_witness_v2](#icasa_witness_v2)

---

## Tanzania TCRA Templates

### tanzania_tcra_witness_V4

⭐ **RECOMMENDED** - Full-featured template with PDF download

#### Overview

**Purpose:** Complete attestation of Tanzania TCRA type approval applications, including application details and PDF certificate download.

**Authority:** Tanzania Communications Regulatory Authority (TCRA)
**Portal:** https://tanzanite.tcra.go.tz
**Version:** 4.0
**Protocol:** 2.0 (Pipeline)
**Status:** ✅ Production Ready

#### Use Case

Extract and attest:
- Application metadata (number, status, dates)
- Equipment details (model, manufacturer)
- License information (number, issue date)
- PDF certificate with hash verification

**Real-world scenario:**
> A telecom equipment supplier needs to prove to investors that their TCRA application (TCRA/APP/TECESRD/0252/2025) was approved and a license was issued. The template extracts the approval status, license number, and downloads the official PDF certificate with cryptographic hash.

#### Parameters

| Parameter | Type | Required | Description | Example |
|-----------|------|----------|-------------|---------|
| `number` | string | Yes | Full TCRA application number | `TCRA/APP/TECESRD/0252/2025` |

#### Processor Pipeline

```
┌─────────────────────────────────────────────────────────┐
│ Step 1: AJAX Capture (fetch_application)               │
│ - Endpoint: /v1/get-all-application-list-from-lnc-...  │
│ - Action: Filter application by number                 │
│ - Output: Application metadata (id, status, license)   │
└────────────────┬────────────────────────────────────────┘
                 │
                 ↓ (if id exists)
┌─────────────────────────────────────────────────────────┐
│ Step 2: HTML Fetch (fetch_application_details)         │
│ - Endpoint: /type-approval-application-details.htm/... │
│ - Action: Extract manufacturer and PDF URL             │
│ - Output: manufacturer_name, licence_pdf_url           │
└────────────────┬────────────────────────────────────────┘
                 │
                 ↓ (if licence_pdf_url exists)
┌─────────────────────────────────────────────────────────┐
│ Step 3: File Download (download_licence_pdf)           │
│ - Endpoint: Dynamic from Step 2                        │
│ - Action: Download PDF and calculate SHA-256 hash      │
│ - Output: file_hash, filename, file_size               │
└─────────────────────────────────────────────────────────┘
```

#### Proof Data (Signed)

Fields included in cryptographic signature:

| Field | Source | Description |
|-------|--------|-------------|
| `application_number` | fetch_application.applicationNumber | Full application ID |
| `application_status` | fetch_application.statusName | Status (APPROVED, PENDING, etc.) |
| `model_name` | fetch_application.equipmentModel | Equipment model name |
| `license_number` | fetch_application.licenceNumber | Issued license number |
| `manufacturer_name` | fetch_application_details.manufacturer_name | Manufacturer company name |
| `application_date` | fetch_application.applicationDate | Application submission date (timestamp) |
| `licence_pdf_hash` | download_licence_pdf.file_hash | SHA-256 hash of PDF certificate |

#### Additional Data (Not Signed)

Fields included in `more_data` (not in cryptographic proof):

| Field | Source | Description |
|-------|--------|-------------|
| `product_name` | fetch_application.productName | Product description |
| `issued_date` | fetch_application.issuedAt | License issue date |
| `application_date_original` | fetch_application.applicationDate | Original date string |
| `licence_pdf_url` | fetch_application_details.licence_pdf_url | PDF download URL |
| `licence_pdf_filename` | download_licence_pdf.filename | Downloaded filename |

#### Example Request

```bash
curl -X POST https://tee-server.phala.network/api/attest \
  -H "Content-Type: application/json" \
  -d '{
    "template_id": "tanzania_tcra_witness_V4",
    "template_hash": "a1b2c3d4e5f6...",
    "parameters": {
      "number": "TCRA/APP/TECESRD/0252/2025"
    },
    "processor_contexts": [
      {
        "processor_id": "fetch_application",
        "session_cookies": [
          {
            "name": "JSESSIONID",
            "value": "ABC123...",
            "domain": "tanzanite.tcra.go.tz",
            "path": "/",
            "secure": true,
            "httpOnly": true
          }
        ],
        "headers": {
          "User-Agent": "Mozilla/5.0...",
          "Accept": "application/json"
        }
      }
    ]
  }'
```

#### Example Response

```json
{
  "success": true,
  "proof_package": {
    "proof_data": {
      "application_number": "TCRA/APP/TECESRD/0252/2025",
      "application_status": "APPROVED",
      "model_name": "SmartPhone X200",
      "license_number": "TA-2025-0252",
      "manufacturer_name": "TechCorp International",
      "application_date": 1736938200000,
      "licence_pdf_hash": "sha256:a1b2c3...",
      "template_hash": "md5:def456...",
      "timestamp": 1704470580000,
      "epoch_id": 0
    },
    "proof_data_hash": "sha256:1234567890abcdef...",
    "witness_signature": "0x1234abcd...",
    "commitment_hash": "0xabcdef1234...",
    "tee_attestation": {
      "app_id": "phala_app_xyz",
      "data_report": "abcdef...|0x1234...|0",
      "tee_quote": "0x5678...",
      "protocol_version": "1.4"
    }
  },
  "all_data_zip": "base64_encoded_zip_containing_proof_and_pdf"
}
```

#### Extractors Used

**Step 1: json_filter**
- Filter JSON array by `applicationNumber` field
- Extract specific application record

**Step 2: css_selector**
```css
td:has(span:contains('Manufacturer Name')) + td  /* Manufacturer */
a.btn-success[href*='lims-ndoo']                 /* PDF link */
```

**Step 3: file_download**
- Downloads PDF binary
- Calculates SHA-256 hash
- Returns file metadata

#### Template Hash

```bash
# Calculate template hash
sha256sum templates/web-templates/tanzania_tcra_witness_V4
# Example output: a1b2c3d4e5f6789012345678901234567890abcdef123456789012345678901234
```

#### When to Use

✅ **Use this template when:**
- Need complete application details
- Need to download and verify PDF certificate
- Need manufacturer information
- Require highest level of proof (includes file hash)

❌ **Don't use when:**
- Only need basic status (use V2 instead)
- Don't need PDF download (use V2 instead)
- Portal structure has changed (create new version)

---

### tanzania_tcra_witness_V4_MOCK

🧪 **Testing Only** - Mock server version of V4

#### Overview

Identical to `tanzania_tcra_witness_V4` but configured to work with local mock server instead of real TCRA portal.

**Purpose:** Development and testing without hitting production portal
**Portal:** http://localhost:4000 (mock server)
**Status:** 🧪 Testing/Development

#### Differences from V4

| Aspect | V4 | V4_MOCK |
|--------|----|----|
| **Portal URL** | https://tanzanite.tcra.go.tz | http://localhost:4000 |
| **Data Source** | Real TCRA portal | Mock server with 104 test applications |
| **Rate Limits** | Subject to portal limits | No limits |
| **Reliability** | Depends on portal availability | Always available offline |

#### Setup

1. **Start Mock Server:**
```bash
cd phala-cloud-node-starter
node mock-tcra-server.cjs
```

2. **Verify Mock Server:**
```bash
curl http://localhost:4000
# Should return: "Mock TCRA Server Running"
```

3. **Use Template:**
Same as V4 but template loads from mock endpoints automatically.

#### Use Cases

✅ **Use for:**
- Local development
- Template testing
- CI/CD pipeline tests
- Demo presentations
- Training

❌ **Never use for:**
- Production attestations
- Real regulatory compliance
- Investor proofs

---

### tanzania_tcra_witness_V3

⚠️ **Deprecated** - Use V4 instead

#### Overview

**Status:** ⚠️ Legacy (Deprecated)
**Recommendation:** Migrate to V4

Multi-processor pipeline with older CSS selector approach.

#### Why Deprecated

1. Less reliable CSS selectors
2. Verbose extractor definitions
3. No date transformation
4. Missing manufacturer field
5. Replaced by V4's cleaner implementation

#### Migration Path

**From V3 to V4:**
- Same parameters (just `number`)
- Same proof_data fields (V4 adds manufacturer)
- Same pipeline structure (V4 improved extractors)
- **Action:** Simply change `template_id` to `tanzania_tcra_witness_V4`

---

### tanzania_tcra_witness_V2

⚠️ **Deprecated** - Use V4 instead

#### Overview

**Status:** ⚠️ Legacy (Deprecated)
**Recommendation:** Migrate to V4

Single-processor AJAX capture with basic proof data.

#### Features

- Single AJAX capture processor
- json_filter extractor
- Basic proof_data (5 fields)
- Date transformation support
- more_data separation

#### Proof Data (5 fields)

1. application_number
2. application_status
3. model_name
4. license_number
5. application_date (transformed to timestamp)

#### When V2 is Sufficient

✅ **Still usable when:**
- Don't need PDF download
- Don't need manufacturer name
- Want fastest response time (single processor)
- AJAX endpoint is reliable

#### Migration to V4

**What V4 adds:**
- Step 2: HTML fetch (manufacturer)
- Step 3: PDF download (file hash)
- More robust extraction
- Complete attestation package

---

### tanzania_tcra_ajax_witness

⚠️ **Deprecated** - Use V2 or V4

#### Overview

**Status:** ⚠️ Legacy (Deprecated)
**Version:** 2.0 (early V2 prototype)

Nearly identical to V2 but **missing proof_data and more_data sections**.

#### Problem

This template extracts data but doesn't define what goes into the cryptographic proof. The TEE server may use all extracted fields or none, making attestations unpredictable.

#### Recommendation

**Use `tanzania_tcra_witness_V2` or `tanzania_tcra_witness_V4` instead** - both have explicit proof_data definitions.

---

### tanzania_tcra_witness

⚠️ **Deprecated** - Legacy regex version

#### Overview

**Status:** ⚠️ Legacy (Deprecated)
**Version:** 1.0
**Protocol:** 1.0 (Pre-pipeline)

Original TCRA template using complex regex extractors.

#### Extraction Method

Uses regex patterns like:
```regex
<td[^>]*>\s*Application Number\s*</td>\s*<td[^>]*>\s*([^<]+)\s*</td>
```

#### Problems

1. **Brittle:** Breaks with any HTML structure change
2. **Complex:** Hard to maintain regex patterns
3. **Limited:** Single HTML fetch only (no AJAX, no PDF)
4. **Protocol 1.0:** Doesn't support pipeline architecture

#### Extractors

- application_number
- certificate_number
- status
- issue_date
- model_name
- tracker_status
- tracker_date

#### Recommendation

**Migrate to V4** for:
- Pipeline architecture
- CSS selectors (more resilient)
- AJAX capture (faster)
- PDF download support

---

### tanzania_tcra_invoice_V1

🚧 **Experimental** - Under Development

#### Overview

**Status:** 🚧 Experimental / Work in Progress
**Purpose:** Extract invoice data from TCRA payment portal

**⚠️ Warning:** This template is incomplete and not ready for production use.

#### Intended Use Case

Extract and attest TCRA invoice/payment information:
- Invoice number
- Amount due
- Payment status
- Due date

#### Current Status

- Template structure defined
- Processors not yet implemented
- Extractors under development
- No proof_data defined

#### Development Roadmap

- [ ] Define processor pipeline
- [ ] Implement extractors
- [ ] Test with sample invoices
- [ ] Define proof_data fields
- [ ] Add to registry (disabled)
- [ ] Testing phase
- [ ] Production release

---

## South Africa ICASA Templates

### icasa_witness_v2

✅ **Production Ready** - ICASA Type Approval Search

#### Overview

**Purpose:** Extract and attest type approval certificates from South Africa ICASA portal

**Authority:** Independent Communications Authority of South Africa (ICASA)
**Portal:** https://online.icasa.org.za/TypeApproval/ManageTypeApprovals
**Version:** 2.0
**Protocol:** 2.0
**Status:** ✅ Production Ready

#### Use Case

Extract and attest ICASA type approval information:
- Certificate/application number
- License holder information
- Equipment description
- Approval status
- Issue/expiry dates

**Real-world scenario:**
> A radio equipment importer needs to prove their ICASA type approval (TA-2023-0123) is valid and approved for customs clearance.

#### Parameters

| Parameter | Type | Required | Description | Example |
|-----------|------|----------|-------------|---------|
| `number` | string | Yes | ICASA certificate or application number | `TA-2023-0123` |

#### Processor Pipeline

```
┌─────────────────────────────────────────────────────────┐
│ Step 1: HTML Fetch (fetch_application)                 │
│ - Endpoint: /TypeApproval/ManageTypeApprovals          │
│ - Action: Fetch HTML table, extract row by number      │
│ - Method: Regex extraction (table row pattern)         │
│ - Output: type, number, holder, description, status    │
└─────────────────────────────────────────────────────────┘
```

#### Extractor: Regex Table Row

Complex regex pattern that matches HTML table row structure:

```regex
<table.*<tr>
  <td>(.+)</td>              <!-- type -->
  <td>{number}</td>          <!-- application number (matched) -->
  <td>(.+)</td>              <!-- licence_holder -->
  <td>(.+)</td>              <!-- description -->
  <td>(.+)</td>              <!-- status -->
  <td>(.+)</td>              <!-- date -->
</tr>
```

**Pattern matches:** Entire table row where the number column matches the parameter.

#### Proof Data

All extracted fields are included in proof (no more_data section):

| Field | Description | Example |
|-------|-------------|---------|
| `type` | Approval type | "Type Approval Certificate" |
| `number` | Certificate/application number | "TA-2023-0123" |
| `licence_holder` | Company/individual name | "TechCorp South Africa (Pty) Ltd" |
| `description` | Equipment description | "Mobile Handset - Model XYZ" |
| `status` | Approval status | "Approved" |
| `date` | Issue/expiry date | "2023-06-15" |

#### Example Request

```bash
curl -X POST https://tee-server.phala.network/api/attest \
  -H "Content-Type: application/json" \
  -d '{
    "template_id": "icasa_witness_v2",
    "template_hash": "xyz789...",
    "parameters": {
      "number": "TA-2023-0123"
    },
    "processor_contexts": [
      {
        "processor_id": "fetch_application",
        "session_cookies": [
          {
            "name": "ASP.NET_SessionId",
            "value": "abc123...",
            "domain": "online.icasa.org.za"
          }
        ]
      }
    ]
  }'
```

#### Example Response

```json
{
  "success": true,
  "proof_package": {
    "proof_data": {
      "type": "Type Approval Certificate",
      "number": "TA-2023-0123",
      "licence_holder": "TechCorp South Africa (Pty) Ltd",
      "description": "Mobile Handset - Model XYZ",
      "status": "Approved",
      "date": "2023-06-15",
      "template_hash": "md5:xyz789...",
      "timestamp": 1704470580000,
      "epoch_id": 0
    },
    "proof_data_hash": "sha256:abcdef...",
    "witness_signature": "0x9876...",
    "commitment_hash": "0xfedcba...",
    "tee_attestation": { ... }
  }
}
```

#### Authentication

**Important:** ICASA portal requires authentication. You must:

1. Login to portal in browser
2. Extract session cookies
3. Include cookies in `processor_contexts`

**Required Cookies:**
- `ASP.NET_SessionId`
- Any authentication tokens

**Cookie Lifespan:** Usually 20-30 minutes of inactivity.

#### Limitations

1. **Single Processor:** Only fetches main table page (no detail page)
2. **Regex-based:** Brittle if HTML structure changes
3. **No PDF Download:** Certificate PDF not included (future enhancement)
4. **Session Required:** Must be authenticated

#### Future Enhancements (V3)

Planned improvements:
- [ ] Add second processor to fetch detail page
- [ ] Download certificate PDF
- [ ] Switch to CSS selectors (more resilient)
- [ ] Add expiry date validation
- [ ] Include certificate PDF hash in proof

#### When to Use

✅ **Use this template when:**
- Need ICASA type approval verification
- Have valid session cookies
- Need basic certificate information
- Quick attestation (single processor)

❌ **Don't use when:**
- Portal structure has changed (regex will fail)
- Need certificate PDF
- Need detailed application history

---

## Template Comparison Matrix

| Template | Processors | PDF Download | Auth Required | Response Time | Proof Fields |
|----------|-----------|--------------|---------------|---------------|--------------|
| **tanzania_tcra_witness_V4** | 3 | ✅ Yes | Optional | 15-30s | 7 |
| **tanzania_tcra_witness_V2** | 1 | ❌ No | Optional | 2-5s | 5 |
| **icasa_witness_v2** | 1 | ❌ No | ✅ Yes | 5-10s | 6 |

---

## Choosing the Right Template

### Decision Tree

```
Need TCRA (Tanzania)?
├─ Yes → Need PDF certificate?
│  ├─ Yes → Use tanzania_tcra_witness_V4 ⭐
│  └─ No  → Use tanzania_tcra_witness_V2
│
└─ No → Need ICASA (South Africa)?
   ├─ Yes → Use icasa_witness_v2 ✅
   └─ No  → Create new template
```

### By Use Case

**Regulatory Compliance (Full Proof):**
- Use **tanzania_tcra_witness_V4** (includes PDF hash)
- Use **icasa_witness_v2** (all available fields)

**Quick Status Check:**
- Use **tanzania_tcra_witness_V2** (fast, basic fields)

**Development/Testing:**
- Use **tanzania_tcra_witness_V4_MOCK** (offline testing)

**Custom Portal:**
- Create new template following `/templates/README.md` guide

---

## Template Naming Convention

```
{authority}_{portal}_{type}_{version}
```

**Examples:**
- `tanzania_tcra_witness_V4` - Tanzania, TCRA, Witness/Attestation, Version 4
- `icasa_witness_v2` - ICASA, Witness/Attestation, Version 2
- `tanzania_tcra_invoice_V1` - Tanzania, TCRA, Invoice, Version 1

**Version Numbering:**
- V1, V2, V3... - Major version increments
- Breaking changes warrant new version
- Use lowercase `v2` or uppercase `V2` consistently

---

## Maintenance & Updates

### Template Lifecycle

1. **Development** (🚧) - Template being created, not in registry
2. **Testing** (🧪) - In registry with `enabled: false`, testing phase
3. **Production** (✅) - In registry with `enabled: true`, active use
4. **Deprecated** (⚠️) - Still works but replaced by newer version
5. **Archived** (📦) - Removed from registry, kept for reference

### When to Create New Version

**Create new version when:**
- Portal structure changes significantly
- Need different processor pipeline
- Different proof_data fields required
- Breaking changes to extractor logic

**Update existing version when:**
- Fix minor regex/CSS selector
- Adjust timeouts
- Update metadata
- Add more_data fields (doesn't affect proof)

### Testing Checklist

Before marking template as production:

- [ ] Test with multiple sample applications
- [ ] Verify all extractors work
- [ ] Confirm proof_data fields extract correctly
- [ ] Test conditional processors (if applicable)
- [ ] Verify file downloads (if applicable)
- [ ] Calculate and record template hash
- [ ] Test with real session cookies
- [ ] Validate TEE proof generation
- [ ] Document example usage
- [ ] Add to registry

---

## Support

### Template Not Working?

1. **Check Portal Status:** Is the portal available?
2. **Check HTML Structure:** Has the portal changed?
3. **Check Session Cookies:** Are they valid and not expired?
4. **Check Logs:** Review TEE server logs for extraction errors
5. **Test with Mock:** Use V4_MOCK to isolate issues

### Request New Template

To request a template for a new portal:

1. Open GitHub issue with:
   - Portal URL
   - Portal authentication method
   - Sample application number
   - Screenshots of data to extract
   - Use case description

2. Or create template yourself following `/templates/README.md`

---

**Last Updated:** 2026-01-07
**Catalog Version:** 1.0
**Total Templates:** 8 (2 active, 5 deprecated, 1 experimental)
