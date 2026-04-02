# 🎯 TCS NQT Digital Interview Prep — Challa Harshith Reddy

> **Interview Format:** TR (Technical) + MR (Managerial) + HR — Combined Round  
> **Key Project:** Agentic Document Intelligence System

---

## 🗣️ SELF-INTRODUCTION (Memorize This)

> *"Good [morning/afternoon]. My name is Challa Harshith Reddy. I'm a final-year ECE student at Vidya Jyothi Institute of Technology, graduating in 2026.*
>
> *I'm a self-taught AI and ML engineer. While my core branch is ECE, I've spent the past two years building production-grade intelligent systems — specifically around Retrieval-Augmented Generation, multi-agent orchestration, and machine learning pipelines.*
>
> *My most significant project is an Agentic Document Intelligence System — a multimodal RAG system that can parse any document — scanned PDFs, handwritten forms, charts — and answer questions about them with cited, verified answers.*
>
> *I've also built a Credit Card Fraud Detection system handling real imbalanced financial data with XGBoost and SHAP-based explainability, deployed as a REST API.*
>
> *My goal is to join an engineering-driven team where I can design and build real scalable AI applications — which aligns very well with what TCS's AI and ML divisions are working on today."*

---

## 📐 PROJECT ARCHITECTURE EXPLANATION (The "Walk Me Through It" Answer)

When asked *"explain your main project"* — use this flow:

> *"The project solves a real problem: most organizations have thousands of documents — scanned PDFs, certificates, marksheets — that machines cannot search or query intelligently.*
>
> *I split the system into two pipelines:*
>
> ***Ingestion Pipeline (runs offline, async):***
> *When a document is uploaded, a heuristic router using PyMuPDF checks each page. If a page has native text, we extract it directly — very fast. If it's a scanned image, we pass it to GLM-OCR, a 1-billion parameter specialized model that processes about 1.8 pages per second. If the page has charts or complex visuals, only then do we invoke Qwen2.5-VL, our vision-language model — this way we avoid unnecessary heavy model calls on 80% of pages.*
>
> *All outputs are standardized to Markdown. Then spaCy extracts entities — names, student IDs, grades, dates — and we build a Knowledge Graph using NetworkX. A validation function cross-checks data arithmetically before anything is indexed.*
>
> ***Query Pipeline (runs every time a user asks):***
> *An embedding-based query router classifies the question as simple or complex — with zero LLM cost. Simple questions go through Hybrid Search (BM25 + Vector) in Qdrant, re-ranked by BGE-Reranker, and directly to generation via Groq API — under 1.5 seconds.*
>
> *Complex, multi-hop questions go to a LangGraph ReAct agent loop. The agent has two tools: hybrid_search and graph_traversal. It decides which to call, observes the result, and iterates — bounded by a max-iteration guard. Finally, a faithfulness guard verifies the answer is grounded in the retrieved context before returning it to the user."*

---

## 🔧 TECHNICAL ROUND (TR) — 15 Questions

---

### Q1. What is RAG and why did you use it instead of fine-tuning an LLM?

> *"RAG — Retrieval Augmented Generation — is a technique where instead of embedding knowledge into model weights through fine-tuning, you keep knowledge external in a database and retrieve the relevant parts at query time.*
>
> *I chose RAG for three reasons: First, organizational documents change constantly — fine-tuned models become stale. With RAG, I just re-index new documents. Second, fine-tuning requires thousands of labeled examples and GPU compute that isn't always available. Third, RAG is inherently more trustworthy — it cites its sources from the actual documents. The faithfulness guard then verifies the answer is grounded in that source. Fine-tuning gives you no such guarantee."*

---

### Q2. What is the difference between a regular RAG and your "Agentic" RAG?

