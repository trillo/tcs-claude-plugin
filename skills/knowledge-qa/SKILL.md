---
description: Query a deployed app's knowledge base — semantic search over its documents, or a grounded, cited answer to a question. Use when the user asks something that should be answered from the app's uploaded/ingested documents, or wants to find relevant passages.
---

# Querying the knowledge base (RAG)

A Trillo app can have **knowledge containers** with ingested documents (chunked
+ embedded). Two tools query them, both scoped three ways — one document, one
container, or all containers in the app:

- **`knowledge_search`** — semantic **retrieval**: returns the most relevant
  chunks `{chunkId, documentName, containerName, fileId, chunkIndex, text,
  score}`. Use when you want the raw passages.
- **`knowledge_ask`** — **grounded answer**: retrieves, then generates an answer
  strictly from those chunks, with citations. Returns `{answer, found,
  confidence, sources: [{documentName, containerName, fileId, chunkIndex,
  snippet}]}`. `found` is false when the documents don't cover the question.

## How to use

- Pass `appId`. **Scope** with `fileId` (one document) or `containerId` (one
  container); pass neither to search the whole app's knowledge base. `k`
  (default 8) caps how many chunks are retrieved.
- For a question the user wants *answered*, use **`knowledge_ask`** and relay
  the answer **with its sources** (document names) so they can trace it.
- For "find / show me the parts about X", use **`knowledge_search`**.

## Requirements

- These are AOS-touching tools (in the **activities** group) — the app must be
  **deployed** (`deployStatus == "deployed"`), and its containers must have been
  **ingested** (documents uploaded + processed). Otherwise you'll get
  `APP_NOT_DEPLOYED` or empty results.
- The answer is grounded only in the app's documents — if `found` is false, say
  so plainly rather than answering from general knowledge.
