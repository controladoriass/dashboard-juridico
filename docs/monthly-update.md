# ATUALIZAÇÃO MENSAL DO DASHBOARD — PLANO DE AÇÃO

**Última atualização deste plano:** 2026-07-24
**Estado atual dos dados:** parcial (Volume + Complexidade OK; Esforço + Horas desatualizados)
**Data-corte alvo próxima atualização:** fechamento de julho/2026 (aguardando ordem do Rangel)

---

## 1. Estado atual (o que já está certo e o que precisa refazer)

Nesta sessão (23/07/2026) foram corrigidos os **BLOCOS 01 - Volume** e **02 - Complexidade** dos 10 grupos. Foram recoletados via `list_processos` com **`id_cliente=<INT>`** (filtro exato — o filtro antigo `cliente=<id>` fazia LIKE parcial e trazia processos de terceiros).

Ainda desatualizados nos 10 grupos:
- **Bloco 03 - Esforço operacional** (prazos, audiências, reuniões e diligências, atividades extrajudiciais)
- **Bloco 04 - Horas dedicadas** (`k-horas`, `h-total`, `h-media`, `h-mediaativos`, `horasano`, tramitação média)

Estes vieram da coleta de junho/2025 e usam filtro parcial (`cliente=<id>`) — podem ter ruído.

---

## 2. Grupos que precisam ser recoletados

**TODOS os 10 grupos** — Volume/Complexidade (para pegar entradas de julho) + Esforço/Horas (recoleta completa).

| Grupo | Código pasta c/ | IDs empresas (para `id_cliente`) |
|---|---|---|
| abc | 17430a49f0 | 34 empresas — ver `dados/abc.json` |
| russi | 4eccda8c78 | 17 empresas — ver `dados/russi.json` |
| wf | 635e9c8ddb | 9 empresas — ver `dados/wf.json` |
| zm | cb54479644 | 7 empresas — ver `dados/zm.json` |
| cn | 70ec698f78 | 5 empresas |
| concept | 25864dde3a | 3 empresas |
| profor | 4c05b109cc | 7 empresas |
| bhm | 0ff3055e50 | 4 empresas |
| integra | 1a33dbec70 | 1 empresa |
| porto | 47b98aa046 | 5 empresas |

Os IDs saem do próprio JSON de cada grupo (chave `_idCliente` de cada empresa, exceto `_panorama`).

---

## 3. Ordem de execução (checklist definitivo)

### Fase 1 — Coleta processos (JÁ AUTOMATIZADO)
Para cada grupo, para cada `_idCliente`:
```
list_processos(id_cliente=<INT>, page_size=100)  # NÃO usar cliente=<id> — filtro parcial!
```
Paginar até `meta.total_pages`. Salvar cada página em `scratchpad/coleta_<grupo>/emp_<id>_p<n>.json` com os campos SLIM (apenas 11 campos essenciais; NÃO perder `instancia_label`, `status`, `data_encerramento`, `origem_info`, `contrario_info`).

### Fase 2 — Coleta Esforço (NOVO — PRECISA FAZER)
Para cada grupo, para cada `_idCliente`, para cada ano 2022..2026:
```
list_agenda(clientes=[id], tipo=PRAZO,       data_interna_inicio=<ano>-01-01, data_interna_fim=<ano>-12-31, mostrar_etapas=2, page_size=1)
list_agenda(clientes=[id], tipo=AUDIENCIA,   ...)
list_agenda(clientes=[id], tipo=TAREFA,      ...)  # TAREFA = "Reuniões e Diligências" no dashboard
list_agenda(clientes=[id], subtipo=18332,    ...)  # Extrajudicial Geral (extrajudano)
```
Só o `meta.total` importa (contagem por ano). NÃO iterar processo por processo — puxa direto por cliente/tipo/ano.

**Campos alimentados:**
- `esforco.prazano` = `{ano: total_prazos}`
- `esforco.audano` = `{ano: total_audiencias}`
- `esforco.tarefano` = `{ano: total_tarefas}` (aparece no dashboard como "Reuniões e Diligências")
- `esforco.atividades` = `{"Prazos": soma, "Audiências": soma, "Reuniões e Diligências": soma}`
- `esforco.extrajudano` = `{ano: total_subtipo_18332}`
- `esforco.tramitacaoDias` = média de dias corridos dos ativos (calcular localmente com `data_cadastro`)
- `esforco.comProducao` = número de processos únicos que tiveram ≥1 compromisso (usado no cálculo de "Atos por processo")

### Fase 3 — Coleta Horas (NOVO — PRECISA FAZER)
Para cada `_idCliente`:
```
list_timesheets(cliente=<id>, page_size=100)
```
Iterar até esgotar.

**Cuidados críticos** (aprendizados de sessões anteriores):
- Validar em cada linha: `p['cliente'] == <id>` (LIKE do endpoint pode trazer ruído — descartar linhas que não batem).
- Descartar `status == "S"` (cancelados).
- Dedup por `id` do timesheet.
- Converter `tempo_timesheet` HH:MM:SS → decimal.
- Ignorar linhas com data > `DATA_CORTE.curta`.
- Marcar `horas._estimativa = true` (o dashboard mostra `^` na frente do total).

**Campos alimentados:**
- `horas.total` = soma total (decimal)
- `horas.horasano` = `{ano: horas_no_ano}`
- `horas.media` = `horas.total / kpi.proc`
- `horas.mediaAtivos` = `horas.total / ativos`
- `horas._estimativa = true`

