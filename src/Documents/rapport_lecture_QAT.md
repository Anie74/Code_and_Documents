# Rapport de lecture : Quality-Aware Tooling (Yuriy Tymchuk, 2017)

*Thèse de doctorat, Université de Berne (direction : Oscar Nierstrasz). Rapport rédigé pour le mémoire « Sélection de règles de qualité fondée sur le profil du développeur ».*

## 1. Périmètre de la lecture

La thèse comporte sept chapitres : introduction, état de l’art (chapitre 2), Renraku (chapitre 3), QualityAssistant (chapitre 4), impact sur l’écosystème (chapitre 5), qualité dans d’autres outils (chapitre 6) et conclusions (chapitre 7).

J’ai lu le texte intégral de l’introduction et des chapitres 2 et 3, ainsi que du chapitre 4.

## 2. L’idée centrale de la thèse

Tymchuk part d’un constat : les analyseurs statiques sont utiles, mais peu utilisés en pratique, parce qu’ils sont des outils séparés qu’il faut lancer volontairement. Il propose le quality-aware tooling qui vise d’intégrer l’analyse statique de façon transparente dans les outils que les développeurs utilisent déjà, d’abord dans l’éditeur de code avec un retour en direct, puis dans d’autres outils.

La thèse défend l’énoncé suivant, que je paraphrase : l’aide automatique à la qualité est une fonction essentielle des outils de développement et améliore l’expérience, à condition que le moteur d’analyse s’adapte au minimum besoin de l’utilisateur. Cette condition d’adaptation est directement le point de départ de ma mémoire.

Le terrain d’étude est Pharo : l’environnement avait déjà un analyseur autonome (SmallLint et son interface CriticBrowser), et les décideurs de Pharo ont accepté d’intégrer les travaux dans l’IDE. Le résultat annoncé est que 90 % des développeurs trouvent le retour en direct utile, que les règles ont été revues, et que l’attitude envers l’analyse statique a changé.

## 3. Chapitre 2 : état de l’art

Le chapitre classe les travaux en trois catégories : les détecteurs de mauvaises pratiques, l’usage de l’analyse statique pour améliorer d’autres outils, et l’évaluation de l’usage de l’analyse statique.

### 3.1 Les analyseurs statiques (section 2.1)

L’auteur distingue deux familles. Les analyseurs légers (Lint, Pylint, Checkstyle, PMD, FindBugs, ESLint, SmallLint et autres) cherchent des motifs suspects, le plus souvent dans l’arbre syntaxique, parfois dans le bytecode ou le texte brut. Ils sont rapides et passent à l’échelle, mais ne garantissent pas de trouver tous les défauts. La seconde famille détecte des code smells avec des métriques, des seuils et de la logique floue (iPlasma, DECOR, approches bayésiennes). Elle est plus lente et ses résultats demandent plus de temps à corriger. Tymchuk choisit l’analyse légère parce qu’elle donne une boucle de retour courte.

Un point mérite attention : les travaux sur les Usage Contracts montrent que les développeurs de règles hésitent à apprendre un nouveau langage pour écrire des règles. Tymchuk en tire le principe d’écrire les règles Smalltalk en Smalltalk. C’est exactement ce que vous faites avec vos règles par niveau.

### 3.2 L’intégration de l’analyse statique (section 2.2)

L’analyse est surtout précieuse lorsqu’elle est intégrée aux outils des tâches courantes :

#### Revue de code

Les relecteurs passent beaucoup de temps sur des problèmes triviaux de conventions, que l’automatisation pourrait traiter (Review Bot). Le Tricorder de Google ajoute des rapports qualité à l’outil de revue, s’appuie sur 16 outils d’analyse et vise moins de 10 % de faux positifs. Le prix est une infrastructure lourde pour exécuter les outils et unifier les résultats. C’est le problème que Renraku veut résoudre avec une API unique.

#### IDE

L’outil inCode de Eclipse détecte des code smells en direct et propose des refactorings, avec de bons résultats mesurés. Eclipse, NetBeans et IntelliJ donnent un retour pendant la frappe, mais aucune étude n’évalue leur utilité ni leur impact.

#### Intégration continue

Quant à SonarQube, avec son extension SonarLint pour le retour en direct, est encore très jeune sur ce point.

#### Analyse au fil de l’eau

