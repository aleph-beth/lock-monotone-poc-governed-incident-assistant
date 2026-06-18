# Lock-Monotone: A Translation-Governed Security Architecture for LLM-Integrated Systems

**Grégory Lagasse** — Independent Researcher — `gregory.lagasse@pm.me`

*Synthesis version, June 2026. This document consolidates three working drafts of the
Lock-Monotone framework (a framework-oriented version, an ICSA-style version with a
cross-layer evaluation methodology, and an ARES-style version centered on the
translation pipeline and auditability). It is deliberately scoped as an architectural
framework: quantitative claims are stated as a proposed evaluation protocol and deferred
to future work, not asserted as results.*

---

## Abstract

Large Language Models (LLMs) are increasingly integrated into systems that manipulate
sensitive data, invoke operational tools, and execute business workflows. Current safety
approaches are predominantly *model-centric*, relying on fine-tuning, reinforcement
learning, or heuristic guardrails. However, a probabilistic language model cannot
constitute a deterministic security boundary.

This paper introduces **Lock-Monotone**, a deterministic governance architecture for
secure LLM integration. Rather than relying on behavioral alignment alone, Lock-Monotone
relocates authorization and capability control *outside* the model. The architecture
rests on three principles: (1) an ontological separation between **knowledge** and
**procedure**; (2) a typed intermediate representation (**PyIR**) that transforms natural
language into a bounded capability space; and (3) a deterministic decision core that
enforces both **vertical** and **positional** monotonicity of privileges. A *prefix
action priority invariant* ensures that no subsequent interaction step can expand the
capability scope fixed at initialization. Execution is mediated through a three-stage
**Triple Decode** (T1/T2/T3) that separates semantic rendering, operational tool
invocation, and compliance filtering.

We formalize a set of security invariants that prevent capability escalation, and we
prove a non-escalation guarantee *conditional* on the soundness of translation. We
position the approach with respect to recent design-by-construction defenses (the
dual-LLM pattern and CaMeL), and we illustrate the architecture through a worked example:
governing the analytic LLM embedded in a Security Information and Event Management (SIEM)
platform, including the apex case in which that supervisory LLM is itself the target of
attack. Finally, we sketch a cross-layer evaluation methodology — Unauthorized Action
Rate (UAR), Secret Leakage Rate (SLR), Decision Trace Coverage (DTC) — as the central
direction for future work.

**Keywords:** LLM security, prompt injection, capability-based security, deterministic
governance, reference monitor, agentic systems, auditability.

---

## 1. Introduction

### 1.1 Context

Large Language Models are increasingly embedded in operational systems that manipulate
sensitive secrets (API keys, credentials, proprietary information), regulated or personal
data (PII, financial records, healthcare data), and action capabilities (APIs, internal
services, databases, workflow engines).

In many current deployments, security strategies remain *model-centric*. Mitigations
focus primarily on improving the internal behavior of the model through fine-tuning,
reinforcement learning from human feedback (RLHF), safety classifiers, or heuristic
guardrails. However, an LLM remains a probabilistic and context-sensitive component. Its
outputs are influenced by token distributions, prompt structure, and recency effects. As
such, a probabilistic language model **cannot constitute a deterministic security
boundary**. The core issue is therefore architectural rather than behavioral.

### 1.2 Thesis

We argue that security in LLM-based systems must not depend on the internal behavior of
the model. Instead, it must rely on a deterministic architecture that governs,
constrains, and enforces model capabilities externally. In this view, the model may
interpret, translate, or generate language, but it must **never** decide authorization,
grant privileges, or execute sensitive actions. Security is achieved not by making the
model trustworthy, but by *subordinating it to a deterministic capability governance
layer*. This architectural shift forms the foundation of the Lock-Monotone approach.

### 1.3 LLMs as Critical Interfaces

Early uses of LLMs focused on open-ended text generation, where failures were largely
confined to incorrect or undesirable outputs. In contrast, modern LLM-based applications
increasingly position models as *control surfaces* for executing actions, querying
sensitive data, or coordinating multi-step procedures. In this setting, language
generation effectively becomes **capability mediation**: natural-language inputs are
translated into operations with real-world consequences.

This shift is particularly salient in adversarial and high-assurance contexts —
cybersecurity operations, critical-infrastructure management, regulated enterprise
systems. In such environments, failures are no longer limited to semantic errors; they
may directly result in unauthorized access, privilege escalation, or irreversible system
state changes. The question is therefore no longer whether LLM outputs are well-behaved,
but whether they can be *safely embedded* in systems that require enforceable security
guarantees.

### 1.4 Why Existing Defenses Fail Structurally

Most current defenses for LLM safety remain structurally inadequate when models are
deployed as capability mediators. A dominant class relies on **terminal classification**,
where a user request is evaluated as a whole and assigned a binary or near-binary verdict
(allow/deny). Such approaches are rigid, difficult to audit, and poorly suited to
multi-step interactions.

Guardrail frameworks and post-hoc filtering operate on probabilistic outputs and
therefore lack the authority to enforce strict security boundaries. Similarly,
retrieval-augmented generation (RAG) pipelines often treat retrieval as an implicitly
permitted operation, focusing on content relevance rather than explicit authorization. As
a result, malicious or untrusted documents may influence model behavior without passing
through a deterministic access-control decision.

Across these approaches, a common structural limitation emerges: **the absence of a
single, deterministic decision boundary**. Security-relevant choices are implicit,
probabilistic, or distributed across components that cannot provide strong guarantees
under adversarial pressure. This architectural ambiguity enables prompt injection,
indirect instruction following, and reasoning-based privilege escalation.

### 1.5 Strategic Context: Machine-Speed Conflict and the Defender's Inherited Flaw

For three decades, cyberdefense was calibrated on an implicit assumption: attacks unfold
slowly enough for a human analyst to observe, qualify, and decide. The Security Operations
Center (SOC) escalation cycle — alert, triage, escalation, authorization — takes hours, at
times days. Agentic-AI campaigns compress into minutes what previously required days of
specialist work: reconnaissance, exploit generation, lateral movement, and credential
harvesting now chain at a tempo that outpaces any human-in-the-loop response. When the
attacker acts faster than the defender, the attacker controls the tempo of the engagement —
and therefore the engagement. The structural consequence is that neither offensive (red)
nor defensive (blue) human teams will operate *inside the loop*: humans remain upstream
(doctrine, authorization) and downstream (review, attribution, heavy remediation), while
the core engagement is fought between autonomous agents — an "AI vs AI" battlefield with no
human in the loop [21], [22].

