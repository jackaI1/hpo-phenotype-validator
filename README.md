# HPO phenotype validator

I built a checker for symptom lists because I found it hard to spot old codes, repeated information and conflicting entries by eye. The Human Phenotype Ontology (HPO) gives observed features, such as low muscle tone, a shared name and code. I wanted to check a list before passing it to another tool, while keeping uncertain wording visible for review.

I can update retired codes when HPO gives a clear replacement, find repeated broad and specific features, and flag a feature marked both present and absent. For phrases, I show suggested terms and a spelling similarity score. Uncertain phrases stay unmapped. Once every row is resolved, I export a Phenopacket, a standard JSON file describing a subject's features.

[Read the saved report](reports/README.md), [inspect a blocked list](reports/review/validation.json), or [open the cleaned Phenopacket](reports/accepted/phenopacket.json) without installing anything. All examples are fictional. This is a data-checking project, not a diagnostic tool.

To try the accepted example, clone this repository and run these commands inside it with [uv](https://docs.astral.sh/uv/getting-started/installation/) installed:

```sh
uv sync --extra dev --locked
uv run hpo-validate validate examples/accepted.json --out output-accepted
```

Open `output-accepted/validation.html` to see the decisions, or use `output-accepted/phenopacket.json` in another tool. Output folders must be new or empty so an older export cannot be confused with a failed run. The validator runs offline with the bundled public ontology. Only the initial environment setup needs a download.

## How I review a list

```sh
uv run hpo-validate validate examples/needs-review.json --out output-review
```

This example intentionally exits with code `2`: it saves the reports and blocks export. It contains an obsolete code, an alternative code, duplicate information, an excluded parent of a present feature, an unknown code and uncertain text. The report shows every original row, replacement path, suggestion and unresolved issue.

I edit the original JSON after reviewing it, then rerun into a fresh folder. In the accepted example I removed the contradictory exclusion and unresolved rows. I did not quietly turn uncertain wording into confirmed findings. Exit `0` means accepted; exit `1` means invalid input or a file error. [Input format and review steps](docs/usage.md) include how to mark an excluded feature and use a different public ontology release.

## What gets changed

A retired code follows only a single explicit replacement chain. Multiple replacements, missing targets and HPO's `consider` suggestions require review. Alternative IDs become primary IDs. Codes for inheritance or other categories outside the phenotype branch also require review.

A broad present feature is redundant when a more specific child is already present, so I keep the child. Exclusions work in the other direction: excluding a broad feature also excludes its children, so I keep the broadest exclusion. A present child contradicts an excluded parent; a present parent and excluded child can coexist. I report all pairs before removing redundancy and never choose between contradictory assertions automatically.

Only a unique preferred label or exact synonym maps automatically, after normalizing case and whitespace. Shared synonyms remain ambiguous. Spelling-based suggestions always need review, even with a high score. The score measures character similarity, not the probability that a medical interpretation is correct. Negation and uncertainty stay in review rather than becoming assertions. This deliberately gives up automatic coverage to avoid guessing.

## What I measured

I hand-labelled an English phrase sample from public HPO names and definitions before running the matcher. It includes exact labels, synonyms, typos, paraphrases and intentionally unmappable phrases. I wrote both the labels and the matcher; this is a small authored check, not an independent clinical benchmark. No real patient notes were used.

The evaluation split produced these results:

| Measure | Result |
| --- | ---: |
| Phrases checked | 40 |
| Automatically accepted | 18 |
| Correct accepted IDs | 18 of 18 |
| Precision among accepted IDs | 100% |
| Automatic coverage | 45% |
| Left unmapped for review | 22 |
| Correct ID or correct abstention | 32 of 40, or 80% |
| Gold IDs among the first three suggestions | 26 of 26 |

These figures come from `uv run python scripts/build_reports.py --check`, which recomputes [summary.json](reports/summary.json) and compares every saved report with its source. Correct abstentions only count when the hand label intentionally has no ID. Abstaining on a mappable phrase counts as a missed mapping. [Every prediction](reports/mapping.json) and the [evaluation protocol](docs/evaluation.md) make the denominators and limitations explicit. The results do not establish performance on clinical notes, unseen languages or real patients.

## Public sources

This project uses the Human Phenotype Ontology, release `2026-09-01`, maintained by the Human Phenotype Ontology Consortium. I bundled the unmodified [official HPO release](https://github.com/obophenotype/human-phenotype-ontology/releases/tag/v2026-09-01), with its download URL, version and checksum in [data/manifest.json](data/manifest.json). No substitute ontology was needed. [Source and license notes](data/README.md) preserve the upstream license pointer and explain the available license reference.

I used the [HPO publication](https://doi.org/10.1093/nar/gkaa1043) for context, the [OBO format guide](https://owlcollab.github.io/oboformat/doc/GO.format.obo-1_4.html) for identifier and replacement tags, and the [Phenopacket specification](https://phenopacket-schema.readthedocs.io/en/latest/phenopacket.html), [metadata rules](https://phenopacket-schema.readthedocs.io/en/latest/metadata.html) and [official schema bindings](https://github.com/ga4gh/phenopacket-schema) for export. The phrase sample and subject lists were authored for this project. No disease, gene or private record source was used.

## Inside the project

`hpo_validator/ontology.py` reads identifiers, exact synonyms and asserted parent links from the ontology's text format. It checks the graph and calculates ancestry. `mapping.py` indexes labels and exact synonyms, shortlists spelling candidates using shared character groups, then ranks them with Python's `SequenceMatcher`. `validation.py` keeps the row audit and performs list checks. `export.py` writes the Phenopacket with explicit exclusions and ontology metadata.

The command-line tool has no runtime package dependencies. The development environment includes the official Phenopacket schema bindings so tests can parse the exported JSON and check its fields. That checks structure and this project's required metadata; it is not a clinical validation service. The schema package currently emits dependency deprecation warnings during tests. [Design decisions](docs/design.md) explain the conservative mapping and exclusion rules.

The bundled release contains 20,482 terms, including 588 obsolete terms and 19,177 active phenotype terms. These counts are printed by the report command above. The parser uses asserted ancestry, not every logical axiom in HPO. It does not infer diagnoses, interpret complete notes or reconcile observations from different times.

I run the following checks locally and in [CI](.github/workflows/ci.yml):

```sh
uv run pytest -q
uv run ruff check .
uv run ruff format --check .
uv run python scripts/build_reports.py --check
uv build --quiet
uv run python scripts/check_wheel.py
```

The test suite currently passes 64 tests, covering obsolete and alternative IDs, graph errors, ambiguous text, exclusion direction, contradictions, strict inputs, escaped HTML, export blocking, CLI output and parsing exports with the official schema. CI runs the suite and artifact checks with Python 3.10 and 3.12 and checks an installed package outside the checkout. The repository started with no existing test suite.
