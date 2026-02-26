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



# 🛡️ Manuel d'Exploitation — PRA / PCA Kubernetes  

**Application :** Flask + SQLite  
**Infrastructure :** Cluster K3d (1 Master, 2 Workers)  
**Image :** `pra/flask-sqlite:1.2`  
**Dernière mise à jour :** 26 Février 2026  

---

## 📌 1. Vue d’Ensemble de l’Architecture

L’architecture repose sur :

- La conteneurisation de l’application Flask  
- L’utilisation de Volumes Persistants (PVC) pour la gestion de l’état (*Stateful*)  
- Un mécanisme de sauvegarde automatisée via CronJob Kubernetes  

### 🔹 Composants

| Composant | Description |
|-----------|------------|
| Pod Flask | Application conteneurisée |
| PVC `pra-data` | Stockage production (`/data/app.db`) |
| PVC `pra-backup` | Stockage sauvegarde (`/backup/`) |
| CronJob Kubernetes | Copie du fichier SQLite toutes les minutes |

---

## 📊 2. Indicateurs de Résilience (SLA)

### 🎯 RPO — Recovery Point Objective

- **Objectif : 1 minute**
- Correspond à la fréquence du CronJob
- Perte maximale admissible : ≤ 1 minute de données

### ⏱️ RTO — Recovery Time Objective

- **Objectif : 5 à 15 minutes**
- Temps nécessaire pour restaurer manuellement le service après sinistre majeur

---

# 🟢 3. Plan de Continuité d’Activité (PCA)

## 📌 Scénario : Crash ou suppression accidentelle du Pod

Kubernetes assure nativement la haute disponibilité via un **Deployment**.

### 🔍 Fonctionnement

1. **Détection**  
   Le Control Plane détecte que le nombre de réplicas est `0` au lieu de `1`.

2. **Auto-guérison**  
   Un nouveau Pod est automatiquement recréé.

3. **Persistance**  
   Le Pod se reconnecte au PVC `pra-data` existant  
   → **Zéro perte de données**

### 🧪 Commande de test

```bash
kubectl -n pra delete pod -l app=flask
```

---

# 🔴 4. Plan de Reprise d’Activité (PRA)

## 📌 Scénario : Perte ou corruption du volume `pra-data`

En cas de perte du stockage de production, une intervention manuelle est requise pour restaurer les données depuis `pra-backup`.

---

## 🔄 Procédure de Restauration Standard

### 1️⃣ Arrêt du service

```bash
kubectl -n pra scale deployment flask --replicas=0
```

### 2️⃣ Réinitialisation

Recréer le PVC vide :

```bash
kubectl apply -f k8s/
```

### 3️⃣ Restauration

Exécuter le Job `sqlite-restore` qui copie le dernier backup valide.

```bash
kubectl apply -f pra/50-job-restore.yaml
```

### 4️⃣ Redémarrage

```bash
kubectl -n pra scale deployment flask --replicas=1
```

---

## 🎯 Procédure de Restauration Ciblée (Atelier 2)

Utilisée pour restaurer un point de sauvegarde spécifique  
(ex : corruption de données ancienne)

### 🔎 1. Identifier le backup

```bash
kubectl -n pra exec -it deployment/flask -- ls /backup/
```

### ⚙️ 2. Configurer le Job

Modifier `pra/50-job-restore.yaml`  
et spécifier le fichier cible dans la commande `cp`.

### ▶️ 3. Exécuter

Supprimer le Job existant puis :

```bash
kubectl apply -f pra/50-job-restore.yaml
```

---

# 📡 5. Monitoring & Observabilité

L’état de santé du système de sauvegarde est disponible via l’endpoint :

```
GET /status
```

### 📄 Champs retournés

| Champ | Description | Seuil d’alerte |
|-------|------------|---------------|
| `count` | Nombre d’entrées en base | `N < 1` (si données attendues) |
| `backup_age_seconds` | Âge du dernier backup | `> 120` secondes |
| `last_backup_file` | Nom du dernier fichier | `"none"` |

---

# ⚠️ 6. Analyse des Risques & Évolutions

## 🚨 Risques Identifiés

### ❌ SPOF (Single Point of Failure)

Le disque physique du nœud héberge :

- La production
- Les sauvegardes

→ Sa panne entraîne une perte totale des données.

### 🌍 Absence de réplication hors-site

- Pas d’externalisation vers Cloud Storage / S3  
- Risque élevé en cas de sinistre majeur  

---

# 🚀 Recommandations Professionnelles

### ☁️ Externalisation des sauvegardes

Envoyer les backups vers :

- Stockage immuable (S3 compatible)
- Bucket versionné

### 🌎 Multi-AZ

Déployer le cluster sur plusieurs zones de disponibilité afin d’éviter les pannes matérielles localisées.

### 🗄️ Base de données managée

Migrer SQLite vers une base managée :

- PostgreSQL  
- Cloud SQL  
- Service DB avec réplication synchrone native  

---

# ✅ Résumé

| Élément | Statut |
|----------|--------|
| Auto-healing Pod | ✅ Kubernetes |
| Sauvegarde automatique | ✅ CronJob |
| RPO | 1 min |
| RTO | 5–15 min |
| PRA manuel | Documenté |
| SPOF disque | ⚠️ À corriger |

---