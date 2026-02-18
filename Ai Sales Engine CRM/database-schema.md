# AI Sales Engine CRM - Database Schema

> **Purpose:** This document defines the complete database structure for the AI Sales Engine CRM, including all tables, fields, relationships, and data types.

**Last Updated:** Nov 4, 2025  
**Version:** 2.0

---

## Database Overview

The system uses three complementary database approaches:
1. **Structured Database (Airtable):** Lead profiles (engaged only), campaigns, clients, products - used for CRM operations
2. **Scalable Database (Supabase/PostgreSQL):** Contacts table (everyone), chat_history - for high-volume data & deduplication
3. **Vector Database (Pinecone):** Vectorized content for semantic search and AI retrieval

### Key Architecture Decision: Contacts vs Leads

| Database | Table | Contains | Purpose |
|----------|-------|----------|---------|
| **Supabase** | `contacts` | Everyone we've ever emailed | Data warehouse, deduplication, AI context |
| **Airtable** | `Leads` | Only engaged contacts | CRM workspace, sales process |

**Flow:** Contact replies → Promoted to Airtable Lead → Human works the deal

---

## Structured Database Schema

### 1. Lead Lists Table (Supabase)

**Purpose:** Track lead lists/batches for bulk operations and organization. Created before uploading contacts to track which batch leads came from.

> **Note:** This table lives in Supabase. Create a lead list first, then use its `list_id` UUID when uploading contacts.

| Field Name | Type | Description | Required |
|------------|------|-------------|----------|
| `list_id` | UUID | Primary key (auto-generated) | Yes |
| `list_name` | VARCHAR | Human-readable list name | Yes |
| `client_id` | VARCHAR | Multi-tenant: which client this belongs to | Yes |
| `description` | TEXT | Optional description of the list | No |
| `source` | VARCHAR | Where leads came from | No |
| `total_contacts` | INTEGER | Total contacts in this list | No |
| `contacts_in_campaigns` | INTEGER | How many are in campaigns | No |
| `status` | VARCHAR | List status | Yes |
| `metadata` | JSONB | Store filters, CSV headers, original query, etc. | No |
| `created_at` | TIMESTAMPTZ | When list was created | Yes |
| `updated_at` | TIMESTAMPTZ | Last update time | Yes |
| `created_by` | VARCHAR | Who created this list | No |

**Status Options:**
| Value | Description |
|-------|-------------|
| `active` | List is being used |
| `archived` | List is old, not actively using |
| `processing` | Currently importing/processing |

**Source Options:**
| Value | Description |
|-------|-------------|
| `csv_import` | Imported from CSV |
| `clay` | From Clay table |
| `instantly` | From Instantly platform |
| `smartlead` | From Smartlead platform |
| `manual` | Manually created |

**Create Table SQL:**
```sql
CREATE TABLE lead_lists (
    list_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    list_name VARCHAR NOT NULL,
    client_id VARCHAR NOT NULL,
    description TEXT,
    source VARCHAR,
    total_contacts INTEGER DEFAULT 0,
    contacts_in_campaigns INTEGER DEFAULT 0,
    status VARCHAR DEFAULT 'active',
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    created_by VARCHAR,
    UNIQUE(client_id, list_name)
);

CREATE INDEX idx_lead_lists_client ON lead_lists(client_id);
CREATE INDEX idx_lead_lists_status ON lead_lists(status);
CREATE INDEX idx_lead_lists_client_status ON lead_lists(client_id, status);
```

---

### 2. Contacts Table (Supabase)

**Purpose:** Central table for ALL contacts we've ever enriched/emailed - serves as our own lead database for deduplication and AI context

> **Note:** This table lives in Supabase for scalability. Contacts are promoted to Airtable Leads when they engage.

| Field Name | Type | Description | Required |
|------------|------|-------------|----------|
| `contact_id` | UUID | Primary key (auto-generated) | Yes |
| `client_id` | VARCHAR | Multi-tenant: which client this belongs to | Yes |
| `email` | VARCHAR | Primary email address | Yes |
| `first_name` | VARCHAR | Contact's first name | No |
| `last_name` | VARCHAR | Contact's last name | No |
| `phone` | VARCHAR | Primary phone number | No |
| `company_name` | VARCHAR | Company name | No |
| `job_title` | VARCHAR | Position/role | No |
| `linkedin_url` | VARCHAR | LinkedIn profile URL | No |
| `company_linkedin_url` | VARCHAR | Company LinkedIn URL | No |
| `company_website` | VARCHAR | Company website | No |
| `company_size` | VARCHAR | Company size (e.g., "10-50") | No |
| `company_industry` | VARCHAR | Industry | No |
| `source` | VARCHAR | Where contact came from | No |
| `list_id` | UUID | Link to lead_lists table (which batch/list this came from) | No |
| `enrichment_date` | TIMESTAMPTZ | When enriched via Clay | No |
| `lead_status` | VARCHAR | CRM relationship status | Yes |
| `pipeline_status` | VARCHAR | Sales pipeline stage | Yes |
| `campaign_status` | VARCHAR | Email sequence state | No |
| `lead_email_status` | VARCHAR | Email engagement outcome | No |
| `current_campaign_id` | VARCHAR | Active campaign ID | No |
| `past_campaign_ids` | JSONB | Array of past campaign IDs | No |
| `last_contact_date` | TIMESTAMPTZ | When we last reached out | No |
| `last_reply_date` | TIMESTAMPTZ | When they last replied | No |
| `last_touch_date` | TIMESTAMPTZ | Most recent interaction | No |
| `reply_count` | INTEGER | Total replies from contact | No |
| `emails_sent_count` | INTEGER | Total emails we've sent | No |
| `follow_up_date` | DATE | Scheduled follow-up date | No |
| `follow_up_reason` | VARCHAR | Why follow-up is scheduled | No |
| `has_engaged` | BOOLEAN | Has contact engaged? (triggers Airtable promotion) | No |
| `airtable_lead_id` | VARCHAR | Link to Airtable lead (if promoted) | No |
| `promoted_at` | TIMESTAMPTZ | When promoted to Airtable lead | No |
| `converted_to_client` | BOOLEAN | Is this contact now a client? | No |
| `converted_client_id` | VARCHAR | Link to clients table | No |
| `ai_context_summary` | TEXT | AI-generated summary | No |
| `created_at` | TIMESTAMPTZ | When record was created | Yes |
| `updated_at` | TIMESTAMPTZ | Last update time | Yes |