> *"Standard RAG is a fixed pipeline: receive query → retrieve top-k chunks → generate answer. It has no decision-making capability. If the query requires information from two different document types, it either misses one or returns a poor answer.*
>
> *Agentic RAG wraps RAG inside an agent loop. The agent reasons: 'What do I need to know? What tool should I call? Is what I retrieved enough?' It can call hybrid_search multiple times with different queries, or switch to graph_traversal when it needs entity relationships. This enables multi-hop reasoning — something a fixed pipeline simply cannot do.*
>
> *The trade-off is latency — each hop is an LLM call. That's why I only route complex queries to the agent and send simple queries through the direct path."*

---

### Q3. Explain Hybrid Search. Why use both BM25 and Vector search?

> *"Vector search uses dense embeddings — it understands semantic meaning. If a student's name is 'Arjun' but the query says 'the student who topped CS,' vector search can connect them semantically.*
>
> *BM25 is a keyword-based search algorithm — it finds exact term matches and is very precise when someone searches for a specific roll number, date, or technical term.*
>
> *Neither alone is complete. Vector search misses exact terms; BM25 misses semantic meaning. By fusing both — which Qdrant supports natively — I get the best of both worlds. Qdrant returns a combined score, and then BGE-Reranker re-scores the top results using a cross-encoder model which reads the query and document together for much higher precision."*

---

### Q4. What is a Cross-Encoder Reranker and how is it different from the Embedding model?

> *"An embedding model is a Bi-Encoder. It encodes the query and document separately into vectors, and similarity is just a dot product. It's fast because you pre-compute document vectors, but it loses nuance because query and document never 'see' each other.*
>
> *A Cross-Encoder like BGE-Reranker processes the query AND document together in a single forward pass. It can understand interactions between them — like 'does this paragraph actually answer this specific question?' It's much more accurate but slower because you can't pre-compute it.*
>
> *So my architecture uses two stages: fast embedding search to get top-20 candidates cheaply, then cross-encoder reranking to pick the best 5. Best of both worlds — speed and accuracy."*

---

### Q5. You mentioned 'LangGraph ReAct loop.' What is ReAct?

> *"ReAct stands for Reasoning and Acting. It's a prompting and architecture pattern where the LLM alternates between three steps: Think — Act — Observe.*
>
> *Think: the LLM reasons about what it knows and decides what to do next. Act: it calls a tool — in my case, either hybrid_search or graph_traversal. Observe: it reads the tool's output and updates its understanding.*
>
> *This loop continues until the LLM decides it has enough information. LangGraph implements this as a stateful graph — each node in the graph represents a step, and the edges represent conditions. I added a max-iteration guard so the loop doesn't run more than 3-4 cycles, which prevents infinite loops in production."*

---

### Q6. Why did you use Groq API instead of OpenAI or a local model?

> *"Two reasons — speed and cost profile.*
>
> *Groq uses custom LPU hardware — Language Processing Units — designed specifically for sequential token generation. They achieve around 300 tokens per second on LLaMA and Gemma models. Standard GPU-based APIs average around 50 tokens per second.*
>
> *For an agentic system where the LLM is called multiple times per query in the ReAct loop, this difference is critical — it's the difference between a 4-second response and a 1.5-second response.*
>
> *Local models would require dedicating a powerful GPU, which isn't practical for a project I need to demonstrate reliably. Groq gives me near-local speeds at API convenience."*

---

### Q7. Explain your FastAPI design. What endpoints does your system have?

> *"FastAPI is my choice because it has native async support which is critical when you have operations like Qdrant search, Redis cache lookup, and Groq API calls happening concurrently.*
>
> *My main endpoints are:*
> - *`POST /ingest` — accepts document upload, immediately returns a task ID and pushes the parsing job to Celery queue. Non-blocking.*
> - *`GET /ingest/status/{task_id}` — polls Celery to check ingestion progress.*
> - *`POST /query` — accepts the user question, runs the adaptive query pipeline, and streams the response back via Server-Sent Events.*
> - *`GET /health` — basic health check.*
>
> *The streaming endpoint is important — instead of waiting for the full response, the user sees the first token in about 200 milliseconds, which feels instant even if the full answer takes 1.5 seconds."*

