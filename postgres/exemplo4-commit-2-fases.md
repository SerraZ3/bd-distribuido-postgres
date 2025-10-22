# Exemplo 4 - Commit de 2 Fases (2PC)

(Este tutorial não precisou de ajustes, pois não usa FDW e, portanto, não tem os problemas de autenticação entre nós. Ele está correto como você o postou.)

Vamos simular uma transação atômica onde `node2` e `node3` são os Participantes, e você (seu terminal `psql`) é o Coordenador.

## 1\. Limpar e Iniciar Ambiente

O `docker-compose.yml` já configura `max_prepared_transactions=10` em todos os nós, então estamos prontos.

```sh
docker-compose down -v
docker-compose up -d
```

## 2\. Preparar os Participantes (node2 e node3)

Vamos criar as contas bancárias em seus respectivos servidores.

```sql
-- Conecte-se ao Participante A (node2)
psql -h localhost -p 5433 -U postgres -d demo_db

CREATE TABLE contas (id TEXT PRIMARY KEY, saldo NUMERIC);
INSERT INTO contas VALUES ('conta_A', 1000);
\q
```

```sql
-- Conecte-se ao Participante B (node3)
psql -h localhost -p 5434 -U postgres -d demo_db

CREATE TABLE contas (id TEXT PRIMARY KEY, saldo NUMERIC);
INSERT INTO contas VALUES ('conta_B', 2000);
\q
```

## 3\. Executar a Transação (Fase 1: Votação)

Abra dois terminais. Você, o Coordenador, enviará os comandos.

```sql
-- No Terminal 1 (conectado ao node2)
psql -h localhost -p 5433 -U postgres -d demo_db

BEGIN;
UPDATE contas SET saldo = saldo - 100 WHERE id = 'conta_A';

-- Voto "Ready" do node2
PREPARE TRANSACTION 'tx_transfer_123';
```

```sql
-- No Terminal 2 (conectado ao node3)
psql -h localhost -p 5434 -U postgres -d demo_db

BEGIN;
UPDATE contas SET saldo = saldo + 100 WHERE id = 'conta_B';

-- Voto "Ready" do node3
PREPARE TRANSACTION 'tx_transfer_123';
```

**Neste ponto, a transação está no "limbo"**. As contas estão bloqueadas, aguardando sua decisão final. Se você (o Coordenador) falhasse agora, o banco ficaria bloqueado.

## 4\. Executar a Transação (Fase 2: Decisão)

Como ambos votaram "Ready" (sem erros), você decide confirmar (`COMMIT`).

```sql
-- No Terminal 1 (node2)
-- Decisão final de Commit
COMMIT PREPARED 'tx_transfer_123';
```

```sql
-- No Terminal 2 (node3)
-- Decisão final de Commit
COMMIT PREPARED 'tx_transfer_123';
```

_(Se um dos `PREPARE` tivesse falhado, você teria que executar `ROLLBACK PREPARED 'tx_transfer_123';` em ambos os terminais)._

## 5\. Teste (Verificação Final)

Vamos checar se a transação foi atômica.

```sql
-- No Terminal 1 (node2)
psql -h localhost -p 5433 -U postgres -d demo_db

SELECT * FROM contas;
-- id      | saldo
--+---------+-------
-- conta_A |   900   <-- SUCESSO!
```

```sql
-- No Terminal 2 (node3)
psql -h localhost -p 5434 -U postgres -d demo_db

SELECT * FROM contas;
-- id      | saldo
--+---------+-------
-- conta_B |  2100   <-- SUCESSO!
```
