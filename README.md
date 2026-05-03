# AU Physio: SMS Appointment Confirmations (Twilio + Google Calendar)

A free n8n workflow for Australian allied-health clinics. When a Google Calendar event is created or updated, your patient automatically gets an SMS confirmation with the appointment details and a reschedule link.

Built specifically for AU clinics — handles `+61` mobile formatting, AWST/AEST/ACST time zones, AHPRA-safe message copy, and Spam Act 2003 opt-out wording.

> **DRAFT-ONLY DISCLAIMER:** This workflow handles appointment logistics. It does not provide clinical advice, store clinical records, or replace patient consent. The clinic owner is responsible for compliance with the Privacy Act 1988, AHPRA advertising guidelines, and the Spam Act 2003.

---

## What it does

- ✉️ Sends an SMS the moment you create or update an appointment in Google Calendar
- 📱 Normalises any AU mobile format (e.g. `0412 345 678`, `+61412345678`, `(04) 1234 5678`) to a valid international number before sending
- 🚫 Skips internal/staff/admin events automatically
- 🚫 Skips landlines (no failed-SMS charges)
- 🚫 Skips events you mark with `[skip-sms]` in the description
- 🔁 Won't send a duplicate SMS if you re-edit an event within 60 minutes
- 📊 Logs every send/skip/failure to a Google Sheet for audit
- 🔔 Emails you if anything errors

---

## Prerequisites

| Item | Notes |
|---|---|
| n8n | Self-hosted (Docker) or Cloud n8n |
| Google account | For Calendar, Sheets, and Gmail credentials |
| Twilio account | AU virtual number required (~AUD 8/mo). Trial credit covers setup. |
| Gmail mailbox | For error alerts (e.g. `admin@yourclinic.com.au`) |

---

## Setup

This section is written for clinic staff. You do not need to code. You will import two files, connect your clinic accounts, paste a few values into workflow variables, then run one test appointment.

If you prefer to skip the technical configuration, I offer a complete installation service for a small flat fee. I will set up your Google Cloud OAuth, Twilio, and n8n connections so the workflow is ready to use immediately. Email wilike0511@gmail.com to arrange a setup.

### Helpful external setup documents

Keep these pages open while setting up:

