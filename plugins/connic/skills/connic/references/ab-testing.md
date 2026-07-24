# A/B testing

Canonical docs: `https://connic.co/docs/v1/test/ab-testing`

A/B tests compare a base agent (control) with a deployed test variant in the same environment. Both analysis modes track reliability, cost, latency, and judge scores.

## Contents

- [Create a variant](#create-a-variant)
- [Choose an analysis mode](#choose-an-analysis-mode)
- [Traffic assignment](#traffic-assignment)
- [Configure a test](#configure-a-test)
- [Safety guardrails](#safety-guardrails)
- [Read results](#read-results)
- [Lifecycle](#lifecycle)

## Create a variant

Use the naming convention `{base-agent}-test-{name}`. The prefix before `-test-` must match an existing base agent or deployment fails. A variant is a normal agent YAML file and can change the model, prompt, tools, temperature, or other agent settings.

```text
agents/
├── order-processor.yaml
├── order-processor-test-faster-model.yaml
└── order-processor-test-new-prompt.yaml
```

```yaml
# agents/order-processor-test-faster-model.yaml
name: order-processor-test-faster-model
model: gemini/gemini-2.5-flash
description: "Processes incoming customer orders"
system_prompt: |
  You process incoming orders...
tools:
  - orders.process
  - inventory.check
```

To test a tool implementation, create another module and reference it only from the variant:

```text
tools/
├── orders.py
└── orders_v2.py
```

## Choose an analysis mode

### Confidence

Use Confidence when the result needs a statistically controlled recommendation. Configure:

| Setting | Meaning |
| --- | --- |
| Primary metric | Success rate, or a normalized score from an enabled automatic judge that evaluates every run without filters |
| Objective | Superiority, or non-inferiority with a declared margin |
| Assumptions | Better direction, baseline, minimum detectable effect or margin, confidence level, power, and optional futility threshold |
| Randomization | Independent runs, or sticky assignment by trusted session identity |
| Preview | Calculated control, variant, and total sample targets |

Confidence tests analyze at four planned looks: 25%, 50%, 75%, and 100% of target information. Connic uses O'Brien–Fleming boundaries and reports a sequentially adjusted interval. Before the first look, the observed effect and interval are descriptive only.

### Exploratory

Use Exploratory for descriptive comparison when no statistically controlled winner claim is needed. Set a minimum completed-run count per group, then decide when to pause, conclude, or change the test.

## Traffic assignment

The traffic percentage is the share routed to the variant; the remainder stays on control.

- Confidence requires traffic in both arms and allows only one confidence test per agent and environment.
- Exploratory tests can run concurrently when their combined variant traffic is at most 100%.
- Run randomization assigns each run independently.
- Session randomization keeps a trusted session identity in one arm; only its first eligible run contributes the session outcome.
- Pausing, concluding, invalidating, or failing a test returns new traffic to the base agent.
- A manually forced control or variant run stays visible in history but is excluded from Confidence analysis and safety checks.

## Configure a test

1. Deploy the test variant beside the base agent.
2. Open the base agent and select **Manage A/B Tests**.
3. Select **New Test**, choose the variant and analysis mode, and set the traffic split.
4. For Confidence, review the sample plan and optional safety guardrails.
5. Create the draft, then select **Start** to route traffic.

## Safety guardrails

Confidence tests support two independent auto-pause rules:

- **Failure auto-pause:** pause when the variant failure rate exceeds a threshold across a complete rolling window of terminal runs.
- **Quality auto-pause:** pause when the recent normalized score from an automatic judge falls below a floor after the configured minimum evaluations.

A judge used as the Confidence metric or quality guardrail must be enabled, automatic, sampled at 100%, and have no run filters.

## Read results

The detail view compares:

- completed runs per arm
- average cost
- average duration plus P50 and P95 latency
- judge score and judge coverage
- success rate

Confidence also shows the primary metric, effect, sample progress, completed and upcoming looks, adjusted interval, allocation validity, and current recommendation. A planned look can recommend the variant, recommend control, continue collecting, or stop as inconclusive. Allocation mismatch, failed judge analysis, or a safety breach pauses without a winner claim.

Runs routed to variants show a test pill in run history and can be filtered by variant.

## Lifecycle

- Tests start as `Draft`, move to `Running`, and can be paused or concluded.
- Exploratory settings remain editable while running or paused.
- A Confidence plan, allocation, and safety settings lock on first start.
- A manually paused Confidence test can resume with its locked plan.
- A Confidence test stopped by a decision boundary, safety rule, allocation mismatch, or analysis error cannot resume.
- Started Confidence tests retain assumptions, judge snapshots, assignments, and completed looks; they cannot be deleted or assigned a manual winner.
- If a deployment removes the variant used by a running or paused test, the test becomes `Failed` and traffic returns to control.
