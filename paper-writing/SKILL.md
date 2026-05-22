---
name: paper-writing
description: Guidance for writing, rewriting, reviewing, or outlining database systems research papers for VLDB, SIGMOD, PVLDB, CIDR, and similar venues. Use when working on paper structure, introductions, contributions, research questions, evaluation sections, figures, tables, related work, limitations, or reviewer-facing clarity for systems papers.
---

# Database Systems Paper Writing

Use this skill when shaping a database systems paper into a clear argument. The paper is not a log of work performed. It is a claim, supported by design, implementation, and evaluation evidence.

## Core Rule

Every section, figure, table, theorem, example, and experiment must help the reader answer:

1. What problem matters?
2. What is the key idea?
3. Why is it technically credible?
4. What evidence shows it works?
5. Where does it not work?

If an element does not answer one of these questions, rewrite it, move it to an appendix, or remove it.

## Paper Shape

### Diagnosis Before Design

For a systems paper, do not introduce the design before the reader understands the bottleneck. A strong flow is:

1. Pick one representative workload, query, or operation.
2. Show the surprising behavior with a simple measurement or example.
3. Explain why the obvious existing designs fail.
4. Derive design requirements from that evidence.
5. Present the architecture as the direct consequence of those requirements.
6. Return to the same workload in evaluation to show the bottleneck has moved or disappeared.

This pattern is especially effective for DBMS papers because it turns implementation choices into necessary consequences rather than arbitrary engineering.

### Introduction

Build the introduction as an argument:

1. State the concrete problem and why database researchers should care.
2. Explain why existing systems or techniques do not solve it in this setting.
3. Give the core insight in one or two crisp paragraphs.
4. Name the system, mechanism, or technique.
5. Preview the strongest evidence, preferably tied to a figure or headline number.
6. List contributions that are specific, falsifiable, and mapped to the rest of the paper.

Do not start with a broad history lesson. Start from a workload, bottleneck, missing capability, or concrete failure mode.

### Running Example

Use one running example to teach the paper. For database systems, this can be one query, one physical plan, one operator, one schema fragment, or one attack.

The example should:

- be simple enough to understand without knowing the whole system
- expose the core technical difficulty
- reappear when explaining the design
- connect to at least one evaluation workload

Avoid switching examples every section. Readers should feel the system becoming more general, not restart from scratch each time.

### Design Space

When the paper combines several mechanisms, include a compact design-space view before the detailed design. Use it to show the alternatives the authors considered and why the chosen point is the right one.

A useful table has columns such as:

| Design | What it buys | What it costs | When it fails |
|---|---|---|---|

This is especially helpful when a paper compares privacy notions, execution strategies, storage layouts, rewrite strategies, or optimizer placements. It prevents the design section from reading like a sequence of unrelated features.

### Contributions

Write contributions as claims, not tasks.

Good contributions:

- "We show that X is the dominant bottleneck under Y workload."
- "We introduce Z, which changes the execution model by doing A instead of B."
- "We implement Z in an open-source DBMS and evaluate it on workloads W."
- "We identify failure modes F and introduce safeguards S."

Weak contributions:

- "We implemented a system."
- "We ran experiments."
- "We discuss related work."
- "We propose an approach" without saying what is technically new.

Each contribution should have a corresponding section and at least one evaluation question, experiment, proof, or artifact claim.

## Figures And Tables

Every figure must have one message. Before including it, write the message in a sentence:

> "This figure shows that ..."

If that sentence is vague, the figure is not ready.

Use captions as miniature arguments:

1. Start with the takeaway.
2. Then state the setup: workload, scale, machine, metric, or configuration.
3. Mention the baseline when relevant.
4. Explain surprising effects in text, not only in the caption.

Avoid decorative diagrams. Architecture figures must explain data flow, control flow, or responsibility boundaries. Performance figures must answer an evaluation question. Tables must support comparison, coverage, parameters, or failure analysis.

Do not force one overloaded figure to answer several questions. Split it, or choose the strongest message.

## Evaluation

Start the evaluation with research questions. Use RQs to make the evaluation legible:

- RQ1: Does the core technique solve the target bottleneck?
- RQ2: What is the end-to-end overhead or benefit on realistic workloads?
- RQ3: Which design choices matter? Use ablations.
- RQ4: What quality, robustness, or correctness tradeoff is introduced?
- RQ5: What breaks? Include stress tests, unsupported cases, or attacks when relevant.

Every RQ should map to experiments and figures. Every major contribution should map to at least one RQ.

For systems papers, include:

- A reproducible setup: hardware, software versions, data scale, settings, repetitions.
- Fair baselines: default system behavior, prior work, simple alternatives, and ablations.
- End-to-end workloads: not only microbenchmarks.
- Microbenchmarks: only when they isolate a mechanism or explain a result.
- Upper-bound or lower-bound baselines when useful: hand-coded kernels, no-op variants, oracle choices, or idealized versions.
- Parameter sweeps for the variables the design claims to control: data size, group count, selectivity, concurrency, skew, memory, or privacy budget.
- Failure cases: cases where the approach is slower, less accurate, unsupported, or unsafe.
- Interpretation: explain why curves have their shape and why tables differ.

Do not present results as a scoreboard only. The goal is insight: what did the experiment teach?

