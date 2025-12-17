# TP3 — Partitionnement (Sharding) sous MongoDB

---

## 1) Objectif du TP

L’objectif est de **mettre en place un cluster MongoDB shardé** composé de :

- **Config Server** : stocke les *métadonnées* du sharding (localisation des chunks, configuration du cluster)
- **Routeur `mongos`** : point d’entrée client, **route** les requêtes vers le(s) shard(s) concerné(s)
- **Deux shards** : serveurs de données (idéalement sous forme de *replica sets*)

Puis :
1. activer le sharding sur une base,
2. sharder une collection,
3. insérer un grand volume de documents (script Python),
4. **observer la création/splitting/migration des chunks** et la répartition des données.

---

## 2) Pré-requis

- MongoDB installé (mongod, mongos, mongosh)
- Python 3 + `pymongo`
- 6 terminaux (recommandé) : `configsvr`, `mongos`, `shard1`, `shard2`, `client-init`, `client`

---

## 3) Mise en place du cluster (mode “terminaux”)

### 3.1) Créer les répertoires de données

```bash
mkdir -p configsvrdb
mkdir -p serv1
mkdir -p serv2
```

---

### 3.2) Démarrer le **Config Server** (port 27019)

Terminal **configsvr** :

```bash
mongod --configsvr --replSet replicaconfig --dbpath configsvrdb --port 27019
```

Initialisation du replica set (terminal **client-init**) :

```bash
mongosh --port 27019
```

Dans `mongosh` :

```js
rs.initiate({
  _id: "replicaconfig",
  members: [{ _id: 0, host: "localhost:27019" }]
})
```

Vérification :

```js
rs.status()
```

---

### 3.3) Démarrer les **shards** (ports 20004 et 20005)

Terminal **shard1** :

```bash
mongod --replSet replicashard1 --dbpath serv1 --shardsvr --port 20004
```

Terminal **shard2** :

```bash
mongod --replSet replicashard2 --dbpath serv2 --shardsvr --port 20005
```

Initialiser chaque replica set :

**Shard1 :**

```bash
mongosh --port 20004
```

```js
rs.initiate({
  _id: "replicashard1",
  members: [{ _id: 0, host: "localhost:20004" }]
})
```

**Shard2 :**

```bash
mongosh --port 20005
```

```js
rs.initiate({
  _id: "replicashard2",
  members: [{ _id: 0, host: "localhost:20005" }]
})
```

---

### 3.4) Démarrer le routeur **mongos**

Deux possibilités :

#### Option A (recommandée si ton script Python utilise `localhost:27017`)
Lancer `mongos` **sur le port 27017** (port MongoDB “standard”) :

```bash
mongos --configdb replicaconfig/localhost:27019 --port 27017
```

#### Option B
Lancer `mongos` sur un autre port (ex. 27020), puis **adapter le script Python** :

```bash
mongos --configdb replicaconfig/localhost:27019 --port 27020
```

---

### 3.5) Ajouter les shards au cluster via `mongos`

Connexion au routeur :

- Option A (mongos sur 27017) :

```bash
mongosh --port 27017
```

- Option B (mongos sur 27020) :

```bash
mongosh --port 27020
```

Ajouter les shards :

```js
sh.addShard("replicashard1/localhost:20004")
sh.addShard("replicashard2/localhost:20005")
```

Vérifier :

```js
sh.status()
```

---

## 4) Activer le sharding sur la base et la collection

On utilise la base et collection du sujet :

- Base : `mabasefilms`
- Collection : `films`

Dans `mongosh` connecté à `mongos` :

```js
sh.enableSharding("mabasefilms")
```

Shard de la collection (clé : `titre`) :

```js
sh.shardCollection("mabasefilms.films", { "titre": 1 })
```

> Remarque : MongoDB exigera généralement un index commençant par la shard key. Si nécessaire :
```js
use mabasefilms
db.films.createIndex({ titre: 1 })
```

