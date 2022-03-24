+++
title="Le guide complet du débutant avec FastAPI - Partie 3 : réorganisation du code, tests automatisés"
date=2021-06-25T19:33:20+01:00

[taxonomies]
author = ["Vincent Jousse"]
tags = ["python", "framework", "fastapi", "web", "fastapi-tutorial"]
+++

## Restructuration du code

Jusqu'ici nous avons placé tout notre code dans le même fichier `main.py`. Même si nous pourrions continuer comme cela, il est souvent préférable de séparer son code dans des fichiers et des modules différents. Cela va nous aider à nous y retrouver et va encourager le fait de séparer les responsabilités/préoccupations ([Separation of concerns](https://en.wikipedia.org/wiki/Separation_of_concerns) en anglais). Lors de la [partie 2](/articles/le-guide-complet-du-debutant-avec-fastapi-partie-2/) nous avions déjà posé quelques bases en créant des répertoires pour les modèles, les templates, le core, etc. Il est maintenant temps d'aller plus loin et de les utiliser à bon escient.

### Les modèles

Créez un fichier `article.py` dans le répertoire précédemment créé à l'emplacement `app/models`.

Mettez-y le contenu de votre modèle qui se trouvait précédemment dans `main.py`


```python
# app/models/article.py

from tortoise import fields
from tortoise.models import Model


class Article(Model):

    id = fields.IntField(pk=True)

    title = fields.TextField()
    content = fields.TextField()

    created_at = fields.DatetimeField(auto_now_add=True)
    updated_at = fields.DatetimeField(auto_now=True)

    def __str__(self):
        return self.title

```

N'oubliez pas de supprimer les lignes correspondantes dans `main.py`. Il nous faut maintenant importer ce nouveau fichier dans `main.py`.

```python
from app.models.article import Article
```

L'entête de votre fichier `main.py` devrait ressembler à cela maintenant :


```python
# app/main.py

from fastapi import FastAPI, Request
from fastapi.staticfiles import StaticFiles
from fastapi.templating import Jinja2Templates
from tortoise.contrib.fastapi import register_tortoise
from app.models.article import Article

# … reste du fichier
```

Pour être sûr que vous n'avez rien cassé, vous pouvez aller vérifier que [la page listant vos articles](/articles) fonctionne toujours correctement.

### Les vues

Maintenant que nous avons rangé notre modèle à sa place, nous allons faire de même avec les vues, aka les fonctions qui construisent le résultat envoyé au navigateur, que ça soit du HTML ou du JSON.

Pour pouvoir mettre les vues autre part que dans le fichier `main.py`, FastAPI utilise le concept de __routeur__. Ce routeur va faire le lien entre votre app principale FastAPI et des vues placées dans des modules/fichiers Python.

Dans votre répertoire `app/views`, créez un fichier nommé `article.py` qui contiendra vos vues. Placez-y le code suivant :

```python
# app/views/article.py

from fastapi import APIRouter
from fastapi import Request
from app.models.article import Article

articles_views = APIRouter()
```

Ici nous préparons les imports dont nous allons avoir besoin et créons un objet `articles_views` de type `APIRouter` qui va nous permettre d'y attacher nos vues. Copiez-collez vos routes du fichier `main.py` vers `app/views/article.py`, ainsi que la déclaration de la variable `templates`. La seule chose que vous devriez avoir à changer est le décorateur situé avant chaque fonction. Au lieu d'appeler `app.get` vous devrez maintenant appeler `articles_views.get`.

Voic le contenu complet du fichier `app/views/article.py` :

```python
from fastapi import APIRouter
from fastapi import Request
from fastapi.templating import Jinja2Templates
from app.models.article import Article

articles_views = APIRouter()

templates = Jinja2Templates(directory="app/templates")


@articles_views.get("/articles/create", include_in_schema=False)
async def articles_create(request: Request):

    article = await Article.create(
        title="Mon titre de test",
        content="Un peu de contenu<br />avec deux lignes"
    )

    return templates.TemplateResponse(
        "articles_create.html",
        {
            "request": request,
            "article": article
        })


@articles_views.get("/articles", include_in_schema=False)
async def articles_list(request: Request):

    articles = await Article.all().order_by('created_at')

    return templates.TemplateResponse(
        "articles_list.html",
        {
            "request": request,
            "articles": articles
        })


@articles_views.get("/api/articles")
async def api_articles_list():

    articles = await Article.all().order_by('created_at')

    return articles


@articles_views.get("/", include_in_schema=False)
async def root(request: Request):

    return templates.TemplateResponse(
        "home.html",
        {
            "request": request
        })
```

Nous allons maintenant devoir faire le ménage dans `main.py` et appeler la fonction `app.include_router` pour inclure le routeur que nous venons de déclarer. Voici à quoi devrait ressembler votre fichier `main.py` :

