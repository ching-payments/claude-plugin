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

In Claude Code, run:

```
/plugin marketplace add ching-payments/claude-plugin
/plugin install ching-api-integration@ching-payments
```

The first command adds this repo as a marketplace; the second installs the skill. Once installed, Claude automatically picks up the skill whenever it sees CHING-related tasks.

### Updating

When a new version ships:

```
/plugin marketplace update ching-payments
/plugin install ching-api-integration@ching-payments
```

Or enable auto-update from the `/plugin` UI under **Marketplaces**.

### Local development

If you're hacking on the plugin locally, point your Claude Code settings at the checkout:

```json
{
  "plugins": ["/absolute/path/to/ching-claude-plugin"]
}
```

## When Claude uses this skill

Claude loads the skill automatically when a conversation involves CHING, Israeli payments, ILS billing, or integrating a payment API for subscriptions / checkout / saved cards / webhooks.

## Version

`0.1.0` - initial public release.
