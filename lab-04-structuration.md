# Lab 4 — Ranger le projet : routers, schémas, configuration · 30 min

## 🎯 Ce que vous allez faire

Votre `app/main.py` fait maintenant une centaine de lignes et mélange tout : les schémas,
les routes, la fausse base de données. Ça tient encore… mais cet après-midi vous allez
ajouter un modèle de ML, une base de données et des tests. **Il faut ranger maintenant.**

Vous n'écrivez **aucune fonctionnalité nouvelle** dans ce lab : vous **déplacez** du code.
Après ce lab, votre API fera exactement la même chose qu'avant. C'est normal, et c'est le
but : ça s'appelle un **refactoring**.

**Point de départ** : Lab 3 terminé.

---

## Ce qu'on vise

```
app/
├── __init__.py
├── main.py           ← l'assemblage : 20 lignes maximum
├── config.py         ← la configuration
├── schemas/
│   ├── __init__.py
│   └── fleur.py      ← les moules Pydantic
└── routers/
    ├── __init__.py
    ├── demo.py       ← /echo, /fleurs
    └── mesures.py    ← /mesures
```

**La règle** : un fichier = une responsabilité.
- `schemas/` → *à quoi ressemblent les données*
- `routers/` → *quelles adresses existent et que font-elles*
- `config.py` → *les réglages* (chemins, noms, options)
- `main.py` → *l'assemblage*, rien d'autre

---

## Étape 1 — Créer les dossiers

Dans un second terminal (celui où le serveur ne tourne pas) :

```bash
mkdir -p app/schemas app/routers
touch app/schemas/__init__.py app/routers/__init__.py
```

> Encore une fois : les `__init__.py` sont **vides mais obligatoires**. Sans eux,
> `from app.schemas.fleur import ...` échouera.

---

## Étape 2 — Sortir les schémas

Créez **`app/schemas/fleur.py`** et déplacez-y vos deux classes :

```python
from datetime import datetime

from pydantic import BaseModel, Field


class MesureFleur(BaseModel):
    """Ce que le CLIENT envoie."""

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


class MesureEnregistree(MesureFleur):
    """Ce que le SERVEUR renvoie."""

    id: int
    recue_le: datetime
```

---

## Étape 3 — Sortir les routes de démo

Créez **`app/routers/demo.py`** :

```python
from fastapi import APIRouter, Query

router = APIRouter(tags=["Démo"])


@router.get("/echo/{message}")
def echo(message: str):
    return {"message_recu": message}


# ⚠️ Route fixe AVANT route dynamique (rappel du Lab 1)
@router.get("/fleurs/")
def lister_fleurs(
    offset: int = Query(default=0, ge=0, description="Nombre d'éléments à sauter"),
    limit: int = Query(default=10, ge=1, le=50, description="Taille de page (max 50)"),
):
    return {"offset": offset, "limit": limit}


@router.get("/fleurs/{fleur_id}")
def lire_fleur(fleur_id: int):
    return {"fleur_id": fleur_id, "type_python": type(fleur_id).__name__}
```

> ### 🔑 Un `APIRouter` est un « mini-`app` »
> Vous remplacez simplement `@app.get(...)` par `@router.get(...)`. Tout le reste est
> identique. Le `router` sera ensuite **branché** sur l'application dans `main.py`.

---

## Étape 4 — Sortir les mesures

Créez **`app/routers/mesures.py`** :

```python
from datetime import datetime, timezone

from fastapi import APIRouter, HTTPException, Response, status

from app.schemas.fleur import MesureEnregistree, MesureFleur

router = APIRouter(prefix="/mesures", tags=["Mesures"])

mesures: list[MesureEnregistree] = []


@router.post("/", response_model=MesureEnregistree, status_code=status.HTTP_201_CREATED)
def creer_mesure(mesure: MesureFleur):
    enregistree = MesureEnregistree(
        id=len(mesures) + 1,
        recue_le=datetime.now(timezone.utc),
        **mesure.model_dump(),
    )
    mesures.append(enregistree)
    return enregistree


@router.get("/", response_model=list[MesureEnregistree])
def lister_mesures():
    return mesures


@router.get("/{mesure_id}", response_model=MesureEnregistree)
def lire_mesure(mesure_id: int):
    for mesure in mesures:
        if mesure.id == mesure_id:
            return mesure
    raise HTTPException(status_code=404, detail=f"Mesure {mesure_id} introuvable")


@router.delete("/{mesure_id}", status_code=status.HTTP_204_NO_CONTENT)
def supprimer_mesure(mesure_id: int):
    for index, mesure in enumerate(mesures):
        if mesure.id == mesure_id:
            mesures.pop(index)
            return Response(status_code=status.HTTP_204_NO_CONTENT)
    raise HTTPException(status_code=404, detail=f"Mesure {mesure_id} introuvable")
```

> ### 🔑 `prefix="/mesures"`
> Grâce au préfixe, les chemins dans le fichier deviennent **relatifs** : `"/"` au lieu de
> `"/mesures/"`, `"/{mesure_id}"` au lieu de `"/mesures/{mesure_id}"`.
> Si demain vous devez tout déplacer sous `/v2/mesures`, **une seule ligne à changer**.

---

## Étape 5 — La configuration

```bash
uv add pydantic-settings
```

Créez **`app/config.py`** :

```python
from functools import lru_cache
from pathlib import Path

from pydantic_settings import BaseSettings

RACINE = Path(__file__).resolve().parent.parent


class Parametres(BaseSettings):
    nom_service: str = "iris-api"
    version: str = "0.1.0"
    chemin_modele: Path = RACINE / "artefacts" / "modele_iris.joblib"

    model_config = {"env_file": ".env", "env_prefix": "IRIS_"}


@lru_cache
def get_parametres() -> Parametres:
    return Parametres()
```

