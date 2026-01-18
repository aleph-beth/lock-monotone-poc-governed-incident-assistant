# Architecture Lock-Monotone

## 1. Vue d’ensemble du pipeline
Pipeline strictement unidirectionnel :

1. Prompt utilisateur
2. LLM amont (traduction ACTION/KNOWLEDGE)
3. Validation & canonicalisation
4. Policy Engine
5. Accès connaissance contrôlé
6. LLM aval (génération contrainte)
7. Audit

## 2. Étapes détaillées

### 2.1 Prompt utilisateur
- **Entrée** : texte libre non fiable.
- **Sortie** : texte libre transmis au LLM amont.
- **Responsabilités** : collecte de la requête.
- **Interdictions** : aucune décision d’accès.

### 2.2 LLM amont (traduction)
- **Entrée** : prompt utilisateur.
- **Sortie** : IR ACTION/KNOWLEDGE au format JSON.
- **Responsabilités** : produire une traduction structurée.
- **Interdictions** : décider des droits, sélectionner des sources non allowlistées, exécuter des actions.

### 2.3 Validation & canonicalisation
- **Entrée** : IR brut.
- **Sortie** : IR validé et canonicalisé.
- **Responsabilités** :
  - Vérifier le schéma et les allowlists.
  - Normaliser les champs (`verb`, `output`, `sources`).
  - Rejeter tout champ interdit.
- **Interdictions** : accès à des données externes, inférence d’intention.

### 2.4 Policy Engine
- **Entrée** : IR validé, identité/role, contexte.
- **Sortie** : décision fermée + enveloppe de capacités.
- **Responsabilités** : appliquer la politique de sécurité.
- **Interdictions** : enrichissement de l’IR, accès direct aux données.

### 2.5 Accès connaissance contrôlé
- **Entrée** : décision + enveloppe de capacités.
- **Sortie** : données ou fragments autorisés.
- **Responsabilités** :
  - Interroger uniquement les sources autorisées.
  - Appliquer les contraintes (temps, sensibilité).
- **Interdictions** : accès hors enveloppe, agrégation non autorisée.

### 2.6 LLM aval (génération contrainte)
- **Entrée** : données autorisées + IR validé.
- **Sortie** : réponse finale.
- **Responsabilités** : synthétiser sans inventer de sources.
- **Interdictions** : accès à des données non fournies, décision de droits.

### 2.7 Audit
- **Entrée** : IR validé, décision, sources consultées, réponse.
- **Sortie** : journal immuable et horodaté.
- **Responsabilités** : traçabilité complète.
- **Interdictions** : modification a posteriori.

## 3. Frontières de confiance

### Non fiable
- LLM amont.
- Prompt utilisateur.
- LLM aval.

### Base de confiance
- Validateurs et canonicalisation.
- Policy Engine.
- Connecteurs de données contrôlés.
- Journal d’audit.

## 4. Propriété de monotonie

### Définition
À chaque étape, l’ensemble des capacités autorisées est identique ou réduit. Il ne peut jamais augmenter.

### Mécanismes d’assurance
- **Validation stricte** : rejette les champs hors schéma.
- **Policy Engine** : applique une intersection entre rôle, contexte et policy version.
- **Connecteurs** : n’exposent que les sources autorisées.
- **LLM aval** : ne reçoit que les données autorisées.

### Pourquoi aucune étape ne peut augmenter les capacités
- La seule étape qui décide est le Policy Engine, avec une décision fermée.
- Les étapes LLM n’ont aucun canal d’élévation des droits.
- Les connecteurs appliquent uniquement l’enveloppe reçue.
