# 04 — Modèle d’Intermediate Representation (IR)

## Schéma IR (JSON)

```json
{
  "schema_version": "1.0",
  "request_id": "uuid",
  "actor_role": "SOC_JUNIOR|SOC_SENIOR|CISO|AUDITOR",
  "procedure_id": "PROC-INC-TRIAGE|PROC-INC-SUMMARY|PROC-IOC-CHECK|PROC-LOG-SCOPE|PROC-ESCALATE|PROC-PLAYBOOK-MAP|PROC-RAG-QUERY|PROC-REFUSE",
  "inputs": {
    "summary": "string",
    "ioc_list": ["string"],
    "log_scope": "string",
    "playbook_hint": "string",
    "rag_query": "string"
  },
  "requested_actions": ["RAG_QUERY"],
  "decision_trace": {
    "policy_version": "string",
    "decision": "ALLOW|DENY|DEFER|REDACT",
    "reason_codes": ["string"]
  }
}
```

## Types de champs
- `schema_version` : string, format `major.minor`.
- `request_id` : UUID v4.
- `actor_role` : enum fermé.
- `procedure_id` : enum fermé (voir 03-procedure-nomenclature.md).
- `inputs` : objet, champs optionnels selon la procédure.
- `requested_actions` : liste fermée, valeurs autorisées `RAG_QUERY`.
- `decision_trace` : objet, produit par le moteur de politique.

## Champs obligatoires
- `schema_version`
- `request_id`
- `actor_role`
- `procedure_id`
- `inputs`

## Champs optionnels
- `requested_actions`
- `decision_trace`

## Champs interdits
- Tout secret (clé API, mot de passe, token).
- Tout champ non défini par le schéma.

## Exemples valides

```json
{
  "schema_version": "1.0",
  "request_id": "550e8400-e29b-41d4-a716-446655440000",
  "actor_role": "SOC_JUNIOR",
  "procedure_id": "PROC-INC-TRIAGE",
  "inputs": {
    "summary": "Échec d’authentification répété sur un compte admin."
  }
}
```

```json
{
  "schema_version": "1.0",
  "request_id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "actor_role": "SOC_SENIOR",
  "procedure_id": "PROC-RAG-QUERY",
  "inputs": {
    "rag_query": "Procédures internes pour incident de phishing."
  },
  "requested_actions": ["RAG_QUERY"]
}
```

## Exemples invalides

```json
{
  "schema_version": "1.0",
  "request_id": "not-a-uuid",
  "actor_role": "SOC_JUNIOR",
  "procedure_id": "PROC-UNKNOWN",
  "inputs": {}
}
```

```json
{
  "schema_version": "1.0",
  "request_id": "550e8400-e29b-41d4-a716-446655440000",
  "actor_role": "SOC_JUNIOR",
  "procedure_id": "PROC-INC-TRIAGE",
  "inputs": {
    "summary": "...",
    "api_key": "SECRET"
  }
}
```
