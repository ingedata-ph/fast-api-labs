# Lab 0 — Votre première API · 20 min

## 🎯 Ce que vous allez faire

Créer le projet `iris-api`, écrire une API de 6 lignes, la lancer, et découvrir la
**documentation automatique** que FastAPI génère pour vous.

À la fin de ce lab, vous aurez un serveur qui tourne et qui répond à votre navigateur.

---

## Étape 1 — Créer le projet

Ouvrez un terminal. Placez-vous là où vous rangez vos projets (par exemple `Documents`
ou `Bureau`), puis :

```bash
uv init --no-package --python 3.12 iris-api
cd iris-api
```

> ### ⚠️ Pourquoi `--no-package` ? Ne l'oubliez pas.
> Sans cette option, `uv` crée un projet « packagé » avec un dossier `src/` et une
> configuration de build. Notre code ira dans un dossier `app/` à la racine, et il **ne serait
> pas trouvé** : vous auriez une erreur `ModuleNotFoundError: No module named 'app'`
> au Lab 4. Avec `--no-package`, le projet est un simple dossier de fichiers Python : c'est ce
> qu'on veut.
>
> `--python 3.12` évite les mauvaises surprises : certaines bibliothèques scientifiques
> (scikit-learn, que l'on installera au Lab 5) ne sont pas encore disponibles pour les
> versions de Python toutes récentes.

Regardez ce qui a été créé :

```bash
ls -a
```

Vous devez voir :

```
.python-version   README.md   main.py   pyproject.toml
```

- **`pyproject.toml`** : la carte d'identité du projet. C'est là que `uv` note les
  bibliothèques dont vous avez besoin.
- **`main.py`** : un fichier d'exemple créé par `uv`. **Nous n'en avons pas besoin, supprimez-le** :

```bash
rm main.py
```

---

## Étape 2 — Installer FastAPI

```bash
uv add "fastapi[standard]"
```

> **Que fait cette commande ?** Elle télécharge FastAPI **et** tout ce qui va avec
> (le serveur Uvicorn, l'outil en ligne de commande `fastapi`, la validation Pydantic…),
> les range dans un dossier caché `.venv/`, et écrit la dépendance dans `pyproject.toml`.
> Les guillemets autour de `"fastapi[standard]"` sont **obligatoires** : sans eux, votre
> terminal interprète les crochets `[ ]` et la commande échoue.

Cela prend 20 à 60 secondes la première fois. Vérifiez que ça a marché :

```bash
uv run python -c "import fastapi; print(fastapi.__version__)"
```

Vous devez voir un numéro de version, par exemple `0.141.1`.

---

## Étape 3 — Créer le dossier de l'application

Tout notre code va vivre dans un dossier `app/`.

```bash
mkdir app
touch app/__init__.py
```

> ### ⚠️ Le fichier `__init__.py` est vide, et il est indispensable
> Ce fichier vide dit à Python : « ce dossier est un **package**, tu peux importer des
> choses depuis l'intérieur ». Sans lui, vous aurez des `ImportError` incompréhensibles
> dès le Lab 4. Sur Windows, si `touch` n'existe pas, créez le fichier depuis votre
> éditeur : clic droit sur `app` → *Nouveau fichier* → nommez-le `__init__.py`.

---

## Étape 4 — Écrire l'API

Créez le fichier **`app/main.py`** et tapez ceci :

```python
from fastapi import FastAPI

app = FastAPI(
    title="Iris API",
    description="Service de prédiction — formation Industrialisation IA",
    version="0.1.0",
)


@app.get("/")
def racine():
    return {"service": "iris-api", "statut": "ok"}
```

**Décortiquons ces 10 lignes**, parce que tout le reste de la journée en découle :

| Ligne | Ce qu'elle fait |
|---|---|
| `from fastapi import FastAPI` | on importe l'outil |
| `app = FastAPI(...)` | on crée l'application. `title`, `description` et `version` apparaîtront dans la documentation |
| `@app.get("/")` | **le décorateur**. Il dit : « quand quelqu'un fait un `GET` sur l'adresse `/`, appelle la fonction juste en dessous » |
| `def racine():` | une fonction Python tout ce qu'il y a de plus normale |
| `return {...}` | on renvoie un dictionnaire Python. **FastAPI le transforme tout seul en JSON** |

Le nom de la fonction (`racine`) n'a aucune importance technique — c'est vous qui choisissez.
Ce qui compte, c'est le décorateur au-dessus.

---

## Étape 5 — Lancer le serveur

```bash
uv run fastapi dev app/main.py
```

Vous devez voir défiler quelque chose comme :

```
INFO   Will watch for changes in these directories: ['/…/iris-api']
INFO   Uvicorn running on http://127.0.0.1:8000 (Press CTRL+C to quit)
INFO   Application startup complete.
```

> **`fastapi dev` = mode développement.** Le serveur surveille vos fichiers : dès que vous
> sauvegardez, il redémarre tout seul. Vous n'aurez plus jamais à le relancer à la main
> aujourd'hui — sauf indication contraire.
>
> **Laissez ce terminal ouvert.** Le serveur tourne dedans. Pour toutes les autres commandes,
> **ouvrez un second terminal**.

---

## Étape 6 — Découvrir Swagger

Ouvrez votre navigateur sur ces deux adresses :

1. <http://127.0.0.1:8000> → vous voyez `{"service":"iris-api","statut":"ok"}`
2. <http://127.0.0.1:8000/docs> → **c'est ici que ça devient intéressant**

Cette page, personne ne l'a écrite. FastAPI a **lu votre code** et en a déduit la
documentation. C'est la norme **OpenAPI**.

**Faites-le maintenant** :
1. Cliquez sur la ligne `GET /`.
2. Cliquez sur le bouton **« Try it out »**.
3. Cliquez sur **« Execute »**.
4. Regardez le bloc **« Curl »** qui apparaît : c'est la commande exacte que votre navigateur
   vient d'exécuter. Vous pouvez la copier dans un terminal, elle marchera.
5. Regardez le **« Server response »** : code `200`, et le corps de la réponse.

Ouvrez aussi <http://127.0.0.1:8000/openapi.json> : c'est le fichier brut, en JSON, que Swagger
lit pour dessiner la page. C'est ce fichier qu'on donne à une équipe frontend ou mobile.

---

## ✅ Vérifiez que ça marche

- [ ] `http://127.0.0.1:8000` affiche `{"service":"iris-api","statut":"ok"}`
- [ ] `http://127.0.0.1:8000/docs` affiche une page avec la route `GET /`
- [ ] « Try it out » → « Execute » renvoie un code **200**
- [ ] Dans `app/main.py`, changez `"statut": "ok"` en `"statut": "opérationnel"`, sauvegardez,
      rechargez la page : la réponse a changé **sans que vous ayez relancé le serveur**

---

## 🔧 Si ça ne marche pas

| Message d'erreur | Cause | Solution |
|---|---|---|
| `command not found: uv` | `uv` pas installé, ou terminal pas rouvert | réinstallez `uv`, **fermez et rouvrez le terminal** |
| `zsh: no matches found: fastapi[standard]` | guillemets oubliés | `uv add "fastapi[standard]"` |
| `[Errno 48] Address already in use` | le port 8000 est déjà pris | `uv run fastapi dev app/main.py --port 8001` |
| `Error loading ASGI app` | mauvais chemin de fichier | vous devez être **dans** le dossier `iris-api`, et le fichier doit être `app/main.py` |
| `{"detail":"Not Found"}` | vous avez tapé une adresse qui n'existe pas | seule `/` existe pour l'instant |
| La page ne se met pas à jour | fichier non sauvegardé | `Cmd+S` / `Ctrl+S`, et regardez le terminal : il doit afficher `Reloading...` |

**Pour arrêter le serveur** : cliquez dans le terminal et faites `Ctrl+C`.

---

## 💡 Ce que vous venez d'apprendre

- Un projet Python moderne se crée et se gère avec **`uv`** — pas de `pip`, pas de
  `virtualenv` à activer à la main.
- Une API FastAPI, c'est **une fonction Python + un décorateur** qui lui associe une adresse.
- **La documentation est générée à partir du code.** Elle ne peut donc jamais être périmée.

> ### 🤔 Question à se poser
> Combien de temps vous aurait pris la rédaction à la main de cette page `/docs` ?
> Et surtout : combien de temps pour la **maintenir à jour** à chaque modification du code ?

---

➡️ **Lab suivant : [Lab 1 — Routes et paramètres](lab-01-routes-parametres.md)**
