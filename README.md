# Dashboard de Indicadores Jurídicos

Dashboard estático (HTML/CSS/JS puro) que apresenta indicadores jurídicos por grupo econômico/cliente, alimentado por dados extraídos do sistema de gestão jurídica (EasyJur).

Este documento descreve **a estrutura do projeto e o procedimento de manutenção** (adicionar grupo, editar/excluir empresa, atualização periódica dos dados). Não contém dados de processos nem de clientes.

---

## 1. Estrutura de arquivos

```
/
├── index.html          → todo o código (CSS + JS + HTML das seções). NÃO contém dados.
└── dados/
    ├── manifest.json    → lista com a ordem de exibição dos grupos: {"ordem": ["russi","wf",...]}
    ├── russi.json       → dados de UM grupo (kpiGrupo + empresas + panorama)
    ├── abc.json
    ├── ...              → um arquivo .json por grupo
    └── (10 grupos no total)
```

- O `index.html` **carrega** os JSONs de `dados/` no boot (via `fetch`), monta o objeto `GRUPOS` em memória e renderiza. Por isso o site **só funciona servido por HTTP** (GitHub Pages, ou um servidor local) — abrir o arquivo direto do disco (`file://`) não carrega os dados.
- Hospedado no GitHub Pages: cada `git push` na branch `main` republica o site automaticamente (leva 1–2 min). Se o deploy falhar (instável), re-disparar com: `gh api -X POST repos/<owner>/<repo>/pages/builds`.

### Formato de um `dados/<grupo>.json`

```
{
  "nome": "Grupo X",
  "nomeHtml": "Grupo <em>X</em>",
  "kpiGrupo": { proc, procSub, horas, horasSub, areas, areasSub, valor, trib, tribSub, ativos, encerrados, valorTotal },
  "empresas": {
    "_panorama": { "nome": "Panorama Geral do Grupo", "_panorama": true, "dataset": {...} },
    "<chave_empresa>": { "nome": "...", "_idCliente": "<ID>", "dataset": {...} },
    ...
  },
  "nEmpresas": <n>
}
```

O `dataset` de cada empresa contém: `estoque`/`entradas`/`encerrados` (por ano), `area`, `areaTotal`, `instancia`, `polo`, `uf`, `tribunais`, `ativos`, `valorTotal`, `qtdComValor`, `esforco` (prazano/audano/tarefano/atividades/comProducao/tramitacaoDias) e `horas` (total/media/mediaAtivos/horasano/_estimativa). Flags: `_dadosReais`, `_semProcessos` (badge "sem processos"), `_semDados` (badge "em breve"), `_panorama`.

O `_panorama` de cada grupo é a **soma consolidada** das empresas (média ponderada por nº de processos para `tramitacaoDias`; ranking = top-5 reordenado). Grupo de 1 empresa não tem `_panorama`.

---

## 2. Manutenção comum

### Excluir um grupo inteiro
1. Apagar `dados/<grupo>.json`
2. Remover o nome do grupo da lista em `dados/manifest.json`
3. `git commit` + `git push`

### Excluir / editar uma empresa de um grupo
1. Abrir `dados/<grupo>.json`
2. Localizar o bloco da empresa (em `empresas`) e apagar/editar
3. (Opcional, para consistência) recalcular o `_panorama` e o `kpiGrupo` somando as empresas restantes
4. `git push`

### Adicionar um grupo novo
1. Coletar os dados do grupo (ver seção 3)
2. Criar `dados/<grupo>.json` no formato acima
3. Adicionar o nome do grupo em `dados/manifest.json`
4. `git push`

**Não é preciso editar o `index.html`** em nenhum desses casos — ele é só o motor de renderização.

---

## 3. Coleta e atualização de dados (procedimento)

A fonte é a API do EasyJur (via MCP). A atualização periódica (ex.: mensal, mudando a **data de corte**) consiste em **re-coletar cada grupo** com o novo corte e regravar os `dados/<grupo>.json`. Não há delta incremental na API — reconsulta-se cada empresa.

