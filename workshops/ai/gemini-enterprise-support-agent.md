# Your First Support Agent with Gemini Enterprise Agent Platform

> **Category:** AI & Agents · **Level:** Beginner · **Duration:** ~20 min · **Cost:** Free tier

## Exam relevance

The PCA exam increasingly covers conversational AI use cases (customer support automation, internal knowledge assistants). Gemini Enterprise Agent Platform (formerly Vertex AI, and before that Vertex AI Search and Conversation) is Google's managed path for this — you should know its capabilities without writing custom NLU code. Google accepts both the old and new naming in transition-era exam questions, but expect "Gemini Enterprise" going forward.

## Objective

Build a chat agent that grounds its answers in a document and correctly maintains conversational state across follow-up questions.

## Prerequisites

- A GCP project with the Gemini Enterprise Agent Platform (formerly Vertex AI) APIs enabled
- A short FAQ document (PDF) to use as grounding data — a fake "product support FAQ" works fine

## Steps

### 1. Open the console

Navigate to **Gemini Enterprise Agent Platform** (console.cloud.google.com → search "Agent Platform" or "Search and Conversation").

### 2. Create a new Chat agent

- Choose **Chat** as the agent type.
- Give it a name, e.g. `pca-support-agent`.
- Select your project region (any region close to you).

### 3. Attach a data source

- Choose **Playbook** for scripted behavior, or upload a **PDF document** as unstructured data.
- If uploading a PDF, create a small fake FAQ first, e.g.:
  - "How do I reset my password?"
  - "What is your refund policy?"
  - "How do I cancel a subscription?"
- Wait for the data source to finish indexing.

### 4. Test in the simulator

Open the built-in chat simulator and try:

1. A direct question: *"How do I reset my password?"*
2. A vague follow-up referencing the previous answer: *"And what if I don't have access to my email anymore?"*
3. A conditional: *"What happens if I do that after my subscription already expired?"*

Observe how the agent uses the indexed document to ground factual answers, and how it carries context from turn to turn without you repeating the subject.

## Cleanup

Delete the agent and any associated data store from the console to avoid ongoing storage/indexing charges.

## Key takeaways

- Grounding data (documents/playbooks) removes the need to hand-write intents and entities.
- Conversational state is handled by the platform — your job as an architect is choosing the right grounding sources and evaluating groundedness.
- This is the "no-code/low-code" answer to conversational AI on the exam — reach for a custom, code-first agent built with the Agent Development Kit (ADK) only when this managed path can't meet a requirement.