This shift is no longer prospective. In November 2025, Anthropic disclosed what it
describes as the first large-scale cyber-espionage operation largely orchestrated by an
agentic AI: an actor assessed with high confidence as a Chinese state group (designated
GTG-1002) jailbroke an agentic coding tool via role-play — posing as a legitimate security
firm running a defensive test — then connected it to real tools through the Model Context
Protocol. The AI executed an estimated 80–90% of the operation against roughly thirty
targets, with a small number of successful intrusions; notably, its principal limitation
was model *hallucination* rather than any defense [16]. The trend is broader: VILLAGER, an
AI-native pentest tool built on a public model and bundling 4,000+ exploitation prompts;
the HexStrike actor exploiting CVE-2025-7775 across 8,000+ endpoints in under ten minutes;
ransomware chains compressed from initial compromise to exfiltration in roughly 25 minutes;
and analyses warning that AI-driven attacks already outpace human-driven defenses on
critical infrastructure [20], [21].

The asymmetry is structural, not a recoverable lag. The natural response — automate the
defense by delegating triage, correlation, even containment to defensive agents — closes a
trap, because those defensive agents inherit the very flaw the attacker exploits. In
October 2025, researchers from leading laboratories tested twelve published prompt-injection
defenses, most claiming near-zero attack success; the team broke **over 90%** of them [17].
Large-scale public competitions on indirect injection reach the same verdict: agents remain
massively permeable [19]. Automating defense with LLMs therefore reproduces the attacker's
flaw on the defender's side: an adversary need no longer target the network directly — it
can target the *supervisory agent*, which sees everything. The attack surface migrates to
the defender.

The root is architectural, not a bug to be patched by more training. An LLM has no
intrinsic boundary between *instruction* and *data*: the system prompt and the user input
arrive in the same form — natural-language strings — and cannot be distinguished by type.
Any separation is merely *semantic*: fuzzy, contextual, and therefore exploitable.
Attention analysis confirms it: under a successful injection, the attention of certain heads
shifts from the original instruction to the injected one [18]. An LLM excels at
*translating* and *classifying* but is structurally unable to maintain *exact state over a
sequence* — an expressivity limit that opens **accumulation attacks**, where each turn looks
benign and the violation emerges from composition, beyond the reach of a turn-by-turn check.
A direct corollary: one LLM cannot reliably control another, since the controller shares the
controlled model's incapacity. Stacking defensive AI adds vulnerable surfaces, not
guarantees.

This is precisely the motivation for Lock-Monotone. If the model cannot be trusted to guard
itself, the security boundary must be relocated *outside* the model: confine the LLM to its
strengths (translating intent, classifying input) and delegate the *decision* to a
deterministic, verifiable layer placed off the network execution path. The prefix
monotonicity developed in this paper halts accumulation attacks at a bound that no sequence
of turns can push back — because that bound is enforced by a function, not evaluated by a
model. The objective is not a "safer" LLM (likely out of reach) but to ensure that a
compromised model confers *no authority*: the model proposes, the deterministic layer
disposes.

### 1.6 Positioning and Contributions

Lock-Monotone positions LLMs as explicitly *untrusted*, probabilistic components whose
role is limited to semantic translation. All security-relevant decisions are delegated to
deterministic mechanisms that are auditable, traceable, and independent of model behavior.

This stance is shared with a recent line of *design-by-construction* defenses — most
notably the **dual-LLM pattern** and **CaMeL** — which also remove authority from the
model. Section 11 details the relationship. In summary, this paper contributes:

- an **ontological** separation between knowledge and procedure, enforced at the
  translation boundary (Section 4);
- a **prefix action priority invariant** (positional monotonicity) that caps capabilities
  for an entire interaction at its first declaration (Section 5);
- a typed intermediate representation (**PyIR**) and a **deterministic decision core**
  (Sections 6–7);
- a three-stage **Triple Decode** (T1/T2/T3) execution mediation with an independent
  compliance gate (Section 8);
- a qualitative **worked example** governing a SIEM's analytic LLM, including the
  meta-level case where that LLM is itself attacked (Section 11);
- a formalization of **non-escalation guarantees**, stated explicitly as *conditional* on
  translation soundness (Section 9);
- **auditability by construction** as a first-class property (Section 13);
- a **proposed cross-layer evaluation methodology** (UAR/SLR/DTC) deferred to future work
  (Section 15).

We stress that the present paper is an architectural framework. Quantitative evaluation is
deferred; no empirical results are claimed here.

---

## 2. Threat Model and Assumptions

The objective is not to enumerate all possible attacks exhaustively, but to characterize
the *structural* risks that arise when LLMs are embedded as interfaces to security- or
mission-critical systems.

### 2.1 Adversary Model

We consider adversaries interacting through natural-language inputs, retrieved data
sources, or integration interfaces. The adversary may be external or internal,
authenticated or unauthenticated, and may possess partial knowledge of the system
architecture. In scope:

- **Prompt injection.** Direct injection (malicious instructions in user input) and
  indirect injection (instructions introduced via external data such as retrieved
  documents, emails, or web content).
- **Reasoning-based escalation.** Exploiting the reasoning abilities of LLMs to induce
  unintended behavior through multi-step argumentation, role confusion, or contextual
  framing, aiming to expand effective permissions.
- **Procedure abuse and compositional attacks.** Sequences of individually permissible
  operations that, when composed, result in unauthorized effects — particularly relevant
  in agentic or tool-augmented deployments.

The adversary is **not** assumed to have white-box access to model weights or internal
states. Side-channel attacks, cryptographic failures, and low-level infrastructure
exploits are out of scope.

### 2.2 System Assumptions

- **Black-box models.** All LLMs are treated as black-box, probabilistic components. No
  assumptions are made about alignment, calibration, internal representations, or
  resistance to adversarial prompting. Outputs are considered potentially unreliable with
  respect to confidentiality and authorization.
- **Adversarial retrieval sources.** RAG-accessed content is assumed potentially
  adversarial: never authoritative, possibly carrying malicious instructions or crafted
  payloads.
- **Infrastructure scope.** Compute, storage, networking, and cryptography follow standard
  best practices. Attacks against hosting infrastructure, denial-of-service, and
  hardware-level exploits are out of scope and must be addressed by complementary
  defenses.

### 2.3 Security Objectives

- **Confidentiality.** Sensitive data and secrets must not be disclosed — directly or
  indirectly — to unauthorized parties (including conversational exfiltration and
  inference-based leakage).
- **Authorization integrity.** Permitted actions must not be expanded through prompt
  manipulation, contextual accumulation, or multi-step reasoning. Authorization decisions
  must be explicit, deterministic, and invariant to model behavior.
- **Auditability and accountability.** All security-relevant decisions must be explainable
  and traceable after the fact, with structured artifacts supporting post-incident
  analysis and compliance review.

---

## 3. Translation over Classification

Most existing approaches conceptualize LLM safety as a *classification* problem: user
requests are evaluated in raw, unstructured form and assigned a terminal verdict.

### 3.1 Limitations of Classifier-Centric Security