---

### Q8. Why Celery + Redis for document ingestion? Why not just process it in the API request?

> *"A 200-page scanned PDF takes about 1.8 minutes to process through GLM-OCR plus embedding plus indexing. An HTTP request will time out after 30-60 seconds. If processing happens inside the API call, the user sees a failed request.*
>
> *Celery is a distributed task queue. When a document is uploaded, the API immediately pushes a task to Redis (acting as the message broker) and returns a task ID. A Celery worker process picks up the task and processes it in the background — completely independent of the HTTP connection. The user can poll for status or get a webhook notification when it's done.*
>
> *This pattern is called async task processing. It's standard in any production system where you have long-running operations — email sending, video processing, ML inference on large files."*

---

### Q9. How does your Redis Semantic Cache work?

> *"A regular cache stores exact key-value pairs. If a user asks 'What is Arjun's GPA?' and the cached key is 'What is Arjun Kumar's GPA?' — it's a cache miss even though it's the same question.*
>
> *A semantic cache stores the embedding vector of each answered query alongside the answer. When a new query comes in, I embed it and check cosine similarity against all cached query embeddings. If the similarity is above 0.92, I return the cached answer directly — no RAG pipeline, no LLM call.*
>
> *This reduces latency from 1.5 seconds to under 50 milliseconds for repeated or paraphrased questions. At 10,000 queries per day, if 30% are semantic duplicates, this saves significant compute cost — roughly $6-8 per day at commercial API rates."*

---

### Q10. What is Qdrant and why did you choose it over FAISS or ChromaDB?

> *"All three are vector databases but they differ significantly in production-readiness.*
>
> *FAISS is a library — it's in-memory, has no persistence without custom code, and no built-in filtering by metadata. It's great for research experiments.*
>
> *ChromaDB is developer-friendly but is primarily an embedded database — it doesn't scale well to distributed deployments.*
>
> *Qdrant is a production vector database — it's written in Rust so it's very fast, it supports hybrid search natively (BM25 + Vector together), has filterable metadata, supports on-disk indexing for large corpora, and can be deployed as a microservice. For a system that needs to handle 10,000+ documents with metadata filtering — like 'search only within CS department documents' — Qdrant is the right choice."*

---

### Q11. What is a Knowledge Graph and why did you build one on top of vector search?

> *"A knowledge graph stores entities and their relationships as nodes and edges. For example: Student 'Arjun Kumar' → enrolled_in → 'Computer Science' → has_grade → '8.7 CGPA' → has_internship → 'Infosys 2024'.*
>
> *Vector search is excellent at answering 'what does this document say about X?' But it struggles with questions that require joining information across entities — like 'show me all CS students with GPA above 8 who also have an internship certificate.'*
>
> *That query requires knowing all CS students, filtering by GPA, then checking internship existence — that's a multi-hop graph traversal. Vector search would return chunks that mention these things individually but cannot reliably join them.*
>
> *The knowledge graph built with NetworkX is exposed as a ReAct agent tool. The agent decides — sometimes it needs the vector search, sometimes it needs the graph, sometimes both in sequence. This hybrid is what makes the system genuinely intelligent."*

---

### Q12. Three questions about the Fraud Detection project — XGBoost, SMOTE, and SHAP.

**XGBoost:**
> *"XGBoost is a gradient boosting algorithm. It builds an ensemble of decision trees sequentially — each tree corrects the errors of the previous one. For tabular financial data with features like transaction amount, merchant category, and time patterns, tree-based models consistently outperform neural networks because of how they handle feature interactions. I chose XGBoost over Random Forest because XGBoost's L1/L2 regularization prevents overfitting on the limited fraud samples."*

