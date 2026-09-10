# Lab 10 ⭐ — Déploiement local : Uvicorn en production et Docker · 40 min

## 🎯 Ce que vous allez faire

**C'est l'objectif final du cours.** Vous allez emballer votre service dans un **conteneur**
qui tourne à l'identique sur votre machine, sur celle de votre voisin, ou sur un serveur.

**Point de départ** : Lab 9 terminé (ou au minimum le Lab 7).

---

## Étape 1 — De `dev` à `production` · 5 min

Jusqu'ici vous avez toujours lancé :

```bash
uv run fastapi dev app/main.py
```

Voici la commande de **production** :

```bash
uv run uvicorn app.main:app --host 0.0.0.0 --port 8000 --workers 4
```

Lancez-la, ouvrez <http://127.0.0.1:8000/docs>, faites une prédiction, puis `Ctrl+C`.

**Ce qui change** :

| Option | Pourquoi |
|---|---|
| pas de `--reload` | en dev, le serveur surveille tous vos fichiers en permanence. Coûteux et inutile en production |
| `--workers 4` | 4 processus indépendants servent les requêtes en parallèle |
| `--host 0.0.0.0` | écoute sur **toutes** les interfaces réseau. **Indispensable dans un conteneur** |
| `app.main:app` | « dans le module `app.main`, prends la variable `app` » |

> ### ⚠️ LE calcul qu'un ingénieur ML doit savoir faire
> **Chaque worker charge sa propre copie du modèle en mémoire.**
>
> Pour Iris (2 Ko), 4 workers = 8 Ko. Indolore.
> Pour un modèle de 2 Go : **4 workers × 2 Go = 8 Go de RAM**.
>
> C'est la question à poser avant tout dimensionnement de serveur. Regardez les logs au
> démarrage : vous verrez `Modèle v1.0.0 chargé.` s'afficher **4 fois**, une par worker.

> ### ⚠️ `--host 0.0.0.0` et la sécurité
> `127.0.0.1` = « seulement ma machine ». `0.0.0.0` = « toutes les interfaces ».
> Dans un conteneur c'est obligatoire (sinon le service est injoignable depuis l'extérieur),
> mais sur une machine exposée à Internet, cela signifie que **n'importe qui peut vous
> appeler**. En production réelle, on place un reverse proxy (nginx, Traefik) devant.

---

## Étape 2 — Figer les dépendances · 5 min

```bash
uv lock
```

Cela produit **`uv.lock`** : la liste des versions **exactes** de toutes vos dépendances,
et de leurs dépendances, et ainsi de suite.

> ### 💡 Pourquoi c'est vital pour un service ML
> Votre artefact `.joblib` a été produit par **une version précise** de scikit-learn (elle
> est écrite dans `version_sklearn`, cf. Lab 5). Si l'image Docker était reconstruite dans
> six mois avec une version majeure plus récente, le chargement du pickle pourrait échouer —
> ou pire, réussir en modifiant subtilement le comportement.
>
> **Le fichier de verrouillage fait partie du modèle**, au même titre que le `.joblib`.
> Versionnez-les ensemble.

---

## Étape 3 — Le `Dockerfile` · 15 min

### D'abord, le `.dockerignore`

Créez **`.dockerignore`** à la racine :

```
.venv/
__pycache__/
*.pyc
.git/
.pytest_cache/
tests/
journal.db
*.md
donnees/
```

> **Sans ce fichier, Docker copie tout** — dont votre `.venv/` de plusieurs centaines de Mo,
> qui serait de toute façon inutilisable dans le conteneur (compilé pour votre OS).
> Le build serait très lent et l'image énorme. **Ne sautez pas cette étape.**

### Puis le `Dockerfile`

Créez **`Dockerfile`** (sans extension) à la racine :

```dockerfile
FROM python:3.12-slim

ENV PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    UV_COMPILE_BYTECODE=1 \
    UV_LINK_MODE=copy

WORKDIR /app

# On récupère l'exécutable uv depuis son image officielle
COPY --from=ghcr.io/astral-sh/uv:latest /uv /usr/local/bin/uv

# On copie D'ABORD uniquement les fichiers de dépendances.
# Cette couche est mise en cache : elle n'est reconstruite que si
# pyproject.toml ou uv.lock changent.
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev --no-install-project

# Le code applicatif change souvent : on le copie EN DERNIER.
COPY app/ ./app/
COPY artefacts/ ./artefacts/

ENV PATH="/app/.venv/bin:$PATH"

EXPOSE 8000

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "2"]
```

**Ligne par ligne** :

