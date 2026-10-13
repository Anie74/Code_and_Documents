# Documentation du package Pharo « Renraku »

*Analyse et explication de chaque classe et de chaque méthode*

Ce document décrit, classe par classe et méthode par méthode, le contenu du fichier RuleList.md fourni. Ce fichier correspond au code source du package Renraku, le moteur de règles de qualité de code (« code critics » / « linter ») de Pharo. Renraku permet de définir des règles (ReAbstractRule et ses sous-classes), de les exécuter sur des entités du système (méthodes, classes, paquets), de produire des « critiques » (ReAbstractCritique) signalant les problèmes détectés, et de gérer leur cycle de vie (bannissement, correction automatique, notification via l'annonceur du système).

*Pour chaque classe, on trouvera : son rôle général, sa place dans la hiérarchie (superclasse / sous-classes), ses variables d'instance le cas échéant, puis le détail de chacune de ses méthodes avec une explication de ce qu'elle fait.*

# 1. ReAbstractRule — la classe racine de toutes les règles

ReAbstractRule est la superclasse abstraite de toutes les règles de Renraku. Elle définit le contrat commun : comment vérifier une entité (basicCheck:), comment transformer un résultat positif en objet « critique » (critiqueFor:), et les métadonnées descriptives d'une règle (nom, groupe, gravité, explication).

## ReAbstractRule (méthodes de la classe de base)

*Catégories : accessing, compatibility, helpers, running, running-helpers, testing, utilities*

- **group** — Retourne 'Unclassified rules' par défaut : le regroupement thématique de la règle (« Clean Code », « Bugs », etc.), redéfini par chaque sous-classe concrète.
- **name** — Méthode abstraite (subclassResponsibility) : doit retourner le nom lisible de la règle.
- **rationale** — Retourne le commentaire de la classe (self class comment) comme explication courte de la règle.
- **severity** — Retourne #warning par défaut ; la gravité peut être #error, #warning ou #information selon la règle.
- **resetResult** — Méthode vide, présente pour compatibilité avec d'autres moteurs de règles (SmallLint).
- **critiqueFor: anEntity** — Construit une ReTrivialCritique générique ancrée sur l'entité entière, via anchorFor:.
- **anchorFor: anEntity** — Construit un ReSourceAnchor pointant sur le code source complet de l'entité.
- **basicCheck: anEntity** — Méthode abstraite : doit répondre vrai si l'entité viole la règle. Lève une erreur si non implémentée.
- **check: anEntity** — Collecte dans une OrderedCollection toutes les critiques trouvées pour l'entité, via check:forCritiquesDo:.
- **check: anEntity forCritiquesDo: aCriticBlock** — Exécute basicCheck: et, si vrai, évalue le bloc avec la critique correspondante (créée par critiqueFor:).
- **check:forCritiquesDo:ifNone:** — Variante qui exécute un bloc alternatif si aucune critique n'a été rencontrée.
- **check:ifNoCritiques:** — Variante pratique qui renvoie la collection des critiques trouvées, ou déclenche le bloc alternatif si vide.
- **critiqueFor: aClass about: aVarName** — Construit une critique ancrée sur une variable précise d'une classe (ReVarSearchSourceAnchor), avec un indice court (tinyHint).
- **createTrivialCritiqueOn: method intervalOf: node hint: hint** — Construit une ReTrivialCritique ancrée sur un intervalle précis du code source d'une méthode ; très utilisée par les règles opérant au niveau des nœuds d'AST.
- **isComposite** — Retourne faux : distingue une règle simple d'une règle composite (regroupement de règles).
- **isRewriteRule** — Retourne faux par défaut ; vrai uniquement pour les règles de réécriture (ReNodeRewriteRule).

## ReClassSideResetMethodProtocolRule

*Sous-classe de ReAbstractRule — groupe « Clean Code »*

Vérifie que les méthodes de classe nommées #reset sont bien classées dans le protocole 'class initialization'.

- **basicCheck: aMethod** — Vrai si la méthode est côté classe, se nomme #reset, et n'est pas catégorisée 'class initialization'.
- **critiqueFor: aMethod** — Retourne une ReRefactoringCritique encapsulant une RBMethodProtocolTransformation qui déplacera automatiquement la méthode vers le bon protocole (correction automatique possible).
- **group** — Retourne 'Clean Code'.
- **name** — Retourne le texte : « Class side methods called #reset should be in the 'class initialization' protocol ».

## ReCompactSourceCodeRule

*Sous-classe de ReAbstractRule — groupe « Clean Code »*

Détecte les caractères superflus (lignes vides finales, point final, espaces de fin) dans le corps d'une méthode.

- **basicCheck: aMethod** — Délègue à ReMethodSourceCleaner class>>isNecessaryToClean:allowFinalDot: pour savoir si un nettoyage est nécessaire.
- **check: aMethod forCritiquesDo: aCriticBlock** — Redéfinie pour produire directement une ReCompactSourceCodeCritic (plutôt que la critique générique) lorsque basicCheck: est vrai.
- **group** — Retourne 'Clean Code'.
- **name** — Retourne 'Unnecessary characters found in method body'.
- **providesChange** — Retourne vrai : la critique produite propose une correction automatique.
- **severity** — Retourne #information : problème purement cosmétique.

## ReExclusivelyBeginnersRule

*Sous-classe de ReAbstractRule*

Classe marqueur regroupant des règles destinées spécifiquement aux programmeurs débutants (aucun corps de méthode n'est fourni dans l'extrait analysé).

## ReInvocationSequenceRule

*Sous-classe abstraite de ReAbstractRule*

Règle générique vérifiant qu'un message doit obligatoirement être suivi (ou précédé) d'un autre message donné au sein d'une même méthode (contrôle d'ordre d'appel, par ex. #initialize puis #yourself).

- **add: preSelectors requiresPostSend: postSelectors** — Enregistre une contrainte « après » : les sélecteurs preSelectors doivent être suivis des sélecteurs postSelectors.
- **add: postSelectors requiresPreSend: preSelectors** — Enregistre une contrainte « avant » : les sélecteurs postSelectors doivent être précédés des sélecteurs preSelectors.
- **check: aNode forCritiquesDo: aCriticBlock** — Ne s'applique qu'aux nœuds de type méthode ; exécute successivement preCheck:forCriticsDo: et postCheck:forCriticsDo:.
- **critic: aCriticClass for: ast at: node regarding: sequence** — Fabrique une critique (classe passée en paramètre) ancrée sur l'intervalle du nœud fautif.
- **initialize** — Initialise les ensembles preConditions et postConditions comme des Set vides.
- **name** — Retourne 'Invocation order incorect' (sic, faute d'orthographe dans le code source).
- **postCheck: anAST forCriticsDo: aCriticBlock** — Parcourt l'AST à l'envers pour repérer les envois de message qui ne sont pas suivis du message requis ; produit une critique par occurrence manquante.
- **postCriticFor: ast at: node regarding: sequence** — Construit une ReLaterInvocationCritique via critic:for:at:regarding:.
- **preCheck: anAST forCriticsDo: aCriticBlock** — Parcourt l'AST dans l'ordre pour repérer les envois de message qui ne sont pas précédés du message requis ; produit une critique par occurrence manquante.
- **preCriticFor: ast at: node regarding: sequence** — Construit une RePriorInvocationCritique via critic:for:at:regarding:.

## ReNoNilAssignationInInitializeRule

*Sous-classe de ReAbstractRule — groupe « Style »*

Signale les affectations inutiles du type « ivar := nil » dans une méthode #initialize, puisque nil est déjà la valeur par défaut d'une variable d'instance.

- **check: aMethod forCritiquesDo: aCriticBlock** — Ne s'applique qu'aux méthodes d'instance nommées #initialize ; recherche les nœuds d'affectation dont la valeur littérale est nil et produit une critique pour chacun.
- **group** — Retourne 'Style'.
- **name** — Retourne 'Initialize method does not need nil assignation'.
- **severity** — Retourne #information.

## ReNoPrintStringInPrintOnRule

*Sous-classe de ReAbstractRule — groupe « Optimization »*

Signale l'appel à #printString à l'intérieur d'une méthode #printOn: (il faut utiliser directement le flux plutôt que de créer une chaîne intermédiaire).

- **check: aMethod forCritiquesDo: aCriticBlock** — Ne s'applique qu'aux méthodes #printOn: ; recherche les envois de message #printString et produit une critique pour chacun, avec un indice (hint) sur le sélecteur.
- **group** — Retourne 'Optimization'.
- **name** — Retourne 'No printString inside printOn'.
- **rationale** — Explique qu'il faut utiliser 'stream print: anObject' plutôt que 'stream nextPutAll: anObject printString'.

## ReNodeBasedRule et sa hiérarchie de règles basées sur des motifs (pattern matching)

ReNodeBasedRule, RePatternCodeRule et ReNodeMatchRule forment une hiérarchie abstraite intermédiaire pour les règles qui opèrent sur des nœuds d'AST individuels en les comparant à des motifs de code (patterns), plutôt que d'analyser toute la méthode « à la main ».

### ReEndTrueFalseRule

*Sous-classe de ReNodeMatchRule — groupe « Optimization »*

Détecte les blocs ifTrue:ifFalse: qui répètent la même instruction au début ou à la fin des deux branches (code dupliqué).

- **afterCheck: aNode mappings: mappingDict** — Validation exécutée après qu'un motif a été reconnu : vérifie que la dernière instruction commune n'est pas simplement une variable temporaire partagée entre les deux blocs (pour éviter les faux positifs).
- **group** — Retourne 'Optimization'.
- **initialize** — Enregistre quatre motifs (matchesAny:) couvrant les quatre combinaisons possibles : instruction dupliquée en début ou fin de bloc, branche ifTrue: ou ifFalse: en premier.
- **name** — Retourne 'Check for same statements at end of ifTrue:ifFalse: blocks'.
- **rationale** — Explique que la règle vérifie la présence du même code au début ou à la fin des blocs ifTrue:ifFalse:.
- **severity** — Retourne #information.

### ReGuardingClauseRule

*Sous-classe de ReNodeMatchRule — groupe « Coding Idiom Violation »*

Recommande l'emploi d'une clause de garde (retour anticipé) plutôt qu'un ifTrue:/ifFalse: final contenant deux instructions ou plus.

- **initialize** — Enregistre deux motifs de méthode (addMatchingMethod:) correspondant à un ifTrue: ou un ifFalse: en dernière instruction d'une méthode, avec un bloc contenant au moins deux instructions.
- **name** — Retourne 'Guarding clauses'.
- **rationale** — Explique que ce code gagnerait probablement à être exprimé avec un retour anticipé (return self) plutôt qu'un bloc conditionnel final.

## ReNodeRewriteRule

*Sous-classe abstraite de ReAbstractRule*

Règle capable non seulement de détecter un motif, mais aussi de proposer automatiquement une réécriture du code correspondant. Elle constitue le socle du DSL de réécriture de Renraku.

- **addMatchingExpression: aString rewriteTo: aRewriteString** — Enregistre une paire « motif à détecter » / « expression de remplacement », toutes deux analysées par RBParser.
- **addMatchingExpression: aString rewriteWith: aBlock** — Enregistre un motif associé à un bloc qui calcule dynamiquement le nœud de remplacement (reçoit le nœud trouvé et le dictionnaire des correspondances de jokers).
- **addMatchingMethod: aString rewriteTo: aRewriteString** — Identique à addMatchingExpression:rewriteTo: mais pour un motif portant sur une méthode entière.
- **addMatchingMethod: aString rewriteWith: aBlock** — Identique à addMatchingExpression:rewriteWith: mais pour un motif de méthode entière.
- **check: aNode forCritiquesDo: aCriticBlock** — Pour chaque motif enregistré, tente une correspondance sur le nœud ; en cas de succès (et après validation optionnelle via afterCheck:mappings:), produit une ReReplaceNodeCritique portant la réécriture proposée.
- **critiqueFor: aNode by: rewriter withMappings: map** — Construit la ReReplaceNodeCritique contenant l'ancien nœud et le nouveau nœud calculé par le rewriter.
- **initialize** — Crée le dictionnaire 'matches' qui associe chaque motif à son remplacement.
- **isRewriteRule** — Retourne vrai (redéfinition) : signale que cette règle propose des réécritures automatiques.
- **replace: aString byEvaluating: aBlock** — Raccourci pratique appelant addMatchingExpression:rewriteWith:.
- **replace: aString with: aRewriteString** — Raccourci pratique appelant addMatchingExpression:rewriteTo:.
- **replaceMethod: aString with: aRewriteString** — Raccourci pratique appelant addMatchingMethod:rewriteTo:.

## RePackageManifestShouldBePackagedInManifestTagRule

*Sous-classe de ReAbstractRule — groupe « Coding Idiom Violation »*

Vérifie que la classe de manifeste d'un paquet est bien rangée sous la catégorie de classe 'Manifest'.

- **basicCheck: aClass** — Vrai si la classe hérite de PackageManifest mais n'est pas incluse dans les classes taguées 'Manifest' de son paquet.
- **group** — Retourne 'Coding Idiom Violation'.
- **name** — Retourne « A package manifest should be tagged using 'Manifest' class tag ».
- **rationale** — Explique que le tag 'Manifest' aligne le paquet sur les conventions des autres paquets Pharo.

## ReProperMethodProtocolNameRule

*Sous-classe abstraite de ReAbstractRule — groupe « Coding Idiom Violation »*

Famille de règles vérifiant que les méthodes sont classées sous un protocole (catégorie de méthode) conforme aux conventions, plutôt que sous un nom de protocole obsolète ou non standard.

- **basicCheck: aMethod** — Mémorise la catégorie actuelle de la méthode puis vérifie si elle figure dans la liste (côté classe) des mauvais noms de protocole.
- **critiqueFor: aMethod** — Retourne une ReRefactoringCritique (corrigeable automatiquement) encapsulant une RBMethodProtocolTransformation qui reclassera la méthode dans le bon protocole.
- **group** — Retourne 'Coding Idiom Violation'.
- **methodProtocolName / methodProtocolName:** — Accesseur et mutateur de la variable d'instance mémorisant le protocole fautif détecté.
- **name** — Compose dynamiquement un message indiquant le protocole proposé et le protocole actuel incorrect.
- **rationale** — Explication générique : vérifie que le nom du protocole de méthode est approprié et respecte les conventions.
- **use: valid insteadOf: arrayOfInvalid** — Méthode utilitaire déléguant à la méthode de classe homonyme, utilisée pour améliorer la lisibilité dans les sous-classes qui définissent les listes de noms valides/invalides.

Cette règle abstraite possède huit sous-classes concrètes, chacune vérifiant la convention de nommage de protocole pour une catégorie particulière de méthodes ; elles partagent toutes la même logique héritée et ne redéfinissent que les listes de noms « bons »/« mauvais » (non détaillées dans l'extrait de code fourni) :

  - ReProperMethodProtocolNameForAccessingRule (protocole 'accessing')

  - ReProperMethodProtocolNameForAddingRemovingRule (protocole d'ajout/suppression d'éléments)

  - ReProperMethodProtocolNameForConvertingRule (protocole de conversion)

  - ReProperMethodProtocolNameForFileInOutRule (protocole de sérialisation / file-in-out)

  - ReProperMethodProtocolNameForInstanceCreationRule (protocole de création d'instances, côté classe)

  - ReProperMethodProtocolNameForRemovalRule (protocole de suppression)

  - ReProperMethodProtocolNameForTestsRule (protocole des méthodes de test SUnit)

  - ReProperMethodProtocolNameForUtilitiesRule (protocole 'utilities')

### Cas particulier — ReProperMethodProtocolNameForTestsRule

Cette sous-classe redéfinit basicCheck: pour ne s'appliquer qu'aux méthodes de test SUnit.

- **basicCheck: aMethod** — Vrai uniquement si la méthode est une méthode de test (isTestMethod) et que sa catégorie actuelle figure parmi les mauvais noms de protocole.

## ReReturnMethodRule

*Sous-classe de ReAbstractRule — groupe « Bugs »*

Détecte les méthodes qui redéfinissent une méthode portant le pragma #shouldReturn mais qui ne se terminent pas par une instruction de retour explicite.

- **basicCheck: aMethod** — Vrai si au moins une méthode redéfinie porte le pragma #shouldReturn et que la dernière instruction de l'AST de la méthode n'est pas un retour.
- **group** — Retourne 'Bugs'.
- **name** — Retourne 'Method should explicitly return a value'.
- **severity** — Retourne #error : considéré comme un véritable bug, pas seulement un problème de style.

## ReThemeAccessRule

*Sous-classe de ReAbstractRule — groupe « API Hints »*

Signale l'accès direct au thème d'interface via 'Smalltalk ui theme' dans les sous-classes de Morph ou de Model (il faudrait passer par l'API de thème appropriée).

- **check: aMethod forCritiquesDo: aCriticBlock** — Ne s'applique qu'aux méthodes de classes héritant de Morph ou de Model ; recherche les envois de #theme dont le récepteur correspond textuellement à 'Smalltalk ui' et produit une critique pour chacun.
- **group** — Retourne 'API Hints'.
- **name** — Retourne 'Access to UI theme.'

# 2. ReCritiqueEvent — hiérarchie des événements d'annonce

Ces classes définissent les événements (announcements, selon le framework d'annonces de Pharo) diffusés par ReSystemAnnouncer tout au long du cycle de vie d'une critique : détection, correction automatique, bannissement, consultation de sa description.

## ReMultipleCritiquesEvent

*Sous-classe de ReCritiqueEvent — ivar : critiques*

Événement de base portant une collection de critiques (plusieurs critiques à la fois).

### ReEntityCriticized

*Sous-classe de ReMultipleCritiquesEvent*

Événement annoncé lorsqu'une entité (méthode, classe...) a été entièrement vérifiée et a produit une liste de critiques.

## ReSingleCritiqueEvent

*Sous-classe de ReCritiqueEvent — ivar : critique*

Événement de base portant une seule critique.

- **critique** — Accesseur de la variable d'instance 'critique'.
- **critique: aCritique** — Mutateur de la variable d'instance 'critique'.

### ReCritiqueAutofixed

*Sous-classe de ReSingleCritiqueEvent*

Événement annoncé lorsque la correction suggérée par une critique a été appliquée automatiquement.

### ReCritiqueBanned

*Sous-classe de ReSingleCritiqueEvent — ivar supplémentaire : entityBanLevel*

Événement annoncé lorsqu'une critique est bannie (supprimée) à un certain niveau d'entité (méthode, classe ou paquet).

### ReCritiqueDescriptionViewed

*Sous-classe de ReSingleCritiqueEvent*

Événement annoncé lorsque l'utilisateur consulte la description détaillée d'une critique dans l'interface graphique.

# 3. ReExceptionStrategy — stratégies de gestion des erreurs

Ces trois classes implémentent le patron de conception « Stratégie » pour décider comment le moteur de règles doit réagir lorsqu'une erreur est levée pendant l'exécution d'une règle. Aucun corps de méthode n'est fourni dans l'extrait analysé : ce sont des sous-classes marqueurs dont le comportement concret est défini (ou hérité) ailleurs dans le paquet.

## ReIgnoreExceptionStrategy

Stratégie qui ignore silencieusement l'exception rencontrée.

## ReLogExceptionStrategy

Stratégie qui journalise (log) l'exception plutôt que de faire échouer l'exécution.

## RePassExceptionStrategy

Stratégie qui laisse remonter (pass) l'exception à l'appelant, sans la traiter localement.

# 4. ReFeedbacker

Seul l'en-tête de cette classe apparaît dans le document source, sans aucune méthode. D'après son nom et son rôle dans l'écosystème Renraku, il s'agit vraisemblablement du composant chargé de retourner un retour d'information (feedback) à l'utilisateur après l'exécution des règles — son implémentation détaillée n'est pas fournie dans l'extrait analysé.

# 5. ReImageRuleBaner et RePersistentImageRuleBaner

Ces classes gèrent le bannissement (la suppression volontaire) d'une règle au niveau de l'image Pharo tout entière, c'est-à-dire pour l'ensemble du système, et non pour une seule classe ou un seul paquet.

## ReImageRuleBaner

Classe de base représentant le niveau de bannissement « image entière ».

## RePersistentImageRuleBaner

*Sous-classe de ReImageRuleBaner*

Variante dont les bannissements sont persistés (conservés même après redémarrage / sauvegarde de l'image), contrairement à ceux de la classe de base qui ne durent probablement que le temps de la session.

# 6. ReInvocationSequenceRuleSequence

Objet valeur (Object subclass) représentant une contrainte « avant/après » utilisée par ReInvocationSequenceRule : un ensemble de sélecteurs devant précéder (before) et un ensemble devant suivre (after) un envoi de message donné.

## ReInvocationSequenceRuleSequence (détail des méthodes)

*Variables d'instance : before, after*

- **after** — Accesseur de l'ensemble des sélecteurs 'après'.
- **afterDescriptionString** — Retourne une description lisible de l'ensemble 'after', via descriptionStringFor:.
- **afterMatches: aSymbol** — Vrai si le sélecteur donné appartient à l'ensemble 'after'.
- **before** — Accesseur de l'ensemble des sélecteurs 'avant'.
- **beforeDescriptionString** — Retourne une description lisible de l'ensemble 'before', via descriptionStringFor:.
- **beforeMatches: aSymbol** — Vrai si le sélecteur donné appartient à l'ensemble 'before'.
- **descriptionStringFor: aSet** — Formate un ensemble de sélecteurs sous forme de texte : '#selecteur message' s'il n'y en a qu'un, ou 'any one of #a, #b messages' sinon.
- **initializeBefore: beforeSymols after: afterSymbols** — Normalise les arguments reçus (symbole unique ou collection) en tableaux, stockés dans before et after.

# 7. ReMethodSourceCleaner

Classe utilitaire chargée de « nettoyer » le texte source d'une méthode (utilisée notamment par ReCompactSourceCodeRule) en supprimant les lignes vides superflues en fin de méthode, le point final éventuel, et les espaces/tabulations de fin de ligne.

## ReMethodSourceCleaner (détail des méthodes)

*Variables d'instance : sourceLines, keepFinalDot*

- **clean** — Orchestre le nettoyage complet : removeTrailingSeparatorLines, puis removeTrailingDot, puis removeTrailingSeparatorsInAllLines.
- **contents** — Reconstruit le texte source nettoyé en une seule chaîne, en joignant les lignes par un retour chariot (Character cr).
- **initFromSource: aString** — Découpe le texte source en objets ReSourceCodeLine (un par ligne, numérotée), stockés dans une DoubleLinkedList.
- **initialize** — Initialise keepFinalDot à partir du paramètre de classe par défaut (self class keepFinalDotInMethodBody).
- **keepFinalDot / keepFinalDot:** — Accesseur et mutateur du booléen indiquant si le point final doit être conservé.
- **removeTrailingDot** — Supprime le point final de la dernière ligne, sauf si keepFinalDot est vrai.
- **removeTrailingSeparatorLines** — Supprime en boucle les lignes finales entièrement vides (ne contenant que des séparateurs).
- **removeTrailingSeparatorsInAllLines** — Supprime les espaces/tabulations de fin sur chacune des lignes du texte.

# 8. ReProperty — hiérarchie des critiques et propriétés

ReProperty est la classe de base des objets « propriété » attachés à une critique ou à une règle. Ses sous-classes principales sont ReAbstractCritique (et ses propres sous-classes, qui représentent les critiques concrètes produites par les règles), ReExceptionProperty et RePropertyAction.

## ReAbstractCritique

*Sous-classe de ReProperty — ivars : rule, tinyHint*

Classe de base commune à toutes les critiques concrètes : mémorise la règle à l'origine de la critique et un indice court optionnel (tinyHint).

### ReCompactSourceCodeCritic

*Sous-classe de ReAbstractCritique*

Critique produite par ReCompactSourceCodeRule lorsqu'un nettoyage du code source est nécessaire.

- **change** — Méthode abstraite (subclassResponsibility) donnant accès à la modification concrète proposée.
- **providesChange** — Retourne vrai : cette critique propose une correction automatique.

### ReInvocationOrderCritique

*Sous-classe de ReAbstractCritique — ivars : targetMethodName, condition*

Classe de base commune aux critiques d'ordre d'invocation (avant/après), produites par ReInvocationSequenceRule.

- **initializeRule: aRule sourceAnchor: anAnchor method: aString condition: aCondition** — Initialise la règle et l'ancre (héritées), puis mémorise le nom de la méthode fautive (targetMethodName) et la contrainte de séquence concernée (condition).
- **providesChange** — Retourne faux : ce type de critique ne propose pas de correction automatique.

### ReLaterInvocationCritique

*Sous-classe de ReInvocationOrderCritique*

Critique émise quand un message n'est pas suivi du/des message(s) requis par la règle.

- **description** — Construit une phrase explicative citant targetMethodName et les messages attendus après (condition afterDescriptionString).
- **title** — Retourne un titre court pour l'interface, tronqué à 30 caractères avec points de suspension.

### RePriorInvocationCritique

*Sous-classe de ReInvocationOrderCritique*

Critique émise quand un message est envoyé sans avoir été précédé du/des message(s) requis par la règle.

- **description** — Construit une phrase explicative indiquant qu'il faut appeler les messages attendus (condition beforeDescriptionString) avant targetMethodName.
- **title** — Retourne un titre court pour l'interface, tronqué à 30 caractères avec points de suspension.

## ReExceptionProperty

*Sous-classe de ReProperty — ivars : stack, message*

Capture les informations d'une erreur (pile d'appel et message) survenue pendant l'exécution d'une règle, afin de pouvoir la diagnostiquer plutôt que de la laisser interrompre le moteur.

## RePropertyAction

*Sous-classe d'Object — ivars : action, icon, description (voir section 9)*

Représente une action rapide réutilisable (par exemple un bouton dans l'interface) pouvant être associée à une critique.

# 9. RePropertyAction — détail des méthodes

Cette section détaille les méthodes de la classe RePropertyAction introduite ci-dessus : elle encapsule un bloc d'action, une icône et une description, typiquement affichés comme bouton d'action associé à une critique dans l'interface graphique.

## RePropertyAction (détail)

- **actOnCritic: aCritic ofEntity: anEntity** — Évalue le bloc 'action' stocké, en lui passant la critique et l'entité concernées (via cull:, ce qui tolère un bloc à 0, 1 ou 2 arguments).
- **action / action:** — Accesseur et mutateur du bloc d'action.
- **description / description:** — Accesseur et mutateur du texte descriptif de l'action.
- **icon / icon:** — Accesseur et mutateur de l'icône (un Form / une image) représentant l'action.
- **initialize** — Définit des valeurs par défaut : une icône verte unie de 16x16 pixels, une description vide, et un bloc d'action ne faisant rien (no-op).

# 10. ReRewriteWrapper

Classe abstraite unifiant les deux manières de produire un nœud de remplacement pour une règle de réécriture (ReNodeRewriteRule) : soit un arbre statique déjà construit (typiquement via une sous-classe ReRewriteTreeWrapper), soit un bloc calculant dynamiquement le remplacement (typiquement via ReRewriteBlockWrapper) — ces deux sous-classes sont référencées dans le code mais non détaillées dans l'extrait fourni.

## ReRewriteWrapper (détail)

- **rewriten: aNode with: aMap** — Méthode abstraite (subclassResponsibility) : étant donné le nœud correspondant au motif et le dictionnaire des correspondances de jokers, doit retourner le nœud de remplacement.

# 11. ReRuleManager

Gère la collection de règles à exécuter et sait les classer selon le type d'entité qu'elles vérifient (méthode, nœud d'AST, classe ou paquet).

## ReRuleManager (détail)

*Variable d'instance : rules*

- **allRules** — Retourne la collection brute de toutes les règles gérées.
- **classRules** — Filtre allRules pour ne garder que les règles dont la classe déclare checksClass (règles s'appliquant à des classes).
- **initializeRules: aCollection** — Stocke une copie de la collection de règles fournie.
- **methodRules** — Filtre allRules pour ne garder que les règles s'appliquant à des méthodes (checksMethod).
- **nodeRules** — Filtre allRules pour ne garder que les règles s'appliquant à des nœuds d'AST (checksNode).
- **packageRules** — Filtre allRules pour ne garder que les règles s'appliquant à des paquets (checksPackage).

# 12. ReSettings

Classe de configuration globale de Renraku (Object subclass: #ReSettings). Aucune méthode d'instance n'apparaît dans l'extrait fourni ; le code utilisé ailleurs dans le document (par exemple ReSettings showErrors, invoqué par ReSmalllintChecker) laisse supposer qu'elle expose des paramètres au niveau classe tels que le comportement à adopter en cas d'erreur pendant l'exécution d'une règle, ou la conservation du point final dans le nettoyage du code source.

# 13. ReSmalllintChecker — le moteur d'exécution des règles

C'est le composant central qui orchestre réellement l'exécution des règles sur un paquet, une classe ou une méthode, collecte les critiques produites, et s'articule avec ReRuleManager, ReSourceAnchor et le mécanisme de bannissement/manifeste (TheManifestBuilder).

## ReSmalllintChecker (détail)

*Variables d'instance notables : critiques, environment, rule, manifestClassCache, methodRules, nodeRules, classRules*

- **addCritique: aCritique** — Ajoute une critique à l'ensemble interne des critiques trouvées.
- **builderManifestClass** — Retourne TheManifestBuilder, la classe utilisée pour persister les bannissements et faux positifs.
- **checkClass: aClass** — Si la classe est dans le périmètre de l'environnement analysé, exécute toutes les classRules sur elle.
- **checkMethodsForClass: aClass** — Pour chaque sélecteur de la classe : exécute methodRules sur la méthode compilée, puis nodeRules sur chaque nœud de son AST (en copiant l'AST au préalable pour les règles de réécriture, qui le modifient).
- **checkPackage: aPackage** — Si le paquet est dans le périmètre analysé, exécute toutes les packageRules sur lui.
- **cleanAllManifest** — Parcourt le manifeste de chaque paquet de l'environnement et appelle cleanUp sur chacun (purge des bannissements obsolètes).
- **criticsOf: aRule** — Retourne les critiques déjà produites spécifiquement par la classe de la règle donnée.
- **falsePositiveOf: aRule** — Filtre les critiques d'une règle pour ne garder que celles dont l'entité a été marquée comme faux positif dans son manifeste.
- **getCritiquesAbout: anEntity by: rules** — Exécute une liste de règles sur une entité, en interceptant les erreurs éventuelles (relancées seulement si ReSettings showErrors est vrai), et ajoute chaque critique trouvée.
- **initialize** — Initialise le cache de manifestes (Dictionary) et réinitialise l'ensemble des critiques (resetResult).
- **isFalsePositive:forRuleId:versionId: / isToDo:forRuleId:versionId: / isTruePositive:forRuleId:versionId:** — Interrogent le manifeste approprié pour savoir si une critique donnée est respectivement un faux positif, un « à faire » (to-do), ou un vrai positif.
- **manifestBuilderOf: aPackage** — Demande au paquet son constructeur de manifeste pour ce checker (manifestBuilderForRuleChecker:).
- **manifestBuilderOfClass: / manifestBuilderOfMethod: / manifestBuilderOfPackage:** — Recherches mises en cache (via manifestClassCache) du TheManifestBuilder responsable d'une classe, d'une méthode ou d'un paquet donné.
- **reParseRule** — Reconstruit les ensembles methodRules, nodeRules et classRules à partir de la règle composite courante.
- **rejectClassesOf: / rejectRulesOf:** — Consultent le manifeste du paquet pour savoir quelles classes ou quelles règles doivent être exclues de l'analyse.
- **resetResult** — Réinitialise l'ensemble 'critiques' (Set vide).
- **rule** — Retourne soit les règles élémentaires (leaves) si la règle courante est une RBCompositeLintRule composite, soit la règle unique sinon — normalise l'accès.
- **run** — Reconstruit la classification des règles (reParseRule) puis délègue à super run (mécanisme hérité de SmallLint).
- **runRules: aCompositeRule onPackage: aPackage** — Configure la règle et l'environnement (RBPackageEnvironment sur le paquet donné) puis lance l'exécution (run).
- **runRules:onPackage:withoutTestCase:** — Variante qui exclut en plus les classes de manifeste et, si demandé, les classes de test (TestCase) de l'environnement analysé.
- **toDoOf: aRule** — Filtre les critiques d'une règle pour ne garder que celles marquées « à faire » dans leur manifeste.

# 14. ReSourceAnchor — ancrage d'une critique dans le code source

Une « ancre » relie une critique à un emplacement précis du code source, afin que l'interface graphique puisse le surligner ou que le moteur de refactoring puisse le remplacer.

## ReSourceAnchor

*Sous-classe d'Object — ivar : sourceEntity*

Classe de base : ancre désignant une entité entière (classe, méthode, paquet).

- **initializeEnitity: anEntiy interval: anInterval** — Initialise l'entité source puis mémorise un intervalle (utilisé par les sous-classes).
- **providesInterval** — Retourne vrai par défaut : déclare que cette ancre peut fournir un intervalle textuel (bien que seules les sous-classes à intervalle en fournissent réellement un exploitable).

## ReIntervalSourceAnchor

*Sous-classe de ReSourceAnchor — ivar : interval*

Ancre pointant sur une plage précise de caractères dans le texte source d'une méthode.

- **interval** — Accesseur de l'intervalle mémorisé.

## ReSearchStringSourceAnchor

*Sous-classe de ReSourceAnchor — ivar : searchString*

Ancre qui se localise en recherchant une chaîne de caractères dans le source, plutôt qu'en mémorisant un intervalle fixe.

### ReVarSearchSourceAnchor

*Sous-classe de ReSearchStringSourceAnchor*

Spécialisation de l'ancre par recherche de chaîne, dédiée à la localisation d'une occurrence de nom de variable.

- **interval** — Réutilise l'intervalle hérité, mais en retirant le dernier caractère (pour exclure un séparateur ou une apostrophe de fin).
- **modify: aString for: aStringCollection** — Formate le nom de variable avec les guillemets/espaces adaptés selon sa position parmi les autres variables déclarées de la classe (seule occurrence, première, dernière, ou au milieu de la liste).
- **searchString** — Calcule la chaîne exacte à rechercher, en l'entourant correctement de guillemets si elle correspond à une variable d'instance ou de classe.

# 15. ReSourceCodeLine

Objet valeur représentant une seule ligne de code source, utilisé par ReMethodSourceCleaner pour effectuer son nettoyage ligne par ligne.

## ReSourceCodeLine (détail)

*Variables d'instance : contents, lineNumber*

- **contents / contents:** — Accesseur et mutateur du texte de la ligne.
- **lineNumber / lineNumber:** — Accesseur et mutateur du numéro de ligne (base 1).
- **onlyContainsSeparators** — Retourne vrai si la ligne ne contient que des caractères de séparation (espaces, tabulations).
- **removeTrailingDot** — Supprime un point final éventuel de la ligne.
- **removeTrailingSeparators** — Supprime les espaces et tabulations en fin de ligne.

# 16. ReSystemAnnouncer

Instance de l'annonceur (Announcer) du système Pharo utilisée par Renraku pour diffuser les événements liés au cycle de vie des critiques décrits en section 2.

## ReSystemAnnouncer (détail)

- **notifyCritique: aCritique descriptionViewedFor: anEntity** — Diffuse un événement ReCritiqueDescriptionViewed, renseigné avec la critique et l'entité concernées.
- **notifyCritique: aCritique of: anEntity bannedOn: anEntityBanLevel** — Diffuse un événement ReCritiqueBanned, avec la critique, l'entité et le niveau de bannissement.
- **notifyEntity: anEntity criticizedWith: critiques** — Diffuse un événement ReEntityCriticized regroupant toutes les critiques trouvées pour l'entité.

# 17. ClassDescription — extension pour la gestion du bannissement

Renraku ajoute directement des méthodes à la classe native ClassDescription de Pharo (extension de classe) pour permettre à n'importe quelle classe du système de participer au mécanisme de critiques et de bannissement de règles.

## ClassDescription (méthodes ajoutées)

- **ban: aCritique** — Diffuse la notification de bannissement via ReSystemAnnouncer, puis l'enregistre dans le manifeste de la classe via TheManifestBuilder.
- **banChecksForValidation** — Retourne la liste des entités (la classe elle-même et sa classe de méthode) dont le statut de bannissement doit être vérifié lors d'une validation.
- **banLevelCandidates** — Retourne la chaîne des entités pouvant bannir une règle pour cette entité (elle-même, suivie des candidats de sa classe de méthode).
- **critiques** — Rassemble les critiques concernant cette entité, en combinant les critiques classiques et les critiques au niveau des nœuds (via ReCriticEngine).
- **overriddenMethods** — Recherche, parmi les superclasses, toutes les méthodes portant le même sélecteur que celle-ci (méthodes redéfinies).
- **reBanName** — Retourne un nom d'affichage suffixé par ' (Method)'.
- **suitableRulesFrom: aReRuleManager** — Retourne le sous-ensemble methodRules du gestionnaire de règles, c'est-à-dire les règles applicables à ce type d'entité.

*Remarque : ce même bloc de méthodes (ban:, banChecksForValidation, banLevelCandidates, critiques, overriddenMethods, reBanName, suitableRulesFrom:) réapparaît identique plus loin (section 18) sur la classe CompiledMethod : Renraku étend ainsi de façon symétrique ClassDescription et CompiledMethod avec le même protocole de bannissement.*

# 18. RBLintRule et extensions diverses (Refactoring-Critics, Traits, Kernel)

Cette section du fichier regroupe plusieurs définitions de classes indépendantes appartenant à d'autres paquets liés (Refactoring-Critics, TraitsV2-Base, Kernel-Methods), avec lesquels Renraku s'interface.

## RBParseTreeLintRule

*Sous-classe de RBBasicLintRule — ivar : matcher*

Fait le pont entre les règles à base de motifs de Renraku et l'ancien cadre RBLintRule (SmallLint historique de Pharo).

- **check: aMethod forCritiquesDo: aCriticBlock** — Si le matcher mémorisé peut s'appliquer à la méthode (canMatchMethod:), exécute la correspondance sur son AST ; si un nœud est trouvé, produit une critique pour ce nœud.
- **critiqueFor: aNode about: aMethod** — Construit une ReTrivialCritique ancrée soit sur la méthode entière (si le nœud trouvé n'est pas un RBProgramNode), soit sur l'intervalle précis du nœud trouvé.

## RBTransformationRule

*Sous-classe de RBLintRule — ivars : rewriteRule, builder, class*

Règle de lint qui sait, en plus de détecter un problème, produire directement une transformation/réécriture du code fautif.

- **check: aMethod forCritiquesDo: aCriticBlock** — Si basicCheck: réussit, produit une ReTransformationCritique portant l'arbre de réécriture (rewriteRule tree).
- **suggestedCode: aCritic** — Recherche, parmi les changements enregistrés, celui correspondant à la classe et au sélecteur de la critique donnée, et retourne son code source proposé.

## MetaclassForTraits

*Sous-classe de TraitedMetaclass — ivar : users (paquet TraitsV2-Base)*

Classe du noyau Pharo gérant les métaclasses supportant les traits ; étendue ici uniquement avec la méthode de bannissement suivante.

- **reBanName** — Retourne un nom d'affichage suffixé par ' (Trait)'.

## Trait

*Sous-classe de Class — ivar : users (paquet TraitsV2-Base)*

Classe du noyau Pharo représentant un trait ; étendue ici de la même façon.

- **reBanName** — Retourne un nom d'affichage suffixé par ' (Trait)'.

## CompiledMethod

*Sous-classe (variableByteSubclass) de CompiledCode — paquet Kernel-Methods*

Classe native représentant une méthode compilée ; étendue par Renraku avec le même protocole de bannissement que ClassDescription (voir section 17), afin qu'une méthode individuelle puisse elle aussi être bannie / critiquée directement.

- **ban: aCritique** — Diffuse la notification de bannissement puis l'enregistre dans le manifeste de la méthode via TheManifestBuilder.
- **banChecksForValidation** — Retourne la liste des entités (elle-même et sa classe de méthode) à vérifier lors d'une validation de bannissement.
- **banLevelCandidates** — Retourne la chaîne des entités pouvant bannir une règle pour cette méthode.
- **critiques** — Rassemble les critiques (classiques + nœuds) concernant cette méthode via ReCriticEngine.
- **overriddenMethods** — Recherche, parmi les superclasses de sa classe, les méthodes portant le même sélecteur.
- **reBanName** — Retourne un nom d'affichage suffixé par ' (Method)'.
- **suitableRulesFrom: aReRuleManager** — Retourne le sous-ensemble methodRules du gestionnaire de règles.

# 19. RBProgramNode — nœuds d'AST pour les motifs de réécriture

Ces sous-classes de nœuds d'AST (RBProgramNode) représentent les jokers (wildcards) utilisés dans les motifs de correspondance du moteur de réécriture (ReNodeRewriteRule, RePatternCodeRule) — par exemple des expressions du type `@objet` ou `@.Statements` dans les motifs vus en section 1.

## RBPatternBlockNode

*Sous-classe de RBBlockNode — ivar : valueBlock (paquet AST-Core-Pattern)*

Joker représentant un bloc de code dans un motif.

- **pattern** — Retourne le texte source correspondant à l'intervalle du nœud (la portion de motif reconnue).

## RBPatternMessageNode

*Sous-classe de RBMessageNode — ivars : isList, isCascadeList*

Joker représentant un envoi de message dans un motif.

- **pattern** — Retourne le sélecteur du message comme représentation du motif.

## RBPatternMethodNode

*Sous-classe de RBMethodNode — ivar : isList*

Joker représentant une méthode entière dans un motif.

- **pattern** — Retourne le sélecteur de la méthode comme représentation du motif.

## RBPatternPragmaNode

*Sous-classe de RBPragmaNode — ivar : isList*

Joker représentant un pragma dans un motif.

- **pattern** — Retourne le sélecteur du pragma comme représentation du motif.

## RBPatternVariableNode

*Sous-classe de RBVariableNode — ivars : recurseInto, isList, isLiteral, isStatement, isAnything*

Le joker le plus général (motifs du type `@quelquechose`), capable de représenter aussi bien une variable, un littéral, une instruction ou « n'importe quoi ».

- **pattern** — Retourne le nom du nœud comme représentation du motif.

# 20. RGMethodDefinition

RGMethodDefinition appartient au méta-modèle « Ring » de Pharo (une représentation réflexive, indépendante de l'image en cours, des éléments du code). Renraku l'étend pour lui permettre de participer au même protocole de critiques que les vraies méthodes compilées.

## RGMethodDefinition (détail)

- **critiques** — Délègue simplement à la méthode compilée sous-jacente (self compiledMethod critiques), pour que les méthodes du modèle Ring bénéficient du même protocole de critiques que CompiledMethod.

# 21. RPackage — extension pour les règles au niveau paquet

Comme ClassDescription et CompiledMethod, la classe RPackage (représentant un paquet Pharo) est étendue par Renraku pour pouvoir participer au protocole de bannissement/critique, mais à l'échelle d'un paquet entier.

## RPackage (méthodes ajoutées)

- **ban: aCritique** — Diffuse la notification de bannissement puis l'enregistre dans le manifeste du paquet via TheManifestBuilder.
- **banChecksForValidation** — Retourne un tableau vide : un paquet n'a pas de niveau « parent » supplémentaire à vérifier lors d'une validation.
- **banLevelCandidates** — Retourne le paquet lui-même, suivi des deux baniers au niveau image (ReImageRuleBaner et RePersistentImageRuleBaner), puisqu'une règle bannie à l'échelle d'un paquet peut aussi l'être finalement à l'échelle de toute l'image.
- **critiques** — Délègue à ReCriticEngine critiquesOf: pour obtenir les critiques concernant ce paquet.
- **package** — Retourne self, afin que le paquet puisse être traité de façon uniforme partout où un accesseur 'package' est attendu.
- **reBanName** — Retourne un nom d'affichage suffixé par ' (Package)'.
- **suitableRulesFrom: aReRuleManager** — Retourne le sous-ensemble packageRules du gestionnaire de règles.

# 22. SmalltalkImage et TheManifestBuilder — sommet de la hiérarchie et persistance des bannissements

SmalltalkImage représente l'image Pharo tout entière : c'est le niveau ultime de la chaîne de bannissement (voir RPackage>>banLevelCandidates en section 21). TheManifestBuilder est, lui, la classe responsable de la persistance effective des bannissements et faux positifs (le « manifeste » d'un paquet).

## SmalltalkImage (méthode ajoutée)

- **reBanName** — Retourne la chaîne fixe 'The whole image' : c'est le nom affiché lorsqu'une règle est bannie pour l'image entière.

## TheManifestBuilder (détail)

- **banRule: aRule for: anEntity** — Calcule un identifiant stable de la règle (uniqueIdentifierName + numéro mineur de version), crée si nécessaire l'entrée « faux positif » correspondante, puis enregistre l'entité comme faux positif pour cette règle/version.
- **bansRule: aRule for: anEntity** — Vérifie (sans rien modifier) si l'entité donnée est déjà enregistrée comme bannissant la règle/version donnée, en interrogeant containsFalsePositive:onRule:version:.

# Synthèse

Le package Renraku met en œuvre un moteur complet d'analyse statique de code pour Pharo : des règles (ReAbstractRule) détectent des problèmes et produisent des critiques (ReProperty / ReAbstractCritique) ancrées précisément dans le code (ReSourceAnchor) ; ReSmalllintChecker et ReRuleManager orchestrent leur exécution sur des paquets, classes et méthodes ; les résultats peuvent être bannis (supprimés) à différents niveaux — méthode, classe, paquet ou image entière — via TheManifestBuilder, et tout le cycle de vie est notifié aux observateurs par ReSystemAnnouncer. Un sous-ensemble spécialisé de règles (RePatternCodeRule, ReNodeRewriteRule et les nœuds RBPattern*) permet en outre de détecter des motifs de code précis et de proposer, voire d'appliquer automatiquement, une réécriture corrective.
