---
name: comres
description: >
  Multi-agent skill for mining winning and non-winning shortlisted entries across 80+ competitions
  and journals. Extracts structural logic and transposes into localized winning proposals.
  Use when user says "comres", "mine competition entries", "find shortlisted essays",
  "build proposal from competition logic", or invokes /skills comres.
---

# /skills comres — Competitive Research Mining & Proposal Transposition Engine

Multi-agent pipeline. 89 target endpoints. Mines ALL historical entries (every year a competition has been held). Extracts logic. Transposes to winning proposals. Sub-agent parallelization for full-year coverage.

## 1. Multi-Agent System Architecture

### 1.1 Agent Role Matrix

| Agent | Role | Timeout |
|-------|------|---------|
| **Orchestrator** | State machine, IPC routing, payload aggregation, sub-agent dispatch | Session |
| **SearchRetrieval** | Multi-endpoint API/MCP execution, domain DB targeting | 30s/endpoint |
| **NLPParser** | PDF/text ingestion, argument graph mapping, logic decomposition | 60s/doc |
| **Localizer** | Global-to-local transposition, regional parameter mapping | 45s |
| **VictoryUpgrade** | Gap analysis, empirical data injection, citation augmentation | 90s |

### 1.1.1 Sub-Agent Parallelization

Orchestrator dispatches sub-agents via `task` tool (subagent_type: "explore" or "general"). Parallelization mandatory for full-year coverage.

**Dispatch strategy:**

| Work Unit | `task` subagent_type | Parallelism | Notes |
|-----------|----------------------|-------------|-------|
| Registry batch (20 endpoints) | `explore` | Up to 5 concurrent | Each sub-agent handles ~20 endpoints, ALL years |
| Single endpoint year crawl | `explore` | Up to 10 concurrent | Per-endpoint year enumeration |
| NLP parsing batch (10 entries) | `general` | Up to 5 concurrent | Each parses ~10 docs in parallel |
| Cross-entry synthesis | `general` | 1 (sequential) | Requires all parsed data |
| Localizer | `general` | 1 (sequential) | Depends on synthesis |
| VictoryUpgrade | `general` | 1 (sequential) | Depends on localized skeleton |

**Sub-agent dispatch example (Orchestrator calls `task`):**

```
task(
  description="SearchRetrieval batch: endpoints 1-20",
  subagent_type="explore",
  prompt="Search registry entries 1-20. For EACH entry:
    1. Fetch the results/archive page
    2. Determine competition start year by crawling to earliest available page
    3. Enumerate ALL years from start year to 2026
    4. For EACH year: fetch that year's results page, extract ALL entries (1st–5th, HM, shortlisted, commended, semi-finalist)
    5. Parse PDFs if content is PDF. Use browser for JS-rendered pages.
    6. Return JSON array per schema 7.2 with year field populated for every entry.
    NO year skipping. NO recency filtering. Complete historical sweep.
    Domains to search: [idea keywords]"
)
```

Orchestrator fires 4–5 such `task` calls in parallel, then collects results.

**Aggregation:**

Sub-agent results collected by Orchestrator. Deduplicated across sub-agents by URL+title+year hash. Merged into unified entry pool before pipeline continues.

**Aggregation:**

Sub-agent results collected by Orchestrator. Deduplicated across sub-agents by URL+title+year hash. Merged into unified entry pool before pipeline continues.

### 1.2 State Machine

IDLE → PARSE_INPUT → CLASSIFY_DOMAIN → GENERATE_QUERIES → SEARCH_EXECUTE → FILTER_ENTRIES → EXTRACT_LOGIC → ANALYZE_GAPS → TRANSPOSE_LOCALIZE → UPGRADE_VICTORY → ASSEMBLE_OUTPUT → DONE

Transitions atomic. Rollback on failure. Exponential backoff: 3 retries, 2s/4s/8s.

### 1.3 IPC Protocol

All inter-agent communication uses typed JSON payloads. No shared memory.

```json
{
  "payload_version": "1.0",
  "source_agent": "string",
  "target_agent": "string",
  "state_id": "uuid",
  "data": {},
  "errors": [],
  "retry_count": 0,
  "timestamp": "ISO-8601"
}
```

### 1.4 Error Handling

| Error Class | Action | Fallback |
|-------------|--------|----------|
| Endpoint unreachable | Retry 3x, skip | Log, continue |
| PDF parse failure | OCR fallback | Flag partial, continue |
| No results for domain | Broaden query 1 tier | Empty domain report |
| Localization conflict | Pause, prompt user | — |
| Timeout | Kill agent, emit partial | Use last valid state |

