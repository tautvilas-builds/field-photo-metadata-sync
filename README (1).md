# Automated Field Photo Documentation & Metadata Sync (n8n Workflow)

A scalable n8n pipeline that ingests site photos via Telegram, uploads them to Google Drive, and automatically logs structured reports into Google Sheets — with built-in photo grouping, job completion tracking, and buffer safeguards.

Works out-of-the-box for: **Field Service Teams, Construction, Renewable Energy (Solar/Wind), Logistics, Property Management, and Landscaping.**

---

## The Problem

Field workers take before/after photos at job sites and send them through random channels — Viber, WhatsApp, Telegram, SMS. Managers waste 30–60 minutes daily collecting, sorting, and matching photos to jobs. Photos get lost, mislabeled, or forgotten entirely.

## The Solution

One Telegram bot. Workers send photos → system handles everything automatically:

- **Collects** any number of photos (1–12 per job) into a buffer
- **Uploads** each photo to Google Drive instantly
- **Groups** all photos from one job into a single spreadsheet row
- **Closes** the job when the worker taps "✅ Finish Job"
- **Auto-closes** forgotten jobs after 7 hours (with a reminder at 6 hours)
- **Notifies** workers at every step so they know their photos landed

No apps to install. No training. If they can send a photo on Telegram, they can use this.

---

## Architecture

The system consists of **two n8n workflows** and one **n8n Data Table** (internal buffer):

### Workflow 1: Field Photo Collector (Universal)

```
Telegram Bot receives message
        │
        ▼
   Photo or /done?
   ┌────┴────┐
   │         │
 Photo    /done or
   │      ✅ Finish Job
   ▼         │
Get File     ▼
   │      Get Buffer
   ▼         │
Drive Upload ▼
   │      Append to Sheets
   ▼      (Photo_1...Photo_12)
Check Buffer  │
   │         ▼
   ▼      Clear Buffer
Buffer        │
exists?       ▼
 ┌──┴──┐   "✅ Job registered!
 │     │    Total photos: X"
YES    NO
 │     │
Update Create
Buffer Buffer
 │     │
 └──┬──┘
    ▼
"📷 Photo received.
 Send more or press
 Finish Job."
 [✅ Finish Job]
```

### Workflow 2: Buffer Safeguard

```
Schedule Trigger (every 20 min)
        │
        ▼
  Get all open buffers
        │
        ▼
  Calculate hours elapsed
        │
        ▼
   Route by time
   ┌────┼────┐
   │    │    │
  7h+  6h+  <6h
   │    │    │
Auto  Remind Skip
Close  worker
   │    │
   ▼    ▼
Sheets "⏰ Reminder:
+Clear  press Finish
Buffer  Job"
   │
   ▼
"⚠️ Job auto-
 closed after
 7 hours"
```

---

## Features

- **Universal photo count** — accepts 1 to 12 photos per job, no fixed before/after structure
- **Inline keyboard button** — workers tap "✅ Finish Job" instead of typing commands (with `/done` as fallback)
- **Smart buffering** — uses n8n Data Tables to track each worker's active job independently (supports 50+ concurrent workers)
- **Auto-close safeguard** — forgotten jobs are automatically closed after 7 hours, with a reminder sent at 6 hours
- **Structured output** — each job becomes one row in Google Sheets with photos in separate columns (Photo_1 through Photo_12)
- **Retry on failure** — all critical nodes retry 3 times on API errors (Telegram rate limits, Drive timeouts)
- **Zero worker training** — send photos, press a button, done

---

## Output Format

### Google Sheets

| Date | Employee | Telegram_ID | Photo_1 | Photo_2 | ... | Photo_12 | Total |
|------|----------|-------------|---------|---------|-----|----------|-------|
| 2026-08-20 12:24 | Tautvilas | 804889224 | [Drive link] | [Drive link] | ... | | 3 |

### Google Drive

Photos are uploaded with descriptive filenames:
```
Tautvilas_20260820_122415_177.jpg
```
Format: `{EmployeeName}_{Date}_{Time}_{MessageID}.jpg`

---

## Requirements

