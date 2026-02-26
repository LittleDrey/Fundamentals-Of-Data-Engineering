# Fundamentals-Of-Data-Engineering
Consolidando Conceitos e Fundamentos do Livro 'Fundamentals-Of-Data-Engineering' em União com Boas Práticas de Trabalho


# ⚽ Futebol Analytics Data Pipeline

> Um projeto de Engenharia de Dados End-to-End aplicando os conceitos fundamentais do livro *"Fundamentals of Data Engineering"* (Joe Reis & Matt Housley).

## 🎯 Objetivo do Projeto
Construir um pipeline de dados robusto, escalável e moderno para processar dados de futebol (jogadores, partidas, torneios, ...), traduzindo a teoria de engenharia de dados em prática real utilizando **Databricks** e **Delta Lake**.

O projeto segue a arquitetura **Medallion** (Bronze, Silver, Gold), com foco em qualidade de dados, governança e otimização de armazenamento.

## 📚 Fundamentação Teórica & Decisões Arquiteturais

Este projeto é guiado pelas fases do Ciclo de Vida da Engenharia de Dados. Abaixo estão as decisões técnicas tomadas para cada etapa:

### 1. Ingestão (Ingestion)
*Baseado no Capítulo 7: Ingestão de Dados*

* **Padrão de Movimentação:** **Push** (Empurrar). Os arquivos são enviados da origem local para o Data Lake.
* **Frequência:** **Batch** (Lote). Como lidamos com dados históricos de partidas e torneios, a ingestão em lote é a escolha pragmática, evitando a complexidade desnecessária de streaming para dados que não exigem latência de milissegundos.
* **Filosofia:** "Encanamento" (Plumbing). Nesta etapa, o foco foi puramente mover os dados do ponto A (Local) para o ponto B (Staging Zone) sem transformações, garantindo uma cópia fiel da origem.

## 🛠️ Infraestrutura e Organização

### Estrutura do Data Lake (Staging Zone)
Para evitar o antipadrão do "Data Swamp" (Pântano de Dados), a zona de aterrissagem (Staging) foi estruturada hierarquicamente para garantir governança e facilitar a leitura automatizada:

```text
staging_zone/ (Volume Databricks)
├── futebol_db/          <-- Sistema de Origem (Source System)
│   ├── torneios/        <-- Entidade de Negócio
│   │   ├── csv/         <-- Formato do Arquivo
│   │       └── _torneios.csv
│   ├── jogadores/
│   │   ├── json/
│   │       └── _jogadores.json
│   └── partidas/
│   │    ├── csv/
│   │       └── _partidas.csv
```
---

## 🚧 Desafios de Engenharia e Soluções (War Stories)

Durante a fase de Ingestão, enfrentei limitações de infraestrutura e problemas de qualidade de dados na origem. Abaixo documentei como superar cada barreira, alinhando com os fundamentos do livro.

### 1. Restrição de Infraestrutura: Streaming em Cluster Compartilhado
* **O Problema:** A intenção original era utilizar o **Databricks Auto Loader** (`cloud_files`) em modo Streaming para garantir checkpoints automáticos. Porém, o ambiente utilizado (Databricks Community Edition/Free Edition) opera em **Shared Clusters**, que bloqueiam permissões de baixo nível necessárias para a gestão de filas de arquivos do Auto Loader.
* **Erro Encontrado:** `[UNSUPPORTED_STREAMING_SOURCE_PERMISSION_ENFORCED]`
* **A Solução (Trade-off):** A arquitetura foi adaptada para **Batch Read**, mantendo o código modular. Aceitei a perda temporária do checkpoint automático em favor da execução funcional, entendendo que em um ambiente Enterprise (Single User Cluster), a chave `use_stream=True` reativaria a capacidade de streaming sem refatoração de código.