```python
from fastapi import FastAPI
from fastapi.staticfiles import StaticFiles
from tortoise.contrib.fastapi import register_tortoise
from app.views.article import articles_views

app = FastAPI()

app.mount("/public", StaticFiles(directory="public"), name="public")

register_tortoise(
    app,
    db_url="sqlite://db.sqlite3",
    modules={"models": ["app.models.article"]},
    generate_schemas=True,
    add_exception_handlers=True,
)

app.include_router(
    articles_views,
    tags=["Articles"])
```

### Configuration

Pour finir cette partie sur la restructuration du code, nous allons voir comment __placer toutes nos configurations au même endroit__ pour s'y retrouver plus facilement. Nous allons par exemple y mettre l'url d'accès à la base de données, l'emplacement des templates ou encore la liste de modèles pour `Tortoise`. Même si notre projet n'est pas très complexe pour l'instant, c'est une bonne pratique qui pourra vous faire gagner du temps à l'avenir.

Créez un fichier `config.py` dans votre répertoire `app/core/` et mettez-y le contenu qui suit :

```python
# app/core/config.py

from pydantic import BaseSettings
import os
from fastapi.templating import Jinja2Templates

dir_path = os.path.dirname(os.path.realpath(__file__))


class Settings(BaseSettings):
    APP_NAME: str = "fastapi-tutorial"
    APP_VERSION: str = "0.0.1"
    SQLITE_URL: str = "sqlite://db.sqlite3"

    TORTOISE_MODELS = [
        "app.models.article"
    ]

    TEMPLATES_DIR = os.path.join(dir_path, "..", "templates")
    STATIC_FILES_DIR = os.path.join(dir_path, "..", "..", "public")


settings = Settings()

templates = Jinja2Templates(directory=settings.TEMPLATES_DIR)
```

Il vous faut maintenant modifier votre fichier `main.py` pour utiliser les valeurs de votre configuration plutôt que celles qui étaient codées en dur :

```python
# app/main.py

from fastapi import FastAPI
from fastapi.staticfiles import StaticFiles
from tortoise.contrib.fastapi import register_tortoise
from app.views.article import articles_views
from app.core.config import settings

app = FastAPI()

app.mount("/public",
          StaticFiles(directory=settings.STATIC_FILES_DIR),
          name="public")

register_tortoise(
    app,
    db_url=settings.SQLITE_URL,
    modules={"models": settings.TORTOISE_MODELS},
    generate_schemas=True,
    add_exception_handlers=True,
)

app.include_router(
    articles_views,
    tags=["Articles"])
```

Il ne vous reste plus qu'à utiliser l'objet `templates` que nous avons créé dans notre configuration au sein de vos vues plutôt que de le recréer à chaque fois (ce que nous aurions du faire si nous avions eu plusieurs fichiers de vues) :

```python
# app/views/article.py

from fastapi import APIRouter
from fastapi import Request
from app.models.article import Article
from app.core.config import templates

articles_views = APIRouter()


@articles_views.get("/articles/create", include_in_schema=False)
async def articles_create(request: Request):
# … reste du fichier

```

Vous noterez que nous avons enlevé la création de l'objet `templates` et avons importé celui déjà créé dans notre fichier `config.py`. N'oubliez pas d'enlever l'import de `Jinja2Templates` qui ne sert plus à rien.

## Tests

L'écriture de tests est un sujet qui reviendra souvent dans le code que nous allons effectuer et pour cause : si vous voulez garantir la __qualité de votre code__, vous devez écrire des __tests automatisés__ pour s'assurer qu'il fonctionne correctement.

Si vous ne savez pas ce que c'est, ce n'est rien de plus qu'un petit __robot__/__bout de code__ qui va se charger d'appeler différentes partie de votre code et va s'assurer qu'il se comporte bien comme il devrait.

Si vous pensez que « c'est bon je peux tester tout seul à la main » ou encore que « mon code n'est pas si compliqué que ça, pas besoin de s'embêter » vous êtes soit très débutant et il serait bien de me croire sur parole, ou soit très expérimenté et là je ne peux plus rien pour vous :wink:

Quoiqu'il en soit, lorsque que l'on utilise un langage comme Python où les erreurs sont détectées au _runtime_, c'est à dire lorsque l'application est lancée et utilisée, c'est une bonne idée de mettre toutes les chances de son côté pour détecter les bugs avant que l'application soit utilisée. Sinon, évidemment, vous laisserez le soin à vos utilisateurs de découvrir vos bugs.

> __Bon à savoir__ : il existe d'autres langages de programmation que l'on appelle _statiquement typés_ et qui disposent d'un compilateur qui vérifira une partie des erreurs avant même que vous puissiez lancer l'application (_Rust_, _Java_,…). Mais comme ce n'est pas le cas de Python, écrire des tests est primordial. À noter que même si ces langages disposent d'un compilateur, il ne dispense pas d'écire tous les tests mais seulement quelques tests basiques.


