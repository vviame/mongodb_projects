# Réplication MongoDB avec Docker

## Description

Ce projet met en œuvre un **jeu de réplicas MongoDB à trois membres** (`rs0`) avec Docker Compose, et fournit un **tableau de bord web** pour observer son fonctionnement en direct : rôles des membres, élection d'un nouveau primaire après une panne, et réplication des données.

Le rapport complet (simulations de réplication et de partitionnement) se trouve dans `Rapport_MongoDB_Replication_Sharding.docx`.

## Technologies utilisées

* MongoDB 8
* Docker et Docker Compose
* FastAPI et PyMongo (backend)
* HTML, CSS et JavaScript (frontend)
* Nginx

## Structure du projet

```text
mongodb_project/
├── backend/
│   ├── main.py
│   └── Dockerfile
├── frontend/
│   ├── index.html
│   ├── nginx.conf
│   └── Dockerfile
├── the_rest/
│   ├── replica4/docker-compose.yml
│   └── sharding/docker-compose.yml
├── docker-compose.yml
├── Rapport_MongoDB_Replication_Sharding.docx
└── README.md
```

## Architecture Docker

| Service    | Rôle                                   | Port (poste)  |
| ---------- | -------------------------------------- | ------------- |
| `mongo1`   | Membre du jeu de réplicas `rs0`        | 27017         |
| `mongo2`   | Membre du jeu de réplicas `rs0`        | 27018         |
| `mongo3`   | Membre du jeu de réplicas `rs0`        | 27019         |
| `backend`  | API FastAPI qui interroge MongoDB      | (interne)     |
| `frontend` | Tableau de bord + proxy vers l'API     | 8080          |

Le backend s'exécute **dans le réseau Docker** : les membres sont enregistrés sous les noms `mongo1:27017`, `mongo2:27017` et `mongo3:27017`, que seuls les conteneurs peuvent résoudre. Le navigateur passe par Nginx, qui redirige `/api/` vers le backend.

Chaque instance conserve ses données dans un volume nommé (`mongo1_data`, `mongo2_data`, `mongo3_data`).

## Lancement du projet

Avant de commencer, vérifier que **Docker Desktop** est lancé. Dans le dossier `mongodb_project/` :

```bash
docker compose up -d --build
```

Ouvrir ensuite le tableau de bord : **http://localhost:8080**

### Initialiser le jeu de réplicas

Au premier lancement, le jeu de réplicas n'existe pas encore. Le tableau de bord affiche alors un bouton **Initialiser le jeu de réplicas** (« Initialise replica set ») qui exécute `rs.initiate()` avec les trois membres. Attendre quelques secondes la première élection.

Il est aussi possible de le faire manuellement avec `mongosh` :

```bash
docker exec -it mongo1 mongosh
```

```js
rs.initiate({
  _id: "rs0",
  members: [
    { _id: 0, host: "mongo1:27017" },
    { _id: 1, host: "mongo2:27017" },
    { _id: 2, host: "mongo3:27017" }
  ]
})
```

## Tableau de bord

* **Membres** : une carte par instance avec son rôle (primaire, secondaire ou injoignable), son port, sa durée de fonctionnement et son retard de réplication. Actualisation toutes les 2 secondes.
* **Résumé** : le primaire actuel, le numéro de terme de l'élection, et l'état des écritures (acceptées ou refusées).
* **Démonstration de réplication** : chaque commande ajoutée est écrite sur le primaire, puis relue directement sur chaque membre. Les compteurs doivent être identiques.

## Reproduire les simulations du rapport

Arrêter le primaire et observer l'élection :

```bash
docker stop mongo1
```

Un des deux membres restants devient primaire. Redémarrer l'ancien primaire :

```bash
docker start mongo1
```

Il revient en tant que **secondaire**.

Arrêter deux membres sur trois :

```bash
docker stop mongo2 mongo3
```

Il n'y a plus de majorité : aucun primaire n'est élu et les écritures sont refusées.

## API du backend

Accessible via Nginx, par exemple `http://localhost:8080/api/status`.

| Méthode  | Chemin         | Description                                            |
| -------- | -------------- | ------------------------------------------------------ |
| `GET`    | `/health`      | Vérifie que le backend fonctionne                      |
| `GET`    | `/status`      | État du jeu de réplicas (membres, primaire, terme)     |
| `POST`   | `/init`        | Initialise le jeu de réplicas `rs0`                    |
| `POST`   | `/orders`      | Ajoute une commande (`product`, `quantity`)            |
| `DELETE` | `/orders`      | Supprime toutes les commandes                          |
| `GET`    | `/replication` | Contenu de la collection lu sur chaque membre          |

Les commandes sont stockées dans la base `shop`, collection `orders`.

## Partitionnement (sharding)

Les dossiers `the_rest/replica4` et `the_rest/sharding` contiennent les configurations utilisées pour le jeu à quatre membres et la simulation de partitionnement. Elles ne sont **pas** couvertes par le tableau de bord et utilisent les mêmes noms de conteneurs que le projet principal : ne pas les lancer en même temps.

## Arrêt du projet

Arrêter les conteneurs en conservant les données :

```bash
docker compose down
```

Tout supprimer, y compris les volumes (le jeu de réplicas devra être réinitialisé) :

```bash
docker compose down -v
```

## Conclusion

Ce projet montre comment une base MongoDB reste disponible malgré la panne d'un membre grâce à la réplication, et comment Docker Compose permet de reproduire cet environnement avec une seule commande :

```bash
docker compose up -d --build
```
