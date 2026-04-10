# Spec: /wiki query — Knowledge Retrieval & Synthesis

## Description

The query command searches the wiki for information relevant to a user's question,
synthesizes an answer from multiple sources, and optionally creates new pages to fill
knowledge gaps. It is the primary read path — the counterpart to ingest (write path).

---

## Requirements

### Phase 1: Search

- REQ-400: The system SHALL parse the user's question to identify relevant
  namespaces, entity names, and keywords.
- REQ-401: The system SHALL glob for candidate pages matching identified
  namespaces (e.g., question about "Strapi" → search Wiki/Tech/ first).
- REQ-402: The system SHALL grep for keywords across all wiki pages to find
  matches beyond the primary namespace.
- REQ-403: The system SHALL read the top 3-5 most relevant pages based on
  keyword density and namespace match.
- REQ-404: The system SHALL load at most 3 pages simultaneously (JIT retrieval).
  If more than 3 are relevant, read in batches.
- REQ-405: The system SHOULD also read L1 Memory files when the question touches
  topics that may have operational rules or gotchas stored there.
- REQ-406: The system SHALL read `llm-wiki.yml` first to determine tool mode,
  wiki path, and memory path.

### Phase 2: Synthesize

- REQ-410: The system SHALL combine information from multiple wiki pages into
  a coherent answer, not just dump raw page content.
- REQ-411: The system SHALL check the `confidence::` property of each source page
  and weight high-confidence sources over low-confidence ones.
- REQ-412: The system SHALL check the `updated::` date of each source page and
  flag sources older than 90 days as potentially stale.
- REQ-413: The system SHALL note when sources contradict each other and present
  both perspectives rather than silently choosing one.
- REQ-414: The system MUST NOT fabricate information that is not present in the
  wiki or L1 memory. If the wiki does not contain an answer, say so.

### Phase 3: Optional Write-Back

- REQ-420: If the query reveals a knowledge gap (question has no answer in the
  wiki), the system SHOULD offer to create a new page.
- REQ-421: If the synthesis produces a useful summary that does not exist as a
  wiki page, the system SHOULD offer to save it as a new page.
- REQ-422: The system MUST NOT write any pages without explicit user confirmation.
  Write-back is always opt-in, never automatic.
- REQ-423: Write-back operations SHALL follow the same rules as ingest Phase 3
  (required properties, append-only, cross-references, hub updates).

### Phase 4: Output

- REQ-430: The system SHALL include source attribution in every answer, listing
  the wiki pages used: "Sources: [[Wiki/Tech/Strapi]], [[Wiki/Reference/Workflows]]"
- REQ-431: The system SHALL explicitly flag stale sources (updated:: > 90 days)
  with a warning: "Note: [[Wiki/Tech/X]] was last updated [date] and may be outdated."
- REQ-432: The system SHALL explicitly flag low-confidence sources with a warning:
  "Note: [[Wiki/Learning/Y]] has confidence:: low — verify before acting on this."
- REQ-433: The system SHOULD suggest related pages that the user might want to
  explore further, even if they were not directly used in the answer.
- REQ-434: If no relevant pages are found, the system SHALL clearly state:
  "No information found in the wiki for this topic." and offer to create a page
  via write-back (REQ-420).

---

## Scenarios

### Scenario 1: Simple question — single page match

```
GIVEN the wiki contains Wiki/Tech/Strapi with content about Strapi CMS
AND the page has confidence:: high and updated:: 2026-04-01
WHEN the user runs /wiki query "what port does Strapi use?"
THEN the system SHALL read Wiki/Tech/Strapi
AND synthesize an answer from that page's content
AND output: "Sources: [[Wiki/Tech/Strapi]]"
AND NOT flag the source as stale (updated 9 days ago)
```

### Scenario 2: Complex question — multi-page synthesis

```
GIVEN the wiki contains:
    - Wiki/Tech/Deployment (deploy process, VPS details)
    - Wiki/Tech/Strapi (CMS configuration, port settings)
    - Wiki/Reference/Workflows (deploy workflow steps)
WHEN the user runs /wiki query "how do I deploy a new blog post?"
THEN the system SHALL read all 3 pages (batching if needed)
AND combine deployment steps from Workflows with Strapi API details
AND present a coherent answer (not just 3 raw page dumps)
AND output: "Sources: [[Wiki/Tech/Deployment]], [[Wiki/Tech/Strapi]],
    [[Wiki/Reference/Workflows]]"
```

