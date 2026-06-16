# TP - Seance 19 : Livraison continue (CD) avec GitHub Actions

**Duree : 1 h 30 - Pre-requis : TP Seance 18 (CI) termine, Dockerfile.api fonctionnel (Seances 8/12)**

## Objectifs
- Distinguer integration continue (CI) et livraison continue (CD).
- Construire et **pousser** automatiquement une image Docker sur un registre (GHCR).
- Gerer l'authentification et les permissions d'un workflow (`GITHUB_TOKEN`, `packages: write`).
- Versionner les images avec des tags (latest, SHA, version semantique).

## Contexte
La CI (Seance 18) **valide** chaque changement. La CD va plus loin : elle **livre** un
artefact pret a deployer. Ici, a chaque fusion sur `main` (et a chaque tag de version), on
construit l'image Docker de l'API et on la pousse sur le **GitHub Container Registry**
(`ghcr.io`). N'importe qui (selon la visibilite) peut alors faire `docker pull` de votre
image, sans rebuild.

Le squelette a completer (marqueurs `TODO S19-n`) est dans
`todo/.github/workflows/cd.yml`. Copiez-le dans `.github/workflows/cd.yml` **a la racine de
votre depot**.

## CI vs CD (a retenir)
| | Declencheur | But | Resultat |
|---|---|---|---|
| **CI** (S18) | push, pull request | valider | checks verts/rouges, artefact de test |
| **CD** (S19) | push sur main, tags | livrer | image poussee sur le registre |

## Etape 1 - Declencheurs (`TODO S19-1`)
On ne livre pas a chaque push de branche : seulement sur `main` (apres fusion d'une PR
validee par la CI) et sur les tags de version :
```yaml
on:
  push:
    branches: [main]
    tags: ["v*.*.*"]
  workflow_dispatch:
```
> Un tag `v1.0.0` declenchera donc une livraison versionnee.

## Etape 2 - Permissions (`TODO S19-2`)
Pousser un package sur GHCR demande une permission explicite. Au niveau du job `build-push` :
```yaml
    permissions:
      contents: read
      packages: write     # S19-2 : requis pour pousser sur GHCR
```
> Principe de moindre privilege : on n'ouvre que ce qui est necessaire.

## Etape 3 - Connexion a GHCR (`TODO S19-3`)
Remplacez l'etape `run: echo ...` par l'action officielle de connexion. Le jeton
`GITHUB_TOKEN` est fourni automatiquement par GitHub Actions (aucun secret a creer) :
```yaml
      - name: Connexion a GHCR
        uses: docker/login-action@v3      # S19-3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
```

## Etape 4 - Calcul des tags (`TODO S19-4`)
L'action `metadata-action` genere des tags coherents a partir du contexte (branche, SHA, tag) :
```yaml
      - name: Calcul des tags et labels
        id: meta
        uses: docker/metadata-action@v5    # S19-4
        with:
          images: ghcr.io/${{ github.repository }}/mlproject-api
          tags: |
            type=raw,value=latest,enable={{is_default_branch}}
            type=sha
            type=semver,pattern={{version}}
```
> `latest` sur la branche par defaut, `sha-xxxx` pour tracer chaque commit, `1.0.0` sur un tag.

## Etape 5 - Build & push (`TODO S19-5`)
Construisez l'image de l'API et poussez-la avec les tags calcules :
```yaml
      - name: Build & push de l'image
        uses: docker/build-push-action@v6   # S19-5
        with:
          context: .
          file: docker/Dockerfile.api
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

## Etape 6 - Declencher et verifier
```bash
git add .github/workflows/cd.yml
git commit -m "Ajoute le pipeline CD"
git push                       # fusionnez sur main pour declencher la livraison
```
Apres le run, allez sur la page **Packages** de votre profil/depot : l'image
`mlproject-api` doit apparaitre. Testez-la :
```bash
docker pull ghcr.io/<votre_compte>/<votre_repo>/mlproject-api:latest
```
Pour une livraison versionnee :
```bash
git tag v1.0.0 && git push --tags
```

## Criteres de reussite
- [ ] Le workflow se declenche sur `main` et sur les tags `vX.Y.Z`.
- [ ] Le job a la permission `packages: write` et se connecte a GHCR.
- [ ] L'image `mlproject-api` est visible dans **Packages** apres le run.
- [ ] Un `docker pull` de l'image `:latest` fonctionne.

## Pour aller plus loin
- **Quality gate avant livraison** : ajoutez un job `quality` (comme en S18) et faites
  `build-push` dependre de lui (`needs`) : on ne livre jamais du code casse.
- **Matrice d'images** : poussez aussi `mlproject-train` et `mlproject-frontend` via une
  `strategy: matrix`.
- **SBOM + provenance** : activez `sbom: true` et `provenance: true` dans `build-push-action`
  (tracabilite de la chaine d'approvisionnement).
- **Release** : sur un tag, creez une GitHub Release et attachez-y le modele entraine
  (action `softprops/action-gh-release`).
- **Dependabot** : ajoutez `.github/dependabot.yml` pour mettre a jour automatiquement les
  actions, les dependances Python et les images de base.