---

## 2. Parameter Parsing & Taxonomy Mapping

### 2.1 Command Syntax

```
/skills comres [idea] [--region=<region>] [--domain=<domain>]
```

- `[idea]` — Free-text research idea. Required. Max 500 chars.
- `--region=<region>` — ISO 3166-1 alpha-2 or free-text. Default: auto-detect.
- `--domain=<domain>` — Override auto-classification. Keys 1–7.

### 2.2 Classification Taxonomy

| Cat | Category | Primary Keywords | Secondary Keywords |
|-----|----------|------------------|--------------------|
| 1 | Multidisciplinary / Global Flagships | development, sustainability, SDGs, cross-border | equity, innovation, multilateral |
| 2 | Academic / University Prizes | research, thesis, methodology, literature review | peer-reviewed, citation, theoretical |
| 3 | Economics / Policy / Global Affairs | economic theory, policy, market, fiscal, trade | macroeconomics, regulation |
| 4 | Philosophy / Law / Ethics | ethics, moral, legal framework, rights, justice | deontology, utilitarianism |
| 5 | STEM / Science | experimental, hypothesis, empirical, algorithm | biotech, quantum, engineering |
| 6 | Humanities / Creative / Environment | narrative, literary, historical, ecological | prose, poetry, environmental justice |
| 7 | Open Access Student Research | undergraduate research, student journal, peer-review | lab study, survey methodology |

### 2.3 Dynamic Query Generator

Per domain emit 10+ Boolean query strings. Base format:

```
"{idea_keywords}" AND ("shortlist" OR "runner-up" OR "honorable mention" OR "finalist" OR "commended" OR "semi-finalist" OR "2nd place" OR "3rd place")
```

Query mutations:
1. Exact phrase match with competition names from registry
2. Broadened keyword expansion (synonyms, hypernyms)
3. Site-specific: `site:domain.com "shortlist" OR "finalist"`
4. PDF-specific: `filetype:pdf "honorable mention" OR "commended"`
5. Year-comprehensive: search ALL years from competition inception to 2026 — no year lower bound
6. Exclusion: none on placement (all placements including 1st place included)
7. Author-context: `"{idea}" AND "student" AND ("essay" OR "paper" OR "proposal")`
8. Platform-specific: domain-scoped per registry entry
9. Cross-category: combine two adjacent categories
10. Historical deep-crawl: for each endpoint, enumerate year-by-year results pages going back to earliest available year

---

## 3. Target Database Registry (89 Endpoints)

89 hardcoded target endpoints. Each entry: name, domain, URL pattern, content type, scrape method.

**CRITICAL: Full Historical Coverage.** For each endpoint:
1. Determine the earliest year the competition published results (auto-discover by crawling archive to earliest page)
2. Crawl EVERY year from start year through 2026
3. Extract ALL qualifying entries per year (winners, finalists, HM, shortlisted, etc.)
4. Year-by-year crawl mandatory — no skipping, no sampling, no recency filtering

Different winners/finalists/HM each year. Full historical sweep mandatory regardless of time or cost. Use sub-agents (§1.1.1) to parallelize across endpoints and years.

### Category A: Global / Multidisciplinary Flagships (1–11)

| # | Competition | Domain | URL Pattern | Content | Scrape |
|---|-------------|--------|-------------|---------|--------|
| 1 | Geneva Challenge | graduateinstitute.ch | `/challenges/advancing-development-goals` | Shortlist PDFs | PDF parse |
| 2 | St. Gallen Symposium Essay | symposium.org | `/global-essay-competition` | Top 100 / top 10 | HTML+JS |
| 3 | John Locke Institute Essay | johnlockeinstitute.com | `/essay-competition` | Shortlisted/commended | HTML |
| 4 | MIT Solve | solve.mit.edu | `/challenges` | Shortlisted solutions | API |
| 5 | Trust for Sustainable Living | trustforsustainableliving.org | `/essay-competition` | Shortlist + HM | HTML |
| 6 | Write the World | writetheworld.org | `/competitions` | Shortlisted youth essays | HTML paginated |
| 7 | Goi Peace Foundation | goipeace.or.jp | `/essay-contest` | Top 10 + HM | HTML+PDF |
| 8 | Harvard Crimson Global Essay | hcgec.org | `/results` | Regional finalists, top 10 | HTML |
| 9 | Immerse Education | immerse.education | `/essay-competition` | Shortlisted 10+ disciplines | HTML |
| 10 | Ayn Rand Institute Contests | aynrand.org | `/essay-contests` | Finalist + HM | PDF archive |
| 11 | The Fountain Essay Contest | thefountain.org | `/contest` | Top 5 + HM | HTML |

