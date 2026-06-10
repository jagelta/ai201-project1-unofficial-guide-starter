# Project Planning: Pentesting & Red Teaming Unofficial Guide

## Domain

This system covers penetration testing and red teaming techniques drawn from 10 published textbooks. This knowledge is valuable for security students and practitioners but is scattered across hundreds of pages with no unified way to query specific techniques, tools, or concepts. A RAG system lets users ask targeted questions and get grounded answers with citations back to the source book and chapter, making it far faster to look up a specific attack or methodology than manually skimming multiple books.

## Documents

10 pentesting/red teaming textbooks in PDF format (digital, text-selectable). Each chapter will be treated as a separate document to keep semantic units coherent and sources attributable at the chapter level. This approach yields well over 10 documents and gives each embedded chunk a precise, meaningful source label (e.g. "Book Title — Chapter 3: Privilege Escalation").

Document list will be finalized and added here once chapter-level metadata is extracted from each PDF.

## Chunking Strategy

**Chunk size:** 500 tokens  
**Overlap:** 100 tokens  
**Method:** Fixed token count with overlap, using LangChain's `RecursiveCharacterTextSplitter`

**Rationale:** Pentesting textbooks explain technical concepts (e.g. Kerberos authentication flow, C2 profile structure) over multiple paragraphs. A 500-token chunk is large enough to capture a complete explanation without merging unrelated topics from different sections. The 100-token overlap ensures that concepts spanning a chunk boundary — where the last sentence of one chunk introduces an idea continued in the next — are still retrievable from either chunk. Fixed-size chunking is preferred over paragraph splitting here because paragraph lengths vary wildly in technical books (some are one sentence, others are a full page), which would produce inconsistent chunk sizes and unpredictable retrieval behavior.

## Retrieval Approach

**Embedding model:** `all-MiniLM-L6-v2` via `sentence-transformers` (runs locally, no API key required)  
**Vector store:** ChromaDB (local)  
**Top-k:** 5

**Why k=5:** Technical questions about attacks or tools often require context from several adjacent chunks. Retrieving 5 gives the LLM enough material to synthesize a complete answer without flooding the context window with loosely related content.

**Production tradeoffs:** For a real deployment, I would consider:
- **OpenAI `text-embedding-3-small`** or **Cohere `embed-english-v3`** for higher accuracy on domain-specific terminology at low cost
- **Context length:** `all-MiniLM-L6-v2` is limited to 256 tokens; longer chunks would need a model like `text-embedding-3-large` (8191 tokens) to avoid truncation
- **Multilingual support:** Not a concern for this corpus, but would matter for a broader security knowledge base
- **Latency:** Local models add no API latency but are slower on CPU; API-based models are faster in practice for large query volumes

## Evaluation Plan

| # | Question | Expected correct answer |
|---|----------|------------------------|
| 1 | How does a red team engagement work? | Should describe scoping, rules of engagement, recon, initial access, lateral movement, post-exploitation, and reporting phases |
| 2 | How does Kerberos work? | Should explain the KDC, AS-REQ/AS-REP, TGT, TGS-REQ/TGS-REP flow, and ticket-based authentication |
| 3 | How do you do an ASREPRoast attack? | Should explain targeting accounts with pre-authentication disabled, requesting AS-REP hashes, and cracking offline with hashcat |
| 4 | What do you have to consider when setting up a malleable C2 profile? | Should mention traffic blending/OPSEC, sleep timers, jitter, staging, HTTP headers, and avoiding known signatures |
| 5 | How do you deal with clients in a red team engagement? | Should cover communication cadence, deconfliction, reporting expectations, scope creep, and handling unexpected findings |

## Architecture

```
10 PDF books (chapters as documents)
        ↓
[Document Ingestion — pdfplumber]
        ↓
[Chunking — RecursiveCharacterTextSplitter, 500 tokens, 100 overlap]
        ↓
[Embedding — all-MiniLM-L6-v2] → [Vector Store — ChromaDB]
        ↓
User query → [Retrieval — semantic search, top-k=5]
        ↓
[Generation — Groq llama-3.3-70b-versatile]
        ↓
Grounded answer + citations
```

## Anticipated Challenges

1. **Textbook boilerplate:** Published PDFs contain tables of contents, indexes, copyright pages, footnotes, and chapter headers that are not substantive content. If these aren't stripped during ingestion, they'll pollute the vector store and surface as irrelevant chunks during retrieval. Mitigation: print and inspect raw extracted text before chunking; add cleaning steps to remove these artifacts.

2. **Technical jargon and OOV terms:** Terms like "ASREPRoast," "malleable C2 profile," and "OPSEC" are niche enough that `all-MiniLM-L6-v2` — trained on general text — may not embed them with high fidelity. This could cause semantic search to return loosely related chunks rather than the specific content needed. Mitigation: test retrieval on the jargon-heavy evaluation questions early and consider adjusting chunk size or switching to a more domain-aware embedding model if results are poor.

3. **Concept spread across books:** The same topic (e.g. Kerberos) may be explained in multiple books. Retrieval might return 5 chunks all from the same book, missing potentially better explanations elsewhere. Mitigation: inspect source diversity in retrieval results during testing.

## AI Tool Plan

| Pipeline component | What I'll give the AI | What I expect it to produce |
|---|---|---|
| Ingestion + cleaning | Documents section (PDF source type), Chunking section, pipeline diagram | Script that loads each PDF with pdfplumber, cleans boilerplate, and outputs structured text per chapter |
| Chunking | Chunking strategy section (500 tokens, 100 overlap, RecursiveCharacterTextSplitter) | `chunk_text()` function that takes cleaned text and returns list of chunks with source metadata |
| Embedding + ChromaDB | Retrieval approach section, pipeline diagram | Script that embeds all chunks with all-MiniLM-L6-v2 and loads them into ChromaDB with source metadata |
| Generation | Grounding requirement (answer from retrieved context only, cite sources), output format | Prompt template + generation function that enforces grounding and returns answer + source list |
| Gradio interface | Interface requirements (text input, answer output, sources output) | Working `app.py` with Gradio Blocks layout |

I will review all generated code before running it, and ask the AI to explain any call or pattern I don't recognize.
