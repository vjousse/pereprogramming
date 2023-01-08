+++
title="Python, Elm et modélisation de données : rendre impossibles les états impossibles"
date=2022-06-02T08:00:03+01:00

[taxonomies]
author = ["Vincent Jousse"]
tags = ["elm", "python"]
+++

_Cet article est fortement inspiré de la vidéo de [Richard Feldman - Making Impossible States Impossible](https://www.youtube.com/watch?v=IcgmSRJHu_8)_


Quand on a le choix entre :
- Vérifier que notre modèle est bien dans un état cohérent quand on le met à jour
- Rendre impossible les états incohérents via la modélisation elle-même, ça empêchera de devoir le faire via le code / la logique

On devrait toujours privilégier la deuxième solution. Comme disait ma grand-mère « mieux vaut prévenir que guérir ! » 👵.

Cet article va vous donner quelques exemples en _Elm_ et en _Python_ sur comment modéliser au mieux vos données pour ne pas rendre possible l'impossible. Je vous aurais bien mis du _Javascript_, mais heu, j'accepte [les Pull Requests avec plaisir sur cet article](#) 😇

<!-- more -->

## Le problème

J'ai des fois tendance à me retrouver avec une modélisation de données qui peut être dans des états qui devraient être impossibles.

Par exemple, prenons une liste de questions (représentée par une liste de strings), puis une liste de réponses (représentée par une liste de string ou d'absence de valeur) associées à ces questions.

Imaginons en _Python_ ça pourrait donner ça :

```python
questions: list[str] = ["question 1", "question 2", "question 3"]
responses: list[str|None] = ["response 1", "response 2", None]
```

> Note: si vous n'avez pas l'habitude des _type annotations_ en Python, `List[str|None]` signifie _une liste contenant des string ou des None_. Le `|` a été ajouté avec Python 3.10, avant Python 3.10 vous pouvez obtenir la même chose avec `Union[str, None]`

Et en _Elm_ ça :

```elm
{ questions : List String
, responses : List (Maybe String)
}

{ questions =
    [ "question 1",
    , "question 2"
    , "question 3"
    ]
, responses =
    [ Just "response 1"
    , Just "response 2"
    , Nothing
    ]
}
```

Le souci ici, c'est que rien dans notre modélisation ne nous empêche d'avoir des réponses sans questions.

_Python_
```python
questions: list[str] = []
responses: list[str|None] = ["response 1", "response 2", None]
```


_Elm_
```elm
{ questions = []
, responses =
    [ Just "response 1"
    , Just "response 2"
    , Nothing
    ]
}
```

Ça sent le bug à plein nez non ? Qu'est-ce que notre application est censée faire de ça ? Vous allez me dire « oui mais bon, je ferai attention quand je mettrai à jour mes questions de bien mettre à jour mes réponses aussi en fonction ». Lorsque votre cerveau vous propose ce type de solution, voici la bonne posture à adopter :

![Gandalf : fuyez pauvres fous](images/fuyez_pautres_fous.jpg "Gandalf : fuyez pauvres fous")

Forcément, vous allez oublier de mettre à jour. Forcément, un jour, un truc ne se passera pas comme prévu. Le mainteneur du projet ça ne sera plus vous et la personne qui prendra votre relève fera la bêtise à votre place.

En programmation, j'ai fini par apprendre que plus on part du fait qu'on fera des conneries, plus la qualité de notre programme augmente.

## Rendre impossibles les états impossibles

Comment pourrions-nous changer notre modélisation pour que, quoiqu'il se passe, ces incohérences ne puissent pas arriver ?

Rien de plus simple, il nous suffirait d'avoir une classe `Question` qui pourrait modéliser ce qu'est une question : un libellé et une possible réponse.



_Python_
```python
from dataclasses import dataclass


@dataclass
class Question:
    prompt: str
    response: str | None


questions: list[Question] = [
    Question(prompt="question 1", response="response 1"),
    Question(prompt="question 2", response="response 2"),
    Question(prompt="question 3", response=None),
]
```

_Elm_
```elm
type alias Question =
    { prompt : String
    , response : Maybe String
    }


questions =
    [ { prompt = "question 1", response = "response 1" }
    , { prompt = "question 2", response = "response 2" }
    , { prompt = "question 3", response = Nothing }
    ]
```

La modélisation de nos données rend maintenant impossible le fait d'avoir une question sans réponse !

Cet exemple est assez simple mais vous comprenez le principe : à chaque fois qu'on modélise quelque chose, il est bon de se poser la question si notre modélisation permet, ou pas, des états qui devraient être impossibles.

## Bonus : modéliser un historique

Essayons d'aller un peu plus loin dans notre modélisation. Imaginons maintenant que nous voulions modéliser un historique de questions. On aimerait connaître quelle est la question actuelle, quelles sont les questions passées et quelles sont les questions à venir.

On pourrait imaginer quelque chose comme cela :


_Python_

```python
from dataclasses import dataclass


@dataclass
class Question:
    prompt: str
    response: str | None


@dataclass
class History:
    questions: list[Question]
    current: Question


questions: list[Question] = [
    Question(prompt="question 1", response="response 1"),
    Question(prompt="question 2", response="response 2"),
    Question(prompt="question 3", response=None),
]

history: History = History(questions=questions, current="question 1")
```


_Elm_

```elm
type alias History =
    { questions : List question
    , current : Question
    }

-- Rest of the code

{ questions = [question1, question2, question3]
, current = question1
}
```

Le problème ici, c'est que rien ne nous empêche d'avoir ce type d'état :

_Python_

```python
history: History = History(questions=[], current="question 1")
```

_Elm_

```elm
{ questions = []
, current = question1
}
```
