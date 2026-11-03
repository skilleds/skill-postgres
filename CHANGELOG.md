# Changelog — schematize-postgres

Todas as mudanças relevantes deste pacote, no formato [Keep a Changelog](https://keepachangelog.com/pt-BR/1.1.0/),
com versionamento [SemVer](https://semver.org/lang/pt-BR/).

## [0.3.0] — 2026-11-03

### Alterado
- Seção "Quem executa" (SKILL.md e assets/CLAUDE.md, piso 16): SQL só como arquivo versionado no git; a IA propõe, o ops aplica por operação nomeada (`ops migrate`, dry-run), humano confirma o destrutivo; `EXPLAIN` por query nomeada somente-leitura, sem `ANALYZE` em DML em prd; vacuum/repack/restore/failover/upgrade/backfill são operações tipadas do ops. Comandos `pg-migrate`, `pg-explain`, `pg-backup`, `pg-upgrade` e references (vacuum, backup, replicação, upgrade, locks) alinhados.
- Todo passo traz `down` (par `NNN_nome.up.sql`/`.down.sql`); contract destrutivo: `down` = restore de backup testado ou ADR (`-- pg-irreversivel:`).
- Restore para ensaio passa por mascaramento antes de acesso humano/agente.
- `upgrade.md`: sem `rsync` solto; réplica refeita por Patroni/`pg_basebackup`/pgbackrest (operação do ops).
- Bloco herdado dos pisos 22–29 da engineering.

### Adicionado
- Gate `check-pg-migration.sh`: `migration-sem-down`, `down-contract-sem-restore`, `do-execute-dinamico`, `where-trivial` (`WHERE true`/`1=1`), com testes vermelho -> verde.

## [0.2.0] — 2026-10-14
Integração ao catálogo: orçamento de description e ponteiros entre as irmãs novas.

### Alterado
- `description` <=190 (orçamento do catálogo).
- Divisas: `schematize-incident` (banco em incidente), `schematize-payments` (ledger), `schematize-git` (`/git-leak` para credencial de banco).
- Motivo: o catálogo estourou o orçamento de listagem com 45 skills; cada fronteira aponta a dona em vez de duplicar.

## [0.1.0] — 2026-10-14
Primeira versão. A `schematize-database` cobre o **desenho** do schema (inclusive o método
expand-contract), mas ninguém na casa cobria o **motor**: qual DDL toma `ACCESS EXCLUSIVE`, por que a
migration "rápida" derruba a produção pela fila de lock, como medir antes de indexar, o que o
vacuum segura, quando o backup vale, como atualizar de major. Esta skill é o lado de operação.

### Adicionado
- `SKILL.md`: 15 pisos inegociáveis (versão suportada e EOL, `pg_stat_statements`, `lock_timeout`,
  `CONCURRENTLY`, `NOT VALID` + `VALIDATE`, sem reescrita sob lock, contract marcado, autovacuum,
  pooler, backup com restore testado, failover ensaiado, roles de menor privilégio, RLS com `FORCE`,
  credencial fora do SQL) e a fronteira com `database`, `data`, `infra`, `observability`.
- `references/`: `versao-e-config`, `explain-e-indices`, `vacuum-e-bloat`, `locks-e-migrations`,
  `pooling`, `replicacao-e-failover`, `backup-e-pitr`, `upgrade`, `roles-e-rls`, `stack-versoes`
  (anexo volátil com fonte e data; comportamentos de DDL provados em PostgreSQL 18.6).
- `scripts/check-pg-migration.sh` (gate estático de `.sql`, 24 regras, sem depender de Postgres) e
  `check-pg-migration.test.sh` (vermelhos e verdes plantados).
- Comandos `/pg-help`, `/pg-load`, `/pg-review`, `/pg-migrate`, `/pg-explain`, `/pg-backup`,
  `/pg-upgrade`, `/pg-claude`, `/pg-cc`, `/pg-handoff`; `assets/CLAUDE.md` sempre-on.
