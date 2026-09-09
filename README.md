# Ryan Kim

**UC Berkeley EECS '29** (sophomore · GPA 3.82) · Palo Alto, CA<br>
Software Engineer (part-time) at **Teserac AI**

**I build full-stack products, and the AI systems inside them.** Django/DRF and React/TypeScript on Postgres, deployed on infrastructure I write myself — plus the model-facing parts of those products: document pipelines, grounding and citation verification, agent-to-UI protocols, on-prem and on-device inference.

**Open to Summer 2027 software engineering and applied-AI internships.**

---

### Why this page looks empty

The contribution graph below shows **2,072 contributions in 2026 across ~147 active days — every one of them in a private repo**: a production monorepo at the startup I work for, plus two startup orgs of my own. GitHub renders private work as a green square and nothing else, so there's nothing here to click.

So this README is the artifact. Everything below is described at the level of mechanism — the actual failure mode, the actual fix — because that's the part you can interrogate in an interview.

**2026 so far:** ~1,350 commits merged to default branches across 15 private repos · 536 pull requests opened, 426 merged · 206 code reviews left on other engineers' work.

My team works in an agent-assisted repo, and so do I. That's why I count **pull requests merged, code reviews given, and systems owned** — not lines written. The design decisions, the incident analysis, and the review findings are the part that's mine, and those are the parts I'm happy to be asked about.

---

## What I've shipped in 2026

| What it is | What's mine |
| --- | --- |
| **Teserac AI** — datacenter operations platform; Django + React monorepo | One of the **top three contributors by commit count**: 227 PRs merged, 204 reviews given in six months |
| **RayNeo X3 Pro AR app** — hands-free technician app for smart glasses (Kotlin / Jetpack Compose) | **Only engineer** — 17 of the 19 pull requests that repo has ever had |
| **React Native field app** — shipped to TestFlight | **Built end to end** — every pull request in the repo is mine |
| **A document-intelligence product for real-estate due diligence** | **342 of 344 commits**, 84 of 88 PRs, and 100% of the infrastructure repo |
| **An AI tutoring platform**, co-built with one other engineer | 438 of 568 commits across four repos; **100% of the infrastructure repo** and the product CI/CD |
| **AI marketing pipeline** — jobsite video in, timestamp-aligned short-form scripts out | Built and productized solo — generated **$650/month in recurring revenue** |

*Neither of those two is publicly launched: the diligence product's AWS environment is written and its CI/CD wired but not yet stood up; the tutoring platform is deployed to a UAT environment on ECS and paused. I'd rather say that than have it come up later.*

**Toolbox** — Python · TypeScript · Kotlin · Swift · SQL · Bash · Django REST Framework · Celery · PostgreSQL · TimescaleDB · Redis · React · React Native · Jetpack Compose · React Flow / ELK.js · TanStack Query · Tailwind · Terraform / Terragrunt · AWS · Docker · Keycloak / OIDC · LiveKit + WebRTC · NVIDIA Orin · SAM 2 · pytest · Vitest

---

## Four problems, and why they were actually hard

**1. Making concurrent edits to a live diagram safe.** A browser-based single-line-diagram editor for datacenter power topology, with two operators editing the same facility while live power telemetry polls underneath them. When structure, operator intent, and rendered geometry all live in one document, every write races every other write.

I re-architected it into **three independently versioned planes**: a flat, renderer-agnostic topology graph; a durable *layout-intent* document that stores semantic sibling ordering and deliberately **never stores resolved geometry**; and geometry itself as a disposable cache. Intent is versioned by an integer revision plus a **SHA-256 hash of a canonically serialized topology payload**, so a save arriving with a stale revision is rejected as an explicit conflict instead of silently overwriting someone. Geometry is a pure function of topology and intent, so it can always be regenerated — which is exactly why it must not be the thing you persist. Shipped as one 135-file PR, behind a flag, alongside the version it replaced. Two operators can reorganize a facility diagram at the same time without one silently overwriting the other.

**2. Determinism under a polling telemetry feed.** Power state polls continuously. If the layout cache key includes telemetry, every poll busts the cache, ELK re-runs, and the diagram moves under the operator's cursor.

