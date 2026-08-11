# OpenAgriNet — How publish and discover work

A plain-language explanation of how the platform works today, written as the baseline for moving onto the current Beckn protocol.

---

## 1. The three products

All three are the same thing wearing different clothes: an **AI assistant that answers farmers' questions** in their own language, by voice or text. They are built from one common codebase and then customised.

| | **BharatVistaar** | **MahaVistaar** | **Amul** |
|---|---|---|---|
| Who uses it | Farmers nationally, via central and state agriculture departments | Farmers in Maharashtra, under the state's POCRA programme | Dairy farmers in the Amul network |
| How they reach it | Website and Android/iOS app | Website and iOS app | Mostly by **phone call**, in Gujarati |
| What it answers | Schemes, crop advice, mandi prices, weather, insurance, fertiliser guidance | Same, focused on Maharashtra schemes | Dairy and farming guidance |
| Runs its own provider node | **Yes** | No — uses a shared node run elsewhere | No |
| Uses the Beckn network | Yes | Yes | **Yes** — as a client of others |

**The short version:** BharatVistaar is the full implementation and the only one operating its own provider node. MahaVistaar is a state variant. Amul is voice-first and dairy-specific.

**They also call each other.** All three are Beckn clients of one another — MahaVistaar can query BharatVistaar, BharatVistaar can query MahaVistaar, and Amul gets its weather, mandi prices and scheme information by querying the Vistaar network rather than building its own. So "is X on the network" has two answers: whether it *operates* a node, and whether it *uses* one. Only BharatVistaar does the first; all three do the second.

---

## 2. Start here: two catalogs, plus one thing that isn't a catalog at all

This is the single thing that makes everything else confusing if you miss it. When someone says "the catalog," they could mean either of two unrelated systems — and a lot of what the platform answers doesn't come from a catalog in the first place.

**The two real catalogs.** Something is published, it is stored, it is searched later.

```mermaid
graph TB
  subgraph C1["CONTENT CATALOG — the Beckn one"]
    direction TB
    P1["Published by:<br/>agriculture departments, ICAR,<br/>scheme owners"]
    W1["Scheme details, advisory articles,<br/>videos, scholarships"]
    S1[("Stored as structured rows<br/>in one database table")]
    P1 -->|"publishes"| W1 -->|"stored"| S1
  end

  subgraph C2["KNOWLEDGE CATALOG — the AI one"]
    direction TB
    P2["Published by:<br/>the internal content team"]
    W2["Scheme guideline PDFs,<br/>government circulars, manuals"]
    S2[("Stored as meaning-indexed<br/>text in a vector database")]
    P2 -->|"publishes"| W2 -->|"stored"| S2
  end
```

They have **different publishers, different approval rules, different storage, and completely different search mechanics**. A request to "add a new scheme to the catalog" means two different jobs depending on which one is meant.

**Live data is not a catalog.** Nothing is published into it and no answer is stored. When a farmer asks, the provider node calls the government system and passes back whatever it says. Nothing survives the request.

```mermaid
graph LR
  Q["Farmer's question<br/>arrives"] --> N["Provider node"]
  N -->|"1 — look up reference data:<br/>which mandi? which weather station?"| Ref[("Reference database<br/>markets, stations, commodities")]
  Ref --> N
  N -->|"2 — call live, per request"| Ext["Agmarknet — prices<br/>IMD / Mausamgram — forecasts"]
  Ext -->|"today's values"| N
  N --> A["Answer<br/>nothing is stored"]
```

The distinction that matters: the **reference data is stored** — the list of markets, weather stations and commodities is held locally and needs maintaining. The **values are not**. So this is a live proxy with a lookup table in front of it, not a catalog with fresh entries.

---

## 3. Publish — what goes in, and how

### The content catalog

**Who publishes:** external organisations — state agriculture departments, ICAR, scheme-owning bodies. They register for an account and are approved as an organisation.

**What they publish:**
- Scheme listings — name, who's eligible, what the benefit is, how to apply
- Advisory content — crop guidance, farming practice articles
- Videos
- Scholarship listings

**How they publish it:** they log in and either fill in a form for a single item, or upload a spreadsheet to load hundreds at once. The spreadsheet has a fixed set of columns they must match.

