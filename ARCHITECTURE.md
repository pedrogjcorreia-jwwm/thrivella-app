# THRIVELLA — Documento de Arquitetura (vivo)

> **Para o assistente (Claude):** lê este ficheiro no início de cada sessão. Descreve a app de A a Z — backend, frontend, pipeline de dados, processo de deploy e as armadilhas já encontradas. Mantém-no atualizado quando algo mudar.

Última atualização: sessão de integração do CF (position_group `CF`, 2052 jogadores).

---

## 1. Visão geral

Thrivella é uma ferramenta de scouting de futebol. Para cada posição, pontua jogadores por métricas convertidas em *z-score dentro da liga* e soma-as num **Talent Index**.

**Dois repositórios, dois serviços:**

| Camada | Repo GitHub | Hosting | URL |
|---|---|---|---|
| Frontend | `pedrogjcorreia-jwwm/thrivella-app` (`index.html`, único ficheiro ~9 MB) | Vercel (auto-deploy do `main`) | https://thrivella-app.vercel.app/ |
| Backend/API | `pedrogjcorreia-jwwm/proscout-backend` (`server.js`, Node/Express) | Railway (auto-deploy do `main`) | https://thrivela.up.railway.app |
| Base de dados | — | PostgreSQL no Railway | — |

**Fluxo de dados:** Excel (motor de pontuação) → JSON → `POST /import/players` (ou `GET /import/cf`) → PostgreSQL → o frontend faz `fetch` da API e renderiza.

**Deploy = git commit.** Um commit no repo respetivo dispara o redeploy automático (Vercel ou Railway). O assistente comita via **GitHub API** (contents API para ficheiros pequenos; Git Data API — blob→tree→commit→ref — para o `index.html` grande).

---

## 2. Backend (`proscout-backend` / Railway)

`API_URL = https://thrivela.up.railway.app`

### Endpoints

| Método | Rota | O que faz |
|---|---|---|
| GET | `/players?limit=&offset=&sort=score&dir=desc` | Devolve **todos** os jogadores (paginado). O frontend filtra por posição no browser. **Não** filtra bem por `?position=` (o campo `position` é a string completa tipo "CF, AMF"). |
| POST | `/import/players` | Importa `{position, players[], weights?}`. Faz upsert por `(name, league)`. |
| GET | `/import/cf` | Lê o `CF_players.json` commitado no repo e importa. **Salta e reporta** jogadores que já existam noutra posição (`skipped_players`). Browser-only. |
| GET | `/cleanup` | **PERIGO:** `DELETE FROM players` — apaga TUDO. Não usar. |
| GET | `/cleanup/cf` | Apaga só `position_group='CF'` (preserva WIN e outros). |
| GET | `/fix/cf-takeover` | Remove as versões **não-CF** de jogadores cujo (nome,liga) está no `CF_players.json` (usado para mover versáteis para CF). |
| GET | `/diag/cf-conflicts` | Diagnóstico: `{cf_in_db, cf_in_file, conflict_count, conflicts[]}`. |
| GET | `/setup` | Corre `CREATE TABLE IF NOT EXISTS` + `ALTER TABLE ADD COLUMN IF NOT EXISTS` (migrações) + índices. Não-destrutivo. **Correr após adicionar colunas novas.** |

### Base de dados — tabela `players`