### Category B: Academic / University Prizes (12–29)

| # | Competition | Domain | URL Pattern | Content | Scrape |
|---|-------------|--------|-------------|---------|--------|
| 12 | Cambridge Re:think | cambridge-rethink.org | `/shortlists` | Ethics/Econ shortlists | HTML |
| 13 | Harvard International Review | hir.harvard.edu | `/essay-competition` | Finalists (policy) | PDF+HTML |
| 14 | Yale International Relations | yira.yale.org | `/publications` | Finalists (global) | HTML |
| 15 | Columbia Undergrad Law | culr.law.columbia.edu | `/submissions` | HS entries (legal) | PDF |
| 16 | Minds Underground | mindsunderground.com | `/essays` | Commended (STEM/Hum) | HTML |
| 17 | Trinity Cambridge (Robson/Gould) | trin.cam.ac.uk | `/prizes` | Commended archive | PDF |
| 18 | Peterhouse Cambridge (Vellacott/Kelvin/Campion) | peterhouse.cam.ac.uk | `/prizes` | Finalists (hist/sci) | PDF |
| 19 | St Hugh's Oxford (Julia Wood) | st-hughs.ox.ac.uk | `/prizes` | Runner-ups (history) | PDF |
| 20 | Corpus Christi Oxford | ccc.ox.ac.uk | `/prizes` | Commended (literature) | PDF |
| 21 | Fitzwilliam Cambridge | fitz.cam.ac.uk | `/prizes` | Highly commended | PDF |
| 22 | Girton Cambridge Humanities | girton.cam.ac.uk | `/humanities-prize` | Shortlists archive | PDF |
| 23 | Marshall Society Cambridge | marshallsociety.com | `/essay-prize` | Econ shortlists | HTML |
| 24 | Oxford Uehiro Prize Ethics | uehiro.ox.ac.uk | `/prize` | Runner-ups | PDF |
| 25 | Oxford Scientist Writing | oxfordscientist.com | `/competition` | Finalists | HTML |
| 26 | Stanford Kalanithi Medicine | med.stanford.edu | `/kalanithi-prize` | HM (med humanities) | PDF |
| 27 | Dartmouth Writing | dartmouth.edu | `/writing-prize` | Runner-ups (public health) | PDF |
| 28 | Princeton Ten-Minute Play | princeton.edu | `/theatre-prize` | HM (dialogue) | PDF |
| 29 | Berkeley Arch Design | berkeley.edu | `/arch-design-prize` | Finalists (spatial) | PDF |

### Category C: Economics / Policy / Global Affairs (30–39)

| # | Competition | Domain | URL Pattern | Content | Scrape |
|---|-------------|--------|-------------|---------|--------|
| 30 | IEA Essay | iea.org.uk | `/essay-competition` | Commended archive | HTML+PDF |
| 31 | RES Young Economist | res.org.uk | `/young-economist` | Shortlists | HTML |
| 32 | Peter Drucker Challenge | druckerchallenge.org | `/challenge` | Top 10 proposals | HTML+PDF |
| 33 | World Bank Youth Summit | youthsummit.worldbank.org | `/pitch` | Finalist pitches | HTML |
| 34 | Carnegie Council Ethics | carnegiecouncil.org | `/essay-contest` | HM | HTML |
| 35 | AFSA Foreign Service Journal | afsa.org | `/student-essay-contest` | Runner-ups | HTML |
| 36 | World Historian WHA | worldhistory.org | `/essay-prize` | Runner-ups | HTML |
| 37 | EUCYS | eucys.eu | `/projects` | Finalists archive | HTML+PDF |
| 38 | Chatham House Prize | chathamhouse.org | `/prizes` | Submissions archive | HTML |
| 39 | Fraser Institute Student | fraserinstitute.org | `/essay-competition` | Runner-ups | HTML+PDF |

### Category D: Philosophy / Law / Ethics (40–46)

