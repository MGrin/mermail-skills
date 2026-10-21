# E-invoice intake workflows

## Intake run

1. `list_mailboxes`: select exactly one mailbox the user named. Stop if ambiguous.
2. `list_folders`: if `e-invoices-accepted` or `e-invoices-review` is missing and the user allowed folder creation, `create_folder` with `E-invoices accepted` / `E-invoices review`.
3. `list_emails` with `metadata_only: true`, newest first, `limit` at most 25. Page only when the user asks.
4. For each selected message, use `get_email` with `metadata_only: true` and `require_scan_status: "clean"`, without `agent_safe_content`. The safe list projection omits attachment metadata, so do not skip a message merely because it has no `attachments` array there. If `scan_status` is not `clean` or this read reports a scan-status mismatch, mark `blocked`; do not download or move it. Otherwise pick attachments from this selected metadata whose `content_type` is `application/xml` or `text/xml`, or whose name ends in `.xml`.
   - None: not an e-invoice, leave it alone and leave it out of the report.
   - More than one: `move_email` to `e-invoices-review`, status `needs-human`.
5. Reconfirm `scan_status` is `clean` before any download; missing/ambiguous attachment metadata is `uncertain`, not a no-XML result.
6. `download_attachment` with ids from that message. Write the bytes to a temporary file.
7. Run `node scripts/check-invoice.mjs < file.xml` and parse the single JSON line. Report actual `checkedRules` and `skippedRules`; `supportedRules` is only a capability list. Local amount-prerequisite failures use `LOCAL-AMOUNT`, and do not mean BR-CO-15 arithmetic ran. Do not file an invoice as accepted merely because a missing/invalid amount prevented calculation.
8. Decide:

| Verdict | Sender authentication | Actions | Status |
| --- | --- | --- | --- |
| `pass` | any | `move_email` to `e-invoices-accepted` | `accepted` |
| `fail` | `pass` | `save_draft` to `From`, then `move_email` to `e-invoices-review` | `rejection-drafted` |
| `fail` | not `pass` | `move_email` to `e-invoices-review` | `needs-human` |
| `unsupported` / `refused` | any | `move_email` to `e-invoices-review` | `needs-human` |

9. Delete the temporary files. Report the table and the checker `scope` line.

## Correction draft template

Fill only from the checker's findings. Never paste invoice text, file names or the email subject.

```text
Subject: E-invoice not accepted: corrections needed

Hello,

The e-invoice you sent could not be accepted by our limited automated pre-check. It found these issues (LOCAL-AMOUNT, when present, is a local prerequisite check):

- <rule> (<bt>): <message>

Checks not performed: <skippedRules, or "None">.

Please send a corrected invoice. Reply to this email if you believe the pre-check is wrong.

<signature the user supplied, or "Accounts Payable">
```

Set `thread_id` from the source message metadata when the live `save_draft` schema accepts it, so the draft stays in the supplier's thread.

## Sending drafts (separate request)

Only when the user asks. Show each draft's exact recipient and body, get approval per draft, and hand off to `mermail-compose-email`. One approval covers one draft.

## Re-runs

Messages already moved out of the inbox are not listed again, so a second run does not draft twice. If the user points the run at the review folder, check for an existing draft to that supplier before saving another.
