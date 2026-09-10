# Labs — Industrialiser un modèle de Machine Learning avec FastAPI

Bienvenue ! Cet après-midi, vous allez construire **`iris-api`** : un vrai service web qui
expose un modèle de Machine Learning, de la première ligne de code jusqu'au conteneur Docker.

> **Le modèle fait 6 lignes. Le travail d'industrialisation en fait 300.**
> C'est ça, le métier. Le sujet de la journée n'est pas le modèle : c'est **tout ce qu'il y a
> autour** — validation des entrées, contrat d'API, chargement, robustesse, traçabilité,
> tests, déploiement.

---

## Comment travailler

Les labs se font **dans l'ordre**. Chacun part de l'état où le précédent s'est arrêté.

1. Ouvrez le fichier du lab.
2. Lisez **« Ce que vous allez faire »** avant de taper quoi que ce soit.
3. Suivez les étapes une par une. **Tapez le code, ne le copiez-collez pas** : c'est en tapant
   qu'on repère les détails (les deux-points, l'indentation, les annotations de type).
4. À la fin de chaque lab, faites la section **« ✅ Vérifiez que ça marche »**.
   Ne passez pas au suivant tant qu'elle n'est pas verte.
5. Chaque lab contient une section **« 🔧 Si ça ne marche pas »** avec les erreurs les plus
   fréquentes et leur solution. Lisez-la avant d'appeler à l'aide, il y a de fortes chances
   que votre message d'erreur y soit.

---

## Les 11 labs

| # | Titre | Durée | Ce que vous saurez faire après |
|---|---|---|---|
| [0](lab-00-premiere-api.md) | Première API | 20 min | Créer un projet et lancer un serveur qui répond |
| [1](lab-01-routes-parametres.md) | Routes et paramètres | 25 min | Faire entrer des variables dans votre API |
| [2](lab-02-pydantic.md) | Pydantic : le contrat de données | 30 min | Valider automatiquement ce qu'on vous envoie |
| [3](lab-03-erreurs.md) | Erreurs et codes HTTP | 20 min | Répondre proprement quand ça se passe mal |
| [4](lab-04-structuration.md) | Ranger le projet | 30 min | Découper en routers, schémas et configuration |
| [5](lab-05-entrainer-le-modele.md) | Entraîner et sérialiser | 30 min | Produire un artefact ML réutilisable |
| [6](lab-06-endpoint-predict.md) ⭐ | L'endpoint `/predict` | 45 min | **Transformer un modèle en service** |
| [7](lab-07-robustesse.md) | Robustesse | 40 min | `/health`, mode batch, validation métier |
| [8](lab-08-journalisation.md) | Traçabilité | 30 min | Journaliser les prédictions en base |
| [9](lab-09-tests.md) | Tests automatisés | 30 min | Tester l'API sans charger le modèle |
| [10](lab-10-docker.md) ⭐ | Déploiement Docker | 40 min | **Livrer un conteneur qui tourne partout** |

Les labs **0, 1, 2, 5, 6 et 10** sont le minimum vital : ils portent la promesse du cours.
Si le temps manque, ce sont ceux-là qu'il faut finir.

---

## Avant de commencer — vérifiez votre poste

Ouvrez un terminal et lancez ces trois commandes. **Les trois doivent répondre.**

```bash
uv --version         # attendu : uv 0.9 ou plus récent
python3 --version    # attendu : Python 3.11, 3.12 ou 3.13
docker --version     # attendu : Docker version 2x.x  (utile seulement au Lab 10)
```

Si `uv` n'est pas installé :

```bash
# macOS / Linux
curl -LsSf https://astral.sh/uv/install.sh | sh

# Windows (PowerShell)
powershell -c "irm https://astral.sh/uv/install.ps1 | iex"
```

Puis **fermez et rouvrez votre terminal**, sinon la commande `uv` reste introuvable.

---

## Le vocabulaire de la journée

Vous allez croiser ces mots en permanence. Gardez cette liste sous la main.

| Mot | Traduction simple |
|---|---|
| **endpoint** | une adresse de votre API, par exemple `POST /predict` |
| **route** | synonyme d'endpoint |
| **schéma Pydantic** | le « moule » qui décrit à quoi doit ressembler une donnée |
| **artefact** | le fichier `.joblib` qui contient le modèle entraîné |
| **pipeline** | la chaîne complète *préparation des données → modèle* |
| **sérialiser** | transformer un objet Python en fichier (ou en texte JSON) |
| **inférence** | faire une prédiction avec un modèle déjà entraîné |
| **lifespan** | le code que FastAPI exécute au démarrage et à l'arrêt du serveur |
| **dépendance (`Depends`)** | une fonction que FastAPI appelle pour vous et dont il vous donne le résultat |
| **conteneur** | une boîte qui embarque votre code + ses dépendances, exécutable partout |

---

## Où vous arriverez ce soir

```
iris-api/
├── pyproject.toml              # les dépendances du projet
├── uv.lock                     # les versions exactes (Lab 10)
├── train.py                    # entraînement du modèle (Lab 5)
├── artefacts/
│   └── modele_iris.joblib      # le modèle entraîné (Lab 5)
├── app/
│   ├── __init__.py
│   ├── main.py                 # l'application FastAPI
│   ├── config.py               # la configuration (Lab 4)
│   ├── database.py             # la connexion SQLite (Lab 8)
│   ├── dependances.py          # l'accès au modèle (Lab 6)
│   ├── ml/
│   │   └── moteur.py           # le chargement + l'inférence (Lab 6)
│   ├── models/
│   │   └── journal.py          # la table des prédictions (Lab 8)
│   ├── schemas/
│   │   ├── fleur.py            # les schémas d'entrée/sortie (Lab 2)
│   │   └── prediction.py       # les schémas de prédiction (Lab 6)
│   └── routers/
│       ├── mesures.py          # /mesures      (Lab 4)
│       ├── prediction.py       # /predict      (Lab 6)
│       ├── systeme.py          # /health       (Lab 7)
│       └── journal.py          # /journal      (Lab 8)
├── tests/                      # les tests (Lab 9)
├── Dockerfile                  # (Lab 10)
└── docker-compose.yml          # (Lab 10)
```

Et vous saurez lancer ceci, **depuis un conteneur**, sur n'importe quelle machine :

```bash
curl -X POST http://localhost:8000/predict/ \
  -H "Content-Type: application/json" \
  -d '{"sepal_length":6.3,"sepal_width":3.3,"petal_length":6.0,"petal_width":2.5}'

# {"classe":"virginica","confiance":0.9914, ...}
```

Bon courage 💪
