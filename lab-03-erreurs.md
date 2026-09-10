# Lab 3 — Erreurs, codes HTTP et `HTTPException` · 20 min

## 🎯 Ce que vous allez faire

Apprendre à répondre **proprement quand ça se passe mal**, et comprendre la différence
fondamentale entre « le client a mal demandé » et « nous avons un bug ».

**Point de départ** : Lab 2 terminé.

---

## Les trois familles de codes HTTP

| Famille | Signification | Qui est responsable ? | Exemples |
|---|---|---|---|
| **2XX** | ça a marché | — | `200` OK · `201` Created · `204` No Content |
| **4XX** | **le client** a mal demandé | l'appelant | `400` · `404` Not Found · `422` Unprocessable |
| **5XX** | **le serveur** a planté | **vous** | `500` Internal Error · `503` Unavailable |

> ### La distinction à retenir pour toute votre carrière
> - Un **`422`**, c'est votre API qui **fait son travail** : elle a détecté une donnée
>   invalide et l'a refusée proprement. Rien à corriger.
> - Un **`500`**, c'est un **bug chez vous**. En production, ça veut dire un ticket à ouvrir,
>   une astreinte réveillée, un post-mortem.
>
> Renvoyer un `500` là où un `404` suffisait, c'est déclencher une alerte pour rien.
> Renvoyer un `200` alors que ça a échoué, c'est bien pire : le client croit que tout va bien.

---

## Étape 1 — Renvoyer un 404 quand la ressource n'existe pas

Pour l'instant, `GET /mesures/{id}` n'existe pas. Ajoutons-la.

Complétez l'import en haut de `app/main.py` :

```python
from fastapi import FastAPI, HTTPException, Query, Response, status
```

Puis ajoutez à la fin du fichier :

```python
@app.get("/mesures/{mesure_id}", response_model=MesureEnregistree, tags=["Mesures"])
def lire_mesure(mesure_id: int):
    for mesure in mesures:
        if mesure.id == mesure_id:
            return mesure
    raise HTTPException(
        status_code=status.HTTP_404_NOT_FOUND,
        detail=f"Mesure {mesure_id} introuvable",
    )
```

> ⚠️ **Rappel du Lab 1 : l'ordre des routes.** `@app.get("/mesures/")` doit être déclarée
> **avant** `@app.get("/mesures/{mesure_id}")`. Vérifiez votre fichier.

**Ce qui compte ici** : on écrit `raise HTTPException(...)`, pas `return`.

- `raise` **interrompt immédiatement** la fonction.
- FastAPI intercepte cette exception et la transforme en réponse HTTP propre :
  `{"detail": "Mesure 99 introuvable"}` avec le code `404`.
