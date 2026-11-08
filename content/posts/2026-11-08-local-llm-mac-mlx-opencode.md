---
layout: post
title: "Local LLMs on a 32 GB Mac: MLX, Qwen, and Making OpenCode Work With Them"
date: 2026-11-08
tags: [llm, mlx, qwen, opencode, apple-silicon, local-ai, ai]
---

I wanted a coding agent that works **without the internet**. Not for everything,
only for simple daily tasks: read a few files, edit a few lines, write a short
doc. My laptop is a Mac with an M1 Max and 32 GB of RAM. This post is
what I learned: which models fit, how fast they are, and why OpenCode needed a
fix before a local model could use it.

All numbers below are from my own machine, on AC power, with small samples. Treat
them as rough. Where I guessed, I say so.

## The setup

The stack is small:

- [MLX](https://github.com/ml-explore/mlx) and `mlx-lm` (0.32.2), on Python 3.14.
- `mlx_lm.server`, which gives an **OpenAI-compatible HTTP endpoint** on
  `http://127.0.0.1:8080/v1`.
- A Makefile, so that `make qwen38` creates the virtualenv and starts the server.

Any tool that speaks the OpenAI API can use it. For me, that tool is
[OpenCode](https://opencode.ai) (v2.2.6).

## Which model fits in 32 GB?

macOS lets the GPU use only part of the RAM (roughly 65-75% by default). So a
model of 29 GB does not "fit in 32 GB" in practice.

| Model | Size | Verdict |
| --- | --- | --- |
| Qwen3.8-27B, 4-bit | about 16 GB | **Works.** 15.5 GB peak memory in my test |
| Qwen3.8-27B, 8-bit | about 29 GB | Too tight |
| Qwen3.8 Flash-Next | 81 GB and up | Does not fit |
| Qwen2.5-Coder-14B, 4-bit | about 8 GB | Works, faster |
| DeepSeek-R1-Distill-Qwen-14B, 4-bit | about 8 GB | Works, but see below |

Flash-Next surprised me. The name sounds small. It is a mixture-of-experts model
with about 177 billion parameters, and the smallest MLX build I found is a 2-bit
one of 81 GB. **Check the size on Hugging Face before you download.**

## The first crash

The server died with `[METAL] Command buffer execution failed: Insufficient
Memory`. The log showed the cause: the **prompt cache** had grown to 7.4 GB, on
top of 16 GB of weights. The server keeps the KV cache of old prompts so that it
can reuse them, and by default it does not stop at a sensible size.

What helped:

- `--prompt-cache-bytes` and `--prompt-cache-size`: cap the cache.
- `--kv-bits 8`: quantize the KV cache.
- `--prefill-step-size 512`: a smaller chunk for reading the prompt, so the peak
  memory is lower.
- I did **not** change the macOS GPU memory limit. The default, about 24 GB on
  this Mac, is enough for models of 12-19 GB once the cache is capped. (If you
  need more, `sudo sysctl iogpu.wired_limit_mb=...` raises it until reboot.)

One more detail: the cache size limit was not strict in my logs. I saw 3.2 GB
and 4.95 GB against limits of 2 GB and 4 GB. Watch the `Prompt Cache` lines.

Also, keep the number of cached prompts big enough. With 4 slots, short side
requests pushed the big agent prompt out of the cache, and the next turn read 14k
tokens again from zero. That cost about 3 minutes. With 12 slots it was fine,
because a cache entry was only about 0.17 GB.

## How fast is it?

Two different speeds matter, and they have different limits.

**Decode** (writing the answer) is limited by memory bandwidth. The M1 Max has
400 GB/s, so a 16 GB model has a theoretical maximum of about 25 tok/s.

**Prefill** (reading the prompt) is limited by compute. This matters for agents,
because the agent sends a very long prompt every time.

| Setup | Decode | Prefill |
| --- | --- | --- |
| Qwen3.8-27B 4-bit, `mlx_lm` | about 20 tok/s | about 60-110 tok/s |
| Qwen3.8-27B 4-bit + MTP draft model | about 25 tok/s in a benchmark, about 18-19 in OpenCode | about 110 tok/s |
| Qwen2.5-Coder-14B 4-bit | about 22 tok/s (small sample) | about 250 tok/s at the start, about 160 deeper |

Notes on this table:

- On battery, my early server logs showed only 9-12 tok/s. I believe the power
  mode was the main reason, but I did not test it separately.
- A 20,000-token prompt takes about **3.5 minutes** to read at 100 tok/s. That is
  the real cost of an agent, not the decode speed.

### MTP: a nice benchmark, a bad fit for an agent

MTP (multi-token prediction) is speculative decoding. A small 0.3 GB draft model
guesses a few tokens ahead, and the big model checks them in one pass. It needs
`mlx-vlm`, because `mlx-lm` drops the MTP weights, and it costs about 0.7 GB more
memory.

In a simple test it looked great: **+25%** (24.8 against 19.8 tok/s), with 86%
of the drafted tokens accepted. I was happy. Then I used it with OpenCode, and I
do not like it any more:

- **No speed gain.** Decode was about 18-19 tok/s with the draft model and about
  18-19 tok/s without it. My benchmark was greedy code generation, which is the
  best case for speculation. An agent writing tool calls is not that.
- **No prompt reuse.** Every turn showed `cached_tokens=0`. The prompt grew from
  2.6k to 5.6k to 11.3k to 16.5k tokens, and each turn read all of it again: 52 s
  for the 5.6k turn and 106 s for the 11.3k one, against about 5 s of decoding.
  About 95% of each turn was prompt reading, and it grows with every step.
- **It is not the draft model's fault.** I started the same server without the
  draft model and got the same result (79 s of prefill for the second turn,
  `cached_tokens=0`). This `mlx_vlm` server just does not reuse the cache in this
  flow. I found no flag for it.
- **It batches requests.** OpenCode's title request ran at the same time as the
  first real request, and both were slow (the title took 41 s). `--max-num-seqs 1`
  makes them queue instead, but then the real request waits for the title.

The plain `mlx_lm` server is the better choice for an agent, because it keeps a
prompt cache. I checked it directly: a 3,100-token conversation took 29.5 s for
the first request, and the next turn (the same conversation plus the reply and a
new question) took **1.4 s**, with `cached_tokens=3121`. You can read the same
number in every response, in `usage.prompt_tokens_details.cached_tokens`. One
surprise: sending the *identical* prompt twice did not hit the cache
(`cached_tokens=0`). The cache matches a conversation that continues; it does
not match a prompt that starts again. My guess is that this model's recurrent
layers cannot be rolled back to a shorter prefix. In practice, an agent turn is
cheap, and a new session pays the full read of the system prompt once. That is
one more reason to keep that prompt short.

I keep the MTP target in my Makefile, but I do not use it.

### Turn thinking off

Qwen3.8 thinks before it answers. For "Say hi" it spent its first 16 tokens on
reasoning. On a real task, thinking can use thousands of tokens, and at 20 tok/s
that is minutes. The chat template has an `enable_thinking` switch, and the
server can set it as a default:

```sh
mlx_lm.server --chat-template-args '{"enable_thinking": false}' ...
```

I made this the default in my Makefile (`THINKING=true` turns it back on).

## OpenCode: the model must support tools

An agent works by calling tools (`read`, `edit`, `write`, `shell`). The model
and the server must both support tool calls.

I tried DeepSeek-R1-Distill first. The server printed `Received tools but model
does not support tool calling`. The result was funny: I asked the agent to add a
post to my docs, and it **wrote a nice post in the chat** and invented a
"./docs directory" it had never looked at. No file was created. The model never
saw the tools.

So for agent work, check the warning in the server log. A reasoning model that
cannot call tools is still fine for chat and questions.

## The big problem: the OpenCode prompt

My first request to OpenCode was **11,658 tokens**, and a later one was 20,157.
I turned on `--log-level DEBUG` in the server, which prints the full request,
and I read it.

The system prompt is not a fixed text. OpenCode builds it from **what is on your
machine**:

- a list of about 45 skills, with long descriptions (AWS, Bedrock, pptx, xlsx,
  and so on). These came from my global skill setup, which I use with
  Claude Code and Codex. They have nothing to do with editing a markdown file;
- a "Code Mode" catalog that tells the model some tools "cannot be called
  directly" and only work as JavaScript inside an `execute` tool;
- tools such as `websearch`, `webfetch`, `question`, and `subagent`.

This is fine for a big hosted model with a huge context and fast prefill. For a
14B model on a laptop it is bad. The skills list was most of the prompt, the
`execute` indirection can confuse a small model, and everything is prefill time.

### The fix: a lean agent

OpenCode has agents with their own permission rules. Denied tools are removed
from the request. I made an agent called `local` in
`~/.config/opencode/opencode.json`:

```json
{
  "agents": {
    "local": {
      "description": "Trimmed prompt/tools for small local models",
      "mode": "primary",
      "permissions": [
        { "action": "skill",      "resource": "*", "effect": "deny" },
        { "action": "websearch",  "resource": "*", "effect": "deny" },
        { "action": "webfetch",   "resource": "*", "effect": "deny" },
        { "action": "question",   "resource": "*", "effect": "deny" },
        { "action": "subagent",   "resource": "*", "effect": "deny" },
        { "action": "execute",    "resource": "*", "effect": "deny" }
      ]
    }
  }
}
```

I also denied the `opencode_*` tools that appear once `execute` is gone (session
and worktree helpers). The names are in the tool list of the request.

To measure it without a model, I pointed OpenCode at a tiny fake OpenAI server
that only logs the request. The result:

| Agent | Request size | System prompt |
| --- | --- | --- |
| `build` (default) | 50.9 KB | 36.6k characters, with all skills |
| `local` | 10.1 KB | 2.4k characters |

That is about 80% less. It is bytes, not tokens, but the prompt shrinks by about
the same share. And in real use it **works much better**. I can do simple tasks
with no internet.

Three practical notes:

- Switch to the agent with **Shift+Tab** inside the TUI. `opencode --agent local`
  did not work for me for the interactive mode. It works with
  `opencode run --agent local "do something"`.
- Start OpenCode with **`opencode --standalone`**. By default OpenCode uses a
  shared background service. With `--standalone` it runs a private server, so no
  other process talks to your local model. This matters: the local server
  handles one request at a time, and a title-generation request or a second
  session in the background will queue behind your work, or break a test that
  depends on timing.

- OpenCode also sends a **title request** at the start of each session, to the
  same model. On my 27B it took about 15 s, and the real request waited behind
  it. Setting `"agents": { "title": { "disabled": true } }` removes it. I checked
  with the fake server that the request is no longer sent. The cost is that your
  sessions get no automatic names.

## What I would do next

Daily tasks need **speed** more than the biggest model. The 27B is good, but
20 tok/s and a 100 tok/s prefill are slow for small edits. The fast local models
are mixture-of-experts models: they have many parameters, but they use only a
few for each token, so decode is limited by the small active part. I have not
tested these yet, but they exist in MLX 4-bit and fit in 32 GB:

- `Qwen3-Coder-30B-A3B-Instruct-4bit` (17.2 GB, about 3B active parameters);
- `Qwen3-30B-A3B-Instruct-2507-4bit` (17.2 GB);
- `gpt-oss-20b-MXFP4-Q8` (12.1 GB).

For each one, I will check three things: decode speed, prefill speed, and, most
important, whether the model calls tools correctly in OpenCode. A fast model that
writes the answer in the chat instead of editing the file is not an agent.

If you take only one thing from this post: **read the prompt your agent sends.**
It is a plain JSON in the server log, and it shows you the real cost.
