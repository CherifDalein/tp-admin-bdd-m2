# TP — InfluxDB ↔ Telegraf ↔ Kafka

## 🎯 Objectifs

Ce TP a pour but de mettre en place un pipeline **End-to-End (E2E)** permettant d’observer le flux complet de données entre **InfluxDB**, **Telegraf**, **Kafka** et du **code Python**.
L’objectif est de comprendre comment ces outils interagissent dans une architecture distribuée et d’observer le flux de messages via **Kafdrop**.

---

## Partie 1 — Prise en main

### 1️ Démarrage des services

La commande :

```bash
docker compose up
```

lance l’ensemble des conteneurs définis dans le fichier `docker-compose.yml`.
Chaque service se met en route selon sa configuration. Certains restent actifs en continu, d’autres exécutent une tâche ponctuelle avant de s’arrêter.

---

### 2️ Observation des logs

* **InfluxDB** : initialise l’organisation `myorg`, le bucket `weather` et crée un utilisateur admin avec un token (`mytoken`).
* **Kafka1 / Kafka2 / Kafka3** : forment un cluster Kafka avec quorum, affichent la synchronisation et démarrent le serveur (`Kafka Server started`).
* **Telegraf** : démarre et se connecte à Kafka pour produire des métriques.
* **Init-topic**, **init-alert**, etc. : exécutent leurs commandes puis s’arrêtent après l’initialisation.

---

### 3️ Conteneurs exécutés durablement

| Conteneur                          | Fonction principale                                  |
| ---------------------------------- | ---------------------------------------------------- |
| **influxdb**                       | Base de données time-series                          |
| **kafka1**, **kafka2**, **kafka3** | Brokers Kafka (cluster distribué)                    |
| **telegraf**                       | Agent de collecte et d’envoi de données              |
| **kafdrop**                        | Interface Web pour visualiser les topics et messages |

Ces conteneurs restent actifs pour assurer la persistance et la circulation des données.

---

### 4️ Conteneurs temporaires

| Conteneur                                | Fonction                                                            |
| ---------------------------------------- | ------------------------------------------------------------------- |
| **init-topic** / **init-topic-telegraf** | Créent les topics Kafka nécessaires (`weather`, `weather-telegraf`) |
| **init-alert**                           | Configure des alertes dans InfluxDB                                 |
| **inject**                               | Insère des données simulées dans InfluxDB                           |
| **producer**                             | Publie les données d’InfluxDB dans Kafka                            |

Ils s’exécutent rapidement, effectuent leur tâche, puis s’arrêtent.

---

##  Partie 2 — Rôle des différents conteneurs

### 5️ Rôle de Telegraf

**Telegraf** est un agent qui collecte des données (métriques systèmes, capteurs ou bases) et les envoie vers d’autres systèmes.
Dans ce TP, il :

* Lit des données météo simulées,
* Les publie dans un **topic Kafka** nommé `weather-telegraf`.

Il agit comme un **pont entre InfluxDB et Kafka**.

---

### 6️ Rôle du conteneur `init-alert`

Ce conteneur configure des **alertes automatiques** dans InfluxDB via son API REST, par exemple pour signaler une valeur anormale dans les données (ex. température trop élevée).

---

### 7️ Rôle du conteneur `init-topic`

Ce conteneur exécute la commande :

```bash
kafka-topics.sh --create --topic weather --partitions 4 --replication-factor 3
```

Il permet de créer le topic principal `weather` (4 partitions, réplication 3) au démarrage du cluster Kafka.

---

## Partie 3 — Ports mappés

### 8 Tableau récapitulatif des ports

| Service             | Port externe | Fonction                                  |
| ------------------- | ------------ | ----------------------------------------- |
| **InfluxDB**        | 8086         | API HTTP & interface web                  |
| **Kafka1**          | 9092         | Communication avec les clients Kafka      |
| **Kafdrop**         | 9000         | Interface web de gestion Kafka            |
| **Kafka2 / Kafka3** | 9092, 9093   | Communication inter-brokers et contrôleur |

---

### 9️ Kafdrop

* **Port exposé :** `9000`
* **Fonction :** Interface web accessible sur `http://localhost:9000` permettant d’explorer les topics, partitions et messages de Kafka.

---

### Kafka3

