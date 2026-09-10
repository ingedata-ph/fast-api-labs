# Lab 9 — Tests automatisés · 30 min

## 🎯 Ce que vous allez faire

Écrire des tests qui vérifient votre API **en une fraction de seconde**, **sans lancer de
serveur** et **sans charger le vrai modèle**.

C'est ici que le `Depends` du Lab 6 va révéler sa vraie raison d'être.

**Point de départ** : Lab 8 terminé.

---

## Étape 1 — Installer les outils

```bash
uv add --dev pytest httpx
```

> **`--dev`** range ces paquets dans les dépendances de **développement**. Elles ne partiront
> **pas** dans l'image Docker au Lab 10 — inutile d'embarquer un framework de test en
> production.
>
> - **pytest** : le lanceur de tests standard en Python.
> - **httpx** : le client HTTP utilisé par le `TestClient` de FastAPI.

---

## Étape 2 — Une configuration indispensable

Ajoutez ces lignes **à la fin de `pyproject.toml`** :

```toml
[tool.pytest.ini_options]
pythonpath = ["."]
testpaths = ["tests"]
```

> ### ⚠️ Ne sautez pas cette étape
> Sans `pythonpath = ["."]`, pytest ne trouve pas votre package et vous obtenez :
> ```
> ModuleNotFoundError: No module named 'app'
> ```
> C'est **l'erreur n°1** sur ce lab. Cette ligne dit à pytest : « ajoute la racine du projet
> aux chemins d'import ».

---

## Étape 3 — Le faux modèle

```bash
mkdir -p tests
```

Créez **`tests/conftest.py`** :

```python
import pytest
from fastapi.testclient import TestClient

from app.dependances import get_moteur, get_moteur_optionnel
from app.main import app


class MoteurFactice:
    """Un faux modèle : réponses déterministes, aucun fichier .joblib requis.

    Il expose exactement la même surface que MoteurIris — c'est tout ce qui compte.
    """

    version = "test-0.0.0"
    features = ["sepal_length", "sepal_width", "petal_length", "petal_width"]
    classes = ["setosa", "versicolor", "virginica"]
    accuracy = 1.0

    def predire(self, donnees: dict) -> dict:
        return {
            "classe": "setosa",
            "indice_classe": 0,
            "confiance": 0.99,
            "probabilites": {"setosa": 0.99, "versicolor": 0.005, "virginica": 0.005},
            "version_modele": self.version,
        }


@pytest.fixture
def client():
    """Un client HTTP de test, câblé sur le faux modèle."""
    faux = MoteurFactice()
    app.dependency_overrides[get_moteur] = lambda: faux
    app.dependency_overrides[get_moteur_optionnel] = lambda: faux

    with TestClient(app) as c:      # le "with" déclenche le lifespan
        yield c

    app.dependency_overrides.clear()
```

> ### 🔑 `dependency_overrides` — LA raison d'être du `Depends`
> Ces deux lignes disent à FastAPI :
> *« pour la durée de ce test, quand un endpoint demande `get_moteur`, donne-lui ça à la
> place. »*
>
> **Aucune ligne du code de l'application n'est modifiée.** Vos routers continuent d'écrire
> `Depends(get_moteur)` sans savoir qu'ils reçoivent un faux.
>
> Si vous aviez écrit `from app.main import etat_ml` directement dans vos endpoints, ce
> remplacement serait **impossible**. C'est très exactement pour ça qu'on est passé par une
> dépendance au Lab 6.

> ### ⚠️ Pourquoi surcharger les DEUX dépendances ?
> Une surcharge s'applique à **une fonction précise**. `/predict` utilise `get_moteur`,
> `/health` utilise `get_moteur_optionnel` (Lab 7). Si vous n'en surchargez qu'une,
> `test_health` échouera avec un `503`.

> ### ⚠️ Le `with` devant `TestClient(app)` est obligatoire
> Sans le `with`, le **`lifespan` n'est jamais exécuté** : la base de données n'est pas
> créée et vos tests échouent avec `no such table`. C'est un piège classique.

---

## Étape 4 — Les tests

Créez **`tests/test_prediction.py`** :

```python
FLEUR_VALIDE = {
    "sepal_length": 5.1,
    "sepal_width": 3.5,
    "petal_length": 1.4,
    "petal_width": 0.2,
}


def test_prediction_nominale(client):
    """Le cas qui marche : structure et bornes de la réponse."""
    reponse = client.post("/predict/", json=FLEUR_VALIDE)
    assert reponse.status_code == 200

    corps = reponse.json()
    assert corps["classe"] == "setosa"
    assert 0 <= corps["confiance"] <= 1
    assert set(corps["probabilites"]) == {"setosa", "versicolor", "virginica"}


def test_feature_manquante_rejetee(client):
    """Il manque petal_width → 422, le modèle n'est jamais appelé."""
    incomplete = {k: v for k, v in FLEUR_VALIDE.items() if k != "petal_width"}
    assert client.post("/predict/", json=incomplete).status_code == 422


def test_valeur_negative_rejetee(client):
    """Contrainte gt=0 du Lab 2."""
    invalide = {**FLEUR_VALIDE, "sepal_length": -1}
    assert client.post("/predict/", json=invalide).status_code == 422


def test_incoherence_metier_rejetee(client):
    """Règle métier du Lab 7 : pétale plus long que sépale."""
    incoherente = {**FLEUR_VALIDE, "sepal_length": 2.0, "petal_length": 6.0}
    assert client.post("/predict/", json=incoherente).status_code == 422


def test_batch_respecte_les_bornes(client):
    """1 à 100 fleurs, pas moins, pas plus."""
    assert client.post("/predict/batch", json={"fleurs": [FLEUR_VALIDE] * 3}).status_code == 200
    assert client.post("/predict/batch", json={"fleurs": []}).status_code == 422
    assert client.post("/predict/batch", json={"fleurs": [FLEUR_VALIDE] * 101}).status_code == 422


def test_health(client):
    """Le service se déclare prêt quand un modèle est disponible."""
    assert client.get("/health").status_code == 200
```

