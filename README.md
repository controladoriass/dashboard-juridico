# Dashboard de Indicadores Jurídicos — Silva & Silva Advogados

Painel de indicadores jurídicos por cliente/grupo econômico, gerado a partir dos dados do EasyJur.

## O que é

Dashboard web (arquivo único, `index.html`) que apresenta, para cada grupo/empresa:

- **Volume** — evolução do estoque de processos, entradas × encerramentos, crescimento
- **Complexidade** — distribuição por área, instância, valor de causa, geografia, tribunais, ranking, polo
- **Esforço operacional** — atividades, profissionais, audiências *(dados estimados — a integrar via API)*
- **Horas dedicadas** — timesheet *(dados estimados — a integrar via API)*

Navegação em 3 níveis: **Grupos → Empresas do grupo → Dashboard da empresa**.

## Como funciona

- Página estática, sem dependências de servidor (só Google Fonts via CDN).
- Os dados de cada empresa ficam no objeto `GRUPOS` dentro do próprio `index.html`.
- Botão **Gerar PDF** (Ctrl+P → paisagem) exporta o relatório do cliente.
- Deep-link: `index.html#grupo/empresa` abre direto no dashboard da empresa.

## Como adicionar um novo cliente/card

Editar o objeto `GRUPOS` no `index.html`, acrescentando um novo grupo com suas empresas.
O card, a navegação e os gráficos são gerados automaticamente.

## Hospedagem

Publicado via Netlify (deploy automático a cada atualização deste repositório).

---
**Confidencial — Silva & Silva Advogados Associados.** Os dados brutos de clientes não integram este repositório.
