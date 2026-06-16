---
layout: post
title: "OpenVole: From a Micro-Agent Core to a Self-Hosted Agent OS"
author: Kürşat Kutlu Aydemir
tags: [OpenVole, "AI agents", "Open source", VoleNet]
---

*How a tiny agent loop grew, over four major versions, into a self-hosted operating system for AI agents — and three features I haven't seen anywhere else: VoleNet, agent server, and embedded paw apps.*

I started [OpenVole](https://github.com/openvole/openvole) in March 2026 with one conviction: most AI agent frameworks are too big. They ship with a worldview baked in, a memory implementation, a tool format, an opinion about how you should talk to your agent. OpenVole started as the opposite. A **microkernel**: the smallest possible thing that other useful things can be built on top of.

Three months and four major versions later it's something I didn't quite plan for, a server you run once and manage a fleet of agents from your browser. This is the story of how it got there, and a quick tour of the parts I'm proudest of.

## The bet: keep the core tiny

The core of OpenVole does exactly one thing: it runs the agent loop.

```
Bootstrap → Perceive → Compact → Think → Act → Observe → loop
```

That's it. The core is **LLM-ignorant** — it doesn't know what an LLM is. Everything that makes an agent *useful* lives outside the kernel, in two kinds of plugin:

- **Paws** — subprocess-isolated tool providers (the Brain, memory, a browser, a shell, a Telegram channel). Each runs in its own Node process behind a permission sandbox.
- **Skills** — behavioral recipes: a folder with a `SKILL.md` and no code.

So the "Think" step is just a Paw the core calls. Memory is a Paw. The dashboard, eventually, is a Paw. A fresh install has zero tools, zero skills, zero opinions, by design. That constraint is the whole point: the kernel stays small and boring while capability accretes at the edges, where it can be swapped, sandboxed, and reasoned about independently. But the strong claim of OpenVole's micro-kernel approach is it allows you to plug in your own brain paw which controls the loop with LLM and given system prompt. OpenVole allows you to plug your own brain paw implementation and you can decide its own low level brain instructions in `BRAIN.md`. You can even add OpenClaw or any custom AI agent's system prompt to `BRAIN.md` and that becomes your own low level LLM instruction within OpenVole.

That was **version 1**. A loop and a plugin contract.

## v1 → v2: capability at the edges

Once the contract was stable, the edges got interesting fast. **version 2.0.0** was the ecosystem leap, and notably *none of it touched the kernel*:

- **Real memory** — hybrid retrieval combining BM25 keyword search with vector similarity via Reciprocal Rank Fusion, plus temporal decay so old facts fade. Embeddings come from whatever you have (Ollama, OpenAI, Gemini), and it degrades gracefully to keyword-only if you have nothing.
- **Multi-agent** — named agent profiles with their own role, instructions, and per-agent tool allow/deny lists; a parent agent can spawn children and wait on them in parallel.
- **Docker sandboxing** — optional container isolation on top of the Node permission sandbox.
- **A skill registry** and a fleet of new paws — database, scraper, PDF, image, social.

The lesson of v2 was the bet paying off: you can grow an enormous amount of capability without growing the thing in the middle.

## v3: agents stop being islands — VoleNet

Every agent framework I'd seen treated an agent as a process on one machine. **v3.0.0** (April 2) asked a different question: what if agents could form a *network*?

[**VoleNet**](https://openvole.github.io/openvole/volenet.html) is, as far as I know, the **first peer-to-peer AI agent networking protocol** — and it's built into the kernel's edges like everything else. OpenVole v3 defined distributed AI agent networking protocol.  VoleNet instances pair using secure Ed25519 tokens, you share a public key, the peer trusts it, and every message from then on is cryptographically signed. You connect OpenVole instances into a mesh, and they share capabilities across machines:

- **Remote tools become local.** A tool running on a peer shows up in your local registry and the Brain calls it transparently — it has no idea the work happened on another box. If several peers offer the same tool, calls are **load-balanced** to the least-busy one.
- **Brain sharing.** A "brainless" worker (no LLM configured, no API cost) can delegate its *thinking* to a coordinator's Brain. Cheap workers, one expensive brain.
- **Shared memory and sessions** sync across the mesh, so a conversation can follow you between devices.
- It's all **Ed25519-authenticated** with replay protection, auto-reconnecting WebSocket transport, peer health monitoring, and **leader election** with automatic failover.

The result is a toolkit for genuinely distributed agents: a single brain driving tools scattered across a dozen machines, a team of independent agents, an autonomous swarm, a company-wide central brain. Eight topologies, one protocol, no central server required.

## v4: OpenVole becomes a server

We have too many agents no matter what framework we use. I am not talking about the spawned sub-agents. A serving AI agent in its own loop. A trading agent here, a research agent there, a personal assistant, each its own project directory with its own process and (in the old world) its own dashboard on its own port. Managing them was the bottleneck.

Version **v4.0.0** is the answer, and it's the biggest shift in the project's life: **OpenVole now runs as a server**.

```bash
mkdir ~/agents && cd ~/agents
npx vole serve
```

`vole serve` starts a single **control plane** — one web server (default `localhost:3000`) that manages *all* your agents from the browser. Each agent is a **space**: an isolated unit with its own config, paws, identity, and data, running as its own engine subprocess. From one dashboard you create, start, stop, switch between, and delete spaces; edit their config as structured forms; edit their identity files; and chat with each one.

![OpenVole dashboard — the Overview tab](/images/blog/2026-06-16-openvole-v1-to-v4/openvole-overview.png)
*The `vole serve` control plane managing a running **stock** space — its paws, 41 tools, live event stream, and the Overview / Chat / Apps / Config / Identity tabs.*

The single-engine CLI workflow (`vole init` / `vole start` / `vole run`) is *gone* — OpenVole is a server now, not a script you babysit. In practice it feels less like a library and more like a **personal operating system for agents** that you self-host: one process, many tenants, model-agnostic, your data on your machine. That last part matters to me — the whole thing runs on your hardware, against whatever model you point it at, with nothing phoning home.

## Paws that bring their own UI — embedded apps

The feature I had the most fun with shipped alongside v4. If you remember the old Facebook canvas — third-party pages and games embedded right inside the platform shell — that's the mental model.

In OpenVole, **a paw can ship its own UI**. It drops a static HTML file in its package and declares it in the manifest:

```json
{ "panel": { "title": "Markets", "html": "panel.html" } }
```

That's the entire integration. The control plane serves the panel and renders it as a sandboxed iframe under an **Apps** tab in the dashboard — one entry per paw that has a panel. The clever bit is what the panel talks to: it calls the paw's own tools directly, proxied over IPC by the control plane, **with no LLM in the loop** —

```js
// from inside the panel, relative to its URL — no Brain, no tokens
fetch('tool/stock_quote', { method: 'POST', body: JSON.stringify({ symbols: ['AAPL'] }) })
```

So a paw author gets a real, interactive app — dashboards, forms, live data views — with **no per-paw web server and no extra port**. Everything flows through the one control-plane server. The reference example is `paw-markets`, a stock-tracking paw whose "Markets" panel — watchlist, sparklines, live alerts — embeds this way. It's deterministic and free, because the brain only gets involved when *you* ask it to comment on the trends.

![OpenVole dashboard — the Apps tab with the Markets panel](/images/blog/2026-06-16-openvole-v1-to-v4/openvole-apps.png)
*The **Apps** tab: `paw-markets`'s embedded **Markets** panel — watchlist, sparklines, and live alerts — served by the control plane with no extra port.*

## Where this is going

Four versions in, the original bet still holds: the kernel is still tiny, still LLM-ignorant, still just a loop and a contract. Everything that grew — memory, the mesh, the server, the embedded apps — grew at the edges. That's the property I care about most, because it means the next big thing can be a plugin too.

OpenVole is open source ([github.com/openvole/openvole](https://github.com/openvole/openvole), docs at [openvole.github.io/openvole](https://openvole.github.io/openvole)). If a self-hosted, model-agnostic agent OS sounds like your kind of thing, `npx vole serve` is one command away.