Le logiciel qui va nous permettre de tester notre programme est appelé __`pytest`__. Commençons pas l'installer dans notre `virtualenv` ainsi qu'une librairie utile à `Tortoise` :

```
(venv) $ pip install pytest asynctest
```

Nous allons commencer par effectuer un test tout simple : s'assurer que notre page d'accueil `/` se charge sans erreur.

### Test de la page d'accueil

Nous allons commencer par écrire le test qui va charger la page d'accueil. Si ce n'est pas déjà fait assurez-vous d'avoir créé le répertoire `app/tests/` et d'y avoir mis un fichier vide nommé `__init__.py`.

Créez ensuite un répertoire où nous allons mettre les tests de nos vues : `app/tests/views/`, placez-y un fichier `__init__.py` vide.

Dans `app/tests/views/` créez un fichier nommé `test_home.py` avec le contenu suivant :

```python
# app/tests/views/test_home.py

from fastapi.testclient import TestClient


def test_home(client: TestClient) -> None:

    response = client.get("/")
    assert response.status_code == 200
    assert "Current url" in response.text

```

Ici, nous utilisons un "faux navigateur/client web" nommé `TestClient` qui va nous permettre d'aller charger des pages comme nous le ferions avec Chrome ou Firefox de manière presque classique (avec l'exécution du Javascript en moins, entre autres).

Nous demandons à notre client d'aller charger la page à l'adresse `/`. Nous utilisons ensuite un mot clé dédié aux tests : __`assert`__. Ce mot clé nous permet de nous assurer que la condition qui suit est vraie. Dans notre cas, nous commençons par vérifier que le _status code_ HTTP renvoyé par FastAPI est bien 200.

Lors d'une requête HTTP sur le web, le serveur renvoie __toujours__ un code de statut pour nous informer du déroulement de la requête. Le code 200 nous indique que tout s'est déroulé sans erreur.

