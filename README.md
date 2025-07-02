# 🛠️ Email Leadgen Automation – n8n Workflow
### By Wissam Moussa, for WzTechno and GenAI Strategy


This workflow fetches forwarded emails on a trigger basis, processes them through simple string manipulation code, LLMs and LLM-power scraper APIs for enrichment, and populates database tables of companies and their respective leads. 
Enriched data is stored into Supabase for persistent access.

---

## 📌 Features

- **Trigger-based execution** whenever the linked Gmail address receives a new email
- **Email body parsing**
- **Sender (lead) identification extraction**
- **Company identification extraction**
- **Storage** of processed lead and company data in Supabase

---

## 🧩 Workflow Breakdown

### 1. Trigger

- The workflow is triggered whenever a new email is received using the `IMAP Trigger` node.

---

### 2. Email Processing

- The sender's `first name`, `last name`, and `email` address are parsed via a simple string manipulation function: Javascript `.split()` based on linebreaks - This function is provider-agnostic and works for both Gmail and Outlook forwards
- The email `body` of main text is parsed in a similar manner
- In case the email is not available, the flow stops
- In case either or both of the `first name` and `last name` are not available, they are still processed and stored as 'n/a'
- The company's domain name is pre-parsed using a simple string manipulation function, by taking the part of the email address that comes after the `@` symbol
- The company's final `domain name` and `company name` are parsed using an LLM chain which has access to the pre-parsed values and the email `body`. This tackles the case of leads using their private email accounts to send messages

---

### 3. Flow Chronology

1) Email is received, workflow is triggered

2) Javascript code is used to parse `first name`, `last name`, `email` and `body`, and pre-parse `domain name`
   
3-A) If email is parsed successfully, a new record is created in the `leads` Supabase table

3-B) If email is not found/parsed, the workflow stops

4) Data undergoes an LLM chain run to extract `domain name` and `company name` - this chain is equipped with a `structured output parser` to enforce data typing and ensure that the required data flows through the node stream

5) Existing data in our Supabase `companies` table is matched with the extracted `domain name`

6-A) If a record is found for the company, skip to step 9

6-B) If a record is not found, we need to extract extra information about the company to create a record

7) Using an `HTTP request` node, we extract the company's `logo url` and a short `description` by feeding the previously parsed `domain name` to the Parsera AI-powered API

8) A new record is created in the `companies` Supabase table with the extracted information

9) Using information returned from step 6-A or step 8 programmatically, with no extra calls to Supabase, the company's row `id` is parsed

10) This `id` is then used to update the `company_id` value of the lead record created in step 3

#### Steps 3-A/B and 6-A/B are routed using `Switch` nodes
#### AI logic is done using GPT-3.5-Turbo for personal cost-efficiency, feel free to switch up to a bigger model for better output quality.
---

### 4. AI Enrichment

- `Summarization Chain`: Produces a one-sentence summary using n8n's built-in OpenAI Summarization Chain (recursive summarization is used for text longer than 1000 characters).
- `Sentiment Analysis`: Determines sentiment (`positive`, `negative`, `neutral`)
- `Generate Dynamic Tags`: Adds contextual tags based on content (these are different from the actual 'labels' set by the creator of the Github issue)
- `Extract Category and Assigned Role`: Categorizes the issue (possible values: `bug`, `feature request`, `question`) and assigns a team to handle it (possible values: `engineering`, `sales`, `support`)

All AI logic is done using GPT-3.5-Turbo for personal cost-efficiency, feel free to switch up to a bigger model for better output quality.
All custom AI chains are equipped with an 'Output Parser' to enforce output data typing for robustness.

---

### 5. Slack Routing (via `Switch`)

- A `Switch` node branches the flow into `sales`, `support`, or `engineering` based on the assigned role
- Each branch sets a Slack webhook URL using a `Set` node
- All branches are merged back using `Merge` node
- A single `Send Slack message` node uses expressions and `[$itemIndex]` to ensure Slack posts match the original issue data

---

### 6. Storage & Metadata Update

- `Parse final data for Supabase`: Aggregates all enriched info and prepares data to be posted into Supabase
- `Add Issues to Supabase`: Inserts records into the `enriched_issues` Supabase table
- `Update last_run_time in Supabase`: Updates workflow metadata with the new `last_run_time` to be used for the next workflow run

---

## 🔐 Secrets / Credentials

This workflow assumes the following credentials are set up in the n8n UI:

- IMAP/Gmail API
- Supabase API
- OpenAI API

The API key for Parsera is stored within the workflow node, since the built-in `Parsera` node in `n8n` is down/buggy.
This is a security concern and not a good practice.
In the future, when this node is fixed, or if another extraction service is used, secret credentials should be set via the `n8n` UI.

---

## ⚙️ Setup

This workflow assumes the following tables and schemas exist in your Supabase project:

```sql
-- A table for company information
CREATE TABLE companies (
    id BIGINT PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
    name TEXT,
    domain TEXT UNIQUE,
    description TEXT,
    logo_url TEXT
);

-- A table to store the captured sales leads
CREATE TABLE leads (
    id BIGINT PRIMARY KEY GENERATED ALWAYS AS IDENTITY,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    first_name TEXT,
    last_name TEXT,
    email TEXT UNIQUE NOT NULL,
    -- This will link a lead to a company
    company_id BIGINT REFERENCES companies(id)
);
```

---

## 🚧 Limitations / TODOs

- No catch-all error handler is currently set up for generic errors. N8n requires `error trigger` nodes to exist in a separate workflow
- No `post-run actions` is currently setup for inbox cleanup and sorting of emails into `Processed` vs `Failed` folders. Even though this is easily achievable with the built-in `Gmail` node, I was not able to find an equivalent for it in the `IMAP trigger` node, which is explicitly requested in this challenge
  
---

## 🎥 Demo

For a Loom video demo, [click here](https://www.loom.com/share/9e55091add6d4233b0e2241631c8f132)
  
---

## 📂 File Structure

This `README.md` assumes the accompanying `workflow.json` is imported directly into an n8n instance via the UI.

