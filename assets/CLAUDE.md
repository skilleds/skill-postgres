# Piso de operação do PostgreSQL (schematize-postgres) — sempre-on

O Postgres perdoa SQL errado e castiga SQL correto no momento errado. O desenho do schema é da
`schematize-database`; aqui é o motor.

1. **Só major suportado**; o EOL entra no calendário com 6+ meses de folga; minor sempre atual.
2. **`pg_stat_statements` + `track_io_timing` em todo cluster.** Meça antes de indexar:
   `EXPLAIN (ANALYZE, BUFFERS)` em dado de tamanho real.
3. **Todo DDL em tabela viva começa com `SET lock_timeout`** (e retry). A fila de lock derruba mais
   que o DDL.
4. **Índice é `CREATE INDEX CONCURRENTLY`** (idem `DROP`/`REINDEX`), sozinho na migration, fora de
   transação; `INVALID` se remove.
5. **Constraint nova é `NOT VALID` e depois `VALIDATE`** (FK, CHECK, NOT NULL via CHECK);
   `UNIQUE`/PK via índice `CONCURRENTLY` + `USING INDEX`.
6. **Nada de reescrever sob `ACCESS EXCLUSIVE`:** default volátil/serial/identity, `GENERATED ...
   STORED`, `ALTER COLUMN TYPE`, `VACUUM FULL`, `CLUSTER`. Nullable, backfill em lotes, troca.
7. **`DROP`/`RENAME`/`TRUNCATE` só na fase contract** com `-- pg-contract: <motivo>`.
   `UPDATE`/`DELETE` sem `WHERE` é vetado.
8. **Autovacuum nunca desligado**; tabela grande com parâmetros próprios; `age(datfrozenxid)` com alerta.
9. **`max_connections` baixo, PgBouncer na frente.** Estado de sessão não vai para o modo transação.
10. **Backup = base + WAL (PITR), cifrado, em outra conta, com restore medido todo mês.** `pg_dump`
    não é o backup de record.
11. **Slot de replicação com `max_slot_wal_keep_size`; failover ensaiado**, não presumido.
12. **Menor privilégio:** owner (migration) ≠ app (DML) ≠ leitura ≠ admin. App nunca superuser nem dona.
    `pg_hba.conf` com `scram-sha-256` + TLS, sem `trust`.
13. **RLS com `ENABLE` + `FORCE`**, tenant por `SET LOCAL`/`set_config(...,true)` na transação,
    fail-closed. Teste com dois tenants.
14. **Credencial nunca em `.sql` versionado**, migration ou log.
15. <!-- herdado:engineering/orquestracao:curto -->**Orquestrador não desenvolve; subagent barato executa.** O agent principal só planeja, despacha e revisa; ação onerosa vira micro-tasks para o executor mais barato que resolve: `haiku` (mecânico) / `sonnet` (código; falhou → mesmo subagent corrige, até 2 rodadas) → Codex `gpt-5.6-terra` → Cursor → subagent `opus` (motivo) → `park`; **visual vai direto ao Codex**; **imagem nunca pelo Claude: detectar Codex/Gemini e perguntar ao usuário qual usar (Codex é a sugestão), nunca API de imagem paga**; papéis em `agentes.toml`. No overdev, cada item do checklist vai a um subagent e o principal revisa antes do `- [x]`. **Sem frota ociosa:** idle com pendência volta ao trabalho; dependente de outro agent → mata e enfileira com gatilho; terminou → mata (§9.6). Detalhe: `schematize-engineering` → `references/orquestracao.md` §9.<!-- /herdado -->

16. **Quem executa:** SQL só nasce como arquivo versionado no git (`NNN_nome.up.sql` +
    `.down.sql`); a IA propõe, o **ops** aplica por operação nomeada (`ops migrate`, dry-run) e o
    humano confirma o destrutivo. `EXPLAIN` roda por query nomeada somente-leitura (sem `ANALYZE`
    em DML em prd). Vacuum/repack/restore/failover/upgrade/backfill são operações tipadas do ops.
    Restore para ensaio é mascarado antes de acesso humano/agente. Todo passo tem `down`; contract
    destrutivo = restore testado ou ADR (`-- pg-irreversivel: ADR-<n> <motivo>`).
<!-- herdado:engineering/pisos-22-29:bloco -->**Herda os pisos 22–29 da `schematize-engineering`** (texto completo em `assets/CLAUDE.md` dela): 22 tudo no git — sem remote, push falho ou HEAD≠origin a demanda NÃO se conclui; 23 deploy só do git (VETADO scp/rsync/tar|ssh/docker save|load/git bundle); 24 o ops é o gate da IA — a IA nunca muda a VPS direto nem escreve SQL, só operação nomeada (`ssh vps "ops <op>"`); 25 sem agents inúteis (entregou, morreu); 26 relatório a cada 30 min e handoff a cada 6 h; 27 CI remoto só antes do hml, a pedido ou em tag (`workflow_dispatch` + `hml-*`/`v*`; VETADO `pull_request`, `push` em branch, `workflow_run`, cron sem justificativa); 28 microserviço autônomo (fila durável + 7 testes de resiliência); 29 build/teste na máquina dev/hml quando existe (ponte só no workspace de build, nunca prd) + higiene (`scripts/higiene.sh`).<!-- /herdado -->

Mídia: imagem/visual gerado nunca por modelo Claude; detecte Codex/Gemini e pergunte qual usar.

Gate: `bash .claude/skills/schematize-postgres/scripts/check-pg-migration.sh <migrations>` —
waiver rastreável `-- pg-ok: <regra> <motivo>`.