> __Bon à savoir__ : 
> Si le code de statut commence par __5__, c'est une erreur du côté du serveur, donc vous ne pouvez a priori rien y faire (500 par exemple)
> 
> Si le code de statut commence par __4__, c'est une erreur du côté du client HTTP (donc vous) : 404 page non trouvée (mauvaise adresse), 403 accès refusé (besoin de se connecter), …
>
> Si le code de statut commence par __3__, le serveur indique une redirection (changement d'adresse)
>
> Si le code de statut commence par __2__, c'est que tout va bien
>
> Si le code de statut commence par __1__, cela indique juste la réception d'une information
>
> Vous trouverez plus de détail sur la [page wikipedia dédiée](https://fr.wikipedia.org/wiki/Liste_des_codes_HTTP)

Vous noterez que l'on teste aussi le contenu de la page HTML en s'assurant que le contenu qui nous est renvoyé contient bien la phrase `Current url`.

Pour pouvoir lancer ce test, il va nous manquer un fichier qui nous permet de configurer `pytest` et notamment qui va nous permettre de créer les objets qui sont nécessaires à l'exécution des tests.

Créez un fichier nommé `conftest.py` dans `app/test/conftest.py` et placez-y le contenu suivant :

```python
# app/tests/conftest.py

import os
import pytest
from tortoise.contrib.test import finalizer, initializer
from app.core.config import settings
from typing import Generator
from fastapi.testclient import TestClient
from app.main import app


@pytest.fixture(scope="module")
def client() -> Generator:
    db_url = os.environ.get("TORTOISE_TEST_DB", "sqlite://:memory:")
    initializer(settings.TORTOISE_MODELS, db_url=db_url, app_label="models")
    with TestClient(app) as c:
        yield c
    finalizer()
```

Je ne vais pas rentrer dans les détails de la tambouille interne de `pytest`, de `FastAPI` et de `tortoise` pour vous expliquer le code ci-dessus, mais croyez-moi sur parole, il fonctionne :wink:

Vous devriez maintenant être capable de lancer votre test avec la ligne de commande suivante à la racine du projet :

```
(venv) $  pytest --disable-warnings app/tests/views/test_home.py
```

> __Warnings__ : 
> La version actuelle de _Tortoise ORM_ affiche des warnings de ce style `DeprecationWarning: "@coroutine" decorator is deprecated since Python 3.8, use "async def" instead` à cause de l'utilisation d'une ancienne librairie nommée `asynctest`.
>
> Avec un peu de chance, le problème sera bientôt réglé, mais pour l'instant il est nécessaire d'utiliser l'option `--disable-warnings` de pytest pour ne pas les afficher
> 

Vous devriez voir un résultat de ce genre s'afficher :

```
====================================== test session starts =======================================
platform linux -- Python 3.9.6, pytest-6.2.4, py-1.10.0, pluggy-0.13.1
rootdir: /home/vjousse/usr/src/python/fastapi-beginners-guide
collected 1 item

app/tests/views/test_home.py .                                                             [100%]

================================= 1 passed, 18 warnings in 0.03s =================================

```

Alors certes nous n'avons pour l'instant qu'une vue à tester mais, croyez-moi, lorsque vous en aurez des dizaines voire des centaines, les tester manuellement ne sera même plus envisageable.

### Test des articles

Créez un fichier nommé `test_articles.py` dans le répertoire `app/tests/views/` et placez-y le contenu suivant :


```python
# app/tests/views/test_articles.py

from fastapi.testclient import TestClient
import asyncio

from app.models.article import Article


async def create_article():
    article = await Article.create(
        title="Mon titre de test",
        content="Un peu de contenu<br />avec deux lignes"
    )

    return article


def test_create_article(client: TestClient,
                        event_loop: asyncio.AbstractEventLoop) -> None:

    article = event_loop.run_until_complete(create_article())

    response = client.get("api/articles")
    assert response.status_code == 200
    content = response.json()
    first_article = content[0]
    assert first_article["title"] == article.title
    assert first_article["content"] == article.content

```

Dans ce test nous créons un article dans la base de données et nous vérifions ensuite qu'il est bien affiché dans le json de la page `api/articles`. Rien de bien sorcier ici, si ce ne sont les références à l'`event_loop`.

Pour essayer de rester simple : `FastAPI` et `Tortoise ORM` tournent dans un environnement asynchrone mais pas `pytest`. C'est pourquoi, à chaque fois que nous voudrons utiliser des accès à la base de données (asynchrones) nous devrons le faire en utilisant explicitement la machinerie asynchrone de FastAPI en passant par l'event loop et la méthode `event_loop.run_until_complete`.

Pour que ce code fonctionne, vous devrez modifier le fichier `app/tests/conftest.py` pour y ajouter la méthode `event_loop` comme ci-dessous :

```python
# app/tests/conftest.py

import os
import pytest
from tortoise.contrib.test import finalizer, initializer
from app.core.config import settings
from typing import Generator
from fastapi.testclient import TestClient
from app.main import app


@pytest.fixture(scope="module")
def client() -> Generator:
    db_url = os.environ.get("TORTOISE_TEST_DB", "sqlite://:memory:")
    initializer(settings.TORTOISE_MODELS, db_url=db_url, app_label="models")
    with TestClient(app) as c:
        yield c
    finalizer()


@pytest.fixture(scope="module")
def event_loop(client: TestClient) -> Generator:
    yield client.task.get_loop()
```

Ensuite, lancez votre test comme précédemment :

```
(venv) $  pytest --disable-warnings app/tests/views/test_articles.py
```

Vous devriez obtenir quelque chose de ce style :


```
====================================== test session starts =======================================
platform linux -- Python 3.9.6, pytest-6.2.4, py-1.10.0, pluggy-0.13.1
rootdir: /home/vjousse/usr/src/python/fastapi-beginners-guide
collected 1 item

app/tests/views/test_articles.py .                                                         [100%]

================================= 1 passed, 18 warnings in 0.03s =================================
```

Vous pouvez lancer tous vos tests en même temps en ne spécifiant que le répertoire des tests à pytest :

```
(venv) $  pytest --disable-warnings app/tests/
```

Vos deux fichiers de tests sont alors mentionnés dans le rapport :

```
====================================== test session starts =======================================
platform linux -- Python 3.9.6, pytest-6.2.4, py-1.10.0, pluggy-0.13.1
rootdir: /home/vjousse/usr/src/python/fastapi-beginners-guide
collected 2 items

app/tests/views/test_articles.py .                                                         [ 50%]
app/tests/views/test_home.py .                                                             [100%]

================================= 2 passed, 18 warnings in 0.05s =================================

```

> __Juste au cas où__ : 
> Si des fois vous obtenez une erreur de ce style en lançant les tests : `got Future <Future pending> attached to a different loop` assurez-vous de ne pas avoir `pytest-asyncio` d'installé. Si c'est le cas, c'est surement lui qui pose souci. Assurez vous de le désinstaller auparavant :
>
> `pip uninstall pytest-asyncio`

## Conclusion

Nous venons d'achever une étape qui peut paraître fastidieuse mais qui est haut combien importante : le refactoring et le test de notre code. Cette étape nous permet maintenant de partir sur des bases propres et solides pour ajouter des fonctionnalités à notre application.

Comme d'habitude, le code pour cette partie est [accessible directement sur Github](https://github.com/vjousse/fastapi-beginners-guide/tree/part3).
