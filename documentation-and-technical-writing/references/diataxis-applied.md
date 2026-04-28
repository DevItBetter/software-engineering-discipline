# Diátaxis, Applied

Daniele Procida's Diátaxis framework distinguishes four kinds of documentation. Each serves a different reader need; mixing two of them in one doc produces a worse version of both.

The four:

| Type | Reader's mode | Goal | Form |
|---|---|---|---|
| **Tutorial** | Studying / learning | Produce a working result while learning | Hands-on, narrative, prescriptive |
| **How-to guide** | Doing | Solve a specific problem | Steps, focused, assumes baseline |
| **Reference** | Looking up | Answer a factual question accurately | Comprehensive, austere, factual |
| **Explanation** | Understanding | Build a mental model | Discursive, framing, considers alternatives |

This file is a working application of the framework: how to recognize which mode a doc should be in, how to convert between modes, and what the failure modes look like.

## Recognizing the mode

When reading or writing a doc, ask: **what does the reader want from this?**

- "Teach me from scratch" → tutorial.
- "I have a specific goal; show me the steps" → how-to.
- "I need to look up a fact" → reference.
- "Help me understand" → explanation.

Note these are *reader-oriented*, not topic-oriented. The same topic can produce four different documents:

- Tutorial: "Build your first authenticated API request."
- How-to: "How to authenticate against the API."
- Reference: "Authentication API: endpoints, parameters, error codes."
- Explanation: "Why we use OAuth and not session tokens."

A docs site needs all four. A single doc trying to be all four is a slog.

## What each mode looks like, concretely

### Tutorial

The reader doesn't know the domain yet. They're following along to learn something work; the working result is a side effect.

Style:
- **Hand-hold.** Explicit steps, expected output at each step.
- **Assume nothing.** "Open a terminal. If you don't have a terminal, [link]."
- **Accept some lack of generality.** "We'll use the simple version; for production, see [link]."
- **Build to a working result.** The reader should produce something concrete by the end.
- **Encourage.** "Great — your first request worked." Tutorials are pedagogy; pedagogy includes morale.

Bad tutorial: branches everywhere ("if you have X, do Y; if you have Z, do W") that overwhelm a beginner; jumps to advanced topics; tells you "what" but not "how" with a specific command.

Example anti-pattern: tutorial that says "now configure your environment" without saying *which* configuration mechanism to use. The reader gets lost.

### How-to guide

The reader has a goal. They're not learning the domain; they need to get this thing done.

Style:
- **Direct.** Steps. Imperative voice. "Run X. Verify Y."
- **Assume baseline knowledge.** "Configure CORS" — assume the reader knows what CORS is.
- **Solve one specific problem.** "How to enable CORS for a development origin." Not "how to do everything related to CORS."
- **Branch on real choices.** "If using framework A, do X; framework B, do Y." But only when the branches matter.

Bad how-to: padded with explanation ("CORS, which stands for Cross-Origin Resource Sharing, is a mechanism that..."); too general ("how to do CORS" with 30 sub-cases); too narrow ("how to enable CORS for a specific endpoint in this specific version of this specific framework on this specific OS" — too pinned-down).

### Reference

The reader is looking something up. They came from a search; they want a fact.

Style:
- **Comprehensive.** Every field, every parameter, every option, every error code.
- **Austere.** No story. No "we hope you find this useful." Just facts.
- **Structured.** Tables, lists, consistent layout. Makes scanning easy.
- **Accurate.** Reference docs that lie are worse than no docs. Often auto-generated to stay accurate.

Bad reference: prose paragraphs where a table would do; missing fields ("for the rest, see source"); examples mixed in with the structural definition.

### Explanation

The reader has the facts; they want to understand. Why does it work this way? What were the alternatives? What's the model?

Style:
- **Discursive.** Paragraphs, not bullet lists.
- **Considers alternatives.** "We could have done X; we did Y because Z."
- **Frames the topic.** "This is part of a broader concept of W."
- **No prescription.** Explanation isn't telling you what to do; it's helping you understand.

Bad explanation: reads as "advanced documentation" — overwhelming the new reader and not deep enough for the old; ends up as a third-rate tutorial-reference hybrid.