**Who approves it:** *nobody, at the item level.* An admin approves the **organisation** once. From that moment, anything that organisation uploads goes live immediately — there is no review queue, no second pair of eyes on individual content.

**Where it lands:** a single table in a standard database. Notably, there are **no separate tables for providers, catalogs, or items**. When a search comes in, the Beckn-shaped response is assembled on the fly from those flat rows.

### The knowledge catalog

**Who publishes:** the internal team, not external organisations.

**What they publish:** long-form government documents — scheme guideline PDFs, circulars, operational manuals. The things too dense for a farmer to read, that the assistant needs to be able to quote from.

**How it works:** the document goes through an automated pipeline that splits it into small passages and converts each passage into a numeric representation of its *meaning*. This is what lets the assistant find "the bit about eligibility" without anyone tagging it.

**Who approves it:** **two humans**, in sequence. A reviewer signs off after the document is processed, and then a superadmin promotes it to production. This is genuinely reviewed content, unlike the content catalog.

**Where it lands:** a vector database for the passages, plus a summary list of available schemes that is cached separately. That cache is what lets a newly-added scheme become searchable **without redeploying the assistant**.

### The two publish paths side by side

```mermaid
graph LR
  Dept["Agriculture dept /<br/>ICAR / scheme owner"] -->|"form or spreadsheet"| Portal["Provider portal"]
  Portal -->|"live immediately,<br/>no item review"| DB[("Content table")]
  Admin["Admin"] -.->|"approves the ORGANISATION,<br/>not the content"| Portal

  Team["Internal content team"] -->|"upload PDF"| Pipe["Processing pipeline<br/>splits + indexes by meaning"]
  Pipe --> Rev{"Reviewer<br/>approves"}
  Rev --> Sup{"Superadmin<br/>promotes"}
  Sup --> VDB[("Vector database")]
  Sup --> Cache[("Cached scheme list")]
```

Note the asymmetry: the **externally published** catalog gets no item-level review, while the **internally published** one requires two human approvals.

### Live data — nothing to publish

Nobody publishes mandi prices or weather forecasts. What *is* maintained is the reference data behind them — the list of markets, weather stations and commodities used to work out which mandi or station a farmer's location maps to. That list is loaded and kept in a database. The prices and forecasts themselves are fetched per request and never stored.

---

## 4. Discover — how a search actually works

### The farmer never searches

This is the key point. There is no search box and no browsing. The farmer **asks a question in plain language** — typed or spoken, in their own language. Everything after that is the assistant's job.

The assistant reads the question, works out what's being asked, and picks where to look. It has roughly seventeen different sources it can reach for.

### Three different ways it looks things up

**1. Structured lookup — for schemes and published content**
The assistant sends a request across the Beckn network describing what it wants. The provider node reads the *category* of that request and routes it to the matching internal service, which queries the database and returns matching rows dressed up in Beckn's catalog format.

**2. Meaning-based search — for documents and videos**
The assistant converts the farmer's question into the same kind of numeric meaning-representation used when the documents were indexed, then finds the passages that sit closest to it. This is why a farmer can ask "will I get money if my crop fails" and get back the right paragraph from an insurance scheme document that never uses those words.

This search **does not go through Beckn at all.** The assistant queries the vector database directly.

**3. Live proxy — for prices and weather**
The assistant does **not** call the government system itself. It sends an ordinary Beckn search, exactly as it would for a scheme. The provider node recognises the category, and *it* makes the outbound call — first resolving which market or weather station applies, then fetching the current values. The assistant cannot tell the difference between this and a stored-catalog answer; both come back in the same shape.

### How the provider node decides

Everything hinges on the category label in the request:

| Category in the request | What the node does |
|---|---|
| price discovery, item = mandi | resolves the nearest market, then calls **Agmarknet** for today's prices |
| weather forecast | resolves the weather station, then calls the **IMD** station service |
| weather forecast (Mausamgram) | calls **Mausamgram** by coordinates instead |
| ICAR schemes | queries its own content database — no external call |
| crop insurance | calls the external insurance service |
| anything it doesn't recognise | falls through to a general-purpose search service |

