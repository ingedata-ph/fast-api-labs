# Lab 1 — Routes, paramètres d'URL et validation · 25 min

## 🎯 Ce que vous allez faire

Apprendre les **deux façons** de faire entrer une variable dans une API, et voir FastAPI
refuser tout seul les données mal formées — sans que vous écriviez une seule ligne de
vérification.

**Point de départ** : vous avez fini le Lab 0, votre serveur tourne.

---

## Les deux façons de passer une variable

```
GET  /fleurs/42?limit=10
          ▲     ▲
          │     └── QUERY parameter : après le « ? », optionnel, souvent un filtre
          └──────── PATH parameter  : dans le chemin, obligatoire, identifie une ressource
```

**La règle simple** :
- **Path** → *quelle* ressource je veux. `/fleurs/42` = la fleur numéro 42.
- **Query** → *comment* je la veux. `?limit=10&offset=20` = par paquets de 10, à partir du 20ᵉ.

---

## Étape 1 — Un paramètre de chemin (texte)

Ouvrez `app/main.py` et **ajoutez** cette fonction à la fin du fichier :

```python
@app.get("/echo/{message}", tags=["Démo"])
def echo(message: str):
    return {"message_recu": message}
```

**Ce qui se passe** : les accolades `{message}` dans l'adresse disent à FastAPI
« cette partie de l'URL est une variable ». Il la capture et la passe à votre fonction dans
le paramètre du **même nom**.

> ⚠️ **Le nom entre accolades et le nom du paramètre Python doivent être identiques.**
> `@app.get("/echo/{message}")` avec `def echo(msg: str)` → erreur au démarrage.

`tags=["Démo"]` sert uniquement à **regrouper** les routes dans `/docs`. C'est cosmétique,
mais quand vous aurez 20 endpoints vous serez content de l'avoir mis.

**Testez** : <http://127.0.0.1:8000/echo/bonjour>

---

## Étape 2 — Un paramètre de chemin typé (entier)

Ajoutez :

```python
@app.get("/fleurs/{fleur_id}", tags=["Démo"])
def lire_fleur(fleur_id: int):
    return {"fleur_id": fleur_id, "type_python": type(fleur_id).__name__}
```

La seule différence avec l'étape précédente : `fleur_id: int` au lieu de `: str`.
Cette petite annotation fait **trois choses à la fois** :

1. Elle **convertit** : dans une URL tout est du texte ; FastAPI transforme `"42"` en `42`.
2. Elle **valide** : si ce n'est pas convertible en entier, la requête est refusée.
3. Elle **documente** : `/docs` affichera « integer » à côté du paramètre.

**Testez les deux** :

- <http://127.0.0.1:8000/fleurs/42> → `{"fleur_id":42,"type_python":"int"}`
  Notez bien : `int`, pas `str`. La conversion a eu lieu.
- <http://127.0.0.1:8000/fleurs/abc> → **erreur 422**

Lisez la réponse d'erreur en entier :

```json
{
  "detail": [
    {
      "type": "int_parsing",
      "loc": ["path", "fleur_id"],
      "msg": "Input should be a valid integer, unable to parse string as an integer",
      "input": "abc"
    }
  ]
}
```

Elle dit **où** est le problème (`["path", "fleur_id"]`), **quoi** (`int_parsing`), et
**ce qui a été reçu** (`"abc"`). Vous n'avez écrit **aucune ligne** pour obtenir ça.

---

## Étape 3 — Des paramètres de requête (query)

Ajoutez d'abord l'import `Query` en haut du fichier. La ligne devient :

```python
from fastapi import FastAPI, Query
```

Puis ajoutez cette route — **attention à l'endroit où vous la placez**, voir l'encadré juste après :

```python
@app.get("/fleurs/", tags=["Démo"])
def lister_fleurs(
    offset: int = Query(default=0, ge=0, description="Nombre d'éléments à sauter"),
    limit: int = Query(default=10, ge=1, le=50, description="Taille de page (max 50)"),
):
    return {"offset": offset, "limit": limit}
```

**Comment FastAPI décide** si un paramètre est *path* ou *query* : si son nom apparaît entre
accolades dans l'adresse, c'est un *path*. Sinon, c'est un *query*.

**`Query(...)` permet d'ajouter des contraintes** :

| Contrainte | Signification |
|---|---|
| `default=0` | valeur si le client ne l'envoie pas → le paramètre est **optionnel** |
| `ge=0` | *greater or equal* : doit être ≥ 0 |
| `le=50` | *less or equal* : doit être ≤ 50 |
| `description=...` | le texte affiché dans `/docs` |

