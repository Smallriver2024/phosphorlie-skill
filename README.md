# PhosphoRLIE — PTM Relation & Literature Information Extraction

[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Status](https://img.shields.io/badge/status-active-success.svg)]()

> **A Claude Code skill for structured extraction and quality-controlled review
> of post-translational modification (PTM) regulatory events from biomedical literature.**

PhosphoRLIE uses a **two-phase LLM workflow** — Extractor + Reviewer — to mine
PubMed titles and abstracts for kinase–substrate–disease–drug relationships.
Each extraction is automatically reviewed for enzyme name standardization,
modification site validation, and multi-entity formatting before output.

Supports: **phosphorylation**, **acetylation**, **ubiquitination**, **SUMOylation**,
**methylation**, **glycosylation**, and other PTMs.
<img width="1448" height="1086" alt="Fig1" src="https://github.com/user-attachments/assets/1b14428a-1a1d-4279-b775-21b43961c908" />

---

## Table of Contents

- [Quick Install (Recommended)](#quick-install-recommended)
- [Manual Install](#manual-install)
- [Two-Phase Architecture](#two-phase-architecture)
- [Usage](#usage)
- [Input Format](#input-format)
- [Output Schema](#output-schema)
- [What the Reviewer Checks](#what-the-reviewer-checks)
- [Examples](#examples)
- [Extending to Other PTMs](#extending-to-other-ptms)
- [Project Structure](#project-structure)
- [Controlling Tool Permissions](#controlling-tool-permissions)
- [Citation](#citation)
- [License](#license)

---

## Quick Install (Recommended)

### Method 1: One-Click Install via Natural Language

Open Claude Code in any project, and simply say:

> **"Please install the PhosphoRLIE skill from https://github.com/Smallriver2024/phosphorlie-skill"**

Or in Chinese:

> **"帮我从 https://github.com/Smallriver2024/phosphorlie-skill 安装 phosphorlie 技能"**

Claude Code will automatically clone the repository and set up the skill under
`~/.claude/skills/phosphorlie/` (global) or `.claude/skills/phosphorlie/` (project-local).
Once installed, it's immediately available — no restart needed.

### Method 2: Curl One-Liner

```bash
# Global install (available in all projects)
mkdir -p ~/.claude/skills/phosphorlie && \
curl -sSL https://raw.githubusercontent.com/Smallriver2024/phosphorlie-skill/main/SKILL.md \
  -o ~/.claude/skills/phosphorlie/SKILL.md
```

```bash
# Project-local install (only in current project)
mkdir -p .claude/skills/phosphorlie && \
curl -sSL https://raw.githubusercontent.com/Smallriver2024/phosphorlie-skill/main/SKILL.md \
  -o .claude/skills/phosphorlie/SKILL.md
```

Optional: also download prompts and examples for reference:

```bash
curl -sSL https://raw.githubusercontent.com/Smallriver2024/phosphorlie-skill/main/prompts/schema.json \
  -o ~/.claude/skills/phosphorlie/prompts/schema.json
```

### Verification

After install, start Claude Code and type `/phosphorlie`. If the skill is
recognized, it will respond with usage help. You can also ask:

> **"What skills are available?"**

---

## Manual Install

### Option A: Git Clone + Copy

```bash
git clone https://github.com/Smallriver2024/phosphorlie-skill.git
cd phosphorlie-skill

# Global install
mkdir -p ~/.claude/skills/phosphorlie
cp SKILL.md ~/.claude/skills/phosphorlie/
cp -r prompts/ ~/.claude/skills/phosphorlie/
cp -r examples/ ~/.claude/skills/phosphorlie/

# Or project-local install
mkdir -p /your-project/.claude/skills/phosphorlie
cp SKILL.md /your-project/.claude/skills/phosphorlie/
```

### Option B: Use Standalone Prompts (No Claude Code Required)

The `prompts/` directory contains standalone prompt files that work with any LLM:

| File | Purpose |
|------|---------|
| `prompts/extractor_system.md` | Phase 1 extraction system prompt |
| `prompts/reviewer_system.md` | Phase 2 review/correction system prompt |
| `prompts/schema.json` | JSON output schema (machine-readable) |
| `prompts/user_prompt.md` | User prompt template |

Send the extractor prompt first, then pass its output to the reviewer prompt.
Compatible with OpenAI, Anthropic, Qwen, DeepSeek, and other LLM APIs.

### Expected Install Structure

After install, the skill directory should look like:

```
~/.claude/skills/phosphorlie/
├── SKILL.md              # Main skill definition (REQUIRED)
├── prompts/              # Optional but recommended
│   ├── extractor_system.md
│   ├── reviewer_system.md
│   ├── schema.json
│   └── user_prompt.md
└── examples/             # Optional
    ├── input.jsonl
    ├── output.jsonl
    └── no_event.jsonl
```

The only **required** file is `SKILL.md`. The `prompts/` and `examples/`
directories are optional but useful for reference and non-Claude-Code usage.

---

## Two-Phase Architecture

```
User Input (PMID + Title + Abstract)
       │
       ▼
┌──────────────────────┐
│ Phase 1: EXTRACTOR    │
│ - Identify ONE focal  │
│   PTM regulatory event│
│ - Extract all 28      │
│   schema fields       │
│ - Assign confidence   │
│   score (0–10)        │
└──────────┬───────────┘
           │  Raw extraction JSON
           ▼
┌──────────────────────┐
│ Phase 2: REVIEWER     │
│ - Standardize enzyme  │
│   names (literature   │
│   name → gene symbol) │
│ - Validate sites      │
│   (must be AAnnn)     │
│ - Fix multi-entity    │
│   separators          │
│ - Cross-field checks  │
│ - Update confidence   │
│   & review flags      │
└──────────┬───────────┘
           │  Reviewed JSON
           ▼
     Final Output
```

Both phases run automatically in a single invocation. The user sees only the
final reviewed result.

---

## Usage

### Full Pipeline (Extract + Review)

```
# English
/phosphorlie Extract phosphorylation events from:
PMID: 12345678
Title: AKT phosphorylates MDM2 at Ser186 leading to p53 degradation
Abstract: The AKT kinase directly phosphorylates...

# 中文
/phosphorlie 从这篇文献中抽提磷酸化事件:
PMID: 12345678
Title: AKT phosphorylates MDM2 at Ser186...
Abstract: ...
```

### Review Only (Already-Extracted JSON)

```
/phosphorlie review this extraction:
{"pmid": "12345678", "ptm_target": "Akt", "modification_sites": ["Ser186"], ...}
```

### Custom PTM Type

```
/phosphorlie --ptm acetylation PMID: 45678901 Title: p300 acetylates p53 at K382...
/phosphorlie --ptm ubiquitination PMID: ...
/phosphorlie --ptm SUMOylation PMID: ...
```

### Batch Processing (JSONL File)

```
/phosphorlie batch process input.jsonl → output_reviewed.jsonl
```

For large-scale batch (>100 papers), consider using the companion Python script
with your own LLM API key (see [Controlling Tool Permissions](#controlling-tool-permissions)).

---

## Input Format

**Interactive mode**: Plain text with PMID, title, and abstract.

**Batch mode (JSONL)**: One JSON object per line:

```json
{"pmid": "12345678", "title": "AKT phosphorylates MDM2 at Ser186...", "abstract": "The AKT kinase..."}
{"pmid": "23456789", "title": "EGFR autophosphorylation at Y1068...", "abstract": "Epidermal growth factor..."}
```

Required fields: `pmid`, `title`, `abstract`.
Optional: `pub_date`, `doi`, `journal`, `authors`.

---

## Output Schema

Each reviewed record contains 30 fields (28 extraction + 2 reviewer-added):

```json
{
  "pmid": "string",
  "record": "string",
  "ptm_type": "phosphorylation | acetylation | ubiquitination | SUMOylation | methylation | glycosylation | other_ptm | unknown",
  "relation_exists": "yes | no | unclear",

  "evidence_text": "string",
  "evidence_scope": "single_sentence | sentence_pair | title_plus_abstract | unknown",

  "ptm_target": "string (multi-entity: semicolon-separated)",
  "ptm_target_role": "enzyme | substrate | unknown",
  "modification_sites": ["string (format: [AminoAcid][Position], e.g. S186)"],
  "upstream_enzyme": "string (multi-entity: semicolon-separated)",
  "ptm_origin": "auto_modification | upstream_enzyme_explicit | unknown",

  "primary_agent": "string (multi-entity: semicolon-separated)",
  "agent_validity": "valid_intervention | assay_reagent | detection_antibody | supporting_reagent | unknown",
  "agent_type": "approved_drug | investigational_drug | tool_compound | antibody | biologic | peptide | extract_mixture | other_reagent | unknown",

  "disease": "string (multi-entity: semicolon-separated)",
  "disease_category": "cancer | non_cancer | unclear",
  "cancer_type": "string",
  "cancer_subtype": "string",
  "phenotype_context": "string",

  "cell_lines": ["string"],
  "species": ["string"],
  "species_is_human": "yes | no | unknown",

  "trigger_phrase": "string",
  "regulation_mode": "enhancing | impairing | associated",
  "relationship": "string",

  "confidence": 0-10,
  "needs_manual_review": "yes | no",
  "review_reasons": ["string"],
  "reviewer_notes": "string"
}
```

### Confidence Rubric

| Score | Meaning |
|-------|---------|
| 9–10 | Explicit eligible relation, clear target, clear context, clear direction |
| 7–8 | Strong evidence, minor ambiguity |
| 5–6 | Probable relation, notable ambiguity |
| 3–4 | Weak or partial evidence |
| 0–2 | Unsupported or not eligible |

---

## What the Reviewer Checks

### 1. Enzyme Name Standardization

Common literature names are mapped to standard gene symbols using Claude's
built-in knowledge of HGNC-approved gene nomenclature:

| Literature Name | Standard Symbol | PTM Type |
|----------------|-----------------|----------|
| AKT | AKT1 | Phosphorylation |
| ERK | MAPK1; MAPK3 | Phosphorylation |
| JNK | MAPK8; MAPK9; MAPK10 | Phosphorylation |
| p38 | MAPK14 | Phosphorylation |
| MEK1 | MAP2K1 | Phosphorylation |
| GSK3β | GSK3B | Phosphorylation |
| IKKβ | IKBKB | Phosphorylation |
| DNA-PK | PRKDC | Phosphorylation |
| p300 | EP300 | Acetylation |
| CBP | CREBBP | Acetylation |
| PCAF | KAT2B | Acetylation |

If a name cannot be confidently mapped, the reviewer keeps the original,
adds `enzyme_name_uncertain` to `review_reasons`, and documents the uncertainty.

### 2. Modification Site Validation

Every site must match: **one-letter amino acid code + position number**
(`^[ACDEFGHIKLMNPQRSTVWY]\d+$`)

| Input | Action | Result |
|-------|--------|--------|
| `S186` | Keep | `S186` |
| `Y490` | Keep | `Y490` |
| `Ser186` | Convert (Ser→S) | `S186` |
| `Tyr-705` | Convert (Tyr→Y, remove hyphen) | `Y705` |
| `pY705` | Remove phospho-prefix | `Y705` |
| `Y` | **Remove** (no position) | — |
| `308` | **Remove** (no amino acid) | — |
| `C-terminal` | **Remove** (descriptive) | — |
| `424-434` | **Remove** (range) | — |

### 3. Multi-Entity Formatting

Multiple entities in the same field use **semicolon + space** (`; `):
- `upstream_enzyme`: `"AKT1; MAPK1; MAPK3"`
- `ptm_target`: `"MDM2; p53"`
- `primary_agent`: `"gefitinib; erlotinib"`
- `disease`: `"breast cancer; ovarian cancer"`

### 4. Cross-Field Consistency

- `upstream_enzyme` non-empty → `ptm_origin` should be `upstream_enzyme_explicit`
- `relation_exists = "no"` → all event fields should be empty
- `disease_category = "cancer"` → `disease` should contain a cancer name
- `species_is_human = "yes"` when known human cell lines are present
- `cancer_type` non-empty → `disease_category` should be `cancer`

---

## Examples

### Input

```
PMID: 12345678
Title: AKT and ERK phosphorylate MDM2 at Ser186 and Ser188 promoting p53 degradation in breast cancer cells
Abstract: Both AKT and ERK kinases directly phosphorylate the MDM2 protein at Ser186 and Ser188, which enhances MDM2-mediated ubiquitination and degradation of p53. Treatment with the AKT inhibitor MK-2206 or the MEK inhibitor PD98059 impaired this phosphorylation in MCF-7 and T47D breast cancer cells, leading to apoptosis.
```

### Output (After Phase 1 + Phase 2 Review)

```json
{
  "pmid": "12345678",
  "ptm_type": "phosphorylation",
  "relation_exists": "yes",
  "evidence_text": "Both AKT and ERK kinases directly phosphorylate the MDM2 protein at Ser186 and Ser188, which enhances MDM2-mediated ubiquitination and degradation of p53.",
  "evidence_scope": "single_sentence",
  "ptm_target": "MDM2",
  "ptm_target_role": "substrate",
  "modification_sites": ["S186", "S188"],
  "upstream_enzyme": "AKT1; MAPK1; MAPK3",
  "ptm_origin": "upstream_enzyme_explicit",
  "primary_agent": "MK-2206; PD98059",
  "agent_validity": "valid_intervention",
  "agent_type": "investigational_drug; tool_compound",
  "disease": "breast cancer",
  "disease_category": "cancer",
  "cancer_type": "breast cancer",
  "phenotype_context": "apoptosis",
  "cell_lines": ["MCF-7", "T47D"],
  "species": ["human"],
  "species_is_human": "yes",
  "trigger_phrase": "AKT and ERK kinases directly phosphorylate the MDM2 protein at Ser186 and Ser188",
  "regulation_mode": "enhancing",
  "relationship": "AKT1; MAPK1; MAPK3 phosphorylate MDM2 at S186 and S188, enhancing p53 degradation; MK-2206 and PD98059 impair this phosphorylation, restoring p53 and inducing apoptosis",
  "confidence": 8,
  "needs_manual_review": "no",
  "review_reasons": [],
  "reviewer_notes": "Standardized AKT→AKT1; Standardized ERK→MAPK1;MAPK3; Converted sites Ser186→S186, Ser188→S188; Changed separator to semicolons; Added agent_type for PD98059"
}
```

**Reviewer corrections applied:**
1. `AKT` → `AKT1` (standard gene symbol)
2. `ERK` → `MAPK1; MAPK3` (ERK1=MAPK3, ERK2=MAPK1)
3. `Ser186` → `S186`, `Ser188` → `S188` (three-letter → one-letter code)
4. Multi-entity separators standardized to `; `
5. `PD98059` agent_type added (`tool_compound`)
6. Confidence reduced 9→8 (substantive corrections made)

---

## Extending to Other PTMs

PhosphoRLIE supports all major PTM types through its `ptm_type` field:

| PTM Type | Enzyme Class | Modified Residues | Site Pattern |
|----------|-------------|-------------------|-------------|
| `phosphorylation` | Kinase | S, T, Y | `[STY]\d+` |
| `acetylation` | Acetyltransferase (HAT) | K | `K\d+` |
| `ubiquitination` | E3 ubiquitin ligase | K | `K\d+` |
| `SUMOylation` | SUMO E3 ligase | K | `K\d+` |
| `methylation` | Methyltransferase | K, R | `[KR]\d+` |
| `glycosylation` | Glycosyltransferase | N, S, T | `[NST]\d+` |

Specify the PTM type when invoking: `/phosphorlie --ptm ubiquitination PMID: ...`

The enzyme/substrate/target logic is PTM-agnostic. Only the enzyme class name
changes (kinase → acetyltransferase → E3 ligase → methyltransferase, etc.).

---

## Project Structure

```
phosphorlie-skill/
├── SKILL.md                       # Claude Code skill definition (REQUIRED)
├── README.md                      # This file
├── LICENSE                        # MIT License
├── CITATION.cff                   # Citation metadata (GitHub-ready)
├── prompts/                       # Standalone prompts for non-Claude-Code usage
│   ├── extractor_system.md        # Phase 1: Extraction system prompt
│   ├── reviewer_system.md         # Phase 2: Review/correction system prompt
│   ├── schema.json                # JSON output schema (30 fields)
│   └── user_prompt.md             # User prompt template
└── examples/                      # Example inputs and outputs
    ├── input.jsonl                # Example input (5 records)
    ├── output.jsonl               # Example reviewed output (5 records)
    └── no_event.jsonl             # Example: technical paper (no eligible event)
```

---

## Controlling Tool Permissions

By default, `SKILL.md` grants `allowed-tools: [Read, WebSearch]` — sufficient for
interactive extraction and enzyme name verification. This conservative setting
ensures the skill won't write files without explicit approval.

### For Batch Processing

If you need batch processing (reading/writing JSONL files), edit your installed
`SKILL.md` and extend the permissions:

```yaml
allowed-tools: [Read, Write, Bash, WebSearch]
```

### For Large-Scale Batch with Your Own API Key

For processing thousands of PubMed records, using Claude Code interactively is
not efficient. Instead, use the companion Python script
`qwen_extract_phospho_event_sharded.py` with your own LLM API key:

```bash
# 1. Set your API key
export DASHSCOPE_API_KEY="your-api-key-here"

# 2. Run batch extraction
python qwen_extract_phospho_event_sharded.py \
  --input-dir ./pubmed_jsonl/ \
  --input-pattern "articles_*.jsonl" \
  --output-dir ./extracted_events/ \
  --output-prefix "phosphorlie_output" \
  --model "qwen3-max" \
  --workers 12 \
  --resume
```

The script uses OpenAI-compatible API format — compatible with Alibaba DashScope
(Qwen models), OpenAI, DeepSeek, and other providers. The prompts in `prompts/`
can be adapted for any LLM API.

> **Note**: The companion Python script is not included in the skill package.
> It lives in the main project repository. Contact the author or check the
> GitHub repo for availability.

---

## Citation

If you use PhosphoRLIE in your research, please cite:

### BibTeX

```bibtex
@software{phosphorlie2025,
  title        = {PhosphoRLIE: A Post-Translational Modification Relation
                  \& Literature Information Extraction Framework},
  year         = {2025},
  publisher    = {GitHub},
  url          = {https://github.com/Smallriver2024/phosphorlie-skill},
  note         = {Two-phase LLM skill (Extractor + Reviewer) for structured
                  PTM event extraction from biomedical literature}
}
```

### APA

> PhosphoRLIE: A Post-Translational Modification Relation & Literature Information Extraction Framework. (2025). GitHub. https://github.com/Smallriver2024/phosphorlie-skill

---

## License

MIT License — see [LICENSE](LICENSE) for details.

---

*For questions, issues, or contributions, please open a GitHub issue at
https://github.com/Smallriver2024/phosphorlie-skill*
