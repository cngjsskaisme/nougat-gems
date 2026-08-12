**"Conversation Platform"**

From a technical perspective, **Frontend / Transport / Backend / AI Engine** are each designed as independent layers.

---

# Overall Structure

```text
                        Browser (Nuxt)

┌────────────────────────────────────────────────────┐
│ UI Components                                      │
│ Deal Chat                                          │
│ Humor Chat                                         │
│ Chat Rooms                                         │
│ Deal Cards                                         │
│ Brand Comparison                                   │
│ Suggestion UI                                      │
└────────────────────────────────────────────────────┘
                        │
                        ▼
┌────────────────────────────────────────────────────┐
│ Chat State Layer                                   │
│ Messages                                            │
│ Conversation                                        │
│ Suggested Questions                                │
│ Chat Rooms                                          │
│ Scroll Manager                                      │
└────────────────────────────────────────────────────┘
                        │
                        ▼
┌────────────────────────────────────────────────────┐
│ API Layer                                          │
│ streamDealChat()                                   │
│ apiFetchRaw()                                      │
│ ReadableStream                                     │
│ Event Dispatcher                                   │
└────────────────────────────────────────────────────┘
                        │
                        ▼
──────────────────────────────────────────────────────────────
                 Nitro routeRules Reverse Proxy
──────────────────────────────────────────────────────────────
                        │
                        ▼
┌────────────────────────────────────────────────────┐
│ Chat Controller                                    │
│ NDJSON Stream                                      │
└────────────────────────────────────────────────────┘
                        │
                        ▼
┌────────────────────────────────────────────────────┐
│ Chat Service                                       │
│ Conversation                                        │
│ Pending Message                                     │
│ Finalize                                            │
└────────────────────────────────────────────────────┘
                        │
                        ▼
┌────────────────────────────────────────────────────┐
│ Chat Orchestrator                                  │
│ Intent                                              │
│ Search                                              │
│ Category                                            │
│ Comments                                            │
│ RAG                                                 │
│ Summary                                             │
└────────────────────────────────────────────────────┘
                        │
                        ▼
┌────────────────────────────────────────────────────┐
│ Prompt Builder                                     │
│ System                                              │
│ Context                                             │
│ Fixed RAG                                           │
│ Conversation Summary                                │
└────────────────────────────────────────────────────┘
                        │
                        ▼
┌────────────────────────────────────────────────────┐
│ LLM Gateway                                        │
│ GPT                                                 │
│ Gemini                                              │
│ Streaming                                           │
└────────────────────────────────────────────────────┘
                        │
                        ▼
┌────────────────────────────────────────────────────┐
│ SectionBlockParser                                 │
│ Event Generator                                     │
└────────────────────────────────────────────────────┘
```

---

# FE Architecture

The Frontend consists of **7 Layers**.

```text
UI

↓

Chat State

↓

API Layer

↓

Transport

↓

Streaming

↓

Event

↓

Rendering
```

---

# 1. UI Layer

The UI consists of

```text
Deal Chat

Humor Chat

ChatMessageItem

DealCardList

CriteriaBox

BrandComparison

SuggestionButtons
```

The UI itself is

Block-based.

---

# 2. Chat State Layer

It holds a lot of state.

```text
messages

conversationId

status

chatRooms

suggestedQuestions

stickToBottom

loading

abortController
```

among others.

---

# Conversation

A Conversation ID exists.

```text
conversationId

↓

Backend

↓

DB
```

It connects to these.

---

# Chat Room

Rooms are also

managed separately.

```text
localStorage

↓

Room List

↓

Restore
```

---

# Scroll Manager

Due to Streaming,

auto-scroll is also

a separate Layer.

```text
IntersectionObserver

↓

stickToBottom

↓

scrollIntoView()
```

If the user

is looking at the top,

auto-scroll stops.

---

# API Layer

It is a Streaming Client.

Representative function

```text
streamDealChat()
```

---

Order