**Lead Status Options:**
| Value | Description |
|-------|-------------|
| `triage` | New lead, not yet contacted |
| `active` | In active outreach |
| `interested` | Showed positive interest |
| `not_interested` | Said no, may circle back later |
| `out_of_office` | Replied OOO, scheduled for follow-up |
| `ghosted` | Replied then stopped responding |
| `no_show` | Booked meeting but didn't show |
| `do_not_contact` | Requested removal, never contact again |
| `won` | Converted to client |
| `declined` | Final rejection after negotiation |

**Pipeline Status Options:**
| Value | Description |
|-------|-------------|
| `open` | In active pipeline |
| `follow_up` | Needs follow-up action |
| `meeting_booked` | Meeting scheduled |
| `closed` | Deal closed (won or lost) |

**Campaign Status Options:**
| Value | Description |
|-------|-------------|
| `null` | Not in any campaign |
| `active` | Sequence currently running |
| `paused` | Sequence temporarily paused |
| `completed` | Sequence ended |

**Lead Email Status Options:**
| Value | Description |
|-------|-------------|
| `null` | No outcome yet |
| `replied` | Lead replied to our emails |
| `no_reply` | All emails sent, no response |
| `bounced` | Email address bounced |
| `unsubscribed` | Requested removal |

**Follow-up Reason Options:**
| Value | Description |
|-------|-------------|
| `ghosted_follow_up` | Replied but stopped responding |
| `out_of_office_return` | Scheduled based on OOO date |
| `lead_requested` | Lead asked us to follow up later |
| `not_interested_re_engagement` | Circle back after 2-3 months |
| `meeting_no_show` | Follow up after missed meeting |
| `custom` | Other reason |

**Source Options:**
| Value | Description |
|-------|-------------|
| `clay` | Enriched via Clay |
| `apollo` | Scraped from Apollo |
| `sales_nav` | Scraped from Sales Navigator |
| `zoominfo` | From ZoomInfo |
| `google_maps` | From Google Maps |
| `manual` | Manually added |
| `import` | Bulk import |

---

### 2. Leads Table (Airtable)

**Purpose:** CRM workspace for ENGAGED contacts only - rich lead profiles for human sales work

> **Note:** Contacts are promoted to Leads when they reply, book a meeting, or otherwise engage. This keeps Airtable clean and focused.

