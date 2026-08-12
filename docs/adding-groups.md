# COMO ADICIONAR UMA NOVA EMPRESA (ou grupo) AO DASHBOARD

Guia prático, testado na inclusão da **ZZM Fomento Mercantil** no grupo ZM (24/07/2026).
Serve para: (a) adicionar 1 empresa a um grupo que já existe; (b) criar um grupo novo do zero.

---

## Conceitos que você precisa saber antes

- Cada grupo é um arquivo `dados/<grupo>.json`, replicado em 4 lugares:
  `dados/`, `_para_github/dados/`, `c/<codigo>/`, `_para_github/c/<codigo>/`.
- Dentro de cada JSON: `empresas._panorama` (a soma do grupo) + uma chave por empresa individual.
- O `_panorama.dataset` alimenta os cards do grupo; o `kpiGrupo` (topo do JSON) alimenta o card do site-mãe.
- **Regra de ouro:** sempre coletar com `id_cliente=<INT>` (processos) ou `clientes=[id]` (agenda).
  NUNCA `cliente=<id>` em processos (faz busca parcial e polui). Timesheet é exceção — ver abaixo.

---

## CASO A — Adicionar 1 empresa a um grupo existente

Foi o que fiz com a ZZM. Passo a passo:

### 1. Confirmar que a empresa é do grupo
- `get_pessoa(pessoa_id=<id>)` → confere nome, CNPJ, se o nome tem o sufixo do grupo ("- ZM", "- ABC").
- Rangel valida se entra ou não (sufixo no nome é forte indício, mas confirmar).

### 2. Coletar os 3 blocos SÓ dessa empresa
**Processos** (Volume + Complexidade):
```
list_processos(id_cliente=<id>, page_size=100, page=1..N)
```
Extrair 11 campos: id_processo, status (1=ativo; 3/4/5/7=encerrado), data_cadastro,
data_encerramento, vinculo, area_info{nome}, uf, instancia_label, origem_info{nome} (só sigla antes de ' - '), contrario_info{nome}.

**Esforço** (agenda) — para cada ano 2021..2026 (2026 até a data-corte):
```
list_agenda(clientes=[id], tipo="PRAZO",     data_interna_inicio="<ano>-01-01", data_interna_fim="<ano>-12-31", mostrar_etapas="2", page_size=1)
list_agenda(clientes=[id], tipo="AUDIENCIA", ...)
list_agenda(clientes=[id], tipo="TAREFA",    ...)   # = "Reuniões e Diligências" no dash
list_agenda(clientes=[id], subtipo=18332,    ...)   # extrajudicial (sem tipo/mostrar_etapas)
```
Só `meta.total` importa. Validar que `data[0].cliente == id` (senão o endpoint trocou a resposta — refazer).

**Horas** (timesheet):
```
list_timesheets(pesquisa_nome="<nome oficial via get_pessoa>", tipo_pesquisa=1, page_size=100, page=1..N)
```
Filtrar: manter só `cliente==id`; descartar `status=="S"`; descartar `data_timesheet > corte`; dedup por `id`.
Converter `tempo_timesheet` HH:MM:SS → decimal.

### 3. Integrar ao grupo (script `integra_zzm_no_zm.py` é o modelo)
- Juntar os processos brutos das empresas antigas do grupo (ficam em `_recoleta_abc/noturna_2407/coleta_<grupo>/`) + os novos.
- **Checar overlap de id_processo** (dedup — empresas do mesmo grupo às vezes dividem processo).
- Recalcular o `_panorama`: total, ativos, encerrados (série por ano), entradas (série por ano),
  Complexidade SÓ-ATIVOS (area/uf/instancia/polo/tribunais em **DICT** `{nome:qtd}`, tribunais em SIGLA).
- Somar Esforço (prazano/audano/tarefano/extrajudano) ao que o panorama já tinha.
- Somar Horas ao total do grupo; marcar `horas._estimativa=true`.
- Recalcular `kpi` (proc, ativos, areas, trib, procSub, areasSub, tribSub — top 3 sem "Nao informado").
- Recalcular `kpiGrupo` no topo (mesmos números, alimenta o site-mãe).
- Adicionar a empresa como chave nova em `empresas` (nome, _idCliente, dataset completo).
- **Incrementar `nEmpresas`** no topo (7→8).
- Empresa com 0 processos: manter mesmo assim (não descartar).

### 3.5. GERAR AUDITORIA (obrigatório, novo desde 30/07/2026)
Antes de publicar, no consolidador chamar:
```python
from auditoria_processos._template_auditoria import gerar_auditoria
gerar_auditoria(
    slug_grupo=SLUG, nome_grupo=NOME, empresas=EMPRESAS_LIST,
    corte=CORTE, fator=FATOR, recorte_area=ID_AREA_OU_NONE,
    kpis_publicados={...}, fonte_dados={'contratos': [...]},
    processos_por_empresa={id_cli: [processos_slim]},
    timesheet_por_empresa={id_cli: [linhas_ts_filtradas]},
    agenda_por_empresa={id_cli: [compromissos]},
)
```
Reutiliza os dados JÁ coletados (não chama MCP de novo). Gera `Dashboard/auditoria_processos/<slug>/`
com CSV+JSON de processos/timesheet/agenda por empresa. Ver [[dashboard-auditoria-processos]].