Les approches « just-in-time » analysent seulement le code consulté ou édité, ce que QualityAssistant reprend. L’auteur distingue aussi les corrections rapides (quick fixes) des « slow fixes », qui donnent plus d’informations pour résoudre un problème non automatisable.

### 3.3 L’usage de l’analyse statique (section 2.3)

La section explique pourquoi l’analyse est peu utilisée :

- Les créateurs de Coverity constatent que toute gêne (difficulté de lancement, sortie incompréhensible, avertissements contestés) démotive les utilisateurs.
- Johnson et al. interrogent 20 développeurs : les principales causes de non-usage sont le nombre et la mauvaise organisation des faux positifs, le faible soutien du travail en équipe et de la personnalisation, et la compréhension difficile des résultats. Les participants veulent aussi être prévenus dès que le problème apparaît.
- Beller et al. observent que beaucoup de projets open source ont des traces d’analyseurs sans les utiliser régulièrement, et recommandent au minimum une intégration au serveur d’intégration continue.
- Yamashita et Moonen montrent qu’environ un tiers des développeurs interrogés ne connaissent pas la notion de code smell, et que les développeurs réclament un outil temps réel, convivial et personnalisable.
- Palomba et al. indiquent que la gravité d’un smell dépend du contexte du projet. D’autres études montrent que les règles propres à un projet ou à un domaine sont plus utiles que les règles génériques.

La conclusion de l’auteur est que la thèse ne s’obstine pas sur des standards de qualité universels, mais suit les besoins des développeurs avec des règles adaptées à leur domaine.

### 3.4 Lecture critique du chapitre 2

Ce chapitre sert de socle bibliographique solide à votre mémoire. Il manque toutefois ce qui est au cœur de votre sujet : les travaux cités parlent de règles adaptées au projet ou au domaine, mais pas de règles adaptées au niveau de l’utilisateur. Les remarques sur le nombre de faux positifs, la compréhension des messages et la personnalisation (Johnson et al., Yamashita et Moonen) justifient pourtant votre approche : un débutant perçoit autrement la pertinence d’un message qu’un expert. C’est un argument à développer dans votre chapitre 1.

## 4. Chapitre 3 : Renraku

### 4.1 Motivation et objectifs

Chaque intégration de l’analyse statique dans un outil (éditeur, navigateur de méthodes, débogueur, inspecteur) demande de lancer les analyseurs, de collecter les résultats et de les afficher. Pour réduire ce coût, Tymchuk conçoit un modèle unifié. Deux objectifs : faciliter la réutilisation des analyseurs existants par les outils, et l’intégration d’analyseurs personnalisés dans les outils existants. Le nom signifie en japonais communication, connexion, coordination. Renraku est pensé comme un framework orienté objet : une règle est une sous-classe, un rapport est un objet, et non du texte brut comme chez la plupart des analyseurs.

### 4.2 La triade qualité (section 3.1)

Le modèle repose sur trois concepts :

- La critique est un rapport unique sur la qualité, qui relie une cible à une règle. Elle peut être spécialisée (explications, suggestions, outils de correction). Elle n’est pas forcément négative : à la limite, ce peut être un simple lien entre une règle et une cible. Plusieurs critiques peuvent relier la même règle et la même cible, par exemple une par variable d’instance inutilisée d’une classe.
- La règle définit une irrégularité de qualité. C’est une fonction qui reçoit une entité et renvoie une ou plusieurs critiques. Elle ne garde pas de référence directe vers la cible. Elle choisit aussi la critique la plus adaptée.
- La cible est le morceau de code visé. Elle doit pouvoir donner ses critiques, de sorte qu’obtenir l’information de qualité revienne à demander à la méthode ou à la classe ses critiques.

On retrouve ces trois concepts dans la documentation actuelle de Pharo 15, où la « cible » s’appelle « entité ».

### 4.3 La critique (section 3.2)

#### Property

Les critiques héritent d’une classe plus générale qui représente toute propriété externe d’un morceau de code (note, entrée de suivi de tickets, couverture de tests) : un titre et une icône. La critique en déduit son titre du nom de la règle, son icône de la sévérité et sa description de la justification.

#### Ancres de source

Une critique pointe vers une ancre qui connaît l’entité et l’intervalle de code concerné. Deux variantes : l’intervalle stocké directement (utile avec les nœuds de l’AST, qui connaissent leur intervalle) et la recherche d’une sous-chaîne dans le source, moins précise si la chaîne apparaît plusieurs fois. Le deuxième cas existe parce que certaines règles travaillent sur le bytecode, plus rapide que l’AST, mais sans positions.

