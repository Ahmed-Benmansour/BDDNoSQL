# TP – Réplication MongoDB
**Module : Bases de données NoSQL**  
**Sujet : TPReplicationMongoDb**

---

## Introduction générale

La réplication est un mécanisme essentiel dans les bases de données distribuées afin de garantir la disponibilité du service, la tolérance aux pannes et la fiabilité des données. MongoDB implémente la réplication à travers les *Replica Sets*, qui constituent la solution native de haute disponibilité du système.

Un Replica Set permet de maintenir plusieurs copies synchronisées des mêmes données sur différents nœuds. En cas de défaillance d’un serveur, le système est capable d’élire automatiquement un nouveau nœud principal et de continuer à fonctionner sans interruption majeure. Cette approche est particulièrement adaptée aux applications critiques nécessitant une continuité de service.

Ce TP a pour objectif de comprendre le fonctionnement des Replica Sets, d’analyser les rôles des différents nœuds, de mettre en œuvre une réplication avec MongoDB et d’étudier le comportement du système dans différents scénarios de panne.

---

## Fonctionnement des Replica Sets MongoDB

Un Replica Set est composé de plusieurs processus `mongod` qui partagent la même base de données. L’architecture repose sur un modèle à maître unique, garantissant une cohérence forte des écritures.

Chaque Replica Set contient :
- un **Primary**, qui reçoit toutes les écritures ;
- un ou plusieurs **Secondaries**, qui répliquent les données ;
- éventuellement un **Arbitre**, qui participe aux votes sans stocker de données.

Les écritures sont d’abord appliquées sur le Primary, puis enregistrées dans un journal appelé **oplog**. Les Secondaries lisent cet oplog et rejouent les opérations dans le même ordre. Cette réplication est asynchrone, ce qui peut entraîner un léger décalage temporel entre les nœuds.

MongoDB intègre également des mécanismes d’élection automatique. Lorsqu’un Primary devient indisponible, une élection est déclenchée afin de désigner un nouveau Primary parmi les Secondaries disponibles, à condition qu’une majorité de votes soit atteinte.

---

## Mise en place d’un Replica Set avec Docker

### Création du réseau Docker

```bash
docker network create mongo-cluster
```

### Lancement des conteneurs MongoDB

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

## Questions et réponses

### Partie 1 — Compréhension de base

**1. Qu’est-ce qu’un Replica Set dans MongoDB ?**  
Un Replica Set est un ensemble de processus `mongod` qui répliquent les données d’une même base à partir de l’oplog afin d’assurer la haute disponibilité et la tolérance aux pannes.

**2. Quel est le rôle du Primary ?**  
Le Primary reçoit toutes les écritures, les applique aux données et les enregistre dans l’oplog.

**3. Quel est le rôle des Secondaries ?**  
Les Secondaries répliquent les données du Primary et peuvent être élus Primary en cas de panne.

**4. Pourquoi MongoDB interdit les écritures sur un Secondary ?**  
Pour éviter les conflits de données et garantir une source unique de vérité.

**5. Qu’est-ce que la cohérence forte dans MongoDB ?**  
Elle garantit que les lectures reflètent les écritures validées par une majorité des nœuds.

**6. Différence entre readPreference primary et secondary ?**  
Lire sur le Primary garantit des données à jour, tandis que lire sur un Secondary permet de répartir la charge avec un risque de retard.

**7. Dans quel cas lire sur un Secondary ?**  
Pour des usages non critiques comme les statistiques, rapports ou sauvegardes.

---

### Partie 2 — Commandes et configuration

**8. Quelle commande initialise un Replica Set ?**  
```javascript
rs.initiate()
```

**9. Comment ajouter un nœud après l’initialisation ?**  
```javascript
rs.add("<host>:<port>")
```

**10. Quelle commande affiche l’état du Replica Set ?**  
```javascript
rs.status()
```

**11. Comment identifier le rôle d’un nœud ?**  
Avec `db.isMaster()` ou `rs.status()`.

**12. Quelle commande force la bascule du Primary ?**  
```javascript
rs.stepDown(60)
```

---

### Partie 3 — Résilience et tolérance aux pannes

