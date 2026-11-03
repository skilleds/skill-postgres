---
description: schematize-postgres — planeja o upgrade de major do PostgreSQL (pg_upgrade ou lógico) com ensaio, rollback e checklist
argument-hint: "<major de origem> <major de destino>"
---

# /pg-upgrade — major sem susto

1. **Confirme as versões e o EOL** em `references/stack-versoes.md` (reverifique a data). Calcule
   a folga até o EOL do major atual.
2. **Escolha a rota** (`references/upgrade.md`): `pg_upgrade --link` (downtime de minutos, sem
   volta sem backup) ou replicação lógica (downtime mínimo, mais trabalho). Registre em ADR.
3. **Levante os riscos do ambiente:** extensões e versões, collation/glibc do SO de destino,
   checksums nos dois lados (PG 18 liga por default), tabelas sem `PRIMARY KEY` (lógica), slots e
   réplicas, parâmetros removidos/alterados nas notas do major de destino.
4. **Monte o ensaio** em cópia de prd restaurada do backup e **mascarada antes de qualquer acesso
   humano/agente** (`references/backup-e-pitr.md` §4): `pg_upgrade --check`, execução
   completa, `vacuumdb --analyze-in-stages` (até PG 17), `pg_amcheck`, comparação de planos das
   top queries do `pg_stat_statements`, tempo de downtime medido.
5. **Escreva o plano de produção:** snapshot/backup imediatamente antes, janela, passos, métricas
   e critério de rollback definidos de antemão, reconstrução das réplicas, novo base backup e
   WAL após o corte.
6. **Execução = operação tipada do ops** (`ops upgrade`, dry-run com `pg_upgrade --check`); a IA
   planeja, o humano confirma o corte. Réplicas se refazem por Patroni/`pg_basebackup`/pgbackrest.
7. **Saída:** runbook (`schematize-docs`) + checklist com dono. Nunca execute em produção.
