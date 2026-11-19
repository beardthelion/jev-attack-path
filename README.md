# Jev Attack Path

Design for a web app that would let you submit a public GitHub repository and watch an AI-assisted security analysis trace potential attack paths through the codebase, rendered as an interactive attack graph.

**Status: spec only. There is no app code in this repo yet.** It holds the MVP product spec (v0.1, revised 2026-09-23); there is nothing to install or run.

**jevsec is the engine. This repo is the product.** The deterministic codebase mapper, the Jev triage layer, and the sandboxed verification harness are planned to come from jevsec, a separate engine that is not public yet, and be imported here, not reimplemented. This repo owns: repository ingestion from a URL, the live investigation UI, the attack graph renderer, share cards, rate limiting, and abuse prevention.

Jev is a decision-only AI model that answers narrow judgment questions with calibrated confidence. Here it is the fast prioritization layer that decides which candidate attack paths are worth investigating further.

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
- Public repos only in v0.1, and the MVP analyzes TypeScript and JavaScript only.

## Spec

The full MVP spec is [jev-attack-path-spec.md](jev-attack-path-spec.md): product concept, pipeline, UI, calibration and red-check discipline, the jevsec engine contract, and abuse prevention.

## Related tools

- [autocatalytic-harness](https://github.com/beardthelion/autocatalytic-harness): the staged audit harness for coding agents, which can run the jevsec engine as an optional tool
- [release-surface-diff](https://github.com/beardthelion/release-surface-diff) and [egress-plane](https://github.com/beardthelion/egress-plane): entry-point and outbound-call inventories

## License

MIT. See [LICENSE](LICENSE).
