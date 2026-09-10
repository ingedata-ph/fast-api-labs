# Lab 8 — Traçabilité : journaliser les prédictions · 30 min

## 🎯 Ce que vous allez faire

Enregistrer **chaque prédiction** dans une base de données SQLite : les mesures reçues, la
classe prédite, la confiance, la version du modèle, l'horodatage et la **durée d'inférence**.

Vous réutilisez exactement le SQLModel vu dans `tuto.md` — mais cette fois pour un vrai
besoin d'industrialisation.

**Point de départ** : Lab 7 terminé.

---

## Pourquoi journaliser ? Trois raisons très concrètes

**1. Auditabilité.** Un client vous appelle : *« le 14 mars, votre API a classé ma fleur en
`virginica`, je conteste. »* Sans journal, vous ne pouvez ni confirmer ni infirmer.

**2. Détection de dérive** (*drift*). Un modèle ne « tombe » pas en panne : il se **dégrade
silencieusement** quand les données d'entrée changent de distribution. Si vos utilisateurs
se mettent à envoyer des fleurs deux fois plus grandes qu'à l'entraînement, **aucune erreur
ne sera levée** — les prédictions deviendront simplement mauvaises. Seul le journal permet
de le voir.

**3. Jeu de ré-entraînement.** Les données réelles reçues en production sont exactement ce
dont vous aurez besoin pour entraîner la v2 du modèle.

---

## Étape 1 — Installer SQLModel

```bash
uv add sqlmodel
```

> **SQLModel** combine Pydantic (validation) et SQLAlchemy (base de données). Une même classe
> décrit à la fois **la table SQL** et **le schéma de données**. C'est l'outil que vous avez
> déjà croisé dans `tuto.md`.

---

## Étape 2 — Ajouter le chemin de la base à la configuration

Dans **`app/config.py`**, ajoutez **une ligne** dans la classe `Parametres` :

```python
class Parametres(BaseSettings):
    nom_service: str = "iris-api"
    version: str = "0.1.0"
    chemin_modele: Path = RACINE / "artefacts" / "modele_iris.joblib"
    chemin_bdd: Path = RACINE / "journal.db"          # ← ajoutez cette ligne

    model_config = {"env_file": ".env", "env_prefix": "IRIS_"}
```

> **Pourquoi passer par la configuration plutôt que d'écrire le chemin en dur ?**
> Parce qu'au Lab 10, le conteneur Docker devra écrire la base dans un **volume** pour
> qu'elle survive à la destruction du conteneur. Il suffira de définir la variable
> d'environnement `IRIS_CHEMIN_BDD`. **Aucune ligne de code à changer.**

---

## Étape 3 — La connexion à la base

Créez **`app/database.py`** :

```python
from sqlmodel import Session, create_engine

from app.config import get_parametres

parametres = get_parametres()

# On s'assure que le dossier parent existe (utile dans Docker au Lab 10).
parametres.chemin_bdd.parent.mkdir(parents=True, exist_ok=True)

moteur_bdd = create_engine(
    f"sqlite:///{parametres.chemin_bdd}",
    # Nécessaire avec SQLite + FastAPI : plusieurs threads accèdent à la même connexion.
    connect_args={"check_same_thread": False},
)


def get_session():
    """Dépendance : ouvre une session, la donne à l'endpoint, puis la referme."""
    with Session(moteur_bdd) as session:
        yield session
```

> ### 🔍 Une dépendance avec `yield`
> Vous avez déjà vu `yield` dans le `lifespan` (Lab 6) — même principe, autre échelle :
> - **avant le `yield`** → on ouvre la session ;
> - **le `yield`** → l'endpoint travaille avec ;
> - **après le `yield`** → le `with` referme la session, **même si l'endpoint a levé une
>   exception**.
>
> Le `lifespan` gère le cycle de vie de **l'application** ; cette dépendance gère celui de
> **la requête**.

---

## Étape 4 — La table

```bash
mkdir -p app/models && touch app/models/__init__.py
```

Créez **`app/models/journal.py`** :

```python
from datetime import datetime, timezone

from sqlmodel import Field, SQLModel


class JournalPrediction(SQLModel, table=True):
    """Une ligne = une prédiction servie."""

    id: int | None = Field(default=None, primary_key=True)
    horodatage: datetime = Field(
        default_factory=lambda: datetime.now(timezone.utc), index=True
    )
    # Ce que le client a envoyé
    sepal_length: float
    sepal_width: float
    petal_length: float
    petal_width: float
    # Ce que le modèle a répondu
    classe_predite: str = Field(index=True)
    confiance: float
    version_modele: str = Field(index=True)
    duree_ms: float
```