**SMOTE:**
> *"SMOTE — Synthetic Minority Oversampling Technique — generates synthetic examples of the minority class (fraud, which was 0.17% of data) by interpolating between existing fraud samples in feature space. Critical rule: I applied SMOTE exclusively on the training split AFTER doing the train-test split. Applying it before splitting would leak synthetic samples into the test set, making test accuracy artificially inflated — a real data leakage bug."*

**SHAP:**
> *"SHAP — SHapley Additive exPlanations — uses game theory to assign each feature an importance value for each individual prediction. It tells you not just 'V14 is important overall' but 'for this specific transaction, V14 contributed -0.7 to the fraud score.' For a financial compliance use case, this is legally important — you need to explain why a transaction was flagged, not just that it was flagged."*

---

### Q13. What is the difference between REST API and GraphQL? Which would you choose for this project and why?

> *"REST uses fixed endpoints — each URL corresponds to a resource and HTTP verbs (GET, POST, PUT, DELETE). GraphQL uses a single endpoint where the client specifies exactly what data it wants in the query itself.*
>
> *For my document intelligence system, I chose REST because the data contract is well-defined and simple — upload a document, query with a question, get a response. REST is also better for streaming responses (SSE), which I use for live answer streaming.*
>
> *GraphQL makes more sense when you have complex, nested data models where clients need different subsets of data — like a social media platform. For an AI inference API, REST is simpler, better supported by all HTTP clients, and easier to document."*

---

### Q14. What database would you use if you needed to store user sessions and conversation history?

> *"For conversation history, I'd use PostgreSQL — a relational database. Each conversation session would have a session_id, and messages would be linked to that session with timestamps, role (user/assistant), and the message content. PostgreSQL's JSONB column type is excellent for storing flexible message metadata.*
>
> *Redis would cache the last N messages of active sessions for hot retrieval — fetching conversation context from Redis takes under 1ms versus 20-50ms from PostgreSQL. Cold/archived sessions can be queried from PostgreSQL.*
>
> *This two-tier pattern — Redis for hot data, PostgreSQL for persistent storage — is standard in production chat applications."*

---

### Q15. What happens if GLM-OCR fails mid-ingestion for a 100-page document?

> *"This is a fault tolerance question. In my Celery task design, I process documents in paginated batches — say, 10 pages per sub-task. If GLM-OCR fails on pages 40-50, only that batch fails, not the entire document.*
>
> *Celery has built-in retry logic — I configure `max_retries=3` with exponential backoff. If all retries fail, the task is marked failed and the user is notified with which pages failed.*
>
> *The pages that did succeed are indexed and queryable. Failed pages are logged with their page numbers and the user can re-submit just those pages. This is called partial failure isolation — you don't discard successful work because one part failed."*

---

## 🧠 MANAGERIAL ROUND (MR) — 6 Questions

---

### Q16. Why this project? What problem motivated you to build it?

> *"I was looking at how colleges and companies manage documents — transcripts, offer letters, certificates — and the workflow is completely manual. An HR person spends hours searching through folders to verify a candidate's credentials. A student affairs office manually checks marksheets to calculate eligibility.*
>
> *I realized that every piece of information is locked inside PDFs that machines can't query intelligently. A simple question like 'show me all final-year ECE students with above 8 CGPA and an internship' requires opening dozens of files manually.*
>
> *That's the problem I set out to solve — making organizational knowledge instantly queryable through a conversational interface, with citations so you can trust the answer."*

---

### Q17. How is your project different from just using ChatGPT with document upload?

> *"Three fundamental differences.*
>
> *First, ChatGPT's document upload has a context window limit — typically 128K tokens. A 200-page PDF exceeds that. My system chunks and indexes the document so you can query any corpus of any size — 10,000 documents, not just one.*
>
> *Second, ChatGPT is a black box — it doesn't cite specific page sources. My system returns exact chunk references and a faithfulness guard verifies every answer. For institutional use, you need to trace every claim.*
>
> *Third, ChatGPT cannot do cross-document entity queries using a knowledge graph. Asking 'compare all CS students with internships' would require it to hold all documents in context simultaneously — impractical. My knowledge graph handles this as a graph traversal."*