### 4. Auditar + publicar
```
python _recoleta_abc/scripts_atualizacao_mensal/auditar_grupos.py   # 0 erros obrigatório
```
Bump `_cb` no index.html → `python _recoleta_abc/bump_cb_p.py` → commit + push em `_para_github`.

---

## CASO B — Criar um grupo NOVO do zero

1. Definir código da pasta (10 chars hex, ex: `cb54479644`) e o slug (ex: `zm`).
2. Coletar TODAS as empresas do grupo (blocos 1-3 acima, por empresa).
3. Rodar `consolidar.py <grupo> <codigo>` — monta o JSON com panorama = união dedup.
4. Criar a pasta `c/<codigo>/` com index.html + o `<grupo>.json` + o `<meta name="cliente-grupo">`.
   **ATENÇÃO:** sem a meta cliente-grupo o link do cliente dá 404 (incidente 23/07). Ver [[dashboard-pastas-cliente-meta]].
5. Adicionar o card do grupo no site-mãe (index.html) + no manifest.
6. Auditar + publicar.

---

## CASO C — Grupo de 1 EMPRESA / recorte por 1 CONTRATO (ex: SERPA, 29/07/2026)

Cliente único de assessoria mensal cujo dashboard cobre só a matéria do contrato.
Ex: **SERPA** — cliente SERPA Pré Fabricados (id 2147299), contrato mensal 103777
(Assessoria Trabalhista), só processos da ÁREA Trabalhista. Pasta `c/5b7507b043/`.

**Definir o recorte (LER a cláusula do contrato):**
- `get_contrato(<id>)` de cada contrato do cliente. O MENSAL é o que tem `meses` alto
  (103777 tinha meses=360) e receitas "Partido Mensal" (`list_receitas_do_contrato`).
- `list_processos(contrato=<id>)` costuma dar 0 (EasyJur não amarra processo a contrato).
  `list_processos(id_cliente=<id>)` traz TODOS; filtrar pela ÁREA que o objeto do contrato
  abrange (SERPA: só Trabalhista; Tributário/Cível eram de contratos avulsos). Confirmar com o PDF do contrato.

**Esforço e Horas com recorte por contrato:** o endpoint conta por CLIENTE (todos os processos),
mas o dash é só de 1 matéria. Aplicar FATOR proporcional = nº proc do recorte / nº total do cliente
(SERPA: 39/61). Horas marcadas `_estimativa:true` (^).

**Estrutura do JSON (grupo de 1 empresa) — igual ao Íntegra:**
- `empresas._panorama` (dataset com `_panorama:true`) + `empresas.<slug>` (a empresa).
- **A EMPRESA NÃO pode compartilhar o mesmo objeto do _panorama** — senão aparece rotulada
  "Consolidado do grupo" (duplicada). Fazer `copy.deepcopy(dataset)` + `pop('_panorama')`.
- `nEmpresas: 1`.

**Passos que EU ESQUECI no SERPA (não repetir):**
1. **Card único ESTICA** na largura — no render de `#cards-empresas`, aplicar classe `.few`
   quando `Object.keys(g.empresas).length<=2` (CSS `.cards.few{minmax(300px,380px);justify-content:start}`).
   O Panorama é full-width de propósito (`.is-panorama{grid-column:1/-1}`), isso NÃO é bug.
2. **Card "Informações do contrato"** (`_CONTRATOS` em index.html ~linha 1432, uso INTERNO):
   adicionar `<slug>:{valor_mensal, indice, reajuste_periodo, data_inicio}`. **valor_mensal = valor
   ATUALIZADO** (reajustado), não o do EasyJur original. SERPA = R$ 4.052,50 · Salário Mínimo · 2022-05-06.
   Não aparece no modo cliente nem no PDF. Data de início MANTÉM o dia (vigência real).
3. **KPIs seguem a lógica GLOBAL do JS** (não recalcular): Ritmo de novos processos = fórmula Dr. Kim
   `(estoque_base+entradas)/estoque_base`; Volume/Complexidade corte 2022+; Esforço/Horas corte 2023+
   (`ANO_MIN_ESFORCO`). Só dar dados brutos no JSON. Conferir simulando em Python + tela.
4. **"Atualizado em" / data de corte = SÓ MÊS/ANO** ("Julho de 2026"), NUNCA o dia.
   Usar `DATA_CORTE.extenso` (não `.curta`) em TEXTO exibido. As linhas que usam `.curta` em
   CÁLCULO (split de mês/ano p/ projeção/tramitação) ficam intactas.
