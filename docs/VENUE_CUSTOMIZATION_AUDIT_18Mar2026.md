# Venue Customization Audit — 18 March 2026

## Purpose

Audit of all hardcoded NeurIPS/conference-specific assumptions in AutoResearchClaw, with recommendations for making the pipeline venue-agnostic so users can target arbitrary outlets (journals, workshops, preprints) with custom word count, style, and formatting requirements.

---

## 1. Hardcoded Assumptions Found

### 1.1 Word Count Targets

**File:** `researchclaw/prompts.py` — Lines 222–232

```python
SECTION_WORD_TARGETS: dict[str, tuple[int, int]] = {
    "abstract": (180, 220),
    "introduction": (800, 1000),
    "related work": (600, 800),
    "method": (1000, 1500),
    "experiments": (800, 1200),
    "results": (600, 800),
    "discussion": (400, 600),
    "limitations": (200, 300),
    "conclusion": (200, 300),
    "broader impact": (200, 400),
}
```

- Total target: 5,000–6,500 words (9-page conference paper).
- Not externalizable via config; hardcoded Python dict.
- Section name aliases at lines 236–256 also assume this fixed section list.

### 1.2 Default Conference Target

**File:** `researchclaw/config.py` — Line 373

```python
target_conference: str = "neurips_2025"
```

- `ExportConfig` defaults to `neurips_2025`.
- Only supports: `neurips_2024`, `neurips_2025`, `iclr_2025`, `iclr_2026`, `icml_2025`, `icml_2026`, `generic`.
- No journal, workshop, or preprint options.

### 1.3 Prompt Templates — Paper Draft

**File:** `prompts.default.yaml` — Lines 192–243

- Line 194: System prompt says "write for NeurIPS/ICML/ICLR".
- Line 209: "approximately 5000-6500 words in the main body".
- Line 210: "9-page conference paper".
- Lines 216–227: Per-section word count requirements repeated in natural language.

### 1.4 Prompt Templates — Peer Review

**File:** `prompts.default.yaml` — Lines 281–322

- Line 292: "simulated peer review from at least 2 reviewer perspectives".
- Line 307: "9 pages" explicit reference.
- Line 308: Section depth checks with conference-specific minimums.

### 1.5 Prompt Templates — Paper Revision

**File:** `prompts.default.yaml` — Lines 263–280

- Enforces "never shorten sections" and minimum word counts — assumes a format where length is a virtue.

### 1.6 Prompt Templates — Quality Gate

**File:** `prompts.default.yaml` (quality gate section)

- "A NeurIPS paper body should be 5,000-6,500 words".
- "Zero figures = desk reject" — minimum 2 figures required.
- "Non-negotiable — top-venue paper MUST have statistical tests".
- "Flag any bullet-point lists in Method/Results/Discussion".

### 1.7 Paper Completeness Validation

**File:** `researchclaw/templates/converter.py` — Lines 1357–1499

| Check | Lines | Details |
|-------|-------|---------|
| Mandatory Limitations section | 1400–1422 | NeurIPS/ICLR requirement, hardcoded |
| Abstract length enforcement | 1424–1447 | Warns if >300 or <150 words |
| Total word count floor | 1466–1473 | Fails if <2,000 words (expects 5,000–6,500) |
| Per-section word count validation | 1476–1498 | Uses `SECTION_WORD_TARGETS` from prompts.py |

### 1.8 NeurIPS Checklist Generation

**File:** `researchclaw/pipeline/executor.py` — Lines 546–586

- `_generate_neurips_checklist()` hardcodes NeurIPS 2025 submission requirements.
- Auto-appended to papers at lines 8642–8650.
- Applies to: `neurips_2024`, `neurips_2025`, `icml_2025`, `icml_2026`, `iclr_2025`, `iclr_2026`.
- Not disableable via config.

### 1.9 Domain-to-Venue Mapping

**File:** `researchclaw/pipeline/executor.py` — Lines 44–105

- Hardcoded keyword → domain → venue mapping:
  - ML → "NeurIPS, ICML, ICLR"
  - Physics → "Physical Review Letters"
  - Chemistry → "JACS, Nature Chemistry"
  - Economics → "AER, Econometrica"
  - etc.
- Default fallback if no domain detected: ML → NeurIPS/ICML/ICLR.

### 1.10 LaTeX Templates

**File:** `researchclaw/templates/conference.py` — Lines 1–366

- Default: `neurips_2025` (line 334).
- Supported templates: NeurIPS 2024/2025, ICLR 2025/2026, ICML 2025/2026, Generic.
- Author format varies by conference (NeurIPS vs ICML `\begin{icmlauthorlist}` vs ICLR standard).
- Bibliography style: `plainnat` (NeurIPS), `iclr20XX_conference` (ICLR), `icml20XX` (ICML).

