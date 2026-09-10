# Lab 6 ⭐ — Charger le modèle et exposer `/predict` · 45 min

## 🎯 Ce que vous allez faire

**C'est le lab central de la journée.** Votre modèle va cesser d'être un fichier sur votre
disque pour devenir un **service** que n'importe qui peut appeler.

Vous allez :
1. écrire une classe qui charge l'artefact et sait prédire ;
2. la charger **une seule fois au démarrage** du serveur ;
3. l'exposer aux endpoints via une **dépendance** ;
4. créer `POST /predict`.

**Point de départ** : Lab 5 terminé, `artefacts/modele_iris.joblib` existe.

---

## Avant d'écrire : la question qui structure tout le lab

Voici la version « naïve » que tout le monde écrit la première fois :

```python
# ❌ NE FAITES PAS ÇA
@app.post("/predict")
def predire(features: FeaturesIris):
    artefact = joblib.load("artefacts/modele_iris.joblib")   # ← ici
    ...
```

**Question : si votre API reçoit 1 000 requêtes, combien de fois ce fichier est-il lu ?**

<details>
<summary>Réponse</summary>

**1 000 fois.** Chaque requête ouvre le fichier, désérialise le modèle, l'utilise une fois,
et le jette. Pour Iris (2 Ko) ça coûte ~50 ms — déjà 20 fois plus que l'inférence elle-même.
Pour un modèle de 2 Go, c'est plusieurs secondes **par requête**. Le service est inutilisable.

</details>

**La solution** : charger le modèle **une fois au démarrage**, le garder en mémoire, et le
réutiliser pour toutes les requêtes. C'est le rôle du **`lifespan`**.

---

## Étape 1 — Le moteur d'inférence

```bash
mkdir -p app/ml && touch app/ml/__init__.py
```

Créez **`app/ml/moteur.py`** :

```python
from pathlib import Path

import joblib
import numpy as np


class MoteurIris:
    """Encapsule l'artefact de ML.

    Cette classe n'importe NI fastapi NI pydantic : elle ne connaît rien au HTTP.
    C'est volontaire — voir l'encadré en fin de lab.
    """

    def __init__(self, artefact: dict):
        self.pipeline = artefact["pipeline"]
        self.features: list[str] = artefact["features"]
        self.classes: list[str] = artefact["classes"]
        self.version: str = artefact["version"]
        self.accuracy: float = artefact["accuracy"]
        self.version_sklearn: str = artefact["version_sklearn"]
        self.entraine_le: str = artefact["entraine_le"]

    @classmethod
    def depuis_fichier(cls, chemin: Path) -> "MoteurIris":
        if not chemin.exists():
            raise FileNotFoundError(
                f"Artefact introuvable : {chemin}. "
                "Avez-vous lancé `uv run python train.py` ?"
            )
        return cls(joblib.load(chemin))

    def _vecteur(self, donnees: dict) -> np.ndarray:
        # ⚠️ On suit l'ordre du CONTRAT d'entraînement (self.features),
        # jamais l'ordre des clés du JSON reçu. Voir Lab 5, idée n°3.
        return np.array([[donnees[nom] for nom in self.features]])

    def predire(self, donnees: dict) -> dict:
        X = self._vecteur(donnees)
        probabilites = self.pipeline.predict_proba(X)[0]
        indice = int(np.argmax(probabilites))
        return {
            "classe": self.classes[indice],
            "indice_classe": indice,
            "confiance": round(float(probabilites[indice]), 4),
            "probabilites": {
                nom: round(float(p), 4)
                for nom, p in zip(self.classes, probabilites)
            },
            "version_modele": self.version,
        }
```

> ### 🔍 Pourquoi tous ces `int()` et `float()` ?
> scikit-learn renvoie des types **numpy** (`numpy.int64`, `numpy.float32`), que le
> convertisseur JSON de Python ne sait **pas** sérialiser : vous auriez une erreur
> `Object of type int64 is not JSON serializable`.
>
> On convertit donc explicitement en types Python natifs. C'est un excellent exemple de la
> frontière entre le monde scientifique et le monde web.

