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
