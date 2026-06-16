# TP — Séance 5 : Suivi d'expériences avec MLflow Tracking

**Durée : 1 h 10 · Pré-requis : baseline fonctionnelle (`PYTHONPATH=todo python -m mlproject.train`), config.py adapté à votre dataset (TP S0)**

## Objectifs
- Lancer un serveur MLflow local.
- Instrumenter un entraînement pour tracer paramètres, métriques, modèle et artefacts.
- Comparer plusieurs runs dans l'interface MLflow.

## Contexte
Le fichier `todo/mlproject/train.py` entraîne un modèle de classification mais **ne garde aucune trace**
des expériences : impossible de comparer deux essais ou de retrouver le modèle d'un run passé.
Vous allez corriger cela avec MLflow. Les emplacements à compléter sont signalés par des
marqueurs `TODO (S5-n)`.

## Étape 1 — Démarrer le serveur de tracking (10 min)
Deux options, au choix :

```bash
# Option A — local, sans Docker
mlflow server --host 127.0.0.1 --port 5000 \
  --backend-store-uri sqlite:///mlflow.db --default-artifact-root ./mlruns

# Option B — via le docker-compose fourni
docker compose up -d mlflow
```
Ouvrez http://localhost:5000 : l'UI doit s'afficher (encore vide).

## Étape 2 — Brancher le client MLflow (`TODO S5-1, S5-2`)
Dans `train.py` :
- importez `mlflow` et `mlflow.sklearn` ;
- au début de `train()`, pointez le client sur le serveur et nommez l'expérience :

```python
mlflow.set_tracking_uri(MLFLOW_TRACKING_URI)   # déjà dans config.py
mlflow.set_experiment(MLFLOW_EXPERIMENT)
```

## Étape 3 — Encadrer l'entraînement dans un run (`TODO S5-3`)
Englobez l'entraînement **et** l'évaluation dans un bloc :
```python
with mlflow.start_run(run_name=f"logreg-c{c}"):
    ...
```

## Étape 4 — Logger params, métriques et modèle (`TODO S5-4 à S5-6`)
À l'intérieur du run :
```python
mlflow.log_params({"c": c, "max_iter": max_iter, "model": "logreg"})
mlflow.log_metrics(metrics)                       # {"f1": ..., "roc_auc": ...}
mlflow.sklearn.log_model(model, name="model")
```

## Étape 5 — Générer plusieurs runs et comparer (15 min)
```bash
PYTHONPATH=todo python -m mlproject.train --c 0.1
PYTHONPATH=todo python -m mlproject.train --c 1.0
PYTHONPATH=todo python -m mlproject.train --c 10.0
```
Dans l'UI : sélectionnez les 3 runs → **Compare**. Identifiez le meilleur `roc_auc` et
observez le modèle enregistré comme artefact de chaque run.

## Bonus (`TODO S5-7`)
Sauvegardez la matrice de confusion en image et loggez-la :
```python
from sklearn.metrics import ConfusionMatrixDisplay
import matplotlib.pyplot as plt
ConfusionMatrixDisplay.from_predictions(y_test, preds)
plt.savefig("confusion.png"); mlflow.log_artifact("confusion.png")
```

## Partie 2 - Centraliser la configuration et tracer le dataset (`tracking.py`)
Répéter `set_tracking_uri` + `set_experiment` dans chaque script devient vite redondant.
Le fichier `todo/mlproject/tracking.py` regroupe cette configuration et ajoute la
**traçabilité des données** (dataset lineage). Complétez les `TODO S5-8` et `S5-9`.

### `setup_experiment()` (`TODO S5-8`)
Pointez le tracking, sélectionnez l'expérience et appliquez-lui sa description et ses tags
(lus depuis `config.py`) :
```python
mlflow.set_tracking_uri(MLFLOW_TRACKING_URI)
experiment = mlflow.set_experiment(MLFLOW_EXPERIMENT)
client = mlflow.MlflowClient()
if MLFLOW_EXPERIMENT_DESCRIPTION:
    client.set_experiment_tag(experiment.experiment_id, "mlflow.note.content",
                              MLFLOW_EXPERIMENT_DESCRIPTION)
for key, value in MLFLOW_EXPERIMENT_TAGS.items():
    client.set_experiment_tag(experiment.experiment_id, key, value)
```
La description (`mlflow.note.content`) et les tags sont visibles dans l'onglet de
l'expérience, dans l'UI.

### `log_dataset()` (`TODO S5-9`)
Rattachez le jeu de données au run courant (source, schéma, profil), visible dans
l'onglet **Datasets** :
```python
dataset = mlflow.data.from_pandas(df, source=str(DATA_PATH), targets=TARGET, name=name)
mlflow.log_input(dataset, context=context)   # context = "training", "evaluation"...
```

### Utiliser le helper
Dans `train()`, remplacez le `set_tracking_uri`/`set_experiment` de l'étape 2 par
`setup_experiment()`, et appelez `log_dataset(df, context="training")` à l'intérieur du
run. Relancez un entraînement : l'expérience a maintenant une description, des tags, et le
run référence le dataset.

## Critères de réussite
- [ ] Chaque exécution crée un run visible dans l'UI sous l'expérience `classification-baseline`.
- [ ] Params et métriques sont consultables et comparables entre runs.
- [ ] Le modèle est récupérable depuis l'onglet *Artifacts* d'un run.
- [ ] L'expérience porte une description et des tags ; le run référence un dataset (onglet *Datasets*).

## Pour aller plus loin (préparation S6)
Activez `mlflow.sklearn.autolog()` avant le `fit` et observez ce qui est capturé
automatiquement. On enregistrera le meilleur modèle dans le **Model Registry** en séance 6.
