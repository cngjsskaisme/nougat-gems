**"Stateless AI Chat + Client State Machine + End-to-End Encryption"** structure.

The overall implementation is very simple, but the responsibilities of each layer are relatively clearly divided.

---

# Overall Structure

```text
                    Browser (Nuxt)

┌─────────────────────────────────────────────┐
│ UI Components                               │
│ DesktopChat / MobileChat                    │
│ MessageBubble                               │
│ ChatMessageActionCard                       │
└─────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────┐
│ Chat State Layer                            │
│ useChatState                                │
│ Inquiry StateMachine                        │
│ Message Store                               │
└─────────────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────┐
│ API Layer                                   │
│ askGemini()                                 │
│ sendMessage()                               │
│ RSA/AES Encryption                          │
└─────────────────────────────────────────────┘
                    │
                    ▼
──────────────────────────────────────────────────────────
                    HTTP
──────────────────────────────────────────────────────────
                    │
                    ▼
┌─────────────────────────────────────────────┐
│ Bun Chat Server                             │
│ RSA                                         │
│ AES                                         │
│ Chat API                                    │
│ Telegram API                                │
└─────────────────────────────────────────────┘
                    │
                    ▼
           GPT / Gemini / Mercury
```

---

# FE Architecture

The FE consists of **6 Layers**.

```text
UI

↓

Chat State

↓

Context Builder

↓

API Client

↓

Encryption

↓

Nuxt API
```

---

# 1. UI Layer

This is the actual user interface.

```
DesktopChat

MobileChat

MessageBubble

ActionCard

FloatingPromptBar
```

The role is simple.

```
Input

↓

Output

↓

State reflection
```

There is almost no business logic.

---

# 2. Chat State Layer

This is the core.

All state is

managed inside

```
useChatState
```

The states it holds are

```text
messages

loading

desktopOpen

mobileOpen

draft

inquiryState
```

among others.

In other words

```text
Application State
```

---

# Inquiry State Machine

This is the biggest feature of Resume.

```text
idle

↓

editing

↓

readyToSend

↓

sending

↓

sent
```

The inquiry feature is

all handled by this State Machine.

---

# Message Store

Messages are

a single in-memory array.

```ts
messages = [
 user,
 assistant,
 user,
 assistant
]
```

There is no DB.

It resets on refresh.

---

# Context Builder

Before sending to the LLM,

it converts the Conversation into a string.

```
handleContextMessageTextArray()
```

Role

```text
messages

+

inquiryState

+

draft

↓

Prompt Text
```

This is it.

In other words

it is Conversation Serialization.

---

# API Layer

This is the actual API call.

The representative function is

```
askGemini()
```

This function calls

```
/api/public-key

/api/chat
```

---

# Encryption Layer

This is a feature of Resume.

The order is

```text
GET PublicKey

↓

Generate AES Key

↓

RSA Encrypt AES

↓

AES Encrypt Prompt

↓

POST /chat
```

This is it.

The response is also

decrypted with AES.

In other words

inside the Application Layer,

a Crypto Layer exists.

---

# Response Layer

The LLM response is

```json
{
 "answerType":"",
 "payload":{}
}
```

This is it.

In the normalize stage,

```
normalizeChatResponse()
```

performs

Validation.

---

# UI Dispatcher

Depending on answerType,

the UI changes.

For example

```text
plainMarkdown

↓

Markdown
```

---

```text
promptTextMessage

↓

Action Card
```

---

```text
confirmSendTextMessage

↓

Send Inquiry
```

This is it.

---

# Nuxt API Layer

The FE has

a thin API Proxy.

```
/api/public-key

/api/chat

/api/send-message
```

Each one

simply proxies

the Backend.

For example

```
Browser

↓

Nuxt

↓

Backend
```

This is it.

---

# Backend Architecture

The Backend also

has Layers.

```text
HTTP

↓

Decrypt

↓

Prompt

↓

LLM

↓

Encrypt

↓

Return
```

It is very simple.

---

# HTTP Layer

It is a single

```
serve()
```

