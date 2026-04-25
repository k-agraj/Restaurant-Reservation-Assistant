# Restaurant Reservation Assistant

An n8n automation that monitors an inbox (`reservation@agrajk.com`) and autonomously books restaurant reservations, surfaces recommendations, and maintains a watchlist for unavailable tables.

---

## How It Works

Email `reservation@agrajk.com` in plain language. The agent classifies your intent and routes it:

| Intent | Example email | What happens |
|--------|--------------|--------------|
| **Book a table** | "Book Giulia in Cambridge for 2 on May 3rd at 7pm" | Searches for booking platform → checks live availability → books instantly → confirms via email |
| **Get recommendations** | "Italian fine dining in Boston for a special occasion, party of 4" | Searches and ranks 4–5 restaurants with 4+ stars → replies with curated list |
| **Check booking window** | "How far in advance does Menton accept reservations?" | Looks up advance booking window and release schedule → replies with tips |
| **Remove from watchlist** | "Remove Giulia from my watchlist" | Marks the entry removed → confirms via reply |

If a requested restaurant has no availability, the agent:
1. Adds it to a Google Sheets watchlist
2. Replies with alternative restaurants
3. Polls automatically at **midnight and 9 AM daily** (peak cancellation windows)
4. Books the moment a slot opens and emails you a confirmation

---

## Setup

### Prerequisites