| # | Competition | Domain | URL Pattern | Content | Scrape |
|---|-------------|--------|-------------|---------|--------|
| 40 | Elie Wiesel Prize Ethics | eliewieselprize.org | `/submissions` | Finalists archive | PDF |
| 41 | Kemper Human Rights | kemperhumanrights.org | `/essay-competition` | HM | HTML |
| 42 | NCH Northeastern London | nchlondon.org | `/prizes` | Runner-ups (phil/law) | PDF |
| 43 | FQXi Essay Contest | fqxi.org | `/essay-contest` | Accepted submissions | HTML |
| 44 | Baltic Sea Essay | balticsea-essay.org | `/submissions` | Runner-ups (regional) | HTML |
| 45 | Center First Amendment Studies | firstamendmentstudies.org | `/shortlists` | Shortlists (constitutional) | PDF |
| 46 | PLATO Philosophy Essay | plato-philosophy.org | `/essay-prize` | HM | HTML |

### Category E: STEM / Science (47–56)

| # | Competition | Domain | URL Pattern | Content | Scrape |
|---|-------------|--------|-------------|---------|--------|
| 47 | Gravity Research Foundation | gravityresearch.org | `/awards` | HM (theoretical physics) | PDF |
| 48 | DNA Day ASHG | ashg.org | `/dnaday` | HM (genetics) | HTML+PDF |
| 49 | EngineerGirl | engineer-girl.org | `/essay-contest` | Highly commended | HTML |
| 50 | Science Without Borders | sciencewithoutborders.org | `/finalists` | Finalists (marine) | HTML |
| 51 | Breakthrough Junior Challenge | breakthroughjuniorchallenge.org | `/finalists` | Finalist scripts | HTML+video |
| 52 | Emperor Science Award | emperorscience.org | `/proposals` | Finalists (cancer) | PDF |
| 53 | AWM Women Math | awm-math.org | `/biographies` | HM | PDF |
| 54 | NASA Scientist Day | nasa.gov | `/scientist-day` | Finalists archive | HTML |
| 55 | Genes in Space | genesinspace.org | `/proposals` | Finalists (aerospace bio) | PDF |
| 56 | DuPont Challenge | dupontchallenge.com | `/submissions` | Runner-ups | HTML+PDF |

### Category F: Humanities / Creative / Environment (57–75)

| # | Competition | Domain | URL Pattern | Content | Scrape |
|---|-------------|--------|-------------|---------|--------|
| 57 | NYT Student Editorial | nytimes.com | `/student-editorial` | Runner-ups (op-ed) | HTML |
| 58 | Queen's Commonwealth Writing | queenscommonwealth.org | `/writing` | Bronze/Silver/Gold | PDF |
| 59 | Bennington Young Writers | bennington.edu | `/young-writers` | Finalists (creative nonfiction) | PDF |
| 60 | Scholastic Art & Writing | artandwriting.org | `/awards` | Silver/Gold medalists | HTML |
| 61 | National History Day NHD | nhd.org | `/contest` | State finalists | HTML |
| 62 | Concord Review | concordreview.org | `/submissions` | Top history papers | PDF |
| 63 | Bow Seat Ocean Awareness | bowseatocean.org | `/awareness` | HM (environmental) | HTML+PDF |
| 64 | Jane Austen Society | janeaustensociety.org | `/essay-prize` | HM (literary criticism) | PDF |
| 65 | JFK Profile in Courage | jfklibrary.org | `/essay-contest` | Finalists archive | HTML |
| 66 | VFW Voice of Democracy | vfw.org | `/voice-democracy` | State finalists | HTML |
| 67 | Optimist International | optimist.org | `/essay-contest` | Regional finalists | HTML |
| 68 | River of Words | riverofwords.org | `/finalists` | Finalists (ecological) | HTML |
| 69 | YoungArts National | youngarts.org | `/winners` | Merit winners | HTML |
| 70 | FOI Oklahoma First Amendment | foioklahoma.org | `/runner-ups` | Runner-ups | HTML |
| 71 | Nancy Thorp Poetry | nancythorpprize.org | `/honorable-mentions` | HM | HTML |
| 72 | Milberg '53 Poetry | milbergprize.org | `/honorable-mentions` | HM | HTML |
| 73 | EarthX | earthx.org | `/honorable-mentions` | HM (sustainability) | HTML |
| 74 | Young Writers UK | youngwriters.co.uk | `/finalists` | Finalists archive | HTML |
| 75 | Wilbur Smith Adventure | wilbursmithadventure.com | `/shortlists` | Shortlists archive | HTML |

### Category G: Open Access Student Journals (76–89)