from Bun.

Routing is also

done directly.

```
/chat

/public-key

/send-message

/debug/chat
```

---

# Crypto Layer

At server startup,

```text
RSA 4096

↓

Key Pair
```

is generated.

When the client

sends an AES Key,

```text
RSA Decrypt

↓

AES Session
```

it becomes this.

---

# Chat Layer

The /chat request has

a clear order.

```text
Decrypt

↓

Prompt

↓

LLM

↓

Encrypt

↓

Return
```

It is Stateless.

---

# Prompt Builder

The Backend has

a massive Prompt Builder.

The order is

```text
Rules

↓

Portfolio

↓

Decision Tree

↓

Conversation

↓

Output Schema
```

This is it.

The Prompt is

almost the Business Logic.

---

# LLM Gateway

It abstracts

Vendors.

```text
GPT

Gemini

Mercury
```

All are called through

```
callGPT()
```

---

# Output Contract

The LLM must

always return

```json
{
"answerType":"",
"payload":{}
}
```

In other words

the LLM

follows the API Contract.

---

# Telegram Layer

Inquiries are

```text
Decrypt

↓

Telegram Format

↓

Telegram API
```

This is it.

Without DB storage,

it is delivered immediately.

---

# Overall Sequence

```text
User

↓

DesktopChat

↓

useChatState

↓

handleContextMessageTextArray

↓

askGemini

↓

RSA Encrypt

↓

Nuxt API

↓

Backend

↓

AES Decrypt

↓

Prompt Builder

↓

LLM

↓

JSON

↓

AES Encrypt

↓

Browser

↓

normalize

↓

answerType

↓

UI
```

---

# Technical Characteristics

## 1. Stateless Backend

The Backend does not

save Conversations.

All requests are

independent.

Advantages

* Scalability
* Simplicity

Disadvantages

* Disadvantageous for Long Conversations

---

## 2. FE State Machine

In Resume,

the Frontend manages

State more than

the Backend.

```
idle

editing

ready

sending

sent
```

This is the core.

---

## 3. Prompt-driven Workflow

In Resume,

the Workflow is

in the Prompt,

not in the code.

For example

```
Collaboration inquiry

↓

LLM

↓

promptTextMessage
```

This is it.

In other words

the LLM

acts as a Workflow Engine.

---

## 4. Contract-based Response

The LLM does not

generate free-form Markdown.

It must always

generate

```json
{
"answerType":"..."
}
```

The FE

only trusts this Contract.

---

## 5. Client-side Encryption

The Prompt is

encrypted with AES

in the Browser.

In other words

```
Browser

↓

AES

↓

Network

↓

Server
```

This is it.

The LLM Prompt itself

does not travel in plaintext.

---

# Technical Advantages

### Very Simple Structure

```text
FE

↓

BE

↓

LLM
```

Since it is this simple,

debugging is easy.

---

### FE-centric Structure

State is

all gathered in one place (useChatState).

---

### Easy Vendor Replacement

Since the LLM Gateway is

abstracted,

GPT

↓

Gemini

↓

Mercury

replacement is easy.

---

### Security

By applying RSA + AES,

the Prompt and inquiry content are encrypted during transmission.

---

# Limitations of the Current Structure

The biggest technical limitation is **that a significant portion of the Business Logic is included in the Prompt**.

In the current structure, the LLM is responsible for the following roles.

* Determining collaboration intent
* Determining the inquiry input stage
* Determining whether to approve transmission
* Selecting `answerType`

In other words, the Prompt rather than the code is the center of state transitions.

Additionally, since the backend is completely Stateless, the following features are fundamentally difficult.

* Long-term conversation memory
* Conversation restoration
* Conversation summarization
* RAG-based search
* Streaming UI
* Structured event-based rendering

On the other hand, thanks to this simplicity, it is a very suitable structure for the purpose of **an AI chatbot for a portfolio site**. The low implementation complexity, easy maintenance, and the fact that it fulfills core requirements including encryption and inquiry workflow with a relatively small codebase are the biggest advantages of this architecture.
