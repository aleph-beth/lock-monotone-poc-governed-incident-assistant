# 05 — Architecture fonctionnelle du PoC

## Pipeline complet (flux unidirectionnels)
1. LLM amont → produit un brouillon de requête en IR.
2. Validation → vérifie le schéma IR et la liste fermée des procédures.
3. Policy engine → décide ALLOW/DENY/DEFER/REDACT.
4. RAG conditionnel → s’exécute seulement si `requested_actions` est autorisé.
5. LLM aval → produit la réponse à partir de l’IR validé et des sources autorisées.
6. Audit → journalise requête, décision, résultat.

## Rôle exact de chaque composant
- LLM amont : propose. Ne décide pas.
- Validation : refuse toute divergence du schéma.
- Policy engine : unique source de décision.
- RAG conditionnel : récupère des documents autorisés uniquement.
- LLM aval : formule la réponse finale.
- Audit : rend chaque étape traçable.

## Frontières de confiance
- Frontière A : entre utilisateur et LLM amont.
- Frontière B : entre LLM amont et validation.
- Frontière C : entre policy engine et RAG.
- Frontière D : entre RAG et LLM aval.
- Frontière E : entre LLM aval et audit.

## Contraintes de flux
- Aucun flux retour du LLM aval vers le policy engine.
- Aucun accès direct du LLM aux sources de données.
- Les décisions ne transitent pas par le LLM.
