# Changelog

Todas as mudanças relevantes deste projeto estão documentadas aqui.
Formato baseado em [Keep a Changelog](https://keepachangelog.com/pt-BR/1.1.0/).

## [Não publicado]

## [2026-08-10] Expansão em massa + padronizações

### Adicionado
- **+64 grupos publicados** (de 39 para 103), cobrindo toda a categoria C de contratos ativos
- Card "Informações do contrato" agora aparece em **todos** os 103 grupos
- 53 contratos com valor mensal, data de vigência e índice de reajuste populados via EasyJur
- Documentação técnica reorganizada em `docs/`

### Modificado
- **Padronização de nomes**: todos os grupos começam com prefixo "Grupo " seguido de Title Case (siglas em CAPS preservadas: ABC, WF, MG7, 300F, ZM S.A., etc.)
- Card de panorama renomeado para "Panorama Geral do Grupo" em todos os grupos
- Nomes de empresas convertidos para Title Case (preservando LTDA, S.A., EIRELI, SPE)
- README simplificado para foco em contexto de projeto (histórico movido para CHANGELOG)

### Removido
- Grupo RHONALDO MOTORSPORTS (distrato)

### Corrigido
- **Meta `cliente-grupo` inserida no `<head>` real** de cada `c/<hex>/index.html` (antes era substituída dentro de um comentário JS, causando "erro ao carregar dados")
- **Propagação do `index.html` master** para as 103 pastas `c/<hex>/` — cada cliente agora recebe versão sincronizada
- **LINKS_CLIENTE** completo com todos os 103 slugs (58 estavam faltando, causando 404 nos botões dos cards)
- Slugs que iniciam com dígito (`'300f'`, `'3rs'`) agora usam aspas em `_CONTRATOS` e `LINKS_CLIENTE` (sem aspas causava SyntaxError e travamento da cortina)
- 24 grupos renomeados conforme revisão editorial

## [2026-08-06] Publicação categoria C

### Adicionado
- **6 grupos** com sufixo próprio: MS Incorporadora, AR, YANY, MG7, Platão, Praiholl
- **4 grupos** adicionais: ERBS, Binotto, Riviera, M.S. Luzitânia
- Auditoria dos processos gerada em pasta local (excluída do repo público)

### Corrigido
- Cache-buster do manifest sincronizado com `dados/<slug>.json`
- Bug OIDC do GitHub Pages contornado com retry manual após restauração do serviço

## [2026-08-03] Card contrato + revisão de grupos

### Adicionado
- Bloco "Informações do contrato" no header do dashboard de cada grupo
- Card exibe valor mensal, índice de reajuste, período e data de início

## [2026-07-24] Cobertura completa dos KPIs

### Adicionado
- **Bloco "Partes atendidas"** no Panorama do grupo (aparece quando há >1 empresa real)
- KPI `k-partes` no grid do topo (6º card) com contagem de empresas
- Seção `#partes-block` com pills douradas por empresa e animação em cascata

## [2026-07-23] Refactor + auditoria

### Modificado
- **Panorama = soma das empresas** — filtro exato `id_cliente=<INT>` (antes usava LIKE parcial que trazia processos de terceiros)
- `DATA_CORTE` atualizada: 30/06/2026 → 23/07/2026
- **KPI Ritmo** com fórmula do Dr. Kim: `(estoque_base + entradas) / estoque_base`
- Blocos Esforço e Horas com recorte a partir de 2023 (`ANO_MIN_ESFORCO`)
- Complexidade "só ativos" aplicada consistentemente nos 10 grupos originais
- Nomenclatura: "Tarefas" → "Reuniões e Diligências"
- `_formatRatio` com 1 casa decimal

### Corrigido
- Donut de complexidade no PDF (número navy visível no papel)
- KPIs travando em "0" quando aba estava oculta durante `countUp` — fallback + rede de segurança 2,5s
- "Partes atendidas" no PDF em grid 2 colunas com pills compactas
- Página "Atividades extrajudiciais" removida especificamente do PDF do ABC

## Versões anteriores

Histórico completo disponível via `git log`.