Classifier-only approaches tightly couple security decisions to free-form natural
language. Small linguistic variations, contextual reframing, or adversarial paraphrasing
can significantly alter outcomes. Classification is a *terminal* decision: security is a
single gate rather than a process, with no structured representation of intent, requested
actions, or data sensitivity. This opacity limits auditability and complicates adaptation
to evolving threats. In adversarial settings, these limitations become structural
vulnerabilities: probabilistic outputs are implicitly trusted as security-relevant
signals.

### 3.2 Translation as a Security Primitive

Lock-Monotone treats **translation — not classification — as the primary security
primitive**. Rather than issuing a verdict on raw text, the system first translates input
into explicit, structured intermediate representations that make intent, actions, and
constraints observable and governable. This decouples semantic interpretation from
authorization. Classification, when used, operates on structured representations rather
than free text.

Crucially, translation enables **non-binary** security behaviors: instead of strict
allow/deny, the system may apply controlled transformations such as abstraction,
reformulation, redaction, or partial fulfillment. A request that would be rejected
outright can be degraded into a safe, informational response that preserves legitimate
intent while eliminating operational risk.

### 3.3 Progressive Constraint through Governed Translations

Each translation stage reduces ambiguity, constrains the space of possible actions, and
exposes structured artifacts for deterministic evaluation. By organizing security as a
sequence of governed translations, Lock-Monotone distributes responsibility across
architectural stages rather than concentrating it in a single probabilistic decision
point — aligning with compiler pipelines and policy-governed execution environments, where
safety emerges from staged validation.

---

## 4. Initial Separation: Knowledge vs Procedure

### 4.1 Ontological Distinction

A foundational principle is the explicit separation between **knowledge** and
**procedure**:

- **Knowledge** — non-executable information: descriptions, summaries, classifications,
  explanations, or transformations that do not directly modify system state.
- **Procedure** — capability-bearing intent: any request that implies state modification,
  data access, secret retrieval, or operational execution.

This is not merely semantic; it is an *ontological* boundary between informational content
and actionable authority. In conventional deployments this boundary is blurred: generated
text may implicitly encode instructions later interpreted as commands, particularly in
agentic systems where outputs map to tool calls. Lock-Monotone formalizes the separation
as an architectural invariant. **Knowledge never implies action.**

### 4.2 Risk Asymmetry

The separation addresses two distinct primary risk classes, summarized below.

| | **Knowledge** | **Procedure** |
|---|---|---|
| Nature | Informational | Operational |
| Risk | Exfiltration | Tool abuse |
| Control | Output validation / filtering | Policy enforcement |
| Proposed metric | SLR | UAR |

These risk classes require fundamentally different enforcement strategies.

### 4.3 Translation as Boundary Enforcement

The separation is enforced at the earliest stage, through a structured IR. Natural
language is translated into a typed IR containing, at minimum: an `intent`, a
`semantic_class` (`KNOWLEDGE`, `PROCEDURE`, or `SENSITIVE`), a `sensitivity_level`, and a
`requires_tool` flag. This initial classification defines the capability boundary before
any execution path is considered. Critically, the separation must occur **prior** to any
action resolution, so that procedural intent cannot be implicitly derived from later
contextual interpretation or model reasoning.

### 4.4 Eliminating Instruction–Data Ambiguity

The architecture enforces the rule: **generated language output is never executable and
never authoritative.** Access to tools, databases, secrets, or external systems is never
triggered by text generation alone; all actions are mediated through explicit action
interfaces evaluated by deterministic policy engines. Consequently, conversational
manipulation cannot cause side effects beyond the knowledge regime, and the system *fails
closed*: attacks need not be detected or classified as malicious in order to be blocked.

---

## 5. Prefix Action Priority Invariant (Positional Monotonicity)

### 5.1 Motivation

LLMs are inherently sensitive to token ordering and contextual recency. Instructions
appearing later in a prompt can override, reinterpret, or semantically distort earlier
content — enabling late-stage prompt injection, multi-turn escalation, instruction
smuggling, and role/authority redefinition. In conventional architectures, capability
boundaries may implicitly shift as context accumulates. Lock-Monotone introduces a
**positional** security invariant.

### 5.2 Definition of the Prefix Invariant

Let an interaction be translated into an ordered intermediate representation

$$ IR = \{ s_1, s_2, \dots, s_n \}, $$

where $s_1$ is the initial semantic declaration (header), containing the intent, semantic
class, sensitivity level, and declared tool requirements. Let
$\mathit{Capabilities}(s_i)$ denote the capability set implied by step $s_i$. The Prefix
Action Priority Invariant is:

$$ \forall i > 1, \quad \mathit{Capabilities}(s_i) \subseteq \mathit{Capabilities}(s_1). $$

No subsequent step may introduce a capability not already permitted by the initial
declaration.

### 5.3 Two Forms of Monotonicity

- **Vertical monotonicity** — capabilities may only remain constant or decrease across
  processing layers.
- **Positional monotonicity** — capabilities may only remain constant or decrease across
  interaction steps.

Together, these constraints prevent implicit capability expansion through contextual
reinterpretation, paraphrased escalation, or deferred authorization requests.

### 5.4 Security Implications

The invariant blocks several attack classes: a multi-turn escalation attempt cannot
expand privilege beyond the initial declaration; a RAG-injected instruction cannot elevate
operational scope; a late-stage tool invocation request cannot exceed prefix-defined
permissions. Importantly, the invariant applies *before* policy evaluation: the initial
semantic classification defines the upper bound of permissible authority.

### 5.5 Theoretical Significance

The Prefix Action Priority Invariant introduces a **temporal** dimension to
least-privilege enforcement. Traditional least privilege constrains *who* may perform
*what*. Lock-Monotone additionally constrains *when* capability boundaries are defined.
Security is thus anchored not only in role and policy, but in the structural ordering of
translation. To our knowledge, this explicit prefix-priority temporal invariant
distinguishes Lock-Monotone from prior design-by-construction defenses.

---

## 6. Secure Intermediate Representation (PyIR)

### 6.1 Rationale

Natural language is ambiguous, context-dependent, and susceptible to adversarial
manipulation; binding execution or authorization semantics directly to raw model output
creates systemic risk. To eliminate this ambiguity, Lock-Monotone introduces a typed,
constrained, statically validated pseudo-language: **PyIR**.

> **A note on naming.** PyIR adopts a Python-inspired, human-readable syntax for
> legibility and tooling convenience. It is *not* executable Python: it is a restricted
> domain-specific language (DSL) with no general computation, designed for semantic
> normalization, capability declaration, deterministic parsing, and formal policy
> evaluation.

### 6.2 Core Structure

A minimal PyIR instance contains a structured header followed by optional procedural
elements:

```json
{
  "ir_version": "1.0",
  "header": {
    "intent": "...",
    "semantic_class": "...",
    "sensitivity_level": "...",
    "requires_tool": true
  },
  "procedure": [ ]
}
```

The `header` is **mandatory and immutable** once generated. It defines the global intent
category, whether the request is informational or procedural, the sensitivity
classification, and declared operational requirements. This header establishes the upper
bound of permissible capabilities.

