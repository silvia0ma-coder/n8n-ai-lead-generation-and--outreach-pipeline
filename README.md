# n8n AI Lead Generation and Outreach Pipeline

An n8n workflow that runs every morning, pulls fresh B2B leads, filters and scores them with AI, saves the good ones to Google Sheets, and contacts each one with a personalised **email** and **WhatsApp** message. Every outcome is logged.

<img width="2160" height="2700" alt="lead-gen-pipeline-diagram" src="https://github.com/user-attachments/assets/272740c5-bdb1-4521-9569-f775436c3c33" />

## What it does

1. **Source.** A schedule trigger fires at 8AM and calls an Apify leads-finder actor for decision-makers (CEO, COO, CFO, CTO, founders, owners, directors) in Lebanon, UAE, Saudi Arabia, Qatar and Jordan, with verified emails and phone numbers.
2. **Quality guard.** A code node rejects bad records (disposable domains, `noreply`-style addresses, malformed emails, leads outside the target countries) and writes a run summary to a `Run Log` tab. If Apify returns an error or nothing usable, the run is logged instead of failing silently.
3. **Qualify.** An email check marks each address deliverable or not. An AI agent then scores each lead from 1 to 10 against your ideal customer profile (seniority is the main factor). Only leads scoring **7 or higher** with a deliverable email continue.
4. **Dedupe and save.** Leads are normalised (name, domain, phone), checked against your Google Sheet by email, and skipped if already there. New leads are appended to the sheet.
5. **Outreach, one lead at a time.**
   - **Email:** validate the address, have an AI agent write a short email, check the copy is not empty, send through Gmail.
   - **WhatsApp:** clean the phone number to international format, check the number is on WhatsApp through Green-API, have an AI agent write a short message, send it.
6. **Track and repeat.** Each lead's email and WhatsApp results are consolidated into one row in an `Outreach Log` tab. The workflow waits 2 minutes, then takes the next lead, which keeps sending slow enough to protect your sender reputation.

## Stack

| Purpose | Tool |
|---|---|
| Orchestration | [n8n](https://n8n.io) |
| Lead sourcing | [Apify](https://apify.com) actor `pipelinelabs/leads-finder-with-emails-apollo-lusha-zoominfo` |
| AI scoring and copywriting | Google Gemini and OpenAI chat models (swap for any model n8n supports) |
| CRM and logs | Google Sheets |
| Email | Gmail (OAuth) |
| WhatsApp | [Green-API](https://green-api.com) community node `@green-api/n8n-nodes-whatsapp-greenapi` |

## Setup

### 1. Import the workflow
In n8n choose **Workflows → Import from file** and select `lead-gen-pipeline.json`. Install the Green-API community node first (**Settings → Community nodes**).

### 2. Create credentials
Open each node that shows a credential warning and connect your own:

- Apify (OAuth2)
- Google Sheets (OAuth2)
- Gmail (OAuth2)
- Green-API (instance ID and token)
- Google Gemini and OpenAI API keys, or n8n's managed AI credits

### 3. Create the Google Sheet
Create one spreadsheet with three tabs, then re-select the spreadsheet and tab in every Google Sheets node (the IDs in the file are placeholders).

**`Linkdin`** (lead list, also used for duplicate checks). Header row:

```
Full Name | Job Title | Company | Industry | Company Domain | Company Size | Revenue Range | Email | Phone | City | Country | LinkedIn | Intent Signal Score
```

**`Outreach Log`** (one row per contacted lead). Header row:

```
Lead ID | Email | Full Name | Company | Phone | Email Quality | Email Quality Reason | Email Status | Email Delivery Class | Email SMTP Code | Email Sent At | Email Message ID | Email Error | Email Delivery Note | WhatsApp Status | WhatsApp Sent At | WhatsApp Message ID | WhatsApp Error | Outreach Status | Last Attempt At
```

**`Run Log`** (daily pull summary). Header row:

```
Run Time | Pulled | Passed Quality Guard | Rejected | Rejection Reasons | Status
```

### 4. Adjust to your market
| What | Where |
|---|---|
| Target countries, titles, seniority, leads per run | **Apify - Generate Leads** node, JSON body (`totalResults` is 10 by default) |
| Run time | **Daily 8AM Trigger** (cron `0 8 * * *`) and the workflow timezone in Settings |
| Minimum score | **IF Qualified** node (default 7) |
| Ideal customer profile and scoring rubric | System message in the **AI Agent** scoring node |
| Email and WhatsApp tone, length, sign-off | Prompts in the two copywriting agent nodes |
| Delay between leads | **Wait Between Batches** node (default 2 minutes) |

### 5. Test, then publish
Run the Apify node alone first and confirm it returns lead objects. Then run the whole workflow manually with a test email you own. Publish the workflow only when the sheet and the sent messages look right.

## Troubleshooting

- **No leads come in.** Check the Apify run log. Filters that are too narrow (a small country list, verified email plus phone required) can exhaust the pool, because the actor remembers progress and does not repeat leads it already gave you. Widen the countries or titles, or set `resetProgress` to `true` for one run.
- **Everything is rejected at the email step.** Open the Quality Guard and email check outputs and read the rejection reasons in the `Run Log` tab.
- **Sheet nodes error.** The IDs in this file are placeholders. Re-select your spreadsheet and tab in every Google Sheets node.
- **WhatsApp is skipped.** The number is missing, could not be normalised to international format, or is not registered on WhatsApp.

## Responsible use

This workflow sends unsolicited commercial messages. Make sure your use complies with the laws in your market and your recipients' (for example GDPR, CAN-SPAM, and local telecom and data-protection rules), respect WhatsApp's terms and Gmail's sending limits, include a way to opt out in your copy, and never contact people who have asked you to stop. You are responsible for how you use it.

## License

MIT. See `LICENSE`.
