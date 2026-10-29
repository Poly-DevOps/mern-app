# TP2 DevOps : Docker et Docker Compose (application MERN)

Containerisation d'une application MERN (MongoDB, Express, React, Node.js)
avec Docker, puis orchestration avec Docker Compose.

## Structure du projet

```
mern-app/
├── client/              # Application React
│   ├── Dockerfile
│   └── .dockerignore
├── server/              # API Express
│   ├── Dockerfile
│   └── .dockerignore
├── docker-compose.yml
├── .env.example
└── docs/screenshots/
```

## Prérequis

- Docker Desktop (moteur démarré) sous Windows avec WSL 2
- Git
- Ports 3000 et 9000 libres sur la machine

## Configuration

Les variables d'environnement sont documentees dans `.env.example`.
Pour changer les valeurs, copier ce fichier en `.env` (non versionne) :

```powershell
Copy-Item .env.example .env
```

| Variable | Rôle |
|---|---|
| `MONGO_URI` | URL de connexion à MongoDB. Dans Docker, l'hôte est le nom du service : `mongodb` |
| `REACT_APP_API_URL` | URL de l'API **vue depuis le navigateur** (`http://localhost:9000`) |

Important : `REACT_APP_API_URL` est intégrée au bundle React lors du build.
Le navigateur ne connaît pas les noms de services Docker : il faut donc
`http://localhost:9000` et non `http://server:9000`.

## Partie 1 : Déploiement manuel avec Docker

### Construction des images

```powershell
cd server
docker build -t mern-server .
cd ..\client
docker build -t mern-client .
cd ..
```

### Réseau, volume et MongoDB

```powershell
docker network create mern-network
docker volume create mern-data
docker run -d --name mongodb --network mern-network -v mern-data:/data/db mongo
```

### Serveur et client

```powershell
docker run -d --name server --network mern-network -p 9000:9000 -e MONGO_URI=mongodb://mongodb:27017/mern mern-server
docker run -d --name client --network mern-network -p 3000:3000 mern-client
```

Verification : `docker ps`, `docker logs server`, puis http://localhost:3000.

![Conteneurs manuels](docs/screenshots/02-manual-containers.png)
![Application](docs/screenshots/03-app-manual.png)

Nettoyage avant de passer à Compose :

```powershell
docker rm -f client server mongodb
docker network rm mern-network
docker volume rm mern-data
```

## Partie 2 : Docker Compose

Le fichier `docker-compose.yml` définit trois services sur le réseau
`mern-network` (driver bridge) et un volume nommé `mern-data` :

- `mongodb` : image `mongo`, volume persistant, politique de redémarrage
  `unless-stopped` et healthcheck (`mongosh` ping).
- `server` : build de `./server`, port 9000, `MONGO_URI`, démarre seulement
  quand `mongodb` est `healthy` (`depends_on` avec `condition: service_healthy`).
- `client` : build de `./client`, port 3000, dépend de `server`.

### Commandes

| Action | Commande |
|---|---|
| Construire et démarrer | `docker compose up --build -d` |
| État des services | `docker compose ps` |
| Logs d'un service | `docker compose logs --tail=50 server` |
| Arrêter (données conservées) | `docker compose down` |
| Arrêter et supprimer les données | `docker compose down -v` |

![docker compose up](docs/screenshots/04-compose-up.png)
![docker compose ps](docs/screenshots/05-compose-ps.png)
![Logs du serveur](docs/screenshots/06-server-logs.png)

## Tests effectués

1. **Bout en bout** : depuis http://localhost:3000, ajout, modification et
   suppression d'un enregistrement (React, API Express, MongoDB).

   ![CRUD](docs/screenshots/07-app-crud.png)

2. **Persistance** : `docker compose down` (sans `-v`), vérification que le
   volume `mern-data` existe toujours, puis `docker compose up -d`.
   L'enregistrement créé avant l'arrêt est toujours présent.

   ![Volume](docs/screenshots/08-volume-after-down.png)
   ![Persistance](docs/screenshots/09-persistence.png)

## Diagnostiquer un service qui échoue au démarrage

1. `docker compose ps -a` : repérer l'état (`Exited`, `Restarting`, `unhealthy`).
2. `docker compose logs --tail=50 <service>` : lire l'erreur.
3. `docker inspect <conteneur>` ou `docker compose config` : vérifier variables
   d'environnement, réseau et healthcheck.
4. `docker compose exec <service> sh` : tester depuis l'intérieur
   (par exemple `ping mongodb`).
5. Corriger puis `docker compose up --build -d`.

Causes fréquentes : `MONGO_URI` pointant vers `localhost` au lieu de `mongodb`,
serveur démarré avant MongoDB, port déjà utilisé sur la machine.

## Difficultés rencontrées et solutions

- **Moteur Docker injoignable** (`failed to connect to the docker API`) :
  Docker Desktop n'était pas démarré. Solution : le lancer et attendre
  "Engine running".
- **Port 3000 déjà utilisé** (`ports are not available`) : un autre processus
  occupait le port. Identifié avec `Get-NetTCPConnection -LocalPort 3000`,
  puis arrêté avec `Stop-Process`.
- **Fichiers `.env` versionnés** : `client/.env` et `server/.env` étaient suivis
  par Git. Retirés avec `git rm --cached`, ajoutés au `.gitignore`, et
  remplacés par `.env.example`.
- **URL de l'API côté client** : utiliser `http://localhost:9000` (appel fait
  par le navigateur) et non le nom du service Docker.
- **Noms d'images avec Compose** : Compose préfixe les images avec le nom du
  dossier (`mern-app-server`, `mern-app-client`).

## Partie 3 : Projet micro-services

(à compléter)