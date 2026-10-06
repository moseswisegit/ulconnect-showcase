# Tests et CI/CD

## Chiffres

| Suite | Outil | Nombre de tests | Fichiers |
|---|---|---|---|
| API | xUnit 2.9, `Microsoft.AspNetCore.Mvc.Testing` | 509, tous au vert | environ 100 fichiers de test |
| Application mobile | Jest 29, `jest-expo`, React Native Testing Library | 58, tous au vert | 12 suites |

Chiffres relevés le 6 octobre 2026 en exécutant les deux suites localement.

## Tests de l'API

### Organisation

- Tests d'intégration de bout en bout : une fabrique d'application démarre l'API complète en mémoire (`WebApplicationFactory`), avec le vrai pipeline (authentification, autorisation, limitation de débit, validation, filtres).
- Chaque classe de test dispose de sa propre base SQLite, créée par les vraies migrations.
- Les dépendances externes sont remplacées par des doublures simples : envoi de courriels (capturés pour lire le code de connexion), stockage d'images et de documents, notifications push, contexte du hub SignalR.
- Le cache partagé est désactivé dans les tests pour garder des résultats déterministes.

### Ce qui est couvert (exemples de familles de tests)

- Authentification : codes de vérification, abus, rotation et rejeu du jeton de renouvellement, séparation des types de jetons.
- Console : rôles et permissions, double authentification, cookie de session et CSRF, invitations, suspension, export.
- Limitation de débit : connexion, actions authentifiées, alertes, routes de la console.
- Confidentialité : suppression de compte (et sa reprise après échec), purge des comptes inactifs, rétention des notifications, masquage des courriels dans les journaux, retrait des métadonnées d'image.
- En-têtes de sécurité, en-têtes transférés par le proxy, mode de globalisation.
- Fonctionnalités : messagerie paginée, temps réel, équipes, entraide, rencontre, monétisation, résumé hebdomadaire, invalidation du cache de sortie.

### Exemple de forme d'un test

Voir [`../snippets/test-exemple.md`](../snippets/test-exemple.md).

## Tests de l'application mobile

- Tests de rendu d'écrans et de composants (accueil, aperçu de profil, consentement, découverte, suppression de compte).
- Tests de services et d'utilitaires (client de la communauté, formatage des dates, thème).
- Le mode d'achat est simulé, puisque les achats intégrés ne sont pas branchés.

## Tests de charge

Réalisés en local, pas sur l'infrastructure de production.

| Condition | Valeur |
|---|---|
| Machine | Un Mac de développement, 8 coeurs |
| Base | PostgreSQL 16 dans un conteneur Docker local |
| Volume de données | 100 000 comptes étudiants et 4 millions de messages générés |
| API | Compilée en Release, exécutée localement, envoi de courriels désactivé |
| Scénario | N nouveaux étudiants en 60 secondes, parcours complet : demande de code, vérification, création du profil, ouverture de l'accueil |
| Outil | Script Node.js maison ; les codes de connexion sont lus dans le journal de l'API ; une adresse IP distincte simulée par étudiant, ou 8 adresses partagées pour simuler le Wi-Fi du campus |

Résultats après optimisation :

| Charge | Résultat |
|---|---|
| 1 000 inscriptions par minute | 95 % des requêtes sous 90 ms |
| 2 000 inscriptions par minute | 100 % réussies, 95 % des requêtes en 2,1 s ou moins |
| 300 étudiants derrière 8 adresses IP | 100 % réussies (64 % bloqués avant correction) |

Avant optimisation, 2 000 inscriptions par minute provoquaient l'échec de 31 % des enregistrements de profil, avec des réponses de 30 secondes.

Limites de cette mesure : une seule machine joue à la fois le client, l'API et la base ; la latence réseau réelle, le service PostgreSQL géré et l'hébergement de production ne sont pas représentés. Ces chiffres montrent l'effet des optimisations, pas la capacité garantie en production.

## Intégration continue

Un workflow s'exécute sur chaque pull request et chaque push sur la branche principale, avec trois tâches en parallèle :

1. **API** : compilation en Release et exécution de toute la suite xUnit.
2. **Mobile** : installation reproductible des dépendances (`npm ci`) et vérification TypeScript (`tsc --noEmit`).
3. **Console** : installation et compilation de production (TypeScript puis Vite).

Les tests Jest de l'application mobile ne sont pas encore exécutés dans la CI : ils sont exécutés localement.

## Déploiement continu

Un second workflow s'exécute à chaque fusion sur la branche principale :

1. **Barrière de tests** : compilation et tests de l'API. Une branche rouge n'atteint jamais l'étape de publication.
2. **Image de l'API** : construction Docker (Buildx, cache des couches), publication dans un registre de conteneurs privé, étiquetée avec le SHA du commit.
3. **Déploiement de l'API** : l'image correspondant exactement à ce SHA est déployée sur le service applicatif.
4. **Console** : compilation avec l'URL de l'API injectée au moment du build, puis téléversement vers le stockage statique.
5. **Démarrage** : le conteneur applique les migrations en attente, puis la sonde de santé vérifie l'accès à la base.

Un groupe de concurrence garantit un seul déploiement à la fois, dans l'ordre des fusions.

L'application mobile est construite séparément avec EAS Build (APK de test et AAB pour Google Play).

Exemples génériques : [`../snippets/github-actions-exemple.yml`](../snippets/github-actions-exemple.yml) et [`../snippets/dockerfile-exemple`](../snippets/dockerfile-exemple).