That last row matters — an unrecognised category doesn't fail, it silently produces a generic answer. See §7.

### What comes back

The assistant takes whatever the source returned, writes an answer in the farmer's language, and streams it back sentence by sentence so the farmer sees it appear as it's written rather than waiting for the whole thing.

```mermaid
graph LR
  F["Farmer asks a question<br/>(text or voice, any language)"] --> A["AI assistant<br/>decides where to look"]
  A -->|"Beckn request,<br/>tagged with a category"| N["Provider node<br/>routes by category"]
  N --> DB[("Content table")]
  N -->|"live, per request"| Gov[("Government systems:<br/>Agmarknet prices,<br/>IMD / Mausamgram weather")]
  A -->|"direct, no Beckn"| V[("Vector database:<br/>documents, videos")]
  A -->|"direct, no Beckn"| Other[("Other services:<br/>fertiliser, soil health,<br/>application status")]
  DB --> A
  Gov --> A
  V --> A
  Other --> A
  A --> R["Answer written in the<br/>farmer's language, streamed back"]
```

---

## 5. Who publishes what, who discovers what

| | **Publishes** | **Discovers** |
|---|---|---|
| Agriculture departments, ICAR, scheme owners | Scheme details, advisory articles, videos, scholarships | nothing |
| Internal content team | Scheme guideline documents and circulars | nothing |
| Government systems (Agmarknet, IMD, Mausamgram, insurance) | nothing — they expose live data on request | nothing |
| **The AI assistant** | nothing | **everything** — it is the only thing that searches |
| **The other two products** | nothing | each other's catalogs, over Beckn |
| Farmers | nothing | nothing directly — they ask the assistant |

**In Beckn terms:** the assistant is the buyer-side app (BAP). The provider node is the seller-side platform (BPP). The departments are the providers. Farmers are the end users, one step removed — they never touch the network themselves.

**One clarification worth making,** because the words collide: the **provider node** is software — one deployment, run by the platform team, that answers searches. The **providers** are the organisations that publish into it. Many providers, one node. A department doesn't run anything; it just has a login.

---

## 6. A worked example

**A farmer asks: "What's the tomato price in Pune today?"**

1. The question arrives — possibly spoken in Marathi, transcribed to text.
2. The assistant identifies this as a price question and picks the mandi tool.
3. It builds a Beckn request tagged as a price enquiry and sends it to the provider node.
4. The node sees the price category and routes it to the mandi handler.
5. The mandi service first resolves which market the farmer's location maps to, using reference data it holds locally, then calls Agmarknet live for that market's current prices.
6. It returns the prices formatted as a Beckn catalog — **immediately, in the same reply**.
7. The assistant turns the numbers into a sentence in Marathi and streams it back.
8. Separately and in the background, a record of the interaction goes to the analytics system.

The whole thing is one round trip. Step 6 is where today's system differs most from real Beckn — see below.

---

## 7. Where this differs from the current Beckn protocol

The reason for this document. Restated plainly:

1. **Answers come back immediately.** Real Beckn expects the provider to say "got it" and send results separately a moment later, so many providers can answer the same question at their own pace. Today it's a single question-and-answer, like any ordinary web request. This is the biggest change, and it affects the assistant too — it would need to start *receiving* results rather than just waiting for them.

2. **Requests aren't signed.** Beckn expects each participant to cryptographically sign its messages so the other side can verify who sent them. None of that happens in the application. It may be handled by the network component the system runs behind, but that has not been confirmed — worth checking before assuming it exists.

3. **There's no directory of participants.** Real Beckn has a registry you consult to find out who's on the network. Here, the assistant talks to exactly one provider address, fixed in configuration. It cannot discover anyone else.

4. **Everything appears under one provider name.** All knowledge results come back attributed to a single provider, regardless of who actually published them. A real network needs each department to appear as itself.

5. **The catalog isn't really a catalog.** There's no stored notion of providers, catalogs, or items — just flat content rows reshaped into Beckn's format at the moment of answering. A proper implementation needs the real structure.

6. **Nobody checks individual content.** Approval is at the organisation level only. If the new network requires per-item trust, that's new work, not a migration.

