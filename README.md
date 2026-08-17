# VIEW MARKETING: Pipeline de Tráfego Pago com Meta Ads API

Dashboard de BI para monitoramento de performance de mídia paga,
cobrindo Meta Ads (Facebook/Instagram) com pipeline de extração
via API, armazenamento em PostgreSQL e visualização em Power BI.

---

## O Problema

A equipe de marketing não tinha visibilidade consolidada dos dados
de tráfego pago. Os números do Meta Ads ficavam presos no Ads Manager,
sem integração com os dados de leads e contratos do sistema interno (IXC).
Não era possível calcular CAC, ROI ou comparar performance por canal
de forma automatizada.

## A Solução

Pipeline completo de dados cobrindo três etapas:

1. **Extração**: Script Python que consome a API do Meta Ads
   (Graph API v19.0) com suporte a paginação completa, cobrindo
   todas as campanhas desde 2026-01-01 com atualização diária.

2. **Armazenamento**: Carga incremental no PostgreSQL com
   `ON CONFLICT` para garantir idempotência: reprocessar não
   duplica nem corrompe o histórico.

3. **Visualização**: Dashboard Power BI com 8 canais de aquisição
   (Meta, Google, TikTok, Orgânico, Indicação, Hotspot, Campanha
   Externa, LP Sem Rastreio), funil MQL→SQL→Contratos e KPIs
   financeiros (CAC, ROI LTV, Ticket Médio, MRR).

## Desafios Técnicos Resolvidos

**Paginação da API:** A API do Meta retorna no máximo 500 registros
por chamada. Sem tratamento de paginação, apenas 7 das 195 campanhas
eram recuperadas. O pipeline implementa loop `while` no cursor
`paging.next` até esgotar os dados.

**Idempotência:** Reprocessamentos não geram duplicidade.
O `ON CONFLICT (date_start, campaign_name) DO UPDATE` garante
que rodar o pipeline duas vezes no mesmo dia produz o mesmo resultado.

**Arquitetura desconectada no Power BI (Rota B):** O slicer de canal
não usa relacionamento direto com a tabela de spend. O roteamento é
feito via `SELECTEDVALUE` + `SWITCH` em DAX, permitindo que cada
medida filha saiba qual canal está selecionado sem depender de
relacionamento no modelo, padrão necessário para cruzar fontes
heterogêneas (API de ads + sistema IXC).

**Validação:** O spend extraído foi validado manualmente contra
o Ads Manager da Meta. Diferença final: menos de R$50 em
R$115,984 de investimento no período.

## Stack

| Camada | Tecnologia |
|---|---|
| Extração | Python 3, Requests, python-dotenv |
| Banco de dados | PostgreSQL 14, psycopg2 |
| Ambiente | WSL 2 Ubuntu, venv |
| Modelagem | Power BI Desktop, DAX |
| Fonte de dados | Meta Graph API v19.0 |

## Estrutura do Repositório

```
view-marketing-bi/

├── extratores/

│   └── meta_extrator.py   # Pipeline Meta Ads → PostgreSQL

├── .env.example           # Modelo de variáveis de ambiente

├── .gitignore

└── README.md
```

## Como Usar

```bash
# 1. Clone o repositório
git clone https://github.com/MihrVMF/view-marketing-bi.git
cd view-marketing-bi

# 2. Crie o ambiente virtual e instale dependências
python -m venv venv
source venv/bin/activate
pip install requests psycopg2-binary python-dotenv

# 3. Configure as variáveis de ambiente
cp .env.example .env
# Edite o .env com seus tokens e credenciais

# 4. Execute o extrator
python extratores/meta_extrator.py
```

## Resultados

- 9 campanhas extraídas, 2554 registros históricos carregados
- Spend validado contra Ads Manager: diferença < R$50
- Pipeline roda diariamente via agendamento
- Dashboard em produção, utilizado pelo time de marketing