### Evidence Ladder

Build evaluation evidence in layers:

1. Diagnostic measurement: show the original bottleneck or failure mode.
2. Idealized baseline: show what is possible if unrelated system costs are removed.
3. Microbenchmark: isolate the proposed mechanism.
4. Ablation: remove one design choice at a time.
5. Integrated system result: show the mechanism inside the DBMS.
6. Full workload: show end-to-end behavior on accepted benchmarks or realistic workloads.
7. Stress test: show where the design breaks or loses utility.

Not every paper needs all seven layers. But if a claim is central, it should appear at more than one layer.

### Performance Claims

For hardware-conscious or execution-engine papers, explain performance through mechanism, not just through wall-clock time.

Use counters, traces, operator breakdowns, or plan-level analysis when the claim depends on CPU, cache, memory bandwidth, I/O, vectorization, compression, concurrency, or optimizer behavior. If the paper says "faster because SIMD," "faster because fewer joins," or "faster because less I/O," include evidence that isolates that cause.

For central performance claims, include a simple accounting model. State what cost is paid per tuple, group, aggregate, column, query, worker, privacy unit, or world. The model does not need to be elaborate, but it should predict the shape of the curves and make the evaluation easier to interpret.

## Systems Paper Style

Prefer this flow for database systems work:

1. Observe a real bottleneck, limitation, or missing capability.
2. Use a simple example or measurement to make the issue concrete.
3. Derive design requirements.
4. Present the design from simple case to full system.
5. Explain how it fits into the DBMS: parser, optimizer, execution engine, storage, transactions, or extension API.
6. Evaluate both isolated components and full workloads.
7. State limitations before reviewers discover them.

Make engineering choices defensible. If the design is simple, say why simplicity is the right choice. If the implementation is constrained by the host DBMS, explain the constraint and its consequence.

### Complete System vs. Feature Paper

Be explicit about what kind of systems paper this is.

For a complete-system paper:

- show the architecture and the major subsystems
- explain how features interact, not only how each works alone
- include lessons learned from integration
- evaluate workloads that exercise the whole system

For a focused feature or extension paper:

- state the narrow technical problem
- explain why the host system makes the problem nontrivial
- show the integration points precisely
- evaluate both the feature in isolation and the host-system overhead it introduces

Do not let reviewers conclude "this is just engineering." Name the design problem, the constraints, and the evidence that the solution changes what the system can do.

### Claim Ledger

Before a submission pass, create a private ledger:

| Claim | Section | Figure/Table | Baseline | Limitation |
|---|---|---|---|---|

Every headline claim should have all five columns filled. If a claim has no figure, table, theorem, or experiment, either add evidence or weaken the wording.

## Related Work

Related work should position the contribution, not prove that the authors read widely.

For each cluster of work:

1. State what the area solves.
2. State what it does not solve for this paper's setting.
3. Explain how this paper differs.

Do not bury threat-model differences, workload differences, or assumptions. These are often the actual distinction.

## Limitations

A strong systems paper is explicit about boundaries:

- unsupported SQL constructs or workloads
- assumptions on schemas, constraints, distributions, or hardware
- cases where utility, performance, or security degrades
- engineering choices that are pragmatic but incomplete
- experiments that remain to be rerun for final numbers

Limitations should not read like apologies. They should clarify the contribution and prevent reviewers from finding the same issue first.

## Rewrite Checklist

Use this checklist when reviewing a draft:

- Can a reviewer state the paper's claim after page 1?
- Does the introduction distinguish this work from prior work?
- Are contributions specific and testable?
- Does every figure have exactly one message?
- Does every caption start with the takeaway?
- Are all RQs answered by results?
- Does every major design choice have a reason?
- Are baselines fair and named precisely?
- Are negative results and limitations visible?
- Is the evaluation setup detailed enough to reproduce?
- Is related work organized by ideas rather than paper-by-paper summaries?
- Are old-paper leftovers still aligned with the new message?
- Is there a diagnostic result before the design?
- Is there a running example that carries through the paper?
- Is there a design-space view when the paper has multiple mechanisms or alternatives?
- Does each headline number have a fair baseline and setup?
- Does each central performance claim have a cost/accounting explanation?
- Are upper-bound or lower-bound baselines used where they clarify the real bottleneck?
- Are integration costs separated from algorithmic costs?

## Common Fixes

- If the paper feels like a manual, add a stronger claim and reorder around it.
- If the paper feels like an implementation report, add the bottleneck and design requirements.
- If reviewers may ask "why not DP / why not prior system / why this guarantee," answer before the system section.
- If a section has many small subsections, merge them into fewer conceptual blocks.
- If an experiment has no RQ, remove it or add the RQ it answers.
- If a figure caption merely names the plot, rewrite it to state the conclusion.
- If the evaluation feels scattered, reorder it by RQ and move setup details to the first evaluation paragraph.
- If the architecture section feels arbitrary, add the design requirements that force the architecture.
- If the paper has many optimizations, identify which one is the core idea and make the others supporting ablations.
- If the design section feels like a catalog, add a design-space table and collapse mechanisms by the tradeoff they address.
- If a performance plot has no explanation, add the cost model or operator accounting that predicts its trend.
