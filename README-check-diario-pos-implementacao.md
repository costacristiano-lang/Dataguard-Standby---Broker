# Oracle Data Guard: Check Diario Pos-Implementacao

![Oracle](https://img.shields.io/badge/Oracle-Data%20Guard-C74634?logo=oracle&logoColor=white)
![Checklist](https://img.shields.io/badge/Rotina-Check%20Diario-1F6FEB)
![Status](https://img.shields.io/badge/Uso-Operacional-0A7F5A)

Guia em `Markdown` para GitHub com um procedimento de acompanhamento diario apos a implantacao de um ambiente `Oracle Data Guard`.

## Objetivo

Este documento serve como roteiro operacional para validar diariamente se o ambiente de `PRIMARY` e `PHYSICAL STANDBY` continua saudavel, sincronizado e pronto para atender uma necessidade de `Disaster Recovery (DR)`.

## Quando Executar

Recomenda-se executar este check:

- no inicio do expediente
- apos mudancas de infraestrutura ou banco
- apos incidentes de rede, storage ou listener
- antes de janelas criticas de negocio

## Sumario

- [Objetivo](#objetivo)
- [Quando Executar](#quando-executar)
- [Checklist Diario](#checklist-diario)
- [1. Validar Papel e Estado dos Bancos](#1-validar-papel-e-estado-dos-bancos)
- [2. Validar Transporte e Apply de Redo](#2-validar-transporte-e-apply-de-redo)
- [3. Validar Lag de Sincronizacao](#3-validar-lag-de-sincronizacao)
- [4. Validar Broker](#4-validar-broker)
- [5. Validar Destinos de Archive](#5-validar-destinos-de-archive)
- [6. Validar Managed Recovery Process](#6-validar-managed-recovery-process)
- [7. Validar Standby Redo Logs](#7-validar-standby-redo-logs)
- [8. Validar Erros Recentes no Alert Log](#8-validar-erros-recentes-no-alert-log)
- [9. Validar Espaco para Archives e FRA](#9-validar-espaco-para-archives-e-fra)
- [10. Validar Flashback e Prontidao para DR](#10-validar-flashback-e-prontidao-para-dr)
- [Modelo de Registro Diario](#modelo-de-registro-diario)
- [Sinais de Alerta](#sinais-de-alerta)
- [Resumo](#resumo)

## Checklist Diario

Use esta lista como roteiro rapido:

- [ ] confirmar qual banco esta como `PRIMARY`
- [ ] confirmar qual banco esta como `PHYSICAL STANDBY`
- [ ] validar `open_mode` e `database_role`
- [ ] validar transporte de redo sem erro
- [ ] validar apply de redo sem atraso anormal
- [ ] validar `transport lag` e `apply lag`
- [ ] validar `Broker` sem `WARNING` ou `ERROR`
- [ ] validar `MRP` em execucao no standby
- [ ] validar destino de archives sem falhas
- [ ] validar uso de FRA e espaco em disco
- [ ] revisar erros recentes no `alert log`
- [ ] registrar evidencias do dia

## 1. Validar Papel e Estado dos Bancos

Execute em cada banco:

```sql
SELECT name,
       db_unique_name,
       open_mode,
       database_role,
       protection_mode,
       protection_level,
       switchover_status
  FROM v$database;
```

Resultado esperado:

- banco principal com `database_role = PRIMARY`
- banco standby com `database_role = PHYSICAL STANDBY`
- ambiente sem status inconsistentes de troca de papel

## 2. Validar Transporte e Apply de Redo

No standby:

```sql
SELECT process,
       status,
       thread#,
       sequence#,
       block#
  FROM v$managed_standby
 ORDER BY process;
```

Resultado esperado:

- processos de `RFS` recebendo redo
- processo de recovery ativo no standby
- `sequence#` evoluindo de acordo com a carga

## 3. Validar Lag de Sincronizacao

No standby:

```sql
SELECT name,
       value,
       unit
  FROM v$dataguard_stats
 WHERE name IN ('transport lag', 'apply lag', 'apply finish time');
```

Resultado esperado:

- `transport lag` baixo ou zerado
- `apply lag` dentro do limite operacional aceito
- `apply finish time` coerente com o volume de redo

## 4. Validar Broker

No `DGMGRL`:

```text
DGMGRL> SHOW CONFIGURATION;
DGMGRL> SHOW DATABASE VERBOSE <PRIMARY_DB_UNIQUE_NAME>;
DGMGRL> SHOW DATABASE VERBOSE <STANDBY_DB_UNIQUE_NAME>;
```

Resultado esperado:

- configuracao em `SUCCESS`
- ausencia de `WARNING` ou `ERROR`
- propriedades principais coerentes com o ambiente

## 5. Validar Destinos de Archive

No primario:

```sql
SELECT dest_id,
       destination,
       target,
       status,
       error
  FROM v$archive_dest_status
 ORDER BY dest_id;
```

Resultado esperado:

- destino remoto do standby com `status` saudavel
- coluna `error` vazia ou sem mensagens relevantes

## 6. Validar Managed Recovery Process

No standby:

```sql
SELECT process,
       client_process,
       status,
       sequence#
  FROM v$managed_standby
 WHERE process IN ('MRP0', 'MRP');
```

Resultado esperado:

- `MRP0` ou processo equivalente ativo
- apply em andamento ou aguardando redo de forma normal

## 7. Validar Standby Redo Logs

No standby:

```sql
SELECT group#,
       thread#,
       sequence#,
       archived,
       status
  FROM v$standby_log
 ORDER BY group#;
```

Resultado esperado:

- grupos disponiveis
- sem sinais de configuracao incompleta
- rotacao coerente com a atividade do ambiente

## 8. Validar Erros Recentes no Alert Log

Revise o `alert log` do primario e do standby procurando por:

- erros `ORA-`
- falhas de transporte
- perda de conectividade
- problemas de listener
- erros de armazenamento ou falta de espaco

Exemplo de verificacao no sistema operacional:

```bash
tail -100 alert_<SID>.log
```

## 9. Validar Espaco para Archives e FRA

Cheque a utilizacao da `Fast Recovery Area`:

```sql
SELECT name,
       space_limit,
       space_used,
       space_reclaimable,
       number_of_files
  FROM v$recovery_file_dest;
```

Resultado esperado:

- espaco livre suficiente para a geracao normal de archives
- ausencia de pressao de espaco que comprometa transporte ou retencao

## 10. Validar Flashback e Prontidao para DR

Verifique se o `Flashback Database` continua habilitado:

```sql
SELECT flashback_on,
       open_mode,
       database_role
  FROM v$database;
```

Resultado esperado:

- `flashback_on` habilitado conforme a politica adotada
- standby apto a participar de `switchover`, `failover` ou `reinstate`

## Modelo de Registro Diario

Voce pode registrar o resultado em formato simples:

```text
Data: ____/____/____
Responsavel: __________________
PRIMARY atual: __________________
STANDBY atual: __________________
Broker: SUCCESS / WARNING / ERROR
Transport lag: __________________
Apply lag: __________________
MRP ativo: SIM / NAO
Erros no alert log: SIM / NAO
Uso de FRA: __________________
Observacoes: __________________________________________
```

## Sinais de Alerta

Investigue imediatamente se encontrar:

- `apply lag` crescendo continuamente
- `transport lag` acima do padrao do ambiente
- `MRP` parado
- `Broker` com `WARNING` ou `ERROR`
- erros na coluna `error` de `v$archive_dest_status`
- saturacao de `FRA`
- mensagens `ORA-` recorrentes no `alert log`

## Resumo

Um check diario simples ajuda a evitar que o standby fique defasado ou inutilizavel quando for realmente necessario. O ideal e transformar esta rotina em evidencia operacional formal, com registro de status, desvios e acoes corretivas.
