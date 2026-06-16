# TP - Séance 11 : Tests du modèle (évaluation automatisée et porte qualité)

**Durée : 1 h · Pré-requis : un modèle enregistré au Model Registry (Séances 5-6), serveur MLflow démarré**

## Objectifs
- Évaluer un modèle enregistré avec `mlflow.models.evaluate` (métriques et graphiques générés automatiquement).
- Mettre en place une **porte qualité** : rejeter automatiquement un modèle dont les performances passent sous un seuil.
- Rattacher le jeu d'évaluation au run comme **dataset MLflow** (traçabilité données -> modèle).

## Contexte
Jusqu'ici, on calculait quelques métriques à la main dans les scripts d'entraînement. En production, on veut **standardiser** l'évaluation et surtout **automatiser une décision** : un modèle qui n'atteint pas un niveau minimal ne doit pas être promu.

`mlflow.models.evaluate` calcule en une passe un ensemble de métriques (`f1_score`, `roc_auc`, `accuracy_score`...) et d'artefacts (matrice de confusion, courbes ROC / précision-rappel). `mlflow.validate_evaluation_results` compare ensuite ces métriques à des seuils (`MetricThreshold`) et **lève une exception** si l'un n'est pas atteint : c'est la porte qualité, réutilisable en CI ou dans un DAG.

Le squelette à compléter est `todo/mlproject/evaluate.py` (marqueurs `TODO S11-n`). Les seuils sont lus depuis `config.py` (`EVAL_ROC_AUC_MIN`, `EVAL_F1_MIN`).

## Étape 1 - Préparer un modèle à évaluer (5 min)
Assurez-vous d'avoir au moins une version enregistrée dans le registry (sinon, relancez un entraînement avec suivi MLflow, Séances 5-6) :
```bash
PYTHONPATH=todo python -m mlproject.train          # ou train_models / train_optuna
```
`evaluate.py` ciblera par défaut la dernière version de `MODEL_NAME` (fonction `latest_model_uri`, fournie).

## Étape 2 - Définir les seuils de validation (`TODO S11-1`)
Dans `build_thresholds()`, retournez un dictionnaire `{metrique: MetricThreshold}` :
```python
from mlflow.models import MetricThreshold

return {
    "roc_auc": MetricThreshold(threshold=EVAL_ROC_AUC_MIN, greater_is_better=True),
    "f1_score": MetricThreshold(threshold=EVAL_F1_MIN, greater_is_better=True),
}
```

## Étape 3 - Évaluer et tracer le dataset (`TODO S11-2`)
Dans `evaluate_model()`, à l'intérieur du `with mlflow.start_run(...)` :
```python
dataset = mlflow.data.from_pandas(eval_df, source=str(DATA_PATH), targets=TARGET, name="eval")
mlflow.log_input(dataset, context="evaluation")

result = mlflow.models.evaluate(
    model_uri, data=eval_df, targets=TARGET,
    model_type="classifier", evaluators=["default"],
)
```
`eval_df` (déjà construit) contient les features **et** la colonne cible : c'est ce qu'attend `evaluate`.

## Étape 4 - Appliquer la porte qualité (`TODO S11-3`)
Toujours dans le run, si `validate` est actif :
```python
mlflow.validate_evaluation_results(build_thresholds(), result)
```
Puis retournez `result`.

## Étape 5 - Exécuter et observer (15 min)
```bash
PYTHONPATH=todo python -m mlproject.evaluate                 # évalue + valide
PYTHONPATH=todo python -m mlproject.evaluate --no-validate   # évalue sans porte qualité
```
Dans l'UI MLflow, ouvrez le run `evaluate` : observez les métriques, les graphiques (matrice de confusion, courbes ROC/PR) et l'onglet **Datasets** (jeu d'évaluation rattaché).

Pour voir la porte qualité **échouer**, relevez temporairement le seuil :
```bash
EVAL_ROC_AUC_MIN=0.99 PYTHONPATH=todo python -m mlproject.evaluate
```
Le script doit sortir en erreur (code de retour non nul) : c'est exactement ce qui bloquerait une promotion automatique.

## Critères de réussite
- [ ] `make ... evaluate` (ou la commande python) crée un run `evaluate` dans MLflow avec métriques et artefacts.
- [ ] Le jeu d'évaluation apparaît dans l'onglet *Datasets* du run.
- [ ] Avec un seuil impossible (ex. `EVAL_ROC_AUC_MIN=0.99`), la validation échoue et le script sort en erreur.

## Pour aller plus loin
- Branchez cette porte qualité dans un DAG Airflow (Séance 17) : un ré-entraînement ne promeut le modèle que s'il passe la validation.
- Ajoutez un modèle de référence (`baseline_result`) à `validate_evaluation_results` pour exiger une **amélioration** par rapport à la version en production.
