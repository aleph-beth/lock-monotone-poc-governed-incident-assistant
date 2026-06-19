# Lock-Monotone PoC — Technical Documentation

> 🇫🇷 Une version française de ce document est disponible : [README_FR.md](README_FR.md)

## Short definition
Lock-Monotone is a security architecture for systems that integrate LLMs. It separates the *understanding* of a request (ACTION/KNOWLEDGE translation) from the *authorization* of access and execution, which remain deterministic and controlled by code. Capabilities can never grow as the request moves through the pipeline.

## Security problem solved
Lock-Monotone reduces the following risks without relying on the model's internal behavior:

- **Prompt injection**: decisions are made on a structured translation, not on free text.
- **Confused deputy**: access is decided by a policy engine external to the LLM.
- **Escalation through reasoning**: capabilities stay monotone and cannot widen.

## ACTION / KNOWLEDGE principle
A user request is translated into a structured artifact made of two sections:

- **ACTION**: the requested effect (procedure, target, output type).
- **KNOWLEDGE**: the sources or knowledge categories required.

The rest of the pipeline operates only on this translation.

## Pipeline overview
A strictly unidirectional pipeline:

1. User prompt
2. Upstream LLM → ACTION/KNOWLEDGE translation
3. Deterministic validation & canonicalization
4. Policy Engine (closed decision)
5. Controlled knowledge access (bounded connectors)
6. Downstream LLM (constrained generation)
7. Audit (full traceability)

## What Lock-Monotone is / is not
**Lock-Monotone is:**

- A deterministic security architecture for LLMs.
- An authorization mechanism based on a structured translation.
- A pipeline with monotone capabilities.

**Lock-Monotone is not:**

- An LLM model made "secure" by magic.
- A system that trusts the model's internal reasoning.
- A mechanism for automatic privilege elevation.

## Minimal conceptual example
**Prompt**: "Give me the onboarding procedure for the Finance service account."

**Translation (IR)**

- ACTION: `procedure.read`
- KNOWLEDGE: `policies/hr/onboarding` (authorized category)

**Decision**

- Policy Engine: `ALLOW` for role `hr-analyst`
- Capability envelope reduced to `policies/hr/*`

**Response**

- The downstream LLM generates a summary only from authorized sources.

## PoC scope
The PoC covers:

- The definition of the ACTION/KNOWLEDGE IR.
- Validation, canonicalization, and the policy decision.
- Bounded knowledge connectors.
- End-to-end audit.

The PoC does not cover:

- Training or modifying an LLM.
- Executing sensitive actions in production.

## Repository structure
- [`Lock-Monotone-EN.md`](Lock-Monotone-EN.md) — Consolidated research article (English).
- [`Lock-Monotone-FR.md`](Lock-Monotone-FR.md) — Consolidated research article (French).
- [`Lock-Monotone.tex`](Lock-Monotone.tex) — LaTeX source of the article.
- [`docs/`](docs) — Detailed technical documentation (context, use case, security goals, intermediate representation, architecture, policy engine, controlled RAG, auditability, attack scenarios, readiness checklist).
- [`LICENSE`](LICENSE) — Project license.
