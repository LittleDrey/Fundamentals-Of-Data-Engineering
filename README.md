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