| Field Name | Type | Description | Required |
|------------|------|-------------|----------|
| **IDENTITY** |
| `Lead ID` | Auto-number | Primary key | Yes |
| `Email` | Email | From Supabase contact | Yes |
| `First Name` | Single line text | From Supabase contact | No |
| `Last Name` | Single line text | From Supabase contact | No |
| `Full Name` | Formula | First + Last | No |
| `Phone` | Phone | From Supabase contact | No |
| **COMPANY** |
| `Company Name` | Single line text | From Supabase contact | No |
| `Job Title` | Single line text | From Supabase contact | No |
| `Company Website` | URL | From Supabase contact | No |
| `LinkedIn URL` | URL | From Supabase contact | No |
| **CRM STATUS** |
| `Lead Status` | Single select | Relationship status | Yes |
| `Pipeline Stage` | Single select | Sales stage | Yes |
| `Lead Score` | Number | Priority score (1-10) | No |
| `Assigned To` | Single select / User | Who's working this lead | No |
| **SALES PROCESS** |
| `Follow-up Date` | Date | Scheduled follow-up | No |
| `Follow-up Reason` | Single line text | Why following up | No |
| `Next Action` | Single line text | What to do next | No |
| **DEAL INFO** |
| `Interested In` | Link to Products | Product they're interested in | No |
| `Budget` | Currency | Estimated budget | No |
| `Timeline` | Single select | When they want to start | No |
| `Deal Value` | Currency | Potential deal value | No |
| `Decision Maker?` | Checkbox | Is this the decision maker | No |
| **QUALIFICATION (from your vision)** |
| `MRR Potential` | Currency | How much money MRR? | No |
| `Ready to Take Action?` | Single select | Yes, No, Maybe | No |
| `Willing to Invest?` | Single select | Yes, No, Unsure | No |
| `Biggest Challenge` | Long text | What's their biggest challenge? | No |
| **DISCOVERY DATA** |
| `Dreams` | Long text | What do they want to achieve? | No |
| `Goals` | Long text | Specific objectives | No |
| `Objections` | Long text | What's holding them back? | No |
| `Questions` | Long text | Open questions from them | No |
| `Blockers` | Long text | What's blocking the deal? | No |
| `Pain Points` | Long text | Identified pain points | No |
| **COMMUNICATION** |
| `Last Conversation Summary` | Long text | AI-generated summary | No |
| `Engagement Source` | Single select | How they engaged | No |
| `First Reply Date` | Date | When they first replied | No |
| **LINKS** |
| `Supabase Contact ID` | Single line text | Link back to contacts table | No |
| `Campaign` | Link to Campaigns | Related campaign | No |
| `Client` | Link to Clients | If converted | No |
| **NOTES** |
| `Internal Notes` | Long text | Internal team notes | No |
| `Call Notes` | Long text | Notes from calls | No |
| `Meeting Notes` | Long text | Notes from meetings | No |
| **SYSTEM** |
| `Created Date` | Created time | When promoted to lead | Yes |
| `Modified Date` | Last modified | Last update | Yes |
| `Created By` | Created by | Who created | Yes |

**Lead Status Options (Airtable):**
| Value | Description |
|-------|-------------|
| `New` | Just promoted, needs review |
| `Working` | Actively being worked |
| `Interested` | Showed positive interest |
| `Not Interested` | Said no |
| `Out of Office` | OOO, scheduled follow-up |
| `Ghosted` | Stopped responding |
| `No Show` | Missed meeting |
| `Qualified` | Ready for proposal/meeting |
| `Won` | Converted to client |
| `Lost` | Final rejection |

**Pipeline Stage Options (Airtable):**
| Value | Description |
|-------|-------------|
| `Engaged` | Replied, needs qualification |
| `Discovery` | In discovery phase |
| `Meeting Scheduled` | Meeting booked |
| `Meeting Complete` | Had meeting, deciding |
| `Proposal Sent` | Sent proposal |
| `Negotiation` | In negotiation |
| `Closed Won` | Deal closed |
| `Closed Lost` | Deal lost |

**Engagement Source Options:**
| Value | Description |
|-------|-------------|
| `Email Reply` | Replied to cold email |
| `Meeting Booked` | Booked meeting via link |
| `Inbound` | Came to us |
| `Referral` | Referred by someone |
| `Manual` | Manually promoted |

---

### 3. Lead Lists Table (Airtable)

**Purpose:** Plan and manage lead lists before importing to Supabase. Triggers automation when status changes to "active".

> **Note:** This is the planning/orchestration layer. When status = "active", it triggers creation in Supabase and import workflow.

| Field Name | Type | Description | Required | Constraints |
|------------|------|-------------|----------|-------------|
| **IDENTITY** |
| `Lead List ID` | Auto-number | Primary key | Yes | Unique |
| `List Name` | Single line text | Human-readable list name | Yes | - |
| `Client` | Link to Clients | Which client this belongs to | Yes | Links to Clients table |
| **STATUS & TRIGGER** |
| `Status` | Single select | List status (triggers automation) | Yes | Planning, Active, Processing, Completed, Archived |
| `Supabase List ID` | Single line text | UUID from Supabase lead_lists table | No | - |
| **SOURCE INFORMATION** |
| `Source Type` | Single select | Where leads come from | Yes | CSV Import, Clay Table, Google Sheet, Google Drive CSV, Manual, Other |
| `Source URL` | URL | Link to source (Clay table, Google Sheet, CSV, etc.) | No | - |
| `Source Description` | Long text | Additional source details | No | - |
| **METADATA** |
| `Description` | Long text | List description/purpose | No | - |
| `Target Audience` | Long text | Who we're targeting | No | - |
| `Filters Applied` | Long text | Filters used (e.g., "Company size: 50-100, Location: US") | No | - |
| `Expected Count` | Number | Expected number of contacts | No | - |
| **IMPORT TRACKING** |
| `Total Contacts` | Number | Total contacts imported | No | Formula/rollup |
| `Contacts in Campaigns` | Number | How many are in campaigns | No | Formula/rollup |
| `Import Started At` | Date & time | When import started | No | - |
| `Import Completed At` | Date & time | When import finished | No | - |
| `Last Sync` | Date & time | Last sync with Supabase | No | - |
| **AUTOMATION** |
| `Auto Create Supabase List` | Checkbox | Auto-create in Supabase when activated | Yes | Default: true |
| `Auto Enrich` | Checkbox | Auto-enrich contacts after import | No | Default: false |
| `Auto Add to Campaign` | Checkbox | Auto-add to campaign after import | No | Default: false |
| `Target Campaign` | Link to Campaigns | Campaign to add contacts to (if auto-add enabled) | No | Links to Campaigns table |
| **SYSTEM** |
| `Created Date` | Created time | When list was created | Yes | Auto |
| `Modified Date` | Last modified time | Last update | Yes | Auto |
| `Created By` | Created by | Who created | Yes | Auto |