The fix was a **structural fingerprint that hashes topology only**, so telemetry never enters the cache key and polling can never force a relayout. Alongside it: fixed-order synthetic ELK ports so component rows come out in the same order on every run, and a clear-then-regroup organize service that returns every asset to unassigned before regrouping, so none survives inside a stale group and none ends up unclassified. Layout stability is a correctness property, not a polish item — an operator who can't trust the diagram to stay still can't trust it at all.

**3. Deleting a facility without holding a transaction open for minutes.** One cascading subtree `DELETE` holds locks for the length of the whole statement and keeps a transaction open for minutes, which blocks vacuuming and queues every concurrent teardown behind it. Added telemetry pushed one purge past what a single transaction could reasonably hold.

I moved it onto a polled async Celery pipeline that **walks primary keys in batches, each committed in its own short transaction** so locks release between them; a **per-step row budget** so a step yields cleanly under the soft time limit rather than running unbounded; and a **stall guard** that fails loudly if a batch's filter stops removing its own rows, instead of spinning forever on a query that will never converge.

Then the reaper behind it, which is where the interesting constraint is: its running-stale threshold sits **deliberately above the purge task's own hard limit**, so a purge that is still running can never be dispatched a second time. Re-enqueues are bounded per tick and candidates ordered oldest-first — a reaper that re-dispatches every stuck job on every beat will amplify a purge bug into API latency, and a capped tick without ordering starves the same jobs forever.

**4. Session security that could ship without breaking anyone.** Overhauled email-domain SSO on Keycloak/OIDC: replaced DNS-based identity-provider inference with an explicit **per-organization domain-to-provider route map**, and collapsed three separate user-provisioning paths into one endpoint that provisions password, SSO, or both. On top of that, a configurable idle window plus a **separate absolute session-age cap enforced on every request**, and **per-request ECDSA P-256 device proofs** so a stolen session cookie is useless off its device — all behind an **off → monitor → enforce** rollout, so monitor mode reports exactly what enforcement *would* have rejected before anything starts getting rejected. Auth changes you can't observe before enforcing are how you take down every client at once.

<details>
<summary><b>More from the same codebase</b></summary>

- **Subsystems I own:** work orders and scheduling, the single-line-diagram editor, auth/SSO/permissions, the in-product AI chat integration, export/portability, and the async deletion pipeline.
- **Review catch — a silent revert.** On a branch I was shipping, I caught a commit that was an exact inverse of an already-merged fix: no conflict, no red diff, because the merge base already contained it. Merging it would have quietly undone a shipped fix and deleted 153 lines of its regression coverage. I blocked my own PR on it.
- **Review catch — a shared failure boundary.** An LLM asset-naming call shared a `try`/`catch` with payload construction, so a 5xx from the naming service aborted the entire save with no bypass. Gave it its own boundary so the save completes unnamed. A model call is a network dependency; it belongs behind the same kind of boundary as any other.
- About a quarter of the files I've touched in that repo are test files.

</details>

---

## Applied AI: what I actually build

A model is a component with a failure mode, and most of the engineering is in the boundary around it.

