# TP – Réplication MongoDB
Module : Bases de données NoSQL  
Sujet : TPReplicationMongoDb  

---

## Introduction

Dans les systèmes de bases de données modernes, la disponibilité et la fiabilité des données sont des enjeux majeurs. Une panne matérielle, logicielle ou réseau peut entraîner une indisponibilité du service, voire une perte de données. MongoDB répond à ces problématiques grâce au mécanisme des **Replica Sets**, qui permettent la réplication automatique des données et la tolérance aux pannes.

Ce TP a pour objectif de comprendre le fonctionnement des Replica Sets, d’étudier les rôles des différents nœuds, de mettre en place une architecture répliquée et d’analyser les comportements du système en cas de panne ou de bascule.

---

## Principe de la réplication dans MongoDB

La réplication dans MongoDB repose sur un **Replica Set**, c’est-à-dire un groupe de processus `mongod` qui maintiennent une copie synchronisée des mêmes données.

Un Replica Set est composé de :
- **un nœud Primary** ;
- **un ou plusieurs nœuds Secondaries** ;
- éventuellement **un arbitre (Arbiter)**.

Les opérations d’écriture sont centralisées sur le Primary. Chaque écriture est enregistrée dans un journal appelé **oplog** (operations log). Les nœuds Secondaries lisent cet oplog et rejouent les opérations dans le même ordre afin de maintenir une copie cohérente de la base de données.

La réplication dans MongoDB est **asynchrone**, ce qui signifie qu’un léger retard peut exister entre le Primary et les Secondaries.

---

## Rôles des nœuds

### Le Primary

Le Primary est le seul nœud autorisé à recevoir les opérations d’écriture. Il garantit une source unique de vérité, évitant ainsi les conflits de données. Par défaut, les lectures sont également dirigées vers le Primary afin d’assurer une cohérence maximale.

### Les Secondaries

Les Secondaries répliquent les données du Primary à partir de l’oplog. Ils peuvent :
- devenir Primary lors d’une élection ;
- servir des lectures si la configuration du client l’autorise.

Ils jouent un rôle essentiel dans la tolérance aux pannes et la haute disponibilité.

### L’Arbitre

Un arbitre est un nœud qui ne stocke aucune donnée. Il participe uniquement aux votes lors des élections. Son rôle principal est de permettre l’obtention d’une majorité lorsqu’on souhaite conserver un nombre impair de votes sans ajouter un nœud de données supplémentaire.

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

Cette commande permet d’identifier le rôle de chaque nœud (PRIMARY, SECONDARY, ARBITER) et l’état global du cluster.

---

## Élections et tolérance aux pannes

Lorsqu’un Primary devient indisponible, les membres du Replica Set déclenchent automatiquement une **élection**. Pour qu’un nouveau Primary soit élu, une **majorité de votes** doit être atteinte.

Les critères principaux utilisés lors d’une élection sont :
- la disponibilité du nœud ;
- sa priorité ;
- la fraîcheur de son oplog.

Si aucune majorité n’est atteinte, le Replica Set reste sans Primary. Dans ce cas, les écritures sont impossibles, mais certaines lectures peuvent rester disponibles selon la configuration.

---

## Lecture et cohérence des données

MongoDB propose plusieurs options pour contrôler la cohérence :

- **readPreference** : définit vers quel nœud les lectures sont envoyées (primary, secondary, etc.) ;
- **readConcern** : définit le niveau de cohérence des données lues ;
- **writeConcern** : définit le niveau d’accusé de réception requis pour une écriture.

Pour une cohérence forte, il est recommandé d’utiliser :
- `readPreference: "primary"` ;
- `readConcern: "majority"` ;
- `writeConcern: "majority"`.

---

## Scénarios pratiques

### Forcer une bascule du Primary

```javascript
rs.stepDown(60)
```

Cette commande force le Primary à se retirer temporairement, déclenchant une nouvelle élection.

### Ajouter un nœud secondaire

```javascript
rs.add("<host_ip>:<port>")
```

### Retirer un nœud défectueux

```javascript
rs.remove("<host_ip>:<port>")
```

### Configurer un Secondary avec un délai de réplication

```javascript
rs.add({
  host: "mongo-delayed:27017",
  priority: 0,
  hidden: true,
  slaveDelay: 120
})
```

Ce type de nœud permet de conserver une copie retardée des données, utile pour la récupération après erreur humaine.

---

## Surveillance de la réplication

MongoDB fournit plusieurs commandes pour surveiller l’état de la réplication :

```javascript
rs.printSlaveReplicationInfo()
db.printReplicationInfo()
```

Les logs MongoDB permettent également d’observer en temps réel les événements liés aux élections et à la réplication.

---

## Conclusion

Ce TP a permis d’acquérir une compréhension approfondie du mécanisme de réplication de MongoDB. Les Replica Sets constituent un élément central pour assurer la haute disponibilité et la tolérance aux pannes. Une bonne configuration des nœuds, des priorités et des options de cohérence est essentielle pour répondre aux exigences des applications critiques en production.

---

## Références

- Documentation officielle MongoDB : https://www.mongodb.com/docs/manual