### 2. Serialização e Encoding (UTF-16)
* **O Problema:** A ingestão inicial dos arquivos CSV resultou em dados corrompidos ("Mojibake") e falha na identificação de colunas (todas lidas como `_c0`), pois o Spark assume `UTF-8` por padrão.
* **A Solução:** Foi implementado um tratamento específico no Reader do Spark para forçar o encoding correto detectado na origem (`UTF-16`), além da definição explícita de delimitadores e cabeçalhos.

### 3. Integridade do Schema (JSON Aninhado)
* **O Problema:** Arquivo JSON (Jogadores) e CSV (Eventos) continham estruturas complexas aninhadas e colunas com strings que representavam objetos JSON.
* **Decisão Arquitetural:** Seguindo o princípio da camada **Bronze** (preservar o dado bruto), optei por não "explodir" ou limpar esses JSONs na ingestão. Eles foram persistidos como Strings ou Structs brutos para serem tratados na camada Silver, garantindo que falhas de parser não interrompam o pipeline de ingestão.

---

## 🔮 Roadmap para Camada Silver (Transformação)

Com base no *Profiling* dos dados da camada Bronze, mapeei as seguintes necessidades de tratamento para a próxima fase:

| Tabela | Problema Identificado (Bronze) | Ação Planejada (Silver) |
| :--- | :--- | :--- |
| **Jogadores** | Coluna `height` contém caracteres sujos (ex: `> 200 cm`). | Limpeza de strings (Regex) e Cast para `Integer`. |
| **Jogadores** | Dados aninhados em estruturas JSON. | Parser e `Flatten` das colunas. |
| **Eventos** | Colunas `team`, `player`, `assist` são strings JSON (`{'id': 10...}`). | Uso de `from_json` para estruturar IDs e Nomes. |
| **Partidas** | Colunas de data sem padrão definido. | Padronização para `DateType` ou `Timestamp`. |
| **Geral** | Nomes de colunas fora do padrão (ex: CamelCase ou com pontos `league.season`). | Renomeação para `snake_case` (ex: `league_season`). |

---

## 🏗️ Fase 2: Transformação Raw (Bronze) -> Silver

Esta fase focou na **limpeza, padronização e estruturação** dos dados brutos. O objetivo foi transformar dados "caóticos" (Raw) em tabelas confiáveis, tipadas e otimizadas para análise (Silver), aplicando conceitos de *Schema Enforcement* e *Data Quality*.

### ⚔️ War Stories: Desafios Técnicos & Soluções

Durante a construção do pipeline, enfrentei inconsistências críticas nos dados de origem. Abaixo, detalho os cenários de "crise" e as soluções de engenharia aplicadas.

#### 1. O Desafio do "Encoding Híbrido" (Mojibake)
**O Problema:** A ingestão da tabela `partidas` falhou silenciosamente. Dados históricos (2011-2019) foram gerados em **UTF-16LE** (padrão Excel legado), enquanto dados recentes (2023) chegaram em **UTF-8**.
* **Sintoma:** Ao forçar uma leitura única, o Spark interpretou bytes UTF-8 como UTF-16, gerando caracteres chineses (ex: `㄰〵...`) na coluna de IDs. Isso é conhecido tecnicamente como *Mojibake*.
* **Impacto:** Corrupção total dos IDs e falha na tipagem (Integers viraram Strings).

**A Solução (Smart Ingestion Pattern):**
Desenvolvi uma função "Sniffer" (Farejadora) que inspeciona os primeiros bytes (Magic Bytes) de cada arquivo antes da leitura total.
* **Lógica:** Se o arquivo inicia com `\xff\xfe` (BOM), o pipeline aplica decoder UTF-16. Caso contrário, assume UTF-8.
* **Resultado:** Ingestão híbrida bem-sucedida, unificando arquivos com encodings diferentes no mesmo DataFrame via `unionByName`.

#### 2. Schema Drift & Conflito no Delta Lake
**O Problema:** Devido à ingestão corrompida anterior, o Delta Lake registrou a coluna `id` como `STRING` nos metadados. Ao corrigir o encoding, os dados chegaram corretamente como `INTEGER`.
* **Erro:** `[DELTA_FAILED_TO_MERGE_FIELDS] Failed to merge fields 'id' and 'id'.`
* **Conceito:** O Delta Lake protege a integridade do schema (Schema Enforcement), impedindo mudanças bruscas de tipo.