### Scenario 3: No results — wiki gap detected

```
GIVEN the wiki has no pages mentioning "Kubernetes"
WHEN the user runs /wiki query "how is Kubernetes configured?"
THEN the system SHALL state: "No information found in the wiki for Kubernetes."
AND offer: "Would you like me to create a Wiki/Tech/Kubernetes page?"
AND NOT fabricate an answer about Kubernetes
```

### Scenario 4: Stale source flagged

```
GIVEN Wiki/Tech/Docker has updated:: 2025-12-01 and confidence:: high
AND today is 2026-04-10 (131 days old, exceeds 90-day threshold)
WHEN the user runs /wiki query "what Docker version are we using?"
THEN the system SHALL use the page to answer the question
AND flag: "Note: [[Wiki/Tech/Docker]] was last updated 2025-12-01
    (131 days ago) and may be outdated."
```

### Scenario 5: Low-confidence source flagged

```
GIVEN Wiki/Learning/Rust has confidence:: low
WHEN the user runs /wiki query "what Rust resources do we have?"
THEN the system SHALL use the page to answer
AND flag: "Note: [[Wiki/Learning/Rust]] has confidence:: low —
    verify before acting on this."
```

### Scenario 6: Write-back offered and accepted

```
GIVEN the user asks /wiki query "what is our Redis setup?"
AND the wiki has no Redis page
AND the user previously ingested Redis information in L1 memory
WHEN the system reports "No wiki page for Redis"
AND offers "Would you like me to create Wiki/Tech/Redis?"
AND the user confirms "yes"
THEN the system SHALL create Wiki/Tech/Redis with required properties
AND update the Wiki/Tech hub page
AND follow all ingest Phase 3 rules (append-only, cross-refs, etc.)
```

### Scenario 7: Write-back offered and declined

```
GIVEN the same setup as Scenario 6
WHEN the system offers to create Wiki/Tech/Redis
AND the user declines "no"
THEN the system SHALL NOT create any pages
AND SHALL NOT modify any existing pages
```

### Scenario 8: L1 Memory supplements wiki answer

```
GIVEN Wiki/Tech/Deployment contains general deploy documentation
AND L1 Memory file feedback_deploy_ram.md contains "Stop ClamAV before deploy"
WHEN the user runs /wiki query "anything I should know before deploying?"
THEN the system SHALL read the wiki page AND the L1 memory file
AND include both in the synthesized answer
AND attribute: "Sources: [[Wiki/Tech/Deployment]] + L1 Memory (deploy gotcha)"
```

### Scenario 9: Contradicting sources

```
GIVEN Wiki/Tech/Strapi says "Strapi runs on port 1337"
AND Wiki/Reference/Workflows says "Strapi API is at localhost:1338"
WHEN the user runs /wiki query "what port is Strapi on?"
THEN the system SHALL present both values
AND note the contradiction: "Wiki sources disagree: Wiki/Tech/Strapi says 1337,
    Wiki/Reference/Workflows says 1338. Verify which is current."
AND NOT silently pick one value
```

### Scenario 10: Batching when many pages match

```
GIVEN a query matches 7 relevant wiki pages
WHEN the system needs to read them for synthesis
THEN the system SHALL read at most 3 pages at a time
AND process in batches: read 3 → extract relevant info → read next 3 → read last 1
AND synthesize from all extracted information
AND list all 7 pages in source attribution
```

---

## Acceptance Criteria

- [ ] All 4 phases execute in order (Search, Synthesize, Write-Back, Output)
- [ ] Namespace-aware search (question about tech → search Wiki/Tech/ first)
- [ ] Max 3 pages loaded simultaneously
- [ ] Answers synthesized from multiple sources (not raw dumps)
- [ ] Confidence levels checked and low-confidence flagged
- [ ] Staleness checked and old sources flagged (90-day threshold)
- [ ] Contradictions surfaced, not silently resolved
- [ ] No fabrication — "not found" when wiki has no answer
- [ ] Write-back requires explicit user confirmation
- [ ] Source attribution in every answer
- [ ] L1 Memory consulted when relevant
- [ ] Works in both Logseq and Obsidian modes

---

## Dependencies

- `llm-wiki.yml` must exist and be valid (see specs/config.md)
- specs/schema.md defines valid page properties checked during synthesis
- specs/ingest.md Phase 3 rules apply to write-back operations
- specs/l1-l2-routing.md defines when L1 Memory is relevant
