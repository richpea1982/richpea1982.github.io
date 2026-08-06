## 🧪 Tests & Vérification : Reconstruction Complète du Cluster et Restauration etcd

La documentation et l'architecture cible sont une chose ; prouver que le
scénario de reprise fonctionne réellement en conditions réelles en est
une autre. Cette section couvre le test d'infrastructure le plus complet
réalisé à ce jour sur ce homelab : une reconstruction complète des nœuds
K3s combinée à une restauration etcd depuis un snapshot hébergé sur S3 —
déclenchée, de manière inattendue, par un incident Terraform plutôt que
par un exercice planifié.

### Ce Qui S'est Passé

Un `terraform apply` destiné à cibler un seul nœud K3s a en réalité
détruit et recréé les trois VMs de nœuds K3s. La cause racine n'est pas
encore confirmée — l'hypothèse principale est un problème d'échappement
shell avec le flag `-target` de Terraform appliqué à un module indexé par
`for_each`, bien que `-target` soit explicitement documenté par HashiCorp
comme un outil de dernier recours plutôt que comme un mécanisme de
ciblage garanti sûr en soi — et c'est probablement la leçon la plus
durable à en tirer, davantage qu'une simple erreur de commande isolée.

Cet incident a transformé une opération de maintenance ordinaire en un
test de reprise après sinistre non planifié mais complet : reconstruire
trois nœuds serveurs K3s via Terraform et restaurer l'intégralité de
l'état du cluster depuis le snapshot etcd le plus récent stocké sur un
espace de stockage objet compatible S3 (MinIO), sans aucun avertissement
ni préparation préalable.

### Le Parcours de Récupération

1. **Reconstruction des nœuds** — les trois VMs de nœuds K3s recréées via
   le module Terraform existant.
2. **Restauration etcd** — le premier nœud restauré directement depuis le
   snapshot etcd le plus récent, à l'aide du mécanisme natif
   `--cluster-reset-restore-path` de k3s, pointant vers le bucket S3
   hébergé sur le NAS ; les deux nœuds restants ont rejoint le cluster
   restauré.
3. **Friction post-restauration** — un snapshot etcd restaure l'*état des
   objets* Kubernetes (Deployments, PersistentVolumeClaims, Secrets),
   mais pas l'*état au niveau du nœud*. Les nœuds reconstruits ne
   disposaient pas des paquets système requis (`nfs-common`), et le
   backend de stockage Ceph RBD conservait des verrous d'attachement de
   volumes obsolètes référençant les identités des nœuds précédents. Ces
   deux points ont dû être résolus avant que les charges de travail
   dépendantes (notamment le volume NFS des originaux de Photoprism) ne
   puissent revenir à un état sain.
4. **Automatisation partielle, lacune confirmée** — le rôle Ansible
   existant a géré proprement la configuration générale de l'hôte, mais
   ne dispose actuellement d'aucun chemin de code pour le cas « ce nœud
   est en cours de restauration depuis un snapshot etcd ». Cette étape,
   ainsi que la levée des verrous de stockage obsolètes, a été effectuée
   manuellement. Il s'agit désormais d'un chantier de suivi identifié et
   cadré, et non d'une lacune dissimulée.
5. **Vérification** — confirmation de la bonne santé de l'ensemble des
   pods du cluster (plan de contrôle, plugins de stockage, pile de
   sécurité, et charges de travail applicatives) via `kubectl get pods
   -A` après la récupération.

### Pourquoi Cela Compte

Un mécanisme de sauvegarde *correctement configuré* et une sauvegarde
*réellement restaurable en conditions réelles* sont deux affirmations
différentes. Une vérification antérieure avait confirmé que la
planification des snapshots etcd s'exécutait correctement et produisait
des snapshots valides sur S3 ; cet incident est la première fois qu'une
restauration complète a été menée jusqu'à un état de cluster sain et
vérifié — la preuve la plus solide des deux, même si (ou précisément
parce que) elle n'était pas planifiée.

Cet exercice a également fait émerger un chantier futur concret et bien
délimité : étendre le rôle Ansible de bootstrap du cluster avec un chemin
dédié à la restauration depuis un snapshot, afin que la prochaine
récupération soit entièrement automatisée plutôt que de nécessiter une
intervention manuelle.
