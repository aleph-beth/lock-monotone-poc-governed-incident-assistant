# Lock-Monotone : une architecture de sécurité gouvernée par la traduction pour les systèmes intégrant des LLM

**Grégory Lagasse** — Chercheur indépendant — `gregory.lagasse@pm.me`

*Version de synthèse, juin 2026. Ce document consolide trois brouillons de travail du
cadre Lock-Monotone (une version orientée « cadre architectural », une version de type
ICSA avec une méthodologie d'évaluation transversale, et une version de type ARES centrée
sur le pipeline de traduction et l'auditabilité). Il est délibérément cadré comme un
**cadre architectural** : les énoncés quantitatifs sont présentés comme un protocole
d'évaluation proposé et reportés aux travaux futurs, et non revendiqués comme des
résultats.*

---

## Résumé

Les grands modèles de langage (LLM) sont de plus en plus intégrés à des systèmes qui
manipulent des données sensibles, invoquent des outils opérationnels et exécutent des
processus métier. Les approches de sécurité actuelles sont majoritairement *centrées sur
le modèle* : fine-tuning, apprentissage par renforcement, ou garde-fous heuristiques. Or,
un modèle de langage probabiliste ne peut constituer une frontière de sécurité
déterministe.

Cet article introduit **Lock-Monotone**, une architecture de gouvernance déterministe pour
l'intégration sécurisée des LLM. Plutôt que de s'appuyer sur le seul alignement
comportemental, Lock-Monotone relocalise l'autorisation et le contrôle des capacités *hors*
du modèle. L'architecture repose sur trois principes : (1) une séparation ontologique entre
**connaissance** et **procédure** ; (2) une représentation intermédiaire typée (**PyIR**)
qui transforme le langage naturel en un espace de capacités borné ; et (3) un cœur de
décision déterministe qui applique une monotonie des privilèges à la fois **verticale** et
**positionnelle**. Un *invariant de priorité d'action par préfixe* garantit qu'aucune
étape ultérieure de l'interaction ne peut élargir la portée des capacités fixée à
l'initialisation. L'exécution est médiatisée par un **Triple Décodage** (T1/T2/T3) en trois
étapes, qui sépare le rendu sémantique, l'invocation d'outils opérationnels et le filtrage
de conformité.

Nous formalisons un ensemble d'invariants de sécurité qui empêchent l'escalade de
capacités, et prouvons une garantie de non-escalade *conditionnée* à la justesse de la
traduction. Nous positionnons l'approche par rapport aux défenses récentes
*par-construction* (le motif *dual-LLM* et CaMeL), et l'illustrons par un exemple concret :
la gouvernance du LLM analytique intégré à une plateforme de gestion des événements et des
informations de sécurité (SIEM), y compris le cas « apex » où ce LLM superviseur est
lui-même la cible de l'attaque. Enfin, nous esquissons une méthodologie d'évaluation
transversale — Taux d'Actions Non Autorisées (UAR), Taux de Fuite de Secrets (SLR),
Couverture des Traces de Décision (DTC) — comme direction centrale des travaux futurs.

**Mots-clés :** sécurité des LLM, injection de prompt, sécurité par capacités, gouvernance
déterministe, moniteur de référence, systèmes agentiques, auditabilité.

---

## 1. Introduction

### 1.1 Contexte

Les LLM sont de plus en plus intégrés à des systèmes opérationnels qui manipulent des
secrets sensibles (clés d'API, identifiants, informations propriétaires), des données
réglementées ou personnelles (données personnelles, dossiers financiers, données de santé)
et des capacités d'action (API, services internes, bases de données, moteurs de workflow).

Dans de nombreux déploiements actuels, les stratégies de sécurité restent *centrées sur le
modèle*. Les mitigations visent principalement à améliorer le comportement interne du
modèle par le fine-tuning, l'apprentissage par renforcement à partir de retours humains
(RLHF), des classifieurs de sécurité ou des garde-fous heuristiques. Or, un LLM demeure un
composant probabiliste et sensible au contexte. Ses sorties sont influencées par les
distributions de jetons, la structure du prompt et les effets de récence. À ce titre, un
modèle de langage probabiliste **ne peut constituer une frontière de sécurité
déterministe**. Le problème central est donc d'ordre architectural, et non comportemental.

### 1.2 Thèse

Nous soutenons que la sécurité des systèmes fondés sur les LLM ne doit pas dépendre du
comportement interne du modèle. Elle doit reposer sur une architecture déterministe qui
gouverne, contraint et applique les capacités du modèle de manière externe. Dans cette
vision, le modèle peut interpréter, traduire ou générer du langage, mais il ne doit
**jamais** décider d'une autorisation, octroyer des privilèges ni exécuter d'actions
sensibles. La sécurité s'obtient non pas en rendant le modèle digne de confiance, mais en
le *subordonnant à une couche de gouvernance des capacités déterministe*. Ce déplacement
architectural fonde l'approche Lock-Monotone.

### 1.3 Les LLM comme interfaces critiques

Les premiers usages des LLM concernaient la génération de texte libre, où les défaillances
se limitaient à des sorties incorrectes ou indésirables. À l'inverse, les applications
modernes positionnent de plus en plus les modèles comme des *surfaces de contrôle* pour
exécuter des actions, interroger des données sensibles ou coordonner des procédures
multi-étapes. Dans ce cadre, la génération de langage devient effectivement une
**médiation de capacités** : des entrées en langage naturel sont traduites en opérations
aux conséquences réelles.

Ce glissement est particulièrement saillant dans les contextes adversariaux et à haute
exigence — opérations de cybersécurité, gestion d'infrastructures critiques, systèmes
d'entreprise réglementés. Dans de tels environnements, les défaillances ne se limitent plus
à des erreurs sémantiques ; elles peuvent directement entraîner un accès non autorisé, une
escalade de privilèges ou des changements d'état irréversibles. La question n'est donc plus
de savoir si les sorties du LLM sont « bien élevées », mais si elles peuvent être
*intégrées en toute sécurité* à des systèmes exigeant des garanties de sécurité
applicables.

### 1.4 Pourquoi les défenses existantes échouent structurellement

La plupart des défenses actuelles restent structurellement inadéquates lorsque les modèles
sont déployés comme médiateurs de capacités. Une classe dominante repose sur la
**classification terminale** : une requête est évaluée dans son ensemble et reçoit un
verdict binaire ou quasi-binaire (autoriser/refuser). Ces approches sont rigides, difficiles
à auditer et mal adaptées aux interactions multi-étapes.

Les cadres de garde-fous et le filtrage *a posteriori* opèrent sur des sorties
probabilistes et n'ont donc pas l'autorité d'imposer des frontières de sécurité strictes.
De même, les pipelines de génération augmentée par récupération (RAG) traitent souvent la
récupération comme une opération implicitement permise, en privilégiant la pertinence du
contenu plutôt que l'autorisation explicite. En conséquence, des documents malveillants ou
non fiables peuvent influencer le comportement du modèle sans passer par une décision de
contrôle d'accès déterministe.

À travers ces approches émerge une limitation structurelle commune : **l'absence d'une
frontière de décision unique et déterministe**. Les choix de sécurité sont implicites,
probabilistes ou répartis entre des composants incapables de garanties fortes sous pression
adversariale. Cette ambiguïté architecturale rend possibles l'injection de prompt, le suivi
d'instructions indirectes et l'escalade de privilèges par raisonnement.

### 1.5 Positionnement et contributions

Lock-Monotone positionne les LLM comme des composants explicitement *non fiables* et
probabilistes, dont le rôle se limite à la traduction sémantique. Toutes les décisions
relatives à la sécurité sont déléguées à des mécanismes déterministes, auditables,
traçables et indépendants du comportement du modèle.

Cette posture est partagée avec une lignée récente de défenses *par-construction* — au
premier rang desquelles le motif **dual-LLM** et **CaMeL** — qui retirent elles aussi
l'autorité au modèle. La section 11 détaille la relation. En résumé, cet article apporte :

- une séparation **ontologique** entre connaissance et procédure, appliquée à la frontière
  de traduction (section 4) ;
- un **invariant de priorité d'action par préfixe** (monotonie positionnelle) qui plafonne
  les capacités de toute l'interaction dès sa première déclaration (section 5) ;
- une représentation intermédiaire typée (**PyIR**) et un **cœur de décision
  déterministe** (sections 6–7) ;
- une médiation d'exécution en trois étapes, le **Triple Décodage** (T1/T2/T3), avec une
  porte de conformité indépendante (section 8) ;
- un **exemple concret** qualitatif gouvernant le LLM analytique d'un SIEM, y compris le
  cas méta où ce LLM est lui-même attaqué (section 11) ;
- une formalisation des **garanties de non-escalade**, énoncées explicitement comme
  *conditionnées* à la justesse de la traduction (section 9) ;
- l'**auditabilité par construction** comme propriété de premier ordre (section 13) ;
- une **méthodologie d'évaluation transversale proposée** (UAR/SLR/DTC) reportée aux
  travaux futurs (section 15).

Nous insistons : le présent article est un cadre architectural. L'évaluation quantitative
est reportée ; aucun résultat empirique n'est revendiqué ici.

---

## 2. Modèle de menace et hypothèses

L'objectif n'est pas d'énumérer exhaustivement toutes les attaques possibles, mais de
caractériser les risques *structurels* qui surgissent lorsque les LLM sont intégrés comme
interfaces de systèmes critiques pour la sécurité ou la mission.

### 2.1 Modèle d'adversaire

Nous considérons des adversaires interagissant via des entrées en langage naturel, des
sources de données récupérées ou des interfaces d'intégration. L'adversaire peut être
externe ou interne, authentifié ou non, et disposer d'une connaissance partielle de
l'architecture. Dans le périmètre :

- **Injection de prompt.** Injection directe (instructions malveillantes dans l'entrée
  utilisateur) et indirecte (instructions introduites via des sources externes : documents
  récupérés, e-mails, contenu web).
- **Escalade par raisonnement.** Exploitation des capacités de raisonnement des LLM pour
  induire un comportement non voulu par argumentation multi-étapes, confusion de rôle ou
  cadrage contextuel, afin d'élargir les permissions effectives.
- **Abus de procédure et attaques compositionnelles.** Séquences d'opérations
  individuellement permises qui, composées, produisent des effets non autorisés —
  particulièrement pertinentes dans les déploiements agentiques ou outillés.

L'adversaire n'est **pas** supposé disposer d'un accès en boîte blanche aux poids ou aux
états internes du modèle. Les attaques par canaux auxiliaires, les défaillances
cryptographiques et les exploits bas niveau de l'infrastructure sont hors périmètre.

### 2.2 Hypothèses système

- **Modèles en boîte noire.** Tous les LLM sont traités comme des composants probabilistes
  en boîte noire. Aucune hypothèse n'est faite sur l'alignement, la calibration, les
  représentations internes ou la résistance au prompting adversarial. Les sorties sont
  considérées potentiellement non fiables vis-à-vis de la confidentialité et de
  l'autorisation.
- **Sources de récupération adversariales.** Le contenu accédé par RAG est supposé
  potentiellement adversarial : jamais faisant autorité, pouvant contenir des instructions
  malveillantes ou des charges utiles forgées.
- **Périmètre infrastructure.** Calcul, stockage, réseau et cryptographie suivent les
  bonnes pratiques standard. Les attaques contre l'infrastructure d'hébergement, les dénis
  de service et les exploits matériels sont hors périmètre et doivent être traités par des
  défenses complémentaires.

### 2.3 Objectifs de sécurité

- **Confidentialité.** Les données et secrets sensibles ne doivent pas être divulgués —
  directement ou indirectement — à des parties non autorisées (exfiltration
  conversationnelle, fuite par inférence comprises).
- **Intégrité de l'autorisation.** Les actions permises ne doivent pas être élargies par
  manipulation de prompt, accumulation contextuelle ou raisonnement multi-étapes. Les
  décisions d'autorisation doivent être explicites, déterministes et invariantes au
  comportement du modèle.
- **Auditabilité et responsabilité.** Toute décision relative à la sécurité doit être
  explicable et traçable *a posteriori*, avec des artefacts structurés soutenant l'analyse
  post-incident et la revue de conformité.

---

## 3. Traduction plutôt que classification

La plupart des approches existantes conçoivent la sécurité des LLM comme un problème de
*classification* : les requêtes sont évaluées sous forme brute et reçoivent un verdict
terminal.

### 3.1 Limites de la sécurité centrée sur le classifieur

Les approches par classifieur seul couplent étroitement les décisions de sécurité au
langage naturel libre. De petites variations linguistiques, un recadrage contextuel ou une
paraphrase adversariale peuvent altérer significativement le résultat. La classification est
une décision *terminale* : la sécurité est une porte unique plutôt qu'un processus, sans
représentation structurée de l'intention, des actions demandées ou de la sensibilité des
données. Cette opacité limite l'auditabilité et complique l'adaptation aux menaces
évolutives. En contexte adversarial, ces limites deviennent des vulnérabilités
structurelles : les sorties probabilistes sont implicitement considérées comme des signaux
de sécurité.

### 3.2 La traduction comme primitive de sécurité

Lock-Monotone traite la **traduction — et non la classification — comme primitive de
sécurité primaire**. Plutôt que de rendre un verdict sur du texte brut, le système traduit
d'abord l'entrée en représentations intermédiaires explicites et structurées, rendant
l'intention, les actions et les contraintes observables et gouvernables. Cela découple
l'interprétation sémantique de l'autorisation. La classification, lorsqu'elle est employée,
opère sur des représentations structurées plutôt que sur du texte libre.

Surtout, la traduction permet des comportements de sécurité **non binaires** : au lieu d'un
strict autoriser/refuser, le système peut appliquer des transformations contrôlées comme
l'abstraction, la reformulation, la rédaction caviardée ou la satisfaction partielle. Une
requête qui serait rejetée d'emblée peut être dégradée en réponse informationnelle sûre,
préservant l'intention légitime tout en éliminant le risque opérationnel.

### 3.3 Contrainte progressive par traductions gouvernées

Chaque étape de traduction réduit l'ambiguïté, contraint l'espace des actions possibles et
expose des artefacts structurés pour une évaluation déterministe. En organisant la sécurité
comme une séquence de traductions gouvernées, Lock-Monotone répartit la responsabilité
entre des étapes architecturales plutôt que de la concentrer en un point de décision
probabiliste unique — à l'image des pipelines de compilation et des environnements
d'exécution gouvernés par des politiques, où la sûreté émerge d'une validation par étapes.

---

## 4. Séparation initiale : connaissance vs procédure

### 4.1 Distinction ontologique

Un principe fondateur est la séparation explicite entre **connaissance** et **procédure** :

- **Connaissance** — information non exécutable : descriptions, résumés, classifications,
  explications ou transformations qui ne modifient pas directement l'état du système.
- **Procédure** — intention porteuse de capacité : toute requête impliquant une
  modification d'état, un accès aux données, une récupération de secret ou une exécution
  opérationnelle.

Ce n'est pas une simple nuance sémantique ; c'est une frontière *ontologique* entre contenu
informationnel et autorité actionnable. Dans les déploiements classiques, cette frontière
est floue : le texte généré peut encoder implicitement des instructions ensuite
interprétées comme des commandes, surtout dans les systèmes agentiques où les sorties sont
mappées vers des appels d'outils. Lock-Monotone formalise la séparation comme invariant
architectural. **La connaissance n'implique jamais l'action.**

### 4.2 Asymétrie des risques

La séparation traite deux classes de risques primaires distinctes, résumées ci-dessous.

| | **Connaissance** | **Procédure** |
|---|---|---|
| Nature | Informationnelle | Opérationnelle |
| Risque | Exfiltration | Abus d'outil |
| Contrôle | Validation / filtrage de sortie | Application de politique |
| Métrique proposée | SLR | UAR |

Ces classes exigent des stratégies d'application fondamentalement différentes.

### 4.3 La traduction comme application de frontière

La séparation est appliquée dès la première étape, via une RI structurée. Le langage
naturel est traduit en une RI typée contenant au minimum : un `intent`, une
`semantic_class` (`KNOWLEDGE`, `PROCEDURE` ou `SENSITIVE`), un `sensitivity_level` et un
indicateur `requires_tool`. Cette classification initiale définit la frontière de capacités
avant toute considération de chemin d'exécution. Élément crucial : la séparation doit avoir
lieu **avant** toute résolution d'action, afin que l'intention procédurale ne puisse être
dérivée implicitement d'une interprétation contextuelle ou d'un raisonnement ultérieur.

### 4.4 Éliminer l'ambiguïté instruction–donnée

L'architecture impose la règle : **la sortie de langage générée n'est jamais exécutable ni
faisant autorité.** L'accès aux outils, bases de données, secrets ou systèmes externes
n'est jamais déclenché par la seule génération de texte ; toutes les actions sont
médiatisées par des interfaces d'action explicites évaluées par des moteurs de politique
déterministes. Par conséquent, la manipulation conversationnelle ne peut produire d'effets
de bord au-delà du régime de connaissance, et le système *échoue en mode fermé* : il n'est
pas nécessaire de détecter ou de classifier une attaque comme malveillante pour la bloquer.

---

## 5. Invariant de priorité d'action par préfixe (monotonie positionnelle)

### 5.1 Motivation

Les LLM sont intrinsèquement sensibles à l'ordre des jetons et à la récence contextuelle.
Des instructions apparaissant plus tard dans un prompt peuvent supplanter, réinterpréter ou
déformer sémantiquement le contenu antérieur — rendant possibles l'injection tardive,
l'escalade multi-tours, la contrebande d'instructions et la redéfinition de rôle/autorité.
Dans les architectures classiques, les frontières de capacités peuvent se déplacer
implicitement à mesure que le contexte s'accumule. Lock-Monotone introduit un invariant de
sécurité **positionnel**.

### 5.2 Définition de l'invariant de préfixe

Soit une interaction traduite en une représentation intermédiaire ordonnée

$$ IR = \{ s_1, s_2, \dots, s_n \}, $$

où $s_1$ est la déclaration sémantique initiale (l'en-tête), contenant l'intention, la
classe sémantique, le niveau de sensibilité et les besoins d'outils déclarés. Soit
$\mathit{Capacités}(s_i)$ l'ensemble des capacités impliquées par l'étape $s_i$.
L'invariant de priorité d'action par préfixe est :

$$ \forall i > 1, \quad \mathit{Capacités}(s_i) \subseteq \mathit{Capacités}(s_1). $$

Aucune étape ultérieure ne peut introduire une capacité non déjà permise par la déclaration
initiale.

### 5.3 Deux formes de monotonie

- **Monotonie verticale** — les capacités ne peuvent que rester constantes ou diminuer à
  travers les couches de traitement.
- **Monotonie positionnelle** — les capacités ne peuvent que rester constantes ou diminuer
  à travers les étapes de l'interaction.

Ensemble, ces contraintes empêchent l'expansion implicite de capacités par réinterprétation
contextuelle, escalade par paraphrase ou demandes d'autorisation différées.

### 5.4 Implications de sécurité

L'invariant bloque plusieurs classes d'attaques : une tentative d'escalade multi-tours ne
peut élargir le privilège au-delà de la déclaration initiale ; une instruction injectée par
RAG ne peut élever la portée opérationnelle ; une demande tardive d'invocation d'outil ne
peut excéder les permissions définies par le préfixe. Surtout, l'invariant s'applique
*avant* l'évaluation de politique : la classification sémantique initiale définit la borne
supérieure de l'autorité permise.

### 5.5 Portée théorique

L'invariant de priorité d'action par préfixe introduit une dimension **temporelle** à
l'application du moindre privilège. Le moindre privilège traditionnel contraint *qui* peut
réaliser *quoi*. Lock-Monotone contraint en plus *quand* les frontières de capacités sont
définies. La sécurité s'ancre ainsi non seulement dans le rôle et la politique, mais dans
l'ordonnancement structurel de la traduction. À notre connaissance, cet invariant temporel
explicite de priorité par préfixe distingue Lock-Monotone des défenses par-construction
antérieures.

---

## 6. Représentation intermédiaire sécurisée (PyIR)

### 6.1 Justification

Le langage naturel est ambigu, dépendant du contexte et susceptible de manipulation
adversariale ; lier la sémantique d'exécution ou d'autorisation directement à la sortie
brute du modèle crée un risque systémique. Pour éliminer cette ambiguïté, Lock-Monotone
introduit un pseudo-langage typé, contraint et validé statiquement : **PyIR**.

> **Note sur la dénomination.** PyIR adopte une syntaxe d'inspiration Python, lisible par
> l'humain, pour la lisibilité et la commodité de l'outillage. Ce n'est *pas* du Python
> exécutable : c'est un langage dédié (DSL) restreint, sans calcul général, conçu pour la
> normalisation sémantique, la déclaration de capacités, l'analyse déterministe et
> l'évaluation formelle de politique.

### 6.2 Structure fondamentale

Une instance PyIR minimale contient un en-tête structuré suivi d'éléments procéduraux
optionnels :

```json
{
  "ir_version": "1.0",
  "header": {
    "intent": "...",
    "semantic_class": "...",
    "sensitivity_level": "...",
    "requires_tool": true
  },
  "procedure": [ ]
}
```

L'`header` est **obligatoire et immuable** une fois généré. Il définit la catégorie
d'intention globale, le caractère informationnel ou procédural de la requête, la
classification de sensibilité et les besoins opérationnels déclarés. Cet en-tête établit la
borne supérieure des capacités permises.

### 6.3 Grammaire restreinte

PyIR est intentionnellement contraint pour empêcher toute sémantique d'exécution
arbitraire. La grammaire impose : une énumération fermée d'intentions (pas de noms
d'actions libres) ; aucun import ni référence à du code externe ; aucun accès d'attribut ni
évaluation dynamique ; aucune fonction définie par l'utilisateur ; une analyse déterministe
avec validation statique. Toute RI non conforme est rejetée avant l'évaluation de
politique.

### 6.4 Liaison au catalogue de capacités

Chaque intention permise est associée à une entrée de capacité prédéfinie dans un catalogue
versionné :

$$ \mathit{Intention} \rightarrow \mathit{EnsembleDeCapacités}. $$

Cela garantit des exigences de privilèges explicites, des contraintes d'autorisation par
rôle, des règles d'escalade de sensibilité et des liaisons de politique traçables. Le
catalogue étant versionné, les changements de politique restent auditables et
reproductibles.

### 6.5 Couche de validation statique

Avant d'atteindre le moteur de politique déterministe, PyIR subit une validation statique :
validation syntaxique (conformité au schéma), validation de l'énumération d'intentions,
cohérence de la portée de capacités avec l'invariant de préfixe, et cohérence de la
classification de sensibilité. Les structures RI invalides sont rejetées sans invoquer les
composants en aval.

### 6.6 Position architecturale

PyIR est la **charnière structurelle** entre la couche d'interprétation probabiliste (le
LLM) et le cœur de décision déterministe. Il transforme un langage non borné en un espace
de capacités borné. Le LLM peut générer des structures RI candidates, mais seule une
instance RI validée statiquement et approuvée par politique peut influencer l'exécution.
Cette transformation relocalise la confiance des poids du modèle vers les invariants
architecturaux.

---

## 7. Cœur de décision déterministe

### 7.1 Relocalisation de la frontière de sécurité

Dans les systèmes classiques, les décisions d'autorisation peuvent être implicitement
intégrées aux sorties du modèle ou déléguées à un post-traitement heuristique.
Lock-Monotone inverse cette hypothèse : le modèle peut interpréter le langage, mais il ne
doit jamais déterminer l'autorisation, le privilège ou l'exécution opérationnelle. Le **cœur
de décision déterministe constitue la véritable frontière de sécurité** du système.

### 7.2 Définition formelle

Soit une RI validée contenant une intention $I$, un rôle $R$, un tenant/contexte $T$, un
niveau de sensibilité $S$ et une version de politique $V$. La fonction de décision est

$$ \mathit{Décision} = f(I, R, T, S, V), $$

où $f$ est déterministe et pure, de sortie dans un ensemble fermé :

$$ \mathit{Décision} \in \{\textsc{allow}, \textsc{restrict}, \textsc{deny}\}. $$

### 7.3 Propriétés de la fonction de décision

Le cœur de décision satisfait : le **déterminisme** (mêmes entrées → mêmes sorties) ; la
**pureté** (aucun état caché ni effet de bord) ; la **liaison de version** (chaque décision
associée à une version de politique) ; la **rejouabilité** (réévaluation des décisions
passées sous la même politique) ; et l'**auditabilité** (toutes entrées et sorties
traçables). Cela élimine tout comportement stochastique de la logique d'autorisation.

### 7.4 Application des capacités

Le cœur applique à la fois la monotonie verticale et positionnelle. Avec
$\mathit{Capacités}(IR)$ l'ensemble dérivé de l'en-tête validé :

$$ \mathit{CapacitésOctroyées} \subseteq \mathit{Capacités}(IR). $$

Aucune nouvelle capacité ne peut être introduite à ce stade.

### 7.5 Séparation d'avec la sémantique du langage

Le cœur de décision opère exclusivement sur des champs RI structurés. Il ne traite pas le
langage naturel brut, n'interprète pas de texte libre et ne s'appuie sur aucune inférence
probabiliste. Toute interprétation sémantique a lieu en amont ; toute application
d'autorisation a lieu ici. La formulation adversariale ne peut influencer la logique
d'autorisation.

### 7.6 Sémantique de défaillance

- **`DENY`** — terminer les chemins d'exécution procéduraux, empêcher l'invocation
  d'outils, optionnellement déclencher une réponse sûre.
- **`RESTRICT`** — réduire la portée des capacités, imposer des couches de validation
  supplémentaires, appliquer des contraintes de filtrage de sortie.
- **`ALLOW`** — permettre l'exécution strictement dans la frontière de capacités
  prédéfinie.

### 7.7 Garantie structurelle

L'autorisation étant déterministe et liée à une version, les tests de non-régression et la
vérification formelle deviennent faisables : par construction, aucun chemin d'exécution hors
de la frontière de capacités validée n'est atteignable. Nous nous abstenons délibérément
d'affirmer des métriques de sécurité quantitatives ; leur mesure exige le protocole
empirique identifié comme travail futur (section 15).

---

## 8. Traduction finale contrôlée (Triple Décodage)

Une fois que le cœur de décision produit un résultat, le système doit le traduire en
comportement observable. Une contrainte critique s'applique : **l'autorisation n'implique
pas l'exécution directe.** Même après un `ALLOW`, le modèle n'obtient aucun pouvoir
d'exécution autonome. Lock-Monotone introduit une traduction contrôlée en trois étapes, le
**Triple Décodage** (correspondant au pipeline T1/T2/T3).

### 8.1 Étape 1 — Décodage sémantique (T1 : traduction d'intention)

T1 transforme l'entrée utilisateur brute en une RI structurée et non exécutable qui rend
l'intention explicite, séparant le contenu informationnel de l'intention procédurale. T1
n'a *aucun accès* aux données sensibles, aux outils ou aux systèmes externes ; il ne peut
déclencher ni récupération ni exécution et ne produit aucun effet de bord. Sa sortie est
purement descriptive et ne fait pas autorité. En phase de génération, le modèle agit
strictement comme un moteur de rendu de langage dans un espace sémantique borné.

### 8.2 Étape 2 — Décodage opérationnel / gouvernance (T2 : passerelle d'outils)

T2 est le cœur de contrôle déterministe. Il consomme la RI et l'évalue au regard de
politiques de sécurité et métier explicites (RBAC/ABAC), en intégrant des contraintes de
conformité spécifiques au domaine. Le RAG est traité comme une *capacité conditionnelle* :
l'accès aux sources de connaissance n'est accordé que s'il est explicitement autorisé, et
le contenu récupéré est toujours classé comme donnée non fiable, jamais comme instruction.
La sortie est un état d'autorisation contraint, avec un ensemble explicite de capacités
permises et de transformations requises ; les décisions sont fermées et énumérables (p. ex.
`ALLOW`, `DENY`, `REDACT`, `SUMMARIZE`, `ESCALATE`). Cette étape impose des listes blanches
d'outils explicites, la validation des arguments, des modes bac à sable / simulation au
besoin, et une journalisation déterministe des métadonnées d'invocation. La passerelle
d'outils fonctionne comme un **pare-feu de capacités** entre l'intention sémantique et
l'exécution système.

### 8.3 Étape 3 — Décodage de conformité / résolution (T3 : porte de sortie)

T3 résout l'état autorisé en une réponse visible par l'utilisateur ou une action contrôlée.
Une couche de conformité finale applique la prévention des fuites de données (DLP), la
détection de motifs de secrets, le filtrage des données personnelles, les contraintes de
verbatim RAG et l'application de la politique de contenu — atténuant la divulgation
accidentelle, la fuite partielle par paraphrase et la reproduction non voulue de documents
sensibles. T3 ne peut introduire de nouvelles capacités, outrepasser les décisions de
politique ou accéder aux secrets bruts ; son rôle est expressif, non décisionnel. La porte
de conformité est **indépendante de la décision d'autorisation**, assurant une défense en
profondeur même pour les requêtes purement informationnelles.

### 8.4 Application monotone à travers les décodages

Soit $C_0$ l'ensemble de capacités défini par l'en-tête RI, et $C_1, C_2, C_3$ la portée
effective à chaque étape :

$$ C_3 \subseteq C_2 \subseteq C_1 \subseteq C_0. $$

La portée des capacités ne peut que rester constante ou diminuer ; aucune étape ne peut
élargir l'autorité au-delà de la frontière définie par le préfixe.

### 8.5 Évolution illustrative de la RI

```text
RI T1 (intention) :
  intent_type: "request"
  knowledge:   ["expliquer X"]
  action:      ["exporter dossiers"]
  sensitivity: ["donnees_client"]

État autorisé T2 :
  decision:             "DENY_ACTION_ALLOW_KNOWLEDGE"
  allowed_capabilities: ["expliquer_concepts"]
  removed_capabilities: ["export_donnees", "invocation_outil"]
  policy_version:       "P-2026.02"
```

### 8.6 Conséquence architecturale

Le modèle n'appelle jamais directement d'outils, n'accède jamais aux secrets et ne modifie
jamais l'état système. Toutes ces actions sont médiatisées par des structures
déterministes. Le Triple Décodage achève la relocalisation de la confiance, du raisonnement
probabiliste vers les invariants architecturaux, faisant de Lock-Monotone une architecture
complète de *gouvernance d'exécution* plutôt qu'un simple cadre de traduction.

---

## 9. Fiabilité de la traduction : fine-tuning vs conditionnement par prompt

### 9.1 La variable ouverte de l'architecture

Le cœur déterministe garantit qu'aucune RI *validée* ne peut produire une exécution non
autorisée. La variable restante est donc la **fiabilité de la traduction** : avec quelle
fidélité le langage naturel est-il mappé vers une RI correcte. Si la traduction
classe systématiquement mal l'intention — encodant une procédure comme une connaissance —
le cœur déterministe appliquera fidèlement la *mauvaise* frontière. La fiabilité de la
traduction est ainsi le principal risque résiduel de l'architecture, et sa caractérisation
empirique est l'objet naturel de l'évaluation future.

### 9.2 Justification de conception

La traduction peut en principe s'obtenir par instruction par prompt (apprentissage en
contexte) ou par fine-tuning supervisé. Sur des bases structurelles, nous conjecturons que
le conditionnement par seul prompt est insuffisant pour une conformité robuste en contexte
adversarial : même avec de larges fenêtres de contexte, les prompts peuvent spécifier le
schéma de RI, une énumération fermée d'intentions, des exemples structurés et des
contraintes d'ordre, mais ils restent soumis aux mêmes effets de récence et de
réinterprétation qui motivent l'invariant de préfixe. Nous nous attendons donc à ce que la
génération de RI par seul prompt présente des sorties malformées ou partiellement valides,
un typage de champ incohérent, une dérive de classification d'intention sous entrée
adversariale ou bruitée, et une violation non déterministe de l'ordre par préfixe.