7. **One search path is silently broken.** The assistant's scheme-document search sends a category label that the provider node doesn't recognise, so it quietly falls through to generic search instead. Verified on both sides. Fix it or remove it — don't carry it forward.

8. **Decide what belongs on the network.** Today the document/video search deliberately bypasses Beckn while structured content goes through it. That split happened by accident rather than design. Choose it consciously this time.

9. **Only one product operates a node, but all three use the network.** BharatVistaar runs the only provider node examined here. MahaVistaar uses a shared node nobody has looked at. Amul runs no node yet is a client of several — it queries the Vistaar network for weather, prices and schemes, its own Amul network for documents and union schemes, and a separate node to place call bookings. The products also query each other. So the network already has real cross-traffic; it just has no registry, no signing and no discovery underneath it. That makes the redesign more urgent, not less.

---

## 8. Appendix — the actual APIs

For engineers. The rest of the document deliberately avoids these.

**Publishing into the content catalog** (provider node)

| Endpoint | Purpose |
|---|---|
| `POST /auth/registerUser`, `POST /auth/login` | organisation registers, then gets a token |
| `PATCH /admin/approval/:id` | admin approves the **organisation** — the only approval that exists |
| `POST /provider/content` | publish one item |
| `POST /provider/createBulkContent` | spreadsheet upload, fixed 28-column layout |
| `POST /provider/icarcontent` | scheme-shaped items |
| `POST /provider/collection`, `/contentCollection` | group items |
| `POST /provider/scholarship`, `/uploadImage` | scholarships, media |

**Publishing into the knowledge catalog** (ingestion pipeline)

| Endpoint | Purpose |
|---|---|
| `POST /documents/{id}/approve-ingestion` | reviewer sign-off |
| `POST /documents/{id}/approve-prod` | superadmin promotes to production |

**Discovery** (provider node) — `POST /mobility/search` is the one that matters. `select`, `init`, `confirm`, `rating`, `status` exist under the same prefix. A legacy `/dsep/*` set still runs alongside it. Routing is decided by `message.intent.category.descriptor.name` / `.code`:

| Category value | Goes to |
|---|---|
| `knowledge-advisory` | general search service |
| `Weather-Forecast` | IMD station lookup |
| `Weather-Forecast-Mausamgram` | Mausamgram by coordinates |
| `schemes-agri` | PM-Kisan handler |
| `icar-schemes` | content database via GraphQL |
| `price-discovery` + item code `mandi` | Agmarknet |
| `pmfby` (or any code starting `pmfby`) | crop insurance service |
| *unmatched* | general search service |

> Two traps here. `schemes-agri` and `icar-schemes` do **not** go where their names suggest. And the assistant sends `scheme-agri-qdrant`, which matches nothing — it lands in the unmatched row.

**Outbound calls the provider node makes**

| Call | Notes |
|---|---|
| `GET {MANDI_BASE_URL}/v1/fetch-agmarknet-vistaar` | 15s timeout, retries with a relaxed query if empty |
| `GET {IMD_WEATHER_API_URL}?id={stationId}` | 30s timeout, 3 attempts, 2s backoff |
| `GET {MAUSAMGRAM_ENDPOINT}/get-daily?lat=&lon=` | Basic auth, same retry policy |

Market and station reference data comes from a separate database the node connects to directly, not from these APIs.

**Assistant-facing** — `GET /api/chat/` streams the answer as server-sent events. Supporting routes: `POST /api/token` (plus `/api-key` and `/play-integrity`), `POST /api/transcribe/`, `POST /api/tts/`, `POST /api/telemetry/{events,feedback,error}`.

**Configuration trap:** the variable named `BAP_ENDPOINT` holds the **provider node's** URL. The assistant is the BAP; that setting is where it sends to, not what it is.

---

## 9. What this document doesn't cover

- **MahaVistaar's network node** — run from a shared repository that was not examined. Don't assume it behaves like BharatVistaar's.
- **The underlying network component** — the off-the-shelf Beckn software the provider node runs behind was never inspected. Claims that signing and registry lookup "happen there" are assumptions.
- **The general knowledge search service** — an external hybrid-search service holding the bulk of advisory content, treated here as a black box.

---

*Traced August 2026 against the running codebase. Where something was inferred rather than confirmed, it says so.*
