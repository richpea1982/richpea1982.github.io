---
layout: default
title: Rétrospective & Leçons Apprises
nav_order: 5
---

# Rétrospective Technique & Leçons Apprises

Cette page fait office de journal d'ingénierie. Elle regroupe les anomalies réelles, les conflits de dépendances et les erreurs de configuration rencontrés lors du développement et du déploiement de ce homelab, ainsi que les correctifs appliqués. 

Documenter ces résolutions démontre une compréhension approfondie du comportement interne des outils (Terraform, Ansible, Kubernetes) face aux contraintes réelles du matériel et des systèmes d'exploitation.

---

## 🚀 Résolution des Anomalies (Session Log)

### 1. Incompatibilité Microarchitecturale des Conteneurs (SIGILL Faults)
* **Symptôme** : Plusieurs images de conteneurs modernes crashaient instantanément au démarrage avec une erreur de type `SIGILL` (Illegal Instruction) sur les nœuds K3s.
* **Cause** : Par défaut, Terraform provisionnait les VMs Proxmox avec un type de processeur virtuel standard `kvm64`. Or, les chaînes de compilation récentes (notamment la bibliothèque `glibc`) intègrent une base de référence exigeant l'architecture CPU `x86-64-v2`. Le processeur `kvm64` masque les instructions modernes requises.
* **Correctif** : Modification du module Terraform personnalisé pour forcer le mode passthrough : `cpu.type = "host"`. Les conteneurs exploitent désormais directement le jeu d'instructions natif des processeurs physiques sous-jacents (Intel i5, AMD A10, i7).

### 2. Rupture de Compatibilité des CRDs Calico (Moteur de Validation CEL)
* **Symptôme** : L'application des manifests du CNI Calico version `3.32.0` échouait lors de la création des Custom Resource Definitions (CRDs).
* **Cause** : Les règles de validation internes des CRDs de Calico 3.32 exploitent la fonction CEL (Common Expression Language) `isCIDR()`. Cette fonction nécessite une API Kubernetes en version `1.31` minimum. Le cluster K3s était initialement figé sur la version `1.30.2`.
* **Correctif** : Alignement et mise à niveau de la variable globale de version K3s vers la version stable `v1.31.5+k3s1` dans les `group_vars` d'Ansible, résolvant instantanément le conflit syntaxique.

### 3. Masquage de Variables Réseau (VLAN Threading)
* **Symptôme** : Les nœuds du cluster K3s se voyaient attribuer des adresses IP hors-contexte et se retrouvaient positionnés par défaut sur le réseau natif non segmenté de Proxmox.
* **Cause** : Bien que la variable `vlan_id` ait été correctement déclarée dans le fichier `.auto.tfvars` et typée dans les déclarations de variables globales de Terraform, elle n'était pas passée au bloc de ressources final du module de création de VM.
* **Correctif** : Correction de la liaison au sein du code Terraform en insérant explicitement l'argument `vlan_id = var.vlan_id` dans la déclaration de l'interface réseau du module.

### 4. Conflit d'Environnement Python sous Debian (PEP 668)
* **Symptôme** : Le playbook Ansible de bootstrap échouait sur le nœud d'automation lors de la tentative d'installation du paquet python `kubernetes-client` via `pip`.
* **Cause** : Protection stricte introduite par la directive PEP 668 sous Debian 12 (Externally Managed Environment). Le système bloque l'usage de `pip install` global pour éviter d'écraser les paquets gérés par le gestionnaire système `apt` (notamment `PyYAML`).
* **Correctif** : Isolation du processus en configurant Ansible pour exécuter ses tâches Kubernetes au sein d'un environnement virtuel dédié (`virtualenv`) ou en s'appuyant de préférence sur les paquets packagés par la distribution (`python3-kubernetes`).

### 5. Obsolescence de Chart Helm (CrowdSec Lifecycle)
* **Symptôme** : La synchronisation ArgoCD du pod de sécurité CrowdSec restait bloquée à l'état *Degraded* ou renvoyait une erreur de récupération d'artefact HTTP 404.
* **Cause** : Le manifest GitOps pointait vers une version historique du chart Helm (`0.9.3`) qui avait été purgée et archivée définitivement des dépôts amonts du fournisseur.
* **Correctif** : Ajustement du fichier de déclaration GitOps `gitops/apps/crowdsec.yaml` pour cibler la révision stable `0.24.0` du chart, restaurant immédiatement la boucle de réconciliation applicative.

