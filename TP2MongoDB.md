# TP – Réplication MongoDB
Module : Bases de données NoSQL  
Sujet : TPReplicationMongoDb  

---

## Introduction

L’objectif de ce TP est d’étudier le mécanisme de réplication proposé par MongoDB afin d’assurer la haute disponibilité, la tolérance aux pannes et la fiabilité des données. MongoDB utilise pour cela le concept de **Replica Set**, qui permet de répliquer automatiquement les données sur plusieurs nœuds.

Ce TP s’appuie sur :
- le sujet officiel fourni,
- deux vidéos explicatives sur la réplication MongoDB,
- la documentation officielle MongoDB,
- une mise en œuvre pratique à l’aide de Docker.

Le travail réalisé vise à comprendre le fonctionnement interne des Replica Sets, les rôles des différents nœuds, les mécanismes d’élection et les choix de configuration possibles en environnement réel.

---

## Présentation des vidéos

### Vidéo 1 – Introduction à la réplication MongoDB

Cette vidéo présente les bases de la réplication dans MongoDB. Elle explique pourquoi la réplication est indispensable dans un système de base de données moderne, notamment pour garantir la disponibilité des données en cas de panne.

Les points clés abordés sont :
- la notion de Replica Set ;
- le rôle central du Primary, unique nœud autorisé à recevoir les écritures ;
- le fonctionnement des Secondaries, qui répliquent les données à partir du journal des opérations (oplog) ;
- le principe d’élection automatique lorsqu’un Primary devient indisponible.

La vidéo insiste sur le fait que MongoDB privilégie la cohérence et évite les conflits en imposant un seul point d’écriture.

---

### Vidéo 2 – Fonctionnement avancé et bascule automatique

La deuxième vidéo approfondit les mécanismes internes des Replica Sets. Elle explique le processus d’élection, la notion de majorité et le rôle des arbitres.

Les notions importantes présentées sont :
- la réplication asynchrone via l’oplog ;
- les critères de sélection d’un nouveau Primary (priorité, fraîcheur de l’oplog) ;
- le rôle des arbitres pour maintenir un nombre impair de votes ;
- l’impact d’une partition réseau et la prévention du split-brain.

Cette vidéo met en évidence l’importance d’une bonne configuration pour garantir la résilience du système.

---

## Mise en place d’un Replica Set avec Docker

### Création du réseau Docker

```bash
docker network create mongo-cluster
```

### Lancement des instances MongoDB

```bash
docker run -d --name mongo1 --net mongo-cluster -p 27017:27017 mongo --replSet rs0 --bind_ip_all
docker run -d --name mongo2 --net mongo-cluster -p 27018:27017 mongo --replSet rs0 --bind_ip_all
docker run -d --name mongo3 --net mongo-cluster -p 27019:27017 mongo --replSet rs0 --bind_ip_all
```

### Initialisation du Replica Set

```bash
docker exec -it mongo1 mongosh
```

```javascript
rs.initiate({
  _id: "rs0",
  members: [
    { _id: 0, host: "<host_ip>:27017" },
    { _id: 1, host: "<host_ip>:27018" },
    { _id: 2, host: "<host_ip>:27019" }
  ]
})
```

### Vérification de l’état du Replica Set

```javascript
rs.status()
```

---

## Questions – Réponses

### 1. Qu’est-ce qu’un Replica Set ?
Un Replica Set est un groupe de processus mongod qui maintiennent la même base de données en répliquant les opérations à partir d’un journal appelé oplog.

### 2. Rôle du Primary
Le Primary reçoit toutes les opérations d’écriture et les enregistre dans l’oplog.

### 3. Rôle des Secondaries
Les Secondaries répliquent les opérations du Primary et peuvent être élus Primary en cas de panne.

### 4. Pourquoi les écritures sont interdites sur les Secondaries ?
Pour éviter les conflits et garantir une source unique de vérité.

### 5. Cohérence forte
La cohérence forte garantit que les lectures reflètent les écritures validées par une majorité des nœuds.

### 6. Différence entre readPreference primary et secondary
Primary garantit des données à jour, secondary permet de décharger le Primary mais avec un risque de retard.

### 7. Lecture sur un Secondary
Utile pour les statistiques, rapports ou sauvegardes où un léger retard est acceptable.

### 8. Initialisation d’un Replica Set
```javascript
rs.initiate()
```

### 9. Ajouter un nœud
```javascript
rs.add("<host>:port")
```

### 10. Afficher l’état du Replica Set
```javascript
rs.status()
```

### 11. Identifier le rôle d’un nœud
```javascript
db.isMaster()
```

### 12. Forcer la bascule du Primary
```javascript
rs.stepDown(60)
```

### 13. Arbitre
Un arbitre participe aux votes mais ne stocke pas de données.

### 14. Secondary avec slaveDelay
```javascript
rs.add({
  host: "mongo-delayed:27017",
  priority: 0,
  hidden: true,
  slaveDelay: 120
})
```

### 15. Panne sans majorité
Aucun Primary n’est élu, les écritures échouent.

### 16. Choix du nouveau Primary
Basé sur la priorité et la fraîcheur de l’oplog.

### 17. Élection
Processus par lequel un nouveau Primary est choisi.

### 18. Auto-dégradation
Un Primary se dégrade en Secondary s’il perd la majorité.

### 19. Nombre impair de nœuds
Facilite l’obtention d’une majorité.

### 20. Partition réseau
Seule la partition majoritaire reste opérationnelle en écriture.

---

## Conclusion

Ce TP a permis de comprendre en profondeur le fonctionnement de la réplication dans MongoDB. Les Replica Sets constituent un mécanisme robuste pour assurer la haute disponibilité et la tolérance aux pannes, à condition d’être correctement configurés. La compréhension des élections, des rôles des nœuds et des options de cohérence est essentielle pour concevoir des architectures MongoDB fiables en production.

---

## Références

- Documentation officielle MongoDB : https://www.mongodb.com/docs/manual
- Vidéos pédagogiques sur la réplication MongoDB
