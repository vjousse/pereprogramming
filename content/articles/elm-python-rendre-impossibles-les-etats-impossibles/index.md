+++
title="Modélisation de données : rendre impossibles les états impossibles"
date=2022-06-02T08:00:03+01:00
draft=true

[taxonomies]
author = ["Vincent Jousse"]
tags = ["elm", "python"]
+++

_Le point de départ de cet article est la vidéo de [Richard Feldman - Making Impossible States Impossible](https://www.youtube.com/watch?v=IcgmSRJHu_8)_


Quand on a le choix entre :
- Vérifier que notre modèle est bien dans un état cohérent quand on le met à jour
- Rendre impossible les états incohérents via la modélisation elle-même

On devrait toujours privilégier la deuxième solution. Comme disait ma grand-mère « mieux vaut prévenir que guérir ! » 👵.

Cet article va vous donner quelques exemples en _Elm_ et en _Python_ sur comment modéliser au mieux vos données pour ne pas rendre possible l'impossible. Je vous aurais bien mis du _Javascript_, mais heu, j'accepte [les Pull Requests avec plaisir sur cet article](#) 😇

<!-- more -->

## Le problème

J'ai des fois tendance à me retrouver avec une modélisation de données qui peut être dans des états qui devraient être impossibles.

Par exemple, prenons une liste de questions, puis une liste de réponses associées à ces questions.

Imaginons en _Python_ ça pourrait donner ça :

```python
questions: list[str] = ["question 1", "question 2", "question 3"]
answers: list[str|None] = ["answer 1", "answer 2", None]
```

> Note: si vous n'avez pas l'habitude des _type annotations_ en Python, `List[str|None]` signifie _une liste contenant des string ou des None_. Le `|` a été ajouté avec Python 3.10, avant Python 3.10 vous pouvez obtenir la même chose avec `Union[str, None]`

Et en _Elm_ ça :

```elm
{ questions : List String
, answers : List (Maybe String)
}

{ questions =
    [ "question 1",
    , "question 2"
    , "question 3"
    ]
, answers =
    [ Just "answer 1"
    , Just "answer 2"
    , Nothing
    ]
}
```

Le souci ici, c'est que rien dans notre modélisation ne nous empêche d'avoir des réponses sans questions.

_Python_
```python
questions: list[str] = []
answers: list[str|None] = ["answer 1", "answer 2", None]
```


_Elm_
```elm
{ questions = []
, answers =
    [ Just "answer 1"
    , Just "answer 2"
    , Nothing
    ]
}
```

Ça sent le bug a plein nez non ? Qu'est-ce que notre application est censée faire de ça ? Vous allez me dire « oui mais bon, je ferai attention quand je mettrai à jour mes questions de bien mettre à jour mes réponses aussi en fonction ». Lorsque votre cerveau vous propose ce type de solution, voici la bonne posture à adopter :

![Fuyez pauvres fous](images/fuyez_pautres_fous.jpg)

Forcément, vous allez oublier de mettre à jour. Forcément, un jour, un truc ne se passera pas comme prévu. Le mainteneur du projet ça ne sera plus vous et la personne qui prendra votre relève fera la bêtise à votre place.

En programmation, j'ai fini par apprendre que plus on part du fait qu'on fera des conneries, plus la qualité de notre programme augmente.

## Rendre impossibles les états impossibles

Comment pourrions-nous changer notre modélisation pour que, quoiqu'il arrive, ces incohérences ne puissent pas arriver ?
