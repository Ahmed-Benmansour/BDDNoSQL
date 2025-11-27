# Introduction à Redis : Exploration d'une Base de Données NoSQL Clé/Valeur

## Pourquoi Redis ?
Redis est une base de données NoSQL ultra-rapide, utilisée pour stocker des données en mémoire. 
Elle est particulièrement appréciée pour sa performance, sa simplicité d’utilisation et ses 
structures de données puissantes. On l’emploie souvent pour le caching, la gestion de sessions, 
le temps réel, ou encore le comptage de données à grande échelle.

## Objectif Pédagogique
Maîtriser les fondamentaux de Redis en utilisant sa CLI (Interface en Ligne de Commande) : depuis l'installation jusqu'à la manipulation des structures de données.
---
## Étape 1 : Déploiement de Redis via Docker
**Finalité :** Mettre en place Redis simplement sans complications d'installation.
1. Lancez votre interface de commandes.
2. Récupérez l'image officielle Redis grâce à cette commande :
   ```bash
   docker pull redis
   ```
3. Initialisez un conteneur Redis :
   ```bash
   docker run --name redis-tp -d -p 6379:6379 redis
   ```
4. Confirmez l'activation du conteneur :
   ```bash
   docker ps
   ```
5. Connectez-vous à l'interface Redis :
   ```bash
   docker exec -it redis-tp redis-cli
   ```
---
## Étape 2 : Découverte des Commandes Fondamentales
**Objectif :** Comprendre les mécanismes de gestion des clés et valeurs.
1. **Instancier une clé :**
   ```bash
   SET nom "Ahmed"
   ```
   Résultat attendu : `OK`
2. **Extraire une valeur :**
   ```bash
   GET nom
   ```
   Résultat attendu : `"Ahmed"`
3. **Mettre à jour la valeur d'une clé :**
   ```bash
   SET nom "Bob"
   ```
   Réponse : `OK`
4. **Supprimer un élément :**
   ```bash
   DEL nom
   ```
   Résultat attendu : `1` si la clé existait.
---
## Étape 3 : Exploration des Structures de Données
**But :** Approfondir les capacités avancées de Redis.
1. **Liste :** Manipuler des collections ordonnées.
   - Intégrer des éléments :
     ```bash
     RPUSH cours "Math" "Physique" "Chimie"
     ```
   - Visualiser le contenu :
     ```bash
     LRANGE cours 0 -1
     ```
     Résultat attendu : `["Math", "Physique", "Chimie"]`
2. **Ensemble :** Gérer des éléments uniques.
   - Ajouter des identifiants :
     ```bash
     SADD noms "Ahmed" "Othman"
     ```
   - Examiner les participants :
     ```bash
     SMEMBERS noms
     ```
   - Retirer un identifiant :
     ```bash
     SREM noms "Charlie"
     ```
3. **Hashmap :** Organiser des informations groupées.
   - Enregistrer des données utilisateur :
     ```bash
     HSET user:1 name "Ahmed" age 22 email "alice@example.com"
     ```
   - Récupérer toutes les informations :
     ```bash
     HGETALL user:1
     ```
---
## Étape 4 : Expérimentation Personnelle
**Objectif :** Encourager l'exploration et la créativité.
- **Incrémenter un indicateur :**
  ```bash
  INCR compteur
  ```
- **Générer une clé éphémère :**
  ```bash
  SET temporaire "Valeur" EX 60
  ```
- **Expérimenter HyperLogLog pour le décompte de visiteurs uniques :**
  ```bash
  PFADD visiteurs "User1" "User2" "User3"
  PFCOUNT visiteurs
  ```
---

## Conclusion
Redis se distingue par sa rapidité, sa simplicité et la richesse de ses structures de données.
Que ce soit pour gérer du cache, du temps réel, des compteurs, ou des données temporaires,
il constitue un outil puissant et polyvalent.  
Ce TP avait pour objectif de vous familiariser avec les bases essentielles : installation,
commandes fondamentales et manipulation des principales structures de données.  
Vous êtes désormais prêt à explorer davantage les fonctionnalités avancées de Redis et l’intégrer
dans des projets plus ambitieux.
