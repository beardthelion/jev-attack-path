# Jev Attack Path — MVP Product Spec (revised 2026-09-23)

Revision notes: this pass (1) states the deterministic layer's actual
contract instead of hand-waving candidate generation, (2) adds a
calibration and red-check discipline, (3) removes "confirmed" language
nothing is confirmed without exploitation, (4) cuts MVP to TypeScript
only, (5) names the jevsec engine relationship, (6) addresses the
reputation abuse vector on share cards.

## 0. Relationship to jevsec

**jevsec is the engine. This repo is the product.** The deterministic
codebase mapper, the Jev triage layer, and the sandboxed verification
harness live in `beardthelion/jevsec` and are imported here, not
reimplemented. This repo owns: repository ingestion from a URL, the
live investigation UI, the attack graph renderer, share cards, rate
limiting, and abuse prevention. If a capability exists in jevsec, this
repo calls it. Duplication is a bug.

## 1. Product Concept

**Jev Attack Path** is a web app that lets a user submit a public GitHub
repository and watch an AI-assisted security analysis identify and
prioritize potentially exploitable attack paths through the codebase.

The core differentiator is that the system doesn't simply report isolated
vulnerabilities. It attempts to answer:

> **"Can an attacker get from an externally controlled input to something security-sensitive?"**

Jev acts as the fast decision/prioritization layer that determines which
potential paths are worth investigating further.

### Core user experience

```text
Paste GitHub repo
       ↓
Repository analysis
       ↓
Identify entry points
       ↓
Generate candidate attack paths
       ↓
Jev evaluates candidates
       ↓
Prioritize promising paths
       ↓
Verify/analyze top paths
       ↓
Interactive attack graph
       ↓
Security findings
```

---

# 2. MVP Goal

The MVP should prove one thing:

> **Can we turn a large codebase into a small number of understandable, visually compelling potential attack paths?**

The MVP does **not** need to autonomously exploit targets.

It should instead produce useful hypotheses such as:

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

with explanations of:

- where attacker-controlled data enters
- how it propagates (to the extent the deterministic layer can show)
- where trust boundaries are crossed
- what sensitive operation it reaches
- why the path deserves investigation
- confidence/evidence supporting the path

---

# 3. Target User

Primary:

- security researchers
- developers
- AI coding-agent users
- bug bounty researchers
- open-source maintainers

Secondary:

- people curious about whether their code is secure
- developers wanting a "security review" of an AI-generated project

The product should be understandable to someone who isn't a professional
security researcher.

---

# 4. Homepage

Extremely simple.

## Hero

> **Can Jev find an attack path in your code?**

Subtext:

> Give Jev a GitHub repository and watch it trace potentially dangerous paths through the code.

Input:

```text
https://github.com/owner/repository

[ Analyze Repository ]
```

Optional:

**"Try a demo repository"**

No account required for the first analysis if feasible.

---

# 5. Repository Ingestion

For MVP:

- public GitHub repositories only
- **TypeScript / JavaScript only**
- clone repository into an isolated workspace
- determine framework (Express, Fastify, Next.js, etc.)
- build a basic repository map
- identify important files
- identify application entry points

Python, Go, and Java move to Phase 2. Source/sink identification is real
per-language work; one language proves the loop before the others
multiply the surface.

---

# 6. Codebase Understanding

Create a lightweight representation of the repository.

The analyzer should identify:

### Entry points

Examples:

- HTTP routes
- API handlers
- CLI arguments
- WebSocket handlers
- file uploads
- environment-controlled inputs
- message queue consumers
- webhook handlers

### Sensitive sinks

Examples:

- SQL queries
- shell execution
- filesystem operations
- template rendering
- deserialization
- authentication/authorization checks
- credential/token handling
- privileged API calls
- database writes
- dynamic code execution

### Trust boundaries

Examples:

```text
Internet
   ↓
API
   ↓
Application
   ↓
Database
```

or:

```text
User
 ↓
Frontend
 ↓
Backend
 ↓
Admin service
```

---

# 6b. Deterministic Layer Contract (read this before anything downstream)

This is the load-bearing section. Everything Jev ranks is only as good
as what this layer can actually prove.

**What the deterministic layer proves (v0.1):**

- Entry points: route handlers found via framework-specific detection
  (decorators, router calls, file conventions).
