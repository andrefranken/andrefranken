# Constituição do Projeto — Plataforma de Orquestração Multi-Agente (codinome "Regente")

> Princípios inegociáveis. São a fonte de verdade de mais alto nível: specs, planos,
> tarefas e código DEVEM respeitá-los. Mudar um artigo exige um ADR (`/adr`) que
> justifique a alteração. Versionado em git, imutável por sessão.

**Versão:** 1.0.0 · **Status:** Proposta (aguardando ratificação humana) · **Data:** 2026-11-12
**Stack (padrão da skill):** Next.js 15 (App Router) + TypeScript 5 no plano de controle.
**⚠ Pendente:** a escolha final de stack (TS ponta a ponta vs. componentes em Python) será
decidida na fase de plano e registrada em ADR-0002. Ver spec 001, questão Q1. Se a decisão
for Python em alguma fronteira, os Artigos I, II e V ganham equivalentes nessa linguagem
(ex.: Pydantic no lugar de Zod), sem afrouxar nenhuma regra.

---

## Artigo I — Arquitetura é lei (Hexagonal / Ports & Adapters)

1. O **domínio** (`src/modules/<contexto>/domain/`) é TypeScript puro: ZERO `import`
   de `next/*`, `react`, Prisma/Drizzle, SDK de provedor de LLM ou qualquer cliente de I/O.
2. A **aplicação** (`application/`) depende somente de interfaces (ports) do domínio.
3. **Infraestrutura** (`infrastructure/`) implementa ports; nunca é importada pelo domínio.
4. Server Actions e Route Handlers são **adaptadores primários** finos: validam (Zod),
   resolvem dependências no composition root e traduzem `Result` em resposta.
5. Provedores de modelo (LLM), servidores MCP, conectores e sandboxes de código são
   **adaptadores secundários** atrás de ports. Nenhum provedor é referenciado pelo domínio.
6. Violação de fronteira de import = **build quebrado** (lint + hook bloqueante).

## Artigo II — Tipos e erros honestos

1. Validação **Zod** em toda fronteira não confiável (entrada de adaptador, env, payloads,
   **saída de modelo e resultado de ferramenta**).
2. IDs e value objects são **branded types** (`.brand<"...">()`), nunca `string` cru.
3. Erros esperados usam `Result<T,E>` (neverthrow); `throw` apenas para erro de programador.
4. `any` é proibido. `tsconfig` em `strict: true` + `noUncheckedIndexedAccess`.

## Artigo III — Segurança por construção

1. **Autorização** é verificada no caso de uso (camada de aplicação), nunca só na UI.
2. **RBAC**: toda ação sensível declara a permissão exigida; default = negar.
3. Segredos só via env validado ou cofre; nunca commitados, nunca em código cliente,
   **nunca no contexto de um modelo**.
4. Toda entrada é validada e toda saída para o cliente é um DTO explícito (sem vazar entidade).
5. Mudança que toca dados pessoais, auth, permissões, credenciais, sandbox ou políticas de
   IA exige passagem pelo `security-auditor`.
6. **Segregação de funções (SoD)**: quem cria uma versão não pode aprovar a publicação dela
   em produção.

## Artigo IV — Separação de responsabilidades física

1. **Backend de domínio** vive em `src/modules/` (isolado do framework).
2. **Frontend** (componentes, design system, estado de UI, canvas) vive em `src/ui/` e `app/`.
3. **APIs** (contratos HTTP) vivem em `app/api/` e têm schema OpenAPI derivado de Zod.
4. **Design system** é uma fronteira própria (`src/ui/design-system/`), com tokens versionados.
5. Comunicação entre bounded contexts é por **eventos de domínio**, nunca import direto.
6. O **motor de execução** (workers) é um adaptador primário próprio (`src/jobs/`): consome
   eventos/filas e invoca casos de uso. Não contém regra de negócio.

## Artigo V — Qualidade verificável (Testing Trophy)

1. Domínio: ≥95% de cobertura de linha + ≥80% de mutation score (Stryker).
2. Casos de uso: ≥90% com adaptadores in-memory (inclui **provedor de LLM falso
   determinístico**; nenhum teste de unidade chama modelo real).