| Instruction | Ce qu'elle fait |
|---|---|
| `FROM python:3.12-slim` | l'image de base. **`-slim`** ≈ 150 Mo, contre ~1 Go pour `python:3.12` |
| `ENV PYTHONUNBUFFERED=1` | les `print()` sortent immédiatement dans les logs Docker |
| `WORKDIR /app` | le dossier de travail dans le conteneur |
| `COPY --from=...uv` | récupère l'exécutable `uv` |
| `uv sync --frozen --no-dev` | installe **exactement** les versions du lock, **sans** les paquets de dev (pytest reste dehors) |
| `COPY app/`, `COPY artefacts/` | le code, puis le modèle |
| `EXPOSE 8000` | documentation : « ce conteneur écoute sur 8000 » |
| `CMD [...]` | la commande lancée au démarrage du conteneur |

> ### 🔑 L'ordre des `COPY` n'est pas cosmétique
> Docker met **chaque instruction en cache**. Si une couche change, toutes celles d'après
> sont reconstruites.
>
> Vos dépendances changent une fois par mois ; votre code, vingt fois par jour. En copiant
> les dépendances **avant** le code, la longue étape `uv sync` est réutilisée depuis le
> cache à chaque rebuild. **Vous passez de 2 minutes à 5 secondes.**

### Construire et lancer

```bash
docker build -t iris-api:1.0.0 .
docker run --rm -p 8000:8000 iris-api:1.0.0
```

Puis ouvrez <http://localhost:8000/docs> et faites une prédiction.

> **`-p 8000:8000`** relie le port 8000 de votre machine au port 8000 du conteneur.
> Sans ça, le service tourne mais reste inaccessible.
> **`--rm`** supprime le conteneur à l'arrêt (pas l'image).
> `Ctrl+C` pour arrêter.

---

## Étape 4 — `docker compose` · 10 min

Créez **`docker-compose.yml`** :

```yaml
services:
  api:
    build: .
    image: iris-api:1.0.0
    ports:
      - "8000:8000"
    environment:
      IRIS_NOM_SERVICE: iris-api
      IRIS_CHEMIN_MODELE: /app/artefacts/modele_iris.joblib
      IRIS_CHEMIN_BDD: /app/donnees/journal.db
    volumes:
      # Le journal des prédictions survit à la destruction du conteneur.
      - ./donnees:/app/donnees
    healthcheck:
      test: ["CMD", "python", "-c", "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')"]
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 10s
    restart: unless-stopped
```

```bash
docker compose up --build
```

Dans un **second terminal** :

```bash
docker compose ps
```

Attendez ~15 secondes : la colonne `STATUS` doit afficher **`Up ... (healthy)`**.

> ### 🔑 Trois choses viennent de se connecter
>
> **1. `environment` ← le Lab 4.** Les variables `IRIS_*` écrasent la configuration
> Pydantic. **Aucune ligne de code n'a changé** entre votre machine et le conteneur.
>
> **2. `healthcheck` ← le Lab 7.** Docker appelle votre `/health` toutes les 10 secondes.
> S'il répond `503`, le conteneur est marqué *unhealthy* — et un orchestrateur le retirerait
> du trafic ou le redémarrerait. **L'endpoint que vous avez écrit sert pour de vrai.**
>
> **3. `volumes` ← le Lab 8.** Le dossier `./donnees` de votre machine est monté dans le
> conteneur. La base SQLite y est écrite, donc elle **survit** à `docker compose down`.
> Sans volume, un conteneur détruit emporte toutes ses données.

---

## Étape 5 — La validation finale de la journée · 5 min

**C'est votre « soutenance ». Exécutez ces quatre commandes.**

```bash
# 1. Le service est-il prêt ?
curl http://localhost:8000/health

# 2. Quel modèle tourne exactement ?
curl http://localhost:8000/model/info

# 3. Une vraie prédiction, depuis le conteneur
curl -X POST http://localhost:8000/predict/ \
  -H "Content-Type: application/json" \
  -d '{"sepal_length":6.3,"sepal_width":3.3,"petal_length":6.0,"petal_width":2.5}'
# attendu : {"classe":"virginica", "confiance":0.99...}

# 4. Le journal a-t-il enregistré l'appel ?
curl http://localhost:8000/journal/
```

Puis testez la persistance :

```bash
docker compose down
ls -la donnees/          # journal.db est toujours là
docker compose up -d
curl http://localhost:8000/journal/    # vos prédictions d'avant sont toujours là
docker compose down
```

---

## ✅ Critères de validation finaux

- [ ] `docker build` se termine sans erreur
- [ ] `docker compose ps` affiche **`(healthy)`**
- [ ] `/predict/` répond correctement **depuis le conteneur**
- [ ] `docker compose down` puis `up` → le service revient tout seul
- [ ] Le fichier `donnees/journal.db` survit à la destruction du conteneur
- [ ] `docker images iris-api` → notez la taille

