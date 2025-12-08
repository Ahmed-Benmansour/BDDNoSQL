# TP MongoDB – Réplication et Replica Sets

---

# **Partie 1 – Questions de compréhension**

## **1. Qu’est-ce qu’un Replica Set dans MongoDB ?**

Un **Replica Set** est un groupe de serveurs MongoDB qui possèdent et maintiennent **la même copie des données** via un mécanisme de réplication. Il garantit :

* la **tolérance aux pannes** (si un nœud tombe, les autres prennent le relais),
* la **haute disponibilité**,
* une **sécurité des données accrue**.

Il est composé généralement d’un **Primary**, de plusieurs **Secondaries**, et parfois d’un **Arbitre**.

---

## **2. Quel est le rôle du Primary ?**

Le **Primary** est le serveur maître :

* il reçoit **toutes les écritures** (insert/update/delete),
* il écrit dans l’**oplog** (journal des opérations),
* il réplique les opérations vers les **Secondaries**,
* il gère par défaut la majorité des lectures,
* il est remplacé automatiquement en cas de panne.

---

## **3. Quel est le rôle des Secondaries ?**

Les **Secondaries** :

* répliquent les données du Primary (réplication **asynchrone**),
* servent de **sauvegarde active**,
* peuvent devenir **Primary** lors d’une élection,
* peuvent servir des lectures si configuré (mais risque de données obsolètes).

---

## **4. Pourquoi MongoDB interdit-il les écritures sur un Secondary ?**

Pour éviter :

* les **conflits de données**,
* l’incohérence liée à la réplication asynchrone,
* les situations de **split-brain**.

Toutes les écritures doivent passer par le **Primary**, garantissant une cohérence forte.

---

## **5. Qu’est-ce que la cohérence forte dans MongoDB ?**

MongoDB garantit que :

* toute lecture effectuée sur le **Primary** renvoie la **version la plus récente** des données,
* une écriture est confirmée seulement après être dans le journal.

Ainsi, le Primary assure toujours une vision exacte et à jour des données.

---

## **6. Différence entre `readPreference: "primary"` et `"secondary"`**

| Option        | Lecture effectuée sur | Cohérence                    | Utilisation                     |
| ------------- | --------------------- | ---------------------------- | ------------------------------- |
| **primary**   | Primary               | Forte                        | Par défaut, lecture sûre        |
| **secondary** | Secondaries           | Faible (peut être en retard) | Distribuer la charge, analytics |

---

## **7. Pourquoi lire sur un Secondary ?**

* réduire la charge sur le Primary,
* diminuer la latence (clients géographiquement proches d’un Secondary),
* exécuter des requêtes lourdes sans bloquer la production,
* cas où une légère incohérence est acceptable (cohérence éventuelle).

---

# **Partie 2 – Commandes Replica Set**

## **8. Initialiser un Replica Set**

```js
rs.initiate()
```

ou avec configuration détaillée.

---

## **9. Ajouter un nœud après initialisation**

```js
rs.add("host:port")
```

---

## **10. Afficher l’état actuel du Replica Set**

```js
rs.status()
```

---

## **11. Identifier le rôle d’un nœud**

```js
rs.isMaster()   // ou db.hello()
```

---

## **12. Forcer une bascule du Primary**

```js
rs.stepDown()
```

---

## **13. Désigner un nœud comme Arbitre**

```js
rs.addArb("host:port")
```

L’arbitre :

* **ne stocke pas de données**,
* participe uniquement aux **votes**,
* permet d’ajouter un votant sans coût disque.

---

## **14. Configurer un Secondary avec un délai (`slaveDelay`)**

```js
cfg = rs.conf()
cfg.members[i].slaveDelay = 120
cfg.members[i].priority = 0
rs.reconfig(cfg)
```

Ce nœud aura un retard volontaire de 120 secondes.

---

# **Partie 3 – Tolérance aux pannes et scénarios**

## **15. Pas de majorité après panne du Primary ?**

→ **Aucun nouveau Primary n’est élu**. Le système passe en **mode dégradé**, les écritures sont refusées.

Cela évite le **split-brain**.

---

## **16. Critères pour choisir un nouveau Primary**

MongoDB élit un nouveau Primary selon :

* fraîcheur des données (oplog à jour),
* priorité la plus élevée,
* état sain (health = 1),
* appartenance au groupe majoritaire.

---

## **17. Qu’est-ce qu’une élection ?**

C’est un processus où :

1. un Secondary se propose,
2. chaque membre vote,
3. le candidat obtenant **> 50 %** devient Primary.

---

## **18. Auto-dégradation du Replica Set**

Lorsqu’un nœud perd la majorité :

* il se **rétrograde automatiquement**,
* il ne peut plus accepter d’écritures.