**Ce que ça fait** : `Parametres` est un schéma Pydantic, comme `MesureFleur` — mais au lieu
de valider une requête HTTP, il valide **la configuration**. Chaque champ peut être écrasé
par une **variable d'environnement** préfixée par `IRIS_` :

| Champ Python | Variable d'environnement |
|---|---|
| `nom_service` | `IRIS_NOM_SERVICE` |
| `chemin_modele` | `IRIS_CHEMIN_MODELE` |

`@lru_cache` fait que `get_parametres()` ne construit l'objet **qu'une seule fois**, même
appelée mille fois.

> `chemin_modele` pointe vers un fichier qui n'existe pas encore : vous le créerez au Lab 5.
> C'est juste un chemin, ça ne pose aucun problème pour l'instant.

---

## Étape 6 — Le nouveau `main.py`

**Remplacez tout le contenu** de `app/main.py` par ceci :

```python
from fastapi import Depends, FastAPI

from app.config import Parametres, get_parametres
from app.routers import demo, mesures

app = FastAPI(
    title="Iris API",
    description="Service de prédiction — formation Industrialisation IA",
    version="0.1.0",
)

app.include_router(demo.router)
app.include_router(mesures.router)


@app.get("/", tags=["Système"])
def racine(parametres: Parametres = Depends(get_parametres)):
    return {"service": parametres.nom_service, "version": parametres.version}
```

**20 lignes.** `main.py` ne fait plus que de l'assemblage — et c'est très bien.

---

## Étape 7 — Comprendre `Depends`, parce que tout repose dessus

Regardez la signature :

```python
def racine(parametres: Parametres = Depends(get_parametres)):
```

Ça se lit : *« pour répondre à cette requête, j'ai besoin d'un objet `Parametres`.
Pour l'obtenir, appelle `get_parametres()`. »*

FastAPI appelle la fonction à votre place et **injecte** le résultat. C'est tout.
Il n'y a aucune magie.

> ### 🔑 `Depends` est la clé de toute la journée. Retenez ces deux usages :
> - **Lab 6** : `Depends(get_moteur)` injectera le **modèle de ML** dans `/predict`.
> - **Lab 9** : une seule ligne — `app.dependency_overrides[get_moteur] = faux_modele` —
>   remplacera le vrai modèle par un faux **dans les tests**, sans toucher au code de l'API.
>
> C'est exactement ce qui rend une API **testable**. Si vous importiez le modèle directement
> en haut du fichier, vous ne pourriez pas le remplacer.

---

## ✅ Vérifiez que ça marche

Relancez le serveur (`Ctrl+C` puis `uv run fastapi dev app/main.py`), puis :

- [ ] Le serveur démarre **sans erreur d'import**
- [ ] `/docs` affiche **trois groupes** : `Système`, `Démo`, `Mesures`
- [ ] Toutes les routes des labs 1 à 3 fonctionnent encore à l'identique
- [ ] `GET /` renvoie `{"service":"iris-api","version":"0.1.0"}`
- [ ] **Le test de la configuration** — arrêtez le serveur et relancez-le ainsi :

```bash
IRIS_NOM_SERVICE="iris-api-prod" uv run fastapi dev app/main.py
```

`GET /` renvoie maintenant `{"service":"iris-api-prod", ...}` — **sans que vous ayez touché
une seule ligne de code**. C'est exactement comme ça qu'on configurera le conteneur Docker
au Lab 10.

*(Sous Windows PowerShell : `$env:IRIS_NOM_SERVICE="iris-api-prod"; uv run fastapi dev app/main.py`)*

---

## 🔧 Si ça ne marche pas

| Message d'erreur | Cause | Solution |
|---|---|---|
| `ModuleNotFoundError: No module named 'app'` | projet créé sans `--no-package`, ou mauvais dossier | vous devez être **à la racine** de `iris-api` ; voir l'encadré du Lab 0 |
| `ModuleNotFoundError: No module named 'app.schemas'` | `__init__.py` manquant | `touch app/schemas/__init__.py app/routers/__init__.py` |
| `ImportError: cannot import name 'MesureFleur'` | classe restée dans `main.py` | elle doit être **uniquement** dans `app/schemas/fleur.py` |
| `ModuleNotFoundError: No module named 'pydantic_settings'` | paquet non installé | `uv add pydantic-settings` (avec un tiret) |
| Les routes ont disparu de `/docs` | `include_router` oublié | vérifiez les deux lignes `app.include_router(...)` |
| `/mesures/mesures/` dans `/docs` | préfixe **et** chemin complet | avec `prefix="/mesures"`, les chemins deviennent `"/"` et `"/{mesure_id}"` |

---

## 💡 Ce que vous venez d'apprendre

- **Un fichier = une responsabilité.** C'est ce qui fait qu'un projet reste lisible à 10
  fichiers comme à 100.
- Un **`APIRouter`** est un groupe de routes autonome, branché sur l'app par `include_router`.
- La **configuration vient de l'environnement**, jamais du code en dur.
- **`Depends`** injecte des ressources dans vos endpoints — et permet de les remplacer.

---

## 🎉 Fin de la première partie

Vous avez une API propre, validée, documentée et bien rangée.

**Maintenant on passe au vrai sujet du cours** : y mettre un modèle de Machine Learning
et le rendre déployable.

---

➡️ **Lab suivant : [Lab 5 — Entraîner et sérialiser le modèle](lab-05-entrainer-le-modele.md)**
