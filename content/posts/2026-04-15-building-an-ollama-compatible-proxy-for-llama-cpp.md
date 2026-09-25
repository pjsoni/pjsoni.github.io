--- 
title: "Building an Ollama-Compatible Proxy for llama.cpp" 
date: 2026-04-15 
slug: building-an-ollama-compatible-proxy-for-llama-cpp
summary: "A lightweight Go proxy that bridges Ollama-compatible applications with llama.cpp, allowing existing tools to work with a llama.cpp backend without modification. The proxy translates Ollama API requests into llama.cpp calls and supports chat, embeddings, and model discovery while preserving the performance and configuration flexibility of llama.cpp."
categories: [Artificial Intellegence, LLM]
tags: [homelab, configuration]
jumbotron:
  meta: true
---

One thing I've learned while building local AI infrastructure is that model serving is only half the battle. The other half is making the surrounding ecosystem work together.

There are plenty of reasons to choose **llama.cpp** as your inference backend. It's lightweight, highly portable, performs well across a wide range of hardware, and gives you complete control over how models are loaded and served.

The challenge is that many AI applications are built with a single backend in mind: **Ollama**.

This isn't necessarily a bad thing. In fact, Ollama has become the de facto standard for local LLM integrations. Many desktop clients, mobile applications, coding assistants, and automation tools simply assume that an Ollama endpoint exists.

That assumption creates an interesting problem for people who prefer running something else underneath.

## The Problem

At a high level, I think of **Ollama** and **llama.cpp** a bit like macOS/iOS and Linux.

Ollama is the "it just works" experience. Install it, pull a model, point your application at it, and you're productive in minutes. For many users, that's exactly the right choice.

llama.cpp sits on the other end of the spectrum. It exposes more of the underlying machinery and gives you significantly more control over how models are loaded, configured, and served. If you're the type of person who enjoys tweaking kernel parameters, tuning storage layouts, optimizing GPU allocation, or squeezing every last bit of performance out of your hardware, llama.cpp feels very familiar.

Neither approach is universally better. They're optimized for different audiences.

For my environment, I preferred llama.cpp.

After spending time tuning model loading, GPU offloading, context sizes, batching behavior, and server parameters, I was able to achieve more than **50 tokens per second running GPT-OSS 20B on my Pascal GPUs**. For a model of that size, I was very happy with the results. Smaller models perform even better, often reaching **80+ tokens per second** on the same hardware. For the agentic workflows I run in my homelab, that level of control and performance mattered.

The obvious question is:

> If Ollama already exposes OpenAI-compatible APIs, why not just use Ollama for everything?

I considered that approach.

While Ollama does provide OpenAI API compatibility, its primary goal is to provide a simple and approachable user experience. That's one of its greatest strengths. In my case, however, I wanted access to every tuning parameter available.

I wanted control over:

* GPU offloading
* Context management
* Batching behavior
* Memory allocation
* Model loading strategies
* Server runtime parameters

In short, I wanted to be able to turn every knob available and optimize the stack specifically for my hardware.

As a result, moving everything to Ollama wasn't the solution I was looking for.

The problem was that several applications I relied on only supported **Ollama-compatible APIs**.

Some offered no OpenAI-compatible configuration at all. Others technically supported OpenAI endpoints but lacked features or integrations when used outside of Ollama.

This left me in an awkward position.

I ended up running two inference stacks:

* **Ollama** for applications that only understood Ollama APIs
* **llama.cpp** for my optimized and performance-sensitive workloads

Functionally, both were serving the same models.

Operationally, it felt wasteful.

I had duplicate model storage, duplicate configurations, separate monitoring, and additional infrastructure to maintain. More importantly, I wasn't getting the benefit of standardizing on the inference backend I actually wanted to use.

I started looking for alternatives.

Ideally, I wanted applications that could talk directly to OpenAI-compatible APIs so I could standardize on llama.cpp. Unfortunately, some of the tools I relied on were tightly coupled to Ollama and offered no practical alternative.

That's when I realized the problem wasn't model serving at all.

The problem was API compatibility.

## The Solution

Instead of changing applications or maintaining multiple backends, I decided to build a lightweight Go-based proxy that sits in front of llama.cpp and translates Ollama API requests into OpenAI-compatible requests.

The goal was simple:

