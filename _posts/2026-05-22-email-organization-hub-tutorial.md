---
title: "Stop Managing Multiple Inboxes Like a Psychopath: A Gmail Hub Setup Guide"
date: 2026-05-22 09:00:00 -0800
categories: [Automation, Tutorial]
tags: [email, automation, n8n, gmail, tutorial]
author: jason
pin: false
image:
  path: /assets/img/posts/email-organization-hub.jpg
  alt: Gmail hub with automatic labeling and n8n briefings
---

At some point I had five email accounts and zero system for managing them. One for clients. One for infrastructure alerts. One I'd used for store signups since 2011 that was receiving upwards of 40 promotional emails per day (Bed Bath & Beyond alone was sending three). I'd open my phone in the morning, see an undifferentiated wall of 300 unread messages across multiple apps, and close it again without reading anything. Inbox zero was a myth I'd heard about, like a well-rested parent.

The fix turned out to be simpler than I expected: one [Gmail](https://mail.google.com) account as a hub, everything else forwarding into it, each source auto-labeled on arrival. Add a Python helper for bulk operations and an [n8n](https://n8n.io) daily briefing, and the whole thing mostly runs itself.

**What you'll end up with:** All email in one inbox, each message labeled by source account, a Python script for bulk operations, and a morning briefing posted to [Discord](https://discord.com) with email + calendar + tasks.

**Prerequisites:** Multiple Gmail/Google Workspace accounts, Python 3, n8n running (see the VPS tutorial), a Discord server.

---

## Step 1: Set Up Email Forwarding

For each secondary account, set up forwarding to your hub account.

**For Gmail accounts:**
1. Go to Settings → See all settings → Forwarding and POP/IMAP
2. Add forwarding address → enter your hub Gmail address
3. Confirm the verification email sent to the hub
4. Set to "Forward a copy of incoming mail" (keep a copy in the secondary if you want it)

**For Google Workspace / custom domain accounts:**
- Same process via the Google Workspace admin console, or
- Set up a forwarding rule directly in Gmail Settings if you have admin access

Repeat for every account you want to consolidate. Yes, even the store-signup one. Especially the store-signup one. You'll want to be able to nuke it from orbit later.

---

## Step 2: Create Source Labels in Gmail

In your hub Gmail, create a label for each forwarding account:

1. Go to Settings → Labels → Create new label
2. Create: `Source`, `Source/hello`, `Source/info`, `Source/junk` (or whatever names match your accounts)

The parent `Source` label lets you collapse/expand all source sub-labels in the sidebar. More usefully, it means you can filter by `label:Source/junk` in a script and trash everything in that subfolder in one command, which is deeply satisfying.

---

## Step 3: Create Gmail Filters for Auto-Labeling

For each account that forwards to your hub, create a filter that labels incoming messages by source:

1. Settings → Filters and Blocked Addresses → Create a new filter
2. **To:** enter the original address (e.g. `hello@yourdomain.com`)
3. Create filter → Apply label → select `Source/hello`
4. Check "Also apply filter to matching conversations" to label existing messages
5. Repeat for each account

Now every email arrives at your hub already tagged with where it came from. You can answer "did anything come in for the client-facing address this week?" in about two seconds.

---

## Step 4: Set Up Gmail API Access

You'll need a Google Cloud project with the Gmail API enabled. This is the bureaucratic part: not hard, just a lot of clicking.

1. Go to [Google Cloud Console](https://console.cloud.google.com)
2. Create a new project
3. Enable the **Gmail API**
4. Create OAuth 2.0 credentials (Desktop app type)
5. Download the credentials JSON and save it as `gmail-client-secret.json`

**Scopes needed:** `gmail.readonly` and `gmail.modify` (can trash and read, no permanent delete. Gmail keeps a 30-day undo window)

These are my scopes as a web developer — read access for the briefing, modify for bulk trash and mark-read. I don't need to permanently delete anything; the 30-day trash window is enough of a safety net. If your use case is different (say, you want to auto-archive instead of trash, or you need to send via the API), adjust accordingly. The [Gmail API scopes reference](https://developers.google.com/gmail/api/auth/scopes) lists everything available.

Run the auth flow once to get a token:

```bash
pip install google-auth-oauthlib google-auth-httplib2 google-api-python-client
python3 gmail-auth.py  # Script that opens OAuth flow and saves token
```

If your VPS has `AllowTcpForwarding no` (it should), the browser redirect won't work normally. Do it the ugly-but-functional way:

```bash
OAUTHLIB_INSECURE_TRANSPORT=1 python3 gmail-auth.py
```

Copy the URL it outputs, open it in a browser, paste the code back. Token saves to `gmail-token.json` and auto-refreshes from then on. You do this once and forget about it.

---

## Step 5: Build the Gmail Helper Script

This script is where the real work gets done. The core functions: search by any Gmail query, bulk-trash, and bulk-mark-read. Create `gmail_helper.py`:

```python
#!/usr/bin/env python3
from google.oauth2.credentials import Credentials
from googleapiclient.discovery import build
import json, sys

TOKEN_FILE = '/path/to/gmail-token.json'
SECRET_FILE = '/path/to/gmail-client-secret.json'

def get_service():
    creds = Credentials.from_authorized_user_file(TOKEN_FILE)
    return build('gmail', 'v1', credentials=creds)

def search(svc, query, max_results=50):
    results = svc.users().messages().list(
        userId='me', q=query, maxResults=max_results
    ).execute()
    messages = results.get('messages', [])
    for msg in messages:
        m = svc.users().messages().get(
            userId='me', id=msg['id'], format='metadata',
            metadataHeaders=['Subject','From','Date']
        ).execute()
        headers = {h['name']: h['value'] for h in m['payload']['headers']}
        print(f"  [{msg['id']}]  {headers.get('Date','')}  From: {headers.get('From','')[:40]}  |  {headers.get('Subject','')[:60]}")

def bulk_trash(svc, query, dry_run=False):
    results = svc.users().messages().list(
        userId='me', q=query, maxResults=500
    ).execute()
    messages = results.get('messages', [])
    if not messages:
        print("No messages found.")
        return
    print(f"{'[DRY RUN] Would trash' if dry_run else 'Trashing'} {len(messages)} message(s)...")
    if dry_run:
        search(svc, query)
        return
    ids = [m['id'] for m in messages]
    svc.users().messages().batchModify(
        userId='me',
        body={'ids': ids, 'addLabelIds': ['TRASH'], 'removeLabelIds': ['INBOX']}
    ).execute()
    print(f"Done -- {len(ids)} messages moved to Trash.")

def bulk_mark_read(svc, query):
    results = svc.users().messages().list(
        userId='me', q=query + ' is:unread', maxResults=500
    ).execute()
    messages = results.get('messages', [])
    if not messages:
        print("No unread messages found.")
        return
    ids = [m['id'] for m in messages]
    svc.users().messages().batchModify(
        userId='me',
        body={'ids': ids, 'removeLabelIds': ['UNREAD']}
    ).execute()
    print(f"Done -- {len(ids)} messages marked as read.")
```

Always dry-run first, because trashing 400 emails without checking is a bad time:

```bash
# Dry run -- show what would be trashed
python3 gmail_helper.py bulk-trash "label:Source/junk" --dry-run

# Actually trash
python3 gmail_helper.py bulk-trash "label:Source/junk"

# Mark all read in a label
python3 gmail_helper.py mark-read "label:Source/junk"

# Search
python3 gmail_helper.py search "from:recruiter@company.com"
```

---

## Step 6: Build the Daily Briefing in n8n

This is the part I use every day. A workflow that fires twice daily, grabs unread email, today's calendar events, and overdue tasks, formats everything into a single message, and posts it to Discord.

Create a new workflow in [n8n](https://docs.n8n.io) with these nodes:

### Schedule Trigger
- Cron expression: `0 8,20 * * *` (fires at 8am and 8pm in your n8n timezone)
- If your n8n container has `GENERIC_TIMEZONE: America/Los_Angeles` set, this runs at 8am and 8pm PDT. No UTC offset math required.

### Get Emails (Gmail node)
- Operation: Get Many
- Search query: `is:unread newer_than:9h`
- Limit: 10
- Simplify: ON

**Note:** With Simplify ON, Gmail returns field names capitalized (`From`, `Subject`, `Date`). Use those exact names in your downstream code nodes or you'll get undefined everywhere and spend 20 minutes confused.

### Get Tasks (HTTP Request node)
- URL: `https://api.todoist.com/api/v1/tasks?limit=200`
- Auth: Bearer token (your [Todoist](https://todoist.com) API token)
- Response: JSON

**Note:** Todoist API v1 wraps responses in `{ results: [], next_cursor }`. It's not a bare array. Access tasks as `taskData.results`.

note: I use [Todoist](https://todoist.com/), have for years, but you're welcome to figure out whatever task manager you use.

### Get Calendar (Google Calendar node)
- Operation: Get Many
- Calendar: primary
- Time range: today (use expressions for start/end of day in your timezone)

### Format Message (Code node)

```javascript
const pdtToday = new Date().toLocaleDateString('en-CA', {timeZone: 'America/Los_Angeles'});

// Deduplicate calendar events by id (multi-day events repeat per overlapping day)
const seen = {};
const calItems = $('Get Calendar').all().filter(e => {
  const key = e.json.id || e.json.summary;
  if (seen[key]) return false;
  seen[key] = true;
  return true;
});

// Emails use capitalized field names (Simplify ON)
const emailItems = $('Get Emails').all();

// Tasks from results array
const taskData = $('Get Tasks').first().json;
const tasks = (taskData.results || []).filter(t => {
  if (!t.due) return false;
  return t.due.date.substring(0, 10) <= pdtToday;
});

// Build your formatted message here
const message = `**Today's Briefing**\n\n` +
  `📅 **Calendar**\n${calItems.map(e => `• ${e.json.summary}`).join('\n') || 'Nothing scheduled'}\n\n` +
  `✉️ **Email**\n${emailItems.map(e => `• ${e.json.From}: ${e.json.Subject}`).join('\n') || 'No new email'}\n\n` +
  `✅ **Tasks**\n${tasks.map(t => `• ${t.content}`).join('\n') || 'All clear'}`;

return [{ json: { message } }];
```

### Post to Discord (HTTP Request node)
- URL: your Discord webhook URL
- Method: POST
- Body: `{ "content": "{% raw %}{{ $json.message }}{% endraw %}" }`

**Important: node error settings.** On any node that hits an external API (Get Tasks, Get Calendar, Get Emails), open its Settings tab and set:
- **Retry On Fail:** ON, Max Tries: 3
- **On Error:** Continue (so a Todoist 503 at 8am doesn't mean you also miss your calendar)

Test it, then activate the workflow.

---

## Step 7: Job Application Follow-up Workflow (Bonus)

If you're job hunting, this one is worth the five minutes to set up. Label an email "Job Alerts/Applied" in Gmail and n8n automatically creates a follow-up task in Todoist.

**Gmail Trigger node:**
- Event: Message Received
- Filter: Label = `Job Alerts/Applied`
- Poll every: 15 minutes

**HTTP Request node (Todoist API):**
- URL: `https://api.todoist.com/api/v1/tasks`
- Method: POST
- Auth: Bearer token
- Body parameters:
  - `content`: `Follow up: {% raw %}{{ $json.Subject }}{% endraw %}`
  - `project_id`: your Todoist project ID

Find your project ID via:
```bash
curl -H "Authorization: Bearer YOUR_TOKEN" https://api.todoist.com/api/v1/projects
```

---

## Step 8: Configure Desktop Clients to Send From Any Address

Forwarding handles receiving. To send from your secondary addresses in any client, you first need to add a send-as alias in Gmail. That's the prerequisite for everything below.

### Part A: Add the alias in Gmail (web, required first)

1. Gmail → Settings → **Accounts and Import** → **Send mail as** → Add another email address
2. Name: your name / Email: `hello@yourdomain.com`
3. Uncheck "Treat as alias" (keeps it as a distinct From identity)
4. SMTP server: `smtp.gmail.com`, port 587, STARTTLS. Use your hub account credentials.
5. Gmail sends a verification email to hello@. Click the link to confirm.

Repeat for every address you want to send from. Then configure your client.

### Part B: Thunderbird

1. **Account Settings** → select your hub account → **Manage Identities**
2. Click **Add**:
   - Name: your name
   - Email: `hello@yourdomain.com`
3. Save. When composing, click the **From** dropdown to switch identities.

### Part C: Outlook (Windows and Mac)

Outlook uses the alias you configured in Gmail. No extra setup in Outlook itself:

1. Open a new message
2. If the **From** field isn't visible: Options → **From**
3. Click the **From** field → **Other Email Address…**
4. Type `hello@yourdomain.com` and click **OK**

Outlook remembers addresses you've used, so next time it appears in the From dropdown directly. Mac is identical: New Message → From field dropdown → **Other Email Address…**

### Part D: Apple Mail

1. **Mail → Settings** (Cmd+,) → **Accounts** → select your hub account
2. **Account Information** tab → **Email Address** field
3. Add a comma after your hub address, then type `hello@yourdomain.com`:
   ```
   hubaccount@gmail.com, hello@yourdomain.com
   ```
4. Close and save. The From dropdown appears when composing.

### Part E: Brevo as an SMTP Relay (Custom Domains)

If you're sending from a custom domain and want better deliverability than routing through Gmail's SMTP, [Brevo](https://www.brevo.com/) is worth considering. Their free tier includes 300 transactional emails per day, and the SMTP credentials work in any client.

1. Create a free account at brevo.com
2. Go to **SMTP & API** → **SMTP** and note your login and SMTP key
3. In your email client, use these settings instead of Gmail's SMTP:
   - SMTP server: `smtp-relay.brevo.com`
   - Port: 587, STARTTLS
   - Login: your Brevo account email
   - Password: your SMTP key (not your account password)

The send-as alias from Part A is still required. Brevo handles the outbound SMTP leg; Gmail still needs to know you're authorized to send as that address.

### Other clients

Most clients (Spark, Airmail, Mimestream) have an **Aliases** or **Identities** section in account settings. The Gmail send-as alias from Part A is always the prerequisite.

---

## Maintenance

**Weekly junk sweep.** The one thing worth doing manually, with a dry run first:

```bash
# Confirm the list before you commit
python3 gmail_helper.py bulk-trash "label:Source/junk" --dry-run
# Then pull the trigger
python3 gmail_helper.py bulk-trash "label:Source/junk"
```

**If OAuth token expires:** The refresh token handles it automatically. If it fully expires (rare, but it happens if you revoke access or the token goes unused for six months) just re-run the auth flow.

---

## Troubleshooting

**n8n calendar events duplicating in the briefing:** Multi-day events appear once per overlapping day in the [Google Calendar API](https://developers.google.com/calendar/api/guides/overview) response. The deduplication block in the Format Message node handles this. If you removed it, put it back.

**Gmail filter isn't labeling forwarded messages:** Check that the filter is matching the `To:` field, not `From:`. Forwarded messages from `hello@yourdomain.com` arrive with that as the original destination. Filter on it, not on the hub address.

**Todoist tasks showing wrong "today":** The code node uses `America/Los_Angeles` for the date. If your VPS is in a different timezone, this is correct behavior. The script compares against PST/PDT, not server time.

---

*Part of a series on building a practical, low-cost homelab with AI agents, self-hosted automation, and a Tailscale backbone.*
