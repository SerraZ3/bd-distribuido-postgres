# Exemplo 3 - Fragmentação Vertical

Vamos usar `node1` (Coordenador), `node2` (dados "quentes": nome/preço) e `node3` (dados "frios": descrição).

## 1\. Limpar e Iniciar Ambiente

```sh
docker-compose down -v
docker-compose up -d
```

## 2\. Configurar os Shards (node2 e node3)

**-- MUDANÇA AQUI --**
Removemos a modificação do `pg_hba.conf`. Vamos usar a autenticação por senha padrão, que é mais segura e confiável.

Agora, apenas crie as tabelas que armazenarão as "fatias" das colunas:

```sql
-- Conecte-se ao Shard 1 (dados quentes)
psql -h localhost -p 5433 -U postgres -d demo_db

CREATE TABLE produtos_info (
    id INT PRIMARY KEY,
    nome TEXT,
    preco NUMERIC(10, 2)
);
\q
```

```sql
-- Conecte-se ao Shard 2 (dados frios)
psql -h localhost -p 5434 -U postgres -d demo_db

CREATE TABLE produtos_descricao (
    id INT PRIMARY KEY, -- A Chave Primária deve existir em ambos
    descricao_longa TEXT,
    imagem_blob TEXT -- Simulando dados pesados
);
\q
```

## 3\. Configurar o Coordenador (node1)

Vamos ao `node1` para criar os "ponteiros" FDW e a `VIEW` que unificará as colunas.

```sql
-- Conecte-se ao Coordenador (node1)
psql -h localhost -p 5431 -U postgres -d demo_db

-- 1. Ativar a extensão
CREATE EXTENSION postgres_fdw;

-- 2. Definir os servidores remotos (os shards)
CREATE SERVER shard_info_server
    FOREIGN DATA WRAPPER postgres_fdw
    OPTIONS (host 'node2', port '5432', dbname 'demo_db');

CREATE SERVER shard_desc_server
    FOREIGN DATA WRAPPER postgres_fdw
    OPTIONS (host 'node3', port '5432', dbname 'demo_db');

-- 3. Mapear o usuário E A SENHA (MUDANÇA AQUI)
CREATE USER MAPPING FOR postgres
    SERVER shard_info_server
    OPTIONS (user 'postgres', password 'postgres');

CREATE USER MAPPING FOR postgres
    SERVER shard_desc_server
    OPTIONS (user 'postgres', password 'postgres');

-- 4. Criar os "ponteiros" (Tabelas Estrangeiras)
CREATE FOREIGN TABLE f_produtos_info (
    id INT,
    nome TEXT,
    preco NUMERIC(10, 2)
) SERVER shard_info_server OPTIONS (table_name 'produtos_info');

CREATE FOREIGN TABLE f_produtos_descricao (
    id INT,
    descricao_longa TEXT,
    imagem_blob TEXT
) SERVER shard_desc_server OPTIONS (table_name 'produtos_descricao');

CREATE VIEW produtos_completos AS
SELECT
    info.id,
    info.nome,
    info.preco,
    -- MUDANÇA AQUI: 'desc.' virou 'd.'
    d.descricao_longa,
    d.imagem_blob
FROM
    f_produtos_info AS info
JOIN
    -- MUDANÇA AQUI: 'AS desc' virou 'AS d'
    f_produtos_descricao AS d ON info.id = d.id;
\q
```

## 4\. Teste

O `INSERT` na fragmentação vertical é menos transparente (precisamos inserir em cada tabela FDW), mas a leitura (`SELECT`) é unificada.

```sql
-- No Coordenador (node1)
psql -h localhost -p 5431 -U postgres -d demo_db

-- Inserimos os dados em suas respectivas tabelas FDW
-- O Coordenador (node1) agora envia a senha 'postgres' ao se conectar.
INSERT INTO f_produtos_info (id, nome, preco) VALUES (101, 'Notebook', 4500.00);
INSERT INTO f_produtos_descricao (id, descricao_longa, imagem_blob)
    VALUES (101, 'Notebook de alta performance...', '0xDEADBEEF...');

-- A consulta na VIEW executa um JOIN distribuído (agora com sucesso)
SELECT * FROM produtos_completos WHERE id = 101;

-- Resultado (unificado):
-- id  |   nome   |  preco  |     descricao_longa      | imagem_blob
-------+----------+---------+--------------------------+-------------
-- 101 | Notebook | 4500.00 | Notebook de alta...      | 0xDEADBEEF...

-- Consultando apenas os dados "quentes" (mais rápido)
SELECT nome, preco FROM f_produtos_info WHERE id = 101;
--   nome   |  preco
--+----------+---------
-- Notebook | 4500.00
```