- Sinks: dangerous operations found via pattern matching on call
  expressions (`eval(`, `exec(`, `query(`, `innerHTML`, etc.).
- Reachability: a call-graph path exists from an entry point's handler
  to a function containing a sink. Built from imports and call
  expressions, no model calls.

**What it does NOT prove:**

- Actual data flow. A call-graph edge from handler to sink does not mean
  attacker input reaches the sink. There is no taint tracking in v0.1.
- Sanitization. The layer cannot tell whether input was validated
  between the entry and the sink.
- Exploitability. Nothing here claims a vulnerability exists.

**Consequence:** candidate paths are *reachability hypotheses*, not
data-flow findings. The Jev questions, the UI copy, and the share cards
must all reflect this. Taint tracking is a Phase 2 upgrade; when it
lands, the candidate quality bar rises and the Jev questions get
rewritten.

---

# 7. Candidate Attack Path Generation

The system constructs possible paths between:

**attacker-reachable entry point → sensitive operation**

using the call-graph reachability from section 6b. For example:

```text
POST /upload
      ↓
req.body.filename
      ↓
handleUpload()
      ↓
normalizeFilename()
      ↓
saveFile()
      ↓
fs.writeFile()
```

Each candidate contains:

```json
{
  "source": "...",
  "sink": "...",
  "nodes": [],
  "edges": [],
  "trust_boundaries": [],
  "reason": "...",
  "evidence": []
}
```

The goal isn't to claim that every candidate is exploitable.

They are **hypotheses**. Specifically: *this sink is reachable from this
entry point; the data flow between them is unverified.*

---

# 8. Jev Decision Layer

This is the core Jev integration.

Instead of sending every possible path to an expensive reasoning model,
ask Jev structured questions. The questions must stay inside what the
evidence supports (see 6b): Jev judges reachability plausibility, not
data flow it cannot see.

Examples:

### Investigation decision

> Given this candidate path and its evidence, how worthwhile is it to investigate further?

Output:

- probability worth investigating
- confidence
- decision

### Security sensitivity

> How security-sensitive is the sink reached by this path?

### Attacker reachability

> How likely is the source to be reachable by an unauthenticated attacker?

### Reachability plausibility

> How strongly does the available evidence support that this call-graph
> path is real (as opposed to a dead branch or framework boilerplate)?

### Escalation

> Does this path have enough evidence to justify deeper analysis?

The exact schema is determined experimentally during implementation.
Evidence serialization budget matters: each question's context is the
path plus code snippets, kept small enough to stay fast and cheap. If
evidence doesn't fit, the path is split or summarized deterministically
first, never silently truncated.

---

# 8b. Calibration and Red-Check Discipline

A security product that cries wolf is worse than no product. Before
launch, the pipeline must pass a seeded evaluation:

- **Fixture repo with known vulnerabilities:** a small TypeScript app
  containing N deliberately planted issues (e.g., an unsanitized
  `exec` on a route parameter, a SQL string-concat login, a path
  traversal in a file download). The pipeline must surface at least
  80% of them in its top-ranked paths.
- **Clean fixture repo:** a well-structured app with no planted issues.
  Fewer than 20% of its candidate paths may rank above the
  "interesting" threshold. Anything higher means the ranker is noisy.
- **Regression:** both fixtures re-run whenever the deterministic
  layer, a rubric, or a Jev question changes. A change that drops
  detection or raises noise does not ship.

No public launch until both bars pass. The fixture repos live in
jevsec (the engine), not here.

---

# 9. Investigation Pipeline

Candidate paths are ranked using Jev.

Example:

```text
47 candidate paths discovered

            JEV
             ↓

Path #12     94%  ██████████
Path #31     89%  █████████
Path #04     83%  ████████
Path #17     41%  ████
Path #28     12%  █
...
```

Only the highest-value candidates proceed to deeper investigation.

This is the primary mechanism for keeping inference costs under control.

---

# 10. Deep Verification

For the top candidates, run a more capable code-analysis pass.

This stage should:

1. inspect relevant files
2. follow function calls
3. inspect validation/sanitization
4. check authentication/authorization
5. verify reachability assumptions
6. look for mitigating controls
7. determine whether the path is actually plausible

The system distinguishes:

