---
layout: post
title: "How Claude Code Manages Context: A Multi-Tier Compaction System"
date: 2026-03-31
description: "The recent axios compromise gave us access to Claude Code's source. After looking through the core loop, here's what I think is the most interesting design choice: a layered context compaction system that goes beyond what any other coding agent does."
---

The recent axios compromise gave us access to Claude Code's source. After looking through the core loop, here's what I think is the most interesting design choice: a layered context compaction system that goes beyond what any other coding agent does.

Most coding agents hit context limits and either truncate or self-summarize. Claude Code does neither as its first move. It runs multiple compaction strategies, escalating from cheapest to most expensive. The system has many moving parts, but the core design breaks down into 3 layers.

## Layer 1: Microcompact

Cost: 0 — no LLM calls

Most of the context bloat in a coding session is tool output: file reads, grep results, shell output. Critical when fresh, almost worthless a few turns later. Microcompact keeps the N most recent tool results and replaces everything older with a stub.

When the user has been away long enough that the prompt cache is cold, microcompact clears more aggressively. A cache-aware variant goes further. Instead of modifying the local messages which would invalidate the prompt cache, it tells the API to remove specific tool results server-side, reclaiming context without the cost of rebuilding the cache.

## Layer 2: Session Memory Compaction

*This is the novel idea.*

Cost: 0 at compaction time

Throughout the session, a background process periodically extracts key information from the conversation into a structured markdown file. It uses a separate LLM call to update this file every time new context has accumulated (measured by token growth and tool call count).

The memory file follows a fixed template: session title, current state, task specification, important files, workflow commands, errors encountered, codebase documentation, learnings, key results, and a worklog. The extraction prompt is aggressive about density (capping the file at 12K tokens), demanding specific file paths, function names, error messages, and exact commands.

**By investing small amounts of compute throughout the session to maintain this document, the actual compaction event becomes free.**

When context pressure hits, session memory compaction just:

1. Reads the pre-built memory file
2. Figures out which messages have already been summarized into it
3. Prunes old messages, keeping a tail of recent ones
4. Inserts the memory file content as the summary

The memory document is built incrementally by an agent that has full context at each extraction point instead of one trying to compress the entire conversation into a summary under time pressure. The result: compaction is fast, cheap, and higher quality than a one-shot summary.

## Layer 3: Full Compact

Cost: 1 LLM call — the expensive fallback

When session memory compaction isn't available or can't reclaim enough space, the fallback is traditional LLM summarization.

A forked subagent summarizes the conversation, piggybacking on the main thread's prompt cache so it doesn't have to rebuild from scratch.

After summarization, the model loses access to files it recently worked with. This context is reconstructed by re-reading the most recently accessed files (up to 5, with a 50K token budget), and reattaching any active plan files, skill content, and async agent status.

## The Circuit Breaker

Full compaction can fail when the context is so large that even the summarization call is too long. Without fallback, compaction retries every single turn, burning API calls on doomed attempts.

A code comment tells the story. On March 10, 2026, someone ran a BigQuery analysis and found 1,279 sessions with 50+ consecutive compaction failures. The worst offender had 3,272 consecutive failures in a single session. Globally, this was wasting roughly 250K API calls per day.

The fix: after 3 consecutive failures, compaction is disabled for the rest of the session. Failure count resets on success.

## When Everything Fails: Reactive Recovery

Even with proactive compaction, the API can still reject requests with "prompt is too long." Most agents treat this as final and show the user an error.

Claude Code treats this as recoverable. The query loop withholds the error from the user, runs a compaction pass on the spot, and retries the request with fewer messages. The API even returns the exact token counts in the error so the retry logic knows exactly how much to cut. Only if the retry also fails does the error surface to the user. The user may never even know it happened.

## What This Means for Agent Design

Claude Code is Anthropic's answer to the scale versus scaffolding debate. Anthropic makes frontier models. If anyone can solve context management by scaling context windows, it's them. Instead, they built a multi-tier compaction system. And this makes sense. **The information density problem doesn't go away with scale. Scaffolding and scale are complementary strategies.**

What's interesting is how many independent lines of research converge on the same ideas Claude Code ships in production.

Charles Packer's MemGPT proposed tiered memory management where the LLM manages its own context across levels. Claude Code validates that design philosophy in production at scale.

Alex Zhang's Recursive Language Models decompose long contexts by recursively querying over chunks rather than processing everything in one pass. Claude Code's session memory applies the same principle in the time domain, building the summary incrementally rather than compressing everything at once.

The contrast with Cursor is also worth noting. Cursor does plenty of scaffolding, but for context management specifically, they used RL to train Composer to produce better summaries. Claude Code's model knows nothing about compaction. The system handles everything. For this particular problem, two of the most capable coding agents chose meaningfully different strategies. **That alone suggests we're still early in understanding which approach wins long-term.**

But if Anthropic is still building scaffolding, I'd bet on it mattering for a while.
