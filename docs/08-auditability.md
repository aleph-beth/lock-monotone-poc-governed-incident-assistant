# 08 — Auditabilité et traçabilité

## Éléments journalisés
- `request_id`
- `actor_role`
- `procedure_id`
- `policy_version`
- `decision`
- `reason_codes`
- `requested_actions`
- `rag_documents` (identifiants uniquement)
- `response_hash`
- Horodatage ISO 8601

## Format des logs
- JSON Lines.
- Une entrée par étape.
- Ordre chronologique strict.

## Corrélation requête / décision
- `request_id` est présent dans chaque entrée.
- `decision` est unique par `request_id`.

## Rejouabilité
- Les entrées permettent de reconstruire la décision.
- Les documents utilisés sont référencés par identifiant stable.

## Usage conformité / forensic
- Export possible par plage de dates.
- Vérification de cohérence par hachage.
