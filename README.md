# JiuTrack Pro 🥋 — Versão Júnior (Backend-only)

> **Objetivo:** um projeto **backend** simples (em camadas) para praticar **TypeScript, Express, SQL, Docker, Autenticação JWT** e **métricas básicas** — com escopo pensado para **nível júnior**.

---

## 🧭 Meta do Projeto

Construir uma **API REST** simples e bem organizada para:

- 👤 **cadastrar/logar usuários** (Auth/JWT)
- 📝 **registrar treinos** (data, sessão, técnicas, RPE, duração, notas)
- 🧩 **catalogar técnicas** e vincular às sessões
- 🎯 **definir metas curtas** e acompanhar evolução (volume, RPE, variedade)
- 🥇 **comparar desempenho** com percentis e leaderboards por faixa/peso
- 🐳 Tudo rodando em **Docker** com **banco SQL**

---

## 🗺️ Roadmap em 4 Etapas (cada uma entrega algo “usável”)

### 1) Base (Auth + Saúde da API)
**Objetivo:** subir servidor Express + TypeScript, com lint, scripts e Docker.

**Endpoints**
- `POST /auth/register` → cria usuário
- `POST /auth/login` → retorna **JWT** (sem refresh no início)
- `GET /health` → `ok`
- `GET /me` → rota protegida com dados do usuário

**Banco (SQL)**
- `users(id, email unique, password_hash, display_name, belt ENUM('white','blue','purple','brown','black'), weight_kg NUMERIC(5,2), created_at)`

**Segurança**
- Hash de senha (**bcrypt**) e JWT curto (ex.: 15–30 min)
- Rate-limit básico em `/auth/*`

**Docker**
- `docker-compose.yml` com `api` + `db` (Postgres) + `db-admin` (Adminer/pgAdmin)

**✅ Critérios de aceite**
- Registrar → logar → acessar rota protegida `GET /me`
- Senha não aparece em logs; `401` sem token

---

### 2) Diário de Treinos (Sessões + Técnicas + RPE)
**Objetivo:** praticar **CRUD REST** e relacionamento sessão↔técnica; métricas simples de volume.

**Banco**
- `techniques(id, name, position, category CHECK IN ('guard','pass','sweep','takedown','submission','escape','control'), difficulty INT 1..5, created_at)`  
- `sessions(id, user_id, date, kind CHECK IN ('gi','nogi','drill','spar'), duration_min INT>0, rpe INT 1..10, notes, created_at)`  
- `session_techniques(session_id, technique_id, reps INT>=0, success_rate NUMERIC(5,2) NULL, PRIMARY KEY(session_id, technique_id))`  
- *(Opcional)* `injuries(id, user_id, date, body_part, severity INT 1..5, notes)`

**Endpoints**
- `GET /techniques?q=...` → busca por nome/posição
- `POST /techniques` → *(admin opcional no fim)*
- `GET /me/sessions?from=&to=`
- `POST /me/sessions`
- `PATCH /me/sessions/:id`
- `DELETE /me/sessions/:id`

**Regras**
- `rpe ∈ {1..10}`, `duration_min > 0`
- Técnicas referenciadas devem existir; `reps ≥ 0`
- Sessão pertence ao **próprio usuário** (validar `user_id` do token)

**✅ Critérios de aceite**
- Criar/editar/excluir sessão com ≥1 técnica
- Listar por intervalo e retornar **duração total** e **RPE médio**

---

### 3) Metas & Indicadores (Goals + Stats)
**Objetivo:** praticar **metas semanais** e **indicadores automáticos** com janela rolante.

**Banco**
- `goals(id, user_id, kind CHECK IN ('sessions_per_week','techniques_new_per_week','duration_per_week_min','prep_competition'), target INT, start_date, end_date, status CHECK IN ('active','done','cancelled') DEFAULT 'active', created_at)`  
- `weekly_stats(user_id, week_start, sessions INT, duration_min INT, avg_rpe NUMERIC(4,2), unique_techniques INT, PRIMARY KEY(user_id, week_start))`

**Endpoints**
- `GET /me/goals`
- `POST /me/goals`
- `PATCH /me/goals/:id` / `DELETE /me/goals/:id`
- `GET /me/stats?range=last4w|last12w|custom&from=&to=`

**Regras**
- Evitar sobrepor metas do mesmo `kind` no mesmo período
- `weekly_stats` calculadas por janela rolante (simplicidade no MVP)

**✅ Critérios de aceite**
- Criar/encerrar metas e ver **% de atingimento**
- `GET /me/stats` mostra **sessões/semana**, **duração**, **avg RPE**

---

### 4) Comparações & Leaderboards (Privacy-aware)
**Objetivo:** **comparação saudável** entre usuários com recortes simples e respeito à **privacidade**.

**Banco**
- `user_privacy(user_id PK, share_in_leaderboards BOOLEAN DEFAULT true, share_weight BOOLEAN DEFAULT false)`  
- `comparison_snapshots(user_id, period_start, period_end, sessions INT, duration_min INT, avg_rpe NUMERIC(4,2), belt, weight_bucket, created_at, PRIMARY KEY(user_id, period_start, period_end))`

**Endpoints**
- `GET /compare/percentile?metric=sessions|duration_min&from=&to=&belt=&weightBucket=` → retorna **percentil** do usuário vs grupo (p50, p75, p90)
- `GET /leaderboards?metric=sessions|duration_min&window=weekly|monthly&belt=&weightBucket=&limit=20` → retorna **top N** (anonimiza quando necessário)
- `GET /me/privacy` / `PUT /me/privacy`

**Regras simples**
- Somente usuários com `share_in_leaderboards=true` aparecem no ranking
- Filtros por `belt` e por `weight_bucket` (ex.: `-64kg`, `64–76kg`, `76–88kg`, `+88kg`)
- Se `share_weight=false`, **nunca** expor `weight_kg` bruto (usar só `weight_bucket`)

**✅ Critérios de aceite**
- Usuário vê seu **percentil** na janela escolhida
- Leaderboard semanal por faixa respeita **anonimização** e **opt-in**

---

## 🔚 Endpoints (resumo rápido)

### 🔐 Auth
- `POST /auth/register`
- `POST /auth/login`
- `GET /me` *(protegido)*

### 🧩 Técnicas
- `GET /techniques?q=term`
- `POST /techniques` *(admin opcional)*

### 📝 Sessões
- `GET /me/sessions?from=&to=`
- `POST /me/sessions`
- `PATCH /me/sessions/:id`
- `DELETE /me/sessions/:id`

### 🎯 Metas
- `GET /me/goals`
- `POST /me/goals`
- `PATCH /me/goals/:id`
- `DELETE /me/goals/:id`

### 📈 Estatísticas
- `GET /me/stats?range=last4w|last12w|custom&from=&to=`

### 🥇 Comparações
- `GET /compare/percentile?metric=&from=&to=&belt=&weightBucket=`
- `GET /leaderboards?metric=&window=weekly|monthly&belt=&weightBucket=&limit=20`

### 🔒 Privacidade
- `GET /me/privacy`
- `PUT /me/privacy`