### 1.11 Style Rules (Scattered Across Prompts)

| Rule | Location |
|------|----------|
| Title ≤ 14 words | `writing_guide.py:13` |
| Minimum 2 figures | `prompts.default.yaml` quality gate |
| 25–40 citations expected | `executor.py:~6100` |
| No bullet points in Method/Results/Discussion | `prompts.default.yaml` quality gate |
| Statistical tests mandatory | `prompts.default.yaml` quality gate |

---

## 2. Recommended Changes

### 2.1 New `venue` Config Block

Add to `config.py` a new `VenueConfig` dataclass:

```yaml
venue:
  name: "NeurIPS 2025"
  type: conference        # conference | journal | workshop | preprint
  page_limit: 9
  total_word_range: [5000, 6500]
  abstract_word_range: [150, 250]
  section_word_targets:
    abstract: [180, 220]
    introduction: [800, 1000]
    related work: [600, 800]
    method: [1000, 1500]
    experiments: [800, 1200]
    results: [600, 800]
    discussion: [400, 600]
    limitations: [200, 300]
    conclusion: [200, 300]
  required_sections:
    - abstract
    - introduction
    - related work
    - method
    - experiments
    - results
    - discussion
    - limitations
    - conclusion
  style_notes: |
    Write in formal academic style for a top-tier ML conference.
    No bullet points in Method, Results, or Discussion.
    Minimum 2 figures. Statistical tests required.
  min_figures: 2
  min_citations: 25
  checklist_enabled: true
  checklist_template: null   # path to custom checklist, or null for built-in
  latex_template: neurips_2025
  bib_style: plainnat
  citation_style: author-year
```

### 2.2 Make `SECTION_WORD_TARGETS` Dynamic

**File to change:** `researchclaw/prompts.py`

- Replace the hardcoded dict with a function that reads from `VenueConfig`.
- Keep the current dict as a fallback default.

### 2.3 Parameterize Prompt Templates

**File to change:** `prompts.default.yaml`

Replace hardcoded references with template variables:

| Current | Replacement |
|---------|-------------|
| "NeurIPS/ICML/ICLR" | `{venue_name}` |
| "9-page conference paper" | `{page_limit}-page {venue_type} paper` |
| "5000-6500 words" | `{total_word_min}-{total_word_max} words` |
| Per-section word counts | Generated from `{section_word_targets}` |
| "Zero figures = desk reject" | Conditional on `{min_figures}` |

### 2.4 Make Validation Configurable

**File to change:** `researchclaw/templates/converter.py`

- Read thresholds from `VenueConfig` instead of hardcoded values.
- Make Limitations section conditional (required for NeurIPS/ICLR, optional elsewhere).
- Make abstract length check use `venue.abstract_word_range`.

### 2.5 Conditional Checklist

**File to change:** `researchclaw/pipeline/executor.py`

- Only append checklist when `venue.checklist_enabled` is true.
- Support custom checklist templates via `venue.checklist_template`.

### 2.6 Ship Venue Presets

Add a `venues/` directory with ready-made configs:

```
venues/
├── neurips2025.yaml
├── icml2026.yaml
├── iclr2026.yaml
├── acl2025.yaml
├── nature.yaml
├── arxiv_preprint.yaml
├── tmlr.yaml
└── custom_template.yaml   # blank template for user customization
```

Users would reference a preset via:

```yaml
venue_preset: neurips2025   # loads venues/neurips2025.yaml as defaults
venue:                       # overrides on top of the preset
  style_notes: |
    Additional custom instructions...
```

---

## 3. Files Requiring Modification

| File | Change Scope |
|------|-------------|
| `researchclaw/config.py` | Add `VenueConfig` dataclass, wire into `ResearchClawConfig` |
| `researchclaw/prompts.py` | Make `SECTION_WORD_TARGETS` read from config; parameterize prompt builder |
| `prompts.default.yaml` | Replace ~15 hardcoded references with template variables |
| `researchclaw/templates/converter.py` | Read validation thresholds from `VenueConfig` |
| `researchclaw/templates/conference.py` | Support custom LaTeX template paths |
| `researchclaw/pipeline/executor.py` | Conditional checklist; parameterize domain-venue mapping |

---

## 4. Migration Path

1. Add `VenueConfig` with NeurIPS 2025 as the default values — **zero behavior change**.
2. Wire `VenueConfig` into prompt builder and validation — replace hardcoded values with config reads.
3. Parameterize `prompts.default.yaml` templates.
4. Add venue preset files.
5. Update `config.researchclaw.example.yaml` with the new `venue` block and documentation.
6. Test with at least one non-ML venue (e.g., Nature, ACL) to validate generality.
