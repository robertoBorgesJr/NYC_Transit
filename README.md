# NYC Mobility Data Pipeline: Arquitetura Medallion com 1.6 Bi de Registros
Este projeto demonstra a construção de um ecossistema de dados completo no Databricks, utilizando o dataset público de táxis e aplicativos (Uber/Lyft) de Nova Iorque. O objetivo é analisar o Market Share entre o transporte tradicional e o digital ao longo de uma década (2009-2019).

#🏗️ Arquitetura do Sistema
O projeto segue a Arquitetura Medallion, garantindo governança, qualidade e escalabilidade:

1. Bronze (Raw): Ingestão fiel dos dados originais em formato Delta. Mantemos a imutabilidade dos dados brutos para permitir reprocessamentos futuros.

2. Silver (Cleaned): Padronização de nomes de colunas (lower case) e conversão dinâmica de tipos. Implementamos lógica de tratamento para colunas inconsistentes (como pickup_datetime vs tpep_pickup_datetime).

3. Gold (Curated): União massiva de dados e enriquecimento temporal.

4. Gold Summary (Analytics): Uma camada de agregação otimizada para consumo imediato por dashboards, reduzindo custos de processamento em 99% na camada de visualização.

----
#🛠️ Tecnologias e Otimizações
* Databricks: Ambiente Spark para processamento distribuído.

* Delta Lake: Armazenamento ACID que permite o uso de OPTIMIZE e Z-ORDER.

* Spark SQL & PySpark: Mix de linguagens para máxima flexibilidade e performance.

* Data Skipping com Z-Order: Implementação de organização física de dados por taxi_type para otimizar filtros em 1.6 bilhão de linhas.

* Adaptive Query Execution (AQE): Configuração de partições dinâmicas para evitar o Small File Problem.

----
#📊 Insights & Visualização
Utilizamos a biblioteca Plotly para gerar visualizações interativas diretamente da camada Gold Summary. O gráfico demonstra a transição histórica do mercado de mobilidade em NYC, revelando o ponto de inflexão onde os aplicativos (Uber/Lyft) superaram os táxis amarelos.

----
#🚀 Como Executar o Projeto
Os notebooks devem ser executados na sequência abaixo, configurada no Databricks Workflows:

1. 00_Config_Setup: Define variáveis globais, catálogos e caminhos de arquivos.

2. 01_Bronze_Jobs: Executa a ingestão bruta.

3. 02_Silver_Jobs: Realiza a limpeza e normalização.

4. 03_Gold_Jobs: Consolida a base final e a tabela de Market Share.

5. 04_Analysis: Gera os gráficos de negócio.
