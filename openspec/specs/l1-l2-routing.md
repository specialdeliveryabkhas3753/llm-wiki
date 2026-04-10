# Spec: L1/L2 Routing — Knowledge Layer Decision Logic

## Description

When new knowledge is extracted (during ingest, query, or user interaction), the system
must decide whether it belongs in L1 (Claude Code memory, auto-loaded every session) or
L2 (wiki, queried on demand). This is the core architectural decision that makes the
dual-layer cache effective. Wrong routing degrades the system: too much in L1 = slow
session starts and context bloat; too little in L1 = repeated mistakes.

---

## Requirements

### The Routing Rule

- REQ-300: The system SHALL evaluate every extracted fact against the routing question:
  "If the LLM does not know this right now, what happens?"
- REQ-301: If the consequence is **data loss, security incident, or production
  failure**, the fact SHALL be routed to L1.
- REQ-302: If the consequence is **embarrassing output** (wrong name, wrong address,
  incorrect brand voice), the fact SHALL be routed to L1.
- REQ-303: If the consequence is **incorrect but easily correctable** (wrong count,
  outdated detail), the fact SHALL be routed to L2.
- REQ-304: If the consequence is **missing context requiring a follow-up question**,
  the fact SHALL be routed to L2.

### L1 Content Categories

- REQ-310: Operational rules and gotchas (things that prevent mistakes in the moment)
  SHALL be stored in L1.
- REQ-311: User identity and preferences (name spelling, address, communication style)
  SHALL be stored in L1.
- REQ-312: Credentials and secrets (API tokens, passwords, connection strings)
  MUST be stored in L1. They MUST NOT be stored in L2 under any circumstances.
