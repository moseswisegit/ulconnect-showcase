# Décisions techniques

Chaque décision est décrite à partir de ce qui est observable dans le code et la configuration. Quand la raison d'un choix n'est pas documentée dans le dépôt, elle est signalée par « Justification à confirmer par l'auteur ».

---

## 1. Connexion sans mot de passe pour les étudiants, comptes séparés pour la console

**Contexte.** L'application est réservée aux étudiants d'une université. Il faut prouver l'appartenance à l'établissement sans gérer de mots de passe pour des milliers de comptes.

**Options envisagées.**
- Mot de passe classique, avec vérification de l'adresse.
- Lien magique par courriel.
- Code à usage unique par courriel.
- Authentification unique (SSO) de l'université : non disponible pour le projet.

**Choix.** Code à 6 chiffres envoyé à l'adresse universitaire, puis JWT de courte durée et jeton de renouvellement avec rotation et détection de rejeu. La console utilise un modèle distinct : comptes sur invitation, mot de passe PBKDF2, TOTP obligatoire, cookie HttpOnly et jeton anti-CSRF.

**Compromis.**
- Aucune base de mots de passe étudiants à protéger, et la possession de l'adresse universitaire prouve l'appartenance.
- En contrepartie, chaque connexion dépend de la délivrabilité des courriels. D'où le domaine d'envoi authentifié (SPF, DKIM, DMARC) et des limites de débit par adresse et par IP pour empêcher l'abus du relais SMTP.
- Le code (plutôt qu'un lien) évite d'avoir à gérer l'ouverture d'un lien profond sur mobile.
- Le SSO de l'université n'était pas disponible pour un projet étudiant indépendant. Le code envoyé à l'adresse universitaire était l'option la plus proche pour prouver l'appartenance à l'établissement.

---

## 2. Un seul modèle de données, deux moteurs de base : SQLite en local et en tests, PostgreSQL en production

**Contexte.** L'API doit pouvoir être exécutée et testée sans dépendance externe, tout en tournant sur une base robuste en production.

**Options envisagées.**
- PostgreSQL partout (local via Docker, tests via conteneurs de test).
- Base en mémoire d'Entity Framework pour les tests.
- SQLite en local et en tests, PostgreSQL en production.

**Choix.** Un indicateur de configuration choisit le fournisseur Entity Framework Core. Les tests d'intégration démarrent l'API complète en mémoire avec une vraie base SQLite par classe de test.

**Compromis.**
- Démarrage local sans aucune installation, et tests rapides qui passent par le vrai pipeline HTTP, la vraie validation et de vraies requêtes SQL (contrairement au fournisseur en mémoire).
- En contrepartie, certaines fonctions propres à PostgreSQL (index spécialisés, comportements de tri) ne sont pas couvertes par les tests. Le SQL spécifique est conditionné au fournisseur actif dans les migrations. Les tests de performance ont été faits à part, sur une copie PostgreSQL locale.

---

## 3. Console d'administration en site statique plutôt que dans un conteneur

**Contexte.** La console était d'abord servie par un conteneur nginx. Le dépôt garde encore ce Dockerfile pour l'usage local avec Docker Compose.

**Options envisagées.**
- Conteneur nginx sur un service applicatif.
- Hébergement statique sur un stockage Blob.
- Service d'hébergement de sites statiques dédié.

**Choix.** Compilation Vite dans GitHub Actions, puis téléversement des fichiers vers un stockage Blob en mode « site web statique », derrière Cloudflare pour le domaine personnalisé et le certificat TLS.

**Compromis.**
- L'objectif principal était de réduire les coûts : l'hébergement de la console devient quasi nul, et aucun serveur n'est à maintenir pour elle.
- En contrepartie, l'URL de l'API est fixée au moment de la compilation (plus d'injection au démarrage du conteneur). Le stockage statique ne fournissant pas de HTTPS sur domaine personnalisé sans CDN, Cloudflare est nécessaire devant.
- Le stockage Blob a été retenu plutôt qu'un service dédié aux sites statiques pour la même raison : réduire les coûts.

---

## 4. Lectures communes mises en cache 30 secondes, à la suite d'un test de charge

**Contexte.** Un test de charge local simulant l'arrivée massive de nouveaux étudiants a montré l'effondrement de l'API à 2 000 inscriptions par minute : base saturée en CPU, pool de connexions plein. Les requêtes les plus coûteuses renvoyaient le même résultat à tout le monde (compteurs, accueil de la communauté, catalogue de programmes) ou une liste complète inutilement longue.

**Options envisagées.**
- Ajouter des ressources à la base.
- Cache distribué externe.
- Cache en mémoire avec coalescence des requêtes simultanées.

**Choix.** Un cache partagé de 30 secondes en mémoire, avec une seule exécution de la requête sous-jacente même si plusieurs requêtes arrivent en même temps ; invalidation lorsque l'administration modifie la donnée. La liste d'entraide a été limitée aux 60 résultats les plus pertinents, avec un résumé séparé. Les limites par IP ont été relevées pour les réseaux partagés.

**Compromis.**
- Résultat mesuré sur le même scénario : 100 % des inscriptions réussies à 2 000 par minute, 95 % des requêtes en 2,1 s ou moins.
- En contrepartie, certaines données peuvent avoir jusqu'à 30 secondes de retard. Le cache étant local au processus, il serait à revoir avec plusieurs instances.

---

## 5. Monétisation livrée mais verrouillée par configuration

**Contexte.** Un abonnement facultatif est prévu. Les règles des stores imposent les achats intégrés d'Apple et de Google, qui ne sont pas encore branchés.

**Choix.** Le modèle complet est en place (page de la console, statuts, préavis aux utilisateurs, période d'essai, application côté serveur sur les routes concernées, écrans mobiles), mais un indicateur de configuration côté serveur empêche toute activation tant que les achats ne sont pas branchés. Le module d'achat mobile lève volontairement une erreur explicite.

**Compromis.**
- Le code peut être fusionné et déployé sans risque de facturer ou de bloquer quiconque.
- En contrepartie, une partie du code n'est pas exercée en conditions réelles.

---

## 6. Déploiement : image Docker étiquetée par commit, un déploiement à la fois

**Contexte.** Plusieurs fusions rapprochées ont déjà provoqué une course : un déploiement plus ancien s'est terminé après un plus récent et a remis l'ancienne image en production.

**Choix.**
- Les tests de l'API servent de barrière avant toute construction d'image.
- L'image est étiquetée avec le SHA du commit, et c'est cette étiquette (pas `latest`) qui est déployée.
- Un groupe de concurrence garantit un seul déploiement à la fois, dans l'ordre des fusions, sans annuler ceux en attente.
- Les migrations sont appliquées au démarrage du conteneur.

**Compromis.**
- Déploiements reproductibles et ordonnés.
- Les migrations au démarrage simplifient l'opération mais lient le schéma au déploiement du code ; un retour arrière du code ne défait pas une migration.

---

## 7. Tâches de fond dans le processus de l'API

**Contexte.** Le produit a besoin de traitements planifiés : rappels, purge des comptes inactifs, résumé hebdomadaire, envoi des notifications push par lots.

**Choix.** Quatre services hébergés ASP.NET Core dans l'API elle-même, plutôt que des fonctions sans serveur ou une file de messages externe.

**Compromis.**
- Un seul artefact à déployer, et pas de service supplémentaire à payer.
- L'API tourne volontairement sur une seule instance, pour simplifier l'exploitation tant qu'il n'y a pas d'utilisateurs réels : pas de verrou distribué, pas de cache partagé externe, pas de service SignalR géré.
- En contrepartie, passer à plusieurs instances demandera d'isoler ou de verrouiller ces tâches, de partager le cache et de brancher le service SignalR géré (déjà prévu par configuration).

---

## 8. Versions de .NET différentes pour l'API et les tests

**Constat.** L'API cible .NET 8 ; le projet de tests cible .NET 10. Le pipeline installe donc les deux SDK.

**Justification à confirmer par l'auteur.**
