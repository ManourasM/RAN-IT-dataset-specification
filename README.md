# RAN-IT Intent Dataset with Cell Scope

**Release:** 1.1.0  
**Profile:** `ran-intent-profile-v1.2.0`  
**Language:** English  
**Prepared:** 12 October 2026 (release 1.0.0: 5 October 2026)  
**License:** [Creative Commons Attribution 4.0 International (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/)

This dataset pairs natural-language Radio Access Network (RAN) performance requests with structured JSON labels. It supports research on intent compilation, structured language-model output, numeric and directional interpretation, and recognizing requests that require clarification or fall outside a bounded profile.

The requests and labels were constructed from a restricted, standards-grounded semantic vocabulary. Numeric values are requested bounds, rather than measured network outcomes. The release combines the original NR RAN SubNetwork corpus with an additive cell-scope extension. Release 1.1.0 adds a separate, test-only failure extension of 117 requests from 39 new failure parents (see [Failure test extension](#failure-test-extension)); the training, validation, and test files are byte-identical to release 1.0.0. It contains no network traces or reconfiguration-action labels.

## Files and dataset size

Each CSV has a single header row and eight columns. Files use UTF-8 without a byte-order mark, comma delimiters, quoted fields, and CRLF record endings. Each JSON label occupies one CSV cell. The ZIP contains this README and the four CSVs at its root.

| File | Examples | Accepted | Typed failures | Accepted: SubNetwork scope | Accepted: cell scope |
|---|---:|---:|---:|---:|---:|
| `train.csv` | 1,673 | 1,512 | 161 | 840 | 672 |
| `validation.csv` | 359 | 324 | 35 | 180 | 144 |
| `test.csv` | 359 | 324 | 35 | 180 | 144 |
| `test_failure_extension.csv` | 117 | 0 | 117 | 0 | 0 |
| **Total** | **2,508** | **2,160** | **348** | **1,200** | **960** |

There are **612 semantic parents**: 300 accepted SubNetwork intents, 240 accepted cell-scoped intents, 33 failure cases in the core splits, and 39 failure cases in the failure extension. A parent represents one fixed structured label and its associated wording variants. Every accepted parent has one canonical sentence and three paraphrases; every core failure parent has one canonical utterance and six paraphrases; every extension failure parent has one canonical utterance and two paraphrases.

The original split assignments are preserved. Parents are distributed 401/86/86 across training/validation/test. There are **568 leakage groups**, distributed 398/85/85. The failure extension adds 39 test-only parents, each in its own leakage group, for **607 leakage groups** in total. Some groups contain several similar parents. These counts refer to this release; wording variants are separate examples, not independent semantic intents.

| Status | Core splits | Failure extension | Total |
|---|---:|---:|---:|
| `accepted` | 2,160 | 0 | 2,160 |
| `clarification-required` | 112 | 51 | 163 |
| `unsupported` | 84 | 39 | 123 |
| `invalid` | 35 | 27 | 62 |

## Column dictionary

| Column | Meaning |
|---|---|
| `id` | Unique short identifier for an example within this release. |
| `parent_id` | Links a canonical request and all its wording variants. |
| `leakage_group_id` | Links examples connected by common parent membership or existing similarity-group membership. Keep a whole group together if creating new partitions. |
| `natural_language` | Exact input sentence from the frozen source dataset. |
| `output_json` | Exact frozen JSON training target, serialized as a string. Parse this cell with a JSON parser. |
| `status` | `accepted`, `unsupported`, `clarification-required`, or `invalid`. |
| `input_variant` | `canonical`, or `paraphrase:<style>` describing the controlled wording construction. |
| `scope` | `ran_subnetwork` or `cell_set` for accepted labels; `not_applicable` for failure labels. |

The IDs support row lookup and grouping; they are not model inputs. `scope` and `status` are browsing aids. For failure rows, `not_applicable` means the failure label has no accepted scope; it does not describe whether the input mentions cells. A wording style such as `teacher_controlled` identifies a construction slot, and does not by itself establish that an API generated the retained sentence.

## JSON labels and target meanings

An accepted training target contains one `intentExpectations` entry with `expectationVerb = ENSURE`, `objectType = RAN_SUBNETWORK`, an NR RAT context, and one or two compatible performance targets. Two targets form a conjunction: both requirements apply.

`output_json` is the **semantic model-training target**. It omits the administrative `userLabel` and generated `expectationId` used in full source payloads. The repository compiler restores those fields deterministically when completing and validating a full intent package. The CSV label alone does not include that package's validation report, normalized semantic frame, or deterministic feature vector.

| Target names | Requested quantity | Unit | Strict condition |
|---|---|---|---|
| `AveDLRANUEThpt`, `AveULRANUEThpt` | Average DL/UL RAN UE throughput over the declared scope | kbit/s | Greater than the bound |
| `AveDLPrbLoad`, `AveULPrbLoad` | Average DL/UL physical resource block (PRB) load over the declared scope | percent | Less than the bound |
| `LowDLRANUEThptRatio`, `LowULRANUEThptRatio` | Percentage of SubNetwork cells whose direction-specific per-cell RAN UE throughput is below a qualifying threshold | percent | Less than the outer bound |
| `HighDLPrbLoadRatio`, `HighULPrbLoadRatio` | Percentage of SubNetwork cells whose direction-specific PRB load is above a qualifying threshold | percent | Less than the outer bound |

DL means downlink; UL means uplink. Throughput values in JSON use **kbit/s**. Natural-language requests commonly use **Mbit/s**, where 1 Mbit/s = 1,000 kbit/s. Percentage values use **0–100**, rather than fractions between zero and one. Equality does not satisfy a strict comparison.

Ratio labels have two separate quantities. The outer `targetValueRange` bounds the percentage of qualifying cells. The nested `targetContexts.contextValueRange` determines which cells qualify. Low-throughput qualifying contexts use `LowDLRANUEThptThreshold` or `LowULRANUEThptThreshold`, a strict less-than condition, and kbit/s. High-load qualifying contexts use `HighDLPrbLoadThreshold` or `HighULPrbLoadThreshold`, a strict greater-than condition, and percent. The low-throughput ratio uses a **cell population**, not a UE population.

SubNetwork requests cover the four target families above in each direction, as single targets and compatible same-direction two-target combinations. Explicit cell scope supports only these six patterns:

| Direction | Supported cell-scoped patterns |
|---|---|
| DL | Average throughput; average PRB load; their conjunction |
| UL | Average throughput; average PRB load; their conjunction |

A cell-scoped label contains a `Cell` context inside `expectationObject.objectContexts`, alongside the NR `RAT` context. Its `IS_ALL_OF` cell list is the scope of the performance expectation. It does not identify cells to reconfigure. Natural-language forms include a single cell, explicit lists, ranges, and mixtures of lists and ranges. Ranges expand into numerically ordered, distinct cell IDs.

The research universe is `NRCell_1` through `NRCell_100`. Retained examples contain **1–22 cells** per scope; the generator's wider permitted cardinality is not fully represented. Ratios, average-plus-ratio combinations, and cross-direction pairs are excluded from accepted explicit cell scope.

**Standards boundary.** The cell identifiers are bounded RAN-IT research placeholders. They do not use the nine-hex-digit `NrCellId` representation required by the pinned official OpenAPI artifact. The source validator retains the official rejection separately from local profile acceptance. Local acceptance therefore does not establish full OpenAPI conformance. The project also documents a narrow semantic correction for high-PRB qualifying contexts in its pinned Release-19 source set; the original standards files were preserved.

A typed failure label contains `result_type = failure`, a status, and an ordered `issues` list with `code`, `path`, and `message`. `unsupported` identifies a request outside supported capabilities; `clarification-required` identifies missing or conflicting information; `invalid` identifies a profile or value violation. Preserve the whole JSON label when reproducing training or evaluation. No accepted intent package or downstream action accompanies a compiler failure.

## Examples from the release

The following are actual canonical rows. JSON is pretty-printed here for reading; the corresponding CSV cells retain the frozen compact serialization.

### SubNetwork throughput request

**Example E0019** (accepted, ran_subnetwork).

> Ensure that for the NR RAN SubNetwork, the average downlink RAN UE throughput is greater than 100 Mbit/s.

```json
{
  "intentExpectations": [
    {
      "expectationObject": {
        "objectContexts": [
          {
            "contextAttribute": "RAT",
            "contextCondition": "IS_ALL_OF",
            "contextValueRange": [
              "NR"
            ]
          }
        ],
        "objectType": "RAN_SUBNETWORK"
      },
      "expectationTargets": [
        {
          "targetCondition": "IS_GREATER_THAN",
          "targetName": "AveDLRANUEThpt",
          "targetValueRange": 100000
        }
      ],
      "expectationVerb": "ENSURE"
    }
  ]
}
```

### Cell-scoped request

**Example E1406** (accepted, cell_set).

> Ensure that for NR cells 7, 24, 39, 41, 56, 58, 73, and 90 in the RAN SubNetwork, the average downlink RAN UE throughput is greater than 500 Mbit/s.

```json
{
  "intentExpectations": [
    {
      "expectationObject": {
        "objectContexts": [
          {
            "contextAttribute": "RAT",
            "contextCondition": "IS_ALL_OF",
            "contextValueRange": [
              "NR"
            ]
          },
          {
            "contextAttribute": "Cell",
            "contextCondition": "IS_ALL_OF",
            "contextValueRange": [
              "NRCell_7",
              "NRCell_24",
              "NRCell_39",
              "NRCell_41",
              "NRCell_56",
              "NRCell_58",
              "NRCell_73",
              "NRCell_90"
            ]
          }
        ],
        "objectType": "RAN_SUBNETWORK"
      },
      "expectationTargets": [
        {
          "targetCondition": "IS_GREATER_THAN",
          "targetName": "AveDLRANUEThpt",
          "targetValueRange": 500000
        }
      ],
      "expectationVerb": "ENSURE"
    }
  ]
}
```

### Ratio with a separate qualifying threshold

**Example E0161** (accepted, ran_subnetwork).

> Ensure that for the NR RAN SubNetwork, the percentage of its cells whose downlink RAN UE throughput is less than 75 Mbit/s is less than 20 percent.

```json
{
  "intentExpectations": [
    {
      "expectationObject": {
        "objectContexts": [
          {
            "contextAttribute": "RAT",
            "contextCondition": "IS_ALL_OF",
            "contextValueRange": [
              "NR"
            ]
          }
        ],
        "objectType": "RAN_SUBNETWORK"
      },
      "expectationTargets": [
        {
          "targetCondition": "IS_LESS_THAN",
          "targetContexts": {
            "contextAttribute": "LowDLRANUEThptThreshold",
            "contextCondition": "IS_LESS_THAN",
            "contextValueRange": 75000
          },
          "targetName": "LowDLRANUEThptRatio",
          "targetValueRange": 20
        }
      ],
      "expectationVerb": "ENSURE"
    }
  ]
}
```

### Typed failure

**Example E0491** (unsupported, not_applicable).

> Ensure average downlink RAN UE throughput is greater than 100 Mbit/s for the WLAN RAN SubNetwork.

```json
{
  "issues": [
    {
      "code": "UNSUPPORTED_RAT",
      "message": "RAT \u0027WLAN\u0027 is outside the NR-only profile",
      "path": [
        "intentExpectations",
        0,
        "expectationObject",
        "objectContexts",
        0,
        "contextValueRange",
        0
      ]
    }
  ],
  "result_type": "failure",
  "status": "unsupported"
}
```

### Failure-extension request

**Example E2438** (invalid, not_applicable), from `test_failure_extension.csv`. "At least" is a non-strict comparison, which the profile's strict conditions reject.

> Ensure that for the NR RAN SubNetwork, the average downlink RAN UE throughput is at least 120 Mbit/s.

```json
{
  "issues": [
    {
      "code": "INVALID_CONDITION",
      "message": "condition 'IS_EQUAL_TO_OR_GREATER_THAN' is not a profile-v1 strict operator",
      "path": [
        "intentExpectations",
        0,
        "expectationTargets",
        0,
        "targetCondition"
      ]
    }
  ],
  "result_type": "failure",
  "status": "invalid"
}
```

## How the dataset was created

1. **Define the semantic vocabulary.** Pin Release-19 TS 28.312 OpenAPI artifacts and project target, context, condition, and unit registries. TS 28.554 and TS 28.552 support metric interpretation. Use the local profile's fixed NR scope, strict comparisons, and permitted target combinations.

2. **Construct legacy gold labels before language.** Enumerate 500 accepted semantic candidates: 250 per direction across ten singleton/conjunction patterns. Assign values from the controlled grids below. For average-throughput plus low-throughput-ratio pairs, require the low-throughput qualifier below the average bound; for average-load plus high-load-ratio pairs, require the high-load qualifier above the average-load bound.

3. **Canonicalize and review.** Verbalize validated candidates with a fixed grammar and check canonical semantic round trips. Human review and replacement select 300 accepted parents, with 150 per direction. Of 60 frozen failure probes, 33 are natural-language requests and 27 are malformed structured-input probes. Only the 33 natural-language failure parents enter this dataset.

4. **Expand legacy language.** Create three paraphrases per accepted parent and two per failure parent, yielding 900 accepted and 66 failure paraphrases. Retained sources comprise 632 deterministic templates, 71 actual teacher outputs, and 263 deterministic fallbacks. The teacher varies wording only; deterministic gold JSON defines labels. Review follows predefined obligations and replacement procedures.

5. **Build legacy partitions.** Keep every semantic parent's variants together. Detect normalized exact duplicates and connect highly similar cross-parent text using token-trigram Jaccard similarity at 0.90. Allocate entire components using deterministic quotas and coverage checks. Accepted parent counts are 210/45/45 and failure parent counts are 23/5/5. The original example counts are 909/195/195.

6. **Augment training failures.** Add four reviewed controlled paraphrases for each of the 23 training failure parents: 92 additional rows. This produces the 1,001-row legacy training successor, preserving the original held-out sets and all existing labels.

7. **Construct the cell extension.** Generate six accepted patterns across five source-expression categories: single cell, explicit list, single range, mixed list/ranges, and multiple ranges. Each pattern has 40 retained parents, giving 240. Review proposals include alternatives for rejected slots. Generate canonical sentences, normalize the exact cell sets, and check semantic round trips. Human review freezes the selected parents. The accepted parent union is then 300 legacy plus 240 cell-scoped parents.

8. **Expand and review cell language.** Create and approve three deterministic paraphrases for each new parent: 720 in total. Constructions vary ranges, explicit enumerations, group order, and sentence syntax while preserving the cell set and targets. Reject duplicates and replace defective proposals through recorded review. Separately approve four additional wording variants for each of the ten held-out legacy failure parents: 40 rows. Cell construction uses deterministic templates and human review, with no teacher/API dependency.

9. **Build the final union and export.** Preserve inherited split assignments. Split each new cell pattern into 28/6/6 parents, giving 168/36/36 cell parents and 672/144/144 cell examples. Add the 40 failure variants to their existing held-out parents. The final counts are 1,673/359/359. Export the eight public columns without changing input text or JSON labels. Combine parent and existing similarity-group relationships to preserve leakage boundaries in the public grouping field.

### Controlled numeric grids

These are candidate-generation grids, not a claim that every possible combination occurs in the retained release. The cell extension uses the same average-throughput and average-load grids.

| Quantity | Candidate values |
|---|---|
| Average throughput, kbit/s | 10,000; 25,000; 50,000; 75,000; 100,000; 150,000; 200,000; 250,000; 300,000; 400,000; 500,000 |
| Average PRB load, percent | 40; 50; 60; 70; 75; 80; 85; 90 |
| Ratio outer bound, percent | 1; 2; 5; 10; 15; 20; 25 |
| Low-throughput qualifying threshold, kbit/s | 10,000; 25,000; 50,000; 75,000; 100,000 |
| High-load qualifying threshold, percent | 70; 75; 80; 85; 90 |

## Failure test extension

The validation and test splits each hold 35 failure requests, but from only five failure parents, so failure-handling results on them carry wide uncertainty. Release 1.1.0 adds `test_failure_extension.csv`: 117 failure requests from 39 new failure parents, for evaluation together with `test.csv`. The two give 152 test failure requests from 44 parents. The extension is test-only: it is never used for training, validation, checkpoint selection, or few-shot demonstrations.

1. **Failure parents.** Each parent is a structured failure probe whose issue code already occurs among the 23 training failure parents, so the extension tests new instances of trained failure behaviour rather than unseen codes. The probes cover other TS 28.312 object types (`5GC_SUBNETWORK`, `EDGE_SERVICE_SUPPORT`, `RADIO_SERVICE`) and a tracking area; the UTRA, LTE, and NB-IoT RATs and a mixed NR/E-UTRA scope; unsupported KPIs such as jitter, handover success, and cell availability; missing values, KPIs, NR scope, or object type; ratio thresholds missing inside two-target requests; reversed comparisons; cross-direction pairs; non-strict comparisons (equal to, at least, at most); and out-of-range values.
2. **Labels before language.** The JSON label of every parent is derived from its probe by the same deterministic validator that reproduces all 33 core failure labels byte for byte; no label is written by hand. Value-range labels include two secondary `INVALID_CARDINALITY` issues produced by that validator, as every training value-range label does.
3. **Uniqueness.** No probe equals any of the 60 frozen failure probes or another new probe, and no wording equals an existing failure wording after case and whitespace folding. Canonical wordings follow the profile's canonical grammar, so some are close in surface form to core failure requests.
4. **Wording.** Each parent has one canonical utterance and two paraphrases that change sentence structure. The wordings were drafted with the help of an AI coding assistant during dataset construction, not with the teacher model used for core paraphrases.
5. **Review.** One domain expert reviewed every parent and wording. The first round approved 33 of 39 parents and 97 of 117 wordings. A request without a verb was replaced by a request without an object type, and two wordings were replaced; the project owner approved these replacements in a second round. Five value-range parents rejected for their secondary `INVALID_CARDINALITY` issues were kept by the project owner, because the training labels follow the same convention.
6. **Export.** IDs continue the release 1.0.0 numbering, and each extension parent forms its own leakage group.

| Status | Issue code of the parent | Parents | Examples |
|---|---|---:|---:|
| `unsupported` | `UNSUPPORTED_OBJECT_TYPE` | 4 | 12 |
| `unsupported` | `UNSUPPORTED_RAT` | 4 | 12 |
| `unsupported` | `UNSUPPORTED_TARGET_NAME` | 5 | 15 |
| `clarification-required` | `CLARIFICATION_RATIO_CONTEXT_MISSING` | 4 | 12 |
| `clarification-required` | `CLARIFICATION_REQUIRED_FIELD_MISSING` | 4 | 12 |
| `clarification-required` | `CLARIFICATION_TARGET_CONDITION_CONFLICT` | 5 | 15 |
| `clarification-required` | `CLARIFICATION_TARGET_DIRECTION_CONFLICT` | 4 | 12 |
| `invalid` | `INVALID_CONDITION` | 4 | 12 |
| `invalid` | `INVALID_VALUE_RANGE` | 5 | 15 |

The extension contains no Cell-scoped failure requests and no issue codes that are absent from the training failures. Report failure results by parent as well as by request: the three wordings of a parent are not independent tests.

## Quality checks and limitations

Human review approved **694 of the 966 historical paraphrases**, comprising 628 accepted paraphrases and all 66 historical failure paraphrases. Review obligations included singleton requests, actual teacher outputs, failure requests, flagged replacements, and a fixed conjunction sample. The remaining historical paraphrases followed deterministic construction and automated checks. All 92 additional training failure variants, 240 new cell parents, 720 cell paraphrases, and 40 added held-out failure variants were reviewed.

Canonical examples and new cell wording have semantic replay checks. Some historical free-form teacher and fallback paraphrases cannot be replayed by the bounded reference parser; their quality evidence is controlled construction and the defined review procedure. Every language example should not be described as independently manually reviewed or universally proven equivalent by parser round trips.

The corpus is bounded and synthetic. It represents controlled English requests, selected numeric grids, and supported target patterns rather than unrestricted operator language or live network performance. The failure corpus derives from historical parents and offers limited coverage of cell-specific rejection cases. The five failure parents in each held-out split do not cover every failure reason; the failure extension adds 39 test-only parents for evaluation, but no training or validation failure parents. This core release includes no separate ontology out-of-distribution benchmark.

The supplied splits prevent observed parent/group leakage and preserve the project's exact/near-duplicate checks. They do not demonstrate generalization to arbitrary unseen numbers, cell identifiers, target families, or writing styles. Use `leakage_group_id` when constructing alternative splits; a random row split can place paraphrases of the same intent in both training and evaluation.

Release checks verify file counts, status counts, unique example IDs, one canonical example per parent, unchanged source sentences and JSON, identical labels within each parent, parent/group disjointness, UTF-8 CSV reload, and exact ZIP membership. For release 1.1.0 they also verify that the three core CSVs are byte-identical to release 1.0.0 and that the extension rows equal the sealed extension artifact, with unique IDs, three wordings and one canonical example per parent, one label per parent, and no parent or group shared with the core splits.

## Load the data and retrieve canonical wording

Run from the directory containing the extracted CSVs. Only the Python standard library is needed.

```python
import csv
import json
from pathlib import Path

def load_split(name):
    with Path(f"{name}.csv").open(encoding="utf-8", newline="") as f:
        rows = list(csv.DictReader(f))
    for row in rows:
        row["parsed_output"] = json.loads(row["output_json"])
    return rows

train = load_split("train")
canonical_by_parent = {
    row["parent_id"]: row["natural_language"]
    for row in train if row["input_variant"] == "canonical"
}
example = train[0]
print(example["natural_language"])
print(json.dumps(example["parsed_output"], indent=2))
print(canonical_by_parent[example["parent_id"]])
```

The same function loads the failure extension: `load_split("test_failure_extension")`.

## GitHub repository

This dataset was used for the RAN-IT project in [ManourasM/NOIRA](https://github.com/ManourasM/NOIRA).

## License and citation

The dataset CSVs and this README are released under **[CC BY 4.0](https://creativecommons.org/licenses/by/4.0/)**. Give appropriate attribution, include the license link, and indicate modifications when redistributing an adapted dataset. Referenced standards and external source material retain their own terms.

For citation, use **RAN-IT Intent Dataset with Cell Scope, version 1.1.0 (2026)** and the permanent URL or DOI assigned to the uploaded dataset record. Include the creators as listed on that record. Also cite the associated RAN-IT paper when its final bibliographic details are available. This local archive does not claim an assigned DOI or a published paper record.
