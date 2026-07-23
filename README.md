# realm-legal

Legal document intelligence for Embabel Me: **typed, graph-cached clause extraction** over ingested
contracts.

```cypher
MATCH (:Concept {value:'acme'})-[:RELEVANT_TO]->(d:Document)
MATCH (d)-[:HAS_CLAUSES]->(c:Clause)
WHERE c.category = 'Cap On Liability'
RETURN c.text
```

## What it ships

| File | What |
|---|---|
| `types/legal.yml` | The `Clause` type: CUAD-taxonomy `category`, verbatim `text`, normalized `answer`, display `name`, own `section`/`title`, cited `references`, `sourceChunkId` provenance; the `HAS_CLAUSES` join anchored on `Document`; few-shot text-to-cypher examples. |
| `producers/legal.yml` | `clauseExtraction` — a `kind: extract` producer (`cache: {kind: graph}`): collects the contract's chunks per document, extracts every clause as a typed record in one model call, persists each record as a committed, scope-stamped **entity** node with a real `HAS_CLAUSES` containment edge, and commits a real `(citing)-[:REFERS_TO]->(cited)` edge for every cross-reference. |

## Query examples

Clauses are **first-class entities**: extracted lazily on the first traversal, ordinary graph
thereafter — every query below works in plain Cypher once a contract has been traversed, and
`RETURN c` (the whole node) renders the clauses as entities rather than a report about them.

```cypher
-- The clauses themselves, as entities
MATCH (d:Document) WHERE toLower(d.title) CONTAINS 'acme'
MATCH (d)-[:HAS_CLAUSES]->(c:Clause)
RETURN c

-- Liability clauses of one contract
MATCH (d:Document) WHERE toLower(d.title) CONTAINS 'acme'
MATCH (d)-[:HAS_CLAUSES]->(c:Clause)
WHERE c.category IN ['Cap On Liability', 'Uncapped Liability']
RETURN c

-- A clause AND the clauses it cites (REFERS_TO is a real edge)
MATCH (d:Document) WHERE toLower(d.title) CONTAINS 'acme'
MATCH (d)-[:HAS_CLAUSES]->(c:Clause)-[:REFERS_TO]->(cited:Clause)
RETURN c, cited

-- "What does the section cited by the liability cap actually say?"
MATCH (d:Document) WHERE toLower(d.title) CONTAINS 'acme'
MATCH (d)-[:HAS_CLAUSES]->(:Clause {category:'Cap On Liability'})-[:REFERS_TO]->(cited:Clause)
RETURN cited

-- Reverse: every clause that leans on the governing-law section
MATCH (d:Document) WHERE toLower(d.title) CONTAINS 'acme'
MATCH (d)-[:HAS_CLAUSES]->(gov:Clause {category:'Governing Law'})<-[:REFERS_TO]-(citing:Clause)
RETURN citing, gov

-- Compare a category across every contract in the corpus
MATCH (:Concept {value:'contract'})-[:RELEVANT_TO]->(d:Document)
MATCH (d)-[:HAS_CLAUSES]->(c:Clause {category:'Renewal Term'})
RETURN d.title AS contract, c

-- Digest instead of rows: summarize what the termination clauses say
MATCH (d:Document) WHERE toLower(d.title) CONTAINS 'acme'
MATCH (d)-[:HAS_CLAUSES]->(c:Clause)
WHERE c.category IN ['Termination For Convenience', 'Post-Termination Services']
RETURN summarize(c.text, 'What are the termination terms?') AS digest
```

Or just ask in chat — the few-shot examples in `types/legal.yml` teach the generator these shapes:

- *"show me the liability clauses in my acme contract"*
- *"what does the section cited by the liability cap say?"*
- *"which clauses reference the governing-law section?"*
- *"does my acme agreement auto-renew, and what notice does it need?"*
- *"summarize the termination clauses in my acme contract"*

## Behaviour

- **Extract once, query forever**: the first `HAS_CLAUSES` traversal extracts and persists; repeats
  are plain graph hits until the document's text changes, at which point the stale clause set is
  REPLACED (never orphaned).
- **Clauses are entities, part of the model**: committed with an `__Entity__` stamp, a display
  `name`, a real containment edge, and real `REFERS_TO` citation edges — after first traversal they
  are ordinary graph, visible to plain Cypher and entity surfaces with no engine involved.
- **Honest-empty**: a document with no recognizable clauses yields no rows — never fabricated ones.
- **Composes with the whole engine**: relevance chains (`RELEVANT_TO → HAS_CLAUSES`), parallel joins
  (`HAS_CLAUSES` + `HAS_SUMMARY` off one document), inline aggregation (`summarize(c.text, '…')` —
  "summarize the termination clauses"), subjective filtering (`WHERE ai.relevant(c, '…')`), and the
  demand budget (a `LIMIT 1` ask extracts only until satisfied).

## Clause vocabulary

The `category` values are CUAD's 41 expert labels (The Atticus Project, CC-BY-4.0) — Parties,
Agreement Date, Renewal Term, Governing Law, License Grant, Cap On Liability, Termination For
Convenience, Anti-Assignment, Audit Rights, Insurance, Source Code Escrow, … — the taxonomy
practicing lawyers use to review contracts, and the one the extraction prompt enumerates.

## Tests (in embabel/me)

- `ClauseExtractionFaithfulNeo4jIT` — engine contract (faked model): typed rows, per-record graph
  cache, stale-set replacement, honest-empty, steered-transient, cross-tenant, demand budget.
- `LegalPackFaithfulNeo4jIT` — THIS realm's YAML drives extraction end-to-end via `packDirs`.
- `ClauseCombinationsFaithfulNeo4jIT` — relevance chains, clause+summary composition,
  summarize-clauses-about-X, per-document digests.
- `ContractClausesLlmIT` (opt-in, real model) — extraction over the real Corio × Commerce One
  agreement (CUAD/EDGAR) asserted against CUAD's lawyer annotations.