**Status Options:**
| Value | Description | Triggers |
|-------|-------------|----------|
| `Planning` | List is being planned, not ready | None |
| `Active` | Ready to import - **TRIGGERS AUTOMATION** | Creates Supabase list, starts import |
| `Processing` | Currently importing contacts | None |
| `Completed` | Import finished successfully | None |
| `Archived` | List is old, not actively using | None |

**Source Type Options:**
| Value | Description | Source URL Format |
|-------|-------------|-------------------|
| `CSV Import` | CSV file upload | N/A (file uploaded) |
| `Clay Table` | Clay table link | `https://app.clay.com/...` |
| `Google Sheet` | Google Sheets link | `https://docs.google.com/spreadsheets/...` |
| `Google Drive CSV` | CSV file in Google Drive | `https://drive.google.com/file/...` |
| `Manual` | Manually created | N/A |
| `Other` | Other source | Custom URL |

**Airtable Views:**
- `Planning` - Status = Planning (ready to activate)
- `Active` - Status = Active (currently processing)
- `Completed` - Status = Completed (import done)
- `All Lists` - All lead lists
- `By Client` - Grouped by client

**Automation Trigger:**
- **When Status changes to "Active":**
  1. Create Supabase lead list (if `Auto Create Supabase List` = true)
  2. Start import workflow (fetch from Source URL if provided)
  3. Update `Supabase List ID` field
  4. Set Status to "Processing"
  5. When import completes, set Status to "Completed"

---

### 4. Campaigns Table (Airtable)

**Purpose:** Manage cold email campaigns and track performance

| Field Name | Type | Description | Required | Constraints |
|------------|------|-------------|----------|-------------|
| `campaign_id` | Auto-number / UUID | Primary key | Yes | Unique |
| `campaign_name` | Single line text | Campaign identifier | Yes | - |
| `campaign_description` | Long text | Purpose and details | No | - |
| `status` | Single select | Campaign status | Yes | Active, Paused, Completed |
| `platform` | Single select | Email platform used | Yes | Smart Lead, Instantly, Other |
| `platform_campaign_id` | Single line text | ID in external platform | No | - |
| `email_sequence` | Link to Documents | Attached email sequence | No | Links to Documents table |
| `target_audience` | Long text | Who we're targeting | No | - |
| `created_date` | Created time | Campaign creation date | Yes | Auto |
| `start_date` | Date | When campaign launched | No | - |
| `end_date` | Date | When campaign ended | No | - |
| `leads` | Link to Leads | All leads in campaign | No | Links to Leads table (multiple) |
| `total_leads` | Number | Count of leads | No | Formula/rollup |
| `emails_sent` | Number | Total emails sent | No | Updated by automation |
| `replies` | Number | Total replies | No | Updated by automation |
| `reply_rate` | Percent | Replies / Sent | No | Formula |
| `interested` | Number | Interested leads | No | Rollup from leads |
| `not_interested` | Number | Not interested leads | No | Rollup from leads |
| `meetings_booked` | Number | Meetings scheduled | No | Manual/auto update |
| `closed_deals` | Number | Deals won from campaign | No | Rollup from leads |
| `revenue_generated` | Currency | Total revenue | No | Rollup from deals |
| `last_analytics_update` | Date & time | Last time analytics updated | No | Auto |
| `webhook_url` | URL | Webhook for this campaign | No | - |
| `created_by` | Created by | Campaign creator | Yes | Auto |
| `notes` | Long text | Campaign notes | No | - |

---

### 5. Documents Table (Airtable)

**Purpose:** Single unified table for ALL vectorizable content - company knowledge, client documents, and sales/lead documents

> **Note:** Uses `namespace` field for Pinecone organization and `context` field for content type. Links to any related records.