### Fase 4 — Consolidação
Rodar o script `scratchpad/consolidar.py <grupo> <codigo>` que:
- Recalcula panorama como UNIÃO DEDUP dos processos (não soma naive)
- Preserva `nome`, `nomeHtml`, `nEmpresas` do JSON antigo
- Preserva empresas com 0 processos (flag `_semProcessos: true` — NÃO descarta)
- Complexidade só-ativos (status=1) com flag `_complexidadeSoAtivos=true`
- Campos em formato DICT (não lista), com chave `nome:qtd`
- `encerrados` como série por ano (NÃO usar nome `encerrados_por_ano` — o JS lê `encerrados`)
- Sub-textos com TOP 3 excluindo "Não informado"/"N/D"/"N/I"
- Tribunais em SIGLA (TJSC, TRT12, TRF4) — não nome completo
- kpi COMPLETO: `{proc, ativos, areas, trib, procSub, areasSub, tribSub}`

### Fase 5 — Auditoria automática (OBRIGATÓRIO — não pular)
Script `scratchpad/auditar_grupos.py` deve validar:
- ✅ `panorama.total == soma(empresa.total)` para todos os 10 grupos
- ✅ `dataset.kpi` tem todos os 7 campos (proc, ativos, areas, trib, procSub, areasSub, tribSub)
- ✅ `area`, `uf`, `instancia`, `polo`, `tribunais` são **DICT** (não lista)
- ✅ `encerrados` é dict-por-ano (não número)
- ✅ Instância NÃO tem >50% "Não informado" (indica bug no slim.py perdendo campo)
- ✅ `_panorama: true` no dataset do panorama
- ✅ `_complexidadeSoAtivos: true`
- ✅ `nome`, `nomeHtml`, `nEmpresas` no topo do JSON
- ✅ `horas._estimativa: true` se `k-horas` deve ter o `^`

**Só publicar se auditoria passa nos 10 grupos.**

### Fase 6 — Ajustes globais
- Atualizar `const DATA_CORTE` em `index.html` para nova data
- Incrementar `var _cb = '?v=YYYYMMDDx'`
- Ajustar `e-tramit-sub` "ativos contam até DD/MM"
- Rodar `_recoleta_abc/bump_cb_p.py` para propagar em todas as pastas `c/`

### Fase 7 — Publicar
```bash
cd _para_github
git add -A
git commit -m "Atualização mensal <mes>/<ano> — corte DD/MM"
git push origin main
```

---

## 4. Erros que NÃO devem se repetir

Documentados no README seção 7 (10 lições aprendidas). Os 5 mais graves:

1. **Nunca `cliente=<id>`, sempre `id_cliente=<INT>`** — filtro parcial pollui dados
2. **Nunca descartar empresas com 0 processos** — mantém com `_semProcessos: true`
3. **Sempre preservar `nome`, `nomeHtml`, `nEmpresas`** no topo do JSON
4. **Nunca perder campos no slim.py** — instancia_label sumiu em 619 procs numa sessão; sempre validar % com dados
5. **`encerrados` (série anual) vs `encerradosTotal` (número)** — JS lê o primeiro, script grava com nome certo

---

## 5. Quando o Rangel me chamar de novo

Ele vai dizer algo como "vamos rodar a atualização mensal do dashboard" ou "atualiza os dados de julho".

**Passos que eu (Claude) devo fazer, nessa ordem:**

1. **Ler `ATUALIZAR_MENSAL.md` (este arquivo)**
2. Ler `README.md` seção 7 (lições aprendidas) e seção 8 (checklist)
3. Ler minha memória `dashboard-format-json-e-kpis.md`
4. **Rodar `scratchpad/atualizar_mensal.py`** — automatiza Fases 1-5
5. Após auditoria passar, ajustar DATA_CORTE + cache-buster
6. Publicar
7. Validar visualmente no browser um grupo por amostra (ABC + Russi)

Não posso pular a auditoria da Fase 5. Foi a auditoria que ficou faltando nesta sessão e causou vários bugs (encerrados=0, subs em lista completa, instancia_label perdida).

---

## 6. Arquivos importantes

| Arquivo | Função |
|---|---|
| `scratchpad/consolidar.py` | Consolida coleta de UM grupo → JSON final |
| `scratchpad/atualizar_mensal.py` | Orquestra recoleta dos 10 grupos (NOVO, criar) |
| `scratchpad/auditar_grupos.py` | Valida os 10 JSONs (NOVO, criar) |
| `scratchpad/slim.py` | Reduz retorno MCP para 11 campos essenciais |
| `_recoleta_abc/bump_cb_p.py` | Propaga index.html + cache-buster para as 10 pastas c/ |
| `_para_github/README.md` | Manual completo (seções 7 e 8 são obrigatórias) |

---

## 7. Números atuais (para conferência após próxima atualização)

Estado em 23/07/2026 (base para saber se as coisas cresceram/mudaram):

| Grupo | Total | Ativos | Áreas | Tribunais |
|---|---:|---:|---:|---:|
| ABC | 641 | 328 | 7 | 19 |
| Russi | 1063 | 637 | 10 | 19 |
| WF | 200 | 65 | 8 | 6 |
| ZM | 193 | 114 | 9 | 20 |
| CN | 235 | 110 | 6 | 11 |
| Concept | 156 | 49 | 7 | 8 |
| Profor | 146 | 82 | 6 | 8 |
| BHM | 88 | 73 | 6 | 10 |
| Íntegra | 18 | 16 | 3 | 4 |
| Porto | 57 | 13 | 3 | 5 |

Cache-buster atual: `?v=20260723ao`
Commit atual: `c0e152c`