5. **TESTAR no browser clicando pela UI real** (não via location.hash — não atualiza GRUPO_ATUAL).
   Ver [[dashboard-grupo-cliente-unico]].

---

---

## ✅ CHECKLIST FINAL — antes de dar por concluído qualquer grupo novo

**REGRA:** só falar "grupo pronto" ou "publicado" quando TODOS os itens abaixo estiverem marcados. Não pular NENHUM. Rangel cobra que essas coisas fiquem sempre feitas — se eu esqueço, quebra.

### 📋 Dados que o Rangel passa no chat (armadilha comum — eu esqueço)

- [ ] **`_CONTRATOS` no index.html** (~linha 1432): adicionei entrada do grupo com **valor mensal atualizado**, índice de reajuste, período, data de início? Sem isso, card "Informações do contrato" fica vazio.
- [ ] **`LINKS_CLIENTE` no index.html** (~linha 2130): adicionei `<slug>:'<hex_pasta>'`? **ARMADILHA CRÍTICA**: se o slug começa com dígito (ex: `3rs`), colocar aspas → `'3rs':'hex'`. Sem aspas quebra o JS inteiro e cortina trava.
- [ ] **Card `_panorama` do JSON**: nome é `"Panorama do grupo"` (não só `"Panorama"`)? Rangel pediu isso em 03/08/2026.
- [ ] **Observações que o Rangel me passou** (empresas extras a incluir, distratos, escopo, etc): apliquei tudo?

### 📋 Estrutura do JSON

- [ ] `dados/<slug>.json` gerado no formato dos modelos (kpi completo, dicts, encerrados série-ano)
- [ ] Empresa NÃO herda flag `_panorama:true` (deepcopy sem a flag)
- [ ] `nome`, `nomeHtml`, `nEmpresas` no topo
- [ ] Auditoria (`auditar_grupos.py`): **0 erros** obrigatório

### 📋 4 réplicas (raiz + _para_github + 2 pastas c/)

- [ ] `dados/<slug>.json` copiado para `_para_github/dados/`
- [ ] Pasta `c/<hex>/` criada em ambas as raízes com index + meta `cliente-grupo` + json do grupo
- [ ] `manifest.json` atualizado nas 2 raízes (adicionar slug em `ordem`)
- [ ] Cache-buster novo em TODOS os index.html (raiz + _para_github + todas as pastas c/)

### 📋 Auditoria de rastreabilidade

- [ ] `auditoria_processos/<slug>/` gerada (CSV+JSON por empresa) via `_template_auditoria`

### 📋 Teste no browser ANTES de push

- [ ] Subir servidor local (`python -m http.server`) e navegar como usuário real
- [ ] Card único não estica (classe `.few` se ≤2 cards)
- [ ] Card de contrato aparece (site interno) com valor certo
- [ ] Botão "Gerar link" copia URL correta
- [ ] Link do cliente `/c/<hex>/` abre direto no Panorama sem tela de seleção
- [ ] Cortina desce (nenhum erro JS no console)

### 📋 Git

- [ ] Commit + push em `_para_github/`
- [ ] Aguardar GitHub Pages propagar (~1-2 min)

**Se você (Claude) chegar até aqui sem marcar TODOS os itens, PARE e volte para o que faltou. Rangel prefere um grupo demorar 30 min a mais que ter um card vazio.**

---

## Erros que NÃO pode repetir (aprendidos na marra)

1. **NUNCA escrever `encerrados` como número** — é dict-por-ano (série). O total vai em `encerradosTotal`.
2. **area/uf/instancia/polo/tribunais = DICT**, nunca lista. O JS usa `Object.keys()`.
3. **Preservar `nome`, `nomeHtml`, `nEmpresas`** no topo — se sumir, card do site-mãe vira "undefined".
4. **`_panorama:true` e `_complexidadeSoAtivos:true`** têm que existir — senão Considerações/donut quebram.
5. **`kpi` completo** (proc, ativos, areas, trib, procSub, areasSub, tribSub) — faltar campo trava o KPI em "0".
6. **Sub-textos = top 3 SEM "Nao informado"/"N/D"/"N/I"**. Tribunais em sigla (TJSC, TRT12), não nome longo.
7. **Timesheet é instável:** se vier "feed grudado" (total 675667 ou total fixo de outro cliente),
   varie o page_size (100→90→80→50) — muda a chave de cache. Nunca paralelizar coletas pesadas de timesheet.
8. **Panorama = soma das empresas** (após dedup). A auditoria checa isso.
9. **Empresa gigante (>5 mil apontamentos):** paginar na mão é inviável. Autorizar estimativa
   (mediana h/ato de grupos reais × atos da empresa), marcada `_estimativa`. Foi o caso da Russi.

Ver também: [[dashboard-format-json-e-kpis]], [[dashboard-atualizacao-mensal-pendente]], `ATUALIZAR_MENSAL.md`.