| # | Journal | Domain | URL Pattern | Content | Scrape |
|---|---------|--------|-------------|---------|--------|
| 76 | Pioneer Academics | pioneer-academics.com | `/journal` | Published papers | PDF |
| 77 | Lumiere Scholars | lumiere-education.com | `/scholars` | Published essays | PDF |
| 78 | Horizon Academic | horizon-research.org | `/journal` | Published papers | PDF |
| 79 | Polygence Symposium | polygence.com | `/symposium` | Past papers | HTML+PDF |
| 80 | JEI | jei.org | `/issues` | Open access | PDF |
| 81 | NJSR | njsr.org | `/issues` | Open access | PDF |
| 82 | Columbia Junior Science | columbia.edu | `/cjsj` | Published research | PDF |
| 83 | MIT MURJ | murj.mit.edu | `/issues` | Submissions archive | PDF |
| 84 | Stanford SURJ | surj.stanford.edu | `/issues` | Accepted papers | PDF |
| 85 | Dartmouth DUJS | dartmouth.edu | `/dujs` | Open access | PDF |
| 86 | Yale JBM | yjbm.yale.edu | `/issues` | Student research | PDF |
| 87 | Harvard Res Publica | hrespublica.org | `/issues` | Politics essays | PDF |
| 88 | Penn Undergrad Law | law.upenn.edu | `/pulj` | Accepted articles | PDF |
| 89 | JSR | jsr.org | `/issues` | Open access | PDF |

---

## 4. Mining Execution & Filtering Pipeline

### 4.1 Strict Inclusion

Entry qualifies if ANY true:
- 1st place / grand prize winner
- Placed 2nd–5th
- Honorable mention
- Shortlisted / semi-finalist
- Commended / highly commended
- Top 100 / 50 / 20
- Published in student journal

### 4.2 Strict Exclusion

Entry excluded if ANY true:
- Commercially published
- Paywall with no open-access alternative
- Fewer than 200 words

No year exclusion. Every year from competition inception to 2026 included.

### 4.3 Scraping Strategy

| Content Type | Primary | Fallback |
|-------------|---------|----------|
| Static HTML | DOM parse | curl+BeautifulSoup |
| JS-rendered SPA | Browser MCP `browser_snapshot` | `browser_evaluate` Playwright |
| PDF | Stream parse `pdf-parse` | OCR via browser |
| API | HTTP GET/POST | Web scrape fallback |
| Video | Metadata+transcript scrape | Thumbnail+description |

Rate limit: 1 req/2s per domain.

### 4.4 Filter Pipeline

```
RawResults → Deduplicate (URL+title+year hash) → InclusionFilter → ExclusionFilter →
QualityGate (min 200 words, has thesis) → YearTagger → DomainTagger → RankedOutput
```

QualityGate rejects:
- Word count < 200
- No identifiable thesis statement
- Duplicate content (fingerprint hash match)

---

## 5. Logic Extraction & Decomposition Model

### 5.1 NLPParser Output Schema

```json
{
  "entry_id": "string (uuid)",
  "competition_name": "string",
  "competition_category": "1-7",
  "year": "integer",
  "placement": "1st|2nd|3rd|4th|5th|HM|shortlisted|commended|semi-finalist",
  "url": "string",
  "source_domain": "string",
  "raw_text_length": "integer",
  "metadata": {
    "author_name": "string|null",
    "word_count": "integer",
    "has_citations": "boolean",
    "citation_count": "integer",
    "discipline_tags": ["string"]
  },
  "thesis": {
    "statement": "string (extracted thesis sentence)",
    "confidence": "float (0-1)",
    "type": "argumentative|expository|analytical|narrative|empirical"
  },
  "logic_graph": {
    "pillars": [
      {
        "pillar_id": "integer (1-N)",
        "label": "string",
        "claims": [
          {
            "claim_text": "string",
            "evidence_type": "empirical|theoretical|anecdotal|statistical|citation",
            "evidence_source": "string|null",
            "strength": "strong|moderate|weak"
          }
        ],
        "logical_flow": "deductive|inductive|abductive|analogical"
      }
    ],
    "supporting_frameworks": ["string (theory names)"],
    "counterargument_handling": "acknowledged|refuted|ignored|partial"
  },
  "methodology": {
    "approach": "qualitative|quantitative|mixed|computational|literary|normative",
    "models_used": ["string"],
    "data_sources": ["string"],
    "limitations_noted": ["string"]
  },
  "gap_analysis": {
    "critical_weaknesses": [
      {
        "area": "string (evidence|argument|structure|novelty|localization|execution)",
        "description": "string",
        "severity": "critical|major|minor",
        "fix_suggestion": "string"
      }
    ],
    "why_not_first_or_strengths": "string (if winner: key strengths; if non-winner: why not 1st place)",
    "upgrade_potential": "float (0-1, estimated improvement if gaps fixed)"
  }
}
```

### 5.2 Extraction Algorithm

