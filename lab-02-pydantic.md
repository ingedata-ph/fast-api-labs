# Lab 2 — Pydantic : le contrat de données · 30 min

## 🎯 Ce que vous allez faire

Recevoir des données **dans le corps d'une requête** (et non plus dans l'URL), en décrivant
leur forme avec un **schéma Pydantic**.

> ### C'est le lab le plus important de la journée après le Lab 6.
> Le réflexe que vous construisez ici — **décrire la donnée avant de l'utiliser** — est
> exactement celui qui protégera votre modèle de ML cet après-midi. Le schéma
> `MesureFleur` que vous écrivez maintenant sera **réutilisé tel quel** au Lab 6 comme
> contrat d'entrée de `/predict`.

**Point de départ** : Lab 1 terminé.

---

## Pourquoi on ne peut pas tout mettre dans l'URL

Jusqu'ici, vos variables passaient par l'adresse : `/fleurs/42`. Ça marche pour un identifiant.

Mais imaginez créer un utilisateur avec un nom, un âge, un email, une adresse, un mot de passe.
Vous n'allez pas écrire :

```
/creer?nom=Alice&age=30&email=a@b.c&adresse=...&motdepasse=secret
```

C'est illisible, limité en longueur, et un mot de passe dans une URL **finit dans les logs de
tous les serveurs traversés**.

Pour `POST` et `PUT`, on envoie donc les données dans le **corps de la requête**
(*request body*), en JSON. Et pour dire à FastAPI à quoi ce corps doit ressembler, on lui
donne un **moule** : une classe Pydantic.

---

## Étape 1 — Écrire le schéma d'entrée

En haut de `app/main.py`, ajoutez les imports :

```python
from datetime import datetime, timezone

from fastapi import FastAPI, Query, status
from pydantic import BaseModel, Field
```

Puis, **juste après la ligne `app = FastAPI(...)`**, ajoutez :

```python
class MesureFleur(BaseModel):
    """Ce que le CLIENT nous envoie."""

    sepal_length: float = Field(..., gt=0, lt=30, description="Longueur du sépale en cm")
    sepal_width: float = Field(..., gt=0, lt=30, description="Largeur du sépale en cm")
    petal_length: float = Field(..., gt=0, lt=30, description="Longueur du pétale en cm")
    petal_width: float = Field(..., gt=0, lt=30, description="Largeur du pétale en cm")

    model_config = {
        "json_schema_extra": {
            "examples": [
                {
                    "sepal_length": 5.1,
                    "sepal_width": 3.5,
                    "petal_length": 1.4,
                    "petal_width": 0.2,
                }
            ]
        }
    }
```

**Décortiquons** :

| Élément | Rôle |
|---|---|
| `class MesureFleur(BaseModel)` | on hérite de `BaseModel` : c'est ça qui rend la classe « intelligente » |
| `sepal_length: float` | le champ s'appelle `sepal_length` et doit être un nombre décimal |
| `Field(...)` | les trois points signifient **obligatoire** (pas de valeur par défaut) |
| `gt=0` | *greater than* : strictement supérieur à 0. Une longueur négative n'existe pas |
| `lt=30` | *less than* : moins de 30 cm. Au-delà, ce n'est plus un iris |
| `model_config` avec `examples` | pré-remplit le formulaire de `/docs`. **Faites-le toujours** : ça change la vie de celui qui teste votre API |

> **Sépale, pétale ?** Le sépale est la partie verte sous la fleur, le pétale la partie
> colorée. Le jeu de données Iris mesure les deux, en longueur et en largeur : 4 nombres.
> Pas besoin d'être botaniste, ce sont juste 4 nombres.

---

## Étape 2 — Écrire le schéma de sortie

Juste en dessous, ajoutez :

```python
class MesureEnregistree(MesureFleur):
    """Ce que le SERVEUR renvoie."""

    id: int
    recue_le: datetime
```

`MesureEnregistree` **hérite** de `MesureFleur` : elle a donc les 4 mesures, **plus**
un `id` et une date.

> ### 💡 Deux schémas, pas un. C'est le point clé du lab.
> - **Entrée** (`MesureFleur`) : ce que le client a le droit d'envoyer.
> - **Sortie** (`MesureEnregistree`) : ce que le serveur garantit de renvoyer.
>
> Pourquoi séparer ? Parce que **le client n'a pas à choisir l'`id`** : c'est le serveur qui
> l'attribue. Si vous n'aviez qu'un seul schéma, un client malin pourrait envoyer
> `{"id": 999, ...}` et écraser vos données.
>
> On appelle ça un **DTO** (*Data Transfer Object*). Cet après-midi ce sera
> `FeaturesIris` en entrée et `ReponsePrediction` en sortie. **Même structure, même raison.**

---

## Étape 3 — L'endpoint POST

Ajoutez une « fausse base de données » (une simple liste en mémoire) et deux routes,
à la fin du fichier :

```python
# Notre "base de données" pour aujourd'hui : une liste en mémoire.
# Elle est vidée à chaque redémarrage du serveur — c'est normal, on branchera
# une vraie base au Lab 8.
mesures: list[MesureEnregistree] = []


@app.post(
    "/mesures/",
    response_model=MesureEnregistree,
    status_code=status.HTTP_201_CREATED,
    tags=["Mesures"],
)
def creer_mesure(mesure: MesureFleur):
    enregistree = MesureEnregistree(
        id=len(mesures) + 1,
        recue_le=datetime.now(timezone.utc),
        **mesure.model_dump(),
    )
    mesures.append(enregistree)
    return enregistree


@app.get("/mesures/", response_model=list[MesureEnregistree], tags=["Mesures"])
def lister_mesures():
    return mesures
```