3. Integração de adaptadores: Testcontainers (Postgres real), ≥80%.
4. E2E: Playwright só nas jornadas críticas.
5. Nenhum código entra em `main` sem passar no pipeline completo.

## Artigo VI — Spec é a fonte de verdade

1. Nada é implementado sem uma spec aprovada (`/specify` → revisão humana).
2. A ordem é sagrada: **constituição → spec → plano → tarefas → implementação → revisão**.
3. Cada decisão arquitetural relevante vira um **ADR** imutável em `docs/adr/`.
4. Specs e ADRs são versionados em git e sobrevivem entre sessões.

## Artigo VII — Trabalho em fatias pequenas e reversíveis

1. Uma tarefa = um commit atômico (Conventional Commits).
2. Trunk-based com branches curtas + feature flags. Sem big-bang.
3. Tarefas paralelizáveis são marcadas com `[P]` e podem ir a subagentes concorrentes.
4. Cada PR referencia a spec e o ADR que o originaram.

## Artigo VIII — Observabilidade da plataforma é parte do pronto

1. Nenhuma feature é "pronta" sem instrumentação: caso de uso com métricas RED e span nomeado.
2. Rotas críticas declaram SLI/SLO com error budget; alertas disparam por taxa de queima, não limiar fixo.
3. Todo erro que estoura ao usuário vai ao Error Ledger (com traceId, sem PII) e vira backlog.
4. `readiness` checa as dependências novas (DB, fila, serviço externo); há health/liveness.
5. Tracing distribuído correlaciona HTTP → caso de uso → adaptador → fila → worker (W3C traceparent).
6. Métricas USE de recursos (DB, fila, CPU, memória) e profundidade de outbox/DLQ são monitoradas.

## Artigo IX — IA governada e rastreável (específico deste produto)

1. Toda chamada a modelo e toda chamada de ferramenta feita por agente é **registrada**
   (entrada efetiva, modelo e versão, parâmetros, saída, custo, latência) e ligada à execução.
2. Conteúdo vindo de modelo, ferramenta ou fonte externa é **não confiável**: nunca altera
   permissões, nunca amplia ferramentas disponíveis, nunca é executado sem sandbox.
3. Agentes executam com **identidade própria e privilégio mínimo**; nunca herdam permissões
   administrativas de quem os criou.
4. Ação classificada como **irreversível** exige aprovação humana em produção, salvo política
   aprovada que dispense explicitamente.
5. Todo agente/fluxo tem **limites declarados** (passos, tokens, tempo, custo); sem limite = inválido.

## Artigo X — Dados regulados e conformidade (específico deste produto)

1. Nenhum dado sai para um provedor de modelo que não esteja **aprovado** para o ambiente e
   para a classificação daquele dado.
2. Políticas de proteção de dados (mascaramento de PII, residência, retenção) são aplicadas
   **antes** do envio ao provedor e são verificáveis por teste.
3. A trilha de auditoria é **append-only e verificável contra adulteração**; nunca é editada
   nem apagada, inclusive por administradores.
4. Expurgo por retenção ou por pedido de titular (LGPD) preserva só os metadados de auditoria
   exigidos, sem conteúdo pessoal.

## Artigo XI — Assíncrono por padrão: mensageria para escala e desempenho (específico deste produto)

1. Todo trabalho aceito (gatilho, passo de execução, chamada de ferramenta, mensagem de canal,
   notificação, registro de auditoria) trafega por **mensageria durável**. Nenhum contexto depende
   de chamada síncrona a outro para concluir um trabalho aceito.
2. Eventos de domínio são publicados via **Outbox na mesma transação** que persiste o agregado;
   um relay publica depois (conforme `references/messaging-async.md` da skill).
3. Entrega **pelo menos uma vez** + consumidores **idempotentes**: reentrega nunca duplica efeito.
4. Toda fila tem política de retentativa, **quarentena (DLQ)** visível ao operador e métrica de
   profundidade (Art. VIII.6).
5. A ordem é garantida por chave de partição (execução, conversa), nunca globalmente.
6. Cada tipo de worker escala horizontalmente e de forma independente; contrapressão respeita os
   limites de taxa dos provedores externos.
7. A tecnologia de mensageria fica atrás de ports (`src/messaging/ports`); trocá-la é decisão de ADR,
   não de reescrita.
