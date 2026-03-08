# ARO Email Intelligence Dashboard

## Overview

Static dashboard hosted on Netlify. The OpenClaw agent reads its Gmail inbox (which receives forwarded emails from Tiffany's Outlook), analyzes and categorizes them, and pushes a structured summary to `data/email_summary.json`. Netlify auto-deploys.

## How the Agent Updates This Dashboard

### Daily Process (Morning, after inbox scan)

1. Read all new emails from the agent's Gmail inbox
2. For each email, determine:
   - **Category** (NIGO, Advisor Follow-Up, New Business, Commissions, Policy Issued, Illustration Request, Rate Updates, System/Notifications, etc.)
   - **Priority** (urgent, high, normal, low)
   - **Summary** (1-2 sentence plain language summary of what the email says)
   - **Action Required** (specific, actionable next step, or null if informational only)
3. Build the JSON file per the schema below
4. Generate the weekly summary (every day, rolling 7-day window)
5. Commit and push:
   ```bash
   cd ~/email-dashboard
   git add data/email_summary.json
   git commit -m "Email summary $(date +%Y-%m-%d_%H%M)"
   git push origin main
   ```

### Priority Classification Rules

| Priority | Criteria |
|----------|----------|
| **urgent** | NIGO with deadline, client complaint, advisor 3rd+ follow-up, compliance issue, account locked |
| **high** | New business confirmation needing HubSpot update, commission statement, policy issued needing case closure, advisor requesting time-sensitive illustration |
| **normal** | Standard policy updates, first/second advisor follow-up, routine carrier correspondence, illustration requests without urgency |
| **low** | Marketing emails, rate sheets (no immediate action), system notifications, newsletters, informational only |

### Category Classification Rules

Categorize each email into one of these standard categories:

- **NIGO / Pending Requirements** - Carrier says submission is Not In Good Order, missing documents
- **Advisor Follow-Up** - Advisor asking for status, updates, or requesting action
- **New Business / Confirmations** - Carrier confirming receipt of application or assignment of policy number
- **Policy Issued / Paid** - Carrier confirming policy is issued, exchange complete, case is paid
- **Commissions / Financials** - Commission statements, payment confirmations, financial reports
- **Policy Updates / Valuations** - Surrender value updates, contract changes, beneficiary updates
- **Illustration Requests** - Advisor requesting product comparisons or illustrations
- **Rate Updates / Product Info** - New rate sheets, product changes, carrier announcements
- **System / Notifications** - HubSpot notifications, IT alerts, automated system messages
- **Other** - Anything that doesn't fit the above

### Action Required Guidelines

For each email with an action item, write the action as if you're telling Tiffany exactly what to do:

**Good:** "Pull the replacement form from advisor Nathanson and resubmit to MassMutual within 5 business days."

**Bad:** "Follow up on this."

Be specific: name the person to contact, the document needed, the system to update, and any deadline.

## Data Schema

### `data/email_summary.json`

```json
{
  "summary": {
    "timestamp": "ISO 8601 (when this summary was generated)",
    "date": "YYYY-MM-DD",
    "total_today": 18,
    "total_week": 72,
    "from_carriers": 9,
    "from_advisors": 5,
    "informational": 4,
    "action_items_count": 7
  },
  "action_items": [
    {
      "priority": "urgent | high | normal | low",
      "subject": "email subject line",
      "from": "full sender email",
      "from_short": "friendly sender name",
      "received": "ISO 8601",
      "summary": "1-2 sentence plain language summary",
      "action_required": "specific actionable next step, or null",
      "category": "one of the standard categories",
      "email_id": "unique identifier"
    }
  ],
  "emails": [
    {
      "email_id": "unique identifier",
      "from": "sender@example.com",
      "from_short": "Friendly Name",
      "subject": "email subject",
      "received": "ISO 8601",
      "category": "standard category",
      "priority": "urgent | high | normal | low",
      "has_attachment": true/false
    }
  ],
  "categories": {
    "Category Name": count,
    "Another Category": count
  },
  "weekly_summary": "HTML string with <strong> tags for emphasis. Rolling 7-day summary highlighting key themes, patterns, outstanding items, and top priorities for the coming week."
}
```

### Key Differences from ARO Dashboard

- `action_items` array contains ONLY emails that require Tiffany to do something. Informational emails appear in `emails` but not here.
- `action_items` are sorted by priority (urgent first, then high, normal, low) and within priority by recency.
- `emails` array contains ALL emails (including informational), sorted by received time (newest first).
- `weekly_summary` is a narrative HTML string, not structured data. It should read like a briefing memo.

## Gmail Reading

The agent reads its own Gmail inbox using the Gmail skill or browser-based login. Emails arrive in this inbox via an Outlook rule on Tiffany's account that auto-forwards incoming mail to the agent's Gmail address.

### What Gets Forwarded

Tiffany's Outlook rule should forward emails from:
- Carrier domains (massmutualascend.com, athene.com, sagicor.com, etc.)
- Known advisor email addresses
- HubSpot notifications (hubspot.com)

The rule should NOT forward:
- Spam
- Internal Hub International HR/IT emails
- Personal emails

### Processing Cadence

- **Morning scan:** Process all new emails, generate daily summary, push to dashboard
- **Midday scan (optional):** If heartbeat detects new high-priority emails, update the summary
- **No evening/overnight processing** unless explicitly requested

## Netlify Setup

Same process as the ARO dashboard:
1. Separate GitHub repo (e.g., `ka1ro-ai/aro-email-dashboard`)
2. Connect to Netlify
3. Publish directory: `/` (root), no build command
4. Add same security headers (noindex, nofollow)

## Security

This dashboard displays email subjects, sender addresses, and email content summaries. Same security considerations as the ARO dashboard: password-protect or keep URL private.