- REQ-313: Tool-specific quirks that apply in every session (e.g., "PM2 reload does
  not work with npm start") SHALL be stored in L1.

### L2 Content Categories

- REQ-320: Project details and timelines SHALL be stored in L2.
- REQ-321: Workflow documentation SHALL be stored in L2.
- REQ-322: Research and learning notes SHALL be stored in L2.
- REQ-323: Business intelligence and strategy SHALL be stored in L2.
- REQ-324: Historical decisions and rationale SHALL be stored in L2.

### Security Boundary

- REQ-330: L1 memory directory MUST be git-excluded (typically at
  `~/.claude/projects/*/memory/` which is not in the repo).
- REQ-331: L2 wiki MUST be assumed git-tracked. All L2 content is potentially
  visible in version control history.
- REQ-332: The system SHALL treat the L1/L2 boundary as a hard security boundary
  for credentials. There is no "soft" credential storage in L2.

### L1 Size Management

- REQ-340: L1 SHOULD contain 10-20 files (optimal range).
- REQ-341: If L1 exceeds approximately 30 files, the system SHOULD recommend an
  audit to move contextual knowledge to L2.
- REQ-342: Each L1 file SHOULD cover one topic and be concise (a few lines, not
  pages of detail).

### Boundary Evolution

- REQ-350: Knowledge MAY be promoted from L2 to L1 when the same mistake is
  repeated across multiple sessions (pattern: operational gotcha discovered the
  hard way).
- REQ-351: Knowledge MAY be demoted from L1 to L2 when it becomes historical
  context rather than an active operational rule (pattern: project completes,
  credentials rotated).
- REQ-352: Multiple related L1 files SHOULD be merged when they cover the same
  system or topic, to keep L1 lean.
- REQ-353: The `/wiki lint` command SHOULD flag L1 files not referenced in 90+
  days as candidates for L2 demotion or deletion.
- REQ-354: The `/wiki lint` command SHOULD flag L2 pages queried in every session
  as candidates for L1 promotion.

### Routing During Ingest

- REQ-360: During /wiki ingest Phase 1, the system SHALL apply the routing rule
  (REQ-300-304) to each extracted fact.
- REQ-361: Facts routed to L1 SHALL NOT be written to wiki pages. The system SHALL
  instead recommend saving them to the memory directory.
- REQ-362: Facts routed to L2 SHALL proceed through the normal ingest pipeline
  (Phases 2-5).
- REQ-363: If a source contains both L1 and L2 facts, the system SHALL process
  L2 facts via ingest AND separately recommend L1 facts for memory storage.
- REQ-364: The system SHOULD present the routing recommendation to the user before
  writing, especially for ambiguous cases (severity:: important facts that could
  go either way).

---

## Scenarios

### Scenario 1: Operational gotcha — routes to L1

```
GIVEN the user ingests "SSH max 3 calls to VPS, otherwise OOM reboot"
WHEN the system evaluates L1/L2 routing
THEN the consequence of not knowing is "production failure" (OOM reboot)
AND the system SHALL recommend: "This is an L1 candidate (operational gotcha).
    Save to memory, not wiki."
AND the system SHALL NOT create a wiki page for this fact
```

### Scenario 2: Project timeline — routes to L2

```
GIVEN the user ingests "Book chapter 1 deadline is April 20"
WHEN the system evaluates L1/L2 routing
THEN the consequence of not knowing is "missing context, ask follow-up"
AND the system SHALL route this to L2 (wiki)
AND proceed with normal ingest to create/update a project page
```

### Scenario 3: Credential in source — hard L1 boundary

```
GIVEN the user ingests text containing "Strapi API token: abc123xyz789..."
WHEN the system evaluates L1/L2 routing
THEN the system SHALL identify this as a credential (REQ-312)
AND the system MUST route to L1
AND the system MUST NOT write the token to any wiki page
AND the system SHALL recommend: "Credential detected. Save to L1 memory only.
    Wiki is git-tracked."
```

### Scenario 4: Mixed source — L1 and L2 facts

```
GIVEN the user ingests a deployment runbook containing:
    - "Always stop ClamAV before deploy" (operational gotcha)
    - "Deploy script is at scripts/deploy-vps.sh" (project context)
    - "VPS IP: 84.234.21.71" (infrastructure detail)
    - "API token: Bearer xyz..." (credential)
WHEN the system evaluates L1/L2 routing
THEN "Stop ClamAV before deploy" SHALL be recommended for L1 (prevents OOM)
AND "API token" MUST be recommended for L1 (credential, hard boundary)
AND "Deploy script path" SHALL be routed to L2 (project context)
AND "VPS IP" MAY go to either L1 or L2 (borderline: wrong IP = failed deploy,
    but easily correctable)
AND the system SHALL process L2 facts via ingest
AND separately list L1 recommendations for user action
```

### Scenario 5: User identity — routes to L1

```
GIVEN the user says "My name is spelled Goekce with a cedille: Goekce"
WHEN the system evaluates L1/L2 routing
THEN the consequence of not knowing is "embarrassing output" (misspelled name)
AND the system SHALL recommend L1 storage
AND the system SHALL NOT create a wiki page for name spelling
```

### Scenario 6: L1 bloat detected — audit recommended

```
GIVEN the L1 memory directory contains 35 files
WHEN the user runs /wiki lint or /wiki status
THEN the system SHALL warn: "L1 has 35 files (recommended: 10-20, audit at 30+).
    Review for candidates to demote to L2."
AND the system SHOULD list the oldest/least-referenced L1 files as demotion candidates
```

### Scenario 7: L2 page frequently queried — promotion candidate

```
GIVEN the wiki page Wiki/Tech/Deployment is queried in 8 of the last 10 sessions
WHEN the user runs /wiki lint
THEN the system SHALL flag the page as an L1 promotion candidate (info)
AND suggest: "Wiki/Tech/Deployment is queried almost every session.
    Consider promoting key rules to L1 memory."
```

### Scenario 8: Ambiguous routing — user decision needed

```
GIVEN the user ingests "Strapi port must be 1338 everywhere"
WHEN the system evaluates L1/L2 routing
THEN the consequence could be "failed deploy" (L1) or "easily fixable config" (L2)
AND the system SHALL present both options to the user:
    "L1 (auto-loaded): Prevents port mismatch every session.
     L2 (wiki): Documented but only loaded when querying Strapi."
AND the system SHALL wait for user confirmation before routing
```

---

## Acceptance Criteria

- [ ] Every extracted fact is evaluated against the routing rule before storage
- [ ] Credentials NEVER reach L2 wiki pages (hard security boundary)
- [ ] L1 candidates are recommended to the user, not silently written to wiki
- [ ] Mixed sources produce both L2 ingest AND L1 recommendations
- [ ] L1 size warnings trigger at ~30 files
- [ ] Ambiguous cases are presented to user for decision
- [ ] Works with both Logseq and Obsidian L2 backends
- [ ] Boundary evolution (promote/demote) is suggested during lint

---

## Dependencies

- `llm-wiki.yml` must specify `memory_path` for L1 location
- specs/ingest.md Phase 1 calls this routing logic
- specs/lint.md Rules 6 (credential leak) and 9 (L1/L2 duplicates) enforce boundaries
