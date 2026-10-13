##### Ce document nous explique le fonctionnement de la classe de règle ReAbstractRule du package Renraku #####



1. Classe principale : "ReAbstractRule"

 Rôle général

"ReAbstractRule" est une classe abstraite représentant la base de toutes les règles de qualité dans Renraku.
Chaque règle de vérification : linting, refactoring suggestion, style check... hérite de cette classe et implémente sa propre logique.

Responsabilités

Elle définit :

la structure commune des règles,
les méthodes à implémenter par les sous-classes,
les mécanismes génériques de vérification et de génération de critiques "critiqueFor:".

2. Sous-classes et leurs logiques

|-------------------------------------| -------------------------------------------------| --------------------------------------|
| Sous-classe                         | Description                                      | Catégorie              | Sévérité     |
| ------------------------------------| -------------------------------------------------| ---------------------- | ------------ |
| `ReClassSideResetMethodProtocolRule`| Vérifie que les méthodes `reset` sur le côté classe 
                                        sont dans le protocole 'class initialization'    | Clean Code             | #warning     |
| `ReCompactSourceCodeRule`           | Vérifie les caractères inutiles dans le corps des 
                                        méthodes (espaces, points finaux, etc.)          | Clean Code             | #information |
| `ReInvocationSequenceRule`          | Vérifie que certaines séquences d’invocation de 
                                        méthodes sont dans le bon ordre (`pre` et `post` conditions).| Bugs       | #warning     |
| `ReNoNilAssignationInInitializeRule`| Interdit les affectations `nil` dans les méthodes                                           `initialize`.                                                                            | Style                  | #information |
|`ReNoPrintStringInPrintOnRule`       | Interdit l’utilisation de `printString` dans 
                                        `printOn:` — suggère d’utiliser le flux (`stream`)| Optimization          | #information |
|`ReEndTrueFalseRule`                 | Vérifie les blocs `ifTrue:ifFalse:` contenant du 
                                        code identique dans les deux branches            | Optimization           | #information |
|`ReGuardingClauseRule`               | Détecte les gardes mal placés (`ifTrue:` 
                                        ou `ifFalse:` à la fin de méthodes)              | Coding Idiom Violation | #information |
| `ReNodeRewriteRule`                 | Gère les règles qui `modifient le code automatiquement``
                                        via un système de correspondance et de réécriture
                                        de l’AST                                         | Refactoring            | -------      |
| `RePackageManifestShouldBePackagedInManifestTagRule` | Vérifie que les classes `PackageManifest`
                                         sont dans la catégorie `Manifest`               | Coding Idiom Violation | #warning     |
| `ReProperMethodProtocolNameRule` (+ sous-règles)     | Vérifie la bonne catégorisation 
                                        des méthodes selon leur rôle (ex. `accessing`, `tests`, `instance creation`, etc.)   
                                                                                         | Coding Idiom Violation | #warning     |
| `ReReturnMethodRule`                 | Vérifie que les méthodes marquées `#shouldReturn` 
                                        retournent explicitement une valeur              | Bugs                   | #error       |
| `ReThemeAccessRule`                  | Détecte les accès directs à `Smalltalk ui theme` 
                                        dans les classes `Morph` ou `Model`              | API Hints              | #information |
|------------------------------------------------------------------------------------------------------------------------------  |


3. Typologie des règles

Renraku distingue plusieurs **familles de règles** :
|------------------------------------------|-------------------------------------------------------------|--------------------------------  |
| Type de règle                            | Description                                                 | Exemple                          |
| ---------------------------------------- | ----------------------------------------------------------- | -------------------------------- |
|   Structurelle (AST)                     | Analyse la structure syntaxique (nœuds, blocs, conditions). | `ReEndTrueFalseRule`             |
|   Sémantique (Méthodes/Classes)          | Vérifie la cohérence de nommage, d’organisation.            | `ReProperMethodProtocolNameRule` |
|   Refactoring (Rewrite)                  | Peut proposer ou appliquer une correction automatique.      | `ReNodeRewriteRule`              |
|   Style / Convention                     | Vérifie les conventions de code (ex. espaces, protocoles).  | `ReCompactSourceCodeRule`        |
|   Défaut logique ou erreur potentielle   | Détecte des erreurs logiques.                               | `ReReturnMethodRule`             |
|------------------------------------------|-------------------------------------------------------------|--------------------------------  |


4. Fonctionnement d’une règle dans le moteur Renraku

a. `Le moteur parcourt le code source` (méthodes, classes, AST).
b. Chaque entité est passée aux `règles enregistrées``.
c. Pour chaque règle :

   - Si `basicCheck:` retourne vrai → alors il y aura une création d’un `critique (`ReTrivialCritique`, `ReRefactoringCritique`, etc.).
d. Le moteur agrège tous les résultats et affiche :

   - le 'groupe',
   - le 'nom',
   - la 'gravité',
   - la 'justification (rationale)',
   - et la 'position dans le code' (via `ReSourceAnchor`).