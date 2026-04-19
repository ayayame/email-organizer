---
name: cleanup-labels
description: Use when auditing, cleaning up, or removing stale email organizer labels from Gmail
---

# Cleanup Labels

Audits Gmail labels managed by the email organizer plugin. Finds stale labels, resolves conflicts, and cleans up with user confirmation.

## Prerequisites

Config file `~/.email-organizer.json` must exist. If it doesn't, tell the user to run `setup-categories` first and stop.

## Process

### Step 1: Load Config and List Labels

1. Read `~/.email-organizer.json` to get the `labelPrefix`
2. Use `mcp__claude_ai_Gmail__list_labels` to get all Gmail labels
3. Filter to labels starting with `{labelPrefix}/`

### Step 2: Audit Each Label

For each organizer label:

1. Use `mcp__claude_ai_Gmail__search_threads` with `label:{label-name}` to find emails with that label
2. Record the count of emails per label
3. Flag labels with **0 emails** as stale
4. Check for emails that have **multiple organizer labels** — these are conflicts

### Step 3: Report Findings

Present the audit results:

```
=== Label Audit ===

Active labels:
  Organizer/Action Required:  12 emails
  Organizer/FYI:              28 emails
  Organizer/Newsletters:      45 emails
  Organizer/Personal:          8 emails
  Organizer/Notifications:    19 emails

Stale labels (0 emails):
  Organizer/Old Project
  Organizer/Temp Category

Conflicts (multiple organizer labels):
  3 emails have more than one organizer label
```

### Step 4: Clean Up (With Confirmation)

Ask the user before taking any action using `AskUserQuestion`:

**For stale labels:**
- Offer to delete them from Gmail
- Also remove from `customCategories` in config if present

**For conflicts:**
- Show each conflicting email with its multiple labels
- Ask which label to keep for each, or let AI pick the best fit
- Remove the extra labels via `mcp__claude_ai_Gmail__unlabel_message`

**For labels not in config:**
- If organizer labels exist in Gmail that aren't in the config (orphaned from a previous setup), flag them
- Offer to add them to config or remove them

### Step 5: Update Config

After cleanup:
- Remove deleted categories from `customCategories`
- Save updated `~/.email-organizer.json`

### Step 6: Output Summary

```
=== Cleanup Complete ===

Removed: 2 stale labels
Resolved: 3 email conflicts
Orphaned labels added to config: 0
```

## Key Rules

- Never delete labels without explicit user confirmation
- Never remove non-organizer labels — only touch labels with the configured prefix
- Always update the config file to stay in sync with Gmail labels
