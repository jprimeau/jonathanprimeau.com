title: Session SQLAlchemy dans une web app d'envergure
date: 2016/8/1 9:50:00
categories:
- SQLAlchemy
---
Session SQLAlchemy dans une web app d'envergure
===============================================

Introduction
------------

J'aime bien utiliser [Flask](http://flask.pocoo.org/) et [SQLAlchemy](http://www.sqlalchemy.org/) pour développer des web services et web apps. Par contre, je préfère tirer avantage du fait que Flask soit un microframework et éviter d'établir des dépendances directes entre les portions "vues" et "modèles" de l'application. Je tente donc de contourner l'usage de l'extension [Flask-SQLAlchemy](http://flask-sqlalchemy.pocoo.org/), par exemple. Loin de moi est l'intention de diminuer l'utilité ce cette extension, mais je veux avoir la liberté de changer la portion ORM d'une web app, sans toucher aux vues et vice-versa.

Bref, le but de cet article est de montrer comment bien utiliser une session SQLAlchemy avec le concept de requête web. La [documentation](http://docs.sqlalchemy.org/en/rel_1_0/orm/contextual.html) de SQLAlchemy explique bien cette approche, mais je vous propose un exemple pratique avec Flask.

Le cycle de vie d'une session dans une web app
----------------------------------------------

Une bonne pratique est d'isoler la gestion du cycle de vie d'une session de la portion qui accède et manipule les données. Il est aussi important de comprendre quand débute et se termine la session. Dans le monde du développement d'applications web, cela veut dire que la session est créée au début de la requête et fermée à la fin de celle-ci.

SQLAlchemy offre l'objet [scoped_session](http://docs.sqlalchemy.org/en/rel_1_0/orm/contextual.html#sqlalchemy.orm.scoping.scoped_session) qui représente un registraire d'objets de session inspiré par le <i>[registry pattern](http://martinfowler.com/eaaCatalog/registry.html)</i>.

Passons à la pratique!
----------------------

Voici comment je procède, généralement, pour gérer les sessions/transactions dans les applications web que je développe. Évidemment, il s'agit d'une façon de procéder parmi plusieurs.

La première étape est commune à toute approche et il s'agit de créer l'engin de base de données. Dans ce cas-ci, je me contente d'utiliser SQLite en mémoire.
```python
db_engine = create_engine('sqlite:///:memory:')
session_factory = sessionmaker(bind=db_engine)
Session = scoped_session(session_factory)
```
Ensuite, la scoped_session est construite en lui passant une factory [sessionmaker](http://docs.sqlalchemy.org/en/rel_1_0/orm/session_api.html#sqlalchemy.orm.session.sessionmaker) afin de pouvoir construire des sessions sur demande.

Pour créer et gérer les sessions sur une base de requête avec Flask, on peut procéder de la façon suivante:
```python
@contextmanager
def session_scope():
    session = Session()
    try:
        yield session
        session.commit()
    except:
        session.rollback()
        raise
    finally:
        session.close()

@app.route("/<int:id>")
def get_item(id):
    with session_scope() as db_session:
        item = db_session.query(Item).filter(Item.id == id).first()
        if not item:
            abort(404)
        ser_item = item.serialize()
        result = jsonify(ser_item)
        return result
```
L'implémentation de la fonction session_scope() permet d'attraper les exceptions et de procéder à un rollback afin d'invalider la transaction. Par exemple, dans le code précédent, il y aurait un rollback de la transaction si la ressource item n'est pas trouvée (404). Bon d'accord, il s'agit d'une opération de lecture dans ce cas-ci, mais le principe demeurerait le même pour une exception lors de la création d'une ressource, par exemple un conflit sur l'intégrité.

Exemple complet pour la gestion des sessions avec Flask
-------------------------------------------------------

Voici un petit exemple pour démontrer l'ensemble de cette approche (app.py).

NOTE: Le code source est également disponible sur [GitHub](https://github.com/jprimeau/blog_sqlalchemy_dbsession).

```python
from contextlib import contextmanager

from flask import Flask, abort, jsonify
from sqlalchemy import Column, Integer, String, create_engine
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import scoped_session, sessionmaker

db_engine = create_engine('sqlite:///:memory:')
session_factory = sessionmaker(bind=db_engine)
Session = scoped_session(session_factory)


@contextmanager
def session_scope():
    session = Session()
    try:
        yield session
        print '[SESSION] Comitting...'
        session.commit()
    except:
        print '[SESSION] Rolling back...'
        session.rollback()
        raise
    finally:
        session.close()

BaseModel = declarative_base()


class Item(BaseModel):
    __tablename__ = 'items'

    id = Column(Integer, primary_key=True)
    name = Column(String(16))

    def serialize(self):
        return {
            'id': self.id,
            'name': self.name,
        }

BaseModel.metadata.create_all(db_engine)

app = Flask(__name__)


@app.errorhandler(404)
def page_not_found(error):
    error = {'error': 'not found'}
    result = jsonify(error)
    return result, 404


@app.route("/", methods=['POST'])
def create_item():
    with session_scope() as db_session:
        item = Item()
        db_session.add(item)
        db_session.flush()

        item.name = 'Item #' + str(item.id)
        db_session.flush()

        ser_item = item.serialize()
        result = jsonify(ser_item)
        return result


@app.route("/")
def get_items():
    with session_scope() as db_session:
        items = db_session.query(Item)
        ser_items = [item.serialize() for item in items]
        result = jsonify({'items': ser_items})
        return result


@app.route("/<int:id>")
def get_item(id):
    with session_scope() as db_session:
        item = db_session.query(Item).filter(Item.id == id).first()
        if not item:
            abort(404)
        ser_item = item.serialize()
        result = jsonify(ser_item)
        return result

if __name__ == "__main__":
    app.run()
```

Pour exécuter cet exemple, vous aurez besoin de Flask, SQLAlchemy et httpie. Le plus simple est d'utiliser [virtualenvwrapper](http://virtualenvwrapper.readthedocs.io/en/latest/) pour construire un environnement de travail. Voici ce qu'il suffit de faire pour installer les dépendances et lancer l'exemple:
```bash
$ pip install Flask SQLAlchemy httpie
$ python app.py runserver
```

Ensuite, dans un autre terminal sous le même environnement virtualenv, on peut utiliser l'application de la façon suivante grâce à [httpie](https://github.com/jkbrzt/httpie):
```bash
$ http GET http://localhost:5000/
HTTP/1.0 200 OK
Content-Length: 18
Content-Type: application/json
Date: Thu, 28 Jul 2016 01:21:06 GMT
Server: Werkzeug/0.11.10 Python/2.7.12

{
    "items": []
}
```

Dans l'appel précédent, la liste d'items est vide, car nous n'avons pas créé de ressource. Pour créer une ressource Item, il suffit de faire un POST comme ceci:
```bash
$ http POST http://localhost:5000/ name='Item #1'
HTTP/1.0 200 OK
Content-Length: 36
Content-Type: application/json
Date: Thu, 28 Jul 2016 01:22:55 GMT
Server: Werkzeug/0.11.10 Python/2.7.12

{
    "id": 1, 
    "name": "Item #1"
}
```

L'item #1 figure maintenant dans la liste des items disponibles:
```bash
$ http GET http://localhost:5000/
HTTP/1.0 200 OK
Content-Length: 73
Content-Type: application/json
Date: Thu, 28 Jul 2016 01:23:38 GMT
Server: Werkzeug/0.11.10 Python/2.7.12

{
    "items": [
        {
            "id": 1, 
            "name": "Item #1"
        }
    ]
}
```

Si l'on tente d'accéder à une ressource inexistante, une exception sera levée et le rollback sera effectué. Voici un exemple:
```bash
$ http GET http://localhost:5000/99
HTTP/1.0 404 NOT FOUND
Content-Length: 27
Content-Type: application/json
Date: Thu, 28 Jul 2016 01:25:52 GMT
Server: Werkzeug/0.11.10 Python/2.7.12

{
    "error": "not found"
}
```

Et dans le terminal avec l'application en cours d'exécution, nous verrons ceci à la sortie:
```bash
[SESSION] Rolling back...                                    
127.0.0.1 - - [27/Jul/2016 21:25:52] "GET /99 HTTP/1.1" 404 -
```
Le rollback est bel et bien appelé lors d'une erreur 400.

Conclusion
----------
Je vous ai démontré une façon de gérer les sessions SQLAlchemy dans le cadre d'une application web. Il existe, évidemment, d'autres façons de faire. N'hésitez pas à consulter la documentation de SQLAlchemy pour plus d'informations.
