# AI: LLMs, RAG, Agents, Spring Integration

## Concepts (simple)
```
AI ⊃ Machine Learning ⊃ Deep Learning ⊃ Large Language Models (LLMs)
```
- **LLM:** predicts the next **token** (word piece) using patterns learned from huge text. Knows what it was trained on; can be wrong confidently (**hallucination**).
- **Prompt** = instructions/context you send. **Context window** = max tokens the model can see at once. **Temperature** = randomness (low = focused).
- **Prompt engineering:** be specific, give role, format, examples (few-shot), constraints.
- **Embedding:** text → list of numbers where similar meaning = close vectors. Stored in a **vector database** (pgvector, Pinecone, OpenSearch).

## RAG (Retrieval-Augmented Generation) ⭐ — most common AI feature in companies
Problem: LLM doesn't know your private documents (policies, product docs) and may hallucinate.
```
INGEST (offline):  documents → split into chunks → embed each chunk → store in vector DB
ANSWER (online):
   User question ─► embed question ─► vector search top-k similar chunks
        ─► prompt = "Answer ONLY from this context: {chunks}\nQuestion: {q}" ─► LLM ─► answer + sources
```
Improvements: good chunking, hybrid (keyword+vector) search, re-ranking, citations, evaluation set, guardrails.

## Calling an LLM from Spring Boot (example: "order assistant")
```java
@Service
class SupportAssistant {
    private final ChatClient chat;                       // Spring AI (or any provider SDK / REST call)
    SupportAssistant(ChatClient.Builder b) { this.chat = b.defaultSystem(
        "You are a support agent for ShopX. Use only the provided order data. If unsure, say you don't know.").build(); }

    String answer(String question, Order order) {
        return chat.prompt()
            .user(u -> u.text("Order: {order}\nQuestion: {q}").param("order", order.summary()).param("q", question))
            .call().content();
    }
}
```
Production rules: **timeouts + retries + circuit breaker** (LLM APIs are slow/flaky), cache repeated answers, **never send secrets/PII** unless policy allows, log tokens/cost, validate/parse structured output (JSON schema), keep API keys in Secrets Manager, rate-limit users, human handoff for risky actions.

## Agents, tools, MCP
- **Tool/function calling:** LLM asks *your code* to run a function (`getOrderStatus(id)`), you run it and return the result → LLM answers with real data.
- **Agent:** loop of *think → call tool → observe → repeat* until goal done.
- **MCP (Model Context Protocol):** standard way to plug tools/data sources into AI assistants.
```
User: "Where is my order 42?"
 → LLM decides tool getOrderStatus(42) → your service returns {status: SHIPPED}
 → LLM: "Order 42 has shipped."
```
Safety: least-privilege tools, confirmation for destructive actions, prompt-injection defence (treat retrieved text as data, not instructions).

## AI use cases in this e-commerce system
Recommendations (ML), fraud detection (consuming Kafka events), search (embeddings), chatbot (RAG), invoice OCR, demand forecasting, log anomaly detection, code assistants for developers (see `SDLC_and_AI_SDLC.md`).
Traditional ML flow: `data → clean → features → train → evaluate → deploy as service → monitor drift`. Metrics: accuracy, precision, recall, F1. Overfitting vs underfitting. Java stack: call Python model services over REST/gRPC, or ONNX runtime / DJL; Spring AI for LLM integration.

## Responsible AI
Bias, privacy, explainability, hallucination checks, human-in-the-loop, audit logs, compliance (GDPR/DPDP).