### Highly plausible

Strong evidence supports the complete path. Still unverified: needs a
human or a sandbox exploit to go further. **Nothing is called
"confirmed" without exploitation.**

### Likely

Most of the path is supported but some assumptions remain.

### Interesting

Potentially meaningful path requiring human review.

### Rejected

Investigation determined that the path is blocked or invalid.

---

# 11. Attack Graph UI

This should be the centerpiece of the product.

Instead of a conventional vulnerability report, show a visual graph.

Example:

```text
             INTERNET
                 │
                 ▼
        POST /api/import
                 │
                 ▼
          importFile()
                 │
                 ▼
          parseArchive()
                 │
                 ▼
         deserialize()
                 │
                 ▼
          processJob()
                 │
                 ▼
       privileged operation
```

Nodes should be clickable.

Clicking a node displays:

- file
- line number
- function
- relevant code
- explanation
- evidence

Edges should explain why the system believes the transition exists.
Where the evidence is call-graph reachability rather than verified data
flow, the edge label says so.

---

# 12. Live Investigation Mode

Make the analysis feel like an agent working.

Example:

> 🔎 Mapping repository...

> Found 23 potential application entry points.

> 🧭 Identifying sensitive operations...

> Found 41 potential sinks.

> 🧠 Generating candidate attack paths...

> Found 52 candidate paths.

> ⚡ Jev is prioritizing investigation targets...

> **Path #17 selected**

> 🔬 Investigating `POST /api/import → deserialize()`

> Checking validation...

> Checking authorization...

> Following `processJob()`...

> ⚠️ Potential security boundary violation discovered.

This is important for the viral/demo aspect.

The user should feel like they are **watching an investigation happen**,
not waiting for a scanner.

Honesty rule: the UI reflects real pipeline stage transitions, not
scripted theater. If a stage takes 3 seconds, it shows for 3 seconds.
Fake progress bars destroy trust the first time someone watches closely.

---

# 13. Result Page

At the top:

## 🔴 Potential Attack Path

**Confidence:** High
**Investigation:** 8.4/10
**Status:** Needs human verification

Then:

```text
Attacker
   ↓
POST /api/import
   ↓
parseImport()
   ↓
deserialize()
   ↓
processJob()
   ↓
Privileged operation
```

### Why this matters

Explain in plain English:

> An unauthenticated request appears capable of reaching code that
> eventually calls a privileged operation. The current analysis did not
> identify sufficient sanitization between these points. Data flow is
> unconfirmed.

### Evidence

Show exact source locations.

### Analysis

Explain the reasoning without exposing hidden chain-of-thought.

### Recommended next step

For example:

> Verify whether `deserialize()` accepts attacker-controlled object types.

---

# 14. Shareable Result

Every analysis should generate a public/shareable result card.

Example:

> **Jev analyzed `example/project`**
>
> 🔴 2 high-interest attack paths
> 🟠 4 paths requiring review
> 🟢 31 paths rejected
>
> **Most interesting path:**
> `POST /import → deserialize() → processJob()`
>
> **[View Attack Graph]**

This is the primary viral mechanism.

**Abuse guardrails (non-negotiable):**

- Every card carries a permanent, un-croppable watermark:
  **"UNVERIFIED HYPOTHESIS — needs human review."** No card may imply
  a vulnerability was found.
- Share links are created only when the user clicks share. The app
  never publishes findings about a repo unprompted, and there is no
  auto-generated public feed of "vulnerable repos."
- Do not expose private repository contents in public results (public
  repos only in the MVP anyway).

Someone *will* use this to make a project look bad. The watermark and
the "hypothesis, not finding" language are the defense. They must
survive cropping as much as technically feasible (baked into the image,
not an HTML overlay).

---

# 15. "Jev vs AI-Generated Code" Mode

After the basic product works, add a dedicated mode.

Landing-page copy:

> **Your AI wrote the code.
> Let Jev try to break it.**

Users submit a repository generated by:

- Claude
- Cursor
- Codex
- other coding agents

The result can show:

```text
AI-generated project

47 possible paths
      ↓
Jev investigated 8
      ↓
2 high-interest paths
```

This creates a very natural social-media challenge.

---

# 16. Cost-Control Strategy

This is critical.

Do **not** send the entire repository to an expensive reasoning model.

