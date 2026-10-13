##### Ce document contient la liste de toutes les règle du package Renraku #####

#### 1. ReAbstractRule



accessing

    group

        ^ 'Unclassified rules'


name
	"Answer a human readable name of the rule."

	^ self subclassResponsibility

rationale
	"Answer an explanation of the rule, usually in one line. Long description can be obtained using longDescription."

	^ self class comment


severity
	"Answer the severity of issues reported by this rule. This method should return one of #error, #warning, or #information."

	^ #warning

compatibility

	resetResult

helpers
	critiqueFor: anEntity

	^ ReTrivialCritique
		withAnchor: (self anchorFor: anEntity)
		by: self

	anchorFor: anEntity

	^ ReSourceAnchor entity: anEntity

running
	basicCheck: anEntity
	"If you end here, you should most likely implement this in a subclass"

	^ self shouldBeImplemented


check: anEntity

	| critiques |
	critiques := OrderedCollection new.
	self
		check: anEntity
		forCritiquesDo: [ :critique | critiques add: critique ].
	^ critiques

check: anEntity forCritiquesDo: aCriticBlock
	"Accepts an entity and a block which could be evaluated for each detected critique
	aCriticBlock may accept one argument: the critique object"

	(self basicCheck: anEntity)
		ifTrue: [ aCriticBlock cull: (self critiqueFor: anEntity) ]


check: anEntity forCritiquesDo: aCriticBlock ifNone: alternativeBlock

	| encounteredCritique |
	encounteredCritique := false.

	self check: anEntity forCritiquesDo: [ :crit |
		encounteredCritique := true.
		aCriticBlock cull: crit ].

	encounteredCritique ifFalse: alternativeBlock

check: anEntity ifNoCritiques: alternativeBlock

	| critiques |
	critiques := OrderedCollection new.
	self
		check: anEntity
		forCritiquesDo: [ :critique | critiques add: critique ]
		ifNone: alternativeBlock.
	^ critiques

running-helpers

	critiqueFor: aClass about: aVarName

	| crit |
	crit := ReTrivialCritique
		withAnchor: (ReVarSearchSourceAnchor
			entity: aClass
			string: aVarName)
		by: self.
	crit tinyHint: aVarName.
	^ crit
testing
	isComposite

	^ false

isRewriteRule

	^ false
utilities
	createTrivialCritiqueOn: method intervalOf: node hint: hint

	^ ReTrivialCritique
		  withAnchor:
		  (ReIntervalSourceAnchor
			   entity: method
			   interval: node sourceInterval)
		  by: self
		  hint: hint