---

## 5) Insertion des données (script Python)

### 5.1) Script fourni (insertion “un par un”)

Le script `monappunparun.py` insère des films générés aléatoirement, **document par document**.

Points importants :
- DB : `mabasefilms`
- Collection : `films`
- Connexion : `MongoClient("mongodb://localhost:27017/")`

➡️ Donc, si tu lances `mongos` sur **27017**, tu peux exécuter le script sans modification.

### 5.2) Exécution

```bash
python3 monappunparun.py
```

---

## 6) Observation / vérification du sharding

### 6.1) Vérifier l’état du cluster

```js
sh.status()
```

### 6.2) Vérifier la distribution des données

Dans `mongosh` :

```js
use mabasefilms
db.films.getShardDistribution()
```

### 6.3) Voir les chunks dans la base `config`

```js
use config
db.chunks.find({ ns: "mabasefilms.films" }).pretty()
```

### 6.4) Surveiller le balancer

```js
sh.getBalancerState()
sh.isBalancerRunning()
```

---

## 7) Analyse : ce qu’on observe en général

- Au début, la collection shardée possède **un chunk initial** sur un shard.
- Quand la taille du chunk dépasse un seuil, MongoDB effectue un **split** (création de 2 chunks plus petits).
- Le **balancer** surveille l’équilibre entre shards (chunks) et peut déclencher des **migrations de chunks** vers l’autre shard.
- Avec une shard key mal choisie (ex. monotone), on peut créer un **hot shard** (un shard reçoit presque toutes les écritures).

---

# 8) Questions / Réponses

## 1. Qu’est-ce que le sharding dans MongoDB et pourquoi est-il utilisé ?
Le sharding est le **partitionnement horizontal** d’une collection : les documents sont répartis sur plusieurs serveurs (shards) selon une **clé de sharding**. Il est utilisé pour la **scalabilité horizontale** (gérer de gros volumes et beaucoup de requêtes) en répartissant stockage et charge.

## 2. Différence entre sharding et réplication ?
- **Réplication** : copie des *mêmes* données sur plusieurs nœuds (haute dispo + tolérance aux pannes).
- **Sharding** : découpe des données en *fragments différents* répartis sur plusieurs nœuds (montée en charge).

## 3. Composants d’une architecture shardée ?
- **Shards** (souvent des replica sets) : stockent les données
- **Config servers (CSRS)** : métadonnées (chunks, placements, config)
- **mongos** : routeur/point d’entrée client

## 4. Responsabilités des config servers (CSRS) ?
Ils conservent les **métadonnées de sharding** : mapping chunks→shards, configuration du cluster, informations nécessaires au routage.

## 5. Rôle du routeur mongos ?
`mongos` reçoit les requêtes, consulte les config servers, **route** vers le(s) shard(s) concerné(s), puis **fusionne** les résultats si plusieurs shards ont répondu.

## 6. Comment MongoDB choisit le shard d’un document ?
À partir de la **shard key** : la valeur de la clé place le document dans un **chunk** (plage de valeurs), et ce chunk est stocké sur un shard donné.

## 7. Qu’est-ce qu’une clé de sharding et pourquoi essentielle ?
C’est le champ (ou ensemble de champs) utilisé pour répartir les documents. Elle est essentielle car elle détermine : distribution, équilibre, performances, ciblage des requêtes.

## 8. Critères d’une bonne clé de sharding ?
- **Cardinalité élevée** (beaucoup de valeurs distinctes)
- **Distribution uniforme** des insertions/accès
- Favorise des **requêtes ciblées** (incluse dans les filtres fréquents)
- Évite une progression **monotone** (risque hot shard)

## 9. Qu’est-ce qu’un chunk ?
Un chunk est une **plage contiguë** de valeurs de shard key représentant un sous-ensemble de documents, stockée sur un shard.

