# Lab 5 — Entraîner et sérialiser le modèle · 30 min

## 🎯 Ce que vous allez faire

Entraîner un modèle de classification sur le jeu de données **Iris**, et le sauvegarder
dans un fichier réutilisable : **l'artefact**.

> ### ⚠️ Ce lab ne contient AUCUNE ligne de FastAPI. C'est volontaire.
> **L'entraînement et l'inférence sont deux mondes séparés.**
>
> | | Entraînement | Inférence (votre API) |
> |---|---|---|
> | Quand ? | une fois, hors ligne | des milliers de fois, en ligne |
> | Durée acceptable | minutes, heures, jours | quelques millisecondes |
> | A besoin de | tout le jeu de données | une seule observation |
> | Tourne où ? | sur un poste, un cluster | dans le service en production |
>
> **Le seul point de contact entre les deux, c'est un fichier : l'artefact.**
> Votre API n'importera jamais `train.py`. Elle ne connaîtra que le `.joblib`.

**Point de départ** : Lab 4 terminé.

---

## Étape 1 — Installer les outils de ML

```bash
uv add scikit-learn joblib
```

Cela prend une à deux minutes (scikit-learn embarque numpy et scipy, qui sont volumineux).

- **scikit-learn** : la bibliothèque de Machine Learning classique en Python.
- **joblib** : sait écrire des objets Python (dont des modèles) dans un fichier, et les relire.

---

## Étape 2 — Le fichier `train.py`

Créez **`train.py`** **à la racine du projet** — pas dans `app/`.
C'est important : ce fichier ne fait **pas** partie de l'application.

```python
"""Entraînement du modèle Iris.

À exécuter à la main, hors de l'API :  uv run python train.py
Ce fichier n'est JAMAIS importé par l'application.
"""

from datetime import datetime, timezone
from pathlib import Path

import joblib
import sklearn
from sklearn.datasets import load_iris
from sklearn.linear_model import LogisticRegression
from sklearn.metrics import accuracy_score, classification_report
from sklearn.model_selection import train_test_split
from sklearn.pipeline import Pipeline
from sklearn.preprocessing import StandardScaler

DOSSIER_ARTEFACTS = Path(__file__).parent / "artefacts"
CHEMIN_MODELE = DOSSIER_ARTEFACTS / "modele_iris.joblib"

# ⚠️ L'ORDRE DE CETTE LISTE EST UN CONTRAT.
# Le modèle est entraîné avec les colonnes dans CET ordre ; l'API devra construire
# son vecteur d'entrée exactement dans le même ordre. Voir l'encadré plus bas.
FEATURES = ["sepal_length", "sepal_width", "petal_length", "petal_width"]
VERSION_MODELE = "1.0.0"


def entrainer() -> dict:
    # 1. Charger les données (fournies avec scikit-learn, aucun téléchargement)
    donnees = load_iris()

    # 2. Séparer apprentissage et test
    X_train, X_test, y_train, y_test = train_test_split(
        donnees.data,
        donnees.target,
        test_size=0.2,
        random_state=42,
        stratify=donnees.target,
    )

    # 3. Construire le PIPELINE : normalisation PUIS modèle
    pipeline = Pipeline(
        [
            ("normalisation", StandardScaler()),
            ("modele", LogisticRegression(max_iter=1000)),
        ]
    )
    pipeline.fit(X_train, y_train)

    # 4. Évaluer sur les données jamais vues
    predictions = pipeline.predict(X_test)
    accuracy = float(accuracy_score(y_test, predictions))
    print(classification_report(y_test, predictions, target_names=donnees.target_names))
    print(f"Accuracy : {accuracy:.4f}")

    # 5. Emballer le pipeline AVEC ses métadonnées
    return {
        "pipeline": pipeline,
        "features": FEATURES,
        "classes": list(donnees.target_names),
        "version": VERSION_MODELE,
        "accuracy": round(accuracy, 4),
        "version_sklearn": sklearn.__version__,
        "entraine_le": datetime.now(timezone.utc).isoformat(),
    }


if __name__ == "__main__":
    artefact = entrainer()
    DOSSIER_ARTEFACTS.mkdir(parents=True, exist_ok=True)
    joblib.dump(artefact, CHEMIN_MODELE)
    print(f"Artefact écrit : {CHEMIN_MODELE} ({CHEMIN_MODELE.stat().st_size / 1024:.1f} Ko)")
```

---

## Étape 3 — Lancer l'entraînement

```bash
uv run python train.py
```

Sortie attendue (vos chiffres peuvent varier légèrement) :

```
              precision    recall  f1-score   support

      setosa       1.00      1.00      1.00        10
  versicolor       0.90      0.90      0.90        10
   virginica       0.90      0.90      0.90        10

    accuracy                           0.93        30
   macro avg       0.93      0.93      0.93        30
weighted avg       0.93      0.93      0.93        30

Accuracy : 0.9333
Artefact écrit : /…/iris-api/artefacts/modele_iris.joblib (1.9 Ko)
```

**Voilà. Le modèle fait 6 lignes utiles et pèse 2 Ko.**
Tout le reste des labs consiste à le rendre utilisable par quelqu'un d'autre que vous.

---

## Les trois idées à comprendre — c'est le cœur du lab

### 1️⃣ On sérialise un **pipeline**, pas un modèle nu

```python
Pipeline([("normalisation", StandardScaler()),
          ("modele", LogisticRegression(max_iter=1000))])
```

