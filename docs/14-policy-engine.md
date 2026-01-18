# Policy Engine

## 1. Rôle du Policy Engine
Le Policy Engine décide à partir de l’IR ACTION/KNOWLEDGE. Il produit une décision fermée et une enveloppe de capacités. Il ne dépend pas d’un LLM.

## 2. Entrées du moteur
- **IR validé** : traduction ACTION/KNOWLEDGE.
- **Identité / rôle** : `subject_id`, `roles`, `org_unit`.
- **Contexte** : `tenant`, `environment`, `request_time`.
- **Version de politique** : `policy_version`.

## 3. Décisions possibles (fermées)
- **ALLOW** : accès autorisé dans l’enveloppe définie.
- **DENY** : aucune donnée n’est accessible.
- **REDACT** : accès autorisé avec masquage partiel.
- **SUMMARIZE** : accès autorisé, sortie obligatoirement résumée.
- **ESCALATE** : demande d’approbation humaine obligatoire.

Aucune autre décision n’est autorisée.

## 4. Enveloppe de capacités

### Définition
L’enveloppe de capacités est l’intersection des droits du sujet, de la politique active et des contraintes de l’IR.

### Structure (exemple JSON)
```json
{
  "policy_version": "2024-10-01",
  "decision": "ALLOW",
  "capabilities": {
    "sources": ["policies/hr"],
    "time_range": "ALL",
    "sensitivity": "internal",
    "outputs": ["summary"]
  },
  "reason_codes": ["ROLE_MATCH", "SOURCE_ALLOWLIST"]
}
```

### Réduction par intersection
- `sources` = intersection (rôle, policy, IR).
- `outputs` = intersection (policy, IR).
- `sensitivity` = minimum autorisé.
- Toute intersection vide ⇒ `DENY`.

## 5. Exemples de décisions

### Exemple 1 — ALLOW
- **IR** : `procedure.read` sur `policies/hr`.
- **Rôle** : `hr-analyst`.
- **Décision** : `ALLOW`.
- **Capacités restantes** : `sources=[policies/hr]`, `outputs=[summary]`.

### Exemple 2 — REDACT
- **IR** : `incident.summary` sur `incidents/siem`, sensibilité `restricted`.
- **Rôle** : `security-analyst`.
- **Décision** : `REDACT`.
- **Capacités restantes** : masquage de champs PII imposé.

### Exemple 3 — DENY
- **IR** : `policy.read` sur `policies/finance`.
- **Rôle** : `intern`.
- **Décision** : `DENY`.
- **Capacités restantes** : aucune.

### Exemple 4 — ESCALATE
- **IR** : `asset.list` sur `assets/cmdb`.
- **Contexte** : environnement `prod` hors heures ouvrées.
- **Décision** : `ESCALATE`.
- **Capacités restantes** : aucune tant que l’approbation n’est pas obtenue.

## 6. Versionnement et audit
- **policy_version** est obligatoire dans toute décision.
- Chaque décision est journalisée avec : IR canonicalisé, identité, contexte, décision, enveloppe.
- La reproductibilité est garantie par la combinaison `policy_version` + IR canonicalisé.
