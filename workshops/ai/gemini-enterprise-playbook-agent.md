# Your First Playbook Agent with Gemini Enterprise Agent Platform

> **Category:** AI & Agents · **Level:** Beginner · **Duration:** ~10 min · **Cost:** Free tier

## Exam relevance

The PCA exam distinguishes between grounded agents (backed by documents/data stores) and instruction-only agents (backed by a **Playbook** — a set of goals and steps written in natural language). You should recognize when a Playbook alone is sufficient and when you need to attach a data store for grounding. This exercise covers the Playbook-only path, with zero file uploads.

## Objective

Build a chat agent whose behavior is defined entirely by written instructions (a Playbook), with no external document to ground it, and see how it maintains conversational state across turns.

## Prerequisites

- A GCP project with the Gemini Enterprise Agent Platform (formerly Vertex AI) APIs enabled

That's it — no PDF, no Cloud Storage bucket, no data store.

## Steps

### 1. Open the console

Navigate to **Gemini Enterprise Agent Platform** (console.cloud.google.com → search "Agent Platform" or "Search and Conversation").

### 2. Create a new Chat agent

- Choose **Chat** as the agent type.
- Give it a name, e.g. `pca-playbook-agent`.
- Select your project region (any region close to you).

### 3. Write a Playbook instead of attaching data

- In the agent's **Playbooks** section, open the default playbook (or create a new one).
- Set the **goal**, e.g.: *"Help the user plan a one-day city trip. Ask for their destination city and interests, then suggest a simple morning/afternoon/evening itinerary."*
- Add a couple of **instructions** (steps), e.g.:
  1. Ask which city the user is visiting and what they're interested in (food, history, nature, etc.).
  2. Propose a 3-part itinerary (morning/afternoon/evening) tailored to their answer.
  3. If the user asks to swap one part of the itinerary, adjust only that part and keep the rest.
- Save the playbook. No indexing wait, no upload — the agent is ready immediately.

### 4. Test in the simulator

Open the built-in chat simulator and try:

1. An opener: *"I want to plan a day trip."*
2. Answer its follow-up question, e.g.: *"Lyon, and I like food and history."*
3. A follow-up adjustment: *"Can you swap the afternoon for something more relaxed?"*

Observe how the agent follows your written steps in order, asks clarifying questions before proposing an itinerary, and keeps the rest of the plan intact when you ask for a partial change.

## Cleanup

Delete the agent from the console — there's no data store or bucket to clean up since none was created.

## Key takeaways

- A **Playbook** defines agent behavior with plain-language goals and steps — no grounding document, indexing, or file upload required.
- Conversational state (remembering the city, the interests, the current itinerary) is handled by the platform even without a data store.
- Reach for a data store (see the companion exercise, `gemini-enterprise-support-agent.md`) only when answers must be grounded in specific facts from a document; use a Playbook alone when the agent just needs to follow a defined conversational flow.