```text
fetch()

↓

ReadableStream

↓

TextDecoder

↓

Buffer

↓

JSON

↓

Event
```

---

# Transport Layer
This is a characteristic feature.

Transport is

an independent Layer.

```text
Browser

↓

/api

↓

Nitro Proxy

↓

Backend
```

---

## Nitro Proxy

There is no separate API Route.

```text
/api/**

↓

routeRules

↓

sendProxy()
```

This is it.

---

### Characteristics

* Reverse Proxy
* Streaming preservation
* CORS removal
* Cache removal

All handled here.

---

# Authentication Layer

It is inside Transport.

Client

```text
Authorization

Timestamp

Nonce

Signature
```

---

SSR

```text
X-internal-Request
```

This is it.

---

# Retry Layer

Transport also

handles 401.

```text
401

↓

guest/init

↓

Retry
```

---

# Streaming Layer

Streaming is also

an independent Layer.

```text
ReadableStream

↓

Chunk

↓

Decode

↓

Line

↓

JSON
```

---

Buffer is

managed directly.

```ts
buffer += decoded

lines = buffer.split("\n")

buffer = lastLine
```

---

# Event Layer

Streaming JSON is

not directly UI.

First

it becomes an Event.

```text
meta

text

dealCards

comparison

questions

done

error
```

---

Event Dispatcher is

```text
applyEvent()
```

---

# Rendering Layer

All Events

become Blocks.

```text
text

↓

Markdown Component
```

---

```text
dealCards

↓

DealCardList
```

---

```text
comparison

↓

BrandComparison
```

This is it.

---

# Backend Architecture

The Backend consists of

7 Layers.

```text
HTTP

↓

Streaming

↓

Conversation

↓

Context

↓

Prompt

↓

LLM

↓

Parser
```

---

# HTTP Layer

It is a Controller.

```text
POST /deals/chat
```

It handles only this.

Role

```text
Response Header

Abort

Stream Write
```

This is it.

---

# Streaming Layer

The Controller is

a Streaming Relay.

```text
LLM

↓

Chunk

↓

Parser

↓

NDJSON

↓

Browser
```

This is it.

---

# Chat Service

The Service

manages Conversations.

```text
Prepare

↓

Pending Message

↓

Stream

↓

Finalize
```

This is it.

---

In Finalize,

```text
Assistant Message

↓

DB

↓

Mentioned Deals
```

are saved.

---

# Conversation Layer

Conversation is

DB-centric.

```text
Conversation

↓

Messages

↓

Summary

↓

Turn Context
```

This is it.

---

# Chat Orchestrator

This is

the core.

All Context is

generated here.

---

The order is

```text
Question

↓

Intent

↓

Keyword Expansion

↓

ElasticSearch

↓

Category

↓

Deals

↓

Comments

↓

RAG

↓

Summary

↓

Conversation Context
```

This is it.

---

# Intent Analyzer

It classifies

questions.

```text
Recommendation

Search

Evaluation

Detail

FollowUp

General

Unknown
```

---

# Search Layer

Search is also

multiple stages.

```text
Expand

↓

ES

↓

Category

↓

Deals

↓

Comments
```

---

# Memory Layer

Conversation Summary also

exists.

There are two types.

```text
Fixed RAG

Conversation Summary
```

---

Fixed

```text
Deal

Market

Comments
```

---

Conversation

```text
Goal

Preference

Rejected

Pending
```

---

# Prompt Builder

Prompt is also

hierarchical.

```text
System

↓

Mode

↓

Intent

↓

RAG

↓

Conversation

↓

Messages

↓

Question
```

---

# LLM Gateway

LLM is also

a Gateway.

```text
createCompletion()

streamCompletion()
```

Both are provided.

---

# Parser Layer

This is the most important Layer.

LLM Output is

not used directly in UI.

```text
Raw Text

↓

SectionBlockParser

↓

Semantic Block

↓

NDJSON Event
```

This is it.

---

# Event Contract