### Regra de ouro (tratamento honesto dos dados)
- **Volume/processos/evolução/áreas/geografia**: sempre 100% reais, nunca estimados.
- **Horas**: reais do timesheet, mas marcadas com `^` e nota de "parcial" (a API não recupera 100% dos apontamentos).
- Nunca inventar dado. Quando um campo não existe no sistema, deixar vazio/“—” ou marcar `[VERIFICAR]`.
- **Valor de causa vazio** = êxito / a arbitrar pelo juízo / peça acessória (não é erro).

### Passo a passo por grupo
1. **Processos** — `list_processos` filtrando por cliente. ATENÇÃO: o parâmetro varia — testar `cliente=<ID>` **e** `id_cliente=<ID>` e usar o que retorna processos cujo `cliente_info.id == <ID>` (o outro às vezes traz empresa errada por busca parcial). Paginar até o fim (`page_size=100`). Agregar: `areaTotal`, `ativos` (status==1), `area`, `instancia`, `polo` (ativo/passivo/terceiro), `uf`, `tribunais`, `ranking`, `valorTotal`/`qtdComValor`, e `estoque`/`entradas`/`encerrados` por ano (entradas por `data_distribuicao`, fallback `data_cadastro`; estoque acumulado nunca negativo, último ano = nº de ativos).
2. **Esforço** — `list_agenda` com `mostrar_etapas=2` (só compromissos-mãe, exclui subetapas de workflow, senão infla). Contar PRAZO/AUDIENCIA/TAREFA por ano via `data_interna_inicio/fim`.
3. **Horas** — `list_timesheets` **por número de processo** (`n_processo=<numero>`), para cada processo do cliente. Regras obrigatórias:
   - Validar `cliente == <ID>` em **cada linha** (o filtro `n_processo` é busca parcial e contamina muito com outros clientes).
   - Descartar `status == "S"` (cancelado).
   - **Dedup por `id`** do timesheet (processos compartilham apontamentos de agenda-mãe).
   - Converter `tempo_timesheet` HH:MM:SS → decimal; somar por ano; ignorar `data_timesheet` após a data de corte.
   - Total marcado como parcial (`^`).
4. **Injeção** — gravar tudo no `dataset` da empresa; recalcular `_panorama` (soma) e `kpiGrupo`.

### Cuidados aprendidos (importantes)
- **Empresas grandes (> ~60 processos)**: dividir a coleta de horas em **lotes de ~50 processos por vez** — coletar 100+ de uma vez estoura o limite/trava.
- **Overload da API**: a API do EasyJur engasga sob carga (erro 500/"Overloaded"). Se falhar, esperar alguns minutos e re-tentar só o lote que falhou. Preferir horários de baixo uso.
- **Empresas com 0h**: normal quando as horas do grupo estão lançadas sob **outro cadastro** do mesmo grupo. Não é erro — o volume/processos delas continua correto. O card de horas fica oculto (não mostra 0 falso).
- **Cadastros duplicados / nomes parecidos**: validar sempre pelo `meta.total` real da API e pelo `cliente_info.id`, para não confundir empresas irmãs (ex.: matriz vs. cadastro secundário).
- **Números de processo corrompidos** no export (notação científica / título no lugar do número): pular e registrar `[VERIFICAR]`.

### Escopo definido (não exibir)
Por decisão do escritório, **não** entram no dashboard: Resultado/êxito, Risco, Financeiro/Honorários (campos com baixa cobertura no sistema). Também foi removido o "Ranking de processos de maior valor".

---

## 4. Publicação

```
# na pasta do repositório
git add index.html dados/
git commit -m "Atualizacao <mês/ano> — corte DD/MM/AAAA"
git push origin main
# aguardar o GitHub Pages republicar (1–2 min)
```

O site fica em: `https://<owner>.github.io/<repo>/`

> **Nota sobre automação:** hoje a atualização é um processo assistido (reconsulta manual via API a cada período). Uma automação completa (backend puxando do EasyJur em cron) seria um projeto à parte.
