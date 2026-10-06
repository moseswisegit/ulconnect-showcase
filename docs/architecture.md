# Architecture

Ce document décrit les composants d'ULconnect, la circulation des données et le déploiement. Les noms de ressources sont volontairement génériques.

## Vue d'ensemble

Le projet est composé de trois applications dans un même dépôt (monorepo) :

| Application | Rôle | Technologie |
|---|---|---|
| Application mobile | Application des étudiants (Android et iOS) | Expo SDK 57, React Native 0.86, React 19, TypeScript |
| API | Logique métier, données, temps réel, tâches planifiées | ASP.NET Core 8 (C#), Entity Framework Core 8 |
| Console d'administration | Modération, gestion, espace organisateur d'événements | React 19, Vite 8, TypeScript, TanStack Query |

Le diagramme source se trouve dans [`../diagrams/architecture.mmd`](../diagrams/architecture.mmd).

## Composants

### Application mobile

- Une seule base de code pour Android et iOS, construite avec EAS Build.
- Navigation par onglets et piles (React Navigation 7).
- Deux langues (français, anglais) et thème clair ou sombre, réglable ou calqué sur le téléphone.
- Jetons stockés dans le stockage sécurisé du système (trousseau iOS, Keystore Android).
- Verrouillage biométrique facultatif.
- Client SignalR pour la messagerie et les notifications en direct.
- Un client HTTP centralisé gère le renouvellement automatique du jeton d'accès sur réponse 401.

### API

- ASP.NET Core 8, contrôleurs REST (50 contrôleurs, 256 routes HTTP déclarées), validation avec FluentValidation.
- Entity Framework Core 8 : PostgreSQL en production, SQLite en développement local et dans les tests.
- Hub SignalR pour la messagerie : messages, indicateur de saisie, réactions, notifications en direct. Le service Azure SignalR peut être branché par configuration si l'API passe à plusieurs instances.
- Quatre services d'arrière-plan hébergés dans l'API :
  - file d'envoi des notifications push, par lots ;
  - rappels de séances d'étude et alertes de sécurité des rendez-vous ;
  - purge des comptes inactifs (avis à 23 mois, suppression à 24 mois) ;
  - résumé hebdomadaire par courriel.
- Cache de sortie et cache partagé de courte durée pour les lectures identiques pour tous les utilisateurs.
- Sonde de santé qui vérifie l'accès à la base.

### Console d'administration

- Application monopage React compilée en fichiers statiques.
- Session dans un cookie HttpOnly, protégée par un jeton anti-CSRF.
- Pages d'administration (vue d'ensemble, file de modération, utilisateurs, équipe, journal d'audit, événements, modules de la communauté, études, parcours, paramètres, modèle économique) et un espace organisateur d'événements séparé.

## Données

- 58 tables, créées par 43 migrations Entity Framework Core.
- Les migrations sont appliquées au démarrage du conteneur.
- Les requêtes SQL propres à PostgreSQL (index spécialisés, par exemple) sont conditionnées au fournisseur actif, pour que les migrations restent exécutables sur SQLite dans les tests.
- Pagination par curseur (keyset) sur les listes longues : découverte de profils, listes de la console ; messages d'une conversation paginés par lots de 50.
- Les clés de protection des données ASP.NET Core sont persistées en base, ce qui permet de redémarrer ou remplacer le conteneur sans invalider les secrets chiffrés.

## Stockage des fichiers

- Photos de profil et documents dans un stockage Blob privé.
- Les photos ne sont jamais servies par une URL publique : l'API produit des URL signées et temporaires.
- Les métadonnées des images (position GPS, appareil) sont retirées à l'envoi.

## Communication temps réel

1. L'application ouvre une connexion SignalR authentifiée par le jeton d'accès.
2. Chaque utilisateur rejoint ses groupes (conversations à deux, équipes, groupes de révision).
3. Un message envoyé par l'API REST est enregistré en base puis diffusé aux membres connectés du groupe.
4. Si le destinataire n'est pas connecté, une notification push est mise en file. Les notifications sensibles sont neutres sur l'écran verrouillé (aucun nom ni contenu).
5. L'appartenance au groupe est contrôlée côté serveur avant toute diffusion, y compris pour l'indicateur de saisie.

## Services externes

| Service | Usage |
|---|---|
| Relais SMTP (Brevo) | Codes de connexion, invitations à la console, résumé hebdomadaire. Domaine d'envoi authentifié par SPF, DKIM et DMARC. |
| Expo Push | Acheminement des notifications vers APNs (Apple) et FCM (Google) |
| Cloudflare | DNS et certificat TLS devant le site statique de la console |

## Déploiement

| Élément | Hébergement |
|---|---|
| API | Image Docker (multi-étapes, utilisateur non root) déployée sur Azure App Service (Linux, conteneur) |
| Base de données | PostgreSQL géré sur Azure |
| Photos | Azure Blob Storage, conteneur privé |
| Console | Fichiers statiques sur Azure Blob Storage (fonction « site web statique »), servis à travers Cloudflare |
| Application mobile | Builds EAS (Expo Application Services) |

Les secrets (chaîne de connexion, clé de signature JWT, identifiants SMTP) sont fournis par les paramètres d'application de l'hébergeur. Ils ne sont ni dans le dépôt ni dans l'image Docker. Hors environnement de développement, l'API refuse de démarrer si la clé JWT (ou une clé trop courte) ou la configuration du stockage Blob est absente.

## Limites connues de l'architecture

- L'API tourne volontairement sur une seule instance, pour simplifier. Les tâches de fond s'exécutent dans son processus, sans verrou distribué ; passer à plusieurs instances demandera de les isoler ou de les verrouiller.
- Les migrations au démarrage sont simples à opérer mais couplent le déploiement du code et le schéma.
