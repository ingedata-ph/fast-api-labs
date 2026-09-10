# Lab 7 — Robustesse : validation métier, batch, `/health` · 40 min

## 🎯 Ce que vous allez faire

Votre `/predict` **marche**. Ce n'est pas la même chose que **tenir debout en production**.

Dans ce lab, vous ajoutez les quatre choses qui séparent un prototype d'un service :

1. une **validation métier** (au-delà des types) ;
2. un endpoint **batch** pour traiter 100 fleurs d'un coup ;
3. **`/health`** — l'endpoint que Docker interrogera ;
4. **`/model/info`** — la carte d'identité du modèle en production.

**Point de départ** : Lab 6 terminé, `/predict` fonctionne.

---

## Étape 1 — La validation métier

Pydantic vérifie déjà les **types** et les **bornes** (Lab 2 : `gt=0`, `lt=30`).
Mais il ne connaît pas votre **métier**.

Chez un iris, le pétale n'est jamais plus long que le sépale. Une requête avec
`sepal_length=2` et `petal_length=6` est **typée correctement** mais **botaniquement
absurde**. Le modèle prédira quelque chose — et ce quelque chose ne voudra rien dire.

Dans **`app/schemas/prediction.py`**, modifiez l'import et la classe `FeaturesIris` :

```python
from pydantic import BaseModel, Field, model_validator

from app.schemas.fleur import MesureFleur


class FeaturesIris(MesureFleur):
    """Entrée de /predict : les contraintes du Lab 2 + une règle métier."""

    @model_validator(mode="after")
    def coherence_petale_sepale(self) -> "FeaturesIris":
        if self.petal_length > self.sepal_length:
            raise ValueError(
                "Mesure incohérente : la longueur du pétale ne peut excéder "
                "celle du sépale chez un iris."
            )
        return self
```

> ### 🔍 `@model_validator(mode="after")`
> - **`Field(gt=0)`** valide **un champ isolé**.
> - **`@model_validator`** valide **la cohérence entre plusieurs champs** — ce qui est
>   impossible champ par champ.
> - `mode="after"` = « exécute-toi **après** que tous les champs ont été convertis et
>   validés individuellement ». Vous avez donc des `float` propres dans `self`.
>
> Vous levez un `ValueError` Python ordinaire ; **Pydantic le transforme en réponse `422`**.
> Vous n'avez pas à gérer le HTTP ici.

**Testez** dans `/docs` avec :

```json
{"sepal_length": 2.0, "sepal_width": 3.0, "petal_length": 6.0, "petal_width": 1.0}
```

→ **422**, avec votre message métier dans le champ `detail`.

> ### 💡 Le message pédagogique
> Votre modèle a été entraîné sur de vraies fleurs. En production, il recevra des données
> venant de formulaires, de capteurs, de fichiers Excel mal remplis. **Une donnée
> syntaxiquement valide peut être métier-invalide** — et c'est à vous de le savoir, pas au
> modèle.
>
> Chaque règle métier que vous encodez ici est une **prédiction absurde en moins** en production.

---

## Étape 2 — Le mode batch

Un data scientist qui veut scorer 10 000 lignes ne va pas faire 10 000 requêtes HTTP.
Il lui faut un endpoint qui accepte une **liste**.

**Ajoutez** ces deux schémas à la fin de `app/schemas/prediction.py` :

```python
class RequeteBatch(BaseModel):
    fleurs: list[FeaturesIris] = Field(
        ..., min_length=1, max_length=100,
        description="Entre 1 et 100 fleurs à classer",
    )


class ReponseBatch(BaseModel):
    nombre_traite: int
    predictions: list[ReponsePrediction]
```

Puis, dans **`app/routers/prediction.py`**, complétez l'import et ajoutez la route :

```python
from app.schemas.prediction import (
    FeaturesIris,
    ReponseBatch,
    ReponsePrediction,
    RequeteBatch,
)


@router.post("/batch", response_model=ReponseBatch)
def predire_lot(requete: RequeteBatch, moteur: MoteurIris = Depends(get_moteur)):
    predictions = [moteur.predire(fleur.model_dump()) for fleur in requete.fleurs]
    return {"nombre_traite": len(predictions), "predictions": predictions}
```

**Testez** dans `/docs` avec :

```json
{"fleurs": [
  {"sepal_length": 5.1, "sepal_width": 3.5, "petal_length": 1.4, "petal_width": 0.2},
  {"sepal_length": 6.3, "sepal_width": 3.3, "petal_length": 6.0, "petal_width": 2.5},
  {"sepal_length": 5.9, "sepal_width": 3.0, "petal_length": 4.2, "petal_width": 1.5}
]}
```