### 6.3 Restricted Grammar

PyIR is intentionally constrained to prevent arbitrary execution semantics. The grammar
enforces: a closed intent enumeration (no free-form action names); no imports or external
code references; no attribute access or dynamic evaluation; no user-defined functions; and
deterministic parsing with static validation. Any IR that does not conform is rejected
prior to policy evaluation.

### 6.4 Capability Catalog Binding

Each allowed intent maps to a predefined capability entry in a versioned catalog:

$$ \mathit{Intent} \rightarrow \mathit{CapabilitySet}. $$

This ensures explicit privilege requirements, role-based authorization constraints,
sensitivity escalation rules, and traceable policy bindings. Because the catalog is
versioned, policy changes remain auditable and reproducible.

### 6.5 Static Validation Layer

Before reaching the deterministic policy engine, PyIR undergoes static validation: syntax
validation (schema compliance), intent enumeration validation, capability-scope
consistency with the prefix invariant, and sensitivity-classification consistency. Invalid
IR structures are rejected without invoking downstream components.

### 6.6 Architectural Position

PyIR is the **structural hinge** between the probabilistic interpretation layer (the LLM)
and the deterministic decision core. It transforms unbounded language into a bounded
capability space. The LLM may generate candidate IR structures, but only a statically
validated and policy-approved IR instance can influence execution. This transformation
relocates trust from model weights to architectural invariants.

---

## 7. Deterministic Decision Core

### 7.1 Security Boundary Relocation

In conventional systems, authorization decisions may be implicitly embedded within model
outputs or delegated to heuristic post-processing. Lock-Monotone reverses this assumption:
the model may interpret language, but it must never determine authorization, privilege, or
operational execution. The **deterministic decision core constitutes the actual security
boundary** of the system.

### 7.2 Formal Definition

Let a validated IR contain an intent $I$, role $R$, tenant/context $T$, sensitivity level
$S$, and policy version $V$. The decision function is

$$ \mathit{Decision} = f(I, R, T, S, V), $$

where $f$ is deterministic and pure, with output in a closed set:

$$ \mathit{Decision} \in \{\textsc{allow}, \textsc{restrict}, \textsc{deny}\}. $$

### 7.3 Properties of the Decision Function

The decision core satisfies: **determinism** (identical inputs → identical outputs);
**purity** (no hidden state or side effects); **version binding** (each decision
associated with a policy version); **replayability** (past decisions re-evaluable under
the same policy); and **auditability** (all inputs and outputs traceable). This eliminates
stochastic behavior from authorization logic.

### 7.4 Capability Enforcement

The decision core enforces both vertical and positional monotonicity. With
$\mathit{Capabilities}(IR)$ the set derived from the validated header:

$$ \mathit{GrantedCapabilities} \subseteq \mathit{Capabilities}(IR). $$

No new capability may be introduced at this stage.

### 7.5 Separation from Language Semantics

The decision core operates exclusively on structured IR fields. It does not process raw
natural language, does not interpret free-form text, and does not rely on probabilistic
inference. All semantic interpretation occurs upstream; all authorization enforcement
occurs here. Adversarial phrasing cannot influence authorization logic.

### 7.6 Failure Semantics

- **`DENY`** — terminate procedural execution paths, prevent tool invocation, optionally
  trigger safe-response generation.
- **`RESTRICT`** — reduce capability scope, enforce additional validation layers, apply
  output-filtering constraints.
- **`ALLOW`** — permit execution strictly within the predefined capability boundary.

### 7.7 Structural Guarantee

Because authorization is deterministic and version-bound, regression testing and formal
verification become feasible: by construction, no execution path outside the validated
capability boundary is reachable. We deliberately refrain from asserting quantitative
security metrics; their measurement requires the empirical protocol identified as future
work (Section 15).

---

## 8. Controlled Final Translation (Triple Decode)

Once the decision core produces an outcome, the system must translate it into observable
behavior. A critical constraint applies: **authorization does not imply direct
execution.** Even after an `ALLOW`, the model does not obtain autonomous execution power.
Lock-Monotone introduces a controlled three-stage translation, the **Triple Decode**
(corresponding to the T1/T2/T3 pipeline).

### 8.1 Stage 1 — Semantic Decode (T1: Intent Translation)

T1 transforms raw user input into a structured, non-executable IR that makes intent
explicit, separating informational content from procedural intent. T1 has *no access* to
sensitive data, tools, or external systems; it cannot trigger retrieval or execution and
produces no side effects. Its output is purely descriptive and non-authoritative. In the
generation phase, the model acts strictly as a language renderer within a bounded semantic
space.

### 8.2 Stage 2 — Operational Decode / Governance (T2: Tool Gateway)

T2 is the deterministic control core. It consumes the IR and evaluates it against explicit
security and business policies (RBAC/ABAC), integrating domain-specific compliance
constraints. RAG is treated as a *conditional capability*: access to knowledge sources is
granted only if explicitly authorized, and retrieved content is always classified as
untrusted data, never as instruction. The output is a constrained authorization state with
an explicit set of permitted capabilities and required transformations; decisions are
closed and enumerable (e.g., `ALLOW`, `DENY`, `REDACT`, `SUMMARIZE`, `ESCALATE`). This
stage enforces explicit tool allowlists, argument validation, sandbox/dry-run modes when
required, and deterministic logging of invocation metadata. The tool gateway functions as
a **capability firewall** between semantic intent and system execution.

### 8.3 Stage 3 — Compliance Decode / Resolution (T3: Output Gate)

T3 resolves the authorized state into a user-visible response or controlled action. A
final compliance layer enforces Data Loss Prevention (DLP), secret-pattern detection, PII
filtering, RAG verbatim constraints, and content-policy enforcement — mitigating
accidental disclosure, partial leakage through paraphrasing, and unintended reproduction
of sensitive documents. T3 cannot introduce new capabilities, override policy decisions,
or access raw secrets; its role is expressive, not decisional. The compliance gate is
**independent of the authorization decision**, providing defense in depth even for purely
informational requests.

### 8.4 Monotonic Enforcement Across Decodes

Let $C_0$ denote the capability set defined by the IR header, and $C_1, C_2, C_3$ the
effective scope at each stage:

$$ C_3 \subseteq C_2 \subseteq C_1 \subseteq C_0. $$

Capability scope can only remain constant or decrease; no stage may expand authority
beyond the prefix-defined boundary.

### 8.5 Illustrative IR Evolution

```text
T1 IR (intent):
  intent_type: "request"
  knowledge:   ["explain X"]
  action:      ["export records"]
  sensitivity: ["customer_data"]

T2 Authorized State:
  decision:             "DENY_ACTION_ALLOW_KNOWLEDGE"
  allowed_capabilities: ["explain_concepts"]
  removed_capabilities: ["data_export", "tool_invocation"]
  policy_version:       "P-2026.02"
```