1. **Ingest**: Read full text/PDF content from SearchRetrieval output.
2. **Segment**: Split into abstract, body sections, conclusion, references.
3. **Thesis Identification**: Locate explicit thesis statements (first/last paragraph of intro, topic sentences). Score confidence via cue phrases ("this essay argues", "I will demonstrate", "the central claim is").
4. **Pillar Extraction**: Identify 2–5 core argument clusters via paragraph-topic clustering. Each cluster becomes a pillar.
5. **Claim-Evidence Pairing**: Within each pillar, extract claim sentences and map to adjacent evidence sentences. Tag evidence type.
6. **Methodology Detection**: Pattern-match methodological keywords ("we conducted", "data shows", "drawing on", "Close reading reveals").
7. **Gap Identification**: Cross-reference extracted logic against first-prize scoring rubrics for each competition. Identify: weak evidence chains, missing counterarguments, narrow scope, poor localization, lack of empirical novelty.
8. **Confidence Scoring**: Each extraction element gets 0–1 confidence. Low-confidence elements flagged for manual review.

---

## 6. Transposition & Victory Upgrade Pipeline

### 6.1 Global-to-Local Transposition Matrix

Input: Logic graph from NLPParser + user `--region` flag.

Mapping rules:

| Global Dimension | Local Parameter Source | Transform |
|------------------|----------------------|-----------|
| Economic framework | Regional GDP, Gini index, sector data | Replace macro stats with local equivalents |
| Policy context | National/regional legislation, recent policy shifts | Map global policy args to local legal landscape |
| Cultural assumptions | Hofstede dimensions, regional norms | Reframe cultural references for local audience |
| Case studies | Local precedents, regional examples | Swap international cases for domestic analogues |
| Data sources | Regional databases, government stats portals | Redirect citations to local data repositories |
| Stakeholder analysis | Local institutional actors | Replace global orgs with regional equivalents |
| Language/tone | Academic register norms in target region | Adjust formality, citation style (APA/APA/local) |

Transposition output: Localized proposal skeleton maintaining original logic graph structure with all parameters replaced.

### 6.2 Victory Upgrade Module

Input: Localized skeleton + gap_analysis from NLPParser.

Upgrade operations:

1. **Empirical Injection**: For each "weak" or "moderate" evidence claim, source local empirical data from regional statistics portals. Inject with proper citation.
2. **Counterargument Strengthening**: If `counterargument_handling` = "ignored" or "partial", generate 2–3 strong counterarguments and refutation paragraphs.
3. **Novelty Augmentation**: Compare thesis against existing literature. Identify 1–2 novel angles not present in mined entries. Inject as differentiator.
4. **Methodology Hardening**: If methodology is "literary" or "normative" only, suggest mixed-methods enhancement or empirical validation step.
5. **Structure Optimization**: Reorder pillars for maximum persuasive impact (strongest → weakest → synthesis).
6. **Citation Elevation**: Replace low-tier citations with high-impact journal sources. Target: minimum 3 citations from Q1 journals per pillar.
7. **Executive Summary Generation**: Auto-generate 200-word executive summary optimized for competition judges.
8. **Risk Mitigation Section**: Add section addressing potential objections and limitations proactively.

### 6.3 Output Scoring

Final proposal receives computed score:

```
victory_score = (evidence_strength * 0.25) + (novelty * 0.20) + (localization * 0.20) + 
                (structure * 0.15) + (citation_quality * 0.10) + (counterargument * 0.10)
```

Score 0.0–1.0. Target: ≥ 0.75 for 1st-place candidate.

---

## 7. End-to-End Inter-Agent JSON Payloads

### 7.1 Orchestrator → SearchRetrieval

```json
{
  "payload_version": "1.0",
  "source_agent": "orchestrator",
  "target_agent": "search_retrieval",
  "state_id": "uuid",
  "data": {
    "idea": "string",
    "classified_domains": ["1", "3"],
    "query_strings": ["string"],
    "target_registry_ids": [1, 30, 31, 34],
    "region": "string",
    "max_results_per_endpoint": "unlimited (all years)",
    "year_range": "competition_start_year to 2026 (all years, no exclusion)",
    "filters": {
      "include_1st_place": true,
      "content_types": ["pdf", "html"]
    }
  }
}
```

### 7.2 SearchRetrieval → Orchestrator