**A Solução:**
Implementação de uma estratégia de **Schema Evolution Controlada** na camada Bronze:
1.  Uso da opção `.option("overwriteSchema", "true")` para forçar a atualização dos metadados.
2.  Execução preventiva de `DROP TABLE` para limpar logs de transação contaminados em ambiente de desenvolvimento.

#### 3. Data Wrangling em "Fake JSONs" (Tabela Eventos)
**O Problema:** A tabela de eventos continha colunas (`player`, `team`, `time`, `assist`) que pareciam JSON, mas eram representações de dicionários Python (aspas simples `'` e `None` em vez de `null`). O parser nativo do Spark falhava.
* **A Solução:** Pipeline de higienização via Regex antes do parsing.
    * Substituição de aspas simples por duplas.
    * Tratamento de literais `None` para `null`.
    * Aplicação de `from_json` com Schema explícito (DDL) para garantir tipagem forte.
 
#### 4. Tratamento de JSONs Verdadeiros "Struct" (Tabela Jogadores)
**O Problema:** A tabela de jogadores continha uma coluna (`birth`) apresentada como JSON, utilizando aspas duplas `"` e `Null`. E dados incosistentes nas colunas `height` e `weight`.
* **A Solução:** Parser nativo do Spark juntamente com remoção de caracteres não numéricos.
  *  Regex Cleanning e Flattening.

#### 5. Regras de Negócio e Correção de Domínio
Para garantir a qualidade analítica na camada Silver, aplicamos regras de negócio corretivas:
* **Futebol Domain Check:** Na tabela `eventos`, detectei minutos negativos (ex: `-5`). Apliquei a função `abs()` (valor absoluto) assumindo erro de digitação na origem.
* **Entity Resolution:** Na tabela `times`, times brasileiros estavam marcados incorretamente como `national = False`. Apliquei regra condicional: `WHEN country = 'Brazil' THEN is_national = True`.
* **Tratamento de Strings Numéricas:** A coluna `score.fulltime.away` continha números formatados como string com ponto flutuante ("2.0"). Apliquei cast (String -> Int) ou regex para limpeza.

#### 6. Observabilidade e o Falso Positivo (Databricks Lakehouse Monitoring)
**O Problema:** A implementação do monitoramento de qualidade estatística via Databricks Unity Catalog apontou ausência de duplicatas em tabelas que, ao serem consultadas via PySpark, possuíam chaves primárias duplicadas.
* **A Causa:** Desalinhamento entre a "Verdade do Log" e a "Verdade do Momento". O Monitoramento operava como um Job agendado (foto do passado), enquanto o cluster consultava o estado atual corrompido por novas cargas.
* **A Solução:** Separação estrita de schemas. Foi criado o schema *Sidecar* `workspace_project.data_governance` exclusivo para isolar as tabelas de métricas (`_profile_metrics` e `_drift_metrics`), garantindo que ferramentas de BI não realizem scans acidentais em metadados. Monitores foram ajustados: *Snapshot* para tabelas estáticas (Dimensões) e *Time Series* para eventos (Fatos).

#### 7. O Paradigma da Deduplicação: Window Function vs dropDuplicates
**O Problema:** Identificação de linhas duplicadas na tabela `jogadores` e `torneios` na camada Silver, gerando distorção analítica. O uso do método nativo `dropDuplicates()` do Spark traria não-determinismo (risco de manter a versão desatualizada de um dado).
* **A Solução (Jogadores - SCD Type 1):** Implementação de uma função modular utilizando `Window Function` (ordenando por `ingestion_date` DESC) para garantir a extração do *Golden Record* (registro mais recente). Após a limpeza em memória, os dados são persistidos no Delta Lake utilizando a instrução transacional `MERGE INTO` (Upsert), garantindo atualização de registros existentes e inserção de novos.
* **A Solução (Torneios - Chave Composta):** Prevenção de perda de dados históricos (SCD Tipo 2). A função de deduplicação foi parametrizada para atuar sobre uma Chave Primária Composta (`tournament_id` + `season_year`), preservando o histórico de todas as temporadas de um mesmo campeonato.

