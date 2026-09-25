# Jev Attack Path

A web app that lets you submit a public GitHub repository and watch an AI-assisted security analysis trace potential attack paths through the codebase, rendered as an interactive attack graph.

**jevsec is the engine. This repo is the product.** The deterministic codebase mapper, the Jev triage layer, and the sandboxed verification harness live in `beardthelion/jevsec` and are imported here, not reimplemented. This repo owns: repository ingestion from a URL, the live investigation UI, the attack graph renderer, share cards, rate limiting, and abuse prevention.

Jev is a decision-only AI model that answers narrow judgment questions with calibrated confidence. Here it is the fast prioritization layer that decides which candidate attack paths are worth investigating further.

> Status: spec v0.1 (revised 2026-09-23). The design is done; the app is not yet built.

## The question it answers

Not "which known vulnerability patterns appear in this code," but:

> **Can an attacker get from an externally controlled input to something security-sensitive?**

The MVP proves one thing: that a large codebase can be turned into a small number of understandable, visually compelling potential attack paths, like:

```text
HTTP Request
    ↓
req.body.file
    ↓
parseUpload()
    ↓
deserialize()
    ↓
processJob()
    ↓
shell command
```

Each path explains where attacker-controlled data enters, how it propagates, where trust boundaries are crossed, what sensitive operation it reaches, and why it deserves investigation, with confidence and evidence attached.

## Who it is for

Security researchers, developers, AI coding-agent users, bug bounty researchers, and open-source maintainers. It is built to be understandable to someone who is not a professional security researcher, including developers wanting a security review of an AI-generated project.

## Trust and abuse

- Findings are **potential** paths with evidence, never "confirmed" without exploitation.
- Share cards are designed against the reputation abuse vector: framing keeps results honest about confidence levels.
- Public repos only in v0.1.

## Spec

The full MVP spec is [jev-attack-path-spec.md](jev-attack-path-spec.md): product concept, pipeline, UI, calibration and red-check discipline, the jevsec engine contract, and abuse prevention.
