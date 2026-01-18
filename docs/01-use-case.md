# 01 — Cas d’usage sensible et maîtrisable

## Description du cas d’usage
Assistant d’analyse d’incident sécurité. Il aide à trier, résumer et orienter des signaux d’alerte internes sans décider seul.

## Acteurs humains
- Analyste SOC junior.
- Analyste SOC senior.
- Responsable sécurité (CISO ou RSSI).
- Auditeur conformité.

## Données manipulées (catégories)
- Journaux d’authentification.
- Journaux réseau.
- Alertes EDR.
- Tickets d’incident.
- Procédures internes d’intervention.
- Indicateurs de compromission (IOC) internes.

## Risques principaux
- Exfiltration de données sensibles.
- Sur-autorisation d’accès aux sources.
- Confusion entre données et instructions.

## Résultat attendu
Un résumé structuré et une recommandation de procédure, sans exécution d’actions ni accès non autorisé.

## Pourquoi ce cas justifie Lock-Monotone
- Le risque d’exfiltration est élevé.
- Les droits doivent évoluer progressivement.
- La séparation décision / raisonnement est nécessaire.
- Les audits doivent pouvoir rejouer les décisions.
