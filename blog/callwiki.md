---
layout: default
title: "callwiki: A Tool to Help Founders Stay Closer to Customers"
date: 2026-04-12
description: "We all talk to customers. The hard part is seeing the patterns across all those conversations. I built callwiki to help with that — open source, runs locally, and I'd love your help making it better."
tags: [ai, founders, customer-discovery, tools, claude]
categories: [product, ai]
---

# callwiki: A Tool to Help Founders Stay Closer to Customers

*We all do the calls. The patterns shouldn't get lost.*

---

## How This Started

Karpathy recently [shared how he uses LLMs to build personal knowledge wikis](https://x.com/karpathy/status/2039805659525644595) — index raw source documents, have an LLM compile them into a structured markdown wiki with backlinks and categories, then do Q&A against the whole thing in Obsidian. The tweet went viral. His closing line stuck with me:

> *"I think there is room here for an incredible new product instead of a hacky collection of scripts."*

I kept thinking about this in the context of what I see working with founders every day. Not research papers. Not code repos. The most valuable raw data most founders sit on is **customer calls** — and we're all losing the patterns buried inside them.

Whether you're pre-seed and talking to your first 10 users or Series B trying to stay close to customers as the team scales, the raw material is the same: conversations with the people who use (or should use) what we're building.

---

## The Thing That's Been Bugging Me

We all love customer calls. We should. "Talk to your customers" is the most repeated advice in startups for a reason — it works.

But here's what I kept running into, both for myself and watching other founders:

You have 30, 50, maybe 100 calls sitting in a Google Drive folder or a Zoom directory. Your meeting tool gave you a summary for each one. Maybe sentiment scores. Maybe action items. That's fine for the individual call.

But try answering this: **What are the top 3 pain points across all your enterprise prospects? Which feature request keeps showing up in calls with churning customers? What patterns are you not seeing because the calls are scattered across folders?**

I couldn't. Not without re-reading every transcript. And I know I'm not the only one.

The individual summaries are fine. What's missing is the **pattern layer** — the cross-call intelligence that should inform what we build next. That knowledge is trapped across dozens of files, and none of the meeting tools surface it.

---

## So I Built callwiki

[callwiki](https://github.com/akshayeKS/callwiki) takes Karpathy's knowledge wiki idea and applies it to customer calls.

Same concept: raw data goes in, an LLM compiles it into a structured markdown wiki, you browse and query it in Obsidian. But the raw data is your customer conversations, and instead of general categories, it surfaces what we actually need as founders — feature requests, churn risks, competitor mentions, customer profiles.

```bash
# First run — auto-detects your transcript folders
callwiki

# Process everything and build the wiki
callwiki sync

# Now explore
callwiki features        # ranked feature requests across all calls
callwiki risks           # churn signals you might have missed
callwiki customer "acme" # everything you know about Acme, in one place

# Or just ask
callwiki ask "what are enterprise customers struggling with?"
```

It runs on your machine, uses an Anthropic API key (~$0.02 per transcript), and nothing leaves your laptop. Open source — [the code is right here](https://github.com/akshayeKS/callwiki).

---

## Why I Think This Helps Us

### We're doing the calls. We're losing the compound knowledge.

One call tells you what one customer thinks. Fifty calls tell you where the market is pulling — *if* you can see across all of them at once. Most of us can't, because the insights are scattered.

callwiki doesn't use hardcoded templates. It reads your actual calls and discovers what themes exist in *your* data. If your conversations are dominated by pricing concerns and integration blockers, the wiki reflects that. If competitors keep coming up in enterprise calls but not SMB, you'll see it. The structure adapts to what your customers are actually telling you.

### Meeting tools solve a different problem

Zoom AI, Fireflies, Otter — they're great at summarizing individual calls. Transcription is solved. But the hard question isn't "what happened in this call?" It's: *What did I learn across the last 3 months of calls?*

That's the question that should shape our next quarter. And right now, answering it means re-reading everything or relying on gut feel.

### It works with what you already have

callwiki reads `.txt`, `.md`, `.vtt` (Zoom, Google Meet), and `.srt` files. It auto-detects common transcript locations — Google Drive, Zoom's local recordings. Export from Fireflies or Otter and drop them in.

No new workflow. No recording bot joining your calls. Just point it at the folder you already have.

---

## What You Get

After `callwiki sync`, your wiki lives at `~/.callwiki/wiki/`:

```
wiki/
├── _index.md         # Master index — start here
├── calls/            # Per-call summaries with key quotes
├── customers/        # Per-customer profiles (history, pain points, sentiment)
├── themes/           # Discovered themes (e.g., "Feature Requests", "Churn Risks")
└── output/           # Saved answers from callwiki ask
```

Everything is markdown with `[[wikilinks]]`. Open it as an Obsidian vault and you've got a navigable knowledge base of every customer conversation.

Only new or changed transcripts are processed on each sync, so it stays fast as the call library grows.

---

## On Privacy

Customer calls are sensitive — pricing discussions, internal struggles, competitive intel. I didn't want any of that leaving my machine, and I'm guessing you feel the same.

- Runs entirely locally. No servers, no telemetry, no accounts.
- Transcripts are sent to the Anthropic API for processing. [Anthropic does not train on API inputs.](https://www.anthropic.com/policies/privacy)
- Wiki, config, extracted data — all stay on your machine.
- Open source — [read every line](https://github.com/akshayeKS/callwiki).

---

## Try It

```bash
# Install
pip install git+ssh://git@github.com/akshayeKS/callwiki.git

# Setup — prompts for your Anthropic API key and transcript folder
callwiki

# Build your wiki
callwiki sync

# Explore
callwiki features
callwiki risks
callwiki open   # opens in Finder/Obsidian
```

You need Python 3.11+ and an [Anthropic API key](https://console.anthropic.com/keys).

---

## Let's Build This Together

I built this because I kept running into the same gap — founders doing the right thing (talking to customers) but not getting the compounding value out of those conversations. The patterns are there. They just need to be surfaced.

This is early. It works, but I want it shaped by the people who need it — fellow founders who are in these calls every week.

If this resonates, give it a spin. Tell me what's missing, what's broken, what would actually make it useful for you. I'd rather build this *with* other founders than in isolation.

DM me on [X (@akshayeKS)](https://x.com/akshayeKS) — happy to chat, hear your feedback, or just nerd out about staying closer to customers.

---

*We should all be talking to customers. Let's make sure we remember what they told us.*

---

[← Back to Blog](/blog) ・ [Home](/) ・ [Contact](/contact)
