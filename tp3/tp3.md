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




