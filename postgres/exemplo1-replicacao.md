# Exemplo 1 - Replicação

## Limpar docker

```sh
docker-compose down -v
```

## Criando banco de dados

```sh
docker-compose up -d

docker-compose ps

psql -h localhost -p 5431 -U postgres -d demo_db
psql -h localhost -p 5433 -U postgres -d demo_db
psql -h localhost -p 5434 -U postgres -d demo_db
```

## Configurando o principal

```sh
psql -h localhost -p 5431 -U postgres -d demo_db
```

```sql
CREATE USER replicator_user REPLICATION LOGIN PASSWORD 'senha123';
\q
```

```sh
# Este comando executa um echo dentro do contêiner node1 para adicionar a linha necessária.
docker-compose exec node1 bash -c "echo 'host replication replicator_user 0.0.0.0/0 md5' >> /var/lib/postgresql/data/pg_hba.conf"

docker-compose restart node1
```

## Configurando o node2 como replica

```sh
docker-compose stop node2

docker-compose run --rm node2 \
  bash -c "rm -rf /var/lib/postgresql/data/* && PGPASSWORD='senha123' pg_basebackup -h node1 -p 5432 -U replicator_user -D /var/lib/postgresql/data -P -v -R"
```

```sh
docker-compose up -d node2
```

## Teste

```sql
--- Principal
psql -h localhost -p 5431 -U postgres -d demo_db
CREATE TABLE teste_replicacao (id INT, msg TEXT);
INSERT INTO teste_replicacao VALUES (1, 'Olaaaaa!');

```

```sql
-- Shad 1
psql -h localhost -p 5433 -U postgres -d demo_db
SELECT * FROM teste_replicacao;
-- (1, 'Olaaaa!')  <-- SUCESSO!

INSERT INTO teste_replicacao VALUES (2, 'Olaaaaa');
-- ERRO: não é possível executar INSERT em um banco de dados somente leitura
```
