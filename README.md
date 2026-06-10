# The Unofficial Guide — Project 1

> **How to use this template:**
> Complete each section *after* you've built and tested the corresponding part of your system.
> Do not write placeholder text — if a section isn't done yet, leave it blank and come back.
> Every section below is required for submission. One-liners will not receive full credit.

---

## Domain

I built a RAG system over a collection of pentesting and red teaming textbooks. The problem is that this kind of knowledge is spread across hundreds of pages across multiple books and there's no good way to look something up quickly — if you want to know how Kerberos delegation works or what to think about when setting up C2 infrastructure, you're manually skimming through chapters hoping you find the right section. The goal here is to let someone ask a targeted question and get an answer that's actually grounded in the source material, with citations so you can go read the full context if you need to.
---

## Document Sources

They are all cybersecurity books that I already own. I will provide the amazon links for them.

| # | Source | Type | URL or file path |
|---|--------|------|-----------------|
| 1 | Black Hat Python, 2nd Edition — Justin Seitz & Tim Arnold (2021, No Starch Press) | PDF | documents/[{Cyber-Security-COVER} ] Justin Seitz & Tim Arnold - Black Hat Python, 2nd Edition (2021, No Starch Press) - libgen.li.pdf |
| 2 | Offensive Security PEN-300: Evasion Techniques and Breaching Defenses (2020) | PDF | documents/Offensive Security - OSEP - PEN-300 - Evasion Techniques and Breaching Defenses (2020) - libgen.li.pdf |
| 3 | The Web Application Hacker's Handbook — Stuttard & Pinto | PDF | documents/The Web Application Hacker's Handbook_ Finding and -- Stuttard, Dafydd & Pinto, Marcus -- Anna's Archive.pdf |
| 4 | Operator Handbook: Red Team + OSINT + Blue Team Reference — Joshua Picolet (2020) | PDF | documents/Operator handbook _ Red Team + OSINT + Blue Team Reference -- Joshua Picolet -- Anna's Archive-1.pdf |
| 5 | Red Team Development and Operations: A Practical Guide — Joe Vest & James Tubberville | PDF | documents/Red Team Development and Operations A Practical Guide (Joe Vest, James Tubberville) (z-library.sk, 1lib.sk, z-lib.sk).pdf |
| 6 | Offensive Security PEN-200 / OSCP (2023 version) | PDF | documents/PEN200 - OSCP - 2023 version (Shared by Tamarisk) (Z-Library).pdf |
| 7 | Cloud Penetration Testing for Red Teamers | PDF | documents/Cloud Penetration Testing for Red Teamers _ Learn How to.pdf |
| 8 | Coding for Penetration Testers: Building Better Tools — Jason Andress | PDF | documents/Coding_for_Penetration_Testers_Building_Better_Tools_Jason_Andress.pdf |
| 9 | Definitive Guide to Cyber Threat Intelligence | PDF | documents/Definitive-Guide-to-CTI-THlink.pdf |
| 10 | NIST Cybersecurity Framework 2.0 (NIST.CSWP.29) | PDF | documents/NIST.CSWP.29.pdf |

---

## Chunking Strategy

<!-- Describe your chunking approach with enough specificity that someone else could reproduce it.
     Include:
     - Chunk size (characters or tokens) and why that size fits your documents
     - Overlap size and why (or why not) you used overlap
     - Any preprocessing you did before chunking (e.g., stripping HTML, removing headers)
     - What your final chunk count was across all documents -->

**Chunk size:** 2000 characters (~500 tokens, using 1 token ≈ 4 chars as a rough estimate)

**Overlap:** 400 characters (~100 tokens)

**Why these choices fit your documents:** These books explain technical concepts over multiple paragraphs — something like a Kerberos auth flow or how to set up C2 infrastructure isn't going to fit in 2-3 sentences. I wanted chunks big enough to contain a full explanation of one concept, not just half of it. The overlap is there because sometimes an explanation starts at the end of one chunk and continues in the next — without overlap you'd only ever retrieve half of it. I went with fixed-size chunking instead of splitting on paragraphs because the paragraph lengths in these books are all over the place. Some are one sentence, some are a full page. That would make retrieval really inconsistent. Before chunking I also ran a cleaning pass to strip out page numbers, running headers/footers, and the broken hyphenation you get when PDFs line-wrap mid-word (like `privi-\nlege`).

**Final chunk count:** 5551 chunks across 14 documents. One PDF was corrupt and got skipped — pdfplumber threw a `No /Root object` error on it so I added error handling to skip bad files rather than crash the whole pipeline.

---

## Embedding Model

<!-- Name the embedding model you used and explain your choice.
     Then answer: if you were deploying this system for real users and cost wasn't a constraint,
     what tradeoffs would you weigh in choosing a different model?
     Consider: context length limits, multilingual support, accuracy on domain-specific text,
     latency, and local vs. API-hosted. -->

