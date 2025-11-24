export BROKER="localhost:9092"
export KSQLDB_URL="http://localhost:8088"


verification
curl -s "$KSQLDB_URL/info" | jq .
curl -s "$KSQLDB_URL/healthcheck"


1) 

docker exec -i kafka-1 \
  kafka-topics --bootstrap-server "$BROKER" \
  --create --topic temperatures --partitions 4 --replication-factor 1 --if-not-exists


la commande correcte est:

docker exec -i kafka-1 \
  kafka-topics --bootstrap-server "$BROKER" --describe --topic temperatures

taper la commande pour executer le script:

bash -lc 'python3 - <<'"'"'PY'"'"' | docker exec -i kafka-1 \
  kafka-console-producer --bootstrap-server '"$BROKER"' \
  --topic temperatures --property parse.key=true --property key.separator=:
import json,random,time,sys
villes=["Clermont-Ferrand","Lyon","Paris","Bordeaux","Nantes"]
for _ in range(200):
    v=random.choice(villes)
    rec={"ville":v,"t":round(random.uniform(5,35),1),"ts":int(time.time()*1000)}
    print(f"{v}:{json.dumps(rec)}"); sys.stdout.flush(); time.sleep(0.2)
PY'


puis aller sur ce lien http://localhost:9021/ pour verifier

Et tu vérifies :

le topic temperatures

la quantité de messages dans chaque partition

Tu devrais voir une répartition très inégale.
Pourquoi ? Parce que la clé détermine la partition via Murmur2.

Certaines villes tombent toujours sur la même partition.

ou bien avec cette commande

docker exec -i kafka-1 \
  kafka-topics --bootstrap-server "$BROKER" --describe --topic temperatures


  APres le script est legerement modifié

  # Option - kafka-console-producer (clé via parse.key)
bash -lc 'python3 - <<'"'"'PY'"'"' | docker exec -i kafka-1 \
  kafka-console-producer --bootstrap-server '"$BROKER"' \
  --topic temperatures --property parse.key=true --property key.separator=:
import json,random,time,sys
villes=["Clermont-Ferrand","Lyon","Paris","Bordeaux","Montpellier"]
for _ in range(200):
    v=random.choice(villes)
    rec={"ville":v,"t":round(random.uniform(5,35),1),"ts":int(time.time()*1000)}
    print(f"{v}:{json.dumps(rec)}"); sys.stdout.flush(); time.sleep(0.2)
PY'


Question : qu’observez-vous sur la répartition des messages ?

 Tu observes que les messages sont répartis différemment entre les partitions.

Plus précisément :

Les villes communes (Lyon, Paris, etc.) restent dans les mêmes partitions qu’avant, car leur hachage ne change pas.

Les messages de la nouvelle ville Montpellier tombent dans une autre partition, déterminée par sa valeur de hachage.

La répartition totale change, car Nantes et Montpellier n’ont pas le même hash → donc pas la même partition.

 La clé modifiée change la distribution globale.


Lien avec la fonction de hachage Murmur2

Kafka utilise :

👉 Murmur2(key) % nombre_de_partitions
pour choisir la partition.

Donc :

chaque clé tombe toujours dans la même partition, tant que le nombre de partitions ne change pas.

si tu remplaces une clé → tu changes son hash → donc sa partition.

c’est exactement pour ça que Nantes ≠ Montpellier → distribution différente.


Remarque: La répartition des messages dépend entièrement de la clé. Kafka utilise la fonction de hachage Murmur2 pour calculer la partition : partition = murmur2(key) % numPartitions.

Lorsque nous remplaçons "Nantes" par "Montpellier", les messages associés à cette clé sont envoyés dans une partition différente, car leur valeur de hachage est différente.

Les autres villes conservent la même partition qu'avant car leur clé n’a pas changé.

2)

Pour se connecter à ksqlDB, j’ai utilisé l’interface Confluent Control Center disponible sur http://localhost:9021.
Dans le menu KSQLDB Cluster, j’ai ouvert le moteur ksqlDB et exécuté la commande SHOW TOPICS;.
Cela m’a permis de visualiser les topics temperatures (4 partitions) et commandes.
L’outil permet de vérifier facilement la présence des topics Kafka et leur configuration sans passer par le CLI.

voici ce que j'ai eu apres la commandes show topics;
{
  "@type": "kafka_topics",
  "statementText": "SHOW TOPICS;",
  "topics": [
    {
      "name": "commandes",
      "replicaInfo": [
        3
      ]
    },
    {
      "name": "temperatures",
      "replicaInfo": [
        1,
        1,
        1,
        1
      ]
    }
  ],
  "warnings": [

  ]
}





