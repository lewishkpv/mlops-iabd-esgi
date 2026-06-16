# TP - Seance 18 : Integration continue (CI) avec GitHub Actions

**Duree : 1 h 30 - Pre-requis : depot GitHub du projet (package `mlproject`), Docker non requis**

## Objectifs
- Comprendre ce qu'est l'integration continue et a quoi elle sert.
- Ecrire un workflow GitHub Actions : declencheurs, jobs, etapes, actions.
- Construire un "quality gate" (lint + types + tests) qui s'execute a chaque push et pull request.
- Enchainer les jobs en graphe (`needs`) et publier un artefact (Continuous Training).

## Contexte
Jusqu'ici, la qualite du code repose sur la discipline de chacun : on pense a lancer
`make check` avant de pousser... ou on oublie. L'integration continue automatise ce reflexe :
a **chaque** push et **chaque** pull request, GitHub execute le lint, les types et les tests
sur une machine neuve. Plus de "ca marche chez moi" : si le pipeline est rouge, on le voit
avant de fusionner.

Le squelette a completer (marqueurs `TODO S18-n`) est dans
`todo/.github/workflows/ci.yml`. Copiez-le dans `.github/workflows/ci.yml` **a la racine de
votre depot** (GitHub n'execute que les workflows places la).

## Vocabulaire (a connaitre)
- **Workflow** : un fichier YAML decrivant un pipeline (`.github/workflows/*.yml`).
- **Trigger** (`on:`) : l'evenement declencheur (push, pull_request, manuel...).
- **Job** : un ensemble d'etapes qui tournent sur une machine (runner). Les jobs sont
  paralleles par defaut ; `needs:` cree une dependance (un graphe, comme un DAG Airflow).
- **Step** : une etape, soit une commande (`run:`), soit une action reutilisable (`uses:`).
- **Action** : une brique reutilisable (ex: `actions/checkout`, `astral-sh/setup-uv`).
- **Artifact** : un fichier produit par un job et telechargeable (ici : le modele).

## Etape 1 - Declencheurs (`TODO S18-1`)
Le squelette ne se declenche que manuellement (`workflow_dispatch`). Faites-le tourner aussi
a chaque `push` et `pull_request` :
```yaml
on:
  push:
  pull_request:
  workflow_dispatch:
```
> Astuce : gardez `workflow_dispatch` pour pouvoir relancer le pipeline a la main depuis
> l'onglet **Actions**.

## Etape 2 - Le quality gate (`TODO S18-2 a S18-5`)
Dans le job `quality`, remplacez les `echo` par les vraies commandes. Le `checkout` et
l'installation de `uv` (avec cache) sont **deja fournis**.
```yaml
      - name: Installation des dependances (projet + dev)
        run: uv sync --extra dev      # S18-2

      - name: Lint (ruff)
        run: uv run ruff check mlproject   # S18-3

      - name: Types (mypy)
        run: uv run mypy mlproject    # S18-4

      - name: Tests (pytest)
        run: uv run pytest            # S18-5
```
> La CI rejoue exactement vos commandes locales (`make check`). Elle ne reinvente rien.

## Etape 3 - Enchainer les jobs (`TODO S18-6`)
Le job `train` ne doit s'executer que si `quality` est **vert**. Ajoutez la dependance :
```yaml
  train:
    name: Entrainement (Continuous Training)
    runs-on: ubuntu-latest
    needs: quality        # S18-6
```
> Sans `needs`, les deux jobs partiraient en parallele : on entrainerait un modele meme si
> les tests echouent. Le `needs` garantit l'ordre.

## Etape 4 - Entrainer et publier le modele (`TODO S18-7, S18-8`)
Preparez les donnees puis entrainez (adaptez a votre projet) :
```yaml
      - name: Donnees + entrainement de la baseline
        run: uv run python -m mlproject.train    # S18-7 (precede de votre prep de donnees)
```
Puis publiez le modele en artefact telechargeable :
```yaml
      - name: Publication du modele en artefact
        uses: actions/upload-artifact@v4          # S18-8
        with:
          name: model
          path: models/model.joblib
          if-no-files-found: error
```

## Etape 5 - Pousser et observer
```bash
git add .github/workflows/ci.yml
git commit -m "Ajoute le pipeline CI"
git push
```
Ouvrez l'onglet **Actions** de votre depot : le workflow doit apparaitre et tourner. Ouvrez
une pull request : les checks doivent s'afficher en bas de la PR.

## Criteres de reussite
- [ ] Le workflow se declenche sur push ET sur pull request.
- [ ] Le job `quality` execute reellement ruff, mypy et pytest (et echoue si l'un echoue).
- [ ] Le job `train` ne demarre qu'apres un `quality` vert (`needs`).
- [ ] Le modele `model.joblib` est telechargeable depuis la page du run (artefact).

## Pour aller plus loin
- **Cache** : observez le gain de temps du cache `uv` entre deux runs (etape "Installation de uv").
- **Couverture** : ajoutez `--cov=mlproject --cov-report=term-missing` a pytest.
- **Lockfile** : remplacez `uv sync` par `uv sync --locked` pour echouer si `uv.lock` n'est
  pas a jour avec `pyproject.toml`.
- **Securite** : ajoutez un job qui scanne les vulnerabilites des dependances
  (`uv run --with pip-audit pip-audit`) et les secrets (binaire `gitleaks`).
- **Filtrer** : avec `paths:`, ne declenchez le pipeline que si le code change (pas la doc).

La livraison de l'image Docker (push sur un registre) fait l'objet du **TP Seance 19 (CD)**.
