---
id: forja-business-rules
type: specification
status: canonical
version: 1.0.0
created: 2026-06-17
updated: 2026-06-17
owners: [ayla, atena]
contributors: [atena, iris, artemis, hermes]
war-room: 2026-06-17-v2
related:
  - forja-story-map.md
  - forja-definicoes-finais.md
  - mockup-pricing-forja.html
---

# Forja — Regras de Negócio Canônicas

**Single source of truth para regras operacionais.**
Produto, Frontend, Backend e QA consultam este documento.
Story Map contém a jornada. Este documento contém as regras que governam cada etapa.

---

## Índice

→ [1. Definições](#1-definições)
→ [2. Tiers & Quotas](#2-tiers--quotas)
→ [3. Reset de Invocações](#3-reset-de-invocações)
→ [4. Memória & Contexto](#4-memória--contexto)
→ [5. Agentes por Tier](#5-agentes-por-tier)
→ [6. Roteamento de Modelo](#6-roteamento-de-modelo)
→ [7. Gates & Enforcement](#7-gates--enforcement)
→ [8. Estados de Erro & Recovery](#8-estados-de-erro--recovery)
→ [9. Edge Cases](#9-edge-cases)
→ [10. Compliance & LGPD](#10-compliance--lgpd)
→ [11. Decisões do War Room](#11-decisões-do-war-room)

---

## 1. Definições

| Termo | Definição |
|-------|-----------|
| **Invocação** | 1 chamada ao agente, iniciada com sucesso (POST /invoke aceito pelo backend). Streaming = 1 invocação. Cancelamento mid-flight pelo usuário = conta. Retry automático por erro 5xx do servidor = NÃO conta. Retry iniciado pelo usuário = conta. |
| **Sessão** | Conjunto de turns vinculados a um `session_id`. Persiste cross-sessão (dias, semanas). Uma sessão = um agente. |
| **Turn** | 1 par prompt+response dentro de uma sessão. Turn incrementa apenas após invoke bem-sucedido + memória salva. |
| **AHA Moment** | 2ª invocação onde o agente retoma contexto sem re-explicação do usuário (AHA-2). AHA-1 = sinalização na 1ª resposta ("I'll remember this"). |
| **Janela de uso** | Período no qual invocações são contadas. Free = mensal (dia 1º ao último do mês). Pro/Team = rolling por atividade. |
| **Tenant** | Organização/workspace de um usuário (org_id). Billing é por usuário, não por tenant. |
| **Agente default** | Ayla — primeiro agente que o usuário encontra. Orquestradora geral. |

---

## 2. Tiers & Quotas

### 2.1 Tabela de Tiers

| Tier | Invocações | Agentes | Preço | Mínimo | Estado |
|------|-----------|---------|-------|--------|--------|
| **Free** | 10/mês | 1 (Ayla) | R$0 | — | R0 (não enforced), R1 (enforced) |
| **Pro** | Janela 5h | Todos (15) | R$99/mês/usuário | — | R0 (não enforced), R1 (enforced) |
| **Team** | Pool 8h compartilhado | Todos (15) | R$119/assento/mês | 3 assentos | R0 (não enforced), R1 (enforced) |

> **Nota R0 (hoje):** Nenhum limite é enforced. Tiers existem no produto mas sem billing counter implementado. Gate obrigatório para R1.

### 2.2 Free Tier — Detalhamento

- **Invocações:** 10 por mês calendário (não por dia, não rolling)
- **Agente disponível:** Ayla (exclusivo — não escolha do usuário)
- **Reset:** Dia 1º de cada mês, 00:00 UTC-3 (horário de Brasília)
- **Hard limit:** Na 11ª invocação → HTTP 429 + mensagem de upgrade
- **Degradação:** NÃO. Sem fallback para Qwen-only após limite. Hard block.
- **Modelo no Free:** Qwen (casual) + Sonnet (técnico), até a quota
- **War room:** NÃO disponível (Team tier)
- **Soft daily limit:** Não há — ICP pode fazer sprint de 3 dias sem punição. Risco de exaurir no dia 1 é aceito (leva ao AHA + conversão)

### 2.3 Pro Tier — Detalhamento

- **Invocações:** Janela rolling de 5h a partir da última invocação do usuário
- **Sonnet:** 50 invocações por janela
- **Opus:** 20 invocações por janela
- **Reset:** 5h após última invoke (não meia-noite, não dia 1º)
- **Agentes:** Todos os 15 ativos
- **Pausa no limite:** 429 + timestamp do reset + CTA Team
- **Modelo:** Qwen + Sonnet (default técnico) + Opus (heavy tasks)
- **Soft daily limit:** Não há

### 2.4 Team Tier — Detalhamento

- **Invocações:** Pool compartilhado entre todos os assentos da org
- **Pool Sonnet:** 150 por janela de 8h
- **Pool Opus:** 60 por janela de 8h
- **Reset:** 8h após última invoke de qualquer membro da org
- **Agentes:** Todos os 15 ativos
- **Pausa no limite:** 429 + timestamp do reset + notificação para o admin da org
- **War room:** Disponível (orquestração multi-agente)
- **Audit log:** Quem invocou, qual agente, quando — visível para admin
- **Workspace compartilhado:** Sessões visíveis para todos os membros

---

## 3. Reset de Invocações

### 3.1 Free — Reset Mensal

- **Quando:** Dia 1º de cada mês às 00:00 UTC-3
- **Implementação:** Cron job diário verifica se `now >= resetAt`. Se sim, `SET count = 0, resetAt = next_reset`.
- **Fuso padrão:** UTC-3 (Brasília). Se timezone do tenant não armazenado, usar UTC-3 como default.
- **Notificação:** Email D-1 antes do reset: "Suas 10 invocações renovam amanhã."

### 3.2 Pro — Janela Rolling 5h

- **Quando:** 5h após a última invocação do usuário
- **Implementação:** Redis counter com TTL = 5h. Key: `quota:{userId}:sonnet` / `quota:{userId}:opus`. TTL reseta a cada invoke.
- **Exemplo:** Usuário invoca às 14:00 → janela expira às 19:00. Invoca às 16:00 → janela expira às 21:00.
- **Sem notificação automática:** CLI/Web mostra timestamp de reset após cada invoke.

### 3.3 Team — Pool Rolling 8h

- **Quando:** 8h após a última invocação de qualquer membro da org
- **Implementação:** Redis counter com TTL = 8h. Key: `quota:{orgId}:sonnet` / `quota:{orgId}:opus`.
- **Admin notificado:** Quando pool atingir 80% → alerta no dashboard admin.

### 3.4 Transições de Tier (Upgrade/Downgrade)

- **Upgrade Free → Pro:** Reset imediato da janela (usuário ganha janela cheia). Goodwill.
- **Upgrade Pro → Team:** Reset imediato do pool. Goodwill.
- **Downgrade Pro → Free:** Limites novos aplicam-se no próximo ciclo mensal (grace period até dia 1º). Invocações já usadas no mês corrente não são recalculadas.
- **Downgrade Team → Pro:** Limites novos no próximo ciclo. Sessions compartilhadas viram sessions individuais do owner.
- **Cancelamento:** Plano ativo até fim do período pago. Depois, downgrade para Free.

---

## 4. Memória & Contexto

### 4.1 Memory TTL por Tier

| Tier | TTL (inatividade) | Comportamento |
|------|-------------------|---------------|
| **Free** | 30 dias | Após 30 dias sem nenhuma invoke na sessão → soft archive (oculta da UI, mantém no banco) |
| **Pro** | 180 dias | Após 180 dias → soft archive |
| **Team** | 365 dias | Após 365 dias → soft archive |

- **Hard delete:** 180 dias após soft archive (independente do tier)
- **Recovery:** Possível até hard delete. Via dashboard (self-serve) ou suporte.
- **Transparência:** Usuário NÃO vê TTL explícito na UI (invisível). Se houver notificação de purge, avisar D-7 via email.

> **Rationale (Hermes):** TTL Free de 30 dias (não 7) — ICP tem ciclos erráticos, pode sumir por 2-3 semanas e voltar. TTL curto mataria o AHA e seria percebido como SaaS predatório. 30 dias cobre sprints normais de founder.

### 4.2 Contexto por Invoke (Session Window)

- **Regra base:** Últimos 20 turns da sessão
- **Teto de tokens por tier:**

| Tier | Max tokens de contexto |
|------|------------------------|
| **Free** | 32.000 tokens |
| **Pro** | 100.000 tokens |
| **Team** | 200.000 tokens |

- **Estimativa de tokens:** `aproximado = (prompt.length + response.length) / 4` (heurística simples)
- **Truncamento:** Se 20 turns > teto do tier → carregar os turns mais recentes que cabem no budget (prioridade decrescente por recência)
- **Aviso ao usuário:** Se sessão tem ≥15 turns → badge amarelo "Sessão longa". Se ≥25 turns → badge vermelho. Não bloqueia.
- **CLI:** Exibe aviso no output: `⚠️ Session has 18 turns. Consider starting a new session for better performance.`

> **Rationale (Hermes + Artemis):** Hermes: same context no free e pro = melhor DX. Artemis: teto técnico necessário (20 turns pode ser 100k tokens com code-heavy sessions). Compromisso: teto maior no free (32k) do que limitar contexto por tier.

### 4.3 Comportamento de Warm-up

- **Timeout de carga de memória:** 5s para carregar contexto do Neon.
- **Se timeout:** Fallback para sessão stateless (sem contexto anterior) + aviso ao usuário: "Context load timed out — continuing without session memory."
- **Warm-up NÃO conta como invocação** se falhar (timeout ou erro de storage).

---

## 5. Agentes por Tier

### 5.1 Whitelist por Tier

| Tier | Agentes disponíveis |
|------|---------------------|
| **Free** | Ayla |
| **Pro** | Todos (15 ativos): Ayla, Apolo, Artemis, Atena, Afrodite, Gaia, Hefesto, Hera, Hermes, Íris, Musa, Quiron, Yara, Zeus, Atlas |
| **Team** | Todos (15) + War Room (orquestração multi-agente) |

### 5.2 Catálogo na UI

- **Free:** Mostra catálogo completo (15 agentes) com badge "Pro" nos bloqueados. Hover = tooltip de upgrade. Click = modal de upgrade.
- **Pro:** Mostra todos disponíveis. Badge "Team" no War Room.
- **Team:** Tudo desbloqueado.

> **Rationale (Hermes + Íris):** Catálogo visível com lock = transparência + FOMO específico (ICP técnico vê "Apolo para backend" bloqueado = motivação para upgrade). Esconder seria antipadrão (Vercel: usuários reclamam de "não sabia que tinha isso" post-churn).

### 5.3 Ayla como Default

- Ayla é o único agente no Free e o agente default ao criar nova sessão no Pro/Team.
- Razão: Ayla é orquestradora, conversa sobre múltiplos domínios, reduz friction de onboarding.
- Ayla pode sugerir especialista: "Para análise de backend aprofundada, o Apolo seria mais preciso. Quer trocar?" (Pro/Team).

### 5.4 Enforcement Técnico

```
app-layer (não RLS):
  if agentId NOT IN allowedAgents[plan]:
    → HTTP 403 "Agent not available on your plan"
    → Link upgrade
```

---

## 6. Roteamento de Modelo

### 6.1 Tabela de Roteamento

| Modelo | Tiers | Trigger |
|--------|-------|---------|
| **Qwen 2.5 Coder 32B** | Free, Pro, Team | Prompt <200 chars, sem código, sem keywords arquiteturais. Papo casual. |
| **Claude Sonnet 4.6** | Free (até quota), Pro (default), Team (default) | Prompt com código, arquitetura, decisões, refactor, review. >80% das invocações. |
| **Claude Opus 4.8** | Pro, Team (NÃO Free) | War room, refactor >500 linhas, handoff Ayla→Atena, task classificada como "heavy". |

### 6.2 Heurística de Roteamento

```
1. Detecta código (```, function, class, import, async)
2. Detecta arquitetura (architecture, design, refactor, ADR, migration, schema)
3. Detecta heavy task (war room, refactor all, 500+ lines, deploy)

Rota:
- Sem match → Qwen
- Match código/arquitetura + Free → Sonnet (até quota)
- Match código/arquitetura + Pro/Team → Sonnet
- Match heavy + Pro/Team → Opus
- Opus no Free → fallback Sonnet + aviso "Opus disponível no Pro"
```

### 6.3 Fallback de Modelo

- Opus indisponível → fallback Sonnet + `warning: "Using Sonnet (Opus unavailable)"`
- Sonnet indisponível → fallback Qwen + aviso `"degraded mode: using Qwen"`
- Qwen indisponível → fallback Sonnet (sem aviso de degradação)

---

## 7. Gates & Enforcement

### 7.1 Usage Gate (sequência obrigatória no invoke_agent)

```
POST /api/v1/invoke_agent
  │
  ├─ 1. Auth (JWT válido) → 401 se não autenticado
  ├─ 2. Plan check → effectivePlan(userId) ∈ {FREE, PRO, TEAM}
  ├─ 3. Agent whitelist → agentId ∈ allowedAgents[plan] → 403 se não
  ├─ 4. Usage check → count < maxPerWindow → 429 se excedido
  ├─ 5. Invoke agent (core/olympo)
  ├─ 6. Record usage (ATÔMICO) → fail-closed se falhar (não retorna resposta)
  └─ 7. Save turn na session_memory → fail-open se falhar (retorna resposta com aviso)
```

### 7.2 Atomicidade do Billing Counter

- **Implementação:** UPSERT PostgreSQL (INSERT ... ON CONFLICT UPDATE) sobre unique constraint `(tenantId, userId, scope)`
- **Race condition:** Edge-case na virada de janela (2 requests simultâneos) → usuário pode ganhar 1 request a mais. Impacto: benigno (fail-open no edge-case raro). Mitigação futura: `SELECT ... FOR UPDATE` se frequência aumentar.
- **Fail-closed:** Se `incrementUsage()` falhar → backend NÃO retorna resposta ao usuário. Log de `BILLING_INCONSISTENCY`. Usuário vê 500 com mensagem: "Billing error — request not counted. Try again."

### 7.3 Precedência de Gates (quando múltiplos limites aplicam)

Ordem de verificação (primeiro gate que falhar retorna):
1. Auth
2. Agent whitelist (tier)
3. Quota mensal (Free) ou quota da janela (Pro/Team)

### 7.4 Rate Limit por IP (Anti-Abuse Free)

- Máximo 3 signups por IP por hora
- Máximo 5 signups por IP por dia
- Implementação: Redis counter com TTL. Retorna 429 se excedido.
- Razão: Free com 10 invocações pode ser abusado (100 emails = 100 invocações/dia).

---

## 8. Estados de Erro & Recovery

### 8.1 Quota Exceeded

| Tier | Código | Mensagem | CTA |
|------|--------|----------|-----|
| Free (mensal) | 429 | "Free tier limit reached (10/10). Resets on [data]." | "Upgrade to Pro (R$99/mês)" |
| Pro (janela) | 429 | "Usage window exhausted. Resets in {time}." | "Upgrade to Team" |
| Team (pool) | 429 | "Team pool exhausted. Resets in {time}." | "Contact admin" |

**Sinalização preventiva (antes do bloqueio):**
- 8/10 usado (Free) → badge amarelo: "2 invocations left"
- 10/10 (Free) → badge vermelho: "Limit reached", botão Invoke desabilitado
- 80% da janela (Pro/Team) → aviso no dashboard admin (Team only)

### 8.2 Agent Not Available

| Código | Mensagem |
|--------|----------|
| 403 | "Agent '{name}' requires {tier} tier. [Upgrade →]" |

### 8.3 Memory Load Error

| Cenário | Comportamento |
|---------|---------------|
| Timeout (>5s) | Continua stateless + aviso: "Context load timed out — starting fresh." |
| Storage error | Continua stateless + aviso: "Context unavailable — this invoke won't remember session history." |
| Session not found | 404: "Session not found. Start a new session: `forja session new`" |
| TTL expirado (archived) | "Session archived (30+ days inactive). [Restore →] or start new." |

### 8.4 Billing Error

| Código | Mensagem | Comportamento |
|--------|----------|---------------|
| 500 | "Billing error — request not counted. Try again." | Fail-closed: não retorna resposta do agente |

### 8.5 Cartão Recusado (Stripe)

- Grace period: 7 dias após falha de cobrança (retenta 3x em 3 dias)
- Após grace period: downgrade para Free até regularização
- Email: D-0 (falha), D-3 (aviso), D-7 (downgrade iminente), D-8 (downgraded)
- Self-cure: usuário atualiza cartão → tier restaurado imediatamente

---

## 9. Edge Cases

### 9.1 Definição de Invocação (gray areas)

| Cenário | Conta como invocação? |
|---------|----------------------|
| Streaming completo | Sim (1 invocação) |
| Usuário cancela mid-flight | Sim (request iniciado = conta) |
| Retry automático por 5xx do servidor | Não (erro do servidor) |
| Retry manual pelo usuário | Sim |
| Timeout de resposta (agente não respondeu) | Não (se não chegou resposta) |
| Warm-up de memória falhou (timeout) | Não conta se invoke não foi processado |

### 9.2 Upgrade Mid-Ciclo

- **Free → Pro:** Janela Pro começa imediatamente. Reset da cota Free. Goodwill.
- **Pro → Team:** Pool Team começa imediatamente. Sessões individuais tornam-se compartilhadas.
- **Qualquer downgrade:** Novos limites no ciclo seguinte (grace period até próximo reset).

### 9.3 Memória: Sessão Ativa vs Expirada

- Sessão ativa (usada nos últimos N dias per TTL do tier): carrega contexto normalmente
- Sessão arquivada (TTL expirado): UI mostra badge "Archived". Click = prompt de restore (grátis, re-ativa TTL)
- Restore automático: se usuário invoca com `--session <id>` de sessão arquivada → restore silencioso + continua
- Hard delete: 180d após archive. Irrecuperável. Notificação por email D-7

### 9.4 Limites Simultâneos

- Usuário Free atingiu quota mensal E está em grace period de downgrade → quota mensal tem precedência (429 primeiro)
- Mensagem deve ser específica ao gate ativo (não genérica "upgrade")

### 9.5 Sessão Multi-Agente (Team)

- War room cria `session_id` compartilhado para todos os agentes da sala
- Cada invoke dentro do war room conta do pool do tier (1 invoke = 1 do pool, independente de quantos agentes respondem)
- Timeout de agente no war room: 30s por agente. Se timeout → output parcial com marcação `[TIMEOUT: agent_name]`

---

## 10. Compliance & LGPD

### 10.1 Direito de Esquecimento

- **Solicitação:** Usuário pode deletar conta via dashboard (self-serve) ou email suporte@forja.dev
- **Prazo:** Hard delete em 48h após confirmação
- **Escopo:** session_memory, project_index_snapshot, subscription, user profile
- **Exceção:** Logs de billing (transações Stripe) retidos por 5 anos (obrigação fiscal BR)

### 10.2 Portabilidade de Dados

- Usuário pode exportar todas as sessões: `forja session export --all --format json`
- Export inclui: session_id, agent_id, turns (prompt + response), timestamps
- Export NÃO inclui: dados de billing, logs internos de infra

### 10.3 Localização de Dados

- Dados em: us-east-1 (Neon US) — atual
- Migração planejada: sa-east-1 (Neon SA) em Q3+ (após R1)
- Disclosure: presente na privacy policy e no signup

### 10.4 Auto-Archive & Purge

- D-7 antes de auto-archive: email "Your session '{name}' will be archived in 7 days."
- D-7 antes de hard delete: email "Your archived session '{name}' will be permanently deleted in 7 days. Export or restore."
- Restauração: sempre disponível até hard delete (self-serve)

---

## 11. Decisões do War Room

**War room:** 2026-06-17 v2 · Atena · Íris · Artemis · Hermes · Decisor: Ayla

| ID | Regra | Decisão | Rationale | Alternativa descartada |
|----|-------|---------|-----------|------------------------|
| BR-01 | Reset Free | **Mensal (dia 1º)**, não diário | ICP faz sprints de 2-3 dias — reset diário puniria uso legítimo. Previsibilidade de budget. | Diário (punitive para sprints) |
| BR-02 | TTL Free | **30 dias** de inatividade | ICP tem ciclos erráticos. 7 dias mataria o AHA. 30d cobre sprints normais. | 7 dias (mataria moat), Infinito (COGS incontrolável) |
| BR-03 | Contexto | **Mesmo para todos** (gate por invocações, não por contexto) | Limitar contexto no free quebra o teste de valor real. ICP técnico detecta e desconfia. | Contexto menor no free (antipadrão Copilot) |
| BR-04 | Agente Free | **Ayla exclusivo**, catálogo visível com lock | Reduz friction, Ayla = rosto do produto. Catálogo visível = FOMO específico para upgrade. | Escolha de 1 entre 15 (friction), esconder catálogo (antipadrão) |
| BR-05 | 11ª invoke Free | **Hard block** (429), sem degradação | Free = acquisition tool, não produto degradado. Qwen-only dilui o moat e canibaliza Pro. | Qwen-only após limite |
| BR-06 | Janela Pro | **5h rolling** após última invoke | Sprints não são punidos (não perde janela no meio da sessão). Modelo Claude (familiar para ICP). | 5h fixed (midnight) |
| BR-07 | Catálogo UI | **Mostrar todos com lock** | Transparência + aspiração. "Não sabia que tinha isso" = churn post-launch. | Esconder locked (Vercel-style) |
| BR-08 | Billing fail | **Fail-closed**: não retorna resposta se billing não foi registrado | Billing inconsistente = dívida técnica acumulada. Fail-open criaria free usage infinito. | Fail-open + corrigir depois |
| BR-09 | Invocação def | **Request iniciado = conta**, exceto retry 5xx servidor | Previsível para o usuário. Alinhado com OpenAI (quem conta é a tentativa do usuário). | Só contar se response completa (complexo de implementar) |
| BR-10 | Downgrade | **Grace period até próximo ciclo** | Não punir usuário que já pagou até o fim do mês. | Downgrade imediato (antipadrão Vercel que gerou reclamações) |

---

**Documento encerrado. Source of truth de regras de negócio do Forja.**
**Próxima revisão:** Com primeira implementação real de R1 (billing counter vivo).

Para implementação técnica: ver `forja-story-map.md` (seção 7 — Dependências) e `migrations/0068_forja_session_memory_billing_rls.sql` (proposta Artemis).