> ### 🔍 `predict_proba` plutôt que `predict`
> `predict()` renvoie juste la classe. `predict_proba()` renvoie **la probabilité de chaque
> classe**. On peut alors donner au client un indice de **confiance** — une information
> capitale : une prédiction à 51 % ne se traite pas comme une prédiction à 99 %.
>
> ⚠️ Tous les modèles n'ont pas `predict_proba` (par ex. `SVC` sans `probability=True`).

---

## Étape 2 — Les schémas de prédiction

Créez **`app/schemas/prediction.py`** :

```python
from pydantic import BaseModel, Field

from app.schemas.fleur import MesureFleur


class FeaturesIris(MesureFleur):
    """Entrée de /predict.

    On hérite de MesureFleur (Lab 2) : les 4 mesures d'une fleur SONT
    les 4 features du modèle. Le contrat d'API est le contrat du modèle.
    """


class ReponsePrediction(BaseModel):
    """Sortie de /predict — ce que le service GARANTIT de renvoyer."""

    classe: str = Field(..., description="Espèce prédite")
    indice_classe: int
    confiance: float = Field(..., ge=0, le=1, description="Probabilité de la classe prédite")
    probabilites: dict[str, float] = Field(..., description="Distribution complète")
    version_modele: str
```

> **Pourquoi renvoyer `version_modele` dans chaque réponse ?**
> Parce que le jour où vous déploierez la v2 du modèle, vous voudrez savoir **quelle version
> a produit quelle prédiction**. Sans ça, impossible de comparer, d'auditer ou de revenir en
> arrière.

---

## Étape 3 — La dépendance d'accès au modèle

Créez **`app/dependances.py`** :

```python
from fastapi import Depends, HTTPException, status

from app.ml.moteur import MoteurIris


def get_moteur_optionnel() -> MoteurIris | None:
    """Renvoie le moteur s'il a été chargé au démarrage, sinon None."""
    # Import à l'INTÉRIEUR de la fonction : voir l'encadré ci-dessous.
    from app.main import etat_ml

    return etat_ml.get("moteur")


def get_moteur(
    moteur: MoteurIris | None = Depends(get_moteur_optionnel),
) -> MoteurIris:
    """Comme get_moteur_optionnel, mais refuse la requête si le modèle manque."""
    if moteur is None:
        raise HTTPException(
            status_code=status.HTTP_503_SERVICE_UNAVAILABLE,
            detail="Modèle non chargé : le service n'est pas prêt.",
        )
    return moteur
```

> ### ⚠️ `from app.main import etat_ml` est DANS la fonction, pas en haut du fichier
> Si vous mettez cet import en haut, vous créez un **import circulaire** :
> `main` importe `routers/prediction` qui importe `dependances` qui importe `main`…
> et Python lève `ImportError: cannot import name ... (most likely due to a circular import)`.
>
> En le plaçant dans la fonction, l'import n'a lieu qu'**au moment de l'appel**, quand tout
> est déjà chargé. C'est une astuce que vous reverrez souvent.

> ### 🔑 Pourquoi deux fonctions ?
> - `get_moteur_optionnel` → « donne-moi le modèle, ou `None` ». Utile pour `/health`
>   (Lab 7), qui doit pouvoir **dire** que le modèle est absent.
> - `get_moteur` → « donne-moi le modèle, ou refuse la requête ». Utile pour `/predict`,
>   qui ne peut rien faire sans lui.

---

## Étape 4 — Le router de prédiction

Créez **`app/routers/prediction.py`** :

```python
import logging

from fastapi import APIRouter, Depends

from app.dependances import get_moteur
from app.ml.moteur import MoteurIris
from app.schemas.prediction import FeaturesIris, ReponsePrediction

logger = logging.getLogger(__name__)

router = APIRouter(prefix="/predict", tags=["Prédiction"])


@router.post("/", response_model=ReponsePrediction)
def predire(features: FeaturesIris, moteur: MoteurIris = Depends(get_moteur)):
    return moteur.predire(features.model_dump())
```

