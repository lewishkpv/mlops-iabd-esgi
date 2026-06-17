# Guide de deploiement sur VPS (Oracle Cloud / DigitalOcean)

Ce guide explique comment deployer la stack complete du projet (MLflow, MySQL,
API FastAPI, frontend Streamlit, Airflow) sur un VPS gratuit ou a credits, avec
`docker compose` et le `Makefile`.

> Avertissement securite : l'API, MLflow et Airflow sont exposes sans
> authentification forte. Ce deploiement est prevu pour une demo de cours, pas
> pour de la production. N'exposez les ports que le temps de l'evaluation, et
> de preference restreints a votre adresse IP.

## 1. Choisir et creer le VPS

La stack demande au moins 4 Go de RAM (Airflow en consomme la majeure partie).
Visez 8 Go pour etre confortable.

### Option A : Oracle Cloud Always Free (gratuit a vie)
- Creer un compte sur https://www.oracle.com/cloud/free/ (carte bancaire
  demandee mais non debitee).
- Creer une instance Compute :
  - Image : Ubuntu 22.04
  - Shape : VM.Standard.A1.Flex (ARM Ampere), 2 a 4 OCPU et 12 a 24 Go de RAM
    (compris dans l'offre Always Free).
- Telecharger la cle SSH proposee a la creation.

### Option B : DigitalOcean (via GitHub Student Pack)
- Activer le GitHub Student Developer Pack avec votre adresse etudiante :
  https://education.github.com/pack (credit DigitalOcean de 200 $).
- Creer un Droplet :
  - Image : Ubuntu 22.04
  - Taille : 8 Go RAM / 2 vCPU (couvert par le credit etudiant)
  - Authentification : cle SSH.

Notez l'adresse IP publique de la machine, vous en aurez besoin partout ci-dessous.

## 2. Se connecter en SSH

```sh
ssh ubuntu@VOTRE_IP        # Oracle (utilisateur "ubuntu")
ssh root@VOTRE_IP          # DigitalOcean (utilisateur "root")
```

## 3. Installer Docker, git et make

```sh
sudo apt-get update
sudo apt-get install -y git make ca-certificates curl

# Docker Engine + plugin compose (methode officielle)
curl -fsSL https://get.docker.com | sudo sh

# Lancer docker sans sudo (se reconnecter ensuite pour appliquer)
sudo usermod -aG docker $USER
newgrp docker

# Verifications
docker --version
docker compose version
make --version
```

## 4. Cloner le depot

```sh
git clone https://github.com/VOTRE_COMPTE/VOTRE_REPO.git
cd VOTRE_REPO/mlops-churn-starter   # adapter au chemin du projet
```

Si le depot est prive, utilisez un token GitHub ou une cle de deploiement.

## 5. Configurer l'environnement (optionnel)

Copier le fichier d'exemple et l'ajuster si besoin :

```sh
cp .env.example .env
```

En mode docker, laissez `DB_URL` commente : les services se parlent par le
reseau interne du compose (mysql, mlflow, api). Aucune modification n'est
necessaire pour un deploiement standard.

## 6. Lancer la stack

```sh
make workflow-docker
```

Cette commande, dans l'ordre :
1. libere les ports locaux occupes (free-ports),
2. demarre le serveur MLflow,
3. entraine le modele dans un conteneur (alimente le volume des modeles),
4. demarre MySQL, MLflow, l'API et le frontend.

Pour ajouter Airflow (re-entrainement planifie) :

```sh
make airflow
make airflow-password   # affiche les identifiants admin
```

## 7. Ouvrir les ports

Deux niveaux de pare-feu doivent autoriser le trafic : celui du fournisseur
cloud, puis celui du systeme (sur Oracle uniquement).

Ports utilises :

| Port | Service        |
|------|----------------|
| 8000 | API FastAPI    |
| 8501 | Frontend Streamlit |
| 5000 | MLflow UI      |
| 8080 | Airflow UI     |

Ne pas exposer le port 3306 (MySQL) : la base reste interne au compose.

### Pare-feu du fournisseur
- Oracle : dans la console, VCN -> Security Lists (ou Network Security Groups),
  ajouter des regles Ingress TCP pour 8000, 8501, 5000, 8080, source
  `0.0.0.0/0` (ou votre IP pour plus de securite).
- DigitalOcean : facultatif. Si vous creez un Cloud Firewall, ajoutez les memes
  ports en entree. Sans Cloud Firewall, passez directement a l'etape suivante.

### Pare-feu systeme (Oracle Ubuntu seulement)
Les images Ubuntu d'Oracle bloquent tout sauf SSH par defaut (iptables). Ouvrir
les ports sur la machine :

```sh
sudo iptables -I INPUT -p tcp --dport 8000 -j ACCEPT
sudo iptables -I INPUT -p tcp --dport 8501 -j ACCEPT
sudo iptables -I INPUT -p tcp --dport 5000 -j ACCEPT
sudo iptables -I INPUT -p tcp --dport 8080 -j ACCEPT
sudo netfilter-persistent save   # rend les regles permanentes
```

Sur DigitalOcean, si `ufw` est actif :

```sh
sudo ufw allow 8000,8501,5000,8080/tcp
```

## 8. Acceder aux services

Depuis votre navigateur, avec l'IP publique du VPS :

- API (docs interactives) : `http://VOTRE_IP:8000/docs`
- Frontend Streamlit : `http://VOTRE_IP:8501`
- MLflow UI : `http://VOTRE_IP:5000`
- Airflow UI : `http://VOTRE_IP:8080`

## 9. Gerer et arreter

```sh
docker compose ps                       # etat des services
docker compose logs -f api              # suivre les logs d'un service
make docker-down                        # arreter (conserve les donnees)
make docker-reset                       # arreter ET effacer les volumes
```

## Depannage

- La page Airflow ne charge pas (timeout) : manque de RAM, les workers du
  webserver sont tues (OOM). Prenez une instance avec plus de RAM, ou reduisez
  `AIRFLOW__WEBSERVER__WORKERS` dans `docker-compose.yml`.
- Un port ne repond pas : verifiez les DEUX pare-feu (fournisseur + systeme).
  Testez en local sur le VPS avec `curl http://localhost:PORT`.
- `docker: permission denied` : vous avez oublie `newgrp docker` ou il faut
  vous reconnecter en SSH apres `usermod -aG docker`.
- Build lent ou manque de memoire pendant l'entrainement : privilegiez une
  instance a 8 Go et evitez de lancer Airflow en meme temps que l'entrainement.