**Model used:** `all-MiniLM-L6-v2` via `sentence-transformers` (runs locally, no API key needed)

**Production tradeoff reflection:** If I was actually deploying this for real users, there are a few things I'd want to change. The biggest issue I ran into is that `all-MiniLM-L6-v2` has a 256 token context limit, but my chunks are around 500 tokens. That means every chunk is getting silently truncated when it gets embedded, which is not great. Something like `text-embedding-3-large` from OpenAI handles up to 8191 tokens so that wouldn't be a problem. The other issue is that this model was trained on general text, not security content. Terms like "ASREPRoast" or "malleable C2 profile" aren't things it's seen before so it doesn't really know how to embed them well — that's exactly what caused my retrieval failure (described below). A model that's been trained or fine-tuned on technical content would handle those terms much better. The tradeoff with switching to an API model is you're now paying per query and adding network latency, but for the scale this project is at, local inference was fine.

---

## Grounded Generation

<!-- Explain how your system enforces grounding — how does it prevent the LLM from answering
     beyond the retrieved documents?
     Describe both your system prompt (what instruction you gave the model) and any structural
     choices (e.g., how you formatted the context, whether you filtered low-relevance chunks).
     Do not just say "I told it to use the documents" — show the actual instruction or explain
     the mechanism. -->

**System prompt grounding instruction:** The system prompt I'm using is:

> You are a knowledgeable assistant specializing in penetration testing and red teaming. Answer questions using ONLY the provided context excerpts. Do not use outside knowledge. If the context does not contain enough information to answer, say so clearly. After your answer, list the sources you drew from.

The "ONLY the provided context excerpts" and "Do not use outside knowledge" parts are doing the main work here. I also added the instruction to say so clearly if the context isn't enough — this was important because without it, the model might try to answer anyway using its training data, which defeats the point. When ASREPRoast retrieval failed, the model correctly said it didn't have enough information rather than just making something up.

**How source attribution is surfaced in the response:** Before sending context to the model, each chunk gets numbered and labeled with its source filename like this:

```
[1] Source: <filename>
<chunk text>

---

[2] Source: <filename>
<chunk text>
```

The model then cites these numbers in its response. I also added a verbose mode that prints the source filename and cosine distance for each retrieved chunk before the answer, so you can see at a glance whether retrieval actually pulled the right stuff.

---

## Evaluation Report

<!-- Run your 5 test questions from planning.md through your system and record the results.
     Be honest — a partially accurate or inaccurate result that you explain well is more
     valuable than a suspiciously perfect result. -->

| # | Question | Expected answer | System response (summarized) | Retrieval quality | Response accuracy |
|---|----------|-----------------|------------------------------|-------------------|-------------------|
| 1 | How does a red team engagement work? | Scoping, rules of engagement, recon, initial access, lateral movement, post-exploitation, reporting | 7-step answer covering ROE, team roles (Lead/Operators), engagement frequency, culmination/reporting, and use of attack flow diagrams. All chunks from Vest & Tubberville at distance 0.32–0.39. | Relevant | Accurate |
| 2 | How does Kerberos work? | KDC, AS-REQ/AS-REP, TGT, TGS-REQ/TGS-REP flow, ticket-based authentication | Explained KDC role, AS-REQ with encrypted timestamp, AS-REP, TGT/credential cache, SPNs, and delegation. Top chunks from PEN-200 (0.33) and PEN-300 (0.40). | Relevant | Accurate |
| 3 | How does DLL hijacking work? | Windows DLL search order abuse, placing malicious DLL earlier in search path, privilege escalation | Explained Windows DLL search order (6 steps), missing DLL attack pattern, DllMain/DLL_PROCESS_ATTACH execution. Top chunk from PEN-200 at distance 0.33. | Relevant | Accurate |
| 4 | What do you have to consider when setting up a malleable C2 profile? | Traffic blending/OPSEC, sleep timers, jitter, staging, HTTP headers, avoiding known signatures | Covered C2 channel selection, domain categorization, SSL certs, infrastructure separation, proxy log behavior, protocol conformity. Good answer but high distances (0.63+) because books discuss the concept without using the term "malleable C2 profile." | Partially relevant | Partially accurate |
| 5 | How do you deal with clients in a red team engagement? | Communication cadence, deconfliction, reporting expectations, scope creep, handling unexpected findings | 9-point answer covering engagement planning, shared lexicon, scope/cost documentation, announced vs. unannounced choice, ROE, assumptions, reporting. All chunks from Vest & Tubberville at distance 0.40–0.48. | Relevant | Accurate |

**Retrieval quality:** Relevant / Partially relevant / Off-target  
**Response accuracy:** Accurate / Partially accurate / Inaccurate

