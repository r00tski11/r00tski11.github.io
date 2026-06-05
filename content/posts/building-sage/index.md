---
title: "Building SAGE: a security audit automation framework (Part 1)"
date: 2026-06-04
description: "How I rebuilt my audit workflow as a 5-command framework, plus what changed from v1."
tags: ["security", "automation", "appsec", "llm", "audit", "owasp"]
categories: ["security"]
toc: true
tocOpen: true
---

> **tldr:** I wanted to build a security assessment automation framework: something that reasons through code the way an application security engineer would and compounds what it knows with every engagement. So I built SAGE (Security Automation and Governance Engine), covering the full lifecycle in five commands:
>
> 1. Workspace setup
> 2. Static analysis (semgrep, trufflehog, eight dep scanners)
> 3. Dynamic testing (Burp Suite Pro, Playwright, Nuclei, sqlmap, ffuf)
> 4. Five-layer false-positive triage
> 5. Final reporting with KPI and cost tracking
>
> What SAGE learns on one engagement feeds into the knowledge base for the next, creating a self-improving closed loop. The verification, chaining, and final call on every finding are still mine. This post covers the architecture and how it got here.

Running an application security assessment means more than running tools and logging what they flag. It means tracing data flows across module boundaries, chaining findings into something catastrophic, evaluating whether a flagged path is actually reachable in the deployment context, and making judgment calls no scanner is going to make for you. The tooling exists. What doesn't is something that ties it all together, gets smarter about the specific patterns it's seen before, and scales across an org the way a single-engineer manual process never will.

So I built SAGE (Security Automation and Governance Engine): a closed-loop security assessment automation framework that reasons through code the way an application security engineer would and compounds what it knows with every engagement. It boils down to five commands covering setup, static analysis, dynamic testing, triage, and reporting. The framework handles the scaffolding and tool orchestration so I can spend my time on the actual analysis and manual verification instead.

I'll cover the results in a follow-up post once I've run enough assessments through SAGE to have something worth analysing.

## v1: the white-box workflow

I was running security assessments on multiple products in parallel. Each one needed the same OWASP coverage, the same toolchain, the same evidence standards. I was copy-pasting the same setup steps across engagements and still missing things. The process needed to be codified, not repeated from memory every time.

