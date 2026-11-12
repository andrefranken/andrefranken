# 0001. Registrar decisões de arquitetura com ADRs

- Status: Aceito
- Data: 2026-11-12

## Contexto
Decisões estruturais precisam de rastreabilidade e contexto histórico para o time e para os
subagentes (que leem `docs/adr/` antes de planejar).

## Decisão
Adotamos ADRs no formato MADR em `docs/adr/NNNN-<slug>.md`, numerados e imutáveis. Mudança de
decisão = nova ADR que substitui a anterior.

## Consequências
- (+) Histórico auditável; onboarding mais rápido; subagente `architect` tem fonte de verdade.
- (−) Disciplina de manter o registro atualizado.

## Confirmação
Toda decisão de carga estrutural referenciada em um PR aponta para uma ADR existente.