| Field Name | Type | Description | Required | Constraints |
|------------|------|-------------|----------|-------------|
| **IDENTITY** |
| `document_id` | Auto-number | Primary key | Yes | Unique |
| `title` | Single line text | Document title | Yes | - |
| `description` | Long text | What this document contains | No | - |
| **CLASSIFICATION** |
| `namespace` | Single select | Pinecone namespace | Yes | See options below |
| `context` | Single select | Content type | Yes | See options below |
| `source` | Single select | Where it came from | No | See options below |
| `tags` | Multiple select | Categorization tags | No | - |
| **SOURCE CONTENT** |
| `source_url` | URL | Original URL (Google Drive, Notion, etc.) | No | - |
| `source_media` | Attachment | Original file upload | No | - |
| `mime_type` | Single line text | File type (video/mp4, etc.) | No | Auto-detected |
| **PROCESSED CONTENT** |
| `content_text` | Long text | Extracted/processed text | No | For vectorization |
| `content` | Attachment | Processed text file | No | Alternative to content_text |
| `transcription_url` | URL | Link to transcription | No | If audio/video |
| **RELATIONSHIPS** |
| `linked_leads` | Link to Leads | Related leads | No | Multiple |
| `linked_clients` | Link to Clients | Related clients | No | Multiple |
| `linked_campaigns` | Link to Campaigns | Related campaigns | No | Multiple |
| `linked_products` | Link to Products | Related products | No | Multiple |
| **VECTORIZATION** |
| `indexed` | Checkbox | Has been vectorized | No | Default: false |
| **AI PROCESSING** |
| `ai_summary` | Long text | AI-generated summary | No | Auto-generated |
| `key_points` | Long text | Main takeaways | No | Auto-generated |
| **MEDIA INFO** |
| `duration` | Number | If audio/video, minutes | No | - |
| `file_size` | Number | Size in MB | No | - |
| **PROCESSING STATUS** |
| `transcription_status` | Single select | Transcription status | No | Pending, In Progress, Complete, Failed |
| `processing_status` | Single select | Processing status | No | Pending, In Progress, Complete, Failed |
| **SYSTEM** |
| `created_date` | Created time | Upload date | Yes | Auto |
| `modified_date` | Last modified time | Last update | Yes | Auto |
| `created_by` | Created by | Who uploaded | Yes | Auto |

**Namespace Options (Pinecone):**
| Value | Description |
|-------|-------------|
| `company_knowledge` | General company/product info, sales process, best practices |
| `client_documents` | Client-specific docs (ICP, Goals, Offers, Client Calls) |
| `documents` | Sales calls, testimonials, case studies, email sequences |

**Context Options (Content Types):**

*Company Knowledge:*
- `Company Info` - About the company
- `Product Page` - Product information
- `Sales Process` - How we sell
- `Objection Handling` - Handling objections
- `Best Practice` - Best practices
- `Training` - Training materials
- `FAQ` - Frequently asked questions
- `Template` - Reusable templates
- `Pricing` - Pricing information

*Client Documents:*
- `ICP Document` - Client's Ideal Customer Profile
- `Goals Document` - Client goals & expectations
- `Offer Document` - Our proposal to client
- `Deliverables` - What we're delivering
- `Client Call` - Client call recording
- `Client Meeting Notes` - Client meeting notes
- `Contract` - Signed contract
- `Onboarding Form` - Onboarding responses
- `Client Feedback` - Client feedback

*Documents:*
- `Sales Call` - Sales call with lead
- `Discovery Call` - Discovery call
- `Demo Call` - Demo/presentation
- `Lead Meeting Notes` - Lead meeting notes
- `Testimonial` - Customer testimonial
- `Case Study` - Case study
- `Email Sequence` - Email sequence copy
- `Cold Email Copy` - Individual email copy
- `Lead Research` - Research on lead
- `Proposal` - Proposal to lead

**Source Options:**
| Value | Used For |
|-------|----------|
| `Google Drive` | Client docs, contracts |
| `Google Doc` | Any text document |
| `Notion` | Knowledge base, notes |
| `Zoom` | Call recordings |
| `Fathom` | AI meeting notes |
| `Google Meet` | Call recordings |
| `Loom` | Training videos, demos |
| `YouTube` | Testimonials, training |
| `Skool` | Community content |
| `Website` | Product pages, company info |
| `Typeform` | Forms, surveys |
| `Slack` | Conversations |
| `Email` | Email threads |
| `Other` | Catch-all |