## How to organize a docs site

Diátaxis suggests four top-level sections, one per mode:

```
docs/
├── tutorials/
│   ├── first-api-request.md
│   ├── building-your-first-app.md
│   └── ...
├── how-to/
│   ├── authenticate.md
│   ├── enable-cors.md
│   ├── deploy-to-production.md
│   └── ...
├── reference/
│   ├── api/
│   ├── cli.md
│   ├── configuration.md
│   └── ...
└── explanation/
    ├── why-oauth.md
    ├── architecture.md
    ├── data-model.md
    └── ...
```

Each top-level directory has a clear shape; the reader's intent maps cleanly.

For smaller projects, this is overkill — a handful of docs can live flat. For mature projects, the structure helps both writers (knowing what to write) and readers (knowing where to look).

## Migrating existing docs

Most docs in the wild are mixed-mode. To improve them:

1. **Identify what mode each doc is currently trying to be.** Often multiple.
2. **Identify what mode it *should* be**, given the reader's most likely intent.
3. **Split mixed-mode docs.** A "Getting Started" that's part tutorial, part how-to, part reference becomes three docs that link to each other.
4. **Migrate gradually.** Don't reorganize everything at once. New docs follow the structure; old docs migrate with the next significant change.

A common starting move: separate the **reference** out of everything else. Reference docs are the most reliably ruined by mixing — a how-to with reference embedded becomes a slog; a tutorial with reference embedded becomes intimidating. Pulling reference out into its own clean location often makes the rest of the docs better immediately.

## Worked example: a CLI tool

Imagine you're documenting a CLI tool called `mycli`.

### Tutorial: "Build your first thing with mycli"

Linear walkthrough. Install. Run a command. See output. Run another. Build to a working result. Maybe 1500 words. The reader who finishes has used the tool successfully and has a working mental model.

Anti-pattern to avoid: this tutorial trying to also document every flag of every command. That's reference; it overwhelms the tutorial.

### How-to guides

Each is task-focused:
- "How to authenticate with a custom server"
- "How to migrate from version 1.x to 2.x"
- "How to use mycli in CI"
- "How to debug a failing run"

Each is short, focused, assumes baseline knowledge. Doesn't try to teach the tool from scratch; doesn't try to be comprehensive.

### Reference

- `mycli` — usage, all subcommands listed, link to each subcommand's reference.
- `mycli auth` — usage, every flag, every error code, exit codes.
- `mycli run` — same.
- Configuration file format — every key, every value type, every default.
- Environment variables.

Auto-generated where possible; verified to match implementation.

### Explanation

- "Why mycli is a single binary" (vs language packages)
- "How mycli's plugin model works" (architecture)
- "Choosing between mycli and alternatives" (trade-offs)

Discursive, considers alternatives, builds mental models.

The reader navigating this docs site finds what they need without having to read everything. The writer adding a new how-to doesn't have to update three other docs that also touch the topic; the structure absorbs the change cleanly.

## Failure modes when applying Diátaxis

**Treating it as bureaucracy.** "We need a tutorial" → write a checkbox doc that's not actually a tutorial. The framework is a tool, not a checklist; if a kind of doc isn't useful for your readers, don't write it.

**Forcing every doc into a category.** Some docs are usefully cross-mode (a reference doc with embedded examples is usually fine; an FAQ that's part how-to / part explanation is fine). The framework is a guide for the *primary* mode; secondary content is OK if it doesn't dominate.

**Reorganizing too aggressively.** Migrating an existing docs site to Diátaxis all at once is a huge undertaking. Do it gradually; new docs follow the structure; old docs migrate as they're updated.

**Ignoring discoverability.** Diátaxis organizes by reader intent; readers don't always know their intent precisely. Cross-link generously. Add a search. The structure helps but doesn't replace discoverability.

## Source

The canonical reference is **diataxis.fr** by Daniele Procida. The "five-minute" intro and the longer essays are excellent.

For implementations: Django, Numpy, Cloudflare, Gatsby, GitLab, Canonical (Ubuntu), and many other open-source and commercial docs sites have adopted some version of Diátaxis. Looking at how mature docs sites apply it is a useful study.