## 10. Comment fonctionne le splitting des chunks ?
Quand un chunk grossit au-delà d’un seuil, MongoDB le **divise** en deux chunks plus petits (processus automatique en arrière-plan).

## 11. Que fait le balancer ?
Le balancer maintient une **répartition équilibrée** des chunks entre shards, en déclenchant des migrations.

## 12. Quand et comment le balancer déplace-t-il des chunks ?
Quand il détecte un déséquilibre (un shard a beaucoup plus de chunks qu’un autre), il migre des chunks du shard “chargé” vers le shard “moins chargé”, en arrière-plan.

## 13. Qu’est-ce qu’un hot shard et comment l’éviter ?
Un hot shard est un shard surchargé (beaucoup d’écritures/lectures). On l’évite en choisissant une shard key **non monotone**, bien distribuée, ou en utilisant une shard key **hashée**.

## 14. Problèmes d’une shard key monotone ?
Toutes les nouvelles écritures tombent dans le dernier chunk, donc sur le même shard → **hot shard**, goulot d’étranglement, scalabilité perdue.

## 15. Comment activer le sharding sur base et collection ?
Via `mongos` :
```js
sh.enableSharding("maBase")
sh.shardCollection("maBase.maCollection", { cle: 1 })
```

## 16. Comment ajouter un nouveau shard ?
Démarrer un nouveau shard (souvent RS), puis :
```js
sh.addShard("replicaName/host:port")
```
Le balancer pourra ensuite redistribuer les chunks.

## 17. Comment vérifier l’état du cluster ?
Commandes utiles :
```js
sh.status()
db.printShardingStatus()
db.films.getShardDistribution()
```

## 18. Quand utiliser une hashed sharding key ?
Quand la clé naturelle est monotone (timestamp, auto-incrément) et qu’on veut **uniformiser les insertions** (éviter hot shard).

## 19. Quand privilégier une ranged sharding key ?
Quand on fait souvent des **requêtes par intervalle** (range queries). Cela permet des requêtes ciblées sur un sous-ensemble de shards.

## 20. Qu’est-ce que le zone sharding ?
Mécanisme qui associe des **plages** de shard key à des shards précis (zones/tags). Intérêt : localisation (réglementation), isolation de charge (premium, etc.).

## 21. Comment MongoDB gère les requêtes multi-shards ?
Si la requête n’est pas ciblable, `mongos` fait un **scatter-gather** : envoie à plusieurs shards, récupère, fusionne/tri/agrège, renvoie au client.

## 22. Optimiser les performances en shardé ?
- Requêtes **ciblées** (inclure shard key dans les filtres)
- Indexation adaptée (dont shard key)
- Clé de sharding pertinente (bonne distribution)
- Monitoring (chunks, migrations, hot shards)

## 23. Que se passe-t-il si un shard devient indisponible ?
Si le shard est un replica set : bascule vers un secondaire (élection).  
Si tout le shard est indisponible : les données de ce shard ne sont plus accessibles → requêtes concernées échouent, le reste du cluster peut rester partiellement disponible.

## 24. Migrer une collection existante vers shardé ?
Étapes :
1. `sh.enableSharding("db")`
2. Créer index shard key si nécessaire
3. `sh.shardCollection("db.col", { key: 1 })`
Puis splitting/migrations progressives.

## 25. Outils / métriques de diagnostic ?
- `sh.status()`, `db.printShardingStatus()`
- `db.collection.getShardDistribution()`
- `config.chunks`, `config.collections`
- logs `mongos`/`mongod`, monitoring (Atlas / tools)

---

## 9) Conclusion

Le sharding permet de **passer à l’échelle horizontalement** en répartissant une collection sur plusieurs shards. La performance et l’équilibre du cluster dépendent fortement du **choix de la shard key**, du comportement des chunks (split) et du **balancer** (migrations). Avec une bonne clé, le cluster distribue les écritures et évite les hot shards ; sinon, on observe une surcharge sur un nœud et des gains limités.