| Task | External document |
|---|---|
| Import workflow JSON files into n8n | [n8n: Export and import workflows](https://docs.n8n.io/workflows/export-import/) |
| Connect Google Calendar / Sheets / Gmail to n8n | [n8n: Google credentials](https://docs.n8n.io/integrations/builtin/credentials/google/) |
| Set up self-hosted Google OAuth2 | [n8n: Google OAuth2 single service](https://docs.n8n.io/integrations/builtin/credentials/google/oauth-single-service/) |
| Find a Google Calendar ID | [Google Calendar Help: Sync your calendar with computer programs](https://support.google.com/calendar/answer/37648?hl=en) |
| Understand n8n variables | [n8n: Custom variables](https://docs.n8n.io/code/variables/) |
| Buy/check a Twilio phone number | [Twilio: Phone Numbers documentation](https://www.twilio.com/docs/phone-numbers) |
| Check Twilio number SMS capability | [Twilio Help: Phone number availability and capabilities](https://help.twilio.com/articles/223183068) |
| Twilio Australia SMS pricing | [Twilio: Australia SMS pricing](https://www.twilio.com/en-us/sms/pricing/au) |
| Gmail node in n8n | [n8n: Gmail node documentation](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.gmail/) |

### Before you start

Have these ready:

| What you need | Example | Where to get it |
|---|---|---|
| n8n login | `https://your-clinic.n8n.cloud` | Your n8n account or server admin |
| Google login | `reception@yourclinic.com.au` | The Google account that can access clinic calendar and sheets |
| Twilio login | Twilio Console account | Your Twilio account owner |
| Clinic calendar ID | `primary` or `bookings@yourclinic.com.au` | Google Calendar settings |
| Clinic name | `Perth Physio Co` | Your public clinic name |
| Reschedule/cancel link | `https://yourclinic.com.au/reschedule` | Your booking page or contact page |
| Twilio sender number | `+61480012345` | Twilio Console |
| Alert email | `manager@yourclinic.com.au` | Inbox for workflow error emails |

### 1. Import the two workflow files

1. Log in to n8n.
2. Open **Workflows**.
3. Choose **Import from File**.
4. Select `workflow-1-sms-confirmation.json`.
5. Save the workflow.
6. Repeat the same steps for `workflow-1-sms-confirmation-errors.json`.

You should now see two workflows:

| Workflow | Purpose |
|---|---|
| `WF-01 — SMS Appointment Confirmation` | Sends appointment SMS and writes audit rows |
| `WF-01 ERR — SMS Confirmation Errors` | Emails you if the main workflow fails |

External help: [n8n: Export and import workflows](https://docs.n8n.io/workflows/export-import/)

### 2. Connect Google credentials

This workflow uses Google for Calendar, Sheets, and Gmail error alerts.

Use the same Google account if possible, usually the reception/admin account. That account needs:

| Google service | Access needed |
|---|---|
| Google Calendar | Can view the appointment calendar |
| Google Sheets | Can edit the audit sheet |
| Gmail | Can send alert emails |

First choose the right setup path:

| Your n8n type | What to do |
|---|---|
| n8n Cloud | Use **Path A** below. This is the easiest option. |
| Self-hosted n8n, Docker n8n, or n8n running on your own server | Use **Path B** below. You must create Google OAuth credentials in Google Cloud first. |

#### Path A: n8n Cloud Google setup

Use this path if your n8n address looks like `https://your-workspace.app.n8n.cloud` or your n8n account is hosted by n8n Cloud.

In n8n Cloud:

1. Open the imported workflow `WF-01 — SMS Appointment Confirmation`.
2. Click the **Google Calendar Trigger — Event Created** node.
3. Under **Credential**, create or select a Google Calendar credential.
4. Name it exactly: `Google Calendar - Clinic`.
5. Click **Sign in with Google**.
6. Sign in with the Google account that can view the clinic calendar.
7. Allow the requested access.
8. Click the **Sheets: Read Audit Log** node.
9. Create or select a Google Sheets credential.
10. Name it exactly: `Google Sheets - Clinic`.
11. Click **Sign in with Google** and allow access.
12. Click the **Sheets: Append Audit Row** node.
13. Select the same `Google Sheets - Clinic` credential.
14. Open the error workflow `WF-01 ERR — SMS Confirmation Errors`.
15. Click **Gmail: Send Error Alert**.
16. Create or select a Gmail credential.
17. Name it exactly: `Gmail - Clinic Alerts`.
18. Click **Sign in with Google** and allow access.

External help: [n8n: Google credentials](https://docs.n8n.io/integrations/builtin/credentials/google/) and [n8n: Gmail node documentation](https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.gmail/)

#### Path B: Self-hosted Google OAuth2 setup

Use this path if n8n is running on your own server, Docker, VPS, NAS, or internal clinic infrastructure.

Self-hosted n8n normally cannot use the simple n8n Cloud **Sign in with Google** setup. You must first create Google OAuth2 credentials in Google Cloud Console, then paste the Client ID and Client Secret into n8n.

##### B1. Create a Google Cloud project

1. Open [Google Cloud Console](https://console.cloud.google.com/).
2. Sign in with the clinic Google account or Google Workspace admin account.
3. Click the project dropdown at the top.
4. Click **New Project**.
5. Name it something clear, for example:

```text
n8n Clinic Automation
```

6. Click **Create**.
7. Make sure the new project is selected at the top of Google Cloud Console.

##### B2. Enable the required Google APIs

In the same Google Cloud project:

1. Open **APIs & Services**.
2. Open **Library**.
3. Search for and enable each API below.

| API | Why it is needed |
|---|---|
| Google Calendar API | Read appointment events from Google Calendar |
| Google Sheets API | Read and write the audit log |
| Google Drive API | Required by Google Sheets access in n8n |
| Gmail API | Send workflow error alert emails |

##### B3. Configure the OAuth consent screen

1. In Google Cloud Console, open **APIs & Services**.
2. Open **OAuth consent screen**.
3. Click **Get started** if prompted.
4. App name:

```text
n8n Clinic Automation
```

5. User support email: use the clinic/admin email.
6. Audience:
   - Choose **Internal** if your clinic uses Google Workspace and only staff from the same organisation will connect.
   - Choose **External** if you are using a normal Gmail account or the connecting account is outside the Workspace.
7. Add your contact email.
8. Accept Google's user data policy if prompted.
9. Save.

If Google asks for scopes, add the Calendar, Sheets, Drive, and Gmail scopes requested by n8n. For a small clinic setup, using Google's testing mode is usually enough during setup.

##### B4. Choose Testing or Production access

If you selected **External**, Google gives you two practical options.

| Option | Best for | What happens |
|---|---|---|
| Keep app in **Testing** and add yourself as a test user | Fast clinic setup, one clinic Google account, first-time installation | Only listed test users can connect. Google may show an unverified-app warning. Tokens can expire and may need reconnecting. |
| Publish app to **Production** | Longer-term setup, multiple staff accounts, fewer reconnect issues | Google may require OAuth verification if sensitive/restricted scopes are requested. This can require privacy policy details, scope justification, and Google review. |

For a quick setup, leave the app in **Testing** and add the Google account you will use in n8n:

1. Open **OAuth consent screen**.
2. Open **Audience**.
3. Find **Test users**.
4. Add the Google email address you will sign in with from n8n.
5. Save.

Without this, Google may block sign-in with a message like `Access blocked` or `Google hasn't verified this app`.

For a more permanent setup, publish the OAuth app:

1. Open **OAuth consent screen**.
2. Open **Audience** or **Publishing status**.
3. Choose **Publish app** or move the app from **Testing** to **Production**.
4. If Google asks for verification, follow the prompts and provide the requested app details.

Do not publish blindly if Google asks for sensitive or restricted scope verification. Calendar, Gmail, Sheets, or Drive access can trigger extra review depending on the scopes requested. If Google asks for verification, the clinic owner or IT admin should complete it because they may need to confirm the clinic domain, support email, privacy policy, and reason for each Google permission.

##### B5. Create OAuth client credentials

1. In Google Cloud Console, open **APIs & Services**.
2. Open **Credentials**.
3. Click **Create Credentials**.
4. Choose **OAuth client ID**.
5. Application type: choose **Web application**.
6. Name:

```text
n8n Clinic OAuth Client
```

7. Leave this browser tab open.

##### B6. Copy the n8n OAuth Redirect URL

Now go back to n8n:

1. Open `WF-01 — SMS Appointment Confirmation`.
2. Click **Google Calendar Trigger — Event Created**.
3. Create a Google Calendar credential.
4. Choose **Custom OAuth2** if n8n asks for an authentication type.
5. n8n will show an **OAuth Redirect URL** or **Redirect URL**.
6. Copy that URL exactly.

It usually looks similar to:

```text
https://your-n8n-domain.com/rest/oauth2-credential/callback
```

##### B7. Paste the redirect URL into Google

Go back to the Google Cloud **OAuth client ID** screen:

1. Find **Authorized redirect URIs**.
2. Click **Add URI**.
3. Paste the exact n8n OAuth Redirect URL.
4. Click **Create**.
5. Copy the **Client ID**.
6. Copy the **Client Secret**.

##### B8. Finish each Google credential in n8n

Back in n8n, finish the credentials:

1. Paste the **Client ID** into the n8n Google credential.
2. Paste the **Client Secret** into the n8n Google credential.
3. Click **Sign in with Google**.
4. Sign in with the clinic Google account.
5. Allow access.
6. Save the credential as `Google Calendar - Clinic`.

Repeat this same Custom OAuth2 process for:

| n8n credential | Node to open |
|---|---|
| `Google Calendar - Clinic` | `Google Calendar Trigger — Event Created` and `Google Calendar Trigger — Event Updated` |
| `Google Sheets - Clinic` | `Sheets: Read Audit Log` and `Sheets: Append Audit Row` |
| `Gmail - Clinic Alerts` | `Gmail: Send Error Alert` in the error workflow |

You can use the same Google Cloud project and OAuth client details for the clinic's Google credentials unless your n8n screen requires a separate credential type.

##### B9. Common Google OAuth2 errors

| Error | Likely fix |
|---|---|
| `redirect_uri_mismatch` | The redirect URL in Google does not exactly match the URL shown in n8n. Copy it again and update Google. |
| `Access blocked` | Add the Google account as a test user, or change the OAuth app audience/settings. |
| `Google hasn't verified this app` | For clinic internal use, continue only if this is your own Google Cloud project. Add the signing-in email as a test user. |
| Credential works for a few days then disconnects | External apps in Testing can have expiring tokens. Reconnect the credential or publish/verify the OAuth app if needed. |
| Google Sheets credential connects but sheet access fails | Make sure Google Sheets API and Google Drive API are both enabled, and the Google account can edit the audit sheet. |

External help: [n8n: Google OAuth2 single service](https://docs.n8n.io/integrations/builtin/credentials/google/oauth-single-service/)

After completing Path A or Path B, confirm these credential names are selected on the workflow nodes:

| Workflow | Node | Credential to select |
|---|---|---|
| `WF-01 — SMS Appointment Confirmation` | `Google Calendar Trigger — Event Created` | `Google Calendar - Clinic` |
| `WF-01 — SMS Appointment Confirmation` | `Google Calendar Trigger — Event Updated` | `Google Calendar - Clinic` |
| `WF-01 — SMS Appointment Confirmation` | `Sheets: Read Audit Log` | `Google Sheets - Clinic` |
| `WF-01 — SMS Appointment Confirmation` | `Sheets: Append Audit Row` | `Google Sheets - Clinic` |
| `WF-01 ERR — SMS Confirmation Errors` | `Gmail: Send Error Alert` | `Gmail - Clinic Alerts` |

### 3. Configure the Google Calendar Triggers

The **Google Calendar Trigger** tells n8n which calendar to watch and which calendar changes should start the workflow.

The n8n Google Calendar Trigger uses a single **Trigger On** selection. To watch both new appointments and edited appointments, this workflow uses two Google Calendar Trigger nodes:

| Trigger node | Trigger On value |
|---|---|
| `Google Calendar Trigger — Event Created` | `Event Created` |
| `Google Calendar Trigger — Event Updated` | `Event Updated` |

Open `WF-01 — SMS Appointment Confirmation`, click **Google Calendar Trigger — Event Created**, then set:

| Field | What to select |
|---|---|
| **Credential** | `Google Calendar - Clinic` |
| **Poll Times → Mode** | `Every Minute` |
| **Calendar** | Select the clinic booking calendar from the list. If you do not see it, use the calendar ID from step 6. |
| **Trigger On** | `Event Created` |

Then click **Google Calendar Trigger — Event Updated** and set:

| Field | What to select |
|---|---|
| **Credential** | `Google Calendar - Clinic` |
| **Poll Times → Mode** | `Every Minute` |
| **Calendar** | Select the same clinic booking calendar |
| **Trigger On** | `Event Updated` |

Both trigger nodes should connect directly into **Set: Extract Inputs**. Do not use only `Event Updated`. If you do, newly created appointments may not send an SMS until someone edits the appointment.

### 4. Connect Twilio

Twilio is the SMS provider. You need one AU SMS-capable Twilio number.

In Twilio:

1. Log in to the [Twilio Console](https://console.twilio.com/).
2. Buy or select an Australian phone number that can send SMS.
3. Copy the number in international format, for example `+61480012345`.
4. Copy your Twilio **Account SID** and **Auth Token** from the Twilio Console.

In n8n:

1. Open `WF-01 — SMS Appointment Confirmation`.
2. Click **Twilio: Send SMS**.
3. Create a new Twilio credential.
4. Name it exactly: `Twilio - Clinic AU`.
5. Paste the Twilio Account SID and Auth Token.
6. Save the credential.

External help: [Twilio: Phone Numbers documentation](https://www.twilio.com/docs/phone-numbers), [Twilio Help: Phone number availability and capabilities](https://help.twilio.com/articles/223183068), and [Twilio: Australia SMS pricing](https://www.twilio.com/en-us/sms/pricing/au)

### 5. Create the audit sheet

The audit sheet records whether each SMS was sent, skipped, or failed. It does not store patient names, date of birth, clinical notes, or calendar descriptions.

1. Open Google Sheets.
2. Create a new spreadsheet named `wf-01 sms confirmation audit log`.
3. Rename the first tab to exactly `audit_log`.
4. Open the included file `audit-log-template.csv`.
5. Copy the header row into row 1 of the Google Sheet.

The header row must be exactly:

```csv
timestamp_iso,event_id,event_start_iso,phone_e164,status,twilio_sid,error_message
```

Now copy the Google Sheet ID:

1. Look at the Google Sheet URL.
2. Copy the long text between `/d/` and `/edit`.

Example:

```text
https://docs.google.com/spreadsheets/d/1AbCDefGHIjkLmnoPQRstuVwxyz1234567890/edit
```

In this example, the Sheet ID is:

```text
1AbCDefGHIjkLmnoPQRstuVwxyz1234567890
```

### 6. Find your Google Calendar ID

If this workflow watches the main calendar for the Google account, use:

```text
primary
```

If this workflow watches a shared clinic calendar:

1. Open Google Calendar.
2. On the left, find the calendar used for bookings.
3. Click the three dots next to the calendar name.
4. Open **Settings and sharing**.
5. Find **Integrate calendar**.
6. Copy **Calendar ID**.

The Calendar ID often looks like an email address, for example:

```text
bookings@yourclinic.com.au
```

### 7. Set clinic-specific values

> **Note — n8n Variables require an Enterprise plan** and are not available in the free/personal/community edition. This workflow uses a **Clinic Config** node instead — a single Edit Fields node near the start of the canvas where you enter all your clinic details in one place.

#### 7a. Four values in the Clinic Config node

1. Open `WF-01 — SMS Appointment Confirmation`.
2. Click the **Clinic Config** node (it is the first node after the two Google Calendar triggers).
3. You will see four fields listed. Replace each placeholder value with your own:

| Field | Replace with | Example |
|---|---|---|
| `clinic_name` | Your clinic's name as it should appear in the SMS | `Perth Physio Co` |
| `reschedule_url` | The link patients use to reschedule or cancel | `https://perthphysio.com.au/reschedule` |
| `twilio_from_number` | Your Twilio AU sender number | `+61480012345` |
| `audit_sheet_id` | The Sheet ID from step 5 | `1AbCDefGHIjkLmnoPQRstuVwxyz1234567890` |

4. Click **Back** to close the node. Do not save the workflow yet.

#### 7b. Calendar ID — Google Calendar Trigger nodes

The triggers default to your primary Google Calendar. If your clinic bookings live on a different calendar, update both triggers:

1. Click **Google Calendar Trigger — Event Created**.
2. In the **Calendar** field, enter your Calendar ID from step 6 (e.g. `bookings@yourclinic.com.au`).
3. Click **Back** and repeat for **Google Calendar Trigger — Event Updated**.

If you use your primary calendar, leave both triggers as-is — they already fall back to `primary`.

#### 7c. Alert email — error workflow

1. Open `WF-01 ERR — SMS Confirmation Errors`.
2. Click the **Gmail: Send Error Alert** node.
3. In the **To** field, replace `alerts@yourclinic.com.au` with the email address that should receive error alerts.
4. Click **Back**.

#### Summary of values to enter

| Value | Where to enter it | Example |
|---|---|---|
| Clinic name | Clinic Config node → `clinic_name` | `Perth Physio Co` |
| Reschedule URL | Clinic Config node → `reschedule_url` | `https://perthphysio.com.au/reschedule` |
| Twilio sender number | Clinic Config node → `twilio_from_number` | `+61480012345` |
| Audit sheet ID | Clinic Config node → `audit_sheet_id` | `1AbCDefGHIjkLmnoPQRstuVwxyz1234567890` |
| Calendar ID | Google Calendar Trigger nodes (×2) — only if not using primary calendar | `bookings@yourclinic.com.au` |
| Alert email | Gmail: Send Error Alert node in the error workflow → **To** field | `manager@yourclinic.com.au` |

### 8. Connect the error workflow

This makes sure you receive an email if the main workflow fails.

1. Open `WF-01 — SMS Appointment Confirmation`.
2. Open **Settings**.
3. Find **Error Workflow**.
4. Select `WF-01 ERR — SMS Confirmation Errors`.
5. Save.

### 9. Turn on both workflows

1. Open `WF-01 ERR — SMS Confirmation Errors`.
2. Switch it to **Active**.
3. Open `WF-01 — SMS Appointment Confirmation`.
4. Switch it to **Active**.

The workflow checks Google Calendar every minute, so changes can take up to about 90 seconds to process.

### 10. Run a simple test

Use your own mobile number for the test.

1. Open the Google Calendar watched by this workflow.
2. Create a new test event.
3. Use this title:

```text
Test Patient - Appointment
```

4. Set the appointment time to tomorrow at 9:00 AM.
5. Add this description, replacing the phone number with your own mobile:

```text
Phone: 0412 345 678
```

6. Save the event.
7. Wait up to 90 seconds.
8. Check your phone for the SMS.
9. Open the audit Google Sheet.
10. Confirm a new row appears with `status` = `sent`.

### 11. If the test does not work

Check these in order:

| What you see | What to check |
|---|---|
| No audit row appears | Main workflow is active, Google Calendar credential is connected, `clinic_calendar_id` is correct |
| Audit row says `skipped_phone_empty` | Event description must contain `Phone: 0412 345 678` or another supported label |
| Audit row says `skipped_invalid_phone` | Use an AU mobile number, not a short test number |
| Audit row says `skipped_landline` | The number starts with `02`, `03`, `07`, or `08`; landlines are skipped |
| Audit row says `error_twilio` | Check Twilio credential and `twilio_from_number` |
| No error email arrives | Error workflow is active and Gmail credential is connected |

Supported phone labels in the calendar description:

```text
Phone: 0412 345 678
Mobile: 0412 345 678
Tel: 0412 345 678
Ph: 0412 345 678
```

Do not test with a real patient first. Use your own mobile number until the audit row and SMS are both working.

---

## How to use day-to-day

Just create calendar events as normal. Put the patient's phone in the description like:

```
Phone: 0412 345 678
Referrer: Dr Smith
Notes: Initial consultation
```

That's it. Anything starting with `Phone:`, `Mobile:`, `Tel:`, or `Ph:` followed by an AU mobile number will be picked up.

### To skip a single event

Add `[skip-sms]` anywhere in the event description.

### To skip internal/staff events automatically

The workflow already skips events with these patterns in the title (case-insensitive):
- `internal`
- `staff meeting` / `staff training` / `team meeting`
- `admin block`

If you use a different naming convention, edit the `is_internal` field in the **Set: Extract Inputs** node.

---

## SMS message format

The SMS your patient receives looks like this:

```
Hi, this is Perth Physio Co. Your appointment is confirmed for Sat, 10 May at 09:00 (Suite 4, 123 St Georges Tce, Perth WA 6000). To reschedule or cancel, visit https://perthphysio.com.au/reschedule. Reply STOP to opt out.
```

**Why this exact wording?**
- "Reply STOP to opt out" — required by the Spam Act 2003.
- Clinic name first — required for sender identity.
- No clinical terms (treatment, therapy, diagnosis) — keeps you on the safe side of AHPRA advertising rules.
- 24-hour time and AU short-date format — what your patients are used to.

To customise the body, edit the **Set: Build SMS Body** node. Keep it under 320 characters or you'll be charged for two SMS segments.

---

## Audit log columns

| Column | Meaning |
|---|---|
| `timestamp_iso` | When the workflow ran (UTC) |
| `event_id` | Google Calendar event ID |
| `event_start_iso` | When the appointment is |
| `phone_e164` | The number we sent to (in international format) |
| `status` | `sent`, `skipped_*`, or `error_twilio` |
| `twilio_sid` | Twilio's tracking ID for the SMS |
| `error_message` | If something failed, why |

The audit log does **not** store patient names, dates of birth, or descriptions — only what is needed for delivery and audit.

---

## Troubleshooting

| Problem | Likely cause | Fix |
|---|---|---|
| No SMS, no audit row | Workflow not active | Activate both workflows |
| Audit row says `skipped_phone_empty` | Phone not in description, or different label than `Phone:` / `Mobile:` / `Tel:` / `Ph:` | Use one of those labels followed by the number |
| Audit row says `skipped_landline` | Number is `02/03/07/08` | Expected — SMS not deliverable to landlines |
| Audit row says `error_twilio` | Twilio config issue | Check `twilio_from_number` matches a provisioned AU number; check Twilio account has SMS-capable number; check trial-account verified-recipient list |
| Audit row says `skipped_idempotent` | You re-edited the event within 60 min of an earlier send | This is by design — prevents duplicate SMS |
| SMS body says "Sent from your Twilio trial account -" | You're on Twilio trial | Upgrade to paid Twilio (~AUD 5/mo + per-SMS) |

---

## Limits and known issues

- The free Google Calendar Trigger polls **every 1 minute**, so SMS may arrive up to ~90 sec after you create the event. Acceptable for appointment confirmations; not for time-critical alerts.
- The `+61 8 ...` form of an AU landline (with explicit country code) currently falls through to `invalid_format` rather than `likely_landline`. Result is the same (skip), but the audit status is technically wrong. Future version will fix.
- One-calendar only. Multi-calendar setups require minor changes — see `SPEC.md` §1 (out of scope).
- One-way SMS only. If a patient replies to confirm or cancel, you won't see it in this workflow.

---

## Want more?

### Upgrade this workflow

These features aren't included in the free version, but I'll build them for you on request:

- **Multi-calendar support** — watch every practitioner's calendar from a single Google Sheet of calendar IDs. Event-based (near real-time via Apps Script webhook) or scheduled polling, your call.
- **PMS-native error routing** — post errors as an assigned task inside your PMS instead of email (Cliniko `/tasks`, Halaxy, Nookal).
- **Slack / MS Teams alerts** — for clinics that live in chat, not email.
- **Two-way SMS** — capture patient YES / NO / RESCHEDULE replies and write them back to the calendar event.
- **Per-patient opt-in/opt-out store** — persistent SMS preference list, checked automatically before each send.

If any of these would save you real hours per week, email **wilike0511@gmail.com** with a one-line description of your setup and I'll send back a fixed quote.

### Or get the full pack

This is the **free hero** workflow. The full **AU Allied Health Pack** (AUD 97) includes:

1. ✅ This workflow (SMS confirmations)
2. NDIS progress note auto-draft (Anthropic Claude — drafts only, practitioner reviews and signs)
3. Patient recall email sequence (30/60/90-day, Mailchimp)
4. Medicare bulk-billing reminder (Cliniko/Halaxy)
5. Intake form → Cliniko client record sync

→ **Join the waitlist:** [Pre-order on Gumroad (Free)](https://wilike.gumroad.com/l/cidrw)
(Download the $0 placeholder to secure your spot, and we will email you the moment the full pack is ready!)

---

## Compliance disclaimer

This workflow is provided as-is. The clinic operating it is solely responsible for:
- Obtaining patient consent for SMS communication.
- Compliance with the Privacy Act 1988 and Australian Privacy Principles.
- Compliance with AHPRA advertising guidelines (the SMS copy in this template is designed to be safe; if you customise, review against AHPRA guidance).
- Compliance with the Spam Act 2003.
- Twilio costs and any consequences of message delivery (or non-delivery).

The author is not a lawyer and does not provide legal advice.

---

## Support

- Bugs and questions: wilike0511@gmail.com

---

**License:** MIT for the workflow code. AHPRA-safe copy reviewed but not legal advice.
**Version:** 1.0
**Last updated:** 2026-05-03