I started building the first version in late February 2026. Two posts I'd been reading around that time shaped the approach. Addy Osmani's [AI coding workflow](https://addyosmani.com/blog/ai-coding-workflow/) and Boris Tane's [writeup on using Claude Code](https://boristane.com/blog/how-i-use-claude-code/). Both structure AI-assisted work into phased processes with human review gates at each step. I took that model and applied it to security assessments.

The first version was a Claude Code skill suite. It started with around ten skills covering the core phases, but as edge cases came up (adjusting findings mid-triage, handling quality checks, managing workspace state), it grew to fifteen. `/new-audit`, `/intake`, `/scan`, `/analyze`, `/finding`, `/triage`, `/report`, plus others for dynamic testing, cleanup, and wrap-up. Each skill ran one phase. Every finding had to carry a `file:line` reference and pass a five-layer false-positive check before it got recorded.

It worked. I ran a few assessments with it and the hit rate was solid. Most of the findings it surfaced were true positives. A handful turned out to be non-issues in the specific product context, and only a couple were outright false positives. Every finding still got a manual review on my end: verifying exploitability in context, checking if findings could be chained together for higher impact, running additional manual tests where the automated pass flagged something worth digging into, and making sure the evidence and reproduction steps actually held up.

The structure was solid. OWASP WSTG v4.2 and MASTG coverage, plus patterns I'd picked up from PortSwigger Academy labs and Hack The Box machines that I'd been noting down as I solved them. A language-agnostic SAST chain (semgrep, trufflehog, npm-audit, pip-audit, OWASP Dep-Check, retire.js, govulncheck, bundler-audit, cargo-audit). Evidence-first finding records.

But fifteen skills is too many. I'd forget the order. Was it `/intake` then `/scan`, or did `/intake` happen inside `/new-audit`? I ended up making a quick-ref file just to remember the sequence. The reason I hadn't combined them earlier is that I wanted to fine-tune each phase separately so I knew it worked exactly as intended and didn't add slop to the next step. But the cognitive overhead of juggling fifteen separate skills was real.

The intake was also heavy. Before any assessment work started, the framework asked ten-plus questions. App name, tech stack, target path, app type, entry points, auth mechanism, sensitive data, known concerns. I was answering things the codebase already knew (after sanitising it with trufflehog and custom semgrep rules to strip secrets and PII before it touched the LLM).

And the workspace state was fragile. If I forgot `/wrap-up` before ending a session, progress was lost. The state file lived inside the assessment workspace, so every new engagement needed a fresh template copy.

A few things became clear after running this for a while:

1. **The phases are stable, the targets aren't.** The assessment steps don't change between products. What changes is the app. Once I'd fine-tuned each phase individually and knew it worked properly, the next step was to abstract them and auto-detect the target.
2. **Findings should compound.** If I already know a particular framework has a pattern of disabling TLS validation, the tool should know that too. What I learn in one assessment should feed into the next.
3. **My time should go to judgment, not orchestration.** If I'm spending cycles remembering which command to run next, that's time not spent on the actual security analysis.

## v2: the abstraction

I collapsed the fifteen skills down to five commands. Five, not fifteen.

| Command | What it does |
|---------|-------------|
| `/sage-setup` | Initialize the workspace, auto-detect everything from prerequisites and source |
| `/sage-scan` | Static plus deep semantic analysis. Eight OWASP modules. Splittable across sessions |
| `/sage-test` | Multi-role authentication, dynamic verification, Burp replay, sqlmap, race conditions |
| `/sage-triage` | Classify findings TP/FP/NR, adjust severities, remove false positives |
| `/sage-report` | Generate full report, executive summary, internal tracker, cost and KPI metrics, with QA checks |

The whole thing installs with one symlink into `~/.claude/skills/` and works in any directory. No per-workspace setup, no wondering where files should go.

The workspace is hardcoded too:

```
~/<target-app-name>/
├── prerequisites/     (questionnaires, policies, docs)
├── source/            (symlink or copy of target code)
├── scans/             (static + DAST tool output)
├── findings/          (individual findings + index)
├── reports/           (deliverables)
├── dynamic-tests/     (config + results, screenshots, race, sqlmap)
├── metrics/           (KPI tracking)
└── .sage/             (internal state)
```

Same structure every time. I never have to think about where things go, and neither does anyone else picking up the workspace.

## Auto-detect over questions

v1 asked questions. v2 reads the filesystem.

App name gets pulled from the scoping documents provided before the engagement. Tech stack from `package.json`, `pom.xml`, `pyproject.toml`, `Cargo.toml`, `go.mod`. Features like AI, cloud, database, auth, GraphQL come from import statements. Applicable OWASP modules come from a platform-map JSON that ties file signals to modules. Target URL comes from a single config file in the workspace.

I drop the scoping docs into `prerequisites/`, link the source, and `/sage-setup` takes it from there. Zero prompts if everything is in place.

This is the biggest UX win in v2. I went from typing ten answers to typing zero. I still review everything it auto-detects and add context where needed, but the baseline is there without me having to spell it out.

## How findings get filtered

Not everything a scanner flags is real. The framework runs every potential finding through five layers before it gets recorded.

1. **Evidence requirement.** No `file:line` reference, no finding. If the scanner can't point to a specific location in the code, it gets dropped.
2. **Contextual analysis.** Is the flagged code actually reachable? Is there already a mitigation in place (input validation, WAF rule, framework-level protection)? The LLM traces data flows across files and checks whether the vulnerable path is actually callable.
3. **Triage review.** The actual code gets read. Not just the flagged line, but the surrounding context. A hardcoded key in a test fixture is different from a hardcoded key in a production config.
4. **Cross-referencing.** Multiple signals need to converge. A single scanner hit isn't enough. If semgrep flags something and the dependency scan shows a related vulnerable package, that's a stronger signal than either alone.
5. **Explicit uncertainty.** If the evidence isn't clear-cut, the finding gets marked "Needs Dynamic Testing" instead of being forced into a TP or FP bucket. No false confidence.

The triage decision itself follows a simple tree:

```
Was the finding dynamically tested?
├── YES
│   ├── Confirmed exploitable     → True Positive
│   ├── Not reproducible          → likely False Positive
│   └── Needs manual verification → Needs Review
└── NO (static evidence only)
    ├── file:line + reachable + no mitigation → True Positive
    ├── Unreachable or mitigated              → Informational
    └── Speculative / no concrete evidence    → Demote or remove
```

This is where the split between tools, LLM, and human judgement matters most. The scanners do pattern matching. The LLM does the semantic work: tracing data flows, evaluating reachability, checking if a framework already mitigates the issue. And I handle the parts that need real-world context: whether a finding is actually exploitable given the deployment environment, whether it chains with something else, and the final call on anything the pipeline marked uncertain.

## What it orchestrates

SAGE doesn't write its own scanners. It drives existing ones. Tools like Snyk or Semgrep Cloud handle scanning. DefectDojo handles vulnerability management. But none of these tools tie the full assessment lifecycle together: intake, scanning, evidence collection, triage, and reporting in one workflow. That's the gap SAGE fills.

Static: semgrep for SAST, trufflehog for secrets, language-specific dependency scanners (npm-audit, pip-audit, OWASP Dependency-Check, retire.js, govulncheck, bundler-audit, cargo-audit).

Dynamic, via MCP servers: Burp Suite Pro (proxy, Repeater, Intruder, Collaborator for blind SSRF and DNS exfil), Playwright (browser automation routed through Burp), Nuclei (9000-plus templates), sqlmap, ffuf for web fuzzing, pd-tools (subfinder, httpx, katana, naabu, dnsx) for asset validation against what the source says is exposed. Several of these are available as MCP servers through projects like [FuzzingLabs' MCP security hub](https://github.com/FuzzingLabs/mcp-security-hub), which is how SAGE drives them. New tools plug in without rewriting the framework.

Layered on top is a knowledge base. Per-product learnings, false-positive patterns, verified-finding patterns. After each assessment, the lessons feed back into the patterns for the next one. The framework gets sharper over time instead of staying static.

## Design decisions

A few choices I'd defend.

**Five commands, not fifteen.** The security engineer's job is judgment. Skill orchestration is not. Cutting the surface area meant cutting cognitive load.

**Auto-detect over questionnaire.** The disk has the answer. Don't ask the human. But always let the human review and override what was detected.

**Universal install.** One symlink, every assessment. No bootstrap, no template copy, no per-workspace setup. A new engineer on the team gets the same framework on day one.

**Knowledge base with a feedback loop.** Findings from assessment N improve patterns for assessment N+1. Without this, the framework gets stale while the threat landscape doesn't.

**Cost-aware from day one.** Every session logs token counts. The `metrics/` folder tracks per-assessment spend and `/sage-report` rolls it into the deliverables. Average is twelve to eighteen dollars per assessment. That has to be visible before it scales to a team.

## What it doesn't catch

SAGE is good at structured white-box assessment. It's not good at everything.

**Runtime behaviour.** Static analysis can flag that TLS validation is disabled in an HTTP client, but it can't confirm whether the production deployment actually enforces certificate pinning. That requires a live MITM test, which is outside SAGE's current scope.

**Authorization bypass through alternate entry points.** SAGE checks auth on the paths it can trace, but if there's an undocumented API route that skips the auth middleware entirely, static data-flow analysis across module boundaries is weak at catching that. These still need manual route mapping and testing.

**Absence of defence.** Scanners flag the presence of bad code. They're much worse at flagging the absence of good code. Missing certificate pinning, missing jailbreak detection, missing rate limiting. If it's not there, there's nothing to pattern-match against. I catch these through checklist-driven manual review against OWASP MASTG and WSTG requirements.

**AI and LLM-specific patterns.** Prompt injection through user content, unvalidated tool execution from model output, session data leaking into model context. These are new enough that traditional SAST tools don't have established rule sets for them yet. I'm building custom patterns as I encounter them, but this is still largely manual territory.

## What's next

SAGE works. But there's a clear list of things I'm actively working on to make it better.

**Decoupling from one CLI harness.** SAGE is bound to Claude Code right now. I'm working on making the assessment engine runnable from any agentic system, either through the Anthropic SDK directly or as an MCP server. Portability is the goal.

**Moving the knowledge base to a real datastore.** Markdown is easy to read and easy to grep. It's also hard to query. I'm evaluating SQLite and a small graph DB to make cross-assessment pattern detection real instead of grep-based.

**Schema-validating findings at write time.** Right now findings are markdown with frontmatter. I'm building a JSON Schema layer to catch missing CWE mappings, missing severities, or missing evidence before they hit triage.

The architecture is solid and the five commands work end to end. What I genuinely don't know yet is whether the closed loop delivers on its core premise: does pattern compounding from one engagement actually make SAGE meaningfully better on the next, or does it plateau after a few runs? That's not a question the architecture can answer. Part 2 will have the data.

## References

- [OWASP WSTG v4.2](https://owasp.org/www-project-web-security-testing-guide/v42/)
- [OWASP MASTG](https://mas.owasp.org/MASTG/)
- [OWASP Top 10](https://owasp.org/Top10/)
- [Claude Code documentation](https://docs.claude.com/en/docs/claude-code/overview) for the universal skills mechanism
- [FuzzingLabs MCP security hub](https://github.com/FuzzingLabs/mcp-security-hub) for the broader MCP security tooling ecosystem
