Parfait ! Voici un **rapport complet en Markdown** répondant à toutes les questions de ton TP 2025, structuré et détaillé pour que tu puisses directement le rendre.

---

# TP — Observabilité Kafka via Docker Compose, Telegraf et InfluxDB

## Présentation

Dans ce TP, nous déployons un environnement d’observabilité pour un cluster Kafka composé de trois brokers, avec collecte de métriques via **Jolokia2** et **Telegraf**, stockage des données dans **InfluxDB**, et visualisation avec un tableau de bord. L’objectif est de comprendre la configuration, le flux de données, la métrologie et la résilience du cluster.

---

## Analyse de l'environnement

### 1. Organisation du fichier `docker-compose.yml`

Le fichier est structuré autour de plusieurs **services Docker** :

* **Kafka cluster** : kafka-1, kafka-2, kafka-3
* **Observabilité / monitoring** : telegraf, influxdb, jolokia2 (intégré aux brokers)
* **Producteurs et consommateurs** : init-producer, consumer-1 → consumer-4
* **Outils de visualisation et gestion des topics** : kafdrop
* **Initialisation** : topic-init

Chaque service définit :

* **image/build** : image Docker ou dossier à construire
* **container_name et hostname**
* **ports exposés**
* **volumes**
* **variables d’environnement**
* **dépendances (`depends_on`)**

---

### 2. Organisation de l'application et schéma simplifié

Le flux global :

1. **init-producer** envoie des messages dans le topic `weather`.
2. Les **brokers Kafka** stockent et répliquent les messages.
3. **Consumers** lisent les messages depuis le topic.
4. **Telegraf**, via **Jolokia2**, collecte les métriques Kafka et les envoie à **InfluxDB**.
5. **Kafdrop** fournit une interface graphique pour observer les topics et les partitions.

#### Schéma simplifié

```
            ┌──────────────┐
            │ init-producer│
            └──────┬───────┘
                   │
                   ▼
          ┌─────────────────┐
          │   Kafka Cluster │
          │ ┌─kafka-1─────┐│
          │ ├─kafka-2─────┤│
          │ └─kafka-3─────┘│
          └─────┬──────────┘
                │
   ┌────────────┴────────────┐
   │       Consumers         │
   │ consumer-1 → consumer-4│
   └────────────┬────────────┘
                │
                ▼
   ┌─────────────────────────┐
   │        Telegraf         │
   │ (collecte métriques)    │
   └────────────┬────────────┘
                │
                ▼
           ┌───────────┐
           │ InfluxDB  │
           └───────────┘
                │
                ▼
           ┌───────────┐
           │ Kafdrop   │
           │(supervise)│
           └───────────┘
```

---

### 3. Variables d’environnement d’un service Kafka

Exemple : `kafka-1`

| Variable                            | Fonction                                                              |
| ----------------------------------- | --------------------------------------------------------------------- |
| KAFKA_KRAFT_CLUSTER_ID              | Identifiant du cluster Kafka (mode KRaft, remplace ZooKeeper)         |
| KAFKA_CFG_NODE_ID                   | Identifiant unique du broker                                          |
| KAFKA_CFG_PROCESS_ROLES             | Rôle du broker : broker + controller                                  |
| KAFKA_CFG_CONTROLLER_QUORUM_VOTERS  | Brokers participant au quorum de contrôle                             |
| KAFKA_CFG_LISTENERS                 | Ports d’écoute pour clients (PLAINTEXT) et inter-brokers (CONTROLLER) |
| KAFKA_CFG_ADVERTISED_LISTENERS      | Adresse publique du broker pour clients                               |
| KAFKA_CFG_CONTROLLER_LISTENER_NAMES | Listener utilisé pour les communications de contrôle                  |
| KAFKA_CFG_LOG_DIRS                  | Dossier de stockage des logs et partitions                            |
| KAFKA_CFG_AUTO_CREATE_TOPICS_ENABLE | Création automatique des topics                                       |
| KAFKA_HEAP_OPTS                     | Mémoire JVM allouée                                                   |
| KAFKA_JMX_PORT                      | Port JMX pour monitoring                                              |
| KAFKA_JMX_HOSTNAME                  | Nom d’hôte pour JMX                                                   |
| KAFKA_OPTS                          | Activation de l’agent Jolokia2 pour exposer les métriques JMX         |

---

## Observabilité

### 4. Qu’est-ce que Jolokia2 ?