Use a funnel:

```text
                 Full repository
                       │
                       ▼
              deterministic analysis
                       │
                       ▼
               candidate generation
                       │
                       ▼
                    Jev
                       │
             ┌─────────┴─────────┐
             ▼                   ▼
        discard 80–95%       investigate
                                 │
                                 ▼
                         expensive reasoning
                                 │
                                 ▼
                           verification
```

The expensive model only sees:

- relevant files
- relevant functions
- relevant paths
- relevant evidence

rather than the entire repository.

---

# 17. Security Boundaries

The system itself must be designed as a security-sensitive application.

Repository analysis should happen in isolated environments.

Requirements:

- sandbox untrusted repositories
- don't execute repository code by default
- restrict network access
- resource/time limits
- filesystem isolation
- ephemeral analysis environments
- never expose analysis credentials to repository code
- prevent repository contents from accessing other users' analyses

For MVP, prefer **static analysis over code execution**.

---

# 18. Abuse Prevention

Because users can submit arbitrary repositories:

- rate-limit analyses
- cap repository size
- restrict analysis duration
- prevent internal/private network access
- don't allow arbitrary shell execution
- don't turn the service into a generic remote execution environment

The MVP should focus on **code analysis**, not active exploitation.

---

# 19. Metrics

Track:

### Product metrics

- repositories analyzed
- analyses completed
- analysis completion rate
- average analysis time
- users returning
- share-link clicks
- percentage of users running another repository

### Security-quality metrics

- candidate paths generated
- candidates selected by Jev
- false-positive rate (against fixture repos, section 8b)
- fixture detection rate (against fixture repos, section 8b)
- human-confirmed findings
- percentage of paths successfully verified

### Viral metrics

Most important:

**What percentage of users share their result?**

The product should optimize around making the result interesting enough
that users naturally want to show someone.

---

# 20. MVP Scope

### Build

- GitHub URL input (public repos, TypeScript only)
- repository cloning
- framework detection
- source/sink identification
- call-graph reachability candidate generation
- Jev prioritization
- limited deep investigation
- attack graph
- source-code evidence
- live investigation UI (honest stage transitions)
- shareable result (watermarked "UNVERIFIED HYPOTHESIS")

### Don't build initially

- private GitHub repositories
- languages beyond TypeScript
- automatic exploitation
- full bug bounty submission
- authenticated scanning
- arbitrary code execution
- enterprise dashboards
- complicated user accounts
- full vulnerability-management platform

---

# 21. Success Criteria

The MVP succeeds if a user can:

1. paste a GitHub URL
2. wait a few minutes
3. watch the investigation
4. see Jev prioritize paths
5. understand at least one interesting attack path
6. click through to the relevant source
7. share the result

The most important test:

> **Does the output make someone want to paste another repository?**

If yes, expand the autonomous investigation capabilities.

Launch gate (from 8b): fixture detection ≥ 80%, clean-fixture noise <
20%. No public launch until both pass.

---

# 22. Phase 2

Once the core loop works:

### Taint tracking

Upgrade the deterministic layer from call-graph reachability to real
data-flow/taint analysis. This is the single biggest quality lever:
candidates stop being reachability hypotheses and start being data-flow
findings. Rewrite the Jev questions when it lands.

### Additional languages

Python, Go, Java — one at a time, each with its own fixture repos and
calibration pass.

### Autonomous research

Allow the deeper agent to:

- generate exploit hypotheses
- test assumptions in a sandbox
- search for bypasses
- inspect related code
- recursively investigate promising paths

### Agent battle

```text
              AI-generated app
                     │
              ┌──────┴──────┐
              │             │
           Defender        Jev
              │             │
          patches       discovers
              │             │
              └──────┬──────┘
                     ↓
                 scorecard
```

### Public challenge mode

Let users submit intentionally vulnerable applications and let Jev
attempt to find the bugs.

This turns the product into a security game rather than simply a scanner.

---

# 23. Product Positioning

Don't market it as:

> "An AI vulnerability scanner."

That's crowded.

Position it around the concept:

> **"Watch an AI trace an attack through your code."**

Or:

> **"Give Jev a codebase. See where an attacker could go."**

The **attack path** is the product.

Jev is the intelligence layer that decides which paths deserve attention.
