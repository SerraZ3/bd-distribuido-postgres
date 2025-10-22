# Exemplo 5 - Commit de 2 Fases (2PC) Automático com FDW

Vamos usar `node1` (Coordenador) e `node2`/`node3` (Participantes). O `node1` irá coordenar automaticamente a transação atômica.

## 1\. Limpar e Iniciar Ambiente

O `docker-compose.yml` já habilita `max_prepared_transactions` em todos os nós, o que é um requisito para o FDW fazer o 2PC.

```sh
docker-compose down -v
docker-compose up -d
```

## 2\. Preparar os Participantes (node2 e node3)

Vamos criar as contas bancárias em seus respectivos servidores. (Note que não precisamos mexer no `pg_hba.conf`, pois usaremos a autenticação por senha).

```sql
-- Conecte-se ao Participante A (node2)
psql -h localhost -p 5433 -U postgres -d demo_db

CREATE TABLE contas (
    id TEXT PRIMARY KEY,
    saldo NUMERIC
    -- Adicionamos uma restrição para o Teste 2
    CHECK (saldo >= 0)
);
INSERT INTO contas VALUES ('conta_A', 1000);
\q
```

```sql
-- Conecte-se ao Participante B (node3)
psql -h localhost -p 5434 -U postgres -d demo_db

CREATE TABLE contas (
    id TEXT PRIMARY KEY,
    saldo NUMERIC
);
INSERT INTO contas VALUES ('conta_B', 2000);
\q
```

## 3\. Configurar o Coordenador (node1)

Agora, vamos conectar o `node1` aos outros dois nós e mapear suas tabelas.

```sql
-- Conecte-se ao Coordenador (node1)
psql -h localhost -p 5431 -U postgres -d demo_db

-- 1. Ativar a extensão
CREATE EXTENSION postgres_fdw;

-- 2. Definir os servidores remotos (os participantes)
CREATE SERVER node2_server
    FOREIGN DATA WRAPPER postgres_fdw
    OPTIONS (host 'node2', port '5432', dbname 'demo_db');

CREATE SERVER node3_server
    FOREIGN DATA WRAPPER postgres_fdw
    OPTIONS (host 'node3', port '5432', dbname 'demo_db');

-- 3. Mapear o usuário E A SENHA
CREATE USER MAPPING FOR postgres
    SERVER node2_server
    OPTIONS (user 'postgres', password 'postgres');

CREATE USER MAPPING FOR postgres
    SERVER node3_server
    OPTIONS (user 'postgres', password 'postgres');

-- 4. Criar os "ponteiros" (Tabelas Estrangeiras)
CREATE FOREIGN TABLE f_contas_node2 (
    id TEXT,
    saldo NUMERIC
) SERVER node2_server OPTIONS (table_name 'contas');

CREATE FOREIGN TABLE f_contas_node3 (
    id TEXT,
    saldo NUMERIC
) SERVER node3_server OPTIONS (table_name 'contas');

\q
```

## 4\. Teste 1: A Transação Atômica (Caminho Feliz)

Agora, vamos executar a transferência. Você só precisa de **um terminal** (conectado ao `node1`).

```sql
-- Conecte-se APENAS ao Coordenador (node1)
psql -h localhost -p 5431 -U postgres -d demo_db

-- Verifique os saldos iniciais
SELECT * FROM f_contas_node2; -- Saldo é 1000
SELECT * FROM f_contas_node3; -- Saldo é 2000

-- Execute a transação distribuída
BEGIN;

-- Modifica dados no node2
UPDATE f_contas_node2 SET saldo = saldo - 100 WHERE id = 'conta_A';

-- Modifica dados no node3
UPDATE f_contas_node3 SET saldo = saldo + 100 WHERE id = 'conta_B';

-- Ao executar este COMMIT, o node1 executa o 2PC automaticamente
COMMIT;

-- Verifique os saldos finais
SELECT * FROM f_contas_node2; -- Saldo agora é 900
SELECT * FROM f_contas_node3; -- Saldo agora é 2100
```

**Sucesso\!** O 2PC foi 100% automático.

## 5\. Teste 2: A Reversão Atômica (Caminho Triste)

O que acontece se uma parte falhar? A `conta_A` no `node2` tem uma restrição `CHECK (saldo >= 0)`. Vamos tentar violá-la, mas _depois_ de uma operação bem-sucedida no `node3`.

```sql
-- Ainda no terminal do node1
psql -h localhost -p 5431 -U postgres -d demo_db

-- Saldos atuais: 900 (A) e 2100 (B)

BEGIN;

-- 1. Operação VÁLIDA no node3
UPDATE f_contas_node3 SET saldo = saldo + 500 WHERE id = 'conta_B';

-- 2. Operação INVÁLIDA no node2 (tentando deixar saldo -100)
UPDATE f_contas_node2 SET saldo = saldo - 1000 WHERE id = 'conta_A';

-- 3. Tente confirmar
COMMIT;

-- O QUE ACONTECEU:
-- 1. node1 enviou PREPARE para node3. node3 votou "Sim".
-- 2. node1 enviou PREPARE para node2. node2 tentou executar,
--    mas violou a restrição CHECK. node2 votou "Não" (erro).
-- 3. node1 (Coordenador) recebeu o "Não".
-- 4. node1 enviou ROLLBACK PREPARED para o node3 (desfazendo a op 1).
-- 5. O COMMIT falha totalmente.
```

**Resultado:**

```
ERROR:  erro de restrição check "contas_saldo_check" em servidor remoto "node2_server"
DETALHE:  Failing row contains (conta_A, -100).
```

## 6\. Verificação Final (Atomicidade)

A transação foi **atômica**. Como a operação 2 falhou, a operação 1 (que teve sucesso temporário) foi revertida.

```sql
-- Ainda no terminal do node1

SELECT * FROM f_contas_node2 WHERE id = 'conta_A';
-- id      | saldo
--+---------+-------
-- conta_A |   900   <-- (Não mudou, continua 900)

SELECT * FROM f_contas_node3 WHERE id = 'conta_B';
-- id      | saldo
--+---------+-------
-- conta_B |  2100   <-- (Não mudou, continua 2100)
```

O sistema está **consistente**.
