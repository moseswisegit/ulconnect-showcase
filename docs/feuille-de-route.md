# Feuille de route

Ce qui reste à faire, déduit de l'état du code (fonctionnalités désactivées, limites documentées) et de l'état du projet.

## Statut actuel

- Application Android en validation sur Google Play (test fermé).
- Aucune version publique sur les stores ; aucun utilisateur réel.
- API et console déployées sur l'infrastructure cible.

## Avant une sortie publique

1. **Terminer la validation Google Play** et préparer la soumission sur l'App Store (compte développeur Apple, builds iOS).
2. **Brancher les achats intégrés** Apple et Google pour l'abonnement facultatif, avec vérification des reçus côté serveur. Tant que ce n'est pas fait, l'abonnement reste désactivé par configuration.
3. **Relecture juridique** de la politique de confidentialité, des conditions d'utilisation et de l'évaluation des facteurs relatifs à la vie privée ; signature de cette évaluation.
4. **Traiter les constats restants de l'audit** avant mise en production (gravité moyenne et basse).

## Qualité

- Exécuter les tests Jest de l'application mobile dans la CI (seule la vérification TypeScript y tourne aujourd'hui).
- Ajouter des tests ciblant PostgreSQL (conteneurs de test) pour couvrir le SQL propre à ce moteur.
- Refaire un test de charge sur une infrastructure proche de la production plutôt qu'en local.

## Exploitation

- Isoler ou verrouiller les tâches de fond avant de passer l'API à plusieurs instances, et brancher le service SignalR géré (déjà prévu par configuration).
- Remplacer le cache en mémoire par un cache partagé si plusieurs instances sont nécessaires.
- Mettre en place une supervision et des alertes (journaux centralisés, disponibilité).
- Envisager un coffre-fort de secrets plutôt que les paramètres d'application.

## Produit

- Permissions élargies pour les modérateurs sur les modules de la communauté (conçu, pas encore intégré).
- Retours des testeurs du test fermé à intégrer.
