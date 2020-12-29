# pgsql-cdc-debezium
PostgreSQL Change Data Capture With Debezium

Esse tutorial tem como objetivo replicar banco de dados heterogêneos.

Debezium é construído sobre o projeto Apache Kafka e o utliza para transportar as mudanças de um sistemas(banco de dados) para outro. Um aspecto é que o núcleo do Debezium utiliza o (CDC (Change Data Capture)) para capturar dados e envia-los(notificar) ao Kafka.

O banco de dados de origem permanece intocado, sendo uma das grandes vantagens, pois 'triggers' degradam o desempenho.
Historicamente, os dados eram mantidos em um enorme armazenamento de dados monolíticos e todos os serviços lidos ou escritos para este datastore. 
Sistemas mais novos estão migrando para microsserviços onde o processamento de dados é dividido em tarefas menores. O desafio nesse ponto é garantir que cada microsserviço tenha uma cópia atualizada dos dados que necessita para seu funcionamento.

Benefícios de utilizar o CDC:
* Usa os registros de gravação antecipada para acompanhar as alterações
* Usa o datastore para gerenciar as alterações (não perca dados se estiver off-line)
* Notifica e envia mudanças imediatamente

Essa vantagens torna o sistem muito mais flexível. Se for adicionar um novo microserviços, basta assinar o tópico no Kakfa no qual é pertinente ao serviço.

# Executar Zookeeper
docker run -it --rm --name zookeeper -p 2181:2181 -p 2888:2888 -p 3888:3888 debezium/zookeeper:0.10

# Executar Kafka  
docker run -it --rm --name kafka -p 9092:9092 --link zookeeper:zookeeper debezium/kafka:0.10

# Criar arquivo na pasta *pgconf/postgresql.conf*
# here are some sane defaults given we will be unable to use the container
# variables
# general connection
listen_addresses = '*'
port = 5432
max_connections = 20
# memory
shared_buffers = 128MB
temp_buffers = 8MB
work_mem = 4MB
# WAL / replication
wal_level = logical
max_wal_senders = 3
# these shared libraries are available in the Crunchy PostgreSQL container
shared_preload_libraries = 'pgaudit.so,pg_stat_statements.so'
EOF

# Criar arquivo pg-env.list:
PG_MODE=primary
PG_PRIMARY_PORT=5432
PG_PRIMARY_USER=postgres
PG_DATABASE=testdb
PG_PRIMARY_PASSWORD=debezium
PG_PASSWORD=not
PG_ROOT_PASSWORD=matter
PG_USER=debeziumEOF

# Iniciar uma instância postgres:
docker run -it --rm --name=pgsql --env-file=pg-env.list --volume=%cd%/pgconf:/pgconf -d crunchydata/crunchy-postgres:centos7-11.4-2.4.1

## Bash:
docker inspect pgsql | grep IPAddress

## Powershell:
docker inspect pgsql | Select-String IPAddress
"SecondaryIPAddresses": null,
"IPAddress": "172.17.0.2",
"IPAddress": "172.17.0.2"

# Executar segunda instância do postgres:
docker run -it --rm --link pgsql:pg11  crunchydata/crunchy-postgres:centos7-11.4-2.4.1 psql -h pg11 -U postgres
Senha: **debezium**

## Executar DDL:
CREATE TABLE customers (id int GENERATED ALWAYS AS IDENTITY PRIMARY KEY, name text);
ALTER TABLE customers REPLICA IDENTITY USING INDEX customers_pkey;
INSERT INTO customers (name) VALUES ('joe'), ('bob'), ('sue');
CREATE DATABASE customers;
**Não é necessário criar nenhuma tabela aqui.**

# Conector:
https://github.com/debezium/debezium-examples/tree/master/end-to-end-demo/debezium-jdbc

## Docker Build
Ir a pasta de imagens debezium-jdbc
docker build .

docker images

Procurar o ID da imagem buildada

docker tag 15faacb550e3 jdbc-sink

docker run -it --rm --name connect -p 8083:8083 -e GROUP_ID=1 -e CONFIG_STORAGE_TOPIC=my_connect_configs -e OFFSET_STORAGE_TOPIC=my_connect_offsets -e STATUS_STORAGE_TOPIC=my_connect_statuses --link zookeeper:zookeeper --link kafka:kafka --link pgsql:pgsql 15faacb550e3

# Teste de conectores disponíveis:
curl -H "Accept:application/json" *localhost:8083/connectors/*

No Postman, realize um GET para **localhost:8083/connectors/** com o seguinte corpo no formato **JSON**:

Resp: []

# Criar conector inventory-connector:
No Postman, realize um POST para **localhost:8083** com o seguinte corpo no formato **JSON**:
{
  "name": "inventory-connector",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "database.hostname": "pgsql",
    "plugin.name": "pgoutput",
    "database.port": "5432",
    "database.user": "postgres",
    "database.password": "debezium",
    "database.dbname" : "postgres",
    "database.server.name": "fullfillment",
    "table.whitelist": "public.customers"
  }
  
# Criar conector jdbc-sink:  
No Postman, realize um POST para **localhost:8083** com o seguinte corpo no formato **JSON**:
{
  "name": "jdbc-sink",
  "config": {
    "connector.class": "io.confluent.connect.jdbc.JdbcSinkConnector",
    "tasks.max": "1",
    "topics": "fullfillment.public.customers",
    "dialect.name": "PostgreSqlDatabaseDialect",
    "table.name.format": "customers",
    "connection.url": "jdbc:postgresql://pgsql:5432/customers?user=postgres&password=debezium",
    "transforms": "unwrap",
    "transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState",
    "transforms.unwrap.drop.tombstones": "false",
    "auto.create": "true",
    "insert.mode": "upsert",
    "pk.fields": "id",
    "pk.mode": "record_key",
    "delete.enabled": "true"
  }
}
}

# Conectar plsql
docker run -it --rm --link pgsql:pg11 debezium/postgres:11 psql -h pg11 -U postgres
table customers;

\c customers

table customers;

# Alterar informação passos acima

update customers set name='paul' where id=1;

\c customers

table customers;

# Excluindo
delete from customers where id=1;

\c customers

table customers;

# Conclusão
Um ponto de atenção é que, a replicação lógica no PostgreSQL não fornece nenhuma informação sobre *sequences*. Eles precisarão ser tratados por algum outro processo.
Tudo isso é muito fácil de configurar, complexidades de cluster e DR não foram tratadas aqui. 
