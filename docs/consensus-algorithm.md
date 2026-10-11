# Multi-model consensus algorithm

[← 中文首页](../README.md) · [English home](../README_EN.md)

> The pseudocode on this page is an independently written portfolio explanation. It describes the design without reproducing private course source code.

## Why consensus is needed

A single model can be confidently wrong, sensitive to prompt phrasing, or inconsistent about source quality. VeriFact therefore asks configured models to assess the same sub-claim against the same retrieved evidence, preserves every valid response, and calculates a consensus rather than selecting one provider blindly.

## Inputs and outputs

Each valid model result contains:

```text
provider key
model name
verdict: Supported | Refuted | Uncertain
confidence: 0–100
evidence summary
reasoning steps
source URLs
search-quality assessment
```

The consensus stage returns:

```text
consensus verdict
consensus confidence
agreement information
representative evidence and sources
all provider-level results for audit
```

## Per-sub-claim consensus

```text
function consensus(results, provider_weights):
    valid = results that passed structured-output validation

    if valid is empty:
        return Uncertain with confidence 0

    groups = group valid by verdict
    total_weight = sum(weight(result.provider) for result in valid)

    for each verdict in [Supported, Refuted, Uncertain]:
        group_weight[verdict] =
            sum(weight(result.provider) for result in groups[verdict])

        weighted_confidence[verdict] =
            sum(result.confidence * weight(result.provider)
                for result in groups[verdict])
            / max(group_weight[verdict], epsilon)

    winner = verdict with greatest group_weight

    if group_weight[winner] >= 0.66 * total_weight:
        confidence = weighted_confidence[winner]
    else if winner has a strict weighted plurality:
        agreement_ratio = group_weight[winner] / total_weight
        confidence = weighted_confidence[winner] * agreement_ratio
    else:
        return Uncertain with reduced confidence

    if evidence is missing or confidence is below the safety floor:
        return Uncertain with an explicit disclaimer

    return winner, bounded(confidence, 0, 100), provider-level results
```

The implementation also considers source overlap and applies disagreement penalties. The important property is that adding another weakly agreeing model does not automatically outweigh a more capable provider, while lack of consensus remains visible.

## Global claim aggregation

A user input may produce several sub-claims. Their consensus results are aggregated conservatively:

| Sub-claim pattern | Global behaviour |
| --- | --- |
| All material claims supported | `Supported` with aggregate confidence |
| At least one high-confidence material claim refuted | Bias toward `Refuted` |
| Mixed low-confidence support and refutation | `Uncertain` or penalised majority |
| All evidence insufficient | `Uncertain` |

The asymmetry is deliberate: a compound statement can be invalidated by one materially false component, while one true component does not prove the entire statement.

## Worked example

Assume three providers with weights `1.0`, `0.8`, and `0.6` return:

| Provider | Weight | Verdict | Confidence |
| --- | ---: | --- | ---: |
| A | 1.0 | Supported | 88 |
| B | 0.8 | Supported | 76 |
| C | 0.6 | Refuted | 70 |

Supported weight is `1.8 / 2.4 = 75%`, exceeding the weighted two-thirds threshold. Its weighted confidence is:

```text
(88 × 1.0 + 76 × 0.8) / 1.8 = 82.7
```

The consensus is therefore `Supported`, approximately 83 before any evidence-quality or disagreement adjustment. Provider C's disagreement remains attached to the report.

## Alternatives considered

| Approach | Advantage | Limitation |
| --- | --- | --- |
| Use one preferred model | Cheapest and simplest | Single-provider failure and bias |
| Unweighted majority | Easy to explain | Treats all providers as equally reliable |
| Highest confidence wins | Minimal aggregation logic | Confidence scales differ between providers |
| Capability-weighted consensus | Preserves diversity and reliability policy | Weights require calibration and maintenance |

The project uses capability-weighted consensus as a pragmatic prototype choice, not as proof that model capability can be captured by a permanent scalar.

## Complexity

For `C` sub-claims and `M` configured models:

- model assessments: `O(C × M)` external calls, executed concurrently where possible;
- consensus calculation: `O(C × M)` local work; and
- memory: `O(C × M)` to retain auditable provider results.

Network latency and provider cost dominate local consensus computation.

## Production improvements

- Estimate provider weights from a labelled evaluation set by domain.
- Calibrate confidence values before combining providers.
- Separate evidence quality, model confidence, and agreement into explainable factors.
- Detect correlated providers so several similar models do not create artificial consensus.
- Version the consensus policy with each stored result.
- Add offline evaluation for accuracy, calibration error, abstention quality, latency, and cost.