**Link Requirements by Context:**
| Context | Required Links |
|---------|----------------|
| Any `client_documents` context | ✅ Client |
| Sales Call, Discovery, Demo, Lead Meeting Notes | ✅ Lead |
| Testimonial, Case Study | ✅ Client (who it's about) |
| Email Sequence, Cold Email Copy | ✅ Campaign |
| Product Page, Demo Call, Proposal | ✅ Product |

**Airtable Views:**
- `Knowledge > All` - All documents
- `Knowledge > Sales` - Sales-related content
- `Knowledge > Indexed` - Already vectorized
- `Automation > Transcribe` - Needs transcription
- `Automation > Summarize` - Needs AI summary
- `Automation > Index Row` - Ready to vectorize (indexed = false, content_text filled)

---

### 6. Products Table (Airtable)

**Purpose:** Store information about products/services we offer

| Field Name | Type | Description | Required | Constraints |
|------------|------|-------------|----------|-------------|
| `product_id` | Auto-number / UUID | Primary key | Yes | Unique |
| `product_name` | Single line text | Product name | Yes | - |
| `slug` | Single line text | URL-friendly name | No | - |
| `description` | Long text | Full description | No | - |
| `short_description` | Long text | Brief description | No | Max 200 chars |
| `price` | Currency | Product price | No | USD |
| `payment_schedule` | Single select | Billing frequency | No | One-time, Monthly, Quarterly, Yearly |
| `ideal_client` | Long text | Who this is for | No | - |
| `bad_client` | Long text | Who this is NOT for | No | - |
| `features` | Long text | Key features | No | - |
| `benefits` | Long text | Main benefits | No | - |
| `product_pages` | Link to Documents | Related pages/docs | No | Links to Documents table (multiple) |
| `lead_magnet` | Link to Documents | Free offer | No | Links to Documents table |
| `interested_leads` | Link to Leads | Leads interested in this | No | Links to Leads table (multiple) |
| `clients_using` | Link to Clients | Clients using this product | No | Links to Clients table (multiple) |
| `is_active` | Checkbox | Currently offered | Yes | Default: true |
| `created_date` | Created time | When created | Yes | Auto |
| `modified_date` | Last modified time | Last update | Yes | Auto |

---

### 7. Clients Table (Airtable)

**Purpose:** Manage converted leads who are now paying clients

| Field Name | Type | Description | Required | Constraints |
|------------|------|-------------|----------|-------------|
| `client_id` | Auto-number / UUID | Primary key | Yes | Unique |
| `client_name` | Single line text | Client/company name | Yes | - |
| `converted_from_lead` | Link to Leads | Original lead record | No | Links to Leads table |
| `conversion_date` | Date | When became client | Yes | - |
| `primary_contact_name` | Single line text | Main contact person | No | - |
| `primary_contact_email` | Email | Main email | Yes | - |
| `company_website` | URL | Client website | No | - |
| `client_status` | Single select | Current status | Yes | Active, On Hold, Churned |
| `products_using` | Link to Products | Products client has | No | Links to Products table (multiple) |
| `monthly_recurring_revenue` | Currency | MRR from this client | No | - |
| `total_revenue` | Currency | Lifetime value | No | Rollup/formula |
| `goals_document` | Link to Documents | Client goals | No | Links to Documents table |
| `expectations_document` | Link to Documents | Expectations doc | No | Links to Documents table |
| `icp_document` | Link to Documents | Their ICP | No | Links to Documents table |
| `offer_document` | Link to Documents | Our offer to them | No | Links to Documents table |
| `deliverables_document` | Link to Documents | Deliverables doc | No | Links to Documents table |
| `all_documents` | Link to Documents | All client docs | No | Links to Documents table (multiple) |
| `slack_channel` | Single line text | Slack channel name/ID | No | - |
| `slack_workspace` | Single line text | Slack workspace | No | - |
| `communication_history` | Link to Chat History | All communications | No | Links to Chat History table (multiple) |
| `account_manager` | Single select / User | Assigned team member | No | - |
| `client_health_score` | Number | 1-10 satisfaction score | No | 1-10 |
| `last_interaction_date` | Date & time | Last communication | No | Auto-updated |
| `next_review_date` | Date | Scheduled review | No | - |
| `notes` | Long text | Client notes | No | - |
| `created_date` | Created time | When created | Yes | Auto |
| `modified_date` | Last modified time | Last update | Yes | Auto |
| `ai_context_summary` | Long text | AI summary of client | No | Auto-generated |

**Client Status Options:**
- `Active` - Currently active client
- `On Hold` - Temporarily paused
- `Churned` - No longer a client

---

### 7. Chat History Table (Supabase)

**Purpose:** Store all communication history across channels

> **Note:** This table lives in Supabase for scalability. See [Chat History Schema](./docs/schemas/chat-history-schema.md) for complete documentation.

| Field Name | Type | Description | Required |
|------------|------|-------------|----------|
| `chat_id` | UUID | Primary key (auto-generated) | Yes |
| `session_id` | VARCHAR | Contact's primary identifier (for n8n AI memory) | Yes |
| `lead_id` | VARCHAR | Links to leads table | No |
| `client_id` | VARCHAR | Links to clients table (if converted) | No |
| `campaign_id` | VARCHAR | Links to campaigns table | No |
| `platform` | VARCHAR | Communication channel | Yes |
| `contact_identifier` | VARCHAR | Their address: email, phone, WhatsApp | No |
| `contact_name` | VARCHAR | Their name | No |
| `account_identifier` | VARCHAR | Our address: email, phone | No |
| `subject` | VARCHAR | Email subject (if applicable) | No |
| `message_content` | TEXT | The actual message | Yes |
| `message_type` | VARCHAR | Type of message | No |
| `message_status` | VARCHAR | Direction + delivery status | No |
| `sequence_step` | INTEGER | Campaign step number | No |
| `external_message_id` | VARCHAR | Platform's message ID | No |
| `thread_id` | VARCHAR | Thread ID for threading | No |
| `ai_sentiment` | VARCHAR | AI-detected sentiment | No |
| `ai_intent` | VARCHAR | AI-detected intent | No |
| `is_auto_reply` | BOOLEAN | Is this an auto-reply? | No |
| `requires_action` | BOOLEAN | Needs human attention? | No |
| `action_taken` | VARCHAR | What action was taken | No |
| `is_vectorized` | BOOLEAN | Added to Pinecone? | No |
| `vector_id` | VARCHAR | Pinecone vector ID | No |
| `message_timestamp` | TIMESTAMPTZ | When message was sent/received | Yes |
| `created_at` | TIMESTAMPTZ | Record creation time | Yes |

**Platform Options:**
- `gmail`, `smartlead`, `instantly`, `slack`, `whatsapp`, `sms`, `phone_call`, `zoom`, `n8n_chat`

**Message Type Options:**
- `campaign` - Automated campaign email
- `reply` - Reply in conversation
- `manual` - Manual one-off message
- `meeting` - Meeting notes/booking
- `follow_up` - Manual follow-up

**Message Status Options:**
- `sent` - We sent it (outgoing)
- `received` - We received it (incoming)
- `bounced` - Email bounced
- `failed` - Failed to send

---

### ~~8. Knowledge Base Table~~ (MERGED INTO DOCUMENTS)

> **Note:** The Knowledge Base table has been merged into the Documents table. Use `namespace = 'company_knowledge'` to filter for company knowledge content.

---

### 8. Automation Logs Table (Supabase)

**Purpose:** Track all automated actions taken by the system

| Field Name | Type | Description | Required | Constraints |
|------------|------|-------------|----------|-------------|
| `log_id` | UUID | Primary key | Yes | Unique |
| `automation_type` | VARCHAR | Type of automation | Yes | See types below |
| `trigger_type` | VARCHAR | What triggered it | Yes | See triggers below |
| `lead_id` | VARCHAR | Related lead | No | - |
| `client_id` | VARCHAR | Related client | No | - |
| `campaign_id` | VARCHAR | Related campaign | No | - |
| `action_taken` | VARCHAR | What action was performed | Yes | - |
| `action_details` | TEXT | Full details | No | - |
| `status` | VARCHAR | Action status | Yes | Success, Failed, Pending |
| `error_message` | TEXT | If failed, why | No | - |
| `timestamp` | TIMESTAMPTZ | When action occurred | Yes | Auto |
| `processing_time` | INTEGER | Time taken in ms | No | - |
| `ai_model_used` | VARCHAR | Which AI model | No | - |
| `tokens_used` | INTEGER | API tokens consumed | No | - |
| `created_at` | TIMESTAMPTZ | Log creation | Yes | Auto |

**Automation Type Options:**
- `Email Reply`
- `Follow-up Email`
- `Status Update`
- `Campaign Removal`
- `Analytics Update`
- `Vectorization`
- `Transcription`
- `Lead Scoring`
- `Task Creation`
- `Other`

**Trigger Type Options:**
- `Webhook`
- `Daily Cron`
- `Hourly Cron`
- `Manual`
- `API Call`
- `Condition Met`
- `Other`

---

## Vector Database Schema (Pinecone)

### Vector Structure

Each vector in Pinecone includes:

**Vector ID Format:** `{type}_{id}_{chunk_number}`
- Example: `doc_123_1`, `email_456_0`, `call_789_2`

**Metadata Structure:**

```json
{
  "id": "doc_123_1",
  "document_id": "doc_123",
  "context": "sales_call",
  "title": "Sales call with John Smith",
  "linked_lead_id": "lead_456",
  "linked_lead_name": "John Smith",
  "linked_client_id": null,
  "linked_campaign_id": "campaign_789",
  "linked_product_id": null,
  "source": "zoom",
  "date": "2026-01-05",
  "timestamp": 1704470400,
  "tags": ["interested", "pricing_question"],
  "chunk_index": 1,
  "total_chunks": 3,
  "content_preview": "First 100 chars of content..."
}
```

**Namespace Organization (3 Namespaces):**
| Namespace | What's Stored | Example Content |
|-----------|---------------|-----------------|
| `company_knowledge` | General company/product info | Company info, product pages, sales process, best practices, FAQs, templates |
| `client_documents` | Client-specific documents | ICP docs, goals, offers, deliverables, client calls, contracts |
| `documents` | Sales/lead/campaign content | Sales calls, testimonials, case studies, email sequences, proposals |

---

## Table Relationships

```
SUPABASE:
Lead Lists (1) ←→ (Many) Contacts (via list_id)
Contacts (1) ←→ (Many) Chat History
Contacts (1) ←→ (1) Leads (Airtable, via airtable_lead_id)
Contacts (Many) ←→ (Many) Campaigns (via current_campaign_id, past_campaign_ids)

AIRTABLE:
Lead Lists (1) ←→ (1) Lead Lists (Supabase, via supabase_list_id)
Lead Lists (1) ←→ (Many) Campaigns (via target_campaign)
Lead Lists (1) ←→ (1) Clients

Leads (1) ←→ (1) Contacts (Supabase, via supabase_contact_id)
Leads (1) ←→ (Many) Documents
Leads (Many) ←→ (Many) Campaigns
Leads (Many) ←→ (Many) Products
Leads (1) ←→ (1) Clients

Clients (1) ←→ (Many) Chat History
Clients (1) ←→ (Many) Documents
Clients (Many) ←→ (Many) Products

Campaigns (1) ←→ (Many) Leads
Campaigns (1) ←→ (Many) Contacts (Supabase)
Campaigns (1) ←→ (Many) Documents
Campaigns (1) ←→ (Many) Chat History

Documents (Many) ←→ (Many) Leads
Documents (Many) ←→ (Many) Clients
Documents (Many) ←→ (Many) Campaigns
Documents (Many) ←→ (Many) Products

Products (Many) ←→ (Many) Leads
Products (Many) ←→ (Many) Clients
Products (1) ←→ (Many) Documents

Knowledge Base → All Tables (for AI context)
Automation Logs → All Tables (for tracking)
```

### Contact → Lead Promotion Flow

```
┌─────────────────────────────────────────────────────────────┐
│  SUPABASE (contacts)                                        │
│  - Everyone we've ever emailed                              │
│  - Deduplication: Check before enriching                    │
│  - Campaign tracking                                        │
└─────────────────────────────────────────────────────────────┘
                         ↓
              [Contact Engages: Reply, Meeting, etc.]
                         ↓
┌─────────────────────────────────────────────────────────────┐
│  n8n Workflow                                               │
│  1. Create Lead in Airtable                                 │
│  2. Update Supabase: has_engaged=true, airtable_lead_id     │
│  3. Log in chat_history                                     │
└─────────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────────┐
│  AIRTABLE (Leads)                                           │
│  - Only engaged contacts                                    │
│  - Rich CRM data (notes, deal info, qualification)          │
│  - Human sales workflow                                     │
└─────────────────────────────────────────────────────────────┘
```

---

## Complete State Matrix (4 Dimensions) - Contacts Table

| lead_status | pipeline_status | campaign_status | lead_email_status | Meaning |
|-------------|-----------------|-----------------|-------------------|---------|
| `triage` | `open` | `null` | `null` | New lead, never contacted |
| `active` | `open` | `active` | `null` | In email sequence |
| `active` | `open` | `completed` | `no_reply` | All emails sent, no response |
| `active` | `open` | `completed` | `bounced` | Email bounced |
| `out_of_office` | `follow_up` | `paused` | `replied` | OOO auto-reply |
| `interested` | `follow_up` | `completed` | `replied` | Positive reply |
| `interested` | `meeting_booked` | `completed` | `replied` | Meeting scheduled |
| `not_interested` | `closed` | `completed` | `replied` | Said no |
| `ghosted` | `follow_up` | `completed` | `replied` | Replied then silent |
| `no_show` | `follow_up` | `completed` | `replied` | Missed meeting |
| `do_not_contact` | `closed` | `completed` | `unsubscribed` | Requested removal |
| `won` | `closed` | `completed` | `replied` | Converted to client |
| `declined` | `closed` | `completed` | `replied` | Final rejection |

---

## Indexes & Performance

**Recommended Indexes (Supabase):**

```sql
-- Contacts table
CREATE INDEX idx_contacts_client ON contacts(client_id);
CREATE INDEX idx_contacts_email ON contacts(email);
CREATE INDEX idx_contacts_status ON contacts(lead_status);
CREATE INDEX idx_contacts_pipeline ON contacts(pipeline_status);
CREATE INDEX idx_contacts_campaign_status ON contacts(campaign_status);
CREATE INDEX idx_contacts_email_status ON contacts(lead_email_status);
CREATE INDEX idx_contacts_follow_up ON contacts(follow_up_date);
CREATE INDEX idx_contacts_last_reply ON contacts(last_reply_date);
CREATE INDEX idx_contacts_current_campaign ON contacts(current_campaign_id);
CREATE INDEX idx_contacts_engaged ON contacts(has_engaged);
CREATE INDEX idx_contacts_source ON contacts(source);
CREATE INDEX idx_contacts_list_id ON contacts(list_id);
CREATE INDEX idx_contacts_client_list ON contacts(client_id, list_id);

-- Lead Lists table
CREATE INDEX idx_lead_lists_client ON lead_lists(client_id);
CREATE INDEX idx_lead_lists_status ON lead_lists(status);
CREATE INDEX idx_lead_lists_client_status ON lead_lists(client_id, status);

-- Chat History table
CREATE INDEX idx_chat_session ON chat_history(session_id);
CREATE INDEX idx_chat_lead ON chat_history(lead_id);
CREATE INDEX idx_chat_client ON chat_history(client_id);
CREATE INDEX idx_chat_campaign ON chat_history(campaign_id);
CREATE INDEX idx_chat_timestamp ON chat_history(message_timestamp DESC);
CREATE INDEX idx_chat_thread ON chat_history(thread_id);
CREATE INDEX idx_chat_platform ON chat_history(platform);
CREATE INDEX idx_chat_status ON chat_history(message_status);
CREATE INDEX idx_chat_lead_timestamp ON chat_history(lead_id, message_timestamp DESC);
```

---

## Data Validation Rules

1. **Email Uniqueness**: Leads cannot have duplicate emails within the same client_id
2. **Required Fields**: email and client_id are required for leads
3. **Status Transitions**: `do_not_contact` leads cannot be changed to any other status
4. **Campaign Links**: A lead can only be in ONE active campaign at a time
5. **Client Conversion**: Converting lead to client requires lead_status = "won"
6. **Follow-up Dates**: Cannot be in the past
7. **Document Vectorization**: Must have processed_text before vectorization

---

## Related Documentation

- [Leads Schema (Supabase)](./docs/schemas/leads-schema.md) - Complete leads table documentation
- [Chat History Schema (Supabase)](./docs/schemas/chat-history-schema.md) - Complete chat_history documentation
- [Lead Journey Map](./docs/schemas/lead-journey-map.md) - Complete branching logic and workflows

---

**Last Updated:** Nov 4, 2025  
**Version:** 2.0