```text
meta

text

block/dealCards

block/comparison

block/questions

done
```

This is it.

---

# Overall Sequence

```text
User

↓

DealChat.vue

↓

streamDealChat()

↓

apiFetchRaw()

↓

Nitro Proxy

↓

ChatController

↓

ChatService

↓

ChatOrchestrator

↓

PromptBuilder

↓

LLM Gateway

↓

Streaming

↓

SectionBlockParser

↓

NDJSON

↓

ReadableStream

↓

applyEvent()

↓

Vue Components
```

---

# Technical Characteristics

## 1. Stateful Backend

Conversation is

saved in DB.

```text
Conversation

↓

Message

↓

Summary

↓

Mentioned Deals
```

This is it.

---

## 2. Streaming First

The entire system is

designed around Streaming.

```text
Token

↓

Semantic

↓

NDJSON

↓

Event

↓

UI
```

This is it.

---

## 3. Layer Separation

Roles are

clearly separated.

```text
Controller

↓

Service

↓

Orchestrator

↓

Prompt

↓

Gateway

↓

Parser
```

This is it.

---

## 4. Context-driven AI

Context generation is

more important than

the Prompt.

LLM does not

search.

It uses

already searched Context.

---

## 5. Component-driven Rendering

UI is also

not Markdown but

Blocks.

```text
LLM

↓

Block

↓

Vue Component
```

This is it.

---

# Technical Advantages

### Scalability

When adding new features, you mostly only need to modify a specific layer.

* New search strategy → `ChatOrchestrator`
* New response block → `SectionBlockParser` + Vue component
* New LLM → `LlmRealtimeGateway`
* New UI → Add Block component

Responsibilities between layers are relatively clearly separated.

---

### Real-time UX

Based on NDJSON + `ReadableStream`, the UI is updated as soon as tokens are generated, providing good perceived responsiveness.

---

### Conversation Persistence

Since Conversation, Message, and Summary are saved, long-term conversations and resuming conversations are possible.

---

### RAG-friendly Structure

Since search, category determination, comment summarization, and conversation summarization are all prepared in stages before the LLM call, the Prompt is consistent and highly reproducible.

---

# Limitations of the Current Structure

The biggest improvement point in the current structure is **that `ChatOrchestrator` has too many responsibilities**.

Currently, `ChatOrchestrator` performs the following roles simultaneously.

* Intent analysis
* Search keyword expansion
* ElasticSearch query
* Category determination
* Comment loading
* RAG Summary retrieval/generation
* Conversation creation and restoration
* Prompt Context assembly

In other words, a single class handles the roles of **Conversation Coordinator + Retrieval Coordinator + Context Builder** all at once.

In the long term, it would be ideal to further subdivide it as follows.

```text
ChatOrchestrator
│
├── IntentPipeline
├── RetrievalPipeline
├── ConversationManager
├── ContextAssembler
└── PromptBuilder
```

By separating it this way, while maintaining the current strengths (streaming, event-driven UI, RAG, long-term conversations), testing and maintenance of each layer would become much easier, and adding new AI features would also be more straightforward.


# FE Transport Layer (Streaming Proxy)

One of the biggest characteristics is **that it does not use Nuxt API Routes, but instead uses Nitro's `routeRules` Reverse Proxy**.

In other words

```text
Browser

↓

/api/**

↓

Nitro Reverse Proxy

↓

Backend
```

This is it.

---

## Architecture
```text
Browser

↓

/api/**

↓

routeRules

↓

sendProxy()

↓

Backend
```

This is it.

There is no API Route.

The Proxy replaces the API Layer.

---

# Nitro routeRules

In practice, it is

```ts
routeRules

↓

proxy

↓

sendProxy()
```

This is it.

All

```text
/api/*
```

requests are

passed directly to the Backend.

In other words

```text
/api/deals/chat

↓

http://backend/api/deals/chat
```

This is it.

---

# Streaming Friendly Proxy

This is the most important part.

