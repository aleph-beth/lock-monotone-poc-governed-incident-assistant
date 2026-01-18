# 07 — RAG contrôlé

## Corpus autorisés (liste fermée)
- Playbooks internes validés.
- Guides de réponse à incident internes.
- Documentation de politiques internes.

## Métadonnées de sensibilité
- `sensitivity_level`: PUBLIC | INTERNAL | CONFIDENTIAL
- `owner_team`: string
- `retention_days`: integer

## Conditions d’accès
- Le rôle doit autoriser PROC-RAG-QUERY.
- La requête doit être validée par le policy engine.
- Seuls les documents avec métadonnées complètes sont admissibles.

## Traitement des documents injectés
- Les documents externes ne sont jamais ingérés.
- Tout document sans métadonnées est rejeté.
- Les extraits retournés sont journalisés.

## Pourquoi le RAG ne décide jamais
- Le RAG ne produit pas de décision.
- Le RAG ne modifie pas l’IR.
- La décision reste externe et déterministe.