Chave única: **`(name, league)`** → *um jogador só pode existir numa posição*. (Ver Armadilha #1.)

Colunas principais: `id` (serial), `name, country, team, league, league_level, position, position_group`, `age, foot, size, weight, mkt_value, contract_end, matches, minutes`, métricas (ver abaixo), totais (`total_def, total_off, total_pass, total, total_physical`), `score`, `sf_rating`, `has_physical`.

**Colunas de métricas** (nomes na BD): `xg, xa, fouls, yellows, def_duels, def_duels_pct, adj_intercept, goals_np, shots90, goals_per_shot, crosses90, crosses_pct, dribles90, dribles_pct, box_touches, prog_carries, accels90, passes90, passes_pct, shot_assist, box_passes, box_passes_pct, recpt_depth, total_dist, hsr90, sprint_dist90, max_speed, sprints90, hsr_sprint_pct` + **específicas de CF** (adicionadas nesta sessão): `off_duels90, off_duels_pct, aerial_duels90, aerial_pct, header_goals`.

### Formato do JSON de import

`{ "position": "CF", "players": [ {…}, … ] }`. Cada jogador é um objeto com as **chaves = nomes das colunas** (com o sufixo `90`: `shots90`, `dribles90`, `accels90`, `passes90`, `sprint_dist90`, `sprints90`). O topo `position` define o `position_group`. Campos ausentes → `null` (o INSERT usa `|| null`). WIN usa crosses/prog_carries/hsr/box_passes; CF deixa esses a null e preenche os de CF.

**Talent Index** (valor do ranking) = `total_off + total_def + total_pass` (**só técnico**), calculado no **frontend** (não vem da BD). O **físico é um índice separado** (Physical Index = `total_physical`), não entra no Talent Index. O campo `score` da BD **tem de ser = técnico** (`total`), porque a API ordena por `score` e filtra `score IS NOT NULL`. Jogadores sem físico: `total_physical = 0`, `has_physical = false`.

---

## 3. Frontend (`thrivella-app` / Vercel)

`index.html` — ficheiro único. Carrega os jogadores da API em `loadPlayers()` (lotes de 500, `sort=score`), preenche `let PLAYERS=[]` e filtra por `position_group` no browser (`groupMap = {…, 'CF':'CF'}`).

### Ranking (comum a todas as posições)
O seletor de posição já inclui CF. As colunas do ranking são comuns. Ao **expandir os totais** (Total OFF/DEF/PASS/Físico), mostra as métricas — e **isto é por-posição**:
- **Desktop:** `function metricBox(groupKey, totalKey, …, posGroup)` → usa `posGroup==='CF' ? METRIC_GROUPS_CF : METRIC_GROUPS`. Chamado com `p.position_group`.
- **Mobile:** `function shSideOpenMetrics(playerName, groupKey)` → usa `p.position_group==='CF' ? SH_METRIC_GROUPS_CF : SH_METRIC_GROUPS`.

Configs: `METRIC_GROUPS`/`SH_METRIC_GROUPS` = WIN (crosses, prog carries, box passes). `METRIC_GROUPS_CF`/`SH_METRIC_GROUPS_CF` = CF (ver §5). Para novas posições, adicionar `*_CF`-style e o ramo de seleção.

### `LEAGUE_CONFIG`
49 ligas com nomes canónicos (ex.: `England1`, `Bulgary1`, `Moldavia1`, `Costa Rica1`, `Austria1/2`, `Australia1`). O `league` no JSON **tem de bater** com estes nomes.

---

## 4. Pipeline de dados (Excel → JSON)

**Excel** (ex.: `CF_PHY_v2_tHRIVELLA.xlsx`): uma folha por liga, nome de folha = **código** (`ENG1.`, `BUL1.`, …). Cabeçalho em 2 linhas, dados a partir da linha 3.

- **Métricas em bruto:** colunas 8–40 (0-indexed): `goals_np=22, xg=23, header_goals=24, shots=25, goals_per_shot=26, dribles=27, dribles_pct=28, off_duels=29, off_duels_pct=30, box_touches=31, accels=32, passes=33, passes_pct=34, shot_assist=35, recpt_depth=36, total_dist=37, sprint_dist=38, max_speed=39, sprints=40, xa=10, def_duels=15, def_duels_pct=16, aerial_duels=17, aerial_pct=18, fouls=19, yellows=20`.
- **Metadados:** `name=0, team=1, position=2, age=3, mkt_value=4, contract_end=5, matches=6, minutes=7, country=11, foot=12, size=13, weight=14`.
- **Totais — DOIS templates:**
  - **PHY** (com físico): `total_def=300, total_off=301, total_pass=302, total=303, total_physical=309`.
  - **NPHY** (sem físico): `total_def=251, total_off=252, total_pass=253, total=254`, sem físico.
  - Usar fallback por jogador: `total = col303 ?? col254`.
- **Filtro de intrusos:** manter só quem tem `position` a começar por "CF" (remove laterais/extremos infiltrados).
- **Mapa código→nome de liga:** `AST1→Austria1, AUS1→Australia1, AUS2→Austria2, BUL1→Bulgary1, MOL1→Moldavia1, CRI1→Costa Rica1, SLV1→Slovakia1, SLO1→Slovenia1`, etc. (⚠ AST=Áustria, AUS=Austrália — ver Armadilha #4.)
- **Dedup por `(name, league)`:** por defeito ficar com o registo de mais minutos; exceções decididas caso a caso pelo utilizador.

---

## 5. Modelo do CF (pesos + drill-down)

**Regra de ouro do estudo de pesos:** z-score dentro da liga; comparar contra a população, não entre transferidos; decidir na convergência de vários métodos; sem deep learning (amostra pequena) — usar d de Cohen + bootstrap, gradient boosting + SHAP.

**Perfil 9-puro:** a âncora é **Box Touches** (o 9 vive na área). O cluster atlético (dribles, acelerações, duelos ofensivos, sprint) infla **extremos** — manter baixo. Golos, xG, finalização e Box Touches são o núcleo robusto em todos os sinais.

**Drill-down do CF (menus do ranking)** — distribuído como o Excel calcula, com ajustes estéticos do utilizador:
- **OFF:** Golos s/pen, xG, Shots/90, Goals/Shot%, Box Touches, Dribles/90, Drible%, Off Duels/90, Off Duel%, Header Goals
- **DEF:** Def Duels/90, Def Duel%, Aerial Duels/90, Aerial%, Fouls/90, Yellows/90
- **PASS:** xA, Passes/90, Pass%, Recpt Depth, Shot Assist
- **FÍSICO:** Total Dist, Sprint Dist, Max Speed, Sprints/90, Accels/90

---

## 6. Processo de deploy / import (passo a passo)

1. Gerar o JSON a partir do Excel (§4). Commitar `<POS>_players.json` no `proscout-backend`.
2. `GET /setup` (se houver colunas novas).
3. `GET /cleanup/<pos>` (apagar a posição antiga).
4. `GET /import/<pos>` (ou `POST /import/players`).
5. `GET /diag/<pos>-conflicts` (verificar `conflict_count:0`).
6. Frontend: se houver métricas/menus novos, editar `index.html` (configs por-posição), **validar com `node --check`** o bloco `<script>`, commitar via Git Data API.

Os endpoints do Railway **não são alcançáveis** do ambiente do assistente (só domínios GitHub/PyPI/npm estão abertos) — o utilizador abre os links no browser. O commit de código é feito pelo assistente via GitHub API.

---

## 7. Armadilhas já encontradas (não repetir)

1. **Chave única `(name, league)` = um jogador, uma posição.** Um versátil (ex.: João Silva RWF/CF) não pode estar em WIN e CF ao mesmo tempo — o segundo import sobrescreve o primeiro. Regra do utilizador: fica onde foi subido primeiro; se estiver nos dois, **avisar e decidir**. O `/import/cf` já salta e reporta (`skipped_players`). ⚠ O `/import/players` (WIN) ainda **não** tem esta proteção — aplicar quando se mexer no WIN.
2. **Chaves do JSON com sufixo `90`.** O endpoint lê `p.shots90`, `p.dribles90`, etc. (não `p.shots`). Um regex `[a-zA-Z_]+` trunca o `90` — usar `[a-zA-Z_0-9]+`.
3. **Templates PHY vs NPHY** têm os totais em colunas diferentes (303 vs 254). Usar fallback.
4. **Códigos de liga:** `AST`=Áustria, `AUS`=Austrália. `AUS2` = Áustria 2 (renomeado de `AST2` por ordem alfabética). Confirmar sempre com `LEAGUE_CONFIG`.
5. **`/cleanup` apaga tudo** — usar sempre `/cleanup/<pos>`.
6. **Elite top-30** (para estudo de pesos) inclui **extremos** (Vinícius, Dembélé…) — contamina o sinal de "traços de elite" para xA/dribles. Para um modelo de 9-puro, usar sinais limpos de posição (top-5 por liga + transferências), não a elite.
7. **Duplicados no Excel:** mesmo (name, league) pode aparecer 2× (transferências a meio da época, registos parciais, ou homónimos). Dedup + decidir caso a caso.
8. **Reconstruir vs saber:** o assistente **não** guarda estado entre sessões. Este documento existe para mitigar isso — ler primeiro, atualizar sempre.

---

## 8. Estado por posição

| Posição | Estudo | Import | Frontend (menus) |
|---|---|---|---|
| **WIN** | concluído | em produção | em produção |
| **CF** | concluído (9-puro, Box Touches âncora) | **em produção (2052)** | **por-posição, desktop+mobile** |
| CB, DM, FB… | por fazer | — | — |