---

### Q18. What was the hardest technical challenge you faced building this?

> *"The hardest challenge was the 'Markdown Contract' — ensuring that three completely different parsers (PyMuPDF native, GLM-OCR, and Qwen2.5-VL) all output structured, consistent Markdown.*
>
> *Like headers, tables, and section breaks must appear in the same format regardless of which parser processed the page. If they don't, the chunker splits incorrectly, chunks straddle page boundaries, and retrieval breaks.*
>
> *I solved it by writing a post-processing normalization layer that maps each parser's output format to a canonical Markdown structure — this became what I call the 'Markdown Contract.' Every downstream component from chunker to embedder only sees the normalized output. This is essentially the same principle as an interface in software — hide the implementation details behind a contract."*

---

### Q19. If you had 3 more months, what would you improve?

> *"Three concrete improvements:*
>
> *One — evaluation pipeline. I'd integrate Ragas to continuously measure Faithfulness, Context Recall, and Answer Relevancy on a test set of known questions. Right now I evaluate qualitatively; in production you need automated regression testing for every model or prompt change.*
>
> *Two — persistent user memory with Mem0. Currently the system has no memory between sessions. A returning user re-uploads the same documents. With Mem0, the agent would remember 'this user's transcript is indexed, their GPA is 8.7, they prefer detailed explanations.' Personalization dramatically improves experience.*
>
> *Three — multi-document reasoning. Currently complex queries require the knowledge graph. I'd add a summarization agent that, for very broad questions, does a map-reduce across all relevant documents — summarize each separately, then synthesize — instead of relying solely on chunk retrieval."*

---

### Q20. How would you handle a situation where the OCR makes a mistake and extracts wrong data?

> *"Prevention and detection at three levels.*
>
> *First, at ingestion — the validation function does arithmetic cross-checks. If the sum of subject marks doesn't equal the total, the page is flagged before indexing. Wrong data doesn't enter the knowledge base.*
>
> *Second, at retrieval — when the faithfulness guard checks the answer, it compares against the raw retrieved chunks. If the answer states a GPA of 8.7 but the chunk shows 8.2, the guard flags it.*
>
> *Third, at the product level — every answer comes with a source citation including the page number. The user can verify against the original document at any time. The system is designed to aid decision-making, not replace human verification for critical decisions.*
>
> *No AI system is perfect. The honest approach is building layers of checks and making errors visible rather than hiding them."*

---

### Q21. How do you stay updated with AI developments given how fast the field moves?

> *"I follow a structured approach. Papers through Hugging Face's daily paper digest and ArXiv — specifically the cs.AI and cs.CL sections. Practical implementations through LangChain and LangGraph's official changelogs and GitHub releases. Community insights through specific Twitter/X accounts of researchers at Anthropic, DeepMind, and Meta.*
>
> *But the most important habit is building — not just reading. For this project, I specifically tracked the release of Qwen2.5-VL, Qwen3.5, and GLM-OCR and benchmarked them against each other for our specific use case. Reading a blog post gives you awareness; actually running models on your data gives you understanding.*
>
> *For TCS's scale, I'd also specifically follow enterprise AI deployment patterns — not just research papers."*

---

## 👤 HR ROUND — 4 Questions

---

### Q22. Tell me about yourself (HR version — shorter, personality-focused).

> *"I'm Harshith, a final-year ECE student who taught himself AI and ML alongside my core engineering studies. I'm driven by building things that solve real problems — not just running model tutorials. My Agentic RAG project started because I genuinely wanted to know if I could build a system that makes organizational documents intelligent.*
>
> *I'm someone who goes deep — when I work on something, I want to understand every layer, from why a specific model is faster to how to prevent hallucinations. I believe the best engineers know not just *how* but *why*.*
>
> *I'm looking to join TCS Digital because it's where I can apply this engineering mindset at real scale — working on AI systems that impact thousands of users, not just demos."*