Nitro does not

reassemble Streaming Responses.

```text
Backend

↓

Chunk

↓

Nitro

↓

Chunk

↓

Browser
```

This is it.

In other words

Streaming is preserved as-is.

---

# Why not server/api?

If you use Nuxt API Routes,

```ts
const result = await fetch(...)

return result
```

you would need to create

a Wrapper like this.

In Streaming,

this Wrapper

actually gets in the way.

---

routeRules only does

```text
Receive

↓

Forward
```

Therefore

there is no Streaming Loss.

---

# Client Transport Layer

The Frontend does not

use High Level APIs.

Representative function

```text
apiFetchRaw()
```

This is it.

Because

in Streaming,

```text
Response.body
```

is needed.

---

# ReadableStream

Transport uses

Browser Native APIs.

```text
fetch()

↓

Response

↓

ReadableStream

↓

Reader

↓

Chunk
```

This is it.

Chunks are processed

as soon as they arrive.

---

# Buffer Management

In Streaming,

a Chunk is

not always one complete JSON.

For example

First Chunk

```text
{"type":"te
```

Second Chunk

```text
xt","content":"안녕하세요"}
```

This is it.

So

```text
buffer

+=

decoded
```

And then

```text
split("\n")
```

The last line is

kept until

the next Chunk.

This is

the core of the Transport Layer.

---

# NDJSON Parser

From the Buffer,

only complete Lines are

```text
JSON.parse()
```

In other words

```text
Bytes

↓

String

↓

Line

↓

JSON

↓

Event
```

This is it.

---

# Abort Propagation

The Transport Layer also

handles cancellation.

```text
Browser Abort

↓

AbortController

↓

fetch cancel

↓

TCP Close

↓

Nitro

↓

Backend

↓

LLM Cancel
```

In other words

when the user

presses the stop button,

even the LLM

is aborted.

---

# Authentication

Inside the Transport Layer,

authentication is also included.

Client

```text
Authorization

Timestamp

Nonce

Signature
```

SSR

```text
X-internal-Request
```

This is it.

Transport itself is

a Security Layer.

---

# Retry

If 401,

Transport

automatically

```text
guest/init

↓

Retry
```

Business Logic

does not know

about Retry.

---

# Runtime

In the Browser,

```text
/api
```

is called.

In SSR,

```text
runtimeConfig.apiBaseUrl
```

is used.

In other words

depending on the Runtime,

Transport changes.

---

# Production Layer

In Production,

```text
Apache / Nginx

↓

HTTPS Wrapper

↓

Nitro

↓

RouteRules Proxy

↓

Backend
```

This is it.

Streaming is

maintained all the way through.

---

# Environment

routeRules are

created at build time.

In other words

```text
nuxt.config.ts

↓

API_BASE_URL

↓

Proxy
```

This is it.

Therefore

environment variables

must exist

before the build.

---

# Transport Layer Sequence

```text
Browser

↓

streamDealChat()

↓

apiFetchRaw()

↓

fetch()

↓

Nitro routeRules

↓

sendProxy()

↓

Backend

↓

NDJSON

↓

sendProxy()

↓

ReadableStream

↓

TextDecoder

↓

Buffer

↓

Line Parser

↓

JSON Parser

↓

applyEvent()
```

---

## In my view, this is one of the core technologies.

Many people first focus on **RAG** or **ChatOrchestrator**, but in reality, **the Transport Layer is quite well-implemented.**

In particular, the following elements are naturally integrated as a single layer.

* **Nitro `routeRules` based Reverse Proxy**
* **NDJSON Streaming**
* **`ReadableStream` based Chunk processing**
* **Buffer management and line-by-line parsing**
* **Abort propagation**
* **Guest authentication and signature headers**
* **401 automatic retry**
* **SSR/CSR-specific API Endpoint abstraction**

In other words, it is not simply at the level of "streaming chat," but can be evaluated as **a project where Transport itself is designed as an independent architectural layer.**
