# Dashboard de Indicadores Jurídicos

Dashboard estático (HTML/CSS/JS puro) que apresenta indicadores jurídicos por grupo econômico/cliente, alimentado por dados extraídos do sistema de gestão jurídica (EasyJur).

Este documento descreve **a estrutura do projeto e o procedimento de manutenção** (adicionar grupo, editar/excluir empresa, atualização periódica dos dados, links por cliente). Não contém dados de processos nem de clientes.

> **Como retomar do zero:** este README é auto-suficiente. Se a conversa/assistente se perder, ele contém tudo para atualizar os dados e republicar. Os scripts auxiliares citados ficam fora do repositório público (pasta de trabalho local).

---

## 1. Estrutura de arquivos

```
/
├── index.html          → todo o código (CSS + JS + HTML). NÃO contém dados.
├── dados/
│   ├── manifest.json   → ordem de exibição dos grupos: {"ordem": ["russi","wf",...]}
│   ├── russi.json      → dados de UM grupo (kpiGrupo + empresas + panorama)
│   ├── ...             → um .json por grupo (10 grupos)
└── c/                  → LINKS ISOLADOS POR CLIENTE (um por grupo)
    └── <codigo>/       → pasta com código secreto (ex.: c/70ec698f78/)
        ├── index.html  → cópia do principal + <meta name="cliente-grupo" content="<grupo>">
        └── <grupo>.json→ SÓ os dados daquele grupo (isolamento)
```

- O `index.html` **carrega** os JSONs de `dados/` no boot (via `fetch`), monta o objeto `GRUPOS` em memória e renderiza. O site **só funciona servido por HTTP** (GitHub Pages ou servidor local) — abrir do disco (`file://`) não carrega os dados.
- Hospedado no GitHub Pages: cada `git push` na `main` republica (1–2 min). Se o deploy travar, re-disparar: `gh api -X POST repos/<owner>/<repo>/pages/builds`.

### Formato de `dados/<grupo>.json`

```
{
  "nome": "Grupo X",
  "nomeHtml": "Grupo <em>X</em>",
  "kpiGrupo": { proc, procSub, horas, horasSub, areas, areasSub, trib, tribSub, ativos, ... },
  "empresas": {
    "_panorama": { "nome": "Panorama Geral do Grupo", "dataset": { ..., "_panorama": true } },
    "<chave_empresa>": { "nome": "...", "_idCliente": "<ID>", "dataset": {...} }
  },
  "nEmpresas": <n>
}
```

O `dataset` contém: `estoque`/`entradas`/`encerrados` (por ano), `area`, `areaTotal`, `instancia`, `polo`, `uf`, `tribunais`, `ativos`, `esforco` e `horas`.
- `esforco`: `prazano`, `audano`, `tarefano` (compromissos por ano), `atividades` (total por tipo), `comProducao` (usado só no cálculo de "Atos por processo"), `tramitacaoDias`, **`extrajudano`** (atividades extrajudiciais por ano).
- `horas`: `total`, `media`, `mediaAtivos`, `horasano`, `_estimativa`.
- Flags: `_dadosReais`, `_semProcessos` ("sem processos"), `_semDados` ("em breve"), `_panorama`.

O `_panorama` é a **soma consolidada** das empresas do grupo. Grupo de 1 empresa não tem `_panorama` (usa a própria empresa como topo).

**"Partes atendidas"** — no Panorama de qualquer grupo com mais de uma empresa, aparecem dois elementos: (1) KPI no topo com `nEmpresas`; (2) bloco `#partes-block` logo abaixo dos KPIs, antes de "01 — Volume", com um card contendo pills douradas — uma por empresa (`_panorama` e `_semDados` excluídas; `_semProcessos` incluído — é empresa atendida sem processo). Título do card é dinâmico: "N empresas atendidas". Aparece também no modo cliente (pasta `c/`). No dashboard de empresa individual (não é `_panorama`) o bloco fica oculto. IDs: `#k-partes-box`, `#k-partes`, `#partes-block`, `#partes-list`.

