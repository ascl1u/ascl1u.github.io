---
layout: post
title: "Probing Long-Horizon Reasoning"
date: 2026-02-08
description: "Investigating brittleness in LLMs through active context management with Distributed-Inventory."
---

## Introduction

Current frontier LLMs "think" to output their best response, which can be seen in the rise of inference-time compute and CoT reasoning. However, this process is brittle. As context length increases, attention mechanisms degrade, and the model's ability to retrieve and utilize relevant info from the distant path falters.

I am investigating this brittleness through a specific lens: active context management. Distributed-Inventory is an RL environment designed to isolate and test an LLM's ability to maintain a coherent state over long horizons without relying on the crutch of massive context windows.

## Environment Design

Adhering to the bitter lesson, the model is overloaded with context, rather than cognitively disabled.

The environment is built using the `verifiers` library and trained using `prime-rl` A `NoiseGenerator` deterministically produces an entropic stream of words using a "common noun" vocabulary. Buried within this stream are commands `[GET]` and `[DROP]` which operate on a distinct "fantasy item" vocabulary. This lexical separation ensures state retention is tested and not the ability to parse ambiguous text.

The critical innovation is how the context window is handled. In a standard loop, the model is able to see the entire conversation history. A strict memory wipe is enforced by overriding the environment's conversation history. This ensures that at any given turn, the model only sees:
1) the system prompt
2) its own previous output
3) the current chunk of noise and commands
This forces the model to explicitly "carry" the inventory state forward in its output.

To facilitate this behavior, the system prompt establishes the memory wipe as an explicit environmental constraint.

```
"You are an Inventory Manager processing a data stream.\n"
            "GOAL: Maintain an accurate list of items based on [GET] and [DROP] commands.\n\n"
            "CRITICAL ENVIRONMENTAL CONSTRAINTS:\n"
            "1. MEMORY WIPE: After you reply, your entire context window is cleared.\n"
            "2. VISIBILITY: The ONLY history you will see in the next turn is the output you write NOW.\n"
            "3. ENTROPY: If you do not write a piece of information down, it is deleted from the universe forever.\n\n"
            "Survive the stream and report the final inventory when asked."
```

A key design choice in this experiment is the use of sparse rewards. The agent receives a reward signal only at the very end of the episode. This forces the model to discover the "carry-over" strategy on its own, treating intermediate state maintenance as a latent variable necessary for solving the final objective.

## Results

I trained `INTELLECT-3` on a lightweight configuration with a batch size of 16 and 8 rollouts per example.

![Experiment A](/assets/images/distributed_inventory/a.png)
Experiment A: `n_chunks = 5, chunk_size = 100, n_ops = 5, max_tokens = 1024`

![Experiment B](/assets/images/distributed_inventory/b.png)
Experiment B: `n_chunks = 10, chunk_size = 1000, n_ops = 5, max_tokens = 512`

![Experiment C](/assets/images/distributed_inventory/c.png)
Experiment C: `n_chunks = 10, chunk_size = 1000, n_ops = 5, max_tokens = 512`

![Experiment D](/assets/images/distributed_inventory/d.png)
Experiment D: `n_chunks = 20, chunk_size = 1000, n_ops = 5, max_tokens = 1024`

## Analysis

### The Gradient Noise Regime

In experiment A, the model learns quickly, but the reward curve remains highly oscillatory. This variance is a direct artifact of our configuration. A larger batch size would likely smooth out this curve, but the noise here proves the signal is strong enough to recover from bad updates

### The Exploration Bottleneck

For experiments B and C, the runs display a grokking pattern, a long period of no reward followed by a phase transition. This delay is the exploration cost of sparse rewards. For the first ~40 episodes, the model wanders blindly, until it generates a valid active state sequence. Once this successful trajectory is found, the gradients cause the policy to collapse almost instantly onto the solution

### The Signal-to-Noise Limit

In experiment D, the exploration space explodes. The flatline suggests the model has hit a sample efficiency wall. The signal exists, but without higher rollout diversity or intermediate rewards, a viable strategy cannot be discovered before the run concludes.

## Future Work

To stabilize long-horizon reasoning, I plan to scale up the training configuration: larger batch sizes, more rollouts per example, and increasingly complex chunk sequences.

[Distributed-Inventory](https://app.primeintellect.ai/dashboard/environments/ascl1u/distributed-inventory) is available on the Prime Intellect Environments Hub for further exploration. Special thanks to Prime Intellect, as this would not have been possible without their hosted RL beta.
