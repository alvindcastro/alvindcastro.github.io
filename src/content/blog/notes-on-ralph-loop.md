---
title: "Ralph Loop"
date: 2026-05-07T00:00:00Z
---

# The Ralph Wiggum Loop And The Engineering Process Behind It

The Ralph Wiggum loop describes a familiar failure in agentic software. The system says it is helping, writes a report, updates progress, and moves on with the confidence of a status meeting that found a microphone. Yet the repository may still lack tests, runtime code, validators, fixtures, or evidence that the promised work exists.

The process used here is different from a pure Ralph loop. It treats useful agent work as an engineering problem, not a performance of confidence. The model may reason probabilistically, but the surrounding loop must give it a bounded task, capture its evidence, and make completion inspectable.

One practical disclaimer belongs near the front: this process can burn tokens. Long-running audits, high reasoning effort, web research, document-everything mode, and repeated story attempts all add cost. This is not a button to press casually before lunch unless lunch includes reading logs and checking the bill.

## The Failure Mode

The failure begins when an AI workflow rewards activity instead of evidence. A backlog can sound complete while no executable path exists. A model can describe safety controls while no validator enforces them. A report can declare progress while no human can reproduce the result, which is where the cheerful loop starts wearing an engineering badge it has not earned.

This failure is not mainly a model problem. It is a systems problem. A strong model inside a weak loop still produces weak engineering results, because the loop has not defined scope, authority, artifacts, or completion.

The process responds by making each of those parts explicit. It asks the agent to inspect a defined slice of the repository. It tells the agent what it may and may not do. It requires findings to cite local evidence. It records the output where the story says the output belongs.

## What The Process Does

The process runs through a Bash audit runner. The runner reads a PRD file, selects the first incomplete story by priority, builds a prompt for Codex, and runs the agent in read-only mode. When Codex returns a non-empty final report, the wrapper writes that report to the story's declared `output_file` and marks the story passed.

The distinction between agent behavior and wrapper behavior matters. Codex runs with `codex exec -s read-only`, so the agent inspects and reports. The wrapper writes operational artifacts, including reports, logs, progress state, attempt counts, and PRD timestamps. This split keeps the audit role clear.

The runner also validates report paths before writing them. Output must stay under the audit report directory. This small rule prevents a story from turning a report path into an arbitrary file write.

## The PRD As Work Queue

The PRD is the loop's work queue. Each story carries an id, title, priority, prompt, scope, acceptance criteria, notes, pass state, and output file. The process does not ask the agent to choose the next useful thing. It gives the agent one scoped audit and one report contract.

That structure makes the result reviewable. A story can ask for a shell-safety audit, a read-only boundary audit, a PRD-schema audit, a logging audit, or a testing audit. The agent must inspect the relevant files and write findings against that scope.

The pass condition stays narrow. A story passes when Codex exits successfully and produces a final message. That does not prove the audited software is correct. It proves only that the audit story produced its required artifact, which is the correct standard for an audit loop.

## Evidence Over Applause

The `--document-everything` mode raises the evidence standard. In that mode the agent must record every evidence-backed issue in scope, including low-severity inconsistencies, stale references, missing pieces, duplicate patterns, and small documentation mismatches. It must not turn suspicion into fact.

The report format forces useful discipline. Each finding needs a severity, category, file path, line number when available, safe excerpt, description, impact, and list of affected instances. Unconfirmed matters belong under `Needs Verification`, not under findings.

This design changes the tone of the loop. The agent does not win by sounding helpful, no matter how politely it arranges the headings. It wins by producing a report that another engineer can check against files in the repository.

## MVP Planning As Example

One sample MVP plan shows why this matters. Its PRD describes a realistic applied AI workflow with synthetic inputs, retrieval, timeline generation, classification, approval gates, evaluation, observability, and prompt-injection defenses. The plan is strong enough to sound like progress.

The process treated the plan as a claim to audit. The evaluation report found no executable harness, no eval dataset, no grader contract, and no reproducibility schema. The observability report found no implementation surface for traces, cost tracking, tool outcomes, or reviewer trace UI. The security report found no prompt templates, tool validators, malicious retrieval fixtures, or output safety contracts.

Those findings are valuable because they name the gap between design and delivery. A pure Ralph loop might praise the plan and call the meeting productive. This process points at the missing engineering artifacts, which is less charming but much more useful.

## Engineering Choices

This process uses plain mechanisms because plain mechanisms are easier to inspect. Bash orchestrates the run. `jq` mutates structured PRD state. Codex performs the audit in read-only mode. Logs capture high-level events in `events.log` and full CLI output in `run.log`.

The loop also contains guardrails for ordinary failure. A per-story attempt counter prevents one bad story from trapping the run. A lock directory blocks concurrent runners. A model preflight checks the requested Codex model before a long run starts. A credential preflight warns when common secret-bearing environment variables are set.

Web search remains an explicit operating choice. The runner enables search by default because some audits depend on current behavior from external tools or APIs. Operators can pass `--no-search` for sensitive repositories, and the agent instructions require source discipline when external research supports a finding. This choice also affects cost, so the practical answer is to choose it deliberately.

## What Engineers Get

Engineers get a concrete way to review agent work. Did the agent inspect the files in scope? Did it write the expected report? Did each finding cite local evidence? Did the wrapper mutate only the expected state? Can the run be diagnosed from logs?

These questions matter because agent output often substitutes a polished artifact for a working system. This process makes that substitution visible. It separates a good plan from implemented behavior, and it separates a confident report from an evidence-backed one.

## What Comes Next

The current loop is audit-first by design. Before agents implement broad changes, they should help us map scope, find drift, and preserve evidence. A reliable audit loop gives later implementation work a cleaner starting point.

The next improvements should tighten the contracts. The PRD schema needs stronger validation. Runner failure modes need automated smoke tests. Audit outputs need better indexing. Generated artifacts need cleaner hygiene. External research needs a sharper boundary from private local evidence.

The Ralph Wiggum loop says, "I'm helping," and asks the engineer to believe it. This process asks for evidence first. That is the engineering difference.