> ### ⚠️ PIÈGE IMPORTANT : l'ordre des routes
> **`/fleurs/` doit être déclarée AVANT `/fleurs/{fleur_id}`.**
>
> FastAPI examine les routes **dans l'ordre où vous les avez écrites** et s'arrête à la
> première qui correspond. Si `/fleurs/{fleur_id}` vient en premier, alors un appel à
> `/fleurs/moyenne` sera capté par elle, qui essaiera de convertir `"moyenne"` en entier →
> **422**, alors que la route `/fleurs/moyenne` existe peut-être plus bas.
>
> **La règle à retenir : les routes fixes se déclarent avant les routes dynamiques.**
>
> Déplacez donc `/fleurs/` au-dessus de `/fleurs/{fleur_id}` dans votre fichier.

---

## Étape 4 — Faites l'expérience

Testez ces quatre URL et **notez le code HTTP** de chacune :

| URL | Code attendu | Pourquoi |
|---|---|---|
| <http://127.0.0.1:8000/fleurs/> | 200 | valeurs par défaut : `offset=0, limit=10` |
| <http://127.0.0.1:8000/fleurs/?limit=25> | 200 | 25 est bien entre 1 et 50 |
| <http://127.0.0.1:8000/fleurs/?limit=999> | **422** | 999 > 50 |
| <http://127.0.0.1:8000/fleurs/?limit=abc> | **422** | pas un entier |

Puis allez sur `/docs` : les paramètres `offset` et `limit` apparaissent avec des champs
pré-remplis, leurs bornes et vos descriptions.

---

## 📄 Votre `app/main.py` à ce stade

```python
from fastapi import FastAPI, Query

app = FastAPI(
    title="Iris API",
    description="Service de prédiction — formation Industrialisation IA",
    version="0.1.0",
)


@app.get("/")
def racine():
    return {"service": "iris-api", "statut": "ok"}


@app.get("/echo/{message}", tags=["Démo"])
def echo(message: str):
    return {"message_recu": message}


# ⚠️ La route fixe AVANT la route dynamique
@app.get("/fleurs/", tags=["Démo"])
def lister_fleurs(
    offset: int = Query(default=0, ge=0, description="Nombre d'éléments à sauter"),
    limit: int = Query(default=10, ge=1, le=50, description="Taille de page (max 50)"),
):
    return {"offset": offset, "limit": limit}


@app.get("/fleurs/{fleur_id}", tags=["Démo"])
def lire_fleur(fleur_id: int):
    return {"fleur_id": fleur_id, "type_python": type(fleur_id).__name__}
```

---

## ✅ Vérifiez que ça marche

- [ ] `/echo/bonjour` renvoie `{"message_recu":"bonjour"}`
- [ ] `/fleurs/42` renvoie `type_python: "int"`
- [ ] `/fleurs/abc` renvoie **422** et le JSON pointe `["path","fleur_id"]`
- [ ] `/fleurs/?limit=999` renvoie **422**
- [ ] Dans `/docs`, les routes sont regroupées sous le tag **Démo**

---

## 🔧 Si ça ne marche pas

| Symptôme | Cause | Solution |
|---|---|---|
| `NameError: name 'Query' is not defined` | import oublié | `from fastapi import FastAPI, Query` |
| `/fleurs/` renvoie 422 sur `fleur_id` | ordre des routes inversé | mettez `/fleurs/` **avant** `/fleurs/{fleur_id}` |
| `AssertionError` au démarrage | nom entre `{}` ≠ nom du paramètre | vérifiez l'orthographe des deux côtés |
| `/fleurs` (sans `/` final) redirige | la route est déclarée `"/fleurs/"` | normal : FastAPI redirige (307) vers `/fleurs/` |
| Le serveur ne redémarre pas | erreur de syntaxe | regardez le terminal : la trace d'erreur y est affichée |

---

## 💡 Ce que vous venez d'apprendre

- **Path** = identifier une ressource · **Query** = filtrer, paginer, trier.
- **Une annotation de type (`: int`) est une validation.** Vous n'avez écrit aucun `if`,
  aucun `try/except`, et pourtant votre API refuse déjà proprement les données invalides.
- Ce `422` vient de **Pydantic**, la bibliothèque qui travaille sous le capot de FastAPI.
  C'est exactement l'outil qu'on utilisera cet après-midi pour protéger le modèle de ML.
- L'ordre de déclaration des routes compte.

> ### 🤔 Le lien avec la suite
> Vous venez de voir FastAPI refuser `"abc"` là où un entier était attendu. Au Lab 6, il
> refusera de la même façon `{"sepal_length": "trente"}` là où un nombre est attendu —
> **avant** que la donnée n'atteigne le modèle. C'est ça, protéger un modèle en production.

---

➡️ **Lab suivant : [Lab 2 — Pydantic, le contrat de données](lab-02-pydantic.md)**