---

## 2. Recorte de exibição 2022→2026 (importante)

O dashboard mostra **apenas 2022 a 2026** nas séries/KPIs por ano (a base pré-2022 é pouco confiável). Isto **NÃO exclui processos** — só corta a série dos gráficos.

- Controlado por `const ANO_MIN = 2022;` no topo do `<script>` + função `cortaAnos(obj)`.
- Cortados: estoque, entradas×encerramentos, prazos/audiências/tarefas por ano, horas por ano, extrajudicial por ano, "Histórico processual".
- Recalculados sobre o recorte (para o KPI bater com o gráfico): crescimento do estoque, entradas/encerramentos, compromissos, horas totais/médias.
- **Preservados reais** (não cortados): totais de processos, ativos, áreas, tribunais.
- Para mudar o corte no futuro: alterar só o `ANO_MIN`.

---

## 2b. Animações (visual)

Os gráficos animam **ao rolar** até cada card (uma vez por visita, não repete): barras sobem, o donut (Distribuição por área) se desenha com o número central contando e a legenda em cascata, as pills de Tribunais entram em cascata, e o "Sobre este indicador" tem efeito de cortina. Vale igual no site-mãe e no link do cliente (o link do cliente ainda abre com a cortina "Silva & Silva").
- Mecânica: `IntersectionObserver` (`ativarObserver`) + `animarCard()` idempotente (flag `_anim`) + fallback de `scroll`. Disparo com leve delay (`animarCardDelay`, ~180ms) e threshold 0.35 (card ~35% visível) — para não animar cedo demais.
- Count-ups baseados em tempo real (`Date`) — sempre cravam o valor exato.
- Para ajustar o "quão cedo/tarde" anima: `threshold` do observer + `h*0.65` do fallback + o delay em `animarCardDelay`.

---

## 3. Manutenção comum

### Excluir um grupo
1. Apagar `dados/<grupo>.json`
2. Remover o grupo de `dados/manifest.json`
3. Apagar a pasta `c/<codigo>/` daquele grupo (link de cliente)
4. `git push`

### Editar / excluir uma empresa
1. Abrir `dados/<grupo>.json`, localizar em `empresas` e editar/apagar
2. Recalcular `_panorama` e `kpiGrupo` (soma das empresas restantes) para consistência
3. Regerar a pasta `c/` (ver seção 5) e `git push`

### Adicionar um grupo
1. Coletar os dados (seção 4)
2. Criar `dados/<grupo>.json` + adicionar em `manifest.json`
3. Definir um código secreto e gerar a pasta `c/<codigo>/` (seção 5)
4. `git push`

---

## 4. Coleta de dados (procedimento por grupo)

Fonte: API do EasyJur (via MCP). Sem delta incremental — reconsulta-se cada grupo.

### Regra de ouro (dados honestos)
- Volume/processos/evolução/áreas/geografia: **100% reais**, nunca estimados.
- Horas: reais do timesheet, marcadas com `^` e nota de "parcial".
- Nunca inventar. Campo inexistente → vazio/"—" ou `[VERIFICAR]`.

### 4.1 Processos
`list_processos` por cliente. ATENÇÃO: testar `cliente=<ID>` **e** `id_cliente=<ID>`, usar o que retorna `cliente_info.id == <ID>` (o outro às vezes traz empresa errada). Paginar (`page_size=100`). Agregar: `areaTotal`, `ativos`, `area`, `instancia`, `polo`, `uf`, `tribunais`, `estoque`/`entradas`/`encerrados` por ano.

### 4.2 Esforço (prazos/audiências/tarefas)
`list_agenda` com `mostrar_etapas=2` (só compromissos-mãe, senão infla com subetapas). Contar `PRAZO`/`AUDIENCIA`/`TAREFA` por ano via `data_interna_inicio/fim`.

### 4.3 Horas (por processo)
`list_timesheets` por `n_processo=<numero>`, para cada processo. Validar `cliente==<ID>` em cada linha; descartar `status=="S"`; **dedup por `id`**; `tempo_timesheet` HH:MM:SS → decimal; ignorar data > corte.