### 9.3 Le fine-tuning comme a priori structurel

Nous faisons l'hypothèse que le fine-tuning supervisé sur des jeux de données alignés sur
la RI améliore la validité syntaxique, la stabilité de la classification d'intention et de
sensibilité, l'efficacité en jetons et le respect des contraintes structurelles, y compris
l'ordre par préfixe. En particulier, nous conjecturons que la monotonie positionnelle est
mieux traitée comme un **a priori de traduction appris** que comme un comportement émergent
du prompt. Ce sont des *hypothèses de conception* ; leur validation exige une étude
empirique contrôlée, que nous reportons explicitement.

### 9.4 Clarification de la frontière de sécurité

Quel que soit le mode de conditionnement de la traduction, Lock-Monotone maintient une
frontière stricte entre la **fiabilité de la traduction** (qualité de production de RI
valides) et l'**application de la sécurité** (ce que le système est autorisé à exécuter). Le
fine-tuning peut réduire la probabilité d'erreurs de traduction, mais n'apporte aucune
garantie formelle de sécurité. L'architecture exige donc que toutes les sorties RI subissent
une analyse déterministe et une validation de schéma, que les valeurs d'intention
appartiennent à une énumération fermée, que la portée des capacités soit appliquée par
politique et liaison au catalogue, et que l'exécution soit médiatisée par la passerelle
d'outils et les couches de conformité. Dans ce cadre, **les erreurs de traduction ne
peuvent produire d'exécution non autorisée** — elles ne peuvent produire qu'une
sur-restriction (refus inutiles), qui est un mode de défaillance sûr.