> ### 🔍 `table=True`, et pourquoi ces trois `index=True`
> - **`table=True`** transforme la classe en **vraie table SQL**. Sans lui, ce serait un
>   simple schéma Pydantic (comme `MesureFleur`).
> - **`id: int | None = Field(default=None, primary_key=True)`** : l'`id` est `None` avant
>   l'insertion, et rempli automatiquement par la base après.
> - Les **index** sur `horodatage`, `classe_predite` et `version_modele` accélèrent
>   exactement les trois questions qu'on posera : *« que s'est-il passé entre telle et
>   telle date ? »*, *« combien de virginica ? »*, *« la v2 fait-elle mieux que la v1 ? »*

---

## Étape 5 — Créer les tables au démarrage

Dans **`app/main.py`**, ajoutez les imports et **une ligne** dans le `lifespan` :

```python
from sqlmodel import SQLModel

import app.models.journal  # noqa: F401 — enregistre la table auprès de SQLModel
from app.database import moteur_bdd
```

Puis, dans la fonction `lifespan`, **avant** le chargement du modèle :

```python
@asynccontextmanager
async def lifespan(app: FastAPI):
    parametres = get_parametres()

    SQLModel.metadata.create_all(moteur_bdd)   # ← crée les tables si absentes
    print("Base de données prête.")

    print(f"Chargement du modèle depuis {parametres.chemin_modele} ...")
    ...
```

> ### ⚠️ L'import `app.models.journal` a l'air inutile. Il ne l'est pas.
> `SQLModel.metadata.create_all()` ne crée que les tables **dont SQLModel a connaissance**.
> Une classe n'est enregistrée qu'au moment où son fichier est **importé**. Sans cet import,
> `create_all()` ne crée… rien, et vous aurez `no such table: journalprediction`
> à la première prédiction.
>
> Le commentaire `# noqa: F401` dit aux outils d'analyse « oui, cet import est volontairement
> "inutilisé", laisse-le tranquille ».

---

## Étape 6 — Journaliser dans `/predict`

Modifiez **`app/routers/prediction.py`**. Voici le fichier **complet** à ce stade :

```python
import logging
import time

from fastapi import APIRouter, Depends, HTTPException, status
from sqlmodel import Session

from app.database import get_session
from app.dependances import get_moteur
from app.ml.moteur import MoteurIris
from app.models.journal import JournalPrediction
from app.schemas.prediction import (
    FeaturesIris,
    ReponseBatch,
    ReponsePrediction,
    RequeteBatch,
)

logger = logging.getLogger(__name__)

router = APIRouter(prefix="/predict", tags=["Prédiction"])


@router.post("/", response_model=ReponsePrediction)
def predire(
    features: FeaturesIris,
    moteur: MoteurIris = Depends(get_moteur),
    session: Session = Depends(get_session),
):
    debut = time.perf_counter()
    try:
        resultat = moteur.predire(features.model_dump())
    except Exception:
        logger.exception("Échec de l'inférence")
        raise HTTPException(
            status_code=status.HTTP_503_SERVICE_UNAVAILABLE,
            detail="Le modèle est momentanément indisponible.",
        )
    duree_ms = (time.perf_counter() - debut) * 1000

    session.add(
        JournalPrediction(
            **features.model_dump(),
            classe_predite=resultat["classe"],
            confiance=resultat["confiance"],
            version_modele=resultat["version_modele"],
            duree_ms=round(duree_ms, 3),
        )
    )
    session.commit()

    return resultat


@router.post("/batch", response_model=ReponseBatch)
def predire_lot(requete: RequeteBatch, moteur: MoteurIris = Depends(get_moteur)):
    predictions = [moteur.predire(fleur.model_dump()) for fleur in requete.fleurs]
    return {"nombre_traite": len(predictions), "predictions": predictions}
```

> **`time.perf_counter()`** est l'horloge la plus précise de Python pour mesurer une durée.
> On l'utilise **toujours** par différence entre deux appels, jamais comme une date.

---

## Étape 7 — Relire le journal

Créez **`app/routers/journal.py`** :

```python
from fastapi import APIRouter, Depends, Query
from sqlmodel import Session, select

from app.database import get_session
from app.models.journal import JournalPrediction

router = APIRouter(prefix="/journal", tags=["Journal"])


@router.get("/", response_model=list[JournalPrediction])
def lire_journal(
    offset: int = Query(default=0, ge=0),
    limit: int = Query(default=20, ge=1, le=100),
    session: Session = Depends(get_session),
):
    requete = (
        select(JournalPrediction)
        .order_by(JournalPrediction.id.desc())   # les plus récentes d'abord
        .offset(offset)
        .limit(limit)
    )
    return session.exec(requete).all()
```

Les paramètres `offset` et `limit` sont **exactement ceux du Lab 1** — la pagination
n'était pas un exercice gratuit.

Branchez le router dans **`app/main.py`** :

```python
from app.routers import demo, journal, mesures, prediction, systeme
...
app.include_router(journal.router)
```

