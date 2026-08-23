# Data Mart de Estimativas de Chegadas de Turistas Internacionais ao Brasil

Este projeto consiste em um Data Mart em arquitetura dimensional (Star Schema) desenvolvido sobre o **PostgreSQL**, populado e orquestrado via **Pentaho Data Integration (PDI)** e consumido no **Power BI**. O objetivo é consolidar dados brutos de chegadas de turistas para análises analíticas de Business Intelligence e consultas OLAP.

## 📊 Diagrama do Modelo Dimensional

<img width="1150" height="990" alt="DataMart - Estimativas de chegadas de turistas internacionais ao Brasil" src="https://github.com/user-attachments/assets/1c996d02-07d8-4cbf-91fe-aaa0e19a223d" />

---

## 🏗️ Arquitetura e Pipeline de Dados

O projeto é estruturado em três camadas principais:

[CSV Bruto] ──> [Staging Area (PostgreSQL)] ──> [Pentaho PDI (ETL)] ──> [Star Schema (PostgreSQL)] ──> [Power BI]
### 1. Camada de Staging (`staging_turistas_internacionais`)
Atua como *Landing Zone* flexível em `VARCHAR(150)` para ingestão segura dos arquivos CSV (delimitados por `;`).

### 2. Camada ETL (Pentaho Data Integration / Kettle)
A transformação e carga dos dados são orquestradas no Pentaho:
* **`tr_carga_dimensoes.ktr`**: Trata strings com `TRIM` e popula as tabelas de dimensões gerando as *Surrogate Keys*.
* **`tr_carga_fato.ktr`**: Lê a staging, realiza os de/para (*Database Lookup*) com as dimensões para capturar os IDs numéricos e grava na fato.
* **`jb_master_pipeline.kjb`**: Job mestre responsável por executar a sequência de carga automatizada.

### 3. Camada Multidimensional (Star Schema)
* **`dim_localidade`**: Origem (`pais_origem`) e entrada (`uf_destino`).
* **`dim_via_acesso`**: Meio de transporte de entrada (`descricao_via`).
* **`dim_tempo`**: Competência temporal (`ano` em INT, `mes` em VARCHAR).
* **`fato_chegadas`**: Tabela central contendo as Foreign Keys e a métrica `quantidade_chegadas` (sem chave primária técnica redundante).

---

## 📁 Estrutura do Repositório

```text
├── SQL/                       # Scripts DDL e DML de criação e carga
│   ├── 01_create_dimensions.sql
│   ├── 02_create_facts.sql
│   └── 03_populate_data_mart.sql
├── pentaho/                   # Pipelines (.ktr) e Job (.kjb) do PDI
│   ├── tr_carga_dimensoes.ktr
│   ├── tr_carga_fato.ktr
│   └── jb_master_pipeline.kjb
├── power_bi/                  # Arquivo do dashboard (.pbix) e visuais
├── docs/                      # Diagrama ERD e dicionário de dados
├── .gitignore                 # Filtros para o Git
└── README.md                  # Documentação principal
📖 Dicionário de Dados1. dim_localidadeNome da ColunaTipoChaveDescriçãoid_localidadeSERIALPKSurrogate Key gerada para a combinação de origem/destino.pais_origemVARCHAR(100)-País de residência do turista.uf_destinoVARCHAR(100)-Estado (UF) brasileiro de entrada.2. dim_via_acessoNome da ColunaTipoChaveDescriçãoid_via_acessoSERIALPKSurrogate Key do meio de transporte.descricao_viaVARCHAR(100)-Meio de transporte (Aérea, Terrestre, Marítima, Fluvial).3. dim_tempoNome da ColunaTipoChaveDescriçãoid_tempoSERIALPKSurrogate Key do período.anoINT-Ano civil (4 dígitos).mesVARCHAR(50)-Nome por extenso do mês.4. fato_chegadasNome da ColunaTipoChaveDescriçãoid_localidadeINTFKRelacionamento com dim_localidade.id_via_acessoINTFKRelacionamento com dim_via_acesso.id_tempoINTFKRelacionamento com dim_tempo.quantidade_chegadasINTMétrica