**Deux lignes de logique.** Tout le reste — validation, documentation, sérialisation,
gestion de l'absence de modèle — est fait par FastAPI, Pydantic et vos dépendances.

---

## Étape 5 — Le `lifespan` : charger une fois pour toutes

**Remplacez** le contenu de **`app/main.py`** par :

```python
from contextlib import asynccontextmanager

from fastapi import Depends, FastAPI

from app.config import Parametres, get_parametres
from app.ml.moteur import MoteurIris
from app.routers import demo, mesures, prediction

# Le "sac" qui contient les ressources partagées, rempli au démarrage.
etat_ml: dict[str, MoteurIris] = {}


@asynccontextmanager
async def lifespan(app: FastAPI):
    # ─────────── DÉMARRAGE : exécuté UNE fois, avant la 1re requête ───────────
    parametres = get_parametres()
    print(f"Chargement du modèle depuis {parametres.chemin_modele} ...")
    try:
        etat_ml["moteur"] = MoteurIris.depuis_fichier(parametres.chemin_modele)
        print(f"Modèle v{etat_ml['moteur'].version} chargé.")
    except FileNotFoundError as erreur:
        print(f"ATTENTION : {erreur}")
        print("Le service démarre en mode dégradé : /predict renverra 503.")

    yield  # ← l'application sert les requêtes pendant tout ce temps

    # ─────────── ARRÊT : libération des ressources ───────────
    etat_ml.clear()
    print("Modèle déchargé.")


app = FastAPI(
    title="Iris API",
    description="Service de prédiction — formation Industrialisation IA",
    version="0.1.0",
    lifespan=lifespan,          # ← on branche le lifespan ici
)

app.include_router(demo.router)
app.include_router(mesures.router)
app.include_router(prediction.router)


@app.get("/", tags=["Système"])
def racine(parametres: Parametres = Depends(get_parametres)):
    return {"service": parametres.nom_service, "documentation": "/docs"}
```

> ### 🔍 Comment lire un `lifespan`
> C'est une fonction coupée en deux par le mot-clé **`yield`** :
> - **avant le `yield`** → au démarrage du serveur ;
> - **le `yield`** → « maintenant, sers les requêtes » (ça dure des heures) ;
> - **après le `yield`** → à l'arrêt du serveur (`Ctrl+C`).
>
> C'est **l'endroit** où l'on charge un modèle, ouvre un pool de connexions à une base,
> se connecte à un cache… Tout ce qui coûte cher et doit être fait une seule fois.

> ### 🔍 Pourquoi un `try/except` autour du chargement ?
> Sans lui, l'absence du `.joblib` **empêche le serveur de démarrer**. Avec lui, le service
> démarre mais se déclare **non prêt** : `/predict` répondra `503`.
>
> C'est le comportement attendu d'un service moderne : un orchestrateur (Docker,
> Kubernetes) interroge `/health`, voit qu'il n'est pas prêt, et ne lui envoie pas de trafic.
> On mettra ça en place au Lab 7, puis on s'en servira pour de vrai au Lab 10.

---

## Étape 6 — Testez !

Relancez le serveur :

```bash
uv run fastapi dev app/main.py
```

**Regardez d'abord le terminal.** Vous devez voir, **une seule fois** :

```
Chargement du modèle depuis /…/artefacts/modele_iris.joblib ...
Modèle v1.0.0 chargé.
```

Puis allez sur <http://127.0.0.1:8000/docs> → `POST /predict/` → « Try it out ».
Le formulaire est pré-rempli grâce à l'exemple du Lab 2. « Execute ».

```json
{
  "classe": "setosa",
  "indice_classe": 0,
  "confiance": 0.9808,
  "probabilites": {"setosa": 0.9808, "versicolor": 0.0192, "virginica": 0.0},
  "version_modele": "1.0.0"
}
```

**🎉 Votre modèle est un service.** Rechargez la page, refaites 10 appels : le message
« Modèle chargé » n'apparaît **plus jamais**. Le modèle est en mémoire.

**Essayez une autre fleur** — dans le corps de la requête, mettez :