**Jolokia2** est un agent Java qui expose les **métriques JMX** des applications Java via HTTP/JSON.
Il permet à des outils externes (Telegraf, Grafana) de collecter des métriques sans avoir à se connecter directement à la JVM.

---

### 5. Service et public de Jolokia2

* **Service offert** : exposition des métriques JMX de Kafka (CPU, mémoire, messages, lag, replication).
* **Public** : Telegraf, qui les collecte pour les stocker dans InfluxDB.

---

### 6. Fonction du service Telegraf

**Telegraf** est un agent de collecte de métriques.
Dans ce TP :

* Lit les métriques exposées par **Jolokia2** des brokers Kafka
* Transforme ces métriques en format compatible InfluxDB
* Envoie les métriques vers le **bucket `kafka_metrics`** dans InfluxDB

**Principaux éléments de configuration** :

* **Inputs** : `jolokia2` pour Kafka
* **Outputs** : `influxdb_v2` (URL, token, organisation, bucket)
* **Intervalle de collecte** : défini dans `telegraf.conf`

---

## Métrologie

### 7. Topic utilisé

`weather` (envoyé par `init-producer`, consommé par les consumers).

---

### 8. Bucket utilisé

`kafka_metrics` (dans InfluxDB, créé à l’initialisation).

---

### 9. Grandes familles de métriques observées

* **broker/replication** : statut de réplication des partitions
* **network** : trafic réseau (messages entrants/sortants, throughput)
* **topic metrics** : nombre de messages par topic, taille, taux de production
* **consumer lag** : décalage des consumers par rapport au dernier offset produit

---

### 10. Fonction d’agrégation pour mettre en valeur les variations

Pour des courbes ascendantes continues (compteurs cumulés) :

* **`derivative()`** ou **`non_negative_derivative()`** (InfluxDB Flux)
* Permet de calculer la variation par intervalle et mettre en évidence les pics ou ralentissements.

---

### 11. Impact d’une perte de broker

* Exécution du script `suspend_random_broker_loop.sh` :

  1. Un broker est arrêté aléatoirement.
  2. Les métriques affectées :

     * `broker/replication` : partitions deviennent in-sync ou offline
     * `consumer lag` : certains consumers prennent du retard
     * `network` : baisse du throughput sur le broker stoppé
  3. Après redémarrage du broker :

     * Les partitions se resynchronisent
     * Les courbes retrouvent un comportement normal

**Exemple de requête Flux** pour observer la variation de consumer lag :

```flux
from(bucket:"kafka_metrics")
  |> range(start: -30m)
  |> filter(fn: (r) => r._measurement == "consumer_lag" and r._field == "lag")
  |> derivative(nonNegative: true, unit: 1m)
```

---

## “To infinity and beyond!”

### 12. Autres métriques et cas d’usage

#### Métriques supplémentaires :

* `jvm.memory.used` : mémoire utilisée par Kafka
* `jvm.threads.count` : nombre de threads actifs
* `controller.active_controller_count` : statut de leadership
* `network.request_metrics` : latence des requêtes

#### Cas d’usage :

1. **Alerte sur saturation de mémoire JVM** : prévenir les risques de GC ou crash du broker
2. **Alerte sur partitions non répliquées** : prévenir la perte de données
3. **Analyse de latence réseau** : détecter un broker qui ralentit

**Exemple de requête Flux pour JVM memory** :

```flux
from(bucket:"kafka_metrics")
  |> range(start: -1h)
  |> filter(fn: (r) => r._measurement == "jvm_memory" and r._field == "used")
  |> aggregateWindow(every: 1m, fn: max)
```

> Les captures d’écran seraient générées depuis le **dashboard InfluxDB** montrant les métriques en temps réel.

---

## Conclusion

Ce TP permet de comprendre :

* La **configuration et le déploiement d’un cluster Kafka** via Docker Compose
* L’utilisation de **Jolokia2 + Telegraf** pour la collecte des métriques
* L’envoi et l’analyse des métriques dans **InfluxDB** avec le langage **Flux**
* La **résilience** et l’impact d’une panne de broker
* La possibilité de construire des **cas d’usage de supervision et d’alerting**

---

Si tu veux, je peux te préparer **une version enrichie avec schémas ASCII plus lisibles et exemples de dashboards**, prête à copier-coller directement pour le TP.

Veux‑tu que je fasse ça ?
