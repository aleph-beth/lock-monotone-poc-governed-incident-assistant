# 06 — Policy Engine (décisionnel)

## Entrées du moteur
- IR validé.
- Registre de rôles (versionné).
- Liste des procédures autorisées (versionnée).
- Table des règles de décision.

## Types de décisions possibles (liste fermée)
- ALLOW
- DENY
- DEFER
- REDACT

## Règles de décision par rôle

| Rôle | Procédures autorisées | Restrictions |
| --- | --- | --- |
| SOC_JUNIOR | PROC-INC-TRIAGE, PROC-INC-SUMMARY, PROC-IOC-CHECK, PROC-LOG-SCOPE, PROC-ESCALATE, PROC-REFUSE | Pas de PROC-RAG-QUERY. |
| SOC_SENIOR | Toutes sauf PROC-REFUSE obligatoire en cas d’anomalie | RAG autorisé si `requested_actions` = RAG_QUERY. |
| CISO | PROC-INC-SUMMARY, PROC-PLAYBOOK-MAP, PROC-RAG-QUERY, PROC-REFUSE | Pas de PROC-IOC-CHECK. |
| AUDITOR | PROC-INC-SUMMARY, PROC-REFUSE | Accès en lecture seule. |

## Versionnement des politiques
- Chaque politique a un identifiant de version immuable.
- Les décisions journalisent `policy_version`.
- La version active est sélectionnée hors LLM.

## Cas de refus, dégradation, escalade
- Refus : procédure non autorisée ou IR invalide.
- Dégradation : décision REDACT si la réponse contient des données sensibles.
- Escalade : décision DEFER si la procédure requiert validation humaine.

## Règle critique
Aucune règle ne dépend d’un raisonnement LLM.
