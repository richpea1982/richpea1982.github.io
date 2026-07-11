---
layout: default
title: Sauvegarde & Reprise
nav_order: 7
---

# Sauvegarde & Reprise (Backup & Disaster Recovery)

La stratégie de sauvegarde et le Plan de Reprise d'Activité (PRA) de ce homelab reposent sur la séparation stricte entre le plan de calcul (le cluster K3s/Ceph) et le plan de gestion (`pve1` et le NAS). En cas de sinistre majeur, l'infrastructure est conçue pour être reconstruite de manière déterministe grâce à l'approche *Infrastructure-as-Code*.

---

## 1. Topologie des Sauvegardes

Le système de sauvegarde est centralisé, immuable et physiquement séparé des hyperviseurs de production.

* **Proxmox Backup Server (PBS)** : Hébergé sous forme de machine virtuelle sur le nœud de gestion hors-cluster `pve1`. Il se charge de capturer des instantanés (snapshots) réguliers et dédupliqués de l'ensemble des VMs (comme les instances WordPress `hantaweb` et `petitsanglais`) et des conteneurs LXC.
* **Le Socle de Stockage (NAS Bare-Metal)** : Les données de sauvegarde ne résident pas sur `pve1`. Elles sont stockées sur le NAS physique, au sein du pool ZFS RAID-Z2, garantissant une tolérance à la panne de deux disques physiques.
* **Optimisation ZFS** : Des jeux de données (*datasets*) spécifiques ont été créés pour optimiser les performances de sauvegarde, notamment `pbs-backups`, `pve1-backups` et `k3s-backups`, tous configurés avec un `recordsize` de `1m` adapté aux gros blocs de données.

## 2. Sauvegarde des États et Configurations

Dans une architecture orientée GitOps, la sauvegarde du "code" et de "l'état" est tout aussi critique que celle des données applicatives.

* **Code Source (Source de Vérité)** : L'intégralité de la configuration, des modules Terraform aux playbooks Ansible en passant par les chartes Helm, est versionnée sur le dépôt Git distant `infra-homelab`.
* **Terraform State (MinIO S3)** : L'état de l'infrastructure provisionnée (`terraform.tfstate`) est stocké de manière sécurisée sur le backend S3 MinIO auto-hébergé sur le NAS. La fonction *Object Locking* de MinIO y est activée pour empêcher toute corruption ou écrasement accidentel du state lors des pipelines CI/CD.
* **Configurations Applicatives** : Les bases de données internes (comme celles de K3s/Etcd ou de l'interface Semaphore) bénéficient de sauvegardes automatisées dirigées vers le dataset ZFS `k3s-backups` via des points de montage NFS.

## 3. Sauvegarde Externalisée Hors Site — Amazon S3 Glacier (Planifié)

Une fois les bibliothèques média (Jellyfin, Photoprism) entièrement indexées et validées, une copie froide sera répliquée vers **Amazon S3 Glacier** afin de compléter la règle 3-2-1 avec une copie véritablement hors site, en dehors de mon réseau physique — ce que ni PBS ni le NAS ne peuvent offrir seuls.

**Pourquoi Glacier plutôt qu'un S3 standard ou un second NAS distant :**
* Le coût au Go est très faible comparé à un stockage S3 standard ou à un NAS distant loué, ce qui est adapté à des volumes de données importants (photos, médias) rarement consultés.
* Ces données sont en grande partie irremplaçables (photos personnelles) mais n'ont pas besoin d'être disponibles immédiatement en cas de sinistre — contrairement au state Terraform ou au cluster K3s, qui doivent être restaurables en quelques minutes, une bibliothèque photo peut attendre plusieurs heures.

**Limitations connues, et pourquoi elles sont acceptables pour cet usage :**
* **Délai de récupération** : selon la classe choisie, la restauration peut prendre de quelques minutes (Expedited, coûteux) à plusieurs heures (Standard), voire jusqu'à 12h (Bulk / Deep Archive). C'est acceptable car Glacier n'est pas ma ligne de restauration rapide — ce rôle reste assuré par PBS et le ZFS local, qui restent la première ligne de défense.
* **Coûts de sortie (egress)** : une restauration complète depuis Glacier peut générer des frais de sortie significatifs chez AWS. Ce risque est accepté car ce scénario correspond à un cas de dernier recours (perte simultanée du NAS et de PBS), jugé statistiquement rare compte tenu de la redondance déjà en place (RAID-Z2 + PBS).
* **Durée de rétention minimale facturée** : Glacier impose une durée minimale de stockage facturée (90 jours en Flexible Retrieval, 180 jours en Deep Archive). Cela correspond bien à l'usage prévu ici : un dépôt d'archivage long terme, pas un stockage à rotation fréquente.

**Mise en œuvre prévue :**
* Synchronisation planifiée (`aws s3 sync` ou `rclone`), déclenchée mensuellement via un job Ansible plutôt qu'en continu, pour limiter les coûts de requêtes.
* Chiffrement côté client avant l'envoi, afin de ne jamais exposer de données en clair chez un tiers.

## 4. Plan de Reprise d'Activité (Disaster Recovery)

En cas de perte matérielle totale ou de corruption sévère du cluster de calcul (`pve2`, `pve3`, `pve4`), la procédure de reconstruction à froid (*Cold Recovery*) suit un ordonnancement strict :

1. **Restauration du Cœur de Gestion** :
   - Reconstruction ou redémarrage du **NAS Bare-Metal** pour retrouver l'accès immédiat aux archives ZFS et au service MinIO.
   - Restauration du nœud physique `pve1` (incluant OPNsense pour le routage, PBS pour l'accès aux archives, et le nœud d'automatisation Ansible).
2. **Récupération du State et du Réseau** :
   - Le nœud d'automatisation se reconnecte au bucket `homelab-tf-state` sur MinIO pour récupérer l'état exact de l'infrastructure avant le crash.
3. **Provisionnement Automatisé (IaC)** :
   - Exécution de `terraform apply` pour recréer instantanément les machines virtuelles du cluster de calcul à leur état nominal stable (CPU, RAM, VLANs).
4. **Bootstrapping K3s & GitOps** :
   - Exécution des playbooks Ansible pour reformer le cluster Kubernetes Haute Disponibilité.
   - Le contrôleur Helm interne synchronise le dépôt Git et redéploie l'intégralité de la pile logicielle (Traefik, CrowdSec, outils de monitoring).
5. **Restauration des Données Stateful** :
   - Réinjection des données critiques (bases de données WordPress, volumes persistants spécifiques) à partir des snapshots Proxmox Backup Server directement vers le stockage distribué Ceph fraîchement reconstruit.

---

* **[← Accueil](/index.html)**
