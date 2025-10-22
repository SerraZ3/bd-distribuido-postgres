### Exemplo 2 - Fragmentação horizontal

Siga estes passos. O tutorial foi ajustado para refletir a geração correta de IDs.

## 1\. Limpar e Iniciar Ambiente

Comece do zero para garantir que as definições de tabela antigas sumam.

```sh
docker-compose down -v
docker-compose up -d
```

## 2\. Configurar os Shards (node2 e node3)

Permita a conexão e crie as tabelas dos shards, **mas com `id INT`**, não `SERIAL`.

```sh
# Permite que node1 acesse node2 (vamos usar senha desta vez)
docker-compose exec node2 bash -c "echo 'host demo_db postgres node1 scram-sha-256' >> /var/lib/postgresql/data/pg_hba.conf"

# Permite que node1 acesse node3
docker-compose exec node3 bash -c "echo 'host demo_db postgres node1 scram-sha-256' >> /var/lib/postgresql/data/pg_hba.conf"

# Reinicia os shards
docker-compose restart node2 node3
```

Agora, crie as tabelas nos shards:

```sql
-- Conecte-se ao Shard 1 (America)
psql -h localhost -p 5433 -U postgres -d demo_db

-- MUDANÇA IMPORTANTE: id é INT, não SERIAL
CREATE TABLE usuarios_america (
    id INT PRIMARY KEY,
    nome TEXT,
    pais TEXT,
    CHECK (pais IN ('EUA', 'Brasil', 'Mexico'))
);
\q
```

```sql
-- Conecte-se ao Shard 2 (Europa)
psql -h localhost -p 5434 -U postgres -d demo_db

-- MUDANÇA IMPORTANTE: id é INT, não SERIAL
CREATE TABLE usuarios_europa (
    id INT PRIMARY KEY,
    nome TEXT,
    pais TEXT,
    CHECK (pais IN ('França', 'Alemanha', 'Espanha'))
);
\q
```

## 3\. Configurar o Coordenador (node1)

Aqui, vamos criar a `SEQUENCE` global e usá-la como `DEFAULT`.

```sql
-- Conecte-se ao Coordenador (node1)
psql -h localhost -p 5431 -U postgres -d demo_db

-- 1. Ativar a extensão
CREATE EXTENSION postgres_fdw;

-- 2. Definir os servidores remotos (os shards)
CREATE SERVER shard_america_server
    FOREIGN DATA WRAPPER postgres_fdw
    OPTIONS (host 'node2', port '5432', dbname 'demo_db');

CREATE SERVER shard_europa_server
    FOREIGN DATA WRAPPER postgres_fdw
    OPTIONS (host 'node3', port '5432', dbname 'demo_db');

-- 3. Mapear o usuário E A SENHA
CREATE USER MAPPING FOR postgres
    SERVER shard_america_server
    OPTIONS (user 'postgres', password 'postgres');

CREATE USER MAPPING FOR postgres
    SERVER shard_europa_server
    OPTIONS (user 'postgres', password 'postgres');

-- 4. MUDANÇA IMPORTANTE: Criar a SEQUENCE global
CREATE SEQUENCE global_user_id_seq;

-- 5. MUDANÇA IMPORTANTE: Criar a tabela "Mãe" com DEFAULT
CREATE TABLE usuarios (
    id INT DEFAULT nextval('global_user_id_seq'), -- Usa a sequence
    nome TEXT,
    pais TEXT
) PARTITION BY LIST (pais);

-- 6. Criar as partições (agora o mapeamento INT -> INT está correto)
CREATE FOREIGN TABLE usuarios_america_part
    PARTITION OF usuarios FOR VALUES IN ('EUA', 'Brasil', 'Mexico')
    SERVER shard_america_server
    OPTIONS (table_name 'usuarios_america');

CREATE FOREIGN TABLE usuarios_europa_part
    PARTITION OF usuarios FOR VALUES IN ('França', 'Alemanha', 'Espanha')
    SERVER shard_europa_server
    OPTIONS (table_name 'usuarios_europa');

\q
```

## 4\. Teste (Agora vai funcionar)

```sql
-- No Coordenador (node1)
psql -h localhost -p 5431 -U postgres -d demo_db

-- O INSERT agora omite o ID
INSERT INTO usuarios (nome, pais) VALUES ('João', 'Brasil');
-- O QUE ACONTECE:
-- 1. node1 recebe o INSERT.
-- 2. O DEFAULT dispara -> id = nextval('global_user_id_seq') -> id = 1
-- 3. node1 envia o comando: INSERT... (id, nome, pais) VALUES (1, 'João', 'Brasil')
-- 4. node2 recebe (1, 'João', 'Brasil'). '1' é um INT válido. SUCESSO.

INSERT INTO usuarios (nome, pais) VALUES ('Sophie', 'França');
-- 1. O DEFAULT dispara -> id = 2
-- 2. node1 envia o comando: INSERT... (id, nome, pais) VALUES (2, 'Sophie', 'França')
-- 3. node3 recebe (2, 'Sophie', 'França'). SUCESSO.

INSERT INTO usuarios (nome, pais) VALUES ('John', 'EUA');
-- 1. O DEFAULT dispara -> id = 3
-- 2. node1 envia o comando: INSERT... (id, nome, pais) VALUES (3, 'John', 'EUA')
-- 3. node2 recebe (3, 'John', 'EUA'). SUCESSO.

-- A consulta agora mostra IDs globalmente únicos!
SELECT * FROM usuarios ORDER BY pais;
-- id |  nome  |  pais
------+--------+--------
--  1 | João   | Brasil
--  2 | Sophie | França
--  3 | John   | EUA

\q
```

```sql
-- Verifique o Shard 1 (America) diretamente
psql -h localhost -p 5433 -U postgres -d demo_db
SELECT * FROM usuarios_america;
-- id | nome | pais
------+------+--------
--  1 | João | Brasil
--  2 | John | EUA
-- (Sophie não está aqui!)
```

```sql
-- Verifique o Shard 2 (Europa) diretamente
psql -h localhost -p 5434 -U postgres -d demo_db
SELECT * FROM usuarios_europa;
-- id |  nome  |  pais
------+--------+--------
--  1 | Sophie | França
-- (João e John não estão aqui!)
```