**Les points à comprendre** :

- **`def creer_mesure(mesure: MesureFleur)`** — le paramètre est annoté avec un schéma
  Pydantic (et non `int` ou `str`). FastAPI en déduit : « cette donnée arrive dans le corps
  de la requête ». Il lit le JSON, le valide, et vous donne un objet Python prêt à l'emploi.

- **`response_model=MesureEnregistree`** — FastAPI **filtre** la réponse selon ce schéma.
  Si votre fonction renvoyait par erreur un mot de passe, il serait retiré. C'est une
  sécurité, pas seulement de la documentation.

- **`status_code=status.HTTP_201_CREATED`** — par défaut FastAPI renvoie `200`. Pour une
  **création**, la norme HTTP demande `201`. Utilisez la constante `status.HTTP_201_CREATED`
  plutôt que le nombre `201` : c'est plus lisible.

- **`**mesure.model_dump()`** — `model_dump()` transforme l'objet Pydantic en dictionnaire
  `{"sepal_length": 5.1, ...}`, et `**` le « déplie » en arguments nommés. C'est un
  raccourci pour éviter de réécrire les 4 champs à la main.

---

## Étape 4 — Testez, et surtout : cassez-le

Allez sur <http://127.0.0.1:8000/docs>.

**a) Le cas qui marche.** `POST /mesures/` → « Try it out ». Le formulaire est **déjà
rempli** avec votre exemple. Cliquez « Execute ».
→ Code **201**, et la réponse contient votre mesure **plus** un `id` et un `recue_le`.

**b) Faites-le échouer.** Remplacez `5.1` par `-5` et exécutez.
→ Code **422**. Lisez le message :

```json
{"detail":[{"type":"greater_than","loc":["body","sepal_length"],
 "msg":"Input should be greater than 0","input":-5}]}
```

**c) Enlevez un champ.** Supprimez la ligne `"petal_width": 0.2` et exécutez.
→ **422**, `"msg": "Field required"`.

**d) Envoyez du texte.** Mettez `"sepal_length": "grand"`.
→ **422**, `"msg": "Input should be a valid number"`.

**e) Le serveur a-t-il planté ?** Non. Il tourne toujours. Vérifiez avec `GET /mesures/` :
seules les mesures valides ont été enregistrées.

> ### 🤔 Arrêtez-vous 30 secondes sur ce que vous venez de voir
> Vous avez envoyé quatre requêtes invalides. Votre API les a toutes refusées, avec un
> message précis pour chacune, **et vous n'avez écrit aucun `if`**.
>
> Imaginez maintenant que derrière cet endpoint il y ait un modèle de Machine Learning.
> Sans cette validation, `"grand"` arriverait jusqu'à `numpy`, qui planterait avec une
> erreur incompréhensible — ou pire, ne planterait pas et prédirait n'importe quoi.

---

## ✅ Vérifiez que ça marche

- [ ] `POST /mesures/` avec des valeurs valides → **201** (pas 200)
- [ ] La réponse contient `id` et `recue_le`, que vous n'avez pas envoyés
- [ ] Une valeur négative → **422**, et le serveur tourne toujours
- [ ] Un champ manquant → **422** avec `Field required`
- [ ] `GET /mesures/` renvoie la liste des mesures créées
- [ ] Dans `/docs`, la section **Schemas** (tout en bas) montre `MesureFleur` avec ses bornes

---

## 🔧 Si ça ne marche pas

| Symptôme | Cause | Solution |
|---|---|---|
| `NameError: name 'BaseModel' is not defined` | import oublié | `from pydantic import BaseModel, Field` |
| `NameError: name 'status' is not defined` | import oublié | `from fastapi import FastAPI, Query, status` |
| `NameError: name 'datetime' is not defined` | import oublié | `from datetime import datetime, timezone` |
| Réponse `200` au lieu de `201` | `status_code=` oublié | ajoutez-le dans le décorateur `@app.post(...)` |
| `422` alors que tout semble bon | virgule ou guillemet manquant dans le JSON | recopiez l'exemple de `/docs` |
| `Input should be a valid dictionary` | vous envoyez le JSON dans l'URL | le corps se met dans « Request body », pas dans les paramètres |
| La liste est vide après redémarrage | c'est **normal** | la liste est en mémoire ; la persistance arrive au Lab 8 |

---

## 💡 Ce que vous venez d'apprendre

- Un **schéma Pydantic** est un contrat : il décrit la donnée, et FastAPI le fait respecter.
- **Deux schémas** : un pour ce qui entre, un pour ce qui sort. Ne les confondez jamais.
- `response_model` ne fait pas que documenter : il **filtre** réellement la réponse.
- Le bon code HTTP compte : `201` pour une création, pas `200`.

> ### 🔗 Le pont vers l'après-midi
> Gardez `MesureFleur` sous les yeux. Au **Lab 6**, vous écrirez :
> ```python
> class FeaturesIris(MesureFleur):
>     ...
> ```
> Les 4 mesures d'une fleur **sont** les 4 features du modèle. Le contrat d'API que vous
> venez d'écrire **est** le contrat du modèle.

---

➡️ **Lab suivant : [Lab 3 — Erreurs et codes HTTP](lab-03-erreurs.md)**
