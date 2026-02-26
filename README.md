Séquence 5 : Exercices
Exercice 1 : Composants critiques et perte de données
Quels sont les composants dont la perte entraîne une perte de données ?

D'après l'architecture analysée, la donnée est définitivement perdue si :

Le volume pra-backup est supprimé : C'est l'unique support qui contient l'historique des sauvegardes générées par le CronJob.

Le "Disque du node" subit une panne matérielle : Comme les deux volumes (pra-data et pra-backup) sont hébergés sur le même disque physique du nœud, une défaillance de ce dernier détruit simultanément la production et ses sauvegardes.

Le cluster K3d est supprimé : S'agissant d'un environnement éphémère local, la suppression du cluster sans externalisation des données entraîne la perte totale de l'infrastructure et du stockage.

Exercice 2 : Résilience face à la suppression du PVC
Expliquez pourquoi nous n'avons pas perdu les données lors de la suppression du PVC pra-data.

La conservation des données repose sur la mise en place d'un Plan de Reprise d'Activité (PRA) basé sur la redondance :

Sauvegarde périodique : Un CronJob Kubernetes exécute un script Bash chaque minute pour copier la base SQLite du volume de production (pra-data) vers le volume de secours (pra-backup).

Indépendance logique : Bien que situés sur le même disque, les deux PVC sont des entités distinctes pour Kubernetes. La suppression de l'un n'affecte pas l'intégrité des fichiers stockés sur l'autre.

Procédure de restauration : Lors du crash simulé, nous avons utilisé un Job de restauration (50-job-restore.yaml) pour copier manuellement le dernier backup valide vers le nouveau volume de production recréé.

Exercice 3 : Analyse des indicateurs RTO et RPO
Quels sont les RTO et RPO de cette solution ?

RPO (Recovery Point Objective) : 1 minute.
Comme le script de sauvegarde s'exécute toutes les minutes, la perte de données maximale correspond à l'intervalle entre deux backups (soit 59 secondes au maximum avant le crash).

RTO (Recovery Time Objective) : ~5 à 15 minutes.
Ce délai correspond au temps nécessaire pour une intervention humaine : détection de l'incident, mise à l'arrêt du pod, recréation du PVC, exécution du job de restauration et redémarrage du service.

[Image showing RTO and RPO disaster recovery metrics]

Exercice 4 : Limites de la solution en production
Pourquoi cette solution ne peut-elle pas être utilisée dans un vrai environnement de production ? Que manque-t-il ?

Cette solution présente plusieurs faiblesses majeures incompatibles avec un environnement critique :

Single Point of Failure (SPOF) : Le stockage est local au nœud ; une panne serveur entraîne une perte totale.

Absence d'externalisation : Les backups ne sont pas envoyés sur un stockage distant (Cloud Storage, S3), ce qui est indispensable pour un vrai PRA.

Processus manuel : La reprise après sinistre nécessite des actions manuelles complexes, augmentant le RTO et le risque d'erreur humaine.

SQLite : Cette base de données n'est pas conçue pour la haute disponibilité ou des accès concurrents intensifs sur Kubernetes.

Exercice 5 : Proposition d'une architecture robuste
Proposez une architecture plus robuste.

Pour sécuriser réellement l'application, il faudrait :

Externaliser les backups : Envoyer les sauvegardes vers un stockage objet distant (ex: AWS S3 ou Azure Blob Storage) pour garantir la survie des données même si tout le cluster Kubernetes est détruit.

Base de données managée : Remplacer SQLite par un service managé (ex: Cloud SQL ou Amazon RDS) offrant une réplication Multi-AZ (plusieurs zones de disponibilité) et des snapshots automatiques.

Stockage persistant distribué : Utiliser des solutions comme Longhorn, Ceph ou des disques réseau Cloud qui répliquent les données sur plusieurs nœuds physiques.

Automatisation (Failover) : Mettre en place des sondes de disponibilité et des procédures de bascule automatique pour réduire le RTO au minimum.