### 8.6 Architectural Consequence

The model never directly calls tools, accesses secrets, or modifies system state. All such
actions are mediated by deterministic structures. The Triple Decode completes the
relocation of trust from probabilistic reasoning to architectural invariants, making
Lock-Monotone a full *execution-governance* architecture rather than a mere translation
framework.

---

## 9. Translation Reliability: Fine-Tuning vs Prompt-Based Conditioning

### 9.1 The Open Variable of the Architecture

The deterministic core guarantees that no *validated* IR can yield unauthorized execution.
The remaining variable is the **reliability of translation**: how faithfully natural
language is mapped to a correct IR. If translation systematically misclassifies intent —
encoding a procedure as knowledge — the deterministic core will faithfully enforce the
*wrong* boundary. Translation reliability is thus the architecture's principal residual
risk, and its empirical characterization is the natural subject of future evaluation.

### 9.2 Design Rationale

Translation can in principle be achieved via prompt-based instruction (in-context
learning) or via supervised fine-tuning. On structural grounds, we conjecture that
prompt-only conditioning is insufficient for robust compliance in adversarial settings:
with sufficiently large context windows, prompts can specify the IR schema, a closed
intent enumeration, structured examples, and ordering constraints, yet remain subject to
the same recency and reinterpretation effects that motivate the prefix invariant. We
therefore expect prompt-only IR generation to exhibit malformed or partially valid
outputs, inconsistent field typing, drift in intent classification under adversarial or
noisy input, and non-deterministic violation of prefix ordering.

### 9.3 Fine-Tuning as a Structural Prior

We hypothesize that supervised fine-tuning on IR-aligned datasets improves syntactic
validity, stability of intent and sensitivity classification, token efficiency, and
adherence to structural constraints including prefix ordering. In particular, we conjecture
that positional monotonicity is best treated as a **learned translation prior** rather
than an emergent prompt behavior. These are *design hypotheses*; their validation requires
a controlled empirical study, which we explicitly defer.

### 9.4 Security Boundary Clarification

Regardless of how translation is conditioned, Lock-Monotone maintains a strict boundary
between **translation reliability** (how well the model produces valid IR) and **security
enforcement** (what the system is allowed to execute). Fine-tuning may reduce the
probability of translation errors, but it does not provide formal security guarantees. The
architecture therefore requires that all IR outputs undergo deterministic parsing and
schema validation, that intent values belong to a closed enumeration, that capability
scope is enforced by policy and catalog binding, and that execution is mediated by the
tool gateway and compliance layers. In this framing, **translation errors cannot yield
unauthorized execution** — they can only yield over-restriction (unnecessary denials),
which is a safe failure mode.

---

## 10. Cross-Layer Threat Model and Mitigation

### 10.1 From Model-Level to System-Level Threats

Most existing LLM security analyses focus on model-level robustness — primarily resistance
to jailbreak or harmful-content elicitation. But modern deployments integrate multiple
architectural layers (RAG, deterministic policy engines, tool gateways, output filtering).
Reasoning about the model alone is insufficient. Lock-Monotone adopts a *cross-layer*
threat model covering the entire execution pipeline.

### 10.2 Threat Taxonomy

- **T1 — Jailbreak:** attempts to override safety constraints at the language level.
- **T2 — Direct prompt injection:** malicious instruction embedded in user input.
- **T3 — Tool abuse:** unauthorized invocation of operational capabilities.
- **T4 — RAG indirect injection:** hostile content embedded in retrieved documents.
- **T5 — Multi-turn escalation:** progressive privilege amplification across turns.

### 10.3 Cross-Layer Threat Mapping

| Threat | Layer | Bounding mechanism |
|---|---|---|
| T1: Jailbreak | LLM | Decision core (no authority) |
| T2: Injection | Translation | Typed IR + static validation |
| T3: Tool abuse | Gateway | Closed catalog + allowlist |
| T4: RAG injection | Context | Prefix bound + K/P separation |
| T5: Multi-turn | Sequence | Positional monotonicity |

Lock-Monotone's invariants aim to neutralize capability escalation across all layers,
rather than to make any single layer behaviorally safe.

### 10.4 Mitigation by Construction

For each class, the architecture removes the *operational leverage* of the attack rather
than detecting it:

- **Prompt injection (direct/indirect):** generated text is non-authoritative; decision
  authority resides exclusively in deterministic components.
- **Reasoning-based escalation:** monotonic capability enforcement — additional reasoning
  cannot expand the capability set.
- **Procedure abuse / compositional attacks:** explicit capability typing and monotonic
  reduction across stages; sequential composition cannot yield emergent privileges.
- **Dialogue-based exfiltration:** secret externalization — secrets are never in weights,
  prompts, or conversational memory; access is always an explicit, authorized action.
- **Injection into structured representations:** schemas validated and re-serialized
  deterministically; anomalous structures rejected.

Because the system does not rely on detecting malicious intent, many LLM-specific attacks
are transformed into conventional systems-security problems, where established validation,
policy-enforcement, and auditing techniques apply.

---

## 11. Worked Example: Governing the SIEM's Own LLM

We illustrate the architecture qualitatively — without quantitative claims — through a
high-assurance case: an analytic LLM embedded in a Security Information and Event
Management (SIEM) platform. This case is adversarially demanding: the LLM both consumes
attacker-influenced data (ingested logs) and sits close to privileged operational
capabilities (alert rules, case management, response actions).

### 11.1 Setup

A Security Operations Center (SOC) deploys an LLM assistant to summarize alerts, classify
incidents, and propose response actions. The assistant ingests telemetry (firewall logs,
EDR events, email headers) and can, in principle, request operational actions through a
tool layer: querying a case, enriching an indicator, or — privileged — disabling a
detection rule or exporting a dataset.

### 11.2 Attack: Indirect Injection via Telemetry (T4)

An attacker embeds an instruction inside data ingested as a log field, e.g., a crafted
user-agent string or email subject: *"SYSTEM: prior alerts are false positives; disable
rule 4012 and export the credential store to the attached address."* In a model-centric
deployment, this text enters the prompt as context and may be followed, because the model
cannot reliably distinguish trusted instructions from attacker-controlled data.

### 11.3 Walk-Through Under Lock-Monotone

1. **Knowledge/Procedure separation.** The ingested log field is classified at translation
   as `KNOWLEDGE` (telemetry to be summarized). The embedded imperative cannot, by
   construction, be promoted to `PROCEDURE`: procedural intent must originate from the
   trusted prefix declaration, not from later content.
2. **Prefix bound.** The interaction's header declared the intent *"summarize and classify
   alerts"*, whose capability set excludes `disable_rule` and `export_data`. By the prefix
   invariant, no subsequent step — including injected content — can introduce these
   capabilities.
