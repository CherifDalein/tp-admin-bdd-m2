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






