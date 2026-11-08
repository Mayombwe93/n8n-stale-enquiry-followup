# Follow-up Reminder Automation (n8n)

An n8n workflow that runs daily, finds sales enquiries with no update for 48+ hours, and sends the owner one Telegram summary. It then records the reminder in Google Sheets so the same enquiry isn't alerted again until someone updates it.

## How it works

1. A schedule trigger runs every day at 09:00 (Africa/Kampala).
2. All rows are read from the **Enquiries** sheet in Google Sheets.
3. Each row is checked for:
   - Age in hours since `LastUpdated`
   - Whether the status is `Closed`
   - Whether a reminder was already sent in the current stale cycle
4. Eligible enquiries are combined into one Telegram message.
5. After the message is sent successfully, `LastReminderAt`, `ReminderSent`, and `ReminderCount` are updated for each enquiry. `LastUpdated` is never changed by the workflow.
6. If a reminder flag is `TRUE` but the enquiry has been updated since the last reminder, the flag is reset to `FALSE`, so a new stale cycle can start.

If nothing is eligible, no Telegram message is sent.

## Files

| File | Purpose |
| --- | --- |
| `workflows/follow-up-reminder-v2.json` | Main workflow |
| `workflows/follow-up-reminder-v2-error-handler.json` | Error workflow that sends a Telegram alert when the main workflow fails |

## Google Sheet setup

Create a Google Sheet with a tab named **Enquiries** and these headers in row 1:

| ID | Name | Email | Status | LastUpdated | LastReminderAt | ReminderSent | ReminderCount |
| --- | --- | --- | --- | --- | --- | --- | --- |

- `LastUpdated` should be a date/time or ISO text such as `2026-11-08T09:30:00+03:00`.
- For new enquiries, leave `LastReminderAt` blank, set `ReminderSent` to `FALSE`, and set `ReminderCount` to `0`.

## Setup

1. In n8n, create a Telegram bot with BotFather and add the token as a **Telegram API** credential.
2. Create a **Google Sheets OAuth2 API** credential.
3. Import `follow-up-reminder-v2-error-handler.json` first. Select the Telegram credential, replace `YOUR_TELEGRAM_CHAT_ID`, and publish it.
4. Import `follow-up-reminder-v2.json`.
5. Replace `YOUR_SPREADSHEET_ID` with your sheet's ID in the three Google Sheets nodes.
6. Replace `YOUR_TELEGRAM_CHAT_ID` in the two Telegram nodes.
7. Select your credentials on every node that uses one.
8. In the workflow settings, set **Error Workflow** to the error handler.
9. Test with **Execute workflow**, then publish.

## Testing

I tested the workflow with six sample rows covering these cases:

- Stale enquiry, never reminded: alerted and recorded
- Enquiry just under 48 hours: not alerted
- Closed enquiry: skipped
- Enquiry already reminded in the current cycle: skipped
- Enquiry updated after its last reminder: flag reset to `FALSE`
- Nothing eligible: no Telegram message sent

Re-running after a reminder was sent did not send a duplicate.

## Tools

- n8n
- Google Sheets
- Telegram Bot API

## Notes

- The workflow reads dates using Google Sheets' serial-date format and ISO text. Ambiguous formats such as `08/11/2026` can be read as the wrong day or month.
- If the Telegram alert fails, no reminder is recorded, so the enquiry will be included again the next day.
