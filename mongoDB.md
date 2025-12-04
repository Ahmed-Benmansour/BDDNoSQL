# Introduction Générale à MongoDB

MongoDB est une base de données **NoSQL orientée documents**, qui stocke les informations sous forme de **documents JSON** flexibles.  
Contrairement à une base relationnelle (SQL) où les données sont organisées en tables, MongoDB permet :

- une structure **souple**, sans schéma strict ;
- des performances élevées pour les requêtes en lecture ;
- la gestion de données complexes (tableaux, objets imbriqués) ;
- une grande scalabilité horizontale grâce au sharding ;
- un langage de requêtes naturel basé sur JSON.

MongoDB est très utilisé pour :
- les applications web modernes,
- les APIs,
- le traitement de données semi-structurées,
- les systèmes distribués,
- l’analyse en temps réel.

Ce rapport vise à comprendre à la fois **les requêtes de base** et **les opérations avancées** grâce aux deux TP fournis.


# Rapport Complet MongoDB — TP MongoDB + TP1 (Films) Avec Explications Intégrées

## Introduction
Dans ce rapport, nous réunissons **les deux TP MongoDB** :
- **TP MongoDB (prise en main)** : filtrage, projection, agrégation, mises à jour et indexation.
- **TP1 avec films.json** : exploration d’une collection de films et requêtes complètes.

Chaque requête est accompagnée **d’explications intégrées**, comme demandé.

---

# 🟥 PARTIE 1 — Prise en main de MongoDB (TPMongoDB) — Avec Explications

## 1. Afficher les 5 films sortis depuis 2015
```js
db.movies.find({ year: { $gte: 2015 } }).limit(5)
```
**Explication :**  
- `$gte` signifie *greater than or equal* → supérieur ou égal.  
- On limite l’affichage avec `.limit(5)`.

---

## 2. Films de genre "Comedy"
```js
db.movies.find({ genres: "Comedy" })
```
**Explication :**  
MongoDB détecte automatiquement si "Comedy" est dans un tableau.

---

## 3. Films sortis entre 2000 et 2005
```js
db.movies.find(
  { year: { $gte: 2000, $lte: 2005 } },
  { title: 1, year: 1 }
)
```
**Explication :**  
- `$gte` = ≥  
- `$lte` = ≤  
- `{title:1, year:1}` → projection : afficher seulement ces champs.

---

## 4. Films “Drama” ET “Romance”
```js
db.movies.find({ genres: { $all: ["Drama", "Romance"] } })
```
**Explication :**  
`$all` oblige MongoDB à vérifier que les deux genres sont présents.

---

## 5. Films sans champ rated
```js
db.movies.find({ rated: { $exists: false } })
```
**Explication :**  
`$exists:false` détecte l’absence de champ.

---

## 6. Nombre de films par année
```js
db.movies.aggregate([
  { $group: { _id: "$year", total: { $sum: 1 } } },
  { $sort: { _id: 1 } }
])
```
**Explication :**  
- `$group` regroupe par année  
- `$sum:1` compte les documents  
- `$sort` trie par année

---

## 7. Moyenne IMDb par genre
```js
db.movies.aggregate([
  { $unwind: "$genres" },
  { $group: { _id: "$genres", moyenne: { $avg: "$imdb.rating" } } },
  { $sort: { moyenne: -1 } }
])
```
**Explication :**  
- `$unwind` casse un tableau → 1 ligne par genre  
- `$avg` calcule la moyenne.  

---

## 8. Nombre de films par pays
```js
db.movies.aggregate([
  { $unwind: "$countries" },
  { $group: { _id: "$countries", total: { $sum: 1 } } },
  { $sort: { total: -1 } }
])
```

---

## 9. Top 5 réalisateurs
```js
db.movies.aggregate([
  { $unwind: "$directors" },
  { $group: { _id: "$directors", total: { $sum: 1 } } },
  { $sort: { total: -1 } },
  { $limit: 5 }
])
```

---

## 10. Trier par note IMDb
```js
db.movies.aggregate([
  { $sort: { "imdb.rating": -1 } },
  { $project: { title: 1, "imdb.rating": 1 } }
])
```
**Explication :**  
- `$sort:-1` = décroissant  
- `$project` = choisir les champs

---