3) j'ai tapé la commande juste apres avoir modifié un peu le code du producer 
while True:
    ville = random.choice(villes)
    message = {"ville": ville, "t": round(random.uniform(5,35),1), "ts": int(time.time()*1000)}
    producer.produce("temperatures", key=ville, value=json.dumps(message))
    producer.flush()
    time.sleep(0.2)  # vitesse adaptée



CREATE STREAM S_TEMPS_RAW1 (
  ville STRING,
  t DOUBLE,
  ts BIGINT
) WITH (
  KAFKA_TOPIC='temperatures',
  VALUE_FORMAT='JSON',
  TIMESTAMP='ts'
);

"pir verofoer la creation j'ai tapé 
SHOW STREAMS;
et j'ai vu le s_temps_raw1

et pour visualiser j'ai tapé

SELECT * FROM S_TEMPS_RAW1 EMIT CHANGES;

pour voir les enregistrements de la ville paris, j'ai tapé


## photo

SELECT * FROM S_TEMPS_RAW1 WHERE ville='Paris' EMIT CHANGES;

pour tout reafficher, on peut utiliser la commande

SELECT * FROM S_TEMPS_RAW
EMIT CHANGES
LIMIT 1000;


j'ai créé un stream partitionné par ville (S_TEMPS_BY_VILLE)

CREATE STREAM S_TEMPS_BY_VILLE
WITH (KAFKA_TOPIC='temperatures_by_ville', PARTITIONS=4)
AS
SELECT ville, t, ts
FROM S_TEMPS_RAW1
PARTITION BY ville
EMIT CHANGES;

pour verifier j'ai tapé ça et ça a bien marche

SHOW STREAMS;
DESCRIBE S_TEMPS_BY_VILLE;

ET enfin Dans le Control Center → onglet Persistent Queries.

Pourquoi S_TEMPS_BY_VILLE est persistante :

Elle lit un stream existant (S_TEMPS_RAW1)

Elle écrit en permanence dans un autre topic (temperatures_by_ville)

Elle reste active et continue de traiter les messages en temps réel

🔹 Les requêtes persistantes sont donc “actives” tant que ksqlDB tourne.


4) 
Creation de la table avec fenêtre TUMBLING

CREATE TABLE T_MAX_5M AS
SELECT
  ville,
  WINDOWSTART AS w_start,
  WINDOWEND   AS w_end,
  MAX(t)      AS t_max
FROM S_TEMPS_BY_VILLE
WINDOW TUMBLING (SIZE 5 MINUTES, GRACE PERIOD 30 SECONDS)
GROUP BY ville
EMIT CHANGES;

Analyse de cette commande :

WINDOW TUMBLING (SIZE 5 MINUTES) : la table regroupe les messages par blocs de 5 minutes, non chevauchants.

GRACE PERIOD 30 SECONDS : permet d’accepter des messages retardataires jusqu’à 30 secondes après la fin de la fenêtre.

MAX(t) : calcule la température maximale dans chaque fenêtre pour chaque ville.

GROUP BY ville : agrégation par ville.

EMIT CHANGES : la table est mise à jour en temps réel, à mesure que de nouveaux messages arrivent.



Que voit-on dans l’onglet "Persistent Queries" ?

Tu devrais voir une nouvelle requête persistante, par exemple :

Query ID	Type	Source Stream	Sink Table	Status
CSAS_T_MAX_5M_1	PERSISTENT	S_TEMPS_BY_VILLE	T_MAX_5M	RUNNING


Explication :

Cette requête est persistante car elle lit un stream (S_TEMPS_BY_VILLE) en continu et écrit les résultats dans une table (T_MAX_5M).

Même si de nouveaux messages arrivent dans le topic temperatures, la table se met automatiquement à jour avec les nouvelles fenêtres et nouveaux maximas.

La persistance vient du fait que ksqlDB garde l’état de la fenêtre et des agrégations en mémoire (ou dans le changelog topic associé).

visualisation des valeurs max
SELECT * FROM T_MAX_5M EMIT CHANGES;

Explications :

Chaque ligne correspond à une fenêtre de 5 minutes.

T_MAX correspond à la température maximale observée dans cette fenêtre pour la ville.

Dès qu’une nouvelle fenêtre commence, de nouvelles lignes apparaissent.

En triant par VILLE, tu peux suivre facilement l’évolution des maximas par ville.


5)

Création de la table T_LAST

CREATE TABLE T_LAST AS
SELECT ville,
       LATEST_BY_OFFSET(t) AS t_last,
       LATEST_BY_OFFSET(ts) AS ts_last
