
# TP4 — MapReduce  
## MongoDB & CouchDB  
### Rapport complet

---

## Introduction générale — Le paradigme MapReduce

**MapReduce** est un modèle de programmation conçu pour le traitement et l’analyse de grandes quantités de données.
Il repose sur une logique simple mais puissante, divisée en plusieurs phases :

- **Map** : chaque document d’entrée est transformé en un ou plusieurs couples *(clé, valeur)*  
- **Reduce** : toutes les valeurs associées à une même clé sont agrégées  
- **Finalize (optionnelle)** : permet de produire un résultat final (moyenne, ratio, etc.)

Ce paradigme est particulièrement adapté aux bases **NoSQL orientées documents**, comme **MongoDB** et **CouchDB**.
Cependant, ces deux systèmes implémentent MapReduce de manière différente, ce que ce TP vise à mettre en évidence.

Le TP4 est donc structuré en **deux grandes parties** :
1. MapReduce avec MongoDB  
2. CouchDB et MapReduce via les vues  

---

# PARTIE 1 — MAPREDUCE AVEC MONGODB

## 1. Présentation de MongoDB

MongoDB est une base de données NoSQL orientée documents.
Les données sont stockées sous forme de documents **JSON/BSON**, regroupés en collections.

MongoDB permet l’agrégation des données via :
- l’**Aggregation Pipeline**
- **MapReduce**, plus général et plus flexible mais aussi plus coûteux en ressources

Dans ce TP, l’accent est mis sur **MapReduce**.

---

## 2. Fonctionnement de MapReduce dans MongoDB

Dans MongoDB :
- MapReduce est exécuté **à la demande**
- le résultat est stocké dans une **collection de sortie**
- les fonctions sont écrites en **JavaScript**

Commande générique :

```js
db.films.mapReduce(mapFunction, reduceFunction, {
  out: "collection_resultat",
  finalize: finalizeFunction // optionnel
})
```

---

## 3. Données utilisées

Tous les exercices utilisent la collection `films`, contenant notamment :
- `title`
- `year`
- `genre` (tableau)
- `director`
- `actors` (tableau)
- `runtime`
- `country`
- `language`
- `rated`

---

## 4. Exercices MongoDB (1 à 15)

### Exercice 1 — Nombre total de films
```js
var map=function(){emit("total",1);};
var reduce=function(k,v){return Array.sum(v);};
db.films.mapReduce(map,reduce,{out:"ex1"});
```

### Exercice 2 — Nombre de films par genre
```js
var map=function(){
 if(this.genre) this.genre.forEach(g=>emit(g,1));
};
var reduce=function(k,v){return Array.sum(v);};
db.films.mapReduce(map,reduce,{out:"ex2"});
```

### Exercice 3 — Nombre de films par réalisateur
```js
var map=function(){emit(this.director,1);};
var reduce=function(k,v){return Array.sum(v);};
db.films.mapReduce(map,reduce,{out:"ex3"});
```

### Exercice 4 — Nombre de films par année
```js
var map=function(){emit(this.year,1);};
var reduce=function(k,v){return Array.sum(v);};
db.films.mapReduce(map,reduce,{out:"ex4"});
```

### Exercice 5 — Nombre moyen de films par réalisateur
```js
var map=function(){emit(this.director,1);};
var reduce=function(k,v){return Array.sum(v);};
db.films.mapReduce(map,reduce,{out:"ex5"});
```

### Exercice 6 — Genres les plus représentés
```js
var map=function(){
 if(this.genre) this.genre.forEach(g=>emit(g,1));
};
var reduce=function(k,v){return Array.sum(v);};
db.films.mapReduce(map,reduce,{out:"ex6"});
```

### Exercice 7 — Acteurs les plus présents
```js
var map=function(){
 if(this.actors) this.actors.forEach(a=>emit(a,1));
};
var reduce=function(k,v){return Array.sum(v);};
db.films.mapReduce(map,reduce,{out:"ex7"});
```

### Exercice 8 — Nombre de films par acteur
```js
var map=function(){
 if(this.actors) this.actors.forEach(a=>emit(a,1));
};
var reduce=function(k,v){return Array.sum(v);};
db.films.mapReduce(map,reduce,{out:"ex8"});
```

### Exercice 9 — Films par couple (acteur, genre)
```js
var map=function(){
 if(this.actors && this.genre)
  this.actors.forEach(a=>{
   this.genre.forEach(g=>{
    emit({actor:a,genre:g},1);
   });
  });
};
var reduce=function(k,v){return Array.sum(v);};
db.films.mapReduce(map,reduce,{out:"ex9"});
```

