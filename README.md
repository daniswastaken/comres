# comres

[![License: GPL v3](https://img.shields.io/badge/License-GPLv3-blue.svg)](https://www.gnu.org/licenses/gpl-3.0)
[![opencode skill](https://img.shields.io/badge/Platform-opencode-black)](https://opencode.ai)

Competitive Research Mining and Proposal Transposition Engine. A multi-agent skill for opencode that mines winning and non-winning entries across 80+ competitions, journals, and academic prizes worldwide, then transposes extracted logic into localized winning proposals.

## What It Does

comres searches 89 target endpoints across 7 categories of competitions and student research journals. It crawls every year from each competition's inception through the current year, extracting all qualifying entries regardless of placement. It then:

1. Mines all entries (1st place winners, finalists, honorable mentions, shortlisted, commended, semi-finalists)
2. Extracts structural logic (thesis, argument pillars, methodology, evidence chains)
3. Analyzes why entries won or fell short
4. Transposes the strongest logic frameworks into a localized proposal tailored to your region and idea
5. Upgrades the proposal with empirical data, counterarguments, and high-tier citations

## Output Sections

Every generated proposal contains:

- **Paper Links** -- direct URLs to every matching paper found across all sources
- **Mined Entries Overview** -- table of all entries with competition, year, placement, and logic summary
- **Extracted Structural Blueprint** -- thesis, argument pillars, methodology, and gap analysis
- **Transposed Localized Proposal** -- your idea rewritten using winning structural patterns, localized to your region
- **1st-Place Victory Upgrade Layer** -- enhanced proposal with victory score breakdown

## Categories Covered

| Category | Scope |
|----------|-------|
| Global Flagships | Geneva Challenge, St. Gallen Symposium, MIT Solve, Goi Peace, and 6 more |
| Academic Prizes | Cambridge Re:think, Harvard IR, Oxford Uehiro, Stanford Kalanithi, and 17 more |
| Economics and Policy | IEA, Royal Economic Society, World Bank Youth Summit, Carnegie Council, and 7 more |
| Philosophy and Ethics | Elie Wiesel Prize, FQXi, Kemper Human Rights, PLATO, and 3 more |
| STEM and Science | Gravity Research, DNA Day ASHG, Genes in Space, NASA Scientist for a Day, and 7 more |
| Humanities and Environment | NYT Student Editorial, Concord Review, Bow Seat Ocean, JFK Profile in Courage, and 16 more |
| Student Research Journals | MIT MURJ, Stanford SURJ, Yale JBM, Harvard Res Publica, and 11 more |

## Installation

### Prerequisites

- [opencode](https://opencode.ai) installed and running
- Internet connection (the skill fetches content from competition websites and journal archives)

### Install the Skill

Copy the `SKILL.md` file into your opencode skills directory:

**Global install (recommended):**

```
cp SKILL.md ~/.config/opencode/skills/comres/SKILL.md
```

**Project install:**

```
cp SKILL.md .opencode/skills/comres/SKILL.md
```

If the directory does not exist, create it first:

```
mkdir -p ~/.config/opencode/skills/comres
```

### Verify Installation

Restart opencode. The skill should appear in your available skills list.

## Usage

### Basic Usage

```
/skills comres [your research idea]
```

Examples:

```
/skills comres AI regulation in developing economies
/skills comres climate adaptation strategies for small island nations
/skills comres the ethics of autonomous weapons systems
```

### With Region Flag

Specify a target region for localization:

```
/skills comres [idea] --region=ID
/skills comres [idea] --region=Kenya
/skills comres [idea] --region="Southeast Asia"
```

Accepts ISO 3166-1 alpha-2 codes or free-text region names.

### With Domain Override

Force classification into a specific category (1-7):

```
/skills comres [idea] --domain=3
/skills comres [idea] --domain=5
```

| Key | Domain |
|-----|--------|
| 1 | Multidisciplinary / Global Flagships |
| 2 | Academic / University Prizes |
| 3 | Economics / Policy / Global Affairs |
| 4 | Philosophy / Law / Ethics |
| 5 | STEM / Science |
| 6 | Humanities / Creative / Environment |
| 7 | Open Access Student Research Journals |

## How It Works

### Agent Pipeline

The skill runs a 5-agent pipeline:

| Agent | Role |
|-------|------|
| Orchestrator | Manages state, routes payloads, dispatches sub-agents |
| SearchRetrieval | Executes searches across 89 endpoints using the edge browser |
| NLPParser | Ingests text, maps argument graphs, extracts logic |
| Localizer | Transposes global concepts to your regional context |
| VictoryUpgrade | Injects empirical data, strengthens arguments, computes victory score |

### Sub-Agent Parallelization

The Orchestrator dispatches sub-agents via the `task` tool to parallelize work:

- 4-5 explore agents crawl endpoint batches simultaneously
- 3-5 general agents parse extracted entries in parallel
- Estimated total runtime: 10-30 minutes depending on endpoint count

### Full Historical Coverage

The skill searches every year from each competition's start year through the current year. No year filtering, no recency sampling. If a competition has run since 2000, all 26 years of entries are crawled.

## Performance

| Metric | Value |
|--------|-------|
| Target endpoints | 89 |
| Categories | 7 |
| Estimated entries per endpoint per year | 5-50 |
| Sub-agent parallelism | Up to 10 concurrent |
| Typical runtime | 10-30 minutes |

## Output Score

Each generated proposal receives a victory score from 0.0 to 1.0 based on:

| Factor | Weight |
|--------|--------|
| Evidence strength | 25% |
| Novelty | 20% |
| Localization | 20% |
| Structure | 15% |
| Citation quality | 10% |
| Counterargument handling | 10% |

Target threshold: 0.75 or higher for 1st-place candidate.

## License

This project is licensed under the GNU General Public License v3.0. See [LICENSE](LICENSE) for details.
