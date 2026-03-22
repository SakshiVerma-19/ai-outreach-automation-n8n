# PitcPerfect — Personalized Outreach with RAG

An n8n workflow that takes a prospect's URL, scrapes their website, detects their tech stack, finds the most relevant case study using vector search, and generates a fully personalized cold email. Every prospect is automatically logged in Airtable.

**Input:** A prospect URL (e.g. `https://stripe.com`)
**Output:** A personalized cold email + Airtable record — in under 20 seconds.

---

## How it works

```
Webhook → Scrape website → Extract tech stack → Generate embeddings
→ Vector search (Astra DB) → Generate email (Gemini) → Log to Airtable → Return response
```

1. **Webhook** receives a POST request with the prospect's URL
2. **HTTP Request** scrapes the prospect's website HTML
3. **Code node** parses the HTML and detects the tech stack (React, Stripe, Shopify, etc.)
4. **Gemini Embeddings** converts the tech stack into a vector via `gemini-embedding-2-preview`
5. **Astra DB** runs a cosine similarity search to find the most relevant case study
6. **Gemini (Message a Model)** generates a personalized cold email referencing the tech stack and case study
7. **Code node** flattens all data into a clean output
8. **Airtable** logs the prospect, tech stack, matched case study, and email draft
9. **Respond to Webhook** returns the result to the caller

---

## Tech Stack

| Tool | Purpose |
|------|---------|
| [n8n](https://n8n.io) | Workflow automation |
| [Google Gemini](https://aistudio.google.com) | Embeddings + email generation |
| [Astra DB (DataStax)](https://astra.datastax.com) | Vector database for case study RAG |
| [Airtable](https://airtable.com) | CRM / prospect logging |

---

## Prerequisites

Before running this workflow you need:

- **n8n** instance (cloud or self-hosted)
- **Google Gemini API key** — get one at [aistudio.google.com](https://aistudio.google.com/app/apikey)
- **Astra DB account** — free tier works at [astra.datastax.com](https://astra.datastax.com)
- **Airtable account** + API token — generate at [airtable.com/create/tokens](https://airtable.com/create/tokens)

---

## Setup Instructions

### 1. Astra DB — Create the collection

1. Log into Astra DB and open your database
2. Go to **Data Explorer → Create Collection**
3. Configure:
   - **Name:** `case_study`
   - **Vector enabled:** ON
   - **Embedding generation:** Bring your own
   - **Dimensions:** `3072`
   - **Similarity metric:** Cosine
4. Save your **API Endpoint** and **Application Token**

### 2. Ingest your case studies

Before running the main workflow, run a one-time ingestion to populate Astra DB.

Use an HTTP Request node with:

```
POST https://YOUR-ASTRA-ID.apps.astra.datastax.com/api/json/v1/default_keyspace/case_study
```

Headers:
```
Token: AstraCS:your-token
Content-Type: application/json
```

Body:
```json
{
  "insertOne": {
    "document": {
      "title": "Your case study title",
      "summary": "Brief outcome summary — what problem was solved and the result.",
      "tags": ["React", "Stripe", "payments"],
      "$vector": [/* 3072-dimension vector from Gemini embeddings */]
    }
  }
}
```

### 3. Airtable — Create the table

Create a base with a table called `Prospects` with these columns:

| Column | Type |
|--------|------|
| Name | Single line text (primary) |
| Prospect URL | URL |
| Tech Stack | Long text |
| Case Study | Long text |
| Email Draft | Long text |
| Created At | Date |

### 4. Import the workflow into n8n

1. Download `workflow.json` from this repo
2. In n8n, go to **Workflows → Import**
3. Upload the JSON file
4. Add your credentials to each node:
   - Gemini API key → HTTP Request nodes
   - Astra DB token + endpoint → Astra DB HTTP Request node
   - Airtable token → Create a record node

---

##  Usage

Send a POST request to your webhook URL:

```bash
curl -X POST https://your-n8n-instance/webhook/YOUR-WEBHOOK-ID \
  -H "Content-Type: application/json" \
  -d '{"url": "https://stripe.com"}'
```

**Response:**
```json
{
  "url": "https://stripe.com",
  "techStack": "React, Next.js, Stripe",
  "caseStudy": "How a fintech startup reduced payment failures by 35%",
  "emailDraft": "Subject: Reducing payment failures with your Stripe stack\n\nHi [Name]..."
}
```

---

##  Tech Stack Detection

The Code node currently detects these technologies out of the box:

`React` · `Next.js` · `Vue` · `Angular` · `Shopify` · `WordPress` · `HubSpot` · `Salesforce` · `Segment` · `Intercom` · `Stripe` · `AWS` · `Vercel`

To add more, edit the `techSignals` object in the Code node:

```JavaScript
const techSignals = {
  "Your Tool": /your-regex-pattern/i,
  // add more here
};
```

---

## Workflow Nodes

| Node | Type | Purpose |
|------|------|---------|
| Webhook | Trigger | Receives prospect URL |
| HTTP Request | HTTP | Scrapes website HTML |
| Code in JavaScript | Code | Extracts tech stack from HTML |
| HTTP Request1 | HTTP | Calls Gemini embeddings API |
| HTTP Request2 | HTTP | Queries Astra DB vector search |
| Message a model | Gemini | Generates personalized email |
| Code in JavaScript1 | Code | Flattens data from all nodes |
| Create a record | Airtable | Logs prospect to CRM |
| Respond to Webhook | Response | Returns result to caller |

---

## Known Limitations

- Free tier Gemini API has a limit of **15 requests/minute**
- Astra DB free tier **hibernates after inactivity** — first request may take ~30s to wake up
- Some websites block scraping and will return a `403` — the workflow handles this gracefully but won't detect a tech stack
- Tech stack detection is regex-based — works best on sites that load scripts in HTML (not heavily server-side rendered)

---

## Roadmap / Possible Improvements

- [ ] Add support for scraping JavaScript-rendered pages (Puppeteer/Playwright)
- [ ] Ingest case studies automatically from a Google Doc or Notion page
- [ ] Add a Slack notification when a high-similarity match is found
- [ ] Build a simple frontend form instead of using Postman
- [ ] Add email sending via Gmail or SendGrid node

---

## 📄 License

MIT — free to use, modify and distribute.

---

Built with n8n, Google Gemini, Astra DB, and a lot of debugging.