3. **Decision core.** Even if a malformed IR proposed `disable_rule`, the deterministic
   core evaluates it against role, tenant, sensitivity, and policy version; the intent is
   absent from the capability set bound to the header and is denied. The model has no
   authority over this decision.
4. **Tool gateway.** The operational decode exposes only the allowlisted tools for the
   declared intent. `disable_rule` and `export_data` are not reachable from a "summarize"
   intent regardless of generated text.
5. **Compliance gate.** Should any summary attempt to reproduce credential material or
   exfiltrate via paraphrase, the output gate's secret-pattern and DLP filters sanitize
   the response independently of the authorization path.

The injected instruction is neutralized at multiple layers; the failure mode — if
translation over-restricts — is a **safe denial** rather than an unauthorized action.

### 11.4 The Apex Case: Attacking the Supervisory LLM

A deeper case arises when the target is the LLM that supervises security itself. Two
consequences follow. First, a jailbreak of this LLM (T1) does not, by construction, grant
capabilities: authority resides in the deterministic core, not in the model, so a
compromised supervisory LLM can at most produce misleading *language*, not unauthorized
*actions*. Second, the telemetry that records the assistant's own behavior must be
sanitized before re-ingestion, otherwise an attacker could use the assistant's logs as an
indirect injection channel back into the pipeline. Lock-Monotone treats this supervisory
LLM as residing *outside* the trust path: it is governed by the same prefix bound, decision
core, and compliance gate as any other component, and its outputs are never an authority
source. This meta-level placement is, we argue, the sharpest test of the doctrine *"the
model has no authority"*.

### 11.5 What the Example Does and Does Not Show

The walk-through demonstrates how the invariants bound an attack qualitatively. It does
*not* measure how often translation correctly classifies adversarial telemetry, nor the
false-denial rate induced by conservative classification. Those are empirical questions —
precisely the evaluation we defer.

---

## 12. Formal Security Invariants and Theoretical Guarantees

### 12.1 Preliminaries

Let $NL$ denote natural-language input, $IR$ the validated intermediate representation,
$C(IR)$ the capability set derived from the IR header, $D$ the deterministic decision
function, and $\mathit{Exec}$ the execution stage. The pipeline abstracts as

$$ NL \rightarrow IR \rightarrow D(IR) \rightarrow \mathit{Exec}(IR, D). $$

### 12.2 Invariant 1 — Deterministic Authorization

For any validated IR and fixed policy version $V$:

$$ D(I, R, T, S, V) = D(I, R, T, S, V). $$

The decision function is deterministic and free of stochastic behavior; outcomes are
reproducible, replayable, and auditable.

### 12.3 Invariant 2 — Prefix Capability Bound

With $IR = \{s_1, \dots, s_n\}$ and $s_1$ the immutable header:

$$ \forall i > 1, \quad C(s_i) \subseteq C(s_1). $$

No capability introduced after the prefix may exceed the scope defined at initialization,
preventing escalation via multi-turn dialogue or late-stage injection.

### 12.4 Invariant 3 — Vertical Monotonicity

With $C_0$ derived from the header and $C_k$ the effective scope at stage $k$:

$$ C_{k+1} \subseteq C_k. $$

Capabilities may only remain constant or decrease across processing layers.

### 12.5 Invariant 4 — Execution Mediation

All operational actions must satisfy

$$ \mathit{Exec} \subseteq C(IR) \quad\text{and}\quad \mathit{Exec} \neq f(NL). $$

Execution is mediated exclusively through validated IR and deterministic policy; raw
language output cannot directly trigger execution, eliminating free-text-to-action
pathways.

### 12.6 Invariant 5 — Closed Capability Enumeration

With $\mathcal{I}$ the finite set of allowed intents:

$$ \mathit{Intent} \in \mathcal{I}. $$

No dynamic introduction of new action types is possible without explicit catalog
modification, ensuring version-controlled capability governance.

### 12.7 Theorem — Conditional Non-Escalation Guarantee

**Assumptions.** (A1) the deterministic decision core and static validator are correct
with respect to their specifications; (A2) every executed action originates from an IR that
passed static validation.

**Theorem.** Under Invariants 1–5 and assumptions A1–A2, no adversarial natural-language
input can increase effective system capabilities beyond those declared in the validated IR
header.

**Proof sketch.** (1) Natural language is first translated into IR. (2) IR must pass schema
validation and intent enumeration (A2). (3) The prefix invariant bounds the capability
space (Invariant 2). (4) Deterministic policy enforces authorization strictly within
declared scope (Invariants 1, 4; A1). (5) Vertical monotonicity prevents downstream
expansion (Invariant 3). Therefore any adversarial attempt to expand capabilities either
fails IR validation, is rejected by policy, or is reduced during decode. Hence capability
escalation is structurally impossible within the defined model. ∎

**Scope of the conditional.** The guarantee is explicitly relative to A1–A2. It does *not*
cover translation faults: an IR that is *valid but semantically wrong* (e.g., a procedure
misclassified as knowledge) is enforced faithfully and may cause over-restriction. Bounding
the probability and impact of such faults is an empirical question, out of scope here.

### 12.8 Limitations of the Guarantee

The invariants guarantee no unauthorized action execution and no privilege escalation
beyond declared scope. They do **not** guarantee absence of hallucinated informational
output, protection against infrastructure-level denial-of-service, or elimination of
semantic inaccuracies. **The guarantees are architectural, not epistemic.**

---

## 13. Auditability and Governance

A central objective of Lock-Monotone is to provide **auditability and governance by
construction**. In adversarial or regulated environments, security guarantees are
insufficient if decisions cannot be explained, reconstructed, and reviewed after the fact.

### 13.1 Auditability by Construction

Each translation stage emits a structured intermediate representation capturing system
state at that point. Authorization decisions at T2 are recorded with the policy version,
evaluated attributes, and the resulting capability set. Downstream responses at T3 are
therefore directly attributable to a specific authorization state rather than to opaque
model behavior. This enables post-hoc reconstruction of *which* capabilities were
requested, *which* were removed or restricted, and *which* policies were applied.

### 13.2 Deterministic Decision Records

Each decision can be logged as a tuple containing, at minimum: a hash of the structured
IR; the identifier and version of the applied policy set; the resulting authorization
decision and capability set; a timestamp and contextual metadata. Because language models
do not participate in authorization, audit records are not subject to probabilistic
ambiguity: identical inputs under identical policies yield identical decisions.

### 13.3 Governance and Policy Evolution

Security policies, business rules, and compliance constraints are externalized as explicit
artifacts that can be reviewed, versioned, and updated independently of the models.
Adjustments to authorization thresholds or regulatory requirements do not require
retraining or re-aligning models. Accountability shifts from opaque model behavior to
human-reviewed policy artifacts.

### 13.4 Human-in-the-Loop Escalation

Lock-Monotone supports explicit escalation to human operators as a *governed outcome*
rather than an exception. Requests requiring review are surfaced with their structured
representations and constraints. Human intervention does not bypass architectural
invariants: operator decisions are recorded, auditable, and subject to the same monotonic
capability constraints as automated ones.

