---
name: organize-inbox
description: Use when organizing Gmail inbox, categorizing emails, applying email labels, or when the user wants AI to sort and triage their email
---

# Organize Inbox

Scans Gmail, categorizes each email using AI, applies labels, cleans up stale labels, and outputs a summary with new category suggestions.

## Prerequisites

Config file `~/.email-organizer.json` must exist. If it doesn't, tell the user to run `setup-categories` first and stop.

## Process

### Step 1: Load Config

Read `~/.email-organizer.json`. Extract:
- `labelPrefix` — the Gmail label prefix
- `categories` + `customCategories` — merged into one list for classification
- `lastRunTimestamp` — for incremental processing

### Step 2: Fetch Emails

Use `mcp__claude_ai_Gmail__search_threads` to fetch emails:
- **First run** (`lastRunTimestamp` is null): fetch the 50 most recent threads
- **Subsequent runs**: fetch threads with `after:{lastRunTimestamp}` to only process new emails

### Step 3: Categorize Each Email

For each thread:

1. Use `mcp__claude_ai_Gmail__get_thread` to read the full thread (sender, subject, body, existing labels)
2. Analyze the email against all configured categories
3. Pick the **single best-fit category** based on:
   - Sender domain and address patterns
   - Subject line content and urgency signals
   - Body content and intent
   - Category descriptions from config
4. If no category fits well (confidence is low), mark as **"Uncategorized"** and flag for new category suggestion

**Classification guidelines:**
- "Action Required" takes priority when emails contain direct requests, questions to answer, deadlines, or tasks
- "Personal" applies to non-automated emails from individuals outside a work context
- "Newsletters" applies to bulk/marketing emails, even if from known senders
- "Notifications" applies to automated messages from services (GitHub, CI, shipping, etc.)
- "FYI" is the catch-all for emails worth reading but not acting on
- Custom categories take precedence over defaults when they are a better fit

### Step 4: Apply Labels

For each categorized email:

1. Determine the target label: `{labelPrefix}/{Category Name}`
2. Check if the email already has an organizer label (any label starting with `{labelPrefix}/`)
3. If it has a **different** organizer label, remove the old one via `mcp__claude_ai_Gmail__unlabel_message`
4. If it already has the **correct** label, skip it
5. Apply the new label via `mcp__claude_ai_Gmail__label_message`
6. If the label doesn't exist in Gmail yet, create it first via `mcp__claude_ai_Gmail__create_label`

### Step 5: Detect New Category Suggestions

After processing all emails, check for patterns among "Uncategorized" emails and any clusters:

- If 3+ uncategorized emails share a common pattern (same sender domain, similar subject structure, or similar content type), suggest a new category
- Present suggestions with example emails so the user understands what would go there

Format suggestions as:
```
Suggested new category: "Shipping & Tracking"
Reason: Found 5 emails about package deliveries and tracking updates
Examples: "Your package has shipped" from amazon.com, "Delivery update" from ups.com
```

### Step 6: Update Config

Update `~/.email-organizer.json`:
- Set `lastRunTimestamp` to current ISO timestamp
- If user accepts any new category suggestions, add them to `customCategories`

### Step 7: Output Summary

Print a clear summary:

```
=== Inbox Organization Summary ===

Processed: 47 emails

By category:
  Action Required:  8 emails
  FYI:             12 emails
  Newsletters:     15 emails
  Personal:         3 emails
  Notifications:    6 emails
  Uncategorized:    3 emails

Recategorized: 2 emails
  - "Re: Project update" moved from Newsletters -> Action Required
  - "Invoice #4521" moved from Notifications -> Receipts & Billing

New category suggestions:
  - "Shipping & Tracking" (5 emails matched this pattern)

Next run will process emails after: 2026-04-18T14:30:00Z
```

## Key Rules

- Each email gets exactly ONE organizer label — never multiple
- Always remove old organizer labels before applying new ones when recategorizing
- Never modify non-organizer labels — only touch labels with the configured prefix
- Process emails in batches if there are many (50 at a time) to avoid overwhelming the API
- If config file is missing, stop and direct user to `setup-categories` — never create a default config silently