---

### 🛠️ Decisões de Arquitetura (Design Patterns)

1.  **Schema Contract (.select vs .withColumn):**
    * Adotamos o uso estrito de `.select()` na transição para Silver.
    * *Por que?* Isso funciona como um "Contrato de Dados". Apenas colunas explicitamente listadas e tipadas entram na Silver. Colunas "lixo" ou temporárias da Bronze são descartadas automaticamente, garantindo uma tabela limpa.

2.  **Chaves Substitutas (Surrogate Keys):**
    * Geramos chaves internas (`sk`) usando `monotonically_increasing_id()`.
    * *Motivo:* Desacoplar o Data Lake dos IDs do sistema de origem, protegendo contra duplicidade ou mudanças de chaves no legado.

3.  **Linhagem de Dados (Data Lineage):**
    * Todas as tabelas Silver mantêm as colunas `source_file` e `ingestion_date`.
    * *Benefício:* Rastreabilidade total. Se um dado estiver errado no Dashboard, sabemos exatamente qual arquivo CSV/JSON originou o erro e quando foi processado.

4.  **FinOps & Otimização:**
    * Conversão de tipos `BigInt` (padrão Spark) para `Integer` onde o domínio de dados permite, reduzindo o tamanho do armazenamento e custo de I/O.
    * Armazenamento em formato **Delta Lake** (Parquet comprimido com Snappy) para leitura colunar otimizada.

5.  **Data Quality Constraints (Fail-Fast):**
    * Aplicamos restrições físicas no banco de dados através do Unity Catalog (ex: `ALTER TABLE ... ADD CONSTRAINT pk_jogadores PRIMARY KEY (player_id)`).
    * *Benefício:* Mudança de uma governança reativa (apagar duplicatas no código) para uma governança ativa (o banco de dados rejeita transações que ferem a integridade relacional, economizando processamento e evitando falhas silenciosas).    

---

### ⚙️ Architecture Decision Record (ADR): Estratégia de Particionamento Físico

**Contexto:**
Durante o design da camada Silver e Gold, avaliei a necessidade de particionar fisicamente as tabelas do Data Lake (ex: `PARTITION BY season_year` ou `match_date`), uma prática comum para acelerar a leitura de dados via *Partition Pruning*.

**Decisão:**
Optei por **NÃO PARTICIONAR** fisicamente as tabelas deste projeto (Times, Jogadores, Partidas, Eventos, Torneios e Estádios).

**Justificativa Técnica (O Problema dos Pequenos Arquivos):**
Em Engenharia de Dados, o particionamento é recomendado exclusivamente para tabelas massivas onde cada partição física resulte em diretórios com, no mínimo, **1 GB a 2 GB de dados**. 
O dataset do domínio de Futebol é de baixa volumetria (megabytes por temporada). Se fosse aplicado o particionamento por ano:
1. Seria criado micro-arquivos (alguns KBs) para cada temporada.
2. Geraria um *Metadata Overhead* massivo: o Apache Spark gastaria muito mais tempo e processamento listando diretórios recursivamente do que efetivamente lendo os dados.
3. Degradaria a performance de leitura global (Small Files Problem).

**Estratégia Adotada (Modern Data Stack):**
Para garantir a otimização das consultas analíticas na camada Gold sem incorrer no erro de *over-partitioning*, utilizarei os recursos nativos de indexação do Delta Lake.
* Execução diária do comando `OPTIMIZE` para consolidar os dados em arquivos Parquet de tamanho ideal.
* Aplicação de `ZORDER BY (match_id, match_season_year)` nas tabelas de Fato (`partidas` e `eventos`) para co-localizar dados relacionados fisicamente, permitindo *Data Skipping* dinâmico em consultas de BI sem a necessidade de pastas físicas.
