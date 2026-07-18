# Your First Support Agent with Gemini Enterprise Agent Platform

> **Category:** AI & Agents · **Level:** Beginner · **Duration:** ~20 min · **Cost:** Free tier

## Exam relevance

The PCA exam increasingly covers conversational AI use cases (customer support automation, internal knowledge assistants). Gemini Enterprise Agent Platform (formerly Vertex AI, and before that Vertex AI Search and Conversation) is Google's managed path for this — you should know its capabilities without writing custom NLU code. Google accepts both the old and new naming in transition-era exam questions, but expect "Gemini Enterprise" going forward.

## Objective

Build a chat agent that grounds its answers in a document and correctly maintains conversational state across follow-up questions.

## Prerequisites

- A GCP project with the Gemini Enterprise Agent Platform (formerly Vertex AI) APIs enabled
- A short FAQ document (PDF) to use as grounding data — [gemini-enterprise-support-agent_FAQ_file.pdf](gemini-enterprise-support-agent_FAQ_file.pdf), a real regulatory FAQ (French AGEC law / Decree 2022-748 on environmental product labeling) included alongside this exercise, works well because it has clearly numbered Q&A sections and cross-references between them — good for testing grounding and follow-up handling
- A GCS bucket in the same project (needed for the import step below) — any bucket works, or let the console create one for you

## Steps

### 1. Open the console

Navigate to **Gemini Enterprise Agent Platform** (console.cloud.google.com → search "Agent Platform" or "Search and Conversation").

### 2. Create a new Chat agent

- Choose **Chat** as the agent type.
- Give it a name, e.g. `pca-support-agent`.
- Select your project region (any region close to you).

### 3. Attach a data source

Data sources live in **Data Stores**, which you create and then attach to the agent — you can't just drag a file onto the agent itself, which is the part that trips people up.

- From the agent's app, go to **Data** (or **Data Stores**) → **Create new data store**.
- Choose **Cloud Storage** as the source type (this is what handles unstructured PDFs).
- You'll see two options: point to an existing bucket/path, or **Upload files** directly from your browser. If you use the direct upload option, the console silently creates a GCS bucket behind the scenes and puts your file there — if you don't see an upload widget in your UI/region, that's why: fall back to uploading `gemini-enterprise-support-agent_FAQ_file.pdf` to a GCS bucket manually first (`gsutil cp` or the Cloud Storage console), then import from that `gs://` path instead.
- Select **Unstructured documents** as the data type when prompted.
- Give the data store a name (e.g. `agec-faq-store`) and create it.
- Back in the agent, attach this data store as the agent's data source.
- Wait for the data source to finish indexing (a few minutes for a single small PDF).

### 4. Test in the simulator

Open the built-in chat simulator and try:

1. A direct question: *"What is the annual turnover threshold for the information obligation since January 2025?"* (answer is in §1.1.2 — €10 million / 10,000 units)
2. A vague follow-up referencing the previous answer: *"And does that apply to a distributor selling under its own brand?"*
3. A conditional/cross-reference question: *"If a product contains 100% recycled material, how should that be stated on the product sheet?"* (answer is in §2.3.1)
4. A question the document does **not** answer, to check the agent doesn't hallucinate: *"What is the fine for late GDPR breach notification?"*

Observe how the agent uses the indexed document to ground factual answers (including pulling specific numbers/thresholds), how it carries context from turn to turn without you repeating the subject, and whether it correctly declines to answer the out-of-scope question in step 4 rather than inventing one.

## Cleanup

Delete the agent and any associated data store from the console to avoid ongoing storage/indexing charges.

## Key takeaways

- Grounding data (documents/playbooks) removes the need to hand-write intents and entities.
- Conversational state is handled by the platform — your job as an architect is choosing the right grounding sources and evaluating groundedness.
- This is the "no-code/low-code" answer to conversational AI on the exam — reach for a custom, code-first agent built with the Agent Development Kit (ADK) only when this managed path can't meet a requirement.
