---
name: setup-categories
description: Use when setting up the email organizer for the first time, reconfiguring email categories, or when the user wants to customize how their inbox is organized
---

# Setup Categories

Interactive category configuration for the email organizer. Establishes the categories AI will use to classify emails, tailored to each user's inbox.

## Process

### Step 1: Check for Existing Config

Read `~/.email-organizer.json`. If it exists, show the user their current categories and ask if they want to reconfigure or add to them. If it doesn't exist, proceed with fresh setup.

### Step 2: Choose Mode

Ask the user which setup mode they prefer using `AskUserQuestion`:

- **AI-first**: AI scans the inbox and proposes all categories from scratch. Best for users who want a fresh, data-driven organization.
- **User seeds + AI expands**: User provides their own starter categories, then AI scans the inbox and suggests additions. Best for users who already know some categories they want.

### Step 3A: AI-First Mode

1. Use `mcp__claude_ai_Gmail__search_threads` to fetch the 100 most recent email threads
2. For each thread, use `mcp__claude_ai_Gmail__get_thread` to read subject, sender, and snippet
3. Analyze patterns across all emails:
   - Group by sender domain (e.g., many emails from github.com = "Developer Notifications")
   - Group by content type (e.g., receipts, newsletters, meeting invites)
   - Group by urgency signals (e.g., "action required", "please respond", "deadline")
4. Start with the **default categories** as a baseline:
   - Action Required
   - FYI / Informational
   - Newsletters / Marketing
   - Personal
   - Notifications
5. Propose additional categories based on discovered patterns
6. Present all categories to user via `AskUserQuestion` for approval, letting them enable/disable defaults and accept/reject suggestions

### Step 3B: User Seeds + AI Expands Mode

1. Ask the user for their starter categories using `AskUserQuestion` (free text via "Other")
2. Use `mcp__claude_ai_Gmail__search_threads` to fetch recent emails
3. Analyze inbox for patterns NOT covered by the user's categories or the defaults
4. Suggest additional categories to fill gaps
5. Present suggestions to user for approval

### Step 4: Configure Label Prefix

Ask the user what label prefix they want (default: `Organizer`). All labels will be created as `{prefix}/{Category Name}` in Gmail.

### Step 5: Save Config

Write the validated configuration to `~/.email-organizer.json`:

```json
{
  "labelPrefix": "Organizer",
  "mode": "ai-first",
  "categories": [
    {"name": "Action Required", "description": "Emails needing a response or action"},
    {"name": "FYI", "description": "Informational updates, no action needed"},
    {"name": "Newsletters", "description": "Subscriptions, promotions, marketing"},
    {"name": "Personal", "description": "Friends, family, non-work"},
    {"name": "Notifications", "description": "Automated system notifications"}
  ],
  "customCategories": [
    {"name": "Receipts & Billing", "description": "Purchase confirmations, invoices, payment notifications"}
  ],
  "lastRunTimestamp": null
}
```

### Step 6: Create Gmail Labels

Use `mcp__claude_ai_Gmail__create_label` to create a Gmail label for each category:
- Format: `{labelPrefix}/{Category Name}` (e.g., `Organizer/Action Required`)
- Skip labels that already exist

### Step 7: Confirm

Output a summary of the setup:
- Mode chosen
- Categories configured (defaults + custom)
- Labels created in Gmail
- Next step: tell the user to run `organize-inbox` to start categorizing

## Key Rules

- Always include the 5 default categories unless the user explicitly disables one
- Never create duplicate labels — check existing labels first via `mcp__claude_ai_Gmail__list_labels`
- Category descriptions should be concise (under 15 words) — they guide AI classification
- Save config before creating labels, so config is valid even if label creation partially fails