```json
{
  "payload_version": "1.0",
  "source_agent": "search_retrieval",
  "target_agent": "orchestrator",
  "state_id": "uuid",
  "data": {
    "entries": [
      {
        "url": "string",
        "title": "string",
        "competition_name": "string",
        "placement": "string",
        "year": "integer",
        "content_type": "pdf|html",
        "raw_text": "string",
        "source_registry_id": "integer",
        "retrieval_status": "success|partial|failed"
      }
    ],
    "summary": {
      "total_endpoints_queried": "integer",
      "total_results": "integer",
      "passed_filter": "integer",
      "failed_endpoints": ["string"]
    }
  }
}
```

### 7.3 Orchestrator → NLPParser

```json
{
  "payload_version": "1.0",
  "source_agent": "orchestrator",
  "target_agent": "nlp_parser",
  "state_id": "uuid",
  "data": {
    "entries_to_parse": [
      {
        "entry_id": "uuid",
        "raw_text": "string",
        "competition_name": "string",
        "competition_category": "1-7",
        "year": "integer",
        "placement": "string"
      }
    ],
    "extraction_config": {
      "extract_thesis": true,
      "extract_pillars": true,
      "extract_methodology": true,
      "run_gap_analysis": true,
      "max_pillars": 5
    }
  }
}
```

### 7.4 NLPParser → Orchestrator

```json
{
  "payload_version": "1.0",
  "source_agent": "nlp_parser",
  "target_agent": "orchestrator",
  "state_id": "uuid",
  "data": {
    "parsed_entries": ["<schema 5.1 entry_id format>"],
    "cross_entry_insights": {
      "common_thesis_patterns": ["string"],
      "shared_methodologies": ["string"],
      "recurrent_gaps": ["string"],
      "dominant_frameworks": ["string"]
    }
  }
}
```

### 7.5 Orchestrator → Localizer

```json
{
  "payload_version": "1.0",
  "source_agent": "orchestrator",
  "target_agent": "localizer",
  "state_id": "uuid",
  "data": {
    "target_region": "string",
    "idea": "string",
    "selected_logic_graphs": ["<schema 5.1 format>"],
    "localization_config": {
      "replace_economic_data": true,
      "replace_case_studies": true,
      "adjust_citation_style": "APA|MLA|Chicago",
      "target_language_register": "formal_academic|professional|accessible"
    }
  }
}
```

### 7.6 Localizer → Orchestrator

```json
{
  "payload_version": "1.0",
  "source_agent": "localizer",
  "target_agent": "orchestrator",
  "state_id": "uuid",
  "data": {
    "localized_proposal": {
      "title": "string",
      "thesis": "string",
      "pillars": [
        {
          "pillar_id": "integer",
          "label": "string",
          "claims": ["string"],
          "localized_evidence": ["string"],
          "local_data_sources": ["string"]
        }
      ],
      "methodology": "string",
      "transposition_notes": "string (what was changed and why)"
    },
    "localization_conflicts": ["string (items needing user decision)"]
  }
}
```

### 7.7 Orchestrator → VictoryUpgrade

```json
{
  "payload_version": "1.0",
  "source_agent": "orchestrator",
  "target_agent": "victory_upgrade",
  "state_id": "uuid",
  "data": {
    "localized_proposal": "<schema 7.6 format>",
    "analysis_entries": ["<schema 5.1 format (includes gap_analysis for non-winners, strengths for winners)>"],
    "upgrade_config": {
      "inject_empirical": true,
      "strengthen_counterarguments": true,
      "add_novelty": true,
      "optimize_structure": true,
      "elevate_citations": true,
      "generate_executive_summary": true,
      "target_victory_score": 0.75
    }
  }
}
```

### 7.8 VictoryUpgrade → Orchestrator (Final Output)

```json
{
  "payload_version": "1.0",
  "source_agent": "victory_upgrade",
  "target_agent": "orchestrator",
  "state_id": "uuid",
  "data": {
    "final_proposal": {
      "title": "string",
      "executive_summary": "string (200 words)",
      "thesis": "string",
      "introduction": "string",
      "pillars": [
        {
          "pillar_id": "integer",
          "label": "string",
          "body": "string",
          "evidence": ["string"],
          "citations": ["string"]
        }
      ],
      "counterarguments_section": "string",
      "methodology": "string",
      "conclusion": "string",
      "references": ["string"],
      "risk_mitigation": "string"
    },
    "victory_metrics": {
      "victory_score": "float (0-1)",
      "evidence_strength": "float",
      "novelty_score": "float",
      "localization_score": "float",
      "structure_score": "float",
      "citation_quality": "float",
      "counterargument_score": "float"
    },
    "paper_links": [
      {
        "title": "string",
        "competition": "string",
        "year": "integer",
        "placement": "string",
        "url": "string (direct link to paper/entry)"
      }
    ],
    "mined_source_entries": [
      {
        "competition": "string",
        "placement": "string",
        "year": "integer",
        "url": "string",
        "contribution": "string (what was extracted from this entry)"
      }
    ],
    "upgrade_changelog": ["string (list of all modifications made)"]
  }
}
```