> ### 🔍 Comment se lit un test
> - **Le nom de la fonction commence par `test_`** — c'est comme ça que pytest les trouve.
> - **Le paramètre `client`** correspond au nom de la *fixture* définie dans `conftest.py`.
>   pytest l'appelle et injecte le résultat. (Oui, c'est le même principe que `Depends`.)
> - **`assert`** : si la condition est fausse, le test échoue et pytest affiche exactement
>   quelle valeur il a obtenue.

> ### ⚠️ Un détail à connaître : les tests écrivent dans `journal.db`
> `/predict` journalise en base (Lab 8), et vos tests appellent `/predict`. Ils écrivent donc
> dans **la même base que votre serveur de développement**. Sans conséquence aujourd'hui,
> mais dans un vrai projet on donne aux tests leur propre base, via la variable
> `IRIS_CHEMIN_BDD` — c'est exactement à ça que sert la configuration du Lab 4.

---

## Étape 5 — Lancer les tests

```bash
uv run pytest -v
```

```
tests/test_prediction.py::test_prediction_nominale PASSED       [ 16%]
tests/test_prediction.py::test_feature_manquante_rejetee PASSED [ 33%]
tests/test_prediction.py::test_valeur_negative_rejetee PASSED   [ 50%]
tests/test_prediction.py::test_incoherence_metier_rejetee PASSED[ 66%]
tests/test_prediction.py::test_batch_respecte_les_bornes PASSED [ 83%]
tests/test_prediction.py::test_health PASSED                    [100%]

====================== 6 passed in 0.04s ======================
```

**0,04 seconde.** Vous venez de vérifier six comportements de votre API — sans navigateur,
sans clic, sans serveur.

> ### ℹ️ Un avertissement que vous allez voir, et qui est normal
> ```
> StarletteDeprecationWarning: Using `httpx` with `starlette.testclient` is deprecated;
> install `httpx2` instead.
> ```
> **C'est bénin, les tests passent.** Pour le faire disparaître : `uv add --dev httpx2`.

---

## Étape 6 — La démonstration qui compte

Prouvez que vos tests ne dépendent **pas** du vrai modèle :

```bash
mv artefacts artefacts.bak
uv run pytest -q
mv artefacts.bak artefacts
```

**Les 6 tests passent toujours.** Le fichier `.joblib` n'existe même plus.

> ### 💡 Pourquoi c'est capital
> Vos tests tourneront sur une **chaîne d'intégration continue** (GitHub Actions, GitLab CI)
> à chaque commit. Cette machine n'a ni votre modèle, ni vos données, ni vos 500 Mo
> d'artefacts. Grâce à `dependency_overrides`, **elle n'en a pas besoin**.
>
> Et vos tests restent **rapides** : 0,04 s au lieu de plusieurs secondes de chargement.

---

## ✅ Vérifiez que ça marche

- [ ] `uv run pytest -v` → **6 tests au vert**
- [ ] Les tests passent **avec `artefacts/` renommé**
- [ ] Cassez volontairement un test (mettez `assert corps["classe"] == "virginica"`) et
      relancez : pytest affiche précisément la valeur attendue et la valeur obtenue.
      **Remettez ensuite `setosa`.**

---

## 🔧 Si ça ne marche pas

| Message d'erreur | Cause | Solution |
|---|---|---|
| `ModuleNotFoundError: No module named 'app'` | bloc pytest absent de `pyproject.toml` | ajoutez `[tool.pytest.ini_options]` + `pythonpath = ["."]` |
| `no such table: journalprediction` | `with` oublié devant `TestClient(app)` | `with TestClient(app) as c:` |
| `test_health` échoue avec 503 | une seule dépendance surchargée | surchargez aussi `get_moteur_optionnel` |
| `fixture 'client' not found` | mauvais nom ou mauvais dossier | le fichier doit s'appeler `conftest.py` et être dans `tests/` |
| `collected 0 items` | noms de fichier/fonction non conformes | fichier `test_*.py`, fonctions `test_*` |
| `AttributeError: 'MoteurFactice' object has no attribute ...` | le faux est incomplet | il doit exposer les mêmes attributs que `MoteurIris` |
| Le test échoue seulement au 2ᵉ lancement | données laissées par le test précédent | supprimez `journal.db` entre deux exécutions |

---

## 💡 Ce que vous venez d'apprendre

- **`dependency_overrides` est la récompense du `Depends`.** Une API bien câblée est une
  API testable ; une API qui importe ses ressources en dur ne l'est pas.
- **On ne teste pas le modèle ici, on teste l'API.** Deux responsabilités distinctes :
  - la qualité du **modèle** se mesure avec l'accuracy (Lab 5) ;
  - la qualité de l'**API** se mesure avec ces tests.
  Ne les confondez jamais.
- Des tests **rapides et sans dépendance externe** sont des tests qui seront **réellement
  exécutés**. Des tests lents finissent désactivés.

> ### 🎯 Ce que ces 6 tests garantissent, concrètement
> Le jour où quelqu'un modifiera `MesureFleur` en enlevant `gt=0`, le test
> `test_valeur_negative_rejetee` **échouera immédiatement**, avant même la revue de code.
> Vous venez de poser un filet de sécurité sur le contrat d'entrée de votre modèle.

---

➡️ **Lab suivant : [Lab 10 — Déploiement Docker](lab-10-docker.md)** ⭐