→ `nombre_traite: 3` et trois prédictions différentes.

Puis testez `{"fleurs": []}` → **422**. Et une liste de 101 fleurs → **422**.

> ### ⚠️ `max_length=100` n'est pas une limite arbitraire, c'est une protection
> Sans elle, un client peut vous envoyer 10 millions de lignes dans une seule requête. Votre
> serveur essaiera de tout charger en mémoire et **tombera**. Ce n'est même pas forcément
> une attaque : c'est souvent un stagiaire avec une boucle mal écrite.
>
> **Toute liste acceptée par une API publique doit avoir une borne supérieure.**

---

## Étape 3 — Gérer l'échec d'inférence

Que se passe-t-il si `predict_proba` lève une exception ? Aujourd'hui : un **500** nu, avec
une trace dans les logs et un message opaque pour le client.

Modifiez la route `/predict/` dans **`app/routers/prediction.py`** :

```python
import logging

from fastapi import APIRouter, Depends, HTTPException, status

logger = logging.getLogger(__name__)


@router.post("/", response_model=ReponsePrediction)
def predire(features: FeaturesIris, moteur: MoteurIris = Depends(get_moteur)):
    try:
        return moteur.predire(features.model_dump())
    except Exception:
        # La trace complète part dans les logs SERVEUR...
        logger.exception("Échec de l'inférence")
        # ...et le client reçoit un message neutre et actionnable.
        raise HTTPException(
            status_code=status.HTTP_503_SERVICE_UNAVAILABLE,
            detail="Le modèle est momentanément indisponible.",
        )
```

> ### 🔍 Pourquoi `503` et pas `500` ?
> | Code | Ce que le client comprend |
> |---|---|
> | `500 Internal Server Error` | « il y a un bug, ça ne sert à rien de réessayer » |
> | `503 Service Unavailable` | « c'est temporaire, réessaie dans un instant » |
>
> Un client bien écrit **réessaie** sur un `503`, **abandonne** sur un `500`.
> Le code que vous choisissez pilote le comportement de tout l'écosystème autour de vous.
>
> Et `logger.exception(...)` écrit la trace complète **côté serveur** — vous gardez
> l'information pour déboguer, sans l'exposer.

---

## Étape 4 — `/health` et `/model/info`

Créez **`app/routers/systeme.py`** :

```python
from fastapi import APIRouter, Depends, HTTPException, status

from app.dependances import get_moteur, get_moteur_optionnel
from app.ml.moteur import MoteurIris

router = APIRouter(tags=["Système"])


@router.get("/health")
def sante(moteur: MoteurIris | None = Depends(get_moteur_optionnel)):
    """Le service est-il VIVANT et PRÊT ?

    Vivant : cette fonction répond.
    Prêt   : le modèle est chargé. Sinon → 503.
    """
    if moteur is None:
        raise HTTPException(
            status_code=status.HTTP_503_SERVICE_UNAVAILABLE,
            detail="Modèle non chargé",
        )
    return {"statut": "ok", "modele_charge": True, "version_modele": moteur.version}


@router.get("/model/info")
def info_modele(moteur: MoteurIris = Depends(get_moteur)):
    """La carte d'identité du modèle actuellement servi."""
    return {
        "version": moteur.version,
        "accuracy_test": moteur.accuracy,
        "features_attendues": moteur.features,
        "classes": moteur.classes,
        "entraine_le": moteur.entraine_le,
        "version_sklearn": moteur.version_sklearn,
    }
```

Branchez le router dans **`app/main.py`** :

```python
from app.routers import demo, mesures, prediction, systeme
...
app.include_router(systeme.router)
```

> ### 🔍 Pourquoi `/health` utilise `get_moteur_optionnel`
> `/predict` a besoin du modèle : sans lui, il **refuse** (`get_moteur` → 503).
> `/health`, lui, doit pouvoir **constater** l'absence pour la signaler. D'où les deux
> dépendances du Lab 6.

> ### 🔍 À quoi sert vraiment `/health` ?
> Un orchestrateur (Docker, Kubernetes) l'appelle toutes les 10 à 30 secondes. Selon la
> réponse, il **retire votre conteneur du trafic** ou le **redémarre**, automatiquement,
> à 3 h du matin, sans réveiller personne.
>
> **Vous utiliserez littéralement cet endpoint au Lab 10** dans le `healthcheck` du
> `docker-compose.yml`. Ce n'est pas un exercice décoratif.

> ### 🔍 Et `/model/info` ?
> C'est la réponse à la question que vous vous poserez un jour à 3 h du matin :
> *« Quelle version tourne réellement en production, entraînée quand, sur quelles
> features ? »* Sans cet endpoint, il faut se connecter au serveur et inspecter un fichier
> binaire.