---

## Failure Case Analysis

<!-- Identify at least one question where retrieval or generation did not work as expected.
     Write a specific explanation of *why* it failed, tied to a part of the pipeline.

     "The answer was wrong" is not an explanation.

     "The relevant information was split across a chunk boundary, so retrieval returned
     only half the context — the model didn't have enough to answer correctly" is an explanation.

     "The embedding model treated the professor's nickname as out-of-vocabulary and returned
     results from an unrelated review" is an explanation. -->

**Question that failed:** "How do you do an ASREPRoast attack?" — I replaced this with DLL hijacking for the final eval, but it's worth explaining why it failed.

**What the system returned:** It said it didn't have enough information to answer. The top 5 chunks all came from completely unrelated sources — CTI guides and the Red Team Development book — with cosine distances around 0.62–0.66. The PEN-300 book definitely covers ASREPRoast but none of its chunks showed up in the results at all.

**Root cause (tied to a specific pipeline stage):** This is an embedding stage problem. "ASREPRoast" is a niche security term that `all-MiniLM-L6-v2` has never seen — it's a portmanteau of "AS-REP" (the Kerberos Authentication Server Reply) and "Roasting" (offline hash cracking). The model doesn't know what it means, so the query vector ends up nowhere near the Kerberos chunks in PEN-300. I tried bumping `top_k` from 5 to 10 to see if the right chunks were just ranked slightly lower, but they still didn't show up. The relevant content exists in the corpus — the model just can't connect the query to it. This is exactly the jargon/OOV problem I flagged in `planning.md` before building.

**What you would change to fix it:** Two things. Add a keyword pre-filter that checks for exact term matches before doing semantic search — if "ASREPRoast" appears in a chunk, include it regardless of the embedding distance. And longer term, switch to an embedding model that's been trained on technical or security content so these terms are actually represented properly in the vector space.

---

## Spec Reflection

<!-- Reflect on how planning.md shaped your implementation.
     Answer both questions with at least 2–3 sentences each. -->

**One way the spec helped you during implementation:** Writing out the Anticipated Challenges section before I started coding meant I already had a diagnosis ready when ASREPRoast retrieval failed. I'd already written down that `all-MiniLM-L6-v2` might struggle with niche security terms that aren't in its training data, so when it happened I wasn't confused — I knew exactly what stage had failed and why. Without that, I probably would have wasted time looking for bugs in the chunking or the ChromaDB setup when the real issue was the embedding model.

**One way your implementation diverged from the spec, and why:** The spec said I'd treat each chapter as its own document with metadata like "Book Title — Chapter 3: Privilege Escalation." I ended up just treating each full book as one document instead. Extracting chapter structure reliably from PDFs turned out to be harder than I expected — you'd need to parse the table of contents or detect chapter headings with some heuristic, and pdfplumber doesn't really help with that. Since retrieval was working fine with just book-level attribution, I decided it wasn't worth the added complexity for this project.

---

## AI Usage

<!-- Describe at least 2 specific instances where you used an AI tool during this project.
     For each: what did you give the AI as input, what did it produce, and what did you
     change, override, or direct differently?

     "I used Claude to help me code" is not sufficient.
     "I gave Claude my Chunking Strategy section from planning.md and asked it to implement
     chunk_text(). It returned a function using a fixed character split. I overrode the
     chunk size from 500 to 200 because my documents are short reviews, not long guides." -->

**Instance 1**

- *What I gave the AI:* The Documents and Chunking Strategy sections from `planning.md` plus the pipeline diagram (pdfplumber → cleaning → RecursiveCharacterTextSplitter → chunks.json).
- *What it produced:* A working `ingest.py` with functions for PDF extraction, text cleaning, grouping pages into documents, and chunking. The cleaning step handled page numbers, short header/footer lines, and the broken hyphenation issue where PDFs split words across lines.
- *What I changed or overrode:* When I ran it, it crashed on one of the books because the PDF file was corrupt. I had it add a try/except around the per-file loop so it skips bad files and prints a warning instead of killing the whole pipeline.

**Instance 2**

- *What I gave the AI:* The Retrieval Approach section from `planning.md` (all-MiniLM-L6-v2, ChromaDB, top-k=5) and the requirement that answers be grounded in retrieved docs with source citations.
- *What it produced:* A working `query.py` with retrieval, context formatting, generation, and an interactive mode that runs the 5 eval questions on first launch as a smoke test.
- *What I changed or overrode:* The initial version used Groq as the LLM. I switched it to OpenRouter with the `nex-agi/nex-n2-pro:free` model instead, which meant swapping the client library and pointing it at the OpenRouter API base URL. I also experimented with raising `top_k` to 10 to try to fix the ASREPRoast miss — that made question 1 return `None` because the context was too long for the model, so I put it back to 5.