| Service | Purpose | Cost |
|---------|---------|------|
| [n8n](https://n8n.io) | Workflow runner | Self-hosted or cloud |
| [Anthropic API](https://console.anthropic.com) | Intent classification + email writing | ~$2–5/month |
| [Serper](https://serper.dev) | Google search for restaurants/platforms | Free tier |
| [Browserless](https://browserless.io) | Headless browser for scraping | Free tier (1k units/month) |
| Google Sheets | Watchlist storage | Free |
| Hostinger email | IMAP/SMTP for `reservation@agrajk.com` | Included in hosting |

---

### Step 1 — Google Sheets Watchlist

Create a Google Sheet named **Reservation Watchlist** with a tab named **Watchlist**.

Add these column headers in row 1 (columns A–Q):

| Col | Header |
|-----|--------|
| A | ID |
| B | Restaurant Name |
| C | Date Requested |
| D | Time Preference |
| E | Party Size |
| F | Booking Platform |
| G | Booking URL |
| H | Venue ID |
| I | Location |
| J | Special Requests |
| K | Status |
| L | Last Checked |
| M | Check Count |
| N | Email Thread ID |
| O | Reply To Email |
| P | Date Added |
| Q | Remove After Date |

**Status values:** `WATCHING` · `BOOKED` · `REMOVED` · `EXPIRED`

Copy the Sheet ID from the URL:
```
https://docs.google.com/spreadsheets/d/<SHEET_ID>/edit
```

---

### Step 2 — n8n Environment Variables

Go to **Settings → Environment Variables** and add:

| Variable | Value |
|----------|-------|
| `SERPER_API_KEY` | From [serper.dev](https://serper.dev) dashboard |
| `ANTHROPIC_API_KEY` | From [console.anthropic.com](https://console.anthropic.com) |
| `BROWSERLESS_API_TOKEN` | From [browserless.io](https://browserless.io) dashboard |
| `WATCHLIST_SHEET_ID` | Your Google Sheet ID |
| `RESY_AUTH_TOKEN` | See below |
| `RESY_PAYMENT_METHOD_ID` | See below |

> **`RESY_API_KEY` is not needed** — the public Resy client key (`vbOpDK1X`) is already hardcoded in the workflow.

#### Getting your Resy auth token
1. Log in at [resy.com](https://resy.com)
2. Open DevTools → Network tab
3. Search any restaurant
4. Find a request to `api.resy.com` and copy the `X-Resy-Auth-Token` header value

#### Getting your Resy payment method ID
After adding a card to your Resy account, call:
```
GET https://api.resy.com/2/user
Authorization: ResyAPI api_key="vbOpDK1X"
X-Resy-Auth-Token: <your token>
```
The `payment_methods[0].id` field is your `RESY_PAYMENT_METHOD_ID`.

---

### Step 3 — n8n Credentials

Create these in **Settings → Credentials → Add**:

**IMAP** (`reservation@agrajk.com IMAP`)
- Host: `imap.hostinger.com` · Port: `993` · Security: SSL/TLS
- User: `reservation@agrajk.com` · Password: your email password

**SMTP** (`reservation@agrajk.com SMTP`)
- Host: `smtp.hostinger.com` · Port: `465` · Security: SSL/TLS
- User: `reservation@agrajk.com` · Password: your email password

**Anthropic API** (`Anthropic API`)
- API Key: your Anthropic key

**Google Sheets OAuth2** (`Google Sheets`)
- Follow the OAuth flow to connect your Google account

---

### Step 4 — Import and Activate

1. In n8n: **New Workflow → Import from File** → select `restaurant-reservation-assistant-workflow.json`
2. Click each node that shows a credential warning and select the matching credential
3. Set the IMAP trigger poll interval to **5 minutes** (default is 1 minute — fine for testing)
4. **Activate** the workflow

---

### Step 5 — Test

Send an email to `reservation@agrajk.com`:

```
Subject: Test Reservation
Body:   Can you book a table for 2 at Pammy's Cambridge this Saturday around 7:30pm?
```

You should get a reply within 5 minutes.

---

## Architecture

```
IMAP Trigger (every 5 min)
  └─ Extract Email Data
       └─ Claude Haiku: Classify Intent
            └─ Route by Intent
                 ├─ BOOK_RESERVATION
                 │    └─ Serper: Find Booking Platform
                 │         └─ Claude Haiku: Identify Platform
                 │              └─ Route by Platform
                 │                   ├─ resy   → Resy Venue Search → Extract Venue ID → Check Resy Availability
                 │                   ├─ opentable → Check OpenTable Availability
                 │                   └─ other  → Browserless: Scrape Booking Page
                 │                        └─ Check Slot Availability
                 │                             ├─ available   → Book (Resy/OpenTable) → Confirmation Email
                 │                             └─ unavailable → Find Similar → Add to Watchlist → Unavailable Email
                 │
                 ├─ FIND_RESTAURANTS
                 │    └─ Serper: Search → Claude Sonnet: Write Recommendations
                 │
                 ├─ CHECK_AVAILABILITY
                 │    └─ Browserless: Scrape → Claude Sonnet: Write Booking Window Reply
                 │
                 └─ REMOVE_WATCHLIST
                      └─ Google Sheets: Mark REMOVED

Watchlist Scheduler (midnight + 9 AM daily)
  └─ Get Active Watchlist (Status = WATCHING)
       └─ Filter Expired Items
            └─ Resy: Check Availability (per item, using stored Venue ID)
                 ├─ slot found → Book Slot → Mark BOOKED → Email Notification
                 └─ no slot   → Increment Check Count → Stay WATCHING
```

---

## Models Used

| Node | Model | Why |
|------|-------|-----|
| Intent classification | `claude-haiku-4-5` | Fast, cheap, structured JSON output |
| Platform identification | `claude-haiku-4-5` | Simple extraction task |
| Alternative suggestions | `claude-sonnet-4` | Better reasoning for similarity matching |
| Email writing | `claude-sonnet-4` | Higher quality prose |

---

## Cost Estimate (Monthly)

| Service | Usage | Est. Cost |
|---------|-------|-----------|
| Anthropic API (Haiku + Sonnet) | ~500 calls/month | ~$2–5 |
| Serper | ~200 searches/month | Free tier |
| Browserless | ~50 sessions/month | Free tier |
| Hostinger email | Included in hosting | $0 |
| Google Sheets | Free | $0 |
| **Total** | | **~$2–5/month** |
