Parfait ! Comme cette requête fonctionne et affiche bien tes données, on va l’utiliser comme base pour répondre aux questions du TP. Voici les réponses en **Markdown**, adaptées à ton environnement et aux tests réels que tu as faits.

---

# TP étudiants 2025 – Réponses

## Analyse de l'environnement

**1. Comment est organisé le fichier docker_compose.yml**
Le fichier est organisé en **services** Docker :

* Trois brokers Kafka (`kafka-1`, `kafka-2`, `kafka-3`) pour former un cluster KRaft.
* Un container InfluxDB pour stocker les métriques.
* Telegraf pour collecter les métriques via Jolokia.
* Un container `kafdrop` pour visualiser les topics Kafka.
* Des containers producteurs et consommateurs pour injecter et lire des messages Kafka.
* Des containers `topic-init` et `init-producer` pour initialiser les topics et produire des messages de test.

Chaque service peut définir des **ports exposés**, des **volumes**, et des variables d’**environnement** pour la configuration.

---

**2. Organisation et relations des services**

```
[Producers] ---> [Kafka Cluster: kafka-1, kafka-2, kafka-3] <--- [Consumers]
                        |
                        v
                     [Telegraf]
                        |
                        v
                    [InfluxDB]
                        |
                        v
                     [Dashboard / Kafdrop]
```

* Les producteurs envoient des messages dans Kafka.
* Les consommateurs lisent les messages.
* Telegraf interroge Kafka via Jolokia pour collecter les métriques.
* InfluxDB stocke ces métriques et sert de source pour les dashboards.

---

**3. Explication du paragraphe `environment` d’un broker Kafka**
Exemple `kafka-1` :

| Variable                              | Fonction                                                     |
| ------------------------------------- | ------------------------------------------------------------ |
| `KAFKA_KRAFT_CLUSTER_ID`              | Identifiant unique du cluster Kafka KRaft.                   |
| `KAFKA_CFG_NODE_ID`                   | Numéro unique de ce broker dans le cluster.                  |
| `KAFKA_CFG_PROCESS_ROLES`             | Rôle du broker (`broker,controller` pour ce cluster).        |
| `KAFKA_CFG_CONTROLLER_QUORUM_VOTERS`  | Liste des brokers pouvant voter pour le contrôleur actif.    |
| `KAFKA_CFG_LISTENERS`                 | Adresses et ports sur lesquels Kafka écoute.                 |
| `KAFKA_CFG_ADVERTISED_LISTENERS`      | Adresse visible par les clients pour se connecter au broker. |
| `KAFKA_CFG_CONTROLLER_LISTENER_NAMES` | Nom du listener dédié pour le contrôleur.                    |
| `KAFKA_CFG_LOG_DIRS`                  | Répertoire où Kafka stocke ses logs.                         |
| `KAFKA_CFG_AUTO_CREATE_TOPICS_ENABLE` | Permet la création automatique de topics.                    |
| `KAFKA_HEAP_OPTS`                     | Limite de mémoire JVM pour Kafka.                            |
| `KAFKA_JMX_PORT`                      | Port pour exposer les métriques JMX.                         |
| `KAFKA_JMX_HOSTNAME`                  | Hostname pour le JMX.                                        |
| `KAFKA_OPTS`                          | Ajoute l’agent Jolokia pour exporter les métriques via HTTP. |

---

## Observabilité

**4. Qu’est-ce que Jolokia2 ?**

* Jolokia2 est un **agent JMX/HTTP**.
* Il permet d’exposer les métriques JMX d’une application Java via des requêtes HTTP JSON.
* Ici, il est utilisé pour exposer les métriques Kafka.

**5. Quel service va-t-il offrir et à qui ?**

* Jolokia expose les métriques Kafka en HTTP.
* Telegraf consomme ces métriques via le plugin `inputs.jolokia2` et les envoie vers InfluxDB.

**6. Fonction du service Telegraf et éléments principaux de sa config**

* Telegraf collecte automatiquement des métriques des brokers Kafka.
* Configuration principale dans `telegraf.conf` :

  * Plugin `inputs.jolokia2` pointant sur chaque broker Kafka.
  * Plugin `outputs.influxdb_v2` pour envoyer les données dans InfluxDB.
  * Fréquence de collecte définie (`interval = "5s"` ou similaire).

---

## Métrologie

**7. Topic utilisé dans cette application**

* `weather` (initialisé par `topic-init` et utilisé par `init-producer`).

**8. Bucket utilisé dans cette application**

* `kafka_metrics` dans InfluxDB.

**9. Grandes familles de métriques observées**

* **Broker/replication** : partitions sous-répliquées, contrôleur actif.
* **Network** : nombre de requêtes entrantes/sortantes, latence réseau.
* **Topic metrics** : messages produits/consommés, taille des topics.
* **Consumer lag** : retard des consommateurs sur les partitions.

**10. Requête Flux pour voir les variations (exemple `kafka_active_controller`)**

```flux
from(bucket: "kafka_metrics")
  |> range(start: -30m)
  |> filter(fn: (r) => r._measurement == "kafka_active_controller")
  |> filter(fn: (r) => r._field == "Value")
  |> aggregateWindow(every: 1m, fn: mean, createEmpty: false)
```

* Fonction d’agrégation `mean` dans une fenêtre de 1 minute pour lisser les valeurs et montrer les changements.




**11. Requête Flux pour observer la perte puis le retour d’un broker (`UnderReplicatedPartitions`)**

```flux
from(bucket: "kafka_metrics")
  |> range(start: -30m)
  |> filter(fn: (r) => r._measurement == "kafka_under_replicated_partitions")
  |> filter(fn: (r) => r._field == "Value")
  |> aggregateWindow(every: 1m, fn: max, createEmpty: false)
```

* Lance `suspend_random_broker_loop.sh`.
* Pendant 60 s, un broker est suspendu → la valeur `UnderReplicatedPartitions` augmente (partitions non répliquées).
* Après 60 s, le broker est relancé → les partitions se répliquent et la métrique redescend à zéro.
* Ce graphique montre clairement l’impact d’une panne temporaire.

---

## “To infinity and beyond!”

**12. Autres métriques et cas d’usage**

* **Metric : kafka_network_request_total** → identifier les pics de requêtes par broker pour optimiser la répartition de charge.
* **Metric : kafka_topic_messages_in_total** → visualiser la production de messages sur chaque topic pour anticiper les besoins de stockage.

Exemple Flux pour `messages_in_total` :

```flux
from(bucket: "kafka_metrics")
  |> range(start: -30m)
  |> filter(fn: (r) => r._measurement == "kafka_topic_messages_in_total")
  |> filter(fn: (r) => r._field == "Value")
  |> aggregateWindow(every: 1m, fn: sum, createEmpty: false)
```

* Ce graphique permet de suivre l’activité de production sur les topics, détecter des anomalies ou des pics soudains.

---

✅ Avec ces requêtes, tu peux maintenant générer tes **dashboards et captures d’écran** pour le rapport.

---

Si tu veux, je peux te **faire un exemple complet de capture d’écran à intégrer au rapport Markdown**, avec un petit texte explicatif pour chaque graphique, comme si tu avais fait le TP.

Veux‑tu que je fasse ça ?
