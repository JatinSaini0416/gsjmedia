# 🎯 AI Sales Engine CRM - Complete Schema & Logic Documentation

> **Purpose:** Complete database schema, state matrix, and branching logic for the Lead Data Architecture
> 
> **Created:** Sep 24, 2026  
> **Updated:** Sep 24, 2026
> **Status:** Final - Ready for Implementation

---

## Architecture Overview

| Database | Table | Purpose | Volume |
|----------|-------|---------|--------|
| **Supabase** | `contacts` | Everyone we've ever emailed (data warehouse, deduplication) | 100K+ |
| **Supabase** | `chat_history` | All communications (AI memory) | Unlimited |
| **Airtable** | `Leads` | Engaged contacts only (CRM workspace) | ~5-10% of contacts |
| **Pinecone** | vectors | AI context, semantic search | As needed |

---

## Table of Contents

1. [Complete Contacts Table Schema](#part-1-complete-contacts-table-schema-supabase)
2. [Complete Chat History Table Schema](#part-2-complete-chat-history-table-schema)
3. [Complete State Matrix (All 4 Dimensions)](#part-3-complete-state-matrix-all-4-dimensions)
4. [Complete Branching Logic (Every Field Updated)](#part-4-complete-branching-logic-every-field-updated)
5. [Quick Reference Cards](#part-5-quick-reference-cards)

---

## Part 1: Complete Contacts Table Schema (Supabase)

> **Architecture Note:** The `contacts` table stores EVERYONE we've ever enriched/emailed. Contacts are promoted to Airtable `Leads` when they engage. This keeps Airtable clean and gives us our own lead database for deduplication.

```sql
-- ============================================================================
-- CONTACTS TABLE - Supabase Schema
-- Purpose: Central table for ALL contacts - our own lead database
-- ============================================================================

CREATE TABLE contacts (
    -- ========== IDENTITY ==========
    contact_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id VARCHAR NOT NULL,              -- Multi-tenant: which client this belongs to
    
    -- ========== CONTACT INFO (from Clay enrichment) ==========
    email VARCHAR NOT NULL,                  -- Primary email address
    first_name VARCHAR,
    last_name VARCHAR,
    phone VARCHAR,                           -- Primary phone number
    
    -- ========== COMPANY INFO (from Clay enrichment) ==========
    company_name VARCHAR,
    job_title VARCHAR,
    linkedin_url VARCHAR,
    company_linkedin_url VARCHAR,
    company_website VARCHAR,
    company_size VARCHAR,                    -- e.g., "10-50", "51-200"
    company_industry VARCHAR,
    
    -- ========== SOURCE & ENRICHMENT ==========
    source VARCHAR,                          -- clay, apollo, sales_nav, zoominfo, google_maps, manual
    enrichment_date TIMESTAMPTZ,             -- When enriched via Clay
    
    -- ========== CRM STATUS (Relationship Quality) ==========
    lead_status VARCHAR DEFAULT 'triage',
    -- Options: triage, active, interested, not_interested, out_of_office, 
    --          ghosted, no_show, do_not_contact, won, declined
    
    -- ========== PIPELINE STATUS (Sales Stage) ==========
    pipeline_status VARCHAR DEFAULT 'open',
    -- Options: open, follow_up, meeting_booked, closed
    
    -- ========== CAMPAIGN STATUS (Email Sequence State) ==========
    campaign_status VARCHAR,
    -- Options: null (not in campaign), active, paused, completed
    
    -- ========== LEAD EMAIL STATUS (Email Engagement Outcome) ==========
    lead_email_status VARCHAR,
    -- Options: null (no outcome), replied, no_reply, bounced, unsubscribed
    
    -- ========== CAMPAIGN ASSOCIATION ==========
    current_campaign_id VARCHAR,             -- Active campaign ID (null if not in one)
    past_campaign_ids JSONB DEFAULT '[]',    -- Array of past campaign IDs
    
    -- ========== TIMESTAMPS ==========
    last_contact_date TIMESTAMPTZ,           -- When WE last reached out
    last_reply_date TIMESTAMPTZ,             -- When THEY last replied
    last_touch_date TIMESTAMPTZ,             -- Most recent interaction (either way)
    
    -- ========== ENGAGEMENT METRICS ==========
    reply_count INTEGER DEFAULT 0,           -- Total replies from this contact
    emails_sent_count INTEGER DEFAULT 0,     -- Total emails we've sent
    
    -- ========== FOLLOW-UP SCHEDULING ==========
    follow_up_date DATE,                     -- Scheduled follow-up date
    follow_up_reason VARCHAR,                -- Why follow-up is scheduled
    -- Options: ghosted_follow_up, out_of_office_return, lead_requested, 
    --          not_interested_re_engagement, meeting_no_show, custom
    
    -- ========== AIRTABLE PROMOTION ==========
    has_engaged BOOLEAN DEFAULT FALSE,       -- Has contact engaged? (reply, meeting, etc.)
    airtable_lead_id VARCHAR,                -- Link to Airtable lead (if promoted)
    promoted_at TIMESTAMPTZ,                 -- When promoted to Airtable lead
    
    -- ========== CONVERSION ==========
    converted_to_client BOOLEAN DEFAULT FALSE,
    converted_client_id VARCHAR,             -- Link to clients table if converted
    
    -- ========== AI CONTEXT ==========
    ai_context_summary TEXT,                 -- AI-generated summary of contact
    
    -- ========== SYSTEM ==========
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    
    -- ========== CONSTRAINTS ==========
    UNIQUE(email, client_id)
);

-- ========== INDEXES ==========
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
```

---

## Part 2: Complete Chat History Table Schema

```sql
-- ============================================================================
-- CHAT_HISTORY TABLE - Supabase Schema
-- Purpose: Store all communication history across all platforms
-- ============================================================================

CREATE TABLE chat_history (
    -- ========== IDENTITY ==========
    chat_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id VARCHAR NOT NULL,             -- Contact's primary identifier (for n8n AI memory)
    
    -- ========== LINKS ==========
    lead_id VARCHAR,                         -- Links to leads table
    client_id VARCHAR,                       -- Links to clients table (if converted)
    campaign_id VARCHAR,                     -- Links to campaigns table
    
    -- ========== PLATFORM ==========
    platform VARCHAR,
    -- Options: gmail, smartlead, instantly, slack, whatsapp, sms, phone_call, zoom, n8n_chat
    
    -- ========== PARTICIPANTS ==========
    contact_identifier VARCHAR,              -- Their address: email, phone, WhatsApp, Slack ID
    contact_name VARCHAR,                    -- Their name
    account_identifier VARCHAR,              -- Our address: email, phone, WhatsApp business number
    
    -- ========== MESSAGE CONTENT ==========
    subject VARCHAR,                         -- Email subject (if applicable)
    message_content TEXT NOT NULL,           -- The actual message
    
    -- ========== MESSAGE CLASSIFICATION ==========
    message_type VARCHAR,
    -- Options: campaign, reply, manual, meeting, follow_up
    
    message_status VARCHAR,
    -- Options: sent, received, bounced, failed
    
    sequence_step INTEGER,                   -- Campaign step number (null if not campaign)
    
    -- ========== THREADING ==========
    external_message_id VARCHAR,             -- Platform's message ID (for replying)
    thread_id VARCHAR,                       -- Thread ID (for reply threading)
    
    -- ========== AI ANALYSIS ==========
    ai_sentiment VARCHAR,
    -- Options: positive, neutral, negative, interested, not_interested
    
    ai_intent VARCHAR,
    -- Options: interested_ready_to_talk, interested_wants_info, pricing_question,
    --          not_interested, not_interested_timing, out_of_office, 
    --          question_about_product, objection_price, objection_timing,
    --          objection_has_solution, unsubscribe, other
    
    is_auto_reply BOOLEAN DEFAULT FALSE,     -- Is this an auto-reply (OOO, bounce-back)
    
    -- ========== ACTION TRACKING ==========
    requires_action BOOLEAN DEFAULT FALSE,   -- Does human need to look at this?
    action_taken VARCHAR,                    -- What action was taken
    
    -- ========== VECTORIZATION ==========
    is_vectorized BOOLEAN DEFAULT FALSE,     -- Has been added to Pinecone?
    vector_id VARCHAR,                       -- Pinecone vector ID
    
    -- ========== TIMESTAMPS ==========
    message_timestamp TIMESTAMPTZ DEFAULT NOW(),  -- When message was sent/received
    created_at TIMESTAMPTZ DEFAULT NOW()          -- When record was created
);

-- ========== INDEXES ==========
CREATE INDEX idx_chat_session ON chat_history(session_id);
CREATE INDEX idx_chat_lead ON chat_history(lead_id);
CREATE INDEX idx_chat_client ON chat_history(client_id);
CREATE INDEX idx_chat_campaign ON chat_history(campaign_id);
CREATE INDEX idx_chat_timestamp ON chat_history(message_timestamp DESC);
CREATE INDEX idx_chat_thread ON chat_history(thread_id);
CREATE INDEX idx_chat_platform ON chat_history(platform);
CREATE INDEX idx_chat_status ON chat_history(message_status);
CREATE INDEX idx_chat_type ON chat_history(message_type);
CREATE INDEX idx_chat_lead_timestamp ON chat_history(lead_id, message_timestamp DESC);
```

---

## Part 3: Complete State Matrix (All 4 Dimensions)

### Status Field Reference

| Field | Purpose | Options |
|-------|---------|---------|
| `lead_status` | Relationship quality | triage, active, interested, not_interested, out_of_office, ghosted, no_show, do_not_contact, won, declined |
| `pipeline_status` | Sales stage | open, follow_up, meeting_booked, closed |
| `campaign_status` | Email sequence state | null, active, paused, completed |
| `lead_email_status` | Email engagement outcome | null, replied, no_reply, bounced, unsubscribed |

### Complete State Matrix

| lead_status | pipeline_status | campaign_status | lead_email_status | Meaning | Next Action |
|-------------|-----------------|-----------------|-------------------|---------|-------------|
| `triage` | `open` | `null` | `null` | New lead from Clay, never contacted | Add to campaign |
| `active` | `open` | `active` | `null` | In email sequence, waiting for response | Monitor for events |
| `active` | `open` | `paused` | `null` | Sequence paused (manual) | Resume or reassign |
| `active` | `open` | `completed` | `no_reply` | All emails sent, no response | Add to re-engagement (2-3 months) |
| `active` | `open` | `completed` | `bounced` | Email bounced | Re-enrich in Clay |
| `out_of_office` | `follow_up` | `paused` | `replied` | OOO auto-reply received | Wait for return date |
| `interested` | `follow_up` | `completed` | `replied` | Positive reply, needs follow-up | Send reply, book meeting |
| `interested` | `meeting_booked` | `completed` | `replied` | Meeting scheduled | Prepare for meeting |
| `not_interested` | `closed` | `completed` | `replied` | Said no | Archive, re-engage in 6 months |
| `ghosted` | `follow_up` | `completed` | `replied` | Replied then went silent | Send follow-up sequence |
| `no_show` | `follow_up` | `completed` | `replied` | Booked but didn't show | Follow-up to reschedule |
| `do_not_contact` | `closed` | `completed` | `unsubscribed` | Requested removal | Never contact again |
| `won` | `closed` | `completed` | `replied` | Converted to client | Start onboarding |
| `declined` | `closed` | `completed` | `replied` | Final rejection after negotiation | Archive |

---

## Part 4: Complete Branching Logic (Every Field Updated)

### STEP 1: Lead Created (from Clay)

```
╔══════════════════════════════════════════════════════════════════════════════╗
║ TRIGGER: Clay webhook / n8n import                                           ║
╠══════════════════════════════════════════════════════════════════════════════╣
║                                                                              ║
║  INSERT INTO leads:                                                          ║
║  ══════════════════                                                          ║
║  • lead_id: gen_random_uuid()                                                ║
║  • client_id: [from config/webhook]                                          ║
║  • email: [from Clay]                                                        ║
║  • first_name: [from Clay]                                                   ║
║  • last_name: [from Clay]                                                    ║
║  • phone: [from Clay, if available]                                          ║
║  • company_name: [from Clay]                                                 ║
║  • job_title: [from Clay]                                                    ║
║  • linkedin_url: [from Clay, if available]                                   ║
║  • company_linkedin_url: [from Clay, if available]                           ║
║  • company_website: [from Clay, if available]                                ║
║  • company_size: [from Clay, if available]                                   ║
║  • company_industry: [from Clay, if available]                               ║
║  • lead_status: 'triage'                                                     ║
║  • pipeline_status: 'open'                                                   ║
║  • campaign_status: null                                                     ║
║  • lead_email_status: null                                                   ║
║  • current_campaign_id: null                                                 ║
║  • past_campaign_ids: []                                                     ║
║  • last_contact_date: null                                                   ║
║  • last_reply_date: null                                                     ║
║  • last_touch_date: null                                                     ║
║  • reply_count: 0                                                            ║
║  • emails_sent_count: 0                                                      ║
║  • follow_up_date: null                                                      ║
║  • follow_up_reason: null                                                    ║
║  • converted_to_client: false                                                ║
║  • converted_client_id: null                                                 ║
║  • ai_context_summary: null                                                  ║
║  • created_at: NOW()                                                         ║
║  • updated_at: NOW()                                                         ║
║                                                                              ║
╚══════════════════════════════════════════════════════════════════════════════╝
```

---

### STEP 2: Added to Campaign

```
╔══════════════════════════════════════════════════════════════════════════════╗
║ TRIGGER: Clay pushes to Instantly/Smartlead OR manual add                    ║
╠══════════════════════════════════════════════════════════════════════════════╣
║                                                                              ║
║  UPDATE leads SET:                                                           ║
║  ═════════════════                                                           ║
║  • lead_status: 'active'                   ← changed from 'triage'           ║
║  • pipeline_status: 'open'                 ← stays same                      ║
║  • campaign_status: 'active'               ← changed from null               ║
║  • lead_email_status: null                 ← stays same                      ║
║  • current_campaign_id: [campaign_id]      ← set to campaign ID              ║
║  • updated_at: NOW()                                                         ║
║                                                                              ║
║  WHERE lead_id = [lead_id]                                                   ║
║                                                                              ║
╚══════════════════════════════════════════════════════════════════════════════╝
```

---

### STEP 3: Email Sent (Campaign or Manual)

```
╔══════════════════════════════════════════════════════════════════════════════╗
║ TRIGGER: Instantly/Smartlead webhook (email sent)                            ║
╠══════════════════════════════════════════════════════════════════════════════╣
║                                                                              ║
║  INSERT INTO chat_history:                                                   ║
║  ═════════════════════════                                                   ║
║  • chat_id: gen_random_uuid()                                                ║
║  • session_id: [lead's email]                                                ║
║  • lead_id: [lead_id]                                                        ║
║  • client_id: null                                                           ║
║  • campaign_id: [campaign_id]                                                ║
║  • platform: 'smartlead' OR 'instantly'                                      ║
║  • contact_identifier: [lead's email]                                        ║
║  • contact_name: [lead's full name]                                          ║
║  • account_identifier: [our sending email]                                   ║
║  • subject: [email subject]                                                  ║
║  • message_content: [email body]                                             ║
║  • message_type: 'campaign'                                                  ║
║  • message_status: 'sent'                                                    ║
║  • sequence_step: [step number: 1, 2, 3...]                                  ║
║  • external_message_id: [from webhook]                                       ║
║  • thread_id: [from webhook]                                                 ║
║  • ai_sentiment: null                                                        ║
║  • ai_intent: null                                                           ║
║  • is_auto_reply: false                                                      ║
║  • requires_action: false                                                    ║
║  • action_taken: null                                                        ║
║  • is_vectorized: false                                                      ║
║  • vector_id: null                                                           ║
║  • message_timestamp: [from webhook]                                         ║
║  • created_at: NOW()                                                         ║
║                                                                              ║
║  ─────────────────────────────────────────────────────────────────────────   ║
║                                                                              ║
║  UPDATE leads SET:                                                           ║
║  ═════════════════                                                           ║
║  • lead_status: 'active'                   ← stays same                      ║
║  • pipeline_status: 'open'                 ← stays same                      ║
║  • campaign_status: 'active'               ← stays same                      ║
║  • lead_email_status: null                 ← stays same                      ║
║  • last_contact_date: [message_timestamp]  ← updated                         ║
║  • last_touch_date: [message_timestamp]    ← updated                         ║
║  • emails_sent_count: emails_sent_count + 1 ← incremented                    ║
║  • updated_at: NOW()                                                         ║
║                                                                              ║
║  WHERE lead_id = [lead_id]                                                   ║
║                                                                              ║
╚══════════════════════════════════════════════════════════════════════════════╝
```

---

### EVENT A.1: Lead Replies - INTERESTED

```
╔══════════════════════════════════════════════════════════════════════════════╗
║ TRIGGER: Instantly/Smartlead webhook (reply received)                        ║
║ CONDITION: AI analysis detects INTERESTED intent                             ║
╠══════════════════════════════════════════════════════════════════════════════╣
║                                                                              ║
║  INSERT INTO chat_history:                                                   ║
║  ═════════════════════════                                                   ║
║  • chat_id: gen_random_uuid()                                                ║
║  • session_id: [lead's email]                                                ║
║  • lead_id: [lead_id]                                                        ║
║  • client_id: null                                                           ║
║  • campaign_id: [campaign_id]                                                ║
║  • platform: 'smartlead' OR 'instantly'                                      ║
║  • contact_identifier: [lead's email]                                        ║
║  • contact_name: [lead's full name]                                          ║
║  • account_identifier: [our email they replied to]                           ║
║  • subject: [reply subject]                                                  ║
║  • message_content: [reply body]                                             ║
║  • message_type: 'reply'                                                     ║
║  • message_status: 'received'                                                ║
║  • sequence_step: null                                                       ║
║  • external_message_id: [from webhook]                                       ║
║  • thread_id: [from webhook]                                                 ║
║  • ai_sentiment: 'positive' OR 'interested'                                  ║
║  • ai_intent: 'interested_ready_to_talk' OR 'interested_wants_info'          ║
║  • is_auto_reply: false                                                      ║
║  • requires_action: true                                                     ║
║  • action_taken: null                                                        ║
║  • is_vectorized: false                                                      ║
║  • vector_id: null                                                           ║
║  • message_timestamp: [from webhook]                                         ║
║  • created_at: NOW()                                                         ║
║                                                                              ║
║  ─────────────────────────────────────────────────────────────────────────   ║
║                                                                              ║
║  UPDATE leads SET:                                                           ║
║  ═════════════════                                                           ║
║  • lead_status: 'interested'               ← changed from 'active'           ║
║  • pipeline_status: 'follow_up'            ← changed from 'open'             ║
║  • campaign_status: 'completed'            ← changed from 'active'           ║
║  • lead_email_status: 'replied'            ← changed from null               ║
║  • current_campaign_id: null               ← cleared (removed from campaign) ║
║  • past_campaign_ids: append [campaign_id] ← add to history                  ║
║  • last_reply_date: [message_timestamp]    ← updated                         ║
║  • last_touch_date: [message_timestamp]    ← updated                         ║
║  • reply_count: reply_count + 1            ← incremented                     ║
║  • follow_up_date: null                    ← clear any scheduled follow-up   ║
║  • follow_up_reason: null                  ← clear                           ║
║  • updated_at: NOW()                                                         ║
║                                                                              ║
║  WHERE lead_id = [lead_id]                                                   ║
║                                                                              ║
║  ─────────────────────────────────────────────────────────────────────────   ║
║                                                                              ║
║  ACTIONS:                                                                    ║
║  ════════                                                                    ║
║  1. Remove lead from campaign sequence (API call to Instantly/Smartlead)     ║
║  2. AI generates reply                                                       ║
║  3. Send to Slack for human review                                           ║
║  4. On approval → Send reply → Update chat_history                           ║
║                                                                              ║
╚══════════════════════════════════════════════════════════════════════════════╝
```

---

### EVENT A.2: Lead Replies - NOT INTERESTED

```
╔══════════════════════════════════════════════════════════════════════════════╗
║ TRIGGER: Instantly/Smartlead webhook (reply received)                        ║
║ CONDITION: AI analysis detects NOT INTERESTED intent                         ║
╠══════════════════════════════════════════════════════════════════════════════╣
║                                                                              ║
║  INSERT INTO chat_history:                                                   ║
║  ═════════════════════════                                                   ║
║  • chat_id: gen_random_uuid()                                                ║
║  • session_id: [lead's email]                                                ║
║  • lead_id: [lead_id]                                                        ║
║  • client_id: null                                                           ║
║  • campaign_id: [campaign_id]                                                ║
║  • platform: 'smartlead' OR 'instantly'                                      ║
║  • contact_identifier: [lead's email]                                        ║
║  • contact_name: [lead's full name]                                          ║
║  • account_identifier: [our email]                                           ║
║  • subject: [reply subject]                                                  ║
║  • message_content: [reply body]                                             ║
║  • message_type: 'reply'                                                     ║
║  • message_status: 'received'                                                ║
║  • sequence_step: null                                                       ║
║  • external_message_id: [from webhook]                                       ║
║  • thread_id: [from webhook]                                                 ║
║  • ai_sentiment: 'negative' OR 'not_interested'                              ║
║  • ai_intent: 'not_interested' OR 'not_interested_timing'                    ║
║  • is_auto_reply: false                                                      ║
║  • requires_action: true                                                     ║
║  • action_taken: null                                                        ║
║  • is_vectorized: false                                                      ║
║  • vector_id: null                                                           ║
║  • message_timestamp: [from webhook]                                         ║
║  • created_at: NOW()                                                         ║
║                                                                              ║
║  ─────────────────────────────────────────────────────────────────────────   ║
║                                                                              ║
║  UPDATE leads SET:                                                           ║
║  ═════════════════                                                           ║
║  • lead_status: 'not_interested'           ← changed from 'active'           ║
║  • pipeline_status: 'closed'               ← changed from 'open'             ║
║  • campaign_status: 'completed'            ← changed from 'active'           ║
║  • lead_email_status: 'replied'            ← changed from null               ║
║  • current_campaign_id: null               ← cleared                         ║
║  • past_campaign_ids: append [campaign_id] ← add to history                  ║
║  • last_reply_date: [message_timestamp]    ← updated                         ║
║  • last_touch_date: [message_timestamp]    ← updated                         ║
║  • reply_count: reply_count + 1            ← incremented                     ║
║  • follow_up_date: NOW() + 6 months        ← schedule re-engagement          ║
║  • follow_up_reason: 'not_interested_re_engagement'                          ║
║  • updated_at: NOW()                                                         ║
║                                                                              ║
║  WHERE lead_id = [lead_id]                                                   ║
║                                                                              ║
║  ─────────────────────────────────────────────────────────────────────────   ║
║                                                                              ║
║  ACTIONS:                                                                    ║
║  ════════                                                                    ║
║  1. Remove lead from campaign sequence                                       ║
║  2. Send polite close reply (optional)                                       ║
║  3. Add to re-engagement list for 6 months later                             ║
║                                                                              ║
╚══════════════════════════════════════════════════════════════════════════════╝
```

---

### EVENT A.3: Lead Replies - OUT OF OFFICE

```
╔══════════════════════════════════════════════════════════════════════════════╗
║ TRIGGER: Instantly/Smartlead webhook (reply received)                        ║
║ CONDITION: AI detects OUT OF OFFICE auto-reply                               ║
╠══════════════════════════════════════════════════════════════════════════════╣
║                                                                              ║
║  INSERT INTO chat_history:                                                   ║
║  ═════════════════════════                                                   ║
║  • chat_id: gen_random_uuid()                                                ║
║  • session_id: [lead's email]                                                ║
║  • lead_id: [lead_id]                                                        ║
║  • client_id: null                                                           ║
║  • campaign_id: [campaign_id]                                                ║
║  • platform: 'smartlead' OR 'instantly'                                      ║
║  • contact_identifier: [lead's email]                                        ║
║  • contact_name: [lead's full name]                                          ║
║  • account_identifier: [our email]                                           ║
║  • subject: [reply subject]                                                  ║
║  • message_content: [OOO message]                                            ║
║  • message_type: 'reply'                                                     ║
║  • message_status: 'received'                                                ║
║  • sequence_step: null                                                       ║
║  • external_message_id: [from webhook]                                       ║
║  • thread_id: [from webhook]                                                 ║
║  • ai_sentiment: 'neutral'                                                   ║
║  • ai_intent: 'out_of_office'                                                ║
║  • is_auto_reply: true                     ← THIS IS AN AUTO-REPLY           ║
║  • requires_action: false                  ← no human action needed          ║
║  • action_taken: null                                                        ║
║  • is_vectorized: false                                                      ║
║  • vector_id: null                                                           ║
║  • message_timestamp: [from webhook]                                         ║
║  • created_at: NOW()                                                         ║
║                                                                              ║
║  ─────────────────────────────────────────────────────────────────────────   ║
║                                                                              ║
║  UPDATE leads SET:                                                           ║
║  ═════════════════                                                           ║
║  • lead_status: 'out_of_office'            ← changed from 'active'           ║
║  • pipeline_status: 'follow_up'            ← changed from 'open'             ║
║  • campaign_status: 'paused'               ← changed from 'active'           ║
║  • lead_email_status: 'replied'            ← changed from null               ║
║  • current_campaign_id: [keep same]        ← keep (will resume)              ║
║  • last_reply_date: [message_timestamp]    ← updated                         ║
║  • last_touch_date: [message_timestamp]    ← updated                         ║
║  • reply_count: reply_count + 1            ← incremented                     ║
║  • follow_up_date: [parsed return date]    ← parse from OOO message          ║
║  • follow_up_reason: 'out_of_office_return'                                  ║
║  • updated_at: NOW()                                                         ║
║                                                                              ║
║  WHERE lead_id = [lead_id]                                                   ║
║                                                                              ║
║  ─────────────────────────────────────────────────────────────────────────   ║
║                                                                              ║
║  ACTIONS:                                                                    ║
║  ════════                                                                    ║
║  1. Pause in campaign sequence (or remove temporarily)                       ║
║  2. Parse return date from OOO message                                       ║
║  3. Schedule follow-up for return date                                       ║
║  4. Daily cron will resume when follow_up_date arrives                       ║
║                                                                              ║
╚══════════════════════════════════════════════════════════════════════════════╝
```

---

### EVENT A.4: Lead Replies - UNSUBSCRIBE

```
╔══════════════════════════════════════════════════════════════════════════════╗
║ TRIGGER: Instantly/Smartlead webhook (reply received)                        ║
║ CONDITION: AI detects UNSUBSCRIBE request                                    ║
╠══════════════════════════════════════════════════════════════════════════════╣
║                                                                              ║
║  INSERT INTO chat_history:                                                   ║
║  ═════════════════════════                                                   ║
║  • chat_id: gen_random_uuid()                                                ║
║  • session_id: [lead's email]                                                ║
║  • lead_id: [lead_id]                                                        ║
║  • client_id: null                                                           ║
║  • campaign_id: [campaign_id]                                                ║
║  • platform: 'smartlead' OR 'instantly'                                      ║
║  • contact_identifier: [lead's email]                                        ║
║  • contact_name: [lead's full name]                                          ║
║  • account_identifier: [our email]                                           ║
║  • subject: [reply subject]                                                  ║
║  • message_content: [unsubscribe message]                                    ║
║  • message_type: 'reply'                                                     ║
║  • message_status: 'received'                                                ║
║  • sequence_step: null                                                       ║
║  • external_message_id: [from webhook]                                       ║
║  • thread_id: [from webhook]                                                 ║
║  • ai_sentiment: 'negative'                                                  ║
║  • ai_intent: 'unsubscribe'                                                  ║
║  • is_auto_reply: false                                                      ║
║  • requires_action: false                  ← handled automatically           ║
║  • action_taken: 'unsubscribed'                                              ║
║  • is_vectorized: false                                                      ║
║  • vector_id: null                                                           ║
║  • message_timestamp: [from webhook]                                         ║
║  • created_at: NOW()                                                         ║
║                                                                              ║
║  ─────────────────────────────────────────────────────────────────────────   ║
║                                                                              ║
║  UPDATE leads SET:                                                           ║
║  ═════════════════                                                           ║
║  • lead_status: 'do_not_contact'           ← changed from 'active'           ║
║  • pipeline_status: 'closed'               ← changed from 'open'             ║
║  • campaign_status: 'completed'            ← changed from 'active'           ║
║  • lead_email_status: 'unsubscribed'       ← changed from null               ║
║  • current_campaign_id: null               ← cleared                         ║
║  • past_campaign_ids: append [campaign_id] ← add to history                  ║
║  • last_reply_date: [message_timestamp]    ← updated                         ║
║  • last_touch_date: [message_timestamp]    ← updated                         ║
║  • reply_count: reply_count + 1            ← incremented                     ║
║  • follow_up_date: null                    ← clear                           ║
║  • follow_up_reason: null                  ← clear                           ║
║  • updated_at: NOW()                                                         ║
║                                                                              ║
║  WHERE lead_id = [lead_id]                                                   ║
║                                                                              ║
║  ─────────────────────────────────────────────────────────────────────────   ║
║                                                                              ║
║  ACTIONS:                                                                    ║
║  ════════                                                                    ║
║  1. Remove from campaign sequence immediately                                ║
║  2. Block from all future campaigns (legal compliance)                       ║
║  3. DO NOT send any reply                                                    ║
║                                                                              ║
╚══════════════════════════════════════════════════════════════════════════════╝
```

---

### EVENT B: Sequence Completes (No Reply)

```
╔══════════════════════════════════════════════════════════════════════════════╗
║ TRIGGER: Smartlead "Email Sent" webhook when last email in sequence is sent   ║
║ CONDITION: campaign_status === 'completed' OR is_last_email === true         ║
╠══════════════════════════════════════════════════════════════════════════════╣
║                                                                              ║
║  INSERT INTO chat_history:                                                   ║
║  ═════════════════════════                                                   ║
║  • Log the last email sent (same as any email sent)                         ║
║                                                                              ║
║  ─────────────────────────────────────────────────────────────────────────   ║
║                                                                              ║
║  UPDATE leads SET:                                                           ║
║  ═════════════════                                                           ║
║  • lead_status: 'active'                   ← stays same                      ║
║  • pipeline_status: 'open'                 ← stays same                      ║
║  • campaign_status: 'completed'            ← changed from 'active'           ║
║  • lead_email_status: 'no_reply'           ← changed from null               ║
║  • current_campaign_id: null               ← cleared                         ║
║  • past_campaign_ids: append [campaign_id] ← add to history                  ║
║  • follow_up_date: NOW() + 3 months        ← schedule re-engagement          ║
║  • follow_up_reason: 'not_interested_re_engagement'                          ║
║  • updated_at: NOW()                                                         ║
║                                                                              ║
║  WHERE email = [lead_email] AND client_id = [client_id]                     ║
║                                                                              ║
║  ─────────────────────────────────────────────────────────────────────────   ║
║                                                                              ║
║  ACTIONS:                                                                    ║
║  ════════                                                                    ║
║  1. Add to re-engagement list (3 months)                                     ║
║  2. Consider trying different channel (LinkedIn, phone)                      ║
║                                                                              ║
║  NOTE: This update happens in the same "Email Sent" webhook workflow,        ║
║        allowing real-time status updates without a separate cron job.        ║
║                                                                              ║
╚══════════════════════════════════════════════════════════════════════════════╝
```

---

### EVENT C: Email Bounced

```
╔══════════════════════════════════════════════════════════════════════════════╗
║ TRIGGER: Instantly/Smartlead webhook (bounce notification)                   ║
╠══════════════════════════════════════════════════════════════════════════════╣
║                                                                              ║
║  INSERT INTO chat_history:                                                   ║
║  ═════════════════════════                                                   ║
║  • chat_id: gen_random_uuid()                                                ║
║  • session_id: [lead's email]                                                ║
║  • lead_id: [lead_id]                                                        ║
║  • client_id: null                                                           ║
║  • campaign_id: [campaign_id]                                                ║
║  • platform: 'smartlead' OR 'instantly'                                      ║
║  • contact_identifier: [lead's email]                                        ║
║  • contact_name: [lead's full name]                                          ║
║  • account_identifier: [our email]                                           ║
║  • subject: [original subject]                                               ║
║  • message_content: [bounce message or original email]                       ║
║  • message_type: 'campaign'                                                  ║
║  • message_status: 'bounced'               ← BOUNCED STATUS                  ║
║  • sequence_step: [step that bounced]                                        ║
║  • external_message_id: [from webhook]                                       ║
║  • thread_id: null                                                           ║
║  • ai_sentiment: null                                                        ║
║  • ai_intent: null                                                           ║
║  • is_auto_reply: false                                                      ║
║  • requires_action: true                   ← needs re-enrichment             ║
║  • action_taken: null                                                        ║
║  • is_vectorized: false                                                      ║
║  • vector_id: null                                                           ║
║  • message_timestamp: [from webhook]                                         ║
║  • created_at: NOW()                                                         ║
║                                                                              ║
║  ─────────────────────────────────────────────────────────────────────────   ║
║                                                                              ║
║  UPDATE leads SET:                                                           ║
║  ═════════════════                                                           ║
║  • lead_status: 'active'                   ← stays same                      ║
║  • pipeline_status: 'open'                 ← stays same                      ║
║  • campaign_status: 'completed'            ← changed from 'active'           ║
║  • lead_email_status: 'bounced'            ← changed from null               ║
║  • current_campaign_id: null               ← cleared                         ║
║  • past_campaign_ids: append [campaign_id] ← add to history                  ║
║  • updated_at: NOW()                                                         ║
║                                                                              ║
║  WHERE lead_id = [lead_id]                                                   ║
║                                                                              ║
║  ─────────────────────────────────────────────────────────────────────────   ║
║                                                                              ║
║  ACTIONS:                                                                    ║
║  ════════                                                                    ║
║  1. Remove from campaign sequence                                            ║
║  2. Flag for re-enrichment in Clay                                           ║
║  3. Find new email address                                                   ║
║  4. Once found, add to new campaign                                          ║
║                                                                              ║
╚══════════════════════════════════════════════════════════════════════════════╝
```

---

### EVENT D: We Reply to Their Reply

```
╔══════════════════════════════════════════════════════════════════════════════╗
║ TRIGGER: Human approves AI reply in Slack → Send via API                     ║
╠══════════════════════════════════════════════════════════════════════════════╣
║                                                                              ║
║  INSERT INTO chat_history:                                                   ║
║  ═════════════════════════                                                   ║
║  • chat_id: gen_random_uuid()                                                ║
║  • session_id: [lead's email]                                                ║
║  • lead_id: [lead_id]                                                        ║
║  • client_id: [client_id if converted]                                       ║
║  • campaign_id: [original campaign_id]                                       ║
║  • platform: 'gmail' OR same as their reply                                  ║
║  • contact_identifier: [lead's email]                                        ║
║  • contact_name: [lead's full name]                                          ║
║  • account_identifier: [our email]                                           ║
║  • subject: 'Re: ' + [original subject]                                      ║
║  • message_content: [our reply]                                              ║
║  • message_type: 'reply'                                                     ║
║  • message_status: 'sent'                  ← WE SENT IT                      ║
║  • sequence_step: null                                                       ║
║  • external_message_id: [from email API]                                     ║
║  • thread_id: [same thread_id]             ← reply in same thread            ║
║  • ai_sentiment: null                                                        ║
║  • ai_intent: null                                                           ║
║  • is_auto_reply: false                                                      ║
║  • requires_action: false                                                    ║
║  • action_taken: null                                                        ║
║  • is_vectorized: false                                                      ║
║  • vector_id: null                                                           ║
║  • message_timestamp: NOW()                                                  ║
║  • created_at: NOW()                                                         ║
║                                                                              ║
║  ─────────────────────────────────────────────────────────────────────────   ║
║                                                                              ║
║  UPDATE leads SET:                                                           ║
║  ═════════════════                                                           ║
║  • lead_status: [stays same]               ← already 'interested' etc.       ║
║  • pipeline_status: [stays same]           ← already 'follow_up' etc.        ║
║  • campaign_status: [stays same]           ← already 'completed'             ║
║  • lead_email_status: [stays same]         ← already 'replied'               ║
║  • last_contact_date: NOW()                ← updated (WE contacted them)     ║
║  • last_touch_date: NOW()                  ← updated                         ║
║  • updated_at: NOW()                                                         ║
║                                                                              ║
║  WHERE lead_id = [lead_id]                                                   ║
║                                                                              ║
╚══════════════════════════════════════════════════════════════════════════════╝
```

---

### EVENT E: Meeting Booked

```
╔══════════════════════════════════════════════════════════════════════════════╗
║ TRIGGER: Cal.com/Calendly webhook (meeting booked)                           ║
╠══════════════════════════════════════════════════════════════════════════════╣
║                                                                              ║
║  INSERT INTO chat_history (optional):                                        ║
║  ════════════════════════════════════                                        ║
║  • chat_id: gen_random_uuid()                                                ║
║  • session_id: [lead's email]                                                ║
║  • lead_id: [lead_id]                                                        ║
║  • platform: 'cal_com' OR 'calendly'                                         ║
║  • contact_identifier: [lead's email]                                        ║
║  • contact_name: [lead's full name]                                          ║
║  • account_identifier: [calendar account]                                    ║
║  • message_content: 'Meeting booked for [date] at [time]'                    ║
║  • message_type: 'meeting'                                                   ║
║  • message_status: 'received'              ← they took action                ║
║  • message_timestamp: NOW()                                                  ║
║  • created_at: NOW()                                                         ║
║                                                                              ║
║  ─────────────────────────────────────────────────────────────────────────   ║
║                                                                              ║
║  UPDATE leads SET:                                                           ║
║  ═════════════════                                                           ║
║  • lead_status: 'interested'               ← stays same                      ║
║  • pipeline_status: 'meeting_booked'       ← changed from 'follow_up'        ║
║  • campaign_status: 'completed'            ← stays same                      ║
║  • lead_email_status: 'replied'            ← stays same                      ║
║  • last_touch_date: NOW()                  ← updated                         ║
║  • follow_up_date: [meeting date]          ← set to meeting date             ║
║  • follow_up_reason: null                  ← clear                           ║
║  • updated_at: NOW()                                                         ║
║                                                                              ║
║  WHERE lead_id = [lead_id]                                                   ║
║                                                                              ║
╚══════════════════════════════════════════════════════════════════════════════╝
```

---

### EVENT F: Meeting No-Show

```
╔══════════════════════════════════════════════════════════════════════════════╗
║ TRIGGER: Manual update OR automated check after meeting time                 ║
╠══════════════════════════════════════════════════════════════════════════════╣
║                                                                              ║
║  UPDATE leads SET:                                                           ║
║  ═════════════════                                                           ║
║  • lead_status: 'no_show'                  ← changed from 'interested'       ║
║  • pipeline_status: 'follow_up'            ← changed from 'meeting_booked'   ║
║  • campaign_status: 'completed'            ← stays same                      ║
║  • lead_email_status: 'replied'            ← stays same                      ║
║  • follow_up_date: NOW() + 1 day           ← schedule follow-up              ║
║  • follow_up_reason: 'meeting_no_show'                                       ║
║  • updated_at: NOW()                                                         ║
║                                                                              ║
║  WHERE lead_id = [lead_id]                                                   ║
║                                                                              ║
║  ─────────────────────────────────────────────────────────────────────────   ║
║                                                                              ║
║  ACTIONS:                                                                    ║
║  ════════                                                                    ║
║  1. Send no-show follow-up email                                             ║
║  2. Try to reschedule                                                        ║
║                                                                              ║
╚══════════════════════════════════════════════════════════════════════════════╝
```

---

### EVENT G: Deal Won

```
╔══════════════════════════════════════════════════════════════════════════════╗
║ TRIGGER: Manual update in Airtable CRM                                       ║
╠══════════════════════════════════════════════════════════════════════════════╣
║                                                                              ║
║  UPDATE leads SET:                                                           ║
║  ═════════════════                                                           ║
║  • lead_status: 'won'                      ← changed from 'interested'       ║
║  • pipeline_status: 'closed'               ← changed from 'meeting_booked'   ║
║  • campaign_status: 'completed'            ← stays same                      ║
║  • lead_email_status: 'replied'            ← stays same                      ║
║  • converted_to_client: true               ← set to true                     ║
║  • converted_client_id: [new client_id]    ← link to clients table           ║
║  • follow_up_date: null                    ← clear                           ║
║  • follow_up_reason: null                  ← clear                           ║
║  • updated_at: NOW()                                                         ║
║                                                                              ║
║  WHERE lead_id = [lead_id]                                                   ║
║                                                                              ║
║  ─────────────────────────────────────────────────────────────────────────   ║
║                                                                              ║
║  CREATE client record in Clients table                                       ║
║                                                                              ║
║  ─────────────────────────────────────────────────────────────────────────   ║
║                                                                              ║
║  ACTIONS:                                                                    ║
║  ════════                                                                    ║
║  1. Create client record                                                     ║
║  2. Start onboarding workflow                                                ║
║  3. Update future chat_history with client_id                                ║
║                                                                              ║
╚══════════════════════════════════════════════════════════════════════════════╝
```

---

### EVENT H: Ghosted (Replied Then Silent)

```
╔══════════════════════════════════════════════════════════════════════════════╗
║ TRIGGER: Daily cron job                                                      ║
║ CONDITION: lead_status = 'interested'                                        ║
║            AND lead_email_status = 'replied'                                 ║
║            AND last_contact_date > last_reply_date                           ║
║            AND last_contact_date < NOW() - INTERVAL '5 days'                 ║
╠══════════════════════════════════════════════════════════════════════════════╣
║                                                                              ║
║  UPDATE leads SET:                                                           ║
║  ═════════════════                                                           ║
║  • lead_status: 'ghosted'                  ← changed from 'interested'       ║
║  • pipeline_status: 'follow_up'            ← stays same                      ║
║  • campaign_status: 'completed'            ← stays same                      ║
║  • lead_email_status: 'replied'            ← stays same (they DID reply)     ║
║  • follow_up_date: NOW()                   ← needs follow-up now             ║
║  • follow_up_reason: 'ghosted_follow_up'                                     ║
║  • updated_at: NOW()                                                         ║
║                                                                              ║
║  WHERE [conditions above]                                                    ║
║                                                                              ║
║  ─────────────────────────────────────────────────────────────────────────   ║
║                                                                              ║
║  ACTIONS:                                                                    ║
║  ════════                                                                    ║
║  1. Add to ghosted follow-up sequence                                        ║
║  2. OR send manual follow-up                                                 ║
║                                                                              ║
╚══════════════════════════════════════════════════════════════════════════════╝
```

---

## Part 5: Quick Reference Cards

### Status Values Quick Reference

**lead_status:**
| Value | Meaning |
|-------|---------|
| `triage` | New, never contacted |
| `active` | In outreach, no response yet |
| `interested` | Showed positive interest |
| `not_interested` | Said no |
| `out_of_office` | OOO auto-reply |
| `ghosted` | Replied then went silent |
| `no_show` | Booked but didn't show |
| `do_not_contact` | Requested removal |
| `won` | Converted to client |
| `declined` | Final rejection |

**pipeline_status:**
| Value | Meaning |
|-------|---------|
| `open` | In pipeline, no specific action |
| `follow_up` | Needs follow-up action |
| `meeting_booked` | Meeting scheduled |
| `closed` | Deal done (won or lost) |

**campaign_status:**
| Value | Meaning |
|-------|---------|
| `null` | Not in any campaign |
| `active` | Sequence currently running |
| `paused` | Sequence temporarily paused |
| `completed` | Sequence ended |

**lead_email_status:**
| Value | Meaning |
|-------|---------|
| `null` | No outcome yet |
| `replied` | Lead replied |
| `no_reply` | All emails sent, no response |
| `bounced` | Email bounced |
| `unsubscribed` | Requested removal |

**message_type:**
| Value | Meaning |
|-------|---------|
| `campaign` | Automated campaign email |
| `reply` | Reply in conversation |
| `manual` | Manual one-off message |
| `meeting` | Meeting notes/booking |
| `follow_up` | Manual follow-up |

**message_status:**
| Value | Direction | Meaning |
|-------|-----------|---------|
| `sent` | Outgoing | We sent it |
| `received` | Incoming | We received it |
| `bounced` | Outgoing | Email bounced |
| `failed` | Outgoing | Failed to send |

---

## Related Documentation

- [Database Schema](../database-schema.md) - Original full database design
- [Chat History Schema](./chat-history-schema.md) - Detailed chat_history documentation
- [Lead Journey Map](./lead-journey-map.md) - Complete lead lifecycle flows
- [n8n Architecture](../n8n-architecture.md) - Workflow implementation details

---

**Last Updated:** January 4, 2026  
**Version:** 1.0 - Final