---

## 📄 Votre `app/schemas/prediction.py` complet à ce stade

Ce fichier a été modifié deux fois dans ce lab. Vérifiez qu'il ressemble exactement à ceci :

```python
from pydantic import BaseModel, Field, model_validator

from app.schemas.fleur import MesureFleur


class FeaturesIris(MesureFleur):
    """Entrée de /predict : les contraintes du Lab 2 + une règle métier."""

    @model_validator(mode="after")
    def coherence_petale_sepale(self) -> "FeaturesIris":
        if self.petal_length > self.sepal_length:
            raise ValueError(
                "Mesure incohérente : la longueur du pétale ne peut excéder "
                "celle du sépale chez un iris."
            )
        return self


class ReponsePrediction(BaseModel):
    """Sortie de /predict — ce que le service GARANTIT de renvoyer."""

    classe: str = Field(..., description="Espèce prédite")
    indice_classe: int
    confiance: float = Field(..., ge=0, le=1, description="Probabilité de la classe prédite")
    probabilites: dict[str, float] = Field(..., description="Distribution complète")
    version_modele: str


class RequeteBatch(BaseModel):
    fleurs: list[FeaturesIris] = Field(
        ..., min_length=1, max_length=100,
        description="Entre 1 et 100 fleurs à classer",
    )


class ReponseBatch(BaseModel):
    nombre_traite: int
    predictions: list[ReponsePrediction]
```

---

## ✅ Vérifiez que ça marche

- [ ] `{"sepal_length":2.0,"sepal_width":3.0,"petal_length":6.0,"petal_width":1.0}` → **422**
      avec votre message métier
- [ ] `POST /predict/batch` avec 3 fleurs → `nombre_traite: 3`
- [ ] `{"fleurs": []}` → **422** · liste de 101 fleurs → **422**
- [ ] `GET /health` → **200**, `{"statut":"ok","modele_charge":true,...}`
- [ ] `GET /model/info` → **200** avec version, accuracy, features, classes
- [ ] **Le test du mode dégradé** :
  ```bash
  # serveur arrêté
  mv artefacts artefacts.bak
  uv run fastapi dev app/main.py
  ```
  → `GET /health` renvoie **503** · `POST /predict/` renvoie **503**
  → puis `mv artefacts.bak artefacts` et relancez

---

## 🔧 Si ça ne marche pas

| Message d'erreur | Cause | Solution |
|---|---|---|
| `ImportError: cannot import name 'model_validator'` | Pydantic v1 | ce cours utilise Pydantic v2 : `uv add "pydantic>=2"` |
| `ValidationError` au démarrage | `return self` oublié dans le validateur | un `model_validator(mode="after")` **doit** renvoyer `self` |
| Le validateur ne se déclenche jamais | ajouté à `MesureFleur` au lieu de `FeaturesIris` | vérifiez la classe |
| `422` sur un batch qui semble correct | une seule fleur invalide suffit | le champ `loc` de l'erreur indique **l'index** fautif |
| `500` au lieu de `503` | le `try/except` n'entoure pas l'appel | vérifiez l'indentation |
| `/health` renvoie 404 | router non branché | `app.include_router(systeme.router)` |
| `/health` renvoie 503 alors que le modèle existe | mauvaise dépendance | `/health` utilise `get_moteur_optionnel` |

---

## 💡 Ce que vous venez d'apprendre

- **Types validés ≠ donnée sensée.** Les règles métier s'encodent avec `@model_validator`.
- **Un endpoint batch** évite 10 000 allers-retours HTTP — et **doit être borné**.
- **`4XX` = le client a tort · `5XX` = nous avons un problème · `503` = réessaie plus tard.**
- **`/health` est l'interface entre votre service et son orchestrateur.** Sans lui, Docker
  ne sait pas si votre conteneur va bien.

> ### 🤔 Une vraie question d'architecture, sans bonne réponse unique
> Vaut-il mieux qu'un service **refuse de démarrer** sans modèle, ou qu'il **démarre en
> mode dégradé** et renvoie `503` ?
>
> - *Refuser de démarrer* : l'erreur est immédiate et visible au déploiement.
> - *Démarrer dégradé* (notre choix) : le service peut répondre à `/health`, l'orchestrateur
>   le voit, et le modèle peut être monté après coup sans redémarrage.
>
> Les deux se défendent. **Ce qui compte, c'est de choisir consciemment.**

---

➡️ **Lab suivant : [Lab 8 — Traçabilité](lab-08-journalisation.md)**
