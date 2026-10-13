# Règles Renraku — Erreurs de développeur débutant en Pharo





## 1. Sur l' image de pharo 14

| Constat | Conséquence |
|---|---|
| `ReTrivialCritique` n'a **pas** de `rule:` ni `entity:` | Ne jamais faire `ReTrivialCritique new rule: self; entity: ...` |
| Méthode de classe `ReTrivialCritique for: anEntity by: aRule` | **C'est la bonne construction** |
| `tinyHint:` / `tinyHint` existent sur l'instance | À enchaîner juste après `for:by:` |
| `Object subclass: ... instanceVariableNames:` et `subclass:slots:classVariables:package:` ne sont pas compris | Pas de création de classe dans les tests |
| `Nil` n'existe pas comme variable | Dans le code de test, déclarer `\| Nil \|` pour que `compile:` accepte le texte |
| Chercher `True` comme simple texte trouve aussi `ifTrue:` | Il faut détecter des **mots entiers** (`includesWord:in:`) |

---

## 2. Prérequis : la classe sandbox (à créer une seule fois)

Les tests compilent du code fautif dans une classe jetable. Crée **une fois** dans le System Browser.
Vérification : `Smalltalk includesKey: #TestBooleanClass` doit afficher `true`.

```
Object << #TestBooleanClass
    slots: {};
    package: 'Renraku-Tests-Temp'
```

Tous les tests ci-dessous utilisent cette classe. Aucun `setUp`, aucun `tearDown`, aucune variable d'instance.

---

## 3. Méthode commune `includesWord:in:`