FROM S_TEMPS_BY_VILLE
GROUP BY ville
EMIT CHANGES;

Explication simple :

LATEST_BY_OFFSET() récupère la dernière valeur arrivée dans le topic pour une clé donnée (ici, la ville).

GROUP BY ville garantit que chaque ville a 1 entrée unique dans la table.

La table est mise à jour à chaque nouveau message.

La table T_LAST représente donc en temps réel la dernière température reçue pour chaque ville.

✔ Méthode 1 : Visualiser la table en streaming

Tu lances dans ksqlDB :
SELECT * FROM T_LAST EMIT CHANGES;

Méthode 2 : Consulter la table comme une table SQL

(Utilise sans EMIT CHANGES si tu veux juste un snapshot)
SELECT * FROM T_LAST;
Tu obtiendras une seule ligne par ville, représentant le dernier état connu.



Requête pour obtenir en permanence la dernière valeur de température pour Lyon

Tu veux suivre en temps réel uniquement Lyon.
SELECT t_last, ts_last
FROM T_LAST
WHERE ville = 'Lyon'
EMIT CHANGES;

Cela te donne une sortie continue, mise à jour dès qu'un nouveau message est produit pour Lyon.



# ✅ **7) HOPPING Windows (option)**

Tu vas maintenant créer une table qui fait une **moyenne glissante des températures** avec :

* **Fenêtre de 10 minutes** (SIZE)
* **Avance / saut de 2 minutes** (ADVANCE BY)

👉 Cela signifie que **toutes les 2 minutes**, une nouvelle fenêtre de 10 minutes est calculée.
Les fenêtres **se chevauchent**, contrairement au TUMBLING.

---

# 1️⃣ **Créer la table T_AVG_10M_HOP2**

Dans ksqlDB (Control Center ou curl) :

```sql
CREATE TABLE T_AVG_10M_HOP2 AS
SELECT
  ville,
  WINDOWSTART AS w_start,
  WINDOWEND   AS w_end,
  AVG(t)      AS t_avg
FROM S_TEMPS_BY_VILLE
WINDOW HOPPING (SIZE 10 MINUTES, ADVANCE BY 2 MINUTES)
GROUP BY ville
EMIT CHANGES;
```

Tu devrais recevoir une réponse du type :

```
Table created
```

Et un **Persistent Query** va apparaître.

---

# 2️⃣ **Que se passe-t-il derrière ?**

Le serveur ksqlDB crée une **query persistante** qui :

* lit en continu le stream `S_TEMPS_BY_VILLE`
* calcule des fenêtres qui se chevauchent
* crée différentes entrées dans la table selon les fenêtres actives

---

# 3️⃣ **Visualiser l’évolution de la table**

Exécute :

```sql
SELECT * FROM T_AVG_10M_HOP2 EMIT CHANGES;
```

Tu vas voir des lignes comme :

| VILLE | W_START  | W_END    | T_AVG |
| ----- | -------- | -------- | ----- |
| Paris | 21:00:00 | 21:10:00 | 23.5  |
| Paris | 21:02:00 | 21:12:00 | 24.1  |
| Paris | 21:04:00 | 21:14:00 | 22.8  |
| Lyon  | 21:00:00 | 21:10:00 | 19.4  |

👉 Tu remarques que *pour une même ville*, plusieurs fenêtres actives existent **en même temps**.

---

# 4️⃣ **Explication à mettre dans ton rapport**

Voici une explication simple et propre :

> La fenêtre HOPPING est une fenêtre glissante avec chevauchement.
>
> * La durée totale de la fenêtre est de 10 minutes.
> * Une nouvelle fenêtre commence toutes les 2 minutes, ce qui crée plusieurs fenêtres simultanées.
>   À chaque nouveau message, toutes les fenêtres qui couvrent ce timestamp sont mises à jour.
>   La table T_AVG_10M_HOP2 stocke alors plusieurs lignes par ville, chacune correspondant à une fenêtre différente.

---

# 5️⃣ **Comment montrer l’évolution ?**

Pendant que ton producer envoie des données, tu observes :

```sql
SELECT * FROM T_AVG_10M_HOP2 EMIT CHANGES;
```

Puis trier dans l’interface par ville, w_start, w_end.

Tu verras les moyennes évoluer comme :

```
Paris | 21:00:00 | 21:10:00 | 23.5
Paris | 21:02:00 | 21:12:00 | 24.1
Paris | 21:04:00 | 21:14:00 | 22.8
```

👉 Chaque nouvelle valeur met à jour toutes les fenêtres où elle appartient.

