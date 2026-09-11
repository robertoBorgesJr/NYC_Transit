# NYC Mobility Data Pipeline

**Arquitetura Medallion · 1,6 bilhão de registros · Databricks + Delta Lake**

[![Platform](https://img.shields.io/badge/Platform-Databricks-FF3621?logo=databricks&logoColor=white)](https://databricks.com)
[![Storage](https://img.shields.io/badge/Storage-Delta_Lake-003366?logo=apachespark&logoColor=white)](https://delta.io)
[![Language](https://img.shields.io/badge/Language-PySpark_%7C_Spark_SQL-E25A1C?logo=apachespark&logoColor=white)](https://spark.apache.org)
[![Viz](https://img.shields.io/badge/Viz-Plotly-3F4F75?logo=plotly&logoColor=white)](https://plotly.com)
[![License](https://img.shields.io/badge/License-MIT-blue)](LICENSE)

---

## Visão Geral

Este projeto constrói um pipeline de dados end-to-end no Databricks para analisar a evolução do **market share** entre táxis amarelos tradicionais e serviços de aplicativo (Uber/Lyft) em Nova Iorque entre 2009 e 2019.

O pipeline processa mais de **1,6 bilhão de registros** de viagens usando a **Arquitetura Medallion** (Bronze → Silver → Gold), entregando uma camada analítica pré-agregada que reduz em ~99% o custo de processamento na camada de visualização.

**Pergunta de negócio central:**  
> *Em que momento os aplicativos de mobilidade superaram os táxis amarelos em NYC — e quão rapidamente essa transição ocorreu?*

---

## Arquitetura

```
┌─────────────────────────────────────────────────────────────────────┐
│                        FONTES DE DADOS                              │
│                                                                     │
│  /databricks-datasets/nyctaxi/tripdata/yellow/*.csv.gz              │
│  /databricks-datasets/nyctaxi/tripdata/fhv/*.csv.gz                 │
│  /databricks-datasets/nyctaxi/tripdata/fhvhv/*.csv.gz               │
│  /databricks-datasets/nyctaxi/taxizone/taxi_zone_lookup.csv         │
└───────────────────────────┬─────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────────┐
│  BRONZE  (01_Bronze_Jobs)                                           │
│  Ingestão fiel · CSV.GZ → Delta · Sem transformações               │
│                                                                     │
│  nyc_transit.yellow_taxi.bronze_yellow                              │
│  nyc_transit.yellow_taxi.bronze_fhv                                 │
└───────────────────────────┬─────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────────┐
│  SILVER  (02_Silver_Jobs)                                           │
│  Padronização · Cast dinâmico · Limpeza · Remoção de nulos         │
│                                                                     │
│  nyc_transit.yellow_taxi.silver_yellow                              │
│  nyc_transit.yellow_taxi.silver_fhv                                 │
└───────────────────────────┬─────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────────┐
│  GOLD  (03_Gold_Jobs)                                               │
│  União · Enriquecimento temporal · Particionamento por ano         │
│                                                                     │
│  nyc_transit.yellow_taxi.gold_mobility_ny          ← fact table     │
│  nyc_transit.yellow_taxi.gold_market_share_summary ← agregação      │
└───────────────────────────┬─────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────────┐
│  ANÁLISE  (04_analysis)                                             │
│  Plotly · Área empilhada 100% · Market share mensal 2009-2019      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Estrutura do Repositório

```
NYC_Transit/
├── pipelines/
│   ├── 00_Config_Setup.ipynb     # Variáveis globais, caminhos, Spark config
│   ├── 01_Bronze_Jobs.ipynb      # Ingestão raw → Delta
│   ├── 02_Silver_Jobs.ipynb      # Limpeza e normalização
│   └── 03_Gold_Jobs.ipynb        # Agregação e cálculo de market share
├── analysis/
│   └── 04_analysis.ipynb         # Visualização interativa (Plotly)
├── sandbox/
│   └── sandbox_exploration.ipynb # Experimentos de performance e Z-Order
├── Dados/
│   └── yellow_tripdata_2024-01.parquet  # Amostra local (48 MB)
└── README.md
```

---

## Fontes de Dados

| Dataset | Período | Volume aprox. | Formato |
|---------|---------|---------------|---------|
| Yellow Taxi | 2009–2019 | ~1,4 B linhas | CSV.GZ |
| FHV (For-Hire Vehicle) | 2015–2018 | ~150 M linhas | CSV.GZ |
| FHV-HV (Uber/Lyft) | 2019+ | ~80 M linhas | CSV.GZ |
| Taxi Zone Lookup | — | ~265 linhas | CSV |

Todos os datasets são públicos e disponíveis nativamente no ambiente Databricks em `/databricks-datasets/nyctaxi/`.

---

## Esquema das Camadas

### Bronze
Preserva o schema original dos arquivos CSV sem qualquer transformação. Garante reprocessamento idempotente.

### Silver

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| `pickup_datetime` | timestamp | Horário de início da corrida (normalizado) |
| `dropoff_datetime` | timestamp | Horário de término (pode ser nulo em FHV) |
| `taxi_type` | string | `"Yellow Taxi"` ou `"App (Uber/Lyft)"` |

> **Desafio de esquema heterogêneo:** O Yellow Taxi usa `tpep_pickup_datetime`; registros FHV usam `pickup_datetime` ou `pickup_date`. A Silver normaliza tudo via `try_to_timestamp()` com mapeamento dinâmico de colunas.

### Gold — `gold_mobility_ny`

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| `pickup_datetime` | timestamp | Horário de partida |
| `dropoff_datetime` | timestamp | Horário de chegada |
| `taxi_type` | string | Tipo de serviço |
| `year` | int | Ano da corrida (partição) |
| `month` | int | Mês da corrida |

### Gold — `gold_market_share_summary`

| Coluna | Tipo | Descrição |
|--------|------|-----------|
| `year` | int | Ano de referência |
| `month` | int | Mês de referência |
| `taxi_type` | string | Tipo de serviço |
| `total_viagens` | long | Volume de corridas no período |
| `market_share_pct` | double | Participação de mercado (%) |
| `reference_date` | date | Data no formato `YYYY-MM-01` |

---

## Otimizações de Performance

### Particionamento por ano
A tabela `gold_mobility_ny` é particionada por `year`, permitindo *partition pruning* nas queries e eliminando leitura de dados irrelevantes.

### Z-Order (Data Skipping)
```sql
OPTIMIZE gold_mobility_ny
ZORDER BY (taxi_type, pickup_datetime)
```
Reorganiza fisicamente os arquivos Delta para maximizar o *data skipping* em filtros por tipo de serviço e intervalo de tempo — crítico em 1,6 B de linhas.

### Adaptive Query Execution (AQE)
```python
spark.conf.set("spark.sql.shuffle.partitions", "200")
```
AQE ajusta dinamicamente o número de partições de shuffle em tempo de execução, evitando o *small file problem* e reduzindo overhead de tarefas.

### Projeção antecipada de colunas
Colunas desnecessárias são descartadas antes de qualquer `join` ou `union`, reduzindo footprint de memória e I/O de rede.

### Camada de agregação (Gold Summary)
A tabela `gold_market_share_summary` consolida 1,6 B de linhas em ~264 linhas mensais (11 anos × 12 meses × 2 tipos), eliminando re-processamento na camada de visualização e reduzindo em **~99%** o custo de cada consulta analítica.

---

## Como Executar

### Pré-requisitos
- Databricks workspace com acesso ao dataset público `/databricks-datasets/nyctaxi/`
- Unity Catalog configurado com catálogo `nyc_transit` e schema `yellow_taxi`
- Cluster com Databricks Runtime 13.x ou superior (suporte a Delta 2.x)

### Execução manual (notebook por notebook)

Execute na seguinte ordem — cada notebook faz `%run ./00_Config_Setup` automaticamente:

```
pipelines/00_Config_Setup.ipynb   → Inicializa variáveis e configurações
pipelines/01_Bronze_Jobs.ipynb    → Ingere os CSVs e salva em Delta
pipelines/02_Silver_Jobs.ipynb    → Limpa, normaliza e tipifica os dados
pipelines/03_Gold_Jobs.ipynb      → Consolida, enriquece e agrega
analysis/04_analysis.ipynb        → Gera o gráfico interativo
```

### Execução via Databricks Workflows

Configure um job com as seguintes tasks sequenciais:

| Ordem | Task | Notebook |
|-------|------|----------|
| 1 | config | `pipelines/00_Config_Setup` |
| 2 | bronze | `pipelines/01_Bronze_Jobs` |
| 3 | silver | `pipelines/02_Silver_Jobs` |
| 4 | gold | `pipelines/03_Gold_Jobs` |
| 5 | analysis | `analysis/04_analysis` |

---

## Resultado & Insight Principal

O gráfico de área 100% empilhada mostra mês a mês a evolução do market share entre 2009 e 2019:

- **2009–2014:** Domínio absoluto do táxi amarelo (>95% do mercado)
- **2015:** Início da penetração acelerada dos aplicativos
- **~2016:** Ponto de inflexão — apps ultrapassam táxis amarelos em volume
- **2019:** Apps respondem por mais de 70% das corridas registradas

> Este padrão reflete a disrupção promovida pela entrada do Uber (2011) e Lyft (2014) no mercado de NYC, intensificada pela aprovação regulatória progressiva da TLC (Taxi & Limousine Commission).

---

## Stack Tecnológico

| Camada | Tecnologia |
|--------|-----------|
| Processamento | Apache Spark (PySpark + Spark SQL) |
| Plataforma | Databricks (Serverless / Job Clusters) |
| Armazenamento | Delta Lake (ACID, versioning, Z-Order) |
| Catálogo | Databricks Unity Catalog |
| Visualização | Plotly |
| Controle de versão | Git |

---

## Catálogo Unity Catalog

```
nyc_transit (catalog)
└── yellow_taxi (schema)
    ├── bronze_yellow
    ├── bronze_fhv
    ├── silver_yellow
    ├── silver_fhv
    ├── gold_mobility_ny          ← particionada por year
    └── gold_market_share_summary ← 264 linhas, pronta para dashboards
```
