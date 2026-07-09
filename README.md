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
- `esforco`: `prazano`, `audano`, `tarefano` (compromissos por ano), `atividades` (total por tipo), `comProducao`, `tramitacaoDias`, **`extrajudano`** (atividades extrajudiciais por ano), **`extrajudtipo`** (extrajudicial por tipo de trabalho).
- `horas`: `total`, `media`, `mediaAtivos`, `horasano`, `_estimativa`.
- Flags: `_dadosReais`, `_semProcessos` ("sem processos"), `_semDados` ("em breve"), `_panorama`.

O `_panorama` é a **soma consolidada** das empresas do grupo. Grupo de 1 empresa não tem `_panorama` (usa a própria empresa como topo). O card **"Partes atendidas"** no Panorama = `nEmpresas` (nº de empresas do grupo).

---

## 2. Recorte de exibição 2022→2026 (importante)

O dashboard mostra **apenas 2022 a 2026** nas séries/KPIs por ano (a base pré-2022 é pouco confiável). Isto **NÃO exclui processos** — só corta a série dos gráficos.

- Controlado por `const ANO_MIN = 2022;` no topo do `<script>` + função `cortaAnos(obj)`.
- Cortados: estoque, entradas×encerramentos, prazos/audiências/tarefas por ano, horas por ano, extrajudicial por ano, "Histórico processual".
- Recalculados sobre o recorte (para o KPI bater com o gráfico): crescimento do estoque, entradas/encerramentos, compromissos, horas totais/médias.
- **Preservados reais** (não cortados): totais de processos, ativos, áreas, tribunais.
- Para mudar o corte no futuro: alterar só o `ANO_MIN`.

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
- **Atividades por ano** (`extrajudano`): `list_agenda clientes=[ids do grupo], subtipo=18332, mostrar_etapas=2, data_interna_inicio/fim=<ANO>, page_size=1` → `meta.total` por ano. O filtro `clientes=[lista]` faz 1 chamada por grupo/ano.
- **Por tipo** (`extrajudtipo`): paginar as descrições (subtipo 18332) e classificar por palavra-chave/sigla no texto: NE=Notificação, CV/Contrato/Minuta/Termo/MOU=Contratos, CE=Cobrança, ADR/Distrato/Rescisão/Acordo=Distratos e acordos, MP/Parecer/Análise=Pareceres, resto=Outros. **Classificação aproximada** (~85-90%) — a nomenclatura do time varia; nota disso fica no card.
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
Por decisão do escritório: Resultado/êxito, Risco, Financeiro/Honorários (baixa cobertura). Também **não** há: "Ranking de processos de maior valor" (removido), textos de "Leitura" interpretativa (removidos), horas do extrajudicial (inviável).

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
   var _cb = '?v=20260708c';   // incrementar a cada atualização
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