### 6. Chemin de Dépôt GitOps Erroné (Root Application ArgoCD)
* **Symptôme** : L'`Application` racine (`root-app`) affichait un statut `SYNC STATUS: Unknown` persistant, sans jamais découvrir la moindre application enfant (Traefik, CrowdSec, Portfolio, etc.).
* **Cause** : La variable `gitops_repo_path` dans `group_vars/k3s_cluster.yml` pointait vers `k3s/gitops/apps` — dupliquant inutilement le nom du dépôt comme sous-répertoire de lui-même. Le chemin réel, à la racine du dépôt `k3s`, est simplement `gitops/apps`.
* **Correctif** : Correction du chemin dans la variable Ansible, puis application immédiate d'un patch live sur l'objet `Application` (`kubectl patch`) pour valider le correctif avant de le reporter dans la source de vérité.

### 7. Provider Manquant sur `VolumeSnapshotLocation` (Velero / Helm Chart Merge)
* **Symptôme** : La synchronisation ArgoCD de l'application `velero` échouait systématiquement avec l'erreur `VolumeSnapshotLocation.velero.io "default" is invalid: spec.provider: Required value`.
* **Cause** : Les valeurs Helm personnalisées omettaient intentionnellement le bloc `configuration.volumeSnapshotLocation` (le projet utilisant Kopia plutôt que les snapshots natifs CSI/RBD). Or, Helm *fusionne* les valeurs custom avec les valeurs par défaut du chart plutôt que de les remplacer intégralement — omettre une clé ne la désactive pas, elle hérite simplement de la valeur par défaut du chart, qui contenait un objet `VolumeSnapshotLocation` sans provider défini.
* **Correctif** : Déclaration explicite de `configuration.volumeSnapshotLocation: []` dans les valeurs Helm, forçant la désactivation plutôt que l'omission silencieuse.

### 8. Erreur de Signature S3 sur les Identifiants MinIO (Velero)
* **Symptôme** : Les sauvegardes Velero échouaient avec `SignatureDoesNotMatch` sur chaque appel à l'API S3 de MinIO, alors que les identifiants semblaient corrects après inspection visuelle du Secret Kubernetes.
* **Cause** : Le Secret contenant les identifiants MinIO (format INI attendu par le plug-in AWS de Velero) avait été généré via une redirection de fichier ajoutant un caractère de nouvelle ligne parasite en fin de valeur. Ce caractère invisible faisait partie de la chaîne utilisée pour signer les requêtes, invalidant systématiquement la signature calculée côté client sans qu'aucune erreur de format ne soit visible au décodage base64.
* **Correctif** : Régénération du Secret via `printf` (sans retour à la ligne final) plutôt qu'un heredoc, éliminant la source de l'incohérence silencieuse.

### 9. Masquage du Kubeconfig Utilisateur par le Binaire `kubectl` Intégré à K3s
* **Symptôme** : Toute commande `kubectl` exécutée par l'utilisateur non-root échouait avec `permission denied` sur `/etc/rancher/k3s/k3s.yaml`, malgré la présence d'un `~/.kube/config` valide et correctement provisionné par Ansible.
* **Cause** : Le binaire `/usr/local/bin/kubectl` était en réalité un lien symbolique vers `k3s` lui-même (`k3s kubectl`), un raccourci intégré par l'installateur K3s. Ce mode d'exécution ignore l'ordre de résolution standard de `kubectl` (`--kubeconfig` → `$KUBECONFIG` → `$HOME/.kube/config`) et pointe systématiquement vers le fichier kubeconfig système, lisible uniquement par root.
* **Correctif** : Remplacement du lien symbolique par le binaire `kubectl` amont officiel (`dl.k8s.io`), restaurant la résolution standard du kubeconfig utilisateur sans recours à `sudo`.

### 10. Ambiguïté de Propriété de Ressource entre Applications ArgoCD (SharedResourceWarning)
* **Symptôme** : `root-app` affichait un statut `OutOfSync` persistant accompagné d'une `SharedResourceWarning` indiquant qu'un objet `Application` était revendiqué par deux Applications ArgoCD simultanément.
* **Cause** : Une copie résiduelle du manifest de l'`Application` Helm (`ceph-csi-rbd.yaml`) avait été sauvegardée par erreur dans le répertoire des manifests bruts (`gitops/manifests/ceph-csi-rbd/`) plutôt que dans `gitops/apps/` uniquement. L'Application chargée de ce répertoire de manifests appliquait donc, sans le vouloir, une seconde définition du même objet `Application` déjà géré directement par `root-app`.
* **Correctif** : Suppression du fichier dupliqué, ne laissant que le `StorageClass` dans le répertoire des manifests — chaque objet n'étant désormais revendiqué que par une seule Application.