---

## 8. Final Output Markdown Template

The orchestrator renders the VictoryUpgrade output into this user-facing format:

```markdown
# [Proposal Title]

## Paper Links

All papers/entries found matching the idea, with direct source URLs:

| # | Title | Competition | Year | Placement | URL |
|---|-------|-------------|------|-----------|-----|
| 1 | [entry title] | [competition name] | [year] | [placement] | [url] |
| 2 | [entry title] | [competition name] | [year] | [placement] | [url] |
| ... | ... | ... | ... | ... | ... |

**[N] papers found across [M] competitions, [earliest]–2026**

---

## Mined Entries Overview (All Years, All Placements)

| # | Competition | Year | Placement | Key Logic Extracted |
|---|-------------|------|-----------|---------------------|
| 1 | [name] | [year] | [placement] | [1-sentence summary] |
| ... | ... | ... | ... | ... |

**Total entries mined:** [N] | **Year range covered:** [earliest]–2026 | **Competitions searched:** [N] | **Domains covered:** [list]

---

## Extracted Structural Blueprint & Logic Trees

### Thesis
[Extracted thesis statement]

### Pillar 1: [Label]
- **Claims:** [list]
- **Evidence type:** [empirical/theoretical/etc.]
- **Logical flow:** [deductive/inductive/abductive]

### Pillar 2: [Label]
[...]

### Methodology
[Detected methodology]

### Gap Analysis / Strengths Analysis
| Area | Severity | Description | Action |
|------|----------|-------------|--------|
| [area] | [critical/major/minor] | [desc] | [fix or strength noted] |

---

## Transposed Localized Proposal Engine

**Target Region:** [region]
**Localization Changes:** [summary of transpositions]

### Localized Thesis
[Thesis adapted to local context]

### Localized Pillar 1: [Label]
[Reframed claims with local evidence, data, case studies]

[...]

---

## 1st-Place Victory Upgrade Layer

**Victory Score:** [0.00] / 1.00

| Metric | Score | Weight |
|--------|-------|--------|
| Evidence Strength | [0.00] | 25% |
| Novelty | [0.00] | 20% |
| Localization | [0.00] | 20% |
| Structure | [0.00] | 15% |
| Citation Quality | [0.00] | 10% |
| Counterarguments | [0.00] | 10% |

### Upgraded Proposal

[Full proposal text with all upgrades applied]

### Upgrade Changelog
1. [modification 1]
2. [modification 2]
[...]
```

---

## 9. Execution Workflow Summary

1. Parse `/skills comres [idea] [--region=X] [--domain=Y]`
2. Classify `[idea]` into domain categories 1–7
3. Generate 10+ Boolean queries per domain (year-comprehensive, no lower bound)
4. **Sub-agent dispatch (§1.1.1):** Split 89 endpoints into batches of ~20. Dispatch 4–5 explore sub-agents in parallel. Each sub-agent:
   - Auto-discovers competition start year for each endpoint
   - Enumerates ALL years from start to 2026
   - Crawls each year's results page
   - Extracts ALL qualifying entries (1st–5th, HM, shortlisted, commended, journal-published)
5. Orchestrator collects sub-agent results. Deduplicates by URL+title+year hash. Merges into unified entry pool.
6. Filter: inclusion (all placements, all years) + exclusion (paywall, commercial, <200 words)
7. **Sub-agent dispatch for parsing:** Split qualifying entries into batches of ~10. Dispatch 3–5 general sub-agents. Each parses entries: thesis, pillars, methodology, gap analysis.
8. Orchestrator aggregates parsed entries. Synthesizes cross-entry insights (common patterns, recurrent gaps, year-over-year evolution).
9. Transpose best logic graphs to localized proposal via Localizer
10. Upgrade localized proposal via VictoryUpgrade (empirical injection, counterarguments, novelty)
11. Compute victory_score. If < 0.75, loop back to step 4 with broader search across additional endpoints.
12. Render final Markdown output with all four sections.

**Estimated time:** 10–30 minutes depending on endpoint count and year depth. Sub-agents parallelize I/O-bound work (steps 4, 7). Parsing-heavy steps (8) sequential due to data dependencies.