À définir **une seule fois**, dans le protocole **`private`** de la superclasse **`ReExclusivelyBeginnersRule`**. Toutes les règles qui l'utilisent (`ReBooleanCapitalizationRule`, `ReSelfCapitalizationRule`, `ReReturnKeywordRule`, `Noob`) en héritent : il ne faut pas la recopier dans chaque règle.
Elle renvoie `true` seulement si le mot apparaît entouré de caractères non alphanumériques (donc `True` n'est pas trouvé dans `ifTrue:`).

```
includesWord: aWord in: sourceCode
    | start index before after |
    start := 1.
    [ index := sourceCode indexOfSubCollection: aWord startingAt: start.
      index > 0 ] whileTrue: [
        before := index > 1
            ifTrue: [ sourceCode at: index - 1 ]
            ifFalse: [ Character space ].
        after := (index + aWord size) <= sourceCode size
            ifTrue: [ sourceCode at: index + aWord size ]
            ifFalse: [ Character space ].
        (before isAlphaNumeric or: [ after isAlphaNumeric ])
            ifFalse: [ ^ true ].
        start := index + 1 ].
    ^ false
```

---

## 4. ReBooleanCapitalizationRule

Détecte `True`, `False`, `Nil`.

### Règle

```
"Protocole: checking"
checkMethod: aMethod
    | critiques sourceCode |
    critiques := OrderedCollection new.
    sourceCode := aMethod sourceCode.

    self detectBoolean: 'True' -> 'true' in: sourceCode for: aMethod critiques: critiques.
    self detectBoolean: 'False' -> 'false' in: sourceCode for: aMethod critiques: critiques.
    self detectBoolean: 'Nil' -> 'nil' in: sourceCode for: aMethod critiques: critiques.

    ^ critiques

"Protocole: private"
detectBoolean: aForm in: sourceCode for: aMethod critiques: critiques
    "Detect incorrect form and create a critic with personnalized message"

    (self includesWord: aForm key in: sourceCode) ifTrue: [
        critiques add: ((ReTrivialCritique for: aMethod by: self)
            tinyHint: aForm key, ' doit être écrit ', aForm value, ' (minuscules)';
            yourself) ]

"Protocole: accessing"
group
    ^ 'Erreurs débutant'

name
    ^ 'Booléens mal capitalisés'

rationale
    ^ 'En Pharo, les booléens et nil doivent être écrits en minuscules: true, false, nil'

severity
    ^ #error

"Côté classe, protocole: manifest"
ReBooleanCapitalizationRule class >> uniqueIdentifierName
    ^ 'BooleanCapitalization'
```

### Tests

```
ReExclusivelyBeginnersRuleTest << #ReBooleanCapitalizationRuleTest
    slots: {};
    tag: 'Rules';
    package: 'Renraku-Tests'

testDetectTrue
    | critiques |
    TestBooleanClass compile: 'testMethod
        ^ True'.
    critiques := ReBooleanCapitalizationRule new
        checkMethod: (TestBooleanClass >> #testMethod).
    self assert: critiques size equals: 1.
    self assert: (critiques first tinyHint includesSubstring: 'true')

testDetectFalse
    | critiques |
    TestBooleanClass compile: 'testMethod
        ^ False'.
    critiques := ReBooleanCapitalizationRule new
        checkMethod: (TestBooleanClass >> #testMethod).
    self assert: critiques size equals: 1.
    self assert: (critiques first tinyHint includesSubstring: 'false')

testDetectNil
    | critiques |
    TestBooleanClass compile: 'testMethod
        | Nil |
        ^ Nil'.
    critiques := ReBooleanCapitalizationRule new
        checkMethod: (TestBooleanClass >> #testMethod).
    self assert: critiques size equals: 1.
    self assert: (critiques first tinyHint includesSubstring: 'nil')

testDetectMultiple
    | critiques |
    TestBooleanClass compile: 'testMethod
        | Nil |
        True ifTrue: [ ^ False ].
        ^ Nil'.
    critiques := ReBooleanCapitalizationRule new
        checkMethod: (TestBooleanClass >> #testMethod).
    self assert: critiques size equals: 3

testNoFalsePositives
    | critiques |
    TestBooleanClass compile: 'testMethod
        true ifTrue: [ ^ false ].
        ^ nil'.
    critiques := ReBooleanCapitalizationRule new
        checkMethod: (TestBooleanClass >> #testMethod).
    self assert: critiques isEmpty

testRuleMetadata
    | rule |
    rule := ReBooleanCapitalizationRule new.
    self assert: rule severity equals: #error.
    self assert: rule group equals: 'Erreurs débutant'.
    self deny: rule name isEmpty.
    self deny: rule rationale isEmpty.
    self assert: rule class uniqueIdentifierName equals: 'BooleanCapitalization'
```

---

## 5. ReSelfCapitalizationRule 

Détecte `Self` au lieu de `self`.

### Règle

```
ReExclusivelyBeginnersRule << #ReSelfCapitalizationRule
    slots: {};
    tag: 'Rules';
    package: 'Renraku'

"Protocole: checking"
checkMethod: aMethod
    (self includesWord: 'Self' in: aMethod sourceCode) ifTrue: [
        ^ { (ReTrivialCritique for: aMethod by: self)
            tinyHint: 'Self doit être écrit self (minuscules)';
            yourself } ].
    ^ #()

"includesWord:in: est héritée de ReExclusivelyBeginnersRule (section 3)"

"Protocole: accessing"
group
    ^ 'Erreurs débutant'

name
    ^ 'Self au lieu de self'

rationale
    ^ 'En Pharo, self s''écrit en minuscules'

severity
    ^ #error

"Côté classe, protocole: manifest"
ReSelfCapitalizationRule class >> uniqueIdentifierName
    ^ 'SelfCapitalization'
```

### Tests

```
ReExclusivelyBeginnersRuleTest << #ReSelfCapitalizationRuleTest
    slots: {};
    tag: 'Rules';
    package: 'Renraku-Tests'

testDetectSelf
    | critiques |
    TestBooleanClass compile: 'testMethod
        | Self |
        ^ Self'.
    critiques := ReSelfCapitalizationRule new
        checkMethod: (TestBooleanClass >> #testMethod).
    self assert: critiques size equals: 1.
    self assert: (critiques first tinyHint includesSubstring: 'self')

testIgnoresLongerIdentifiers
    | critiques |
    TestBooleanClass compile: 'testMethod
        | SelfishValue |
        ^ SelfishValue'.
    critiques := ReSelfCapitalizationRule new
        checkMethod: (TestBooleanClass >> #testMethod).
    self assert: critiques isEmpty

testNoFalsePositives
    | critiques |
    TestBooleanClass compile: 'testMethod
        ^ self printString'.
    critiques := ReSelfCapitalizationRule new
        checkMethod: (TestBooleanClass >> #testMethod).
    self assert: critiques isEmpty

testRuleMetadata
    | rule |
    rule := ReSelfCapitalizationRule new.
    self assert: rule severity equals: #error.
    self assert: rule group equals: 'Erreurs débutant'.
    self deny: rule name isEmpty.
    self deny: rule rationale isEmpty.
    self assert: rule class uniqueIdentifierName equals: 'SelfCapitalization'
```

---

## 6. ReReturnKeywordRule

Détecte le mot `return` au lieu de `^`.

### Règle

```
ReExclusivelyBeginnersRule << #ReReturnKeywordRule
    slots: {};
    tag: 'Rules';
    package: 'Renraku'

"Protocole: checking"
checkMethod: aMethod
    (self includesWord: 'return' in: aMethod sourceCode) ifTrue: [
        ^ { (ReTrivialCritique for: aMethod by: self)
            tinyHint: 'Utilisez ^ au lieu de return';
            yourself } ].
    ^ #()

"includesWord:in: est héritée de ReExclusivelyBeginnersRule (section 3)"

"Protocole: accessing"
group
    ^ 'Erreurs débutant'

name
    ^ 'return au lieu de ^'

rationale
    ^ 'En Pharo, le retour se fait avec ^ et non avec le mot-clé return'

severity
    ^ #error

"Côté classe, protocole: manifest"
ReReturnKeywordRule class >> uniqueIdentifierName
    ^ 'ReturnKeyword'
```

### Tests

```
ReExclusivelyBeginnersRuleTest << #ReReturnKeywordRuleTest
    slots: {};
    tag: 'Rules';
    package: 'Renraku-Tests'

testDetectReturn
    | critiques |
    TestBooleanClass compile: 'testMethod
        | return |
        return := 42.
        ^ return'.
    critiques := ReReturnKeywordRule new
        checkMethod: (TestBooleanClass >> #testMethod).
    self assert: critiques size equals: 1.
    self assert: (critiques first tinyHint includesSubstring: '^')

testIgnoresLongerIdentifiers
    | critiques |
    TestBooleanClass compile: 'testMethod
        | returnCode |
        returnCode := 42.
        ^ returnCode'.
    critiques := ReReturnKeywordRule new
        checkMethod: (TestBooleanClass >> #testMethod).
    self assert: critiques isEmpty

testNoFalsePositives
    | critiques |
    TestBooleanClass compile: 'testMethod
        ^ 42'.
    critiques := ReReturnKeywordRule new
        checkMethod: (TestBooleanClass >> #testMethod).
    self assert: critiques isEmpty

testRuleMetadata
    | rule |
    rule := ReReturnKeywordRule new.
    self assert: rule severity equals: #error.
    self assert: rule group equals: 'Erreurs débutant'.
    self deny: rule name isEmpty.
    self deny: rule rationale isEmpty.
    self assert: rule class uniqueIdentifierName equals: 'ReturnKeyword'
```

---

## 7. ReIfTrueColonRule

Détecte `ifTrue[` et `ifFalse[` (deux-points oublié).

Le texte `x ifTrue[ ^ 1 ]` ne se compile pas en Pharo. Les tests placent donc le motif fautif **dans un commentaire** : la règle analyse le texte source, elle le détecte quand même, et `compile:` accepte la méthode.

### Règle

```
ReExclusivelyBeginnersRule << #ReIfTrueColonRule
    slots: {};
    tag: 'Rules';
    package: 'Renraku'

"Protocole: checking"
checkMethod: aMethod
    | critiques sourceCode |
    critiques := OrderedCollection new.
    sourceCode := aMethod sourceCode.

    (sourceCode includesSubstring: 'ifTrue[') ifTrue: [
        critiques add: ((ReTrivialCritique for: aMethod by: self)
            tinyHint: 'ifTrue[ doit être écrit ifTrue: [';
            yourself) ].

    (sourceCode includesSubstring: 'ifFalse[') ifTrue: [
        critiques add: ((ReTrivialCritique for: aMethod by: self)
            tinyHint: 'ifFalse[ doit être écrit ifFalse: [';
            yourself) ].

    ^ critiques

"Protocole: accessing"
group
    ^ 'Erreurs débutant'

name
    ^ 'ifTrue/ifFalse nécessitent un deux-points'

rationale
    ^ 'Les messages ifTrue: et ifFalse: nécessitent un deux-points avant le bloc'

severity
    ^ #error

"Côté classe, protocole: manifest"
ReIfTrueColonRule class >> uniqueIdentifierName
    ^ 'IfTrueColon'
```

### Tests

```
ReExclusivelyBeginnersRuleTest << #ReIfTrueColonRuleTest
    slots: {};
    tag: 'Rules';
    package: 'Renraku-Tests'

testDetectIfTrueWithoutColon
    | critiques |
    TestBooleanClass compile: 'testMethod
        "x ifTrue[ ^ 1 ]"
        ^ 1'.
    critiques := ReIfTrueColonRule new
        checkMethod: (TestBooleanClass >> #testMethod).
    self assert: critiques size equals: 1.
    self assert: (critiques first tinyHint includesSubstring: 'ifTrue:')

testDetectIfFalseWithoutColon
    | critiques |
    TestBooleanClass compile: 'testMethod
        "x ifFalse[ ^ 0 ]"
        ^ 1'.
    critiques := ReIfTrueColonRule new
        checkMethod: (TestBooleanClass >> #testMethod).
    self assert: critiques size equals: 1.
    self assert: (critiques first tinyHint includesSubstring: 'ifFalse:')

testDetectBoth
    | critiques |
    TestBooleanClass compile: 'testMethod
        "ifTrue[ ifFalse["
        ^ 1'.
    critiques := ReIfTrueColonRule new
        checkMethod: (TestBooleanClass >> #testMethod).
    self assert: critiques size equals: 2

testNoFalsePositives
    | critiques |
    TestBooleanClass compile: 'testMethod
        true ifTrue: [ ^ 1 ] ifFalse: [ ^ 0 ]'.
    critiques := ReIfTrueColonRule new
        checkMethod: (TestBooleanClass >> #testMethod).
    self assert: critiques isEmpty

testRuleMetadata
    | rule |
    rule := ReIfTrueColonRule new.
    self assert: rule severity equals: #error.
    self assert: rule group equals: 'Erreurs débutant'.
    self deny: rule name isEmpty.
    self deny: rule rationale isEmpty.
    self assert: rule class uniqueIdentifierName equals: 'IfTrueColon'
```

---

## 8. ReAndOperatorRule

Détecte `&&` au lieu de `and:`.

### Règle

```
ReExclusivelyBeginnersRule << #ReAndOperatorRule
    slots: {};
    tag: 'Rules';
    package: 'Renraku'

"Protocole: checking"
checkMethod: aMethod
    (aMethod sourceCode includesSubstring: '&&') ifTrue: [
        ^ { (ReTrivialCritique for: aMethod by: self)
            tinyHint: 'Utilisez and: au lieu de &&';
            yourself } ].
    ^ #()

"Protocole: accessing"
group
    ^ 'Erreurs débutant'

name
    ^ '&& au lieu de and:'

rationale
    ^ 'En Pharo, l''opérateur logique ET s''écrit and: et non &&'

severity
    ^ #error

"Côté classe, protocole: manifest"
ReAndOperatorRule class >> uniqueIdentifierName
    ^ 'AndOperator'
```

### Tests

```
ReExclusivelyBeginnersRuleTest << #ReAndOperatorRuleTest
    slots: {};
    tag: 'Rules';
    package: 'Renraku-Tests'

testDetectDoubleAmpersand
    | critiques |
    TestBooleanClass compile: 'testMethod
        "a && true"
        ^ 1'.
    critiques := ReAndOperatorRule new
        checkMethod: (TestBooleanClass >> #testMethod).
    self assert: critiques size equals: 1.
    self assert: (critiques first tinyHint includesSubstring: 'and:')

testNoFalsePositives
    | critiques |
    TestBooleanClass compile: 'testMethod
        | a |
        a := true.
        ^ a and: [ true ]'.
    critiques := ReAndOperatorRule new
        checkMethod: (TestBooleanClass >> #testMethod).
    self assert: critiques isEmpty

testRuleMetadata
    | rule |
    rule := ReAndOperatorRule new.
    self assert: rule severity equals: #error.
    self assert: rule group equals: 'Erreurs débutant'.
    self deny: rule name isEmpty.
    self deny: rule rationale isEmpty.
    self assert: rule class uniqueIdentifierName equals: 'AndOperator'
```

---

## 9. ReOrOperatorRule 

Détecte `||` au lieu de `or:`.

### Règle

```
ReExclusivelyBeginnersRule << #ReOrOperatorRule
    slots: {};
    tag: 'Rules';
    package: 'Renraku'

"Protocole: checking"
checkMethod: aMethod
    (aMethod sourceCode includesSubstring: '||') ifTrue: [
        ^ { (ReTrivialCritique for: aMethod by: self)
            tinyHint: 'Utilisez or: au lieu de ||';
            yourself } ].
    ^ #()

"Protocole: accessing"
group
    ^ 'Erreurs débutant'

name
    ^ '|| au lieu de or:'

rationale
    ^ 'En Pharo, l''opérateur logique OU s''écrit or: et non ||'

severity
    ^ #error

"Côté classe, protocole: manifest"
ReOrOperatorRule class >> uniqueIdentifierName
    ^ 'OrOperator'
```

### Tests

```
ReExclusivelyBeginnersRuleTest << #ReOrOperatorRuleTest
    slots: {};
    tag: 'Rules';
    package: 'Renraku-Tests'

testDetectDoublePipe
    | critiques |
    TestBooleanClass compile: 'testMethod
        "a || false"
        ^ 1'.
    critiques := ReOrOperatorRule new
        checkMethod: (TestBooleanClass >> #testMethod).
    self assert: critiques size equals: 1.
    self assert: (critiques first tinyHint includesSubstring: 'or:')

testNoFalsePositives
    | critiques |
    TestBooleanClass compile: 'testMethod
        | a |
        a := false.
        ^ a or: [ true ]'.
    critiques := ReOrOperatorRule new
        checkMethod: (TestBooleanClass >> #testMethod).
    self assert: critiques isEmpty

testRuleMetadata
    | rule |
    rule := ReOrOperatorRule new.
    self assert: rule severity equals: #error.
    self assert: rule group equals: 'Erreurs débutant'.
    self deny: rule name isEmpty.
    self deny: rule rationale isEmpty.
    self assert: rule class uniqueIdentifierName equals: 'OrOperator'
```

---

## 10. Noob 

Regroupe toutes les détections ci-dessus dans une seule règle.

### Règle

```
ReExclusivelyBeginnersRule << #Noob
    slots: {};
    tag: 'Rules';
    package: 'Renraku'

"Protocole: checking"
checkMethod: aMethod
    | critiques sourceCode |
    critiques := OrderedCollection new.
    sourceCode := aMethod sourceCode.

    { 'True' -> 'true'. 'False' -> 'false'. 'Nil' -> 'nil'. 'Self' -> 'self' }
        do: [ :aForm |
            self detectWord: aForm in: sourceCode for: aMethod critiques: critiques ].

    (self includesWord: 'return' in: sourceCode) ifTrue: [
        critiques add: ((ReTrivialCritique for: aMethod by: self)
            tinyHint: 'Utilisez ^ au lieu de return';
            yourself) ].

    { 'ifTrue[' -> 'ifTrue: ['. 'ifFalse[' -> 'ifFalse: ['. '&&' -> 'and:'. '||' -> 'or:' }
        do: [ :aForm |
            self detectPattern: aForm in: sourceCode for: aMethod critiques: critiques ].

    ^ critiques

"Protocole: private"
detectWord: aForm in: sourceCode for: aMethod critiques: critiques
    (self includesWord: aForm key in: sourceCode) ifTrue: [
        critiques add: ((ReTrivialCritique for: aMethod by: self)
            tinyHint: aForm key, ' doit être écrit ', aForm value, ' (minuscules)';
            yourself) ]

detectPattern: aForm in: sourceCode for: aMethod critiques: critiques
    (sourceCode includesSubstring: aForm key) ifTrue: [
        critiques add: ((ReTrivialCritique for: aMethod by: self)
            tinyHint: 'Utilisez ', aForm value, ' au lieu de ', aForm key;
            yourself) ]

"includesWord:in: est héritée de ReExclusivelyBeginnersRule (section 3)"

"Protocole: accessing"
group
    ^ 'Erreurs débutant'

name
    ^ 'Erreurs courantes de débutant'

rationale
    ^ 'Détecte les erreurs de syntaxe typiques des débutants en Pharo'

severity
    ^ #error

"Côté classe, protocole: manifest"
Noob class >> uniqueIdentifierName
    ^ 'NoobRule'
```

### Tests

```
ReExclusivelyBeginnersRuleTest << #NoobTest
    slots: {};
    tag: 'Rules';
    package: 'Renraku-Tests'

testDetectAllErrors
    | critiques |
    TestBooleanClass compile: 'testMethod
        | Nil Self return |
        "ifTrue[ ifFalse[ && ||"
        True ifTrue: [ ^ False ].
        ^ Nil'.
    critiques := Noob new
        checkMethod: (TestBooleanClass >> #testMethod).
    "True, False, Nil, Self, return, ifTrue[, ifFalse[, &&, ||"
    self assert: critiques size equals: 9

testNoFalsePositives
    | critiques |
    TestBooleanClass compile: 'testMethod
        true ifTrue: [ ^ false ].
        ^ nil'.
    critiques := Noob new
        checkMethod: (TestBooleanClass >> #testMethod).
    self assert: critiques isEmpty

testHintMentionsCorrection
    | critiques |
    TestBooleanClass compile: 'testMethod
        ^ True'.
    critiques := Noob new
        checkMethod: (TestBooleanClass >> #testMethod).
    self assert: critiques size equals: 1.
    self assert: (critiques first tinyHint includesSubstring: 'true')

testRuleMetadata
    | rule |
    rule := Noob new.
    self assert: rule severity equals: #error.
    self assert: rule group equals: 'Erreurs débutant'.
    self deny: rule name isEmpty.
    self deny: rule rationale isEmpty.
    self assert: rule class uniqueIdentifierName equals: 'NoobRule'
```