* **Ports utilisés :**

  * `9092` → communication avec les producteurs/consommateurs
  * `9093` → communication interne de contrôle dans le cluster (Controller listener)

---

### 11 InfluxDB

* **Port 8086** : interface d’administration et API REST.
  Accessible via `http://localhost:8086`.

---

## Partie 4 — Détails de configuration

### 12️ Bucket InfluxDB

* **Nom :** `weather`
* **Créé par :** le conteneur `influxdb` lui-même
* **Mécanisme :** via les variables d’environnement :

```yaml
DOCKER_INFLUXDB_INIT_BUCKET=weather
```

---

### 13️ Sécurité du déploiement

Le déploiement est fonctionnel mais **peu sécurisé** :

* Identifiants faibles : `admin / adminpass`
* Token (`mytoken`) exposé en clair
* Pas de chiffrement TLS
* Port 8086 accessible publiquement

**Améliorations recommandées :**

* Utiliser des variables d’environnement sécurisées (fichiers `.env`)
* Ajouter TLS/HTTPS
* Restreindre les ports aux réseaux internes Docker
* Modifier les identifiants par défaut

---

### 14️ Observation du topic dans Kafdrop

* **Topic :** `weather`
* **Partitions :** 4
* **Facteur de réplication :** 3
* **Auto-creation :** désactivée, car le topic est créé manuellement par `init-topic`.

---

### 15️ Vérification dans le docker-compose

On retrouve ces paramètres dans la section :

```yaml
kafka-topics.sh --create --topic weather --partitions 4 --replication-factor 3
```

---

## Partie 5 — Résilience de Kafka

### 16️ Arrêt d’un broker

Exemple :

```bash
docker stop kafka2
```

* Le cluster reste disponible grâce à la réplication.
* Certaines partitions peuvent devenir inaccessibles si leur leader était sur `kafka2`.
* Pour corriger : redémarrer le broker (`docker start kafka2`) ou reconfigurer la répartition des leaders.

---

### 17️ Redémarrage d’un broker

Au redémarrage, le broker réintègre automatiquement le cluster et récupère les partitions manquantes.
Kafka rééquilibre ensuite le leadership entre les brokers pour restaurer la haute disponibilité.

---

### 18️ Différence entre `inject` et `producer`

| Service      | Rôle principal                                           |
| ------------ | -------------------------------------------------------- |
| **inject**   | Génère des données et les envoie vers InfluxDB           |
| **producer** | Récupère les données d’InfluxDB et les publie dans Kafka |

Ainsi, `inject` alimente la base, et `producer` alimente le flux Kafka à partir de cette base.

---

## Partie 6 — Aller plus loin

### 19️ Améliorations possibles

* Ajouter **Grafana** pour la visualisation des données InfluxDB.
* Sécuriser la configuration (TLS, tokens, mots de passe).
* Mettre en place une **intégration continue** pour automatiser le déploiement.
* Ajouter un **consumer Python** pour analyser les messages Kafka en temps réel.
* Gérer la persistance et la sauvegarde automatique des volumes Docker.

---

### 20️ Pourquoi encapsuler le code Python dans un conteneur ?

* Isolation complète de l’environnement (pas de conflit de dépendances).
* Portabilité entre machines (le même code fonctionne partout).
* Intégration facile dans le pipeline Docker Compose.
* Reproductibilité et cohérence lors du déploiement.

---

### 21️ Correction du schéma (schema.md)

Le schéma doit montrer clairement le flux complet de données :

```text
           ┌──────────────┐
           │   inject     │
           └──────┬───────┘
                  │
                  ▼
           ┌──────────────┐
           │   InfluxDB   │
           └──────┬───────┘
             │             │
             ▼             ▼
      ┌──────────┐     ┌──────────┐
      │ Producer │     │ Telegraf │
      └────┬─────┘     └────┬────┘
           │                 │
           ▼                 ▼
           ┌─────────────────┐
           │     Kafka       │
           └─────────────────┘
                     │
                     ▼
              ┌────────────┐
              │  Kafdrop   │
              └────────────┘
```

---

## Conclusion

Ce TP a permis de mettre en place et d’observer un pipeline complet reliant **InfluxDB**, **Telegraf** et **Kafka**, en comprenant le rôle de chaque conteneur et la résilience du cluster.
Il illustre parfaitement comment orchestrer des services interconnectés via Docker Compose pour gérer un flux de données temps réel.