**13. Comment désigner un Arbitre et pourquoi ?**  
Avec `rs.addArb()`, pour obtenir une majorité sans ajouter un nœud de données.

**14. Comment configurer un Secondary avec un délai de réplication ?**  
```javascript
rs.add({
  host: "mongo-delayed:27017",
  priority: 0,
  hidden: true,
  slaveDelay: 120
})
```

**15. Que se passe-t-il si le Primary tombe sans majorité ?**  
Aucun Primary n’est élu et les écritures sont bloquées.

**16. Comment MongoDB choisit-il un nouveau Primary ?**  
Selon la priorité, la disponibilité et la fraîcheur de l’oplog.

**17. Qu’est-ce qu’une élection ?**  
Un processus automatique de vote pour élire un nouveau Primary.

**18. Que signifie l’auto-dégradation ?**  
Un Primary se dégrade en Secondary s’il perd la majorité.

**19. Pourquoi un nombre impair de nœuds ?**  
Pour faciliter l’obtention d’une majorité.

**20. Conséquences d’une partition réseau ?**  
Seule la partition majoritaire conserve un Primary.

---

### Partie 4 — Scénarios pratiques

**21. Panne du Primary avec un arbitre présent**  
Le Secondary restant est élu Primary grâce à la majorité.

**22. Utilité d’un Secondary avec slaveDelay**  
Permet de récupérer des données après une erreur humaine.

**23. Lecture toujours à jour**  
`readPreference: primary`, `readConcern: majority`, `writeConcern: majority`.

**24. Écriture confirmée par au moins deux nœuds**  
```javascript
writeConcern: { w: 2 }
```

**25. Lecture obsolète depuis un Secondary**  
Due à la réplication asynchrone.

**26. Vérifier le Primary actuel**  
```javascript
rs.status()
```

**27. Forcer une bascule manuelle**  
```javascript
rs.stepDown(60)
```

**28. Ajouter un nouveau Secondary**  
```javascript
rs.add("<host>:<port>")
```

**29. Retirer un nœud défectueux**  
```javascript
rs.remove("<host>:<port>")
```

**30. Configurer un Secondary caché**  
Définir `hidden: true` et `priority: 0`.

**31. Modifier la priorité d’un nœud**  
```javascript
cfg = rs.conf()
cfg.members[0].priority = 10
rs.reconfig(cfg)
```

**32. Vérifier le retard de réplication**  
```javascript
rs.printSlaveReplicationInfo()
```

---

### Questions complémentaires

**33. Que fait rs.freeze() ?**  
Empêche temporairement un nœud de devenir Primary.

**34. Comment redémarrer sans perdre la configuration ?**  
La configuration est stockée dans la base `local`.

**35. Comment surveiller la réplication ?**  
Avec `rs.status()`, `rs.printSlaveReplicationInfo()` et les logs.

**37. Qu’est-ce qu’un Arbitre ?**  
Un nœud votant sans données.

**38. Vérifier la latence de réplication**  
```javascript
rs.printSlaveReplicationInfo()
```

**39. Afficher le retard de réplication**  
```javascript
rs.printSlaveReplicationInfo()
```

**40. Réplication synchrone vs asynchrone**  
MongoDB utilise une réplication asynchrone.

**41. Peut-on modifier la configuration sans redémarrer ?**  
Oui, avec `rs.reconfig()`.

**42. Secondary très en retard**  
Il ne peut pas être élu Primary.

**43. Gestion des conflits de données**  
Ils sont évités par l’unicité du Primary.

**44. Peut-on avoir plusieurs Primary ?**  
Non, grâce au mécanisme de majorité.

**45. Pourquoi éviter les écritures sur un Secondary ?**  
Elles seraient écrasées par la réplication.

**46. Conséquences d’un réseau instable**  
Élections fréquentes et dégradation des performances.

---

## Conclusion

Ce TP a permis de comprendre en profondeur le fonctionnement des Replica Sets MongoDB. Ces mécanismes sont indispensables pour concevoir des systèmes distribués fiables, capables de résister aux pannes tout en garantissant la cohérence des données.

---

## Références

- https://www.mongodb.com/docs/manual