- **Grounding, so citations can't be hallucinated.** Every quote an LLM cites in a verification report is checked to actually appear in the OCR it cites — normalized substring, then whitespace-collapsed compare, then a sliding-window fuzzy match at a 0.92 similarity threshold to survive OCR drift. Cited document IDs are separately validated against the property's real document index. A finding that can't be grounded doesn't ship to the report.
- **Indirect prompt-injection defense over user-uploaded PDFs.** A shared safety rule block appended to every indexing system prompt, separate untrusted-content boundaries wrapping OCR body text and upload metadata, and sanitization of forged document-boundary markers an attacker can embed in a PDF to make their text look like the system's own framing.
- **Map-reduce over data rooms larger than a context window.** 60k-character chunks with 3k overlap, mapped per chunk, then **deterministically** merged, deduped, re-ranked and capped severity-first before a final summary pass. The merge is code, not another model call — it's the part that has to be reproducible.
- **Vision coordinates that land where you think they do.** For automated signature placement, a vision model returns pixel boxes in *its own* resized frame, so I reimplemented the resize rule client-side and pre-resize before upload — returned coordinates then map 1:1 onto the image that was sent. I deliberately don't reproduce padding: normalizing by padded dimensions puts a small scale error on every single coordinate.
- **On-prem and on-device inference.** For the AR app: wake word → VAD → on-prem NVIDIA Orin ASR → intent classification that resolves navigation intents to **local deep links rather than a network round trip**, on-device SAM 2 segmentation, a WebRTC remote-expert mode with annotation overlay, and an offline-first cache with a retry-classifying outbox. Technicians in a datacenter don't have reliable connectivity; that constraint drives the whole design.
- **Agent-to-UI command protocol.** Structured action payloads streamed from an agent service execute against a live diagram canvas — focus an entity, toggle a group, stage location changes — including a snapshot-hash handshake so agent-staged edits reconcile against concurrent human edits instead of clobbering them.
- **Evals, so prompt and model changes get compared rather than eyeballed.** Intent evaluations over recorded voice fixtures for the AR pipeline; for the tutor agent, the offline eval set and deterministic scorers I contributed into the harness we ran it against.

**Where I draw the line:** I don't train or fine-tune models, and none of this is research. I build the pipelines, grounding, evaluation, serving, and edge-inference systems *around* models — applied AI systems engineering — and I'd rather state that precisely than blur it.

---

<details>
<summary><b>Smaller things, built solo</b></summary>

- **Clutch** (SwiftUI, Jun 2026) — bulk-prices LEGO from photos. On-device Apple Vision OCR reads set numbers; S3 presigned uploads hand off to AWS Lambda, where Claude with extended thinking and web search resolves the sets a camera can't read. Pricing is a router over three marketplace connectors, so adding a source is one dispatch-table entry plus a module. Every identification lands as a *suggestion* for human confirmation rather than being auto-applied.
- **AI marketing pipeline** (Dec 2025 – May 2026) — FFmpeg audio extraction → Deepgram word-level timestamps → translation → LLM topic and step extraction → automatic clip cutting. The interesting part: the model reasons over *translated English*, but the cuts have to land on the *original* audio, so script segments are mapped back onto the word-level timing array to produce exact in/out points. Productized into a client-facing service for a Bay Area roofing contractor.
- **Internal cloud-cost dashboard** — Django + Celery-beat collector persisting a monthly time series in integer cents, with a pluggable per-provider collector interface where one failing credential can't block the rest.

</details>

---

## Trajectory

**2024:** zero commits. **2025:** taught myself data structures and algorithms — 221 commits and 203 Python files of graph algorithms, DP, heaps, tries, and backtracking, before I started at Berkeley. **2026:** fifteen private repos and four products — two shipped inside a company, two of my own still pre-launch.

**Coursework:** CS61C Machine Structures and CS70 Discrete Math & Probability (in progress, Fall 2026); Data Structures & Algorithms, Data 8 Foundations of Data Science, and Circuits & Devices completed.

---

## Attribution, since none of this is clickable

- The company monorepo is a **team** codebase. I'm one of several engineers on it and not its lead. I own the subsystems named above; I did not build the platform.
- On the **diligence product**, the backend, frontend, and infrastructure are mine. The chat agent service is a teammate's code; the system architecture document that specifies it is mine.
- The **tutoring platform** was two of us. My half is the production path — the deployed tutor agent worker, model and TTS integration, containerization, the whiteboard sync protocol over a single LiveKit data channel, the session-analysis pipeline, and all of the infrastructure and CI. My co-founder originated the sketch DSL, the grounding module, the narrator, the batch transcript pipeline, and the model gateway, among other core AI internals.
- I have never trained or fine-tuned a model or done modeling research, and nothing here should read as if I had.

---

## Contact

**LinkedIn** — [linkedin.com/in/rkim0709](https://linkedin.com/in/rkim0709) · **Email** — ryankim1@berkeley.edu

Happy to walk through any system on this page in detail — the tradeoffs are more interesting than the summaries.

*Off the clock: long-distance running.*