Le `StandardScaler` centre et réduit les données (il retire la moyenne et divise par
l'écart-type). Le modèle a appris **sur des données normalisées**. Si vous lui envoyez des
données brutes, il prédit n'importe quoi.

> **Si vous ne sauvegardiez que le `LogisticRegression`**, vous devriez réimplémenter la
> normalisation dans l'API, avec les moyennes et les écarts-types exacts de l'entraînement.
> Une erreur de recopie, et…
>
> **… l'API ne plante pas. Elle prédit faux, silencieusement.**
>
> C'est la source n°1 de bugs en production sur les modèles de ML. Ça a un nom :
> le **training/serving skew**. Un bug qui ne lève aucune erreur est infiniment plus
> dangereux qu'un bug qui plante.
>
> En sauvegardant le pipeline complet, la normalisation part **avec** le modèle. Le problème
> disparaît par construction.

### 2️⃣ On sauvegarde des **métadonnées**

On n'écrit pas `joblib.dump(pipeline, ...)` mais `joblib.dump({...}, ...)` — un dictionnaire.

> Dans six mois, face à un fichier `.joblib` sur un serveur, les seules questions qui
> comptent sont :
> - *Quelle version ?* → `version`
> - *Entraîné quand ?* → `entraine_le`
> - *Avec quelles features, dans quel ordre ?* → `features`
> - *Il valait quoi ?* → `accuracy`
> - *Avec quelle version de scikit-learn ?* → `version_sklearn`
>
> Sans ces informations, un artefact est un fichier binaire anonyme. Et vous n'oserez pas
> le supprimer, ni le déployer.

### 3️⃣ L'ordre des features est un **contrat**

Un modèle scikit-learn ne connaît pas le nom des colonnes. Il reçoit `[[5.1, 3.5, 1.4, 0.2]]`
et **suppose** que la 1ʳᵉ valeur est `sepal_length`, la 2ᵉ `sepal_width`, etc.

> Or le JSON envoyé par un client n'a **aucun ordre garanti** :
> `{"petal_width": 0.2, "sepal_length": 5.1, ...}` est un JSON parfaitement valide.
>
> Si votre API construit le vecteur dans l'ordre des clés reçues, elle enverra les mesures
> **dans le désordre** au modèle. Encore une fois : **aucune erreur, juste une prédiction
> fausse.**
>
> C'est pour ça qu'on sauvegarde la liste `features` : au Lab 6, l'API construira son
> vecteur en suivant **cette liste**, et jamais l'ordre du JSON.

---

## ✅ Vérifiez que ça marche

- [ ] Le fichier `artefacts/modele_iris.joblib` existe (`ls -la artefacts/`)
- [ ] L'accuracy affichée est supérieure à 0.85
- [ ] Cette commande de relecture fonctionne :

```bash
uv run python -c "
import joblib
a = joblib.load('artefacts/modele_iris.joblib')
print('version   :', a['version'])
print('accuracy  :', a['accuracy'])
print('features  :', a['features'])
print('classes   :', a['classes'])
print('sklearn   :', a['version_sklearn'])
print('prédiction:', a['classes'][a['pipeline'].predict([[5.1, 3.5, 1.4, 0.2]])[0]])
"
```

Vous devez obtenir `prédiction: setosa`.

**Vous venez de recharger un modèle depuis un fichier et de faire une prédiction, sans
réentraîner quoi que ce soit.** C'est exactement ce que fera votre API au Lab 6.

---

## 🔧 Si ça ne marche pas

| Message d'erreur | Cause | Solution |
|---|---|---|
| `ModuleNotFoundError: No module named 'sklearn'` | paquet non installé | `uv add scikit-learn` (le paquet s'appelle `scikit-learn`, le module `sklearn`) |
| L'installation échoue ou dure très longtemps | version de Python trop récente | vérifiez `uv run python --version` : 3.11 / 3.12 / 3.13 |
| `FileNotFoundError` sur `artefacts/` | dossier absent | c'est `mkdir(parents=True, exist_ok=True)` qui le crée ; vérifiez que la ligne est bien dans le bloc `if __name__ == "__main__":` |
| `ConvergenceWarning` | l'optimiseur n'a pas convergé | sans gravité ; `max_iter=1000` le corrige déjà |
| Vous lancez `python train.py` sans `uv run` | mauvais interpréteur Python | toujours `uv run python train.py` |

---

## ⚠️ Deux avertissements de professionnel

**Portabilité.** Un artefact `joblib` **n'est pas garanti compatible** entre deux versions
majeures de scikit-learn. C'est pour ça qu'on stocke `version_sklearn`, et c'est pour ça
qu'au Lab 10 on **figera** les versions dans un fichier de verrouillage.
👉 *Le fichier de verrouillage fait partie du modèle*, au même titre que le `.joblib`.

**Sécurité.** `joblib.load()` déserialise du **pickle**, et le pickle **exécute du code
Python** à la lecture. Charger un artefact venant d'une source non fiable revient à exécuter
un programme inconnu sur votre serveur. **On ne charge jamais un `.joblib` non maîtrisé.**

---

## 💡 Ce que vous venez d'apprendre

- Entraînement et inférence sont **deux mondes** ; l'artefact est leur unique point de contact.
- On sérialise **le pipeline complet**, pas l'estimateur seul.
- Un artefact sans **métadonnées** est ingérable en production.
- **L'ordre des features est un contrat**, à respecter côté API.

> ### 🔗 Ce qui vous attend
> Au Lab 6, vous allez charger ce fichier **une seule fois au démarrage** de l'API, et
> le réutiliser pour toutes les requêtes. Vous verrez pourquoi « une seule fois » est
> la différence entre une API à 3 ms et une API à 300 ms.

---

➡️ **Lab suivant : [Lab 6 — L'endpoint `/predict`](lab-06-endpoint-predict.md)** ⭐
