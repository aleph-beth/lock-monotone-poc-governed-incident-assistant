# 02 — Objectifs de sécurité et invariants

## Invariants Lock-Monotone appliqués au PoC

| Invariant | Définition opérationnelle | Preuve attendue |
| --- | --- | --- |
| Séparation raisonnement / décision | Le LLM propose. Le moteur de politique décide. | Les décisions proviennent uniquement du moteur de politique. |
| Monotonie des droits | Un droit accordé ne peut pas être supprimé dans une même session. | L’historique de droits est append-only. |
| Externalisation des secrets | Aucun secret dans les prompts ni l’IR. | Vérification par inspection de l’IR et des logs. |
| Auditabilité | Chaque décision est journalisée et rejouable. | Un journal complet lie requête, décision et sortie. |

## Menaces couvertes (liste fermée)
- Prompt injection directe.
- Prompt injection indirecte via documents.
- Escalade de procédure par reformulation.
- Accès non autorisé à un corpus sensible.
- Fuite de données par réponse du LLM.

## Menaces hors périmètre (liste fermée)
- Compromission du système d’exploitation.
- Attaques réseau actives (MITM, DNS spoofing).
- Abus par un administrateur root.
- Fuite par canaux auxiliaires matériels.
