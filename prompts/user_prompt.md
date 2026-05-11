# PhosphoRLIE User Prompt Template

> Parameterized with `{pmid}`, `{title}`, `{abstract}`, `{ptm_type}`, and
> `{schema_json}`. Use this template when constructing prompts for LLM APIs.

```
Extract one {ptm_type}-related regulatory evidence event from the following
PMID record and return exactly one JSON object.

PMID: {pmid}

Title:
{title}

Abstract:
{abstract}

Instructions:
- Use English values.
- Use only the provided title and abstract.
- Keep the PTM target at the granularity reported in the text.
- Do not guess missing isoforms or hidden entities.
- Return only ONE best-supported eligible PTM event for this PMID record.
- If no eligible event exists, return relation_exists = "no".
- If the article contains only technical assay/detection content without an
  eligible PTM-regulatory evidence event, return relation_exists = "no" or
  mark manual review as appropriate.
- trigger_phrase should be copied from the original text when possible.
- regulation_mode must be one of: enhancing, impairing, associated.
- Fill upstream_enzyme and ptm_origin only when explicitly supported by the
  same focal PTM event; otherwise leave upstream_enzyme empty and set
  ptm_origin = "unknown".
- confidence must be an integer from 0 to 10.
- When an enzyme-to-substrate modification relation is explicitly stated,
  use the substrate as ptm_target and record the enzyme in upstream_enzyme.
- ptm_type must be one of: phosphorylation, acetylation, ubiquitination,
  SUMOylation, methylation, glycosylation, other_ptm, unknown.

Schema:
{schema_json}
```

## PTM Type Examples

### Phosphorylation (default)
```
ptm_type = "phosphorylation"
entity: kinase → phosphorylates → substrate at Ser/Thr/Tyr
```

### Acetylation
```
ptm_type = "acetylation"
entity: acetyltransferase → acetylates → substrate at Lys (K)
```

### Ubiquitination
```
ptm_type = "ubiquitination"
entity: E3 ubiquitin ligase → ubiquitinates → substrate at Lys (K)
```

### SUMOylation
```
ptm_type = "SUMOylation"
entity: SUMO E3 ligase → SUMOylates → substrate at Lys (K)
```

### Methylation
```
ptm_type = "methylation"
entity: methyltransferase → methylates → substrate at Lys/Arg
```