- Le message est **pour le client** : il doit être clair et utile, mais **ne jamais révéler
  de détail interne** (chemin de fichier, requête SQL, trace d'erreur).

**Testez** :
- Créez une mesure via `POST /mesures/`, puis `GET /mesures/1` → **200**
- `GET /mesures/99` → **404** avec votre message

---

## Étape 2 — Renvoyer un 204 quand il n'y a rien à dire

Ajoutez :

```python
@app.delete("/mesures/{mesure_id}", status_code=status.HTTP_204_NO_CONTENT, tags=["Mesures"])
def supprimer_mesure(mesure_id: int):
    for index, mesure in enumerate(mesures):
        if mesure.id == mesure_id:
            mesures.pop(index)
            return Response(status_code=status.HTTP_204_NO_CONTENT)
    raise HTTPException(
        status_code=status.HTTP_404_NOT_FOUND,
        detail=f"Mesure {mesure_id} introuvable",
    )
```

**`204 No Content`** veut dire : « ça a marché, et je n'ai **rien** à te renvoyer ».
Le corps de la réponse est **vide** — c'est la norme HTTP, une réponse 204 ne doit pas
avoir de contenu. D'où le `return Response(status_code=204)` au lieu d'un `return {...}`.

**Testez** : `DELETE /mesures/1` → **204**, et dans Swagger le « Response body » est vide.
Refaites la même requête → **404** (elle n'existe plus).

---

## Étape 3 — Provoquer un 500 (volontairement !)

Ajoutez cette route absurde :

```python
@app.get("/boum", tags=["Démo"])
def boum():
    return {"resultat": 1 / 0}
```

**Testez** <http://127.0.0.1:8000/boum> → **500 Internal Server Error**.

Maintenant **regardez le terminal où tourne le serveur**. Vous y voyez la trace complète :

```
Traceback (most recent call last):
  ...
  File "app/main.py", line ..., in boum
    return {"resultat": 1 / 0}
                        ~~^~~
ZeroDivisionError: division by zero
```

> ### 💡 Deux observations importantes
> 1. **Le client n'a rien vu de tout ça.** Il a juste reçu `Internal Server Error`. C'est
>    voulu : exposer une trace d'erreur, c'est offrir à un attaquant la structure de votre
>    code. La trace reste **côté serveur**, dans les logs.
> 2. **Le serveur n'est pas mort.** Il a encaissé l'exception, renvoyé un 500, et continue
>    de servir les autres requêtes. Testez `/` : ça marche toujours.

**Une fois l'expérience faite, supprimez la route `/boum`** — on ne laisse pas de bombe
dans son code.

---

## Étape 4 — Le tableau récapitulatif

Complétez ce tableau à partir de ce que vous venez d'observer. **C'est le livrable du lab.**

| Situation | Code observé | Qui est fautif ? |
|---|---|---|
| `GET /mesures/1` sur une mesure existante | ? | — |
| `POST /mesures/` réussi | ? | — |
| `DELETE /mesures/1` réussi | ? | — |
| `GET /fleurs/?limit=999` (max 50) | ? | ? |
| `GET /mesures/99` (inexistante) | ? | ? |
| `GET /boum` | ? | ? |

<details>
<summary>👉 Cliquez pour voir le corrigé (essayez d'abord !)</summary>

| Situation | Code | Qui est fautif ? |
|---|---|---|
| Lecture réussie | `200` | — |
| Création réussie | `201` | — |
| Suppression réussie, rien à renvoyer | `204` | — |
| `limit=999` alors que le max est 50 | `422` | **le client** |
| Identifiant inexistant | `404` | **le client** |
| Division par zéro dans le code | `500` | **le serveur — vous** |

</details>

---

## ✅ Vérifiez que ça marche

- [ ] `GET /mesures/99` → **404** avec un message lisible
- [ ] `DELETE` d'une mesure existante → **204** avec un corps vide
- [ ] `DELETE` de la même mesure une deuxième fois → **404**
- [ ] `/boum` → **500** côté client, trace complète côté serveur
- [ ] La route `/boum` a été supprimée après l'expérience
- [ ] Le tableau récapitulatif est rempli

---

## 🔧 Si ça ne marche pas

| Symptôme | Cause | Solution |
|---|---|---|
| `NameError: name 'HTTPException' is not defined` | import oublié | `from fastapi import ..., HTTPException, ...` |
| `NameError: name 'Response' is not defined` | import oublié | ajoutez `Response` à la même ligne d'import |
| `GET /mesures/1` renvoie 422 | ordre des routes | `/mesures/` doit être **avant** `/mesures/{mesure_id}` |
| Le 404 renvoie une page HTML | vous avez fait `return` au lieu de `raise` | `raise HTTPException(...)` |
| `204` avec un corps non vide | vous faites `return {...}` | `return Response(status_code=204)` |
| `GET /mesures/` renvoie `[]` | la liste est vide | créez d'abord une mesure avec `POST` |

---

## 💡 Ce que vous venez d'apprendre

- `raise HTTPException(status_code=..., detail=...)` est **la** façon de signaler une erreur
  métier dans FastAPI.
- Le bon code HTTP porte du sens : `404` ≠ `422` ≠ `500`, et ils ne déclenchent pas les
  mêmes réactions côté client et côté supervision.
- Les traces d'erreur restent **côté serveur**. Jamais dans la réponse.

> ### 🔗 Le pont vers le Lab 7
> Au **Lab 7**, on se posera la même question pour le modèle de ML :
> *si l'inférence échoue, quel code renvoyer ?*
> Réponse : **`503 Service Unavailable`** — « le service est temporairement incapable de
> répondre » — et surtout pas un `500` muet. Un `503` dit à l'appelant « réessaie dans un
> instant » ; un `500` dit « il y a un bug, ne réessaie pas ».

---

➡️ **Lab suivant : [Lab 4 — Ranger le projet](lab-04-structuration.md)**
