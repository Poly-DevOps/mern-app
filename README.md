# MERN APP — Docker & Docker Compose

## 1. Vue d'ensemble

Ce projet est une application full-stack MERN composée d'un frontend React, d'un backend Node.js avec Express et d'une base de données MongoDB.

L'application est conteneurisée avec Docker et orchestrée avec Docker Compose. Le frontend utilise une **construction multi-stage** : Node.js sert à construire l'application React, puis Nginx sert les fichiers statiques en production.

L'architecture utilise un réseau Docker dédié pour la communication entre les services et un volume nommé pour assurer la persistance des données MongoDB.

## 2. Technologies utilisées

- **Frontend :** React
- **Serveur Web Frontend :** Nginx
- **Backend :** Node.js, Express
- **Base de données :** MongoDB
- **Conteneurisation :** Docker
- **Orchestration :** Docker Compose
- **Réseau :** Docker Bridge Network
- **Persistance :** Docker Named Volume

## 3. Architecture du projet

```text
mern-app-devops/
├── client/
│   ├── Dockerfile
│   ├── nginx.conf
│   ├── .dockerignore
│   ├── package.json
│   └── ...
├── server/
│   ├── Dockerfile
│   ├── .dockerignore
│   ├── package.json
│   ├── server.js
│   ├── db/
│   └── routes/
├── docker-compose.yml
├── .gitignore
└── README.md
```

## 4. Configuration des services

L'application comprend trois services Docker.

| Service | Technologie | Port accessible depuis l'hôte |
|---|---|---|
| Client | React + Nginx | 3000 |
| Serveur | Node.js + Express | 9000 |
| Base de données | MongoDB | Accessible via le réseau Docker |

### 4.1 Frontend — React et Nginx

Le Dockerfile du client utilise deux étapes :

1. **Build :** Node.js installe les dépendances et génère les fichiers de production avec `npm run build`.
2. **Production :** Nginx sert les fichiers générés depuis `/usr/share/nginx/html`.

La configuration `client/nginx.conf` permet également de rediriger les routes frontend vers `index.html`, ce qui est nécessaire pour une application React utilisant le routage côté client.

Le frontend est accessible à l'adresse :

http://localhost:3000

### 4.2 Backend — Node.js et Express

Le serveur Express écoute sur le port `9000`. Il expose notamment une route de vérification à la racine `/`.

Le serveur utilise la variable d'environnement `MONGO_URI` pour se connecter à MongoDB.

L'API est accessible à l'adresse :

http://localhost:9000

### 4.3 Base de données — MongoDB

MongoDB utilise un volume Docker nommé `mern-data`, monté sur `/data/db`.

Ce volume permet de conserver les données de la base de données lorsque le conteneur est recréé, à condition de ne pas supprimer le volume lui-même.

## 5. Variables d'environnement

| Variable | Utilisation | Valeur locale |
|---|---|---|
| `REACT_APP_API_URL` | URL de l'API utilisée par le frontend | `http://localhost:9000` |
| `MONGO_URI` | URI de connexion MongoDB du backend | `mongodb://mongodb:27017/todos` |

**Remarques :**

- `REACT_APP_API_URL` est injectée pendant la construction de React. Si le code frontend utilise cette variable, sa valeur est intégrée au bundle généré.
- `MONGO_URI` utilise le nom du service `mongodb`, résolu automatiquement par le DNS du réseau Docker.
- Le fichier `server/.env` n'est pas copié dans l'image Docker. La variable `MONGO_URI` est fournie par Docker Compose.
- Ne publiez jamais de mots de passe ou de secrets dans le dépôt Git.

## 6. Configuration Docker

### 6.1 Dockerfiles

Le projet contient deux Dockerfiles :

- `client/Dockerfile` : construction multi-stage avec Node.js et exécution avec Nginx.
- `server/Dockerfile` : installation des dépendances Node.js et démarrage du serveur Express.

Les fichiers `.dockerignore` permettent d'exclure les éléments inutiles du contexte de construction, notamment `node_modules`, les fichiers `.env` et les fichiers générés.

### 6.2 Réseau Docker

Le réseau `mern-network` permet aux conteneurs de communiquer entre eux.

Le backend se connecte à MongoDB en utilisant le nom d'hôte `mongodb`, sans utiliser `localhost` à l'intérieur du conteneur.

### 6.3 Volume persistant

Le volume nommé `mern-data` conserve les données MongoDB indépendamment du cycle de vie du conteneur.

### 6.4 Healthcheck et ordre de démarrage

Docker Compose vérifie la disponibilité de MongoDB avec un `healthcheck`.

Le backend utilise `depends_on` avec `condition: service_healthy` pour attendre que MongoDB soit opérationnel avant son démarrage.

Les services utilisent la politique de redémarrage `unless-stopped`.

## 7. Comment exécuter le projet

### Prérequis

- Docker installé
- Docker Compose disponible via `docker compose`
- Git installé pour cloner le dépôt

### Étape 1 — Cloner le dépôt

```bash
git clone <URL_DU_DEPOT>
cd mern-app-devops
```

Remplacez `<URL_DU_DEPOT>` par l'URL réelle de votre dépôt Git.

### Étape 2 — Construire et démarrer l'application

```bash
docker compose up -d --build
```

Cette commande construit les images et démarre les trois services en arrière-plan.

### Étape 3 — Vérifier les conteneurs

```bash
docker compose ps
```

MongoDB doit finir par afficher l'état `healthy`.

### Étape 4 — Consulter les logs

```bash
docker compose logs
```

Pour consulter les logs d'un service en particulier :

```bash
docker compose logs server
docker compose logs client
docker compose logs mongodb
```

### Étape 5 — Accéder à l'application

- Frontend : http://localhost:3000
- Backend : http://localhost:9000

### Étape 6 — Arrêter les services

```bash
docker compose down
```

Cette commande arrête et supprime les conteneurs et le réseau créé par Compose, mais conserve le volume nommé `mern-data`.

Pour supprimer également le volume et les données MongoDB, il faudrait exécuter explicitement :

```bash
docker compose down -v
```

**Attention :** cette dernière commande supprime les volumes gérés par Compose et peut entraîner une perte des données persistées.

## 8. Commandes utiles pour les tests

Vérifier la réponse du frontend :

```bash
curl -I http://localhost:3000
```

Vérifier la réponse du backend :

```bash
curl http://localhost:9000/
```

Vérifier l'état de santé de MongoDB :

```bash
docker inspect mongodb --format '{{.State.Health.Status}}'
```

Vérifier le volume de données :

```bash
docker volume inspect mern-data
```

Vérifier le réseau :

```bash
docker network inspect mern-network
```

## 9. Résultats attendus

Après le démarrage avec Docker Compose :

- Les trois conteneurs sont actifs.
- MongoDB devient `healthy`.
- Le frontend React est servi par Nginx.
- Le backend Express est accessible sur le port `9000`.
- Le backend peut joindre MongoDB via le réseau Docker.
- Les données MongoDB sont stockées dans le volume `mern-data`.

## 10. Conclusion

Ce projet met en pratique les principes de conteneurisation et d'orchestration d'une application full-stack avec Docker et Docker Compose.

Il utilise une construction multi-stage pour le frontend, un réseau dédié pour la communication entre services, un volume nommé pour la persistance des données et un contrôle de santé pour améliorer l'ordre de démarrage des conteneurs.
