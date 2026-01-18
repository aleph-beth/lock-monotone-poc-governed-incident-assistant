# Lock-Monotone PoC — Documentation technique

## Définition courte
Lock-Monotone est une architecture de sécurité pour systèmes intégrant des LLM. Elle sépare la compréhension d’une requête (traduction ACTION/KNOWLEDGE) de l’autorisation d’accès et d’exécution, qui restent déterministes et contrôlées par du code. Les capacités ne peuvent jamais augmenter au fil du pipeline.

## Problème de sécurité résolu
Lock-Monotone réduit les risques suivants, sans dépendre du comportement interne du modèle :

- **Prompt injection** : la décision se fait sur une traduction structurée, pas sur le texte libre.
- **Confused deputy** : les accès sont décidés par un moteur de politiques externe au LLM.
- **Escalation par raisonnement** : les capacités restent monotones et ne peuvent pas s’élargir.

## Principe ACTION / KNOWLEDGE
Une requête utilisateur est traduite en un artefact structuré composé de deux sections :

- **ACTION** : l’effet demandé (procédure, cible, type de sortie).
- **KNOWLEDGE** : les sources ou catégories de connaissances nécessaires.

Le reste du pipeline traite uniquement cette traduction.

## Vue d’ensemble du pipeline
Pipeline strictement unidirectionnel :

1. Prompt utilisateur
2. LLM amont → traduction ACTION/KNOWLEDGE
3. Validation & canonicalisation déterministes
4. Policy Engine (décision fermée)
5. Accès connaissance contrôlé (connecteurs bornés)
6. LLM aval (génération contrainte)
7. Audit (traçabilité complète)

## Ce que Lock-Monotone est / n’est pas
**Lock-Monotone est :**

- Une architecture de sécurité déterministe pour LLM.
- Un mécanisme d’autorisation basé sur une traduction structurée.
- Un pipeline à capacités monotones.

**Lock-Monotone n’est pas :**

- Un modèle LLM « sécurisé » par magie.
- Un système qui fait confiance au raisonnement interne du modèle.
- Un mécanisme d’élévation automatique des droits.

## Mini exemple conceptuel
**Prompt** : « Donne-moi la procédure d’onboarding pour le compte de service Finance. »

**Traduction (IR)**

- ACTION : `procedure.read`
- KNOWLEDGE : `policies/hr/onboarding` (catégorie autorisée)

**Décision**

- Policy Engine : `ALLOW` pour le rôle `hr-analyst`
- Enveloppe de capacités réduite à `policies/hr/*`

**Réponse**

- LLM aval génère un résumé uniquement à partir des sources autorisées.

## Périmètre du PoC
Le PoC couvre :

- La définition de l’IR ACTION/KNOWLEDGE.
- La validation, la canonicalisation et la décision de politiques.
- Des connecteurs de connaissance bornés.
- L’audit de bout en bout.

Le PoC ne couvre pas :

- L’entraînement ou la modification d’un LLM.
- L’exécution d’actions sensibles en production.
