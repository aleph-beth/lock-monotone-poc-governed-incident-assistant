# IR (Intermediate Representation) — ACTION / KNOWLEDGE

## 1. Rôle de la traduction

### Pourquoi on traduit
- Le prompt utilisateur est ambigu et non déterministe.
- Une traduction structurée permet un contrôle explicite des intentions et des sources.

### Pourquoi la sécurité travaille sur la traduction
- Les validateurs et la policy engine opèrent sur un schéma fermé.
- Aucun accès n’est décidé sur du texte libre.
- Le LLM ne possède aucun pouvoir d’autorisation.

## 2. Schéma formel de l’IR (JSON)

### Format général
```json
{
  "version": "1.0",
  "action": {
    "verb": "procedure.read",
    "target": "string",
    "output": "summary|extract|list"
  },
  "knowledge": {
    "sources": ["source_id"],
    "constraints": {
      "time_range": "P30D|P365D|ALL",
      "sensitivity": "public|internal|restricted"
    }
  },
  "context": {
    "locale": "fr-FR",
    "request_id": "uuid"
  }
}
```

### Champs obligatoires
- `version` : chaîne, version de schéma. Valeurs autorisées : `"1.0"`.
- `action.verb` : chaîne, verbe canonique allowlisté.
- `action.target` : chaîne, identifiant logique de la cible.
- `action.output` : chaîne, type de sortie autorisée.
- `knowledge.sources` : liste non vide de `source_id` allowlistés.

### Champs optionnels
- `knowledge.constraints.time_range`
- `knowledge.constraints.sensitivity`
- `context.locale`
- `context.request_id`

### Types et valeurs autorisées
**`action.verb` (allowlist fermée)**
- `procedure.read`
- `policy.read`
- `incident.summary`
- `asset.list`

**`action.output` (allowlist fermée)**
- `summary`
- `extract`
- `list`

**`knowledge.sources`**
- Format : `namespace/resource`.
- Exemples : `policies/hr`, `incidents/siem`, `assets/cmdb`.

**`knowledge.constraints.time_range`**
- `P30D`, `P365D`, `ALL`.

**`knowledge.constraints.sensitivity`**
- `public`, `internal`, `restricted`.

### Champs interdits
- Tout champ non défini par le schéma.
- Toute clé libre comme `tool`, `code`, `sql`, `shell`.
- Toute structure exécutable (scripts, requêtes arbitraires).

## 3. Séparation explicite

### Section `action`
Décrit **uniquement** l’effet demandé :
- Verbe canonique.
- Cible logique.
- Type de sortie.

### Section `knowledge`
Décrit **uniquement** les sources consultables :
- Identifiants de sources.
- Contraintes de fenêtre temporelle.
- Niveau de sensibilité.

Aucun champ ne doit apparaître dans la mauvaise section.

## 4. Contraintes de validation

### Allowlists
- `action.verb` limité à la liste fermée.
- `action.output` limité à la liste fermée.
- `knowledge.sources` limité aux connecteurs déclarés.

### Bornes de complexité
- `knowledge.sources` : 1 à 5 entrées.
- Taille maximale de l’IR sérialisée : 8 KB.
- Longueur `action.target` : 1 à 128 caractères.

### Interdictions formelles
- Aucune requête SQL libre.
- Aucun code exécutable.
- Aucun paramètre d’outil.
- Aucun champ non prévu par le schéma.

## 5. Exemples

### Exemples valides
**Exemple A**
```json
{
  "version": "1.0",
  "action": {
    "verb": "procedure.read",
    "target": "finance/service-account-onboarding",
    "output": "summary"
  },
  "knowledge": {
    "sources": ["policies/hr"],
    "constraints": {
      "time_range": "ALL",
      "sensitivity": "internal"
    }
  }
}
```

**Exemple B**
```json
{
  "version": "1.0",
  "action": {
    "verb": "incident.summary",
    "target": "INC-2024-0917",
    "output": "extract"
  },
  "knowledge": {
    "sources": ["incidents/siem"],
    "constraints": {
      "time_range": "P365D",
      "sensitivity": "restricted"
    }
  },
  "context": {
    "locale": "fr-FR",
    "request_id": "7b92c0e2-7c7d-4c9a-9a1b-9d2c8e3cbb81"
  }
}
```

### Exemples invalides
**Exemple invalide A — champ interdit**
```json
{
  "version": "1.0",
  "action": {
    "verb": "procedure.read",
    "target": "finance/service-account-onboarding",
    "output": "summary",
    "sql": "SELECT * FROM users"
  },
  "knowledge": {
    "sources": ["policies/hr"]
  }
}
```
Rejet : champ `sql` interdit, présence de requête libre.

**Exemple invalide B — verbe non allowlisté**
```json
{
  "version": "1.0",
  "action": {
    "verb": "admin.reset_password",
    "target": "user/jdoe",
    "output": "summary"
  },
  "knowledge": {
    "sources": ["assets/cmdb"]
  }
}
```
Rejet : `action.verb` non autorisé par la allowlist.
