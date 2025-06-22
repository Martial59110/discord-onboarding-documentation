# Guide de Déploiement

Ce guide vous explique comment déployer l'application, qui se compose d'un backend (NestJS), d'un frontend (Angular) et d'une base de données PostgreSQL. Le tout est orchestré avec Docker.

## 1. Prérequis

Pour déployer ce projet, vous aurez besoin des outils suivants installés sur votre serveur :
- **Git**
- **Docker**

## 2. Structure du Projet

Le projet est un monorepo contenant les deux applications :
- `bots-discord-api/` : Le backend en NestJS.
- `front-bot/` : Le frontend en Angular.

## 3. Déploiement Manuel

Suivez ces étapes pour une installation propre et maîtrisée.

### Étape 1 : Cloner le Dépôt

Pour commencer, connectez-vous à votre serveur et clonez le dépôt :

```bash
git clone <URL_DU_DEPOT_GIT>
cd <NOM_DU_DOSSIER_PROJET>
```

### Étape 2 : Configurer le Backend

Le backend a besoin de plusieurs variables d'environnement pour fonctionner, notamment pour les accès à la base de données et à l'API Discord.

1.  **Créer le fichier `.env` :**
    Depuis la racine du projet, déplacez-vous dans le dossier du backend et copiez le fichier d'exemple pour créer votre configuration de production :
    ```bash
    cd bots-discord-api
    cp .env.production.example .env.production
    cd .. 
    ```

2.  **Remplir les variables :**
    Éditez le nouveau fichier `bots-discord-api/.env.production` et renseignez chaque variable. Voici à quoi elles correspondent :

| Variable                  | Description                                                                 | Exemple                           |
| ------------------------- | --------------------------------------------------------------------------- | --------------------------------- |
| `DB_USERNAME`             | Nom d'utilisateur pour la base de données PostgreSQL.                       | `postgres`                        |
| `DB_PASSWORD`             | Mot de passe pour la base de données. **À sécuriser !**                     | `un_mot_de_passe_solide`          |
| `DB_DATABASE`             | Nom de la base de données.                                                  | `bots_prod_db`                    |
| `DISCORD_CLIENT_ID`       | ID Client de votre application OAuth2 Discord.                              | `123456789012345678`              |
| `DISCORD_CLIENT_SECRET`   | Secret Client de votre application OAuth2 Discord. **À sécuriser !**        | `un_secret_discord_tres_long`     |
| `DISCORD_REDIRECT_URI`    | URL de callback configurée dans le portail développeur Discord.             | `https://votre-domaine.com/auth/callback` |
| `ALLOWED_GUILD_ID`        | ID du serveur Discord autorisé à utiliser le bot.                           | `123456789012345678`              |
| `DISCORD_BOT_TOKEN`       | Token de votre bot Discord. **À sécuriser !**                               | `un_token_de_bot_tres_long`       |
| `FRONTEND_URL`            | URL publique de votre application frontend.                                 | `https://votre-domaine.com`       |
| `JWT_SECRET`              | Chaîne de caractères longue et aléatoire pour signer les tokens JWT.        | `une_phrase_secrete_tres_longue_et_aleatoire` |
| `JWT_EXPIRATION`          | Durée de validité d'un token JWT.                                           | `24h`                             |
| `NODE_ENV`                | Environnement d'exécution (toujours `production` ici).                      | `production`                      |


### Étape 3 : Préparer Docker Compose

Maintenant, retournez à la racine du projet pour y créer un fichier `docker-compose.yml`. C'est ce fichier qui va orchestrer nos trois services (base de données, backend, et frontend).

```yaml
version: '3.8'

services:
  # Service de la base de données PostgreSQL
  db:
    image: postgres:15-alpine
    container_name: bots-db-prod
    restart: unless-stopped
    environment:
      POSTGRES_USER: ${DB_USERNAME}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: ${DB_DATABASE}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${DB_DATABASE}"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Service du Backend NestJS
  backend:
    build: ./bots-discord-api
    container_name: bots-api-prod
    restart: unless-stopped
    env_file:
      - ./bots-discord-api/.env.production
    ports:
      - "3000:3000"
    depends_on:
      db:
        condition: service_healthy

  # Service du Frontend Angular
  frontend:
    build: ./front-bot
    container_name: bots-front-prod
    restart: unless-stopped
    ports:
      - "80:80"

volumes:
  postgres_data:
    driver: local
```

