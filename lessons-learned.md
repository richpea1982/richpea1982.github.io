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

# Addendum — Reconstruction Complète du Cluster & Exercice de Reprise après Sinistre etcd
 
**Note de statut (pour toi, pas pour publication telle quelle) :** la
section sur la restauration etcd ci-dessous est une reconstruction a
posteriori faite au mieux, pas un journal de commandes vérifié. Considère
les flags/l'ordre exact comme « probablement corrects », pas « certainement
corrects » — si tu reconstitues un jour la chronologie plus précisément
(historique shell, horodatages `journalctl`), affine ce texte avant qu'il
ne devienne définitivement public. J'ai signalé les points fragiles au fil
du texte.
 
---
 
## Incident A — `terraform apply -target` a détruit l'intégralité du parc
K3s au lieu d'un seul nœud
 
### Symptôme
Un `terraform apply` ciblé avec `-target` sur un seul nœud K3s a abouti à
la destruction et à la recréation par Terraform de **l'ensemble des trois**
VMs de nœuds K3s (`k3s-pve2`, `k3s-pve3`, `k3s-pve4`), et non du seul nœud
ciblé.
 
### Ce qui est réellement établi
- L'intention était de cibler une seule instance
  `module.k3s_node["<clé>"]`.
- Le résultat a été une destruction/recréation complète sur l'ensemble
  `for_each` `k3s_nodes`.
- La commande exacte exécutée, ainsi que la sortie du plan Terraform
  affichée avant validation, n'ont pas été capturées — c'est la plus
  grande lacune pour reconstituer la cause racine avec certitude.
### Cause racine la plus probable (hypothèse, non confirmée)
Compte tenu de la structure des modules dans `terraform/main.tf`, deux
explications candidates correspondent au symptôme, et elles ne
s'excluent pas mutuellement :
 
1. **Problème d'échappement/de guillemets shell sur l'index `for_each`.**
   Une valeur `-target` du type `module.k3s_node["pve2"]` contient des
   caractères (`[`, `]`, `"`) que la plupart des shells dénaturent si
   elle n'est pas soigneusement mise entre guillemets (ex. :
   `-target='module.k3s_node["pve2"]'`). Si les guillemets étaient
   incorrects, le flag peut échouer silencieusement à être interprété
   comme prévu — certains shells transmettent une chaîne de ciblage
   malformée, Terraform peut renvoyer une erreur, mais selon la version
   de Terraform et la façon dont l'erreur a été gérée dans le terminal,
   il est possible que l'apply se soit poursuivi comme un apply **non
   ciblé** plutôt que d'échouer proprement.
2. **`-target` restreint l'*apply*, pas la surface de risque du
   *plan*.** `-target` est explicitement documenté par HashiCorp comme un
   outil de dernier recours (« break-glass »), pas un mécanisme de
   ciblage sûr au quotidien — Terraform évalue toujours le graphe de
   dépendances complet pour déterminer ce dont la ressource ciblée a
   besoin. Selon la dérive d'état (*state drift* — par exemple si le
   checksum/URL partagé de l'image `debian_cloud`, ou le snippet
   `vendor_data` par nœud, avait changé depuis le dernier apply), un
   apply `-target` peut malgré tout déclencher le remplacement de
   ressources dont le nœud ciblé ne « dépend » pas de manière évidente
   pour un humain, si le graphe de Terraform en décide autrement.
**Les deux hypothèses sont plausibles avec la structure actuelle du dépôt,
et aucune ne peut être écartée sans la sortie réelle du plan de cette
exécution** — c'est précisément pourquoi une reproduction sécurisée est
nécessaire, et non une supposition traitée comme un fait acquis.
 