### Exercice 10 — Films par décennie
```js
var map=function(){
 var d=Math.floor(this.year/10)*10;
 emit(d,1);
};
var reduce=function(k,v){return Array.sum(v);};
db.films.mapReduce(map,reduce,{out:"ex10"});
```

### Exercice 11 — Durée moyenne des films
```js
var map=function(){
 emit("avg",{sum:this.runtime,count:1});
};
var reduce=function(k,v){
 var s=0,c=0;
 v.forEach(x=>{s+=x.sum;c+=x.count;});
 return {sum:s,count:c};
};
var finalize=function(k,v){return v.sum/v.count;};
db.films.mapReduce(map,reduce,{out:"ex11",finalize:finalize});
```

### Exercice 12 — Films par pays
```js
var map=function(){emit(this.country,1);};
var reduce=function(k,v){return Array.sum(v);};
db.films.mapReduce(map,reduce,{out:"ex12"});
```

### Exercice 13 — Films par langue
```js
var map=function(){emit(this.language,1);};
var reduce=function(k,v){return Array.sum(v);};
db.films.mapReduce(map,reduce,{out:"ex13"});
```

### Exercice 14 — Films par classification
```js
var map=function(){emit(this.rated,1);};
var reduce=function(k,v){return Array.sum(v);};
db.films.mapReduce(map,reduce,{out:"ex14"});
```

### Exercice 15 — Analyse libre
```js
var map=function(){
 emit(this.year,{count:1,runtime:this.runtime});
};
var reduce=function(k,v){
 var c=0,r=0;
 v.forEach(x=>{c+=x.count;r+=x.runtime;});
 return {count:c,runtime:r};
};
var finalize=function(k,v){return v.runtime/v.count;};
db.films.mapReduce(map,reduce,{out:"ex15",finalize:finalize});
```

---

# PARTIE 2 — COUCHDB ET MAPREDUCE

## 1. Présentation de CouchDB

CouchDB est une base de données NoSQL orientée documents, conçue autour du web.
Toutes les interactions avec CouchDB se font via une **API REST HTTP**.

Contrairement à MongoDB :
- CouchDB ne propose pas de MapReduce ponctuel
- MapReduce est utilisé via des **vues persistantes**
- les résultats sont automatiquement indexés

---

## 2. MapReduce dans CouchDB

Dans CouchDB :
- MapReduce est utilisé pour créer des **vues**
- les vues sont stockées dans des **design documents**
- les résultats sont recalculés automatiquement à l’insertion/modification des documents

Une vue contient :
- une fonction **map**
- éventuellement une fonction **reduce**

---

## 3. Installation CouchDB (Docker)

```bash
docker volume create couchdb_data

docker run \
  --name couchdb \
  -e COUCHDB_USER=ethan \
  -e COUCHDB_PASSWORD=nicolas \
  -p 5984:5984 \
  -v couchdb_data:/opt/couchdb/data \
  -d couchdb
```

Interface Fauxton :  
`http://localhost:5984/_utils/`

---

## 4. Manipulation des données

### Création de base
```bash
curl -X PUT http://ethan:nicolas@localhost:5984/films
```

### Insertion simple
```bash
curl -X POST http://ethan:nicolas@localhost:5984/films \
 -H "Content-Type: application/json" \
 -d '{"title":"Inception","year":2010}'
```

### Insertion en lot
```bash
curl -X POST http://ethan:nicolas@localhost:5984/films/_bulk_docs \
 -H "Content-Type: application/json" \
 -d @films_couchdb.json
```

---

## 5. Vues MapReduce CouchDB

### Vue 1 — Nombre de films par année
```js
function(doc){
 emit(doc.year,doc.title);
}
```
```js
function(keys,values){
 return values.length;
}
```

### Vue 2 — Index par acteurs
```js
function(doc){
 if(doc.actors)
  doc.actors.forEach(a=>emit(a,doc.title));
}
```
```js
function(keys,values){
 return values.length;
}
```

### Vue 3 — Films par genre
```js
function(doc){
 if(doc.genre)
  doc.genre.forEach(g=>emit(g,1));
}
```
```js
function(keys,values){
 return sum(values);
}
```

---

## 6. Comparaison MongoDB / CouchDB

| MongoDB | CouchDB |
|-------|---------|
| MapReduce ponctuel | Vues MapReduce persistantes |
| Résultat dans une collection | Résultat indexé |
| Exécution manuelle | Mise à jour automatique |
| Flexible pour analyse | Optimisé pour consultation |

---


