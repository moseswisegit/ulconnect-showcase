# Sécurité et Loi 25

Ce document liste les mesures réellement implémentées dans le code. Il ne constitue pas une certification. Les textes juridiques (politique de confidentialité, conditions d'utilisation, évaluation des facteurs relatifs à la vie privée) n'ont pas encore été relus par un juriste.

## Authentification des étudiants

- Connexion sans mot de passe : un code à 6 chiffres est envoyé par courriel, uniquement à une adresse universitaire vérifiée de façon stricte.
- Le code est haché (SHA-256) avant d'être enregistré, a une courte durée de vie et est annulé après 5 essais ratés.
- Jeton d'accès JWT de courte durée et jeton de renouvellement stocké haché en base.
- Rotation du jeton de renouvellement à chaque usage. Si un jeton déjà révoqué est présenté de nouveau, l'API le traite comme un vol possible et révoque toutes les sessions du compte.
- Chaque jeton porte un type (étudiant, administrateur, étape de double authentification). Les politiques d'autorisation vérifient ce type, pour qu'un jeton de la console ne soit pas accepté par les routes étudiantes, et inversement.

## Authentification de la console d'administration

- Comptes distincts des comptes étudiants, créés uniquement par invitation.
- Mot de passe haché avec PBKDF2-SHA256 (100 000 itérations, sel aléatoire).
- Double authentification TOTP obligatoire, avec écran d'inscription par code QR. Le secret TOTP est chiffré au repos avec l'API de protection des données d'ASP.NET Core.
- Session dans un cookie HttpOnly, avec jeton anti-CSRF exigé sur les requêtes qui modifient des données.
- Six rôles : super administrateur, modérateur senior, modérateur, registraire, lecture seule, organisateur d'événements. Chaque action de la console déclare explicitement les rôles autorisés ; l'organisateur d'événements ne voit que son propre espace.
- Journal d'audit des actions administratives, exportable.

## Limitation de débit

Implémentée avec le middleware de limitation de débit intégré à ASP.NET Core :

- limites globales chaînées : par adresse IP pour la connexion, par utilisateur pour les lectures authentifiées, par IP pour le trafic anonyme ;
- politiques nommées par route sensible : demande de code (3 par 3 minutes), vérification de code, connexion et double authentification de la console (5 par 15 minutes), renouvellement de jeton ;
- politiques par utilisateur pour les actions sociales : messages et mentions « j'aime » (60 par minute), envoi de photos, création d'équipes ;
- limite supplémentaire par adresse courriel sur la connexion, pour qu'un réseau partagé (Wi-Fi du campus) ne bloque pas tout le monde à cause d'une limite par IP.

Les en-têtes transférés par le proxy de l'hébergeur sont pris en compte de façon explicite, faute de quoi toutes les requêtes sembleraient venir de la même adresse et les limites par IP deviendraient globales. Ce point a été corrigé à la suite d'un audit.

## Protection des réponses HTTP

- En-têtes de sécurité : `Strict-Transport-Security`, `X-Frame-Options: DENY`, `X-Content-Type-Options`, politique de sécurité du contenu stricte sur les pages HTML servies par l'API.
- CORS limité à une liste d'origines fournie par configuration.
- Clé de signature JWT d'au moins 256 bits, sinon l'API refuse de démarrer.

## Contenu et modération

- Toutes les publications de la communauté (annonces, événements, idées) sont validées par l'équipe de modération avant d'être visibles.
- Filtre de mots bloqués configurable dans la console.
- Signalements avec file de modération ; un compte signalé par 3 personnes distinctes est masqué automatiquement en attendant une décision humaine.
- Blocage entre utilisateurs, appliqué à toutes les formes de conversation.
- Avertissement, suspension, bannissement depuis la console.

## Données personnelles et fichiers

- Métadonnées des images (position GPS, modèle d'appareil) retirées à l'envoi.
- Photos dans un stockage privé, servies uniquement par des URL signées et temporaires.
- Adresses courriel masquées dans les journaux ; aucun code ni jeton journalisé en production.
- Notifications push neutres sur l'écran verrouillé pour tout contenu sensible.

## Mesures liées à la Loi 25 (Québec)

| Exigence | Mise en oeuvre |
|---|---|
| Consentement | Écran de consentement au premier lancement : confirmation de l'âge (18 ans et plus), acceptation des conditions et de la politique. La version acceptée et la date sont enregistrées ; une nouvelle version redemande le consentement. |
| Transparence | Politique de confidentialité et conditions d'utilisation servies par l'API sous forme de pages publiques. Un responsable de la protection des renseignements personnels est désigné. |
| Accès et portabilité | Export complet des données de l'utilisateur depuis l'application (document PDF généré côté serveur). |
| Suppression | Suppression du compte depuis l'application, photos comprises, et page web de demande de suppression (exigée par les stores). |
| Conservation limitée | Comptes inactifs : avis par courriel à 23 mois, suppression à 24 mois. Codes de vérification et notifications anciennes purgés automatiquement. |
| Paramètres par défaut protecteurs | Le mode Rencontre est désactivé par défaut et nécessite une activation explicite. |
| Incidents | Registre des incidents de confidentialité tenu dans la documentation du projet. |
| Communications hors Québec | Évaluation des facteurs relatifs à la vie privée rédigée pour les sous-traitants (courriel, notifications push). Elle reste à finaliser et à signer. |
| Minimisation | Aucune publicité, aucun outil de pistage, aucune géolocalisation. |

## Audits

- Un premier audit de sécurité a relevé 17 constats. Chacun a été corrigé ou documenté comme risque accepté, avec la justification.
- Un audit complet avant mise en production a relevé des constats classés par gravité (critique, haute, moyenne, basse). Le constat critique et les constats de gravité haute ont été corrigés. Les constats de gravité moyenne et basse restent à traiter.

## Ce qui n'est pas en place

- Aucun test d'intrusion externe.
- Aucune relecture juridique des textes.
- Pas de gestion centralisée des secrets de type coffre-fort : les secrets sont dans les paramètres d'application de l'hébergeur.
