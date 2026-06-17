---
id: forja-story-map
type: product-artifact
status: canonical
version: 2.0.0
created: 2026-06-17
updated: 2026-06-17
owners: [ayla, gaia]
contributors: [atena, iris, artemis]
pack: product-foundation
artifact: story-map
scope: forja-saas
framework: user-story-mapping-patton
icp: founder-tecnico-solo-dupla-pre-seed-seed
related:
  - forja-definicoes-finais.md
  - forja-v2/produto/forja-business-rules.md
  - forja-value-proposition-canvas.html
  - forja-sistema-arquitetura.html
  - mockup-fit-staircase.html
next_review: 2026-07-01
---

# Forja — User Story Map

**Source of Truth — Camada de Produto**
War room v2 2026-06-17: Ayla (decisor) · Atena · Íris · Artemis · Hermes · Gaia.
Correção canônica: Olimpo = motor interno (invisível). Forja = único produto público.
Decisões D1–D6 (v2). Backbone: 6 atividades. Releases: R0 (hoje) → R1 (Q3 gate) → R2/R3.

---

## Índice

**Para Produto:**
→ [1. Contexto](#1-contexto-de-produto) | [2. Persona](#2-persona-primária--lucas) | [3. Backbone](#3-backbone) | [4. Story Map](#4-story-map) | [5. Releases](#5-releases-e-walking-skeleton)

**Para Dev:**
→ [3. Backbone](#3-backbone) | [6. Task Flows](#6-task-flows-detalhados) | [7. Dependências](#7-dependências-técnicas-e-bloqueios) | [8. Mockups](#8-mockup-tracking)

**Para QA:**
→ [6. Task Flows](#6-task-flows-detalhados) (Acceptance Criteria + Error States) | [7. Dependências](#7-dependências-técnicas-e-bloqueios)

---

## 1. Contexto de Produto

### 1.1. Moat

**Agentes que lembram sem repetir contexto.**
A segunda invocação é o momento AHA: usuário percebe que o agente retoma o trabalho exatamente de onde parou, sem reamble, sem reprocessamento. Diferencial técnico: memória estruturada (Neon PostgreSQL) + orquestração local-first (CLI em localhost:8080 → fallback EC2).

**Moat visível no R1:** Free tier (10 invocações, 1 agente) leva usuário até a segunda invocação — suficiente para experimentar o moat. Conversão Pro = precisa de mais agentes ou mais invocações.

**Moat completo (3 camadas):**
1. Memória episódica nativa — contexto cross-sessão persistente (Neon), não RAG stateless
2. Personas psicométricas evolutivas — calibração DISC/Eneagrama por agente
3. Veto hierarchy — decisões críticas com fallback para aprovação humana

### 1.2. ICP

**Hoje (v0.5.0, N=3):**
- **Lucas** — founder técnico, solo ou dupla, pré-seed a seed, usa CLI npm, precisa de agentes que "não esquecem" entre sessões.
- Tamanho atual: N=1 externo + 2 founders internos. MRR R$297.

**Q3+ (R1 target):**
- Mesmo ICP, mas self-serve (signup público, free tier enforced, Stripe ativo).
- Meta R1: N=10 usuários free, N=3 conversões Pro.

**ICP Target Post-Q3 (Hypothesis — NÃO validado):**
- CTO / VP Engineering, Series A-B, 5-15 devs. Validar Q3+.

### 1.3. Arquitetura Real (v0.5.0)

| Componente   | Estado Hoje                                                        |
|--------------|--------------------------------------------------------------------|
| **CLI**      | npm global, local-first (localhost:8080 → EC2 fallback), funcional |
| **Agentes**  | 15 agentes ativos, memória Neon, invoke funciona                   |
| **Mobile**   | Angular, privado, não deployado, EC2-only                          |
| **Web**      | Em construção, EC2-only                                            |
| **Auth**     | Supabase JWT, tenant manual (admin cria via script)                |
| **Billing**  | Ausente — tiers existem na teoria, sem enforcement                 |
| **Memória**  | Neon PostgreSQL, turn-based, funciona, sem limite enforced         |

### 1.4. Estado Real vs Q3 Foundation

**v0.5.0 (R0 — hoje):**
- ✅ Invoke funciona (tenant manual)
- ✅ Memória Neon ativa
- ✅ CLI npm instalável
- ❌ Sem self-serve signup
- ❌ Sem Stripe
- ❌ Free tier não enforced
- ❌ invoke_agent requer role admin (não liberado para role comum)
- ❌ Onboarding manual (>30min)

**Q3 Foundation (R1 — gate obrigatório):**
- ✅ Self-serve signup (email/senha, Supabase)
- ✅ Stripe ativo (checkout Pro/Team)
- ✅ Free tier enforced (10 invocações, 1 agente, hard limit)
- ✅ invoke_agent liberado para role comum (RLS ajustado)
- ✅ Onboarding <5min (signup → CLI setup → 1ª invocação)
- ✅ CLI setup <5min (npx one-liner)

**⚠️ Nenhum release SaaS público antes de R1 completo.**

### 1.5. Tiers

| Tier       | Invocações/mês | Agentes   | Preço              | Estado Hoje        |
|------------|----------------|-----------|--------------------|--------------------|
| Free       | 10             | 1         | R$0                | Não enforced       |
| Pro        | Ilimitado      | 5         | R$99/mês           | Manual, sem Stripe |
| Team       | Ilimitado      | 15        | R$119/assento/mês  | Manual, sem Stripe |

---

## 2. Persona Primária — Lucas

**Lucas, 32 anos, founder técnico, pré-seed**
CTO ou founder solo, stack Python/TS, equipe de 1–2, orçamento apertado, precisa de automação que "não esquece" entre sessões.

### Jobs to Be Done

1. Invocar agente para revisar código — feedback técnico sem contratar sênior
2. Aplicar output do agente no código — diff/patch direto, sem copiar/colar manual
3. Retomar trabalho de ontem — agente lembra o contexto da última sessão
4. Coordenar múltiplos agentes (P1, Team tier) — orquestrar Ayla + Artemis + Gaia em war room

### Pains

- **Contexto perdido:** LLMs "esquecem" entre sessões — tem que re-explicar tudo
- **Setup longo:** ferramentas enterprise exigem onboarding >1h, integrações complexas
- **Custo opaco:** pricing por token, overage surpresa, difícil de prever
- **Output não aplicável:** resposta em texto solto, sem patch/diff/PR automatizado

### Gains

- **Memória visível na 2ª invocação** — agente retoma sem re-explicação (moat)
- **Setup <5min** — npx one-liner, primeira invocação em <5min
- **Free tier honesto** — 10 invocações para testar o moat antes de pagar
- **Output aplicável** — diff/patch pronto para `git apply` ou PR draft

---

## 3. Backbone

**6 atividades — caminho do usuário do signup até o moat e além.**

| #  | Atividade  | Outcome                                              | Frequency       | Priority |
|----|------------|------------------------------------------------------|-----------------|----------|
| 1  | Avaliar    | Decidir se Forja resolve o problema                  | Once (signup)   | P0       |
| 2  | Integrar   | Setup CLI + credenciais em <5min                     | Once (onboard)  | P0       |
| 3  | Invocar    | Executar 1ª e 2ª invocação (AHA = memória visível)  | Daily           | P0       |
| 4  | Validar    | Verificar output + iterar (refinamento)              | Per invocation  | P0       |
| 5  | Retomar    | Sessão seguinte — agente lembra contexto             | Weekly          | P0       |
| 6  | Escalar    | War room multi-agente (Team tier)                    | Monthly         | P1       |

**Decisões de backbone (D2):**
- "Iterar" absorvido por "Validar" — refinamento no loop da mesma invocação
- "Colaborar" renomeado "Escalar" — função Team tier, war room

---

## 4. Story Map

**Estrutura:** Atividade (backbone) → Tarefas (nível 2) → Stories (nível 3).
**Campos:** Prioridade (P0/P1/P2) | Release (R0/R1/R2/R3) | Esforço (XS/S/M/L/XL) | Dependências.

---

### 4.1. Avaliar

**Outcome:** Decidir se Forja resolve o problema.

#### T1.1 — Navegar landing page · P0 | R1 | S

| Story  | Descrição                                              | Priority | Release | Esforço |
|--------|--------------------------------------------------------|----------|---------|---------|
| S1.1.1 | Ver proposta de valor ("Agentes que lembram")          | P0       | R1      | XS      |
| S1.1.2 | Ver pricing (Free/Pro/Team) com limites claros         | P0       | R1      | XS      |
| S1.1.3 | Ver demo em vídeo (2ª invocação = AHA)                 | P1       | R2      | M       |

#### T1.2 — Comparar com alternativas · P1 | R2 | M

| Story  | Descrição                                              | Priority | Release | Esforço |
|--------|--------------------------------------------------------|----------|---------|---------|
| S1.2.1 | Ver tabela comparativa (Forja vs Cursor vs Claude)     | P1       | R2      | S       |
| S1.2.2 | Ver use cases (code review, doc generation, refactor)  | P1       | R2      | S       |

#### T1.3 — Acessar signup · P0 | R1 | XS

| Story  | Descrição                                              | Priority | Release | Esforço |
|--------|--------------------------------------------------------|----------|---------|---------|
| S1.3.1 | CTA "Start Free" na landing → redirect signup page     | P0       | R1      | XS      |

---

### 4.2. Integrar

**Outcome:** Setup CLI + credenciais em <5min.

#### T2.1 — Criar conta (self-serve) · P0 | R1 | M · deps: supabase-auth, stripe

| Story  | Descrição                                              | Priority | Release | Esforço | Deps             |
|--------|--------------------------------------------------------|----------|---------|---------|------------------|
| S2.1.1 | Signup com email/senha (Supabase)                      | P0       | R1      | S       | supabase-auth    |
| S2.1.2 | Confirmar email (link Supabase)                        | P0       | R1      | S       | supabase-auth    |
| S2.1.3 | Escolher tier (Free/Pro/Team)                          | P0       | R1      | S       | —                |
| S2.1.4 | Checkout Stripe (Pro/Team)                             | P0       | R1      | M       | stripe-checkout  |
| S2.1.5 | Redirect para onboarding CLI após signup               | P0       | R1      | XS      | —                |

#### T2.2 — Instalar CLI · P0 | R1 | S · deps: cli-npm-package

| Story  | Descrição                                              | Priority | Release | Esforço |
|--------|--------------------------------------------------------|----------|---------|---------|
| S2.2.1 | `npx @olimpo/forja setup` ou `npm install -g @olimpo/forja` | P0 | R1 | XS |
| S2.2.2 | Verificar instalação (`forja --version`)               | P0       | R1      | XS      |
| S2.2.3 | Setup concluído em <3min (medido em teste de usuário)  | P0       | R1      | S       |

#### T2.3 — Autenticar CLI · P0 | R1 | M · deps: supabase-jwt, cli-auth-flow

| Story  | Descrição                                              | Priority | Release | Esforço | Deps                  |
|--------|--------------------------------------------------------|----------|---------|---------|-----------------------|
| S2.3.1 | `forja login` (device flow ou JWT paste)               | P0       | R1      | M       | supabase-device-flow  |
| S2.3.2 | Armazenar token local (~/.forja/credentials.json)      | P0       | R1      | S       | —                     |
| S2.3.3 | Refresh token automático                               | P0       | R1      | M       | supabase-refresh      |

#### T2.4 — Listar agentes disponíveis · P0 | R1 | S · deps: free-tier-enforcement

| Story  | Descrição                                              | Priority | Release | Esforço |
|--------|--------------------------------------------------------|----------|---------|---------|
| S2.4.1 | `forja agents list` (1 agente no Free tier)            | P0       | R1      | S       |
| S2.4.2 | Ver description/capabilities de cada agente            | P1       | R2      | S       |

---

### 4.3. Invocar

**Outcome:** Executar 1ª e 2ª invocação (AHA = memória visível).

#### T3.1 — Primeira invocação · P0 | R1 | M · deps: invoke_agent-rls-fix, billing-counter

| Story  | Descrição                                              | Priority | Release | Esforço | Deps                |
|--------|--------------------------------------------------------|----------|---------|---------|---------------------|
| S3.1.1 | `forja invoke <agent-id> --prompt "..."` (CLI)         | P0       | R1      | M       | rls-fix             |
| S3.1.2 | Orquestração local-first (localhost:8080 → EC2 fallback) | P0     | R0      | M       | ✅ já funciona      |
| S3.1.3 | Contador Free tier decrementado (10 → 9)               | P0       | R1      | M       | billing-service     |
| S3.1.4 | Streaming output (turno a turno)                       | P0       | R0      | M       | ✅ já funciona      |
| S3.1.5 | Salvar turn na memória Neon (session_id gerado)         | P0       | R0      | M       | ✅ já funciona      |
| S3.1.6 | CLI exibe session_id + invocações restantes ao final   | P0       | R1      | S       | billing-service     |

#### T3.2 — Segunda invocação (momento AHA) · P0 | R1 | M · deps: neon-memory, session-resume

| Story  | Descrição                                              | Priority | Release | Esforço | Deps            |
|--------|--------------------------------------------------------|----------|---------|---------|-----------------|
| S3.2.1 | `forja invoke <id> --session <session-id> --prompt "..."` | P0    | R1      | M       | session-resume  |
| S3.2.2 | Agente carrega contexto da sessão anterior (últimos 20 turns) | P0 | R0   | M       | ✅ já funciona  |
| S3.2.3 | Output retoma sem re-explicação (moat visível)         | P0       | R1      | M       | —               |
| S3.2.4 | CLI exibe confirmação "✅ Agent remembered context"    | P0       | R1      | S       | —               |

#### T3.3 — Ver histórico de sessões · P1 | R2 | S

| Story  | Descrição                                              | Priority | Release | Esforço |
|--------|--------------------------------------------------------|----------|---------|---------|
| S3.3.1 | `forja sessions list` (últimas 10 sessões)             | P1       | R2      | S       |
| S3.3.2 | `forja session show <session-id>` (turnos completos)   | P1       | R2      | S       |

---

### 4.4. Validar

**Outcome:** Verificar output + iterar (refinamento).

#### T4.1 — Revisar output · P0 | R0/R1

| Story  | Descrição                                              | Priority | Release | Esforço |
|--------|--------------------------------------------------------|----------|---------|---------|
| S4.1.1 | Output markdown formatado no terminal                  | P0       | R0      | S       | ✅ já funciona |
| S4.1.2 | Export para arquivo (`--output output.md`)             | P1       | R2      | S       |

#### T4.2 — Iterar na mesma sessão · P0 | R1 | M

| Story  | Descrição                                              | Priority | Release | Esforço |
|--------|--------------------------------------------------------|----------|---------|---------|
| S4.2.1 | Re-invocar com `--session <id>` + novo prompt          | P0       | R1      | M       |
| S4.2.2 | Agente refina output mantendo contexto                 | P0       | R1      | M       |

#### T4.3 — Aplicar output no código (diff/patch) · P0 | R2 | L

| Story  | Descrição                                              | Priority | Release | Esforço | Deps              |
|--------|--------------------------------------------------------|----------|---------|---------|-------------------|
| S4.3.1 | Agente retorna diff estruturado (formato unificado)    | P0       | R2      | L       | code-diff-parser  |
| S4.3.2 | `forja apply <session-id>` (aplica diff no working dir) | P0      | R2      | M       | git-integration   |
| S4.3.3 | Preview diff antes de aplicar (`--dry-run`)            | P0       | R2      | S       | —                 |
| S4.3.4 | Criar PR draft no GitHub (`--create-pr`)               | P1       | R3      | L       | github-api        |

---

### 4.5. Retomar

**Outcome:** Sessão seguinte — agente lembra contexto.

#### T5.1 — Listar sessões anteriores · P1 | R2 | S

| Story  | Descrição                                              | Priority | Release | Esforço |
|--------|--------------------------------------------------------|----------|---------|---------|
| S5.1.1 | `forja sessions list --agent <agent-id>`               | P1       | R2      | S       |
| S5.1.2 | Ver resumo (última mensagem + timestamp)               | P1       | R2      | S       |

#### T5.2 — Retomar sessão de dias atrás · P0 | R2 | M

| Story  | Descrição                                              | Priority | Release | Esforço |
|--------|--------------------------------------------------------|----------|---------|---------|
| S5.2.1 | `forja invoke <id> --session <session-id>` (sessão antiga) | P0   | R2      | M       |
| S5.2.2 | Agente carrega contexto completo (últimos 20 turns)    | P0       | R2      | M       |
| S5.2.3 | Continuidade visível no output (não reinicia do zero)  | P0       | R2      | M       |

#### T5.3 — Deletar sessão · P2 | R3 | S

| Story  | Descrição                                              | Priority | Release | Esforço |
|--------|--------------------------------------------------------|----------|---------|---------|
| S5.3.1 | `forja session delete <session-id>` (soft delete)      | P2       | R3      | S       |

---

### 4.6. Escalar

**Outcome:** War room multi-agente (Team tier).

#### T6.1 — Criar war room · P1 | R3 | L · deps: war-room-orchestration, team-tier

| Story  | Descrição                                              | Priority | Release | Esforço |
|--------|--------------------------------------------------------|----------|---------|---------|
| S6.1.1 | `forja war-room create --agents ayla,artemis,gaia`     | P1       | R3      | L       |
| S6.1.2 | Orquestração paralela (agentes colaboram na mesma task) | P1      | R3      | XL      |
| S6.1.3 | Session compartilhada (todos os agentes veem contexto) | P1       | R3      | L       |

#### T6.2 — Monitorar progresso · P1 | R3 | M

| Story  | Descrição                                              | Priority | Release | Esforço |
|--------|--------------------------------------------------------|----------|---------|---------|
| S6.2.1 | `forja war-room status <room-id>`                      | P1       | R3      | M       |
| S6.2.2 | Output de cada agente em tempo real                    | P1       | R3      | M       |

#### T6.3 — Consolidar output · P1 | R3 | M

| Story  | Descrição                                              | Priority | Release | Esforço |
|--------|--------------------------------------------------------|----------|---------|---------|
| S6.3.1 | Merge automático de outputs (sem conflito)             | P1       | R3      | L       |
| S6.3.2 | Resolução manual de conflitos (CLI prompt)             | P1       | R3      | M       |

---

## 5. Releases e Walking Skeleton

**Gate R1:** Nenhum release SaaS público antes de R1 completo.

---

### R0 — Estado Atual (v0.5.0, hoje)

**Objetivo:** Validar moat técnico do Forja com N=3 usuários internos. Os agentes (Ayla, Atena, Hermes etc.) rodam por baixo — o usuário interage com o Forja, não com o Olimpo diretamente.

**Walking Skeleton:**
1. Rafael cria tenant no Forja manualmente (script admin)
2. Usuário recebe credenciais do Forja
3. Instala CLI (`npm install -g @olimpo/forja`)
4. Autentica manualmente (JWT paste)
5. `forja invoke ayla --prompt "..."` → Ayla responde com contexto
6. `forja invoke ayla --session <id> --prompt "..."` → Ayla lembra (moat validado)

**Métricas atuais:**
- N=3 (1 externo + 2 internos), MRR R$297
- Onboarding manual ~30–45min (não aceitável para SaaS)
- Moat técnico validado ✅

**Lacunas conhecidas:** self-serve, Stripe, free tier, RLS, onboarding automatizado.

---

### R1 — Q3 Foundation (Q3 2026) ← GATE CRÍTICO

**Objetivo:** Self-serve SaaS público. Usuário chega na 2ª invocação (AHA) dentro do free tier.

**Walking Skeleton:**
1. Signup (email/senha) → email confirmado → escolhe Free tier
2. `npx @olimpo/forja setup` (<3min)
3. `forja login` (device flow)
4. `forja agents list` → 1 agente disponível (Free)
5. `forja invoke ayla --prompt "review my code"` → contador = 9 restantes
6. `forja invoke ayla --session <id> --prompt "continue"` → contador = 8, **AHA = agente lembra**

**Métricas de sucesso:**
- N=10 usuários free (signup self-serve)
- N=3 conversões Pro
- Tempo onboarding <5min (signup → 1ª invocação)
- Tempo CLI setup <3min
- Churn mês 1 <20%

**Dependências críticas:**
- Supabase auth (signup flow)
- Stripe integration (checkout Pro/Team)
- Billing service (contador invocações, hard limit)
- RLS fix (invoke_agent para role comum)
- CLI onboarding script (npx one-liner)

---

### R2 — Value Amplification (Q4 2026)

**Objetivo:** Output aplicável no código (diff/patch), histórico, export.

**Walking Skeleton:**
1. Invoke com output estruturado (diff/patch)
2. `forja apply <session-id> --dry-run` → preview diff
3. `forja apply <session-id>` → git apply automático
4. `forja sessions list --agent ayla` → histórico
5. `forja invoke ayla --session <id-antigo>` → retoma contexto de dias atrás

**Métricas de sucesso:**
- N=30 usuários free, N=10 conversões Pro
- Retention 30d >60%, NPS >40

**Dependências críticas:** code-diff-parser, git-integration, CLI history command.

---

### R3 — Multiplayer (Q1 2027)

**Objetivo:** War room multi-agente (Team tier) — orquestração paralela, session compartilhada.

**Walking Skeleton:**
1. `forja war-room create --agents ayla,artemis,gaia --task "refactor auth module"`
2. `forja war-room status <room-id>` → progresso em tempo real
3. Merge automático ou resolução manual de conflitos
4. `forja apply <room-id>` → diff consolidado

**Métricas de sucesso:**
- N=5 contas Team (R$119/assento), war rooms >20/mês, Retention 60d >70%

**Dependências críticas:** war-room-orchestration, multi-agent-merge, team-tier-enforcement.

---

## 6. Task Flows Detalhados

**P0:** Onboarding, Invocar com contexto, Aplicar output.
**P1:** War Room (estrutura definida, steps TBD em R3).

---

### 6.1. Task Flow — Onboarding (P0, R1)

**Objetivo:** Signup → 1ª invocação → 2ª invocação (AHA) em <5min.
**Trigger:** Usuário clica "Start Free" na landing page.
**Actors:** Usuário (Lucas), Forja (signup page), CLI, Supabase, Billing Service.

#### Steps

1. **Landing → Signup**
   - Usuário clica CTA "Start Free" → redirect `/signup`

2. **Criar Conta**
   - Preenche email/senha
   - Supabase cria user + envia email de confirmação
   - Usuário confirma email (link)

3. **Escolher Tier**
   - Página `/onboarding/tier`
   - Free (default) → próximo passo
   - Pro/Team → Stripe checkout → volta após pagamento

4. **Instalar CLI**
   - Página `/onboarding/cli` exibe:
     ```bash
     npx @olimpo/forja setup
     ```
   - Usuário executa no terminal → CLI instala deps, cria `~/.forja/config.json`

5. **Autenticar CLI**
   - `forja login` → device flow (browser) ou JWT paste
   - CLI salva token em `~/.forja/credentials.json`
   - CLI valida token (`GET /api/v1/auth/me`)

6. **Listar Agentes**
   - `forja agents list`
   - Backend retorna 1 agente (Free) ou 5/15 (Pro/Team)

7. **Primeira Invocação**
   - CLI instrui: `forja invoke ayla --prompt "Hello, review my code"`
   - Backend: decrementa contador (9 restantes) → orquestra → salva turn Neon
   - CLI exibe output streaming + `Session ID: ses_abc123 · Invocations: 9/10`

8. **Segunda Invocação (AHA)**
   - CLI instrui: `forja invoke ayla --session ses_abc123 --prompt "Continue from where we left off"`
   - Backend: carrega contexto (últimos 20 turns) → agente retoma sem re-explicação
   - CLI exibe: `✅ Onboarding complete! Your agent remembered the context. · Invocations: 8/10`

#### Decision Points

- **DP1 (Step 3):** Free → Step 4 · Pro/Team → Stripe → Step 4
- **DP2 (Step 5):** Auth OK → Step 6 · Falha → retry (3x) ou JWT paste
- **DP3 (Step 7):** Invoke OK → Step 8 · Falha → Error E2

#### Error States

| Código | Cenário                  | Recovery                              |
|--------|--------------------------|---------------------------------------|
| E1     | Email não confirmado     | Reenviar email (button "Resend")      |
| E2     | Invoke failed            | Retry 1x → erro + logs               |
| E3     | CLI auth failed          | Retry device flow ou JWT paste manual |
| E4     | Stripe cancelado         | Voltar para `/onboarding/tier`, escolher Free |

#### Acceptance Criteria

- [ ] Tempo total onboarding <5min (signup → 2ª invocação)
- [ ] Tempo CLI setup <3min (npx one-liner → autenticado)
- [ ] Contador decrementado corretamente (10 → 9 → 8)
- [ ] Session ID gerado na 1ª e reutilizado na 2ª invocação
- [ ] Agente carrega contexto na 2ª invocação (últimos 20 turns)
- [ ] Output streaming funciona (não espera invoke completo)
- [ ] Free tier enforced (bloqueio após 10 invocações)
- [ ] Error recovery funciona para E1, E2, E3

---

### 6.2. Task Flow — Invocar com Contexto Preservado (P0, R1)

**Objetivo:** Usuário invoca agente em sessão existente — agente retoma sem repetir contexto.
**Trigger:** `forja invoke <agent-id> --session <session-id> --prompt "..."`
**Actors:** Usuário, CLI, Backend (invoke service), Neon.

#### Steps

1. **Validação CLI**
   - Token autenticado? Session ID válido (UUID)? Quota disponível?

2. **Enviar Request**
   ```json
   POST /api/v1/agents/invoke
   { "agent_id": "ayla", "session_id": "ses_abc123", "prompt": "Refine the code review" }
   ```

3. **Backend — Carregar Contexto**
   ```sql
   SELECT turn, prompt, response FROM memory
   WHERE session_id = 'ses_abc123' ORDER BY turn DESC LIMIT 20;
   ```

4. **Orquestrar Invoke**
   - Local-first: tenta localhost:8080 → fallback EC2 se timeout >5s

5. **Agente Processa**
   - Lê context window (últimos 20 turns)
   - Gera resposta baseada em contexto anterior (não re-explica)

6. **Salvar Turn**
   ```sql
   INSERT INTO memory (session_id, turn, prompt, response) VALUES (...);
   ```
   - Decrementa contador Free tier

7. **CLI Exibe Output**
   - Streaming turno a turno
   - `Session ID: ses_abc123 · Turn: 3 · Invocations: 7/10`

#### Decision Points

- **DP1:** Session ID válido? → Sim: continua · Não: Error E1
- **DP2:** Quota disponível? → Sim: continua · Não: Error E2
- **DP3:** Orquestração OK? → Sim: continua · Não: Error E3

#### Error States

| Código | Cenário             | Recovery                                       |
|--------|---------------------|------------------------------------------------|
| E1     | Session not found   | `forja sessions list` → escolher sessão ativa  |
| E2     | Quota exceeded      | Link upgrade Pro (R$99/mês)                    |
| E3     | Invoke timeout      | Retry 1x → erro + logs                         |

#### Acceptance Criteria

- [ ] Agente carrega contexto da sessão (query Neon últimos 20 turns)
- [ ] Response baseada em contexto anterior (não re-explica, não repete)
- [ ] Turn salvo na memória Neon com turn incrementado
- [ ] Contador decrementado corretamente
- [ ] Free tier enforced (bloqueio após 10 invocações)
- [ ] Local-first funciona (localhost:8080 → EC2 fallback)
- [ ] Output streaming funciona
- [ ] Error recovery para E1, E2, E3

---

### 6.3. Task Flow — Aplicar Output no Código (P0, R2)

**Objetivo:** Usuário aplica diff/patch do agente no working directory — valor materializado.
**Trigger:** `forja apply <session-id>` após invoke com diff estruturado.
**Actors:** Usuário, CLI, Backend (session retrieval), Git (local).

#### Steps

1. **Validação CLI**
   - Token OK? Session ID válido? Working directory é git repo?

2. **Buscar Output**
   ```
   GET /api/v1/sessions/ses_abc123/output
   → { "session_id": "ses_abc123", "turn": 3, "response": "<markdown com diff>" }
   ```

3. **Parsear Diff**
   - CLI extrai blocos ` ```diff ` ou ` ```patch ` do response
   - Formato unificado (`--- a/file.ts`, `+++ b/file.ts`, `@@ -X,Y +A,B @@`)
   - Sem diff estruturado → Error E1

4. **Preview (--dry-run)**
   - CLI exibe diff colorizado (red = removido, green = adicionado)
   - Exibe arquivos afetados
   - Pergunta confirmação: `Apply this diff? [y/N]`

5. **Aplicar Diff**
   - Salva diff em `/tmp/forja_ses_abc123.patch`
   - `git apply /tmp/forja_ses_abc123.patch`

6. **Validar**
   - `git diff --name-only` → arquivos modificados conforme esperado?
   - Falha → Error E2

7. **Cleanup + Sucesso**
   - Remove `/tmp/forja_*.patch`
   - Exibe:
     ```
     ✅ Diff applied successfully.
     Files modified: src/auth.ts, src/utils.ts
     Next: git add . && git commit -m "Apply Forja suggestions from ses_abc123"
     ```

#### Decision Points

- **DP1 (Step 1):** Git repo? → Não: Error E3
- **DP2 (Step 3):** Diff encontrado? → Não: Error E1
- **DP3 (Step 4):** Usuário confirma? → Não: abort (cleanup)
- **DP4 (Step 5):** git apply OK? → Não: Error E2

#### Error States

| Código | Cenário           | Recovery                                            |
|--------|-------------------|-----------------------------------------------------|
| E1     | No diff found     | Sugerir ao usuário pedir "Please return a structured diff" |
| E2     | git apply failed  | Exibe erro do git + diff em /tmp para aplicação manual |
| E3     | Not a git repo    | Sugerir `git init` ou navegar para repo válido      |

#### Acceptance Criteria

- [ ] CLI extrai diff estruturado do output do agente
- [ ] Preview correto com `--dry-run` (diff colorizado)
- [ ] Confirmação do usuário antes de aplicar (`[y/N]`)
- [ ] `git apply` executado corretamente
- [ ] `git diff --name-only` valida modificações
- [ ] Cleanup do `/tmp/forja_*.patch`
- [ ] Mensagem de sucesso com next steps (git add + commit)
- [ ] Error recovery para E1, E2, E3

---

### 6.4. Task Flow — War Room Multi-Agente (P1, R3)

**Objetivo:** Orquestrar múltiplos agentes em sessão compartilhada — colaboração paralela.
**Trigger:** `forja war-room create --agents ayla,artemis,gaia --task "..."`
**Actors:** Usuário (Team tier), CLI, Backend (war room orchestration), Agentes.

**⚠️ Steps, Decision Points, Error States e ACs: TBD — implementação R3.**

```
Estrutura prevista:
1. Criar war room (--agents, --task, Team tier validation)
2. Backend: war_room_id + session compartilhada
3. Invokes paralelos (3 agentes, mesma task)
4. Backend: merge automático ou flag conflito
5. CLI: `forja war-room status <room-id>`
6. `forja apply <room-id>` (diff consolidado)
```

---

## 7. Dependências Técnicas e Bloqueios

| Dependência                       | Release | Tipo     | Owner   | Status       | Blocker Para                     |
|-----------------------------------|---------|----------|---------|--------------|----------------------------------|
| Supabase auth (signup flow)       | R1      | Backend  | Artemis | In Progress  | T2.1 Criar conta self-serve      |
| Stripe integration (checkout)     | R1      | Backend  | Artemis | Not Started  | T2.1.4 Checkout Pro/Team         |
| Billing service (contador inv)    | R1      | Backend  | Artemis | Not Started  | T3.1.3 Free tier enforcement     |
| RLS fix (invoke_agent role comum) | R1      | Backend  | Artemis | Not Started  | T3.1.1 Primeira invocação        |
| CLI onboarding script (npx)       | R1      | CLI      | Íris    | Not Started  | T2.2.1 Instalar CLI <5min        |
| Landing page v1                   | R1      | Frontend | Íris    | Not Started  | T1.1 Navegar landing page        |
| Code diff parser                  | R2      | Backend  | Artemis | Not Started  | T4.3.1 Agente retorna diff       |
| Git integration (apply diff)      | R2      | CLI      | Íris    | Not Started  | T4.3.2 forja apply               |
| CLI history command               | R2      | CLI      | Íris    | Not Started  | T5.1 Listar sessões              |
| War room orchestration            | R3      | Backend  | Artemis | Not Started  | T6.1 Criar war room              |
| Multi-agent merge                 | R3      | Backend  | Artemis | Not Started  | T6.3 Consolidar output           |
| Team tier enforcement             | R3      | Backend  | Artemis | Not Started  | T6.1 War room requer Team tier   |

**Bloqueios críticos para R1 (gate SaaS):**
1. **RLS fix** — invoke_agent hoje requer role admin. Sem fix, self-serve inútil.
2. **Billing service** — contador invocações não existe. Precisa table + triggers + API.
3. **Stripe checkout** — sem Stripe, sem conversão Pro/Team.
4. **CLI onboarding** — setup manual >30min inaceitável. Precisa npx one-liner + device flow.

---

## 8. Mockup Tracking

| Tela / Mockup             | Estado      | Task Flow Coberto              | Priority | Release |
|---------------------------|-------------|--------------------------------|----------|---------|
| Landing page              | Not Started | T1.1 (navegar landing)         | P0       | R1      |
| Signup page               | Not Started | T2.1 (criar conta self-serve)  | P0       | R1      |
| Onboarding / tier         | Not Started | T2.1.3 (escolher tier)         | P0       | R1      |
| Onboarding / CLI          | Not Started | T2.2 (instalar CLI)            | P0       | R1      |
| Stripe checkout           | Not Started | T2.1.4 (checkout Pro/Team)     | P0       | R1      |
| CLI output (terminal)     | ✅ Existe   | T3.1, T3.2 (invocar)           | P0       | R0      |
| CLI sessions list         | Not Started | T5.1 (listar sessões)          | P1       | R2      |
| CLI apply preview         | Not Started | T4.3.3 (preview diff)          | P0       | R2      |
| War room dashboard        | Not Started | T6.2 (monitorar progresso)     | P1       | R3      |

**Gap atual:** nenhuma tela web existe (landing, signup, onboarding). CLI output funciona mas sem onboarding automatizado.

---

## 9. Registro de Decisões do War Room

### War Room v1 (inicial) — 2026-06-17
Atena · Íris · Artemis · Gaia. Premissa incorreta (Olimpo como produto separado).

### War Room v2 — 2026-06-17 (CANÔNICO)
**Decisor:** Ayla · **Debate:** Atena (estratégia) · Íris (UX/produto) · Artemis (honesty audit) · Hermes (market intelligence)
**Correção central:** Olimpo = motor interno. O usuário nunca vê o Olimpo. Forja = único produto público.

| ID | Decisão                                                                                  | Rationale                                                                                    |
|----|------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------|
| D1 | **Posicionamento:** "Stop building agents. Start using the best one." — agentes prontos, zero config. Olimpo invisível. | Atena: diferencial vs LangChain/CrewAI (que exigem construção). Hermes: Blue Ocean — nenhum player tem personas psicométricas + memory-first + dev tooling juntos. Artemis auditou: **claim é aspiracional no R0, meta R1** — hoje exige role admin e onboarding 30min. |
| D2 | **Backbone:** Avaliar → Integrar → Invocar → Validar → Retomar → Escalar (6 atividades, mantido). "Integrar" = auth/CLI install, **não** configurar agente. Usuário não cria nem configura agente — ele escolhe qual usar e invoca. | Atena propôs reduzir para 4 (Descobrir/Invocar/Validar/Retomar), mas Artemis apontou que auth+CLI é real e incontornável. "Integrar" permanece com semântica corrigida. Íris: Ayla é o agente default — discovery de outros é expansão progressiva, não pré-condição. |
| D3 | **AHA composto** (dois momentos): AHA-1 = sinalização na 1ª invocação ("I'll remember this") + AHA-2 = evidência real na 2ª (agente retoma sem re-explicação). Onboarding deve forçar sequência 1→2 na mesma sessão. | Hermes: benchmark mostra que AHA dev-first (Vercel/Stripe/Cursor) acontece em <3min na 1ª ação. 2ª invocação é correto mas atrasado — precisa de teaser. Artemis: risco ALTO de churn antes da 2ª se 1ª invocação for genérica (sem contexto de projeto). |
| D4 | **Walking Skeleton P0 corrigido:** além de self-serve+Stripe+invoke público, adicionar **contexto de projeto** (repo link ou indexação express de README/package.json na entrada) como bloqueador P0 para R1. Sem contexto, 1ª invocação é genérica → churn antes do AHA. | Artemis (honesty audit): "funciona tecnicamente, mas o fluxo de valor não fecha". Resp genérica na 1ª = dev compara com ChatGPT e sai. Hermes: Cursor e GitHub Copilot já têm contexto do projeto — Forja precisa paridade mínima na 1ª call. |
| D5 | **Task Flows P0 mantidos:** (1) Onboarding, (2) Invocar com contexto preservado, (3) Aplicar output no código. Adicionar **"Debug em 30s"** como flow WOM para R2 (maior potencial viral dev BR — dor universal, screenshot fácil, resultado concreto). P1: War Room (R3). | Íris: onboarding domina, descoberta de agentes é expansão. Hermes: "Debug em 30s" é o flow com maior ROI para word-of-mouth (Cursor-style virality). Atena: "Aplicar diff" é P0 porque output não-aplicável = agente inútil. |
| D6 | Template: YAML front matter + índice duplo (produto/dev/QA). Documento canônico único. | Proposto pela Gaia (v1), confirmado na v2. |

### Dependências P0 adicionadas (D4)

| Dependência                          | Release | Status      | Blocker Para                              |
|--------------------------------------|---------|-------------|-------------------------------------------|
| Contexto de projeto (repo link)      | R1      | Not Started | 1ª invocação não genérica (gate AHA-1)    |
| Indexação express (README/pkg.json)  | R1      | Not Started | Ayla responde com contexto real na D1     |
| Sinalização de memória na 1ª resp    | R1      | Not Started | AHA-1 teaser ("I'll remember this")       |

### Flow WOM adicionado (D5)

**6.5. Task Flow — "Debug em 30s" (WOM, R2)**
> Dev cola stack trace → Ayla diagnostica (linha exata, causa raiz, fix) em <10s → resultado aplicável → shareável no Twitter.
> **Diferencial:** resposta contextualizada ao projeto (não "tente reiniciar o servidor").
> **Shareable artifact:** template de screenshot estilizado (stack trace + resposta Ayla).
> Steps, DPs, ACs: TBD em sprint R2.

**Próximos passos:**
- [ ] Artemis: criar issues R1 bloqueadores (RLS fix, billing, Stripe, CLI onboarding, repo link/indexação express)
- [ ] Íris: iniciar mockup landing page + signup (D1: "Ayla is ready. Start now." como hero)
- [ ] Hefesto: QA gate — confirmar ACs completos nos task flows R1
- [ ] Sprint R1: adicionar AHA-1 sinalização na resposta da Ayla ("I'll remember this session")

---

**Documento encerrado. Source of truth da camada de produto do Forja.**
**Próxima revisão:** 2026-07-01 ou quando primeiro bloqueio R1 for resolvido.
