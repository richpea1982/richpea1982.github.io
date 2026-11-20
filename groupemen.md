---
layout: default
title: Projet Groupemen
nav_order: 5
lang: fr
---

# Projet Groupemen — environnement Windows de pré-production

Dans le cadre d'un stage d'administration systèmes et réseaux, j'ai construit de bout en bout un environnement Windows isolé reproduisant un modèle d'identité de production. Il sert de support aux équipes développement et systèmes pour intégrer des services sans toucher à la production. La publication de cette description a été autorisée ; aucun nom de domaine interne, aucune adresse IP et aucun identifiant n'est divulgué.

## Topologie

```text
                         +----------------------------------+
                         | Contrôleur de domaine            |
                         | Windows Server                   |
                         | AD DS · DNS · Autorité de cert.  |
                         +---+--------------+---------------+
                             | LDAPS 636    | Certificats
                             | DNS 53       | (LDAPS + web)
   Utilisateurs              v              v
   (navigateur) --> +-------------------------------+
   HTTPS 443        | Serveur GLPI (Linux, Docker)  |
                    | GLPI · MariaDB · Caddy (TLS)  |
                    +-------------------------------+

   VM TSE --> Contrôleur de domaine : identité, DNS
   VM TSE --> Serveur WSUS : HTTP 8530, approbation des KB
```

### Composants

- **Contrôleur de domaine** — Windows Server, AD DS, DNS et autorité racine d'entreprise. Tous les rôles FSMO y sont portés. L'administration utilise un compte nommé plutôt que le compte intégré `Administrateur`.
- **Serveur GLPI** — Linux, Docker Compose, GLPI 12, MariaDB 11 et Caddy pour la terminaison TLS. L'image applicative est épinglée sur une version testée.
- **Serveur WSUS** — Windows Server, distribution des correctifs en HTTP sur le port 8530, groupes cibles et approbation manuelle des mises à jour. La procédure détaillée existe encore sous forme de notes de travail et n'est pas présentée comme un document finalisé.
- **Parc TSE** — 24 machines virtuelles de bureau à distance réparties sur quatre hôtes et deux sites.

## Choix de conception

| Choix | Raison |
| --- | --- |
| Adresses statiques pour les serveurs | Le service DHCP reste sur le routeur ; aucun serveur, en particulier le contrôleur de domaine, n'en dépend. |
| Autorité racine d'entreprise unique | Elle signe le certificat LDAPS et le certificat web de GLPI : un seul point de confiance à distribuer. |
| LDAPS sur le port 636 | Évite d'envoyer les identifiants de liaison en clair sur le port 389. |
| Comptes GLPI créés à la première connexion | Un seul couple identifiant/mot de passe pour le domaine et pour GLPI, sans administration utilisateur unitaire. |
| Images GLPI épinglées | Évite les incompatibilités entre versions de PHP et de MariaDB. |
| Environnement isolé de la production | Aucun routage vers la production et aucun risque de collision de noms DNS. |

## Flux réseau

| Source | Destination | Protocole | Objectif |
| --- | --- | --- | --- |
| Serveur GLPI | Contrôleur de domaine | DNS 53 | Résolution du domaine. |
| Serveur GLPI | Contrôleur de domaine | LDAPS 636 | Liaison à l'annuaire et authentification des utilisateurs. |
| Navigateurs | Serveur GLPI | HTTPS 443 | Interface web avec un certificat émis par l'AC. |
| VM TSE | Contrôleur de domaine | Kerberos / DNS | Appartenance au domaine et identité des utilisateurs. |
| VM TSE | Serveur WSUS | HTTP 8530 | Distribution des correctifs approuvés. |

## Automatisation du patch management

J'ai développé un rôle Ansible qui automatise le patch management du parc de 24 VM TSE réparties sur quatre hôtes et deux sites : contrôles de prérequis et de sécurité, exécution des tests, puis déploiement. Le rôle distingue deux sources de mises à jour — le catalogue Microsoft en mode pilote et WSUS comme unique voie de production.

Un contrôle de parité compare le produit, le numéro KB et la révision entre le correctif testé et celui proposé par WSUS. Tant que les trois valeurs ne correspondent pas, le rapport reste en attente de vérification plutôt que de certifier une version qui ne sera pas livrée. Cela évite de confondre « correctif testé » et « correctif distribué ».

## Validation

- Contrôleur de domaine promu, rôles FSMO détenus, résolution de noms et réplication saines.
- LDAPS joignable et approuvé ; la recherche dans l'annuaire renvoie les utilisateurs attendus.
- GLPI servi en HTTPS avec un certificat émis par l'autorité de certification.
- Utilisateurs du domaine connectés à GLPI avec leur identifiant et leur mot de passe de domaine, validé sur plusieurs comptes.
- Invités arrêtés redémarrés par Wake-on-LAN, depuis le même réseau et depuis un réseau routé.

## Limites assumées

- L'autorité de certification est co-localisée sur le contrôleur de domaine. Les bonnes pratiques recommandent de séparer l'AC racine du contrôleur : cet écart est identifié et reste à réévaluer.
- L'environnement ne contient aucune donnée de production et n'est pas routable vers la production.
- La documentation de WSUS reste en notes de travail ; elle n'est pas présentée ici comme une procédure finalisée.
- L'API REST de GLPI et les tests de jetons n'ont pas été réalisés dans cet environnement.

## Pages liées

- [← Accueil](/)
- [IaC et automatisation →](/iac-automation.html)