---

## **19. Pourquoi un nombre impair de nœuds ?**

Pour éviter les égalités lors des votes et assurer :

* une **majorité claire**,
* une meilleure disponibilité.

---

## **20. Effets d’une partition réseau**

* un seul côté (majoritaire) peut élire un Primary,
* l’autre côté reste sans Primary,
* évite le split-brain.

---

# **Partie 4 – Scénarios avancés**

## **21. 3 nœuds : Primary (27017), Secondary (27018), Arbiter (27019)**

Si 27017 tombe :

* 27018 + 27019 = 2 votes → **majorité**,
* 27018 devient **Primary**.

---

## **22. Utilité d’un Secondary retardé (`slaveDelay`)**

* protection contre erreurs humaines (drop accidentel),
* restauration d’un état antérieur,
* nœud dédié aux audits.

---

## **23. Lecture toujours à jour après bascule : readConcern/writeConcern**

Recommandé :

```json
writeConcern: "majority"
readConcern: "majority"
```

Option encore plus stricte : `readConcern: "linearizable"`.

---

## **24. Garantir 2 nœuds qui confirment une écriture**

```json
writeConcern: { w: 2 }
```

---

## **25. Lecture obsolète depuis un Secondary : pourquoi ?**

→ réplication **asynchrone** → retard possible.

Solutions :

* lire sur le Primary,
* utiliser `readConcern: "majority"`,
* limiter le lag (`maxStalenessSeconds`).

---

## **26. Identifier le Primary**

```js
rs.status()
```

Regarder `stateStr: "PRIMARY"`.

---

## **27. Forcer un basculement manuel**

```js
rs.stepDown(60)
```

---

## **28. Ajouter un Secondary dans un Replica Set actif**

1. lancer un nouveau mongod avec `--replSet`,
2. depuis le Primary :

```js
rs.add("host:port")
```

3. attendre la synchronisation.

---

## **29. Retirer un nœud défectueux**

```js
rs.remove("host:port")
```

---

## **30. Configurer un Secondary caché**

```js
cfg = rs.conf()
cfg.members[i].hidden = true
cfg.members[i].priority = 0
rs.reconfig(cfg)
```

Usage : analytics, sauvegardes, tests.

---

## **31. Modifier la priorité d’un nœud**

```js
cfg = rs.conf()
cfg.members[i].priority = 10
rs.reconfig(cfg)
```

---

## **32. Vérifier le lag de réplication**

```js
rs.printSecondaryReplicationInfo()
```

---

# **Questions complémentaires**

## **33. Fonction de `rs.freeze()`**

Empêche temporairement un nœud d’être candidat :

```js
rs.freeze(60)
```

Utile pour : maintenance, synchronisation, tests.

---

## **34. Redémarrer un Replica Set sans perdre la configuration**

→ redémarrage progressif (rolling restart).

La configuration est stockée dans : `local.system.replset`.

---

## **35. Surveiller la réplication en temps réel**

* `rs.status()`
* `rs.printReplicationInfo()`
* `rs.printSecondaryReplicationInfo()`
* logs :

```bash
tail -f mongod.log
```

---

## **37. Qu’est-ce qu’un Arbitre ?**

Un **Arbiter** :

* ne stocke **aucune donnée**,
* n’est jamais Primary,
* sert uniquement à voter.

---

## **38. Vérifier la latence de réplication**

```js
rs.printSecondaryReplicationInfo()
```

Comparaison des `optimeDate`.

---

## **39. Commande affichant le retard des Secondaries**

```js
rs.printSecondaryReplicationInfo()
```

---

## **40. Réplication asynchrone vs synchrone**

MongoDB utilise une réplication **asynchrone** par défaut.

---

## **41. Modifier la configuration d’un Replica Set sans redémarrer**

```js
rs.reconfig(cfg)
```

(sauf changements de port/dbpath nécessitant un restart)

---

## **42. Secondary très en retard**

Possible conséquences :

* données obsolètes,
* mise en état `RECOVERING`,
* resynchronisation complète si dépasse la fenêtre de l’oplog.

---

## **43. Gestion des conflits en réplication**

→ modèle Primary unique → pas de conflit.

En cas rare de divergence : **rollback** automatique.

---

## **44. Plusieurs Primary simultanés ?**

Impossible.

* système de majorité,
* auto-dégradation,
* prévention du split-brain.

---

## **45. Pourquoi ne pas écrire sur un Secondary ?**

Car MongoDB ne permet **jamais** l’écriture sur un Secondary.
`readPreference` ne s’applique **que** aux lectures.

---

## **46. Effets d’un réseau instable**

* élections fréquentes,
* réplication ralentie,
* erreurs heartbeat,
* indisponibilité temporaire,
* risques de rollback.

---

