##### Ce document nous explique le fonctionnement d'autres règle du package Renraku #####


Nous allons découper l’analyse par familles fonctionnelles car tout Renraku tourne autour de 4 axes conceptuels majeurs :

1. Les Events : annonces, notifications critiques
2. Les Stratégies de gestion d’exception
3. Les Critiques + propriétés + actions : le cœur du rule engine
4. Le Rule manager + checker : pipeline d’exécution des règles

Ensuite nous analyserons les rôles, les implémentations et les fonctionnements pour chaque bloc.

###### 1. ReCritiqueEvent & sous classes ######

### Rôle

Ils servent à **annoncer un évènement lié à une critique** : 
critique vue, critique multiple, critique auto-fixée, critique bannie.
Ils sont utilisés par `ReSystemAnnouncer` pour notifier l’UI, les outils side-tool, l’IDE.

### Implémentation

On a deux familles :

| Classe                      | Instance var                                    | rôle                                 |
| --------------------------- | ----------------------------------------------- | ------------------------------------ |
| ReMultipleCritiquesEvent    | critiques                                       | plusieurs critiques sur un entity    |
| ReEntityCriticized          | hérite et représente un entity complet critiqué |                                      |
| ReSingleCritiqueEvent       | critique                                        | critique unique                      |
| ReCritiqueAutofixed         | zéro variable                                   | cas où la critique a été autofixée   |
| ReCritiqueBanned            | entityBanLevel                                  | quand on banne une règle             |
| ReCritiqueDescriptionViewed |                                                 | juste l’action de vue de description |

### Fonctionnement

Le pattern ici = **Announcer Pattern**.
Renraku ne traite pas l'UI directement → il émet des events.
Ensuite l’IDE (Calypso, QualityAssistant, etc) écoute ces évènements.

`ReSystemAnnouncer uniqueInstance notifyCritique: …` émet ces events.

### Cycle complet d’exécution Renraku (vraiment runtime)

a. On choisit une **scope** d’analyse : Package, Class, Method
b. Le `ReRuleManager` fournit la liste des règles applicables à ce scope
c. Le `ReSmallLintChecker` itère chaque règle + chaque entité du scope
d. Chaque règle applique sa méthode `check:`
e. Si un problème est détecté : elle retourne un objet **ReAbstractCritique**
f. `ReSmallLintChecker` stocke la critique dans sa collection locale
g. Le `ReSystemAnnouncer` émet les events correspondants (`ReCritiqueEvent`)
h. Les UI / tools réagissent : QA assistant, Calypso, Refactoring tool etc.

=> Renraku est un moteur d’analyse AST qui émet un flux d’évènements

###### 2. ReExceptionStrategy  #####

### Rôle

Définir comment le rule engine doit réagir aux exceptions pendant l’analyse.

### Implémentations

***ReIgnoreExceptionStrategy** : ignore complètement
***ReLogExceptionStrategy** : log
***RePassExceptionStrategy** : fait remonter

### Fonctionnement

`ReSmallLintChecker >> getCritiquesAbout: anEntity`
fait du `on: Error do: [ … ]`
et c’est ici que la stratégie choisie dit quoi faire.

C’est donc une couche de robustification pour éviter qu'une règle buggée bloque tout.

Les phases Critique, Action et Rule sont séparées

| Objet    | responsabilité                                |
| -------- | --------------------------------------------- |
| Rule     | détecter (algorithme / heuristique)           |
| Critique | représenter le problème détecté               |
| Action   | représenter ce qu’on peut faire pour corriger |

Beaucoup de systèmes Lint mélangent les 3.
Ici non : **Renraku sépare la detection, la représentation et la réparation**.

Cela rend le système scalable, plugable, sandboxable.

###### 3. Properties, Critiques, Actions  ######

### Rôle

Ce sont les modèles d’une **critique générée** par une règle.

 `ReAbstractCritique` est la base.
 `ReCompactSourceCodeCritic`, `ReInvocationOrderCritique` → critiques spécialisées.
 `RePropertyAction` → représente une action possible face à la critique (auto-fix, jump to code, …)

### Implémentation

Une critique contient :

- la règle
- tinyHint
- un anchor vers le code (`ReSourceAnchor`)

Les variations `PriorInvocationCritique`, `LaterInvocationCritique` ajoutent la sémantique sur séquence avant/après (via `ReInvocationSequenceRuleSequence`).

### Fonctionnement

Quand une règle détecte un problème elle construit une 'ReAbstractCritique' avec un anchor.
Plus tard : l’IDE peut proposer des actions via RePropertyAction.

Cela permet d’avoir un modèle d’objet homogène pour toutes critiques.

### Cas très particulier : InvocationOrderCritique

Ici on voit clairement l'intelligence Renraku : il ne vérifie pas seulement la syntaxe, il vérifie surtout la **logique métier dans l’ordre d’invocation message**.

C’est un niveau d’analyse beaucoup plus subtil que du lint classique.




###### 4. Le pipe execution: ReRuleManager + ReSmalllintChecker  ######

### Rôle

C’est **le cœur moteur** de Renraku.
Ce sont eux qui prennent les règles, exécutent, collectent les critiques, calculent false positive, to-do, etc.

### Implémentation

**ReRuleManager** classifie les règles par type :

* methodRules
* classRules
* nodeRules
* packageRules.

**ReSmalllintChecker** exécute réellement sur l’environnement package / classes / AST.


### Fonctionnement global pipeline:

| Étape                               | fait par           | output                         |
| ----------------------------------- | ------------------ | ------------------------------ |
| récupération règles                 | ReRuleManager      | Set de règles classifiées      |
| dispatch entity (class/method/node) | ReSmalllintChecker | application des règles sur AST |
| création critique                   | règles             | ReAbstractCritique             |
| accumulation                        | ReSmalllintChecker | Set critiques                  |
| event emission                      | ReSystemAnnouncer  | UI notifications               |

C’est exactement un lint engine **polymorphe AST**.

### Système des “ban levels” & manifest builders

Quand un développeur clique un “false positive”,
Renraku ne stocke pas cela en mémoire volatile mais dans un **Manifest** lié à l’entité code.

Ce mécanisme permet de dire :

* pour une méthode / classe / package particulier : cette règle doit être ignorée

=> Il y a donc une **persistence locale de configuration de qualité par entité**.

###### 5. ReSourceAnchor / SourceLine / Interval ######

Ce sont objets qui permettent de retrouver le **mapping code exact** où la critique a eu lieu.
C’est essentiel pour ensuite proposer highlight, navigation, auto fix pointer.



## Résumé

| Bloc                  | Ce qu'il apporte                         |
| --------------------- | ---------------------------------------- |
| Events                | communication outward                    |
| Strategies            | robustesse runtime                       |
| Critiques             | modèle d’erreur structuré manipulable    |
| RuleManager / Checker | moteur d’analyse (AST / package / class) |
| SourceAnchor          | précision des lieux dans code            |

Renraku est proprement découpé, c'est donc du Domain Driven et du LintEngine.