#### Tiny hint

Un court texte placé devant le titre pour localiser le problème, par exemple le nom de la variable inutilisée entre crochets.

#### Actions personnalisées

Pour que les outils n’aient pas à être modifiés à chaque nouveau type de critique, chaque propriété porte des `PropertyAction` (icône, description, fonction), que l’outil liste et exécute. Les critiques de base offrent la description et le bannissement (ban), que l’auteur utilise surtout pour ignorer les faux positifs.

#### Critiques spécialisées

Critique de transformation (correction automatique avec différence affichée), méthode manquante, ordre d’appel des messages.

### 4.4 La règle (section 3.3)

#### Héritage de SmallLint

Ils sont de mêmes propriétés (nom, justification, groupe, sévérité), mais la différence fondamentale est la facilité d’usage : avec SmallLint, chaque règle devait être interrogée à sa manière, ce qui donnait des rapports de mauvaise qualité dans les outils. Avec Renraku, la règle est une boîte noire qui reçoit une cible et produit des critiques.

#### Approche en flot

La méthode principale reçoit l’entité et un bloc évalué pour chaque critique (avec `cull:`). Avantages cités : environ 10 % de gain avec Pharo 6 (pas de collection créée pour chaque paire méthode-règle), exécution des règles lourdes dans un processus séparé avec mise à jour de l’interface au fil des critiques, et arrêt possible à la première critique. Des méthodes de confort renvoient des collections.

#### Déclaration d’intérêt

Les méthodes de classe `checksMethod`, `checksClass`, `checksPackage` et `checksNode` permettent aux outils de regrouper les règles par type d’entité. L’auteur discute des variantes écartées (intérêt pour les métaclasses, tableau de types) et note que la plupart des règles par nœud commencent par tester le type du nœud.

#### Découverte des règles

Toutes les sous-classes de la règle abstraite sont collectées. Une règle abstraite peut se masquer avec `isVisible`, et les règles livrées avec un framework sont automatiquement disponibles.

#### Règles spécialisées

Règles d’ordre d’appel, règles par nœud (qui déclarent d’elles-mêmes l’intérêt pour les nœuds) et règles par motif avec transformation. Dans SmallLint, ces dernières parcouraient tout l’AST et le réécrivaient. Dans Renraku, elles vérifient un seul nœud et stockent le nœud de remplacement : l’AST peut être parcouru une seule fois pour toutes les règles, avec un gain annoncé d’environ 40 %.

### 4.5 La cible, la compatibilité et les exemples (sections 3.4 à 3.7)

#### Cible

Toute entité de code répond au message `critiques`, qui applique toutes les règles actives. Une méthode plus générale, `externalProperties`, agrège toutes les propriétés externes déclarées avec une pragma.

#### Compatibilité avec SmallLint

Les règles SmallLint peuvent devenir des règles Renraku via quelques méthodes d’extension, et l’inverse via un enveloppeur. La migration est volontairement lente et manuelle, règle par règle, car certaines règles et certains outils (le serveur d’intégration continue de Pharo) attendent encore l’ancien modèle.

#### Créer une règle

L’exemple d’une règle qui vérifie qu’une configuration de projet du catalogue Pharo définit les méthodes requises suit les étapes que votre documentation décrit aujourd’hui, et montre comment passer d’une critique simple à une critique plus précise (méthode manquante avec création d’un squelette).

#### Créer un outil

Une visualisation Roassal où la couleur des classes dépend du nombre de critiques s’obtient avec deux messages. Le message est que l’intégration est peu coûteuse pour l’auteur d’un outil.

### 4.6 Au-delà des analyseurs standards (section 3.8) et conclusions

Trois prototypes montrent la souplesse du modèle : le lien avec les entrées de suivi de tickets (intégré en une journée), la couverture de tests SmartTest (deux jours, par un étudiant) et une propriété d’exception qui rend visibles les erreurs des analyseurs sans interrompre le développeur (environ une heure). L’auteur en tire une remarque intéressante : les avertissements d’analyse statique et les résultats de tests appartiennent à la même catégorie de rapports sur la qualité. Il indique que QualityAssistant, bâti sur Renraku, a été utilisé par environ 500 développeurs pendant trois ans, et reconnaît que le modèle n’a été éprouvé que sur des cas simples.