> ### 🤔 Regardez la taille de votre image
> ```bash
> docker images iris-api
> ```
> Vous devriez lire **entre 1 et 1,3 Go**. C'est beaucoup pour un modèle de 2 Ko !
>
> **D'où ça vient ?** numpy, scipy et scikit-learn pèsent à eux seuls plusieurs centaines
> de Mo, et l'image de base ~150 Mo.
>
> **Comment on réduit, en vrai ?** *multi-stage build*, image de base `alpine`, ou
> `--no-deps` avec des roues précompilées. Hors sujet aujourd'hui, mais sachez que la
> question se pose : une image de 1 Go, c'est 1 Go à transférer à **chaque** déploiement,
> sur **chaque** machine.

---

## 🔧 Si ça ne marche pas

| Symptôme | Cause | Solution |
|---|---|---|
| `curl` ne répond pas | `--host 127.0.0.1` dans le `CMD` | mettez `0.0.0.0` |
| `port is already allocated` | `fastapi dev` tourne encore | `Ctrl+C` dessus, ou `-p 8001:8000` |
| `FileNotFoundError` sur le `.joblib` | `artefacts/` non copié ou ignoré | vérifiez `COPY artefacts/` et l'absence d'`artefacts` dans `.dockerignore` |
| Build très lent, image énorme | `.venv` copié dans l'image | vérifiez le `.dockerignore` |
| `uv.lock` not found | `uv lock` pas exécuté | lancez `uv lock` avant le build |
| `error: failed to sync: frozen` | lock désynchronisé du `pyproject.toml` | relancez `uv lock` |
| `(unhealthy)` | `/health` renvoie 503 | le modèle n'est pas chargé : vérifiez `IRIS_CHEMIN_MODELE` et la copie de `artefacts/` |
| `Cannot connect to the Docker daemon` | Docker Desktop pas lancé | ouvrez Docker Desktop et attendez qu'il démarre |
| Le journal est vide après redémarrage | volume non monté | vérifiez le bloc `volumes:` **et** `IRIS_CHEMIN_BDD` |

### Variante sans `uv` dans l'image

Si `uv` pose problème en salle, remplacez les trois lignes `uv` du Dockerfile par :

```dockerfile
COPY requirements.txt ./
RUN pip install --no-cache-dir -r requirements.txt
```

et le `CMD` devient `["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]`.
Générez d'abord le fichier :

```bash
uv export --no-dev --format requirements-txt > requirements.txt
```

---

## 🎉 Vous y êtes

Récapitulons la journée. Vous êtes parti d'un modèle dans un notebook. Vous avez :

| Lab | Ce que vous avez ajouté |
|---|---|
| 0-1 | une API qui répond, documentée toute seule |
| 2 | un **contrat de données** validé automatiquement |
| 3 | des erreurs propres et des codes HTTP justes |
| 4 | une **architecture** qui tient la charge |
| 5 | un **artefact** versionné avec ses métadonnées |
| 6 | un modèle **chargé une fois** et servi mille fois |
| 7 | de la **robustesse** : batch, validation métier, `/health` |
| 8 | de la **traçabilité** en base |
| 9 | des **tests** qui tournent sans le modèle |
| 10 | un **conteneur** reproductible |

> ### 💬 Le mot de la fin
> *« Ce conteneur, vous pouvez le pousser sur n'importe quel serveur, n'importe quel cloud,
> n'importe quel cluster Kubernetes. Il y tournera à l'identique.*
>
> ***C'est ça, industrialiser un modèle : le rendre reproductible et exécutable ailleurs que
> sur votre portable.*** *Le modèle faisait 6 lignes. Tout le reste, c'était le métier. »*

---

## 🚀 Pour aller plus loin (chez vous)

1. **Changez de modèle.** Remplacez `LogisticRegression` par `RandomForestClassifier` dans
   `train.py`, passez la version à `2.0.0`, relancez l'entraînement puis
   `docker compose up --build`. **Aucune ligne de l'API à modifier** — c'est la preuve que
   l'architecture était la bonne.

2. **Un modèle lourd.** Branchez un modèle Hugging Face d'analyse de sentiment. Vous
   découvrirez pourquoi l'inférence doit alors passer par un *threadpool* (`run_in_threadpool`)
   pour ne pas bloquer la boucle asynchrone.

3. **Observabilité.** Ajoutez des logs structurés en JSON et un endpoint `/metrics` au
   format Prometheus (nombre de prédictions par classe, latence médiane).

4. **Intégration continue.** Ajoutez un fichier GitHub Actions qui exécute `uv run pytest`
   à chaque `push`. Vos tests du Lab 9 passeront — ils n'ont pas besoin du modèle.

---

⬅️ **Retour à l'[index des labs](README.md)**
