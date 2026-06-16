# TP - Séance 15 : Tester l'API (client de prévisions)

**Durée : 45 min · Pré-requis : API FastAPI fonctionnelle (Séance 12), un modèle dans `models/model.joblib`**

## Objectifs
- Écrire un client qui interroge l'API sur ses endpoints (`/health`, `/predict`, `/model-info`).
- Construire des payloads valides à partir de votre propre jeu de données.
- Vérifier le comportement de l'API sur plusieurs cas réels.

## Contexte
Une API qui démarre n'est pas une API qui marche : il faut la **tester** avec de vraies requêtes. `scripts/predict_client.py` est un petit client en ligne de commande qui envoie quelques clients de test à l'API et affiche les réponses. C'est la base d'un test d'intégration (et de la simulation de trafic du DAG de prévisions, Séance 17).

Les payloads dépendent de **votre** dataset : le client les construit en échantillonnant `data/` et en retirant la colonne cible (fonction `build_payloads`, fournie). Vous n'avez donc rien à coder de spécifique à votre cas d'usage.

Le squelette à compléter est `todo/scripts/predict_client.py` (marqueurs `TODO S15-n`).

## Étape 1 - Démarrer l'API (5 min)
Dans un terminal, avec un modèle déjà entraîné :
```bash
PYTHONPATH=todo uvicorn mlproject.api:app --host 127.0.0.1 --port 8000
```
Vérifiez la doc interactive sur http://127.0.0.1:8000/docs.

## Étape 2 - Appeler /health (`TODO S15-1`)
Dans `main()`, à l'intérieur du `with httpx.Client(...) as client:` :
```python
health = client.get("/health")
logger.info("GET /health -> %s %s", health.status_code, health.json())
```

## Étape 3 - Envoyer des prévisions et lire /model-info (`TODO S15-2`)
```python
for i, payload in enumerate(payloads):
    response = client.post("/predict", json=payload)
    logger.info("POST /predict (#%d) -> %s %s", i, response.status_code, response.json())

info = client.get("/model-info")
logger.info("GET /model-info -> %s %s", info.status_code, info.json())
```

## Étape 4 - Exécuter le client (10 min)
Depuis la racine du projet, l'API tournant dans l'autre terminal :
```bash
PYTHONPATH=todo python todo/scripts/predict_client.py
PYTHONPATH=todo python todo/scripts/predict_client.py --url http://127.0.0.1:8000
```
Vous devez voir un `200` sur `/health`, une prédiction (`0`/`1` et une probabilité) par client, puis les infos du modèle.

## Étape 5 - Tester les cas limites (10 min)
- Coupez l'API et relancez le client : observez l'erreur de connexion (httpx).
- Envoyez un payload invalide (champ manquant ou type incorrect) avec `curl` ou depuis `/docs` : l'API doit répondre `422` (validation Pydantic).

## Critères de réussite
- [ ] Le client affiche les réponses de `/health`, de plusieurs `/predict` et de `/model-info`.
- [ ] Les codes de statut sont `200` quand l'API tourne avec un modèle chargé.
- [ ] Un payload invalide renvoie `422` (et non `500`).

## Pour aller plus loin
- Transformez ces appels en vrais tests automatisés avec `pytest` et le `TestClient` de FastAPI (pas besoin de lancer un serveur).
- Réutilisez `build_payloads` dans le DAG `daily_predictions` (Séance 17) pour simuler un trafic quotidien de prévisions.