* Keep llama.cpp as the only inference backend
* Preserve all llama.cpp tuning and performance benefits
* Allow Ollama-only applications to continue working unchanged

No client modifications.

No forks.

No hacks.

Just a lightweight compatibility layer.

---

## High-Level Architecture

The final architecture looks like this:

![Ollama Proxy](/assets/images/ollama-proxy.png "Ollama Proxy")

In this setup:

* `chat.example.com` routes directly to llama.cpp
* `chat.example.com/ollama` routes to the proxy
* The proxy exposes only Ollama endpoints
* llama.cpp continues exposing only OpenAI-compatible endpoints

This keeps responsibilities clean and avoids turning the proxy into a generic API gateway.

---

## Design Goals

Before writing any code, I established a few requirements.

### Exact Ollama Compatibility

The primary objective was making existing applications believe they were talking to a real Ollama server.

That includes:

* `/api/chat`
* `/api/generate`
* `/api/tags`
* `/api/show`
* `/api/embeddings`

along with:

* NDJSON streaming
* Model metadata
* Multimodal requests
* Embeddings support

If an application expects Ollama behavior, it should simply work.

### Keep the Proxy Focused

The proxy is not intended to be another OpenAI gateway.

Its only purpose is exposing an Ollama-compatible interface backed by llama.cpp.

Anything outside that scope remains the responsibility of llama.cpp.

### Optional Debugging

During development, visibility into upstream traffic was invaluable.

A single environment variable enables request and response logging:

```text
DEBUG_UPSTREAM=true
```

When disabled, the proxy remains silent.

### No Assumptions About Models

Different organizations use wildly different naming conventions.

The proxy does not attempt to infer architecture, quantization level, or model family from a filename. Instead, it generates predictable metadata while leaving actual model management to llama.cpp.

---

## How It Works

### Chat Requests

When a client sends a request to:

```text
/api/chat
```

the proxy:

1. Parses the Ollama request
2. Converts messages into OpenAI format
3. Preserves roles and conversation history
4. Handles multimodal content
5. Forwards the request to:

```text
/v1/chat/completions
```

on the llama.cpp server.

The response is then translated back into Ollama format before being returned to the client.

---

### Streaming

Streaming compatibility was one of the most important pieces.

Ollama streams responses as NDJSON:

```text
{"response":"Hello","done":false}
{"response":" world","done":false}
{"done":true}
```

Meanwhile, llama.cpp streams using Server-Sent Events (SSE).

The proxy consumes the SSE stream and reconstructs Ollama-style NDJSON output on the fly.

From the client's perspective, the experience is indistinguishable from talking to a native Ollama server.

---

### Model Discovery

Many applications query:

```text
/api/tags
```

to discover available models.

llama.cpp provides only minimal model metadata.

To bridge that gap, the proxy synthesizes an Ollama-style response that includes:

* Name
* Model identifier
* Digest
* Size
* Modification timestamp
* Model details

This provides enough information for clients to display and select models correctly.

---

### Embeddings

Embedding requests are simply forwarded from:

```text
/api/embeddings
```

to:

```text
/v1/embeddings
```

with the response converted back into Ollama format.

---

### Model Management Endpoints

Ollama supports operations such as:

```text
/api/copy
/api/delete
```

Since llama.cpp doesn't implement model management in the same way, these endpoints are implemented as lightweight compatibility stubs.

They exist solely to satisfy client expectations.

---

## Why This Approach Works

The biggest benefit isn't API translation.

It's operational simplicity.

Instead of maintaining separate Ollama and llama.cpp deployments, I can standardize on a single inference backend while continuing to use applications that expect Ollama.

That means:

* One model repository
* One inference stack
* One tuning strategy
* One monitoring pipeline
* One place to optimize performance

The applications remain unchanged.

The infrastructure becomes simpler.

---

## Final Thoughts

This project started as a workaround for a handful of applications that refused to speak anything other than Ollama.

It ended up becoming a useful compatibility layer that allowed me to standardize on the backend I actually wanted to run.

Ollama excels at making local LLMs accessible.

llama.cpp excels at giving operators and infrastructure enthusiasts complete control over how inference is performed.

With a lightweight proxy sitting between the two, there's no reason you can't benefit from both.

Sometimes the most useful projects aren't the complex ones.

They're the small pieces of glue that let everything else work together.