### 4.4 Extrajudicial (NOVO)
O time do extrajudicial registra tudo como **tarefa** marcada com o workflow **"Extrajudicial Geral" = `subtipo 18332`**.
- **Atividades por ano** (`extrajudano`) — ÚNICO card extrajudicial exibido: `list_agenda clientes=[ids do grupo], subtipo=18332, mostrar_etapas=2, data_interna_inicio/fim=<ANO>, page_size=1` → `meta.total` por ano. O filtro `clientes=[lista]` faz 1 chamada por grupo/ano.
- **Por tipo de trabalho: REMOVIDO** (a pedido). Era um `extrajudtipo` classificado por palavra-chave na descrição (contratos/notificações/etc.), mas a classificação era só aproximada — foi tirado. Se quiser retomar, a lógica era paginar as descrições do subtipo 18332 e classificar por sigla (NE=notificação, CV=contrato, CE=cobrança, ADR=distrato, MP=parecer).
- **Horas do extrajudicial: NÃO é viável.** O `list_timesheets` filtra por tipo de tarefa (subtipo) mas **não por cliente/grupo** — isolar por grupo exigiria processar 100 mil+ apontamentos do escritório à mão. Descartado (não confiável). Card de horas extrajudicial não existe.

### 4.5 Injeção
Gravar no `dataset` da empresa/panorama; recalcular `_panorama` e `kpiGrupo`.

### Cuidados aprendidos
- Empresas grandes (>~60 proc): coletar horas em lotes de ~50 (100+ trava).
- Overload da API (erro 500): esperar minutos e re-tentar só o lote que falhou.
- Empresa com 0h: normal quando as horas estão sob outro cadastro do grupo (card oculto).
- Cadastros parecidos: validar por `meta.total` e `cliente_info.id`.
- Nº de processo corrompido (notação científica): pular e `[VERIFICAR]`.

### Escopo (NÃO exibir)
Por decisão do escritório: Resultado/êxito, Risco, Financeiro/Honorários (baixa cobertura). Também **não** há: "Ranking de processos de maior valor", textos de "Leitura" interpretativa, "Extrajudicial por tipo de trabalho", KPI "Processos com produção", e horas do extrajudicial (inviável) — todos removidos a pedido.

---

## 5. Links isolados por cliente (pasta `c/`)

Cada grupo tem um link próprio e secreto para enviar ao cliente, mostrando **só o grupo dele**. Ex.: `.../c/70ec698f78/`.

- **Modo cliente:** o `index.html` da pasta tem `<meta name="cliente-grupo" content="<grupo>">`. O loader detecta a meta, carrega só aquele JSON, entra direto no Panorama (com cortina + animação), e esconde os botões de sair (Voltar/Grupos) e o logo clicável.
- **Botão "Gerar link"** (no Panorama de cada grupo, uso interno): copia o link secreto daquele grupo. O mapa grupo→código está em `LINKS_CLIENTE` no `index.html`.
- **Regenerar as pastas `c/`:** sempre que o `index.html` OU qualquer `dados/*.json` mudar, é preciso recriar a pasta `c/` (as pastas de cliente são cópias). Recriar: para cada grupo, copiar o `index.html` injetando a meta + copiar só o `<grupo>.json` para `c/<codigo>/`. (Feito por script auxiliar local que recria a pasta do zero.)

### Segurança (estado atual e pendência)
- Dentro do link do cliente, os JSONs dos outros grupos dão 404 (isolamento local OK).
- **PENDENTE:** o site-mãe (`/dados/` com todos os grupos) continua público — quem editar a URL subindo de pasta ainda alcança os outros. Para blindar 100%, só publicar as pastas `c/` (tirar o dashboard geral do ar público). Decisão em aberto.

---

## 6. Publicação e cache

Na atualização periódica:
1. Regravar os `dados/<grupo>.json`.
2. Mudar a data de corte (um lugar só) no `index.html`:
   ```js
   const DATA_CORTE = { curta:'30/06/2026', extenso:'Junho de 2026' };
   ```