---

## 📄 Votre `app/main.py` complet à ce stade

`main.py` a été modifié aux labs 6, 7 et 8. **C'est le fichier où l'on se trompe le plus.**
Comparez le vôtre avec celui-ci, ligne par ligne :

```python
from contextlib import asynccontextmanager

from fastapi import Depends, FastAPI
from sqlmodel import SQLModel

import app.models.journal  # noqa: F401 — enregistre la table auprès de SQLModel
from app.config import Parametres, get_parametres
from app.database import moteur_bdd
from app.ml.moteur import MoteurIris
from app.routers import demo, journal, mesures, prediction, systeme

# Le "sac" qui contient les ressources partagées, rempli au démarrage.
etat_ml: dict[str, MoteurIris] = {}


@asynccontextmanager
async def lifespan(app: FastAPI):
    # ─────────── DÉMARRAGE : exécuté UNE fois, avant la 1re requête ───────────
    parametres = get_parametres()

    SQLModel.metadata.create_all(moteur_bdd)
    print("Base de données prête.")

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
    lifespan=lifespan,
)

app.include_router(systeme.router)
app.include_router(demo.router)
app.include_router(mesures.router)
app.include_router(prediction.router)
app.include_router(journal.router)


@app.get("/", tags=["Système"])
def racine(parametres: Parametres = Depends(get_parametres)):
    return {"service": parametres.nom_service, "documentation": "/docs"}
```

---

## ✅ Vérifiez que ça marche

Relancez le serveur. Au démarrage : `Base de données prête.` puis `Modèle v1.0.0 chargé.`

- [ ] Un fichier `journal.db` est apparu à la racine du projet
- [ ] Faites **3 appels** à `POST /predict/` (avec des fleurs différentes)
- [ ] `GET /journal/` renvoie **3 lignes**, la plus récente en premier
- [ ] Chaque ligne contient `duree_ms`, `version_modele` et `horodatage`
- [ ] En ligne de commande :

```bash
uv run python -c "
from sqlmodel import Session, select
from app.database import moteur_bdd
from app.models.journal import JournalPrediction
with Session(moteur_bdd) as s:
    for j in s.exec(select(JournalPrediction)).all():
        print(j.id, j.classe_predite, j.confiance, f'{j.duree_ms} ms')
"
```

> ### 🔍 Regardez la colonne `duree_ms`
> Vous devriez lire des valeurs entre **0,1 et 3 ms**.
>
> **Vous venez de mesurer la latence de votre modèle en production.**
> Retenez l'ordre de grandeur : l'inférence est *rapide*. Ce qui est lent dans un service
> ML, c'est presque toujours ce qu'il y a **autour** — le réseau, la base, la sérialisation.

---

## 🔧 Si ça ne marche pas

| Message d'erreur | Cause | Solution |
|---|---|---|
| `no such table: journalprediction` | import du modèle oublié | ajoutez `import app.models.journal` dans `main.py` |
| `NameError: name 'SQLModel' is not defined` | import oublié | `from sqlmodel import SQLModel` dans `main.py` |
| `ModuleNotFoundError: No module named 'app.models'` | `__init__.py` manquant | `touch app/models/__init__.py` |
| `SQLite objects created in a thread…` | option de connexion oubliée | `connect_args={"check_same_thread": False}` |
| `TypeError: ... got multiple values for keyword argument` | doublon de champ | `**features.model_dump()` fournit déjà les 4 mesures : ne les répétez pas |
| `journal.db` reste vide | `session.commit()` oublié | sans `commit()`, rien n'est écrit |
| `duree_ms` toujours à 0 | `debut` pris après l'inférence | `debut = time.perf_counter()` **avant** l'appel |

---

## 💡 Ce que vous venez d'apprendre

- Une prédiction non journalisée est une prédiction que vous ne pourrez **ni expliquer, ni
  auditer, ni réutiliser**.
- Le champ **`version_modele` est indispensable** : c'est lui qui permettra de comparer
  v1 et v2 sur du trafic réel.
- Les dépendances `yield` gèrent proprement l'ouverture/fermeture des ressources.
- Le chemin de la base passe par la **configuration** — c'est ce qui rendra la persistance
  possible dans Docker au Lab 10.

> ### ⚠️ Une limite à connaître (à ne pas corriger dans ce lab)
> Écrire en base **à chaque prédiction, de façon synchrone**, ajoute de la latence : le
> client attend que l'écriture soit terminée. Sur un service à fort trafic, on utiliserait
> les **`BackgroundTasks`** de FastAPI, ou une file de messages (Kafka, RabbitMQ).
>
> Sachez que le problème existe. Pour l'instant, à 1 ms d'écriture, ce n'en est pas un.

---

➡️ **Lab suivant : [Lab 9 — Tests automatisés](lab-09-tests.md)**