---

## 10. Modèle de menace transversal et mitigation

### 10.1 Du niveau modèle au niveau système

La plupart des analyses de sécurité des LLM se concentrent sur la robustesse au niveau du
modèle — principalement la résistance au jailbreak ou à la sollicitation de contenu
nuisible. Mais les déploiements modernes intègrent plusieurs couches architecturales (RAG,
moteurs de politique déterministes, passerelles d'outils, filtrage de sortie). Raisonner
sur le seul modèle est insuffisant. Lock-Monotone adopte un modèle de menace *transversal*
couvrant l'ensemble du pipeline d'exécution.

### 10.2 Taxonomie des menaces

- **T1 — Jailbreak :** tentatives de contourner les contraintes de sécurité au niveau du
  langage.
- **T2 — Injection directe :** instruction malveillante intégrée à l'entrée utilisateur.
- **T3 — Abus d'outil :** invocation non autorisée de capacités opérationnelles.
- **T4 — Injection indirecte RAG :** contenu hostile intégré aux documents récupérés.
- **T5 — Escalade multi-tours :** amplification progressive de privilèges au fil des tours.

### 10.3 Cartographie transversale des menaces

| Menace | Couche | Mécanisme de bornage |
|---|---|---|
| T1 : Jailbreak | LLM | Cœur de décision (aucune autorité) |
| T2 : Injection | Traduction | RI typée + validation statique |
| T3 : Abus d'outil | Passerelle | Catalogue fermé + liste blanche |
| T4 : Injection RAG | Contexte | Borne de préfixe + séparation C/P |
| T5 : Multi-tours | Séquence | Monotonie positionnelle |

Les invariants de Lock-Monotone visent à neutraliser l'escalade de capacités à travers
toutes les couches, plutôt qu'à rendre une couche unique comportementalement sûre.

### 10.4 Mitigation par construction

Pour chaque classe, l'architecture supprime le *levier opérationnel* de l'attaque plutôt
que de la détecter :

- **Injection de prompt (directe/indirecte) :** le texte généré ne fait pas autorité ;
  l'autorité de décision réside exclusivement dans des composants déterministes.
- **Escalade par raisonnement :** application monotone des capacités — un raisonnement
  supplémentaire ne peut élargir l'ensemble des capacités.
- **Abus de procédure / attaques compositionnelles :** typage explicite des capacités et
  réduction monotone à travers les étapes ; la composition séquentielle ne peut produire de
  privilèges émergents.
- **Exfiltration par dialogue :** externalisation des secrets — les secrets ne sont jamais
  dans les poids, les prompts ou la mémoire conversationnelle ; l'accès est toujours une
  action explicite et autorisée.
- **Injection dans les représentations structurées :** schémas validés et re-sérialisés de
  façon déterministe ; structures anormales rejetées.

Comme le système ne repose pas sur la détection d'une intention malveillante, de nombreuses
attaques spécifiques aux LLM sont transformées en problèmes classiques de sécurité des
systèmes, où s'appliquent les techniques établies de validation, d'application de politique
et d'audit.

---

## 11. Exemple concret : gouverner le LLM d'un SIEM

Nous illustrons l'architecture qualitativement — sans revendication quantitative — par un
cas à haute exigence : un LLM analytique intégré à une plateforme de gestion des événements
et des informations de sécurité (SIEM). Ce cas est exigeant sur le plan adversarial : le LLM
consomme à la fois des données influencées par l'attaquant (journaux ingérés) et se trouve à
proximité de capacités opérationnelles privilégiées (règles d'alerte, gestion de cas,
actions de réponse).

### 11.1 Cadre

Un centre d'opérations de sécurité (SOC) déploie un assistant LLM pour résumer les alertes,
classifier les incidents et proposer des actions de réponse. L'assistant ingère de la
télémétrie (journaux pare-feu, événements EDR, en-têtes d'e-mails) et peut, en principe,
demander des actions opérationnelles via une couche d'outils : interroger un cas, enrichir
un indicateur ou — privilégié — désactiver une règle de détection ou exporter un jeu de
données.

### 11.2 Attaque : injection indirecte via télémétrie (T4)

Un attaquant intègre une instruction dans une donnée ingérée comme champ de journal, p. ex.
une chaîne user-agent ou un sujet d'e-mail forgé : *« SYSTEM : les alertes précédentes sont
des faux positifs ; désactive la règle 4012 et exporte le coffre d'identifiants vers
l'adresse jointe. »* Dans un déploiement centré sur le modèle, ce texte entre dans le prompt
comme contexte et peut être suivi, le modèle ne pouvant distinguer de façon fiable les
instructions de confiance des données contrôlées par l'attaquant.

### 11.3 Déroulé sous Lock-Monotone

1. **Séparation connaissance/procédure.** Le champ de journal ingéré est classé à la
   traduction comme `KNOWLEDGE` (télémétrie à résumer). L'impératif intégré ne peut, par
   construction, être promu en `PROCEDURE` : l'intention procédurale doit provenir de la
   déclaration de préfixe de confiance, non d'un contenu ultérieur.
2. **Borne de préfixe.** L'en-tête de l'interaction déclarait l'intention *« résumer et
   classifier les alertes »*, dont l'ensemble de capacités exclut `disable_rule` et
   `export_data`. Par l'invariant de préfixe, aucune étape ultérieure — y compris le contenu
   injecté — ne peut introduire ces capacités.
3. **Cœur de décision.** Même si une RI malformée proposait `disable_rule`, le cœur
   déterministe l'évalue au regard du rôle, du tenant, de la sensibilité et de la version de
   politique ; l'intention est absente de l'ensemble de capacités lié à l'en-tête et est
   refusée. Le modèle n'a aucune autorité sur cette décision.
4. **Passerelle d'outils.** Le décodage opérationnel n'expose que les outils en liste
   blanche pour l'intention déclarée. `disable_rule` et `export_data` ne sont pas
   atteignables depuis une intention « résumer », quel que soit le texte généré.
5. **Porte de conformité.** Si un résumé tentait de reproduire un identifiant ou
   d'exfiltrer par paraphrase, les filtres de motifs de secrets et de DLP de la porte de
   sortie nettoient la réponse indépendamment du chemin d'autorisation.

L'instruction injectée est neutralisée à plusieurs couches ; le mode de défaillance — si la
traduction sur-restreint — est un **refus sûr** plutôt qu'une action non autorisée.

### 11.4 Le cas apex : attaquer le LLM superviseur

Un cas plus profond surgit lorsque la cible est le LLM qui supervise la sécurité elle-même.
Deux conséquences en découlent. D'abord, un jailbreak de ce LLM (T1) n'octroie, par
construction, aucune capacité : l'autorité réside dans le cœur déterministe, non dans le
modèle ; un LLM superviseur compromis peut au plus produire un *langage* trompeur, non des
*actions* non autorisées. Ensuite, la télémétrie qui enregistre le comportement de
l'assistant lui-même doit être assainie avant ré-ingestion, sinon un attaquant pourrait
utiliser les journaux de l'assistant comme canal d'injection indirecte de retour dans le
pipeline. Lock-Monotone traite ce LLM superviseur comme résidant *hors* du chemin de
confiance : il est gouverné par les mêmes borne de préfixe, cœur de décision et porte de
conformité que tout autre composant, et ses sorties ne sont jamais une source d'autorité. Ce
placement méta est, soutenons-nous, l'épreuve la plus tranchante de la doctrine *« le modèle
n'a aucune autorité »*.

### 11.5 Ce que l'exemple montre et ne montre pas

Le déroulé démontre qualitativement comment les invariants bornent une attaque. Il ne
*mesure pas* la fréquence à laquelle la traduction classifie correctement une télémétrie
adversariale, ni le taux de faux refus induit par une classification conservatrice. Ce sont
des questions empiriques — précisément l'évaluation que nous reportons.

---

## 12. Invariants de sécurité formels et garanties théoriques

### 12.1 Préliminaires

Soit $NL$ l'entrée en langage naturel, $IR$ la représentation intermédiaire validée,
$C(IR)$ l'ensemble de capacités dérivé de l'en-tête, $D$ la fonction de décision
déterministe et $\mathit{Exec}$ l'étape d'exécution. Le pipeline s'abstrait en

$$ NL \rightarrow IR \rightarrow D(IR) \rightarrow \mathit{Exec}(IR, D). $$

### 12.2 Invariant 1 — Autorisation déterministe

Pour toute RI validée et version de politique fixée $V$ :

$$ D(I, R, T, S, V) = D(I, R, T, S, V). $$

La fonction de décision est déterministe et exempte de comportement stochastique ; les
résultats sont reproductibles, rejouables et auditables.

### 12.3 Invariant 2 — Borne de capacité par préfixe

Avec $IR = \{s_1, \dots, s_n\}$ et $s_1$ l'en-tête immuable :

$$ \forall i > 1, \quad C(s_i) \subseteq C(s_1). $$

Aucune capacité introduite après le préfixe ne peut excéder la portée définie à
l'initialisation, empêchant l'escalade par dialogue multi-tours ou injection tardive.

### 12.4 Invariant 3 — Monotonie verticale

Avec $C_0$ dérivé de l'en-tête et $C_k$ la portée effective à l'étape $k$ :

$$ C_{k+1} \subseteq C_k. $$

Les capacités ne peuvent que rester constantes ou diminuer à travers les couches de
traitement.

### 12.5 Invariant 4 — Médiation d'exécution

Toute action opérationnelle doit satisfaire

$$ \mathit{Exec} \subseteq C(IR) \quad\text{et}\quad \mathit{Exec} \neq f(NL). $$

L'exécution est médiatisée exclusivement par une RI validée et une politique déterministe ;
la sortie de langage brute ne peut déclencher directement l'exécution, éliminant les chemins
texte-libre-vers-action.

### 12.6 Invariant 5 — Énumération fermée des capacités

Avec $\mathcal{I}$ l'ensemble fini des intentions permises :

$$ \mathit{Intention} \in \mathcal{I}. $$

Aucune introduction dynamique de nouveaux types d'action n'est possible sans modification
explicite du catalogue, garantissant une gouvernance des capacités contrôlée par version.

### 12.7 Théorème — Garantie de non-escalade conditionnelle

**Hypothèses.** (A1) le cœur de décision déterministe et le validateur statique sont
corrects vis-à-vis de leurs spécifications ; (A2) toute action exécutée provient d'une RI
ayant passé la validation statique.

**Théorème.** Sous les invariants 1–5 et les hypothèses A1–A2, aucune entrée adversariale en
langage naturel ne peut accroître les capacités effectives du système au-delà de celles
déclarées dans l'en-tête de la RI validée.

**Esquisse de preuve.** (1) Le langage naturel est d'abord traduit en RI. (2) La RI doit
passer la validation de schéma et l'énumération d'intentions (A2). (3) L'invariant de
préfixe borne l'espace de capacités (invariant 2). (4) La politique déterministe applique
l'autorisation strictement dans la portée déclarée (invariants 1, 4 ; A1). (5) La monotonie
verticale empêche toute expansion en aval (invariant 3). Toute tentative adversariale
d'élargir les capacités échoue donc à la validation de la RI, est rejetée par la politique,
ou est réduite lors du décodage. L'escalade de capacités est donc structurellement
impossible dans le modèle défini. ∎

**Portée du conditionnel.** La garantie est explicitement relative à A1–A2. Elle ne couvre
*pas* les fautes de traduction : une RI *valide mais sémantiquement erronée* (p. ex. une
procédure classée comme connaissance) est appliquée fidèlement et peut causer une
sur-restriction. Borner la probabilité et l'impact de telles fautes est une question
empirique, hors périmètre ici.

### 12.8 Limites de la garantie

Les invariants garantissent l'absence d'exécution d'action non autorisée et l'absence
d'escalade de privilège au-delà de la portée déclarée. Ils ne garantissent **pas** l'absence
de sortie informationnelle hallucinée, la protection contre les dénis de service au niveau
infrastructure, ou l'élimination des inexactitudes sémantiques. **Les garanties sont
architecturales, non épistémiques.**

---

## 13. Auditabilité et gouvernance

Un objectif central de Lock-Monotone est de fournir l'**auditabilité et la gouvernance par
construction**. En environnement adversarial ou réglementé, les garanties de sécurité sont
insuffisantes si les décisions ne peuvent être expliquées, reconstruites et revues
*a posteriori*.

### 13.1 Auditabilité par construction

Chaque étape de traduction émet une représentation intermédiaire structurée capturant
l'état du système en ce point. Les décisions d'autorisation en T2 sont enregistrées avec la
version de politique, les attributs évalués et l'ensemble de capacités résultant. Les
réponses en aval en T3 sont donc directement attribuables à un état d'autorisation
spécifique plutôt qu'à un comportement opaque du modèle. Cela permet la reconstruction
*a posteriori* de *quelles* capacités ont été demandées, *lesquelles* ont été retirées ou
restreintes, et *quelles* politiques ont été appliquées.

### 13.2 Enregistrements de décision déterministes

Chaque décision peut être journalisée comme un tuple contenant au minimum : un hachage de la
RI structurée ; l'identifiant et la version de l'ensemble de politiques appliqué ; la
décision d'autorisation résultante et l'ensemble de capacités ; un horodatage et des
métadonnées contextuelles. Les modèles de langage ne participant pas à l'autorisation, les
enregistrements d'audit ne sont pas soumis à l'ambiguïté probabiliste : des entrées
identiques sous des politiques identiques produisent des décisions identiques.

### 13.3 Gouvernance et évolution des politiques

Les politiques de sécurité, règles métier et contraintes de conformité sont externalisées
comme artefacts explicites pouvant être revus, versionnés et mis à jour indépendamment des
modèles. Les ajustements de seuils d'autorisation ou d'exigences réglementaires ne
nécessitent ni réentraînement ni réalignement des modèles. La responsabilité passe d'un
comportement de modèle opaque à des artefacts de politique revus par des humains.

### 13.4 Escalade avec humain dans la boucle

Lock-Monotone prend en charge l'escalade explicite vers des opérateurs humains comme
*résultat gouverné* plutôt que comme exception. Les requêtes nécessitant une revue sont
remontées avec leurs représentations structurées et leurs contraintes. L'intervention
humaine ne contourne pas les invariants architecturaux : les décisions des opérateurs sont
enregistrées, auditables et soumises aux mêmes contraintes de monotonie des capacités que
les décisions automatisées.

---

## 14. Standardisation et nomenclature des actions

### 14.1 La gouvernance déterministe comme standardisation

Au-delà de l'application de l'autorisation, le cœur de décision permet une seconde propriété
structurelle : la **standardisation des actions**. Toute capacité opérationnelle ou
informationnelle doit être explicitement nommée, formellement définie, contrôlée par
version et associée à des règles d'évaluation déterministes — transformant le système d'une
interface souple pilotée par prompt en une taxonomie d'actions gouvernée.

### 14.2 Énumération fermée des actions

Soit $\mathcal{A} = \{A_1, \dots, A_n\}$ l'ensemble fini des actions permises. Chaque $A_i$
est défini par un nom canonique, une classe sémantique (`KNOWLEDGE`, `PROCEDURE`,
`SENSITIVE`), un niveau de privilège requis, des rôles autorisés, des contraintes de
sensibilité et des contraintes d'exécution. Aucune action hors de $\mathcal{A}$ ne peut être
exécutée ; toute extension exige une modification explicite du catalogue et un incrément de
version de politique.

### 14.3 Liaison des règles métier et de sécurité

Chaque action lie des contraintes fonctionnelles et d'autorisation/conformité :

$$ A_i = (\mathit{Nom}, \mathit{ClasseSémantique}, \mathit{EnsembleDePrivilèges}, \mathit{RèglesMétier}, \mathit{RèglesDeSécurité}). $$

Les règles métier peuvent inclure la validation de domaine, les paramètres requis, les
préconditions d'exécution et l'ordonnancement de workflow. Les règles de sécurité incluent
le RBAC, l'isolation des tenants, les exigences d'escalade de sensibilité et les exigences
de journalisation/traçabilité.

### 14.4 La nomenclature comme primitive de sécurité

Le modèle ne pouvant inventer de nouveaux noms d'actions canoniques, tout comportement
opérationnel doit correspondre à des entrées prédéfinies. Cela crée un vocabulaire contrôlé
pour l'autorité d'exécution et permet l'interprétabilité inter-équipes, la cohérence
d'audit, la testabilité déterministe et la traçabilité de version.

---

## 15. Méthodologie d'évaluation proposée (travaux futurs)

Cette section esquisse — comme travail futur — une méthodologie d'évaluation transversale.
**Aucun résultat n'est revendiqué ici.** L'objectif de la méthodologie est de convertir
l'argument structurel présent en une garantie *mesurée*.

### 15.1 Principe d'évaluation

> La sécurité doit être mesurée à la *frontière du système*, non à la sortie du modèle.

Même si un modèle génère un langage adversarial, l'architecture doit empêcher l'escalade de
capacités. L'évaluation vérifie donc que la monotonie positionnelle est respectée, que
l'application de politique déterministe est cohérente, et qu'aucun chemin d'exécution non
autorisé n'est atteignable.

### 15.2 Protocole en deux phases

- **Phase 1 — Référence au niveau modèle.** Évaluer le LLM sous-jacent sur des benchmarks
  canoniques : Taux de Succès d'Attaque (ASR), Taux de Succès d'Injection (ISR), Taux de
  Faux Positifs (FPR). Établit une référence comportementale.
- **Phase 2 — Évaluation système transversale.** Évaluer le pipeline Lock-Monotone complet
  via des suites d'attaques structurées : injection directe, escalade multi-tours, documents
  RAG hostiles, abus d'outils simulés et sondes d'exfiltration de secrets par jetons canari.
  Mesure l'application architecturale plutôt que la conformité du modèle.

### 15.3 Métriques proposées

Cibles architecturales non négociables (à tester empiriquement, non à supposer) :

- **Taux d'Actions Non Autorisées (UAR)** — cible 0.
- **Taux de Fuite de Secrets (SLR)** — cible 0.
- **Couverture des Traces de Décision (DTC)** — cible 100 %.

Métriques de robustesse complémentaires : ASR, ISR, Taux d'Injection RAG (RIR), Taux
d'Escalade Multi-tours (MER), Taux de Capture par Garde-fous (GCR).

### 15.4 Harnais d'évaluation

Un harnais structuré avec des tests unitaires pour la logique de politique, des tests
d'intégration entre composants du pipeline, des tests de non-régression entre versions de
politique, et des paquets d'audit déterministes par cas de test. Chaque cas de test définit
un scénario d'attaque structuré, la décision de politique attendue, la portée de capacités
permise et les sorties ou appels d'outils interdits. Le harnais sépare la *génération
d'attaques* de la *validation par oracle*, garantissant reproductibilité et comparabilité
entre versions de modèle. Nous proposons de l'instancier sur le cas SIEM de la section 11.

---

## 16. Discussion et limites

- **Portée architecturale des garanties.** Lock-Monotone assure l'autorisation déterministe,
  l'exécution bornée des capacités, la prévention de l'escalade de privilèges et
  l'élimination des chemins directs langage-vers-action. Il ne dépend pas de la conformité
  probabiliste du modèle — mais s'applique strictement à la gouvernance des capacités et au
  contrôle d'exécution, non à l'exactitude du contenu informationnel.
- **Limites épistémiques.** Il n'élimine pas le contenu factuel halluciné, les inexactitudes
  de raisonnement ou les malentendus sémantiques dans les tâches purement informationnelles.
  L'architecture contraint *ce que le système peut faire*, non *si le modèle est
  épistémiquement correct*.
- **Dépendance à une traduction correcte.** La sécurité ne repose pas sur l'exactitude
  probabiliste, mais bien sur une traduction réussie en RI valide. Une mauvaise
  classification systématique produit des refus excessifs — bornés par l'application
  déterministe et incapables d'escalader le privilège, mais dégradant l'utilisabilité. C'est
  la principale motivation de l'évaluation empirique prévue.
- **Dépendance à l'ingénierie des politiques.** La responsabilité se déplace vers
  l'ingénierie des politiques ; des politiques trop permissives minent la sécurité, trop
  restrictives réduisent l'utilité. Les politiques sont des artefacts de premier ordre à
  revoir, tester et versionner avec la rigueur du contrôle d'accès traditionnel.
- **Risques au niveau infrastructure.** Les invariants ne traitent pas le déni de service,
  la mauvaise configuration d'infrastructure, la compromission matérielle ou les
  vulnérabilités externes. Lock-Monotone gouverne le flux logique de capacités ; il ne
  remplace pas les contrôles traditionnels. Il suppose aussi que les composants
  déterministes (moteurs de politique, validateurs, connecteurs) sont correctement
  implémentés et sécurisés.
- **Gestion du catalogue et rigidité de schéma.** Le catalogue fermé et la RI contrainte
  créent des avantages de gouvernance mais aussi une charge opérationnelle et une
  flexibilité expressive réduite — un compromis délibéré entre flexibilité et contrôle
  formel.
- **Facteurs humains.** L'ingénierie sociale des opérateurs, la mauvaise configuration ou le
  contournement intentionnel peuvent encore causer des défaillances ; l'architecture fournit
  des frontières techniques, non un substitut à la gouvernance organisationnelle.
- **Évolution des modèles.** L'application étant externalisée, les mises à niveau de modèle
  ne modifient pas les frontières de capacités, les tests de non-régression restent
  faisables et les invariants de politique restent stables entre familles de modèles —
  atténuant la dérive systémique.

---

## 17. Travaux connexes

**Sécurité centrée sur le modèle.** Un large corpus améliore la sécurité des LLM par des
techniques au niveau modèle — RLHF, IA constitutionnelle, fine-tuning adversarial. Elles
réduisent les sorties nuisibles en façonnant le comportement interne mais restent
fondamentalement probabilistes : la sécurité est tributaire d'une conformité statistique
plutôt que de garanties structurelles. Lock-Monotone relocalise l'autorisation hors du
modèle, traitant les améliorations d'alignement comme complémentaires mais non suffisantes.

**Injection de prompt et garde-fous.** L'injection directe et indirecte via du contenu
récupéré ou renvoyé par les outils motive des techniques de garde-fous : durcissement du
prompt système, application de sortie structurée, filtrage par regex, validation de schéma.
Elles opèrent souvent *a posteriori* et ne redéfinissent pas les frontières d'autorité
d'exécution. Lock-Monotone s'en distingue en imposant une RI typée et une évaluation de
politique déterministe avant l'exécution.

**Défenses par-construction.** La lignée la plus proche retire l'autorité au modèle par
construction. Le motif **dual-LLM** isole le contenu non fiable dans un modèle en
quarantaine dont les sorties ne sont référencées que symboliquement par un modèle
privilégié, de sorte que les données contaminées ne pilotent jamais d'actions privilégiées.
**CaMeL** étend cette idée : un modèle privilégié émet du code dans un DSL en bac à sable
capturant les flux de contrôle et de données de la requête *de confiance*, et des capacités
appliquent des politiques de flux de données au moment de l'appel d'outil, produisant une
sécurité prouvable pour une classe de tâches. Lock-Monotone s'aligne sur cette posture et
s'en distingue selon quatre axes. Premièrement, notre séparation est *épistémique* : la
classification connaissance/procédure conditionne l'existence même d'une autorité
actionnable, avant toute analyse de flux de contrôle ou de données. Deuxièmement,
l'*invariant de priorité d'action par préfixe* introduit une borne de moindre privilège
*temporelle* explicite — capacités plafonnées à la première déclaration pour toute
l'interaction — ciblant directement l'escalade multi-tours et, à notre connaissance, non
explicité dans dual-LLM ou CaMeL. Troisièmement, le *Triple Décodage* ajoute une porte de
conformité indépendante (DLP/données personnelles/secrets) en aval de l'autorisation,
offrant une défense en profondeur au-delà des capacités de flux de données. Quatrièmement,
la nomenclature d'actions fermée et versionnée transforme la gouvernance des capacités en
une couche de standardisation auditable. Nous notons une asymétrie de maturité : CaMeL est
un système implémenté avec des garanties démontrées sur un benchmark, tandis que
Lock-Monotone est pour l'instant un cadre dont l'évaluation reste à venir.

**Fondations classiques.** Lock-Monotone hérite de la théorie classique de la sécurité. Le
concept de *moniteur de référence* (inviolable, toujours invoqué, vérifiable) sous-tend le
cœur de décision déterministe ; la conception vérifiée de systèmes sécurisés motive la
relocalisation de la confiance vers un composant petit et analysable ; et le contrôle
d'accès par politique (RBAC, ABAC) fournit la tradition d'autorisation déterministe étendue
ici aux environnements médiatisés par LLM via la traduction typée et la monotonie
positionnelle. Nous suivons une pratique architecturale établie et nous alignons sur les
cadres de gouvernance émergents (NIST AI RMF, AI Act de l'UE).

---

## 18. Conclusion et travaux futurs

Les LLM sont de plus en plus déployés dans des environnements qui manipulent des données
sensibles et des capacités opérationnelles, où la génération de langage probabiliste ne peut
servir de frontière de sécurité. Ce travail a introduit **Lock-Monotone**, une architecture
de gouvernance déterministe qui relocalise la confiance : du comportement du modèle vers les
invariants architecturaux, de l'alignement statistique vers l'application déterministe, et
de l'exécution implicite vers la gouvernance explicite des capacités.

L'architecture est formalisée par une séparation ontologique entre connaissance et
procédure, une représentation intermédiaire typée (PyIR), un invariant de priorité d'action
par préfixe imposant la monotonie positionnelle, un cœur de décision déterministe, une
couche de médiation d'exécution en triple décodage, une nomenclature d'actions fermée et
versionnée, et l'auditabilité par construction. Sous les hypothèses énoncées, l'architecture
garantit qu'aucune entrée adversariale en langage naturel ne peut élargir les capacités du
système au-delà de celles déclarées dans l'en-tête de la RI validée. La sécurité devient
ainsi une propriété *structurelle* du système plutôt qu'une propriété probabiliste du
modèle.

Cet article est cadré comme un cadre architectural. Son principal risque résiduel — la
fiabilité de la traduction — n'est explicitement pas mesuré ici. La direction de travail
future immédiate et la plus importante est une évaluation empirique qui, contrairement aux
benchmarks centrés sur le modèle focalisés sur la résistance au jailbreak, mesure la
fidélité de la traduction langage-naturel-vers-RI sous entrée adversariale : le taux de
mauvaise classification sémantique, le taux de faux refus induit, et les conditions sous
lesquelles le fine-tuning améliore la conformité structurelle. Une telle évaluation —
idéalement instanciée sur le cas SIEM et comparable aux bancs d'essai de sécurité
agentique — convertirait l'argument structurel présent en une garantie mesurée. D'autres
travaux incluent la vérification formelle des règles de politique, la synthèse automatisée
de catalogues d'actions et l'intégration aux cadres de conformité réglementaire.

Plutôt que de rendre les modèles de langage intrinsèquement dignes de confiance,
Lock-Monotone les rend *subordonnés à une gouvernance déterministe*, transformant la
sécurité des LLM d'un problème de réglage en une discipline architecturale.

---

## Références

1. T. B. Brown et al., « Language models are few-shot learners », *NeurIPS*, 2020.
2. J. Wei et al., « Chain-of-thought prompting elicits reasoning in large language models »,
   *NeurIPS*, 2022.
3. L. Ouyang et al., « Training language models to follow instructions with human
   feedback », *NeurIPS*, 2022.
4. Y. Bai et al., « Constitutional AI: Harmlessness from AI feedback », arXiv:2212.08073,
   2022.
5. A. Zou, Z. Wang, J. Z. Kolter, M. Fredrikson, « Universal and transferable adversarial
   attacks on aligned language models », arXiv:2307.15043, 2023.
6. K. Greshake et al., « Not what you've signed up for: Compromising real-world
   LLM-integrated applications with indirect prompt injection », *ACM AISec*, 2023.
   (arXiv:2302.12173)
7. Y. Liu et al., « Prompt injection attacks and defenses in LLM-integrated applications »,
   arXiv:2306.05499, 2023.
8. S. Willison, « The dual LLM pattern for building AI assistants that can resist prompt
   injection », 2023. https://simonwillison.net/2023/Apr/25/dual-llm-pattern/
9. E. Debenedetti, I. Shumailov, T. Fan, J. Hayes, N. Carlini, D. Fabian, C. Kern, C. Shi,
   A. Terzis, F. Tramèr, « Defeating prompt injections by design » (CaMeL),
   arXiv:2503.18813, 2025.
10. J. H. Saltzer, M. D. Schroeder, « The protection of information in computer systems »,
    *Proc. IEEE*, 63(9):1278–1308, 1975.
11. J. Rushby, « Design and verification of secure systems », *ACM SOSP*, 1981.
12. R. Anderson, *Security Engineering*, 3e éd., Wiley, 2020.
13. L. Bass, P. Clements, R. Kazman, *Software Architecture in Practice*, 3e éd.,
    Addison-Wesley, 2012.
14. National Institute of Standards and Technology, « Artificial Intelligence Risk
    Management Framework (AI RMF 1.0) », NIST, 2023.
15. Union européenne, « Règlement (UE) 2024/1689 (Règlement sur l'intelligence
    artificielle) », 2024.

> **Note.** Les identifiants arXiv doivent être re-vérifiés avant toute soumission
> formelle.
