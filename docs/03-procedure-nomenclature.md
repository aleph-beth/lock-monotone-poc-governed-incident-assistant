# 03 — Nomenclature des procédures

## Règle
Liste fermée. Aucune procédure implicite. Ce qui n’est pas nommé n’existe pas.

## Procédures autorisées (8)

| Identifiant stable | Description fonctionnelle | Niveau de risque | Types de réponses autorisées | Interdictions explicites |
| --- | --- | --- | --- | --- |
| PROC-INC-TRIAGE | Trier un incident à partir d’un résumé de signaux. | Bas | Texte structuré, IR | Pas d’accès aux données brutes. |
| PROC-INC-SUMMARY | Résumer un incident avec un plan d’action proposé. | Bas | Texte structuré, IR | Pas de recommandations d’actions techniques. |
| PROC-IOC-CHECK | Vérifier la cohérence d’IOC fournis par l’humain. | Moyen | Texte structuré | Pas d’enrichissement externe. |
| PROC-LOG-SCOPE | Définir un périmètre de logs à demander. | Moyen | Texte structuré, IR | Pas d’accès direct aux logs. |
| PROC-ESCALATE | Préparer une escalade vers un senior. | Moyen | Texte structuré | Pas de décision finale. |
| PROC-PLAYBOOK-MAP | Associer un incident à un playbook existant. | Moyen | Texte structuré, IR | Pas d’exécution de playbook. |
| PROC-RAG-QUERY | Demander une recherche documentaire contrôlée. | Élevé | IR uniquement | Pas de réponse finale sans validation. |
| PROC-REFUSE | Refuser une demande non autorisée. | Bas | Texte court de refus | Pas de justification sensible. |