` ReClassSideResetMethodProtocolRule`
	
	basicCheck: aMethod

	^ aMethod isClassSide and: [
		  aMethod selector = #reset and: [
			  aMethod protocol ~= #'class initialization' ] ]

	critiqueFor: aMethod

	| methodSymbol classSymbol |
	methodSymbol := aMethod selector asSymbol.
	classSymbol := aMethod methodClass name asSymbol.

	^ (ReRefactoringCritique
		   withAnchor: (self anchorFor: aMethod)
		   by: self) refactoring: (RBMethodProtocolTransformation
			   protocol: 'class initialization'
			   inMethod: methodSymbol
			   inClass: classSymbol) asRefactoring


group
	^ 'Clean Code'


name

	^ 'Class side methods called #reset should be in the ''class initialization'' 		protocol'



` ReCompactSourceCodeRule ``
	
basicCheck: aMethod

	^ ReMethodSourceCleaner isNecessaryToClean: aMethod allowFinalDot: self class criticizeFinalDotInMethodBody not


check: aMethod forCritiquesDo: aCriticBlock

	(self basicCheck: aMethod)
		ifTrue: [
			aCriticBlock cull: (
				ReCompactSourceCodeCritic
					for: aMethod
					by: self)  ]

group

	^ 'Clean Code'
name

	^ 'Unnecessary characters found in method body'

providesChange

	^ true
severity

	^ #information

` ReExclusivelyBeginnersRule `



` ReInvocationSequenceRule `

add: preSelectors requiresPostSend: postSelectors

	postConditions add: (
		ReInvocationSequenceRuleSequence
			before: preSelectors
			after: postSelectors)

add: postSelectors requiresPreSend: preSelectors

	preConditions add: (
		ReInvocationSequenceRuleSequence
			before: preSelectors
			after: postSelectors)

check: aNode forCritiquesDo: aCriticBlock

	aNode isMethod ifFalse: [ ^ self ].

	(self  preCheck: aNode forCriticsDo: aCriticBlock) &
	(self postCheck: aNode forCriticsDo: aCriticBlock)

critic: aCriticClass for: ast at: node regarding: sequence

	^ aCriticClass
		withAnchor: (
			ReIntervalSourceAnchor
				entity: ast
				interval: node sourceInterval)
		by: self
		method: node selector
		condition: sequence

initialize
	super initialize.

	preConditions := Set new.
	postConditions := Set new

name

	^ 'Invocation order incorect'

postCheck: anAST forCriticsDo: aCriticBlock
	| remainingPostConditions encounteredCritics |
	postConditions ifEmpty: [ ^ false ].
	encounteredCritics := false.

	anAST reverseNodesDo: [ :node |

		remainingPostConditions ifEmpty: [ ^ encounteredCritics ].
		node isMessage ifTrue: [

			"remove the matched conditions while criticizing before-matches"

	remainingPostConditions := postConditions copy.
			remainingPostConditions removeAllSuchThat: [ :cond |
				| beforeMatched |
				(beforeMatched := cond beforeMatches: node selector) ifTrue: [

					encounteredCritics := true.
					aCriticBlock cull: (
						self postCriticFor: anAST at: node regarding: cond) ].
				 "if the following expression is true, the condition will be removed"
				beforeMatched or: [ cond afterMatches: node selector] ] ] ].


	^ encounteredCritics



postCriticFor: ast at: node regarding: sequence

	^ self critic: ReLaterInvocationCritique for: ast at: node regarding: sequence

preCheck: anAST forCriticsDo: aCriticBlock
	| remainingPreConditions encounteredCritics |

	preConditions ifEmpty: [ ^ false ].
	encounteredCritics := false.

	remainingPreConditions := preConditions copy.

	anAST nodesDo: [ :node |
		remainingPreConditions ifEmpty: [ ^ encounteredCritics ].
		node isMessage ifTrue: [

			"remove the matched conditions while criticizing after-matches"
			remainingPreConditions removeAllSuchThat:  [ :cond |
				| afterMatched |
				(afterMatched := cond afterMatches: node selector) ifTrue: [
					encounteredCritics := true.
					aCriticBlock cull: (
						self preCriticFor: anAST at: node regarding: cond) ].

				"if the following expression is true, the condition will be removed"
				afterMatched or: [ cond beforeMatches: node selector] ] ] ].


	^ encounteredCritics


preCriticFor: ast at: node regarding: sequence

	^ self critic: RePriorInvocationCritique for: ast at: node regarding: sequence





` ReNoNilAssignationInInitializeRule `

check: aMethod forCritiquesDo: aCriticBlock
	| problemAssigments|
	(aMethod selector = #initialize and: [ aMethod methodClass isInstanceSide ]) ifFalse: [ ^ self ].

	problemAssigments := aMethod ast assignmentNodes select: [ :assignment | assignment value value isNil ].

	problemAssigments do: [ :assignment |
			aCriticBlock cull: (self createTrivialCritiqueOn: aMethod intervalOf: assignment hint: 'nil').
		 ]



group

	^ 'Style'

name

	^ 'Initialize method does not need nil assignation'

severity
	^ #information



` ReNoPrintStringInPrintOnRule `

check: aMethod forCritiquesDo: aCriticBlock
	| problemPrintStrings |
	aMethod selector = #printOn: ifFalse: [ ^self ].
	problemPrintStrings :=
		aMethod sendNodes select: [ :msgSend | msgSend selector = #printString].

	problemPrintStrings do: [ :msgSend |
			aCriticBlock cull: (self
				createTrivialCritiqueOn: aMethod
				intervalOf: msgSend
				hint: msgSend selector asString )]
group
	^ 'Optimization'
name
	^ 'No printString inside printOn'
rationale
	^ 'Use the stream argument to #printOn: e.g.( stream print: anObject ) instead of creating an extra stream by using printString. e.g.( stream nextPutAll: anObject printString ).'

` ReNodeBasedRule `

	RePatternCodeRule
		ReNodeMatchRule
			ReEndTrueFalseRule

			afterCheck: aNode mappings: mappingDict
	"after a node was match you can do a post-validation here on the node and the dictionalry of wildcars with matched nodes as values"

	^ mappingDict
		ifNil: [ | statement |
			statement := aNode arguments first body statements last.
			( statement isVariable and: [ statement = aNode arguments last body statements last ] ) not]
		ifNotNil: [ true ]

				group
	^ 'Optimization'

				initialize

	super initialize.
	self matchesAny:
			#('`@object
				ifTrue: [| `@temps1 | `@.Statements1. `.Statement]
				ifFalse: [| `@temps2 | `@.Statements2. `.Statement]' '`@object
				ifTrue: [| `@temps1 | `.Statement. `@.Statements1]
				ifFalse: [| `@temps2 | `.Statement. `@.Statements2]' '`@object
				ifFalse: [| `@temps1 | `@.Statements1. `.Statement]
				ifTrue: [| `@temps2 | `@.Statements2. `.Statement]' '`@object
				ifFalse: [| `@temps1 | `.Statement. `@.Statements1]
				ifTrue: [| `@temps2 | `.Statement. `@.Statement2]')

				name
				^ 'Check for same statements at end of ifTrue:ifFalse: blocks'


				rationale
				^ 'Checks for ifTrue:ifFalse: blocks that have the same code at the beginning or 					end.'

				severity
				^ #information


			ReGuardingClauseRule
			group
			^ 'Coding Idiom Violation'

			initialize

	super initialize.
	self addMatchingMethod:
				'`@MethodName: `@args
				| `@temps |
				`@.Statements.
				`@condition ifTrue: [| `@BlockTemps | `.Statement1. `.Statement2. `@.BStatements]'.
	self addMatchingMethod:
				'`@MethodName
				`@.Statements.
				`@condition ifFalse: [| `@BlockTemps | `.Statement1. `.Statement2. `@.BStatements]'

			name
	^ 'Guarding clauses'

rationale
	^ 'Checks for ifTrue: or ifFalse: conditions at end of methods that have two or more statements inside their blocks. Such code might better represent the true meaning of the code if they returned self instead.'
			
	

`	ReNodeRewriteRule `


addMatchingExpression: aString rewriteTo: aRewriteString
	"add a 'from->to' pair of strings that represent a rewrite expression string to match and a rewrite expression to replace the matched node."

	matches
		at: (RBParser parseRewriteExpression: aString)
		put: (ReRewriteTreeWrapper on: (
			RBParser parseRewriteExpression: aRewriteString))


addMatchingExpression: aString rewriteWith: aBlock
	"add 	a 'from->to' pair, first element of which is a rewrite expression in a form of a string that is used to match nodes. The second parameter is a block that has to return a node which should replace the matched one. The block may accept 2 atguments: the matched node, and a dictionary of wildcard variables mapping"

	matches
		at: (RBParser parseRewriteExpression: aString)
		put: (ReRewriteBlockWrapper on: aBlock)


addMatchingMethod: aString rewriteTo: aRewriteString
	"add a 'from->to' pair of strings that represent a match method string to match and a rewrite method to replace the matched mathod"

	matches
		at: (RBParser parseRewriteMethod: aString)
		put: (ReRewriteTreeWrapper on: (
			RBParser parseRewriteMethod: aRewriteString))

addMatchingMethod: aString rewriteWith: aBlock
	"add 	a 'from->to' pair, first element of which is a rewrite method code in a form of a string that is used to match methods. The second parameter is a block that has to return a method ast which should replace the matched one. The block may accept 2 atguments: the matched ast, and a dictionary of wildcard variables mapping"

	matches
		at: (RBParser parseRewriteMethod: aString)
		put: (ReRewriteBlockWrapper on: aBlock)

check: aNode forCritiquesDo: aCriticBlock

	matches keysAndValuesDo: [ :matcher :rewriter |
		matcher
			match: aNode
			onSuccess: [ :map |
				(self
					afterCheck: aNode
					mappings: (self flattenMatchMap: map))
						ifTrue: [
							aCriticBlock cull: (
								self
									critiqueFor: aNode
									by: rewriter
									withMappings: map). ] ]
			onFailure: [  ] ]


critiqueFor: aNode by: rewriter withMappings: map


	^ ReReplaceNodeCritique
		rule: self
		achor: (self anchorFor: aNode)
		oldNode: aNode
		newNode: (rewriter rewriten: aNode with: map)

initialize
	super initialize.
	matches := Dictionary new


isRewriteRule
	^true

replace: aString byEvaluating: aBlock
	"add 	a 'from->to' pair, first element of which is a rewrite expression in a form of a string that is used to match nodes. The second parameter is a block that has to return a node which should replace the matched one. The block may accept 2 atguments: the matched node, and a dictionary of wildcard variables mapping"

	self addMatchingExpression: aString rewriteWith: aBlock

replace: aString with: aRewriteString
	"add a 'from->to' pair of strings that represent a match expression string to match and a rewrite expression to replace the matched node"

	self addMatchingExpression: aString rewriteTo: aRewriteString

replaceMethod: aString with: aRewriteString
	"add a 'from->to' pair of strings that represent a match method string to match and a rewrite method to replace the matched code"

	self addMatchingMethod: aString rewriteTo: aRewriteString

RePackageManifestShouldBePackagedInManifestTagRule


basicCheck: aClass

	^ (aClass inheritsFrom: PackageManifest) and: [
		((aClass package classesForClassTag: 'Manifest') includes: aClass) not ]

group

	^ 'Coding Idiom Violation'

name
	^ 'A package manifest should be tagged using ''Manifest'' class tag'

rationale
	^ 'Check if the package manifest is tagged in class category ''Manifest'' to align with other packages and fulfil common expectations.'


` ReProperMethodProtocolNameRule `


basicCheck: aMethod
	self methodProtocolName: aMethod category.
	^ self class badMethodProtocolNames includes: aMethod category

critiqueFor: aMethod
	| proposedCategory |
	proposedCategory := self class goodMethodProtocolName asSymbol.

	^ (ReRefactoringCritique
		   withAnchor: (self anchorFor: aMethod)
		   by: self) refactoring: (RBMethodProtocolTransformation
			   protocol: { proposedCategory }
			   inMethod: aMethod selector
			   inClass: aMethod methodClass name) asRefactoring

group

	^ 'Coding Idiom Violation'

methodProtocolName

	^ methodProtocolName

methodProtocolName: anObject

	methodProtocolName := anObject

name

	^ 'Method categorization: use '''
		  	, self class goodMethodProtocolName
  	  		, ''' as protocol name instead of '''
	  		, self methodProtocolName asString , ''''

rationale

	^ 'Check if the method protocol name is appropriate and fulfils common expectations.'

use: valid insteadOf: arrayOfInvalid
	"Instead of directly creating an array we use this utility method for better readability
	 in senders of this method"

	^self class use: valid insteadOf: arrayOfInvalid


		` ReProperMethodProtocolNameForAccessingRule `


		` ReProperMethodProtocolNameForAddingRemovingRule `

		` ReProperMethodProtocolNameForConvertingRule`

		` ReProperMethodProtocolNameForFileInOutRule `

		` ReProperMethodProtocolNameForInstanceCreationRule `

		` ReProperMethodProtocolNameForRemovalRule `
		
        ` ReProperMethodProtocolNameForTestsRule `
			basicCheck: aMethod
				"We only check for SUnit test methods"

				^ aMethod isTestMethod and: [ self class badMethodProtocolNames includes: 					aMethod category ]


		` ReProperMethodProtocolNameForUtilitiesRule` 



` ReReturnMethodRule` 


basicCheck: aMethod
	(aMethod overriddenMethods anySatisfy: [ :method | method hasPragmaNamed: #shouldReturn ]) ifFalse: [ ^ false ].

	^ aMethod ast lastIsReturn not

group

	^ 'Bugs'

name

	^ 'Method should explicitly return a value'

severity

	^ #error


` ReThemeAccessRule ` 
check: aMethod forCritiquesDo: aCriticBlock
	| problemThemes |
	((aMethod methodClass inheritsFrom: Morph)
		or: [aMethod methodClass inheritsFrom: Model]) ifFalse: [ ^ self ].

	problemThemes :=
		(aMethod sendNodes select: [ :msgSend | msgSend selector = #theme
				and: [msgSend receiver sourceCode withSeparatorsCompacted = 'Smalltalk ui'] ]).

	problemThemes do: [ :msgSend |
			aCriticBlock cull: (self
				createTrivialCritiqueOn: aMethod
				intervalOf: msgSend
				hint: msgSend selector asString) ]


group
	^ 'API Hints'

name
	^ 'Access to UI theme.'


### 2.ReCritiqueEvent


ReCritiqueEvent subclass: #ReMultipleCritiquesEvent
	instanceVariableNames: 'critiques'
	classVariableNames: ''
	package: 'Renraku-Announcements'


		ReMultipleCritiquesEvent subclass: #ReEntityCriticized
		instanceVariableNames: ''
		classVariableNames: ''
		package: 'Renraku-Announcements'


ReCritiqueEvent subclass: #ReSingleCritiqueEvent
	instanceVariableNames: 'critique'
	classVariableNames: ''
	package: 'Renraku-Announcements'

critique
	^ critique

critique: aCritique
	critique := aCritique


		ReSingleCritiqueEvent subclass: #ReCritiqueAutofixed
		instanceVariableNames: ''
		classVariableNames: ''
		package: 'Renraku-Announcements'


		ReSingleCritiqueEvent subclass: #ReCritiqueBanned
		instanceVariableNames: 'entityBanLevel'
		classVariableNames: ''
		package: 'Renraku-Announcements'



		ReSingleCritiqueEvent subclass: #ReCritiqueDescriptionViewed
		instanceVariableNames: ''
		classVariableNames: ''
		package: 'Renraku-Announcements'


### 3.ReExceptionStrategy


ReExceptionStrategy subclass: #ReIgnoreExceptionStrategy
	instanceVariableNames: ''
	classVariableNames: ''
	package: 'Renraku-Utility'


ReExceptionStrategy subclass: #ReLogExceptionStrategy
	instanceVariableNames: ''
	classVariableNames: ''
	package: 'Renraku-Utility'


ReExceptionStrategy subclass: #RePassExceptionStrategy
	instanceVariableNames: ''
	classVariableNames: ''
	package: 'Renraku-Utility'


### 4.ReFeedbacker


	
### 5.ReImageRuleBaner


	ReImageRuleBaner subclass: #RePersistentImageRuleBaner
	instanceVariableNames: ''
	classVariableNames: ''
	package: 'Renraku-Utility'


### 6.ReInvocationSequenceRuleSequence


	Object subclass: #ReInvocationSequenceRuleSequence
	instanceVariableNames: 'before after'
	classVariableNames: ''
	package: 'Renraku-Rules'



		after
			^ after


		afterDescriptionString

			^ self descriptionStringFor: after


		afterMatches: aSymbol
			^ after includes: aSymbol


		before
			^ before




		beforeDescriptionString

			^ self descriptionStringFor: before

		beforeMatches: aSymbol
			^ before includes: aSymbol


		descriptionStringFor: aSet

			^ aSet size = 1
			ifTrue: [ '#', aSet anyOne, ' message' ]
			ifFalse: [ 'any one of #', (aSet joinUsing: ', #'), ' messages' ]


		initializeBefore: beforeSymols after: afterSymbols

		before :=
		 (beforeSymols isCollection and:
		[ beforeSymols isString not ])
			ifFalse: [ Array with: beforeSymols asSymbol ]
			ifTrue: [ beforeSymols ].
	after :=
		(afterSymbols isCollection and:
		[ afterSymbols isString not ])
			ifFalse: [ Array with: afterSymbols asSymbol ]
			ifTrue: [ afterSymbols ]

### 7.ReMethodSourceCleaner



clean

	self
		removeTrailingSeparatorLines;
		removeTrailingDot;
		removeTrailingSeparatorsInAllLines


contents

	^ String streamContents: [:s |
			sourceLines asArray do: [:each | s nextPutAll: each contents ] separatedBy: [ s nextPut: Character cr ]
	]


initFromSource: aString

	| lines |
	lines := aString lines withIndexCollect: [ :each :index |
		         ReSourceCodeLine lineNumber: index contents: each ].

	sourceLines := DoubleLinkedList new
		               addAll: lines;
		               Yourself


initialize

	super initialize.
	self keepFinalDot: self class keepFinalDotInMethodBody


keepFinalDot

	^ keepFinalDot

keepFinalDot: aBoolean

	keepFinalDot := aBoolean


removeTrailingDot

	self keepFinalDot ifTrue: [ ^ self ].
	sourceLines last removeTrailingDot

removeTrailingSeparatorLines

	[
	sourceLines isNotEmpty and: [
		sourceLines last onlyContainsSeparators ] ] whileTrue: [
		sourceLines removeLast ]



removeTrailingSeparatorsInAllLines

	 sourceLines do: [:each | each removeTrailingSeparators ]

### 8.ReProperty


ReProperty subclass: #ReAbstractCritique
	instanceVariableNames: 'rule tinyHint'
	classVariableNames: ''
	package: 'Renraku-Critiques'
		



		ReAbstractCritique subclass: #ReCompactSourceCodeCritic
		instanceVariableNames: ''
	classVariableNames: ''
		package: 'Renraku-Critiques'


			change

			^ self subclassResponsibility



providesChange

			^ true

		ReAbstractCritique subclass: #ReInvocationOrderCritique
		instanceVariableNames: 'targetMethodName condition'
		classVariableNames: ''
		package: 'Renraku-Critiques'


			initializeRule: aRule sourceAnchor: anAnchor method: aString 				condition: aCondition

			self initializeRule: aRule sourceAnchor: anAnchor.
			targetMethodName := aString.
			condition := aCondition


			providesChange
			^ false
				
		ReInvocationOrderCritique subclass: #ReLaterInvocationCritique
		instanceVariableNames: ''
		classVariableNames: ''
		package: 'Renraku-Critiques'

		description

		^ 'The functionality of #', targetMethodName, ' is intended to influence ', 		condition afterDescriptionString, '. In your case it is not followed by the 			previously mentioned messages which suggests that you are doing 			something wrong.'


		title

		^ 'Message #', targetMethodName, ' not followed by ',
		(condition afterDescriptionString truncateWithElipsisTo: 30)

		ReInvocationOrderCritique subclass: #RePriorInvocationCritique
		instanceVariableNames: ''
		classVariableNames: ''
		package: 'Renraku-Critiques'


			description

			^ 'In order for #', targetMethodName, ' to work you have to call ', 			condition beforeDescriptionString, ' before'



			title

			^ 'Message #', targetMethodName, ' not preceded by ',
			(condition beforeDescriptionString truncateWithElipsisTo: 30)

ReProperty subclass: #ReExceptionProperty
	instanceVariableNames: 'stack message'
	classVariableNames: ''
	package: 'Renraku-Critiques'


Object subclass: #RePropertyAction
	instanceVariableNames: 'action icon description'
	classVariableNames: ''
	package: 'Renraku-Critiques'


### 9.RePropertyAction


actOnCritic: aCritic ofEntity: anEntity

	action cull: aCritic cull: anEntity


action
	^ action

action: anObject
	action := anObject


description
	^ description


description: anObject
	description := anObject


icon
	^ icon


icon: anObject
	icon := anObject


initialize

	icon := (Form extent: 16@16 depth: 32) fillColor: Color green.
	description := ''.
	action := [ :critic :entity | ]



### 10.ReRewriteWrapper


rewriten: aNode with: aMap

	^ self subclassResponsibility



### 11.ReRuleManager


allRules

		^ rules


	classRules

	^ self allRules select: [ :rule | rule class checksClass ]


	initializeRules: aCollection

	rules := aCollection copy



	methodRules

	^ self allRules select: [ :rule | rule class checksMethod ]



	nodeRules

	^ self allRules select: [ :rule | rule class checksNode ]



	packageRules

	^ self allRules select: [ :rule | rule class checksPackage ]

### 12.ReSettings


	Object subclass: #ReSettings
	instanceVariableNames: ''
	classVariableNames: ''
	package: 'Renraku-Utility'

### 13.ReSmalllintChecker

	addCritique: aCritique
		^ critiques add: aCritique


	builderManifestClass
	^ TheManifestBuilder


	checkClass: aClass

	(environment includesClass: aClass) ifFalse: [ ^ self ].

	self getCritiquesAbout: aClass by: classRules


	checkMethodsForClass: aClass

	environment
		selectorsForClass: aClass
		do: [ :selector | | method ast |
			method := aClass>>selector.
			self getCritiquesAbout: method by: methodRules.
			nodeRules do: [ :r |
				ast := method ast.
				"for rewrite rules, we run every rule on a copy of the ast"
				r isRewriteRule ifTrue: [ ast := ast copy  ].
				ast nodesDo: [ :node |
						r
							check: node
							forCritiquesDo: [ :crit |
								crit sourceAnchor initializeEnitity: method.
								self addCritique: crit ] ] ]]



	checkPackage: aPackage

	(environment includesPackage: aPackage) ifTrue: [
		self getCritiquesAbout: aPackage by: packageRules ]


cleanAllManifest
	| manifests |
	manifests := environment packages collect: [ :cl | self builderManifestClass of: cl ] as: Set.
	manifests do: [ :each | each cleanUp ]



criticsOf: aRule

	^ critiques
		select: [ :c | c rule class = aRule class ]


falsePositiveOf: aRule

	| ruleCritiques rId rV mb |

	ruleCritiques := self criticsOf: aRule.
	rId := aRule class uniqueIdentifierName.
	rV := aRule class identifierMinorVersionNumber.
	^ ruleCritiques
		select: [ :critique |
			| entity |

			entity := critique sourceAnchor entity.
			mb := self manifestBuilderOf: entity.
			mb ifNil: [ false ] ifNotNil: [ mb isFalsePositive: entity onRule: rId version: rV ]
			]



getCritiquesAbout: anEntity by: rules

	rules do: [ :r |
		[ r
			check: anEntity
			forCritiquesDo: [ :crit |
				self addCritique: crit ] ]

			on: Error
			do: [ :er | ReSettings showErrors ifTrue: [ er pass ]  ] ]


initialize
	super initialize.
	manifestClassCache := Dictionary new.
	self resetResult


isFalsePositive: aCritic forRuleId: ruleId versionId: versionId

	| mb |

	mb := self manifestBuilderOf: aCritic.
	^ mb ifNil: [ false ] ifNotNil: [ mb isFalsePositive: aCritic onRule: ruleId version: versionId ]
isToDo: aCritic forRuleId: ruleId versionId: versionId

	| mb |

	mb := self manifestBuilderOf: aCritic.
	^ mb ifNil: [ false ] ifNotNil: [ mb containsToDo: aCritic onRule: ruleId version: versionId ]


isTruePositive: aCritic forRuleId: ruleId versionId: versionId

	^ (self manifestBuilderOf: aCritic)
			containsTruePositive: aCritic onRule:  ruleId version: versionId


manifestBuilderOf: aPackage

	^ aPackage manifestBuilderForRuleChecker: self



manifestBuilderOfClass: aClass
	| key |
	key := aClass instanceSide category.
	^ manifestClassCache
		at: key
		ifAbsentPut: [
			(self builderManifestClass hasManifestFor: aClass)
				ifTrue: [ self builderManifestClass of: aClass ]
				ifFalse: [ nil ] ]



manifestBuilderOfMethod: aMethod
	| key cl |
	cl := aMethod methodClass.
	key := cl name , aMethod category asString.
	^ manifestClassCache
		at: key
		ifAbsentPut: [
			(self builderManifestClass hasManifestFor: aMethod)
				ifTrue: [ self builderManifestClass of: aMethod ]
				ifFalse: [ nil ] ]



manifestBuilderOfPackage: aPackage
	| key |
	key := aPackage packageName.
	^ manifestClassCache
		at: key
		ifAbsentPut: [
			(self builderManifestClass hasPackageNamed: key)
				ifTrue: [ self builderManifestClass ofPackageNamed: key ]
				ifFalse: [ nil ] ]



reParseRule

	methodRules := Set new.
	nodeRules := Set new.
	classRules := Set new.
	packageRules := Set new.

	self rule do: [ :r |
		r class checksMethod ifTrue: [ methodRules add: r ].
		r class checksNode ifTrue: [ nodeRules add: r ].
		r class checksClass ifTrue: [ classRules add: r ] ]


rejectClassesOf: aPackage

	| mb |

	mb := self manifestBuilderOf: aPackage.
	^ mb ifNil: [ {} ] ifNotNil: [ mb rejectClasses ]


rejectRulesOf: aPackage

	| mb |

	mb := self manifestBuilderOf: aPackage.
	^ mb ifNil: [ {} ] ifNotNil: [ mb rejectRules ]


resetResult

	critiques := Set new


rule

	^ (rule isKindOf: RBCompositeLintRule)
		ifTrue: [ rule leaves ]
		ifFalse: [ rule ]


run
	self reParseRule.
	super run



runRules: aCompositeRule onPackage: aPackage

	rule := aCompositeRule.
	environment := RBPackageEnvironment package: aPackage.
	self run


runRules: aCompositeRule onPackage: aPackage withoutTestCase: avoidTestCases
	rule := aCompositeRule.
	environment := RBPackageEnvironment package: aPackage.

	environment := environment withoutClasses: (environment classes select: #isManifest).
	avoidTestCases
		ifTrue: [ environment := environment withoutClasses: (environment classes select: #isTestCase) ].
	self run



toDoOf: aRule

	| ruleCritiques rId rV mb |

	ruleCritiques := self criticsOf: aRule.
	rId := aRule class uniqueIdentifierName.
	rV := aRule class identifierMinorVersionNumber.
	^ ruleCritiques
		select: [ :critique |
			| entity |

			entity := critique sourceAnchor entity.
			mb := self manifestBuilderOf: entity.
			mb ifNil: [ false ] ifNotNil: [ mb containsToDo: entity onRule: rId version: rV ]
			]




### 14.ReSourceAnchor

Object subclass: #ReSourceAnchor
	instanceVariableNames: 'sourceEntity'
	classVariableNames: ''
	package: 'Renraku-SourceAnchors'


	initializeEnitity: anEntiy interval: anInterval

	self initializeEnitity: anEntiy.
	interval := anInterval



	providesInterval

	^ true

a°

ReSourceAnchor subclass: #ReIntervalSourceAnchor
	instanceVariableNames: 'interval'
	classVariableNames: ''
	package: 'Renraku-SourceAnchors'

	interval

	^ interval

b° 
	ReSourceAnchor subclass: #ReSearchStringSourceAnchor
	instanceVariableNames: 'searchString'
	classVariableNames: ''
	package: 'Renraku-SourceAnchors'
		
		bb° ReSearchStringSourceAnchor subclass: #ReVarSearchSourceAnchor
		instanceVariableNames: ''
		classVariableNames: ''


		package: 'Renraku-SourceAnchors'
		interval
		^ super interval allButLast


		modify: aString for: aStringCollection

		aStringCollection size = 1 ifTrue: [
			^ aString surroundedBySingleQuotes ].
		aStringCollection first = aString ifTrue: [
			^ '''', aString, String space ].
		aStringCollection last = aString ifTrue: [
			^ String space, aString, '''' ].
			^ aString surroundedBy: String space




searchString
		| origString |

		origString := super searchString.
		(sourceEntity instVarNames includes: origString) ifTrue: [
		^ self modify: origString for: sourceEntity instVarNames ].
		(sourceEntity classVarNames includes: origString) ifTrue: [
		^ self modify: origString for: sourceEntity classVarNames ].

		^ origString

### 15.ReSourceCodeLine


contents

	^ contents



	contents: anObject

	contents := anObject


	lineNumber

	^ lineNumber


	lineNumber: anObject

	lineNumber := anObject



	onlyContainsSeparators
	^ self contents isAllSeparators



	removeTrailingDot
	contents := contents trimRight: [ :char | char = $. ]



	removeTrailingSeparators
	contents := contents trimRight: [ :char | char = Character space or: [ char = Character tab ] ]


### 16.ReSystemAnnouncer
	
	notifyCritique: aCritique descriptionViewedFor: anEntity

	self announce: (
		ReCritiqueDescriptionViewed new
			critique: aCritique;
			entity: anEntity;
			yourself)



	notifyCritique: aCritique of: anEntity bannedOn: anEntityBanLevel

	self announce: (
		ReCritiqueBanned new
			critique: aCritique;
			entity: anEntity;
			entityBanLevel: anEntityBanLevel;
			yourself)




	notifyEntity: anEntity criticizedWith: critiques

	self announce: (
		ReEntityCriticized new
			critiques: critiques;
			entity: anEntity;
			yourself)

	
		
### 17.ClassDescription
	ban: aCritique

	ReSystemAnnouncer uniqueInstance
		notifyCritique: aCritique
		of: aCritique sourceAnchor entity
		bannedOn: self.

	(TheManifestBuilder of: self)
		banRule: aCritique rule for: self

	
	banChecksForValidation
"returns ban levels that have to be checked before validation"

	^ { self . self methodClass }

banLevelCandidates
"returns code entities which can ban a rule for this entity"

	^ { self } , self methodClass banLevelCandidates

	critiques
	| critiques |

	critiques := ReCriticEngine critiquesOf: self.
	critiques :=
		critiques,
		(ReCriticEngine nodeCritiquesOf: self).

	^ critiques


	overriddenMethods
	"all the methods of my superclasses that are overriden by me"
	| selector |
	selector := self selector.
	^ self methodClass allSuperclasses
		select: [ :class | class includesSelector: selector ]
		thenCollect: [ :class | class compiledMethodAt: selector ]

	reBanName

	^ self name, ' (Method)'


suitableRulesFrom: aReRuleManager
	^ aReRuleManager methodRules


### 18.RBLintRule

	RBBasicLintRule subclass: #RBParseTreeLintRule
	instanceVariableNames: 'matcher'
	classVariableNames: ''
	package: 'Refactoring-Critics'

	check: aMethod forCritiquesDo: aCriticBlock

	(self matcher canMatchMethod: aMethod) ifTrue: [
		| node |
		node := self matcher
			executeTree: aMethod ast
			initialAnswer: nil.
		node ifNotNil: [
			^ aCriticBlock cull: (
				self critiqueFor: node about: aMethod) ] ]


	critiqueFor: aNode about: aMethod
	| anchor |

	anchor := (aNode isKindOf: RBProgramNode)
		ifFalse: [ ReSourceAnchor entity: aMethod ]
		ifTrue: [
			ReIntervalSourceAnchor
				entity: aMethod
				interval: aNode sourceInterval ].

	^ ReTrivialCritique
		withAnchor: anchor
		by: self

RBLintRule subclass: #RBTransformationRule
	instanceVariableNames: 'rewriteRule builder class'
	classVariableNames: 'RecursiveSelfRule'
	package: 'Refactoring-Critics'


check: aMethod forCritiquesDo: aCriticBlock

	(self basicCheck: aMethod)
		ifTrue: [
			aCriticBlock cull: (
				ReTransformationCritique
					for: aMethod
					by: self
					tree: rewriteRule tree)  ]


suggestedCode: aCritic

	^ (self changes detect: [ :ch |
			(ch  isMeta = aCritic  methodClass isMeta) &
			(ch changeClassName = aCritic criticTheNonMetaclassClass name) &
			(ch selector = aCritic selector)]) source

	
	TraitedMetaclass subclass: #MetaclassForTraits
	instanceVariableNames: 'users'
	classVariableNames: ''
	package: 'TraitsV2-Base'


reBanName

	^ self name, ' (Trait)'


	Class subclass: #Trait
	instanceVariableNames: 'users'
	classVariableNames: ''
	package: 'TraitsV2-Base'


	reBanName

	^ self name, ' (Trait)'



	CompiledCode variableByteSubclass: #CompiledMethod
	instanceVariableNames: ''
	classVariableNames: ''
	package: 'Kernel-Methods'


	ban: aCritique

	ReSystemAnnouncer uniqueInstance
		notifyCritique: aCritique
		of: aCritique sourceAnchor entity
		bannedOn: self.

	(TheManifestBuilder of: self)
		banRule: aCritique rule for: self

	banChecksForValidation
"returns ban levels that have to be checked before validation"

	^ { self . self methodClass }



	banLevelCandidates
"returns code entities which can ban a rule for this entity"

	^ { self } , self methodClass banLevelCandidates


	critiques
	| critiques |

	critiques := ReCriticEngine critiquesOf: self.
	critiques :=
		critiques,
		(ReCriticEngine nodeCritiquesOf: self).

	^ critiques



	overriddenMethods
	"all the methods of my superclasses that are overriden by me"
	| selector |
	selector := self selector.
	^ self methodClass allSuperclasses
		select: [ :class | class includesSelector: selector ]
		thenCollect: [ :class | class compiledMethodAt: selector ]




	reBanName

	^ self name, ' (Method)'


	suitableRulesFrom: aReRuleManager
	^ aReRuleManager methodRules


	
### 19.RBProgramNode


	RBBlockNode subclass: #RBPatternBlockNode
	instanceVariableNames: 'valueBlock'
	classVariableNames: ''
	package: 'AST-Core-Pattern'
	
	pattern

	^ self source
		copyFrom: self sourceInterval first
		to: self sourceInterval last
	
	RBMessageNode subclass: #RBPatternMessageNode
	instanceVariableNames: 'isList isCascadeList'
	classVariableNames: ''
	package: 'AST-Core-Pattern'

	pattern

	^ self selector

	RBMethodNode subclass: #RBPatternMethodNode
	instanceVariableNames: 'isList'
	classVariableNames: ''
	package: 'AST-Core-Pattern'
	pattern

	^ self selector

	RBPragmaNode subclass: #RBPatternPragmaNode
	instanceVariableNames: 'isList'
	classVariableNames: ''
	package: 'AST-Core-Pattern'

	pattern

	^ self selector

	RBVariableNode subclass: #RBPatternVariableNode
	instanceVariableNames: 'recurseInto isList isLiteral isStatement isAnything'
	classVariableNames: ''
	package: 'AST-Core-Pattern'


	pattern

	^ self name

### 20.RGMethodDefinition

critiques
	^ self compiledMethod critiques

### 21.RPackage


ban: aCritique

	ReSystemAnnouncer uniqueInstance
		notifyCritique: aCritique
		of: aCritique sourceAnchor entity
		bannedOn: self.

	(TheManifestBuilder of: self)
		banRule: aCritique rule for: self


banChecksForValidation

	^ {}



banLevelCandidates
"returns code entities which can ban a rule for this entity"

	^ { self . ReImageRuleBaner . RePersistentImageRuleBaner }



critiques
	^ ReCriticEngine critiquesOf: self


package
	^ self


reBanName

	^ self name, ' (Package)'


suitableRulesFrom: aReRuleManager
	^ aReRuleManager packageRules


 
### 22.SmalltalkImage

	reBanName
	^ 'The whole image'
	
22.TheManifestBuilder


banRule: aRule for: anEntity
	| ruleId number |
	ruleId := aRule class uniqueIdentifierName.
	number := aRule class identifierMinorVersionNumber.

	(self hasFalsePositiveOf: ruleId version: number) ifFalse: [
		self installFalsePositiveOf: ruleId version: number ].

	self
		addFalsePositive: anEntity
		of: ruleId
		version: number



bansRule: aRule for: anEntity
	| ruleId number |
	ruleId := aRule class uniqueIdentifierName.
	number := aRule class identifierMinorVersionNumber.

	^ self
		containsFalsePositive: anEntity
		onRule: ruleId
		version: number