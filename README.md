# CHING Claude Plugin

A Claude Code plugin that teaches Claude how to integrate the [CHING](https://ching.co.il) payments API into a SaaS product. Ship this to your customers alongside their CHING API keys and Claude will know how to wire subscriptions, checkout, saved cards, webhooks, and the billing portal correctly the first time.

## What's inside

- `skills/ching-api-integration/SKILL.md` - Core integration guide: two golden rules for card handling, canonical flows, webhook verification, common mistakes.
- `skills/ching-api-integration/api-reference.md` - Complete endpoint reference (customers, products, prices, checkout/setup sessions, subscriptions, charges, refunds, webhooks, billing portal).

## The two golden rules

Any CHING integration must follow these:

1. **To subscribe a customer to a plan -> always open a CHING checkout session.** Do not build a card form.
2. **To save a card without charging -> always open a CHING setup session.** Do not tokenize cards yourself.

The skill expands on both rules with complete request/response examples and explains why (PCI, 3DS, installments, tax, receipts are all handled by the hosted session).

## Install

### Option 1: Persistent install via settings

Add the absolute path to `~/.claude/settings.json`:

```json
{
  "plugins": ["/absolute/path/to/ching-claude-plugin"]
}
```

Reload with `/reload-plugins`. The skill is then available as `ching-api-integration:ching-api-integration`.

### Option 2: One-off local run

```bash
claude --plugin-dir /absolute/path/to/ching-claude-plugin
```

## When Claude uses this skill

Claude loads the skill automatically when a conversation involves CHING, Israeli payments, ILS billing, or integrating a payment API for subscriptions / checkout / saved cards / webhooks.

## Version

`0.1.0` - initial public release.