### 11. Incompatibilité entre Topologie CSI Désactivée et Binding Mode `WaitForFirstConsumer`
* **Symptôme** : Le provisionnement de volumes RBD via `ceph-csi-rbd` échouait systématiquement avec `error generating accessibility requirements: no topology key found for node`, bloquant indéfiniment les PVC à l'état `Pending`.
* **Cause** : Le cluster Ceph étant un pool plat sans domaines de panne (aucune notion de zone/région), la topologie CSI avait été désactivée côté driver (`topology.enabled: false`). Cependant, le `StorageClass` conservait `volumeBindingMode: WaitForFirstConsumer`, qui déclenche côté `external-provisioner` une tentative de résolution des exigences d'accessibilité à partir des étiquettes de topologie du nœud sélectionné — étiquettes qui n'existaient plus puisque la topologie était désactivée en amont.
* **Correctif** : Passage à `volumeBindingMode: Immediate`, pertinent ici puisque le pool Ceph est identiquement accessible depuis chaque nœud K3s, sans contrainte de localité à respecter.
* **Complication additionnelle** : `volumeBindingMode` est un champ **immuable** sur un objet `StorageClass` — une tentative de mise à jour via ArgoCD (`selfHeal`) reste bloquée indéfiniment à l'état `OutOfSync` sans jamais échouer explicitement ni se corriger d'elle-même. La suppression manuelle de l'objet existant, suivie d'une resynchronisation forcée, a été nécessaire pour que l'Application recrée le `StorageClass` avec la spécification correcte.

---

## 🔁 Optimisations d'Automatisation et Idempotence

Au-delà des corrections de bugs bloquants, des refactorisations ont été menées pour fiabiliser la robustesse des scripts de déploiement :

* **Vérification d'État vs Présence de Fichiers** : Le playbook Ansible d'initialisation de K3s utilisait un test basé sur la présence de fichiers locaux (`stat`) pour déterminer si le cluster était déjà initialisé. Ce mécanisme masquait les installations partielles ou corrompues. Il a été remplacé par une inspection dynamique de l'état du service systemd (`systemctl is-active --quiet k3s`).
* **Instabilité des Clés d'Hôtes SSH** : Chaque reconstruction de VM via Terraform générait une nouvelle paire de clés d'hôte SSH, provoquant des blocages de sécurité lors des exécutions Ansible suivantes (*Host key verification failed*). Intégration d'une tâche idempotente de nettoyage automatique et de rafraîchissement des clés via `ssh-keyscan` dans la phase préliminaire du playbook.
* **Gestion des Secrets Vault Interrompue** : Erreur d'intégration de la variable `vault_k3s_cluster_token`. Le rôle Ansible avait été mis à jour pour exiger un jeton de sécurité fort partagé pour l'etcd, mais le coffre chiffré Ansible Vault associé n'avait pas été provisionné avec cette nouvelle clé. La variable a été canonicalisée et documentée dans le fichier `vault.yml`.

---

## 📈 Enseignements Clés pour les Projets Futurs

1. **La Frontière des Outils** : Bien délimiter où s'arrête un orchestrateur et où commence le suivant est vital. Confier le déploiement applicatif à ArgoCD et la configuration OS à Ansible (plutôt que de surcharger Ansible avec des modules Helm complexes) simplifie radicalement le débogage.
2. **Ne Jamais Faire Confiance aux Paramètres par Défaut** : Les profils de virtualisation génériques (comme le CPU `kvm64` ou l'absence de tag VLAN) créent des comportements imprévisibles en couches hautes. Valider les configurations matérielles et réseau au plus bas niveau possible avant de monter l'orchestrateur.
3. **L'Omission n'est pas la Désactivation (Fusion de Valeurs Helm)** : Un chart Helm fusionne les valeurs personnalisées avec ses valeurs par défaut plutôt que de les remplacer intégralement. Omettre une clé indésirable dans un fichier de valeurs custom n'a aucun effet si le chart définit déjà une valeur par défaut pour cette même clé — il faut la surcharger explicitement (souvent avec une valeur vide ou nulle) pour la neutraliser réellement.
4. **Les Champs Immuables Bloquent Silencieusement le GitOps** : Certains champs d'objets Kubernetes (comme `volumeBindingMode` sur un `StorageClass`) sont immuables après création. Une réconciliation ArgoCD `selfHeal` contre un tel champ ne provoque ni erreur explicite ni convergence — l'objet reste `OutOfSync` indéfiniment. La suppression manuelle suivie d'une recréation reste, dans ces cas précis, la seule voie de correction via GitOps.

---

* **[← Retour à l'Accueil](/index.html)**
---