```json
{"sepal_length": 6.3, "sepal_width": 3.3, "petal_length": 6.0, "petal_width": 2.5}
```

→ `virginica`, confiance ≈ 0.99.

**Depuis un terminal** (le second, pas celui du serveur) :

```bash
curl -X POST http://127.0.0.1:8000/predict/ \
  -H "Content-Type: application/json" \
  -d '{"sepal_length":5.1,"sepal_width":3.5,"petal_length":1.4,"petal_width":0.2}'
```

---

## ✅ Vérifiez que ça marche

- [ ] Au démarrage, `Modèle v1.0.0 chargé.` s'affiche **une seule fois**
- [ ] `{5.1, 3.5, 1.4, 0.2}` → `setosa`, confiance > 0.95
- [ ] `{6.3, 3.3, 6.0, 2.5}` → `virginica`
- [ ] Une valeur négative → **422** (la validation du Lab 2 protège le modèle)
- [ ] Un champ manquant → **422**
- [ ] Les 10 requêtes suivantes ne rechargent pas le modèle
- [ ] **Le test du mode dégradé** : arrêtez le serveur, renommez le dossier
      (`mv artefacts artefacts.bak`), relancez.
      → le serveur démarre, affiche `ATTENTION : Artefact introuvable`, et `/predict`
      renvoie **503**. Remettez ensuite : `mv artefacts.bak artefacts`

---

## 🔧 Si ça ne marche pas

| Message d'erreur | Cause | Solution |
|---|---|---|
| `ImportError: ... most likely due to a circular import` | `from app.main import etat_ml` en haut du fichier | déplacez-le **dans** la fonction |
| `FileNotFoundError: Artefact introuvable` | `train.py` pas lancé | `uv run python train.py` |
| `KeyError: 'moteur'` | le modèle n'a pas été chargé | passez par `Depends(get_moteur)`, jamais par `etat_ml["moteur"]` en direct |
| `Object of type int64 is not JSON serializable` | conversions oubliées | `int(...)` et `float(...)` dans `predire()` |
| `KeyError: 'sepal_length'` dans `_vecteur` | noms de features ≠ noms des champs | les deux doivent être identiques à ceux de `train.py` |
| `405 Method Not Allowed` | vous faites un `GET` sur `/predict/` | c'est un `POST` |
| `307 Temporary Redirect` | vous appelez `/predict` sans `/` final | la route est `/predict/`. Avec `curl`, ajoutez `-L` ou le `/` |
| `ModuleNotFoundError: No module named 'app.ml'` | `__init__.py` manquant | `touch app/ml/__init__.py` |
| Le modèle se recharge à chaque requête | `joblib.load()` dans l'endpoint | il doit être **uniquement** dans le `lifespan` |

---

## 💡 Ce que vous venez d'apprendre — les trois idées à retenir

**1. Le `lifespan` charge les ressources coûteuses une seule fois.**
C'est la différence entre une API à 3 ms et une API à 300 ms.

**2. `MoteurIris` n'importe ni `fastapi` ni `pydantic`.**
Regardez ses imports : `joblib`, `numpy`. C'est tout.
La logique ML est **indépendante** de la couche web. Conséquences directes :
- vous pouvez la tester sans lancer de serveur ;
- vous pouvez remplacer Iris par un autre modèle **sans toucher aux routers** ;
- au Lab 9, vous la remplacerez par un faux modèle en **une ligne**.

**3. La validation Pydantic est votre pare-feu.**
Aucune donnée invalide n'atteint `numpy`. Un `422` propre, jamais un `500` incompréhensible.

> ### 🤔 La question à se poser maintenant
> Votre `/predict` marche. Mais :
> - Que se passe-t-il si un client envoie **10 000 fleurs** ?
> - Comment Docker saura-t-il que votre service est **prêt** ?
> - Que renvoyez-vous si l'inférence **plante** en plein vol ?
>
> C'est tout le sujet du Lab 7.

---

➡️ **Lab suivant : [Lab 7 — Robustesse](lab-07-robustesse.md)**