3. **Trocar o cache-buster** no loader (força o navegador a pegar os dados novos, senão o usuário vê a versão antiga):
   ```js
   var _cb = '?v=20260722a';   // incrementar a cada atualização
   ```
4. Regerar as pastas `c/` (seção 5).
5. Publicar:
   ```
   git add index.html dados/ c/
   git commit -m "Atualizacao <mês/ano> — corte DD/MM/AAAA"
   git push origin main
   # aguardar o GitHub Pages (1–2 min); se travar: gh api -X POST repos/<owner>/<repo>/pages/builds
   ```

O site fica em `https://<owner>.github.io/<repo>/`.

> **Automação:** hoje é um processo assistido (reconsulta manual via API). Uma automação completa (backend puxando do EasyJur em cron) seria um projeto à parte.

---

## 7. Lições aprendidas (LEIA ANTES DE ATUALIZAR)

Erros que já cometi e que **NÃO devem se repetir na próxima atualização**. Estão listados por ordem de gravidade.

### 7.1 ⚠️ Coleta: use `id_cliente=<INT>`, NUNCA `cliente=<id>`

O parâmetro `cliente` do MCP `list_processos` faz **busca parcial** (LIKE) por id/nome, então retorna processos de OUTROS clientes cujo id/nome parcialmente casa. O parâmetro correto para filtro **exato** é `id_cliente=<INT>`.

**Sintomas do bug:** panorama do grupo (feito por busca larga) não bate com a soma das empresas individuais (feita por `cliente=<id>` que traz ruído). Ex.: ABC tinha 610 vs 581, WF/ZM/etc. também divergiam.

**Como validar sempre:**
```
Panorama total  ==  soma dos datasets das empresas
Panorama ativos ==  soma dos ativos das empresas
```
Se não bater, coleta está errada.

### 7.2 ⚠️ Atualize a DATA_CORTE quando os dados forem regenerados

A `const DATA_CORTE` no `index.html` marca a data até quando os dados foram coletados. Se você reconsulta o EasyJur em julho e esquece de atualizar, aparece "Junho de 2026" nos PDFs e no topo — cliente vê data velha.

Um só lugar pra atualizar:
```js
const DATA_CORTE = { curta:'23/07/2026', extenso:'Julho de 2026' };
```
Também troque o cache-buster (senão o navegador do cliente serve HTML/JSON velho da cache).

### 7.3 ⚠️ Panorama = União dedup dos processos das empresas

Nunca pegue apenas os totais do EasyJur para o panorama. **O panorama é a união deduplicada dos processos das empresas** (`id_processo` como chave). Se um processo aparece em duas empresas do grupo (ex.: Autor+Reu do mesmo grupo), conta 1× no panorama, 2× na soma naive.

O `consolidar.py` (em `scratchpad/`) faz isso automaticamente. Nunca ache que "o EasyJur me deu X ativos e Y encerrados, coloco no panorama".

### 7.4 ⚠️ Bugs de render que já apareceram

Estes bugs foram corrigidos, mas se voltarem a aparecer:

**KPIs travados em "0":** o `renderDashboard` seta `textContent='0'` como placeholder do count-up. Se `entrarDashboard` adiciona `.in` no card ANTES de `renderDashboard` popular `data-count`, o observer nunca dispara — KPI fica em "0". Fix atual: função `countUp` tem fallback quando `document.hidden` (aba background) e rede de segurança de 2,5s.

**PDF donut "0 processos":** `montarPrint` clona ANTES do count-up SVG terminar. Fix: forçar valor final no `_donutNum` imediatamente após `revealAll()` e reescrever `fill` dos `<text>` do SVG (senão fica branco no papel).

**PDF card "Partes atendidas" estourado:** 34 pills não cabiam em pill flex livre. Fix: `grid-template-columns: repeat(2, minmax(0, 1fr))`, `word-break: break-word`, `overflow-wrap`.