### Pourquoi cela dépasse l'incident immédiat
Il ne s'agit pas simplement « d'une mauvaise commande » — cela signifie
qu'**on ne peut actuellement pas faire confiance à `-target` comme
mécanisme de ciblage sûr dans ce dépôt** tant que la cause n'est pas
confirmée et qu'un garde-fou n'est pas en place. Chaque future opération
« corriger juste ce nœud-là » risque de reproduire cet incident tant que
ce point n'est pas résolu.
 
### Actions correctives entreprises
- Aucune n'a encore été appliquée au code Terraform lui-même — cet
  addendum constitue la première étape (documenter avant de corriger,
  afin que le correctif vise une cause confirmée, pas une supposition).
### Tests planifiés (à effectuer avant le prochain apply `-target` en
production)
1. **Monter un environnement de test jetable** — une seule VM Proxmox à
   faible enjeu (ou même une seconde paire fichier `.tfvars`/state
   dédiée, pointant vers un bucket MinIO de test) reproduisant la forme
   `for_each` de `k3s_nodes` avec 2-3 instances factices. Peu coûteux à
   détruire, rien n'en dépend.
2. **Reproduire l'invocation `-target` exacte** supposée utilisée, avec
   des guillemets corrects, et inspecter la **sortie du plan**
   (`terraform plan -target=... -out=tfplan`, puis `terraform show
   tfplan`) avant d'exécuter `apply`. Vérifier si le plan lui-même
   annonce ne toucher que l'instance ciblée.
3. **Tester délibérément le mauvais échappement suspecté** dans le même
   environnement de test, pour voir s'il reproduit la destruction
   complète du parc ou s'il échoue proprement à la place. C'est le seul
   moyen de transformer l'hypothèse ci-dessus en cause confirmée.
4. **Une fois la cause racine confirmée**, le correctif relèvera
   probablement d'un (ou plusieurs) des éléments suivants :
   - Un motif d'invocation `-target` documenté et prêt à copier-coller
     dans le README (exemple de guillemets corrects, par shell).
   - Une étape de revue `terraform plan` rendue obligatoire avant tout
     `apply` pour une opération ciblée (c'est-à-dire ne jamais faire
     `apply -target` directement — toujours plan-revue-apply en étapes
     séparées).
   - Éventuellement restructurer `k3s_nodes` pour qu'une opération sur un
     seul nœud soit moins sujette à une évaluation involontaire de
     l'ensemble du graphe — à revoir une fois la cause réelle connue, pas
     avant.
5. **Ne plus jamais répéter un apply `-target` non testé** contre le
   cluster en production tant que les étapes 1 à 4 ne sont pas
   effectuées. Avoir perdu les trois nœuds une fois était récupérable
   grâce aux snapshots etcd existants ; ce n'est pas une raison pour
   traiter cela comme un risque mineur à l'avenir.
---
 
## Incident B — Restauration Manuelle d'etcd depuis un Snapshot S3/MinIO
(Exercice de Reprise après Sinistre Non Planifié)
 
### Contexte
À la suite de l'Incident A, les trois VMs de nœuds K3s ont dû être
reconstruites et le cluster restauré depuis le snapshot etcd le plus
récent. Il ne s'agissait pas d'un exercice planifié — c'était la
conséquence directe de l'Incident A — mais cela s'est avéré être la
validation la plus complète, à ce jour, du scénario de reprise après
sinistre : du provisionnement des VMs jusqu'à un cluster restauré et
fonctionnel.
 
### Déroulement (reconstruction faite au mieux)
 
**1. Restauration du premier nœud depuis un snapshot etcd hébergé sur S3,
en utilisant directement les flags de restauration natifs de k3s (pas via
Ansible) :**
 
```bash
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION="v1.31.5+k3s1" sh -s - server \
  --cluster-reset \
  --cluster-reset-restore-path='etcd-snapshot-k3s-pve4-1785823205' \
  --token '<vault_k3s_cluster_token>' \
  --flannel-backend=none \
  --disable traefik \
  --tls-san 10.0.20.20 \
  --etcd-s3 \
  --etcd-s3-endpoint=10.0.10.15:9000 \
  --etcd-s3-bucket=k3s-etcd-snapshots \
  --etcd-s3-access-key='<MINIO_ROOT_USER>' \
  --etcd-s3-secret-key='<MINIO_ROOT_PASSWORD>' \
  --etcd-s3-insecure=true
```
 