---

### Q23. Why TCS? Why not a startup?

> *"Two honest reasons.*
>
> *First, scale. The problems TCS Digital works on — enterprise AI, large-scale data pipelines, mission-critical systems — operate at a scale that a startup in my domain rarely touches in the first few years. I want to learn production AI at real enterprise scale, and TCS is one of the few places where a fresh graduate can get that exposure.*
>
> *Second, breadth. TCS works across industries — finance, healthcare, logistics, government. For someone building general-purpose AI systems, that breadth of domain exposure in the first few years of a career is invaluable.*
>
> *That said, I'm not against startups long-term. But right now, I want to build a strong foundation in large-scale systems engineering, and TCS Digital is the right place for that."*

---

### Q24. What is your biggest weakness?

> *"I tend to over-engineer solutions. When I was designing the agent loop for this project, I initially planned eight specialized agents in parallel before stepping back and realizing most of those could be simple Python functions.*
>
> *I've learned to actively ask myself: 'Does this need an agent or is this just a function?' That discipline has made me a much more practical engineer.*
>
> *I still need to improve on the time-estimation side — going deep into problems sometimes means I underestimate how long a well-engineered solution takes. I've started breaking work into smaller deliverable phases to address this."*

---

### Q25. Where do you see yourself in 5 years?

> *"In 5 years, I see myself as a senior AI systems engineer who has shipped production AI systems that real users depend on. I want to have gone from building prototypes to owning the architecture of systems that handle millions of requests.*
>
> *More specifically, I'm interested in the intersection of AI reliability and scale — how do you build systems that are not just accurate in demos but consistently accurate under real-world production load? That requires deep engineering, not just ML knowledge.*
>
> *I'd also like to mentor junior engineers — I've learned enormously from online communities and want to give that back."*

---

## ⚡ RAPID-FIRE QUESTIONS (Quick concept checks — common in TCS NQT Digital)

| Question | Best Answer |
|:---|:---|
| What is a vector embedding? | A numerical representation of text in high-dimensional space where semantically similar texts have similar vectors |
| What is cosine similarity? | A measure of the angle between two vectors — 1 means identical direction, 0 means orthogonal, used to find semantic similarity |
| What is an API? | Application Programming Interface — a contract that allows two software systems to communicate |
| What is Docker? | Containerization tool that packages an application with all its dependencies so it runs identically on any machine |
| What is the difference between SQL and NoSQL? | SQL: structured, relational, ACID compliant. NoSQL: flexible schema, horizontally scalable, for unstructured data |
| What is overfitting? | When a model learns the training data too well including its noise, and performs poorly on new unseen data |
| What is Git? | Distributed version control system — tracks changes in code, allows collaboration and reverting to previous states |
| What is async programming? | Executing tasks concurrently without waiting for each to finish — allows a program to handle many tasks simultaneously |
| What is a load balancer? | Distributes incoming network requests across multiple servers to prevent any single server from being overwhelmed |
| What is OCR? | Optical Character Recognition — technology that converts images of text into machine-readable text |

---

## 🏆 CRITICAL MENTAL MODEL FOR THE INTERVIEW

Remember these three sentences when you get confused:

1. **On architecture:** *"Every step in my pipeline is either a deterministic Python function or a justified LLM call. I only call an LLM when there is no cheaper way to do the same task."*

2. **On your edge:** *"Most RAG demos work on 50 documents and don't handle scanned or visual documents. My system handles 10,000+ documents including scanned pages, charts, and tables — with a hybrid parser, Knowledge Graph, and validation layer."*

3. **On production mindset:** *"I designed for failures — async ingestion with retries, max-iteration guards on the agent loop, faithfulness checking before every answer. The system assumes things will go wrong and handles them gracefully."*

---

*Good luck, Harshith. You built something real. Now just explain it exactly as it is. 🚀*