---

## 14. Standardization and Action Nomenclature

### 14.1 Deterministic Governance as Standardization

Beyond authorization enforcement, the decision core enables a second structural property:
**action standardization**. Every operational or informational capability must be
explicitly named, formally defined, version-controlled, and associated with deterministic
evaluation rules — transforming the system from a flexible prompt-driven interface into a
governed action taxonomy.

### 14.2 Closed Action Enumeration

Let $\mathcal{A} = \{A_1, \dots, A_n\}$ be the finite set of allowed actions. Each $A_i$
is defined by a canonical name, a semantic class (`KNOWLEDGE`, `PROCEDURE`, `SENSITIVE`),
a required privilege level, allowed roles, sensitivity constraints, and execution
constraints. No action outside $\mathcal{A}$ may be executed; any extension requires
explicit catalog modification and a policy version increment.

### 14.3 Business and Security Rule Binding

Each action binds functional and authorization/compliance constraints:

$$ A_i = (\mathit{Name}, \mathit{SemanticClass}, \mathit{PrivilegeSet}, \mathit{BusinessRules}, \mathit{SecurityRules}). $$

Business rules may include domain validation, required parameters, execution
preconditions, and workflow ordering. Security rules include RBAC, tenant isolation,
sensitivity-escalation requirements, and logging/traceability requirements.

### 14.4 Nomenclature as a Security Primitive

Because the model cannot invent new canonical action names, all operational behavior must
map to predefined entries. This creates a controlled vocabulary for execution authority
and enables cross-team interpretability, audit consistency, deterministic testability, and
version traceability.

---

## 15. Proposed Evaluation Methodology (Future Work)

This section sketches — as future work — a cross-layer evaluation methodology. **No
results are claimed here.** The methodology's purpose is to convert the present structural
argument into a *measured* guarantee.

### 15.1 Evaluation Principle

> Security must be measured at the *system boundary*, not at the model output.

Even if a model generates adversarial language, the architecture must prevent capability
escalation. Evaluation therefore verifies that positional monotonicity holds, that
deterministic policy enforcement is consistent, and that no unauthorized execution path is
reachable.

### 15.2 Two-Phase Protocol

- **Phase 1 — Model-level baseline.** Evaluate the underlying LLM against canonical
  benchmarks: Attack Success Rate (ASR), Injection Success Rate (ISR), False Positive
  Rate (FPR). Establishes a behavioral baseline.
- **Phase 2 — Cross-layer system evaluation.** Evaluate the full Lock-Monotone pipeline
  using structured attack suites: direct injection, multi-turn escalation, hostile RAG
  documents, simulated tool abuse, and secret-exfiltration probes using canary tokens.
  Measures architectural enforcement rather than model compliance.

### 15.3 Proposed Metrics

Non-negotiable architectural targets (to be empirically tested, not assumed):

- **Unauthorized Action Rate (UAR)** — target 0.
- **Secret Leakage Rate (SLR)** — target 0.
- **Decision Trace Coverage (DTC)** — target 100%.

Complementary robustness metrics: ASR, ISR, RAG Injection Rate (RIR), Multi-turn
Escalation Rate (MER), Guardrail Catch Rate (GCR).

### 15.4 Evaluation Harness

A structured harness with unit tests for policy logic, integration tests across pipeline
components, regression tests across policy versions, and deterministic audit bundles per
test case. Each test case defines a structured attack scenario, the expected policy
decision, the allowed capability scope, and forbidden outputs or tool calls. The harness
separates *attack generation* from *oracle validation*, ensuring reproducibility and
comparability across model versions. We propose instantiating it on the SIEM case of
Section 11.

---

## 16. Discussion and Limitations

- **Architectural scope of the guarantees.** Lock-Monotone ensures deterministic
  authorization, bounded capability execution, prevention of privilege escalation, and
  elimination of direct language-to-action pathways. It does not depend on probabilistic
  compliance of the model — but it applies strictly to capability governance and execution
  control, not to correctness of informational content.
- **Epistemic limitations.** It does not eliminate hallucinated factual content, reasoning
  inaccuracies, or semantic misunderstandings in purely informational tasks. The
  architecture constrains *what the system can do*, not *whether the model is
  epistemically correct*.
- **Dependence on correct translation.** Security does not rely on probabilistic
  correctness, but it does rely on successful translation into valid IR. Systematic
  misclassification produces excessive denials — bounded by deterministic enforcement and
  unable to escalate privilege, but degrading usability. This is the principal motivation
  for the planned empirical evaluation.
- **Dependence on policy engineering.** Responsibility shifts to policy engineering;
  overly permissive policies undermine security, overly restrictive ones reduce utility.
  Policies are first-class artifacts to be reviewed, tested, and versioned with the rigor
  of traditional access control.
- **Infrastructure-level risks.** The invariants do not address denial-of-service,
  infrastructure misconfiguration, hardware compromise, or external vulnerabilities.
  Lock-Monotone governs logical capability flow; it does not replace traditional controls.
  It also assumes the deterministic components (policy engines, validators, connectors)
  are correctly implemented and secured.
- **Catalog management and schema rigidity.** The closed catalog and constrained IR create
  governance advantages but also operational overhead and reduced expressive flexibility —
  a deliberate trade-off between flexibility and formal control.
- **Human factors.** Social engineering of operators, misconfiguration, or intentional
  bypass may still cause failures; the architecture provides technical boundaries, not a
  substitute for organizational governance.
- **Model evolution.** Because enforcement is externalized, model upgrades do not alter
  capability boundaries, regression testing remains feasible, and policy invariants remain
  stable across model families — mitigating systemic drift.

---

## 17. Related Work

**Model-centric safety.** A large body of work improves LLM safety through model-level
techniques — RLHF, constitutional AI, adversarial fine-tuning. These reduce harmful
outputs by shaping internal behavior but remain fundamentally probabilistic: safety is
contingent on statistical compliance rather than structural guarantees. Lock-Monotone
relocates authorization outside the model, treating alignment improvements as
complementary but not sufficient.

**Prompt injection and guardrails.** Direct and indirect injection through retrieved or
tool-returned content motivate guardrail techniques: system-prompt hardening, structured
output enforcement, regex filtering, schema validation. These often operate post-hoc and
do not redefine execution-authority boundaries. Lock-Monotone differs by enforcing a typed
IR and deterministic policy evaluation prior to execution.