> **Important :** Pour que les variables `${DB_...}` soient comprises par Docker Compose pour le service `db`, vous avez deux options :
> - Créez un fichier `.env` à la racine du projet contenant uniquement `DB_USERNAME`, `DB_PASSWORD` et `DB_DATABASE`.
> - Remplacez directement les variables dans le `docker-compose.yml` par leurs valeurs.

### Étape 4 : Lancer l'Application

Une fois que tout est configuré, vous pouvez lancer l'ensemble de l'application avec une seule commande :

```bash
docker-compose up -d --build
```
- `-d` : Lance les conteneurs en arrière-plan.
- `--build` : Reconstruit les images si le code a changé.

### Étape 5 : Appliquer les Migrations de la Base de Données

La première fois que vous lancez l'application (et à chaque mise à jour qui modifie la base de données), vous devez appliquer les migrations :

```bash
docker-compose exec backend npm run migration:run
```
Cette commande se connecte au conteneur du backend et y exécute le script de migration.

## 4. Mises à Jour

Pour mettre à jour l'application, le processus est simple :

```bash
# 1. Récupérer les derniers changements depuis Git
git pull

# 2. Reconstruire et relancer les conteneurs
docker-compose up -d --build

# 3. Appliquer les nouvelles migrations si besoin
docker-compose exec backend npm run migration:run
```

---

## 5. Déploiement Automatisé avec GitHub Actions

En plus du déploiement manuel, ce projet est équipé d'un workflow d'intégration et de déploiement continus (CI/CD) qui automatise la mise en production du backend.

Comme vous l'avez demandé, le processus de CI et de CD est regroupé dans un unique fichier : `.github/workflows/deploy.yml`.

### Comment ça marche ?

Le principe est simple : à chaque `push` sur la branche `main`, une "action" GitHub se lance et s'occupe de tout :

1.  **Intégration Continue (CI) - La construction :**
    - Le code est récupéré.
    - Une image Docker est construite pour le backend en utilisant son `Dockerfile.prod`.
    - L'image est ensuite envoyée sur Docker Hub.

2.  **Déploiement Continu (CD) - La mise en production :**
    - L'action se connecte en SSH à votre serveur.
    - Elle télécharge la nouvelle image depuis Docker Hub.
    - Elle recrée le fichier `.env.production` sur le serveur à partir des secrets GitHub.
    - Enfin, elle relance les services avec `docker-compose` pour utiliser la nouvelle version.

### Configuration

Pour que la magie opère, vous devez donner au workflow les clés de la maison. Allez dans les paramètres de votre dépôt GitHub, section `Settings > Secrets and variables > Actions`, et ajoutez les secrets suivants :

| Secret                  | Description                                                                 |
| ----------------------- | --------------------------------------------------------------------------- |
| `DOCKERHUB_USERNAME`    | Votre nom d'utilisateur Docker Hub.                                         |
| `DOCKERHUB_TOKEN`       | Un token d'accès généré depuis Docker Hub.                                  |
| `SSH_HOST`              | L'adresse IP ou le nom de domaine de votre serveur de production.           |
| `SSH_USERNAME`          | Le nom d'utilisateur pour la connexion SSH (ex: `root`, `ubuntu`).          |
| `SSH_KEY`               | La clé SSH privée (contenu du fichier `~/.ssh/id_rsa`) pour la connexion.   |
| `SSH_PORT`              | Le port SSH de votre serveur (généralement `22`).                           |
| `POSTGRES_HOST`         | Nom du service de la base de données (si sur le même réseau Docker).        |
| `POSTGRES_PORT`         | Port de la base de données.                                                 |
| `POSTGRES_USER`         | Nom d'utilisateur pour la base de données.                                  |
| `POSTGRES_PASSWORD`     | Mot de passe pour la base de données.                                       |
| `POSTGRES_DB`           | Nom de la base de données de production.                                    |

> **À noter :** Ce workflow ne gère que le déploiement du backend. Nous n'avons pas de workflow pour l'instant pour le projet `front-bot`. 