## 11. Ajouter un champ
```js
db.movies.updateOne(
  { title: "Jaws" },
  { $set: { etat: "culte" } }
)
```

---

## 12. Incrémenter un champ
```js
db.movies.updateOne(
  { title: "Inception" },
  { $inc: { "imdb.votes": 100 } }
)
```

---

## 13. Supprimer un champ
```js
db.movies.updateMany({}, { $unset: { poster: "" } })
```

---

## 14. Modifier un réalisateur
```js
db.movies.updateOne(
  { title: "Titanic" },
  { $set: { directors: ["James Cameron"] } }
)
```

---

## 15. Films mieux notés par décennie
```js
db.movies.aggregate([
  { $match: { "imdb.rating": { $exists: true } } },
  { $project: {
      title: 1,
      decade: { $subtract: ["$year", { $mod: ["$year", 10] }] },
      "imdb.rating": 1
  }},
  { $group: { _id: "$decade", maxRating: { $max: "$imdb.rating" } } },
  { $sort: { _id: 1 } }
])
```
**Explication :**  
- `decade` calcule la décennie (ex : 1994 → 1990).  

---

# 🟦 PARTIE 2 — TP1 MongoDB : Collection films.json (Films) — Avec Explications

## Import de la collection
```bash
mongoimport --db lesfilms --collection films films.json --jsonArray
```
**Explication :**  
- `--jsonArray` indique que le fichier contient un tableau JSON.  

---

## 1. Voir le nombre de films
```js
db.films.count()
```

---

## 2. Voir un document
```js
db.films.findOne()
```

---

## 3. Films d’action
```js
db.films.find({ genre: "Action" })
```
**Explication :**  
Recherche simple par égalité.

---

## 4. Nombre de films d’action
```js
db.films.count({ genre: "Action" })
```

---

## 5. Films d’action français
```js
db.films.find({ genre: "Action", country: "FR" })
```

---

## 6. Films d’action français (1963)
```js
db.films.find({ genre: "Action", country: "FR", year: 1963 })
```

---

## 7. Masquer un champ
```js
db.films.find(
  { genre: "Action", country: "FR" },
  { grades: 0 }
)
```
**Explication :**  
`0` signifie masquer.

---

## 8. Masquer l’identifiant
```js
db.films.find(
  { genre: "Action", country: "FR" },
  { _id: 0 }
)
```

---

## 9. Afficher titres + notes
```js
db.films.find(
  { genre: "Action", country: "FR" },
  { _id: 0, title: 1, grades: 1 }
)
```

---

## 10. Notes toutes supérieures à 10
```js
db.films.find({
  genre: "Action",
  country: "FR",
  grades: { $not: { $elemMatch: { note: { $lte: 10 } } } }
})
```
**Explication :**  
- `$elemMatch` analyse chaque note  
- `$not` inverse la condition  
- résultat : aucune note ≤ 10  

---

## 11. Genres distincts
```js
db.films.distinct("genre")
```

---

## 12. Grades distincts
```js
db.films.distinct("grades.grade")
```

---

## 13. Films avec artistes donnés
```js
db.films.find({
  "actors._id": { $in: ["artist:4","artist:18","artist:11"] }
})
```
**Explication :**  
`$in` vérifie si l’un des artistes apparaît dans le tableau `actors`.

---

## 14. Films sans résumé
```js
db.films.find({ summary: { $exists: false } })
```

---

## 15. Films avec Leonardo DiCaprio en 1997
```js
db.films.find({
  year: 1997,
  "actors.last_name": "DiCaprio"
})
```

---

## 16. Films DiCaprio OU 1997
```js
db.films.find({
  $or: [
    { "actors.last_name": "DiCaprio" },
    { year: 1997 }
  ]
})
```

---

# Conclusion
Les deux TP combinés permettent de comprendre :
- les requêtes simples et filtres complexes,
- les agrégations,
- la manipulation documentaire,
- l’indexation,
- l’analyse d’une base réelle (films.json).

MongoDB est un outil flexible et puissant pour manipuler des données semi-structurées.





# Conclusion Générale

Ces deux TP réunis permettent d'acquérir une compréhension solide de MongoDB :

- manipulation de documents JSON,
- filtrage, projection et requêtes avancées,
- agrégations pour produire des statistiques,
- mises à jour ciblées,
- création et analyse d'index,
- utilisation concrète d'une collection réelle (films.json).