- **n8n Cloud** (or self-hosted n8n v1.113.1+ with Data Tables enabled)
- **Telegram Bot** (created via [@BotFather](https://t.me/BotFather))
- **Google Cloud Project** with Drive API and Sheets API enabled
- **Google OAuth2 credentials** configured in n8n

---

## Setup Guide

### 1. Create the Telegram Bot

1. Open Telegram, find `@BotFather`
2. Send `/newbot`, follow the prompts
3. Save the bot token (you'll need it for n8n credentials)

### 2. Create the n8n Data Table

In n8n, go to **Data Tables → Create Data Table** named `photo_buffer` with these columns:

| Column | Type |
|--------|------|
| `telegram_id` | String |
| `sender_name` | String |
| `photo_links` | String |
| `photo_count` | String |
| `started_at` | String |
| `reminded` | String |

### 3. Create the Google Sheet

Create a Google Sheet with these column headers in Row 1:

```
Date  Employee  Telegram_ID  Photo_1  Photo_2  Photo_3  Photo_4  Photo_5  Photo_6  Photo_7  Photo_8  Photo_9  Photo_10  Photo_11  Photo_12  Total
```

### 4. Import the Workflows

1. In n8n, go to **Workflows → Import from File**
2. Import `Field_Photo_Collector_Universal.json`
3. Import `Buffer_Safeguard.json`

### 5. Configure Credentials

In each workflow, connect your credentials:

- **Telegram Bot** — paste your bot token from BotFather
- **Google Drive OAuth2** — connect your Google account
- **Google Sheets OAuth2** — connect your Google account

### 6. Re-link Data Tables

> ⚠️ **Important:** After importing, n8n Data Table nodes lose their internal reference. You must manually re-select `photo_buffer` in every Data Table node:

**Field Photo Collector:**
- Check Buffer
- Create Buffer
- Update Buffer
- Get Buffer
- Clear Buffer

**Buffer Safeguard:**
- Get All Buffers
- Clear Buffer (Auto)
- Mark Reminded

### 7. Configure Google Sheets Reference

In both workflows, open the Google Sheets node(s) and select your sheet from the dropdown (or paste the Sheet ID).

### 8. Activate

1. Set **Field Photo Collector** to **Active**
2. Set **Buffer Safeguard** to **Active**
3. Make sure any old/duplicate workflows using the same bot are **Inactive** (Telegram allows only one trigger per bot)

---

## File Structure

```
├── README.md
├── Field_Photo_Collector_Universal.json    # Main workflow
└── Buffer_Safeguard.json                   # Auto-close safeguard
```

---

## How Workers Use It

1. Open the Telegram bot
2. Send site photos (as many as needed)
3. Bot confirms each photo: *"📷 Photo received. Send more or press Finish Job."*
4. When done, tap **✅ Finish Job**
5. Bot confirms: *"✅ Job registered! Total photos: X"*

If a worker forgets to press Finish Job:
- After **6 hours** → bot sends a reminder
- After **7 hours** → job is automatically closed and logged

---

## Configuration

### Timing (Buffer Safeguard)

Edit the `Calculate Time` Code node to adjust thresholds:

```javascript
if (hoursElapsed >= 7) {
  action = 'auto_close';    // Change 7 to your preferred auto-close hours
} else if (hoursElapsed >= 6 && item.json.reminded === 'no') {
  action = 'remind';        // Change 6 to your preferred reminder hours
}
```

Schedule Trigger interval (default: 20 minutes) can be adjusted in the Schedule Trigger node.

### Max Photos

The system supports up to 12 photos per job. To increase:
1. Add more `Photo_N` columns to your Google Sheet
2. Add corresponding `split(',')[N]` expressions in both the main workflow (Append to Sheets) and safeguard (Auto Close to Sheets)

---

## Error Handling

- All critical nodes have **Retry On Fail** enabled (3 attempts, 2-second intervals)
- Covers: Telegram API rate limits, Google Drive timeouts, Sheets API errors, Data Table operations
- Optional: Create an Error Notification workflow with n8n's Error Trigger for real-time alerts

---

## Security Notes

- ⚠️ **Never commit credentials** — the exported JSON files contain credential references (IDs), not actual tokens. Always verify before pushing.
- Bot token = full access to your bot. Keep it secret.
- Google OAuth tokens are managed by n8n and not included in exports.
- Consider restricting bot access to known Telegram IDs (allowlist) for production use.

---

## Scaling

Tested for 50+ concurrent workers. Key considerations:

- **Race conditions:** Minimal risk because each worker's buffer is keyed by unique `telegram_id`
- **Album handling:** Workers sending multiple photos simultaneously are processed sequentially by n8n's execution queue
- **Concurrency:** n8n Cloud manages execution limits per plan. Monitor during peak hours (e.g., end of workday when all workers submit at once)

---

## License

MIT

---

## Author

Built by **Tautvilas Sakalauskas** / WorkFinity — Operations Infrastructure & Process Automation for Field Operations (tautvilas@workfinity.eu).

Powered by [n8n](https://n8n.io) workflow automation.