**Meta `cliente-grupo` apagada:** ao copiar `index.html` para as pastas `c/`, esquecemos de reinjetar a `<meta>`. Todas as 10 pastas perderam a meta e caíram no modo padrão (404 no manifest). Fix: script `bump_cb_p.py` (em `_recoleta_abc/`) sempre injeta a meta corretamente. USE SEMPRE ESSE SCRIPT ao propagar mudanças do index.html.

### 7.5 ⚠️ Métrica "Ritmo de novos processos" — decisão do Dr. Kim

O KPI `v-cresc` do bloco Volume mede: `(estoque_no_ano_base + total_entradas_no_periodo) ÷ estoque_no_ano_base`.

Ex. ABC: (188 + 475) ÷ 188 = 3,5×.

**Observação matemática:** essa fórmula conta os processos abertos no ano-base duas vezes (estão no estoque de 2022 E nas entradas de 2022). Decisão explícita do Dr. Kim priorizar a leitura de negócio ("atuamos em N processos") sobre pureza matemática. Ver conversa 23/07/2026.

Se algum sócio questionar, o histórico das 3 fórmulas testadas está no repo (git log) — estoque[fim]/estoque[base] = 1,7× → entradas[período]/entradas[base] = 6,3× → **(estoque+entradas)/estoque = 3,5× (atual)**.

### 7.6 ⚠️ Recorte 2023 nos blocos Esforço e Horas

O timesheet só foi implantado a partir de 2023. Dados de 2022 ficam sub-registrados e distorcem comparação ano-a-ano nos blocos "03 — Esforço" e "04 — Horas". Solução: `const ANO_MIN_ESFORCO = 2023` + `cortaAnosEsforco()` — usado só nesses dois blocos. Volume e Complexidade seguem `ANO_MIN=2022`.

### 7.7 ⚠️ Complexidade só com ativos

As distribuições do bloco 02 (área, UF, instância, polo, tribunais, ranking) refletem **só os processos ATIVOS** (status 1), não o histórico total. Sinalizado pelo flag `dataset._complexidadeSoAtivos = true`.

Faz sentido de negócio: cliente quer saber onde está o esforço ATUAL, não o histórico completo. Regra vale para os 10 grupos.

### 7.8 ⚠️ Nomenclatura "Reuniões e Diligências"

Onde antes se lia "Tarefas" no bloco Esforço, agora se lê **"Reuniões e Diligências"** (título do card, sub-texto do KPI de compromissos, texto das Considerações §3). Mudança pedida pelo escritório para refletir melhor o que essas atividades são na prática.

### 7.9 ⚠️ Página "Atividades extrajudiciais por ano" NÃO sai no PDF do ABC

Só no PDF do ABC. Nos outros 9 grupos, continua saindo. Controle no `montarPrint`:
```js
if(visivel('extrajudano') && GRUPO_ATUAL !== 'abc') sheet([cloneCardOf('extrajudano')], ...);
```

### 7.10 ⚠️ `_formatRatio` sempre com 1 casa decimal

A função arredondava para inteiro quando o valor era ≥ 2 (2,3 virava "~2×"). Agora sempre mostra 1 casa: 2,3 fica "2,3", 6,3 fica "6,3".

### 7.11 ⚠️ Consolidar.py preserva `nome`, `nomeHtml`, `nEmpresas` e empresas com 0 procs

Bug detectado 23/07/2026 (pós-refactor): 4 grupos (ABC, WF, ZM, Concept) apareceram como "undefined" no card do site-mãe. Causas:

- **`consolidar.py` não estava preservando `nome` e `nomeHtml`** do topo do JSON (só preservava o `nome` dentro das empresas). Cards do site-mãe usam `data.nome` como título → aparecia "undefined".
- **Também não preservava `nEmpresas`** — o KPI "N partes atendidas" mostrava "0".
- **Descartava empresas com 0 processos** — 3 empresas do ABC (Gralha Azul, AG&F, Triunfante) sumiram do JSON. Deveriam permanecer com flag `_semProcessos: true`.

