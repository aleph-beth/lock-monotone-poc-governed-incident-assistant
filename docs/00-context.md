# 00 — Contexte et hypothèses

## Objectif du PoC
Définir un assistant d’analyse d’incident sécurité gouverné par Lock-Monotone, documenté de façon assez précise pour être codé sans ambiguïté.

## Ce que le PoC démontre
- Un flux décisionnel déterministe séparant raisonnement et décision.
- Une montée des droits strictement monotone.
- L’externalisation des secrets hors LLM.
- Une traçabilité complète de chaque décision.

## Ce que le PoC ne cherche pas à démontrer
- La performance d’un modèle LLM.
- La détection automatique d’attaques réelles.
- L’optimisation d’un moteur de recherche documentaire.
- L’ergonomie d’une interface utilisateur.

## Hypothèses de sécurité (vérifiables)
- Les rôles humains sont pré-enregistrés dans un registre de rôles versionné.
- Les procédures autorisées sont une liste fermée et versionnée.
- Les secrets applicatifs ne sont jamais présents dans les prompts ni dans l’IR.
- Chaque décision du moteur de politique est journalisée avec un identifiant unique.
- Les documents RAG portent des métadonnées de sensibilité validées.

## Hors périmètre explicite
- Contournement par compromission du serveur hôte.
- Sécurité réseau (TLS, pare-feu, segmentation).
- Gestion d’identités d’entreprise (SSO, MFA).
- Chiffrement au repos des bases de données.

## Contraintes
- Phrases courtes.
- Aucun jargon inutile.
- Tout comportement doit être déterministe.
- Aucun comportement implicite.