Le même processus de rejointure/join a été répété sur les deuxième et
troisième nœuds.
 
**2. Friction rencontrée que la restauration seule ne résout pas :** un
snapshot etcd restaure l'état des *objets* Kubernetes (Deployments, PVCs,
définitions de PV, Secrets, etc.) — il ne restaure **pas** les paquets
système au niveau du nœud, la configuration runtime `kubelet`/`k3s`, ni
ne nettoie les verrous de pilotes de stockage laissés par l'ancienne
identité du nœud. Concrètement :
   - `nfs-common` n'était pas présent sur les nœuds fraîchement
     réimagés, donc les montages NFS (le PV des originaux de Photoprism)
     ont échoué.
   - Les volumes Ceph RBD renvoyaient des erreurs de verrou (« *is still
     being used* ») — le pilote/backend CSI Ceph conservait encore un
     état d'attachement référençant les *anciennes* identités de nœuds
     auxquelles les PV avaient été attachés avant la reconstruction.
**3. Exécution du rôle Ansible `k3s_cluster` existant sur les nœuds
reconstruits.** Cela a résolu une partie réelle de la friction
(installation de paquets, configuration de base) mais **pas la
totalité** — confirmant que le rôle suppose actuellement soit a) une
véritable initialisation fraîche via `--cluster-init`, soit b) une
rejointure propre à un VIP toujours sain. Il **n'existe aucun chemin
représentant « ce nœud a été restauré depuis un snapshot etcd et
nécessite un nettoyage post-restauration ».**
 
**4. Nettoyage manuel encore nécessaire après l'exécution Ansible :**
   - Suppression de certificats TLS invalides/obsolètes et de
     configurations auto-générées héritées de l'identité de nœud
     précédente (fichiers exacts non capturés — si cela se reproduit,
     `journalctl -u k3s` au moment de l'échec de la négociation TLS,
     ainsi que `ls -la /var/lib/rancher/k3s/server/tls/`, seraient les
     bons endroits pour capturer les détails précis la prochaine fois).
   - Redémarrage de `k3s.service` après le nettoyage manuel pour que les
     changements prennent effet proprement.
**5. Réconciliation une fois les blocages au niveau hôte levés :**
   - Éviction des pods bloqués dans un état dégradé en conséquence
     directe des étapes 2 à 4 — notamment Photoprism bloqué en état
     `Init` (en attente du montage NFS pas encore disponible) et
     CrowdSec en boucle de crash.
   - Une fois le problème sous-jacent au niveau hôte réellement corrigé
     (paquet manquant / verrou obsolète), les contrôleurs Kubernetes ont
     eux-mêmes recréé des instances de pods saines sans intervention
     manuelle supplémentaire au niveau des pods — cette partie a
     effectivement fonctionné comme l'auto-guérison GitOps/K8s est censée
     le faire.
**6. Vérification :**
   - `kubectl get pods -A` — confirmation que l'ensemble des composants
     du plan de contrôle, des plugins de stockage (Ceph CSI), de
     CrowdSec, et des charges de travail applicatives étaient sains/en
     état `Running`.
   - *(Recommandé pour la prochaine fois, non confirmé pour celle-ci :)*
     `kubectl get nodes -o wide` pour confirmer que les trois nœuds
     affichent bien les rôles `control-plane,etcd,master` et l'état
     `Ready` ; `kubectl -n argocd get applications` pour confirmer
     qu'ArgoCD a repris proprement la réconciliation GitOps contre l'état
     restauré ; `ceph -s` pour confirmer que le cluster Ceph lui-même est
     bien revenu à `HEALTH_OK` indépendamment de K3s.
