# Contributing

This repository is a curated public knowledge base, so correctness, provenance, and disclosure safety take priority over speed.

## Before adding a note

1. Search for an existing note on the same topic.
2. Prefer improving a canonical note over creating a near-duplicate.
3. Verify that the material is appropriate for public disclosure.
4. Prefer primary literature and official documentation for material claims.

## Content expectations

A useful note should make clear:

- what question or decision it helps with
- what the evidence supports
- what is interpretation or practical recommendation
- assumptions and applicability limits
- important caveats or conflicting evidence
- relevant software / package versions
- provenance such as DOI, PMID, canonical URLs, releases, or repositories

## Public-disclosure requirements

Do not contribute:

- confidential employer/client information
- unpublished internal results
- personal data
- credentials or secrets
- large copied passages from copyrighted sources
- unsupported claims presented as established fact

## Promotion from research-inbox

Notes promoted from the private staging repository must be independently reviewed and rewritten. Staging content is not trusted merely because it was previously generated or summarized.

Follow `docs/PUBLISHING_POLICY.md`.

## File naming

Prefer lowercase kebab-case Markdown names for topic notes, for example:

```text
single-cell/pseudobulk-differential-expression.md
single-cell/rna-velocity-model-assumptions.md
tools/seurat-v5-large-dataset-considerations.md
```

## Changes to existing notes

When updating a note because of a new paper, benchmark, or software release, preserve older context where it remains relevant and make version/time sensitivity explicit.
