# Dashboard de Indicadores Jurídicos

Dashboard estático que apresenta indicadores jurídicos por grupo econômico/cliente, alimentado por dados extraídos do sistema de gestão jurídica (EasyJur).

**Live:** https://controladoriass.github.io/dashboard-juridico/

---

## Sobre

Sistema de visualização de indicadores jurídicos para 100+ grupos econômicos atendidos pelo escritório Silva & Silva Advogados. Cada grupo é uma página independente que consolida:

- Volume de processos (estoque, entradas, encerrados)
- Complexidade da carteira (por área, UF, instância, tribunal)
- Esforço operacional (prazos, audiências, tarefas, extrajudicial)
- Horas dedicadas (via timesheet)
- Informações contratuais (valor, vigência, reajuste)

O dashboard mãe apresenta cards de todos os grupos com KPIs consolidados. Cada grupo tem um link isolado (`/c/<hex>/`) que pode ser enviado ao cliente sem expor os demais.

## Stack

- **HTML/CSS/JS puro** — sem build, sem dependências. Um único `index.html` propagado pra 103 pastas.
- **JSON estático** — cada grupo tem um `dados/<slug>.json` gerado offline a partir do EasyJur.
- **GitHub Pages** — deploy automático no push pra `main`.

## Estrutura

```
.
├── index.html          Aplicação completa (CSS + JS + HTML inline)
├── dados/
│   ├── manifest.json   Ordem de exibição dos grupos
│   └── <slug>.json     Um arquivo por grupo (103 grupos)
├── c/<hex>/
│   └── index.html      Cópia do master com meta cliente-grupo (link isolado)
└── docs/               Documentação técnica interna
```

Cada `dados/<slug>.json` contém:
- `kpiGrupo` — KPIs consolidados usados no card do site-mãe
- `empresas._panorama` — visão agregada do grupo
- `empresas.<slug_empresa>` — visão individual de cada empresa

## Deploy

Push pra `main` dispara build automático do GitHub Pages. Processo típico leva 1-2min.

## Documentação

- [`docs/adding-groups.md`](docs/adding-groups.md) — como adicionar/editar grupos e empresas
- [`docs/monthly-update.md`](docs/monthly-update.md) — procedimento de atualização mensal
- [`CHANGELOG.md`](CHANGELOG.md) — histórico de mudanças relevantes

## Estado atual

- **103 grupos publicados** (corte: 27/07/2026)
- 68 grupos com contrato completo (valor + vigência + reajuste)
- 35 grupos aguardando dados contratuais

## Licença e uso

Repositório privado da Silva & Silva Advogados Associados. Todos os direitos reservados.
Dados de clientes e processos são confidenciais e não são publicados aqui — o repositório contém apenas dados agregados anônimos suficientes para renderizar os indicadores.