**Fix aplicado no `consolidar.py`:**
- Preserva `nome`, `nomeHtml` do JSON antigo (topo)
- Grava `nEmpresas` = número de empresas reais (exclui `_panorama`)
- Mantém empresas sem processos com `dataset._semProcessos = true` (para o card exibir "em atualização" e ela continuar contando em "Partes atendidas")

**Se o bug voltar:** verificar em `dados/<grupo>.json` que `nome`, `nomeHtml` e `nEmpresas` existem no topo do arquivo.

---

## 8. Fluxo de atualização mensal (checklist)

Sempre nesta ordem:

1. **Coletar** processos via MCP com `id_cliente=<INT>` por empresa (nunca `cliente`).
   Para todos os 10 grupos: `abc`, `russi`, `wf`, `zm`, `cn`, `concept`, `profor`, `bhm`, `integra`, `porto`.

2. **Consolidar** com `scratchpad/consolidar.py <grupo> <codigo>`:
   - Recalcula panorama como união dedup
   - Recalcula cada empresa individual
   - Aplica Complexidade só-ativos automaticamente
   - Preserva `esforco` e `horas` do JSON anterior (se houver — recoleta desses é passo separado)

3. **Verificar** que bate: script imprime `Bate? True/True/True` para cada grupo.

4. **Atualizar** `DATA_CORTE` no `index.html`:
   ```js
   const DATA_CORTE = { curta:'DD/MM/YYYY', extenso:'Mês de YYYY' };
   ```

5. **Bump cache-buster:**
   ```js
   var _cb = '?v=YYYYMMDDa';
   ```

6. **Propagar** para as 10 pastas `c/` (script `_recoleta_abc/bump_cb_p.py`).

7. **Publicar:**
   ```bash
   cd _para_github
   git add -A
   git commit -m "Atualização mensal — corte DD/MM/YYYY"
   git push
   ```

---

## 9. Changelog

### 2026-07-23 (grande refactor + auditoria)
- **Panorama = soma das empresas** — 10/10 grupos com `id_cliente=<INT>` (filtro exato). Antes, filtro parcial trazia processos de terceiros.
- **DATA_CORTE:** 30/06/2026 → 23/07/2026.
- **KPI Ritmo** com fórmula do Dr. Kim: `(estoque_base + entradas) / estoque_base`. ABC = 3,5×.
- **Blocos Esforço e Horas** com recorte 2023 (`ANO_MIN_ESFORCO`).
- **Complexidade só-ativos** aplicada nos 10 grupos.
- **Nomenclatura** "Tarefas" → "Reuniões e Diligências".
- **PDF fixes:** donut Complexidade (número navy, não some no papel); "Partes atendidas" em grid 2 colunas com pills compactas; página "Atividades extrajudiciais" removida só do PDF do ABC.
- **`_formatRatio`** com 1 casa decimal (2,3 não vira ~2×).
- **34 empresas individuais do ABC populadas** com dataset completo (Volume + Complexidade só-ativos).
- **Bug KPIs em "0"** corrigido no `countUp` (fallback aba oculta + rede de segurança 2,5s).
- **CSVs de auditoria** gerados: `auditoria_ABC_ativos_COMPLETO_*.csv`, `auditoria_ABC_TAREFAS_*.csv`, `ABC_empresas_contagem.csv`, `ABC_processos_DIFERENCA.csv`.

### 2026-07-22
- **Novo bloco "Partes atendidas" no Panorama do grupo.** Aparece só quando `DATA.dataset._panorama` e há >1 empresa real (exclui `_panorama`; exclui `_semDados`; inclui `_semProcessos`).
  - KPI `k-partes` no grid do topo (6º card) com o número de empresas.
  - Seção nova `#partes-block` entre o grid de KPIs e "01 — Volume", com card `.pills` (pill dourada por empresa, animação `pill-anim` em cascata).
  - Título do card é dinâmico: "N empresas atendidas".
  - Vale igual no modo cliente (pasta `c/`). No dashboard de empresa individual o bloco fica oculto.
- Cache-buster: `?v=20260722a`.
