# Gmail Summarizer & Action-Item Extractor

## Overview
An automated agentic workflow built in n8n that reads unread emails from multiple Gmail accounts, filters out spam, newsletters, and promotional content, extracts action items and priorities, and sends a daily digest directly to WhatsApp.

---

## Problem Statement
As students, we get a lot of cluttered emails: some related to internships, some related to college assignments, and some unrelated ads that we signed up for once and now have to deal with everyday newsletters.

### Target User
Anyone that is fed up with reading hundreds of emails every few hours.

### Pain Point
Receiving a lot of useless emails and having no way to filter and clear them on a case-by-case basis.

### Solution
This workflow solves just that: it executes every 2 hours (can be changed to any amount of time), reads the emails received in this time, and gives the notification in a structured way on the user's WhatsApp.

### Expected Output
It gives a heading of the email, 2-3 line summary, priority, and action required for that email, and ignores useless emails.

---

## Workflow Architecture

### High-Level Flow
Below is the execution flow of the n8n workflow:

```text
┌─────────────────────────────┐
│ Schedule Trigger            │
│ Runs Every 2 Hours          │
└──────────────┬──────────────┘
               │
     ┌─────────┴─────────┐
     ▼                   ▼
┌───────────────┐ ┌───────────────┐
│ Scaler Gmail  │ │ Main Gmail    │
│ Fetch Unread  │ │ Fetch Unread  │
└───────┬───────┘ └───────┬───────┘
        │                 │
        ▼                 ▼
┌───────────────┐ ┌───────────────┐
│ Mark as Read  │ │ Mark as Read  │
└───────┬───────┘ └───────┬───────┘
        │                 │
        └────────┬────────┘
                 ▼
        ┌─────────────────┐
        │ Merge Node      │
        └────────┬────────┘
                 ▼
        ┌─────────────────┐
        │ JS Code Node    │
        │ Extract Fields  │
        └────────┬────────┘
                 ▼
        ┌─────────────────┐
        │ Claude Opus 4.6 │
        └────────┬────────┘
                 ▼
        ┌─────────────────┐
        │ JS Code Node    │
        │ Format Response │
        └────────┬────────┘
                 ▼
        ┌─────────────────┐
        │ Twilio          │
        │ WhatsApp Alert  │
        └─────────────────┘
```

---

## Agentic Design

The workflow uses an LLM-based agent pattern to perform semantic filtering and structuring:

### Agent 1: Email Action-Item Extractor (Anthropic Claude Opus 4.6)
**Purpose:** Acts as a cognitive filter and summary generator. It distinguishes between newsletters/spam (which are discarded) and actionable emails.

**Input:**
- List of unread emails (with metadata: sender, subject, date, body snippet).

**Prompt Rules:**
- Extract ONLY useful/actionable emails.
- Ignore test emails, spam, promotions, newsletters, random chats, OTPs, and meaningless content.
- Remove duplicate information.
- Format strictly into a clean WhatsApp template.

**Output:**
- A concise WhatsApp message containing:
  - 📌 [Short Title]
  - Summary: [ONE short sentence]
  - Action: [ONE clear action item or "None"]
  - Deadline: [date or "None"]
  - Priority: [Low/Medium/High/Critical]

---

## AI Components

| Component | Purpose |
|------------|----------|
| Anthropic Node (Claude Opus) | Performs semantic analysis of all unread emails to filter out clutter, extract action items, assign priority, and format details. |

---

## Deterministic Components

| Component | Purpose |
|------------|----------|
| Schedule Trigger | Triggers the workflow every 2 hours to keep the user updated. |
| Gmail API Nodes (`main` & `scaler`) | Pulls up to 30 unread emails received after a specified checkpoint. |
| Gmail API Mark-as-Read | Automatically marks messages as read to prevent reprocessing. |
| Merge Node | Merges unread emails fetched from multiple Gmail accounts. |
| JavaScript (n8n Code Nodes) | Cleans incoming JSON metadata and extracts text responses for Twilio. |
| Twilio API Node | Formats and delivers the final text message to the user's WhatsApp number. |

---

## Tools & Integrations

- **n8n**: Workflow orchestration engine.
- **Gmail API**: Email inbox access.
- **Anthropic API (Claude Opus 4.6)**: Advanced reasoning and parsing.
- **Twilio API**: WhatsApp Business API connection.

---

## Workflow Steps

### Step 1: Triggering
The schedule trigger initiates the workflow execution at the specified intervals (configured for every 2 hours).

### Step 2: Parallel Gmail Ingestion
The workflow retrieves up to 30 unread emails from two separate configured Gmail accounts (`scaler` and `main`).

### Step 3: Inbox Management
Simultaneously, the workflow marks the fetched emails as read in the respective Gmail inboxes to maintain inbox hygiene.

### Step 4: Data Merging & Transformation
Emails from both inboxes are merged. The JS code node extracts and sanitizes essential fields: `gmailId`, `from`, `subject`, `body`, and `date`.

### Step 5: AI Summarization & Filtering
The structured email list is forwarded to the Anthropic Claude node, which filters out noise and constructs the WhatsApp digest.

### Step 6: Text Normalization
A second JS code node parses the LLM output, extracting raw text and cleaning the format.

### Step 7: Delivery
The finalized message is sent to the user's WhatsApp number via Twilio.

---

## Sample Input

```json
[
  {
    "gmailId": "msg_12345",
    "from": "careers@google.com",
    "subject": "Google Software Engineering Internship Interview Schedule",
    "body": "Hi student, congratulations! We'd like to schedule your first round interview. Please pick a slot on our Calendly link by June 10, 2026...",
    "date": "2026-06-04T10:00:00Z"
  },
  {
    "gmailId": "msg_67890",
    "from": "newsletter@spammyads.com",
    "subject": "50% off on all items! Buy now!",
    "body": "Special offer only for you! Don't miss this limited-time sale...",
    "date": "2026-06-04T10:15:00Z"
  }
]
```

## Sample Output

```
📌 Google Internship Interview
Summary: Received interview scheduling invitation from Google Careers.
Action: Choose interview slot on the provided Calendly link.
Deadline: June 10, 2026
Priority: Critical
```

---

## Error Handling

- **Parallel Run Safe:** Gmail messages are marked as read in parallel as they are processed, ensuring they aren't parsed twice in case of downstream failures.
- **Empty Inbox Handling:** If no actionable or important emails are found, the LLM outputs "No important emails found.", and Twilio sends this status.

---

## Human-in-the-Loop

The workflow is fully automated. The user acts as the human-in-the-loop by reviewing the alerts on their WhatsApp and manually taking the action items requested (e.g., booking an interview slot, completing an assignment).

---

## Project Structure

```text
Agentic Workflow Design/
│
├── gmail_summarizer_n8n.json             # n8n Workflow JSON export
├── README.md                             # Documentation
└── Screenshot 2026-06-04 at 15.13.24.png # Workflow visualization screenshot
```

---

## Future Improvements

- Add database storage (e.g., PostgreSQL or Supabase) to keep track of processed emails instead of relying entirely on unread status.
- Implement Telegram or Discord bot alternative destinations.
- Integrate calendar auto-booking for identified deadlines.

---

## Learning Outcomes

- Configured parallel multi-inbox Gmail extraction in n8n.
- Implemented LLM-based semantic filtering to distinguish critical emails from newsletter noise.
- Connected n8n with Anthropic LangChain nodes and external webhook messaging (Twilio WhatsApp).

---

## Author

Name: Mayank Gupta

Student ID: 10069

Contribution:
Designed and implemented the complete workflow, prompts, routing logic, integrations, and testing.