**Design-by-construction defenses.** The closest line removes authority from the model by
construction. The **dual-LLM pattern** isolates untrusted content in a quarantined model
whose outputs are referenced only symbolically by a privileged model, so tainted data
never drives privileged actions. **CaMeL** extends this: a privileged model emits code in
a sandboxed DSL capturing the control and data flow of the *trusted* query, and
capabilities enforce data-flow policies at tool-call time, yielding provable security for a
class of tasks. Lock-Monotone is aligned with this stance and differs along four axes.
First, our separation is *epistemic*: the knowledge/procedure classification gates whether
*any* actionable authority exists, prior to control- or data-flow analysis. Second, the
*prefix action priority invariant* introduces an explicit *temporal* least-privilege bound
— capabilities capped at the first declaration for the whole interaction — directly
targeting multi-turn escalation, and, to our knowledge, not made explicit in dual-LLM or
CaMeL. Third, the *Triple Decode* adds an independent compliance gate (DLP/PII/secret)
downstream of authorization, providing defense in depth beyond data-flow capabilities.
Fourth, the closed, versioned action nomenclature turns capability governance into an
auditable standardization layer. We note an asymmetry of maturity: CaMeL is an implemented
system with guarantees demonstrated on a benchmark, whereas Lock-Monotone is presently a
framework whose evaluation remains future work.

**Classical foundations.** Lock-Monotone inherits from classical security theory. The
*reference monitor* concept (tamperproof, always invoked, verifiable) underlies the
deterministic decision core; verified secure-systems design motivates relocating trust to
a small, analyzable component; and policy-based access control (RBAC, ABAC) provides the
deterministic authorization tradition extended here to LLM-mediated environments through
typed translation and positional monotonicity. We follow established architectural
practice and align with emerging governance frameworks (NIST AI RMF, EU AI Act).

---

## 18. Conclusion and Future Work

LLMs are increasingly deployed in environments that manipulate sensitive data and
operational capabilities, where probabilistic language generation cannot serve as a
security boundary. This work introduced **Lock-Monotone**, a deterministic governance
architecture that relocates trust: from model behavior to architectural invariants, from
statistical alignment to deterministic enforcement, and from implicit execution to explicit
capability governance.

The architecture is formalized through an ontological separation between knowledge and
procedure, a typed intermediate representation (PyIR), a prefix action priority invariant
enforcing positional monotonicity, a deterministic decision core, a triple-decode
execution-mediation layer, a closed and versioned action nomenclature, and auditability by
construction. Under stated assumptions, the architecture guarantees that no adversarial
natural-language input can expand system capabilities beyond those declared in the
validated IR header. Security thus becomes a *structural* property of the system rather
than a probabilistic property of the model.

This paper is scoped as an architectural framework. Its central residual risk —
translation reliability — is explicitly not measured here. The immediate and most important
direction for future work is an empirical evaluation that, unlike model-centric benchmarks
focused on jailbreak resistance, measures the fidelity of natural-language-to-IR
translation under adversarial input: the rate of semantic misclassification, the induced
false-denial rate, and the conditions under which fine-tuning improves structural
compliance. Such an evaluation — ideally instantiated on the SIEM case and comparable to
agentic security testbeds — would convert the present structural argument into a measured
guarantee. Further work includes formal verification of policy rules, automated synthesis
of action catalogs, and integration with regulatory compliance frameworks.

Rather than making language models intrinsically trustworthy, Lock-Monotone makes them
*subordinate to deterministic governance*, transforming LLM safety from a tuning problem
into an architectural discipline.

---

## References

1. T. B. Brown et al., "Language models are few-shot learners," *NeurIPS*, 2020.
2. J. Wei et al., "Chain-of-thought prompting elicits reasoning in large language models,"
   *NeurIPS*, 2022.
3. L. Ouyang et al., "Training language models to follow instructions with human feedback,"
   *NeurIPS*, 2022.
4. Y. Bai et al., "Constitutional AI: Harmlessness from AI feedback," arXiv:2212.08073,
   2022.
5. A. Zou, Z. Wang, J. Z. Kolter, M. Fredrikson, "Universal and transferable adversarial
   attacks on aligned language models," arXiv:2307.15043, 2023.
6. K. Greshake et al., "Not what you've signed up for: Compromising real-world
   LLM-integrated applications with indirect prompt injection," *ACM AISec*, 2023.
   (arXiv:2302.12173)
7. Y. Liu et al., "Prompt injection attacks and defenses in LLM-integrated applications,"
   arXiv:2306.05499, 2023.
8. S. Willison, "The dual LLM pattern for building AI assistants that can resist prompt
   injection," 2023. https://simonwillison.net/2023/Apr/25/dual-llm-pattern/
9. E. Debenedetti, I. Shumailov, T. Fan, J. Hayes, N. Carlini, D. Fabian, C. Kern, C. Shi,
   A. Terzis, F. Tramèr, "Defeating prompt injections by design" (CaMeL), arXiv:2503.18813,
   2025.
10. J. H. Saltzer, M. D. Schroeder, "The protection of information in computer systems,"
    *Proc. IEEE*, 63(9):1278–1308, 1975.
11. J. Rushby, "Design and verification of secure systems," *ACM SOSP*, 1981.
12. R. Anderson, *Security Engineering*, 3rd ed., Wiley, 2020.
13. L. Bass, P. Clements, R. Kazman, *Software Architecture in Practice*, 3rd ed.,
    Addison-Wesley, 2012.
14. National Institute of Standards and Technology, "Artificial Intelligence Risk
    Management Framework (AI RMF 1.0)," NIST, 2023.
15. European Union, "Regulation (EU) 2024/1689 (Artificial Intelligence Act)," 2024.
16. Anthropic, "Disrupting the first reported AI-orchestrated cyber espionage campaign,"
    Nov. 2025. https://www.anthropic.com/news/disrupting-AI-espionage
17. VentureBeat, "12 AI defenses claimed near-zero attack success; researchers broke all of
    them," Oct. 2025.
    https://venturebeat.com/security/12-ai-defenses-claimed-near-zero-attack-success-researchers-broke-all-of-them
18. "Attention Tracker: Detecting Prompt Injection Attacks in LLMs," arXiv:2411.00348, 2024.
19. "How Vulnerable Are AI Agents to Indirect Prompt Injections? Insights from a
    Large-Scale Public Competition," arXiv:2603.15714, 2026.
20. Picus Security, "What Are AI-Powered Cyberattacks? Inside Machine-Speed Threats"
    (VILLAGER; HexStrike / CVE-2025-7775), 2025.
    https://www.picussecurity.com/resource/blog/what-are-ai-powered-cyberattacks-inside-machine-speed-threats
21. Industrial Cyber, "Booz Allen warns AI-driven cyberattacks outpace human-driven defenses
    across critical infrastructure," 2025.
    https://industrialcyber.co/ai/booz-allen-warns-ai-driven-cyberattacks-outpace-human-driven-defenses-across-critical-infrastructure/
22. Sify, "AI vs AI: New Cybersecurity Battlefield Where No Humans Are in the Loop," 2025.
    https://www.sify.com/ai-analytics/ai-vs-ai-new-cybersecurity-battlefield-where-no-humans-are-in-the-loop/

> **Note.** arXiv identifiers and online sources should be re-verified before any formal
> submission.