### Lacune confirmée : le rôle Ansible `k3s_cluster` ne prend pas en
charge la restauration depuis un snapshot etcd
 
En examinant `ansible/roles/k3s_cluster/tasks/bootstrap.yml`, le rôle
dispose actuellement d'exactement deux chemins :
- **Chemin A (`cluster_exists == true`) :** rejoindre un cluster existant
  et toujours actif via le VIP.
- **Chemin B (`cluster_exists == false`) :** initialisation fraîche via
  `--cluster-init`.
Aucun des deux chemins n'exécute `--cluster-reset
--cluster-reset-restore-path=<snapshot>`. Cela signifie que **la seule
fois où une restauration depuis une sauvegarde était réellement
nécessaire, l'automatisation n'en disposait pas, et l'intégralité de la
récupération est retombée sur un `curl | sh` manuel suivi d'un nettoyage
manuel.** C'est le constat le plus exploitable de tout cet incident — le
discours de reprise après sinistre était jusqu'ici « nous avons des
snapshots etcd », et cet exercice a prouvé que c'est nécessaire mais pas
suffisant sans un chemin d'automatisation conscient de la restauration.
 
### Prochaine étape recommandée (pas encore construite)
Ajouter un troisième chemin à `bootstrap.yml` — par exemple une variable
`k3s_restore_from_snapshot` (par défaut `false`) qui, lorsqu'elle est
activée, exécute la variante `--cluster-reset
--cluster-reset-restore-path=<snapshot>` au lieu du Chemin A/B, et intègre
les étapes de nettoyage manuel identifiées ci-dessus (s'assurer que
`nfs-common` est présent *avant* la tentative de restauration, pas après ;
documenter/automatiser la levée des verrous Ceph RBD obsolètes ; gérer la
régénération des certificats TLS obsolètes). Ce point devrait être cadré
comme un chantier à part entière, pas ajouté à la hâte — la valeur de cet
incident tient au fait qu'il a révélé la lacune en conditions réelles
plutôt que par supposition.
 
---
 
## Enseignements Clés (ajouts à la liste existante « Enseignements Clés
pour les Projets Futurs »)
 
5. **`-target` est un outil de dernier recours, pas un mécanisme de
   ciblage sûr.** C'est le graphe de dépendances de Terraform, et non la
   chaîne de ciblage, qui détermine le rayon d'impact — un apply peut
   toucher des ressources inattendues, en particulier en cas de dérive
   d'état ou d'erreur de guillemets shell. Toujours faire `plan`, revoir,
   *puis* `apply` en étapes séparées pour toute modification ciblée ;
   ne jamais faire confiance aveuglément à `-target` sur une
   infrastructure de production sans avoir d'abord reproduit l'invocation
   exacte dans un environnement de test.
2. **Un snapshot etcd restaure l'état des objets Kubernetes, pas l'état
   du nœud.** Les paquets, la configuration runtime, et les verrous
   détenus par les pilotes de stockage référençant l'ancienne identité du
   nœud survivent tous à une reconstruction et bloqueront activement le
   retour à un état sain des charges de travail, même une fois les objets
   Kubernetes eux-mêmes correctement restaurés. Tout chemin
   d'automatisation de restauration doit prendre cela en compte
   explicitement, pas seulement l'étape `k3s
   --cluster-reset-restore-path` elle-même.
3. **Confirmer que les snapshots etcd fonctionnent (2026-08-01) et
   réussir une restauration complète en conditions réelles sont deux
   niveaux de preuve différents.** Le premier confirme que le mécanisme
   de sauvegarde s'exécute ; le second confirme que la sauvegarde est
   réellement utilisable sous pression. Cet incident a fourni le second,
   et il vaut davantage sur le plan de la preuve que la vérification
   initiale, même s'il s'est produit de manière non planifiée et
   partiellement non documentée.

---


* **[← Retour à l'Accueil](/index.html)**
---
