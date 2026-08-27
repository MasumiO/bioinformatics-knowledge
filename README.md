# Bioinformatics Knowledge

Public, curated technical knowledge base for reusable bioinformatics methods, tools, evidence, and practical analysis guidance.

The repository is intended to support both personal and professional use. Content should therefore be written so that it can be safely referenced in work without depending on private Family Knowledge, employer-confidential context, or client-specific material.

## Scope

Initial focus:

- single-cell omics
- spatial transcriptomics
- differential expression
- trajectory and RNA velocity
- cell–cell communication
- gene regulatory networks
- enrichment / pathway analysis
- reproducible analysis workflows
- software behavior, versions, benchmarks, and pitfalls

## Knowledge flow

```text
external sources
      ↓
private research-inbox
      ↓ verify / sanitize / rewrite
bioinformatics-knowledge
```

Raw or unreviewed external research belongs in the private staging repository `MasumiO/research-inbox`, not here.

## Suggested structure

```text
single-cell/
spatial/
methods/
tools/
benchmarks/
references/
templates/
docs/
```

Directories are created as content is added.

## Quality principles

- Prefer primary literature and official software documentation.
- Preserve DOI, canonical URLs, software versions, and dates where relevant.
- Separate evidence from interpretation and recommendation.
- State assumptions, applicability limits, and uncertainty explicitly.
- Avoid overstating benchmark results or treating one dataset as universally representative.
- Keep public notes free of private, employer-confidential, and client-confidential material.

See [`CONTRIBUTING.md`](CONTRIBUTING.md), [`docs/PUBLISHING_POLICY.md`](docs/PUBLISHING_POLICY.md), and [`templates/knowledge-note.md`](templates/knowledge-note.md).

## Relationship to private repositories

This public repository contains generalizable technical knowledge only.

Private decisions, project history, personal context, and confidential work remain outside this repository. `MasumiO/family-knowledge` is a separate private trust domain and is not a source that public research agents should automatically access.

## License

No license has been selected yet. Until a license is explicitly added, normal copyright applies to repository-authored content. Third-party sources retain their own licenses and copyrights.
